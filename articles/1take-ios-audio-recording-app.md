---
title: "ミュージシャン向け録音アプリ「1Take」を個人開発した話"
emoji: "🎙️"
type: "tech"
topics: ["ios", "swift", "swiftui", "audio", "個人開発"]
published: true
---

自分はピアノを弾くんだけど、練習の録音でずっと不満があった。

- iPhoneのボイスメモだと音がショボい
- DAWを立ち上げるのは練習にはオーバーキル
- 既存の録音アプリはエフェクトが後処理ばかりで、「録りながら良い音」にならない

「録音ボタン1つで、あとは勝手にいい感じにしてくれるアプリが欲しい」。なかったので作った。

https://apps.apple.com/app/id6738847498

## リアルタイムエフェクトチェーン

一般的な録音アプリは録った後に加工する。1Takeは**録音中にリアルタイムでエフェクトを適用して、結果をそのままファイルに焼き付ける**。

```
Input → Trim → Noise Gate → 4-band EQ → Compressor × 2段
→ Saturation → M/S Processor → Limiter → Output
```

DAWのインサートチェーンと同じ構成をiOS上で動かしてる。

## プリセット

プリセットを選ぶだけで、スタジオ機材のキャラクターが乗る。

| プリセット | モデル | コンプの特性 | おすすめシーン |
|:---:|:---:|:---:|:---:|
| Flat | なし | 無加工 | DAW素材録り |
| Studio Light | LA-2A | Opto / スムーズ | ピアノ・アコギ |
| Studio Heavy | 1176 | FET / パンチ | バンド・ロック系 |
| Live | API 2500 | VCA / 安定 | ライブ・大音量環境 |

「LA-2Aの音」「1176のパンチ感」と聞けばピンとくる人も多いと思う。それがiPhoneのマイクから直接得られる。

## 4モードメーター

録音画面のメーター部分をスワイプすると4種類のモードを切り替えられる。

- VU（300msバリスティクスの伝統的なやつ）
- GR（コンプレッサーの効き具合をリアルタイム表示）
- スペクトラムアナライザー（FFT周波数表示、EQカーブのオーバーレイ付き）
- LUFS（ITU-R BS.1770-4準拠。Momentary / Short-term / Integrated / True Peak）

レベルメーターしかない録音アプリが多い中、ここはこだわった。

## AI自動最適化

録音が終わるとAIが自動解析して、**次回の録音設定を調整してくれる**。

- クリッピングが起きた → Input Trimを下げる
- ノイズが多かった → Noise Gateを自動ON
- 低域がモコモコ → Low-Cutフィルターを有効化
- 音が小さすぎた → ゲインとリミッターを調整

オーディオの知識がなくても、使うたびに勝手に良くなっていく。調整内容は詳細シートで確認できるし、ワンタップで元に戻せる。

## プリロール（遡り録音）

録音ボタンを押す**最大30秒前**の音声を自動キャプチャする。循環バッファで常にバックグラウンドで保持してるので、「今のよかったのに録ってなかった！」がなくなる。

## クラウド自動同期

iCloud Drive、Dropbox、Google Drive、Backblaze B2の4つに対応。録音が終わったら自動でアップロード。設定でWi-Fi制限もかけられる。

B2が従量制で安いので個人的には推してる。詳しくは別記事に書いた。

https://zenn.dev/hakaru/articles/1take-cloud-sync-backblaze-b2

## 技術スタック

| 項目 | 技術 |
|:---|:---|
| 言語 | Swift 6.1（Strict Concurrency） |
| UI | SwiftUI（MVパターン / ViewModelなし） |
| オーディオ | AVAudioEngine + Accelerate/vDSP |
| データ | SwiftData |
| パッケージ管理 | Swift Package Manager |
| 課金 | StoreKit 2（買い切り） |

### ViewModelを使わない理由

SwiftUIの`@Observable` + `@State` + `@Environment`だけで十分クリーンに書ける。ViewModelは不要な抽象レイヤーを増やすだけだと思ってる。

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

True Peak検出にはEBU Tech 3341準拠の4倍オーバーサンプリング + 48タップポリフェーズFIRフィルターを実装した。`Accelerate`フレームワークのvDSPを使って、リアルタイム処理でもCPU負荷は最小限。

SwiftUIのライフサイクル問題（`onAppear`/`onDisappear`が予期しない順序で呼ばれる）への対策として、スペクトラムアナライザーの共有リソースには参照カウント方式を採用。

```swift
// ❌ SwiftUIのライフサイクルで競合する
.onAppear { service.spectrumEnabled = true }
.onDisappear { service.spectrumEnabled = false }

// ✅ 参照カウントで安全に管理
.onAppear { service.acquireSpectrumAnalysis() }
.onDisappear { service.releaseSpectrumAnalysis() }
```

### GCDゼロ

Swift 6のStrict Concurrency Modeを全面採用。`DispatchQueue`は一切使わず、`async/await` + `actor` + `@MainActor`のみ。コンパイラがデータ競合を教えてくれるので、ランタイムクラッシュがほぼゼロになった。面倒だけど正しい選択だったと思う。

## 苦労した話

一番きつかったのはリアルタイムオーディオのスレッド制約。オーディオスレッドではメモリアロケーション禁止なので、バッファプールで事前確保して対処した。

LUFS計算もITU-R BS.1770-4の仕様書を読みながらKフィルター + ゲーティングを実装。仕様書が正しいのか自分の実装が正しいのか分からなくなる時間が長かった。

あとSwiftUIとオーディオの共存。UIの再描画がオーディオ処理に干渉しないよう、責務の分離にはかなり気を使った。

## ビジネスモデル

無料で全機能使える。Proは買い切りで、WAV/BWF 24bitエクスポートとプリセットカスタマイズが追加される。サブスクは採用しなかった。録音アプリに月額課金は合わない。広告もなし。

## おわりに

個人開発でここまでオーディオ処理を深掘りしたのは初めてだった。CoreAudioのドキュメントとにらめっこする日々はしんどかったけど、自分の練習録音が明らかに良い音になった瞬間は嬉しかった。

ミュージシャンの方、よかったら使ってみてください。

https://apps.apple.com/app/id6738847498
