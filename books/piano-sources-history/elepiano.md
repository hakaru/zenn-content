---
title: "elepiano — Raspberry Pi 5 上で Keyscape を鳴らす"
---

最初の出発点。**Raspberry Pi 5 上で動く、サンプリング + 物理モデリング併用のピアノ／オルガン音源**。

期間は 2026-03-01 〜 03-10 の 10日間で、86 commits。Pi 5 上の C++20 メインエンジン + iOS の BLE-MIDI リモコンの 2 つ組構成。

## やりたかったこと

- ハイエンドサンプル音源（Spectrasonics Keyscape）の音を Pi 5 で鳴らす
- 同じ筐体で物理モデリングのトーンホイールオルガン（setBfree）も鳴らす
- 低レイテンシ（< 2ms）
- iOS から BLE-MIDI で遠隔操作

ピアノ系はサンプル、オルガンは物理モデル。**プログラムチェンジ一発で切り替えられる**ハイブリッド構成。

## 一番の難所：Keyscape SpCA フォーマット

Keyscape のサンプルファイルは独自フォーマット（SpCA）に包まれている。展開のリバースエンジニアリングが、このプロジェクトの最大の山だった。

判明したことを並べると:

- SpCA は FLAC をラップした独自形式
- 先頭 4 バイトを `53 70 43 41`（"SpCA"）→ `66 4C 61 43`（"fLaC"）に置換すれば FLAC として読める
- 大多数のサンプル（Rhodes / Wurlitzer）は **暗号化なし**（magic 置換のみ）
- ところが **Sustain サンプルだけは XOR 暗号化** されている
  - キー: `[0xEF, 0x42, 0x12, 0xBC]`（4 バイト周期）
  - フレーム 0 は平文（ただし CRC-16 が意図的に破損、libFLAC は許容）
  - フレーム 1 以降は全バイト XOR

*…なんでフレーム 0 だけ平文なんだろう。たぶん「ヘッダーは読めないとライブラリが拒否するけど、データは隠したい」って妥協だと思う。*

実装は `tools/extract_samples.py` で `.db` XML をパース → バイナリ抽出 → XOR 復号 → `samples.json` 生成、という流れ。

ついでに NKX（Kontakt 形式）も調べたけど、こっちは **AES-128-CBC** でガッチリ守られていて手出し不可能。`docs/NKX_format.md` に分析だけ残してある。

## アーキテクチャ

```
iOS アプリ (BLE-MIDI)
    ↓ CoreBluetooth / MIDIKit
Raspberry Pi 5
    ├─ ble_bridge.py (BlueZ GATT, ALSA MIDI 変換)
    └─ elepiano (C++ メインプロセス)
        ├─ SynthEngine (32 ボイス, FLAC サンプルベース)
        ├─ OrganEngine (物理モデル, U/L/P マニュアル)
        ├─ FxChain (Tremolo/Phaser/NAM/IR/Space/Chorus)
        └─ AudioOutput (ALSA PCM, SCHED_FIFO RT)
```

3 つのスレッドで動く。

```
メインスレッド          : StatusReporter (100ms ポーリング)、ログ flush
MIDI スレッド (SCHED_OTHER) : ALSA seq から MIDI 受信 → SPSC queue
オーディオスレッド (SCHED_FIFO 80) : SynthEngine.mix / OrganEngine.mix / FxChain.process
```

スレッド間通信は **lock-free SPSC キュー**（`SpscQueue<T, N>`）。malloc も lock も例外も使わない。この `SpscQueue` は以降全プロジェクトで使い回すことになる。

## オルガンは setBfree を抱き込んだ

Hammond B3 オルガンは setBfree（C のオープンソース実装）を `vendor/setBfree/` に静的リンクして取り込んだ。Leslie ローター（回転スピーカー）の物理モデルが `whirl.c` にあって、これがかなり良い出来。

```
[1] 800Hz Butterworth クロスオーバー（ホーン / ドラム分離）
[2] ホーン: 可変ディレイライン（ドップラー効果）
[3] ドラム: AM 変調（cos ベース）
[4] ステレオミックス + 正規化
```

ホーン回転速度は Slow=0.672Hz / Fast=7.056Hz、ドラムは Slow=0.600Hz / Fast=5.955Hz。加減速の時定数まで実機準拠（τ_accel=0.161s ほか）。

全バッファ静的確保（`horn_delay_buf_[512]`, `sin_lut_[4096]`）で RT-safe にしてある。

## NAM（Neural Amp Modeler）統合

アンプシミュレーション。Chameleon 7603 モデルを `NeuralAmpModelerCore`（Eigen 3 ベース）経由で統合。

FX チェーンの中で `Channel9(LV2) → NAM → IR Cabinet → 出力` という並びにして、CC88（Gain）/ CC117（Wet）/ CC119（Model 選択）で制御できるようにした。

…ただこの NAM、**iOS には移植できない**（Eigen 依存が重い）。次の iRhodes では手書きの WaveNet 推論エンジンに置き換えることになる。

## RT-safe にどこまで詰めたか

Pi 5 で PREEMPT_RT カーネル（`linux-image-rpi-v8-rt 6.12.62`）を入れて、`cyclictest` で Max jitter ≈ 6μs、Avg ≈ 1μs まで出した。

やったこと:

- SCHED_FIFO priority=80 でオーディオスレッド隔離
- `threadirqs` カーネルパラメータ（割込みスレッド化）
- CPU governor `performance` 固定（ondemand → 常時 2.4GHz）
- period_size=32 で理論レイテンシ ~1.3ms
- サンプル先頭無音トリミング（92.9ms → 0ms、体感レイテンシ激変）

*このへんは Pi で実機シンセを作るときのほぼ定番セット。* 一度組んでおくと以降の世代では悩まなくていい。

## BLE-MIDI で iOS から制御

Pi の上に Python の BLE ブリッジ（`ble/`）が常駐していて、独自 GATT サービス（UUID `e1e00000-...`）を立てる。

```
ble_bridge.py        : BlueZ D-Bus + GATT サービス登録
status_monitor.py    : elepiano UNIX socket (/tmp/elepiano-status.sock) から JSON 受信 → BLE Notify
alsa_midi_sender.py  : BLE から CC/PC 受信 → ALSA seq に送信
```

GATT キャラクタリスティックはこんな構成。

| UUID 末尾 | 名前 | R/W/N | フォーマット |
|---|---|---|---|
| 01 | Status | read, notify | JSON（最大 512B） |
| 02 | CC Control | write-without-response | [cc, val] 2B |
| 03 | Program Change | read, write, notify | [pg] 1B |
| 05 | Batch CC | write-without-response | [cc1,v1,...] 最大 10 ペア |
| 06 | Command | write | UTF-8 コマンド |

iOS 側は Swift 6.0 + SwiftUI + MIDIKit 0.11+ + SwiftData + CloudKit。Feature-based の `@Observable` 構造で、Connection / Piano FX / Organ / Presets / Settings の 5 タブ。

プリセットは iCloud 同期にしておいた。BLE スロットリングは「同じ CC を 25ms 以内に重複送信しない（最大 40Hz）」。

## 開発の節目

86 commits の中で骨格になったコミット:

| 日付 | コミット | 内容 |
|---|---|---|
| 03-01 | `22f9a23` | PiSound MIDI synth engine for Rhodes Classic samples（最初の動くやつ） |
| 03-03 | `b85fa08` | Organ engine — 2 manuals + pedal board |
| 03-04 | `76be981` | Organ engine — Full physical modeling (Leslie/click/V-C/tonewheel) |
| 03-06 | `c5c52f9` | PREEMPT_RT kernel, parallel sample loading, PCM cache, program change |
| 03-08 | `0071e7c` | Biquad/DelayLine extraction, FxChain heap migration, fast_tanh |
| 03-09 | `d9a167a` | NAM (Neural Amp Modeler) integration |
| 03-09 | `a3540c8` | **SpCA CRC-16 copy protection 修正**（最後の謎の解明） |
| 03-10 | `ad02e12` | iOS Phase 1 — architecture refactor, SwiftData, iPad support |

ピークは 3月 5日の 21 commits / 日。PREEMPT_RT + BLE integration の山。

## やり残し

`tasks/todo.md` に OPEN issue が 12件。主なものを抜くと:

- HIGH: `tellg()` 失敗時の過剰割り当て防止、`samples.json` 入力サイズ制限
- MEDIUM: `extract_samples` 並列化、無条件 MIDI イベントロギングによる xrun
- LOW: SpCA XOR 検出のサンプルレート汎用化、ALSA デバイス自動検出

それと iOS Phase 2（MIDI Learn / A-B NAM 比較）は未着手のまま、次のプロジェクトに移った。

## このプロジェクトが残したもの

ここで作った C++ コア — `SynthEngine` / `Voice` / `SampleDB` / `SpscQueue` / `biquad` / `delay_line` / `flac_decoder` / `fx_chain` — は、**ほぼそのまま iRhodes で iOS に移植される**。

CC マッピング体系も継承される。CC1-4 = Tremolo/Phaser、CC88/117/119 = NAM、CC104/105 = IR、CC108-116 = EQ/Chorus/Space。これが以降のシリーズの共通言語になった。

10日で密度が高すぎて、`docs/ClaudeWorklog202603*.md` という日次ワークログが 10ファイル残っている。後で経緯を辿るのに役立った。

次の章は iOS への移植 — iRhodes の話。
