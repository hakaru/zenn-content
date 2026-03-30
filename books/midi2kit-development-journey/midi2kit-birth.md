---
title: "MIDI2Kitの設計と実装"
---

## SimpleMidiControllerの2,800行を捨てる

SimpleMidiControllerに直書きしていたMIDI-CI/PEスタック2,800行を削除して、ライブラリとして再設計する。3フェーズで進めた。

- Phase 1: MIDI2Kitを依存に追加、旧コードと並行稼働
- Phase 2: アプリのビューをMIDI2Kit APIに切り替え
- Phase 3: 旧コード2,800行を削除

Phase 3のコミット（`b4ae272`）で旧実装を全削除。最終的にSimpleMidiControllerから`import CoreMIDI`が消えた。アプリはMIDI2Kitの純粋なコンシューマーになった。

## 4層アーキテクチャ

```
MIDI2Kit     — CIManager, PEManager（高レベルasync API）
MIDI2CI/PE   — Discovery, PE chunking/subscriptions
MIDI2Transport — MIDITransport protocol, CoreMIDI/Mock/Loopback
MIDI2Core    — UMP型, MUID, DeviceIdentity, Mcoded7
```

一番大事な設計判断は**Transport層をプロトコルにした**こと。

```swift
protocol MIDITransport: Sendable {
    func send(_ data: [UInt8], to destination: MIDIDestination) throws
    func makeReceiveStream() -> AsyncStream<MIDIReceivedData>
}
```

これで3種類のTransportが差し替えられる。

- `CoreMIDITransport` — 本番用。実際のMIDIポートに送受信
- `MockMIDITransport` — ユニットテスト用。送受信を記録
- `LoopbackTransport` — ペアで使う。一方の送信が他方の受信になる。同一プロセス内でInitiatorとResponderをテストできる

## Actor-based Concurrency

`CIManager`と`PEManager`はSwiftのactorとして実装。ロックやミューテックスなしでスレッドセーフ。

```swift
actor CIManager {
    private var discoveredDevices: [MUID: DiscoveredDevice] = [:]

    func startDiscovery() async { ... }
    func handleDiscoveryReply(_ reply: DiscoveryReply) { ... }
}
```

イベント配信は`AsyncStream`。デリゲートパターンは使わない。

```swift
for await event in client.makeEventStream() {
    switch event {
    case .deviceDiscovered(let device):
        // デバイス発見
    case .deviceLost(let muid):
        // デバイス消失
    case .propertyChanged(let muid, let resource):
        // PEプロパティ変更
    }
}
```

## MockDeviceでハードウェアなしテスト

実機がなくてもMIDI-CIの全フローをテストできるMockDeviceを作った。

```swift
let (initiatorTransport, responderTransport) = LoopbackTransport.createPair()
let mock = MockDevice(transport: responderTransport, preset: .korgModulePro)
let client = MIDI2Client(transport: initiatorTransport)

// KORGデバイスのように振る舞うモックに対してPEリクエスト
let info = try await client.getDeviceInfo(from: mock.muid)
```

プリセットは5種類: `.korgModulePro`, `.generic`, `.rolandStyle`, `.yamahaStyle`, `.minimal`。

KORGの独自仕様（Mcoded7不使用、文字列型canGet、X-ProgramEdit）もモックに再現してあるので、KORGとの互換性問題をハードウェアなしで検出できる。

## テスト

705テスト、77テストスイート。ローンチ時（2026年3月8日）の602から増え続けてる。

LoopbackTransportのおかげで、Discovery→PE Capability→ResourceList→GET/SETの全フローをユニットテストで回せる。タイムアウト、NAK、チャンクロスト、不正なJSON、Mcoded7エンコーディングエラーなど、エッジケースをすべてテストしている。

## 16秒 → 144ms

SimpleMidiControllerのKORGとのPE通信は初回接続に約16秒かかっていた。ウォームアップ＋ResourceList＋個別リソース取得の直列実行。

MIDI2Kitに移行後、KORGのmanufacturerNameを検出したらResourceListをスキップしてX-ParameterListに直接アクセスする最適化を入れた。

16.4秒 → 144ms。99.1%の改善。ライブラリレベルで最適化できるから、全てのアプリが恩恵を受ける。
