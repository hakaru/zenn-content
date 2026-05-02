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

| | **単発呼び出し** | **+ aider などのツール** |
|---|---|---|
| **商用最新モデル** (GPT-5 / Claude Opus) | – | ◎ Claude Code / Codex CLI |
| **ローカル中型** | × ← 前回やったやつ | **○ ← 本記事で検証** |

「ローカル中型 LLM に grep / read ツールを持たせたら、単発呼び出しから **どこまで** 改善するのか?」  
これを実測してみたのが本記事。

:::message
**aider とは**

aider は、ターミナルで動くコード編集アシスタント。LLM に「ファイルを読む・コードを検索する・変更を提案する」といったツールを持たせ、「調べる → 考える → 修正する」のサイクルを自律的に繰り返させる。通常の API 呼び出しでは LLM はプロンプトに書いた内容しか見られないが、aider 経由だとリポジトリを grep したり関連ファイルを読み込んで答えたりできる。

こうした「LLM にツールを持たせて自律的に動かす仕組み」全般を "agentic harness" と呼ぶ。aider のほか Codex CLI や Claude Code も同じ仕組み。本記事では aider 一本で試す。
:::

## 何を測りたいのか

前回観測した失敗パターンは大きく 2 種類に分けられる:

1. **知識軸** — Swift の言語仕様や DX7 ドメインの知識がモデルに入っているかどうか。`tuple ≠ Array`、`&+`/`&-` は意図的なラップ演算子、DX7 の固定小数点表現など。これはモデルの素の能力で、cheat sheet / RAG / LoRA で対処する別軸 (前回記事の続編 B / C 案)
2. **参照軸** — プロンプトに書かれていない外部コードを自分で取りに行けるかどうか。依存ライブラリを見ずに答える、存在しない行番号を引用するなど、「プロンプトの外を見られない」ことが根本原因

aider は **(2) を狙い撃ち** できる。grep / read で上流コードに触れられるなら、「知らないなら捏造」を「実コードを引いて答える」に変えられるはず。

これを定量化するために、**チェック用質問** を仕込んだ:

> Q: `DX7SysExParser` に届く前に、SysEx のバイト数は上流ライブラリで制限されているか? されているなら **どのファイル / 何行目 / 上限値** を答えよ。

答えるには **MIDI2Kit のソースを実際に grep する** しかない。各モデルが (i) 正解する / (ii) 捏造する / (iii) "わからない" のどれを返すかが、参照軸が効いているかを見る判定指標になる。

## 正解

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

各モデルがこの「65536」をどう答えるか見ていく。

## 実験設計

### 監査対象 (1,800 行)

M2DX で **信頼できない外部入力が触れる経路全部**:

- `DX7SysExParser.swift` (120行) — iCloud Drive / AirDrop 経由の `.syx` パース
- `UserBankManager.swift` (90行) — fileImporter まわり
- `MIDIInputManager.swift` (1,132行) — MIDI 2.0 UMP デコード、PE JSON
- `PEResponderHost.swift` (374行) — Property Exchange responder
- `USBResetHelper.c` (84行) — IOKit USB reset

### 比較セル (3 モデル × 2 方式 = 6 セル)

| | **aider あり** (grep/read 可能) | **単発呼び出し** (ollama API 直接) |
|---|---|---|
| **codestral:22b** | ✅ 走行 | ✅ 走行 |
| **qwen2.5-coder:14b** | ✅ 走行 | ✅ 走行 |
| **llama3.3:70b** | ✅ 走行 | ✅ 走行 |

aider は `OPENAI_BASE_URL=http://localhost:11434/v1` で ollama を OpenAI 互換 endpoint として叩く。MIDI2Kit のソースを `--read` で渡し、モデルが必要なら参照できるようにした。

両方式で **同じプロンプト** (Swift 6 の注意事項 + 脅威モデル + チェック質問 + 出力フォーマット) を使う。違うのは「上流ソースにアクセスできるか」だけ。

## 結果

| モデル | 方式 | 指摘数 | チェック質問の回答 | 正解? | 時間 |
|---|---|---|---|---|---|
| codestral:22b | aider あり | 4 | **65536** (ファイル名が微妙にずれ) | ✅ | 179s |
| codestral:22b | 単発 | 8 | **1024** | ❌ **捏造** | 306s |
| qwen2.5-coder:14b | aider あり | 4 | **65535** (guard 境界の 1 ずれ) | ≈ | 430s |
| qwen2.5-coder:14b | 単発 | 0 ("no vulns") | **65536** | ✅ | 95s |
| llama3.3:70b | aider あり | 5 | **4104** (パーサー層の別上限) | ✅ 実在する値 | 548s |
| llama3.3:70b | 単発 | 1 | **4104** | ✅ | 295s |

**真陽性はどのセルも 0** — 速報スキャンの時点で全 22 件は、iOS パス構造で構造的に不可能な path traversal / IOKit のリリース忘れ誤読 / JSON DoS の上流キャップ済み などの既知の誤指摘パターンに帰着。指摘ひとつひとつの詳細な TP/FP 分類は次回に持ち越し。

## ハイライト #1: codestral の捏造が事実に化けた

ここが本記事のいちばんの収穫。

### 単発呼び出しで codestral が言ったこと

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
upper_bound_bytes: 65536 (maxPacketSize)  ← 数字は正解 (変数名は maxBufferSize が正)
```

ファイル名と行番号は依然ガタガタだが、 **数字 65536 は正解**。実コードに存在する値で、実際の上限を当てている。

つまり:
- 単発 → 自信満々に嘘
- aider あり → 細部はズレるが本質を当てる

**同じモデル / 同じプロンプト**で、aider の有無だけでこれが起きた。これが参照軸の効きの直接観察。

## ハイライト #2: qwen14b は単発でも答えられた

ここで予想外の結果。**qwen2.5-coder:14b の単発呼び出しは 65536 を当てた**。

```
upstream_capped: yes
evidence_file: MIDI2Kit/Sources/MIDI2Core/UMP/UMPSysEx8Assembler.swift
evidence_line: L53-L61
upper_bound_bytes: 65536
```

MIDI2Kit のソースを与えていないのに。「学習データに MIDI2Kit が含まれていた」「Swift の MIDI ライブラリの典型的なデフォルト値を推測した」のいずれかだろう。

つまり参照軸の効きは **モデル依存**:
- 弱いモデル (codestral) → aider で救われる
- 強いモデル (qwen14b, llama70b) → 単発でも答えられる

「aider を被せれば全部解決する」ではない。 *「捏造する傾向のあるモデルが aider 経由なら捏造しなくなる」* という防御的な効果。

## ハイライト #3: llama70b は別レイヤの上限を発見

llama3.3:70b は両方式とも `4104` と答えた。これは MIDI2Kit の 65536 とは違うが、**別レイヤで実在する有効な上限値**:

```swift
// DX7SysExParser.swift, L19-L30
private static let bulkDumpSize = 4104
...
public static func parse(data: Data, ...) -> DX7SysExBank? {
    guard data.count == bulkDumpSize else { return nil }
    ...
}
```

DX7 の 32-voice bulk dump は厳密に 4104 バイトで、これに合わない data は **パーサー自身が即 reject** する。つまり攻撃者が大きい SysEx を送っても、MIDI2Kit (64KB) をすり抜けたとしても、パーサーでさらに 4104 で弾かれる。

llama70b は **この内側の防御を発見した**。チェック質問が求めた「上流ライブラリでのキャップ」とは違うが、**本物の防御層を引用したという意味では誤りではない**。

「正解が一意でない」というのはチェック質問自体が内包する問題点で、改善の余地がある。

## ハイライト #4: aider は指摘を「引き出す」

指摘数を方式別に並べると:

| | aider あり | 単発 |
|---|---|---|
| codestral | 4 | 8 |
| qwen14b | **4** | **0** ← "no vulnerabilities" |
| llama70b | 5 | 1 |

**aider 方式は指摘数がほぼ一定 (4〜5)**、単発方式は **0 〜 8 と大きく振れる**。

特に qwen14b は単発だと「脆弱性なし」と完全に黙るが、aider を入れると 4 件出してきた。 **aider がモデルに「探せ」「JSON で答えろ」を繰り返しやらせることで、指摘を引き出している**形が見える。

これは真陽性率の改善とは別の効果。指摘が多いほど良いわけではない (qwen14b の 4 件もどうせ誤指摘) けど、「黙りっぱなしのモデルを動かす」効果は確実にある。

## ハイライト #5: 真陽性は依然 0

前回の累計 52 件・真陽性 0 件が、本実験で aider あり 13 件 + 単発 9 件 = 計 22 件追加されて、**累計 74 件 / 真陽性 0 件**。

aider で **参照軸 (上流ライブラリを実際に見る能力) は向上** したが、**知識軸 (Swift / DX7 ドメインの言語的な理解)** は別の問題で、aider だけでは真陽性には届かない。

主な誤指摘パターンも前回と同じ:
- iOS の `lastPathComponent` に `/` が含まれる前提の path traversal (iOS の構造上あり得ない)
- `FileManager.copyItem` の symlink 追従 (Apple はシンボリックリンクをリンクとしてコピーする)
- IOKit IOService のリリース忘れ指摘 (実コードは全分岐で release している)
- JSON DoS (上流 MIDI2Kit で 64 KB 制限済み)

→ **aider で「ある種の誤指摘は防げるが、別の種類の誤指摘には効かない」** という境界が本実験で観測された。

## 時間

| | aider あり | 単発 |
|---|---|---|
| codestral:22b | 179s | 306s |
| qwen14b | 430s | 95s |
| llama70b | 548s | 295s |

aider 方式は **単発より速いケースもある** (codestral)、**単発より遅いケースもある** (qwen14b)、傾向は一定しない。aider 内部でのリポジトリマップ構築・ファイル読み込み・LLM 呼び出しの回数にも依存。

ちなみに aider は `--no-stream` で動かし、`--read` で MIDI2Kit を渡しているが、本実験のセッションで明示的なファイル検索ツールの呼び出しはログに観測されなかった。aider は `--read` で同梱したファイルをモデルが必要に応じて読む形であり、能動的な grep ループは起きていなかった可能性が高い。

つまり今回観測した「参照軸の改善」は、**本格的な自律ループではなく `--read` で事前に渡したコンテキストの効果** かもしれない。次の実験では、より能動的なエージェント (Codex CLI / Claude Code を上位比較として) を試したい。

## まとめ

- **aider は参照軸を構造的に改善する**: codestral が単発で "1024" と捏造していた答えが、aider 経由で "65536" に化けた
- ただし **モデル依存**: qwen14b と llama70b は単発でも捏造せず、正解に近い答えを出した
- 副次効果: **黙りっぱなしのモデルを動かす** (qwen14b 0件 → 4件)
- **真陽性は依然 0** (累計 74 件、全誤指摘) — 知識軸の壁は aider では突破できない
- "夜中に走らせるレビューワー" としての実用性: **参照軸 ✓ / 知識軸 × → 全体評価は依然、商用最新モデル + aider が必要**

要するに、**aider は「捏造するモデル」を「実コードを引用するモデル」に変える** が、**真陽性を生み出す力は別問題**。前回記事の「ノイズは減らせるが、未発見のバグを掘り出す力にはならない」と同じパターンが、別の角度からも確認された形。

本記事の続編 (RAG / LoRA で知識軸を直接攻める) は別プロジェクトで設計完了済 (`swift-audit-rag`, `swift-audit-lora`)、進捗が出たらまた書きます。

---

- 母体記事: <https://zenn.dev/hakaru/articles/m2dx-local-llm-audit-zero-true-positives>
- **M2DX-Core (OSS)**: <https://github.com/hakaru/M2DX-Core>
- **M2DX (TestFlight)**: <https://testflight.apple.com/join/BAtGszPw>
