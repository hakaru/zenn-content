---
title: "Rhodesシンセを4回作り直して学んだこと — C++17物理モデル + CoreML + AUv3"
emoji: "🎹"
type: "tech"
topics: ["swift", "ios", "macos", "coreml", "dsp"]
published: false
---

# Rhodesシンセを4回作り直して学んだこと

## はじめに

**TineModeler4** は、Fender Rhodes Electric Pianoを物理モデル合成・DDSP（CoreML）・実録サンプルのハイブリッドで再現するAUv3/スタンドアロンアプリです。macOS 15 / iOS 18対象、Swift 6 + C++17 Interopで実装中です。

同じシンセを4回作り直しました（elepiano → iRhodes → TineModeler1〜3 → 4）。毎回「動いた」と思ったら致命的な問題が後から出てきました。この記事はその失敗から学んだことと、4回目でどう解決したかの記録です。

---

## 過去3回の主な失敗

### 失敗1: GRU recurrenceがサイレントに死んでいた（TineModeler3）

TineModeler3はDDSPモデル（GRU + スペクトル補正）を搭載していました。動作検証を通過し、Logic Proで鳴ることも確認しました。

ところが後から気づいたのですが、CoreMLモデルのGRUが毎フレームリセットされていました。`h_in`/`h_out` をただの入出力テンソルとして宣言していたため、coremltools変換後に隠れ状態が引き継がれていなかったのです。モデルは数値的には動くのでテストが通ってしまいます。でも音は「statefulなGRU」ではなく「毎回リセットされる前向き計算」でした。

**対策（TineModeler4）:**

```python
# NG: 普通の入出力テンソル
# h_in = torch.zeros(1, 1, hidden_size)
# h_out = model(x, h_in)

# OK: StateType でラップ
state = ct.StateType(
    wrapped_type=ct.TensorType(shape=state_shape, dtype=np.float16),
    name="hidden_state",
)
mlmodel = ct.convert(traced, states=[state])
```

さらに `state.copy_(h_new)` はcoremltools変換で "No matching select or slice" エラーになります。

```python
# NG
state.copy_(h_new)

# OK
delta = h_new - state
state.add_(delta)
```

`StateType`のdtypeはfp16必須（fp32は"State only support fp16"エラー）、coremltools 9.0からop名が`coreml_update_state` → `write_state`に変更されています。

これを機械的に検証するテスト (`DDSPStatefulTests`) を Phase 0 で書きました:

```swift
// GRUの隠れ状態が実際に更新されていることを確認
XCTAssertGreaterThan(norm(h1 - h0), 0, "GRU recurrenceが死んでいる")
```

### 失敗2: CoreML推論をRenderスレッドで同期実行していた

TineModeler3ではAU renderスレッドから直接 `MLModel.prediction()` を呼んでいました。CoreML推論は約42ms（RhoDexモデル, M2)。Logic Proの256フレームバッファ≈5.3msなので確実にunderrunします。

最初は「M1/M2なら速い」と楽観していましたが、AI処理が入るとフレームサイズに関係なくJITスパイクが発生します。

**対策（TineModeler4）:**

renderスレッドはfloat1個だけをatomicで読む。CoreML推論は別スレッドで回す。

```
[Renderスレッド]
  → requestFrame(voiceID, f0, loudness)  // non-blocking enqueue
  ← latestBroadbandGain(voiceID, gen, &out)  // atomic read

[Workerスレッド (DispatchQueue .userInitiated)]
  → CoreML.prediction()  // ≈42ms OK、ここで時間を使ってよい
  → packed.store(generation << 32 | gain_bits, .release)
```

世代タグ付きrejectionで、ボイスが奪われた後の古い推論結果が漏れるのを防いでいます。

### 失敗3: Core/Shell分離が設計だけで実装で崩れる

過去3回とも「レイヤー分離する」と設計書に書いていましたが、実装が進むにつれてどこからでもCoreMLをimportするようになりました。

**対策（TineModeler4）:**

CIで機械的に検証します。

```bash
# tools/check-layer-deps.sh
# Core層にApple SDKのimportがあればfail
grep -r "import CoreML\|import AVFoundation" Sources/Core/ && exit 1
```

```
Sources/
  Core/       → Pure C++17。Apple SDK一切なし
  Platform/   → CoreML / Accelerate はここだけ
  Bridge/     → Swift ⇄ C++ interop
  Shells/     → AUv3Plugin / StandaloneApp / TestHarness
```

このルール違反は `swift build` より前にCIで検出されます。

---

## ETIサンプルパイプライン

Rhodes実音のベースとして **ETI Roads MKII**（オープンソース）を使いました。ただし実態を調べると仕様書と大きく異なっていました:

| 項目 | 仕様書 | 実際 |
|------|--------|------|
| サンプルレート | 48kHz | **96kHz** |
| ビット深度 | 16bit | **24bit** |
| 鍵盤数 | 88鍵 | **73鍵（E0〜E6）** |
| 破損ファイル | 0件 | **36件（68byteスタブ）** |

5,788ファイルのWAVから313枚のFLAC（1.92MB）に絞り込んで `Resources/samples/` に同梱しています。

パイプラインで詰まったのはFADE処理のバグでした。

```python
# NG: 最終サンプルがゼロにならない
t = np.arange(n) / n   # t[-1] = (n-1)/n ≠ 1.0

# OK: 最終サンプルが確実に0
t = np.linspace(0, 1, n)  # t[-1] = 1.0
gain = np.sqrt(np.clip(1 - t, 0, 1))
```

この1行の差で `07_validate.py` の `atk_end_click` 警告が出続けました。

また中間WAVを16bitで保存していたため、後段のTPDFディザが機能していませんでした:

```python
# NG: 16bit保存 → 後のTPDFが意味なし
write_wav_pcm16(tmp, audio)

# OK: 中間は24bit保持、最終変換の直前だけ16bit量子化+ディザ
write_wav_pcm24(tmp, audio)  # 03_crop_atk / 04_crop_rel
# → 05_to_flac でTPDFディザ + 16bit → FLAC
```

---

## DDSP再学習（Vast.ai RTX 4090で19分、約450円）

DDSPモデルのアーキテクチャは TineModeler2 の `MethodA`（GRU + 3ヘッド）をベースにしています。ETI実音でファインチューニングするため Vast.ai のRTX 4090インスタンスを使いました。

- データセット: ETI 5,788 WAV → 44.1kHz HDF5 (3.96GB)
- 入力: F0 + A-weighted loudness + physics_audio (C++物理モデルからctypes経由)
- バッチ32、workers 6でGPU 97%飽和
- 200 epoch、19分、final loss 9.11
- 費用: 約$3（≈450円。想定3,000〜6,000円の大幅下回り）

torchcrepeでF0を検出する際、中音域（MIDI 36〜72）では2倍音をCREPEが拾ってしまうため自動補正が必要でした:

```python
ratio = median(f0) / midi_to_hz(target_midi)
if abs(ratio - 2.0) < 0.3:
    f0 /= 2.0  # octave correction
    octave_divisor = 2.0
```

MIDI 84以上は2倍音がCREPEの上限2006Hzを超えて検出失敗するため、MIDIラベルで補完しています。

---

## AUv3実装で詰まったポイント

### NSExtensionPointIdentifier の罠

```xml
<!-- NG: requestViewController必須になる -->
<string>com.apple.AudioUnit-UI</string>

<!-- OK: generic UIでよければこちら -->
<string>com.apple.AudioUnit</string>
```

`-UI` サフィックスをつけると `requestViewController` の実装が必須になりますが、Phase 5ではhost generic UIを採用したのでこちらは不要でした。

### AUParameterTreeのimplementorValueProvider無限再帰

```swift
// NG: parameterTree.value(forKey:) が再帰呼び出しになる
tree.implementorValueObserver = { param, value in
    param.setValue(value, originator: nil)  // ← これが再帰
}

// OK: shadowバッファで管理
var _paramValues = [Float](repeating: 0, count: 12)
tree.implementorValueProvider = { [unowned self] param in
    let addr = Int(param.address)
    guard addr < 12 else { return param.minValue }
    return self._paramValues[addr]  // shadowを直接返す
}
```

### iOS実機デプロイ

```swift
// NG: iOSで使えない
options: [.loadInProcess]

// OK: プラットフォーム分岐
#if os(macOS)
let options: AudioComponentInstantiationOptions = [.loadInProcess]
#else
let options: AudioComponentInstantiationOptions = []
#endif
```

iOSではAUv3は常にout-of-processで動作します。

---

## マルチエージェント開発

コードレビューに **Copilot CLI** と **Codex CLI** を並列で使っています。両者の観点が補完的でした:

| レビュアー | 得意な観点 |
|---|---|
| Copilot CLI | 関数単位の正確性、型安全性、境界条件 |
| Codex CLI | データフロー全体、ビルド構成の整合性、パイプライン連携 |

実際にCopilotが見つけた最重要バグは `compute_loudness_db` の集約が `mean` になっていて（仕様は `sum`）、loudness値が+30dBずれていた問題でした。Codexは独立に、中間WAVの16bit問題（P2-2）とproject.ymlのresourcesブロック漏れ（P2-1）を指摘しました。

```bash
# 両方を並列レビュー
scripts/review.sh --title my-change --base main
# → docs/REVIEW_my-change_{COPILOT,CODEX}.md
```

---

## 現在の状態（Phase 6）

- **テスト**: 77 tests / 0 failures（CoreTests / IntegrationTests / AudioRegressionTests / DDSPStatefulTests）
- **CPU**: 16音ポリフォニー @ 44.1kHz で実時間の1/4未満
- **FX**: Tremolo（bi-phaseステレオ、RhodesSuitcaseスタイル）/ Chorus / Phaser / NeveFX
- **実機**: macOS + iPhone 14 Pro Max + iPhone 15 Pro Max にインストール済み
- **監査修正済み**: セキュリティ監査 3 High / コードレビュー 4 Critical をサブエージェント駆動開発で全対応
- **残**: β評価（5〜10人）→ RC評価（20人）→ Phase 7（codesign / App Store提出）

DDSPの実weightを使った広帯域ゲイン補正は現在stubで、物理モデル単体で動作しています。物理モデル（C++17 Karplus-Strong + ハンマー + ピックアップ + ダンパー）は完成していて、これ単体でも演奏できます。DDSPは「補正のおまけ」であって、なければ音が出ないわけではありません。

---

## まとめ

4回作り直して身についた教訓をまとめます:

1. **「動いてる」≠「設計通り動いてる」** — GRUがサイレントに死んでいた経験から、機能の本質を確認する回帰テストを0日目に書くようにした
2. **CoreML推論はRenderスレッドから絶対に切り離す** — float1個のatom exchangeだけがスレッド境界を越える
3. **レイヤー分離はCIで機械検証しないと崩れる** — 設計書に書くだけでは3回崩れた
4. **FADEはlinspaceで書く** — `arange(n)/n` は最終サンプルが1.0にならない
5. **中間ファイルは最高ビット深度で保持** — ディザは最後の一回だけ

---

## リポジトリ

https://github.com/hakaru/TineModeler4 (private / 公開予定)

---

*2026-05-15時点のPhase 6進行中スナップショット。β評価結果次第で追記予定。*
