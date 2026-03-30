---
title: "MIDI 2.0のProperty Exchangeを実装してKORGキーボードと会話させた話"
emoji: "🎹"
type: "tech"
topics: ["midi", "swift", "ios", "音楽", "個人開発"]
published: true
---

前回の記事でDX7のFM音源エンジンをSwiftで再実装した話を書いた。

https://zenn.dev/hakaru/articles/m2dx-core-dx7-fm-synth-swift

エンジンが動いたので次はMIDIキーボードをつなぎたい。手元にあるのはKORG KeyStage。MIDI 2.0対応のキーボードで、Property Exchange（PE）という仕組みでアプリとキーボードが双方向にデータをやりとりできる。

「キーボードのLCDに音色名を表示したい」。それだけの話のはずだった。

## MIDI 2.0とは何が変わったのか

MIDI 1.0は1983年の規格で、ノートオン/オフとCC（コントロールチェンジ）が7ビット（0〜127）。40年以上これで音楽が動いてきたのは正直すごいけど、さすがに精度が足りない場面が増えてきた。

MIDI 2.0の主な変更点:

- ベロシティが16ビット（0〜65535）に。ピアノの打鍵ニュアンスが段違い
- ピッチベンドが32ビットに。微分音やグリッサンドが滑らかに
- Per-Note Controller。1音ごとにピッチベンドやCCをかけられる（MPEの正式版みたいなもの）
- Property Exchange。デバイス間でJSONデータをやりとりできる
- MIDI-CI（Capability Inquiry）。デバイス同士が能力を自動交渉する

通信フォーマットもUMP（Universal MIDI Packet）に変わった。MIDI 1.0のバイト列ではなく、32ビットワードの固定長パケット。

## MIDI2Kitを選んだ理由

AppleのCoreMIDIはiOS 15からUMPに対応している。ノートオン/オフやCCを受け取るだけならCoreMIDI直接でいい。

問題はProperty ExchangeとMIDI-CI。CoreMIDIにはPEのAPIがない。自分でMIDI-CIのDiscovery、Capability Inquiry、PEのGET/SET/SUBSCRIBEを全部実装する必要がある。

候補は3つあった。

| ライブラリ | MIDI-CI | Property Exchange | Per-Note CC |
|:---:|:---:|:---:|:---:|
| CoreMIDI（直接） | 部分的 | なし | UMPレベルで可 |
| MIDIKit | なし | なし | あり |
| MIDI2Kit | あり | あり（async/await） | UMPレベル |

MIDI2Kitを選んだのはPEサポートがあったから。async/awaitベースのAPIで、`MIDI2Client`を作ってデバイスを発見し、PEでJSONをやりとりできる。

```swift
let client = try MIDI2Client(name: "M2DX")

// デバイス発見を待つ
for await event in client.makeEventStream() {
    if case .deviceDiscovered(let device) = event {
        // PEでデバイス情報を取得
        let info = try await client.getDeviceInfo(from: device.muid)
    }
}
```

## M2DX-CoreのMIDIイベント設計

シンセエンジン（M2DX-Core）自体にはCoreMIDIへの依存を入れたくなかった。ライブラリとして使う側がどのMIDIフレームワークを選ぶかは自由であるべき。

なので、エンジンが受け取るのは独自の`MIDIEvent`構造体。MIDI 2.0の全機能を表現できるようにした。

```swift
public struct MIDIEvent: Sendable {
    public enum Kind: UInt8, Sendable {
        case noteOn, noteOff, controlChange, pitchBend
        case channelPressure, polyPressure
        case perNotePitchBend     // MIDI 2.0: 1音ごとのピッチベンド
        case perNoteCC            // MIDI 2.0: 1音ごとのCC
        case perNoteManagement    // MIDI 2.0: Detach/Reset
        case registeredController // RPN（ピッチベンドレンジ、チューニング等）
        case assignableController // NRPN
    }
    public let kind: Kind
    public let data1: UInt8
    public let data2: UInt32    // 32ビット精度
}
```

ベロシティは`data2`にUInt32で入る。MIDI 1.0の7ビットベロシティはホスト側でビットレプリケーションして16ビットに拡張する。

```swift
// 7bit → 16bit: ビットパターンを繰り返して精度を埋める
let vel16 = (v << 9) | (v << 2) | (v >> 5)
```

単純に `v << 9` だけだと下位ビットが0になって階段状になる。ビットを折り返してコピーすることで滑らかな値になる。MIDI 2.0の公式スケーリング仕様に準拠した方法。

## Per-Note Controllerの実装

MIDI 2.0で個人的に一番面白いのがPer-Note Controller。通常のCCはチャンネル全体にかかるけど、Per-Noteは特定のノートだけに効く。

たとえばCの音だけピッチベンドをかけて、Eはそのまま——みたいなことがMIDI規格レベルでできる。MPE（MIDI Polyphonic Expression）の後継にあたる機能。

エンジン側では、各ボイスにper-noteステートを持たせた。

```swift
// DX7Voice内のper-noteステート
var perNotePitchBendFactor: Float = 1.0
var perNoteVolume: Float = 1.0
var perNoteAftertouch: Float = 0.0
var detached: Bool = false  // チャンネルコントローラから切り離す
```

`detached`フラグはMIDI 2.0のPer-Note Managementメッセージで制御する。detachedなボイスはチャンネルレベルのピッチベンドやLFOの影響を受けなくなる。「この音だけ独立して動かしたい」を実現する仕組み。

## KORGとの対話：Property Exchange

ここからが本題。KORG KeyStageはProperty Exchange対応で、キーボードのLCDにアプリ側の音色名やパラメータを表示できる。仕組みとしてはHTTPに近い。デバイスがJSONでGETリクエストを送ってきて、こちらがJSONで返す。

### 自分をKORGと名乗る

最初のハードル。KeyStageはPE相手の`manufacturerName`を見て、KORG製品にだけKORG独自のリソースをリクエストしてくる。

```swift
// DeviceInfoレスポンスで"KORG"を名乗る
let deviceInfo = """
{"manufacturerName":"KORG",
 "productName":"M2DX DX7 Synthesizer",
 "softwareVersion":"1.0"}
"""
```

さらにMIDI-CIのDevice Identityでも`manufacturerID: .korg`を設定する。

```swift
MIDI2ResponderClient(configuration: .init(
    deviceIdentity: DeviceIdentity(
        manufacturerID: .korg,
        familyID: 0x0001,
        modelID: 0x0001,
        versionID: 0x00010000
    ),
    connectionPolicy: .korgSpecific,
    muid: kM2DXSharedMUID
))
```

これをやらないとKeyStageはX-ProgramEditやX-ParameterListといったKORG独自リソースを要求してこない。MIDI 2.0の仕様上はmanufacturerNameは自由に設定できるフィールドだけど、KORGはこれをゲートに使ってる。

公開仕様には書かれていない。実機を繋いでSysExをスニファして初めて分かった。

### KORG独自のPEリソース

KeyStageがリクエストしてくるリソースは標準のもの（DeviceInfo、ResourceList、ProgramList）に加えて、KORG独自のものがある。

- **X-ProgramEdit** — 現在の音色名とCCの値。KeyStageのLCDに音色名が表示される
- **X-ParameterList** — CCパラメータの定義（名前、CC番号、デフォルト値）。KeyStageのノブに名前が出る
- **parameterListSchema / programEditSchema** — 上2つのJSONスキーマ

X-ProgramEditの音色名フォーマットは`"1:E.PIANO 1"`のように1-based番号＋コロン＋名前。KORG Moduleと同じ形式にしないとKeyStageのLCDで正しく表示されない。これも仕様書には書いてない。KORG ModuleとKeyStageの通信をスニファして突き止めた。

### PE Notifyのエコーバック問題

一番ハマったバグ。KeyStageからCCを受信すると、アプリ側のパラメータが更新される。パラメータが更新されるとPE Notifyでサブスクライバーに通知する——つまりKeyStageに送り返す。KeyStageがそれを受けてまたCCを送る。無限ループ。

実際にはループにはならないけど、KeyStageのPEプロセッサが処理しきれなくなってLCDがフリーズする。

```swift
// ❌ MIDIで受けたCCをそのままPE Notifyで返すとLCDフリーズ
func onMIDICCReceived(cc: UInt8, value: UInt32) {
    updateParameter(cc, value)
    sendPENotify()  // これがKeyStageを殺す
}

// ✅ MIDI経由のCC変更はNotifyしない。UI操作のみNotify
func syncCC(_ cc: UInt8, _ value: UInt32) {
    updateParameter(cc, value)
    // PE Notifyは送らない
}

func updateCCFromUI(_ cc: UInt8, _ value: UInt32) {
    updateParameter(cc, value)
    sendPENotify()  // UI操作のみKeyStageに通知
}
```

### Discovery再送の必要性

KORG KeyStageはMIDI-CIのDiscovery Inquiry（sub-ID2: 0x70）を送ってくることがある。このとき、こちらもDiscoveryを再送しないとPEセッションが確立しない。

```swift
// KORGからDiscoveryが来たら200ms待って再送
if subID2 == 0x70 {
    try? await Task.sleep(for: .milliseconds(200))
    await ci.sendDiscoveryInquiry()
}
```

200msの待ちが必要な理由は正直よく分からない。即座に返すとKeyStageが無視する。タイミングの問題だと思うけど、仕様には記載がない。

### 固定MUIDでセッション永続化

MIDI-CIではデバイスごとにMUID（28ビットの識別子）が割り当てられる。通常はランダム生成だけど、M2DXでは固定値を使っている。

```swift
private let kM2DXSharedMUID = MUID(rawValue: 0x5404629)!
```

KeyStageは前回接続したMUIDを覚えていて、同じMUIDなら再Discoveryなしでセッションを再開する。ランダムMUIDだと毎回ゼロからネゴシエーションが走る。

### KORGはPE Initiator-only

これが一番大きな発見。KORGデバイスはPEのInitiator（リクエストを送る側）としてのみ動作する。こちらからPE GETリクエストを送っても応答がない。

```
PE: All dests tried, no response. KORG likely PE Initiator-only.
PE: (KORG queries US, but doesn't respond to PE GET)
```

つまりKORGはこちらのリソースを読みに来るけど、KORGのリソースはこちらから読めない。MIDI 2.0のPE仕様では双方向が前提だけど、現時点のKORG実装はInitiator-onlyということ。

自分がテストできたのはKORG KeyStageだけ。他のメーカーがどう実装してるかは分からない。

## BLE MIDIの信頼性問題

KORG Module ProとBLE MIDI経由でPEをやる場合、さらに問題が増える。BLEはパケットロスが起きる。

ResourceList（利用可能なリソース一覧）は複数チャンクに分割されて送られるので、途中のパケットが落ちるとレスポンス全体が壊れる。

MIDI2Kitの`.korgBLEMIDI`プリセットで対策してる。

```swift
let client = try MIDI2Client(name: "MyApp", preset: .korgBLEMIDI)
```

やっていること:
- PEタイムアウトを10秒に延長（デフォルトより長い）
- ResourceListの前にDeviceInfoを送る（ウォームアップ。BLE接続を安定させる）
- リトライ3回、リトライ間隔500ms
- 同時リクエスト数を1に制限（BLEの負荷を下げる）

それでもResourceListが失敗することがある。その場合はResourceListをスキップして、既知のリソースに直接アクセスする。

```swift
// ResourceListがタイムアウトしたら既知のリソースにフォールバック
let resources = [
    "DeviceInfo",
    "CMList",
    "ProgramList"
]
```

## オーディオスレッドへのMIDI配送

MIDIイベントをシンセエンジンに渡すのもひと工夫必要だった。オーディオレンダリングスレッドではロック禁止。

ロックフリーのSPSCリング（Single Producer Single Consumer）でUIスレッドからオーディオスレッドにイベントを流す。

```swift
// 256イベント分のFIFOリングバッファ
let midiRing = SPSCRing<MIDIEvent>(capacity: 256)

// UIスレッド → push
midiRing.push(event)

// オーディオスレッド → drain
while let event = midiRing.pop() {
    processEvent(event)
}
```

`Synchronization.Atomic`のacquire/releaseオーダリングだけで同期。ミューテックスもCASも使わない。

処理順序も大事で、最初はイベントを逆順に処理するバグがあってノートが張り付いた。サスティンペダルの状態がノート処理より後に反映されると、ペダルオフ→ノートオフの順序が狂う。今は2パスで処理してる。パス1でCCとピッチベンド、パス2でノートイベント。

## メーカーごとの「秘密」

MIDI 2.0は規格としてはオープンなんだけど、メーカー独自の実装がかなりある。

自分がKORGで見つけたもの:
- manufacturerNameで相手を判定して独自リソースの公開を切り替える
- X-ProgramEdit、X-ParameterListなどの非標準リソース
- 音色名のフォーマット（1-based番号＋コロン＋名前）
- PE Initiator-only（Responder機能がない）
- Discovery再送の200msタイミング依存

これらは公開仕様書には書かれていない。実機を繋いでSysExをスニファして、KORG ModuleとKeyStageの通信を傍受して突き止めた。MIDI2KitにはPEスニファモードを実装してあって、デバイス間の通信をパッシブに監視できる。

他のメーカー（Roland、Yamaha、Native Instruments等）がどうしてるかはテストできてない。MIDI 2.0対応のハードウェア自体がまだ少ないし、手元にKORG以外のMIDI 2.0デバイスがない。

MIDI 2.0は「規格」と「実装」のギャップがまだ大きい。MIDI 1.0のときも似たような過渡期があったんだろうけど、PEやMIDI-CIのような高レベルプロトコルが絡むと互換性の問題はMIDI 1.0時代より複雑になる。

## おわりに

「KeyStageのLCDに音色名を出したい」から始まって、MIDI-CIのDiscovery、Property Exchangeの実装、KORGの独自仕様のリバースエンジニアリング、BLE MIDIの信頼性対策まで。MIDIキーボードを繋ぐだけでこんなに深い穴があるとは思わなかった。

MIDI2Kitのドキュメントとソースコードは公開してある。MIDI 2.0のPEを実装したい人の参考になれば。

https://midi2kit.dev/

M2DX-Core（DX7 FMシンセエンジン）:
https://github.com/hakaru/M2DX-Core
