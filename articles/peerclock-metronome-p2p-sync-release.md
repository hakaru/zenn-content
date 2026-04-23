---
title: "WiFi上の複数iPhoneで±2msで同期するメトロノーム「PeerClock Metronome」を公開した"
emoji: "🎵"
type: "tech"
topics: ["ios", "swift", "swiftui", "audio", "music"]
published: true
---

ピアノを弾くので個人開発で音楽系のアプリをいくつか作っている。バンド練習で「全員のiPhoneから同じテンポのメトロノームを鳴らしたい」というニーズがあった。市販のメトロノームアプリは単独動作前提で、複数台でクリックを鳴らすとジワジワとズレていく。

だから作った。今日 App Store に出た。

https://apps.apple.com/jp/app/id6762972307

## できること

- 同じWi-Fi上のiPhoneが自動で相互接続（ペアリング・設定なし）
- どの端末のクリックも**±2msで一致**
- 誰かがBPMや拍子を変えると全端末に即座に反映
- 完全オフライン / サーバー不要
- 無料

## 自作ライブラリ [PeerClock](https://github.com/hakaru/PeerClock) を使っている

このアプリは先月公開したSwiftライブラリの上に載っている。PeerClock自体の解説はこちら。

https://zenn.dev/hakaru/articles/peerclock-p2p-clock-sync-swift

要するに「Appleデバイス同士を±2msで時刻同期するP2Pライブラリ」。PeerClock Metronome はその最初の実使用例であり、リファレンス実装も兼ねている。

## なぜメトロノームに同期クロックが要るのか

120 BPM なら 1 拍 = 500ms。人間の拍認識は±10ms 前後が限界と言われる。ここに対して、iPhone の水晶発振器のドリフトは 20〜50ppm。

- 「せーの」で 2 台起動しても、開始時点で人間の反応誤差で ±100ms はズレる
- 仮にピッタリ揃って始まっても、30 秒後には定常ドリフトだけで ~1.5ms 開く
- 10 分後には 30ms。もう耳で気持ち悪い

要件は3つ。全デバイスで一致する時刻軸、その軸にクリック時刻を乗せる仕組み、そしてドリフトに応じた周期的な再同期。これは PeerClock の設計そのもの。NTP 風の 4-timestamp 交換で同期し、best-half フィルタリングで Wi-Fi ジッタを均す。定常ドリフトにはバックオフしながら再同期で追従する。アプリから見れば `clock.now` を呼ぶだけで、全端末で揃った `UInt64` ナノ秒が返る。

## 実装の芯: synced clock 駆動のビートスケジューラ

`MetronomeEngine` は普段 `mach_absolute_time()` ベースで動く。P2P 同期モードではそこに PeerClock から synced clock を注入して、次のビート時刻をそこから計算する。

```swift
actor MetronomeEngine {
    var syncedNowProvider: (@Sendable () -> UInt64)?

    private func calculateNextBeatFromSyncedClock() {
        guard let provider = syncedNowProvider else { return }
        let syncedNow = provider()
        let hostNow = mach_absolute_time()
        let subIntervalNs = UInt64(config.subIntervalSeconds * 1_000_000_000)

        // 同期時刻軸で次のサブビート境界を見つける
        let nextSubBeatSynced = ((syncedNow / subIntervalNs) + 1) * subIntervalNs
        let deltaNs = nextSubBeatSynced - syncedNow

        // host 時刻へ変換（AVAudioTime がこれを要求する）
        nextBeatHostTime = hostNow + nsToMach(deltaNs)

        // その境界が小節内のどの位置か決定する
        let totalSubsPerBar = UInt64(config.totalSubsPerBar)
        currentSubBeat = Int((nextSubBeatSynced / subIntervalNs) % totalSubsPerBar)
    }
}
```

肝は、同期時刻上で `(n+1) * subInterval` の瞬間を次のビートと決めること。この式を各端末が独立に計算しても、`syncedNow` が±2ms で揃っているので結果もその精度で一致する。小節内位置 (`currentSubBeat`) も同期時刻から導出するため、後から参加したピアがいきなり正しい位置で鳴り始める。

## BPM / 拍子の伝搬

誰かが UI で BPM を動かすと、その端末は `Command(type: "metronome.config", payload: ...)` を全ピアに broadcast する。ペイロードには「いつ適用するか」の絶対同期時刻が埋まっている。

```swift
// 送信側
let applyAtNs = peerClock.now + 200_000_000  // 200ms 先の同期時刻
let config = MetronomeConfig(bpm: 140, timeSignature: .fourFour)

var payload = Data()
var applyAtBE = applyAtNs.bigEndian
payload.append(Data(bytes: &applyAtBE, count: 8))
payload.append(try JSONEncoder().encode(config))

try await peerClock.broadcast(
    Command(type: "metronome.config", payload: payload)
)

// 受信側
for await (_, cmd) in peerClock.commands where cmd.type == "metronome.config" {
    let (applyAtNs, newConfig) = parse(cmd.payload)
    await metronomeEngine.updateConfigAt(newConfig, applyAtNs: applyAtNs)
}
```

`updateConfigAt` は受信した `applyAtNs` が来るまで既存 config で刻み続け、その瞬間に切り替える。200ms の猶予を持たせてあるので、多少遅延の大きいピアでも間に合う。

結果、UI スライダーで BPM を動かすと次の拍から全端末が揃って新しいテンポになる。誰が送信者でも同じで、2〜5 台どの端末からでもいじれる。

## アーキテクチャ

```
┌──────────────────────────────────┐
│        SwiftUI View              │
│  MetronomeView / ConductorView   │
└──────────────┬───────────────────┘
               │ @Observable
┌──────────────▼───────────────────┐
│     MetronomeViewModel           │
│  - UI イベント処理                │
│  - 状態発行                      │
└─────┬──────────────────────┬─────┘
      │                      │
┌─────▼──────────┐ ┌─────────▼──────────┐
│ MetronomeEngine│ │ PeerMetronomeService│
│  actor         │ │  actor              │
│  - スケジューリング│ │  - Command 送受信  │
│  - AVAudioEngine│ │  - PeerClock 駆動   │
└────────────────┘ └─────────┬──────────┘
                             │
                    ┌────────▼───────┐
                    │   PeerClock    │
                    │  (SPM package) │
                    └────────────────┘
```

UI、音声エンジン、ネットワークを actor で分離。SwiftUI は `@Observable` ViewModel を観察するだけ。Swift 6 strict concurrency + complete で警告ゼロ。

## アプリ側から見た PeerClock の使い心地

要するにこれだけ。

```swift
// 1. 起動（全デバイスで同じコード。master/slave 不要）
let clock = PeerClock()
try await clock.start()

// 2. 同期時刻を取る
let now = clock.now  // UInt64, nanoseconds, 全デバイスで揃う

// 3. コマンドを broadcast
try await clock.broadcast(Command(type: "metronome.config", payload: data))

// 4. 受信
for await (sender, command) in clock.commands {
    handle(command)
}

// 5. ピア数を監視
for await peers in clock.peers {
    peerCount = peers.count
}
```

PeerClock はコマンドのセマンティクスを一切知らない。BPM 変更でも録音開始でも、展示物のシーン遷移でも、アプリ側が `type` 文字列で好きに定義する。時刻合わせとトランスポートの面倒なところだけライブラリが引き取る、という切り分け。

## 使ってみる

- App Store (無料): https://apps.apple.com/jp/app/id6762972307
- PeerClock (Apache-2.0): https://github.com/hakaru/PeerClock

2台以上の iPhone を同じ Wi-Fi につないでアプリを起動するだけで、自動で相互を発見して同期が始まる。「本当に揃ってる」というのを耳で確認できるのが、地味に気持ちいい。

バンド練習、ドラム教室、複数モニターへのクリック配信、リハでの仮テンポ共有——意外と使い道がある。もともとは自分のために作ったやつだけど、PeerClock ライブラリの実戦投入例としても使えるので App Store に出した。

PeerClock 自体は音楽専用ではなくて、ローカルネットワーク上での同期基盤として汎用化してある。展示物の同時制御や複数端末での同時計測、ゲームの同時スタートなんかにも流用できる設計。興味あれば Issue / PR 歓迎。
