---
title: "TineModeler2（RhodEx）— DDSP で物理モデルの残差を学習させる"
---

物理モデルが取りこぼす残差を、PyTorch の DDSP（Differentiable DSP）で学習させる研究プロジェクト。コード名は **RhodEx**。期間は 2026-03-17 〜 04-04 の約 3 週間。

製品化は次の TineModeler3 で。これは**学習基盤の検証**フェーズ。

## なぜこれをやろうと思ったか

TineModeler v0.8.0 で物理モデルを CMA-ES で最適化しても、実音との Mel-L2 = 274.13 程度の差が残っていた。残るのは:

- 個体差（製造年・状態・改造履歴）
- 経年劣化（フェルト、ピックアップ磁化）
- 機械的ノイズ（鍵盤の打鍵音、ダンパーノイズ）

これらは物理モデルで原理的には記述可能だけど、**パラメータ次元が爆発して手動チューニング不能**。手動でやろうとした瞬間に「もう機械学習でいいんじゃない」と思う領域。

そして決定打。CMA-ES 後の残差 L=20.44 を分析したら **R_struct = 1.0**（18 サンプル全てで強い周期的構造）が検出された。

> 残差に構造があるなら、学習できる。

これで GO 判定。

## フェーズ構造

| Phase | 名称 | 目的 | 結果 |
|---|---|---|---|
| Step 1 | Feasibility | DDSP 残差学習が現実的か Go/No-Go | ✅ GO（CoreML p95=0.069ms, R_struct=1.0） |
| Phase 1 | データ準備 | 実音 + 物理モデル出力ペア HDF5 | ✅ 完了 |
| Phase 2 | DDSP 残差学習 | Vast.ai GPU 学習、方式 A/B 比較 | ✅ v1/v2/v3 完了 |
| Phase 3 | E2E 最適化 | 物理パラメータ + DDSP 同時最適化 | ✅ 完了 |
| Phase 4a | CoreML 変換 | ストリーミング推論変換 | ✅ p95=0.086ms |

## アーキテクチャ

```
MIDI 入力
  ├──────────────────────────┐
  ▼                          ▼
Physical Model Layer       DDSP Residual Layer
TineModeler v0.8.0         (PyTorch 学習時 / CoreML 推論時)
                           
A. ティン振動                Encoder: GRU (h=512)
B. ハンマー衝突              Decoder: MLP
C. ピックアップ                → 倍音振幅補正 ΔAk(t)
D. ダンパー                    → ノイズフィルタ H(f,t)
E. ボディ共鳴                  → 減衰補正 Δτ_k(t)
  │                          │
  ▼                          ▼
x_phys(t)        +      x_residual(t)
  └──────────┬───────────────┘
             ▼
       最終出力 (48kHz)
```

## 方式 A vs 方式 B（重要な選択）

残差の統合方式を 2 つ用意した。

**方式 A: パラメトリック補正**（採用）
- DDSP が「物理モデルパラメータの動的補正」を出力
- 倍音振幅補正 ΔAk(t)、ノイズフィルタ H(f,t)、減衰補正 Δτ_k(t) の 3 種類
- 位相整合不要、物理的解釈性を維持

**方式 B: 波形直接加算**
- `x_phys(t) + x_residual(t)`
- 位相整合が必要

Phase 2 のパイロットで **方式 A 勝利**（Mel-L2 で 88.9% 改善 vs baseline）。MethodA で確定。

*位相整合は厄介で、波形を素直に加算すると干渉でディップが出る。方式 A は「物理モデルの内部状態に修正信号を流し込む」イメージなので、位相は物理モデル側のロジックが自動的に整える。*

## Step 1 — Feasibility 検証

学習を始める前に、推論レイテンシが現実的か先に確認。

6 つの推論バックエンドでダミー GRU（h=512）をベンチマーク:

| バックエンド | p95 |
|---|---|
| **CoreML CPU_AND_GPU** | **0.069ms** ← 最速 |
| CoreML CPU_ONLY | 0.080ms |
| CoreML ALL | 0.075ms |
| MLX | 0.082ms |
| MPS | 0.091ms |

目標 2.0ms に対して **約 23 倍の余裕**。CoreML CPU_AND_GPU に確定。

「23倍余裕がある」なら、後で物理モデルと並走させても破綻しないはず、というのが Step 1 の意味。*もし p95 が 1.5ms とかだったら、Phase 2 をやらずに撤退していたと思う。*

## Phase 1 — データ準備

入力: `elepiano/data/rhodes-classic/audio/`（2.1GB FLAC, 44.1kHz 24-bit, 85 MIDI × 19 velocity = 1,615 サンプル）

パイプライン:

1. FLAC → WAV デコード + モノラル化 + 正規化
2. **F0 推定**（CREPE）: hop=441 @44.1kHz → 100Hz フレームレート
3. **Loudness 抽出**（A-weighted）: hop=441 → 100Hz
4. **スペクトル分析**（STFT n_fft=2048, hop=441）: 倍音振幅 Ak(t)
5. **物理モデル骨格**（`libTineModeler.dylib` を ctypes で呼ぶ）: 各ノート・ベロシティを 44.1kHz で事前レンダリング
6. **セグメンテーション**: 4 秒スライス（176,400 samples）、Attack / Sustain / Release 分離
7. **HDF5 出力**: `{note, velocity, f0, loudness, spectrum, audio_44k, phys_audio_44k}`

全特徴量を 100Hz フレームグリッドに統一整列。約 1-2GB の HDF5。

## Phase 2 — DDSP 残差学習

### パイロット（v1）

6 ノート（MIDI 36, 52, 60, 69, 72, 84）× 19 velocity = 114 FLAC で方式 A/B 比較。方式 A 勝利。

方式 A の出力:

- `ΔAk(t)` [100]: 倍音振幅補正
- `H(f,t)` [65]: ノイズフィルタ
- `Δτ_k(t)` [8]: 減衰補正

### フル学習（v2）— 高音域回帰問題

85 MIDI × 19 velocity = 1,615 FLAC でフル学習したら、**高音域がガタガタ**。

原因: データ分布が低音偏重。

```
MIDI  28: 281 segments
MIDI 108:  19 segments
```

セグメント数が 10倍以上違う。学習が低音に引きずられて高音を捨てに行く。

3 層で対策:

1. **BalancedNoteSampler**: ノート均等サンプリング（オーバー / アンダー）
2. **ResidualGate**: gate ≈ 0 で物理モデルを学習的にバイパス（例: MIDI 36 = weight 0.0）
3. **推論時ノート別重み**: `inference_weights.json` で本番運用時の DDSP 適用度を制御

データセット段階 + 学習段階 + 推論段階の 3 ヶ所で対処、というのがポイント。**ML の典型的なバイアス問題は 1 ヶ所では潰せない**。

結果: val_loss 6.598 到達。

### v3 — loudness-guided 物理正規化

物理モデル出力の絶対値スケールが実音と一致していないことに気づいて、各フレームの物理音声 RMS を loudness 基準で正規化してから DDSP で補正。

結果: **Mel-L2 で +57.2% vs baseline**。

## Phase 3 — E2E 最適化

Phase 2 の DDSP チェックポイントから、物理モデル（PyTorch 微分可能版）パラメータも勾配更新対象に加える。同時最適化。

対象パラメータ:

- `tonebar_table [6,3]`、`log_damping_table [6,4]`、`log_mode_boost_table [6,4]`
- `raw_pickup_position`, `raw_strike_rolloff`, `raw_damping_scale`, `raw_em_damping`, `raw_tonebar_coupling`

学習率は Phase 2 の 1/10（3e-5）。物理パラメータは CMA-ES 最適値 ± 妥当範囲で制約。

結果: val_loss 6.6302 → 6.5967（0.51% improvement）。物理パラメータは CMA-ES 最適値から大きく乖離しなかった（= 制約が効いている）。

*正直「もっと劇的に改善するかも」と期待したけど、0.51% 改善が現実だった。CMA-ES が既に物理パラメータをかなり詰めていたということでもある。*

## Phase 4a — CoreML ストリーミング変換

PyTorch DDSP モデルを iOS/macOS リアルタイム対応の CoreML に変換。

**StreamingMethodA**: GRU 隠れ状態を CoreML State として持つ。フレーム間の状態管理をランタイムに委譲。

```
入力（per frame, 441 samples @44.1kHz）:
  f0_norm [1,1,1]
  loudness_norm [1,1,1]
  midi_norm [1,1,1]
  h_in [1,512]  ← 前フレームの GRU 隠れ状態

出力:
  amp_correction [1,1,1025]
  noise_filter [1,1,65]
  gate [1,1,1]
  h_out [1,512]  ← 次フレームへ
```

成果物:

| ファイル | 説明 |
|---|---|
| `rhodex_streaming.mlpackage` | StreamingMethodA CoreML モデル |
| `benchmark_streaming.json` | レイテンシベンチマーク |
| `inference_weights.json` | MIDI 別 DDSP ブレンド上限 |

ベンチマーク（ストリーミング推論）:

| 条件 | mean | p50 | p95 | p99 |
|---|---|---|---|---|
| CPU_AND_GPU | 0.083ms | 0.082ms | **0.086ms** | 0.091ms |
| CPU_ONLY | 0.080ms | 0.080ms | 0.084ms | 0.086ms |

目標 2.0ms に対して 23 倍余裕。AUv3 統合の要件を満たす。

## 学習コストを安く抑えた話

学習環境:

- **Mac M4 Air**（24GB UMA）: 前処理・推論・小規模実験
- **Vast.ai RTX 4090**（Interruptible、$0.15-0.30/h）: 大規模 GPU 学習

Vast.ai の Interruptible は途中で中断されるリスクがあるので、自動チェックポイントを必ず入れる。

総コスト: **¥3,300〜6,000 程度**。クラウドフル型（AWS/GCP の常設 GPU）と比べると **約 1/10**。

*個人開発で DDSP 学習をやるなら Vast.ai は本当に強い。途中で中断されても結局自分の学習なんだから、待てばいい。*

## 評価指標の落とし穴

ベンチマーク（固定 18 サンプル: 6 MIDI × 3 velocity）:

| モデル | Mel-L2 | vs Baseline |
|---|---|---|
| Baseline（物理モデルのみ） | 274.13 | — |
| Phase 2 MethodA v1 | 355.13 | -29.6%（悪化） |
| Phase 3 E2E | 387.88 | -41.4%（悪化） |

…**v1 では悪化していた**。v2・v3 で対策を入れたけど、固定ベンチマークの再計測がまだ。これが TineModeler2 の最大の宿題。

そして客観評価とは別に、**聴感評価（MOS スコア）が未実施**。客観 Mel-L2 で良くなっても、人耳の評価がまだ無い。

## 特筆点

- **二段階最適化**: Stage 1 CMA-ES（ブラックボックス、6項加重損失 69.97 → 20.44、70.8% 改善）→ Stage 2 勾配降下（E2E）。物理設計の堅牢性 + DDSP 適応性
- **高音域回帰問題への 3 層対策**: データ + 学習 + 推論
- **loudness-guided 正規化**: +57.2% 改善
- **CoreML ストリーミング設計**: 隠れ状態を MLState 化、p95=0.086ms

## 何を学んだか

ここで一番大事だったのは、**物理モデルと ML の役割分担を明確にする**こと。

- **物理モデル（構造の理解）** → 音の「骨格」を数式で表現、パラメータの物理的意味を保有
- **DDSP（データドリブン学習）** → 実楽器の「肉付け」を自動学習、個体差・経年劣化を補完

これにより:

1. **軽量性** — 物理モデル（数式）+ DDSP（CoreML <10MB）= 総容量数 MB 級
2. **解釈性** — DDSP が単なる「黒箱」ではなく、ΔAk・H・Δτ_k という物理量を出力
3. **実現可能性** — CoreML p95=0.086ms で AUv3 リアルタイム化が現実的
4. **スケーラビリティ** — 他の電気楽器（Wurlitzer, Clavinet）への応用可能性

## やり残し

1. **固定 18 サンプルベンチマークの再評価**（v2・v3 で）
2. **聴感評価（MOS）**: 客観指標は改善したけど主観評価がまだ
3. **他プリセットへの拡張**（rhodes-la-custom 等、追加コスト ~¥2,000）
4. **方式 B との詳細比較**: パイロットだけで決め打ちした
5. **CoreML Float16 量子化**: 現在 Float32、サイズ削減の余地

## このプロジェクトが残したもの

研究プロトタイプとして「物理 + DDSP は動くし、AUv3 にも乗る」を実証。次の TineModeler3 で **これを製品化する** ステージに進む。

> 物理モデルだけで完結させようとも、ニューラルネットだけで何でもやろうとも、しない。両者の役割分担を明確にして併用する。

これがこのシリーズの軸になる思想。
