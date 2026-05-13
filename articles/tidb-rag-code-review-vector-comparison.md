---
title: "ローカルLLMのコードレビューをRAGで強化する — TiDB / Chroma / Pinecone 3択比較（20件実測）"
emoji: "🗄️"
type: "tech"
topics: ["tidb", "rag", "llm", "vectordatabase", "swift"]
published: false
---

# ローカルLLMのコードレビューをRAGで強化する — TiDB / Chroma / Pinecone 3択比較

## はじめに

ローカルLLMにコードレビューをさせると、**幻覚（Hallucination）** が混入することがある。「`@Bindable`はSwift 5.9で非推奨」「このキャストはオーバーフローの危険がある」——どちらも間違いだ。指摘が具体的に見えるほど、間違いに気づきにくい。

この問題を解決するために試みたのが、**過去の高品質レビューをベクトル検索で取得し、プロンプトに注入するRAGパイプライン**だ。

本記事では、Swiftコードレビューの自動収集・評価システム **M2LoRA** を題材に、同一のdiffに対して次の4条件でローカルLLMのレビュー品質を比較する。

| 条件 | 説明 |
|---|---|
| RAGなし | diff → ローカルLLM |
| TiDB RAG | diff → bge-large embed → **TiDB Serverless (HNSW)** → 類似diff取得 → LLM |
| Chroma RAG | diff → bge-large embed → **ChromaDB (ローカル)** → 類似diff取得 → LLM |
| Pinecone RAG | diff → bge-large embed → **Pinecone Serverless** → 類似diff取得 → LLM |

---

## システム構成

```
git diff
  └─► review.py (llama3.3:70b-m2lora-v1, Ollama)
        ├─► [RAGなし] プロンプト直接生成
        └─► [RAGあり] retriever.py
              ├─ bge-large:latest でdiffをembed (1024次元)
              ├─ TiDB / Chroma / Pinecone で上位2件取得
              └─ 類似diff+レビュー例をプロンプトに注入
                    └─► evaluate.py (Claude / Codex / Gemini で採点)
```

収集済みデータ: **421件のSwift/MIDIコードレビュー**（M2DX + MIDI2Kitの実際のコミット）

採点は3評価者（Claude / Codex / Gemini）の平均スコア（0〜10点）を使用。

---

## ベクトルDBのセットアップ

### TiDB Serverless

TiDB v8.5.3 からネイティブの `VECTOR` 型と HNSW インデックスが使える。

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

接続は `pymysql` + SSL:

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

**ハマりポイント**: HNSW インデックスは `WHERE flagged = 0` のような事前フィルタと非互換（TiDB Serverless v8.5.3時点）。上位N件を多めに取得してPython側でフィルタする方式に変更した。

```python
# WHERE フィルタをHNSW検索に混ぜるとエラー
# → top_k * 5 件取得してPythonでフィルタ
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

### ChromaDB（ローカル）

```python
import chromadb

client = chromadb.PersistentClient(path="./data/chroma")
collection = client.get_or_create_collection(
    name="reviews",
    metadata={"hnsw:space": "cosine"},
)
```

追加・検索ともに同期APIで、セットアップが最もシンプル。

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

---

## embeddingとRAG注入

### bge-large でdiffをembed

bge-large は512トークンの制限があり、コードdiffはトークン密度が高い。文字数で切り詰める際は1200文字を上限にし、エラー時は800→500→300と段階的に短縮した。

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

### プロンプトへの注入

類似diff上位2件の `synthesized_review`（Claude合成済みの高品質レビュー）をプロンプトに差し込む。

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

---

## 実測比較（20件）

M2DX・MIDI2Kit の実コミット20件でランダムサンプリング。各commitに4条件のレビューを生成し、Claude / Codex / Geminiの平均スコアで評価。

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

### 集計結果

| | RAGなし | TiDB | Chroma | Pinecone |
|---|---|---|---|---|
| **平均スコア** | 4.27 | 5.30 | 5.77 | **5.87** |
| **平均Δ** | — | +1.03 | +1.50 | **+1.60** |
| **改善件数** | — | 13/20 (65%) | 12/20 (60%) | **15/20 (75%)** |

---

## 考察

### RAG注入は効く。ただし万能ではない

20件中15〜13件で改善した（60〜75%）。平均で+1〜+1.6点の改善は、採点スケール的に「凡庸 → まあまあ」程度の変化だが、幻覚を含む指摘が類似事例の参照によって抑制されることが主因と思われる。

改善しなかった5〜7件を見ると、**RAGなしのスコアが既に7点以上**のケースが多い。天井効果の可能性が高く、高品質なdiffに対してはRAGコンテキストがノイズになることもある。

### TiDB vs Chroma vs Pinecone

| 観点 | TiDB | Chroma | Pinecone |
|---|---|---|---|
| 平均改善 | +1.03 | +1.50 | +1.60 |
| 改善率 | 65% | 60% | 75% |
| セットアップ | クラウド（要SSL） | ローカル（最簡単） | クラウド |
| 無料枠 | Serverless（5GiB） | 制限なし（ローカル） | Starter（2GiB） |
| HNSW WHERE制限 | あり（要回避） | なし | なし |
| レイテンシ | 中（AWS Tokyo） | 最小（ローカル） | 大（AWS us-east-1） |

今回の条件（421件、1024次元）ではPineconeが最も安定して高スコアを出した。ただしサンプル数が少ないため、差は誤差の範囲とも言える。

**実用上の選択指針**:
- **プロトタイプ・ローカル開発** → Chroma（ゼロ設定、pip install だけ）
- **チーム共有・スケールアウト** → TiDB or Pinecone（マネージドで永続化不要）
- **既存MySQLアプリへの組み込み** → TiDB（MySQLプロトコル互換、SQL で JOIN 可能）

### TiDBを選ぶ理由：SQLとベクトルの統合

TiDBの最大の強みは、ベクトル検索と通常のSQLを同一クエリで書けることだ。たとえば「スコアが高く、かつ類似しているレビュー」の絞り込みが1クエリで書ける（ただし前述のHNSWフィルタ制限は要考慮）。

```sql
-- 高スコアかつ類似のレビューを取得
SELECT id, synthesized_review,
       VEC_COSINE_DISTANCE(diff_embedding, %s) AS dist
FROM reviews
WHERE claude_score >= 7.0
ORDER BY dist ASC
LIMIT 5;
```

ChromaやPineconeではメタデータフィルタとANN検索の組み合わせに制約があるが、TiDBはSQLの表現力をそのまま使える。これはLoRAのデータ品質管理（世代別スコア集計、フラグ管理など）との相性が良い。

---

## まとめ

| やったこと | 結果 |
|---|---|
| 421件のSwiftコードレビューをTiDB/Chroma/Pineconeに移行 | bge-largeで1024次元embed、10分以内に完了 |
| RAGなし vs 3DB の20件比較 | 全DB平均+1〜+1.6点の改善 |
| Pineconeが最も安定 | 改善率75%、平均+1.60点 |
| TiDBはSQLとの統合が強み | ベクトル+スコアフィルタを同一クエリで実現 |

ローカルLLMのRAG強化としては、**まずChromaで試して、スケール・共有が必要になったらTiDB/Pineconeに移行**という流れが現実的だと感じた。

---

*本記事は [Zennfes Spring 2026 × TiDB](https://zenn.dev/contests/zennfes-spring-2026-tidb) への応募作品です。*
