---
title: "ローカルLLMコードレビューで3つのベクトルDBを比較した — TiDB / ChromaDB / Pinecone"
emoji: "📊"
type: "tech"
topics: ["tidb", "rag", "llm", "vectordatabase", "swift"]
published: false
---

## はじめに

iOS/macOS アプリを個人開発中！

**1Take** — ミュージシャン向け録音 iOS アプリ。録音ボタン 1 つで LA-2A / 1176 系のリアルタイムコンプレッサー掛け取り。

https://apps.apple.com/us/app/1take/id6757945099

**M2DX** — iOS/macOS 向け MIDI 2.0 対応 DX7 互換 FM シンセサイザー。TestFlight で公開ベータ配布中。

https://testflight.apple.com/join/BAtGszPw

---

M2DX の開発中に「ローカル LLM でコードレビューを自動化できないか」を試し始めたのがシリーズの始まり。

（１）では llama 系を中心に 10 機種試して、指摘 52 件・真陽性 0 件という結果だった。
（２）では Swift 仕様書を RAG に入れて誤検出を 76% 減らした。
（５）では git push のたびにレビュー・採点・合成が自動で走るパイプラインを作り、**421 件**のレビューが溜まった。

で、今回。

（２）の RAG（= 外部 DB から関連情報を引いてプロンプトに注入する手法）は「Swift の言語仕様」を注入した。でも 421 件のほうは「実際のコードに対してどんなレビューをすべきか」のパターン集だ。仕様書とは別物で、「こういう diff が来たらこういう観点で見る」という具体事例になっている。

これを RAG に入れたらどうなるか。ついでに **TiDB Serverless / ChromaDB / Pinecone Serverless の 3 択**で同じデータを入れて性能差も測ってみた。

---

## パイプライン構成

```
git diff
  └─► review.py (llama3.3:70b-m2lora-v1, Ollama)
        └─► retriever.py
              ├─ bge-large:latest で diff をベクトル化（1024次元）
              ├─ TiDB / Chroma / Pinecone で類似 diff 上位 2 件を取得
              └─ 類似レビュー例をプロンプトに注入
                    └─► evaluate.py (Claude / Codex / Gemini で採点)
```

収集済みデータ: **421件のSwift/MIDIコードレビュー**（M2DX + MIDI2Kitの実コミット）

embedding モデルは bge-large:latest（1024次元）をOllama でローカル実行。コード特化じゃないけど「この変更が何をしようとしているか」の意味的な類似度で引いてくるには十分だった。

---

## ベクトルDB 3択のセットアップ

### TiDB Serverless

TiDB v8.5.3 からネイティブの `VECTOR(1024)` 型と HNSW インデックス（= グラフ構造で近傍を高速探索するアルゴリズム。ほぼ全ベクトルDBが採用してる。らしい）が使えるようになった。MySQL プロトコル互換なので `pymysql` + SSL で繋がる。

```sql
CREATE TABLE reviews (
    id              VARCHAR(36)   NOT NULL PRIMARY KEY,
    code_diff       MEDIUMTEXT    NOT NULL,
    synthesized_review MEDIUMTEXT,
    claude_score    FLOAT,
    codex_score     FLOAT,
    gemini_score    FLOAT,
    flagged         TINYINT(1)    DEFAULT 0,
    review_model    VARCHAR(64),
    diff_embedding  VECTOR(1024),
    created_at      DATETIME      DEFAULT CURRENT_TIMESTAMP,
    VECTOR INDEX idx_diff_emb ((VEC_COSINE_DISTANCE(diff_embedding)))
        USING HNSW
);
```

早速 TiDB の癖にハマった。

```sql
SELECT ... FROM reviews
WHERE flagged = 0
ORDER BY VEC_COSINE_DISTANCE(diff_embedding, %s) ASC
LIMIT 10
```

これがエラーになる。HNSW インデックスと WHERE フィルタが同時に使えない（v8.5.3 時点）。ANN 検索（= 厳密な全探索をやめて高速に近傍を近似取得する方式）とメタデータフィルタの同時使用が未サポートらしい。。。

回避策: フィルタなしで多めに取ってきて Python 側でフィルタ。

```python
cur.execute(sql, (vec_str, top_k * 5))
rows = [r for r in cur.fetchall() if not r["flagged"]][:top_k]
```

### ChromaDB（ローカル）

```python
import chromadb

client = chromadb.PersistentClient(path="./data/chroma")
collection = client.get_or_create_collection(
    name="reviews",
    metadata={"hnsw:space": "cosine"},
)
```

以上。`pip install chromadb` して 3 行。WHERE フィルタと ANN の同時使用も問題なし。セットアップコストほぼゼロ。ローカルで簡単。一番楽。ただしローカルファイルなのでマシンをまたぐには別途対応が必要。

### Pinecone Serverless

```python
pc = Pinecone(api_key=os.environ["PINECONE_API_KEY"])
pc.create_index(
    name="m2lora-reviews",
    dimension=1024,
    metric="cosine",
    spec=ServerlessSpec(cloud="aws", region="us-east-1"),
)
```

Starter プランはリージョンが `us-east-1` 固定。東京から遠いのが気になる。

---

## embedding のハマり

bge-large は 512 トークンの入力制限がある。コードの diff はトークン密度が高くて普通に投げると `the input length exceeds the context length` が返ってくる。

文字数で段階的に切り詰めながら retry する方式に落ち着いた。1200 → 800 → 500 → 300 文字と縮めていって成功したら返す。実際には 1200 文字でほぼ通る。

```python
async def embed(text: str) -> list[float]:
    for limit in (1200, 800, 500, 300):
        async with aiohttp.ClientSession() as session:
            async with session.post(
                "http://localhost:11434/api/embed",
                json={"model": "bge-large:latest", "input": text[:limit]},
                timeout=aiohttp.ClientTimeout(total=60),
            ) as resp:
                data = await resp.json()
        if "embeddings" in data:
            return data["embeddings"][0]
        if data.get("error", "").startswith("the input length"):
            continue
    raise ValueError("embed failed at all truncation levels")
```

ウォームアップ後は 35〜45ms/回 で安定した（Apple M2 MacBook Pro）。

### RAG 注入

類似 diff 上位 2 件の `synthesized_review`（= Claude / Codex / Gemini の 3 者評価を Claude が統合した合成レビュー）をプロンプトの頭に差し込む。

```python
rag_context = f"=== 類似コードの過去レビュー例 ===\n{examples}\n=== END ===\n\n"
prompt = f"あなたはSwift/MIDIコードレビューの専門家です。\n{rag_context}..."
```

---

## ベンチマーク① — DB 単体の性能を測る

ちゃんと比較したかったので、embedding（Ollama bge-large）は事前にキャッシュして DB の処理だけを計測する設計にした。

...と思ったら Phase 0 のキャッシュ計算が `_original_embed` を直接呼んでいてキャッシュに書き込まれていなかった。「embed cache 保存: 0 件」になってて焦った。。。`_cached_embed` 経由に直したら正常に動作。こういうバグに気づかないまま計測してたら嫌だった。

データは SQLite から `avg_score ≥ 4.0` の高品質レビュー 94 件を抽出。70 件を ingest、20 件で検索性能を計測。

p50/p95/p99 はレイテンシの分布指標（小さい順に並べたとき 50番目/95番目/99番目の値）。平均値より外れ値の影響を受けにくいのでレイテンシ計測でよく使われる。

### Ingest スループット（70件）

| DB | 総時間 | スループット |
|---|---|---|
| ChromaDB（ローカル） | 26.8 秒 | **2.6 件/秒** |
| TiDB Serverless | 79.4 秒 | 0.9 件/秒 |
| Pinecone Serverless | 111.8 秒 | 0.6 件/秒 |

ローカル vs クラウドの差がそのまま出た。クラウド同士だと TiDB が Pinecone の 1.5 倍速い。

### 検索レイテンシ（20クエリ、embedding時間を除くDB単体）

| DB | p50 | p95 | p99 |
|---|---|---|---|
| ChromaDB（ローカル） | **42 ms** | 396 ms | 414 ms |
| TiDB Serverless | 1,157 ms | 1,260 ms | 1,264 ms |
| Pinecone Serverless | 1,623 ms | 2,025 ms | 2,027 ms |

ChromaDB の p95 が 396ms と高いのは、ローカル HNSW の初回インデックス展開が走るから。2回目以降は低レイテンシに安定する。

クラウド同士だと TiDB が Pinecone より約 30% 速かった。us-east-1 固定の影響が数字に出てる。

### Top-K 一致率（TiDB 基準）

3 DB が返す上位 5 件のどれだけが一致するか（コサイン距離 = ベクトルの向きの差を 0〜2 で表す指標で近い順に取ってくる）。

| 比較対象 | TiDBとの一致率 |
|---|---|
| ChromaDB | **99%** |
| Pinecone | **98%** |

どの DB を選んでも検索品質はほぼ変わらない。同じ HNSW + コサイン距離で実装されてるんだからそりゃそうか。。という結果だった。

---

## ベンチマーク② — HTAP。TiDB が「1本で済む」ことを測る

これが今回一番面白かった実験。

### 2システム問題

ChromaDB はベクトル専用 DB なので `AVG(score) GROUP BY review_model` みたいな分析クエリが書けない。だから実運用だと SQLite + ChromaDB の 2 システム構成になりがちだ。

```
書き込み:
  ├─► SQLite  INSERT（スコア・メタデータ管理）
  └─► ChromaDB upsert（ベクトル検索）

読み取り:
  ├─► SQLite  SELECT AVG(score)...（分析クエリ）
  └─► ChromaDB query(vec, top_k)（ベクトル検索）
```

2か所に書いて2か所から読む。同期コストも高いし、片方だけ書き込み失敗したときの不整合も怖い。

### TiDB は SQL + ベクトルが 1 クエリで完結する

```sql
SELECT review_model,
       AVG((claude_score + codex_score + gemini_score) / 3.0) AS avg_score,
       COUNT(*) AS cnt
FROM reviews
WHERE diff_embedding IS NOT NULL
GROUP BY review_model
```

これと同じ DB にベクトルも入っている。1 システムで完結。

TiDB の HTAP（= トランザクションと分析を同一 DB で同時にこなすアーキテクチャ）は行ストア（TiKV）と列ストア（TiFlash）を内部で分離していて、INSERT と AVG/GROUP BY が互いに干渉しない。。。らしい。本当にそうなのか測ってみた。

### 計測: 書き込みしながら分析クエリを叩く

INSERT を一定レートで投入しながら、分析クエリ（`AVG(score) GROUP BY review_model`）を 3 秒ごとに実行してレイテンシを計測。負荷を 0 → 10 → 30 件/分と変えて影響を見る。

**OLAP クエリ p50 レイテンシ（ms）**

| INSERT 負荷 | TiDB | SQLite | ChromaDB（別システム） |
|:-----------:|:----:|:------:|:---------------------:|
| 0 件/分（ベースライン） | 15.1 | 3.5 | 1.3 |
| 10 件/分 | 14.5 | 5.0 | 1.5 |
| 30 件/分 | **14.1** | 3.6 | 1.5 |

TiDB、**全く劣化しない**。30件/分 INSERT しながらでも 14ms で安定している。

SQLite は 10件/分 で 5.0ms に増加（ライトロックの競合）。そして ChromaDB への別クエリも必要になる。

```
ChromaDB + SQLite:  OLAP 3.5ms ✅  Vector 1.3ms ✅  2システム管理 ❌
TiDB Serverless:    OLAP  15ms ✅  Vector 1.2s  ✅  1システムで完結 ✅
```

絶対値では SQLite が速い。でも「書きながら集計する」用途でクラウド運用するなら TiDB の 1 本管理はかなり強い。

---

## RAG 品質比較（実コミット20件）

M2DX・MIDI2Kit の実コミット 20 件に対して RAGなし / TiDB / ChromaDB / Pinecone で生成したレビューを Claude / Codex / Gemini の平均スコアで比較。

| | RAGなし | TiDB | ChromaDB | Pinecone |
|---|:---:|:---:|:---:|:---:|
| **平均スコア** | 4.27 | 5.30 | 5.77 | **5.87** |
| **平均Δ** | — | +1.03 | +1.50 | **+1.60** |
| **改善件数** | — | 13/20 (65%) | 12/20 (60%) | **15/20 (75%)** |

全 DB で改善した。Pinecone が数字上は最良だけど、Top-K 一致率 98〜99% だから DB の差というより誤差の範囲だと思う。

効かなかった 5〜7 件は RAGなしでも 7 点以上のコミットが多かった。もともと高品質な diff に RAG を足してもコンテキストがノイズになるだけ、という天井効果っぽい。

---

## どの DB を選ぶか

| 観点 | TiDB | ChromaDB | Pinecone |
|---|---|---|---|
| Ingest速度 | 0.9件/秒 | **2.6件/秒** | 0.6件/秒 |
| 検索p50 | 1,157ms | **42ms** | 1,623ms |
| 検索品質（Top-5一致率） | 基準 | 99% | 98% |
| OLAPクエリ | **SQL標準 ✅** | 不可 ❌ | 不可 ❌ |
| HTAP（書き込み負荷耐性） | **劣化なし ✅** | — | — |
| 無料枠 | Serverless 5GiB | ローカル無制限 | Starter 2GiB |
| MySQLプロトコル互換 | **✅** | ❌ | ❌ |

- **プロトタイプ・ローカル** → ChromaDB 一択。`pip install` して終わり
- **クラウド本番・チーム共有** → TiDB（Pinecone より速くて SQL とベクトルが 1 本）
- **既存 MySQL アプリに追加** → TiDB（JOIN・集計がそのまま使える）

---

## まとめ

| 検証内容 | 結論 |
|---|---|
| 3DB の検索品質 | Top-5 一致率 98〜99%。DB で品質は変わらない |
| 3DB の Ingest 速度 | ChromaDB 2.6件/秒 > TiDB 0.9 > Pinecone 0.6 |
| 3DB の検索レイテンシ | ChromaDB 42ms ≪ TiDB 1,157ms < Pinecone 1,623ms |
| RAG 効果 | 全 DB 平均 +1〜+1.6 点改善。75% のコミットで有効 |
| HTAP | TiDB: 30件/分 INSERT 中も OLAP 14ms で安定。SQLite + Chroma は 2 システム管理が必要 |

まずは ChromaDB でプロトタイプして、スケールアウトや分析が必要になったら TiDB に移行、が現実的かな。LoRA のデータ収集みたいに「書きながら集計もしたい」ユースケースには TiDB の相性がよさそう。

コード: https://github.com/hakaru-dev/M2LoRA（記事公開後にpublic化予定）

まだまだ続く。。。

---

:::message
**シリーズの他の記事**

- [（１）監査編 — 指摘 52 件、真陽性 0 件](https://zenn.dev/hakaru/articles/m2dx-local-llm-audit-zero-true-positives)
- [（２）RAG 編 — Swift 仕様書で誤検出 76% 削減](https://zenn.dev/hakaru/articles/m2dx-local-llm-audit-rag)
- [（３）aider 編 — 参照軸は改善、知識軸は依然ダメ](https://zenn.dev/hakaru/articles/m2dx-local-llm-agentic-harness-eval)
- [（４）LoRA 編 — 誤検知 93% 削減](https://zenn.dev/hakaru/articles/swift-audit-lora-fp-reduction)
- [（５）M2LoRA パイプライン編 — 開発しながら自動でデータが貯まる仕組み](https://zenn.dev/hakaru/articles/m2lora-code-review-pipeline)
- [（番外）ベクトルDB比較 RAG編 — TiDB/Chroma/Pinecone](https://zenn.dev/hakaru/articles/tidb-rag-code-review-vector-comparison)
:::

*本記事は [Zennfes Spring 2026 × TiDB](https://zenn.dev/contests/zennfes-spring-2026-tidb) への応募作品です。*
