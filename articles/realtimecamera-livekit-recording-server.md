---
title: "iPhoneの映像、リアルタイムでサーバに送れる？から1日でLiveKitの録画基盤ができた"
emoji: "📹"
type: "tech"
topics: ["livekit", "webrtc", "nodejs", "typescript", "claudecode"]
published: false
---

## iPhoneで撮ってる映像、そのまま別サーバに送れる？

この一言から始まったプロジェクト。要件を整理するとこうなった。

- 遅延は数秒までOK
- カメラ（iPhone）は数台〜10台
- サーバ側で再生・録画、解析は将来やりたい
- サーバ側からカメラの起動・設定変更（ズーム・フォーカス・露出・前後切替）をしたい

先に言っておくと、iOSには「アプリを遠隔起動する」手段がない。カメラはアプリがフォアグラウンドにいる間しか使えない。なので「専用端末としてアプリを起きっぱなしにして、以降の操作は全部サーバから」という運用に倒した。ここを最初に握っておかないと、あとで全部ひっくり返る。

## 結論

1日でここまで動いた。

- LiveKit（セルフホスト）+ control-server（Node/TypeScript）+ 録画パイプライン
- E2E: テスト映像 publish → webhook で自動録画 → 無変換MP4 → 一覧API → Range 配信(206)
- テスト50件、18タスク、27コミット、+6,464行

iPhoneアプリはまだ1行も書いてない。なのに受け入れ試験が通っている。なぜかというと `lk`（LiveKit公式CLI）が ffmpeg で作った H.264 ファイルをカメラのフリして publish できるから。実機もアプリもない段階でサーバ側を完成させられるの、地味に効く。

## 方式選定: MediaMTX を蹴って LiveKit

候補は3つ。MediaMTX + SRT（既製メディアサーバ）、LiveKit（WebRTCフルスタック）、フルスクラッチ。遅延数秒OKという要件だけ見れば MediaMTX が一番楽で、実際そっちを推した。けど最終的に LiveKit にした。

決め手は制御チャネル。カメラの遠隔操作が要件にあって、LiveKit なら WebRTC のデータチャネルに **RPC**（participant 間のリクエスト/レスポンス）が標準で乗っている。ブラウザのダッシュボードから iPhone の participant に直接 `camera.setZoom` を投げて、適用結果が返ってくる。SRT 構成だとここを WebSocket + 自前プロトコルで丸ごと書くことになる。

短所は録画。「LiveKit の録画（Egress）は重い」というのが通説で…ここで調べたら通説が半分間違いだった。次の話。

## 調べて分かった LiveKit の事実5つ

実装に入る前に、2026年6月時点の現行APIをひととおり裏取りした。計画に書いたコードが古いAPIを向いてると、後工程が全部崩れるので。

### 1. Track Egress は無変換で、コンテナは選べない

Track Egress（単一トラックの録画）はトランスコードを一切しない。コーデック→コンテナの対応がソースにハードコードされている。

```go
// livekit/egress pkg/types/types.go
TrackOutputTypes = map[MimeType]OutputType{
    MimeTypeOpus: OutputTypeOGG,
    MimeTypeH264: OutputTypeMP4,   // ← H.264 は MP4 固定。選択肢なし
    MimeTypeVP8:  OutputTypeWebM,
    MimeTypeVP9:  OutputTypeWebM,
}
```

*コンテナくらい選ばせてよ…と一瞬思ったけど、H.264→MP4 しか使わないので実害ゼロ。*

むしろ「Egress は重い」はトランスコードする方式（Room Composite とか）の話。Track Egress の cpu_cost はデフォルト 0.5 で、Room Composite（4.0）の 1/8。iPhone がハードウェアエンコードした H.264 をそのまま MP4 に詰め直すだけなので、10台同時録画でもサーバCPUはほぼ無風。

### 2. webhook は raw ボディじゃないと署名検証が壊れる

LiveKit の webhook は Content-Type が `application/webhook+json`。標準と違う Content-Type なのはわざとで、`express.json()` に食われないようにするため。Authorization ヘッダの JWT に「リクエストボディの sha256」が入っていて、一度パースして再シリアライズした JSON だとハッシュが合わない。

```typescript
router.post('/livekit', raw({ type: 'application/webhook+json' }), async (req, res) => {
  const event = await receiver.receive(
    (req.body as Buffer).toString('utf8'),  // ← raw のまま渡す
    req.get('Authorization'),
  );
```

ここを `express.json()` で受けると全 webhook が 401 になる。知らずに踏むと数時間溶けるタイプの罠。

### 3. `lk room join --publish-demo` は H.264 保証がない

E2E用にデモ映像を publish しようとすると `--publish-demo` が目に入る。ところがこれ、内蔵クリップのローテーションで、butterfly/cartoon は H.264、crescent/neon/tunnel は **VP8**。Track Egress は VP8 だと WebM を吐くので、E2E が確率で壊れる。*ローテすな。*

ffmpeg で Annex-B を自作して `--publish` するのが確実。

```bash
ffmpeg -f lavfi -i testsrc2=size=1280x720:rate=30 -t 30 \
  -c:v libx264 -profile:v baseline -g 30 -f h264 test.h264
lk room join --identity cam-01 --publish test.h264 --fps 30 cameras
```

これで「カメラのフリをする lk」が手に入る。

### 4. hidden participant は RPC を呼べない

ダッシュボードの閲覧者は映像を publish しないので、つい `hidden: true` を付けたくなる。付けると RPC が全部エラーになる。仕様として「hidden participant の RPC 呼び出しは失敗する」と明記されている。カメラ操作が RPC 前提のシステムでこれを踏むと、原因がトークンの grant にあるとは思えなくてハマる。

トークン発行コードに「hidden は絶対に付けない」とコメントを書いた。未来の自分への防御。

### 5. egress は Redis 必須・非 root 実行

egress は livekit-server と Redis のキュー経由でしか会話しない。別々の Redis を見ていると "no response from egress service" という雑なエラーだけ出る。あと egress コンテナは非 root で動くので、録画出力ディレクトリは chmod 777 が要る。git はディレクトリの permission を保存しないから、fresh clone した人が必ず踏む。README の起動手順に1行足した。

## アーキテクチャ

```
iPhone × 10台（予定）── WebRTC ──┐
                                  ▼
            ┌─ VPS ──────────────────────────┐
            │ livekit-server（SFU・転送のみ）      │
            │ egress（Track Egress → 無変換MP4）   │
            │ redis（server ↔ egress の連携）      │
            │ control-server（自作 Node/TS）       │
            └──────────────┬─────────────┘
                            ▼
            ブラウザのダッシュボード（次の計画で作る）
```

全カメラを1つの Room に集約して、participant identity = デバイスID。自作したのは control-server だけで、責務はこう。

- デバイス認証（キーは sha256 で保存、トークンは publish 専用）
- 運用者ログインと subscribe + RPC 用トークン発行
- webhook 受信 → 録画の自動開始/終了
- 録画一覧 API と MP4 の Range 配信
- 保持期限切れの削除、ディスク残量監視 → 逼迫したら録画停止

録画ポリシーはデバイスごとに auto / manual の二択。manual の ON 状態（armed）は DB に永続化してあって、回線が瞬断して publish が復帰すると `track_published` webhook 経由で録画が勝手に再開する。「手動録画中に電波が切れたら録画も終わり」だと運用で確実に事故るので、ここだけは設計時点で作り込んだ。

## 作り方: 計画書にコードを全部書いてから、タスクごとにサブエージェントへ

今回は Claude Code のサブエージェント駆動で作った。流れはこう。

1. 設計をブレストで固める（ここで MediaMTX → LiveKit の転換が起きた）
2. 現行APIのリサーチを並列エージェント4本 + 裏取り3本で回す
3. 実装計画を書く。**3,467行**。全18タスクに失敗するテストのコードと実装コードを丸ごと埋め込む
4. タスクごとに新しいサブエージェントが実装 → 仕様レビュー → 品質レビューの2段確認

「計画書にコードを全部書く」のは過剰に見えて、レビューの質に直結した。レビュアーは計画と差分を突き合わせるだけでよくなるので、指摘が具体的になる。実際に拾われた問題がこれ。

- デバイスキー検証が `===` の文字列比較 → `timingSafeEqual` に（タイミング攻撃面）
- ブラウザの二重送信で UNIQUE 制約違反が生の 500 で漏れる → 409 に変換
- DB の冪等性テストが `:memory:` を2回開いていて、実は何も検証していなかった → 実ファイル再オープンに書き直し
- 最終レビュー: `egress_ended` webhook を取りこぼすと active 行が残り続けて、そのカメラの録画が**無音で恒久停止**する → 起動時に LiveKit 側の実在 egress と突合して回収する経路を追加

最後のやつはレビューなしでは本番で確実に踏んでた。webhook ベースの状態管理は「届かなかったとき」の回収経路を最初から設計に入れるべき、という学び。

## 単体テスト50件が全部緑なのに、E2E で 500

一番おいしかったバグの話。

録画MP4を配信する `res.sendFile` は**絶対パス必須**。パス解決は `path.join(recordingsDir, filename)` で書いてあって、単体テストは `mkdtemp`（絶対パスを返す）で通る。ところが開発用 .env は `RECORDINGS_DIR=../deploy/dev/recordings`、相対パス。join しても相対のままで、E2E で初めて 500 が出た。

修正は `join` → `resolve` の一語。再発防止に「recordingsDir が相対でも絶対パスを返す」回帰テストを足した。

「テストが全部通る」と「動く」の間には、設定ファイルの中身という名の谷がある。1日の中でこれを体験できるの、教材としてよくできてる。

## 次

計画2が dashboard（React + @livekit/components-react でライブグリッドとカメラ操作パネル）、計画3が iOS アプリ本体。E2E が lk で通っているので、iOS は「lk がやってることを Swift SDK でやる」だけ…のはず。

…のはず、が一番怖いんだけど。
