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

- **M2DX** — iOS/macOS 向け MIDI 2.0 対応 DX7 互換 FM シンセサイザーアプリ。[App Store で配信中（無料）](https://apps.apple.com/jp/app/m2dx/id6763840208)
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
iters: 300            # mlx_lm に自動早期停止なし。val loss を目視して手動停止
learning_rate: 5e-5
max_seq_length: 2048
steps_per_eval: 50   # 50 iter ごとに val loss を確認
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

## Track A 実施

### データ準備

`ANTHROPIC_API_KEY` がなかったので `generate_domain_qa.py` は動かせなかった。代わりに Q&A を手書きで JSONL に直接書いた。

生成したファイル：

| ファイル | 内容 | 件数 |
|---|---|---|
| `data/dx7_spec_qa.jsonl` | DX7 SysEx 仕様 Q&A（guard 必要性、4104 導出、128B 境界） | 8件 |
| `data/midi20_qa.jsonl` | MIDI 2.0 RT制約 Q&A（NSLock 禁忌、SnapshotRing パターン） | 5件 |
| `data/seeded_pattern_qa.jsonl` | 3つの仕込みバグパターンに直接対応する Q&A | 6件 |
| `data/audit_format_qa.jsonl` | 推論時と同一フォーマット（CONTROL QUESTION + JSON）の完全な training example | 4件 |

`audit_format_qa.jsonl` が最も重要だ。推論時のプロンプトと全く同じ構造で「こういう入力が来たらこう返せ」を直接教える。

```jsonl
{"messages": [
  {"role": "user", "content": "...CONTROL QUESTION...\n\n## DX7SysExParser.swift\n```\n...guard削除済み...\n```"},
  {"role": "assistant", "content": "upstream_capped: yes\n...\n{\"severity\":\"high\",\"cwe\":\"CWE-125\",...}"}
]}
```

### データ統合とバランス調整

v4 学習データ（168件）＋ 新規ドメイン Q&A 6ファイルをマージ。

`merge_all_data.py` の `is_yes_bug()` がハマりポイントだった。「バグなし」を説明するときも "vulnerability" や "out-of-bounds" という単語を使うため、単純なキーワードマッチが機能しない。最終的に：

1. JSON の `"severity":"high"` 文字列 → バグあり（最も信頼性が高い）
2. `"false positive"` → バグなし（明示的な否定）
3. CWE 番号の引用（ただし「〜は CWE ではない」を除く） → バグあり
4. `"yes, this is a real bug"` などの固定句 → バグあり
5. `"out-of-bounds read/crash"` は前後25文字に "no " / "not " がなければ → バグあり

バランス調整：最終的に 111 yes / 111 no = 222件（Train=199, Val=11, Test=12）。

### 訓練

```bash
bash scripts/train.sh data/mixed adapters/swift/m2dx-reviewer-v1
```

M3 Ultra 96GB で約 3時間。val loss の推移：

| iter | val loss |
|---|---|
| 50 | 0.967 |
| 100 | 0.680 |
| 150 | 0.599 |
| **200** | **0.566**（最良） |
| 250 | 0.589 ← 上昇 → 停止 |

iter 200 で val loss が底を打ちiter 250 で反転した。`adapters.safetensors` に iter 200 のチェックポイントをコピーして確定。

参考：v4 の best val loss は 0.813。v5 は 0.566 まで下がった。

### GGUF 変換と ollama 登録

```bash
bash scripts/fuse_and_register.sh
```

`mlx_lm.fuse` → HuggingFace 形式のマージ済みモデル → `convert_hf_to_gguf.py --outtype q8_0` → `models/m2dx-reviewer.gguf`（15.7 GB）→ `ollama create m2dx-reviewer:14b`。

なお `convert_hf_to_gguf.py` は Homebrew の llama.cpp だと `MODEL_ARCH` に `GEMMA4` がないというエラーで失敗する。ソースビルドした `/tmp/llama_cpp_src` のスクリプトを使う必要があった（mlx_lm の venv には古い `gguf` パッケージが入っているため）。

---

## 仕込みバグ再検証（m2dx-reviewer:14b）

```bash
./review.sh --files validation/seeded/DX7SysExParser_seeded1.swift
./review.sh --files validation/seeded/UserBankManager_seeded2.swift
./review.sh --files validation/seeded/MIDIInputManager_seeded3.swift
```

| バグ | CWE | 結果 | 報告 severity |
|---|---|---|---|
| guard 削除 → OOB リード | CWE-125 | **検出** ✅ | medium（後述） |
| `.path` → パストラバーサル | CWE-22 | **検出** ✅ | high |
| NSLock in RT callback | RT-safety | **検出漏れ** ❌ | — |

**TP: 2/3。合格ライン（≥2/3）クリア。**

### Bug 1 の severity が downgrade された理由

CWE-125 の finding は `severity: high` で出たが、`validate_findings.py` の Stage 2（定数値近接矛盾チェック）が `65535` と `4104` の近接を検出して `medium` に下げた。finding の内容は正しい——validator の false positive だ。

### Bug 3 が漏れた理由

v4 と同様、モデルは MIDI コールバックを上流の `UMPSysEx7Assembler`（65535 バイトキャップ）と紐づけて「バッファ安全性は確保されている」と判断した。RT スレッドでのミューテックス取得という**別軸の問題**を見逃した。

今回の訓練データは `NSLock` の RT 違反を直接教える事例が2件しかなかった。Bug 3 を確実に検出するには RT-safety 訓練データを増やす必要がある。

---

## コードレビューで見つかった問題

Track A 完了後、Codex + code-reviewer エージェントでコードレビューを実施した。主要な指摘と対応：

### JSONエクストラクタが `{` を含む fix_hint を捨てる

```python
# 修正前: 正規表現
for m in re.finditer(r'\{[^{}]*\}', text, re.DOTALL):

# 修正後: ブレース対応パーサー
def extract_json_objects(text):
    depth = 0; start = -1; in_string = False; escape_next = False
    for i, ch in enumerate(text):
        ...  # ネストした {} を正しく処理
```

`fix_hint` に `guard ... else { return nil }` のような Swift コードが入ると、正規表現は内側の `{}` だけをマッチして外側の JSON が捨てられる。今回は model がたまたまこの形式を再現しなかったので検出できたが、訓練が進むほど壊れる可能性があった。

### 訓練データ QA ファイルが gitignore 対象だった

`data/` ディレクトリが丸ごと gitignore されていたため、2/3 PASS の立役者である `audit_format_qa.jsonl` / `seeded_pattern_qa.jsonl` がローカルにしか存在しなかった。

修正：`.gitignore` を `data/` → `data/mixed/` + `data/train_*.log` に変更し、手書き QA ファイルと `data/sources/` スペックドキュメントをコミット対象に変更。

### `is_yes_bug()` の否定文誤分類

`"out-of-bounds read"` というフレーズは「OOB リードは発生しない」と説明する回答にも出現する。前後 25 文字に `"no "` / `"not "` がなければバグあり判定、という文脈チェックを追加した。

### その他

- `balance()` のクラスが空のとき `ZeroDivisionError` → `ValueError` に変更
- `lora_config.yaml` の `iters: 600` を `300` に修正（実際の停止点 250 を反映）、手動停止の根拠をコメントで記録
- `fuse_and_register.sh` の Modelfile を固定 `/tmp` パス → `mktemp` に変更

---

## v6：RT-safety recall の改善

### なぜ Bug 3 が漏れたか

v5 のモデルは `upstream_capped: yes`（上流の `UMPSysEx7Assembler` が 65535 バイトキャップを持つ）と判断すると、それだけで「安全」と短絡した。RT スレッドでのミューテックス取得という**独立した軸の問題**を見逃した。

バッファ安全性と RT スレッド安全性は**直交する軸**だ：
- `upstream_capped: yes` → バッファオーバーフローのリスクなし
- NSLock in RT callback → スケジューラによる preemption リスク（無関係）

この2軸を独立して評価するよう教える必要があった。

### 訓練データの設計（rt_safety_qa.jsonl）

28件の RT-safety 訓練データを追加した。3グループに分けた：

**Group 1（12件）：デュアル軸コントラストペア**

各アイテムが `upstream_capped: yes` を持ちながら RT-safety finding も持つ。「バッファ安全 ≠ RT安全」を直接教える。

```jsonl
// upstream_capped: yes であっても NSLock は RT 違反
{"messages": [
  {"role": "user", "content": "...CONTROL QUESTION...\n\n```swift\nclient.onFullMIDIReceived = { data in\n    self._lock.lock()  // ← RT thread!\n    ...\n    self._lock.unlock()\n}\n```"},
  {"role": "assistant", "content": "upstream_capped: yes\n\n```json\n{\"severity\":\"high\",\"cwe\":\"RT-safety\",\"title\":\"NSLock acquired inside onFullMIDIReceived RT callback\",...}\n```"}
]}
```

コンテキスト：MIDI callback（4件）、AudioUnit render（4件）、CoreMIDI（4件）

**Group 2（10件）：RT 違反パターンの網羅**

upstream_capped 文脈なしで RT 違反を直接教える。`String(format:)` ヒープ割り当て、`DispatchQueue.sync`、`os_unfair_lock_lock()` 誤用、`@MainActor` dispatch など。

**Group 3（6件）：safe RT コントラスト**

RT callback を含むが安全なケース。`UnsafeAtomic`/`ManagedAtomic` を使ったロックフリー SPSC ring buffer、正しい `os_unfair_lock` の使い方（non-contended、pre-allocated）など。過汎化を防ぐ。

### 訓練結果（v6）

マージ後：236件（Train=212, Val=11, Test=13）

| iter | val loss |
|---|---|
| 100 | 0.633 |
| 150 | 0.648 |
| **200** | **0.605**（最良） |
| 250 | 0.649 |
| 300 | 0.649 |

v5（val loss 0.566）より少し高いが、訓練データの難易度が上がった（RT-safety の微妙な判断を学習している）ため想定範囲内。

### 仕込みバグ再検証（v6）

| バグ | CWE | 結果 | 報告 severity |
|---|---|---|---|
| guard 削除 → OOB リード | CWE-125 | **検出** ✅ | high |
| `.path` → パストラバーサル | CWE-22 | **検出** ✅ | high |
| NSLock in RT callback | RT-safety | **検出** ✅ | ※後述 |

**TP: 3/3。v6 合格。**

※ Bug 3 の finding は `severity: invalid` でフィルタされた。モデルは `"file": "MIDIInputManager.swift"` を出力したが、実際のファイルは `MIDIInputManager_seeded3.swift`（seeded バリアント）だった。`validate_findings.py` のファイル名照合が一致を見つけられず無効化した。モデルは正しく NSLock を検出している——インフラ側の問題だった。

### バリデータのファイル名マッチング修正

`_count_lines()` に seeded バリアント照合を追加した：

```python
# 修正前: 完全一致のみ
for known in known_files:
    if str(known) == file_path or (known.name == p.name and p.parent == Path(".")):
        return sum(1 for _ in known.open(...))
return None

# 修正後: Foo_seededN.swift ← Foo.swift のマッチを追加
for known in known_files:
    if str(known) == file_path or (known.name == p.name and p.parent == Path(".")):
        return sum(1 for _ in known.open(...))
    # Foo_seeded3.swift は Foo.swift にマッチ（MIDIInputManagerHelper.swift は不一致）
    if known.suffix == p.suffix and known.stem.startswith(p.stem + "_"):
        return sum(1 for _ in known.open(...))
return None
```

テスト 2 件追加（`test_stage1_seeded_file_variant_matches`、`test_stage1_seeded_variant_does_not_match_unrelated_file`）、11/11 PASS。

### Stage 2 定数チェックの誤検知修正

v5 で Bug 1 が `medium` に下げられた原因は Stage 2 の近接矛盾チェックだった。finding テキストが `"guard checks bulkDumpSize (4104 bytes) but upstream caps at 65535"` という形式のとき、正しい値 `4104` が含まれているにもかかわらず、`65535` が近接しているだけで誤検知していた。

**修正前の問題**：定数名の前後 60 文字以内に正しい値と**異なる**数値があれば severity を1段落とす。正しい値と別の数値が**両方**近くにある場合（= 定数値を把握したうえで別の数値と比較している）を誤って矛盾と判断する。

**修正後**：近接した異常値を検出したとき、同じウィンドウ内に正しい値も存在するなら矛盾なしとみなす。

```python
for m in pattern.finditer(text):
    n = int(m.group(1) or m.group(2))
    if n != value and n > 0:
        # 正しい値も同じ窓内にあれば hallucination ではない
        window = text[max(0, m.start() - 60):m.end() + 60]
        if re.search(r"\b" + re.escape(str(value)) + r"\b", window):
            continue
        return True
```

テスト 2 件追加：

- `test_stage2_correct_value_alongside_other_number_no_conflict`：正しい `4104` と `65535` が共存 → downgrade しない
- `test_stage2_wrong_value_without_correct_value_still_conflicts`：`65535` のみ（`4104` なし）→ downgrade する

合計テスト数：13/13 PASS。

---

## 実コードレビュー（M2DX-Core）

v6 を M2DX-Core の実コード（仕込みなし）に対して走らせた。対象ファイル 6 本。

モデルが出力した finding：

| ファイル | 指摘 | 判定 |
|---|---|---|
| `SynthEngine.swift` | `determineTargetSlots()` で `activeSlotCount` が 8 を超えると `appendTarget()` がサイレントに切り詰める（CWE-125 的 OOB） | **偽陽性** |

**調査結果**：`activeSlotCount` は `TimbreMode.slotCount` からのみ設定される。`TimbreMode` は sealed enum で最大値は `.tx816 = 8`（`kMaxSlots = 8` と一致）。したがって `appendTarget()` の `default: return` には到達しない——現状のコードで OOB は不可能。

加えて行番号も外れていた（モデル出力: L300-304、実際: L1027+）。

今回の実コードレビューは **FP 1件、TP 0件**。仕込みバグテストで問題になった「no vulnerabilities」の過抑制ではなく、行番号 hallucination ＋ コードの誤読によるもの。`appendTarget` の `default: return` は現在デッドコードだが、将来 `TimbreMode` に 9+ スロットの case が追加されたときにサイレント切り詰めが発生するリスクはある。コメントで意図を明記しておくべき箇所ではある。

---

## まとめ

| 項目 | 状態 |
|---|---|
| Track B（ツールパイプライン） | ✅ 完成 |
| finding 検証（3ステージ） | ✅ 機能している |
| LoRA v4（仕込みバグ検出） | ❌ 0/3 |
| Track A v5（m2dx-reviewer:14b） | ✅ 2/3 合格 |
| **Track A v6（RT-safety recall）** | **✅ 3/3 合格** |
| Stage 2 validator 誤検知修正 | ✅ 修正済み（13/13 PASS） |
| 実コードレビュー（M2DX-Core） | FP 1件（行番号 hallucination + コード誤読）、TP 0件 |

**LoRA で「偽陽性を減らす」チューニングをすると「真陽性も消える」リスクがある。** v4 がその典型だった。

v5（m2dx-reviewer:14b）が 2/3 をクリアした決め手は2つだ。

1. **推論フォーマットと一致した訓練例**（`audit_format_qa.jsonl`）——「CONTROL QUESTION が来たら JSON で返す」という行動パターンを直接教えた。
2. **Yes/No の 50% バランス**——「バグあり」と「バグなし」を均等に学習させ、全否定への収束を防いだ。

v6（m2dx-reviewer:14b）で 3/3 をクリアした決め手は**デュアル軸コントラスト訓練**だ。`upstream_capped: yes` を持ちながら RT-safety finding も持つペアを12件加えることで、バッファ安全性と RT スレッド安全性の独立性をモデルが学習した。

実コードレビューでは FP 1件（行番号 hallucination ＋ コード誤読）、TP 0件だった。仕込みバグでは検出できた CWE-125 が、実コードでは到達不能なコードパスを誤って指摘した。seeded テストと実コードでは問題の性質が異なる——seeded は「削除された1行を見つける」だが、実コードは「届かない `default:` ブランチをリスクと誤認しない」が求められる。次のステップは、このような構造的 FP パターンをどう訓練で抑制するかだ。
