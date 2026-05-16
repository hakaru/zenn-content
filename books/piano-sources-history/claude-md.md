# 08. CLAUDE.md — Claude Code に渡したリポジトリガイド

このシリーズは Claude Code を使いながら開発してきた。各プロジェクトには `CLAUDE.md`（Claude Code がリポジトリを開いたとき最初に読む設定ファイル）を置いてきたが、本 Zenn book リポジトリ自体にも `CLAUDE.md` を作成した。

この章では、その内容と、何をどう書いたかを記録する。

---

## CLAUDE.md とは

Claude Code（Anthropic の CLI ツール）がリポジトリで作業を始める際に自動的に読み込むマークダウンファイル。「このコードベースをどう理解すべきか」「何に注意すべきか」を書いておくことで、毎回ゼロから説明しなくて済む。

開発プロジェクトでは主に「ビルド方法」「テスト方法」「アーキテクチャの概要」を書くことが多い。本 Zenn book は実行可能なコードが主役ではなくドキュメントが主役なので、書く内容は少し違う。

---

## 作成した CLAUDE.md の内容

### このリポジトリについて

Rhodes 系エレクトリックピアノ音源を 8 週間で 5 世代開発した経緯をまとめた Zenn book（2026-03-01〜04-26）。

開発系譜：**elepiano**（Raspberry Pi 5 / C++）→ **iRhodes**（iOS）→ **TineModeler**（macOS、純物理モデル）→ **TineModeler2 / RhodEx**（Python DDSP）→ **TineModeler3 / RhoDex**（AUv3、製品版）。

`TineModeler2/3` の対外プロダクト名は `RhodEx/RhoDex`（内部コード名と外部名が異なる）。

### ドキュメント構成

- `00-overview.md` — 8 週間のタイムライン、技術スタック変遷、各世代の位置づけ
- `01〜05` — 各プロジェクトの技術詳細（elepiano, iRhodes, TineModeler, TineModeler2, TineModeler3）
- `06-evolution.md` — 技術選択の変遷と世代間継承の総括
- `07-lessons.md` — 横断的な反省と次世代への設計指針
- `TineModeler4-design.md` — 歴史的スナップショット（Rev.2、2026-05-11 時点）。最新版は `TineModeler4/docs/DESIGN.md` にある

読み方：技術詳細は 01〜05 章、学びのみなら 06〜07 章で足りる。

### experiments/

TineModeler4 設計前の事前検証実験が 2 件：

- `experiments/ddsp-warmup/` — Swift Package + Python スクリプト。`rhodex_streaming.mlpackage` の GRU 隠れ状態が CoreML 呼び出し間で更新されないことを発見（`coreml_update_state` op 欠落）。クロスフェードの根拠が「GRU ウォームアップ待ち」から「STFT レイテンシ回避（42.6ms @ 48kHz）」に変更された。

- `experiments/sample-interpolation/` — `librosa` を使った Python スクリプト。20 ノート録音 + 線形補間で 88 鍵カバーできるかを検証。アタック 30ms の Mel-L2 は許容範囲内だが、ピッチ誤差が致命的（中央値 90 cent、実音 2.1 cent）。ETI Roads MKII 採用（全 88 鍵）により補間問題は消滅。

両実験とも音声データは外付け SSD `/Volumes/Dev/` を参照しており、このリポジトリには含まれない。

### 技術的背景

全 5 世代を通じて継承された C++ コア：`SpscQueue`、`Biquad`、`DelayLine`、`FlacDecoder`、`Voice/SampleDB`、`FxChain`。プラットフォーム層（Linux ALSA → iOS AVAudioEngine → AUv3）は世代ごとに切り替わったが、C++ コアは一貫して流用されている。

ビルドシステムの変遷：elepiano（CMake + XcodeGen）→ iRhodes（Xcode 直接）→ TineModeler（Swift Package + Xcode）→ TineModeler3（Swift Package + XcodeGen + Xcode）。

3 層合成モデル（Sample + Physics + DDSP）の現在形は TineModeler3 で確立：
- **Sample 層**：アタック/リリースのみ（TineModeler4 から ETI Roads MKII 採用）
- **Physics 層**：Euler-Bernoulli tine モデル + Hunt-Crossley ハンマー + Faraday 則ピックアップ
- **DDSP 層**：GRU 残差学習 PyTorch → CoreML（stateful 変換漏れは TineModeler4 で修正）

### RT スレッド設計原則（全世代共通）

5 世代を通じて不変の制約：

- `malloc` しない
- `lock` しない
- 例外を投げない
- 状態を直接共有しない（必ずキュー越し）

実装形は世代ごとに進化：`SpscQueue<T,N>` → `paramSerialQueue` → `memory_order_acquire/release` 明示 → `swift-atomics` パッケージ対応。

### 5 つの反省点（TineModeler4 設計の出発点）

1. **聴感評価の不足** — Mel-L2 / val_loss は丹念に取られているが MOS / A/B テストが未整備。TineModeler4 では Day 0 から A/B ハーネス構築。
2. **データセット拡張の遅れ** — 学習データが rhodes-classic のみ。他機種（LA Custom, Wurlitzer 等）へ未拡張。
3. **ストア出荷の未準備** — TineModeler3 は機能完成・テスト 67/67 まで到達しているが、codesign / entitlements / sandbox 整備が残存。
4. **プラットフォームごとの再実装** — Core C++ ライブラリ / Platform シェル分離が最初から設計されていなかった。TineModeler4 では `Sources/Core/` と `Sources/Shells/` を初期から分離。
5. **法務 / IP リスク** — Keyscape SpCA の逆エンジニアリングは商用配布に法務リスクあり。TineModeler4 以降は ETI Roads MKII（埋込ライセンス確認済）に完全移行。

---

## CLAUDE.md を書いて気づいたこと

書いていると、「これは 07-lessons.md に書いたことと同じでは？」という箇所が何度か出てきた。

違いは**誰に向けて書くか**だと思う。Zenn の記事は読者（人間）に向けて経緯や背景を説明する。CLAUDE.md は将来の Claude Code インスタンスに向けて、即座に作業に入れるコンテキストを渡す。同じ内容でも、前者は「なぜそうなったか」が主役で、後者は「どうなっているか」「何に気をつけるか」が主役になる。

RT スレッドの原則や反省点を CLAUDE.md に書いたのは、このリポジトリで「コードを修正してほしい」という依頼が来たときに（そういう依頼は来ないかもしれないが）、Claude Code が既存のコードの設計思想を外さないようにするため。言語化されていない前提を明文化する、というのが CLAUDE.md の本質的な役割だと感じた。
