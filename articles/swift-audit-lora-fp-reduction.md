---
title: "ローカルLLMって本当に開発に使える？（４）LoRA編 — Swift監査の誤検知を93%削減した話"
emoji: "🦾"
type: "tech"
topics: ["swift", "lora", "llm", "mlx", "machinelearning"]
published: false
---

v2（cheat sheet）、v3（RAG）と試してきて、どれも TP = 0 のままだった。

*じゃあ知識をモデル本体に焼き込んでしまえ* というのが LoRA（Low-Rank Adaptation = モデル全体を再学習せず、小さなアダプタを差し込む軽量ファインチューニング手法）を選んだ理由。

結果: 手作業 73 件のサンプルで誤検知 93% 削減。ただし「量より密度」という教訓と、まだ消えない FP（偽陽性）が残った。

## 背景：誤検知だらけの監査

M3 Ultra (96GB) + Ollama で10種類のモデルを使って Swift の音楽シンセサイザーエンジン（DX7エミュレーター）を監査した。結果は52件の「発見」（バグ監査36件＋セキュリティ監査16件）、真の陽性は0件。

失敗パターンは一貫していた：

| Swift イディオム | モデルの誤認 |
|----------------|------------|
| `&+` / `&-` | 「オーバーフローのバグ」 |
| `Int32(clamping:)` | 「精度損失のバグ」 |
| `nonisolated(unsafe)` | 「スレッド安全性の問題」 |
| Dictionary のオプショナル返却 | 「nullポインタの危険」 |
| Q24 固定小数点演算 | 「整数オーバーフロー」 |

モデルは Swift の意味論を知らない。`&+` が意図的なラッピング演算子であること、`Int32(clamping:)` が飽和演算子であること、DSP コードでは折り返し算術が正常であることを理解していない。

推論時にチートシートを注入する（v2）、RAGで Swift 仕様書を検索する（v3-RAG）といったアプローチも試したが、どちらも根本解決にならなかった。ならばモデルの重みに知識を埋め込もう、というのが LoRA を選んだ動機だ。

## 実験環境

- **ベースモデル**: Qwen2.5-Coder-14B-Instruct
- **ハードウェア**: Apple M3 Ultra 96GB
- **学習スタック**: mlx-lm 0.31.3（Apple Silicon ネイティブ）
- **推論**: llama.cpp（GGUF Q8_0、量子化済みモデル形式）+ Ollama
- **評価**: 同じ DX7 エンジンコードを監査し findings 数を計測

## データパイプライン

4種類のデータソースを用意した。

### 1. Swift 仕様書 Q&A（book_qa.jsonl）

`sources/swift-book/` と `sources/swift-evolution/proposals/`（Implemented/Accepted のみ）を Markdown チャンクに分割し、Claude API で Q&A ペアを生成。`&+`、`Int32(clamping:)`、Dictionary オプショナル、Q フォーマット演算など、モデルが誤解するトピックに焦点を当てた。

```
36 ペア
```

### 2. DPO ペア（dpo_pairs.jsonl）

実際の監査で出た誤検知19件を DPO（Direct Preference Optimization = 「正しい応答」と「誤った応答」のペアで選好を学習させる手法）ペアに変換：`prompt`（コードと質問）、`chosen`（正しい「バグなし」の説明）、`rejected`（モデルが実際に出した誤検知）。

:::message
mlx-lm 0.31.3 に `mlx_lm.dpo` は存在しない。SFT（Supervised Fine-Tuning = 正解例を直接与える教師あり微調整）形式（`chosen` をアシスタントターンとして使用）に変換して代替した。
:::

```
19 ペア（SFT 変換後）
```

### 3. 実バグサンプル（real_bugs.jsonl）

「バグなし」だけを学習させると「常にバグなし」と答えるモデルができあがる。これを避けるため、本物のバグ18件を手書きした：ヒープアロケーション（オーディオコールバック内）、`NSLock`（リアルタイムスレッド内）、範囲外アクセス、データレースなど。

```
18 ペア
```

### 4. ターゲット修正サンプル（clamping_fix.jsonl）

v4 評価後に残った唯一の FP（`Int32(clamping:)` の誤認）を潰すために後から追加。後述するが効果はなかった。

```
8 ペア
```

## 学習戦略

### SFT ハイパーパラメータ

```yaml
# scripts/lora_config.yaml
lora_parameters:
  rank: 16
  dropout: 0.05
  scale: 20.0  # alpha = rank × scale = 320
```

```bash
# v4 で実際に使ったパラメータ（scripts/train.sh のデフォルトとは異なる）
mlx_lm.lora \
  --model Qwen/Qwen2.5-Coder-14B-Instruct \
  --fine-tune-type lora \
  --num-layers 16 \
  --batch-size 2 \
  --iters 300 \
  --learning-rate 5e-5 \
  --max-seq-length 2048 \
  --config scripts/lora_config.yaml
```

### Yes/No バランス

「バグなし」サンプルが9割以上になると、モデルは常に「バグなし」と答えるようになる。`real_bugs.jsonl` の正例を3倍オーバーサンプリングして約50%に調整した。

## 結果

| バージョン | Findings | 明確な FP | データ構成 |
|----------|---------|---------|----------|
| baseline | ~41† | ~17 | raw qwen2.5-coder:14b |
| v2 | 5 | 3–4 | 手作業 DPO+Q&A |
| **v3** | **14** | **6+** | v2 + OSS コード 488件 |
| **v4** | **3** | **1** | 手作業のみ、yes 49% |
| v5 | 4 | 2 | v4 + clamping 修正 8件（効果なし）|

† baseline の ~41件は qwen2.5-coder:14b 単体での再計測値。親記事の「36件」は10モデル合計のバグ監査結果で、実験設定が異なる。

### v3 が v2 より悪くなった理由

最大の驚きは v3 の失敗だった。`extract_swift_patterns.py` で swift-foundation、swift-collections、AudioKit から抽出した488件の「バグなし」パターンを追加したところ、findings が 5→14 件に増加した。

原因の仮説：OSS コードは多様な Swift パターンを含むが、監査プロンプトに対する「正しい判断」のシグナルが薄い。量が多いほど手作業 DPO ペアの信号が希釈された。**73件の密なデータが488件の薄いデータに負けた。**

### 早期停止が重要

v4 の val loss（検証データ上の損失値。低いほど汎化性能が良い）推移：

```
Iter  50: Val 0.817  Train 1.049
Iter 100: Val 0.813  Train 0.212  ← ベスト
Iter 150: Val 0.855  Train 0.069  ← 過学習開始
Iter 200: Val 0.898  Train 0.035
Iter 300: Val 0.977  Train 0.022
```

Iter 100 以降は val loss が単調増加。87件の学習データ・バッチサイズ2で、100 iter は約2.3エポック（100 / (87/2)）に相当する。小データセットでは油断するとすぐ過学習する。

### GGUF 変換の落とし穴

`mlx_lm.fuse --export-gguf` は qwen2 アーキテクチャをサポートしていない。回避策：

```bash
# 1. アダプターをマージ（GGUF なし）
mlx_lm.fuse --model Qwen/Qwen2.5-Coder-14B-Instruct \
  --adapter-path adapters/qwen-swift-v4 \
  --save-path models/qwen-swift-v4-merged

# 2. llama.cpp で GGUF 変換
python /path/to/llama.cpp/convert_hf_to_gguf.py \
  models/qwen-swift-v4-merged \
  --outfile models/qwen-swift-coder-14b.gguf \
  --outtype q8_0

# 3. Ollama 登録
ollama create qwen-swift-coder:14b -f Modelfile
```

PyPI の `gguf` パッケージ（0.18.0）は最近のアーキテクチャ定数が不足している。llama.cpp リポジトリ内の `gguf` を `pip install -e "gguf[nexus]"` でインストールすること。

## 残った誤検知の分析

v4 の3件の findings：

1. **`Int32(clamping:)` の誤認** — 「精度損失のバグ」と報告。飽和演算子であることを最後まで学習しきれなかった
2. **EG ステージ advance のオフバイワン** — DX7 固有のエンベロープ動作。ドメイン知識なしには真偽を判断できない
3. **fbBuf のレースコンディション** — オーディオスレッドとメインスレッドからの非同期アクセス。要実コード検証

### v5 で学んだこと：ベースモデルの strong prior

v5 では `Int32(clamping:)` に特化した8サンプルを追加したが、FP は消えなかった。モデルの出力は変わった — 「精度損失」から「wrap する」へ — が誤りの本質は同じだった。

これはベースモデル（Qwen2.5-Coder）が事前学習で「clamping = 値を切り捨てて精度を失う操作」という強い prior（事前確率 = 事前学習で形成された確信）を形成していることを示す。73件の SFT では上書きできない。本格的に直すには：

- DPO（`rejected` に誤った説明を含めた選好学習）
- もしくは数百件規模のサンプル

## 教訓

**ベースモデルの strong prior は SFT 少数サンプルでは覆せない。** 8サンプル追加しても `Int32(clamping:)` の誤解は消えなかった。ベースモデル（Qwen2.5-Coder）が事前学習で「clamping = 値を切り捨てて精度を失う操作」という強い prior（事前確率 = 事前学習で形成された確信）を形成していることを示す。73件の SFT では上書きできない。本格的に直すには DPO（`rejected` に誤った説明を含めた選好学習）か数百件規模のサンプルが必要。

**データの密度 > 量。** 73件の手作業サンプルが488件のOSSコードを圧倒した。ターゲットドメインに直結しないデータはシグナルを希釈する。

**Yes/No バランスは明示的に管理する。** 「バグなし」だけを学習させるとモデルは常に「バグなし」と答える。真のバグサンプルを意図的に追加し、50%前後を維持する。

**mlx-lm での DPO の代替。** mlx-lm 0.31.3 は DPO をサポートしない。`chosen` をアシスタントターンとした SFT で代替できる。効果は限定的だが実装コストは低い。

**早期停止を怠らない。** 小データセット（< 100件）ではわずか100イテレーションで過学習する。val loss の監視と最良チェックポイントの保存は必須。

## まとめ

手作業73件の SFTサンプル（Q&A 36件 + DPO由来 19件 + 実バグ 18件）+ 早期停止 + yes/no バランス管理で、Swift コード監査の誤検知を 41件 → 3件（93%削減）に抑えた。大量の汎用データより、問題に直結した少数の高品質データが効く、という教訓を改めて確認した実験だった。

`Int32(clamping:)` の strong prior 問題はまだ残っており、DPO による続報を書く予定。
