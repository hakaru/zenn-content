---
title: "BLE MIDIの不可解な世界"
---

## BLE MIDIでProperty Exchangeをやる狂気

BLE MIDI（Bluetooth Low Energy MIDI）は便利。ケーブルなしでiPhoneとキーボードが繋がる。

MIDI 1.0のノートオン/オフを送る分には問題ない。レイテンシは若干あるけど実用レベル。

ところがProperty Exchangeの大きなJSONデータをBLE経由で送ると、話が変わる。

## パケットロスとチャンク再構築

PEのレスポンスが大きい場合、複数のSysExチャンクに分割される。ResourceList（デバイスが持つリソース一覧）は典型的なマルチチャンクレスポンス。

BLEはパケットロスが起きる。途中の1チャンクが落ちたら、レスポンス全体が壊れる。タイムアウトを待つしかない。

しかも再送機構がない。PEにはリトライの概念がない（正確には仕様書に記載がない）。チャンクが1つ落ちたら最初からやり直し。

## iPadだけ壊れる

前の章で触れたけど、iPad（iPad14,10 / iOS 18.6.2）でResourceListのチャンク2を受信すると、バイト338〜342が毎回バイナリのゴミに化ける。100%再現。

iPhoneでは同じコード、同じデバイス、同じBLE接続で完璧に動く。

- CoreMIDIのBLE MIDIドライバのiPad固有バグ？
- KORG Module ProのiPad版の送信バグ？
- BLEのMTU（Maximum Transmission Unit）がiPadで異なる？

切り分けできなかった。CoreMIDIのソースコードは見られないし、KORGのファームウェアも見られない。再現手順をまとめてKnown Limitationsに記録するしかなかった。

## ウォームアップの必要性

BLE MIDI接続直後にResourceListを要求すると、高確率で失敗する。

DeviceInfo（単一チャンクの小さいレスポンス）を先に1回取得してからResourceListを要求すると、成功率が大幅に上がる。

「ウォームアップ」と呼んでいる。BLE接続が安定するまでの時間稼ぎ。なぜこれが必要なのか、正確な理由は分からない。BLEのコネクションパラメータのネゴシエーションが完了するまでの遅延か、KORG Module Pro側のBLEスタックの初期化タイミングか。

MIDI2Kitの`.korgBLEMIDI`プリセットにはこのウォームアップが組み込まれている。

```swift
let client = try MIDI2Client(name: "MyApp", preset: .korgBLEMIDI)
// ↑ 内部でDeviceInfoを先に取得してからResourceListをリクエストする
```

## タイムアウト設計

BLE MIDIのレイテンシは不安定。100msで返ることもあれば3秒かかることもある。

デフォルトのPEタイムアウト5秒だと、BLE経由のマルチチャンクレスポンスは間に合わないことがある。`.korgBLEMIDI`プリセットでは10秒に延長している。

同時リクエスト数も制限する。BLEのリンクに負荷をかけると、パケットロス率が上がる。`maxInflightPerDevice = 1`で直列化している。

## ResourceListを諦める

それでもResourceListが取れないことがある。

最終手段: ResourceListをスキップして、既知のリソースに直接アクセスする。

```swift
// ResourceListがタイムアウトしたら既知のリソースにフォールバック
let knownResources = ["DeviceInfo", "CMList", "ProgramList"]
```

KORGのリソース構成は分かっているので、一覧を取得しなくても直接GETできる。汎用性は落ちるけど、動く。

## SysEx送信パスの罠

macOS 15以降では`MIDISendEventList`（UMPネイティブの送信パス）が使えるようになった。新しいAPIだし、使いたくなる。

だがKORG BLE MIDIデバイスはこのパスで送ったSysExを正しく処理できない。レガシーの`MIDISend`（MIDI 1.0パケットリスト）で送る必要がある。

MIDI2Kitでは`SysExSendStrategy`という列挙型で制御している。

```swift
enum SysExSendStrategy {
    case legacyOnly  // MIDISend（安全、全デバイス互換）
    case auto        // UMPネイティブならMIDISendEventList、それ以外はlegacy
    case umpOnly     // 強制的にMIDISendEventList
}
```

`.korgBLEMIDI`プリセットは`.legacyOnly`を強制。UMP送信パスのKORGとの互換性が確認できるまで変えない。

## BLE MIDIの教訓

BLE MIDIは「繋がるだけ」なら簡単。その上でMIDI-CIやProperty Exchangeのような複雑なプロトコルを動かすのは、想定されている使い方ではなかったのかもしれない。

でもKORG KeyStageはまさにそれをやってる。ケーブルレスでPEの全機能を使う。だから対応しないわけにはいかない。

対策は全部ヒューリスティック。「なぜ動くのか」より「どうすれば動くか」を優先せざるを得なかった。
