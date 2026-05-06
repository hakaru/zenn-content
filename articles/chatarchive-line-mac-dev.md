---
title: "LINE for Mac の暗号化DBをメモリからキー取得して読む macOS アプリを作った"
emoji: "💬"
type: "tech"
topics: ["swift", "swiftui", "macos", "sqlite", "reverseengineering"]
published: false
---

自作の macOS アプリ「ChatArchive」の開発経緯を書く。LINE for Mac のトーク履歴を閲覧・検索できるアーカイバーで、LINE が使っている AES-128-CBC 暗号化 SQLite を復号するために LINE のプロセスメモリからキーを取得するという実装になっている。

https://hakaru.gumroad.com/l/chatarchive

## なぜ作ったか

LINE のトーク履歴をちゃんと手元に保存したかった。

LINE for Mac はトーク履歴のエクスポートが .txt 形式のみで、フォーマットが独自で検索しづらい。過去の特定のやりとりを探すのに毎回 LINE アプリを開いてスクロールするのが面倒だった。バックアップも取れない。

「SQLite に入ってるはずだからそのまま読めばいいのでは」と思って調べ始めたのがきっかけ。

---

## LINE for Mac のデータ保存形式

### ファイルの場所

LINE for Mac のコンテナは `~/Library/Containers/jp.naver.line.mac/` にある。

```
~/Library/Containers/jp.naver.line.mac/
  Data/
    Library/
      Application Support/
        line/
          db/
            naver-line-XXXXXXXX.edb   ← 暗号化 SQLite
```

`.edb` という拡張子。`file` コマンドで見ると SQLite と認識されず、先頭バイトに SQLite のマジックナンバー（`53 51 4c 69 74 65`）がない。暗号化されている。

### 暗号化の方式

SQLite の暗号化拡張としてよく使われるのは SQLCipher だが、LINE が使っているのは [SQLite3 Multiple Ciphers](https://utelle.github.io/SQLite3MultipleCiphers/) という別のライブラリ。AES-128-CBC モードが使われていた。

SQLite3 Multiple Ciphers は PRAGMA でモードを指定する：

```sql
PRAGMA key = 'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx';
PRAGMA cipher = 'aes128cbc';
PRAGMA kdf_iter = 1;
PRAGMA kdf_algorithm = 0;
PRAGMA hmac_algorithm = 0;
PRAGMA legacy = 1;
```

### キーの保存場所

キーがどこに保存されているかを調べた：

| 場所 | 結果 |
|------|------|
| Keychain | 存在しない |
| UserDefaults / plist | 存在しない |
| バイナリに埋め込み | 存在しない |
| 環境変数 | 存在しない |
| Security.framework | リンクあり（詳細不明） |

結論として、**キーはプロセスメモリ上にのみ存在する**。LINE 起動時に何らかの方法でキーを生成または取得し、メモリに保持したまま DB 操作をしている。LINE が起動していない状態でオフライン復号は事実上不可能で、LINE が起動中にメモリスキャンするしかない。

### DB スキーマ

復号に成功すると、以下のテーブルが見える（抜粋）：

**`_profile`** — 自分のプロフィール
```sql
_mid TEXT          -- 自分のユーザーID (u...)
_displayName TEXT  -- 表示名
```

**`_message`** — メッセージ本体
```sql
_chatId      TEXT     -- チャットID（_groupChat._chatMid または _contact._mid）
_from        TEXT     -- 送信者のMID（自分なら自分のMID）
_createdTime INTEGER  -- Unix ミリ秒
_text        TEXT     -- 本文（NULL の場合はメディア等）
_contentType INTEGER  -- コンテンツ種別（後述）
```

**`_contact`** — 友だち・コンタクト
```sql
_mid                  TEXT  -- ユーザーID
_displayName          TEXT  -- 元の表示名
_displayNameOverridden TEXT  -- ユーザーが付けたニックネーム
```

**`_groupChat`** — グループ・複数人トーク
```sql
_chatMid  TEXT  -- グループID（チャットIDと同値）
_chatName TEXT  -- グループ名
```

**`_contentType` の値（主なもの）**

| 値 | 内容 |
|----|------|
| 0 | テキスト |
| 1 | 画像 |
| 2 | 動画 |
| 3 | 音声 |
| 4 | ファイル |
| 5 | スタンプ |
| 7 | ファイル共有 |
| 14 | LINE VOOM |
| 16 | コール |
| 17 | ギフト |

チャットIDによるメッセージ取得：

```sql
SELECT m._createdTime, m._from, m._text, m._contentType,
       COALESCE(c._displayNameOverridden, c._displayName) as sender_name
FROM _message m
LEFT JOIN _contact c ON m._from = c._mid
WHERE m._chatId = ?
ORDER BY m._createdTime ASC
```

---

## LINE iOS iCloud バックアップのデータ保存形式

### ファイルの場所

LINE iOS は iCloud Drive にバックアップを置く。Mac からは以下のパスでアクセスできる：

```
~/Library/Mobile Documents/iCloud~com~linecorp~line/
  Data-{ユーザーID}/
    Backup/
      archive.zip
```

`Data-{ユーザーID}` のディレクトリ名に含まれる文字列は LINE のユーザー MID と一致する。

### archive.zip の中身

```
archive.zip
├── Line.sqlite        (約120MB) ← メインDB
├── MessageExt.sqlite  (約2MB)   ← リアクション・音楽
└── env.dump                     ← NSKeyedArchiver バイナリ
```

`env.dump` は iOS / LINE のバージョン情報が入った NSKeyedArchiver 形式のバイナリ。人間が読める内容はほぼない。

`Line.sqlite` は**暗号化なし**の素の SQLite ファイル。LINE for Mac の `.edb` と異なり、そのまま `sqlite3` コマンドで開ける。

### Line.sqlite のスキーマ

LINE iOS は Core Data を使っているため、テーブル名はすべて `Z_` プレフィックスになる。

**`ZCHAT`** — チャット一覧
```sql
Z_PK       INTEGER   -- 内部PK（メッセージとの結合に使用）
ZMID       VARCHAR   -- チャットID（MID）
ZTYPE      INTEGER   -- チャット種別（後述）
ZLASTMESSAGE VARCHAR -- 最後のメッセージ本文
ZLASTUPDATED TIMESTAMP
```

`ZTYPE` の値：

| 値 | 種別 |
|----|------|
| 0 | 1:1 チャット |
| 1 | 複数人トーク |
| 2 | グループ |
| 4 | 公式アカウント |
| 100 | Keep メモ（自分メモ） |

**`ZMESSAGE`** — メッセージ本体
```sql
ZID          VARCHAR   -- メッセージID（MessageExtとの結合キー）
ZTIMESTAMP   INTEGER   -- Unix ミリ秒（※後述の注意点あり）
ZCHAT        INTEGER   -- ZCHAT.Z_PK への参照
ZSENDER      INTEGER   -- ZUSER.Z_PK への参照（NULL = 自分）
ZTEXT        VARCHAR   -- 本文
ZCONTENTTYPE INTEGER   -- コンテンツ種別
ZCONTENTMETADATA BLOB  -- メタデータ（スタンプIDなど）
ZTHUMBNAIL   BLOB      -- サムネイル画像バイナリ
ZLATITUDE    FLOAT     -- 位置情報（緯度）
ZLONGITUDE   FLOAT     -- 位置情報（経度）
```

自分のメッセージは `ZSENDER IS NULL`。これは Core Data の設計上、自分自身を `ZUSER` テーブルに持たないことによる。

**`ZUSER`** — ユーザー（友だち・グループメンバー）
```sql
Z_PK        INTEGER
ZMID        VARCHAR  -- ユーザーID (u...)
ZNAME       VARCHAR  -- 表示名
ZPROFILEIMAGE VARCHAR
ZISFRIEND   INTEGER  -- 友だち登録済みか
```

**`ZGROUP`** — グループ
```sql
ZID   VARCHAR  -- グループID
ZNAME VARCHAR  -- グループ名
```

**中間テーブル（多対多）**

| テーブル | 用途 |
|---------|------|
| `Z_1MEMBERS` | 複数人トーク（ZTYPE=1）のメンバー |
| `Z_4MEMBERS` | グループのメンバー |
| `Z_4INVITEE` | グループへの招待中メンバー |

チャット種別ごとに名前解決のロジックが異なる：

```sql
-- Type 0: 1:1 → ZUSER.ZNAME で解決
SELECT c.Z_PK, u.ZNAME
FROM ZCHAT c
LEFT JOIN ZUSER u ON c.ZMID = u.ZMID
WHERE c.ZTYPE = 0

-- Type 1: 複数人 → メンバー名を結合
SELECT u.ZNAME FROM Z_1MEMBERS m
JOIN ZUSER u ON m.Z_12MEMBERS = u.Z_PK
WHERE m.Z_1CHATS = ?  -- ZCHAT.Z_PK

-- Type 2: グループ → ZGROUP.ZNAME
SELECT c.Z_PK, g.ZNAME
FROM ZCHAT c
LEFT JOIN ZGROUP g ON c.ZMID = g.ZID
WHERE c.ZTYPE = 2
```

### ⚠️ ZTIMESTAMP の注意点

`ZMESSAGE.ZTIMESTAMP` は **Unix エポック（1970年基準）のミリ秒**。

Core Data の `TIMESTAMP` 型は通常 **2001年1月1日基準の秒数** を使う慣習があるが、LINE はそれに従っていない。

```swift
// ❌ 間違い（年が58000年代になる）
Date(timeIntervalSinceReferenceDate: Double(timestamp))

// ✅ 正しい
Date(timeIntervalSince1970: Double(timestamp) / 1000.0)
```

この間違いをやらかして、GRDB が「58343-05-30」という日付文字列を Date にデコードしようとして `RowDecodingError` が出た。エラーメッセージに年数が入っていたので原因がすぐわかった。

### MessageExt.sqlite のスキーマ

**`ZMESSAGEREACTION`** — スタンプリアクション
```sql
ZMESSAGEID   VARCHAR    -- ZMESSAGE.ZID への参照
ZREACTORMID  VARCHAR    -- リアクションしたユーザーのMID
ZREACTIONTYPE INTEGER   -- リアクション種別（後述）
ZCHATMID     VARCHAR    -- チャットMID
ZCREATEDAT   TIMESTAMP
```

`ZMESSAGEID` は `ZMESSAGE.ZID` と結合できる：

```sql
-- Line.sqlite に MessageExt.sqlite をアタッチ
ATTACH '/path/to/MessageExt.sqlite' AS ext;

SELECT r.ZREACTIONTYPE, r.ZREACTORMID, m.ZTEXT
FROM ext.ZMESSAGEREACTION r
JOIN ZMESSAGE m ON m.ZID = r.ZMESSAGEID
```

`ZREACTIONTYPE` の推定マッピング（確定値は非公開）：

| 値 | 絵文字 | 件数（実測） |
|----|--------|-------------|
| 0 | ❤️ | 154 |
| 2 | 👍 | 2,863 |
| 3 | 😄 | 731 |
| 4 | 😮 | 496 |
| 5 | 😢 | 569 |
| 6 | 😡 | 122 |
| 7 | 🎉 | 124 |

**`ZMESSAGEMUSICTRACK`** — 音楽共有
```sql
ZTRACKID    VARCHAR  -- Apple Music のトラックID
ZTITLE      VARCHAR  -- 曲名
ZARTISTNAME VARCHAR  -- アーティスト名
ZTIMESTAMP  TIMESTAMP
ZMETADATA   BLOB
```

---

## LINE for Mac と iCloud バックアップの比較

| 項目 | LINE for Mac (.edb) | iCloud バックアップ (Line.sqlite) |
|------|---------------------|----------------------------------|
| 暗号化 | AES-128-CBC | なし |
| スキーマ | `_` プレフィックス | `Z_` プレフィックス (Core Data) |
| 自分判定 | `_from = 自分のMID` | `ZSENDER IS NULL` |
| タイムスタンプ | Unix ms | Unix ms |
| リアクション | 未確認 | MessageExt.sqlite に分離 |
| ORMフレームワーク | 独自 | Core Data |

---

## キー抽出の実装

### 方式1: Mach VM 直接スキャン（プライマリ）

macOS の `mach_vm_region` / `mach_vm_read` API を使って LINE プロセスのメモリを直接スキャンする。

```swift
var address: mach_vm_address_t = 0
var size: mach_vm_size_t = 0

while true {
    var info = vm_region_basic_info_data_64_t()
    var count = mach_msg_type_number_t(VM_REGION_BASIC_INFO_COUNT_64)
    var objectName: mach_port_t = 0

    let kr = mach_vm_region(task, &address, &size,
                            VM_REGION_BASIC_INFO_64,
                            &info, &count, &objectName)
    guard kr == KERN_SUCCESS else { break }

    // 読み書き可能なリージョンだけ対象にする
    if info.protection & (VM_PROT_READ | VM_PROT_WRITE) != 0 {
        // 32文字の16進数文字列（AES-128キー）をスキャン
    }
    address += size
}
```

対象は 32 バイトの16進数文字列（AES-128 の 16バイトキーを hex エンコードしたもの）。これを実行するには `com.apple.security.cs.debugger` エンタイトルメントが必要で、App Sandbox も無効にする必要がある。

### 方式2: lldb フォールバック

Mach VM スキャンが失敗した場合、lldb をアタッチして Python スクリプトでメモリを検索する。Xcode コマンドラインツールが必要になるため UX が悪く、現在はほぼ使われていない（Mach VM がほぼすべての環境で動く）。

### セキュリティ上の考慮

取得したキーは DB 復号が完了したらすぐにゼロ埋めしてメモリから消す。`memset` だとコンパイラの最適化で消えてしまうことがあるので `memset_s` を使っている。

```swift
var mutableKey = Array(key.utf8)
mutableKey.withUnsafeMutableBufferPointer { ptr in
    memset_s(ptr.baseAddress, ptr.count, 0, ptr.count)
}
```

---

## アプリの構造

MVVM + SwiftUI。状態マシン的な構成で、`OneClickImportViewModel` がインポートのパイプライン全体を管理する。

```
idle → detectingDatabase → waitingForLINE → extractingKey
     → decrypting → importing → success / error
```

DBレイヤーは2系統：

- **LINE DB**: CSQLite3MC（ローカル SPM パッケージ）で暗号化 `.edb` を開く
- **App DB**: GRDB で管理する内部 SQLite。チャット・メッセージ・コンタクトを保存

全文検索は GRDB の FTS5 仮想テーブルを使っている。

---

## セキュリティ対応の経緯

開発途中に自分でレビューして出てきた問題をいくつか修正した。

**TOCTOU 脆弱性（SEV-014）**: lldb 用の一時 Python スクリプトを `/tmp/line_key_extractor.py` という固定パスに書いていた。攻撃者がシンボリックリンクを張れば任意ファイルに書き込める。`mkstemp` でランダムパスに変更した。

**LIKE句インジェクション（SEV-002）**: チャット検索の LIKE 句でワイルドカード（`%`, `_`）をエスケープしていなかった。GRDB の `.like(_:escape:)` を使って対処した。

**キーの永続化防止**: 取得したキーを Keychain や UserDefaults に保存しないようにした。メモリ上のみで保持し、復号後は即消去。

**ライセンス認証の改ざん防止**: ライセンス状態を UserDefaults だけで管理していたとユーザーに書き換えられる。Keychain に認証情報を持たせ、UserDefaults の値との整合性チェックで改ざんを検出するようにした。

---

## OOM 対策

初期実装は全メッセージを一度にメモリに読み込んでいた。チャット数が多いとクラッシュする。

対策として GRDB のカーソル API で1行ずつ処理するようにした：

```swift
let cursor = try Row.fetchCursor(db, sql: "SELECT ... FROM ZMESSAGE ...")
while let row = try cursor.next() {
    // 1行分だけメモリに持つ
    try processRow(row)
}
```

---

## 命名の変遷

`LINEbackup` → `LineArchive` → `ChatArchive`

最初は機能直球の名前だったが、LINE の商標に引っかかる可能性と、将来的に他のサービスのチャット履歴も扱えるようにという意図でリネームした。

## リリース

Hardened Runtime + Developer ID で署名し、`notarytool` で公証して DMG で配布している。公証は App Store Connect API Key を使って CI から自動化した。

販売は当初 LemonSqueezy を使っていたが、Gumroad に移行した。

---

コードは MIT ライセンスで公開している。暗号化 SQLite の扱い方や Mach VM スキャンの実装で参考になれば。
