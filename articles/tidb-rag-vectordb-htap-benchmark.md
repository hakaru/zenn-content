---
title: "「書きながら集計したい」 TiDB を選んだ理由 — HTAP をコードレビュー基盤で実測"
emoji: "📊"
type: "tech"
topics: ["tidb", "rag", "llm", "vectordatabase", "swift"]
published: true
---

TiDB Cloud（旧 Serverless / 2025年8月に Starter へ改名）で **INSERT を 30件/分 投入し続けても OLAP クエリの p50 が 14ms で安定する**ことを実測した。SQLite + ChromaDB の 2 システム構成と同条件で比較した結果も合わせて、ベクトル DB 選定の実データを共有する。

[前の記事（番外編①）](https://zenn.dev/hakaru/articles/tidb-rag-code-review-vector-comparison) でローカル LLM コードレビューに TiDB / ChromaDB / Pinecone の 3 DB を使った RAG を試して、品質はどの DB もほぼ同じ（Top-K 一致率 98〜99%、全 DB で +1〜+1.6 点改善）と分かった。

じゃあ何で DB を選ぶのか。

今回は **性能（Ingest スループット・検索レイテンシ）** と **運用コスト（2 システム問題と HTAP）** を実測した。後半では「個人開発で TiDB を選ぶと何が嬉しいか」を、朝のダッシュボードを開く場面から書いている。

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

...と思ったら Phase 0 のキャッシュ計算が `_original_embed` を直接呼んでいてキャッシュに書き込まれていなかった。「embed cache 保存: 0 件」。。。`_cached_embed` 経由に直したら正常に動作。

---

## ベンチマーク① — Ingest & Search

SQLite から `avg_score ≥ 4.0` の高品質レビュー 94 件を取得。70 件で ingest、20 件で検索のベンチ取得

### Ingest スループット（70件）

| DB | 総時間 | スループット |
|---|---|---|
| ChromaDB（ローカル） | 26.8 秒 | **2.6 件/秒** |
| TiDB Serverless | 79.4 秒 | 0.9 件/秒 |
| Pinecone Serverless | 111.8 秒 | 0.6 件/秒 |

ローカル vs クラウドの差がそのまま出ただけですかね。クラウド同士だと TiDB が Pinecone の 1.5 倍速い。

### 検索レイテンシ（20クエリ、エンドツーエンド）

p50/p95/p99 は分布指標（小さい順に並べたとき 50番目/95番目/99番目の値）。平均値より外れ値に引っ張られにくい。

| DB | p50 | p95 | p99 |
|---|---|---|---|
| ChromaDB（ローカル） | **42 ms** | 396 ms | 414 ms |
| TiDB Cloud（東京リージョン） | 1,157 ms | 1,260 ms | 1,264 ms |
| Pinecone（us-east-1） | 1,623 ms | 2,025 ms | 2,027 ms |

念のため断っておくと、これは**エンジン速度の比較ではない**。クラウド勢の p50/p95/p99 がほぼ団子（TiDB 1,157/1,260/1,264）なのは、HNSW 検索そのものじゃなくて**東京→クラウドの往復ネットワークが律速**してる指紋。HNSW の検索が1秒かかるわけがない。ローカルの Chroma はプロセス内呼び出しでネットワークがゼロだから速くて当然で、ここから「TiDB のエンジンが遅い」とは言えない。クラウド同士で TiDB が Pinecone より速く見えるのも、東京と us-east-1 の地理差。*エンジン単体のレイテンシを測るには同一リージョンのクライアントから叩く必要がある（次の宿題）。*

ChromaDB の p95 が 396ms と高いのは、ローカル HNSW の初回インデックス展開が走るから。2回目以降は安定する。

### Top-K 一致率（TiDB 基準）

上位 5 件の重複度。3 DB が同じ HNSW + コサイン距離（= 1 - コサイン類似度。0 に近いほど似ている）で実装されているので、返す結果もほぼ一致する。

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

これがTiDBの売りなんですが。。。
同じテーブルにベクトルも入っている。管理するシステムは 1 つで済む。

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
ChromaDB + SQLite:  OLAP 3.5ms ✅  Vector 1.3ms(ローカル)   ✅  2システム管理 ❌
TiDB Cloud:         OLAP  15ms ✅  Vector 1.2s(往復NW込み)  ✅  1システムで完結 ✅
```

「書きながら集計する」LoRA データ収集みたいなユースケースには TiDB の 1 本管理がかなり合ってる。

---

## TiDB が自然にフィットする開発スタイル

数字だけだとイメージしにくいので、具体的な場面で書いてみる。

### 朝、コーヒー飲みながらダッシュボードを開く

前の晩に M2DX と 1Take 両方で何件かコミットがあった。パイプラインが自動で回って採点まで終わっている。

「先週のコミット、スコア低いの何件？」「v3 と v4 のレビューモデル、どっちが品質高い？」「1Take だけで見ると傾向どう？」

```sql
-- 全部同じ DB から一発で引ける
SELECT project, review_model,
       AVG((claude_score + codex_score + gemini_score) / 3.0) AS avg_score,
       COUNT(*) AS cnt
FROM reviews
WHERE created_at >= NOW() - INTERVAL 7 DAY
GROUP BY project, review_model
ORDER BY avg_score DESC;
```

ChromaDB + SQLite の 2 システムだと、集計は SQLite、ベクトル検索は ChromaDB と 2 か所に問い合わせないといけない。TiDB なら SQL 1 本で終わる。

### バックフィルしながらリアルタイムで確認する

100 件のバックフィルを流している最中に、別ウィンドウで「今どこまで処理した？低スコアなコミットはどれ？」をリアルタイム確認できる。

HTAP の数字がここで効いてくる。書き込みが走っていても OLAP は 14ms で安定している。SQLite だとバックフィル中に集計クエリを叩くとライトロックが競合して遅くなる。

### アプリが増えても同じ DB に足していくだけ

M2DX → 1Take → 次のアプリ、と増えていっても `project` カラムで絞るだけ。同じパイプライン、同じ DB。ChromaDB だとアプリごとに collection を増やして管理が散らかっていく。

---

一言で言うと **「CI/CD の感覚で品質を追い続ける開発スタイル」** にフィットする。

コミットが来るたびに自動で蓄積されて、いつでも「最近のスコアトレンド」「モデル世代の比較」「プロジェクト横断の傾向」が SQL で引ける状態。単一クエリの速度より「ベクトルと集計が同じ場所にある」ことの方が日々の開発で効いてくる。

個人開発の規模だと TiDB の本領（水平スケール、大量トラフィック下の HTAP）はまだ眠っている。でもコミットが 1,000、10,000 と増えたときに 2 システム構成のままだと運用が破綻するのは目に見えていて、今のうちから 1 本にしておく投資は悪くない、というのが今回の体感。

---

## まとめ

| 計測内容 | 結果 |
|---|---|
| Ingest スループット | ローカルの ChromaDB 2.6件/秒。クラウド勢は TiDB 0.9 > Pinecone 0.6 |
| 検索 p50（往復NW込み） | ローカル Chroma 42ms。クラウド勢はネットワーク律速で TiDB 1,157ms / Pinecone 1,623ms（エンジン速度の比較ではない） |
| Top-K 一致率 | 98〜99%。どのDBも同じ結果を返す＝**検索品質はDBで変わらない** |
| HTAP | TiDB: 30件/分 INSERT 中も OLAP 14ms で無劣化。敗者は SQLite + Chroma の **2システム構成**（同期コストと不整合リスク） |

- **プロトタイプ・ローカルだけで完結** → ChromaDB。`pip install` で終わり、速い
- **クラウド本番・SQLと一緒に運用** → TiDB。ベクトルと集計が1テーブルに同居して管理が1システムで済む（2システム構成の同期地獄を回避できるのが本質）
- **書きながら集計したい（ダッシュボード・世代比較）** → TiDB。HTAP で書き込み負荷に強く、管理クエリが SQL でそのまま書ける

RAG 品質の比較は[前の記事](https://zenn.dev/hakaru/articles/tidb-rag-code-review-vector-comparison)に書いたのでそちらも。まだまだ続く。。。

---

:::details 対象プロジェクト

**1Take** — ミュージシャン向け録音 iOS アプリ。録音ボタン 1 つで LA-2A / 1176 系のリアルタイムコンプレッサー掛け取り。

https://apps.apple.com/us/app/1take/id6757945099

**M2DX** — iOS/macOS 向け MIDI 2.0 対応 DX7 互換 FM シンセサイザー。TestFlight で公開ベータ配布中。

https://testflight.apple.com/join/BAtGszPw

**M2LoRA** — 上記アプリのコミットを自動レビュー・採点・合成して LoRA 学習データを溜めるパイプライン（本記事の題材）。
:::

---

:::message
**シリーズの他の記事**

- [（１）監査編 — 指摘 52 件、真陽性 0 件](https://zenn.dev/hakaru/articles/m2dx-local-llm-audit-zero-true-positives)
- [（２）RAG 編 — Swift 仕様書で誤検出 76% 削減](https://zenn.dev/hakaru/articles/m2dx-local-llm-audit-rag)
- [（３）aider 編 — 参照軸は改善、知識軸は依然ダメ](https://zenn.dev/hakaru/articles/m2dx-local-llm-agentic-harness-eval)
- [（４）LoRA 編 — 誤検知 93% 削減](https://zenn.dev/hakaru/articles/swift-audit-lora-fp-reduction)
- [（５）M2LoRA パイプライン編 — 開発しながら自動でデータが貯まる仕組み](https://zenn.dev/hakaru/articles/m2lora-code-review-pipeline)
- [（番外）ベクトルDB比較 RAG品質編](https://zenn.dev/hakaru/articles/tidb-rag-code-review-vector-comparison)
- [（応用）AIエージェントの記憶は「夢」で整理する — TiDB に観測2,218件を溜めた8週間の実録](https://zenn.dev/hakaru/articles/memdream-tidb-vector-ai-memory)
:::

*本記事は [Zennfes Spring 2026 × TiDB](https://zenn.dev/contests/zennfes-spring-2026-tidb) への応募作品です。*
