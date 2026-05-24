---
title: "自作MCPサーバーに記憶を食わせたら、3プロジェクト横断でバグが11件見つかった"
emoji: "🧠"
type: "tech"
topics: ["mcp", "claude", "codex", "gemini", "codereview"]
published: false
---

## AIに「前の会話覚えてる？」って聞いたこと、ある？

Claude Code を毎日使って開発してるんだけど、セッションが変わるたびに「このプロジェクトの構成は…」って説明し直すのが地味にだるい。claude-mem というプラグインで observation（作業記録）は溜まるんだけど、プロジェクトを横断して「M2DX-Core で直したあのバグ、MIDI2Kit 側にも影響してない？」みたいな検索ができない。

で、作ったのが **memdream**。MCP サーバーとして Claude Code に接続して、TiDB Cloud のベクトル検索で「意味的に近い記憶」を引っ張ってくる仕組み。エコシステム単位で記憶を共有できるのがポイント。

https://github.com/hakaru/memdream

今回やったのは、この memdream 自身と、自分が開発してる3プロジェクト（M2DX、M2DX-Core、MIDI2Kit）を **Codex と Gemini に並列レビューさせて、結果を全部 memdream に食わせる**、という実験。

結論から。

- **レビュー結果**: 3プロジェクト × 2エージェント = 6回のフルレビュー
- **発見したバグ**: CRITICAL 4件、HIGH 12件以上
- **その場で修正**: 12件（memdream 6件 + M2DX 4件 + MIDI2Kit 3件、テスト全パス）
- **memdream に蓄積**: 110 observations（過去履歴インポート含む）

…1セッションでここまで回ったのは正直びっくりした。

## memdream って何

pnpm モノレポで、中身はシンプル。

```
packages/
  mcp-server/    ← 9ツールの MCP サーバー（stdio）
  dream-agent/   ← 統合処理エージェント（Phase 2、未実装）
```

MCP ツールが9個:

- `memory_observe` — 観測を記録（Ollama bge-large で 1024次元ベクトル生成）
- `memory_search` — セマンティック検索（TiDB の `VEC_COSINE_DISTANCE`）
- `memory_recall` — プロジェクトの統合記憶を取得
- `memory_session_start` — セッション開始時のコンテキスト注入
- あと5個（`stats`, `timeline`, `correct`, `list_projects`, `dream_status`）

ストレージは **TiDB Cloud Serverless**。VECTOR(1024) カラムに HNSW インデックスを張って、コサイン距離でベクトル検索する。エンベディングは **Ollama bge-large**（ローカル、API キー不要）。

```
AI ツール → MCP tools → memdream → TiDB Cloud
                ↓
          Ollama bge-large（ローカル）
```

## Codex × Gemini 並列レビューの仕組み

やったことはシンプルで、Codex と Gemini を **バックグラウンドで同時に走らせる**。

```
Claude Code（親）
  ├── Agent(codex:rescue) → Codex にフルレビュー依頼
  └── Bash(gemini -p ...) → Gemini CLI にソース全文パイプ
```

Codex は Agent tool で `subagent_type: "codex:codex-rescue"` を指定。Gemini は CLI の `-p` フラグで非対話実行。どちらも `run_in_background: true` で並列。

*ここで最初にハマったのが Gemini CLI の構文。*

```bash
# ❌ これだとエラー（-p の後に引数がない）
cat source.ts | gemini -p

# ✅ こう
cat source.ts | gemini -p "レビューしてください" -o text
```

`-p` は「非対話モードで実行する」フラグだけど、プロンプト文字列を直接引数に取る。stdin だけじゃ動かない。

## memdream 自身のレビューで見つかったもの

### Gemini の評価

> 「非常にクリーンで完成度の高いコード。TiDB Vector の実戦的な投入例として優れている」

…嬉しいけど、Codex のほうが厳しかった。

### Codex が見つけた指摘（48件）

11カテゴリ、ファイルパスと行番号付き。特に刺さったのがこのあたり。

**レビュー中に見つけた実バグ:**

```typescript
// memory-search.ts — LIMIT ? が TiDB の prepared statement で動かない
ORDER BY distance ASC LIMIT ?
// → "Incorrect arguments to LIMIT" エラー
```

`pool.execute()` は MySQL の prepared statement を使うんだけど、TiDB（MySQL 互換）で `LIMIT ?` にパラメータを渡すと「Incorrect arguments to LIMIT」で落ちる。既知の問題らしい。`limit` は既にバリデーション済み（1-50の整数）なので直接埋め込みに修正。

```typescript
// 修正後
ORDER BY distance ASC LIMIT ${limit}
```

*レビューを依頼したら、レビュー対象のツール自体にバグがあって検索が動かない、という状況。*

### 修正した5件

1. `dream-agent` の空 src が `pnpm build` を壊す → stub 作成
2. `config.test.ts` が OpenAI/1536 前提 → Ollama/1024 に修正
3. `memory_observe` の INSERT に `embedding_model` 未保存 → カラム追加
4. `memory_recall` でエコシステムスコープの記憶が取れない → OR 条件追加
5. `setup-db.ts` が `002_vector_indices.sql` を適用しない → 追加

全部ビルド + テスト通過。

## M2DX のレビューで見つかった CRITICAL

M2DX は MIDI 2.0 FM シンセサイザー（DX7互換）。Swift 6 で書いてて、オーディオレンダースレッドは「アロケートしない、ブロックしない」が絶対ルール。

…なんだけど。

### FXParamBox が `os_unfair_lock` を使ってた

```swift
// FXParamBox.swift — レンダーパスから呼ばれる
func snapshot() -> FXSnapshot {
    os_unfair_lock_lock(&lock)   // ← これ
    defer { os_unfair_lock_unlock(&lock) }
    return current
}
```

Gemini も Codex も同じ場所を指摘。`os_unfair_lock` は Darwin で最軽量のロックだけど、厳密には Priority Inversion のリスクがある。M2DX-Core の `SnapshotRing`（Atomic ベースの triple buffer）と同じパターンで lock-free 化した。

### Downsampler の短バッファ OOB クラッシュ

```swift
// Downsampler.swift L179 — tailStart が負になる
let tailStart = oversampledCount - tailSize
// frameCount * 2 < taps - 1 のとき → 負 → srcL[tailStart + i] で OOB
```

これは **Codex だけが見つけた実バグ**。Gemini は周波数計算の精度問題を指摘してたけど、クラッシュするバグは見逃してた。

### 修正した4件

1. FXParamBox → Atomic triple buffer（レンダー wait-free）
2. midiRing 容量 256→1024 + Note Off リトライ（stuck note 対策）
3. Downsampler 短バッファガード追加
4. oversampling >1024 frame のチャンク分割レンダリング

M2DX-Core 108テスト全パス + M2DX アプリ `xcodebuild BUILD SUCCEEDED`。

## MIDI2Kit のレビューで CRITICAL 3件

MIDI2Kit は Swift 6 の MIDI 2.0 ライブラリ。4モジュール構成（Core/Transport/CI/PE）。

Gemini の評価:

> 「Swift 6 時代における MIDI 2.0 開発のスタンダードになり得る」

…からの Codex が CRITICAL を3つ出してきた。

| # | 問題 | 場所 |
|---|---|---|
| #45 | `PEManager.deinit` が continuation を resume せず破棄 → 永久ハング | PEManager.swift:240 |
| #46 | `CoreMIDITransport` の mutable state がロックなし → データレース | CoreMIDITransport.swift:322 |
| #47 | SysEx7 受信バッファが無制限に増加 → メモリ枯渇 | CoreMIDITransport.swift:262 |

3件とも修正して **705テスト/100スイート全パス**。

## Codex vs Gemini、どっちが強い？

6回のレビューを通して見えたパターン。

| 観点 | Gemini | Codex |
|---|---|---|
| 第一印象 | 褒めてから指摘する | いきなり CRITICAL から始める |
| 得意領域 | 設計評価、アーキテクチャ俯瞰 | メモリ安全性、スレッド安全性、バッファ境界 |
| 見逃し | Downsampler OOB（実クラッシュバグ） | — |
| 独自発見 | MIDI 2.0 ビットレプリケーション仕様 | PEManager continuation 漏れ、CIManager 未追跡 Task |
| ファイル精度 | 行番号がたまにずれる | 行番号が正確 |

*どっちか一方だと見落とす。両方走らせて突き合わせるのが一番信頼できる。*

## claude-mem → memdream 一括インポート

過去セッションの observation を memdream に移行するスクリプトも書いた。

```javascript
// import-m2dx.mjs — 各 observation を Ollama でベクトル化して TiDB に INSERT
for (const o of obs) {
  const res = await fetch("http://localhost:11434/api/embed", {
    method: "POST",
    body: JSON.stringify({ model: "bge-large", input: o.title + "\n" + o.content }),
  });
  const vector = (await res.json()).embeddings[0];
  await conn.execute(
    `INSERT IGNORE INTO observations (..., embedding, ...) VALUES (?, ?, ?, ?, ?, ?, ?)`,
    [projectId, o.type, o.title, o.content, o.source, "[" + vector.join(",") + "]", o.created]
  );
}
```

memdream 49件 + M2DX 15件 + M2DX-Core 30件 = **94件**を一括投入。レビュー結果と合わせて **110 observations** が TiDB に入った。

## 数字で振り返る

| 指標 | 値 |
|---|---|
| フルレビュー実施 | 6回（3プロジェクト × Codex + Gemini） |
| 発見した指摘 | CRITICAL 4 + HIGH 12 + MEDIUM/LOW 多数 |
| 修正完了 | 12件（全テスト通過） |
| GitHub issue 起票 | 7件 |
| memdream observations | 110件 |
| 使ったモデル | Claude Opus 4.7 + Codex (GPT) + Gemini 2.5 |

1セッション、数時間。3プロジェクトのフルレビュー → バグ修正 → issue 起票 → 記憶蓄積まで回った。

## 次にやること

- `dream-agent`（Phase 2）— observations を統合して consolidated_memories を生成する夜間バッチ
- `memory_search` のセマンティック検索が動くようになったので、次回セッションから「前にこのバグ見たことある？」が使える
- MIDI2Kit #48（ビットレプリケーション）の対応

…Codex と Gemini を討論させるのも面白そう。「この設計どう思う？」を両方に投げて、意見が割れたところだけ人間が判断する、みたいな。
