---
title: "GPU で料理しよう。 MacSlowCooker"
emoji: "🍲"
type: "tech"
topics: ["macos", "swift", "appstore", "xpc", "powermetrics"]
published: true
---

![MacSlowCooker の Dock アイコン: idle → 高負荷 → 沸騰 → クールダウン](/images/macslowcooker/hero.gif)

GPUフル回転させていると暖房に使える。これだけ熱くなれば煮込み料理も作れるんじゃない? 思ったことがありますよね。

ということで作った。**MacSlowCooker** という名前の常駐アプリ。GPU が忙しくなると鍋の下の炎がでかくなって、SoC 温度が上がると鍋が赤くなって、ファンが回ると湯気がモコモコ立つ。
Macの上に載せた鍋の温度管理にも使えます。

というわけでGPU 使用率を **鍋として** Dock に置きたい。
LLM 推論やってると GPU が今ヒマか忙しいかチラ見できないストレスが溜まる。Activity Monitor の GPU タブ開きっぱなしにしてたけどただ邪魔なだけ。Dock の鍋アイコンが理想。

| 物理量 | 鍋での表現 |
|---|---|
| GPU 使用率 | 鍋の下の炎の高さ |
| SoC 温度 | 鍋のボディの色（白 → 赤橙） |
| ファン RPM | 蓋から立ち昇る湯気の本数・太さ |
| `thermal_pressure`（= macOS が報告する温度カテゴリ、Nominal/Fair/Serious/Critical） | 沸騰演出（蓋ガタガタ + 赤い湯気） |

夢）Macアルミ天板の温度取得ができれば、調理器具としても完璧なのだが。。

## 実装

構成:

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

## App Store に出せない！！！
❌ App Store 配布は無理。


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

`@_silgen_name`（= Swift で C/Obj-C シンボルに直接リンクする属性、本来は標準ライブラリ実装用）で直接シンボルを拾ってる時点で App Store のスタティック解析に引っかかる。Stats・iStat Menus・Chromium の `m1_sensors_mac.mm` 全員が同じ事をやっていて、**他にやり方がない**

。
 https://github.com/hakaru/MacSlowCooker

##  MRTG 　　（インターネット老人会向け機能）

履歴系統。

調理器具としても何時間なん度℃で煮込まれていたかの可視化は大変に重要。

**MRTG**（Multi Router Traffic Grapher、1995年〜）でいこう。
今でいう **Grafana + Prometheus** の祖先。

![MacSlowCooker の MRTG 風履歴画面: Compute (GPU% + Power) と Thermal (温度 + Fan) の Daily / Weekly / Monthly / Yearly 8パネル](/images/macslowcooker/mrtg.png)


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

もう本家 MRTG でやれよ。`SwiftUI` の `ImageRenderer`（= SwiftUI の view を CGImage に焼く API、macOS 13+）で MRTG パネルを rasterize して 8枚のPNG + auto-refresh 入りの `index.html` を5分ごとに吐き出す。

```bash
python3 -m http.server -d ~/Library/Application\ Support/MacSlowCooker/web/
# http://localhost:8000/ で 8枚並んで auto-refresh
```

老人会の会員が見たら「ああ、これな」


https://github.com/hakaru/MacSlowCooker

Testflightテスターお願いします。
https://testflight.apple.com/join/Vk9S4kmn
https://testflight.apple.com/join/BAtGszPw

---

## 追加情報: MacSlowCooker Cooking Plate (コンセプト)

![MacSlowCooker Cooking Plate concept — Mac Studio の天板に乗せる調理用アルミプレートのモック](/images/macslowcooker/cooking-plate-concept.jpeg)


| Mac | 天板 | 発熱 | 鍋向き |
|---|---|---|---|
| **Mac Studio** (M1/M2/M3 Max/Ultra) | フラットなアルミ ~19×19cm | 300W 級 (Ultra) | ◎ 最高 |
| **Mac mini** (M1〜M4 Pro) | フラットなアルミ ~12.7×12.7cm | 60-90W | ◯ 弱火 |
| **Mac Pro** (M2 Ultra) | チーズグレーター穴あき | 300W+ | △ 平らじゃない |
| **MacBook Pro 16"** (M3 Max) | 閉じた状態の天板はアルミ平面 | 100W+ | △ 排熱は底面なので天板はそこまで熱くない |
| **iMac 24"** | アルミ背面、上面は丸み | 30-50W | × 鍋置けない |

実用度ランキング:

1. **Mac Studio** ─ 平らで広い + Ultra なら本当に熱い。M3 Ultra は理論上ローストビーフ低温調理可能 (60〜70℃ 帯に GPU 全開状態)
2. **Mac mini** ─ サイズ的にコーヒー保温が現実的 
3. それ以外は厳しい

…と書きつつ、Apple は天板温度を **室温+15℃前後** で抑えるよう冷却設計してるので、実際は煮込み料理には全然足りない (60℃以上必要)。**保温には十分**。

本物の調理器具ではありません。Mac Studio M3 Ultra で GPU 100% 連続稼働しても天板は ~65℃ までしか上がらないので、低温調理ローストビーフが限界です（やらないでください）。

---

## 最新追加情報: Cluster Edition (Mac mini 4台積み)

![MacSlowCooker Cooking Plate Cluster Edition — Mac mini 4台を縦に積んで上に Cooking Plate を載せた構成のコンセプト製品ページ](/images/macslowcooker/cluster-edition.jpeg)

Mac Studio 単体で物足りないあなたへ、**Cluster Edition**。Mac mini を **4台縦積み** して上に1枚の Cooking Plate を載せる構成。LLM 学習ラックの自然な形態として注目される (注目されてない)。

スペック:

| 項目 | Cluster Edition |
|---|---|
| 対応機種 | Mac mini (M1/M2/M3/M4)、最大4台積み |
| 最大調理温度 | **70℃ / 158°F** (4台並列推論時) |
| 材質 | 陽極酸化アルミニウム (積み重ね対応の強化版) |
| 到達時間 | 分散 LLM 推論時 約6分 |
| 推奨料理 | タンドール風ナン / 大鍋ソーヴィード / パン生地発酵 |
| 価格 | ¥29,800 から |

ジョークの実装としては:

- **発熱4倍** ─ 4台の Mac mini が同時に並列推論を回す = 4 × 60-90W = ~300W 級
- **空気の流れが死ぬ** ─ 縦積みすると下のミニが上のミニに排熱を浴びせるので、実際の温度は単体時より高くなる (= ジョークとしては嬉しい)
- **MacSlowCooker は per-Mac 監視 + クラスター集計対応** ─ 各台の Dock アイコンがそれぞれ別の鍋になり、合算データはダッシュボードへ

…と書きつつ、**やらないでください**。空気が回らないので Mac mini が熱で死にます。アルミ筐体の温度を上げる代わりに SoC が thermal throttle を発動して LLM 推論性能が落ちます。物理的にも経済的にも普通に loss。

ローカル LLM 学習ラックを組むなら、**横に並べて間隔を空けて** くださいね。それはそれで MacSlowCooker (per-Mac 版) で全台監視できます (本当)。
