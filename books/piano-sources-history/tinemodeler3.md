---
title: "TineModeler3（RhoDex）— AUv3 製品版"
---

シリーズの到達点。AUv3 楽器プラグイン + Mac/iOS スタンドアロンアプリ。プロダクト名は **RhoDex**（製造者 "Rhox"）。

開発は 2026-04-26 から、現在進行中。コード規模は C++17 物理モデル 6,827行 + Swift 5.10 が 2,565行 + XCTest 67件。

## やったこと

- **物理モデル（C++17）** と **CoreML DDSP（Swift）** をリアルタイムブレンド合成する AUv3 プラグイン
- 16 ボイスポリフォニック
- UI で `ddspMix` パラメータでブレンド比を調整可能
- Mac (macOS 14+) / iOS (17+) 両対応

TineModeler2 の学習基盤（rhodex_streaming.mlpackage）と物理モデル（TineModeler v0.8.0 の知見）を **全面実装し直して**統合した版。

## アーキテクチャ

```
MIDI 入力 (16 ボイス)
  ↓
RhoDexEngine
  ├─ Voice Allocator（ボイススチール戦略）
  └─ 16 × RhoDexVoice
       ├─ PhysicsVoiceAdapter（C++17 物理モデル）
       │   ├─ Hammer: 初速ベース + stiffness
       │   ├─ Tine: 8 モード + 減衰 + damper coupling
       │   ├─ Pickup: 電磁誘導
       │   └─ BodyFilter: Neve EQ + tremolo/chorus/phaser
       │
       └─ DDSP Pipeline（Worker Pool → CoreML）
           ├─ STFT: 512 サンプル hop（Swift vDSP）
           ├─ CoreML runner（16 voice × 2 worker）
           │   ├─ Input: f0_norm, loudness_norm, midi_norm
           │   ├─ State: GRU 隠れ状態（voice 単位）
           │   └─ Output: 倍音補正 + ノイズ
           └─ STFT inverse + dry/wet FIFO

        ↓ Blend (physics:ddsp = 1-ddspMix : ddspMix)
        ↓ ステレオ widening（note ベース L/R panning）

Final Output (L/R 16-bit, 48 kHz)
```

## Swift 5.10 C++ Interop — ブリッジ層が消えた

これがでかい。Swift 5.10 の C++ Interop で、**Objective-C++ ブリッジ層を廃止**できた。

iRhodes や elepiano では `iRhodesEngine.h/.mm` のような Objective-C++ ファイルを書いて、C++ オブジェクトを Swift から扱える形に橋渡ししていた。これは保守性も悪いし、ARC との相性も微妙だった。

Swift 5.10 だと:

```swift
// Package.swift
.target(
    swiftSettings: [
        .interoperabilityMode(.Cxx)
    ]
)

// RhoDexVoice.swift
var physicsState: RhodesVoice_CXX   // C++ opaque handle、lifetime は Swift が管理
```

これで 2,565行の Swift と 6,827行の C++ を直接統合できている。*書き味が全然違う。`@unsafe(...)` も最小限で済む。*

## RT スレッド安全性 — Phase 1-9 の修正

機能実装は v1.0-v2.0 で完了していたけど、2026-04-26 に詳細アセスメント（`ASSESSMENT-2026-04-26.md`）をやったら **9 個の本質的な issue** が指摘された。

その後 Phase 1-9 のリファクタで全部潰した、というのが直近の山。

### HIGH（本質的な安全性）

| # | 問題 | 修正 |
|---|---|---|
| H1 | SPSCQueue atomic 非対応 | `swift-atomics` パッケージで acquire/release |
| H2 | `maximumFramesToRender` 未宣言 | AU 仕様に適合 |
| H3 | DDSP input/hidden storage 単一化 | リング ring-buffer で double-buffer 化 |
| H4 | hiddenState/gate 無同期共有 | FIFO 値渡し |

H4 が一番厄介だった。Worker スレッドが計算した GRU 隠れ状態を、オーディオスレッド（RT）が次のフレームで参照する。**両者で同じメモリを直接共有していた**。たまたま動いてたけど、いつデータレース起こしてもおかしくない。

修正は `VoiceResultFIFO` で値渡し:

```
Worker thread        VoiceResultFIFO        Audio thread (RT)
[隠れ状態を計算]  ──→ [値コピーで push] ──→ [pop して使う]
```

メモリの所有関係が明確になり、無同期共有が消える。

### MEDIUM

| # | 問題 | 修正 |
|---|---|---|
| M1 | `usleep(1000)` ポーリング | DispatchSemaphore イベント駆動 |
| M2 | AU `parameterRamp` 未処理 | per-sample 線形補間 |
| M3 | MLState worker 単位 | voice 単位に正規化 |
| M4 | stereo widening AU 出口でモノ複製 | `renderStereo()` 実装 |
| M5 | STFT 毎回 `[Float]` 新規確保 | `UnsafeMutablePointer` API |

M3 が地味に音質に効いた。

修正前: Worker A が voice の隠れ状態を計算 → 次回その voice が Worker B に割り当てられると、Worker B 内の MLState は別物 → **音色クロストーク**（A4 を弾いた直後に C5 を弾くと C5 の出だしに A4 の余韻が混ざる）

修正後: 隠れ状態を voice の寿命に紐付け、Worker 間で持ち回りせず単純に値渡し。クロストーク消滅。

## DDSP モデル再学習

TineModeler2 で学習した重みを CoreML に変換したら、**iOS 転送時に音劣化**が出た。Float16 量子化の問題と、サンプリングレート整合の問題。

Vast.ai RTX 4090 で再学習。epoch 33 まで回して val_loss 5.96 達成。高域の音色精度が **57.2% 向上**。

成果物は `Sources/DDSPLayer/Resources/rhodex_streaming.mlpackage` と `inference_weights.json`。

## CoreML Streaming の応用

PyTorch GRU の stateful streaming を CoreML に変換する話は TineModeler2 でやった。それを Swift 側で動かす実装が `DDSPSynthesizer` + `CoreMLRunner`。

**MLState migration**: macOS 15+ / iOS 18+ では自動でステート管理してくれる。旧 OS は手動フォールバック。

## ARM NEON + vDSP 最適化

```
コンパイラフラグ: -O3 -ffast-math -ffp-contract=fast -mcpu=apple-m1
```

STFT blend は vDSP `vsmul` で `(1-ddspMix) × phys + ddspMix × ddsp` をベクトル化。

結果: **16 voice × 2 worker でも CPU 使用率 ~20%**（M1 Pro 実測）。十分にヘッドルームある。

## followups リポジトリ — worktree 並列管理

TineModeler3 の修正フェーズは、リポジトリを 2 つに分けて並列管理した。

- `TineModeler3` — メインリポジトリ
- `TineModeler3-followups` — RT 修正用の worktree

さらに 2 つの feature branch:

- `rt-correctness` — H1-H4 本質的修正（15 commits）
- `rt-followups` — M1-M5 補完的修正（4 commits）

両者を main にマージしたチェックポイントが `6afa6cb`, `def5ab3`。

*こういう「リポジトリを並列で動かす」管理は、エージェント駆動開発と相性がいい。Claude Code を 2 つ別ディレクトリで走らせて、独立にレビューさせて、最後にマージ。*

## AGENTS.md で運用方針を記録

`AGENTS.md` に Claude mem 形式で memory context を記録している:

```
# Memory Context — TineModeler3 recent context, 2026-04-26 5:41pm GMT+9

Legend: 🎯session 🔴bugfix 🟣feature 🔄refactor ✅change 🔵discovery ⚖️decision
Format: ID TIME TYPE TITLE

### Apr 26, 2026
700 5:40p 🔵 RhoDex (TineModeler3) — プロジェクト全体構造の確認
```

運用ルール:

- Codex / Gemini 並列レビュー + Claude Code 統合化
- 高コスト検証は並列実行
- 見解が割れた場合は実コード確認で決着

エージェントの意見を独立に得て、対立を物理コードで決着する、というメタ手法。

## TineModeler2 → 3 の段階的改善

| 項目 | TineModeler2 | TineModeler3 |
|---|---|---|
| 言語 | Python（PyTorch） | Swift 5.10 + C++17 |
| 形態 | Python スクリプト | AUv3 + Xcode App |
| 推論エンジン | PyTorch inference | CoreML（streaming） |
| 物理層 | なし（参照のみ） | C++17 TineModel + FX |
| RT 設計 | なし | SPSC + Worker Pool + FIFO |
| ポリフォニー | 単発ノート | 16 voice |
| UI | CLI / Jupyter | SwiftUI + MiniKeyboard |
| テスト | pytest | XCTest（67件） |

主要な進化:

1. **PyTorch → CoreML**: 推論速度 3 倍向上、ディスク容量 1/10
2. **single-shot → streaming**: 隠れ状態持続で自然な音色進化
3. **Python preprocessing → vDSP**: STFT を RT 内で実装
4. **MIDI script → AUv3 plugin**: DAW native 統合

## コード規模

- **C++ 物理モデル**: 6,827 行（多くは `TineModelTables.cpp` の lookup tables）
  - 実装: 2,400+ 行（core models + FX）
  - ヘッダ: 1,400+ 行
- **Swift**: 2,565 行
  - `AUv3Plugin`: 435 行
  - `RhoDexEngine` + `Voice`: 274 + 273 = 547 行
  - `DDSPLayer`（Job types + CoreML + STFT）: 235 + 204 + 134 = 573 行
- **Tests**: 67 件 passing

## ROADMAP 進捗

v1.0-v2.0 はほぼ完走:

- v1.0: Core engine（物理 + DDSP + 16 voice + AUv3）✅
- v1.1: DAW 統合（MIDI CC64 sustain, preset 保存/読込, iOS ビルド）✅
- v1.2: Sound Quality（velocity curve, stereo widening, DDSP 再学習）✅
- v1.3: Performance & Effects（CoreML compute units, Tremolo/Chorus/Phaser, NEON）✅
- v2.0: Advanced Features（MPE, 12-TET / Just Intonation）✅

## 残り

検証・テスト:

- `auval` によるプラグイン検証（macOS + 複数 DAW）
- 全サンプルレート（44.1, 48, 96kHz）動作確認
- 全フレームサイズ（64〜4096）スイープ
- `-ffast-math` 配下での NaN/denormal 継続監視（テスト追加済）
- 64 voice 過負荷シナリオ
- fullState preset ラウンドトリップ

未実装機能:

- メモリプロファイリングダッシュボード
- Impulse Response cabinet simulator
- **App Store 提出準備**（codesign / entitlements / sandbox）

最後のやつが地味に大きい。**機能完成度が高くても出荷できなければ意味がない**。

## 改善案（v2.1〜）

1. **per-voice Worker**: 現在の round-robin → 1:1 固定化、context switch 削減
2. **Temporal coherence**: GRU hiddenState を連続保持、音色進化の滑らかさ
3. **Adaptive DDSP routing**: `ddspMix` を音響エネルギーで自動調整
4. **Cabinet impulse**: 実機録音の反射特性を convolver で

## 現在地

- 機能: 100% 実装完了（ROADMAP v1.0-v2.0）
- テスト: 67 passing（0 failures）→ TSAN 通過
- AU 堅牢性: 本質的 9 issue を Phase 1-9 で全修正

技術的な意味でのプロダクションレディに到達。残るは **`auval` 通過 + DAW 実運用テスト + App Store 提出準備**。

これで 5 世代の旅の終わり。次の 2 章で振り返り。
