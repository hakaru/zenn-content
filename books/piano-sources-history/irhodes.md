---
title: "iRhodes — iOS への完全移植とハイブリッドの萌芽"
---

elepiano の C++ コアを iOS に移植したやつ。iPhone 単独で USB MIDI キーボードを繋いで弾ける。

ただこれ、ただの移植では済まなかった。**サンプル単独だと容量が 2.7GB に膨らんで配布不可能**。ここで初めて「サンプル + 物理モデル」のハイブリッドエンジンに手をつけることになる。

## まずぶつかった壁

iOS で Rhodes サンプル音源を配布するとき、フルベロシティ層を全部入れると 2.7GB。**iPhone 12 のメモリ予算では無理**。iPhone 14 でもアプリ起動時に OOM クラッシュ。

最初はメモリ予算管理（機種別、RCU 風 2 世代保持）で頑張った。

```
iPhone 12 / SE3 (3GB RAM) : ~400MB 実用上限、Attack 先頭 5秒
iPhone 13 / 14  (4GB RAM) : ~600MB 実用上限、Attack 先頭 8秒
iPhone 15 Pro+ (6GB+)    : ~900MB 実用上限、Attack 先頭 12秒
```

でも先頭 12秒しか入れない、というのは「サスティンが切れる」ことを意味する。Rhodes のサスティンは 20秒以上伸びる。

*…これは物理モデルにやらせるしかない。*

## ハイブリッドエンジン（時間方向の繋ぎ目）

`HybridVoice` クラスで、1 ボイス内で **時間軸を 3 段階に分ける**:

```
HybridVoice
├─ ATTACK (0–100ms)
│   ・SamplePlayer 再生（サンプルの瞬間的なアタック質感）
│   ・TineModel も同時起動（出力は破棄、ウォームアップ）
│
├─ CROSSFADE (100–300ms)
│   ・等パワークロスフェード： sqrt(1-t) × sample + sqrt(t) × model
│
└─ SUSTAIN (300ms–)
    ・TineModel のみ（ベロシティ 128 段階が完全連続）
```

ATTACK の 100ms 間は両方走らせる。物理モデルが定常振動状態に達するまでウォームアップしておく。100ms 時点で等パワークロスフェードに入り、300ms でサンプルを完全に切る。

**サンプル容量 2.7GB → 186MB**。99.3% 削減。CPU 追加負荷は ~1.05倍（ATTACK 期間のみ両駆動）。

*この発想が出てきたとき、「物理モデルってちゃんと役に立つんだな」と思った。* 容量だけじゃなくて、ベロシティ連続応答（サンプル方式だと 19層 → モデル方式だと 128 段階連続）も同時に手に入る。

## サンプル段階ロード（Instant Play）

3 段階で時間差ロードする方式。

```
T+0.5s : CRITICAL — 中央オクターブ C3-C5 × ベロシティ 2層 × 先頭 1秒 ≈ 25MB
T+2-5s : HIGH     — 全ノート × ベロシティ 3層 × 先頭 2秒
T+5-20s: NORMAL   — 機種別上限（5秒 / 8秒 / 12秒）
```

T+0.5s 時点で中央オクターブだけは弾ける状態にする、というのがポイント。ユーザーは「アプリ起動 → 即演奏」を体感できる。残りはバックグラウンドで埋まっていく。

FLAC → PCM デコード結果は iOS `CachesDirectory` にキャッシュ。2回目以降の起動は爆速。

ライフタイム管理は **RCU 風 2 世代保持**（`atomic<SampleDB*> + prev_attack / prev_release`）で、オーディオスレッドが参照中の SampleDB を解放しないようにした。

## NAM を手書きで再実装

elepiano の NAM は `NeuralAmpModelerCore` + Eigen 3 依存だったけど、これを iOS に持っていくのは重すぎる。

`.nam` JSON 形式を直接読む手書き WaveNet 推論エンジンを書いた:

- Conv1D（dilated causal convolution, ring buffer）
- Conv1x1
- Layer / LayerArray
- 重みレイアウト: 13,802 パラメータ
- 44,100 ↔ 48,000Hz 線形補間リサンプリング（位相連続）
- モノミックス → 推論 → ステレオ書き戻し（CPU 節約）
- Ring buffer のビットマスク最適化（整数除算 → AND、20-40 サイクル → 1 サイクル）

性能は CPU < 1-2%（有効時）、オーディオスレッド確保ゼロ。Eigen 不要。

…これだけ書くと簡単そうに聞こえるけど、Conv1D の const UB（undefined behavior）に嵌まったり、`ring_write` インクリメント忘れてるバグで一晩潰れたりはした。

## IR Convolver（Overlap-Save）

キャビネットシミュレーション。Accelerate.framework の `vDSP_fft_zrip` ベース:

- P=256（処理ブロック）、FFT=512、最大 IR 長 1024 サンプル
- 4 パーティション分割
- 48kHz → 44,100Hz 線形補間リサンプリング内蔵
- バッファは全てコンストラクタで `posix_memalign` 確保（オーディオスレッド確保ゼロ）
- 最大 16 プリセット（CC104 選択、CC105 wet）

CPU ~2-3%、メモリ ~64KB。

## paramSerialQueue — Producer 一元化のトリック

elepiano の SPSC キュー は「単一 Producer → 単一 Consumer」を前提にしていた。Pi 上では MIDI スレッド 1つだけが Producer だったから問題なし。

ところが iOS だと CoreMIDI コールバック（リアルタイムスレッド）と SwiftUI（メインスレッド）の **両方から CC/MIDI イベントが発生**する。SPSC 前提違反。

解決:

```
CoreMIDI callback  → paramSerialQueue.dispatch ──┐
SwiftUI KnobView   → paramSerialQueue.dispatch ──┴→ SpscQueue.push (single producer)
Audio thread       ←────────────────────────────── SpscQueue.pop  (single consumer)
```

`paramSerialQueue` というシリアル `DispatchQueue` を 1つ挟む。両方ともそこに投げるので、SPSC の前提が保たれる。ロックは完全排除。

*このトリックは、後で TineModeler3 でも同じ問題に直面するので、ここで覚えておくと役に立つ。*

## ARM64 最適化・デノーマル対策

iOS で地味に効くやつ:

- FPCR ビット設定（FZ = Flush-to-Zero）→ デノーマル数をゼロに変換
- NaN/Inf ガード（フィルタ不安定時に出力クリア）
- Voice stealing 時の 2ms 高速フェード + アンチクリック
- Phaser フィードバック状態のデノーマルキル

デノーマル数を放置すると CPU スパイクで underrun になる。スピーカー保護にもなる。

## CPU バジェット

| 項目 | CPU% |
|---|---|
| TineModeler 16 ボイス（5 モード） | ~6.4% |
| SamplePlayer 16 ボイス | ~1.0% |
| FxChain（全 FX ON） | ~4.3% |
| BodyResonance | ~0.2% |
| AVAudioEngine | ~1.0% |
| **合計** | **~13%** |

Gig Safety 閾値 80% に対して 6 倍の余裕。

## elepiano から引き継いだもの / 変えたもの

| 観点 | elepiano（Linux/Pi） | iRhodes（iOS） |
|---|---|---|
| OS | Linux（ALSA） | iOS 17+ |
| オーディオ API | ALSA / JACK | AVAudioEngine |
| MIDI | ALSA MIDI | CoreMIDI |
| UI | TUI | SwiftUI（ランドスケープ） |
| エフェクト | LV2 ホスト | 内蔵 + NAM/IR 手書き |
| サンプル | FLAC バンドル | FLAC → PCM キャッシュ |
| 段階ロード | なし | **3 段階（Instant Play）** |
| ハイブリッド | なし | **タイムオーバーラップ方式** |
| 物理モデル | なし | **PhysicalModel/ 6 クラス** |
| プリセット | ファイル型 | JSON + Documents |

C++ コア（`SynthEngine`, `Voice`, `SpscQueue`, `biquad`, `flac_decoder`, `dr_flac`, `delay_line`）はほぼそのまま。CC マッピングも同じ。

ただ、`PhysicalModel/` の 6 クラス（TineModel, NoteVoice, HammerModel, PickupModel, DamperModel, BodyResonance）はここで初登場。これが次の TineModeler で独立リポジトリに分離されることになる。

## やり残し

| Phase | 内容 | 状態 |
|---|---|---|
| H2 | FftModalAnalyzer + setModalState() — クロスフェード精度向上 | 計画中 |
| H3 | CC88-92 公開 + Economy/HQ モード切替 | 計画中 |
| Preset Cloud | CloudKit ベースプリセット共有 | Post-launch |
| Bluetooth MIDI | ワイヤレス対応 | Post-launch |

技術的な残課題:

- **スペクトル精度（低音・高音）**: C2 / C3 で FFT bin 幅限界、C5 以上で +500cent 超のズレ。音域別クロスフェード長の動的調整が必要
- **IR Convolver の BLOCK_SIZE 不一致**: FX 全体とのブロック境界不一致がデジタルノイズ原因の可能性

## このプロジェクトが残したもの

iRhodes は「elepiano の iOS 版」という側面と、**ハイブリッドエンジンの最初の試作場**という側面を持つ。

そして 1 つ重要な認識が生まれた。

> 物理モデル側の品質を上げないと、ハイブリッドの意味がない。

サスティン区間を物理モデルに任せる以上、その物理モデルが「Rhodes っぽく聴こえる」必要がある。iRhodes の中の `PhysicalModel/` は最低限の実装で、まだ Rhodes として説得力が足りなかった。

ここで次の判断:

> 物理モデルを独立リポジトリにして、単独で品質を追い込もう。

これが TineModeler の始まり。
