---
title: "1Take Pro — iPhoneだけでマルチカメラ同期録音。Tentacle Sync不要の時代へ"
emoji: "🎬"
type: "tech"
topics: ["ios", "swift", "video", "audio", "testflight"]
published: true
---

## TL;DR

- 1Take（プロ向けオーディオ録音アプリ）に **カメラ同期 + マルチデバイス録音** を追加した **1Take Pro** のパブリックTestFlightベータを公開しました
- **PeerClock** によるデバイス間の高精度時刻同期で、複数 iPhone/iPad を使ったマルチカメラ録音が **ハードウェア不要** で実現
- TestFlight（無料）: https://testflight.apple.com/join/Vk9S4kmn

---

## 背景：「カメラ角度を増やしたいが、同期が地獄」問題

ライブ演奏を複数アングルで撮りたい。インタビューをカメラ2台で収録したい。

**現実のワークフロー（Before）:**

1. iPhone A で音声録音 + カメラ録画
2. iPhone B でサブカメラ録画
3. 編集時に波形を目視で合わせる or クラップボードで手動同期
4. ズレてたらやり直し…

音楽制作のプロ現場では **Tentacle Sync**（タイムコード同期デバイス、約30,000円/台）が定番ですが、個人制作・YouTuber・インディーズバンドには価格と準備の手間がネックです。

---

## 1Take Pro のアプローチ

### PeerClock による時刻同期

**PeerClock**（[App Store](https://apps.apple.com/app/id6762972307)）は、LAN/Wi-Fi Direct 経由でデバイス間の時刻をサブミリ秒精度で合わせる iOS アプリです。1Take Pro はこの時刻同期基盤を利用して、録音開始タイムスタンプをデバイス間で統一します。

```
iPhone A（マスター）  ──Wi-Fi──  iPhone B（スレーブ）
        ↓                               ↓
   PeerClock 同期済み             PeerClock 同期済み
        ↓                               ↓
  1Take Pro 録音開始 ────同一タイムスタンプ────→ 1Take Pro 録音開始
```

編集ソフトに読み込んだとき、**クリップがすでに同期された状態** で並びます。

### マルチカメラ録音

- iPhone を複数台並べて、各デバイスで 1Take Pro を起動
- マスターデバイスから「録音開始」を送信すると、全スレーブが同時録画開始
- 音声・映像が同一タイムラインで記録される

---

## 競合比較

| | **1Take Pro** | **Tentacle Sync** | **FiLMiC Remote** | **マルチカム（手動）** |
|---|---|---|---|---|
| 価格 | 未定（無料ベータ中） | 約 ¥30,000/台 × n台 | 無料（FiLMiC Pro別途） | 無料 |
| ハードウェア | **不要** | 必須 | 不要 | 不要 |
| 同期精度 | サブms（PeerClock） | フレーム精度 | アプリ依存 | 手動（数フレームズレ） |
| 音声品質 | **1Take 同等**（24bit/48kHz） | カメラ内蔵マイク依存 | カメラ内蔵マイク依存 | 別録り必要 |
| 操作 | ワンタップ | タイムコード設定 | ペアリング | クラップボード |
| 編集時の同期作業 | **自動（不要）** | タイムコード読み込み | 手動 or 自動 | 手動 |
| iOS専用 | ✅ | ❌（専用機器） | ✅ | ✅ |

**ポジション**: 「Tentacle Sync が欲しいが、専用機材に予算をかけたくない」層のための Pure iOS ソリューション。

---

## 1Take との関係

**1Take**（[App Store](https://apps.apple.com/jp/app/id6757945099)）は、プロ向けオーディオ録音アプリです。VUメーター、スペクトラムアナライザー、24bit/48kHz 録音、ノイズゲートなどを備えています。

1Take Pro はその録音エンジンをそのまま継承しつつ、**カメラ録画 + マルチデバイス同期** のレイヤーを追加したものです。

---

## TestFlight で試す

現在パブリックベータ公開中です。iOS/iPadOS 18+ 対応。

👉 **https://testflight.apple.com/join/Vk9S4kmn**

フィードバックは TestFlight のスクリーンショット報告、または X（@hakaruapps）へ。

特に以下の点でご意見をいただけると助かります：

- Wi-Fi 環境での同期精度（ズレ具合）
- 2台以上での動作安定性
- 実際に使いたいシーン・ユースケース

---

## 技術的な補足

- Swift 6 / SwiftUI
- PeerClock との連携は NWBrowser（Bonjour/mDNS）経由
- 録音エンジン: AVAudioEngine（1Take と共通コードベース）
- タイムスタンプ同期: PeerClock の時刻補正値を利用した録音開始トリガー

---

## まとめ

- ハードウェア不要でマルチカメラ同期録音が iPhone だけで完結
- Tentacle Sync の代替になりうる Pure iOS ソリューション
- TestFlight 無料公開中 → ぜひお試しください

https://testflight.apple.com/join/Vk9S4kmn
