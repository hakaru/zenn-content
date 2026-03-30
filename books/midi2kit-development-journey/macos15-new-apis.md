---
title: "macOS 15とmacOS 26.4の新API対応"
---

## Appleが黙って追加したAPI

macOS 15（Sequoia）のSDKヘッダーに、ドキュメント化されていない新しいCoreMIDI APIがいくつか追加されていた。リリースノートには載ってない。ヘッダーを読んで見つけた。

### MIDIUMPEndpointManager

UMPエンドポイント（MIDI 2.0ネイティブデバイス）を列挙するクラス。ファンクションブロック情報、MIDIバージョン、メーカー名が取れる。

macOS 15で追加されたが、**macOS 26.4（Tahoe）でやっとまともに動き始めた**。

### MIDICIDeviceManager

MIDI-CI対応デバイスの一覧を返すクラス。CoreMIDIが認識しているMIDI-CIの参加者がリストアップされる。

ここに罠があった。

## Apple製MIDI-CI参加者の謎

`MIDICIDeviceManager`でリストアップされるデバイスの中に、AppleのmanufacturerSysExID（`0x11 0x00 0x00`）を持つ参加者がいる。

Discoveryブロードキャストには反応する。存在は見える。でもPEリクエストを送っても一切応答しない。タイムアウトするだけ。

macOS自体がMIDI-CI参加者としてネットワークに存在しているらしい。でもResponder機能は実装されていない（か、特定の条件でしか応答しない）。

MIDI2KitではこのmanufacturerIDを自動的にブラックリストに入れて、PEリクエストの送信対象から除外している。公式ドキュメントにはこの挙動の説明が一切ない。

## MIDIUMPMutableEndpoint — 16パターンの実験

macOS 15で自分のアプリをUMPエンドポイントとして登録する`MIDIUMPMutableEndpoint`を使える。ファンクションブロックを登録して、MIDI 2.0ネイティブデバイスとして振る舞う。

macOS 15では全16パターンのパラメータ組み合わせが`Foundation._GenericObjCError code=0`でエラー。全滅。

macOS 26.4ではエラーは消えた。でも16パターン中、実際にファンクションブロックが`MIDIUMPEndpointManager`に表示されるのは**1パターンだけ**。

動く組み合わせ:

| パラメータ | 値 |
|:---|:---|
| direction | `.bidirectional` |
| maxSysEx8Streams | `0` |
| midi1Info | `.notMIDI1` |
| uiHint | `.unknown` |
| markAsStatic | `true` |
| タイミング | `setEnabled(true)`の**前に**registerする |

他の15パターンはエラーなしで黙って失敗する。ファンクションブロック数が0のまま。エラーが返ってくれればまだ対処できるけど、サイレント失敗は厳しい。

## Smart SysEx Sending

macOS 15+で`MIDISendEventList`（UMPネイティブ送信パス）が使える。MIDI 2.0デバイスにはData 64パケットとして送れる。

でもKORG BLE MIDIデバイスはこのパスで壊れる。レガシーの`MIDISend`で送る必要がある。

デバイスによって送信方法を切り替える`SysExSendStrategy`を実装した。

```swift
// UMPネイティブなデバイスかどうかで自動判定
case .auto:
    if destination.isUMPNative {
        sendViaEventList(data)  // Data 64
    } else {
        sendViaLegacy(data)     // MIDISend
    }
```

`.korgBLEMIDI`プリセットは`.legacyOnly`を強制。安全側に倒す。

## MIDI-CI v1.2対応

macOS 15のタイミングでMIDI-CI v1.2への対応も入れた。

v1.1からの変更点:
- PE Notifyのメッセージタイプが`0x38`→`0x3F`に変更
- Management Messages（ACK/NAK）の追加
- Discovery時にCIバージョンを交渉

MIDI2Kitでは全ビルダーに`ciVersion`パラメータを追加。デフォルトは`.v1_1`で後方互換を保ちつつ、Discovery Replyから相手のバージョンを検出して自動切り替え。

## Network MIDI 2.0

macOS 26.4のリリースノートで唯一明記されたCoreMIDIの変更: Network MIDI 2.0（UDP）対応。

MIDI2Kitの受信パス（`MIDIInputPortCreateWithProtocol(._2_0)`）はNetwork MIDI 2.0を透過的に処理できるはず。ただし実際のNetwork MIDI 2.0セッションでのテストはまだできてない。対向のNetwork MIDI 2.0デバイスがない。

## macOS 26.4の評価

総合判定: **PARTIAL-GO**。

動くもの:
- `MIDIUMPMutableEndpoint`（Baseline設定のみ）
- `MIDICIDeviceManager`によるデバイス検出
- `MIDIUMPEndpointManager`によるUMPエンドポイント情報取得
- MIDI-CI v1.2のバージョンネゴシエーション

未検証:
- ファンクションブロックのフルカスタマイズ（15/16パターンがサイレント失敗）
- UMP受信パスでのData 64 SysEx受信
- Network MIDI 2.0セッション上のPE

レガシーAPIは引き続き推奨。macOS 15+の新APIは`@available`ガードで全部囲んでいるので、macOS 14でも問題なく動く。
