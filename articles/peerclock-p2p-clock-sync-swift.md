---
title: "iPhoneを複数台±2msで同期させるSwiftライブラリ「PeerClock」を作った"
emoji: "🕐"
type: "tech"
topics: ["ios", "swift", "p2p", "network", "opensource"]
published: true
---

## なぜ作ったか

[1Take](https://apps.apple.com/us/app/1take/id6757945099)という録音アプリを個人開発している。バンド練習で「iPhone複数台で同時に録音したい」というニーズがあって、マルチデバイス同時録音機能を作り始めた。

問題は**時刻の同期**。iPhoneの水晶発振器は20〜50ppmのドリフトがある。2台のiPhoneで「せーの」で録音を開始しても、数十msのズレが生じる。音楽だとこれは致命的。

既存のライブラリを調べたが、どれも要件を満たさなかった。

| | TrueTime / Kronos | PeerKit / sReto |
|--|---|---|
| 同期対象 | 外部NTPサーバー | N/A（データ転送のみ） |
| インターネット | **必要** | 不要 |
| デバイス間同期 | **不可** | **不可** |

「ローカルネットワーク上のiPhone同士で、サーバーなしで、±2ms以内に時刻を合わせる」。こんなライブラリは存在しなかった。だから作った。

https://github.com/hakaru/PeerClock

## PeerClockができること

### 1. ±2msのクロック同期

同じWi-Fi上にいるAppleデバイス同士の時刻を±2msで同期する。外部サーバー不要。インターネット不要。

```swift
let clock = PeerClock()
try await clock.start()

// 全デバイスで一致する時刻（±2ms）
let timestamp = clock.now
```

### 2. 全ノード対等（peer-equal）

**マスターもスレーブもない。** 全デバイスが同じコードを走らせる。内部的にはクロック同期の基準が1台必要だが、それはUUIDが最小のピアが自動的に担当する。アプリからは完全に不可視。

```swift
// どのデバイスも同じコード。role指定なし。
let clock = PeerClock()
try await clock.start()
```

### 3. 汎用コマンドチャネル

PeerClockはコマンドのセマンティクスを知らない。「録音開始」も「停止」も、アプリが自由に定義する。

```swift
// 送信
try await clock.broadcast(
    Command(type: "com.myapp.record.start", payload: Data())
)

// 受信
for await (sender, command) in clock.commands {
    handleCommand(command, from: sender)
}
```

### 4. ステータス共有

各デバイスの状態（録音中、バッテリー残量など）をリアルタイムで共有。デバウンス付き、世代番号で新旧判定。

```swift
await clock.setStatus("recording", forKey: "app.state")

for await status in clock.statusUpdates {
    // リモートピアのステータスが来る
}
```

### 5. 精密イベントスケジューリング

「全デバイスで同時に録音開始」を実現するための仕組み。同期済み時刻軸でアクションを予約する。

```swift
let fireTime = clock.now + 3_000_000_000  // 3秒後
let handle = try await clock.schedule(atSyncedTime: fireTime) {
    startRecording()  // 全デバイスで±2ms以内に発火
}
```

同期状態が不十分な場合はスケジューリングを拒否する安全ガード付き。

### 6. 接続ヘルスモニタリング

ハートビートベースの3段階状態遷移。

```
connected → degraded → disconnected
```

```swift
for await event in clock.connectionEvents {
    print("\(event.peerID): \(event.state)")
}
```

### 7. トランスポート自動フェイルオーバー

WiFiが使えなくなったら自動でMultipeerConnectivityに切り替え。WiFiが復帰したら戻る。

## PeerClockができないこと

ここを明確にしておくのは大事。

- **サブマイクロ秒精度** — ハードウェアワードクロックが必要。iPhoneでは物理的に不可能
- **音声・映像の処理** — PeerClockは時刻とコマンドの同期基盤。メディア処理はアプリの責務
- **ファイル転送** — コマンドのペイロードは軽量データ向け。ファイルは別の仕組みで
- **20台以上の大規模接続** — 小規模ピアグループ（2〜10台）向けの設計
- **クロスプラットフォーム** — Apple専用（iOS 17+ / macOS 14+）。Android/Windowsは対象外
- **インターネット越しの同期** — 同一ローカルネットワーク前提

## アーキテクチャ

```
PeerClock (Facade — 全ピア対等、roleなし)
│
├── Transport          reliable + unreliable チャネル
│   ├── WiFiTransport  Network.framework (UDP + TCP)
│   ├── MultipeerTransport  MPC フォールバック
│   ├── FailoverTransport   自動切替ステートマシン
│   └── MockTransport  インメモリ（テスト用）
│
├── Coordination       最小PeerIDで自動選出
│
├── ClockSync          NTP風 4-timestamp 交換
│   ├── NTPSyncEngine  40測定 + best-half フィルタリング
│   ├── DriftMonitor   ジャンプ検出 → 完全再同期
│   └── BackoffController  動的同期間隔 [5→30s]
│
├── Command            汎用コマンド send/broadcast
│   └── CommandRouter  ストリーム分離 + コマンドID
│
├── Status             ステータス共有
│   ├── StatusRegistry   ローカル（デバウンス付きプッシュ）
│   └── StatusReceiver   リモート（世代番号デデュプ）
│
├── Heartbeat          接続ヘルスモニタリング
│   └── HeartbeatMonitor  3段階ステートマシン
│
├── EventScheduler     同期時刻精密発火
│
└── Wire               バイナリプロトコル（5バイトヘッダ）
    └── MessageCodec   エンコード/デコード
```

### 同期アルゴリズムの仕組み

NTPから着想を得た4-timestampプロトコルを使う。

```
Peer A (フォロワー)         Peer B (コーディネーター)
    │                            │
    │── SYNC_REQUEST [t0] ──────>│
    │                      t1 = 受信時刻
    │                      t2 = 送信時刻
    │<── SYNC_RESPONSE [t0,t1,t2]│
    │ t3 = 受信時刻              │
    │                            │
    offset = ((t1 - t0) + (t2 - t3)) / 2
```

1. 30ms間隔で40回測定（約1.2秒）
2. RTTでソートし、高速な上位50%だけ使う（best-half filtering）
3. フィルタ後のオフセット平均を算出
4. 同期安定後は再同期間隔を5s → 10s → 20s → 30sとバックオフ
5. オフセットが10ms以上ジャンプしたら全リセット

### なぜ±2msで済むのか

| 誤差源 | 生の誤差 | 緩和策 | 残差 |
|--------|---------|--------|------|
| Wi-Fi UDPジッター | 1〜10ms | best-halfフィルタリング | ~1〜2ms |
| 水晶発振器ドリフト | 50ppm (0.25ms/5s) | 周期的再同期 | <0.25ms |
| iOSスケジューリング | <1ms | `mach_continuous_time` | <1ms |
| **合計** | | | **±2ms** |

### ワイヤプロトコル

5バイトヘッダ + ペイロード。

```
┌──────────┬──────────┬──────────┬──────────────┐
│ Version  │ Category │ Flags    │ Length       │
│ 1 byte   │ 1 byte   │ 1 byte   │ 2 bytes BE  │
└──────────┴──────────┴──────────┴──────────────┘
```

同期パケットはUDP（unreliable）、コマンドとステータスはTCP（reliable）。チャネルの使い分けは `Transport` プロトコルが抽象化している。

## 設計で意識したこと

### Protocol at every boundary

すべてのレイヤー境界にProtocolを置いた。`Transport`、`SyncEngine`、`CommandHandler`。実装のないProtocolは作らない。

これにより`MockTransport`（インメモリ）でネットワークなしに全ロジックをユニットテストできる。

```swift
let network = MockNetwork()
let clock = PeerClock(transportFactory: { peerID in
    network.createTransport(for: peerID)
})
```

現在127テスト、26スイート。全部MockTransportベースで実機不要。

### Swift 6 strict concurrency

全公開型は`Sendable`準拠。可変状態を持つクラスは`@unchecked Sendable` + `NSLock`。actor も積極活用（`StatusRegistry`、`StatusReceiver`、`HeartbeatMonitor`、`EventScheduler`）。

### 外部依存ゼロ

Foundation + Network.framework + MultipeerConnectivity のみ。サードパーティライブラリなし。

## 使い方

### インストール

```swift
// Package.swift
dependencies: [
    .package(url: "https://github.com/hakaru/PeerClock.git", from: "0.2.0")
]
```

### 最小構成

```swift
import PeerClock

let clock = PeerClock()
try await clock.start()

// ピアが見つかるまで待つ
for await peers in clock.peers {
    if peers.count >= 2 { break }
}

// 同期済み時刻
print("Synced time: \(clock.now)")

// コマンド送信
try await clock.broadcast(Command(type: "start"))
```

Info.plistに以下が必要:

```xml
<key>NSLocalNetworkUsageDescription</key>
<string>PeerClock uses the local network for device synchronization.</string>
<key>NSBonjourServices</key>
<array>
    <string>_peerclock._udp</string>
</array>
```

## ロードマップ

Phase 1〜3.7は完了済み（v0.2.0）。

**今後の予定（Phase 4）:**
- 合議ベース同期（全ペア測定 + 中央値基準）
- ネットワーク品質ベースのコーディネーター選出
- 超音波同期マーカー（音響パルス + 相互相関）
- watchOS対応

## まとめ

iOSエコシステムに「ローカルネットワーク上のデバイス間で±2msの時刻同期を提供するpeer-equalなライブラリ」は存在しなかった。PeerClockはこのギャップを埋める。

1Takeのマルチデバイス録音用に作ったが、汎用的に使えるように設計してある。同期再生、マルチカメラ撮影、デバイスフリート管理、なんでもいい。デバイスが「今」を共有する必要があるなら、PeerClockがインフラになる。

https://github.com/hakaru/PeerClock
