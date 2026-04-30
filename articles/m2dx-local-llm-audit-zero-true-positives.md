---
title: "ローカルLLM 10機種にDX7エンジンの監査をやらせたら、57件中ハズレ57件だった話"
emoji: "🤖"
type: "tech"
topics: ["llm", "ollama", "swift", "security", "codereview"]
published: false
---

## やりたかったこと

iOS/macOS向けのMIDI 2.0 FM音源アプリ **M2DX** を一人で書いている。心臓部の **M2DX-Core**（DX7互換エンジン、Pure Swift）はOSSで公開していて、そろそろ第三者の目を入れたい時期だった。

普段の開発支援は Claude Code と Codex を使っているが、リポジトリ全体を機械的に舐めて issue 起票してくれる「夜中に勝手に走らせるレビューワー」が欲しかった。Cloud に毎回投げるのもAPIコスト的に微妙だし、手元の **Mac Studio M2 Ultra / 96GB** に ollama を入れて 70B〜141B クラスを動かせる環境はあるので、「ローカルLLMでバグ監査・セキュリティ監査をやってみよう」という素朴な発想だった。

結論から言うと:

- **バグ監査**: 10モデル横断で **41件の指摘 → 真陽性 0件**
- **セキュリティ監査**: 5モデル中4モデル完走、**16件の指摘 → 真陽性 0件**
- 累計 **57件、TP 0件**

なぜそうなったかを書く。

## 監査対象

- **DX7Envelope.swift** (178行): EG（4-rate/4-level）、Q16固定小数点
- **DX7Operator.swift** (85行): オペレータ、Q24位相、フィードバックバッファ
- **DX7Voice.swift** (472行): 6オペレータ × 32アルゴリズム ディスパッチ、PitchEG
- **Algorithm.swift**: 32アルゴリズムのフラグテーブル

セキュリティ監査では追加で:

- **DX7SysExParser.swift** (120行): iCloud/AirDrop経由の untrusted な .syx パース
- **UserBankManager.swift** (90行): fileImporter、security-scoped resource
- **MIDIInputManager.swift** (1,132行): MIDI 2.0 UMP デコード、PE JSON
- **PEResponderHost.swift** (374行): Property Exchange responder
- **USBResetHelper.c** (84行): IOKit USB reset (macOS)

合計 約 1,800 行のSwift + 100行のC。

## ollama API の使い方

最初は `ollama run mistral` みたいに対話的に投げてたけど、長文プロンプトだとブロックバッファリングで進捗が出ないし、ANSIスピナー（`⠙ ⠹ ⠸`）が出力に混ざる事故が起きた。

ちゃんとAPIを叩く形に切り替え:

```python
import json, urllib.request

body = json.dumps({
    "model": "mixtral:8x22b",
    "prompt": prompt,
    "stream": False,
    "options": {"num_ctx": 40960}
}).encode()

req = urllib.request.Request(
    "http://127.0.0.1:11434/api/generate",
    data=body,
    headers={"Content-Type": "application/json"}
)
with urllib.request.urlopen(req, timeout=3600) as r:
    d = json.loads(r.read())
    print(d["response"])
```

`num_ctx` は要注意で、ollamaのデフォルトは2048。コードを大量に貼ると黙って後ろを切り捨てるので、ファイル全体を渡すなら **40k トークン以上明示的に指定**しないといけない。これに気づくまで「なぜLLMは50行目以降を読んでないのか」と悩んだ。

## プロンプト設計

最初に渡したプロンプトは「DX7エンジンのバグを探して」程度のラフなもの。これだと「コメントを書け」「変数名を変えろ」みたいな bikeshedding が混ざる。改善:

- **検証可能な観点に絞る**: 整数オーバーフロー、境界チェック、リアルタイムスレッド安全性
- **出力フォーマットを固定**: 1行1 JSON、`severity / file / lines / title / what / symptom / fix_hint`
- **PoCを要求**: 「攻撃者が送り込む具体的バイト列を書け」（書けないなら spec ulative なので drop）
- **CWE番号を要求**: ハルシネーションかどうか即座に分かる

セキュリティ監査の方はさらにスレットモデルを明文化した:

> Attacker can send arbitrary bytes via:
> - SysEx files imported from iCloud Drive / AirDrop / Files app
> - MIDI 2.0 UMP from any USB / Bluetooth / network MIDI device
> - Property Exchange JSON-over-MIDI responses
>
> Goal: find vulnerabilities that lead to crash / memory corruption / arbitrary file write / DoS / info disclosure.

## Phase 1: バグ監査

### 走らせたモデル

| モデル | サイズ | 実行時間 | findings | TP |
|---|---|---|---|---|
| phi4:14b | 9 GB | 446 s | 1 | 0 |
| qwen3:14b | 9 GB | 696 s | 4 | 0 |
| gemma3:12b | 8 GB | 517 s | 8 | 0 |
| hermes3:8b | 5 GB | 582 s | 4 | 0 |
| codestral:22b | 12 GB | 2,081 s | 10 | 0 |
| gemma4:31b | 19 GB | 943 s | 4 | 0 |
| qwen3.6:35b | 23 GB | 843 s | 0\* | – |
| llama3.3:70b | 42 GB | 405 s | 5 | 0 |
| mixtral:8x22b | 79 GB | 2,139 s | 1 | 0 |
| llama4:scout | 67 GB | – | – | 失敗 |

\* 22,714 トークンの reasoning を吐いて結局JSON出力なし。  
llama4:scout は HTTP 500 で 0 byte。

### 共通誤検出パターン

5モデル以上が独立に同じ誤指摘をしてきたパターンを並べる。

#### 1. `rawInc = (4 + (qrate & 3)) << (8 + (qrate >> 2))` を「整数オーバーフロー」と指摘

```swift
var qrate = (rate * 41) >> 6
qrate = min(63, qrate + rateScaling)  // ← クランプ済
let rawInc = (4 + (qrate & 3)) << (8 + (qrate >> 2))
inc = Int32((Int64(rawInc) * srMultiplier) >> 24)
```

`qrate` は `min(63, ...)` で制限済み。最大値は `7 << 23 = 58M` で `Int32` に余裕で入る。さらに `Int64(rawInc) * srMultiplier` で安全に拡張してから `>> 24`。

LLMは **直前の `min(63, ...)` を読み飛ばし**、`qrate >> 2` が「大きい値になりうる」という連想だけで指摘してくる。8B〜70Bのほぼ全モデルが同じハマり方をした。

#### 2. Swift tuple を「ヒープ配列」と誤認

```swift
package struct DX7Voice {
    var ops = (DX7Operator(), DX7Operator(), DX7Operator(),
               DX7Operator(), DX7Operator(), DX7Operator())
    ...
}
```

これを見た複数のモデル（特にhermes3:8b、gemma4:31b）の指摘:

> "Array creation in real-time path. Array creation is a heavy operation
> and must be avoided in the critical section of the real-time processing loop."

Swiftの **tuple はスタック確保** で、Arrayとは全く別物。型シグネチャを丁寧に読まずに「6個並んでる→Array」と決めつけている。`@inline(__always)` で switch-case に展開してるホットパスを「冗長」とまで言ってきた。

#### 3. `&+` / `&-` を「未チェックなオーバーフロー」と指摘

```swift
level = level &- inc
```

`&-` は Swift の **明示的wrap-around演算子**。固定小数点のラップは意図的な振る舞いで、`-` だとクラッシュさせるので使い分けてる。これを「underflow check が無い」と指摘してきたモデルが3つ。

#### 4. `min(algorithm, 31)` のクランプを見落とす

```swift
let alg = kAlgorithmFlags[min(algorithm, 31)]
```

これを「OOB read」と指摘するモデル（codestral:22b）。`min(_, 31)` はそのために書いた防御。

#### 5. DX7仕様で固定の32アルゴリズムを「ハードコード」と批判

```swift
public let kAlgorithmFlags: [(UInt8, UInt8, UInt8, UInt8, UInt8, UInt8)] = [...]
```

DX7のアルゴリズムは **仕様で固定32種**。変えたら DX7 ではなくなる。「動的生成すべき」みたいな提案は仕様無視。

### つまり:

70Bクラスでも **存在しない行番号を引用してくる**。`llama3.3:70b` は `DX7Operator.swift L100-L120` を指摘してきたが、ファイルは85行しかない。`gemma3:12b` は `DX7Envelope.swift L192-L216` (実178行)、`mixtral:8x22b` は `DX7Voice.swift L364-L405` を `advanceStage` 関数として指摘 (実関数は L54-L95)。

## Phase 2: セキュリティ監査

スレットモデルを明文化、PoC を要求、CWE 番号を強制。観点を絞った分マシになるかと思った。

### 結果

| モデル | 実行時間 | findings | TP |
|---|---|---|---|
| gemma3:12b | 57 s | 0 (info) | – |
| qwen3:14b | 200 s | 3 | 0 |
| codestral:22b | 214 s | 10 | 0 |
| llama3.3:70b | 1,903 s | 3 | 0 |
| mixtral:8x22b | timeout (60min) | – | – |

### qwen3:14b の「もっともらしい」3件

#### F1: Path Traversal in UserBankManager (CWE-22)

```swift
let filename = sourceURL.lastPathComponent
let destURL = userBanksDirectory.appendingPathComponent(filename)
```

PoC: `../etc/passwd.syx` というファイル名で arbitrary file write、と。

**実態**: iOS/macOS のファイルシステムは **filename に `/` を含められない**。`URL.lastPathComponent` は単一コンポーネントのみ返すので、`../` は構造的に発生しない。`..` 単体だと `appendingPathComponent("..")` が親ディレクトリになるが、`copyItem` の dest が既存で失敗する。

**理屈は正しいが、iOS の OS レイヤで遮断済み。Exploit 不可。**

#### F2: Symlink Follow in copyItem (CWE-59)

```swift
try FileManager.default.copyItem(at: sourceURL, to: destURL)
```

「シンボリックリンクを copyItem するとリンク先がコピーされて sandbox を超える」という指摘。

**実態**: Apple のドキュメント:

> If srcURL is a symbolic link, this method copies the symbolic link, not the file.

`copyItem` は **symlink を symlink としてコピーする**。リンク先を読みに行くわけではない。仮にリンク先を `Data(contentsOf:)` で読んだとしても、ユーザは fileImporter でそのリンクへのアクセス権を **既に明示的に付与している**。Sandbox 越境は発生しない。

#### F3: Unbounded JSON Parsing → DoS (CWE-400)

`MIDIInputManager.decodePEPayload` で巨大JSONをデコードすると DoS、と。

**実態**: 上流の `UMPSysEx8Assembler` / `UMPSysEx7Assembler` (MIDI2Kit) が `maxBufferSize: Int = 65536` を既定で設定済。SysEx は **64KB で打ち切り**。`decodePEPayload` まで届く時点で 64KB を超えていない。

つまり M2DX 側は:

- iOS のファイルシステム制約 (`/` 不可)
- Apple の copyItem 仕様 (symlink 非追従)
- iOS sandbox (security-scoped resource)
- MIDI2Kit の maxBufferSize 64KB

の **三重・四重で防御済み**。LLM はコードに書かれた局所的なパターンだけ見て、エコシステム全体の防御は認識できなかった。

### codestral:22b は USBResetHelper.c に存在しないコードを捏造

USBResetHelper.c は IOKit を叩く 84 行の C で、`offset` / `endIndex` などの変数は **存在しない**。にもかかわらず:

```json
{
  "cwe": "CWE-190",
  "file": "USBResetHelper.c",
  "lines": "L43",
  "title": "Integer Overflow in USB Device Indexing",
  "what": "The index 'offset' may overflow the 'data' array
           when calculating 'endIndex'.",
  "poc": "Send a large value for 'limit' or 'offset'
          in the header of a PEGET request."
}
```

PEGETリクエストはMIDI Property Exchange の用語で、IOKit USB と無関係。**他のファイルの記憶を混ぜて捏造**してる。10個中10個がこのレベル。

## なぜ失敗するのか

3つの理由があると思う。

### 1. 防御コードを「読まない」

LLMは「危険そうなパターン」を見つけるのは得意だが、その**直前直後にある防御コード**（`min`, `max`, `guard`, `clamping:`）を読み飛ばす傾向が顕著。`min(63, ...)` の3行先を見て「overflow可能」と判断する。

### 2. 言語固有のセマンティクスが弱い

- Swift tuple ≠ Array
- `&+` / `&-` は意図的wrap
- `Int32(clamping:)` は精度を失わない（飽和）
- Dictionary subscript は Optional を返す（クラッシュしない）

これらは Swift を知っていれば即座に分かるが、LLM は **「C/C++ から類推する」** のかこれらを誤認しがち。

### 3. エコシステム全体を見ない

`UMPSysEx8Assembler` の `maxBufferSize: 65536` は MIDI2Kit のソースを読まないと分からない。LLM はプロンプトに含まれていない依存先のコードを参照できないので、「上流で何が保証されているか」を考慮した監査ができない。

## 何が「向いていない」のか

「ローカル LLM は使い物にならない」という結論ではない。問題は **タスクとの相性**で、コード監査は特に厳しい:

- **false positive が一つでも混ざると、残り全部の信頼が崩れる**: 50件の指摘の中から 1件の真のバグを掘り当てるより、最初から全部捨てた方が早い
- **コードに書かれていない情報（依存ライブラリ、OS API のセマンティクス、言語仕様）に大きく依存する**: ローカル LLM はプロンプトの外を見ないので、ここで負ける

逆に、ローカル LLM がローカルである意味があるのは:

- 完全オフラインの環境（飛行機・出張先）
- 外に出したくない短いメモや会話の要約
- 概念説明・用語確認（ネット代わり）

…くらいで、「OSS のレビューを夜中に走らせる」用途では、現時点では **何かしら別のアプローチを取った方がいい**。

## まとめ

- ollama の num_ctx はデフォルト 2048。明示しないと黙って切り捨てる
- ローカル LLM 10機種に同一プロンプトでバグ + セキュリティ監査 → **57 findings, TP 0**
- 共通の失敗モード: 防御コード見落とし、言語セマンティクス（Swift tuple, `&+`）誤認、依存ライブラリ未参照、行番号の捏造
- バグ監査・セキュリティ監査のような **false positive が許されない** タスクには現状の 70B〜141B でも厳しい

「コードの問題を勝手に見つけて issue 起票してくれる便利アシスタント」を期待してたなら、**今のローカル LLM ではまだ早い**。

---

監査対象のM2DX-CoreはApache 2.0で公開しています。検証用のpromptと出力ログは `/tmp/m2dx-audit/` および `/tmp/m2dx-secaudit/` に残してあるので、別モデルで再走行して反証してくれる方がいたら歓迎です。

- **M2DX-Core (OSS)**: https://github.com/hakaru/M2DX-Core
- **M2DX (iOS app)**: https://apps.apple.com/jp/app/m2dx/id6753466996
