---
title: "GPU 使用率を Dock の鍋アイコンで可視化したかった ─ App Store に出せない件"
emoji: "🍲"
type: "tech"
topics: ["macos", "swift", "appstore", "xpc", "powermetrics"]
published: false
---

![MacSlowCooker の Dock アイコン: idle → 高負荷 → 沸騰 → クールダウン](/images/macslowcooker/hero.gif)

Mac の GPU 使用率と温度を、Dock の鍋アイコンで眺められたら楽しくない?

ということで作った。**MacSlowCooker** という名前の常駐アプリ。GPU が忙しくなると鍋の下の炎がでかくなって、SoC 温度が上がると鍋が赤くなって、ファンが回ると湯気がモコモコ立つ。

…で、最後に App Store 配布できるか確認したら **無理**だった。理由が macOS の権限モデルの解像度を上げる教材としてちょうどよかったので、企画から「出せない」までを残しておく。

## 結論

❌ App Store 配布は無理。理由は3つ:

- root の `LaunchDaemon` が使えない（App Store は `SMAppService.loginItem` しか許さない）
- Sandbox の中から `/usr/bin/powermetrics` を spawn できない
- `IOHIDEventSystem` と `AppleSMC` を直叩きしている（私的 API）

iStat Menus も Stats も TG Pro も、ハードウェアを覗くタイプの常駐アプリは全部 Notarized DMG 配布。**「Mac のハードウェアを覗くアプリ」と「App Store」は構造的に両立しない**。

## 企画

GPU 使用率を画面の隅で常時眺めたい、というのが出発点。LLM のローカル推論をやっていると、GPU が今ヒマか忙しいか **チラ見できない**ストレスが溜まる。Activity Monitor の GPU タブをわざわざ開きに行くと「動いてるのを動いてるか確認しに行く」体験になって萎える。

ただ「数字が出てるだけ」だとつまらない。せっかく Dock に居座らせるなら、見ていて楽しい何かにしたい。

そこで **鍋メタファー**。

| 物理量 | 鍋での表現 |
|---|---|
| GPU 使用率 | 鍋の下の炎の高さ |
| SoC 温度 | 鍋のボディの色（白 → 赤橙） |
| ファン RPM | 蓋から立ち昇る湯気の本数・太さ |
| `thermal_pressure` | 沸騰演出（蓋ガタガタ + 赤い湯気） |

GPU が忙しい = 火が強い = 鍋が煮える。温度が上がる = 鍋が赤くなる。**全部物理的に同じ向きに動く**ので、メタファーとして破綻しない。

名前は **MacSlowCooker**。スロークッカーは「弱火でずっと煮込む鍋」だけど、その逆向き — *強火でフル稼働してたら鍋が泣くよ* — というニュアンス。

## 実装

ざっくり構成:

```
MacSlowCooker.app（非特権、ユーザーセッション）
  ├── DockIconAnimator       — 補間 / wiggle / 沸騰フェード
  ├── DutchOvenRenderer      — Core Graphics で鍋・炎・湯気を描画
  ├── PopupView              — SwiftUI ダッシュボード
  └── XPCClient              — NSXPCConnection (.privileged)、2 Hz polling

HelperTool（root LaunchDaemon）
  ├── PowerMetricsRunner     — /usr/bin/powermetrics 常駐、plist パース
  ├── IOAcceleratorReader    — Activity Monitor と一致する GPU%
  ├── SMCReader              — AppleSMC 直叩きで fan RPM
  └── TemperatureReader      — IOHIDEventSystem 経由で SoC 温度
```

ポイント:

- **GPU% は IOAccelerator の `Device Utilization %`** を直読み。Activity Monitor の数字と一致する
- **Fan RPM は AppleSMC を直叩き**。`powermetrics --samplers smc` は macOS 26 で削除されたので、`IOServiceMatching("AppleSMC")` で `FNum` と `F[i]Ac`（fpe2 形式）を読む
- **SoC 温度は IOHIDEventSystem**。M3 Ultra には「GPU MTR Temp Sensor」が存在せず、`PMU tdie*` / `PMU tdev*` を77個拾って平均する
- **Universal Binary** (arm64 + x86_64) で Apple Silicon と Intel 両方動く

ここまでは API を引っ張れば普通に作れる。**問題は配布の段になってから**。

## App Store に出せない件（本題）

### 1. root の LaunchDaemon が使えない

`powermetrics` は **root じゃないと動かない**。普通のアプリは root を取れないので、`SMAppService.daemon` で root の LaunchDaemon を「ヘルパーツール」として登録し、XPC 経由で叩く。Apple が用意した正規ルート。

…なんだけど、App Store のレビューガイドライン的には `SMAppService.loginItem`（ユーザー権限のログイン項目）しか認められない。`SMAppService.daemon` を使った時点で **Reject 確定**。

> Apps that need to run continuously in the background should use SMAppService.loginItem.

### 2. Sandbox の中から powermetrics を spawn できない

App Store アプリは Sandbox **必須**。`Process` で `/usr/bin/powermetrics` を spawn しようとしても、`powermetrics` 自体が root を要求するし、root じゃなくても **任意のプロセスを spawn する権限**自体が Sandbox にない。`com.apple.security.temporary-exception.process.spawn` みたいな entitlement は App Store では通らない。

### 3. 私的 API を使っている

`IOHIDEventSystemClientCreate` / `IOHIDServiceClientCopyEvent` あたりは全部 **私的 API**。

```swift
@_silgen_name("IOHIDEventSystemClientCreate")
private func IOHIDEventSystemClientCreate(_ allocator: CFAllocator?) -> Unmanaged<AnyObject>?
```

`@_silgen_name` で直接シンボルを拾ってる時点で App Store のスタティック解析に引っかかる。Stats・iStat Menus・Chromium の `m1_sensors_mac.mm` 全員が同じ事をやっていて、**他にやり方がない**。

### 残された配布ルート

App Store がダメなら **Notarized DMG**。Apple Developer ID で署名 → `xcrun notarytool submit` で公証 → `stapler staple` でチケット添付 → DMG。Gatekeeper の警告なしでインストールできる。

…と書きつつ、MacSlowCooker は個人用に作ったので、配布するつもりはない。OSS にして「自分で署名して clone してビルド」が現実解。Apache 2.0 で公開: https://github.com/hakaru/MacSlowCooker

## その後 ─ MRTG (1995年) リスペクト

App Store がダメなら好きに作ろう、と思って生やしたのが履歴系統。

最初は ring buffer に60サンプルしか持っておらず過去30秒分しか見えなかった。LLM トレーニング一晩回した後の温度推移を振り返れない。で、思い出したのが **MRTG**（Multi Router Traffic Grapher、1995年〜）。SNMP polling して Daily/Weekly/Monthly/Yearly の4枚のグラフを淡黄色背景に濃緑塗りで吐く、あの牧歌的なやつ。Grafana + Prometheus の祖先。

これを Mac の GPU 監視に持ち込んだら *いい感じに懐かしくないか* と思って、3系統の出力を生やした。

### 1. アプリ内 History ウィンドウ (Cmd+Shift+H)

5min/30min/2hour/1day の4階層 SQLite ラウンドロビンに 24h/7d/31d/400d で貯める。XPC で 2Hz で上がってくるサンプルを 5分間メモリにバッファ → 5min テーブルに insert → 境界跨いだら30min/2hr/1d へ cascading rollup。pure logic は `Shared/HistoryAggregator.swift` に切り出して in-memory SQLite で完結ユニットテスト。

UI は **MRTG** をそのまま再現。白背景に濃緑塗り、青線、dual Y axis（Compute = GPU% + Power、Thermal = SoC °C + Fan RPM）、X軸は実時間（hour-of-day / day-of-week / day-of-month / 月略）、Max/Avg/Cur のフッタ。

### 2. Prometheus エクスポーター (`/metrics`)

外から scrape させたいので `Network.framework` の `NWListener` で HTTP/1.1 を手書き。

```
macslowcooker_gpu_usage_ratio 0.42
macslowcooker_gpu_power_watts 8.4
macslowcooker_temperature_celsius 67.2
macslowcooker_fan_rpm{fan="0"} 1850
macslowcooker_thermal_pressure 0
```

デフォルトはオフ、127.0.0.1 のみ、port 9091。「Bind to all interfaces」で LAN 公開（macOS firewall プロンプトが出る）。

### 3. PNG + index.html （本物のMRTG workflow）

ここまでくると「もう本家 MRTG そのままやれば？」という気持ちになる。`SwiftUI` の `ImageRenderer` で MRTG パネルを rasterize して 8枚のPNG + auto-refresh 入りの `index.html` を5分ごとに吐き出す。

```bash
python3 -m http.server -d ~/Library/Application\ Support/MacSlowCooker/web/
# http://localhost:8000/ で 8枚並んで auto-refresh
```

これは **本物の MRTG そのままの workflow**。1995年の人が見たら「ああ、これな」って言うはず。

## まとめ

GPU 使用率を鍋アイコンで可視化したい、という思いつきから始まったので、最初は楽しい開発だった。

そこから macOS 26 で `powermetrics` の plist スキーマが変わってる / SMC sampler が削除されてる / `@main` AppDelegate が動かない …と、気付くたびに地雷の解像度が上がっていく。最後に App Store 配布の検証で「全部詰んでた」と分かった。**Mac のハードウェアを覗くタイプのアプリは構造的に App Store 不可**、というのは知識として知ってたけど、自分で踏み抜いて初めて重みが分かる。iStat Menus が App Store に出ない理由が、自分で同じ穴に落ちて初めて納得できた。

App Store ダメ → 配布する気もないし好きに作ろう、で生やした履歴系統が思いのほか実用的になった。手元で見るならアプリ内 (Cmd+Shift+H)、Grafana に流すなら Prometheus、社内 wiki に貼るなら PNG。MRTG リスペクトは *よい*。1995年のツールへのオマージュなので。

リポジトリ: https://github.com/hakaru/MacSlowCooker
