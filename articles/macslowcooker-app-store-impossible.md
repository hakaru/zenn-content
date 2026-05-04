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

- root の `LaunchDaemon`（= macOS のシステム起動時に常駐させる root プロセス）が使えない（App Store は `SMAppService.loginItem`、つまり**ユーザー権限のログイン項目**しか許さない）
- Sandbox（= App Store アプリ必須のプロセス隔離環境）の中から `/usr/bin/powermetrics` を spawn できない
- `IOHIDEventSystem` と `AppleSMC` を直叩きしている（**Apple のプライベート API**）

iStat Menus も Stats も TG Pro も、ハードウェアを覗くタイプの常駐アプリは全部 Notarized DMG（= Apple のサーバで公証を受けた DMG、Gatekeeper 警告なしで開ける）配布。**「Mac のハードウェアを覗くアプリ」と「App Store」は構造的に両立しない**。

## 企画

きっかけはわりと物理的な話。M3 Ultra でローカル LLM 推論を一晩回していると、机に置いた本体が **本気で熱を持つ**。手をかざすと暖房代わりになるし、上に保温したい料理を載せたら本当にサーブまでいけそうな温度になる。

> *これ、上にカレー鍋載せたら煮込めるんじゃ?*

…と思ったわけではないけど、本気で **GPU の熱で何か作れそう** 感は何度かあった。アルミ筐体の Mac は本物の調理器具感がある。

そこから「GPU 使用率を **鍋として** Dock に置きたい」に直結した。LLM 推論やってると GPU が今ヒマか忙しいかチラ見できないストレスが溜まる。Activity Monitor の GPU タブをわざわざ開きに行くと「動いてるのを動いてるか確認しに行く」体験になって萎える。Dock の鍋アイコンで完結するなら、ウィンドウを切り替えずに済む。

ただ「数字が出てるだけ」だとつまらない。**鍋メタファー**で物理量を全部マッピングする。

| 物理量 | 鍋での表現 |
|---|---|
| GPU 使用率 | 鍋の下の炎の高さ |
| SoC 温度 | 鍋のボディの色（白 → 赤橙） |
| ファン RPM | 蓋から立ち昇る湯気の本数・太さ |
| `thermal_pressure`（= macOS が報告する温度カテゴリ、Nominal/Fair/Serious/Critical） | 沸騰演出（蓋ガタガタ + 赤い湯気） |

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

XPC（= Apple の RPC、プロセス間通信）でメインアプリと root の HelperTool が会話する。ポイント:

- **GPU% は IOAccelerator**（= GPU ドライバの統計を IOKit から覗くインタフェース）の `Device Utilization %` を直読み。Activity Monitor の数字と一致する
- **Fan RPM は AppleSMC**（= Mac の System Management Controller、ファン制御 + 温度センサ）を直叩き。`powermetrics --samplers smc` は macOS 26 で削除されたので、`IOServiceMatching("AppleSMC")` で `FNum` と `F[i]Ac`（fpe2 形式）を読む
- **SoC 温度は IOHIDEventSystem**（= HID イベント経由でセンサ値を引っ張る非公開 API）。M3 Ultra には「GPU MTR Temp Sensor」が存在せず、`PMU tdie*` / `PMU tdev*` を77個拾って平均する
- **Universal Binary**（= 1個のバイナリに arm64 と x86_64 を両方入れる macOS の方式）で Apple Silicon と Intel 両方動く

ここまでは API を引っ張れば普通に作れる。**問題は配布の段になってから**。

## App Store に出せない件（本題）

### 1. root の LaunchDaemon が使えない

`powermetrics` は **root じゃないと動かない**。普通のアプリは root を取れないので、`SMAppService.daemon` で root の LaunchDaemon を「ヘルパーツール」として登録し、XPC 経由で叩く。Apple が用意した正規ルート。

…なんだけど、App Store のレビューガイドライン的には `SMAppService.loginItem`（ユーザー権限のログイン項目）しか認められない。`SMAppService.daemon` を使った時点で **Reject 確定**。

> Apps that need to run continuously in the background should use SMAppService.loginItem.

### 2. Sandbox の中から powermetrics を spawn できない

App Store アプリは Sandbox **必須**。`Process` で `/usr/bin/powermetrics` を spawn しようとしても、`powermetrics` 自体が root を要求するし、root じゃなくても **任意のプロセスを spawn する権限**自体が Sandbox にない。`com.apple.security.temporary-exception.process.spawn` みたいな entitlement は App Store では通らない。

### 3. プライベート API を使っている

`IOHIDEventSystemClientCreate` / `IOHIDServiceClientCopyEvent` あたりは全部 **Apple のプライベート API**（= 公開ヘッダに無い内部 API。使うと App Store で reject される）。

```swift
@_silgen_name("IOHIDEventSystemClientCreate")
private func IOHIDEventSystemClientCreate(_ allocator: CFAllocator?) -> Unmanaged<AnyObject>?
```

`@_silgen_name`（= Swift で C/Obj-C シンボルに直接リンクする属性、本来は標準ライブラリ実装用）で直接シンボルを拾ってる時点で App Store のスタティック解析に引っかかる。Stats・iStat Menus・Chromium の `m1_sensors_mac.mm` 全員が同じ事をやっていて、**他にやり方がない**。

### 残された配布ルート

App Store がダメなら **Notarized DMG**。Apple Developer ID で署名 → `xcrun notarytool submit` で公証 → `stapler staple` でチケット添付 → DMG。Gatekeeper の警告なしでインストールできる。

…と書きつつ、MacSlowCooker は個人用に作ったので、配布するつもりはない。OSS にして「自分で署名して clone してビルド」が現実解。Apache 2.0 で公開: https://github.com/hakaru/MacSlowCooker

## その後 ─ MRTG (1995年) リスペクト

App Store がダメなら好きに作ろう、と思って生やしたのが履歴系統。

最初は ring buffer（= 古い要素を上書きしていく固定長バッファ）に60サンプルしか持っておらず過去30秒分しか見えなかった。LLM トレーニング一晩回した後の温度推移を振り返れない。冒頭の「鍋でカレー煮込めるんじゃ?」感の延長で、*どれくらいの時間どれくらいの温度で煮込まれてたか* を知りたい。

で、思い出したのが **MRTG**（Multi Router Traffic Grapher、1995年〜）。SNMP polling して Daily/Weekly/Monthly/Yearly の4枚のグラフを淡黄色背景に濃緑塗りで吐く、あの牧歌的なやつ。今でいう **Grafana + Prometheus** の祖先。

これを Mac の GPU 監視に持ち込んだら *いい感じに懐かしくないか* と思って、3系統の出力を生やした。

### 1. アプリ内 History ウィンドウ (Cmd+Shift+H)

5min/30min/2hour/1day の4階層 SQLite **ラウンドロビン**（= 古い行を捨てて新しい行で上書きする固定容量テーブル、RRDtool 由来）に 24h/7d/31d/400d で貯める。XPC で 2Hz で上がってくるサンプルを 5分間メモリにバッファ → 5min テーブルに insert → 境界跨いだら30min/2hr/1d へ **cascading rollup**（= 細かい解像度から粗い解像度へ自動的に集計を積み上げる）。pure logic は `Shared/HistoryAggregator.swift` に切り出して in-memory SQLite で完結ユニットテスト。

UI は **MRTG** をそのまま再現。白背景に濃緑塗り、青線、dual Y axis（Compute = GPU% + Power、Thermal = SoC °C + Fan RPM）、X軸は実時間（hour-of-day / day-of-week / day-of-month / 月略）、Max/Avg/Cur のフッタ。

### 2. Prometheus エクスポーター (`/metrics`)

外から scrape（= 一定間隔でメトリクスを取得）させたいので `Network.framework` の `NWListener`（= TCP/UDP リスナの新 API）で HTTP/1.1 を手書き。

```
macslowcooker_gpu_usage_ratio 0.42
macslowcooker_gpu_power_watts 8.4
macslowcooker_temperature_celsius 67.2
macslowcooker_fan_rpm{fan="0"} 1850
macslowcooker_thermal_pressure 0
```

デフォルトはオフ、127.0.0.1 のみ、port 9091。「Bind to all interfaces」で LAN 公開（macOS firewall プロンプトが出る）。これで Prometheus（時系列メトリクスDB）+ Grafana（ダッシュボード）に流せば「ちゃんとした監視」になる。

### 3. PNG + index.html （本物のMRTG workflow）

ここまでくると「もう本家 MRTG そのままやれば？」という気持ちになる。`SwiftUI` の `ImageRenderer`（= SwiftUI の view を CGImage に焼く API、macOS 13+）で MRTG パネルを rasterize して 8枚のPNG + auto-refresh 入りの `index.html` を5分ごとに吐き出す。

```bash
python3 -m http.server -d ~/Library/Application\ Support/MacSlowCooker/web/
# http://localhost:8000/ で 8枚並んで auto-refresh
```

これは **本物の MRTG そのままの workflow**。1995年の人が見たら「ああ、これな」って言うはず。

## まとめ

GPU 使用率を鍋アイコンで可視化したい、という思いつきから始まったので、最初は楽しい開発だった。

そこから macOS 26 で `powermetrics` の plist スキーマが変わってる / SMC sampler が削除されてる / `@main` AppDelegate が動かない …と、気付くたびに地雷の解像度が上がっていく。最後に App Store 配布の検証で「全部詰んでた」と分かった。**Mac のハードウェアを覗くタイプのアプリは構造的に App Store 不可**、というのは知識として知ってたけど、自分で踏み抜いて初めて重みが分かる。iStat Menus が App Store に出ない理由が、自分で同じ穴に落ちて初めて納得できた。

App Store ダメ → 配布する気もないし好きに作ろう、で生やした履歴系統が思いのほか実用的になった。手元で見るならアプリ内 (Cmd+Shift+H)、Grafana に流すなら Prometheus、社内 wiki に貼るなら PNG。MRTG リスペクトは *よい*。1995年のツールへのオマージュなので。

最初の動機 — *Mac に手を当てたら煮込み料理できそうなくらい熱い* — を、ちゃんと「いつどれくらい煮込まれてたか」をグラフで振り返れるようになった。鍋アイコンに対してフィードバックが完結している、ぐらいの達成感はある。

リポジトリ: https://github.com/hakaru/MacSlowCooker
