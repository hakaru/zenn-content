---
title: "MIDI 2.0仕様書の闇"
---

## 仕様書は綺麗、現実は混沌

MIDI 2.0の仕様書（MMA / AMEI発行）はよくできてる。UMPのパケット構造、MIDI-CIのメッセージフォーマット、Property Exchangeのリクエスト/レスポンス。図入りで整理されていて、読むだけなら理解できる。

問題は「仕様書に書いてないこと」が多すぎること。

## Property Exchangeのエンコーディング問題

PEのデータはSysEx経由で送る。SysExは7ビット制約（各バイトの最上位ビットが0）があるので、8ビットデータを7ビットに詰め直す**Mcoded7**というエンコーディングを使う。仕様書にはそう書いてある。

KORGの実装: Mcoded7を使わず、素のUTF-8をそのまま送ってくる。

こちらがMcoded7デコードを通すとデータが壊れる。素のUTF-8だと正しく読める。仕様違反だけど、現実にはこれで動いてるデバイスがある。

## ResourceListのフィールド型が違う

仕様書では`canGet`/`canSet`はブーリアン（true/false）。

KORGの実装: 文字列。`"full"`とか`"none"`が来る。

```json
// 仕様書
{"resource": "DeviceInfo", "canGet": true, "canSet": false}

// KORGの現実
{"resource": "DeviceInfo", "canGet": "full", "canSet": "none"}
```

Swiftの`Codable`でBoolとしてデコードすると、KORGの独自リソース2つが黙って消える。エラーも出ない。`canGet`のデコードに失敗した時点でそのエントリ全体がスキップされる。

修正するには`CanValue`という列挙型を作って、BoolでもStringでもデコードできるようにする必要があった。

```swift
enum CanValue: Decodable {
    case bool(Bool)
    case string(String)

    var isEnabled: Bool {
        switch self {
        case .bool(let v): return v
        case .string(let s): return s == "full"
        }
    }
}
```

## `schema`フィールドの型も違う

仕様書では文字列（スキーマ名への参照）。KORGはJSONオブジェクトをそのまま入れてくる。同じフィールドなのに型が違う。

## リクエストIDの仕様

PEのリクエストIDは7ビット（0〜127）。デバイスごとに128個まで同時リクエスト可能——と仕様書には書いてある。

現実には、トランザクションが失敗してもIDが解放されないケースがある（NAKが来ない、タイムアウト後もオブジェクトが残る）。128個のIDが枯渇すると新しいリクエストが送れなくなる。

仕様書はIDの割り当て方法に言及してない。解放タイミングも明記されてない。実装者が考えるしかない。

## PE Notifyのメッセージタイプが変わった

MIDI-CI v1.1ではPE Notifyのサブタイプが`0x38`。v1.2で`0x3F`に変更された。

同じ「Notify」なのにバージョンでメッセージタイプが変わる。相手がv1.1なのかv1.2なのかはDiscovery時に確認して、デバイスごとに使い分ける必要がある。

## ドキュメント化されてないAppleの動作

macOS 15で追加された`MIDIUMPEndpointManager`と`MIDICIDeviceManager`。SDKのヘッダーには存在するけど、Appleのリリースノートには載ってない。ドキュメントもほぼない。

`MIDICIDeviceManager`で発見されるデバイスの中に、Appleの製造者SysEx ID（`0x11 0x00 0x00`）を持つ参加者がいる。Discoveryブロードキャストには応答するのに、PEリクエストには一切応答しない。タイムアウトを待つだけ時間の無駄になる。

公式ドキュメントにはこの挙動の説明がない。実機で観察してブラックリストに入れるしかなかった。

## 「動くコード」が仕様書

結局、MIDI 2.0の「本当の仕様」は仕様書ではなく、実際のデバイスの挙動にある。仕様書は理想、実装は現実。そのギャップを埋めるのがライブラリの仕事になる。
