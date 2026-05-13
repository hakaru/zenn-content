---
title: "ローカルLLMって本当に開発に使える？（番外編）過去レビューをRAGで注入したら幻覚が減った — TiDB/Chroma/Pinecone 3択比較"
emoji: "🗄️"
type: "tech"
topics: ["tidb", "rag", "llm", "vectordatabase", "swift"]
published: false
---

:::message
**この記事の対象プロジェクト**

- **M2DX** — iOS/macOS 向け MIDI 2.0 対応 DX7 互換 FM シンセサイザーアプリ。[TestFlight 公開ベータ](https://testflight.apple.com/join/BAtGszPw) で試せる
- **MIDI2Kit** — M2DX-Core が依存する Swift 製 MIDI 2.0 ライブラリ
- **M2LoRA** — 上記リポジトリのコミットを自動でレビュー・採点・合成し、LoRA 学習データを貯めるパイプライン。[github.com/hakaru/M2LoRA](https://github.com/hakaru/M2LoRA)
:::

## 前回までのあらすじ

このシリーズでは M3 Ultra (96GB) + Ollama でローカル LLM を Swift/MIDI プロジェクトの開発に使い倒している。

- **（１）監査**: llama3.3:70b 他 10 モデルで DX7 エンジンを監査 → 指摘 52 件・真陽性 0 件
- **（２）RAG**: Swift 仕様書をベクトル DB に入れて検索させる → 誤検知 76% 削減
- **（３）aider**: ツール統合でコーディングアシスト → 参照軸は改善、知識軸は依然ダメ
- **（４）LoRA**: 手作り 73 件でファインチューン → 誤検知 93% 削減
- **（５）M2LoRA パイプライン**: コミットのたびに自動でレビュー・採点・合成が走る仕組みを作った

（５）で「高品質レビューが自動で貯まる」仕組みができた。現在 421 件蓄積済み。

問題は、**貯まったレビューを新しいレビュー生成に使えていない**ことだ。せっかく過去に Claude / Codex / Gemini が合議して作った良質なレビューがあるのに、次のレビュー時はゼロから始めている。

これを解決するために、過去レビューを**ベクトル検索で取得してプロンプトに注入する RAG**（= Retrieval-Augmented Generation、外部知識を引っ張ってきて LLM の回答を補強する手法）を作った。

ついでに「どのベクトル DB を使うか」問題があったので、**TiDB Serverless / ChromaDB / Pinecone Serverless の 3 択を同じデータで比較**してみた。

---

## アーキテクチャ

```
git diff
  └─► review.py (llama3.3:70b-m2lora-v1, Ollama)
        ├─► [RAGなし] プロンプト直接生成
        └─► [RAGあり] retriever.py
              ├─ bge-large:latest でdiffをembed（1024次元）
              ├─ TiDB / Chroma / Pinecone で上位2件取得
              └─ 類似diff+レビュー例をプロンプトに注入
                    └─► evaluate.py（Claude / Codex / Gemini で採点）
```

収集済みデータは **421 件の Swift/MIDI コードレビュー**（M2DX + MIDI2Kit の実コミット）。

---

## ベクトル DB のセットアップ

### TiDB Serverless

TiDB v8.5.3 からネイティブの `VECTOR` 型と HNSW（= Hierarchical Navigable Small World、近傍探索用のグラフ構造インデックス、Chroma や Pinecone でも使われてる）インデックスが入った。MySQL プロトコル互換なので、接続は `pymysql` + SSL でいける。

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

```python
conn = pymysql.connect(
    host="gateway01.ap-northeast-1.prod.aws.tidbcloud.com",
    port=4000,
    user="<user>",
    password="<password>",
    database="m2lora",
    ssl={"ca": "/etc/ssl/cert.pem"},
)
```

これで 421 件の embedding 移行はだいたい 10 分で終わった。

**が、すぐ詰まった。**

最初はこう書いた:

```sql
SELECT ... FROM reviews
WHERE flagged = 0
  AND diff_embedding IS NOT NULL
ORDER BY VEC_COSINE_DISTANCE(diff_embedding, %s) ASC
LIMIT %s
```

実行するとエラー。TiDB Serverless v8.5.3 時点で、**HNSW インデックスは `WHERE` フィルタと一緒に使えない**らしい。ANN 検索（= Approximate Nearest Neighbor、完全一致ではなく「だいたい近い」を高速に探す方法）とメタデータフィルタの同時適用は未サポート。

回避策: フィルタなしで `top_k × 5` 件多めに取ってきて、Python 側で絞る。

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

*WHERE フィルタが使えないなら多めに取ってから捨てればいい。* ちょっと行儀悪いが、421 件規模なら実用上の問題はない。

### ChromaDB（ローカル）

```python
import chromadb

client = chromadb.PersistentClient(path="./data/chroma")
collection = client.get_or_create_collection(
    name="reviews",
    metadata={"hnsw:space": "cosine"},
)
```

以上。セットアップはこれだけ。`pip install chromadb` して 3 行で終わる。WHERE フィルタの制限もなく、メタデータフィルタと ANN を同時に使える。

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

Pinecone はリージョンが `us-east-1` 固定（Starter プランの制約）で、東京からのレイテンシは体感で一番大きい。あと **Pinecone は類似度スコア（1 = 完全一致）を返す**ので、距離に変換するときに `1 - score` が必要。これを忘れると「遠いほど良い」が逆転する。

---

## embedding まわりのハマり

### bge-large のトークン制限

bge-large（= 中国語・コード対応の BERT 系 embedding モデル、1024 次元）は 512 トークンの入力制限がある。コード diff はトークン密度が高くて、普通に投げると `the input length exceeds the context length` エラーが返ってくる。

最初は文字数 2000 で切っていたが、日本語コメントが多い diff だと 2000 文字でも超過した。結局、**段階的に縮めて成功するまで retry** する方式に落ち着いた。

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
            continue  # 短くして再試行
        raise ValueError(f"unexpected: {list(data.keys())}")
    raise ValueError("embed failed at all truncation levels")
```

1200 → 800 → 500 → 300 文字と縮めていって、300 文字でも失敗したら諦める。実際には 1200 文字でほぼ通る。

### RAG の注入方法

類似 diff 上位 2 件の `synthesized_review`（= Claude が 3 者評価を統合して作った高品質レビュー）を、プロンプトの頭に差し込む。

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

「参考」と書いてあるので、LLM はそのまま丸コピーはしない（はず）。実際に見ると、似た観点を自分の言葉でアレンジして書いていることが多かった。

---

## 実測比較（20 件）

M2DX・MIDI2Kit の実コミット 20 件をランダムサンプリング。各 commit に 4 条件（RAGなし / TiDB / Chroma / Pinecone）のレビューを生成し、Claude / Codex / Gemini の平均スコアで評価。

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

### RAG は効く。ただし全件ではない

20 件中 60〜75% で改善した。**平均 +1〜+1.6 点**は「凡庸なレビュー → まあまあなレビュー」程度の変化だが、幻覚を含む指摘（存在しない API の言及、誤ったオーバーフロー警告など）が類似事例の参照によって抑制されているケースが体感で多かった。

改善しなかった 5〜7 件を見ると、**RAGなしのスコアが既に 7 点以上**のパターンが大半。天井効果で、元から良いレビューに RAG コンテキストを追加しても上積みにならず、むしろノイズになることがある。

### `9fd81f6` の逆転現象

1 件だけ顕著な例を挙げると、`9fd81f6`（M2DX, NaN ガード追加）はノーRAGで **7.33** だったが、TiDB RAG では **3.67** まで落ちた。

このケースでは「Swift の `isNaN` チェックはオーバーヘッドがある」という類似レビューが注入されていたが、diff の文脈では全く無関係だった。**類似ベクトルが近くても、注入された情報が的外れになることがある**。RAG は常に「文脈を豊かにする」わけではない、という当たり前の話。

### TiDB vs Chroma vs Pinecone

| 観点 | TiDB | Chroma | Pinecone |
|---|---|---|---|
| 平均Δ | +1.03 | +1.50 | **+1.60** |
| 改善率 | 65% | 60% | **75%** |
| セットアップ | クラウド（要SSL） | ローカル（最簡単） | クラウド |
| 無料枠 | Serverless 5GiB | 制限なし（ローカル） | Starter 2GiB |
| HNSW WHERE 制限 | **あり（要回避）** | なし | なし |
| レイテンシ | 中（AWS Tokyo） | **最小（ローカル）** | 大（us-east-1） |

今回の 421 件・1024 次元という条件では Pinecone が一番スコアを出した。ただし **サンプル 20 件なので差は誤差の範囲**かもしれない。DB の選択よりも「どんなレビューを入れるか」の方が影響が大きそう。

スコアより実感として差があったのは**セットアップのしやすさ**。Chroma は `pip install chromadb` だけで動く。TiDB と Pinecone はクレデンシャル管理・SSL・API キーが必要で、最初に動くまでが数十分かかる。

### TiDB を使う理由：SQL とベクトルの統合

スコアで TiDB が 3 番手だったとはいえ、このパイプラインとの相性が一番良いと思っている理由がある。

ベクトル検索と通常の SQL を**同一クエリで書ける**。

たとえば「スコアが高くて、かつ類似した diff を取ってきたい」は 1 クエリで書ける:

```sql
SELECT id, synthesized_review,
       VEC_COSINE_DISTANCE(diff_embedding, %s) AS dist
FROM reviews
WHERE claude_score >= 7.0
ORDER BY dist ASC
LIMIT 5;
```

Chroma や Pinecone ではメタデータフィルタと ANN 検索の組み合わせに制限があるが、TiDB は SQL の表現力をそのまま持ち込める。`review_model` 別の集計、`flagged` フィルタ、プロジェクト別スコア比較——全部 SQL で書ける。LoRA データ品質管理との相性がいい。

:::message
ただし前述の通り、現時点（v8.5.3）では HNSW と `WHERE` を組み合わせると動かない。将来のバージョンで解消される可能性はある。
:::

---

## 選択指針

正直なところ、どれを選んでもスコア差は誤差に近い。決め手はユースケース:

- **プロトタイプ・ローカル開発** → **Chroma**（pip install だけ、制限なし）
- **チーム共有・スケールアウト** → **TiDB or Pinecone**（マネージドで永続化不要）
- **既存アプリに SQL と一緒に組み込む** → **TiDB**（MySQL プロトコル互換、JOIN / 集計が使える）

自分のケースでは「LoRA データ管理の SQL とベクトル検索を統合したい」という理由で TiDB をメインに使い続けることにした。Chroma はローカル開発の高速プロトタイプ用として併用。

---

## まとめ

| やったこと | 結果 |
|---|---|
| 421 件のレビューを TiDB / Chroma / Pinecone に移行 | bge-large 1024 次元 embed、10 分以内 |
| 4条件 × 20 commit で比較 | 全 DB で RAGなし比 +1.0〜+1.6 点 |
| Pinecone が数字は最良 | 改善率 75%、平均 +1.60 |
| TiDB は SQL 統合が強み | スコアより「管理のしやすさ」に価値 |
| HNSW + WHERE は未サポート（v8.5.3） | top_k × 5 件取得 → Python フィルタで回避 |

「まず Chroma で試して、スケールが必要になったら TiDB / Pinecone へ」が現実的な移行パス。自分のプロジェクトみたいに SQLite の管理 DB が既にある場合は、最初から TiDB を選んで SQL と一緒に使う方が長期的に楽だと感じた。

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

- **M2LoRA**: https://github.com/hakaru/M2LoRA
- **M2DX-Core (OSS)**: https://github.com/hakaru/M2DX-Core
- **M2DX (TestFlight)**: https://testflight.apple.com/join/BAtGszPw
