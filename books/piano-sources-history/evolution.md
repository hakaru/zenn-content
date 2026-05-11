---
title: "5世代を貫いた技術選択の変遷"
---

ここまで 5 つのプロジェクトを順に見てきた。横断で見ると、何が継承され、何が変わったか。

## 継承された資産

```
elepiano (C++20)
  ├─ SpscQueue       ──┐
  ├─ biquad           │
  ├─ delay_line       │
  ├─ flac_decoder     │
  ├─ Voice / SampleDB │
  ├─ FxChain         ─┤
  └─ CC マッピング体系 │
                       ▼
iRhodes (C++17 + Swift)
  ├─ ↑全部継承
  ├─ NAM → 手書き WaveNet に置換
  ├─ IR Convolver 新規（Accelerate）
  ├─ HybridVoice 新規（タイムオーバーラップ）
  ├─ PhysicalModel/ 6 クラス     ─┐
  └─ paramSerialQueue 新規        │
                                   ▼
TineModeler (C++17 + Swift + Python)
  ├─ PhysicalModel/ をリポジトリ独立化
  ├─ Symplectic Euler
  ├─ Hunt-Crossley ハンマー
  ├─ Faraday 磁束差分ピックアップ
  ├─ tonebar 連成（Gabrielli 2020 データ融合）
  ├─ CMA-ES 最適化基盤
  └─ DDSP 逆推定パイプライン
                                   ▼
TineModeler2 (Python / PyTorch)
  ├─ libTineModeler.dylib 呼出
  ├─ DDSP 残差学習（GRU h=512）
  ├─ MethodA 採用
  ├─ BalancedNoteSampler + ResidualGate
  ├─ loudness-guided 正規化
  ├─ E2E 最適化
  └─ CoreML streaming 変換（p95=0.086ms）
                                   ▼
TineModeler3 (C++17 + Swift 5.10 + CoreML)
  ├─ 物理モデル再実装（Swift C++ Interop）
  ├─ CoreML streaming 統合
  ├─ Voice 単位 MLState
  ├─ atomic SPSCQueue + VoiceResultFIFO
  └─ AUv3 プラグイン
```

`SpscQueue` は 5 世代を通じて生き残った。形は徐々に厳密化（簡易版 → memory_order 明示 → swift-atomics）したけど、原則は同じ。

## サンプル容量の変遷

| 世代 | 形式 | 容量 | 戦略 |
|---|---|---|---|
| elepiano | FLAC バンドル | ~2.7GB（フル） | Pi に常駐 |
| iRhodes（フル） | FLAC | 2.7GB | iOS では非現実的 |
| iRhodes（ハイブリッド） | FLAC + 物理 | **186MB** (-99.3%) | アタックのみサンプル |
| TineModeler | サンプル不使用 | **0MB** | 純粋な物理モデル |
| TineModeler2 | （学習用に参照） | — | 学習データセット |
| TineModeler3 | サンプル不使用 | **数 MB** | 物理 + DDSP |

**最終形ではサンプルが配布物から完全に外れている**。法務的にも実用的にも、数 MB の AUv3 を売る方が筋がいい。

## サンプル ↔ 物理 ↔ DDSP の比重シフト

```
            サンプル     物理モデル     DDSP / ML
elepiano    ████████      ██            ・
iRhodes     ███████       ████          ███   (NAM/IR)
TineModeler ・            ████████      ███   (逆推定のみ)
TineModeler2 (参照)        (参照)         ████████ (残差学習)
TineModeler3 ・            ██████        ██████ (CoreML streaming)
```

elepiano は **サンプル主、オルガンだけ物理**。iRhodes で **サンプル + 物理ハイブリッド**。TineModeler でいったん **物理モデル単独に振り切って**から、TineModeler2 で **DDSP 残差** という第3の柱を発見、TineModeler3 で **物理 + DDSP の二本立て** に着地。

## オーディオエンジンの変遷

| 世代 | 選択 | 理由 |
|---|---|---|
| elepiano | ALSA + SCHED_FIFO | Pi 5 上の低レイテンシ |
| iRhodes | AVAudioEngine + AVAudioSourceNode | iOS 標準 |
| TineModeler | AVAudioEngine | macOS / iOS 共通 |
| TineModeler2 | （ホスト不要） | 学習基盤 |
| TineModeler3 | AUv3 + AVAudioSourceNode | DAW 統合 |

## ML 推論の変遷

| 世代 | 推論技術 | 用途 |
|---|---|---|
| elepiano | NeuralAmpModelerCore (Eigen) | NAM アンプモデル |
| iRhodes | 手書き WaveNet | NAM の iOS 移植 |
| TineModeler | librosa（解析） | DDSP 逆推定 |
| TineModeler2 | PyTorch + CoreML | 残差学習 + 変換 |
| TineModeler3 | CoreML streaming + MLState | リアルタイム推論 |

ML 用途が「アンプモデル → 物理パラメータ逆推定 → 残差学習 → リアルタイム補正」と変遷している。**ML の役割が「音色加工の一部品」から「物理モデルと対等な合成器」に格上げ**された。

## 「ハイブリッド」の意味の進化

3 つとも「ハイブリッド」と呼べるけど、軸が違う。

- **elepiano**: 音色種別でハイブリッド（ピアノ=サンプル / オルガン=物理）
- **iRhodes**: 時間方向でハイブリッド（アタック=サンプル / サスティン=物理）
- **TineModeler3**: 残差方向でハイブリッド（物理音 + DDSP 補正）

最後の残差方向ハイブリッドが最も洗練されている。容量も小さく、解釈性もある。

## 物理モデルの位置づけの進化

- elepiano: オルガン専用、setBfree 外部ライブラリ
- iRhodes: サンプルの「補完役」
- TineModeler: 単独主役、限界探索
- TineModeler2: DDSP との対比対象
- TineModeler3: DDSP との協調役（再び主役だが補完あり）

「協調役（再び主役だが補完あり）」がゴール地点。**物理モデルが主役だけど、最後の 5-10% は DDSP が補う**、という協業モデル。

## 決定的な技術判断

| 世代 | 判断 | 結果 |
|---|---|---|
| elepiano | Keyscape SpCA 逆エンジニアリング | サンプル品質を担保、商用ライブラリ再利用 |
| elepiano | PREEMPT_RT カーネル | Pi 5 で 6μs jitter |
| iRhodes | サンプル単独配布を放棄、ハイブリッド方式 | 2.7GB → 186MB |
| iRhodes | NAM を Eigen から手書きに | iOS 配布の自由度 |
| TineModeler | Symplectic Euler | 長時間サステインの数値安定性 |
| TineModeler | CMA-ES + 6項加重損失 | 27.7% の自動改善 |
| TineModeler2 | 残差を「学習対象」と見做す | DDSP 残差補正パラダイム |
| TineModeler2 | MethodA 採用 | 位相整合不要、物理的解釈性維持 |
| TineModeler3 | Swift 5.10 C++ Interop で ObjC++ 廃止 | コード簡素化 |
| TineModeler3 | Voice 単位 MLState | 音色クロストーク回避 |

## レイテンシの変遷

| 世代 | 環境 | 実測 |
|---|---|---|
| elepiano | Pi 5 + PREEMPT_RT | period_size=32 → ~1.3ms 理論 |
| iRhodes | iPhone | 256 frame @44.1kHz → 5.8ms |
| TineModeler | macOS | （未測定、低レイテンシ） |
| TineModeler2 | CoreML benchmark | p95=0.086ms（推論のみ） |
| TineModeler3 | AUv3 / M1 Pro | CPU ~20% / 16 voice |

## ドキュメンテーション文化の進化

elepiano の `docs/ClaudeWorklog202603*.md` という日次ワークログが 10 ファイル残った。これは Claude Code の対話履歴を整理したもの。後で経緯を辿るのに非常に役立った。

iRhodes 以降は `README.md` がメインドキュメント、TineModeler では `CHANGELOG.md` が詳細変更履歴を担う。

TineModeler3 では `AGENTS.md`（memory context）、`ASSESSMENT-2026-04-26.md`（評価）、`ROADMAP.md`（計画）と **役割別のドキュメント分離** が確立された。

*この本も結局、その既存ドキュメントから再構築している。*ドキュメントを開発のフローの一部として書いておくと、後で振り返るのが楽になる。

## 5 世代を貫く設計原則

### 原則 1: RT スレッドからは何も奪わない

- malloc しない
- lock しない
- 例外を投げない
- 状態を直接共有しない（必ずキュー越し）

形は変われど原則は不変。`SpscQueue` → `paramSerialQueue` → `VoiceResultFIFO`。

### 原則 2: 観測可能性を最初から仕込む

- elepiano: `StatusReporter` が UNIX socket で JSON 出力、BLE で iOS に届く
- iRhodes: Gig Safety（CPU / underrun 監視）
- TineModeler: CMA-ES の損失ログ
- TineModeler2: Weights & Biases
- TineModeler3: XCTest 67件 + TSAN

「動いているか」を常に観測できる仕掛けが常に存在している。

### 原則 3: プラットフォーム標準のヘッドルームを優先

依存ライブラリを徐々に減らして標準に寄せていった:

- Eigen → 手書き Conv1D
- LV2 → 内蔵 + Accelerate
- ObjC++ → Swift C++ Interop
- 独自 GATT → MIDIKit 標準 BLE-MIDI

**配布の自由度** と **長期保守性** を確保するため。

### 原則 4: 「物理 vs ML」を排他にしない

iRhodes 以降、物理モデルと ML は常に協調関係。

- iRhodes: ハイブリッドエンジン（時間方向の協調）
- TineModeler: CMA-ES で物理パラメータを ML 的に最適化
- TineModeler2: 物理ベースライン + DDSP 残差補正
- TineModeler3: 物理 × DDSP のリアルタイムブレンド

**ニューラルネットだけで何でもやろうとも、物理モデルだけで完結させようとも、しなかった**。両者の役割分担を明確にして併用する。

これが結局このシリーズで一番強い学びだった、と振り返って思う。
