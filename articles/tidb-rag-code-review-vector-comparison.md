---
title: "ローカルLLMって本当に開発に使える？（番外編）ベクトルDB比較、TiDB/Chroma/Pinecone"
emoji: "🗄️"
type: "tech"
topics: ["tidb", "rag", "llm", "vectordatabase", "swift"]
published: false
---

iOS/macOS アプリを個人開発している。最近リリースしたのが 2 本。

**1Take** — 練習録音用 iOS アプリ。録音ボタン1つで LA-2A / 1176 系のリアルタイムコンプレッサーが乗って、DAW なしで「録りながら良い音」になる。

https://apps.apple.com/us/app/1take/id6757945099

**M2DX** — iOS/macOS 向け MIDI 2.0 対応 DX7 互換 FM シンセサイザー。TestFlight で公開ベータ配布中なので、MIDI 2.0 環境がある方はぜひ試してフィードバックをもらえると嬉しい。

https://testflight.apple.com/join/BAtGszPw

この M2DX の開発中に「ローカル LLM でコードレビューを自動化できないか」を試し始めたのが、このシリーズの始まり。

---

:::message
**この記事の対象プロジェクト**

- **M2DX** — iOS/macOS 向け MIDI 2.0 対応 DX7 互換 FM シンセサイザーアプリ。[TestFlight 公開ベータ](https://testflight.apple.com/join/BAtGszPw) で試せる
- **M2DX-Core** — M2DX の DX7 互換エンジン部分。Pure Swift、Apache 2.0 で OSS 公開
- **MIDI2Kit** — M2DX-Core が依存する Swift 製 MIDI 2.0 ライブラリ
- **M2LoRA** — 上記リポジトリのコミットを自動レビュー・採点・合成し、LoRA 学習データを貯めるパイプライン（private）
:::

## 前回までのあらすじ

M3 Ultra + Ollama でローカル LLM を Swift/MIDI プロジェクトの開発に使い倒している。

まず（１）でローカル LLM 10 機種に DX7 エンジンの監査をさせた。

**52 件指摘、真陽性 0 件**。

しかもその理由が笑えなくて、「Swift をわかってない」が根本原因だった。具体的には:

- `(Op, Op, Op, Op, Op, Op)` という tuple を「ヒープ確保された Array」と判定して "リアルタイムスレッドで Array 生成するな" と指摘してくる
- `level = level &- inc` の `&-` を「overflow チェックが抜けてる」と言う。Swift で `&-` は **意図的に wrap させる**演算子で、むしろ「ラップしてほしいから `&-` と書いてる」のに
- `qrate = min(63, qrate + rateScaling)` で上限を貼った 3 行後に「`qrate >> 2` が大きくなりすぎて overflow します」と来る。**`min(63, ...)` 何のためにあると思ってんの…**
- 行番号を平気で捏造する。`DX7Operator.swift` は 85 行しかないのに "L100-L120" を引用してくる

これは「モデルが弱い」というより **学習データ中の Swift コードが薄い**のが原因で、特に DSP・固定小数点・リアルタイム制約みたいな「Apple アプリ寄りでない Swift」はさらに希少。C/C++ で覚えたパターンをそのまま当ててくる感じ。

---

（２）では Swift Programming Language Book + Swift Evolution proposals を chunk に分けてベクトル DB に入れ、コードに `&+` や `tuple` が出てきたときに関連する仕様ページを動的に retrieval してプロンプトに差し込む RAG を試した。

結果: **Swift セマンティクス系の誤検出が -76%**。`&+`/`&-` の誤読は完全消滅した。

でも **TP は依然 0**。

「Swift を知らない」という問題は仕様書 RAG でかなり抑えられた。ただしそれは「誤った指摘をしなくなる」方向の改善で、「本物のバグを見つける能力」は別軸。

---

さらに（５）では「コミットのたびに自動でレビュー・採点・合成が走るパイプライン」を作った。Claude / Codex / Gemini に同じ diff を投げて採点させ、合議で高品質なレビューを合成する仕組み。

421 件の Swift/MIDI コードレビューが蓄積された。

**ここで一つ気づいた。**

（２）の RAG は「Swift の言語仕様」を注入した。でも今手元にある 421 件は、**実際の Swift/MIDI コードに対して、実際にどんなレビューをすべきかを示す事例**だ。仕様書とは違う種類の情報で、「こういう diff にはこういう観点で見るといい」という具体的なパターン集になっている。

これをRAGに入れたら何か変わるのか？ 試した。

---

## やったこと

新しい diff が来たとき、**ベクトル検索で類似した過去 diff を引いてきて、そのレビュー例をプロンプトに差し込む**。

```
新しい diff
  └─► bge-large（= BERT 系の embedding モデル、1024 次元）で diff をベクトル化
        └─► ベクトル DB で類似 diff を検索（上位 2 件）
              └─► 類似 diff のレビュー例をプロンプト先頭に注入
                    └─► llama3.3:70b-m2lora-v1 がレビュー生成
                          └─► Claude / Codex / Gemini が採点
```

ベクトル DB は 3 種類で試した:

| 条件 | DB |
|---|---|
| RAGなし | diff をそのまま LLM へ |
| TiDB RAG | TiDB Serverless（HNSW インデックス） |
| Chroma RAG | ChromaDB（ローカル） |
| Pinecone RAG | Pinecone Serverless |

同じ 421 件を 3 つ全部に入れて、同じ 20 件の実コミットに対して 4 条件のレビューを生成・採点する。

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
    flagged         TINYINT(1)    DEFAULT 0,
    diff_embedding  VECTOR(1024),
    VECTOR INDEX idx_diff_emb ((VEC_COSINE_DISTANCE(diff_embedding)))
        USING HNSW
);
```

421 件の embedding 移行は約 10 分で終わった。

**が、すぐ詰まった。**

最初に書いたクエリ:

```sql
SELECT ... FROM reviews
WHERE flagged = 0
  AND diff_embedding IS NOT NULL
ORDER BY VEC_COSINE_DISTANCE(diff_embedding, %s) ASC
LIMIT %s
```

これを実行するとエラーになる。TiDB Serverless v8.5.3 時点で、**HNSW インデックスは `WHERE` フィルタと一緒に使えない**。ANN 検索（= Approximate Nearest Neighbor、近似的に近い順に高速取得する方式）とメタデータフィルタの組み合わせは未サポート。

回避: `top_k × 5` 件をフィルタなしで取ってきて、Python 側で絞る。

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
rows = [r for r in cur.fetchall() if r["synthesized_review"] and not r["flagged"]][:top_k]
```

*余分に取って捨てる。行儀は悪いが、421 件規模なら実用上は問題ない。*

### ChromaDB（ローカル）

```python
import chromadb

client = chromadb.PersistentClient(path="./data/chroma")
collection = client.get_or_create_collection(
    name="reviews",
    metadata={"hnsw:space": "cosine"},
)
```

以上。`pip install chromadb` して 3 行。WHERE フィルタと ANN の同時使用も問題ない。セットアップコストはほぼゼロ。

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

Starter プランはリージョンが `us-east-1` 固定で、東京からのレイテンシが 3 つの中で一番大きい。あと Pinecone は**類似度（1 = 完全一致）を返す**ので、距離に変換するときに `1 - score` が必要。これを忘れると「遠いほど良い」が逆転する。

---

## embedding まわりのハマり

### bge-large の 512 トークン制限

bge-large（= 中国語・コード対応の embedding モデル、1024 次元）には 512 トークンの入力制限がある。コード diff はトークン密度が高くて普通に投げると `the input length exceeds the context length` エラーが返ってくる。

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

### RAG 注入の方法

類似 diff 上位 2 件の `synthesized_review`（= Claude が 3 者評価を統合して作った合成レビュー）をプロンプトの頭に差し込む。

```python
_RAG_SECTION = """\
=== 類似コードの過去レビュー例（参考） ===
{examples}
=== END ===

"""

async def review_diff(diff: str, use_rag: bool = True) -> str:
    rag_context = ""
    if use_rag:
        similar = await find_similar(diff, top_k=2)
        if similar:
            examples = "\n".join(
                f"--- 例{i+1} ---\n{r.synthesized_review[:400]}"
                for i, r in enumerate(similar)
            )
            rag_context = _RAG_SECTION.format(examples=examples)

    prompt = f"""\
あなたはSwift/MIDIコードレビューの専門家です。
{rag_context}
=== コードdiff ===
{diff}
=== END ===

レビュー:"""
    return await _generate(prompt)
```

「参考」と明示しているので、モデルはそのままコピーはせず、似た観点を自分の言葉でアレンジして書く。実際に見ると、文面ではなく観点が引き継がれている感じだった。

---

## 実測比較（20 件）

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

---

## 何が起きているか

### 仕様書 RAG との違い

（２）の仕様書 RAG は「LLM が Swift を知らない」という問題を攻めた。`&+` は wrap 演算子だという仕様を与えれば、LLM は `&+` を誤解しなくなる。

今回の「過去レビュー RAG」は違う軸で効く。仕様書は「この言語構文はこう動く」という**静的な知識**だが、過去レビューは「この commit パターンにはこういう観点で見ると質問が出る」という**事例ベースの判断**だ。たとえば「MIDI2Kit の Protocol Negotiation 周りのコードが来たら、タイムアウト伝播の確認を必ず入れる」みたいな暗黙知が、過去レビューには含まれている。

採点が平均 **+1〜+1.6 点**改善した主因は、こうした事例からの観点引き継ぎで、LLM が「的外れな指摘」を減らす方向に動いた結果と見ている。

### RAGなしでも高スコアな commit には効かない

改善しなかった 5〜7 件を見ると、**RAGなしのスコアが既に 7 点以上**のパターンが多い。元から良いレビューに RAG コンテキストを足してもノイズにしかならない。

`9fd81f6`（M2DX、NaN ガード追加）が顕著で、RAGなし **7.33** → TiDB **3.67** と大きく下がった。注入された類似レビューが「Swift の `isNaN` チェックはオーバーヘッドがある」という内容で、このコードには完全に無関係だった。**類似ベクトルが近くても注入情報が的外れになることはある**。

### `&+` 誤読や tuple 誤判定は改善されたか？

仕様書 RAG のときは `&+` 系が完全消滅した。今回の過去レビュー RAG では、仕様書を入れているわけではないので Swift 構文の誤読に対する直接的な改善はない。ただ「過去の良質なレビューを参考にする」という文脈が入ることで、LLM が「的外れに見える指摘は書かない」方向に引っ張られる副作用はある。

純粋に「Swift を教える」なら仕様書 RAG。「実際のコードパターンに対する判断を安定させる」なら過去レビュー RAG。組み合わせが最強候補。

### TiDB vs Chroma vs Pinecone

| 観点 | TiDB | Chroma | Pinecone |
|---|---|---|---|
| 平均Δ | +1.03 | +1.50 | **+1.60** |
| 改善率 | 65% | 60% | **75%** |
| セットアップ | クラウド（要SSL） | ローカル（最簡単） | クラウド |
| 無料枠 | Serverless 5GiB | 制限なし（ローカル） | Starter 2GiB |
| HNSW WHERE 制限 | **あり（要回避）** | なし | なし |
| レイテンシ | 中（AWS Tokyo） | **最小（ローカル）** | 大（us-east-1） |

スコアで Pinecone がわずかに上だが、**サンプル 20 件なので差は誤差の範囲**と見ている。選択の決め手はスコアより実用上の要件に依存する。

セットアップのしやすさで言うと Chroma が圧倒的に楽。`pip install chromadb` だけで動く。TiDB と Pinecone はクレデンシャル管理・SSL・API キーが必要で、動くまでに数十分かかった。

### TiDB を使う理由: SQL とベクトルの統合

スコアは 3 番手でも、このパイプラインとの相性でいうと TiDB がいちばん都合がいい。

ベクトル検索と通常の SQL を**同一クエリで書ける**。

```sql
-- スコアが高くて類似した diff を取ってくる
SELECT id, synthesized_review,
       VEC_COSINE_DISTANCE(diff_embedding, %s) AS dist
FROM reviews
WHERE claude_score >= 7.0
ORDER BY dist ASC
LIMIT 5;
```

（HNSW と `WHERE` の同時使用はできないので、このクエリはフルスキャン側で動く。ANN での高速検索ではなく exact search になる点は注意。件数が増えてきたら要検討。）

Chroma や Pinecone ではメタデータフィルタと ANN の同時使用に制約があるが、TiDB は SQL の表現力がそのまま使える。`review_model` 別の集計、`flagged` 状態の管理、プロジェクト別スコア比較——全部 SQL で書ける。M2LoRA パイプライン全体が SQLite ベースで動いているので、その延長線上に TiDB がある、という感覚。

:::message
HNSW + WHERE の制限は v8.5.3 時点の話。将来のバージョンで解消されるかもしれないので、使う前に確認。
:::

---

## 仕様書 RAG と過去レビュー RAG の比較

この記事で試したものを（２）と並べると:

| 観点 | （２）仕様書 RAG | 今回：過去レビュー RAG |
|---|---|---|
| 入れたもの | Swift Book + Evolution proposals | 過去の実コードレビュー 421 件 |
| 効く問題 | `&+` / tuple などの Swift 構文誤読 | 的外れな観点・文脈ずれな指摘 |
| `&+` 誤読 | **完全消滅** | 変化なし（仕様書を入れていない） |
| 平均スコア改善 | 未測定（FP件数で評価） | **+1.0〜+1.6 点** |
| 組み合わせ | ← 両方入れたら最強候補 | → |

---

## 選択指針

どの DB を選んでもスコア差は誤差に近い。決め手はユースケース:

- **プロトタイプ・ローカル開発** → **Chroma**（pip install だけ、ゼロ設定）
- **チーム共有・スケールアウト** → **TiDB or Pinecone**（マネージドで永続化不要）
- **既存アプリに SQL と統合** → **TiDB**（MySQL 互換、JOIN や集計がそのまま使える）

---

## まとめ

| やったこと | 結果 |
|---|---|
| 421 件のレビューを TiDB / Chroma / Pinecone に移行 | bge-large 1024 次元 embed、10 分以内 |
| 4 条件 × 20 commit で比較 | 全 DB で RAGなし比 +1.0〜+1.6 点 |
| Pinecone が数字は最良 | 改善率 75%、平均 +1.60 |
| （２）仕様書 RAG との違いが明確 | 仕様書 → Swift 構文誤読に効く、過去レビュー → 的外れ指摘に効く |
| HNSW + WHERE は未サポート（v8.5.3） | top_k × 5 取得 → Python フィルタで回避 |

「まず Chroma で試して、スケールが必要になったら TiDB / Pinecone へ」が現実的な移行パス。

仕様書 RAG と過去レビュー RAG を組み合わせたら、互いに別軸の問題を補えるので、次はその組み合わせを試したい。

---

:::message
**シリーズの他の記事**

- [（１）監査編 — 指摘 52 件、真陽性 0 件](https://zenn.dev/hakaru/articles/m2dx-local-llm-audit-zero-true-positives)
- [（２）RAG 編 — Swift 仕様書で誤検出 76% 削減](https://zenn.dev/hakaru/articles/m2dx-local-llm-audit-rag)
- [（３）aider 編 — 参照軸は改善、知識軸は依然ダメ](https://zenn.dev/hakaru/articles/m2dx-local-llm-agentic-harness-eval)
- [（４）LoRA 編 — 誤検知 93% 削減](https://zenn.dev/hakaru/articles/swift-audit-lora-fp-reduction)
- [（５）M2LoRA パイプライン編 — 開発しながら自動でデータが貯まる仕組み](https://zenn.dev/hakaru/articles/m2lora-code-review-pipeline)
:::

*本記事は [Zennfes Spring 2026 × TiDB](https://zenn.dev/contests/zennfes-spring-2026-tidb) への応募作品です。*

---

- **M2DX-Core (OSS)**: https://github.com/hakaru/M2DX-Core
- **M2DX (TestFlight)**: https://testflight.apple.com/join/BAtGszPw
