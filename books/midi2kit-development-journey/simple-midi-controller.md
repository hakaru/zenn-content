---
title: "SimpleMidiController — 最初のアプリ"
---

## 作りたかったもの

iPhoneをMIDIコントローラーにするアプリ。3つの画面がある。

- スライダーページ — CC1（モジュレーション）、CC7（ボリューム）、CC10（パン）などを6本のスライダーで操作
- プログラムチェンジページ — バンクセレクト付きでPC 0〜127を送信
- XYパッド — 2つのXYパッドで任意のCCを2軸操作

ここまではMIDI 1.0の世界。CoreMIDIで`[0xB0 | channel, cc, value]`のバイト列を送るだけ。SwiftUI + CoreMIDIで普通に書ける。

## MIDI 2.0に手を出す

KORG KeyStageを繋いだときに気づいた。KeyStageはMIDI 2.0のProperty Exchange対応で、接続先のアプリに音色名やパラメータ情報を問い合わせてくる。こちらが応答すればKeyStageのLCDにアプリの音色名が表示される。ノブにパラメータ名も出る。

「これ対応したらめちゃくちゃ便利じゃん」と思って手を出した。

## 手で書いたMIDI-CIスタック

MIDI 2.0のProperty Exchangeを使うには、まずMIDI-CI（Capability Inquiry）の実装が必要。デバイス同士が自己紹介して、何ができるかを交渉するプロトコル。

手で全部書いた。7ファイル、約2,800行。

1. Discovery — CI Discovery Inquiryをブロードキャスト、返答からMUID（デバイスID）を抽出
2. PE Capability Inquiry — 相手がProperty Exchangeに対応してるか確認
3. ResourceList取得 — 相手が持ってるリソース一覧を取得
4. 個別リソースのGET — DeviceInfo、ProgramList、パラメータリストなど

SysExメッセージの組み立てと解析、チャンク分割されたレスポンスの再構築、タイムアウト管理、リクエストID（7ビット、0〜127）の割り当て。全部アプリ内に直書き。

動いた。KORG Module Proから音色名が取れた。KeyStageのLCDに表示された。

そしてバグが出始めた。

## 壊れ始める

### SysExの順序が壊れる

CoreMIDIの`MIDIPacketList`には複数のパケットが入ることがある。最初の実装では1パケットごとに`Task`を生成して非同期で処理していた。Swiftの`Task`は順序保証がないので、SysExのフラグメントが入れ替わる。

```swift
// ❌ パケットごとにTaskを生成 → 順序が壊れる
for packet in packetList {
    Task { await assembler.process(packet) }
}

// ✅ PacketList全体を1回のawaitで処理
await assembler.processPacketList(packetList)
```

### MUIDが毎回変わる

`startDiscovery()`を呼ぶたびに新しいMUIDを生成していた。進行中のPEトランザクションのレスポンスが古いMUID宛てに届いて、受け取れない。

修正: MUIDはinit時に1回だけ生成。

### iPadでPEデータが壊れる

iPad（iPad14,10 / iOS 18.6.2）でResourceListのレスポンスを受け取ると、チャンク2のバイト338〜342が毎回バイナリのゴミに化ける。100%再現。同じコードがiPhoneでは完璧に動く。

CoreMIDIのBLE MIDIドライバのバグか、KORG Module ProのiPad固有の送信バグか。切り分けできないまま、「iPhoneを推奨」として既知の制限に分類した。

### リクエストIDの枯渇

PEトランザクションが失敗したとき（NAKや無応答）、トランザクションオブジェクトが解放されない。リクエストIDは7ビットで128個しかないので、長時間使うと枯渇する。

### GCDとSwift Concurrencyの混在

`MIDICIManager`の中で`DispatchQueue`と`@MainActor`が共存していて、Swift 6のStrict Concurrency Checkingでデータ競合の警告が出る。

## 「これ、ライブラリにしないとダメだ」

2,800行のCI/PEコードがアプリに直接埋まっていて、再利用もテストもできない。バグを直すたびに「これアプリ固有の問題じゃなくて、プロトコル実装の問題だ」と気づく。

2026年1月16日、設計会議を開いた（自分とAIで）。議事録の冒頭にこう書いてある:

> SimpleMIDIController には CoreMIDI 直叩き、SysEx 組み立て、MIDI-CI Discovery、Property Exchange の transaction/chunk/timeout 管理が同居しており、再利用・テスト・保守が難しい。

ここからMIDI2Kitが始まった。
