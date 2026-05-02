---
title: "ローカルLLMって本当に開発に使える？（２）RAG編 — Swift仕様をベクトルDBに入れたら誤検出が76%減った"
emoji: "🔍"
type: "tech"
topics: ["llm", "ollama", "swift", "rag", "llamaindex"]
published: false
---

https://zenn.dev/hakaru/articles/m2dx-local-llm-audit-zero-true-positives

:::message
**この記事の対象プロジェクト**

- **M2DX** — iOS/macOS 向け MIDI 2.0 対応 DX7 互換 FM シンセサイザーアプリ。[TestFlight 公開ベータ](https://testflight.apple.com/join/BAtGszPw) で試せる
- **M2DX-Core** — M2DX の DX7 互換エンジン部分。Pure Swift、Apache 2.0 で OSS 公開: [github.com/hakaru/M2DX-Core](https://github.com/hakaru/M2DX-Core)
- **MIDI2Kit** — M2DX-Core が依存する Swift 製 MIDI 2.0 ライブラリ。SysEx の受信・バッファ管理・UMP デコードを担う。開発の経緯は[こちらの本](https://zenn.dev/books/midi2kit-development-journey/)にまとめてある
:::

前回記事で、ローカル LLM 10 機種に DX7 エンジン（Pure Swift）のバグ監査をさせたら **36 件の指摘 → 真陽性 0 件** という結果になった（セキュリティ監査 16 件を含めた全実験の累計は 52 件）。

失敗の主原因は **Swift 言語仕様をわかってない**だった:

- `(A, A, A, A, A, A)` tuple を「ヒープ確保された Array」と判定 → "RT 安全性違反" と指摘
- `&+` / `&-` を「未チェックなオーバーフロー」と判定（明示的 wrap 演算子だと知らない）
- `Int32(clamping:)` を「精度を失う truncation」と判定（飽和演算なのに）
- `min(63, x)` のクランプを無視して「shift overflow しうる」と指摘

前回記事で「Swift の知識をどう注入するか」を考えた。3 つの案の B 番:

> **B. RAG で Swift Book / Evolution を引かせる** — 公式仕様書をまるごと検索エンジンとして使い、コードの構文に応じて関連ページを動的に retrieval してプロンプトに差し込む

これを実装して 9 モデルでやってみる。

## パイプライン

```
[1] M2DX-Core の Swift ファイル
      ↓
[2] analyze_code.py — &+ / tuple / clamping: などの構文を正規表現で検出
      ↓ 検出した構文リスト
[3] retrieve.py — bge-large でベクトル検索、上位 3 chunk を取得
      ↓ 取得した Swift 仕様の断片 (~3KB)
[4] prompt builder — retrieved chunks をプロンプト先頭に追加
      ↓ v3-prompt.md (v1 プロンプト + RAG ブロック)
[5] Ollama /api/generate (num_ctx=40960)
      ↓
[6] findings (JSON)
```

### データソース

- **The Swift Programming Language Book** (`swiftlang/swift-book` の markdown ソース) — Swift 6 対応版、全章
- **Swift Evolution proposals** (`swiftlang/swift-evolution` の proposals/) — SE-0104 (overflow operators)、SE-0235 (clamping) など約 400 件

### 埋め込みモデル

`bge-large-en-v1.5`（テキスト埋め込み専用モデル）を Ollama でローカル実行（`ollama pull bge-large`）。外部 API に一切投げない。

### chunk 設計

markdown を H1/H2 境界で分割 → 400 文字を上限に再帰的に段落→行→ハードカット。bge-large の 512 トークン上限に収める。Swift Book + Evolution で約 8,000 chunk（分割ブロック）。

## 結果

v1 と同じ 9 モデル、同じ M2DX-Core コード（DX7Envelope/Operator/Voice/Algorithm）、v3-prompt.md を投入。mixtral はタイムアウト継続だが、フォールトトレラント版スクリプトでループを途切れさせず 9 モデル走行。

| モデル | v1 findings | v3 findings | 変化 |
|---|---|---|---|
| phi4:14b | 1 | **6**† | ↑（markdown 形式に変化） |
| qwen3:14b | 4 | 3 | ↓ -25% |
| gemma3:12b | 8 | 1 | ↓ -88% |
| hermes3:8b | 4 | 4 | →（内容変化） |
| codestral:22b | 10 | **11** | ↑ 微増 |
| gemma4:31b | 3 | 3 | →（Swift 系→ドメイン系） |
| qwen3.6:35b | 0\* | **5** | 出力復活 |
| llama3.3:70b | 5 | 4 | ↓ -20% |
| mixtral:8x22b | 1 | 0\*\* | タイムアウト |
| **合計 (JSON)** | **36** | **31** | **-14%** |

\* v1 / v2 ともに reasoning ループのみで JSON なし  
\*\* フォールトトレラント版でタイムアウト後も継続、結果は FAILED  
† phi4 は markdown 形式で 6 件出力。JSON 抽出パイプラインでは 0 件扱い

## 結果の解釈

### H1 ✅ — Swift セマンティクス系 FP（偽陽性）が約 76% 削減

v1 で観測した 5 系統の誤検出の変化:

| v1 FP パターン | v1 件数 | v3 件数 |
|---|---|---|
| `&+` / `&-` を "overflow 未チェック" と判定 | 3+ | **0 — 完全消滅** |
| tuple を "Array allocation" と判定 | 3+ | 1 (hermes3 のみ) |
| `Int32(clamping:)` を truncation と誤読 | 2+ | 1 (qwen3 のみ) |
| `min(63, ...)` 等の防御コードを無視 | 2+ | 1〜2 |
| 行番号の捏造 | 5+ | 3 |

**`&+` / `&-`** 系は完全に消えた。Swift Book の overflow operators に関する chunk が retrieval で引かれ、モデルが正式な仕様を参照した結果と見られる。

Swift セマンティクス系 FP の推定総数: v1 の約 17 件 → v3 の約 4 件（**-76%**）。仮説「70% 以上削減」は **達成**。

### H2 ✅ — 依存ライブラリ未参照・行番号捏造は残る

- codestral:22b が DX7Voice.swift の L841-L863 を指摘したが、このファイルは **472 行**。存在しない行
- llama3.3:70b が DX7Operator.swift の L120-L130 を指摘したが、このファイルは **85 行**
- RAG はプロンプト内のコードだけを見る。上流ライブラリ (`MIDI2Kit` の `maxBufferSize`)・iOS の sandbox 制約・ファイルシステム仕様はプロンプト外なので参照不可

これは予測通りで、RAG で解決する問題ではない。agent harness (grep/Read を LLM に持たせる) が必要。

### H3 △ — RAG は cheat sheet より広いカバレッジ、ただし総件数抑制は cheat sheet の勝ち

前回記事の A 案（手書きの Swift カンペを先頭に貼る）は総件数を **36→9 件**に圧縮した。B の RAG では **36→31 件**でほぼ変わらない。

なぜ抑制が弱いか: cheat sheet は「この 5 系統を flag するな」という *明示的な禁止*。RAG は「これが Swift の仕様だ」という *知識の追加*。モデルが "仕様に基づいた自信" を持って、かえって多く報告する副作用がある。

一方、`&+` 系の消え方は cheat sheet を凌駕した。手書きのカンペでは qwen3:14b と mixtral がタイムアウトしていたが、RAG ではこれらの挙動が変化している。「明示的に載っている仕様」への対応は RAG の方が確実。

## 注目 finding: Q16/Q24 スケールミスマッチ

gemma4:31b と qwen3.6:35b が独立して、同じコード箇所を critical として報告してきた:

```json
{
  "severity": "critical",
  "title": "Q16/Q24 fixed-point scale mismatch in gain calculation",
  "what": "exp2LookupQ24 に渡す値が DX7Envelope 出力 (Q16) と定数 (Q24) の
           混在で、massive な scale error が発生する"
}
```

コードを精査すると: DX7Envelope の出力は確かに Q16 スケール（`>> 16` で正規化）、`exp2LookupQ24` の定数は Q24 スケール — **両モデルの観察は正確**。ただし実際の音声出力は正常に動作しており、コメントの精度記述のずれ（スケールの明記漏れ）が原因と判断した。本実験は RAG 有効性の検証なので修正は行わない。

v1 でも v2 (cheat sheet) でも一切出なかった種類の指摘が、RAG 注入によって出てきた。2 つの独立したモデルが同じ箇所を指摘したことは、単なるハルシネーションではなく retrieval コンテキストによる一貫した推論の結果と見られる。H3「B は自分が思いつかなかった誤読パターンにも効く」の部分確認として記録に値する。

## Cheat Sheet方式 vs RAG の比較

前回記事の A (cheat sheet) と今回の B (RAG) を並べると:

|  | A (cheat sheet) | B (RAG) |
|---|---|---|
| 実装コスト | 低（プロンプトに数百トークン追加するだけ） | 中（chunk + embed + 検索パイプライン） |
| 総 findings 数 | **9件**（36→9, -75%） | 31件（36→31, -14%） |
| Swift 系 FP 削減 | -76%（前回記事の結論） | -76% |
| `&+` 誤検出の消え方 | 部分的（タイムアウトモデルあり） | **完全消滅** |
| 周辺領域（Q16/Q24 等） | 効かない | 出てくる |
| TP | 0 | 0 |

**費用対効果で見ると A が勝っている**。総件数の抑制は A が圧倒的で、TP は両方 0 で変わらない。

ただし B の優位点は: `&+` の完全消滅と、*自分が事前に書いておかなかった構文*（Q16/Q24 スケール、actor isolation、Sendable など）にも自動的に対応できること。A はカンペに書いた 5 系統しかカバーできないが、B は Swift Book 全体をカバーする。

**A + B の組み合わせ**が総量も精度も最強候補になりそうで、追加コストも小さい。

## 結論

仮説 H1 ✅ Swift セマンティクス系 FP -76% → **達成**  
仮説 H2 ✅ 依存ライブラリ / 行番号の問題は残存 → **確認**  
仮説 H3 △ cheat sheet より広いカバレッジ → **部分確認**（`&+` 系は完全消滅、DX7 ドメインには届かず）  
**TP は依然 0**

RAG は「Swift を知らない」という問題は実用レベルで解決できる。ただし「バグを見つける能力がない」は別軸の問題で、A も B も届かなかった。

次の打ち手は 2 つ:

- **C (LoRA)**: v1 で集まった 52 件の false positive をそのまま訓練データに転用し、モデル本体に Swift 知識を刻む。失敗が再起動の燃料になる再帰的な構造が面白い。`swift-audit-lora` として別記事に続く
- **Agentic harness**: grep/Read を LLM に持たせて依存ライブラリを参照させる。行番号捏造・上流ライブラリ未参照の問題はこちらで構造的に解決する

---

https://github.com/hakaru/M2DX-Core

https://apps.apple.com/jp/app/m2dx/id6753466996
