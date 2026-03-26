---
title: "App Store審査の監視を自動化するGitHub Actionを作った"
emoji: "🚀"
type: "tech"
topics: ["githubactions", "ios", "appstore", "apple", "automation"]
published: true
---

## TL;DR

App Store Connectの審査ステータスを3時間ごとに自動チェックし、変化があればGitHub Issue / Slack / Discord / Teamsに通知するGitHub Actionを作りました。

https://github.com/marketplace/actions/app-store-review-monitor

```yaml
- uses: hakaru/appstore-review-monitor@v1
  with:
    app-id: ${{ secrets.APP_STORE_APP_ID }}
    asc-key-id: ${{ secrets.ASC_KEY_ID }}
    asc-issuer-id: ${{ secrets.ASC_ISSUER_ID }}
    asc-private-key: ${{ secrets.ASC_PRIVATE_KEY }}
```

## 作った動機

iOSアプリを個人開発していると、App Storeに提出した後の審査待ちが地味にストレスです。

- 「そろそろ審査終わったかな…」とApp Store Connectを何度もリロード
- リジェクトされてたのに気づくのが遅れて対応が後手に
- 承認されてたのに気づかずリリースが遅れる

**「提出したら忘れて、変化があったら教えてくれ」** を実現したくて作りました。

## 仕組み

```
GitHub Actions (cron: 3時間ごと)
  ↓
App Store Connect API (JWT認証 / ES256)
  ↓
最新バージョンのステータス取得
  ↓
前回のステータスと比較（GitHub Issueのラベルで管理）
  ↓ 変化あり
GitHub Issue作成 + Slack/Discord/Teams通知
```

### ポイント

**1. 前回ステータスの保存方法**

GitHub Actionsには永続ストレージがないため、`asc-monitor`ラベルのIssueをステータスキャッシュとして使っています。ステータス変化時に古いIssueをクローズして新しいIssueを作成。外部DBやファイルストレージ不要。

**2. JWT認証（ES256）**

App Store Connect APIはJWT認証が必要です。Node.jsの`crypto`モジュールだけで実装しています。

```javascript
const sign = crypto.createSign('SHA256');
sign.update(signingInput);
const signature = sign.sign(
  { key: privateKey, dsaEncoding: 'ieee-p1363' },
  'base64url'
);
```

`dsaEncoding: 'ieee-p1363'`がポイント。これがないとDER形式になりApp Store Connect APIが受け付けません。

**3. リジェクト時の詳細取得**

リジェクトを検出した場合、`appStoreReviewDetail`と`reviewSubmissions`エンドポイントからできる限り詳細を取得してIssueに記録します。

## 通知チャネル

GitHub Issues以外に、Webhook URLを設定するだけで通知先を追加できます。

| 通知先 | 設定 | 形式 |
|--------|------|------|
| GitHub Issues | 常時有効 | ラベル付きIssue |
| Slack | `slack-webhook-url` | カラー付きアタッチメント |
| Discord | `discord-webhook-url` | Embedメッセージ |
| Microsoft Teams | `teams-webhook-url` | Adaptive Card |

全てオプショナル。複数同時指定OK。

## ステータス一覧

| ステータス | 絵文字 | 意味 |
|-----------|--------|------|
| WAITING_FOR_REVIEW | 🕐 | 審査キューに入った |
| IN_REVIEW | 🔍 | 審査中 |
| REJECTED | 🚨 | リジェクト |
| READY_FOR_DISTRIBUTION | 🎉 | 承認！ |
| PROCESSING_FOR_DISTRIBUTION | ⏳ | 配信準備中 |
| PENDING_DEVELOPER_RELEASE | 📦 | 手動リリース待ち |

## セットアップ

### 1. App Store Connect APIキーの作成

[App Store Connect](https://appstoreconnect.apple.com/access/integrations/api) > Users and Access > Integrations > Team Keys でAPIキーを作成し、`.p8`ファイルをダウンロード。

### 2. GitHub Secretsに登録

```bash
gh secret set ASC_KEY_ID --body "YOUR_KEY_ID"
gh secret set ASC_ISSUER_ID --body "YOUR_ISSUER_ID"
gh secret set ASC_PRIVATE_KEY < AuthKey_XXXXXXXX.p8
gh secret set APP_STORE_APP_ID --body "YOUR_APP_ID"

# 任意: 通知チャネル
gh secret set SLACK_WEBHOOK --body "https://hooks.slack.com/services/..."
gh secret set DISCORD_WEBHOOK --body "https://discord.com/api/webhooks/..."
```

App IDは App Store ConnectのURL `https://appstoreconnect.apple.com/apps/XXXXXXXXXX/appstore` から確認できます。

### 3. ワークフローを追加

```yaml
name: App Store Review Monitor
on:
  schedule:
    - cron: '0 */3 * * *'  # 3時間ごと
  workflow_dispatch: {}     # 手動実行も可能

jobs:
  check:
    runs-on: ubuntu-latest
    permissions:
      issues: write
    steps:
      - uses: hakaru/appstore-review-monitor@v1
        with:
          app-id: ${{ secrets.APP_STORE_APP_ID }}
          asc-key-id: ${{ secrets.ASC_KEY_ID }}
          asc-issuer-id: ${{ secrets.ASC_ISSUER_ID }}
          asc-private-key: ${{ secrets.ASC_PRIVATE_KEY }}
          slack-webhook-url: ${{ secrets.SLACK_WEBHOOK }}       # 任意
          discord-webhook-url: ${{ secrets.DISCORD_WEBHOOK }}   # 任意
          teams-webhook-url: ${{ secrets.TEAMS_WEBHOOK }}       # 任意
```

## 他のActionと組み合わせる

outputsを使って後続の処理を分岐できます。

```yaml
- uses: hakaru/appstore-review-monitor@v1
  id: review
  with:
    app-id: ${{ secrets.APP_STORE_APP_ID }}
    asc-key-id: ${{ secrets.ASC_KEY_ID }}
    asc-issuer-id: ${{ secrets.ASC_ISSUER_ID }}
    asc-private-key: ${{ secrets.ASC_PRIVATE_KEY }}

# 承認されたらデプロイ
- if: steps.review.outputs.status == 'READY_FOR_DISTRIBUTION'
  run: echo "App approved!"

# リジェクトされたらSlack通知
- if: contains(steps.review.outputs.status, 'REJECTED')
  run: echo "Rejected. See issue #${{ steps.review.outputs.issue-number }}"
```

| Output | 説明 |
|--------|------|
| `status` | 現在の審査ステータス |
| `version` | バージョン文字列 |
| `changed` | ステータスが変化したか（`true`/`false`） |
| `issue-number` | 作成されたIssue番号 |

## 既存ツールとの比較

| ツール | 審査ステータス監視 | 通知先 | Marketplace |
|--------|-------------------|--------|-------------|
| appstore-status-bot | あり | Slackのみ | なし |
| ZReviewTender | なし（レビュー収集） | Slack | あり |
| **appstore-review-monitor** | **あり** | **Issues + Slack + Discord + Teams** | **あり** |

## まとめ

「提出したら忘れる」を実現するために作ったGitHub Actionです。

- セットアップ5分
- 外部サービス不要（GitHub Actionsだけで完結）
- Slack / Discord / Teams対応
- 無料・オープンソース

https://github.com/marketplace/actions/app-store-review-monitor

フィードバックやPRお待ちしています。
