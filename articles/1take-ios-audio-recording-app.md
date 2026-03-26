---
title: "ミュージシャン向け録音アプリ「1Take」を個人開発した話"
emoji: "🎙️"
type: "tech"
topics: ["ios", "swift", "swiftui", "audio", "個人開発"]
published: true
---

## 概要

ミュージシャンが練習やリハーサルを「ワンタップでプロ品質の音」で録音できるiOSアプリ **1Take** を作りました。

https://apps.apple.com/us/app/1take/id6757945099

「録音ボタンを押すだけで、スタジオ機材を通したような音で録れる」がコンセプトです。

## 作った動機

自分自身がピアノを弾くのですが、練習の録音で毎回こう思っていました。

- iPhoneのボイスメモだと音がショボい
- DAWを立ち上げるのは練習にはオーバーキル
- 既存の録音アプリはエフェクトが後処理ばかりで「録りながら良い音」にならない

**「録音ボタン1つで、あとは勝手にいい感じにしてくれるアプリが欲しい」** — これが開発の動機です。

## 特徴

### 1. リアルタイムエフェクトチェーン

一般的な録音アプリは録った後に加工しますが、1Takeは **録音中にリアルタイムでエフェクトを適用し、結果をそのままファイルに焼き付けます**。

```
Input → Trim → Noise Gate → 4-band EQ → Compressor × 2段
→ Saturation → M/S Processor → Limiter → Output
```

DAWのインサートチェーンと同じ構成をiOS上で実現しています。

### 2. 名機をモデルにしたプリセット

プリセットを選ぶだけで、スタジオ機材のキャラクターが適用されます。

| プリセット | モデル | コンプの特性 | おすすめシーン |
|:---:|:---:|:---:|:---:|
| **Flat** | — | 無加工 | DAW素材録り |
| **Studio** | LA-2A | Opto / スムーズ | ピアノ・アコギ |
| **Studio+** | 1176 | FET / パンチ | バンド・ロック系 |
| **Live** | API 2500 | VCA / 安定 | ライブ・大音量環境 |

音楽をやる人なら「LA-2Aの音」「1176のパンチ感」と聞けばピンとくるはずです。それがiPhoneのマイクから直接得られます。

### 3. プロ仕様の4モードメーター

録音画面のメーター部分をタップすると4種類のモードを切り替えられます。

- **VU** — 300msバリスティクスの伝統的VUメーター
- **GR（Gain Reduction）** — コンプレッサーの効き具合をリアルタイム表示
- **スペクトラムアナライザー** — FFTによる周波数表示。EQカーブもオーバーレイ
- **LUFS** — ITU-R BS.1770-4準拠のラウドネスメーター。Momentary / Short-term / Integrated / True Peak

「レベルメーターだけ」の録音アプリとは一線を画す、エンジニア向けの情報量です。

### 4. AI自動最適化

録音完了後にAIが自動解析し、**次回の録音設定を自動調整** します。

- クリッピングが起きた → Input Trimを自動で下げる
- ノイズが多かった → Noise Gateを自動ON
- 低域がモコモコ → Low-Cutフィルターを有効化
- 音が小さすぎた → ゲインとリミッターを自動調整

ユーザーがオーディオの専門知識を持っていなくても、使うたびに録音品質が向上していく仕組みです。調整内容は詳細シートで確認でき、ワンタップで元に戻せます。

### 5. プリロール（遡り録音）

録音ボタンを押す **最大5秒前** の音声を自動キャプチャ。循環バッファで常にバックグラウンドで保持しているため、「あ、今のよかったのに録音してなかった！」がなくなります。

### 6. スマートマーカー

- **CLIPマーカー** — 0 dBFS超え時に自動挿入
- **バッファドロップマーカー** — 音切れ箇所を自動記録
- **手動マーカー** — 録音中にタップで挿入（5秒遡及機能付き）

再生時にマーカーをタップすれば該当箇所へ即ジャンプ。長い練習録音の中から「あの瞬間」を素早く見つけられます。

### 7. BWF（Broadcast Wave Format）対応

Pro版ではiXMLメタデータ付きのBWFエクスポートに対応。デバイス情報・エフェクト設定・マーカーなどがファイルに埋め込まれ、DAWでの後作業がスムーズになります。

## 技術スタック

| 項目 | 技術 |
|:---|:---|
| 言語 | Swift 6.1（Strict Concurrency） |
| UI | SwiftUI（MV パターン / ViewModel不使用） |
| オーディオ | AVAudioEngine + Accelerate/vDSP |
| データ | SwiftData |
| パッケージ管理 | Swift Package Manager |
| アーキテクチャ | Workspace + SPM Package構成 |
| 課金 | StoreKit 2（買い切り） |
| CI | GitHub Actions |

### なぜViewModelを使わないのか

SwiftUIの `@Observable` + `@State` + `@Environment` だけで十分にクリーンなコードが書けます。ViewModelは不要な抽象レイヤーを増やすだけだと判断しました。

```swift
struct RecordingView: View {
    @Environment(AudioEngine.self) private var engine
    @State private var viewState: ViewState = .ready

    var body: some View {
        // Viewは状態の純粋な表現
    }
}
```

### オーディオ処理の工夫

**True Peak検出** にはEBU Tech 3341準拠の4倍オーバーサンプリング + 48タップポリフェーズFIRフィルターを実装。`Accelerate` フレームワークのvDSPを活用し、リアルタイム処理でもCPU負荷を最小限に抑えています。

**SwiftUIのライフサイクル問題** への対策として、スペクトラムアナライザーなどの共有リソース制御には **参照カウント方式** を採用。`onAppear` / `onDisappear` の予期しない呼び出し順序でもリソースが正しく管理されます。

```swift
// ❌ SwiftUIのライフサイクルで競合する
.onAppear { service.spectrumEnabled = true }
.onDisappear { service.spectrumEnabled = false }

// ✅ 参照カウントで安全に管理
.onAppear { service.acquireSpectrumAnalysis() }
.onDisappear { service.releaseSpectrumAnalysis() }
```

### GCDゼロ

Swift 6のStrict Concurrency Modeを全面採用。`DispatchQueue` は一切使わず、`async/await` + `actor` + `@MainActor` のみで並行処理を実装しています。

## ビジネスモデル

- **無料**: MP3/AACエクスポート、全プリセット、全メーター、AI最適化
- **Pro（買い切り）**: WAV/BWF 24bitエクスポート、プリセットカスタマイズ

サブスクリプションは採用しませんでした。録音アプリに月額課金は合わないという判断です。広告も一切ありません。

## 開発を振り返って

個人開発でここまでオーディオ処理を深掘りしたのは初めてでした。特に苦労したのは：

1. **リアルタイムオーディオのスレッド制約** — オーディオスレッドではメモリアロケーション禁止。バッファプールで事前確保して対処
2. **LUFS計算の精度** — ITU-R BS.1770-4の仕様書を読みながらKフィルター + ゲーティングを実装
3. **SwiftUIとオーディオの共存** — UIの再描画とオーディオ処理が干渉しないよう、責務を明確に分離

逆に、Swift 6のStrict Concurrencyは「面倒だけど正しい」選択でした。コンパイラがデータ競合を教えてくれるので、ランタイムクラッシュがほぼゼロになりました。

## まとめ

1Takeは「録音ボタンを押すだけ」のシンプルさの裏側に、プロオーディオの技術をできる限り詰め込んだアプリです。

ミュージシャンの方、ぜひ試してみてください。フィードバックもお待ちしています。

https://apps.apple.com/us/app/1take/id6757945099
