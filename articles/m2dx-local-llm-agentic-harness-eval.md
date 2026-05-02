---
title: "ローカルLLMって本当に開発に使える？（３）aiderを試してみる"
emoji: "🛠️"
type: "tech"
topics: ["llm", "ollama", "aider", "swift", "codereview"]
published: false
---

:::message
**この記事の対象プロジェクト**

- **M2DX** — iOS/macOS 向け MIDI 2.0 対応 DX7 互換 FM シンセサイザーアプリ。[TestFlight 公開ベータ](https://testflight.apple.com/join/BAtGszPw) で試せる
- **M2DX-Core** — M2DX の DX7 互換エンジン部分。Pure Swift、Apache 2.0 で OSS 公開: [github.com/hakaru/M2DX-Core](https://github.com/hakaru/M2DX-Core)
- **MIDI2Kit** — M2DX-Core が依存する Swift 製 MIDI 2.0 ライブラリ。SysEx の受信・バッファ管理・UMP デコードを担う。本記事では「上流ライブラリがどこまで守っているか」を検証する主役になる。開発の経緯は[こちらの本](https://zenn.dev/books/midi2kit-development-journey/)にまとめてある
:::

## 前回のあらすじ

ひと月前くらいに [ローカルLLMって本当に開発に使える？DX7エンジンの監査をやらせたら、全然だめでした。。](https://zenn.dev/hakaru/articles/m2dx-local-llm-audit-zero-true-positives) という記事を書いた。要点だけ:

- iOS/macOS向けのMIDI 2.0 FM音源アプリ **M2DX** ([TestFlight 公開ベータ](https://testflight.apple.com/join/BAtGszPw)) を開発中、心臓部は OSS の DX7 互換エンジン **M2DX-Core**
- 手元の Mac Studio M3 Ultra 96GB で ollama に **10 機種のローカル LLM** (qwen3:14b, gemma3:12b, codestral:22b, llama3.3:70b, mixtral:8x22b 等) を動かし、バグ監査 + セキュリティ監査をやらせた
- 結果: **累計 52 件の指摘 → 真陽性 0 件**

その記事の最後で、解決策として 4 象限マトリクスを描いた:

| | **単発呼び出し** | **+ aider などのツール** |
|---|---|---|
| **商用最新モデル** (GPT-5 / Claude Opus) | – | ◎ Claude Code / Codex CLI |
| **ローカル中型** | × ← 前回やったやつ | **○ ← 本記事で検証** |

「ローカル中型 LLM に grep / read ツールを持たせたら、単発呼び出しから **どこまで** 改善するのか?」  
これを実測してみたのが本記事。

:::message
本記事はシリーズ 4 本立ての 3 本目です。2 本目 (RAG) はすでに公開、4 本目 (LoRA) も公開済みです。執筆時点では RAG/LoRA が準備中だったため、先に公開しました。
:::

:::message
**aider とは**

aider は、ターミナルで動くコード編集アシスタント。リポジトリの構造マップと指定ファイルを LLM の会話文脈に組み込み、「確認する → 考える → 編集を提案する」のサイクルを繰り返させる。通常の API 呼び出しでは LLM はプロンプトに書いた内容しか見られないが、aider 経由だとリポジトリ構造や指定ファイルの内容を踏まえて回答できる。

こうした「LLM にリポジトリ文脈を持たせて反復的に動かす仕組み」全般を "agentic harness" と呼ぶ。aider のほか Codex CLI や Claude Code も同じ仕組み。本記事では aider 一本で試す。
:::

## 何を測りたいのか

前回観測した失敗パターンは大きく 2 種類に分けられる:

1. **知識軸** — Swift の言語仕様や DX7 ドメインの知識がモデルに入っているかどうか。`tuple ≠ Array`、`&+`/`&-` は意図的なラップ演算子、DX7 の固定小数点表現など。これはモデルの素の能力で、cheat sheet / RAG / LoRA で対処する別軸 (前回記事の続編 B / C 案)
2. **参照軸** — プロンプトに書かれていない外部コードを自分で取りに行けるかどうか。依存ライブラリを見ずに答える、存在しない行番号を引用するなど、「プロンプトの外を見られない」ことが根本原因

aider は **(2) を狙い撃ち** できる。上流コードを文脈として与えられるなら、「知らないなら捏造」を「実コードを引いて答える」に変えられるはず。

これを定量化するために、**チェック用質問** を仕込んだ:

> Q: `DX7SysExParser` に届く前に、SysEx のバイト数は上流ライブラリで制限されているか? されているなら **どのファイル / 何行目 / 上限値** を答えよ。

答えるには **MIDI2Kit のソースを実際に確認する** しかない。各モデルが (i) 正解する / (ii) 捏造する / (iii) "わからない" のどれを返すかが、参照軸が効いているかを見る判定指標になる。

## 正解

:::message
**[MIDI2Kit](https://zenn.dev/books/midi2kit-development-journey/) のアーキテクチャ上の位置づけ**

外部から届いた SysEx バイト列は、M2DX-Core のパーサーに直接渡されるわけではない。まず MIDI2Kit の `UMPSysEx7Assembler` / `UMPSysEx8Assembler` がバイト列を組み立て、`maxBufferSize` を超えた時点で弾く。M2DX-Core の `DX7SysExParser` に届くのはそこをくぐり抜けたものだけ。

「M2DX-Core のコードだけ見ても防御の全体像は分からない」というのが、単発呼び出しの LLM が依存ライブラリを参照できないと詰む理由。
:::

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
- `MIDIInputManager.swift` (1,132行) — MIDI 2.0 UMP（Universal MIDI Packet、MIDIの統一パケット形式）デコード、PE（Property Exchange、MIDI 2.0 の双方向プロパティ交換プロトコル）JSON
- `PEResponderHost.swift` (374行) — Property Exchange responder
- `USBResetHelper.c` (84行) — IOKit（Apple の I/O ドライバフレームワーク）USB reset

### 比較セル (3 モデル × 2 方式 = 6 セル)

| | **aider あり** | **単発呼び出し** (ollama API 直接) |
|---|---|---|
| **codestral:22b** | ✅ 走行 | ✅ 走行 |
| **qwen2.5-coder:14b** | ✅ 走行 | ✅ 走行 |
| **llama3.3:70b** | ✅ 走行 | ✅ 走行 |

aider は `--model ollama/<model>` で ollama に接続する。今回選んだモデルは前回実験 (qwen3:14b 等) とは別の組み合わせで、コーディング特化の qwen2.5-coder:14b と、前回から継続の codestral:22b・llama3.3:70b を採用した。MIDI2Kit の `UMPSysEx7Assembler.swift` と `UMPSysEx8Assembler.swift` の 2 ファイルを `--read` で読み取り専用の文脈として渡した。

両方式で **同じプロンプト** (Swift 6 の注意事項 + 脅威モデル + チェック質問 + 出力フォーマット) を使う。違うのは「上流ソースにアクセスできるか」だけ。

## 結果

| モデル | 方式 | 指摘数 | チェック質問の回答 | 正解? | 時間 |
|---|---|---|---|---|---|
| codestral:22b | aider あり | 4 | **65536** (ファイル名が微妙にずれ) | ✅ | 179s |
| codestral:22b | 単発 | 8 | **1024** | ❌ **捏造** | 306s |
| qwen2.5-coder:14b | aider あり | 0 ("no vuln") | **65535** (guard 境界の 1 ずれ) | ≈ | 430s |
| qwen2.5-coder:14b | 単発 | 0 ("no vuln") | **65536** | ✅ | 95s |
| llama3.3:70b | aider あり | 4 | **4104** (パーサー層の別上限) | ✅ 実在する値 | 548s |
| llama3.3:70b | 単発 | 1 | **4104** | ✅ | 295s |

**真陽性はどのセルも 0** — 速報スキャンの時点で全 17 件は、iOS パス構造で構造的に不可能な path traversal / IOKit のリリース忘れ誤読 / JSON DoS の上流キャップ済み などの既知の誤指摘パターンに帰着。指摘ひとつひとつの詳細な TP/FP 分類は次回に持ち越し。

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

一方 aider 方式では `65535` (1 ずれ) と回答し、しかも指摘件数はゼロだった。文脈を追加しても qwen2.5-coder は「脆弱性なし」という判断を変えなかった。

つまり参照軸の効きは **モデル依存**:
- 弱いモデル (codestral) → aider で救われる
- 強いモデル (qwen14b, llama70b) → 単発でも大きくは外さない

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

ただし aider 方式の transcript が引用した証跡は `MIDIInputManager.swift:1554` だったが、そのファイルは 1132 行しかなく、その行番号は実在しない。**値 4104 は正確、引用先の行番号は捏造** — codestral と同じ「部分的な捏造」が llama70b でも残っていた。

「正解が一意でない」というのはチェック質問自体が内包する問題点で、改善の余地がある。

## ハイライト #4: 指摘件数への影響はモデルによってまちまち

指摘数を方式別に並べると:

| | aider あり | 単発 |
|---|---|---|
| codestral | 4 | 8 |
| qwen14b | 0 | 0 |
| llama70b | 4 | 1 |

一貫した傾向は出ない:

- **codestral**: aider で減少 (8 → 4)。単発では同じ内容を繰り返していた部分が整理された可能性
- **qwen14b**: 両方式とも 0。aider を被せても「脆弱性なし」という判断を変えなかった
- **llama70b**: aider で増加 (1 → 4)。文脈情報が増えたことで追加の指摘を絞り出した

「aider を被せると指摘が増える」は成立しない。モデルの性格と与えられた文脈量の組み合わせで方向が変わる。どのセルも真陽性はゼロなので、件数の増減は精度の改善を意味しない。

## ハイライト #5: 真陽性は依然 0

前回の累計 52 件・真陽性 0 件が、本実験で aider あり 8 件 + 単発 9 件 = 計 17 件追加されて、**累計 69 件 / 真陽性 0 件**。

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

ちなみに今回の実験では、aider が能動的にファイルを検索するようなログは観測されなかった。`--read` で渡した 2 ファイルがそのまま会話文脈に入る形であり、「自律的に grep して取りに行く」ループとは異なる。

つまり今回観測した「参照軸の改善」は、**本格的な自律ループではなく `--read` で事前に渡した文脈の効果** かもしれない。次の実験では、より能動的なエージェント (Codex CLI / Claude Code を上位比較として) を試したい。

## まとめ

- **aider は参照軸を構造的に改善する**: codestral が単発で "1024" と捏造していた答えが、aider 経由で "65536" に化けた
- ただし **モデル依存**: qwen14b と llama70b は単発でも大きくは外さず、aider の恩恵は限定的だった
- **指摘件数への影響は一定でない**: モデルによって増えたり減ったり、変わらなかったりする
- **真陽性は依然 0** (累計 69 件、全誤指摘) — 知識軸の壁は aider では突破できない
- "夜中に走らせるレビューワー" としての実用性: **参照軸 ✓ / 知識軸 × → 全体評価は依然、商用最新モデル + aider が必要**

要するに、**aider は「捏造するモデル」を「実コードを引用するモデル」に変える** が、**真陽性を生み出す力は別問題**。前回記事の「ノイズは減らせるが、未発見のバグを掘り出す力にはならない」と同じパターンが、別の角度からも確認された形。

:::message
**シリーズの続き**

知識軸（Swift 言語仕様の誤読）を攻める 2 本はすでに公開済:

- [（２）RAG 編 — Swift 仕様をベクトル DB に入れたら誤検出が 76% 減った](https://zenn.dev/hakaru/articles/m2dx-local-llm-audit-rag)
- [（４）LoRA 編 — Swift 監査の誤検知を 93% 削減した話](https://zenn.dev/hakaru/articles/swift-audit-lora-fp-reduction)
:::

---

- 母体記事: <https://zenn.dev/hakaru/articles/m2dx-local-llm-audit-zero-true-positives>
- **M2DX-Core (OSS)**: <https://github.com/hakaru/M2DX-Core>
- **M2DX (TestFlight)**: <https://testflight.apple.com/join/BAtGszPw>
