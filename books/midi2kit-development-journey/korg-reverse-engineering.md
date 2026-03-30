---
title: "KORGのリバースエンジニアリング"
---

## スニファで盗聴する

KORGの独自仕様を解明するには、KORGデバイス同士の通信を覗くのが一番早い。

MIDI2Kitにはデバッグ用のPEスニファモードがある。PE Responderを無効化して、MIDI-CIのSysExを全部パッシブに記録する。KORG KeyStageとKORG Module Proの間の通信をこのモードで傍受した。

## manufacturerNameゲート

KeyStageはPE相手のDeviceInfoレスポンスに含まれる`manufacturerName`を見ている。`"KORG"`と名乗った場合だけ、KORG独自のリソースを要求してくる。

つまりサードパーティアプリがKeyStageとフルに連携するには、自分を「KORG」と名乗る必要がある。

```swift
// DeviceInfoレスポンスで"KORG"を名乗る
{"manufacturerName":"KORG","productName":"M2DX DX7 Synthesizer"}
```

MIDI-CIのDevice Identityでも`manufacturerID: .korg`を設定する。

これは仕様違反ではない。manufacturerNameは自由に設定できるフィールド。でもKORGがこれをゲートに使ってるとは思わなかった。

## KORG独自リソース

標準のPEリソース（DeviceInfo、ResourceList、ProgramList）に加えて、KORGは独自リソースを持っている。

### X-ProgramEdit

現在の音色名とCCの値。KeyStageのLCDに音色名が表示される元データ。

フォーマットが独特で、音色名は`"1:E.PIANO 1"`のように1-basedの番号＋コロン＋名前。KORG Moduleと同じ形式にしないとKeyStageで正しく表示されない。

CC値は`currentValues`という配列で返す。これもPE標準にはないフィールド。

### X-ParameterList

CCパラメータの定義。名前、CC番号、デフォルト値。KeyStageのノブやスライダーにパラメータ名を表示するのに使う。

### JSONSchema

上2つのスキーマ定義。KeyStageは接続時に`parameterListSchema`と`programEditSchema`をリクエストしてくる。

## PE Notifyエコーバック

KeyStageからCCをMIDIで受信する。アプリのパラメータが更新される。パラメータが更新されたのでPE Notifyをサブスクライバーに送る——つまりKeyStageに送り返す。

KeyStageのPEプロセッサがこれを処理しきれなくなって、LCDがフリーズする。

対策: MIDI経由で受けたCC変更はPE Notifyを送らない。UI操作によるCC変更だけNotifyする。入力ソースの区別が必要になった。

## PE Initiator-only

一番驚いた発見。KORG Module ProはPEのInitiator（リクエストする側）としてのみ動作する。こちらからPE GETを送っても応答がない。

```
PE: All dests tried, no response. KORG likely PE Initiator-only.
PE: (KORG queries US, but doesn't respond to PE GET)
```

PE仕様は双方向を前提としている。Initiator-onlyは仕様の部分実装。でも「相手が何をサポートしてるか」はやってみないと分からない。PE Capability Replyに「Initiator-only」を示すフラグはない。

## Discovery再送の謎

KeyStageがDiscovery Inquiry（sub-ID2: 0x70）を送ってきたとき、こちらもDiscoveryを再送しないとPEセッションが確立しない。しかも200msの間を空ける必要がある。

即座に返すとKeyStageが無視する。なぜ200msなのかは分からない。試行錯誤で見つけた値。

## 固定MUIDによるセッション永続化

MIDI-CIのMUIDは通常ランダム生成だけど、M2DXでは固定値`0x5404629`を使っている。

KeyStageは前回接続したMUIDを覚えていて、同じMUIDなら再DiscoveryなしでPEセッションを再開する。ランダムだと毎回フルネゴシエーションが走って遅い。

この挙動もどこにも書いてない。実機で試して気づいた。

## PEスニファモードの実装

これらの発見はすべて、SysExの生バイトを記録・解析して得たもの。MIDI2Kitの`#if DEBUG`ブロックにスニファモードを実装してある。

CI SysExを全部16進ダンプし、UMP MIDI 2.0メッセージをデコードしてログに出す。KORG ModuleとKeyStageの通信を観察することで、公開仕様書にない独自仕様を1つずつ解明できた。
