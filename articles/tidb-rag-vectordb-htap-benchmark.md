---
title: "ベクトルDB 3択を性能で測った + TiDB の HTAP を実証した — TiDB / ChromaDB / Pinecone"
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

[前の記事（番外編）](https://zenn.dev/hakaru/articles/tidb-rag-code-review-vector-comparison) でローカル LLM コードレビューに TiDB / ChromaDB / Pinecone の 3 DB を使った RAG を試した。

RAG の品質（レビュースコア改善幅）はどの DB もほぼ同じだった。全 DB で +1〜+1.6 点、Top-K 一致率も 98〜99% で誤差の範囲。**DB を変えても検索品質は変わらない**、というのが前回の結論。

じゃあ何で DB を選ぶんだという話になる。

今回は **性能（Ingest スループット・検索レイテンシ）** と **運用コスト（2システム問題と HTAP）** を実測した。

---

## 計測の前提

構成は前の記事と同じ。

```
git diff → bge-large:latest（1024次元）でベクトル化 → DB で類似 diff 検索 → プロンプト注入 → llama3.3:70b でレビュー生成
```

データは SQLite に蓄積した **421 件の Swift/MIDI コードレビュー**（M2DX + MIDI2Kit の実コミット）。

DBのセットアップや embedding のコードは前の記事に書いたので省略。ひとつだけ今回のベンチマーク用に工夫した点がある。

### embedding を切り離して DB 単体の性能を測る

3 DB の速度を比べるとき、embedding（Ollama bge-large、35〜45ms/回）の時間が毎回乗ると DB の差が見えなくなる。

なので embedding を事前に全件計算してキャッシュし、Phase 1/2 では DB 呼び出しだけを計測する設計にした。

```python
_embed_cache: dict[str, list[float]] = {}

async def _cached_embed(text: str) -> list[float]:
    key = text[:200]
    if key in _embed_cache:
        return _embed_cache[key]
    vec = await _original_embed(text)
    _embed_cache[key] = vec
    return vec

# 計測前に差し替える
_ret_mod.embed = _cached_embed
```

...と思ったら Phase 0 のキャッシュ計算が `_original_embed` を直接呼んでいてキャッシュに書き込まれていなかった。「embed cache 保存: 0 件」。。。`_cached_embed` 経由に直したら正常に動作。こういうバグに気づかないまま計測してたら嫌だった。

---

## ベンチマーク① — Ingest & Search

SQLite から `avg_score ≥ 4.0` の高品質レビュー 94 件を取得。70 件で ingest、20 件で検索のベンチを取った。

### Ingest スループット（70件）

| DB | 総時間 | スループット |
|---|---|---|
| ChromaDB（ローカル） | 26.8 秒 | **2.6 件/秒** |
| TiDB Serverless | 79.4 秒 | 0.9 件/秒 |
| Pinecone Serverless | 111.8 秒 | 0.6 件/秒 |

ローカル vs クラウドの差がそのまま出た。クラウド同士だと TiDB が Pinecone の 1.5 倍速い。

### 検索レイテンシ（20クエリ、embedding 時間を除くDB単体）

p50/p95/p99 は分布指標（小さい順に並べたとき 50番目/95番目/99番目の値）。平均値より外れ値に引っ張られにくい。

| DB | p50 | p95 | p99 |
|---|---|---|---|
| ChromaDB（ローカル） | **42 ms** | 396 ms | 414 ms |
| TiDB Serverless | 1,157 ms | 1,260 ms | 1,264 ms |
| Pinecone Serverless | 1,623 ms | 2,025 ms | 2,027 ms |

ChromaDB の p95 が 396ms と高いのは、ローカル HNSW の初回インデックス展開が走るから。2回目以降は低レイテンシに安定する。

クラウド同士だと TiDB が Pinecone より約 30% 速かった。Pinecone の Starter プランが us-east-1 固定なのが効いてる。

### Top-K 一致率（TiDB 基準）

上位 5 件の重複度。3 DB が同じ HNSW + コサイン距離（= ベクトルの向きの差を 0〜2 で表す指標）で実装されているので、返す結果もほぼ一致する。

| 比較対象 | TiDBとの一致率 |
|---|---|
| ChromaDB | **99%** |
| Pinecone | **98%** |

**DB を変えても検索品質は変わらない**。前の記事の結論と一致した。どの DB を選ぶかはスコアではなく性能と運用で決めることになる。

---

## ベンチマーク② — HTAP。これが一番面白かった

### 2システム問題

ChromaDB はベクトル専用 DB なので `AVG(score) GROUP BY review_model` みたいな分析クエリが書けない。LoRA のデータ収集パイプラインで「世代ごとの平均スコアを追いたい」「フラグされたレビューの割合を見たい」みたいな需要が普通に出てくる。

だから実運用では SQLite + ChromaDB の 2 システム構成になりがちだ。

```
書き込み:
  ├─► SQLite  INSERT（スコア・メタデータ）
  └─► ChromaDB upsert（ベクトル）
            ↑ 2か所に書く

読み取り:
  ├─► SQLite  SELECT AVG(score)...（集計）
  └─► ChromaDB query(vec, top_k)（検索）
            ↑ 2か所から読む
```

同期コストも高いし、片方だけ書き込み失敗したときの不整合も怖い。

### TiDB は SQL + ベクトルが同一 DB で完結する

```sql
-- 世代別の平均スコア集計とベクトル検索が同一DBで書ける
SELECT review_model,
       AVG((claude_score + codex_score + gemini_score) / 3.0) AS avg_score,
       COUNT(*) AS cnt
FROM reviews
WHERE diff_embedding IS NOT NULL
GROUP BY review_model;
```

これと同じテーブルにベクトルも入っている。管理するシステムは 1 つで済む。

TiDB の HTAP（= Hybrid Transactional/Analytical Processing、トランザクションと分析を同一 DB でこなすアーキテクチャ）は行ストア（TiKV）と列ストア（TiFlash）を内部で分離していて、INSERT と AVG/GROUP BY が互いに干渉しないらしい。本当にそうなのか測ってみた。

### 計測方法

非同期で 2 つのタスクを同時に走らせる。

- **writer**: INSERT を N 件/分のレートで投入し続ける
- **reader**: 分析クエリ（`AVG(score) GROUP BY review_model`）を 3 秒ごとに実行、レイテンシを記録

INSERT 負荷を 0 → 10 → 30 件/分と変えて、OLAP レイテンシへの影響を計測。TiDB と SQLite+ChromaDB（2システム）を同条件で比較。

### 結果

**OLAP クエリ（avg_score_by_model）p50 レイテンシ（ms）**

| INSERT 負荷 | TiDB | SQLite | ChromaDB（別システム） |
|:-----------:|:----:|:------:|:---------------------:|
| 0 件/分（ベースライン） | 15.1 | 3.5 | 1.3 |
| 10 件/分 | 14.5 | 5.0 | 1.5 |
| 30 件/分 | **14.1** | 3.6 | 1.5 |

TiDB、**全く劣化しない**。30件/分 INSERT しながらでも 14ms で安定している。行ストアへの書き込みが列ストアへのクエリに干渉していない、というのが数字に出た。

SQLite は 10件/分 で 5.0ms に増加（ライトロックの競合）。絶対値は速いけど、ベクトル検索のために ChromaDB への別クエリが必要になる。

```
ChromaDB + SQLite:  OLAP 3.5ms ✅  Vector 1.3ms ✅  2システム管理 ❌
TiDB Serverless:    OLAP  15ms ✅  Vector 1.2s  ✅  1システムで完結 ✅
```

「書きながら集計する」LoRA データ収集みたいなユースケースには TiDB の 1 本管理がかなり合ってる。

---

## まとめ

| 計測内容 | 結果 |
|---|---|
| Ingest スループット | ChromaDB 2.6件/秒 ≫ TiDB 0.9 > Pinecone 0.6 |
| 検索 p50（DB単体） | ChromaDB 42ms ≪ TiDB 1,157ms < Pinecone 1,623ms |
| Top-K 一致率 | 98〜99%。どのDBも同じ結果を返す |
| HTAP | TiDB: 30件/分 INSERT 中も OLAP 14ms で安定。SQLite + Chroma は 2 システム管理が必要 |

- **プロトタイプ・ローカル** → ChromaDB 一択。`pip install` して終わり、速い
- **クラウド本番・チーム共有** → TiDB（Pinecone より速くて SQL とベクトルが 1 本）
- **書きながら集計もしたい** → TiDB（HTAP で書き込み負荷に強く、SQL で管理クエリが書ける）

RAG 品質の比較は[前の記事](https://zenn.dev/hakaru/articles/tidb-rag-code-review-vector-comparison)に書いたのでそちらも。まだまだ続く。。。

---

:::message
**シリーズの他の記事**

- [（１）監査編 — 指摘 52 件、真陽性 0 件](https://zenn.dev/hakaru/articles/m2dx-local-llm-audit-zero-true-positives)
- [（２）RAG 編 — Swift 仕様書で誤検出 76% 削減](https://zenn.dev/hakaru/articles/m2dx-local-llm-audit-rag)
- [（３）aider 編 — 参照軸は改善、知識軸は依然ダメ](https://zenn.dev/hakaru/articles/m2dx-local-llm-agentic-harness-eval)
- [（４）LoRA 編 — 誤検知 93% 削減](https://zenn.dev/hakaru/articles/swift-audit-lora-fp-reduction)
- [（５）M2LoRA パイプライン編 — 開発しながら自動でデータが貯まる仕組み](https://zenn.dev/hakaru/articles/m2lora-code-review-pipeline)
- [（番外）ベクトルDB比較 RAG品質編](https://zenn.dev/hakaru/articles/tidb-rag-code-review-vector-comparison)
:::

*本記事は [Zennfes Spring 2026 × TiDB](https://zenn.dev/contests/zennfes-spring-2026-tidb) への応募作品です。*
