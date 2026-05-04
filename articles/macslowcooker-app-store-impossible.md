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

…と書きつつ、MacSlowCooker は個人用に作ったので、配布するつもりはない。OSS にして「使いたい人は自分で署名して clone してビルド」が現実解。リポジトリは [hakaru/MacSlowCooker](https://github.com/hakaru/MacSlowCooker)（公開予定は未定）、Apache 2.0 ライセンス。

## まとめ

GPU 使用率を鍋アイコンで可視化したい、という思いつきから始まったので、最初は楽しい開発だった。

そこから **macOS 26 で `powermetrics` の plist スキーマが変わってる**ことに気付いて parser を書き直し、**SMC sampler が macOS 26 で削除されてる**ことに気付いて AppleSMC を直叩きに切り替え、**`@main` AppDelegate が macOS 26 で動かない**ことに気付いて `main.swift` で明示的に delegate をセット…と、気付くたびに地雷の解像度が上がっていく。

最後に App Store 配布の検証で「全部詰んでた」と分かったのが今回。**Mac のハードウェアを覗くタイプのアプリは構造的に App Store 不可**、というのは知識として知ってたけど、実際に自分で踏み抜いて初めて重みが分かる、という話。

iStat Menus が App Store に出ない理由が、自分で同じ穴に落ちて初めて納得できた。
