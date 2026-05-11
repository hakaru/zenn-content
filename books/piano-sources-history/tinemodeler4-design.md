---
title: "番外編 — TineModeler4 を設計したら TineModeler3 のバグが出てきた"
---

ここまでの本を書きながら反省を整理していたら、自然と「次やるならどう作る？」が気になり始めた。**TineModeler4 の設計を始めて、検証実験を 2 本走らせたら、片方で TineModeler3 の実装バグが見つかった**。

しかも単発のバグじゃなくて、「**設計の中心概念がそもそも動いていなかった**」という重い系統。本編で「機能 100% 完成、テスト 67/67 passing」と書いた TineModeler3 の話を、ここで一部訂正することになる。

順を追って書く。

## まず TineModeler4 を考え始めた動機

本編の `07-lessons.md` で書いた未解決課題：

- アタック 0-30ms のメカ感が物理 + DDSP だけだと薄い
- 機械ノイズ（キークリック、ダンパー音、ペダル音）は DDSP の残差学習でもうまく拾えない
- 個体差の説得力

これらは「物理モデル + ML 単独では原理的に届かない領域」。**少量の実サンプルを足せばどうにかなる**んじゃないか、というのが TineModeler4 の出発点。

## 自録音 vs ETI Roads MKII

最初は「自録音 20 ノート × 3 vel × 30ms = ~150KB」という案で設計を組んだ。法務的に完全自前にしたかった。

ところが手元の SSD に **ETI Roads MKII** という市販 Kontakt ライブラリがあった。これは元々 TineModeler の DDSP 逆推定で参照データとして使っていたもの。`/Volumes/Media/AudioSample/ETI Roads MKII/WAV/` に WAV が直接展開してあり、構成は：

```
SUS (sustain)  : 3,064 ファイル （88 鍵 × ~30 vel）
REL (release)  : 2,701 ファイル
PER (percuss)  :    11 ファイル （メカ音）
PED (pedal)    :    11 ファイル （ペダルノイズ）
NOISE          :     1 ファイル
```

ライセンスを確認すると、**他製品への埋込配布が許諾されている**（ライセンス所有者として自分が確認）。

これが使えるなら自録音はやらなくていい。**20 ノート × 3 vel ではなく、88 鍵 × 3 vel が手に入る**。録音セッション 2 日も浮く。

ただし、ライセンス通りに使う前提として：

- 自分が確認した内容を `docs/ETI_LICENSE.md` に記録する
- 加工後の埋込が許諾範囲か疑問なら別途文書確認する

…という運用ルールを Day 0 で決めた。法務面は本シリーズで一番慎重に扱ってきた領域だから、ここはちゃんとやる。

## 設計骨子（Rev.1）

ETI 採用前の自録音前提で組んだ初版：

```
HybridVoice (Rev.1)
├─ ATTACK (0–30ms)   : 自録音トランジェント
├─ MID    (30–100ms) : クロスフェード
└─ SUSTAIN(100ms–)   : Physics + DDSP
```

クロスフェード区間 30-100ms の根拠は「**DDSP の GRU 隠れ状態が定常状態に達するまでのウォームアップ時間**」。100ms あれば GRU が立ち上がる、という前提で組んだ。

…**この前提が崩壊する**ことを、検証で知ることになる。

## 検証 (1) DDSP ウォームアップ計測

何 ms で GRU 隠れ状態が定常に達するかを実機計測。手順は単純：

```
1. coremltools で rhodex_streaming.mlpackage をロード
2. 一定の f0 / loudness / midi 入力を 100 フレーム流す
3. 各フレームの隠れ状態 h_out の L2 ノルムを記録
4. 「漸近値の 95% に達するフレーム数」を測定
```

結果の数値だけ見ると：

```
midi=60, loudness=0.833:
  frame 0:   h_norm = X
  frame 1:   h_norm = X    ← 同じ
  frame 50:  h_norm = X    ← 同じ
  frame 99:  h_norm = X    ← 同じ
```

全フレームで同じ。`||h(t) - h(t-1)||_2 = 0` がずっと続く。

*…え？*

最初は実験コードを疑った。一定入力だから出力も一定、というのは GRU でもありうる。でも `gate` 出力（モデルが「物理 + DDSP のどっちをどれくらい混ぜるか」を指示する値）も完全に固定値 0.9326。100 フレーム間 distinct value = 1。**音色が時間進行で変化していない**。

### 状態書き込みテスト

`MLState.write_state` で隠れ状態を手で初期化してみる：

| 初期 h_in | gate 出力 |
|---|---|
| zeros | 0.9326 |
| 0.5 × ones | 0.8013 |
| N(0, 0.1) random | 0.9912 |

**初期値を変えると出力が変わる**。つまりモデルは隠れ状態を「入力としては読んでいる」。でも、推論後に状態を「書き戻していない」。

### MIL プログラム検査

CoreML の中間表現（MIL）を `coremltools.get_spec().mlProgram` で walk する：

```
read_state op            : 1 個   ← 入口で h_state を読む
coreml_update_state op   : 0 個   ← ★ ない
内側 while_loop の scatter: GRU 内部の time-axis 累積（外側 state じゃない）
```

**`coreml_update_state` が無い**。

これは PyTorch → CoreML 変換時に **`ct.convert(states=[ct.StateType(...)])` 引数を渡し忘れていた** ことを意味する。stateless として変換されたモデルに、後から無理やり MLState API を取り付けても、状態は読まれるだけで書き戻されない。

## つまり何が起きていたか

TineModeler3 の StreamingMethodA は、**実装上は完全に stateless** だった。

GRU の recurrence がフレーム間で機能していない。voice 単位 MLState を頑張って正規化した M3 修正（本編の TineModeler3 章で書いたやつ）は、**理論上は正しいが実態としては効いていなかった**。

じゃあ TineModeler3 で音色が時間進行で変化して聴こえていたのは何か。整理すると：

1. f0 の微揺らぎ（物理モデル側で変化）
2. loudness_norm が hop 毎に再計算される
3. STFT の OLA（Overlap-Add）による重畳

この 3 つの組み合わせで「動的に変化している感じ」が出ていただけ。**学習させた GRU の時間文脈情報は、再生に一切活かされていなかった**。

…これはかなり厄介な発見。

## 設計への影響

「GRU が定常になるまでクロスフェード区間を確保する」という設計根拠が消えた。じゃあクロスフェード区間は不要か、というとそうでもない：

- STFT 窓 2048 サンプル @ 48kHz = **42.6ms のアルゴリズミックレイテンシ**
- loudness_norm は物理層 RMS から計算される
- サンプル区間の attack transient が混ざると loudness 推定値が狂う
- DDSP は loudness_norm に応じた補正を出すから、その狂いが音色に影響する

つまり「**STFT/loudness aliasing 回避のため**」という別の理由でクロスフェード区間は要る。推奨は **50-100ms**（Rev.1 の 30-100ms より少し長い）。

## 検証 (2) サンプル補間

並行して走らせていたもう一つの検証。「20 ノート録音 + 線形補間で 88 鍵カバー」は破綻しないか、既存の rhodes-classic データで事前評価した。

Mel-L2（TineModeler2 baseline = 274）の結果：

| 区間 | ±1 半音 | ±2 半音 | 全体 |
|---|---|---|---|
| アタック 30ms mean | 78 | 112 | **88** |
| フル 1.0s mean | 152 | 250 | 182 |

アタック区間限定なら baseline の 1/3。**スペクトル的には破綻しない**。

…と思ったら、F0（基本周波数）が壊れていた：

| | 実音 | 補間音 |
|---|---|---|
| mean \|cents\| | 2.1 | **84.7** |
| max \|cents\| | 10 | **160** |

線形ブレンドはピッチ補間にならない。波形をそのまま blend すると、pYIN は近傍 2 ノートのいずれかにロックする。**±50 cent 以内という設計許容はまったく満たせない**。

スペクトル包絡は OK、でもピッチが 84 セントずれる。これは「アタックの音色は近い、でも音程が違う」状態で、可聴的にはかなり厳しい。

ただ、これは ETI 採用で **問題そのものが消える**。全 88 鍵が手元にあるなら、補間する必要がない。**実験 (2) の結論は「ETI 採用は正しい」を補強する結果に転じた**。

## Rev.2 設計

両実験の発見を踏まえて全面書き直し：

```
HybridVoice (Rev.2 — 4 層)
├─ ATTACK (0–50ms)    : ETI ATK サンプル（全 88 鍵 × 3 vel）
├─ MID    (50–100ms)  : クロスフェード（loudness aliasing 回避区間）
├─ SUSTAIN(100ms–)    : Physics + DDSP（再エクスポート + ETI 再学習）
└─ RELEASE (noteOff–) : ETI REL サンプル（全 88 鍵 × 1 vel）
```

主要な変更：

| 項目 | Rev.1 | Rev.2 |
|---|---|---|
| サンプル供給源 | 自録音 20 ノート + 線形補間 | ETI Roads MKII 全 88 鍵 |
| 録音セッション | 必須（2 日） | 不要 |
| クロスフェード根拠 | GRU ウォームアップ | STFT/loudness aliasing 回避 |
| クロスフェード推奨 | 30-100ms | **50-100ms** |
| DDSP 再エクスポート | 不要 | **必須**（stateful 変換修正） |
| DDSP 学習データ | rhodes-classic（Keyscape） | ETI（法務クリーン） |
| REL レイヤー | 未計画 | **新規追加** |
| 総配布サイズ追加 | ~150KB | **10-18MB** |

REL レイヤーが追加できたのは大きい。**Rhodes の note-off 時の「ちゃい〜ん」という余韻**を、TineModeler シリーズで初めて表現できる。

## DDSP 再エクスポート手順

`ct.convert` 呼び出しで `states` 引数を正しく渡す：

```python
import coremltools as ct

mlmodel = ct.convert(
    model,
    convert_to="mlprogram",
    inputs=[
        ct.TensorType(name="f0_norm", shape=(1,1,1)),
        ct.TensorType(name="loudness_norm", shape=(1,1,1)),
        ct.TensorType(name="midi_norm", shape=(1,1,1)),
    ],
    states=[
        ct.StateType(
            wrapped_type=ct.TensorType(shape=(1, 512)),
            name="hidden_state",
        ),
    ],
    minimum_deployment_target=ct.target.macOS15,
)
```

これで `coreml_update_state` op がモデルに入る。検証は実験 (1) の Swift ハーネスを再実行して `||h(t) - h(t-1)||_2 > 0` を確認すればいい。

旧 OS（macOS 14 / iOS 17）は `MLState` 非対応なので、`DDSPRunner/StateManager` で OS バージョン分岐して、手動で `h_in` を保持する従来方式にフォールバックする。

## ETI による再学習

ETI Roads MKII で DDSP を再学習する。設計上は **stateful 変換修正と再学習を同時にやる** 方が効率的：

- データ：ETI SUS 88 鍵 × ~30 vel = 3,064 ファイル
- 環境：Vast.ai RTX 4090 Interruptible（$0.15-0.30/h）
- 設定：TineModeler2 Phase 2 v2 設定踏襲（BalancedNoteSampler + ResidualGate + loudness-guided 正規化）
- 予算：~¥3,000-6,000
- 期間：Phase 1（Day 4-10）

これにより法務的に **rhodes-classic（Keyscape）から完全に切り離せる**。TineModeler シリーズで長く付き合った Keyscape 由来データに、ここで完全に別れを告げる。

## 60 日ロードマップ

```
Phase 0 (Day 1-3)   : Setup（リポ初期化、SHIP.md、ETI_LICENSE.md）
Phase 1 (Day 4-10)  : DDSP Re-export + Re-train【最優先】
Phase 2 (Day 11-14) : ETI Sample Pipeline
Phase 3 (Day 15-19) : Core Extraction（TineModeler3 から Core 抽出）
Phase 4 (Day 20-27) : HybridVoice + Engine Integration
Phase 5 (Day 28-34) : Shell & UI、α 評価開始
Phase 6 (Day 35-50) : β 評価 → RC 評価 → 出荷判定
Phase 7 (Day 51-60) : Ship Preparation（codesign / App Store 提出）
```

DDSP 再エクスポート + 再学習を Phase 1 最優先にした。これが全フェーズの前提なので。

TineModeler3 はリリースしない。「機能完成 + テスト通過」と本編で書いたが、**今回の検証で実態と乖離が見つかった**以上、社内検証用に留めて、TineModeler4 一本に集中する。

## 教訓

このボーナス章を書いていて、本シリーズの反省点 #1 「**聴感評価の不足**」がそのまま今回も効いた、と気づいた。

> Mel-L2 / val_loss / CPU% / レイテンシ / テスト件数

これらの客観指標がすべて緑だった TineModeler3 で、**「設計上は stateful、実態は stateless」**という致命的な不整合が、テストでは検出できなかった。

なぜか。**「GRU 隠れ状態がフレーム間で本当に変化しているか」を確認するテストがなかった**から。

XCTest は API の戻り値しか見ない。Swift CoreML の MLState API を叩けば値は返ってくる。「数値再現性テスト」も入力が同じなら出力も同じなのは当然なので、stateless でも通る。

**動作と意図の乖離を検出するテストは、意識しないと書けない**。普通の単体テストでは絶対に出てこない。

Rev.2 設計では `DDSPStatefulTests` を新規追加した。「100 フレーム連続推論で `||h(t) - h(t-1)||_2 > 0` がフレーム経過と共に減衰すること」を assert する。これは「意図通りに動いている」を検証するテスト。

…こういうの、本来は最初から書いておくべきだった。**ML 系コンポーネントは「実装と意図の乖離」がブラックボックス内で起きやすい**から、専用の回帰テストを切るべき。次から徹底する。

## 余談

ここで言うと矛盾するけど、TineModeler3 のサウンドそのものは別に悪くなかった。GRU recurrence が死んでいても、f0 揺らぎ + loudness 再計算 + STFT OLA でそれなりに動的な音色は出ていた。

つまり **「設計通りに動いていなかったが、それなりに鳴っていた」**。これが一番厄介で、**「動いてる」と「設計通り動いてる」は別物** という当たり前のことを、改めて噛みしめている。

Rev.2 でちゃんと stateful にしたら、もっと豊かに鳴るはず。…と思っているけど、これも実機で確認するまでは「思っている」止まりにしておく。
