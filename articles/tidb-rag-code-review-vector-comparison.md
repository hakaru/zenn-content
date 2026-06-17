---
title: "ローカルLLMって本当に開発に使える？（番外編）ベクトルDB比較、TiDB/Chroma/Pinecone"
emoji: "🗄️"
type: "tech"
topics: ["tidb", "rag", "llm", "vectordatabase", "swift"]
published: true
---

iOS/macOS アプリを個人開発中！

**1Take** — 録音iOS アプリ。録音ボタン1つで LA-2A / 1176 系のリアルタイムコンプレッサー掛け取り　AI最適化

https://apps.apple.com/us/app/1take/id6757945099

**M2DX** — iOS/macOS 向け MIDI 2.0 対応 DX7 互換 FM シンセサイザー。App Store で配信中（無料、WWDC26 週に公開）。

https://apps.apple.com/jp/app/m2dx/id6763840208

この M2DX の開発中に「ローカル LLM でコードレビューを自動化できないか」を試し始めたのがシリーズの始まり。

---

:::message
**この記事の対象プロジェクト**

- **M2DX** — iOS/macOS 向け MIDI 2.0 対応 DX7 互換 FM シンセサイザーアプリ。[App Store で配信中（無料）](https://apps.apple.com/jp/app/m2dx/id6763840208)
- **M2DX-Core** — M2DX の DX7 互換エンジン部分。Pure Swift、Apache 2.0 で OSS 公開
- **MIDI2Kit** — M2DX-Core が依存する Swift 製 MIDI 2.0 ライブラリ
- **M2LoRA** — 上記リポジトリのコミットを自動レビュー・採点・合成し、LoRA 学習データを貯めるパイプライン（private）
:::

## 前回までのあらすじ

M3 Ultra (96GB) + Ollama でローカル LLM を Swift/MIDI プロジェクトの開発に使い倒している。

まず（１）でローカル LLM 10 機種に DX7 エンジンの監査をさせた。

**52 件指摘、真陽性 0 件**。

しかも失敗の理由が「Swift をわかってない」だった。具体的には:

- `(Op, Op, Op, Op, Op, Op)` という tuple を「ヒープ確保された Array」と判定して *"リアルタイムスレッドで Array 生成するな"* と指摘してくる
- `level = level &- inc` の `&-` を「overflow チェックが抜けてる」と言う。Swift で `&-` は**意図的に wrap させる**演算子なのに
- `qrate = min(63, qrate + rateScaling)` で上限を貼った 3 行後に「`qrate >> 2` が大きくなりすぎて overflow します」と来る。*`min(63, ...)` 何のためにあると思ってんの…*
- 行番号を平気で捏造する。`DX7Operator.swift` は 85 行しかないのに "L100-L120" を引用してくる

C/C++ で覚えたパターンをそのまま当ててくる感じ。学習データ中の Swift コードが薄いのが本質で、DSP・固定小数点・リアルタイム制約みたいな「Apple アプリ寄りでない Swift」はさらに希少。

（２）では Swift Programming Language Book + Swift Evolution proposals をベクトル DB に入れ、`&+` や `tuple` が出てきたときに関連する仕様ページを動的に引いてプロンプトに差し込む RAG を試した。**Swift セマンティクス系の誤検出が -76%**。`&+`/`&-` の誤読は完全消滅。でも TP は依然 0。

（５）では「コミットのたびに自動でレビュー・採点・合成が走るパイプライン」を作った。Claude / Codex / Gemini の合議で高品質なレビューを合成し続けて、**421 件**が蓄積された。

---

ローカルLLMのテスト、RAG、LoRAと進めて実験していく流れなのだが。。

（２）の RAG は「Swift の言語仕様」を注入した。でも手元の 421 件は「実際の Swift/MIDI コードに対して、どんなレビューをすべきかの事例」だ。仕様書とは違う種類の情報で、「こういう diff が来たらこういう観点で見る」という具体的なパターン集になっている。

これを RAG に入れたら何か変わるか？

ついでに「どのベクトル DB を使うか」も気になっていたので、**TiDB Serverless / ChromaDB / Pinecone Serverless の 3 択**で同じデータを入れて比較することにした。
TiDBの無料サインアップも見つけたので、さらに TiDB の「SQL + ベクトル統合」を活かした**品質重み付き RAG** も試すことにする。

---

## 構成

```
新しい diff
  └─► bge-large（1024 次元）で diff をベクトル化
        └─► ベクトル DB で類似 diff を検索（上位 2 件）
              └─► 類似 diff のレビュー例をプロンプト先頭に注入
                    └─► llama3.3:70b-m2lora-v1 がレビュー生成
                          └─► Claude / Codex / Gemini が採点
```

比較条件は 5 つ:

| 条件 | 内容 |
|---|---|
| RAGなし | diff をそのまま LLM へ |
| TiDB | TiDB Serverless（HNSW）で類似検索 |
| Chroma | ChromaDB（ローカル）で類似検索 |
| Pinecone | Pinecone Serverless で類似検索 |
| TiDB 重み付き | `dist / avg_score` で品質 × 類似度の複合ランキング |

---

## ベクトル DB のセットアップ

### TiDB Serverless

TiDB v8.5.3 からネイティブの `VECTOR` 型と HNSW（= Hierarchical Navigable Small World、グラフ構造で近傍探索するインデックス）が入った。MySQL プロトコル互換なので `pymysql` + SSL で接続できる。

```sql
CREATE TABLE reviews (
    id              VARCHAR(36)   NOT NULL PRIMARY KEY,
    project         VARCHAR(64)   NOT NULL,
    commit_hash     VARCHAR(40),
    code_diff       MEDIUMTEXT    NOT NULL,
    synthesized_review MEDIUMTEXT,
    claude_score    FLOAT,
    codex_score     FLOAT,
    gemini_score    FLOAT,
    flagged         TINYINT(1)    DEFAULT 0,
    diff_embedding  VECTOR(1024),
    VECTOR INDEX idx_diff_emb ((VEC_COSINE_DISTANCE(diff_embedding)))
        USING HNSW
);
```

TiDB の癖？発見。

```sql
SELECT ... FROM reviews
WHERE flagged = 0
ORDER BY VEC_COSINE_DISTANCE(diff_embedding, %s) ASC
LIMIT 10
```

これがエラーになる。（こういう引用を書かせるのはAIだとラクだね笑）
TiDB は　HNSW（インデックス階層型ナビゲート可能スモールワールド＝Hierarchical Navigable Small World、探索効率と精度のバランスに優れる。らしい。。。）
はWHEREとかORDERとか使えない？？？
ANN 検索（= Approximate Nearest Neighbor、近似的に近い順を高速取得する方式）とメタデータフィルタの同時使用はできない模様。

回避策: フィルタなしで データ多めに取ってきて、加工！

### ChromaDB（ローカル）

```python
import chromadb

client = chromadb.PersistentClient(path="./data/chroma")
collection = client.get_or_create_collection(
    name="reviews",
    metadata={"hnsw:space": "cosine"},
)
```

以上。`pip install chromadb` して 3 行。WHERE フィルタと ANN の同時使用も問題なし。セットアップコストはほぼゼロ。
ローカルで簡単にインストーる。一番使いやすいように思える。


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
index = pc.Index("m2lora-reviews")
```

Starter プランはリージョンが `us-east-1` 固定で、東京から遠いのが気になる

---

## embedding のハマり

### bge-large の 512 トークン制限

bge-largeには 512 トークンの入力制限がある。コード diff はトークン密度が高くて普通に投げると `the input length exceeds the context length` エラーが返ってくる。

:::message
**なぜ。bge-large**

= 中国語・コード対応の embedding モデル、1024 次元、採用理由はOllamaで簡単にローカル動作するからぐらい。だが、そのうち深掘りしたい

1. Ollamaでローカル動作 — パイプライン全体をローカル完結にしたかった。bge-large:latest は ollama pull 一発で使える

2. 1024次元 — よく使われる高品質サイズ。TiDBの VECTOR(1024) もこれに合わせて決めた（子嘘）

3. MTEB ベンチマークで高性能 — 当時の "Ollamaで手軽に使える中で精度が高い" 定番候補

  ただし コード特化モデルではない。一般テキスト向け。代替候補として nomic-embed-text（768次元、速い）や  mxbai-embed-large（同1024次元）もOllama対応。コードの意味理解に特化させるなら codebert-base系も選択肢だが、Ollamaで直接使えるのかな

  実際の使い方（diff同士の類似検索）を考えると、コード構文より「この変更が何をしようとしているか」の意味的類似性で十分機能するので、bge-largeで問題は出ていない、という感じか。

:::

最初は文字数 2000 で切っていたが、日本語コメントが多い diff はそれでも超過した。**段階的に縮めながら retry** する方式に落ち着いた。

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
        raise ValueError(f"unexpected: {list(data.keys())}")
    raise ValueError("embed failed at all truncation levels")
```

1200 → 800 → 500 → 300 文字と縮めていって成功したら返す。実際には 1200 文字でほぼ通る。

### RAG 注入

類似 diff 上位 2 件の `synthesized_review`（= Claude が 3 者評価を統合して作った合成レビュー）をプロンプトの頭に差し込む。
書き方によって結果が変わりそう。。「参考」と明示してみる、

```python
prompt = f"""\
あなたはSwift/MIDIコードレビューの専門家です。
=== 類似コードの過去レビュー例（参考） ===
{examples}
=== END ===

=== コードdiff ===
{diff}
=== END ===

レビュー:"""
```

---

## 実測比較① — DB 3択 × 20 件

M2DX・MIDI2Kit の実コミット 20 件をランダムサンプリング。各 commit に 4 条件のレビューを生成し、Claude / Codex / Gemini の平均スコアで評価した。

### 全件スコア

| commit | project | RAGなし | TiDB | Chroma | Pinecone |
|---|---|---|---|---|---|
| 2f25b04 | M2DX | 5.33 | 5.33 | 5.33 | **7.00** |
| 761ff5f6 | MIDI2Kit | 1.00 | 3.33 | **7.67** | 7.00 |
| 9fd81f6 | M2DX | **7.33** | 3.67 | 7.33 | 6.33 |
| 1be38940 | M2DX | 5.00 | 5.50 | **8.00** | **8.00** |
| 1aa496f4 | MIDI2Kit | 3.67 | 6.67 | 6.50 | **8.00** |
| a6dd8188 | MIDI2Kit | 2.00 | **5.00** | 2.50 | 2.50 |
| faeb721c | M2DX | 5.00 | 2.50 | **6.00** | 3.50 |
| 53ba4f3f | M2DX | 7.00 | 7.50 | 7.50 | 7.50 |
| 1be3894 | M2DX | 5.50 | **8.50** | 9.00 | 9.00 |
| 93580685 | MIDI2Kit | 2.50 | **6.50** | 6.00 | 3.00 |
| 181ad50 | M2DX | 5.50 | **7.50** | 6.50 | 7.00 |
| 61e739e | M2DX | 4.50 | 5.00 | 3.00 | **5.50** |
| 9ba3b9fd | M2DX | **3.50** | 2.50 | 3.00 | 2.50 |
| 58e5f728 | MIDI2Kit | 4.00 | 7.00 | **8.00** | 7.00 |
| 5fa8a593 | M2DX | 3.00 | 4.50 | 5.50 | **6.50** |
| 8dd2276 | M2DX | 4.50 | 4.00 | 4.50 | **6.00** |
| bf6fe14b | MIDI2Kit | 3.00 | **9.00** | 8.00 | 8.00 |
| 6eb8ae13 | M2DX | **5.00** | 3.00 | 3.00 | 4.00 |
| 7e8f8e49 | M2DX | 3.00 | 4.00 | 3.00 | **4.00** |
| 1e8cddfd | M2DX | 5.00 | 5.00 | 5.00 | 5.00 |

### 集計

| | RAGなし | TiDB | Chroma | Pinecone |
|---|---|---|---|---|
| **平均スコア** | 4.27 | 5.30 | 5.77 | **5.87** |
| **平均Δ** | — | +1.03 | +1.50 | **+1.60** |
| **改善件数** | — | 13/20 (65%) | 12/20 (60%) | **15/20 (75%)** |

全 DB で改善した。Pinecone が数字上は頭一つ出てるけど、**n=20 では DB 間に有意差はない**。+1.0〜+1.6点で横並び＝*どの DB に入れても RAG の品質はほぼ変わらない*、と読むのが正直なところ。だとすると DB を選ぶ基準は「品質」じゃなくて「運用」に移る（→ [性能・HTAP 実測編](https://zenn.dev/hakaru/articles/tidb-rag-vectordb-htap-benchmark)で掘った）。

### 逆効果になったケース

`9fd81f6`（M2DX、NaN ガード追加）はノーRAG **7.33** → TiDB **3.67** と大きく下がった。注入された類似レビューが「Swift の `isNaN` チェックはオーバーヘッドがある」という内容で、このコードには完全に無関係。**ベクトルが近くても、注入する情報が的外れになることはある**。元から高スコアな diff に RAG を足してもノイズになるだけ、という傾向があった。

---

## 実測比較② — TiDB 固有機能：品質重み付き RAG

せっかくTiDB　Cloud試してるんで SQL + ベクトル統合でしかできないことを試すよ。

### 発想

純粋な類似度検索は「最も似た diff のレビュー」を引いてくる。でも「似ているが採点が低い（= 質が悪い）レビュー」が混じる可能性がある。

TiDB なら SQL でベクトル距離とスコアを同時に計算して **`dist / avg_score` で複合ランキング**できる。これは Chroma や Pinecone だと「ANN で取ってから Python で再ランキング」の 2 段になるが、TiDB は 1 クエリで完結する。

```sql
SELECT commit_hash, code_diff, synthesized_review,
       VEC_COSINE_DISTANCE(diff_embedding, %s) AS dist,
       (claude_score + codex_score + gemini_score) / 3.0 AS avg_score,
       VEC_COSINE_DISTANCE(diff_embedding, %s) /
           NULLIF((claude_score + codex_score + gemini_score) / 3.0, 0)
           AS weighted_dist
FROM reviews
WHERE diff_embedding IS NOT NULL
  AND synthesized_review IS NOT NULL
  AND claude_score IS NOT NULL
  AND (claude_score + codex_score + gemini_score) / 3.0 >= 4.0
ORDER BY weighted_dist ASC
LIMIT %s
```

`weighted_dist` が小さいほど「類似していて品質が高い」。HNSW + WHERE の制限がある通常クエリと違い、これはフルスキャンになるが WHERE フィルタを SQL 側で完結できる。

### 結果（10 件）

| commit | project | RAGなし | TiDB（生距離） | TiDB（重み付き） |
|---|---|---|---|---|
| 2f25b04 | M2DX | 5.00 | 6.00 | 6.00 |
| 761ff5f6 | MIDI2Kit | 3.00 | **7.00** | 6.00 |
| 9fd81f6 | M2DX | 3.00 | **7.00** | 6.00 |
| 1be38940 | M2DX | 2.00 | **7.00** | 4.00 |
| 1aa496f4 | MIDI2Kit | 4.00 | 6.00 | 6.00 |
| a6dd8188 | MIDI2Kit | 3.00 | 5.00 | **6.00** |
| faeb721c | M2DX | 3.00 | **5.50** | 4.00 |
| 53ba4f3f | M2DX | 4.00 | 5.50 | 5.50 |
| 1be38940 | M2DX | 4.00 | **7.00** | 6.50 |
| 93580685 | MIDI2Kit | 3.00 | 6.50 | **7.00** |

| | RAGなし | TiDB（生距離） | TiDB（重み付き） |
|---|---|---|---|
| **平均スコア** | 3.40 | **6.25** | 5.70 |
| **平均Δ** | — | **+2.85** | +2.30 |
| **Weighted > 生距離** | — | — | **2/10件** |

**重み付きの方が低かった。** 生距離が 10/10 で改善（全勝）に対して、重み付きも 10/10 で改善するが、平均スコアは生距離より -0.55 点。

### ？？逆効果か。。。。予想外。

仮説「**スコアが高い ≠ 文脈が合う**」

`weighted_dist = dist / avg_score` は「スコアが高いレビュー」を強く引き寄せる。高スコアなレビューは往々にして「典型的で整ったコードへのレビュー」で、今の diff とコードパターンが違っていても数値上は有利になる。

一方、純粋な類似度は「最も似た diff のレビュー」を取ってくる。コードパターンが近ければ、レビューの観点も自然と文脈に合いやすい。

`1be38940` で顕著で、重み付きで注入された 2 件目が `avg=4.7` と低めだった（フィルタの下限 4.0 ギリギリ）。品質フィルタを通した上でスコアを重み付けしているのに、それでも文脈がズレたものが入ってきた。

*「品質が高い事例が参考になる」は人間の直感では正しいが、RAG では「文脈が近い事例が参考になる」の方が勝った。*

これは TiDB のせいじゃなくて、`dist / avg_score` という**重み付けの発想自体が外した**だけ。「品質が高い事例を優先」は人間の直感では正しいのに、RAG では「文脈が近い事例」の方が勝った、という RAG 側の学び。

注目したいのは別のところ。Chroma/Pinecone なら「ANN で取ってから Python で再ランキング」の2段になる複合ランキングを、**TiDB は SQL 1本で書ける**。重み付けのヒューリスティクスは不発だったけど、SQL × ベクトルを1クエリに畳める土俵は本物で、これが効くのはこの後に出てくる管理クエリ（スコア集計・世代比較・フラグ管理）の方だった。
---

## DB 選択の整理

| 観点 | TiDB | Chroma | Pinecone |
|---|---|---|---|
| 20件比較 平均Δ | +1.03 | +1.50 | **+1.60** |
| 改善率 | 65% | 60% | **75%** |
| セットアップ | クラウド（要SSL） | **ローカル（最簡単）** | クラウド |
| 無料枠 | Serverless 5GiB | 制限なし | Starter 2GiB |
| HNSW + WHERE | **未サポート（v8.5.3）** | サポート | サポート |
| SQL + ベクトル統合 | **◎（同一クエリ）** | △ | △ |
| レイテンシ | 中（AWS Tokyo） | **最小（ローカル）** | 大（us-east-1） |

スコア差は誤差範囲だが、使い分けの感触は明確にある:

- **プロトタイプ・ローカルだけで完結** → **Chroma**。`pip install` で動いて速い、立ち上げは一番ラク
- **SQL と一緒に運用・管理クエリを書きたい** → **TiDB**（MySQL 互換、ベクトルと JOIN・集計が同じテーブルに同居）
- **書きながら集計したい / 2システムにしたくない** → **TiDB**（HTAP。詳細は[性能・HTAP 実測編](https://zenn.dev/hakaru/articles/tidb-rag-vectordb-htap-benchmark)）

TiDB は「スコアが高くて類似した diff だけ引いてくる」「世代別・プロジェクト別のスコア集計」「フラグ管理」といった、SQL の表現力が必要な操作をベクトル検索と組み合わせて一発で書ける。今回の重み付き実験でスコア上は裏目に出たが、管理クエリとしての使い道は依然として強い。実際この「SQL × ベクトルが1テーブルに同居」が本当に効いたのは、この後に作った AI エージェントの長期記憶システム(memdream)の方だった → [TiDB に観測2,218件を溜めた8週間の実録](https://zenn.dev/hakaru/articles/memdream-tidb-vector-ai-memory)。

---

## まとめ

| 実験 | 結果 |
|---|---|
| 421 件のレビューを 3 DB に移行して比較 | 全 DB で RAGなし比 +1.0〜+1.6 点改善 |
| 3 DB のスコア差 | Pinecone 微差で最良、ただし誤差範囲 |
| TiDB 品質重み付き RAG | 生距離より -0.55 点（逆効果） |
| 逆効果の理由 | 「品質が高い」 > 「文脈が近い」は RAG では成立しなかった |
| TiDB の強みが出る場面 | SQL + ベクトルの管理クエリ（スコア集計・フラグ管理・世代別分析） |

（２）の仕様書 RAG（Swift 構文の誤読を抑える）と今回の過去レビュー RAG（的外れな観点を抑える）は別軸の問題に効くので、組み合わせが次の候補。
もっと素直にSwift、MIDI,CoreAudioの仕様書とかをベクトルDBに入れて検証するところから始めるべきだったか。。

まだまだ続く。。。。。。。

---

:::message
**シリーズの他の記事**

- [（１）監査編 — 指摘 52 件、真陽性 0 件](https://zenn.dev/hakaru/articles/m2dx-local-llm-audit-zero-true-positives)
- [（２）RAG 編 — Swift 仕様書で誤検出 76% 削減](https://zenn.dev/hakaru/articles/m2dx-local-llm-audit-rag)
- [（３）aider 編 — 参照軸は改善、知識軸は依然ダメ](https://zenn.dev/hakaru/articles/m2dx-local-llm-agentic-harness-eval)
- [（４）LoRA 編 — 誤検知 93% 削減](https://zenn.dev/hakaru/articles/swift-audit-lora-fp-reduction)
- [（５）M2LoRA パイプライン編 — 開発しながら自動でデータが貯まる仕組み](https://zenn.dev/hakaru/articles/m2lora-code-review-pipeline)
- [（番外②）性能編 — Ingest/Search を実測 + TiDB HTAP を実証](https://zenn.dev/hakaru/articles/tidb-rag-vectordb-htap-benchmark)
- [（応用）AIエージェントの記憶は「夢」で整理する — TiDB に観測2,218件を溜めた8週間の実録](https://zenn.dev/hakaru/articles/memdream-tidb-vector-ai-memory)
:::

*本記事は [Zennfes Spring 2026 × TiDB](https://zenn.dev/contests/zennfes-spring-2026-tidb) への応募作品です。*

---

- **M2DX-Core (OSS)**: https://github.com/hakaru/M2DX-Core
- **M2DX (App Store)**: https://apps.apple.com/jp/app/m2dx/id6763840208
