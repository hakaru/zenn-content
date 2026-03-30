---
title: "これから"
---

## MIDI 2.0の現在地

2026年3月時点で、MIDI 2.0のエコシステムはこんな感じ。

ハードウェア: KORG Keystage、Roland A-88MKII、StudioLogic SLシリーズ、Rhodes MK8。対応キーボードは増えてきたけど、まだ主流とは言えない。

OS: Appleは2020年（macOS 11 / iOS 14）からUMPサポート。WindowsはWindows MIDI Servicesが2026年2月にGA。両プラットフォームでOS レベルのMIDI 2.0トランスポートが揃った。

DAW: Cubase 14がMIDI 2.0→VST3の変換を実装。Logic ProがProperty Exchangeでオートマッピング。Studio OneがMIDI-CI Discovery。AbletonとFL Studioはまだ。

ライブラリ: 自分が知る限り、MIDI-CIとProperty Exchangeをフルサポートするオープンソースのswiftライブラリは、MIDI2Kitだけ。

## 残ってる課題

### テストデバイスが足りない

KORGでしか検証できてない。Roland、Yamaha、Native Instrumentsの実装がどうなってるかは不明。仕様の解釈がメーカーごとに違う可能性は高い。

MockDeviceに`.rolandStyle`や`.yamahaStyle`プリセットは作ったけど、推測でしかない。実機での検証が必要。

### Network MIDI 2.0の実地テスト

macOS 26.4で追加されたNetwork MIDI 2.0。MIDI2Kitのコードは対応しているはずだけど、実際のネットワークセッション上でPEを動かしたテストはまだない。

### ドキュメント

DocCベースのAPI ドキュメントとインタラクティブチュートリアルを準備中。ライブラリを使う人が増えるにはドキュメントが不可欠。

### サンプルアプリ

iOS用のMIDI 2.0 Device Explorer、macOS用のMIDI Monitor。MIDI2Kitの使い方を示すリファレンス実装としても機能させたい。

## MIDI2Kitの立ち位置

CoreMIDIは「パイプ」を提供する。バイトを送って、バイトを受け取る。

その上にMIDI-CIのDiscovery、Property Exchangeのトランザクション管理、チャンク再構築、タイムアウト、メーカー固有のワークアラウンド——このレイヤーを毎回アプリが実装するのは無理がある。自分がSimpleMidiControllerで2,800行書いて痛感した。

MIDI2Kitはそのレイヤーを引き受ける。メーカーの独自仕様は1箇所に集約されて、全アプリが恩恵を受ける。

## 個人開発の限界と可能性

正直に言うと、MIDI 2.0対応ライブラリを個人で作るのはかなり無茶だった。テストデバイスは手持ちのKORGだけ。仕様書は有料。Appleの未ドキュメントAPIは実機で試すしかない。BLE MIDIのバグはソースコードが見られない。

でもSimpleMidiControllerを作らなかったら、MIDI 2.0の現実の問題には気づけなかった。仕様書を読むだけでは分からないことが多すぎた。

「小さいアプリを作って、壁にぶつかって、その壁を解決するライブラリを作る」。結果的にはいいサイクルだったと思ってる。

https://midi2kit.dev/
https://github.com/hakaru/SimpleMidiController
