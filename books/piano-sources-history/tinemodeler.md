---
title: "TineModeler — サンプル無しで Rhodes をどこまで再現できるか"
---

Rhodes Electric Piano の発音体である「Tine」（金属棒、トーンバー）の物理モデル化に特化したプロジェクト。

期間は **2026-03-11 〜 03-16 の 5日間で 156 commits / 8 マイナーバージョン**。1日あたり 30+ commits、ほぼ 12時間ごとにバージョンを切る速度感。さすがに人間一人では無理で、エージェントを並走させての数字。

サンプリングとの併用ではなく、**純粋な物理モデリング単独**。数 GB のサンプルセットを数 MB の物理パラメータセットで置換するのが目標。

## 物理モデルの 4 層構成

Rhodes の発音メカニズムをそのまま離散化する。

### (1) Tine 振動モデル

Euler-Bernoulli 梁の固有振動を 8 モードで展開。各モードの運動方程式:

```
q̈_n + 2ζω_n q̇_n + ω_n² q_n = F_n(t)   （n=0…7）
```

数値積分は **Symplectic Euler 法**。Runge-Kutta より精度は落ちるけど、**エネルギー保存型**なので長時間サステインで暴走しない。周波数シフトは +0.85 セント程度（無視可能）。

非調和性補正は `tonebarCoupling` パラメータ [0,1] で、実測モード周波数比（Gabrielli et al. JASA 2020）を補間する。

- 低音域 f2/f1 ≈ 4.5〜6.0
- 高音域 f2/f1 ≈ 2.5

*Euler-Bernoulli の理論値だと f2/f1 = 6.27 とか出るけど、実機を測ると低音は 5.9〜6.0、高音は 2.5。「弦の根本がトーンバーに連成して挙動が変わる」とかそういう話で、理論と実測のギャップを `tonebarCoupling` で潰す。*

### (2) ハンマーモデル

Hunt-Crossley のヒステリシス付き Hertz モデル:

```
F = K_h(v) · δ^p · (1 + λ·δ̇)   （接触時のみ）
K_h(v) = K_min + (K_max - K_min) · v^γ
```

打鍵速度 v によって剛性が変わる（速く弾くとフェルトが圧縮されて硬くなる）。Shore 硬度ベースで `feltExponent` と `hysteresisCoeff` を自動計算。

### (3) 非線形ピックアップモデル

ここが一番面白いところ。**Faraday 則の厳密な離散化**:

```
Φ(y) = B·N·w / (y0 + offset + y)       （単磁気双極子近似）
V(t) = -(Φ(y_new) - Φ(y_old)) / dt     （磁束差分 ≠ 速度近似）
```

速度近似 `V = -B·N·w·∂y/∂t / (y0+y)²` ではなく、**磁束そのものの差分を取る**。これによって:

> ティネ静止位置オフセット → 非対称ギャップ → 偶数倍音の自然発生

これが Rhodes の「ベル感」の源泉。**偶数倍音は理論的には弦振動だけからは生まれない**（梁の clamped-free 境界条件だと奇数倍音が支配的）。ピックアップの非線形性で発生する、というのが実機の挙動。

RLC フィルタ（2次 IIR, Direct Form II Transposed）でロールオフも入れる:

- L=0.5H, R=180Ω, C=4700pF → f0 ≈ 3.28kHz, Q=175

### (4) EM ダンピング + フェルトダンパー

```
c_n += emDampingCoeff × (φ_n(x_pickup) / φ_0(x_pickup))²
```

**基音選択的減衰**。Lenz の法則で、磁場を通る導体は速度に比例する制動力を受ける。基音が最も強くダンピングされる結果、高次モードが相対的に長く持続する。これが実機の聴感に近い。

スムーズダンパーエンゲージは 4.5ms で `alpha` [0,1] を遷移。クリックノイズが出ないように。

## ボディ共鳴フィルタ

3 段カスケードピーキング EQ（Direct Form II Transposed）:

| 段 | 中心 | Q | ゲイン | 役割 |
|---|---|---|---|---|
| 1 | 200Hz | 2.0 | +3dB | ボディの温かみ |
| 2 | 800Hz | 1.5 | +2dB | 中域プレゼンス |
| 3 | 2500Hz | 3.0 | -2dB | 高中域ディップ |

Dry/Wet = 0.85（v0.8.0 で固定値化）。

## DDSP で逆推定する

`ddsp/` ディレクトリ。実機 Rhodes（ETI Roads MKII）の FLAC/WAV サンプルから物理パラメータを **逆推定** する Python パイプライン。**学習じゃなくて解析**が主目的。

```
01_analyze_dataset.py    : スペクトル解析（FFT → パーシャル抽出 → T60）
02_extract_modes.py      : モード抽出（mode0…mode7 の振幅・周波数・減衰）
03_map_parameters.py     : 倍音バランス → pickup_position 等に逆推定
evaluate_model.py        : 合成音 vs 実音 周波数応答比較
```

T60 測定は **Hilbert 解析信号法**を使った。RMS 窓法より精度がいい。

成果:

- 低音域補正: `noteAtten` 除数 24 → 18（A0 vs A4 差 -3.7dB → -1.1dB）
- ベロシティ応答: 19層 Rhodes Classic サンプル解析から `strikeRolloff` の velFactor を設計
- トーンバー周波数テーブル: 手推定 mode2 比 3.03 → 実測 6.0 に修正
- `pickup_gain` テーブル: ノート別・モード別の最適ゲイン

## CMA-ES で自動最適化

CMA-ES（Covariance Matrix Adaptation Evolution Strategy）でパラメータをブラックボックス最適化。`pycma` 使用。

最適化対象:

| パラメータ | デフォルト | 範囲 |
|---|---|---|
| `pickup_position` | 0.978 | [0.7, 0.99] |
| `em_damping` | 0.058 | [0.01, 0.5] |
| `tonebar_coupling` | 0.922 | [0.1, 1.0] |
| `hammer_hysteresis` | 0.354 | [0.0, 0.8] |
| `strike_rolloff` | 0.35 | [0.05, 2.0] |
| `mode_damp_factor` | 1.0 | [0.1, 5.0] |
| `pickup_freq_alpha` | 0.8 | [0.3, 1.0] |

ゲイン系（`pickup_distance`, `coilTurns`, `hammer_stiffness_max`, `body_resonance_mix`）は **固定**。理由は `noteAtten` との相互作用で NaN / クリッピング → 永続的無音化を引き起こすから。*一度永続無音化を踏むと「あれ、シンセが死んだぞ…」となって、原因切り分けにすごい時間が溶ける。* なので固定化。

損失関数は 4項加重:

```
L_total = 1.0·L_partial_amp    （倍音振幅比 log²）
        + 0.8·L_partial_t60    （per-partial T60 log 距離）
        + 0.5·L_inharmonicity  （モード周波数比の相対誤差）
        + 0.1·L_mss            （Multi-scale Spectral Loss）
        + 0.3·L_rms            （絶対音量、無音化への収束防止）
```

`L_rms` を入れたのは大事。これが無いと CMA-ES が「音を小さくすればロスが下がる」と気づいて、極端なゲイン低下に収束する。

### 設定

- 集団サイズ: 16
- 最大世代: 200
- 初期探索幅 σ₀: 0.3
- 早期終了: 50 世代改善なしで停止
- 評価ノート: [36, 52, 60, 69, 72, 84]（C2, E3, C4, A4, C5, C6）

### 成果（v0.8.0）

- 損失値: **34.91 → 25.23**（27.7% 改善）
- `pickup_position`: 0.95 → 0.930
- `tonebar_coupling`: 0.994 → 0.390
- `hammer_hysteresis`: 0.3 → 0.226

採用基準: スペクトル系パラメータのみ採用（ゲイン影響なし）。

## バグ修正の苦い記憶

v0.5.0 で `sigma` バグ修正。モード形状関数の式が逆数になっていた。

```
× sigma = (sinh + sin) / (cosh + cos)   ← 間違い
○ sigma = (cosh + cos) / (sinh + sin)
```

これに気づくまでに **2 日**かかった。「なんか低音域だけ Bell 感が弱い」「なんか mode2 が出にくい」と症状はあったけど、根本原因が式の逆数だとは思わない。

教訓: **物理モデルのバグは「音はそれっぽく聴こえるけど、定量計測すると合わない」という形で潜む**。サンプル波形と DDSP 逆推定の比較計測を最初から組んでおくべき。これがあったから 2日で済んだとも言える。

## ポリフォニック管理

```
MIDI Callback Thread                Audio Thread
├─ noteOn()  ──→ CommandQueue (SPSC, 256→2048)
├─ noteOff() ──→ drainCommands()
└─ set*()    ──→ syncAtomicParams()
                                    ├─ NoteVoice::process() × 16 ボイス
                                    ├─ BodyResonance フィルタ
                                    └─ AVAudioSourceNode renderBlock
```

`memory_order_acquire/release` ペアでバリア明示。ボイススチール：17 ノート以上で最古ボイス再割り当て。

マルチサンプルレート対応: 44.1 / 48 / 96kHz（モード数動的クランプ、安定性限界 ω < 1.8 × sampleRate を超えないように）。

## バージョンの節目

5 日間で v0.1.0 から v0.8.0 まで:

| Version | Phase | 主な内容 |
|---|---|---|
| 0.1.0 | 0+1 | プロトタイプ：物理モデル基盤、C++/Swift ブリッジ、SPSC キュー |
| 0.2.0 | 3 | AUv3 プラグイン、波形ビジュアライゼーション、全 88 ノート安定性 |
| 0.2.1 | 4a | Hunt-Crossley ヒステリシス、EM ダンピング |
| 0.2.2 | 4b | RLC ピックアップフィルタ、ビンテージ非均一性 |
| 0.3.0 | 4c | トーンバー連成、ボディ共鳴 EQ、逆モデリング解析 |
| 0.4.0 | 4d | 磁束差分ピックアップ刷新（Faraday 則厳密離散化） |
| 0.5.0 | 4e | sigma バグ修正、mass 正規化、modeShape() 精度改善 |
| 0.6.0 | 5 | Shore 硬度ベースハンマー指数 |
| 0.7.0 | 6 | **CMA-ES v2 最適化パイプライン**導入 |
| 0.8.0 | 7 | CMA-ES v3 + App Store 対応（NaN ガード多重化） |

## やり残し

コードレビューで指摘された HIGH 優先度の課題:

1. **mode boost 再有効化**（`TineModel.cpp:556-559`）: 現在コメント化、低音域 mode1 の初期励起が 2〜3倍過小
2. **excite() 呼び出し追加**（`NoteVoice.cpp:90-110`）: noteOn 時のモード速度初期化が未実装、pp 領域での励起不足
3. **saturationLevel 段階的引き下げ**（2.0 → 1.2）: mode boost と飽和の相互作用で THD 悪化

iOS アプリは Core 実装済みだけど UI 磨き不足のまま放置。

## このプロジェクトが残したもの

物理モデル単独で Rhodes を再現する基盤は完成した。CMA-ES で 27.7% の自動改善も出た。

ただ、**実音との Mel-L2 差が 274 程度残っていた**（Baseline 評価）。CMA-ES でいくら最適化しても、ここから先は手つかず。

そして、残差を分析してみたら **R_struct = 1.0**（18 サンプル全てで強い周期的構造）が検出された。「**残差にパターンがある = データドリブンで学べる相手だ**」という発見。

これが次の TineModeler2 への直接の動機。

> 残差を DDSP で学習させよう。
