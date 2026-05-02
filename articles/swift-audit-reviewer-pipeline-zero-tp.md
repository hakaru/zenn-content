---
title: "ローカルLLMって本当に開発に使える？（５）パイプライン完成 → 仕込みバグ 0/3 の衝撃"
emoji: "🧱"
type: "tech"
topics: ["llm", "ollama", "swift", "lora", "security"]
published: false
---

https://zenn.dev/hakaru/articles/swift-audit-lora-fp-reduction

:::message
**この記事の対象プロジェクト**

- **M2DX** — iOS/macOS 向け MIDI 2.0 対応 DX7 互換 FM シンセサイザーアプリ。[TestFlight 公開ベータ](https://testflight.apple.com/join/BAtGszPw) で試せる
- **M2DX-Core** — M2DX の DX7 互換エンジン部分。Pure Swift、Apache 2.0 で OSS 公開: [github.com/hakaru/M2DX-Core](https://github.com/hakaru/M2DX-Core)
- **MIDI2Kit** — M2DX-Core が依存する Swift 製 MIDI 2.0 ライブラリ。SysEx の受信・バッファ管理・UMP デコードを担う。開発の経緯は[こちらの本](https://zenn.dev/books/midi2kit-development-journey/)にまとめてある
:::

前回（4）は LoRA v4 で誤検知を 93% 削減した話だった。**「バグなし」を正しく「バグなし」と言えるようになった。**

今回はその続き——**本物のバグを見つけられるかどうか**を確かめた。

結果は 0/3。パイプラインは完成した。モデルは全然ダメだった。

---

## 前回までのあらすじ

M2DX-Core（DX7 FMシンセ、Pure Swift）のコードレビューをローカル LLM にやらせようとしている。

| 回 | アプローチ | 結果 |
|---|---|---|
| 1 | ゼロショット（10モデル） | TP 0/52件 |
| 2 | RAG（Swift仕様書） | FP 76%削減 |
| 3 | aider（エージェント型） | 多段推論は改善するが遅い |
| 4 | LoRA v4（手作業73件） | **FP 93%削減、残FP 3件** |

v4 の課題は「まだ `Int32(clamping:)` を誤検知する」だった。しかし評価に使っていたのは**元のコードをそのまま読ませたときの偽陽性数**だ。「バグを仕込んだコードを読ませたとき真陽性を出せるか」は確かめていなかった。

---

## 仕込みバグテスト

受け入れ基準を定量化するために「仕込みバグ」テストを設計した。M2DX-Core の実コードに意図的に3本のバグを埋め込み、モデルが検出できるか確認する。

| # | カテゴリ | 仕込み方法 | 期待する検出 |
|---|---|---|---|
| 1 | CWE-125 OOBリード | `parse(data:)` の先頭 guard を1行削除 | バッファ境界チェックなしの `subdata(in:)` アクセスを指摘 |
| 2 | CWE-22 パストラバーサル | `sourceURL.lastPathComponent` → `sourceURL.path` | フルパスをファイル書き込みに使うことを指摘 |
| 3 | RTスレッド安全性違反 | MIDIコールバック内に `NSLock().lock()/unlock()` を追加 | リアルタイムスレッドでのロック取得を指摘 |

これらはどれも Swift 意味論とは無関係だ。`&+` でも `Int32(clamping:)` でもない。削除した guard は1行、変えた関数名は1つ、追加したコードは2行——どれも人間が見れば一瞬で気づく類のバグだ。

### Bug 1：guard 削除

```swift
// 正常なコード
func parse(data: Data) -> [DX7Voice]? {
    guard data.count == bulkDumpSize else { return nil }  // ← これを削除
    for rawVoiceIdx in 0..<voicesPerBank {
        let voiceOffset = rawVoiceIdx * packedVoiceSize + 8
        let voiceData = data.subdata(in: voiceOffset..<(voiceOffset + packedVoiceSize))
        // ...
    }
}
```

guard がなければ `data` が4104バイト未満でも `subdata` が走る。典型的な CWE-125。

### Bug 2：パストラバーサル

```swift
// 正常なコード
let destURL = userBanksDirectory.appendingPathComponent(sourceURL.lastPathComponent)

// 仕込みバグ
let destURL = userBanksDirectory.appendingPathComponent(sourceURL.path)
```

`sourceURL.path` は `/../../etc/shadow` のようなパスをそのまま通す。`lastPathComponent` はファイル名部分だけを取り出すので安全。

### Bug 3：NSLock in RT callback

```swift
func onFullMIDIReceived(_ data: [UInt8]) {
    let lock = NSLock()   // ← 追加（RTスレッドでの確保 + ロック）
    lock.lock()
    // ... パース処理 ...
    lock.unlock()
}
```

MIDI受信コールバックはリアルタイムスレッドで動く。`NSLock` はミューテックスを取得するためブロックする可能性があり RT スレッドでは禁忌だ。

---

## パイプライン：swift-audit-reviewer

モデルを評価する前に、まずパイプライン全体を組み直した。前回までは「プロンプトを手で作ってモデルに渡す」スクリプトだった。今回は本番運用を想定した構成にした。

```
review.sh
  └── collect_files.sh      差分／指定ファイルの収集
  └── build_prompt.py       3層コンフィグからプロンプトを組み立て
  └── ollama (14B)           推論
  └── JSON 抽出             モデル出力から finding を切り出す
  └── validate_findings.py  3ステージ検証
```

使い方は：

```bash
./review.sh --diff                         # git staged files
./review.sh --files DX7SysExParser.swift
```

### プロンプト組み立ての3層コンフィグ

プロンプトに注入する知識を3種類に分けて管理する。

| 層 | ファイル | 内容 |
|---|---|---|
| Language | `config/languages/swift.yaml` | RAG インデックス（Swift ブック、Evolution）、モデルタグ |
| Domain | `config/domains/dx7-midi20.yaml` | ドメイン RAG、既知定数（`bulkDumpSize: 4104` など） |
| Project | `config/projects/m2dx.md` | プロンプトに直接埋め込むチートシート |

文字予算はコード 64k + 言語RAG 16k + ドメインRAG 16k + チートシート 4k。セクション順序はこの通りに積む：

```
Swift 言語リファレンス（RAG）
  ↓
ドメインリファレンス（DX7 / MIDI 2.0 RAG）
  ↓
プロジェクトチートシート（m2dx.md）
  ↓
ソースコード
  ↓
タスク指示（audit_task.md）
```

RAG は llama-index + `bge-large`。チャンク上限 400 文字（bge-large の 512 トークン制限に合わせた）。

### finding 検証の3ステージ

モデルの出力をそのまま信用するのは危険だ。`validate_findings.py` で3段階のフィルタをかける。

**Stage 1：行番号範囲チェック**

「DX7SysExParser.swift の L1554 に OOB リード」という finding が来たとき、実際のファイルが 200 行しかなければ架空の finding だ。範囲外はモデルに1回だけ問い直す（reflection）。

**Stage 2：定数値の近接矛盾チェック**

`bulkDumpSize が 4096` という finding が来たとき、本当の値は `4104` なので偽陽性の可能性が高い。finding テキスト中で定数名の前後60文字以内に矛盾する数値があれば severity を1段階下げる。

前後60文字に限定しているのは、`CWE-125` の `125` のような無関係な数値を誤検知しないためだ。

**Stage 3：reflection ラウンド**

Stage 1 で引っかかった finding を1回だけモデルに戻す。「このファイルは最大 N 行です。正しい行番号に修正してください」と伝え、修正後に再度 Stage 1 を通す。

---

## 結果：0/3

パイプラインが完成したので仕込みバグテストを走らせた。

| バグ | CWE | 結果 |
|---|---|---|
| OOB リード（guard 削除） | CWE-125 | **MISSED** — 出力「no vulnerabilities」 |
| パストラバーサル（`.path`） | CWE-22 | **MISSED** — 出力「no vulnerabilities」 |
| NSLock in RT callback | RT-safety | **MISSED** — 出力「no vulnerabilities」 |

**TP: 0/3。合格ライン（≥2/3）に遠く届かなかった。**

モデルは3本全てに「no vulnerabilities」を返した。

---

## なぜ 0/3 になったか

v4 は**Swift 意味論的偽陽性の抑制に特化して学習**されていた。

- `&+` を「バグではない」と学習
- `Int32(clamping:)` を「安全な飽和キャスト」と学習
- `bulkDumpSize` 絡みの誤検知パターンを「偽陽性」と学習

この方向性は正しい。ただし LoRA が「バグは報告しない」という**全体的な抑制**に過汎化した。

仕込んだ3本は v4 が学習した「抑制パターン」とは無関係だ。`lastPathComponent` → `.path` はパストラバーサルの典型例で Swift 意味論とは何も関係ない。NSLock の RT 違反も同様だ。にもかかわらずモデルは黙った。

**LoRA が「否定パターン」だけで学習されると、全否定に収束しやすい。** 特に少数サンプル（今回は 73 件）では顕著だ。「これは偽陽性だ」という教師データを積み重ねると、モデルは「何かを指摘すること = 誤り」という方向に引っ張られる。

前回の教訓「量より密度」の裏面だった。密なデータで強くシグナルを入れたからこそ、そのシグナルの方向性がそのまま過汎化した。

### ベースモデルの能力は壊れていないはず

guard を1行削除すれば OOB リードになることは、コードを読める LLM であれば気づけるはずだ。Qwen2.5-Coder-14B-Instruct のベースモデルはこれを指摘できる。

問題は LoRA アダプタがベースモデルの「報告する」という行動を上書きしたことだ。削除したものを「何も削除していない」に見せることはできない——しかしモデルは黙った。

---

## Track A：モデルを作り直す

v4 の延長線上（v5, v6...）で改善するのは難しい。アダプタの方向性が根本的にずれている。

新しい方針は**ドメイン知識を中心に据えたmix-training**だ。ターゲットモデルは `m2dx-reviewer:14b`（仮称）。

### 何を変えるか

v4 は「偽陽性を出さないこと」に最適化した。次は「バグを検出すること」と「偽陽性を出さないこと」の両方を、ドメイン知識ベースの事例で均等に教える。

**新規データ：DX7 / MIDI 2.0 ドメイン Q&A**

仕様書から Q&A ペアを Claude API で生成する。例：

> Q: `bulkDumpSize` が 4104 である理由は？
> A: DX7 bulk dump は 32 voice × 128B = 4096B に 8B の SysEx ヘッダーを加えて 4104B。この値は仕様から数学的に導出される定数であり、`parse(data:)` の guard チェックは正しい。

DX7 の定数導出、`UMPSysEx7Assembler` の上流キャップ、RT スレッド制約、Swift の安全な演算子——これを「なぜ安全か」「なぜ危険か」の両面で学習させる。

**Yes/No バランスを明示的に 50% に揃える**

`merge_all_data.py` で既存サンプルと新規ドメイン Q&A を統合し、「バグあり」「バグなし」の比率を約 50% に調整する。

**ベースモデルから再学習**

v4 アダプタを起点にしない。ベースモデル（Qwen2.5-Coder-14B-Instruct）から毎回再学習する。

### 学習パラメータ

```yaml
# scripts/lora_config.yaml
model: Qwen/Qwen2.5-Coder-14B-Instruct
fine_tune_type: lora
num_layers: 16

lora_parameters:
  rank: 16
  dropout: 0.05
  scale: 20.0   # alpha = rank × scale = 320

batch_size: 2
iters: 600
learning_rate: 5e-5
max_seq_length: 2048
steps_per_eval: 50   # 50 iter ごとに val loss を確認、early stop
```

早期停止は前回の教訓通り。小データセットでは 100 iter 前後で過学習が始まる。

### GGUF 変換の手順（おさらい）

前回も書いたが `mlx_lm.fuse --export-gguf` は Qwen2 に対応していない。2ステップが必要：

```bash
# 1. HuggingFace 形式でアダプタをマージ
mlx_lm.fuse \
  --model Qwen/Qwen2.5-Coder-14B-Instruct \
  --adapter-path adapters/swift/m2dx-reviewer-v1 \
  --save-path models/m2dx-reviewer-merged

# 2. llama.cpp で GGUF 変換
python convert_hf_to_gguf.py \
  models/m2dx-reviewer-merged \
  --outfile models/m2dx-reviewer.gguf \
  --outtype q8_0

# 3. ollama に登録
ollama create m2dx-reviewer:14b -f Modelfile
```

### 合格ライン

仕込みバグ 3 本中 **≥ 2/3 を検出** すること。

TP 2/3 で合格としたのは「完璧な検出器」ではなく「使える検出器」を目指しているからだ。実コードレビューでは FP と FN のトレードオフがある。今の v4 は FP ≈ 0、FN = 100%——使えない。目標は FP を低く保ちつつ FN を下げること。

---

## 実装で詰まったところ（パイプライン編）

パイプラインを作る過程でいくつかハマった点を記録しておく。

### RAG の `top_k` がサイレントに無視される

llama-index で `as_retriever()` を引数なしで呼ぶとデフォルトの `top_k=2` が使われる。`config/languages/swift.yaml` に `top_k: 4` を書いていても静かに無視されていた。

```python
# 修正前
return load_index_from_storage(ctx).as_retriever()

# 修正後
return load_index_from_storage(ctx).as_retriever(similarity_top_k=top_k)
```

テストを書くまで気づかない系のバグ。

### 正規表現のバックスラッシュが消える

`&+` などのパターンを RAG クエリ文字列に変換する処理で `str.replace("\\", "")` を書いていた。これで `r"Int\d+\(clamping:"` が `"Intd+(clamping:"` になっていた。

修正はパターンと人間可読クエリの辞書に切り替えること：

```python
PATTERN_QUERIES = {
    r"&[+\-*]":           "Swift overflow operators &+ &- &*",
    r"Int\d+\(clamping:": "Int clamping initializer",
    ...
}
```

### `trap EXIT` は最後に書いたものだけ有効

bash では `trap` を2回書くと2番目が上書きする。

```bash
# バグ：PROMPT_FILE が消えない
trap 'rm -f "$PROMPT_FILE"' EXIT
FINDINGS_FILE=$(mktemp)
trap 'rm -f "$FINDINGS_FILE"' EXIT   # ← 最初の trap が消える

# 修正：最初から両方まとめて
PROMPT_FILE=$(mktemp)
FINDINGS_FILE=$(mktemp)
trap 'rm -f "$PROMPT_FILE" "$FINDINGS_FILE"' EXIT
```

### macOS bash 3.2 の `mapfile` 問題

macOS のデフォルト bash は 3.2 で `mapfile` が使えない。改行区切りの文字列を配列にするには：

```bash
IFS=$'\n' read -r -d '' -a FILES_ARR <<< "$FILES" || true
```

末尾の `|| true` は空入力時の終了コード 1 を吸収するために必要。

---

## まとめ

| 項目 | 状態 |
|---|---|
| Track B（ツールパイプライン） | ✅ 完成。`./review.sh --diff` で動く |
| finding 検証（3ステージ） | ✅ 機能している |
| LoRA v4 の仕込みバグ検出 | ❌ 0/3（合格ライン ≥2/3） |
| Track A（m2dx-reviewer:14b） | 🔄 進行中 |

**LoRA で「偽陽性を減らす」チューニングをすると「真陽性も消える」リスクがある。** 今回の v4 がその典型だった。

対処は偽陽性抑制と真陽性検出の**両方の事例をドメイン知識ベースで均等に学習させること**——それが Track A の本質だ。

次回は Track A 完了後の仕込みバグ再検証結果を書く予定。
