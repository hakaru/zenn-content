---
title: "AIコーディングアシスタントに「長期記憶」を持たせたら、プロジェクト横断でバグが見つかるようになった"
emoji: "🧠"
type: "tech"
topics: ["tidb", "mcp", "claude", "vectordb", "ai"]
published: false
---

## Claude Code、昨日の会話覚えてない問題

iOS/macOS 向けの MIDI 2.0 FM シンセサイザー **M2DX** と、録音アプリ **1Take** を並行開発してる。どっちも TestFlight でベータ配布中。

https://testflight.apple.com/join/BAtGszPw

普段の開発は Claude Code + Codex + Gemini CLI の3本立て。で、毎日使ってると気づく。

*「昨日直したあのバグ、今日のセッションでは忘れてる」*

セッションが切れるとコンテキストがリセットされる。CLAUDE.md に書けば多少は残るけど、プロジェクトをまたいだ記憶は持てない。M2DX-Core で修正した SPSC リングバッファの問題が TineModeler3 にも同じパターンであるのに、AI はそれを知らない。

で、作ったのが **memdream**。TiDB Cloud のベクトル検索を使って、AI コーディングアシスタントに「長期記憶」を持たせる仕組み。

https://github.com/hakaru/memdream

## 何が起きるようになったか

先に結果を書く。

M2DX のオーディオエンジンで `FXParamBox` というクラスが `os_unfair_lock` を使ってた。オーディオのレンダースレッドは「絶対にロックを取らない」が鉄則なんだけど、見落としてた。

Codex と Gemini の両方が同じ場所を指摘して、memdream に記録した。

その後、別のプロジェクト TineModeler3 のセッションで `memory_search("lock-free audio thread")` すると、M2DX の修正履歴がヒットする。「Atomic triple buffer で置き換えた」という具体的な解決策付き。

*プロジェクトが違うのに、同じ問題の解決策が引ける。*

これが「長期記憶」の効果。1回の修正経験が、他のプロジェクトの未来のセッションに伝播する。

## 記憶の3層構造 — ここが設計の肝

memdream の記憶は3層ある。人間の記憶と同じで、生の体験をそのまま全部覚えてるわけじゃない。

**1層目: 観測（observations）** — 生の作業ログ

「1Take の paywall バグを直した」「M2DX のオーディオスレッド安全性をレビューした」「TineModeler3 の SPSC リングを Atomic 化した」…。セッション中に起きたことをそのまま記録する。今185件ある。

**2層目: 統合記憶（consolidated memories）** — 要約された知見

観測が溜まってくると、ノイズが多すぎて検索精度が落ちる。だから dream-agent というバッチ処理で、12件ずつまとめてローカル LLM（Ollama qwen3:14b）に「要約して」と投げる。

例えば M2DX の観測12件が「M2DX RT-Safety とロックフリーアーキテクチャ修正」という1つの統合記憶に昇華される。今26件。

ここにスコープの概念がある。project（プロジェクト単位）、ecosystem（プロジェクト群単位）、global（全体）の3段階。M2DX と MIDI2Kit と M2DX-Core は m2dx-ecosystem として括られてるので、M2DX のセッションで `memory_recall` すると MIDI2Kit の修正履歴も一緒に返ってくる。

*依存してるプロジェクトの記憶を自動で引っ張ってくる。* これはプロジェクト名を手で指定するんじゃなくて、DB のエコシステム定義から自動展開される。

**3層目: ナレッジグラフ（knowledge graph）** — 関係性

```
M2DX --implements--> lock-free triple buffer
M2DX --depends-on--> MIDI2Kit
TineModeler3 --fixed--> SPSC data race
```

主語-述語-目的語のトリプル。今143件。「M2DX が何を実装してるか」「何に依存してるか」が構造的に引ける。ベクトル検索は「意味的に近いもの」を引くけど、ナレッジグラフは「構造的に繋がってるもの」を引く。両方あるから精度が出る。

## なぜ TiDB なのか — RDB の機能がそのまま使える

ベクトル DB は選択肢が多い。Pinecone、Weaviate、Qdrant、pgvector…。その中で TiDB Cloud Serverless を選んだ。

理由は「ベクトル検索以外のことも全部やりたかった」から。

memdream の記憶は「書いて、検索して、重複を弾いて、関係性を辿って、フラグで処理済みを管理する」という一連の操作の組み合わせ。ベクトル検索はその中の1つでしかない。

具体的に TiDB の RDB 機能が効いてるところ:

**重複排除。** 同じ内容を2回記録しない仕組み。`content_hash` という生成カラム（project_id + title + content の SHA2 ハッシュを DB が自動計算）に UNIQUE 制約をかけてる。`INSERT IGNORE` 1文で済む。ベクトル DB だと「まず検索して類似度を見て、閾値以上なら書かない」というアプリ側のロジックが必要になる。

**エコシステムの関係性管理。** projects テーブルに `ecosystem` カラムがあって、M2DX・M2DX-Core・MIDI2Kit が同じエコシステムに属してることを定義してる。`memory_recall` が呼ばれると、まずプロジェクトの所属エコシステムを引いて、同じエコシステム内の兄弟プロジェクトを展開して、そのプロジェクト群の統合記憶をまとめて返す。外部キーとJOINの世界。

**dream の冪等性。** observations テーブルに `consolidated` フラグ（BOOLEAN）があって、dream-agent が処理済みの観測にマークをつける。2回目の dream run は `WHERE consolidated = FALSE` で未処理分だけ拾う。初回168件 → 2回目17件、という増分処理が `UPDATE ... SET consolidated = TRUE` だけで実現できる。

*これ全部、ベクトル専用 DB だとアプリ側に持つ必要がある。* TiDB だと SQL で書くだけ。

## HTAP — 書き込みと検索が干渉しない

TiDB の特徴で一番効いてるのが HTAP（Hybrid Transactional and Analytical Processing）。

memdream は「記録」と「検索」を同時にやる。Claude Code が `memory_observe` で観測を INSERT したすぐ後に、`memory_search` でベクトル検索を走らせる。同じテーブルに対して。

TiDB はこれを行ストア（TiKV）と列ストア（TiFlash）で分担する。INSERT は行ストアに書かれて、ベクトル検索は列ストア上の HNSW インデックスで走る。物理的に別のストレージエンジンで処理されるから、書き込みが検索を遅くしたり、その逆が起きたりしない。

普通の RDB にベクトル拡張を足しただけだと（pgvector とか）、同じストレージ上で INSERT と検索が競合する。小規模なら問題ないけど、セッション中にリアルタイムで書き込みと検索が交互に走る memdream の用途だと、HTAP の恩恵がある。

## dream-agent — 「寝てる間に記憶を整理する」

人間が寝てる間に記憶を整理するように、dream-agent は観測を統合記憶に昇華する。

やってることはシンプルで:
1. 未処理の observations をプロジェクト別に拾う
2. 12件ずつチャンクにして、ローカル LLM に「要約して、関係性をトリプルで抽出して」と投げる
3. 返ってきた JSON から統合記憶とナレッジグラフを INSERT
4. 同じエコシステム内のプロジェクトをまたいだ横断分析もやる

実際に走らせた結果:

```
📊 対象プロジェクト: 6件
   - 1take (7件の未統合観測)
   - memdream (6件の未統合観測)
   - midi2kit, tinemodeler4, m2dx, m2dx-core (各1件)

🌐 m2dx-ecosystem: 3プロジェクト横断分析
  ✅ "Concurrency Management and Buffer Optimization in m2dx-ecosystem"

✨ 完了: 17観測 → 7メモリ + 32トリプル
```

m2dx-ecosystem の横断分析が面白い。M2DX と M2DX-Core と MIDI2Kit で別々に修正した並行性とバッファの問題を、AI が勝手に「このエコシステム全体の傾向」としてまとめてくれた。

*3つのプロジェクトを個別に見てたら気づかないパターンが、エコシステム単位で見ると浮かび上がる。*

冪等性も確認済み。2回目の dream run では新規17件だけが処理されて、前回の168件はスキップされた。

## 1Take での実例 — レビュー結果が自動蓄積

1Take は iOS/macOS の録音アプリ。クラウド同期（iCloud、Dropbox、Google Drive、Backblaze B2）と、最近 Pro 版（マルチカム同期録画）をリリースした。

https://apps.apple.com/us/app/1take/id6757945099

1Take のセッションで paywall 周りのバグ修正をやると、Claude Code が自動で `memory_observe` を呼んで記録してくれる。`~/.claude/CLAUDE.md` に「作業完了時は必ず memory_observe で記録すること」と書いてあるから。

実際に蓄積されたのがこういう記録:

- paywall_purchase_tapped に source パラメータ追加
- recording_saved paywall を初回保存のみ表示に変更
- v2.2.2 (build 18) アーカイブ・ASCアップロード完了
- 3モデルレビュー結果 — P-001/P-002 正確性確認

次のセッションで `memory_session_start` すると、これが統合記憶として返ってくる。「あ、昨日 paywall 周り2件直して審査に出したんだった」と、文脈ゼロから再開できる。

## MIDI2Kit — エコシステム記憶が依存先のバグを防ぐ

MIDI2Kit は M2DX が依存してる MIDI 2.0 ライブラリ。Codex にレビューさせたら CRITICAL が3件出た:

- `PEManager.deinit` が非同期の continuation を resume せずに破棄 → 呼び出し元が永久ハング
- `CoreMIDITransport` の設定プロパティが同期なしで concurrent access → データレース
- SysEx7 受信バッファがサイズ制限なしで際限なく増える → メモリ枯渇

3件とも修正して705テスト全パス。結果は memdream に記録済み。

ここからが memdream の本領。M2DX のセッションで `memory_recall(project="m2dx", scope="ecosystem")` すると、MIDI2Kit の修正内容がエコシステム記憶として返ってくる。M2DX は MIDI2Kit に依存してるから。

*依存ライブラリで直したバグが、依存元のプロジェクトの文脈に自動で入ってくる。*

手動で「あ、MIDI2Kit で先週こういう修正したから M2DX でも気をつけないと」って思い出す必要がない。DB のエコシステム定義と SQL の OR 条件が、そこを自動化してくれる。

## 数字

| 指標 | 値 |
|---|---|
| プロジェクト | 8（M2DX, M2DX-Core, MIDI2Kit, 1Take, TineModeler3, TineModeler4, PeerClock, memdream） |
| observations | 185 |
| consolidated memories | 26 |
| knowledge graph triples | 143 |
| エコシステム | 3 |
| dream runs | 2回（初回168件 → 2回目17件） |

全部ローカル（Ollama）+ TiDB Cloud Serverless の無料枠で動いてる。外部 API キー不要。TiDB の無料枠は月 25 GiB + 250M RU で、開発記憶用途だと使用量は1%以下。

## ベクトル DB じゃなくて TiDB を選ぶ理由

整理するとこう。

ベクトル検索「だけ」なら Pinecone や Qdrant でいい。でも memdream みたいに「ベクトル検索 + 重複排除 + 外部キー + フラグ管理 + エコシステム JOIN」が全部要るなら、それを全部アプリ側で実装するより TiDB に1本化した方が楽。

特に HTAP。書き込みと検索が同時に走る用途で、行ストアと列ストアが物理的に分離されてるのは安心感がある。pgvector だと同じストレージ上で競合するし、ベクトル専用 DB だとトランザクションが弱い。

MySQL 互換なのも地味に大きい。`mysql2` でそのまま繋がるから、新しいクライアントライブラリを覚える必要がない。

## これから

- dream-agent を launchd で毎晩自動実行
- TiDB の TTL（Time-to-Live）で古い observations に有効期限を付けて自動アーカイブ
- ベクトル検索 + FULLTEXT のハイブリッド検索
- Cursor や Windsurf からの MCP 接続

…TTL で `consolidated_memories` に有効期限を付けて、古い統合記憶を自動で消す。3ヶ月前のレビュー結果はもう要らないかもしれないけど、ナレッジグラフの関係性は残したい、みたいな使い分けが `expires_at` カラム1つでできる。RDB の機能がそのまま使えるのがやっぱり楽。
