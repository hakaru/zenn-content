---
title: "AIコーディングアシスタントに「長期記憶」を持たせたら、プロジェクト横断でバグが見つかるようになった"
emoji: "🧠"
type: "tech"
topics: ["tidb", "mcp", "claude", "vectordb", "ai"]
published: false
---

## Claude Code、昨日の会話覚えてない問題

iOS/macOS 向けの MIDI 2.0 FM シンセサイザー **M2DX** と、録音アプリ **1Take** を開発してる。どっちも TestFlight でベータ配布中。

https://testflight.apple.com/join/BAtGszPw

普段の開発は Claude Code + Codex + Gemini CLI の3本立て。で、毎日使ってると気づく。

*「昨日直したあのバグ、今日のセッションでは忘れてる」*

セッションが切れるとコンテキストがリセットされる。CLAUDE.md に書けば多少は残るけど、プロジェクトをまたいだ記憶は持てない。M2DX-Core で修正した SPSC リングバッファの問題、TineModeler3 にも同じパターンがあるのに、AI はそれを知らない。

で、作ったのが **memdream**。TiDB Cloud のベクトル検索を使って、AI コーディングアシスタントに「長期記憶」を持たせる MCP サーバー。

https://github.com/hakaru/memdream

## なぜ TiDB なのか

ベクトル DB は選択肢が多い。Pinecone、Weaviate、Qdrant、pgvector…。その中で TiDB Cloud Serverless を選んだ理由は3つ。

**1. HTAP（= トランザクション + 分析が同一 DB で動く）**

memdream は「観測の書き込み」と「ベクトル検索」を同時にやる。MCP ツールが `memory_observe` で INSERT したデータを、直後に `memory_search` で `VEC_COSINE_DISTANCE` 検索する。普通の RDB にベクトル拡張を足しただけだと、書き込み直後の検索が遅延したりする。TiDB は TiFlash レプリカにベクトルインデックス（HNSW）を持てるので、書き込みはそのまま MySQL 互換の行ストアに入り、検索は列ストア側で走る。

```sql
-- 普通の INSERT（行ストア）
INSERT INTO observations (project_id, title, content, embedding, ...)
VALUES (?, ?, ?, ?, ...);

-- 直後にベクトル検索（TiFlash の HNSW）
SELECT *, VEC_COSINE_DISTANCE(embedding, ?) AS distance
FROM observations
WHERE embedding IS NOT NULL
ORDER BY distance ASC
LIMIT 10;
```

これが同じテーブルに対して同時に走る。別々の DB を立てる必要がない。

**2. MySQL 互換**

`mysql2/promise` でそのまま繋がる。ORM 不要。TLS 必須だけど `ssl: { minVersion: "TLSv1.2" }` を足すだけ。

```typescript
import { createPool } from "mysql2/promise";
const pool = createPool({
  host: process.env.TIDB_HOST,   // gateway01.ap-northeast-1.prod.aws.tidbcloud.com
  port: 4000,
  user: process.env.TIDB_USER,
  password: process.env.TIDB_PASSWORD,
  database: "memdream",
  ssl: { minVersion: "TLSv1.2" },  // ← これだけ
  connectionLimit: 5,
});
```

ベクトル専用 DB だと独自クライアントや REST API が必要になるけど、TiDB は使い慣れた MySQL ドライバーでいい。

**3. Serverless の無料枠が実用的**

TiDB Cloud Serverless は月 25 GiB ストレージ + 250M RU（Request Unit）が無料。個人の開発記憶用途なら余裕で収まる。今 168 observations + 19 consolidated memories + 111 knowledge graph triples が入ってて、ストレージは数 MB。

## スキーマ設計: ベクトルと RDB の合わせ技

TiDB の面白いところは「普通の RDB テーブルにベクトルカラムを足せる」こと。

```sql
CREATE TABLE observations (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  project_id BIGINT NOT NULL,
  title VARCHAR(256) NOT NULL,
  content TEXT NOT NULL,
  type VARCHAR(32),
  -- ↓ ここがベクトル
  embedding VECTOR(1024),
  embedding_model VARCHAR(64) DEFAULT 'bge-large',
  -- ↓ 重複排除: 同じ内容を二重記録しない
  content_hash VARCHAR(64) AS (SHA2(CONCAT(project_id, ':', title, ':', LEFT(content, 1000)), 256)) STORED,
  UNIQUE INDEX idx_content_dedup (content_hash),
  FOREIGN KEY (project_id) REFERENCES projects(id)
);
```

`content_hash` は生成カラム（STORED）。`project_id:title:content[:1000]` の SHA2 ハッシュを自動計算して UNIQUE 制約をかける。`INSERT IGNORE` すれば同じ内容の二重登録をゼロコストで弾ける。

*これ、ベクトル専用 DB だとできない。* Pinecone や Qdrant にはトランザクション制約もハッシュ生成カラムもない。RDB の機能とベクトル検索が同居してるのが TiDB の強み。

ベクトルインデックスは HNSW で作る:

```sql
ALTER TABLE observations SET TIFLASH REPLICA 1;
ALTER TABLE observations ADD VECTOR INDEX idx_obs_vec ((VEC_COSINE_DISTANCE(embedding)))
  USING HNSW;
```

TiFlash レプリカを1つ追加して、その上に HNSW インデックスを張る。検索は自動で TiFlash 側に飛ぶので、行ストアへの書き込み性能には影響しない。

## 3つのテーブルで「記憶の階層」を作る

memdream のスキーマは6テーブルあるけど、記憶の本体は3つ。

| テーブル | 役割 | ベクトル |
|---|---|---|
| `observations` | 生の記録。「何を見つけたか」 | ✅ VECTOR(1024) |
| `consolidated_memories` | dream 処理後の要約。「何を学んだか」 | ✅ VECTOR(1024) |
| `knowledge_graph` | 関係性。「何が何に依存してるか」 | なし（RDF トリプル） |

観測（observations）は日々のセッションで `memory_observe` 経由で溜まる。1Take の paywall バグ修正、M2DX のオーディオスレッド安全性レビュー、TineModeler3 の SPSC リング修正…。生の作業ログ。

統合記憶（consolidated_memories）は dream-agent が作る。観測を12件ずつまとめて Ollama qwen3:14b に「要約して」と投げ、結果をベクトル化して格納。`scope` カラムで project / ecosystem / global の3段階にスコープ分け。

```sql
-- エコシステム横断の統合記憶を取得
SELECT * FROM consolidated_memories
WHERE (scope = 'project' AND scope_ref IN ('m2dx', 'm2dx-core', 'midi2kit'))
   OR (scope = 'ecosystem' AND scope_ref = 'm2dx-ecosystem')
ORDER BY created_at DESC;
```

ナレッジグラフ（knowledge_graph）は主語-述語-目的語のトリプル。

```
M2DX --implements--> lock-free triple buffer
M2DX --depends-on--> MIDI2Kit
TineModeler3 --fixed--> SPSC data race
```

ベクトル検索で「意味的に近い記憶」を引き、ナレッジグラフで「構造的な関係」を引く。両方あるから「M2DX のロックフリー設計どうだっけ？」と聞くと、過去のレビュー結果と関連プロジェクトの修正履歴がまとめて返ってくる。

## MCP サーバーで AI ツールと繋ぐ

memdream は MCP（Model Context Protocol）サーバーとして Claude Code に接続する。stdio トランスポートでステートレス。9つのツールを公開。

```bash
# ユーザースコープで登録（全プロジェクトで使える）
claude mcp add-json -s user memdream '{
  "command": "node",
  "args": ["/path/to/memdream/packages/mcp-server/dist/index.js"],
  "env": { "TIDB_HOST": "...", ... }
}'
```

実際の使われ方はこう:

```
# セッション開始時（自動）
memory_session_start(project="m2dx")
→ 統合記憶5件 + ナレッジグラフ10件 + 最近の観測5件が返る

# 実装中に過去の知見を検索
memory_search(query="SPSC リングバッファの修正")
→ M2DX-Core と TineModeler3 の両方の修正履歴がヒット

# 作業完了時に記録
memory_observe(project="m2dx", type="bugfix", title="FXParamBox lock-free化",
  content="os_unfair_lock → Atomic triple buffer。レンダースレッド wait-free。")
```

ここで TiDB の HTAP が効く。`memory_observe` の INSERT と `memory_search` の `VEC_COSINE_DISTANCE` が同じ DB に対して同時に走っても、行ストアと列ストアで分離されてるから干渉しない。

## 嵌まったポイント: TiDB の prepared statement と LIMIT

開発中に面白いバグに当たった。`pool.execute()` で `LIMIT ?` にパラメータを渡すと、TiDB が「Incorrect arguments to LIMIT」で落ちる。

```typescript
// ❌ これが動かない
const sql = `SELECT * FROM observations ORDER BY distance ASC LIMIT ?`;
await pool.execute(sql, [10]);
// → "Incorrect arguments to LIMIT"

// ✅ バリデーション済みの値を直接埋め込む
const limit = Math.min(Math.max(args.limit ?? 10, 1), 50);
const sql = `SELECT * FROM observations ORDER BY distance ASC LIMIT ${limit}`;
```

MySQL の prepared statement で LIMIT にパラメータを渡すのは元々グレーゾーンらしいけど、TiDB では明確にエラーになる。`limit` は 1-50 の整数に事前バリデーション済みなので、直接埋め込みでも SQL インジェクションリスクはない。

*レビューを依頼したら、レビュー対象のツール自体にバグがあって検索が動かない、という状況。* Codex と Gemini に memdream のフルレビューを依頼して、そのレビュー結果を memdream に記録しようとしたら `memory_search` が壊れてた。

## dream-agent: 観測を「記憶」に昇華する

生の観測を溜めるだけだと、ノイズが多すぎて検索精度が落ちる。dream-agent は観測を統合記憶に昇華するバッチ処理。

```
168 observations
  → プロジェクト別にグループ化
  → 12件ずつチャンク
  → Ollama qwen3:14b で要約 + トリプル抽出（JSON）
  → Ollama bge-large でベクトル生成
  → consolidated_memories + knowledge_graph に INSERT
  → dream_runs に完了記録
```

LLM の出力を JSON でパースするところが地味に面倒で、qwen3 は `<think>` ブロックを出力に混ぜてくるし、JSON を markdown フェンスで囲むこともある。防御的にパースしてる。

```typescript
function extractJSON(raw: string): object | null {
  // qwen3 の <think>...</think> を除去
  let cleaned = raw.replace(/<think>[\s\S]*?<\/think>/g, "");
  // markdown フェンス除去
  cleaned = cleaned.replace(/```json?\s*/g, "").replace(/```/g, "");
  // JSON オブジェクトを抽出
  const match = cleaned.match(/\{[\s\S]*\}/);
  if (!match) return null;
  return JSON.parse(match[0]);
}
```

168件の観測を処理して、**19件の統合記憶 + 111件のナレッジグラフトリプル**が生成された。処理時間は約7分。

## 実際に効果が出た例

### M2DX: オーディオスレッドで os_unfair_lock を使ってた

M2DX のオーディオエンジンで `FXParamBox` が `os_unfair_lock` を使っていた。オーディオレンダースレッドは「絶対にロックを取らない」が鉄則なのに。

Codex と Gemini の両方が同じ場所を指摘。memdream に記録した。

次に TineModeler3 のセッションで `memory_search("lock-free audio thread")` すると、M2DX の修正履歴がヒットする。「Atomic triple buffer で置き換えた」という具体的な解決策付き。

*プロジェクトが違うのに、同じ問題の解決策が引ける。*

### 1Take: レビュー結果が自動蓄積

1Take のセッションで paywall 周りのバグ修正をやると、Claude Code が自動で `memory_observe` を呼んで記録してくれる。`~/.claude/CLAUDE.md` に「作業完了時は必ず memory_observe」と書いてあるから。

次のセッションで `memory_session_start` すると、前回の修正内容が統合記憶として返ってくる。「あ、昨日 P-001 と P-002 直したんだった」と思い出す手間がない。

### MIDI2Kit: CRITICAL 3件を発見 → 修正 → 記録

Codex が MIDI2Kit に CRITICAL 3件を出した:
- `PEManager.deinit` が continuation を resume せず破棄 → 永久ハング
- `CoreMIDITransport` の mutable state がロックなし → データレース
- SysEx7 受信バッファが無制限増加 → メモリ枯渇

3件とも修正して705テスト全パス。結果は memdream に記録済み。M2DX のセッションで `memory_recall(project="midi2kit")` すると、この修正内容がエコシステム記憶として返ってくる。M2DX は MIDI2Kit に依存してるから。

## dream-agent を実際に回してみた

記事を書いてる途中にも observations が増えるので、dream-agent を走らせてみた。

```
🌙 memdream dream-agent 起動

📊 対象プロジェクト: 6件
   - 1take (7件の未統合観測, ecosystem: music-apps)
   - memdream (6件の未統合観測)
   - midi2kit (1件の未統合観測, ecosystem: m2dx-ecosystem)
   - tinemodeler4 (1件の未統合観測, ecosystem: tinemodeler-ecosystem)
   - m2dx (1件の未統合観測, ecosystem: m2dx-ecosystem)
   - m2dx-core (1件の未統合観測, ecosystem: m2dx-ecosystem)

  📦 1take: 7件の観測を処理中...
    ✅ メモリ作成: "Paywall UX Optimization and Analytics Tracking Enhancements"
  📦 memdream: 6件の観測を処理中...
    ✅ メモリ作成: "MemDream Global Implementation and System Enhancements"

🌐 m2dx-ecosystem: 3プロジェクト、3件のメモリを横断分析中...
  ✅ エコシステムメモリ作成: "Concurrency Management and Buffer Optimization in m2dx-ecosystem"

✨ Dream Run #30001 完了
   観測処理数: 17
   メモリ作成数: 7
   トリプル作成数: 32
```

17件の新規 observations が処理されて、7件の統合メモリ + 32件のナレッジグラフトリプルが生成された。処理時間は約2分。

ポイントは **冪等性**。前回の dream run で処理済みの168件はスキップされて、新規17件だけが処理される。`observations` テーブルの `consolidated` フラグ（BOOLEAN）で管理してる。

```sql
-- dream-agent が参照するクエリ
SELECT * FROM observations
WHERE project_id = ? AND consolidated = FALSE
ORDER BY created_at ASC;

-- 処理後に更新
UPDATE observations SET consolidated = TRUE, dream_run_id = ?
WHERE id IN (?);
```

この「フラグ + FK で処理済みを追跡」ができるのも RDB ならでは。ベクトル DB に boolean カラムや外部キーはない。

エコシステム統合も面白い。m2dx-ecosystem（M2DX + M2DX-Core + MIDI2Kit）の3プロジェクトのメモリを横断分析して「Concurrency Management and Buffer Optimization」というエコシステムレベルの知見が自動生成された。*3つの別プロジェクトで同じ「並行性とバッファ管理」の問題を直してた、ということを AI が勝手にまとめてくれる。*

## 数字で見る memdream

| 指標 | 値 |
|---|---|
| 登録プロジェクト | 8（M2DX, M2DX-Core, MIDI2Kit, 1Take, TineModeler3, TineModeler4, PeerClock, memdream） |
| observations | 185 |
| consolidated_memories | 26 |
| knowledge_graph triples | 143 |
| エコシステム | 3（m2dx-ecosystem, tinemodeler-ecosystem, music-apps） |
| dream runs | 2回（初回168件 → 2回目17件、冪等に増分処理） |
| TiDB ストレージ | 数 MB（無料枠の1%以下） |
| エンベディング | Ollama bge-large, 1024次元, ローカル実行 |
| 要約 LLM | Ollama qwen3:14b, ローカル実行 |

全部ローカル + TiDB Cloud Serverless の無料枠で動いてる。外部 API キー不要。

## TiDB がこの用途に向いてる理由まとめ

| 要件 | TiDB の対応 | ベクトル専用 DB だと |
|---|---|---|
| ベクトル検索 | `VEC_COSINE_DISTANCE` + HNSW | ✅ 得意 |
| トランザクション | MySQL 互換の ACID | ❌ 弱いか無い |
| 生成カラム + UNIQUE 制約 | `content_hash AS SHA2(...) STORED` | ❌ できない |
| 外部キー | FK で projects → observations を参照整合 | ❌ できない |
| HTAP | 書き込みと検索が干渉しない | N/A（検索専用） |
| MySQL エコシステム | `mysql2` でそのまま繋がる | 独自クライアント |
| 無料枠 | 25 GiB + 250M RU | サービスによる |

*ベクトル検索「だけ」なら専用 DB でいい。でも「ベクトル + リレーション + トランザクション」が全部要るなら TiDB が楽。*

## 次にやりたいこと

- dream-agent のスケジュール実行（launchd で毎晩走らせる）
- `memory_search` の text fallback（ベクトル検索 + FULLTEXT のハイブリッド）
- TiDB の TTL（Time-to-Live）で古い observations を自動アーカイブ
- 他の AI ツール（Cursor, Windsurf）からの MCP 接続

…TiDB の TTL 機能で `expires_at` カラムを使えば、consolidated_memories に有効期限を付けて古い記憶を自動で消せる。RDB の機能がそのまま使えるのは地味に便利。
