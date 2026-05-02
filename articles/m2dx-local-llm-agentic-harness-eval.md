---
title: "ローカルLLMって本当に開発に使える？（３）aiderを試してみる"
emoji: "🛠️"
type: "tech"
topics: ["llm", "ollama", "aider", "swift", "codereview"]
published: false
---

## 前回のあらすじ

ひと月前くらいに [ローカルLLMって本当に開発に使える？DX7エンジンの監査をやらせたら、全然だめでした。。](https://zenn.dev/hakaru/articles/m2dx-local-llm-audit-zero-true-positives) という記事を書いた。要点だけ:

- iOS/macOS向けのMIDI 2.0 FM音源アプリ **M2DX** ([TestFlight 公開ベータ](https://testflight.apple.com/join/BAtGszPw)) を開発中、心臓部は OSS の DX7 互換エンジン **M2DX-Core**
- 手元の Mac Studio M3 Ultra 96GB で ollama に **10 機種のローカル LLM** (qwen3, gemma4, codestral, llama3.3:70b, mixtral:8x22b 等) を動かし、バグ監査 + セキュリティ監査をやらせた
- 結果: **累計 52 件の指摘 → 真陽性 0 件**

その記事の最後で、解決策として 4 象限マトリクスを描いた:

| | **単発呼び出し** | **+ agentic harness** |
|---|---|---|
| **Frontier** (GPT-5 / Claude Opus) | – | ◎ Claude Code / Codex CLI |
| **ローカル中型** | × ← 前回やったやつ | **○ ← 本記事で検証** |

「ローカル中型 LLM に grep / read ツールを持たせたら、単発呼び出しから **どこまで** 改善するのか?」  
これを実測してみたのが本記事。

## 何を測りたいのか

前回観測した失敗パターンは大きく 2 種類に分けられる:

1. **knowledge axis** — Swift tuple ≠ Array 誤認、`&+`/`&-` の wrap 演算子セマンティクス、DX7 ドメイン知識など。これはモデルの素の能力で、cheat sheet / RAG / LoRA で対処する別軸 (前回記事の続編 B / C 案)
2. **capability axis** — 「**プロンプトの外を見られない**」ことに起因。依存ライブラリ未参照、行番号の捏造など

agentic harness は **(2) を狙い撃ち**できる。grep / read で上流コードに触れられるなら、「知らないなら捏造」を「実コードを引いて答える」に変えられるはず。

これを定量化するために、**コントロール質問** を仕込んだ:

> Q: `DX7SysExParser` に届く前に、SysEx のバイト数は upstream で cap されているか? されているなら **どのファイル / 何行目 / 上限値**を答えよ。

答えるには **MIDI2Kit のソースを実際に grep する**しかない。各モデルが (i) 正解する / (ii) 捏造する / (iii) "わからない" のどれを返すかが、capability axis の binary signal になる。

## 正解 (ground truth)

先に答えを書いておく。MIDI2Kit のソースから:

```swift
// MIDI2Kit/Sources/MIDI2Core/UMP/UMPSysEx7Assembler.swift, L41
public init(maxBufferSize: Int = 65536) {
    self.maxBufferSize = maxBufferSize
}

// L77
let newSize = buffers[group]!.count + bytes.count
guard newSize <= maxBufferSize else { ... }
```

`UMPSysEx8Assembler.swift` も同じく `maxBufferSize: Int = 65536` がデフォルト。  
つまり **64 KB で打ち切り**。これより大きい SysEx は MIDI2Kit のレイヤで弾かれて、`DX7SysExParser` まで届かない。

これが "正解" の数字。各モデルがこれをどう答えるか見ていく。

## 実験設計

### 監査対象 (1,800 行)

M2DX で **untrusted な外部入力が触れる経路全部**:

- `DX7SysExParser.swift` (120行) — iCloud Drive / AirDrop 経由の `.syx` パース
- `UserBankManager.swift` (90行) — fileImporter まわり
- `MIDIInputManager.swift` (1,132行) — MIDI 2.0 UMP デコード、PE JSON
- `PEResponderHost.swift` (374行) — Property Exchange responder
- `USBResetHelper.c` (84行) — IOKit USB reset

### 比較セル (3 モデル × 2 mode = 6 セル)

| | **aider** (= ollama + grep/read 可能) | **single-shot** (= ollama API 単発) |
|---|---|---|
| **codestral:22b** | ✅ 走行 | ✅ 走行 |
| **qwen2.5-coder:14b** | ✅ 走行 | ✅ 走行 |
| **llama3.3:70b** | ✅ 走行 | ✅ 走行 |

aider は `OPENAI_BASE_URL=http://localhost:11434/v1` で ollama を OpenAI 互換 endpoint として叩く。MIDI2Kit のソースを `--read` で渡し、agent が必要なら参照できるようにした。

両 mode で **同じ prompt** (Swift 6 conventions の cheat sheet + 脅威モデル + コントロール質問 + 出力フォーマット) を使う。違うのは「上流ソースに access できるか」だけ。

## 結果

| モデル | mode | findings | control_Q answer | OK? | 時間 |
|---|---|---|---|---|---|
| codestral:22b | aider | 4 | **65536** (MIDI2Kit, 名前微妙誤) | ✅ | 179s |
| codestral:22b | single-shot | 8 | **1024** | ❌ **捏造** | 306s |
| qwen2.5-coder:14b | aider | 4 | **65535** (1 off, guard 境界の解釈) | ≈ | 430s |
| qwen2.5-coder:14b | single-shot | 0 ("no vulns") | **65536** | ✅ | 95s |
| llama3.3:70b | aider | 5 | **4104** (parser layer の別 cap) | ✅ valid | 548s |
| llama3.3:70b | single-shot | 1 | **4104** | ✅ | 295s |

**真陽性 (TP) は 6 セルとも 0** — quick scan の段階で全 22 件は path-traversal (iOS は `/` 不可で防御済) / IOKit memory leak の誤読 / JSON DoS の上流済 cap などの既知 FP パターンに帰着。詳細な finding-by-finding の TP/FP 分類は次回に持ち越し。

## ハイライト #1: codestral の捏造が事実に化けた

ここが本記事のいちばんの収穫。

### single-shot で codestral が言ったこと

```
upstream_capped: yes
evidence_file: MIDI2Kit/Sources/MIDI2Core/UMP/UMPSysEx8Assembler.swift
evidence_line: 56 (`guard numBytes <= maxPacketSize else { return nil }`)
upper_bound_bytes: 1024
```

**これは丸ごと捏造**。実コードに `maxPacketSize` という変数は無い (実際は `maxBufferSize`)。`return nil` のような guard も該当箇所にない。「1024」という数字も実コードには存在しない。

おそらく codestral は学習データ中の「他の MIDI ライブラリの典型的なパケットサイズ」を勝手に持ち出してきた。**自信満々に、しかし全部嘘**。

### aider 経由で codestral が言ったこと

```
upstream_capped: yes
evidence_file: MIDI2Kit/Sources/MIDI2Core/UMP/SysEx7Assembler.swift  ← UMP プレフィクス抜け
evidence_line: 108-113  ← 実際は L41 / L77、ずれている
upper_bound_bytes: 65536 (maxPacketSize)  ← 数字は正解 (名前は maxBufferSize が正)
```

ファイル名と行番号は依然ガタガタだが、 **数字 65536 は正解**。実コードに存在する value で、actual cap を当てている。

つまり: 
- single-shot → 自信満々に嘘
- aider → 細部はズレるが本質を当てる

**同じモデル / 同じプロンプト**で、harness の有無だけでこれが起きた。これが capability axis の効きの直接観察。

## ハイライト #2: qwen14b は単発でも答えられた

ここで予想外。**qwen2.5-coder:14b の single-shot は 65536 を当てた**。

```
upstream_capped: yes
evidence_file: MIDI2Kit/Sources/MIDI2Core/UMP/UMPSysEx8Assembler.swift
evidence_line: L53-L61
upper_bound_bytes: 65536
```

しかも MIDI2Kit のソースを与えていないのに。「学習データに MIDI2Kit が含まれていた」「typical な Swift MIDI ライブラリの default を推測した」のいずれかだろう。

つまり capability axis の効きは **モデル依存**。
- 弱いモデル (codestral) → harness で救われる
- 強いモデル (qwen14b, llama70b) → 単発でも答えられる

「agentic harness を被せれば全部解決する」ではない。 *「捏造する素質のあるモデルが harness 経由なら捏造しなくなる」* という防御的な効果。

## ハイライト #3: llama70b は別レイヤの cap を発見

llama3.3:70b は両 mode とも `4104` と答えた。これは MIDI2Kit の 65536 とは違うが、**別レイヤで実在する valid な cap**:

```swift
// DX7SysExParser.swift, L19-L30
private static let bulkDumpSize = 4104
...
public static func parse(data: Data, ...) -> DX7SysExBank? {
    guard data.count == bulkDumpSize else { return nil }
    ...
}
```

DX7 の 32-voice bulk dump は厳密に 4104 バイトで、これに合わない data は **parser 自身が即 reject** する。なので攻撃者が大きい SysEx を送っても、MIDI2Kit (64KB) を抜けたとしても、parser でさらに 4104 で落とされる。

Llama70b は **この内側の防御を発見**した。ground truth から見れば不正解 (control 質問は upstream cap を聞いた) だが、**本物の防御層を citing したという意味では FP ではない**。

ground truth が一意でないのは control 質問の reflexive な弱点 — 改善余地。

## ハイライト #4: harness は output を "誘発" する

findings 数を mode 別に並べると:

| | aider | single-shot |
|---|---|---|
| codestral | 4 | 8 |
| qwen14b | **4** | **0** ← "no vulnerabilities" |
| llama70b | 5 | 1 |

**aider 経路は findings 数がほぼ一定 (4-5)**、single-shot は **0 〜 8 と大きく振れる**。

特に qwen14b は single-shot だと「no vulnerabilities」で完全沈黙、aider 入れると 4 件出してきた。 **harness がモデルに「探せ」「JSON で答えろ」を強制している**形が見える。

これは TP 率の改善とは別軸の効果。 *findings が多いほど良いわけではない (qwen14b の 4 件もどうせ FP)*、けど "黙りっぱなしのモデル" を「とりあえず output させる」効果は確実にある。

## ハイライト #5: TP 率は依然 0

前回の累計 52 件 TP 0 が、本実験で aider mode 13 件 + single-shot 9 件 = 計 22 件追加されて、**累計 74 件 / TP 0 件**。

agentic harness で **capability axis (上流参照能力) は向上**したが、**knowledge axis (Swift / DX7 ドメインの言語的理解)** は別軸で、harness だけでは TP > 0 に届かない。

主な FP パターンも前回と同じ:
- iOS の `lastPathComponent` で `/` が含まれる前提の path traversal (構造的に不可能)
- `FileManager.copyItem` の symlink follow (Apple は symlink を symlink としてコピーする)
- IOKit IOService のリリース忘れ指摘 (実コードは全パスで release してる)
- JSON DoS (上流 MIDI2Kit で 64 KB cap 済)

→ **agentic harness で「ある種のクラスの FP は防げるが、別のクラスの FP には効かない」** が本実験で観測された境界。

## 時間

| | aider | single-shot |
|---|---|---|
| codestral:22b | 179s | 306s |
| qwen14b | 430s | 95s |
| llama70b | 548s | 295s |

aider mode は **単発より速いケースもある** (codestral)、 *単発より遅いケースもある* (qwen14b)、傾向は一貫しない。aider が repo-map 構築・ファイル読み込み・LLM 呼び出しを内部で何度するかにも依存。

ちなみに aider は `--no-stream` で動かし、`--read` で MIDI2Kit を渡しているが、本実験のセッションで明示的な grep tool calls は transcript に観測されなかった。aider は `--read` で同梱した context を **モデルが必要に応じて見る**だけで、能動的なツール呼び出しは起きていなかった可能性が高い。

つまり今回観測した「capability axis 改善」は **本格的な agent loop ではなく、`--read` 同梱コンテキストの効果**かもしれない。次の実験では、より能動的な agent (Codex CLI / Claude Code を frontier 比較として) を試したい。

## まとめ

- **agentic harness は capability axis を構造的に改善する**: codestral の "1024" hallucination が aider 経由で "65536" に化けた
- ただし **モデル依存**: qwen14b と llama70b は単発でも捏造せず、ground truth に近い答えを出した
- harness の副次効果: **モデルに output を誘発**する (qwen14b 0件 → 4件)
- **TP 率は依然 0** (累計 74 件、全 FP) — knowledge axis の壁は harness では突破できない
- "夜中に走らせるレビューワー" としての実用性: **capability axis ✓ / knowledge axis × → 全体評価は依然 frontier + harness が必要**

要するに、 **harness は弱いモデルを「捏造しないモデル」に変える** が、 **TP を生み出す力は別軸**。前回記事の「ノイズは減らせるが、未発見のバグを掘り出す力にはならない」と同じパターンが、別の角度からも確認された形。

本記事の続編 (RAG / LoRA で knowledge axis を直接攻める) は別プロジェクトで spec 完了済 (`swift-audit-rag`, `swift-audit-lora`)、進捗が出たらまた書きます。

---

実験のスクリプト一式 + 全 transcript + control_Q 解答のフルテキストは `/Users/hakaru/DEVELOP/swift-audit-agentic/` に保管。母体記事の検証用 prompt と output ログは `/tmp/m2dx-audit/` および `/tmp/m2dx-audit-v2/` 。別モデル / 別 harness で再走行して反証してくれる方がいたら歓迎。

- 母体記事: <https://zenn.dev/hakaru/articles/m2dx-local-llm-audit-zero-true-positives>
- **M2DX-Core (OSS)**: <https://github.com/hakaru/M2DX-Core>
- **M2DX (TestFlight)**: <https://testflight.apple.com/join/BAtGszPw>
