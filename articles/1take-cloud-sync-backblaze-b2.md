---
title: "録音アプリに4つのクラウド同期を実装した話（Backblaze B2が安すぎる）"
emoji: "☁️"
type: "tech"
topics: ["ios", "swift", "cloudstorage", "backblaze", "music"]
published: true
---

自分で作ってる音楽録音アプリ「1Take」に、録音ファイルのクラウド自動同期を実装した。対応したのはiCloud Drive、Dropbox、Google Drive、Backblaze B2の4つ。

https://apps.apple.com/app/id6738847498

で、結論から言うと**Backblaze B2が安すぎて笑った**ので、その話を中心に書く。

## なぜクラウド同期が必要だったか

1Takeはリハーサルやライブを録音するアプリなので、WAVファイルがどんどん溜まる。16bit/48kHzステレオで1分あたり約10MB。1時間のリハなら600MB。

iPhoneのストレージを圧迫するし、機種変したらデータ移行が面倒。「録ったら勝手にクラウドに上がってほしい」というのは自分自身の切実なニーズだった。

## 4つのサービスを比較する

まず各サービスの料金を整理しておく。

### 無料枠

| サービス | 無料容量 |
|---------|---------|
| iCloud Drive | 5GB |
| Google Drive | 15GB |
| Dropbox | 2GB |
| Backblaze B2 | 10GB |

### 有料プラン

iCloud、Google Drive、Dropboxは月額定額制。最安プランはこんな感じ。

| サービス | 容量 | 月額 | GB単価 |
|---------|------|------|--------|
| iCloud+ | 50GB | ¥130 | ¥2.6/GB |
| Google One | 100GB | ¥250 | ¥2.5/GB |
| Dropbox Plus | 2TB | ¥1,500 | ¥0.75/GB |
| Backblaze B2 | 従量制 | 10GB無料+$0.006/GB | **約¥0.9/GB** |

Dropboxは2TBからしかプランがないので、少量しか使わない人には割高。逆に大量に使うなら安い。

## Backblaze B2の料金を具体的に計算してみる

ここが一番伝えたいところ。

B2の料金体系はシンプルで、ストレージが**$6/TB/月**（$0.006/GB/月）。最初の10GBは無料。ダウンロード（エグレス）は保存量の3倍まで無料。

音楽録音の実際の使用量で計算してみる。

### ケース1: 週2回、1時間のバンドリハを録音（WAV 16bit）

- 1回 = 約600MB
- 月8回 = 4.8GB/月
- 1年で約58GB

**B2の月額コスト（1年後）:**
58GB - 10GB（無料枠）= 48GB × $0.006 = **$0.29/月（約¥44）**

iCloudだと50GBプラン（¥130/月）に収まるかギリギリ。超えたら200GBプラン（¥400/月）に上げる必要がある。

### ケース2: 毎日ピアノ練習を30分録音（WAV 16bit）

- 1日 = 約300MB
- 月30日 = 9GB/月
- 1年で約108GB

**B2の月額コスト（1年後）:**
108GB - 10GB = 98GB × $0.006 = **$0.59/月（約¥89）**

iCloudだと200GBプラン（¥400/月）が必要。Google Oneでも100GBプラン（¥250/月）。B2なら¥89。

### ケース3: MP3で録音する人（256kbps）

MP3なら1分あたり約2MB。ケース1と同条件だと月960MB。1年で約11.5GB。

**B2の月額コスト（1年後）:**
11.5GB - 10GB = 1.5GB × $0.006 = **$0.009/月（約¥1.4）**

月1円。もはや誤差。

## なぜB2を選択肢に入れたか

正直に言うと、iCloud / Dropbox / Google Driveだけで十分だと最初は思ってた。

でも自分がB2ユーザーだったのと、「録音データのバックアップに月¥400払うのは高い」と感じる人は多いだろうなと思った。自分がそうだから。

B2はUIがエンジニア向けで、一般ユーザーには敷居が高い。でもアプリ側でKey IDとApplication Keyを入力するだけにしたら、そこまで難しくない。バケット名も入れてもらう必要はあるけど、B2のダッシュボードでバケットを1つ作るだけ。

## 実装の話

### 認証方式がバラバラ

4つのサービスで認証方式が全部違うのが一番大変だった。

- iCloud Drive → 認証不要（システムのApple IDを使う）
- Dropbox → OAuth2 PKCE（`ASWebAuthenticationSession`）
- Google Drive → OAuth2 PKCE（同上、ただしエフェメラルセッション不可）
- Backblaze B2 → Application Key（APIキー直接入力）

Dropboxはエフェメラル（シークレット）ブラウザセッションが使えるけど、Googleはブロックしてくる。なのでGoogleだけ`prefersEphemeralWebBrowserSession = false`にしてる。

### 自動同期の仕組み

録音が完了したタイミングで、有効なプロバイダー全部に並列でアップロードする。Wi-Fi制限のオン/オフも設定できる。

ネットワークが切れてたら`.pending`ステータスにして、`NWPathMonitor`でネットワーク復帰を検知したらリトライ。最大5件ずつ。

### リモートパスの設計

```
/1Take/2026-03/MyRecording.wav
/1Take/2026-03/MyRecording.markers.json
```

年月でフォルダを切ってる。マーカーファイル（録音中に付けたGOOD/BAD/MEMOマーカー）も一緒にアップロードする。

### ファイルはメモリに載せない

WAVファイルは数百MBになるので、`URLSession.upload(for:fromFile:)`でディスクから直接ストリーミング。B2の場合はアップロード前にSHA1ハッシュが必要なので、64KBチャンクで分割計算してる。

## セットアップ手順（B2の場合）

1. [backblaze.com](https://www.backblaze.com/cloud-storage) でアカウント作成（10GBまで無料、クレカ不要）
2. B2 Cloud Storageで「Create a Bucket」→ バケット名を決める（例: `my-music-backup`）
3. App Keys → 「Add a New Application Key」で、作ったバケットだけに権限を絞ったキーを発行
4. 1Takeの設定 → Cloud Providers → Backblaze B2 → Key ID、Application Key、Bucket Nameを入力
5. Auto Syncをオンにする

これで録音するたびに自動でB2にアップロードされる。

## 自分の運用

自分はB2をメインにして、iCloud Driveもサブで有効にしてる。B2は長期アーカイブ、iCloudは直近のファイルにMacからすぐアクセスしたいとき用。

月のコストはB2が¥50くらい。iCloudは元々50GBプランに入ってるのでそのまま。

正直、¥50でリハ音源が全部クラウドにバックアップされるのは安すぎる。iPhoneが壊れても録音データは無事、というのは安心感がある。

## まとめ

| やりたいこと | おすすめ |
|------------|---------|
| とにかく手軽に | iCloud Drive（設定ゼロ） |
| 無料で始めたい | Google Drive（15GB無料） |
| 安く大量に保存したい | Backblaze B2（10GB無料、以降¥0.9/GB） |
| 既にDropbox使ってる | Dropbox |

個人的にはB2推し。特にWAVで録る人は容量がすぐ膨らむので、従量制の安さが効いてくる。1年録り続けても月¥100いかない。

1Takeは[App Storeで配信中](https://apps.apple.com/app/id6738847498)。クラウド同期は無料機能なので、よかったら試してみてください。
