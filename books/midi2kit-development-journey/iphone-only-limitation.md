---
title: "iPhoneでしか動かない問題"
---

## Inter-App MIDI-CIの壁

MIDI2Kitのコードは動く。テストも通る。MockDeviceでは完璧に動く。

じゃあ実機は？と思って同じiPhone上でMIDI2Kitアプリ（Initiator）とKORG Module Pro（Responder）を動かしてみた。

動かない。

## なぜ動かないか

CoreMIDIのVirtual Portを作ると、同じデバイス上の他のアプリからMIDIを送受信できる。MIDI 1.0のノートやCCはVirtual Port経由で問題なく届く。

ところがKORG Module ProはVirtual Portに届いたMIDI-CIメッセージを無視する。Discovery Inquiryを送っても、15秒待っても、Discovery Replyが返ってこない。

KORG Module ProのMIDI-CI実装は**BLE MIDIインターフェースにのみバインドされている**。Virtual Portは対象外。

## Bluetoothの物理的制約

BLE MIDIはBluetoothの通信。同じデバイス上の2つのアプリがBLE MIDIで通信するのは、物理的に不可能。Bluetoothは外部デバイスとの通信プロトコルであって、ローカルのプロセス間通信には使えない。

つまり:

- iPhone A（MIDI2Kitアプリ）↔ iPhone B（KORG Module Pro）: BLE MIDI経由で動く
- iPhone A（MIDI2Kitアプリ）↔ iPhone A（KORG Module Pro）: Virtual Portは無視される。BLEは物理的に不可能

## 2台構成が必須

現時点でKORG Module ProとMIDI-CI/PEで通信するには、iPhoneが2台必要。片方にMIDI2Kitアプリ、もう片方にKORG Module Pro。BLE MIDIで接続。

1台のiPhoneで完結しない。これはユーザーにとってかなりハードルが高い。

## MIDI2Kitアプリ同士なら動く

ただし、両方のアプリがMIDI2Kitを使っている場合は話が変わる。

MIDI2KitのCIManagerはVirtual Portも監視する設定が可能。両方のアプリがVirtual Port上のMIDI-CIに対応していれば、同じデバイス上でMIDI-CIの全フローが動く。

問題はKORG Module ProのようなサードパーティアプリがVirtual Port上のMIDI-CIを実装していないこと。これはKORGの設計判断であって、MIDI 2.0仕様の制約ではない。

## 将来の展望

macOS/iPadOS上ではUSB MIDIやNetwork MIDI 2.0（macOS 26.4で追加）も使える。BLE MIDIの制約はワイヤレス固有の問題。

USB MIDI接続のKORG KeyStageではMIDI-CIが動く（実機確認済み）。ただしKeyStageはInitiator側なので、KeyStageからM2DXに問い合わせが来る形。こちらからKeyStageにPE GETを送る用途では、やはりInitiator-only問題にぶつかる。

デバイスメーカーがVirtual Port上のMIDI-CIを有効にしてくれれば解決するけど、そこはメーカー次第。仕様書には「Virtual PortでMIDI-CIをサポートすべき」とは書いてない。
