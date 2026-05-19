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

ローカルLLMにコードレビューをさせると、**幻覚（Hallucination）** が混入することがある。「`@Bindable`はSwift 5.9で非推奨」「このキャストはオーバーフローの危険がある」——どちらも間違いだ。指摘が具体的に見えるほど、間違いに気づきにくい。

この問題を緩和するために試みたのが **RAG**（Retrieval-Augmented Generation）だ。

> **RAG（検索拡張生成）とは**
> LLMに回答を生成させる前に、関連する情報を外部DBから検索してプロンプトに注入する手法。LLMが「知らないはずの情報」を参照できるようになり、幻覚が減りやすい。

本記事では、Swiftコードレビューの自動収集・評価システム **M2LoRA** を題材に、次の3点を実測で比較する。

1. **ベクトルDB 3択の検索品質**（TiDB / ChromaDB / Pinecone）
2. **ベクトルDB 3択の性能**（Ingest スループット・検索レイテンシ）
3. **HTAPの実証**（書き込み負荷下でのOLAPクエリ劣化を計測）

---

## システム構成

### M2LoRA パイプライン

```
git diff
  └─► review.py (llama3.3:70b-m2lora-v1, Ollama)
        └─► retriever.py
              ├─ bge-large:latest でdiffをembed (1024次元)
              ├─ TiDB / Chroma / Pinecone で類似diff上位2件を取得
              └─ 類似レビュー例をプロンプトに注入
                    └─► evaluate.py (Claude / Codex / Gemini で採点)
```

収集済みデータ: **421件のSwift/MIDIコードレビュー**（M2DX + MIDI2Kitの実際のコミット）

> **Embedding（埋め込み）とは**
> テキストを固定長の数値ベクトルに変換する処理。意味的に似たテキストは近いベクトルになる。ここでは `bge-large:latest`（1024次元）をOllamaでローカル実行。

---

## ベクトルDBのセットアップ

### TiDB Serverless

TiDB v8.5.3 からネイティブの `VECTOR` 型と **HNSWインデックス** が使える。

> **HNSW（Hierarchical Navigable Small World）とは**
> 高次元ベクトルの近傍探索アルゴリズム。グラフ構造を階層的に構築し、O(log n) に近い速度で類似ベクトルを探せる。ほとんどのベクトルDBが採用している。

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

接続は `pymysql` + SSL（TiDB ServerlessはMySQLプロトコル互換）:

```python
import pymysql

conn = pymysql.connect(
    host="gateway01.ap-northeast-1.prod.aws.tidbcloud.com",
    port=4000,
    user="<user>",
    password="<password>",
    database="m2lora",
    ssl={"ca": "/etc/ssl/cert.pem"},
)
```

**ハマりポイント**: HNSWインデックスは `WHERE flagged = 0` のような事前フィルタと非互換（v8.5.3時点）。上位 `top_k * 5` 件を取得してPython側でフィルタする方式に変更した。

```python
sql = """
    SELECT commit_hash, code_diff, synthesized_review, flagged,
           VEC_COSINE_DISTANCE(diff_embedding, %s) AS dist
    FROM reviews
    WHERE diff_embedding IS NOT NULL
    ORDER BY dist ASC
    LIMIT %s
"""
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

セットアップが最もシンプル。`pip install chromadb` だけ動く。ただしローカルストレージなので複数マシン間での共有は別途対応が必要。

### Pinecone Serverless

```python
from pinecone import Pinecone, ServerlessSpec

pc = Pinecone(api_key=os.environ["PINECONE_API_KEY"])
pc.create_index(
    name="m2lora-reviews",
    dimension=1024,
    metric="cosine",
    spec=ServerlessSpec(cloud="aws", region="us-east-1"),
)
```

マネージドクラウドサービス。APIキー1本で使えるが、検索クエリがus-east-1経由になるため日本からのレイテンシは大きい。

---

## Embedding とRAG注入

### diff を embed してベクトル化

`bge-large` は512トークン制限があり、コードdiffはトークン密度が高い。文字数で段階的に切り詰めるフォールバックを実装した。

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

計測したところ、**初回ウォームアップ後は約35〜45ms/回**で安定した（Apple M2 MacBook Pro）。

### プロンプトへの注入

類似diff上位2件の `synthesized_review`（Claude合成済みの高品質レビュー）をプロンプトに差し込む。

```python
async def review_diff(diff: str) -> str:
    similar = await find_similar(diff, top_k=2)
    rag_context = ""
    if similar:
        examples = "\n".join(
            f"--- 例{i+1} ---\n{r.synthesized_review[:400]}"
            for i, r in enumerate(similar)
        )
        rag_context = f"=== 類似コードの過去レビュー例 ===\n{examples}\n=== END ===\n\n"

    prompt = f"""\
あなたはSwift/MIDIコードレビューの専門家です。
{rag_context}
=== コードdiff ===
{diff[:500]}
=== END ===

レビュー:"""
    return await _generate(prompt)
```

---

## ベンチマーク①: ベクトルDB性能比較

### 計測方法

- データ: SQLiteから `avg_score ≥ 4.0` の高品質レビュー94件を使用
- **Ingest**: 70件を各DBにupsert、スループット（件/秒）を計測
- **Search**: 20件のdiffで類似検索、レイテンシの分布を計測
- embed（Ollama bge-large）は事前にキャッシュし、**DB単体の性能を分離**して計測

> **p50/p95/p99（パーセンタイル）とは**
> 100件の計測値を小さい順に並べたとき、50番目がp50（中央値）、95番目がp95、99番目がp99。平均値と違い外れ値の影響を受けにくく、レイテンシ計測の標準指標。

### Ingest スループット

| DB | 総時間 | スループット |
|---|---|---|
| ChromaDB（ローカル） | 26.8 秒 | **2.6 件/秒** |
| TiDB Serverless | 79.4 秒 | 0.9 件/秒 |
| Pinecone Serverless | 111.8 秒 | 0.6 件/秒 |

ChromaDB がローカルのため圧倒的に速い。クラウド同士ではTiDBがPineconeの**1.5倍のスループット**。

### 検索レイテンシ（embed時間を除くDB単体）

| DB | p50 | p95 | p99 |
|---|---|---|---|
| ChromaDB（ローカル） | **42 ms** | 396 ms | 414 ms |
| TiDB Serverless | 1,157 ms | 1,260 ms | 1,264 ms |
| Pinecone Serverless | 1,623 ms | 2,025 ms | 2,027 ms |

ChromaDBのp95が396msと高分散なのはローカルHNSWのJIT的な初期化によるもの（2回目以降は低レイテンシに安定）。クラウド同士ではTiDBがPineconeより**約30%速い**。

> **コサイン距離とは**
> 2つのベクトルの向きの差を0〜2の値で表す指標（0=完全一致、2=逆向き）。ベクトルの大きさを無視して「意味の近さ」を比較できるため、テキスト検索で広く使われる。

### Top-K一致率（TiDB基準）

3DBが返す上位5件の重複度を比較した。

| 比較対象 | TiDBとの一致率 |
|---|---|
| ChromaDB | **99%** |
| Pinecone | **98%** |

**3DBの検索品質はほぼ同等。** 同じHNSW+コサイン距離を使っている限り、DBが違っても返す結果はほぼ一致する。どのDBを選んでも検索品質の差で悩む必要はない。

---

## ベンチマーク②: HTAP — TiDBが「1本で済む」理由

### HTAP とは

> **HTAP（Hybrid Transactional/Analytical Processing）とは**
> 同一DBでトランザクション処理（OLTP: INSERT/UPDATE）と分析処理（OLAP: AVG/GROUP BY）を同時にこなすアーキテクチャ。TiDBはOLTP用の行ストア（TiKV）とOLAP用の列ストア（TiFlash）を内部で持ち、2種類のワークロードを分離して処理する。

### ChromaDB + SQLite「2システム問題」

ChromaDBはベクトル専用DBのため、`AVG(score) GROUP BY review_model` のような分析クエリは書けない。そのため実運用では**SQLiteとChromaDBを並列に管理する2システム構成**になりがちだ。

```
書き込み時:
  ├─► SQLite  INSERT (スコア・メタデータ管理)
  └─► ChromaDB upsert (ベクトル検索)

読み取り時:
  ├─► SQLite  SELECT AVG(score)...  (分析クエリ)
  └─► ChromaDB query(vec, top_k)   (ベクトル検索)
```

2システムを同期し続けるのは運用コストだ。書き込みの一方が失敗したときの不整合も起きうる。

### TiDBなら1本で完結

```python
# ベクトル検索 + SQLフィルタ + 集計 — 全部同一DBで完結
sql = """
    SELECT review_model,
           AVG((claude_score + codex_score + gemini_score) / 3.0) AS avg_score,
           COUNT(*) AS cnt,
           VEC_COSINE_DISTANCE(diff_embedding, %s) AS dist
    FROM reviews
    WHERE diff_embedding IS NOT NULL
    ORDER BY dist ASC
    LIMIT 5
"""
```

### 計測: 書き込み負荷下でのOLAPレイテンシ

INSERT を秒あたり `N` 件投入しながら、分析クエリ（`AVG(score) GROUP BY review_model`）を3秒ごとに実行してレイテンシを計測した。

> **OLTP / OLAP とは**
> - **OLTP（オンライントランザクション処理）**: 頻繁な INSERT/UPDATE。バックフィルやリアルタイム書き込みがこれに相当。
> - **OLAP（オンライン分析処理）**: 大量レコードの集計・分析クエリ（AVG, GROUP BY, COUNT など）。

**OLAP クエリ p50 レイテンシ（ms）**

| INSERT 負荷 | TiDB | SQLite | ChromaDB（別システム） |
|:-----------:|:----:|:------:|:---------------------:|
| 0 件/分（ベースライン） | 15.1 | 3.5 | 1.3 |
| 10 件/分 | 14.5 | 5.0 | 1.5 |
| 30 件/分 | **14.1** | 3.6 | 1.5 |

**TiDBのOLAPレイテンシはINSERT負荷でほぼ変化しない**（15.1 → 14.1ms）。

これはTiDBのHTAPアーキテクチャが行ストア（TiKV）と列ストア（TiFlash）を分離しているため、書き込みが分析クエリの実行パスに干渉しないためだ。

SQLiteは単純なローカルファイルDBなので生のOLAPは速い（3.5ms）が、10件/分の書き込みで5.0msに増加（ライトロックの競合）し、ChromaDBへの追加接続も必要になる。

### アーキテクチャ比較まとめ

```
ChromaDB + SQLite:  OLAP 3.5ms ✅  Vector 1.3ms ✅  ただし 2システム管理 ❌
TiDB Serverless:    OLAP  15ms ✅  Vector 1.2s  ✅  1システムで完結 ✅
```

絶対速度ではSQLite+ChromaDBが速いが、**クラウド運用・スケールアウト・HTAP**を求めるならTiDBが合理的な選択になる。

---

## RAG品質比較（実コミット20件）

M2DX・MIDI2Kit の実コミット20件を3評価者（Claude / Codex / Gemini）の平均スコアで評価した。

### 集計結果

| | RAGなし | TiDB | ChromaDB | Pinecone |
|---|:---:|:---:|:---:|:---:|
| **平均スコア** | 4.27 | 5.30 | 5.77 | **5.87** |
| **平均Δ** | — | +1.03 | +1.50 | **+1.60** |
| **改善件数** | — | 13/20 (65%) | 12/20 (60%) | **15/20 (75%)** |

全3DBでRAG注入が有効（平均+1〜+1.6点）。Pineconeが最も安定したが、**前章のベンチマーク①で3DBの検索品質はほぼ同等（99%一致）と確認済みのため、今回の差は誤差の範囲**と考えられる。

### RAGが効かなかったケース

改善しなかった5〜7件を見ると、**RAGなしのスコアが既に7点以上**のケースが多かった。類似事例のコンテキストがノイズになる「天井効果」と思われる。

---

## 考察: どのDBを選ぶか

| 観点 | TiDB | ChromaDB | Pinecone |
|---|---|---|---|
| Ingest速度 | 0.9件/秒 | **2.6件/秒** | 0.6件/秒 |
| 検索p50 | 1,157ms | **42ms** | 1,623ms |
| 検索品質（Top-5一致率） | 基準 | 99% | 98% |
| OLAPクエリ | **SQL標準 ✅** | 不可 ❌ | 不可 ❌ |
| HTAP（書き込み負荷耐性） | **劣化なし ✅** | — | — |
| 無料枠 | Serverless 5GiB | ローカル無制限 | Starter 2GiB |
| MySQLプロトコル互換 | **✅** | ❌ | ❌ |

**選択指針:**

- **ローカル開発・プロトタイプ** → ChromaDB（`pip install` だけ、設定ゼロ）
- **クラウド本番・チーム共有** → TiDB（Pineconeより速く、SQLとベクトルが1本）
- **既存MySQLアプリへの追加** → TiDB（テーブル構造やJOINをそのまま流用できる）

---

## まとめ

| 検証内容 | 結論 |
|---|---|
| 3DB の検索品質 | Top-5一致率98〜99%。品質差は誤差の範囲 |
| 3DB のIngest速度 | ChromaDB 2.6件/秒 > TiDB 0.9 > Pinecone 0.6 |
| 3DB の検索レイテンシ | ChromaDB 42ms ≪ TiDB 1,157ms < Pinecone 1,623ms |
| RAG効果 | 全DB平均+1〜+1.6点の改善。75%のコミットで有効 |
| HTAP | TiDB: 30件/分のINSERT中もOLAP 14msで安定。SQLite+Chromaは2システム管理が必要 |

ローカルLLMのRAG強化としては、**まずChromaDBで試して、スケール・共有・分析が必要になったらTiDBへ移行**という流れが現実的だ。TiDBはSQLとベクトル検索を1本に集約できるため、LoRAデータ収集のような「書きながら集計する」ユースケースとの相性がよい。

コード: https://github.com/hakaru-dev/M2LoRA（記事公開後にpublic化予定）

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
