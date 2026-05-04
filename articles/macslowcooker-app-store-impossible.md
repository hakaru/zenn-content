---
title: "GPU 使用率を Dock の鍋アイコンで可視化したかった ─ App Store に出せない件"
emoji: "🍲"
type: "tech"
topics: ["macos", "swift", "appstore", "xpc", "powermetrics"]
published: false
---

![MacSlowCooker の Dock アイコン: idle → 高負荷 → 沸騰 → クールダウン](/images/macslowcooker/hero.gif)

Mac の GPU 使用率と温度を、Dock の鍋アイコンで眺められたら楽しくない?

ということで作った。**MacSlowCooker** という名前の常駐アプリ。GPU が忙しくなると鍋の下の炎がでかくなって、SoC 温度が上がると鍋が赤くなって、ファンが回ると湯気がモコモコ立つ。Activity Monitor の "GPU の履歴" みたいな数字の羅列じゃなくて、鍋として「煮込まれてる感」を出したい、ただそれだけのアプリ。

…で、最後に App Store 配布できるか確認したら **無理**だった。理由が macOS の権限モデルの解像度を上げる教材としてちょうどよかったので、企画から「出せない」までを記事に残しておく。

## 結論

❌ App Store 配布は無理。理由は3つ:

- root の `LaunchDaemon` が使えない（App Store は `SMAppService.loginItem` しか許さない）
- Sandbox の中から `/usr/bin/powermetrics` を spawn できない
- `IOHIDEventSystem` と `AppleSMC` を直叩きしている（私的 API、Reject 確定）

iStat Menus も Stats も TG Pro も、ハードウェアを覗くタイプの常駐アプリは全部 Notarized DMG 配布。App Store にあるのは `task_info()` 程度しか見ない「軽量版」だけ。**「Mac のハードウェアを覗くアプリ」と「App Store」は構造的に両立しない**。

詳しくは後半で。先に企画と実装の話。

## 企画

GPU 使用率を画面の隅で常時眺めたい、というのが出発点。理由:

- LLM のローカル推論をやっていると、GPU が今ヒマか忙しいか **チラ見できない**ストレスが溜まる
- Activity Monitor の GPU タブをわざわざ開きに行くと「動いてるのを動いてるか確認しに行く」体験になって萎える
- Dock アイコンで完結できれば、ウィンドウを切り替えずに済む

ただ「数字が出てるだけ」だとつまらない。せっかく Dock に居座らせるなら、見ていて楽しい何かにしたい。

そこで **鍋メタファー**。

| 物理量 | 鍋での表現 |
|---|---|
| GPU 使用率 | 鍋の下の炎の高さ |
| SoC 温度 | 鍋のボディの色（白 → 赤橙） |
| ファン RPM | 蓋から立ち昇る湯気の本数・太さ |
| `thermal_pressure` | 沸騰演出（蓋ガタガタ + 赤い湯気） |

GPU が忙しい = 火が強い = 鍋が煮える。温度が上がる = 鍋が赤くなる。ファンが本気を出す = 湯気がモコモコ。**全部物理的に同じ向きに動く**ので、メタファーとして破綻しない。

名前は **MacSlowCooker**。スロークッカーは「弱火でずっと煮込む鍋」だけど、その逆向き — *強火でフル稼働してたら鍋が泣くよ* — というニュアンス。

## 実装

ざっくり構成:

```
MacSlowCooker.app（非特権、ユーザーセッション）
  ├── DockIconAnimator       — Timer-driven state machine（補間 / wiggle / 沸騰フェード）
  ├── DutchOvenRenderer      — Core Graphics で鍋・炎・湯気を毎回描画
  ├── PopupView              — SwiftUI + Swift Charts ダッシュボード
  └── XPCClient              — NSXPCConnection (.privileged)、2 Hz でポーリング

HelperTool（root LaunchDaemon、Contents/MacOS/HelperTool）
  ├── PowerMetricsRunner     — /usr/bin/powermetrics 常駐、NUL 区切り plist パース
  ├── IOAcceleratorReader    — IOAccelerator → "Device Utilization %"
  ├── SMCReader              — AppleSMC 直叩き、F0Ac/F1Ac で fan RPM
  └── TemperatureReader      — IOHIDEventSystem 経由で SoC 温度
```

ポイントだけ:

- **GPU 使用率は IOAccelerator の `Device Utilization %`** を直読み。Activity Monitor が出してる数字と一致する。powermetrics の `idle_ratio` を使うと、Activity Monitor とちょっとずれる
- **Fan RPM は AppleSMC を直叩き**。`powermetrics --samplers smc` は macOS 26 で削除されてしまったので、`IOServiceMatching("AppleSMC")` → `IOConnectCallStructMethod` で `FNum` と `F[i]Ac`（fpe2 形式）を読む
- **SoC 温度は IOHIDEventSystem**。M3 Ultra には「GPU MTR Temp Sensor」という名前のセンサーが存在せず、`PMU tdie*` / `PMU tdev*` を 77 個拾って平均する
- **Universal Binary** (arm64 + x86_64) なので Apple Silicon でも Intel でも動く。Intel powermetrics は `gpu_busy` / `busy_ns` を出すので、parser が両系統に対応

ここまでは macOS の API を引っ張れば普通に作れる。**問題は配布の段になってから**。

## App Store に出せない件（本題）

App Store のレビューガイドラインと Sandbox を順番に見ていくと、このアプリは詰むポイントが3つある。

### 1. root の LaunchDaemon が使えない

`powermetrics` は **root じゃないと動かない**。普通のアプリは root を取れないので、`SMAppService.daemon` で root の LaunchDaemon を「ヘルパーツール」として登録し、XPC 経由で叩く。Apple が用意した正規ルート。

…なんだけど、App Store のレビューガイドライン的には `SMAppService.loginItem`（ユーザー権限のログイン項目）しか認められない。`SMAppService.daemon` を使った時点で **Reject 確定**。

> Apps that need to run continuously in the background should use SMAppService.loginItem.

これだけでもう詰む。

### 2. Sandbox の中から powermetrics を spawn できない

仮に root daemon を諦めて、ユーザー権限で頑張ろうとしても次の壁。

App Store アプリは Sandbox が**必須**。Sandbox 下では `Process` で `/usr/bin/powermetrics` を spawn しようとしても、`powermetrics` 自体が root を要求するし、root じゃなくても **任意のプロセスを spawn する権限**自体が Sandbox にない（`com.apple.security.temporary-exception.process.spawn` みたいな entitlement は App Store では通らない）。

> Apps must be sandboxed and may not include private APIs.

### 3. 私的 API を使っている

`IOHIDEventSystemClientCreate` / `IOHIDServiceClientCopyEvent` / `IOHIDEventGetFloatValue` あたりは全部 **私的 API**。AppleSMC の `IOConnectCallStructMethod` も同様。

```swift
@_silgen_name("IOHIDEventSystemClientCreate")
private func IOHIDEventSystemClientCreate(_ allocator: CFAllocator?) -> Unmanaged<AnyObject>?
```

`@_silgen_name` で直接シンボルを拾ってる時点で App Store のスタティック解析に引っかかる。これは powermetrics・Stats・iStat Menus・Chromium の `m1_sensors_mac.mm` 全員が同じ事をやっていて、**他にやり方がない**。Apple が公開 API を出してくれれば変わるけど、現状そうなっていない。

### 同類アプリの実例

App Store に出ていない常駐モニター系アプリの一覧:

| アプリ | 配布形態 | 理由 |
|---|---|---|
| iStat Menus | Notarized DMG（公式サイト） | 同上 |
| Stats（OSS） | Notarized DMG（公式サイト） | 同上 |
| TG Pro | Notarized DMG（公式サイト） | 同上 |
| Memory Cleaner | App Store にあり | `task_info()` だけ使う「軽量版」だから OK |

App Store にいるのは「OS が提供する公開 API だけで取れる情報」しか出してないアプリだけ。**ハードウェアを覗いた時点で App Store ルートは閉じる**。

### 残された配布ルート

App Store がダメなら **Notarized DMG**。Apple Developer ID で署名 → `xcrun notarytool submit` で公証 → `stapler staple` でチケット添付 → DMG にして配布。Gatekeeper の警告なしでインストールできる。

課金したい場合は Stripe / Gumroad / Paddle を経由する。App Store の30%（Apple Tax）も払わなくていい代わりに、**自前で売り場を作る必要**はある。

…と書きつつ、MacSlowCooker は個人用に作ったので、配布するつもりはない。OSS にして「使いたい人は自分で署名して clone してビルド」が現実解。リポジトリは [hakaru/MacSlowCooker](https://github.com/hakaru/MacSlowCooker)、Apache 2.0 ライセンス。

## その後 ─ MRTG (1995年) リスペクトに走る

App Store がダメだった件はもういいとして、自分で使ってるうちに気になってきたのが「**過去の履歴を振り返れない**」こと。`@Observable` ring buffer に60サンプルだけ持ってて、つまり過去30秒しか見えない。

ローカル LLM のトレーニングを一晩回して「あの温度推移、後で振り返りたい」って何度かなって、*60サンプルじゃなあ…* と思っていた。

で、思い出したのが **MRTG**。Multi Router Traffic Grapher、1995年〜。SNMP polling して Daily/Weekly/Monthly/Yearly の4枚の PNG を吐く、淡黄色背景に濃緑塗りのあのグラフ。

> （知らない世代向け）昔のサーバ監視は cron で MRTG を回して、Apache が `~/Public_html/mrtg/` を serve するだけで、ブラウザで4時間軸の使用率グラフが見られた。今でいう Grafana + Prometheus の祖先。

これを Mac の GPU 監視に持ち込んだら *いい感じに懐かしくないか* と思って、3つの出力系統を生やした。

1. **アプリ内 History ウィンドウ** (Cmd+Shift+H) — MRTGっぽい4段グラフ
2. **Prometheus エクスポーター** — `/metrics` HTTP エンドポイント
3. **PNG 書き出し** — 8枚のPNG + index.html を定期書き出し（本物のMRTGそのまま）

実装は1日。でも *MRTG感* を出すのにデザイン6往復した。

### Stage 1: SQLite ラウンドロビン + 履歴ビューワ

データを貯める層は素直に SQLite。MRTG は元々 RRDtool っぽいラウンドロビンだけど、Mac で1台分のメトリクスなら4階層のテーブルで足りる。

| 解像度 | 保持期間 | パネル名 |
|---|---|---|
| 5min | 24h | Daily |
| 30min | 7d | Weekly |
| 2hour | 31d | Monthly |
| 1day | 400d | Yearly |

XPC で 2Hz で上がってくるサンプルを5分間メモリにバッファ → 平均を5min テーブルに insert → 5min が30min境界を跨いだら30minへ rollup → 30min が2hr境界跨いだら…という cascading rollup。pure logic は `Shared/HistoryAggregator.swift` に切り出して in-memory SQLite で完結ユニットテスト。

ここまでは普通。デザインで詰まった。

### MRTG感の追求（6往復）

最初に作ったやつ:

- ダーク背景
- 単色の緑塗り
- 軸ラベルなし、タイトルなし

自分（ユーザー兼開発者）の感想:

> 「MRTGぽくない、全然」

そらそう。MRTG の特徴を順番に思い出して直していった。

| 仮説 | やったこと |
|---|---|
| ✅ 1パネル2系統 | Compute = GPU% (緑塗り) + Power W (青線)、Thermal = SoC °C + Fan RPM。dual Y axis |
| ✅ 白背景 + MRTG-green | 背景 `Color.white`、塗り `(0, 0.80, 0)`、線 `(0, 0.40, 0)`、青線 `(0, 0, 0.80)` |
| ✅ 高密度の縦線 | 1h ごとに細い線、4h ごとに濃い線（granularity 別に密度可変） |
| ✅ X軸は実時間 | `-24h` じゃなくて `00 02 04 06 ... 22`。Daily は 2h ステップで13ラベル、Weekly は曜日略、Monthly は日付、Yearly は月略 |
| ✅ Y軸ラベルに単位 | `100% 75% 50% 25% 0%` / `150W 112W ...` / `4500rpm ...` Mac Studio (4000+rpm) で `4000rpm` がクリップしたので幅 44→52px |
| ✅ グラフ上部に余白 | 100% より上に 10px。MRTG の特徴 |
| ✅ フォント | タイトル帯は SF Pro Rounded semibold、軸と数値は **Menlo**（Retina で角がシャープ） |

全部直したら *MRTG感* は出た。Cmd+Shift+H で開いて、Compute/Thermal は Picker で切替、30秒ごとに自動更新。

### Stage 2: Prometheus エクスポーター

履歴をアプリ内で見るだけじゃなくて、**外から scrape させたい**。Grafana に流せたら一気に「ちゃんとした監視」になる。

`Network.framework` の `NWListener` で HTTP/1.1 を手書き。GET `/metrics` だけサポート、それ以外は404。

```
# HELP macslowcooker_gpu_usage_ratio GPU usage as 0..1 ratio
# TYPE macslowcooker_gpu_usage_ratio gauge
macslowcooker_gpu_usage_ratio 0.42
macslowcooker_gpu_power_watts 8.4
macslowcooker_temperature_celsius 67.2
macslowcooker_fan_rpm{fan="0"} 1850
macslowcooker_fan_rpm{fan="1"} 2100
macslowcooker_thermal_pressure 0
macslowcooker_helper_connected 1
```

**デフォルトはオフ**。Preferences の Toggle で起動。デフォルト port 9091、127.0.0.1 のみ。「Bind to all interfaces」にすると LAN から scrape できる（macOS firewall プロンプトが出る）。

設計:

- `PrometheusFormatter` (純粋関数) は `Shared/` に置いてユニットテスト
- `PrometheusExporter` 自体は private な serial DispatchQueue で listener と snapshot を保護
- `NWParameters.allowLocalEndpointReuse = true` で SIGKILL → TIME_WAIT → 再起動の `EADDRINUSE` を回避

```bash
curl -s http://127.0.0.1:9091/metrics | head
```

### Stage 3: PNG 書き出し（本物のMRTGそのまま）

ここまでくると「もう本家MRTGそのままやれば？」という気持ちになる。`SwiftUI` の `ImageRenderer` で `MRTGGraphView` を rasterize。

```swift
let view = MRTGGraphView(records: rows, panel: panel, granularity: g, nowTs: nowTs)
    .frame(width: 600, height: 140)
let renderer = ImageRenderer(content: view)
renderer.scale = 2  // retina
guard let cgImage = renderer.cgImage else { return }
let bmp = NSBitmapImageRep(cgImage: cgImage)
let png = bmp.representation(using: .png, properties: [:])
try png.write(to: url, options: .atomic)
```

これを 8枚（Compute/Thermal × Daily/Weekly/Monthly/Yearly）+ `index.html`（`<meta http-equiv="refresh" content="60">` 入り）で吐き出す。5分ごとに `Timer` で再描画。

```bash
python3 -m http.server -d ~/Library/Application\ Support/MacSlowCooker/web/
# http://localhost:8000/ で 8枚並んで auto-refresh
```

これは **本物のMRTGそのままの workflow**。1995年の人が見たら「ああ、これな」って言うはず。

### ハマりポイント

1. **GPU% のスケール** ── GPUSample が `0..1` ratio なのに `gpuPct` という名前で生で突っ込んで Y軸 yMax=100 に張り付かせた。気付くまで *履歴ウィンドウが完全に黒* だった。テストでは `gpuUsage: 42.5` とか書いてたから検出できなかった。
2. **`Shared/` に Display 型を置きそうになった** ── `HistoryPanel` を Shared に置いたら HelperTool 側にもコンパイルされて pure-helper pattern 違反。`MacSlowCooker/` に移動。
3. **`ImageRenderer.cgImage` が nil 返す疑惑** ── ドキュメントには「最初の呼び出しで nil の可能性」と書いてある。実機・XCTest 両方で nil は観測されず、結局 double-call workaround は不要だった。
4. **Preferences ウィンドウ伸びすぎ** ── 機能追加3回でセクションが増えて固定高さに収まらなくなった。`.scrollDisabled(true)` を消して fallback で scroll させる。

## まとめ

GPU 使用率を鍋アイコンで可視化したい、という思いつきから始まったので、最初は楽しい開発だった。

そこから **macOS 26 で `powermetrics` の plist スキーマが変わってる**ことに気付いて parser を書き直し、**SMC sampler が macOS 26 で削除されてる**ことに気付いて AppleSMC を直叩きに切り替え、**`@main` AppDelegate が macOS 26 で動かない**ことに気付いて `main.swift` で明示的に delegate をセット…と、気付くたびに地雷の解像度が上がっていく。

App Store 配布の検証で「全部詰んでた」と分かったのが最初の山。**Mac のハードウェアを覗くタイプのアプリは構造的に App Store 不可**、というのは知識として知ってたけど、実際に自分で踏み抜いて初めて重みが分かる。iStat Menus が App Store に出ない理由が、自分で同じ穴に落ちて初めて納得できた。

App Store ダメ → だったら配布する気もないし好きに作ろう、と思って生やしたのが MRTG リスペクトな History ビューワ + Prometheus + PNG export。出力は3系統:

- **手元で見る** → アプリ内 History ウィンドウ (Cmd+Shift+H)
- **Grafana に流す** → Prometheus `/metrics`
- **静的 HTML として serve** → PNG + index.html

…なんだけど、たぶん大半の人は「MRTGって何？」ってなる気がする。それはそれで *よい*。1995年のツールへのオマージュなので。

リポジトリ: https://github.com/hakaru/MacSlowCooker
