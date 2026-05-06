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

## LINE for Mac のデータ構造

LINE for Mac のコンテナは `~/Library/Containers/jp.naver.line.mac/` にある。中を見ると：

```
Data/
  Library/
    Application Support/
      line/
        db/
          naver-line-XXXXXXXX.edb   ← 暗号化 SQLite
```

`.edb` という拡張子で SQLite ファイルが置いてある。`file` コマンドで見ると SQLite と認識されず、先頭バイトを確認すると SQLite のマジックナンバーがない。暗号化されている。

## 暗号化の解析

SQLite の暗号化拡張としてよく使われるのは SQLCipher。ただし LINE が使っているのは [SQLite3 Multiple Ciphers](https://utelle.github.io/SQLite3MultipleCiphers/) という別のライブラリで、AES-128-CBC モードが使われていた。

キーの保存場所を探した：

- **Keychain**: 存在しない
- **UserDefaults / plist**: 存在しない
- **バイナリに埋め込み**: 存在しない（起動ごとに変わる可能性もある）
- **環境変数**: 存在しない

結論として、**キーはプロセスメモリ上にのみ存在する**。LINE 起動時に何らかの方法でキーを生成または取得し、メモリに保持したままDB操作をしている。オフラインでの復号は事実上不可能で、LINE が起動中にメモリスキャンするしかない。

## キー抽出の実装

### 方式1: Mach VM 直接スキャン（プライマリ）

macOS の `vm_read` API を使って LINE プロセスのメモリを直接スキャンする。

```swift
var regions: [MemoryRegion] = []
var address: mach_vm_address_t = 0
var size: mach_vm_size_t = 0

while true {
    var info = vm_region_basic_info_data_64_t()
    var count = mach_msg_type_number_t(VM_REGION_BASIC_INFO_COUNT_64)
    var objectName: mach_port_t = 0
    
    let kr = mach_vm_region(task, &address, &size, ...)
    guard kr == KERN_SUCCESS else { break }
    
    // 読み取り可能かつ書き込み可能なリージョンをスキャン
    if info.protection & (VM_PROT_READ | VM_PROT_WRITE) != 0 {
        // 32バイト（AES-128 = 16バイトの hex = 32文字）の hex 文字列パターンを探す
    }
    address += size
}
```

対象は 32 バイトの16進数文字列（AES-128 のキーを hex エンコードしたもの）。メモリをリージョンごとに読み取って正規表現でマッチングする。

これを実行するには `com.apple.security.cs.debugger` エンタイトルメントが必要で、App Sandbox も無効にする必要がある。

### 方式2: lldb フォールバック

Mach VM スキャンが失敗した場合、lldb を使ってアタッチし Python スクリプトでメモリを検索する。Xcode コマンドラインツールが必要なため、現在は setupガイドからこのステップを削除してプライマリ方式のみ案内している（ほぼすべての環境でMach VMが動く）。

### セキュリティ上の考慮

取得したキーはDB復号が完了したらすぐにゼロ埋めしてメモリから消す。`memset` だとコンパイラの最適化で消えてしまうことがあるので `memset_s` を使っている。

```swift
var mutableKey = Array(key.utf8)
mutableKey.withUnsafeMutableBufferPointer { ptr in
    memset_s(ptr.baseAddress, ptr.count, 0, ptr.count)
}
```

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

## セキュリティ対応の経緯

開発途中に自分でレビューして出てきた問題をいくつか修正した。

**TOCTOU 脆弱性（SEV-014）**: lldb 用の一時 Python スクリプトを `/tmp/line_key_extractor.py` という固定パスに書いていた。攻撃者がシンボリックリンクを張れば任意ファイルに書き込める。`mkstemp` でランダムパスに変更した。

**LIKE句インジェクション（SEV-002）**: チャット検索の LIKE 句でワイルドカード（`%`, `_`）をエスケープしていなかった。GRDB の `.like(_:escape:)` を使って対処した。

**キーの永続化防止**: 取得したキーを Keychain や UserDefaults に保存しないようにした。メモリ上のみで保持し、復号後は即消去。

**ライセンス認証の改ざん防止**: ライセンス状態を UserDefaults だけで管理していたとユーザーに書き換えられる。Keychain に認証情報を持たせ、UserDefaults の値との整合性チェックで改ざんを検出するようにした。

## iCloud バックアップ対応

LINE iOS のバックアップを Mac の iCloud Drive 経由で読む機能も追加した。

保存場所：
```
~/Library/Mobile Documents/iCloud~com~linecorp~line/
  Data-XXXXXXXX/Backup/archive.zip
```

`archive.zip` を展開すると `Line.sqlite` が入っている。こちらは**暗号化なし**の素の SQLite。LINE iOS の Core Data スキーマで、テーブル名に `Z_` プレフィックスがつく。

```sql
-- メッセージ取得例
SELECT m.ZTIMESTAMP, m.ZSENDER, m.ZTEXT, m.ZCONTENTTYPE, u.ZNAME
FROM ZMESSAGE m
LEFT JOIN ZUSER u ON m.ZSENDER = u.Z_PK
WHERE m.ZCHAT = ?
ORDER BY m.ZTIMESTAMP ASC
```

`ZSENDER = NULL` が自分のメッセージ。`ZTIMESTAMP` は Unix ミリ秒（Core Data の参照日付 2001年基準ではなく 1970年基準）。ここを間違えて年が58343年になるバグを踏んだ。

## 命名の変遷

- `LINEbackup` → `LineArchive` → `ChatArchive`

最初は機能直球の名前だったが、LINE の商標に引っかかる可能性と、将来的に他のサービスのチャット履歴も扱えるようにという意図でリネームした。

## OOM 対策

初期実装は全メッセージを一度にメモリに読み込んでいた。チャット数が多いと iOS シミュレータ相当のメモリでクラッシュする。

対策として GRDB のカーソル API で1行ずつ処理するようにした：

```swift
let cursor = try Row.fetchCursor(db, sql: "SELECT ... FROM ZMESSAGE ...")
while let row = try cursor.next() {
    // 1行分だけメモリに持つ
    try processRow(row)
}
```

## リリース

Hardened Runtime + Developer ID で署名し、`notarytool` で公証してDMGで配布している。公証はApp Store Connect API Keyを使ってCIから自動化した。

販売は当初 LemonSqueezy を使っていたが、Gumroad に移行した。

---

コードは MIT ライセンスで公開している。暗号化 SQLite の扱い方や Mach VM スキャンの実装で参考になれば。
