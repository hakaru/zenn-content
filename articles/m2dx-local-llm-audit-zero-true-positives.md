---
title: "ローカルLLMって本当に開発に使える？DX7エンジンの監査をやらせたら、全然だめでした。。"
emoji: "🤖"
type: "tech"
topics: ["llm", "ollama", "swift", "security", "codereview"]
published: false
---

## 夜中に勝手に働いてくれるレビューワー、欲しくない？

iOS/macOS向けのMIDI 2.0 FM音源アプリ **M2DX** を開発中。 **TestFlight で公開ベータ配布中** → <https://testflight.apple.com/join/BAtGszPw> （試してフィードバックもらえると嬉しい）。心臓部の **M2DX-Core**（DX7互換エンジン、Pure Swift）はOSSで公開している。

普段の開発支援は Claude Code と Codex を使ってるんだけど、欲しいのはリポジトリ全体を機械的に舐めて、夜中に勝手に走らせて朝起きたら issue が並んでる、みたいなレビューワーが欲しかった。手元には **Mac Studio M3 Ultra / 96GB**。70B〜141Bクラスのローカルモデルがちゃんと動くスペック。これ使えるでしょう。 ollama を立ち上げた。
戦いの記録、長くなってしまい。さらにRAG、LoRAでモデルチューニングへ続く

結論❌

- **バグ監査**: 10モデル走らせて **36件指摘 → 真陽性 0件**
- **セキュリティ監査**: 5モデル中4モデル完走、**16件指摘 → 真陽性 0件**
- 累計 **52件、当たり 0件**
- ……だけど、 **追加でリファクタリング提案を試したら 10 モデル中 7 が完璧通過した**。
監査タスクは全敗。けどタスクを変えると景色が変わる、という二段構えの話。なぜそうなったか。

## 何を読ませたか

DX7エンジンの中核ファイル4つ:

- **DX7Envelope.swift** (178行) — EG（4-rate/4-level）、Q16固定小数点
- **DX7Operator.swift** (85行) — オペレータ、Q24位相、フィードバックバッファ
- **DX7Voice.swift** (472行) — 6オペレータ × 32アルゴリズム ディスパッチ
- **Algorithm.swift** — 32アルゴリズムのフラグテーブル

セキュリティ監査の方ではさらに、外から触れる経路を全部足した:

- **DX7SysExParser.swift** (120行) — iCloud/AirDrop経由で入ってくる怪しい .syx パーサ
- **UserBankManager.swift** (90行) — fileImporter まわり
- **MIDIInputManager.swift** (1,132行) — MIDI 2.0 UMP デコード、PE JSON
- **PEResponderHost.swift** (374行) — Property Exchange responder
- **USBResetHelper.c** (84行) — IOKit USB reset (macOS用)

合計で Swift が約1,800行、Cが100行。「変なバイト列を流し込まれたとき何が起きるか」を見て欲しい場所をひと通り。

## 最初にハマった ollama の罠

最初は雑に `ollama run mistral < prompt.md` みたいに流してた。これが地味に良くなくて、

- 長文プロンプトだとブロックバッファリングで進捗が見えない
- 端末のスピナーっていうらしい（`⠙ ⠹ ⠸`）みたいなやつが出力に混ざる。
- そして最大の問題: **`num_ctx` のデフォルトが 2048 トークン**

ファイル食わしたつもりが「LLMが50行目以降を全く知らない」みたいな奇妙な挙動になって、お前ホント賢くないな…

ちゃんと API を叩く形に切り替え:

```python
import json, urllib.request

body = json.dumps({
    "model": "mixtral:8x22b",
    "prompt": prompt,
    "stream": False,
    "options": {"num_ctx": 40960}  # ← ここ
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

`num_ctx` は使うモデルの最大値（mixtral なら 65k くらい）まで上げて良い。コードを貼って監査させるなら最低でも 32k 〜 40k は要る。

## プロンプトをどう書く？

最初に渡した雑プロンプト（「DX7エンジンのバグを探して」）の出力は、`コメントを書け / 変数名を変えろ / もっと関数を分けろ` みたいな bikeshedding ばかりで、 *いや、そうじゃなくて…* と頭を抱えた。

なので絞った:

- **検証可能な観点だけ** に限定 — 整数オーバーフロー、境界チェック、リアルタイムスレッド安全性
- **出力フォーマットを固定** — 1行1 JSON、`severity / file / lines / title / what / symptom / fix_hint`
- **PoC（concrete attacker payload）を要求** — 書けないなら spec ulative なので drop しろ、と命令
- **CWE番号を要求** — 適当な番号を言ってきたらハルシネーションだと即わかる

セキュリティ監査の方はさらに脅威モデルを明文化した:

> Attacker can send arbitrary bytes via:
> - SysEx files imported from iCloud Drive / AirDrop / Files app
> - MIDI 2.0 UMP from any USB / Bluetooth / network MIDI device
> - Property Exchange JSON-over-MIDI responses
>
> Goal: find vulnerabilities that lead to crash / memory corruption /
> arbitrary file write / DoS / info disclosure.

このくらい絞れば、それなりに役に立つ指摘が出てくるだろう、と思った。 *思っていた*。

## まずはバグ監査から

走らせたのはこの10モデル（機種名は手元 ollama install のタグそのまま、世代混在は意図的 — `qwen3` と `qwen3.6`、`gemma3` と `gemma4`、`llama3.3` と `llama4` は別系統の別モデルで、8B〜141B の幅をカバーするためにわざと混ぜている）。表の `TP` 列は **真陽性 (true positive) = 実コードを照合して "本物のバグだった" と確認できた件数**:

| モデル | サイズ | 実行時間 | findings | TP |
|---|---|---|---|---|
| phi4:14b | 9 GB | 446 s | 1 | 0 |
| qwen3:14b | 9 GB | 696 s | 4 | 0 |
| gemma3:12b | 8 GB | 517 s | 8 | 0 |
| hermes3:8b | 5 GB | 582 s | 4 | 0 |
| codestral:22b | 12 GB | 2,081 s | 10 | 0 |
| gemma4:31b | 19 GB | 943 s | 3 | 0 |
| qwen3.6:35b | 23 GB | 843 s | 0\* | – |
| llama3.3:70b | 42 GB | 405 s | 5 | 0 |
| mixtral:8x22b | 79 GB | 2,139 s | 1 | 0 |
| llama4:scout | 67 GB | – | – | 失敗 |

\* qwen3.6:35b は **2万2千トークンの reasoning** を吐いてから結局JSON 1行も出さずに終了。  
llama4:scout は HTTP 500 で 0 byte。

検証は地道に手作業で、各 finding の引用行を実コードと突き合わせて、「本当にバグか / 行が実在するか」を一個ずつ見ていった。

36件の指摘を全部読んで、いくつかの**よく似たパターン**で間違えていることに気づいた。これが面白いのでパターン別に並べる。

### パターン1: クランプ済みの値に「overflow するぞ」と警告してくる

DX7のEGの内部計算で、こういうコードがある:

```swift
var qrate = (rate * 41) >> 6
qrate = min(63, qrate + rateScaling)  // ← 上限 63 でクランプしてる
let rawInc = (4 + (qrate & 3)) << (8 + (qrate >> 2))
inc = Int32((Int64(rawInc) * srMultiplier) >> 24)
```

`qrate` は 63 に上限が貼ってある。ということは、最後のシフト幅は最大 `8 + 15 = 23` で、`(4+3) << 23 = 約 5800万`。Int32（約21億）の中にスッポリ収まる。さらに `Int64` に拡張してから掛け算してるので、本当に二重に守ってる。

これを見て **8B〜70Bのほぼ全モデルが** 「`qrate >> 2` が大きい値になりうるので overflow します」と警告してきた。3行前の `min(63, ...)` を読んでない。70Bでもこのレベル、というのが正直一番ガッカリした。

### パターン2: Swift の tuple を「ヒープ確保された配列」と勘違いする

これは結構ウケた:

```swift
package struct DX7Voice {
    var ops = (DX7Operator(), DX7Operator(), DX7Operator(),
               DX7Operator(), DX7Operator(), DX7Operator())
    ...
}
```

これに対して、複数のモデル（特に hermes3:8b や gemma4:31b）が:

> Array creation in real-time path. Array creation is a heavy operation
> and must be avoided in the critical section of the real-time processing loop.

…と。**Swift の tuple は固定サイズの値型**で、Array が裏で持つヒープバッファとは別物（ただし「常にスタックに置かれる」かは仕様で保証はされず、最適化次第）。「6個並んでる、配列だ」っていう連想だけで指摘してきてる。さらに `@inline(__always)` 付けて switch-case で展開してるホットパスを「冗長だから関数化しろ」とまで言われた。 *いや、だからわざわざ inline 展開してんだって…*

### パターン3: `&+` / `&-` を「未チェックなオーバーフロー」と取る

```swift
level = level &- inc
```

`&-` は **わざと wrap させる** Swift の演算子。固定小数点ロジックでは普通に使う。これを「underflow check が抜けてる」と指摘してきたモデルが3つ。 *いや、`-` でもよかったんだけど、わざわざ `&-` 書いてるのは「ラップして欲しいから」なんだよ…*

### パターン4: 防御コードを綺麗に踏み越えていく

```swift
let alg = kAlgorithmFlags[min(algorithm, 31)]
```

`min(_, 31)` がついてるのに、これを「OOB アクセスの可能性あり」と書いてきた（codestral:22b）。 **その `min` 何のために書いたと思ってんのよ…**

### パターン5: 仕様で固定されてるものを「ハードコード批判」してくる

DX7のアルゴリズムは仕様で32種類に固定されてる。それを以下のように書いてある:

```swift
public let kAlgorithmFlags: [(UInt8, UInt8, UInt8, UInt8, UInt8, UInt8)] = [
    /* alg 1 */ (...),
    /* alg 2 */ (...),
    /* ...全32個... */
]
```

これに対して「動的生成すべき」と提案してくるモデルがいた。動的生成したら **DX7 じゃない別のシンセになっちゃう**。

### 最悪。行番号を平気で捏造する

70Bでも安心できないのが、 **存在しない行番号を引用してくる** こと。

- `llama3.3:70b`: `DX7Operator.swift L100-L120` → ファイル全体で 85 行
- `gemma3:12b`: `DX7Envelope.swift L192-L216` → 実 178 行
- `mixtral:8x22b`: `DX7Voice.swift L364-L405` を `advanceStage` 関数として指摘 → 実関数は L54-L95（全然違う場所）


## 次にセキュリティ監査

「バグ監査では言語仕様を読み違えるけど、セキュリティ観点なら多少マシになるんじゃ？」という淡い期待を抱えて、もう一度回した。

| モデル | 実行時間 | findings | TP |
|---|---|---|---|
| gemma3:12b | 57 s | 0 (info) | – |
| qwen3:14b | 200 s | 3 | 0 |
| codestral:22b | 214 s | 10 | 0 |
| llama3.3:70b | 1,903 s | 3 | 0 |
| mixtral:8x22b | timeout (60min) | – | – |

**結果: TP 0件**。同じだった。

ただ、出てきた偽陽性は **「もっともらしさ」が一段上がった**。具体例を見ていく。

### qwen3:14b が出した、もっともらしい3件

#### Path Traversal (CWE-22)

```swift
let filename = sourceURL.lastPathComponent
let destURL = userBanksDirectory.appendingPathComponent(filename)
```

「`../etc/passwd.syx` というファイル名で arbitrary file write 可能」と。

**実態**: iOS/macOS の POSIX 経路では **filename に `/` を含められない**（`/` は path separator で予約済）。`URL.lastPathComponent` も単一コンポーネントしか返さないので、fileImporter から渡される実ファイル URL に `../` を仕込むことは構造的にできない。理屈は完璧、でも前提が成立しない。なお、`appendingPathComponent` 自体は任意文字列に対する一般的なサニタイザではないので、別経路で `/` 入りの文字列を渡せば話は別 — 今回は実ファイル URL だから安全、というだけ。

#### Symlink Follow (CWE-59)

```swift
try FileManager.default.copyItem(at: sourceURL, to: destURL)
```

「symlink を copyItem したらリンク先がコピーされて sandbox 越境」。これも一見正しそう。

**実態**: Apple のドキュメント:

> If srcURL is a symbolic link, this method copies the symbolic link, not the file.

`copyItem` は **symlink を symlink としてコピー**する。リンクを辿らない。仮に辿ったとしても、ユーザは fileImporter で「そのファイルへのアクセス権」を**自分で明示的に付与してる**ので、sandbox 越境にならない。

#### Unbounded JSON DoS (CWE-400)

「PEペイロードのJSONをサイズ制限なしでデコードしてるから、巨大JSONを送ったら DoS」。

**実態**: 上流の MIDI2Kit (`UMPSysEx8Assembler` / `UMPSysEx7Assembler`) が `maxBufferSize: Int = 65536` をデフォルトで設定済。**SysEx は 64KB で打ち切られる**。`decodePEPayload` まで届く時点で既に頭打ち。

つまり M2DX 側は:

- iOS のファイルシステム制約 (`/` 不可)
- Apple の copyItem 仕様 (symlink 非追従)
- iOS sandbox (security-scoped resource)
- MIDI2Kit の maxBufferSize 64KB

の **三重・四重の防御**でカバー済み。LLM は手元のファイルだけ見て指摘してくるので、「上流ライブラリと OS でどう守られてるか」を **全く考慮できない**。これがセキュリティ観点で一番効いてくる弱点だった。

### codestral:22b は USBResetHelper.c に存在しないコードを「指摘」してくる

84行のシンプルなC（IOKit USB を叩くだけ）に対して、こう来た:

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

`offset` も `endIndex` も **このファイルには存在しない**。`PEGET` リクエストは MIDI Property Exchange の用語で、IOKit USB と何の関係もない。 **他のファイルの記憶が混ざって、知らないコードを生成してる**。

そしてこのレベルの指摘が10個中10個。 *うーん…*

## 問題点の整理

3つ

### 1. 防御コードを「読まない」

LLMは「危険そうなパターン」（シフト演算、配列アクセス、文字列パース）を見つけるのは得意。でも **その3行前にある `min(63, ...)` や `guard data.count == ...`** を読み飛ばす。学習データの中に「危険なシフト演算 → overflow」というペアが大量にあって、それを引っ張ってきてるんだろう。**コンテキストの局所性**が低い。

### 2. 言語固有のセマンティクスが弱い

Swift はメジャー言語のはずだけど、ローカルLLMは細部で結構間違える:

- **tuple ≠ Array**（heap allocation じゃない）
- **`&+` / `&-`** は明示的wrap（チェックが「ない」のではなく「いらない」）
- **`Int32(clamping:)`** は範囲外の値を `Int32.min` / `Int32.max` に飽和させる（truncate と違ってラップしない）
- **Dictionary subscript** は Optional を返す（クラッシュしない）

C/C++ で覚えた感覚で類推してくる、と言うと当たってる気がする。

### 3. プロンプトの外を見られない

`UMPSysEx8Assembler` の `maxBufferSize: 65536` は MIDI2Kit のソースを開かないと分からない。ローカルLLMは依存先を grep する手段がないので、**「上流で何が保証されてるか」を加味した監査**ができない。

これは agent 化（grep/Read を投げ返せるツール）すれば改善するはずだけど、ollama 単体では無理。

## というか、LLMはSwiftをそもそも知らない疑惑

3つの理由のうち2番目を、もっと強い言い方で書き直したい。 **ローカル LLM は Swift をそんなに分かってない**。

今回の誤検出の中身を眺め直すと、5パターンのうち3〜4つは「Swift 仕様の誤読」が原因になっている:

- tuple `(A, A, A, A, A, A)` を **「ヒープに乗る配列」と判定** → "RT 安全性違反" と的外れな指摘
- `&+` / `&-` を **「未チェックなオーバーフロー」と判定** → 明示的 wrap 演算子だと知らない
- `Int32(clamping:)` を **「精度を失う truncation」と判定** → 範囲外を飽和させる動作を知らない
- Dictionary subscript の Optional 戻りを **「OOB クラッシュ」と判定** → 安全な subscript 仕様を知らない
- `min(_,31)` のクランプを **無視** → これは Swift というより読解の問題だが、組み合わさって悪化する

70Bクラスでもこのレベル。 **C/C++/Java で覚えたパターンをそのまま当ててくる感じ**。これはモデルの「能力不足」というより、 **学習データの中の Swift 比率が薄い**のが本質だと思う:

- GitHub 全体に占める Swift コードは Python / JS / Java / C++ より圧倒的に少ない
- Apple 純正 SDK の大半（SwiftUI、AVFoundation、CoreMIDI などの中身）は **クローズドソース**で学習データに入りにくい（Foundation 系は `swift-foundation` がオープンの部分もあるが、プラットフォーム実装は別）
- Swift 5 → 6 で言語仕様が大きく動いた（concurrency、strict mode）ので、古い学習データで覚えた Swift は陳腐化しやすい
- DSP、固定小数点、リアルタイム制約のような **「Apple アプリ寄りでない Swift コード」はさらに希少**

つまり LLM の Swift 知識は「素朴な iOS アプリ部分は得意 / 言語の細部や非 GUI ドメインは弱い」というアンバランスな状態にある。今回の DX7 エンジン（固定小数点 DSP + リアルタイム制約 + Swift 6 strict concurrency）は、 *そのアンバランスの「弱い側」をモロに踏んでる*。

## じゃあ、Swift をどうやって学習させる？

ここが本記事の肝。「ローカル LLM は Swift を知らない」が原因と分かったので、**じゃあ知識をどう注入するか** を真面目に考えた。書きながら手を動かすうちに、 *気づけば 3 つ別プロジェクトを立ち上げる*ことになった。本節はその全体マップになる。

選択肢は ３つある。安い順から、A → C。

```
A. プロンプトに「Swift カンペ」を差し込む    ← 一番安い、効果未知
B. RAG で Swift Book / Evolution を retrieval  ← 中コスト、広く効くはず
C. LoRA で 14B〜22B モデルを fine-tune        ← 高コスト、効果最大候補
```

それぞれ「コスト・期待効果・どんな副作用が出そうか」が違う。並べて比較できるように、**同じ 9 モデル × 同じ M2DX-Core コード × 同じプロンプト**で v1 (本記事の baseline) からの改善幅を測る設計にしている。

### A. プロンプトに「Swift カンペ」を差し込む（一番安い）

コードを貼る前に、こういう pre-amble を入れる:

```text
You are reviewing Swift 6 code with these conventions in mind:

- Tuples like `(A, A, A)` are stack-allocated, NOT heap arrays.
- `&+` / `&-` / `&*` are intentional wrap-around operators.
  Absence of overflow check is by design, not a bug.
- `Int32(clamping: x)` saturates instead of truncating.
  No precision loss to flag.
- `dict[key]` returns Optional, not crash. Verify before flagging
  "OOB access".
- `@inline(__always)` switch-case fan-outs are intentional hot-path
  optimizations; do not flag as "redundant".

When reviewing, verify each potential issue against these
conventions before reporting.
```

これだけで「`&-` は overflow 未チェック」みたいな絶対的な誤検出は **ほぼ消えるはず**。Claude Code の `CLAUDE.md` や aider の設定に書いておけば毎回同じ前提で走る。

> **仮説 (A)**: Swift セマンティクス系の誤検出 (前述 5 系統) を 70% 以上削減  
> **コスト**: prompt 末尾に数百トークン足すだけ  
> **副作用**: ほぼ無し（プロンプトが少し長くなるだけ）  
> **状態**: ✅ **完了**。9 モデルで再走行した結果は次節 (A の答え合わせ) にまとめた

### B. RAG で Swift Book / Evolution を引かせる

The Swift Programming Language Book と Swift Evolution proposals を chunk にしてベクトル DB に入れ、LLM が `&+` や `tuple` のような語彙に出会ったとき、 *該当する仕様ページを動的に retrieval してプロンプトに差し込む*構成。

A は人間がカンペを手書きする。B は **公式仕様書をまるごと検索エンジンとして使う**。だから A が見落とした「actor isolation」「Sendable」「`guard let` の制御フロー」みたいな周辺領域にも、自動的に手が届く可能性がある。

llama-index か LangChain で 2〜3 日。embedding は `bge-large` をローカル ollama で動かせばクラウドに何も出さなくて済む。

> **仮説 (B)**: A よりカバー範囲が広く、 *自分が思いつかなかった誤読パターン*にも効く  
> **コスト**: 中。chunk + embed + 検索パイプラインの構築  
> **副作用**: prompt が長くなる（推論時間と TPS への影響）  
> **状態**: 📝 **spec ドキュメント完了、実装待ち**。`swift-audit-rag` という別プロジェクトとして切り出した

### C. LoRA でファインチューニングする — 一番効きそうで、一番楽しい

A と B は「推論時にプロンプトで何かを注入する」方向。C はそうじゃなくて、 **モデル本体の重みに Swift 知識を刻み込む**。

Mac Studio M3 Ultra 96GB なら `mlx-lm` で 14B〜22B クラスの LoRA fine-tuning が普通に動く。学習データの構成案:

- **The Swift Programming Language Book + Swift Evolution**（〜1000ページ）を Q&A 形式に整形
- **OSS の「言語の細部を踏んでる Swift コード」**: swift-foundation、swift-collections、swift-numerics、AudioKit など
- **自分の M2DX-Core / MIDI2Kit のコード**（DSP / RT / Q24 固定小数点 のような *狭くて深い*知識を埋め込む）
- そして本記事のいちばん美味しい再利用先: **「誤検出 → 正解」の対照学習データ** — 今回集まった **52 件の false positive をそのまま DPO** に流す

```jsonl
{
  "prompt": "Is `let rawInc = (4 + (qrate & 3)) << (8 + (qrate >> 2))` an overflow bug?",
  "rejected": "Yes, the shift could be large and overflow Int32.",
  "chosen":   "No. The previous line `qrate = min(63, ...)` clamps to 63, so the shift count ≤ 23 and the result ≤ 7<<23 = 58M, which fits Int32 with margin."
}
```

つまり、 ***LLM が今回間違えた事例を、LLM 自身に学ばせる***フィードバックループを組める。これが C のいちばん面白いポイントだと思う。

ベースは **qwen2.5-coder:14b** か **codestral:22b**。GPU 時間で半日〜1日、データ作りに数日。 *DGX Spark を買う言い訳がここにある*（けど、Mac Studio M3 Ultra で完結する見込み）。

ただし、こうして特化モデルを作っても **agentic harness（依存先 grep + 行番号検証）は別軸の問題**で、これは harness 側で解決する必要がある。両方そろって初めて完璧な構成になる。

> **仮説 (C)**: Swift 系誤検出を 80% 以上削減 + プロンプト augmentation 不要  
> **コスト**: データ整形 3-4 日 + GPU 6-24 時間 + 評価 1 日  
> **副作用 (重要)**: 他言語の能力（Python, JS）が劣化していないかを **HumanEval-Python で常時測定**する必要あり  
> **状態**: 📝 **spec ドキュメント完了、実装待ち**。`swift-audit-lora` という別プロジェクトとして切り出した



### 全体マップ

| 案 | コスト | 仮説効果 | 進行 |
|---|---|---|---|
| **A** cheat sheet | 低 | Swift 誤検出 -70% | ✅ 完了 — **36→9 件 (-75%)、TP 0 据置** |
| **B** RAG | 中 | カバー範囲広 | 📝 spec 完了 (`swift-audit-rag`) |
| **C** LoRA | 高 | 最大効果 + 副作用 | 📝 spec 完了 (`swift-audit-lora`) |

### 続編予告

A → B → C と段階的に、 *Swift 知識をどこに置けば、ローカル LLM の何が改善するか* を測っていく。最終的に並べる比較表のイメージはこう:

| | v1 baseline | v2 (A) | v3 (B) | v4 (C) | v5 (B + C) |
|---|---|---|---|---|---|
| 総 findings | 36 | **9** | ? | ? | ? |
| **真陽性 (TP)** | **0** | **0** | **?** | **?** | **?** |

「?」の B と C は、続編 2 本に持ち越し:

- **続編 #1（B: RAG）**: 自分が思いつかなかった誤読パターンが retrieval でカバーされるか
- **続編 #2（C: LoRA）**: 52 件の false positive を訓練データに転用したフィードバックループの効き目。 ***LLM が間違えた事例で LLM を再教育するとどうなるか***、というメタ的に面白いやつ

特に C は、 **本記事で集まったデータがそのまま訓練データに化ける**ところがいちばん美味しい。失敗が再起動の燃料になるという、再帰的な気持ちよさがある。

進捗が出たら、それぞれ別記事で報告する予定。

## A の答え合わせ — 36 件 → 9 件 (-75%)、TP は依然 0

「Swift カンペを差し込むだけで誤検出が消えるはず」という雑な仮説 (A)、そのまま 9 モデルで走らせた。同じコード、同じ prompt、違うのは先頭に約 800 トークンの cheat sheet を貼ったかどうかだけ。

結果がこう。

| モデル | v1 findings | v2 findings | 変化 |
|---|---|---|---|
| phi4:14b | 1 | **4** | ↑ 増加 (FP のまま) |
| qwen3:14b | 4 | timeout | × 60 分でハング |
| gemma3:12b | 8 | 1 | ↓ -88% |
| hermes3:8b | 4 | 0 | ↓ -100% (no bugs found) |
| codestral:22b | 10 | 1 | ↓ -90% |
| gemma4:31b | 3 | 3 | →（数は同じ、指摘内容は Swift 系→DX7 ドメイン系に変質） |
| qwen3.6:35b | 0\* | 0\* | 変わらず (出力 0 byte) |
| llama3.3:70b | 5 | 0 | ↓ -100% (no bugs found) |
| mixtral:8x22b | 1 | timeout | × 60 分でハング |
| **合計** | **36** | **9** | **-75%** |

\* parse 不可 (reasoning だけ大量に吐いて JSON が出ない)

**TP は 0 → 0。**仮説の半分は当たり、半分は外れた。

### 観察

**1. 量的には大成功**: -75%。"`&-` を未チェック overflow と誤判定" のような **絶対的な誤検出はほぼ消えた**。9B〜70B の "素直なモデル" (hermes3, llama3.3, gemma3) は cheat sheet を読んで "no bugs found" と素直に黙る側に傾いた。

**2. でも TP は増えない**: 当然と言えば当然で、cheat sheet は「**この既知パターンを誤って bug 扱いするな**」とは言えるが、「**未発見のバグを掘り出せ**」とは言えない。 *誤検出を抑える効果と、本物を見つける能力は別軸*。

**3. cheat sheet 違反の常習者**: codestral:22b は「`min(algorithm, 31)` の clamp を見落とすな」と明示的に書いた箇所をまさにそのまま flag してきた。 *モデルの instruction-following が弱いと、cheat sheet の効きも限定的*。

**4. 大型モデルが timeout する副作用**: qwen3:14b と mixtral:8x22b は、cheat sheet の "verify each potential issue against these conventions" 指示を真面目に解釈して、 *モデル内の reasoning loop に入ったまま 60 分 戻ってこなかった*。**pre-amble の制約は LLM を慎重にさせるが、慎重すぎると詰む**。

**5. 誤検出が "Swift 系 → DX7 ドメイン系" に変質**: gemma4:31b が顕著。v1 では tuple/`&+` 誤読だったのが、v2 では「DX7 attack jump 1716 のロジックがおかしい」「noteOff の advance(3) が DX7 spec に反する」みたいな **ドメイン誤読** に shift した。Swift 知識の補強だけでは domain-specific FP は防げない。

**6. phi4:14b だけは増えた (1→4)**: 仮説と逆方向。 cheat sheet で「もっと細かく見ろ」と言ったつもりが、Swift 知識が薄いまま視野を広げた結果、Swift 系 FP がかえって増えた。 *小型モデルには cheat sheet が裏目に出る*。

### 結論 (A について)

仮説 ✅ 「**Swift セマンティクス系の誤検出は 70% 以上削減**」→ **76% で達成**。  
仮説 ❌ 「TP が増える」→ そんなこと言ってないけど、誰も期待してた数字 (TP > 0) には届かず。

cheat sheet は **「ローカル LLM のノイズを減らす」** には実用的に効く。前回 36 件の山を読まされる苦行が 9 件まで減った。 *ただしそこから真陽性を生み出す力はないので、本物のバグを夜中に拾い上げてくれるアシスタントは依然として frontier + agent harness が要る*。

B (RAG) と C (LoRA) の答え合わせは、別記事に持ち越し。

## ところがリファクタリングだと話が逆転する

ここまで「ローカル LLM は監査に向いてない」と書いてきた。でも *別のタスクならどうなんだろう?* と気になって追加実験をやった。テーマは **「同じ DX7Envelope のコードに対して、リファクタリング案を出させてみる」**。

なぜ気になったかと言うと、

- バグ監査 = 「真陽性 / 偽陽性」の判断が必要 → false positive 1件で信頼ゼロに
- リファクタリング = **コンパイル通る? + テスト通る? + 出力が同一?** で機械的にスコアできる
- LLM の本領（パターン変換）に近いタスク

評価軸はこの3段で:

1. ✅ `swift build` 通過
2. ✅ 既存テスト全 pass
3. ✅ **8 ケース（attack-only / full ADSR / staccato / piano-long など）の envelope 出力が完全一致** ← Int32 シーケンスの RMS が 0

これを 10 モデルに同じプロンプト（「`DX7Envelope.advance` を refactor。出力動作を完全に保ったまま読みやすくして」）で投げる。

### 結果

| モデル | LLM 時間 | 結果 |
|---|---|---|
| qwen2.5-coder:7b | 8 s | ❌ build_failed |
| hermes3:8b | 9 s | ❌ build_failed |
| phi4:14b | 14 s | ✅ exact_match |
| qwen3:14b | 70 s | ✅ exact_match |
| gemma3:12b | 12 s | ✅ exact_match |
| codestral:22b | 20 s | ✅ exact_match |
| gemma4:31b | 184 s | ✅ exact_match |
| qwen3.6:35b | 194 s | ✅ exact_match |
| llama3.3:70b | 62 s | ✅ exact_match |
| mixtral:8x22b | 456 s | ❌ build_failed |

**10 モデル中 7 が完璧通過 (70%)**。バグ監査の 0% から劇的に逆転した。

### 失敗パターン

`hermes3:8b` の失敗は前回記事のパターンそのまま:

```swift
if let lastLevel = levels[safe: ix + 1] { ... }
```

`levels` は `(Int, Int, Int, Int)` の tuple なので subscript できない。 *Swift tuple を Array と勘違いするやつ、refactor タスクでも普通に出てくる*。

`mixtral:8x22b` (141B MoE、最大モデル) はもっと派手で、こんなコードを出してきた:

```swift
newLevel = levelsArray[ix]  // ← let も var もない
```

未宣言変数。さらに `advance()` の中に **`getsample()` の積分ロジックを混ぜ込んで while ループにしようとした**形跡があって、 *アーキテクチャを根本的に勘違い*してる。「最大モデル = 最良の結果」ではない、いい例。

### 通った 7 モデルでも質はバラバラ

形式上は全部 exact_match で同じスコアだが、コードを並べると面白い差が見える。

**質的にベスト**: gemma4:31b は **Swift 5.9 で導入された expression-style switch** を使ってきた:

```swift
let stageLevel = switch ix {
    case 0: levels.0
    case 1: levels.1
    case 2: levels.2
    case 3: levels.3
    default: 0
}
```

これは modern Swift の idiom で、人間が書くベストプラクティス寄り。

**質的にワースト寄り**: phi4:14b、qwen3:14b、codestral:22b、llama3.3:70b、qwen3.6:35b の5つが揃って同じパターンに着地:

```swift
let levelsArray = [levels.0, levels.1, levels.2, levels.3]
let newLevel = levelsArray[ix]
```

…これ、 **Array 値を毎回新しく作ってる**（最適化で消える可能性はあるが、ヒープバッファ確保が走り得るパターン）。前回章で他のモデルが「Array creation in real-time path!」と誤検出してきた、まさにそのパターン。 *皮肉なことに、refactor を任せると今度は自分で同じ「違反」をやらかす*。

ただし `advance()` はステージ遷移時にだけ呼ばれる関数で `getsample()` のような毎ブロック関数ではないので、現実の audio thread への影響は **実質ゼロ**。それでも「RT 安全性を理解してない」という意味では同じ。

**興味深い refactor**: qwen3.6:35b は **再帰を while ループに展開**してきた:

```swift
while targetLevel == level {
    if stageIx == 3 && down { return }
    stageIx += 1
    guard stageIx < 4 else { ix = -1; return }
    // ...
}
```

これは構造的に大きな refactor で、テストケースでは exact_match。ただ細かく追うと *「stage 0,1,2 すべて L_n == 既存 level かつ stage 3 で down=true」* という極端な経路で `self.ix` の最終値が変わる可能性がある（テストでは triggered せず）。 **eval を通っても、untested code path には微妙な行動差が混入し得る**、という小さな注意。

### つまり何が言えるか

- バグ監査 + セキュリティ監査 = **0/52件** TP（全敗）
- リファクタリング = **7/10モデル** が exact_match（70%成功）

この差は何かというと、

- バグ監査は **「コードに問題がある」と判断する側に偏った仕事**。LLM はもともと「危険そうなパターン」に過敏なので、false positive を量産する
- リファクタリングは **「同じ動作を別の書き方で」** という変換タスク。LLM の本領にハマる
- そして決定的なのは **正解が機械判定できる**こと（build + test + RMS = 0）。間違ったら即 reject される。人間が判定する余地がない

つまりローカル LLM を実用に乗せるかどうかの境目は、 **「LLM が間違ったら自動で弾けるパイプライン」を組めるか**にある気がする。バグ監査は「人間が判定」で詰む。リファクタリングは「テスト」で守れる。

## というわけで、向いてなかった……は半分だけ正しい

「ローカル LLM は使い物にならない」と言いたいわけではない、というのは **冒頭からそう書いてきた通り**。問題は **タスクとの相性**:

- **監査**: false positive が一つでも混ざると、残り全部の信頼が崩れる。50件読んで1件のバグを掘り当てるより、最初から全部捨てた方が早い、になる。コードに書かれてない情報（依存ライブラリ、OS API、言語仕様の細部）に重く依存する。ローカルLLMはここで決定的に負ける
- **リファクタリング**: ↑で書いたとおり、機械判定が可能で、変換タスクに近く、ローカルLLMでも7割通る

逆にローカルLLMが「ローカルである意味」が出るのは、

- 完全オフライン環境（飛行機・出張先）
- 外に出したくない短いメモや会話の要約
- 概念や用語のサクッと確認（ネット代わり）
- **テストでガードできる範囲のリファクタリング** ← 今回の発見

…で、「OSSのレビューを夜中に走らせたい」は、現時点では別アプローチを考えた方がいい。一方「テスト充実してる箇所のリファクタリングを夜間バッチで」は、案外いけそう。

## じゃあ DGX Spark か Mac Studio 256GB 買えば解決する？

ここまで書いてくると、「もっといいハード買えば突破できるのか」という気持ちが湧いてくる。**NVIDIA DGX Spark**（128GB LPDDR5x、2台で 256GB リンク。米国 NVIDIA Marketplace 直販で執筆時点 $4,699 ≈ 約 70 万円、国内は流通・税で変動）か、**Mac Studio M3 Ultra 256GB**（256GB / 1TB 構成だと国内 100 万円台前半〜）のどっちか。 *最初は「512GB 構成で 405B を一発で乗せたい」と妄想していたんだけど、Apple は M3 Ultra の上限を **256GB に縮めた**(現行 Apple Store 構成では 512GB が引っ込んでる) ので、選択肢から消えた*。…ところがこれ、 **解決しない** という結論にしかならなかった。

理由は今回観測した失敗パターンを並べてみると分かる:

| 失敗 | パラメタ数で消える？ |
|---|---|
| 行番号の捏造 | 大型モデルでも続きうる（405B 単体未検証） |
| `min(63,…)` 見落とし | サイズで微改善、根本治らず |
| Swift tuple → Array 誤認 | 学習データ依存、サイズの問題ではない |
| `&+` / `&-` セマンティクス誤読 | 同上 |
| 依存ライブラリ未参照 | **agent化しないと無理（パラメタ数の問題ではない）** |

最後のがいちばん効いてる。**プロンプトの外を grep する能力**がないと、「上流ライブラリでクランプ済」「OS API で守られてる」「sandbox がある」みたいな情報を加味した監査ができない。これはモデルサイズじゃなくて **アーキテクチャ（ツール呼び出し）の問題**。

参考までに2機種を並べると:

|  | **DGX Spark** | **Mac Studio M3 Ultra 256GB** |
|---|---|---|
| 価格 | 約 70 万円〜（米 $4,699） | 256GB/1TB 入門で 100 万円台〜 |
| 統合メモリ | 128GB（2台で 256GB リンク） | 256GB（**現行 Apple Store では上限**） |
| 帯域 | 273 GB/s | 819 GB/s |
| CUDA | ◎ | × (MLX で代替) |
| 一番効く用途 | fine-tuning / RDMA | 大型モデル単体 inference |

推論目的なら **Mac Studio の 800 GB/s 帯域のほうが効く**（llama 405B Q4 が単体で乗る）。fine-tuning や CUDA エコシステムを使うなら DGX Spark。 *…が、どっちでも今回の失敗パターンは消えない*。

じゃあ何で解決するかというと、結局この4象限のどこを取るかになる:

| 構成 | 精度の感触 | 例 |
|---|---|---|
| **Frontier model + agentic harness** | ◎ | Claude Code / Codex CLI / aider + GPT-5 / Claude Opus |
| **ローカル中型 LLM + agentic harness** | ○ | aider や cline + qwen2.5-coder / codestral / llama3.3:70b |
| ローカル大型 LLM 単発呼び出し ← **今回やったやつ** | × | ollama に 141B / 405B 投げて返事を待つ |
| ローカル中型 LLM 単発呼び出し | × | ollama に 7B〜70B 投げて返事を待つ |

ポイントは、**「単発 vs エージェント」の差のほうが「中型 vs 大型」の差より大きい**ところ。 *70Bにツールを持たせた構成のほうが、405Bにツールなしで単発で投げるより精度が出る*、というのが最近の肌感。

なぜ agent harness が効くかというと、

- **依存ライブラリのコードを実際に grep して読める**（今回の `maxBufferSize` のような「上流で何が保証されてるか」が分かるようになる）
- **diff 適用時に行番号が合わなければ即エラーで落ちる**（捏造された L364〜L405 みたいな指摘が構造的にフィルタされる）

の2点が決定的に効く。今回 ollama 単発で詰んだ失敗パターンのうち、**「依存ライブラリ未参照」「行番号の捏造」はツール化するだけで構造的に消える**。

…が、これでも消えないのが **「Swift tuple を Array と勘違い」「`&+` を意図しない wrap と誤読」みたいな言語セマンティクスの誤読**で、ここはモデルの素の能力で決まるので harness では救えない。ローカル中型 + agent でもここはフロンティアに敵わなくて、結果として「ローカルで agent 化」は完全な解にはならない。

というわけで「夜中に勝手に走らせるレビューワー」を目的にすると、現状ベストは **Claude Code か Codex CLI を crontab で回す** に落ち着いた。OSS なら privacy 懸念もないし、subscription の範囲でだいたい収まる。

つまり「夜中にレビューを回したい」が目的なら、**70 万円のハードを増やすより crontab で Codex CLI を回したほうが安くて確実**で、これに落ち着いた。OSS なので privacy 懸念もないし。

…なんだけど、 *DGX Spark の見た目はめっちゃカッコいい*ので、買う言い訳が他にあるなら買うのは全然アリだと思う。

## まとめ

- **ollama の num_ctx はデフォルト 2048** ← 最初にこれで嵌まった
- ローカル LLM 10機種にバグ + セキュリティ監査をさせた結果、**累計 52 findings、TP は 0件**
- 共通の失敗モード: 防御コードの見落とし、Swift セマンティクス（tuple, `&+`）の誤読、依存ライブラリ未参照、行番号の捏造
- 監査タスクは false positive が許されないので、現状の70B〜141Bでも厳しい
- **Swift カンペを pre-amble に貼った A 案の追加実験**: バグ監査は **36 件 → 9 件 (-75%)** に減ったが、**TP は依然 0**。ノイズは減らせるが、未発見のバグを掘り出す力にはならない
- **ところがリファクタリングを試したら 10/10中 7 が exact_match** で通った。タスク次第で景色が逆転する
- 境目は **「LLM の出力を機械判定で弾けるパイプライン」** が組めるかどうか。テストで守れる作業なら、ローカルLLMは案外戦力になる

「コードの問題を勝手に見つけて issue 起票してくれる便利アシスタント」を期待してたなら、 **今のローカル LLM ではまだ無理**。けど「テスト整備済みのコードを夜間バッチで refactor する」みたいな限定された使い方なら、ローカルでも十分に手伝ってくれる。

---

監査対象の M2DX-Core は Apache 2.0 で公開している。検証用のプロンプトと出力ログも残してあるので、別モデルで再走行して反証してくれる方がいたら歓迎。

- **M2DX-Core (OSS)**: https://github.com/hakaru/M2DX-Core
- **M2DX (iOS app)**: https://apps.apple.com/jp/app/m2dx/id6753466996
