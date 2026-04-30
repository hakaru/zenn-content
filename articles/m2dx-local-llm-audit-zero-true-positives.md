---
title: "ローカルLLM 10機種にDX7エンジンの監査をやらせたら、57件中ハズレ57件だった話"
emoji: "🤖"
type: "tech"
topics: ["llm", "ollama", "swift", "security", "codereview"]
published: false
---

## 夜中に勝手に働いてくれるレビューワー、欲しくない？

iOS/macOS向けのMIDI 2.0 FM音源アプリ **M2DX** を一人で書いている。心臓部の **M2DX-Core**（DX7互換エンジン、Pure Swift）はOSSで公開していて、そろそろ第三者の目を入れたい時期だった。

普段の開発支援は Claude Code と Codex を使ってるんだけど、欲しいのはちょっと違う方向性で、「リポジトリ全体を機械的に舐めて、夜中に勝手に走らせて朝起きたら issue が並んでる」みたいなレビューワーが欲しかった。手元には **Mac Studio M2 Ultra / 96GB**。70B〜141Bクラスのローカルモデルがちゃんと動くスペック。これを遊ばせとくのもなあ、と思って ollama を立ち上げた。

結論を先に書くと:

- **バグ監査**: 10モデル走らせて **41件指摘 → 真陽性 0件**
- **セキュリティ監査**: 5モデル中4モデル完走、**16件指摘 → 真陽性 0件**
- 累計 **57件、当たり 0件**

全敗である。なぜそうなったかを書いていく。

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
- 端末のスピナー（`⠙ ⠹ ⠸`）が出力に混ざる事故が起きる
- そして最大の問題: **`num_ctx` のデフォルトが 2048 トークン**

最後のこれに気づくのに時間を食った。ファイル全体を貼ったつもりが「LLMが50行目以降を全く知らない」みたいな奇妙な挙動になって、しばらく「お前ホント賢くないな…」とか思ってたら自分が悪かった。

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

## プロンプトをどう書いたか

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

走らせたのはこの10モデル:

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

\* qwen3.6:35b は **2万2千トークンの reasoning** を吐いてから結局JSON 1行も出さずに終了。  
llama4:scout は HTTP 500 で 0 byte。

検証は地道に手作業で、各 finding の引用行を実コードと突き合わせて、「本当にバグか / 行が実在するか」を一個ずつ見ていった。

41件の指摘を全部読んで、いくつかの**よく似たパターン**で間違えていることに気づいた。これが面白いのでパターン別に並べる。

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

…と。**Swift の tuple はスタック確保**で、Array とは全然違う。「6個並んでる、配列だ」っていう連想だけで指摘してきてる。さらに `@inline(__always)` 付けて switch-case で展開してるホットパスを「冗長だから関数化しろ」とまで言われた。 *いや、だからわざわざ inline 展開してんだって…*

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

### あと、行番号を平気で捏造する

70Bでも安心できないのが、 **存在しない行番号を引用してくる** こと。

- `llama3.3:70b`: `DX7Operator.swift L100-L120` → ファイル全体で 85 行
- `gemma3:12b`: `DX7Envelope.swift L192-L216` → 実 178 行
- `mixtral:8x22b`: `DX7Voice.swift L364-L405` を `advanceStage` 関数として指摘 → 実関数は L54-L95（全然違う場所）

「L300〜350」とか書いてあると、人間は「具体的だな、見にいくか」と思う。これがブラフだった、というのが何度もあった。**具体性 ≠ 正確性**。

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

**実態**: iOS/macOS のファイルシステムは **filename に `/` を含められない**。`URL.lastPathComponent` も単一コンポーネントしか返さないので、`../` という構造を持ったファイル名は OS レイヤで存在しえない。理屈は完璧、でも前提が成立しない。

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

## なぜ全敗したのか、自分なりの整理

3つあると思う。

### 1. 防御コードを「読まない」

LLMは「危険そうなパターン」（シフト演算、配列アクセス、文字列パース）を見つけるのは得意。でも **その3行前にある `min(63, ...)` や `guard data.count == ...`** を読み飛ばす。学習データの中に「危険なシフト演算 → overflow」というペアが大量にあって、それを引っ張ってきてるんだろう。**コンテキストの局所性**が低い。

### 2. 言語固有のセマンティクスが弱い

Swift はメジャー言語のはずだけど、ローカルLLMは細部で結構間違える:

- **tuple ≠ Array**（heap allocation じゃない）
- **`&+` / `&-`** は明示的wrap（チェックが「ない」のではなく「いらない」）
- **`Int32(clamping:)`** は精度を失わない（飽和してくれる）
- **Dictionary subscript** は Optional を返す（クラッシュしない）

C/C++ で覚えた感覚で類推してくる、と言うと当たってる気がする。

### 3. プロンプトの外を見られない

`UMPSysEx8Assembler` の `maxBufferSize: 65536` は MIDI2Kit のソースを開かないと分からない。ローカルLLMは依存先を grep する手段がないので、**「上流で何が保証されてるか」を加味した監査**ができない。

これは agent 化（grep/Read を投げ返せるツール）すれば改善するはずだけど、ollama 単体では無理。

## というわけで、向いてなかった

「ローカル LLM は使い物にならない」と言いたいわけではない。問題は **タスクとの相性** で、コード監査は特に厳しい組み合わせだった:

- false positive が一つでも混ざると、残り全部の信頼が崩れる。50件読んで1件のバグを掘り当てるより、最初から全部捨てた方が早い、になる
- コードに書かれてない情報（依存ライブラリ、OS API のセマンティクス、言語仕様の細部）に重く依存する。ローカルLLMはここで決定的に負ける

逆にローカルLLMが「ローカルである意味」が出るのは、

- 完全オフライン環境（飛行機・出張先）
- 外に出したくない短いメモや会話の要約
- 概念や用語のサクッと確認（ネット代わり）

…くらいかな、というのが今のところの実感。「OSSのレビューを夜中に走らせたい」は、現時点では別アプローチを考えた方がいい。

## じゃあ DGX Spark か Mac Studio 256GB 買えば解決する？

ここまで書いてくると、「もっといいハード買えば突破できるのか」という気持ちが湧いてくる。**NVIDIA DGX Spark**（128GB、2台で 256GB リンク、$4k 〜）か、**Mac Studio M3 Ultra 256GB**（$9k 〜、512GB 構成可）のどっちか。…ところがこれ、 **解決しない** という結論にしかならなかった。

理由は今回観測した失敗パターンを並べてみると分かる:

| 失敗 | パラメタ数で消える？ |
|---|---|
| 行番号の捏造 | 405Bでも続く |
| `min(63,…)` 見落とし | サイズで微改善、根本治らず |
| Swift tuple → Array 誤認 | 学習データ依存、サイズの問題ではない |
| `&+` / `&-` セマンティクス誤読 | 同上 |
| 依存ライブラリ未参照 | **agent化しないと無理（パラメタ数の問題ではない）** |

最後のがいちばん効いてる。**プロンプトの外を grep する能力**がないと、「上流ライブラリでクランプ済」「OS API で守られてる」「sandbox がある」みたいな情報を加味した監査ができない。これはモデルサイズじゃなくて **アーキテクチャ（ツール呼び出し）の問題**。

参考までに2機種を並べると:

|  | **DGX Spark** | **Mac Studio M3 Ultra 256GB** |
|---|---|---|
| 価格 | $4,000 〜 | $9,000 〜 |
| 統合メモリ | 128GB (2台で256GB) | 256GB (上位 512GB) |
| 帯域 | 〜270 GB/s | 〜800 GB/s |
| CUDA | ◎ | × (MLX で代替) |
| 一番効く用途 | fine-tuning / RDMA | 大型モデル単体 inference |

インフェレンス目的なら **Mac Studio の 800 GB/s 帯域のほうが効く**（llama 405B Q4 が単体で乗る）。fine-tuning や CUDA エコシステムを使うなら DGX Spark。 *…が、どっちでも今回の失敗パターンは消えない*。

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

つまり「夜中にレビューを回したい」が目的なら、**$4k のハードを増やすより crontab で Codex CLI を回したほうが安くて確実**で、これに落ち着いた。OSS なので privacy 懸念もないし。

…なんだけど、 *DGX Spark の見た目はめっちゃカッコいい*ので、買う言い訳が他にあるなら買うのは全然アリだと思う。

## まとめ

- **ollama の num_ctx はデフォルト 2048** ← 最初にこれで嵌まった
- ローカル LLM 10機種を回した結果、**累計 57 findings、TP は 0件**
- 共通の失敗モード: 防御コードの見落とし、Swift セマンティクス（tuple, `&+`）の誤読、依存ライブラリ未参照、行番号の捏造
- バグ監査やセキュリティ監査のような **「false positive が許されないタスク」** には、現状の70B〜141Bでも厳しい

「コードの問題を勝手に見つけて issue 起票してくれる便利アシスタント」みたいなのを期待してたなら、 **今のローカル LLM ではまだ無理**、というのが結論。

---

監査対象の M2DX-Core は Apache 2.0 で公開している。検証用のプロンプトと出力ログも残してあるので、別モデルで再走行して反証してくれる方がいたら歓迎。

- **M2DX-Core (OSS)**: https://github.com/hakaru/M2DX-Core
- **M2DX (iOS app)**: https://apps.apple.com/jp/app/m2dx/id6753466996
