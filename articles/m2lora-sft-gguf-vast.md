---
title: "ローカルLLMって本当に開発に使える？（６）SFTしてGGUFに変換してOllamaで動かすまで：7つの罠"
emoji: "🏋️"
type: "tech"
topics: ["llm", "lora", "ollama", "vastai", "python"]
published: false
---

:::message
**この記事の対象プロジェクト**

- **M2DX** — iOS/macOS 向け MIDI 2.0 対応 DX7 互換 FM シンセサイザーアプリ。[TestFlight 公開ベータ](https://testflight.apple.com/join/BAtGszPw) で試せる
- **M2LoRA** — M2DX の git commit を自動でレビューさせ、LoRA 学習データを貯めるパイプライン
:::

## 前回までのあらすじ

このシリーズでは M3 Ultra + Ollama で自分の Swift/MIDI プロジェクトのコードレビューを学習した LoRA モデルを育てている。

前回（5）でパイプラインの全体像を作り、Stage 1 Domain CPT（仕様書を読ませる事前学習）まで完走した。

- v1: vast.ai H100 / 35分 / **$1.70** / train_loss 0.46（3266 chunks）
- v2: vast.ai H100 / 93分 / **$2.71** / train_loss 0.50（8782 chunks）
- Gemini 評価改善（v1）: 高評価率 13.7% → 40.8%

今回は **Stage 2 SFT（Supervised Fine-Tuning）**。実際のコードレビューを alpaca 形式で学習させ、GGUF に変換して Ollama に登録する。

---

## Stage 2 SFT の概要

Stage 1 でコードの文体・語彙を学習したモデルに、今度は「どう指摘するか」のスキルを上乗せする。

```
input:  code_diff  →  alpaca "input"
output: synthesized_review  →  alpaca "output"
instruction: "以下のSwiftコードの差分をレビューしてください。"
```

学習データは M2DX の 174 件のレビューから avg スコア ≥5.0 でフィルタした **38 件**（Codex が 93% NULL だったため claude+gemini の 2-evaluator mean を採用）。

---

## VAST.ai のセットアップ

70B モデルの SFT には A100 SXM4 80GB を使用（$1.175/h）。

```bash
vastai create instance <id> \
  --image pytorch/pytorch:2.4.0-cuda12.4-cudnn9-devel \
  --disk 300 \   # ← GGUF変換に必須（後述）
  --ssh
```

### 罠 1: torch バージョン地獄

`pip install unsloth` だけで **torch 2.11 + cu13** に差し替えられ A100 と非互換になる。

```bash
# cu13 系を全削除してから cu124 で固定
pip uninstall -y torch torchvision torchao unsloth unsloth-zoo \
  $(pip list | grep "nvidia.*cu13" | awk '{print $1}')

pip install torch==2.5.1 torchvision==0.20.1 \
  --index-url https://download.pytorch.org/whl/cu124
pip install xformers==0.0.28.post3
pip install --no-deps unsloth unsloth_zoo
pip install trl==0.24.0 transformers==5.3.0 datasets==4.3.0
```

### 罠 2: trl 0.24.0 で削除された API

`DataCollatorForCompletionOnlyLM` は trl 0.24.0 で削除済み。`SFTTrainer` の `dataset_text_field` で代替。

### 罠 3: bf16 のみ（fp16 不可）

```python
# A100 では bf16 のモデルに fp16=True を渡すとエラー
TrainingArguments(bf16=True, ...)  # ← これだけ
```

### 罠 4: stdin ブロック

Unsloth が起動時に `apt-get install libssl-dev` の確認を求める。

```bash
python train_sft.py </dev/null > train.log 2>&1
```

---

## `train_sft.py`

```python
import unsloth  # 必ず最初に
from unsloth import FastLanguageModel
from trl import SFTTrainer
from transformers import TrainingArguments

model, tokenizer = FastLanguageModel.from_pretrained(
    model_name="lora-domain-v2",   # Stage 1 v2 adapter
    max_seq_length=2048,
    load_in_4bit=True,
)
model = FastLanguageModel.get_peft_model(
    model, r=16, lora_alpha=16,
    target_modules=["q_proj","k_proj","v_proj","up_proj","down_proj"],
    use_gradient_checkpointing="unsloth",
)

trainer = SFTTrainer(
    model=model, tokenizer=tokenizer,
    train_dataset=dataset,
    dataset_text_field="text",
    max_seq_length=2048,
    args=TrainingArguments(
        num_train_epochs=8,
        per_device_train_batch_size=1,
        gradient_accumulation_steps=4,
        bf16=True,
        learning_rate=2e-5,
        lr_scheduler_type="cosine",
        optim="adamw_8bit",
    ),
)
trainer.train()
model.save_pretrained("lora-sft-v3")
```

### 結果

```
データ:   38件 / 8 epochs / 244秒（4分）
train_loss: 1.585
コスト:    ~$0.08
```

---

## GGUF 変換：本番の地雷原

Ollama に登録するには GGUF が必要。ここで連続して踏んだ。

### 罠 5: llama.cpp の BitsAndBytes 非対応

```
NotImplementedError: Quant method is not yet supported: 'bitsandbytes'
```

`convert_lora_to_gguf.py` は bnb-4bit ベースを dequant できない。  
**→ Unsloth の `save_pretrained_gguf` を使う**（fp16 シャードを自動 DL してマージ）。

### disk 容量の計算

| フェーズ | 一時使用 |
|---|---|
| bnb-4bit キャッシュ | 37GB |
| fp16 シャード 30本 | 135GB |
| BF16 GGUF（split 後） | 116GB |
| Q4_K_M 出力 | 40GB |
| **ピーク** | **~285GB** |

200GB では足りない。**300GB disk が必須**。

### 罠 6: BF16 GGUF が 3 分割される

`save_pretrained_gguf` 内部で `--split-max-size 50G` が渡され GGUF が 47GB × 2 + 22GB に分割される。`llama-quantize` に直接渡すと別シャードの tensor を読めずエラー。

```bash
# ① まずマージ
llama-gguf-split --merge \
  Llama-3.3-70B-Instruct.BF16-00001-of-00003.gguf \
  Llama-3.3-70B-Instruct.BF16-merged.gguf

# ② 不要ファイルを削除して空きを作る
rm lora-sft-v3-gguf/model-*.safetensors    # 135GB 解放
rm Llama-3.3-70B-Instruct.BF16-0000{1,2,3}-of-00003.gguf  # 116GB 解放

# ③ Q4_K_M 量子化（~15分）
llama-quantize \
  Llama-3.3-70B-Instruct.BF16-merged.gguf \
  lora-sft-v3-gguf/m2lora-sft-v3-q4_k_m.gguf Q4_K_M
```

```
model size  = 134573 MiB (16.00 BPW)
quant size  =  40543 MiB (4.82 BPW)
```

---

## Ollama 登録

40GB GGUF をローカルに落とす。`scp` は途中切断に弱いので `rsync --partial` を使う。

```bash
rsync -av --partial --progress \
  -e "ssh -i ~/.ssh/vastai_key -p 26036 \
      -o ServerAliveInterval=30 -o ServerAliveCountMax=20" \
  root@ssh7.vast.ai:/root/lora-sft-v3-gguf/m2lora-sft-v3-q4_k_m.gguf \
  lora-models/v3-sft-smoke/
```

:::message alert
**FROM vs ADAPTER の違い（重要）**

`save_pretrained_gguf` が生成するのはフルマージ済みモデル（standalone GGUF）。  
Ollama の `ADAPTER` ディレクティブは LoRA delta GGUF 専用で、merged model を渡すと推論が正常に動かない。
:::

```
# NG: merged full model に ADAPTER を使う
FROM llama3.3:70b
ADAPTER /path/to/m2lora-sft-v3-q4_k_m.gguf   ← 出力がゴミになる

# OK: FROM で直接指定
FROM /path/to/m2lora-sft-v3-q4_k_m.gguf
PARAMETER num_ctx 8192
PARAMETER temperature 0.3
```

---

## 致命的な罠：Adapter Stacking ミス

登録後に `/api/chat` でテストすると完全に壊れた出力。

```
spleivil chain tied half695 Highbool Register...
```

`adapter_config.json` を確認すると:

```json
{
  "base_model_name_or_path": "unsloth/llama-3.3-70b-instruct-bnb-4bit"
}
```

v2（domain-CPT adapter）なしの base を指している。

**問題の構造:**

```
[正しいはずの変換]
base + v2-domain-CPT + v3-SFT → merge all → GGUF

[実際に行われた変換]
base + v3-SFT のみ → GGUF    ← v2 が欠落
```

SFT は「base + v2 の合成モデル」の上で訓練されたが、PEFT は `base_model_name_or_path` に元の HF モデル名しか記録しない。GGUF 変換スクリプトが v2 を知らずに v3 adapter だけをマージした結果、活性化パターンがずれて出力が崩壊。

**次回の正しい手順:**

```python
from unsloth import FastLanguageModel
from peft import PeftModel

# 1. base をロード
model, tokenizer = FastLanguageModel.from_pretrained(
    "unsloth/llama-3.3-70b-instruct-bnb-4bit", load_in_4bit=True
)

# 2. v2 adapter を適用してマージ（★ここが欠けていた）
model = PeftModel.from_pretrained(model, "/root/lora-domain-v2")
model = model.merge_and_unload()

# 3. v3 SFT adapter を適用してマージ
model = PeftModel.from_pretrained(model, "/root/lora-sft-v3")
model = model.merge_and_unload()

# 4. GGUF 変換
model.save_pretrained_gguf("/root/out", tokenizer, quantization_method="q4_k_m")
```

---

## コストまとめ

| ステップ | 時間 | コスト |
|---|---|---|
| SFT 学習（38件, 8epoch）| 4分 | $0.08 |
| GGUF 変換（300GB A800）| ~90分 | $1.32 |
| ダウンロード（40GB）| ~45分 | $0 |
| **合計** | | **~$1.40** |

---

## 教訓まとめ

1. **torch 固定**: `pip install unsloth` だけで cu13 に差し替えられる。cu124 を手動で固定
2. **trl 0.24.0**: `DataCollatorForCompletionOnlyLM` 削除済み。`dataset_text_field` で代替
3. **A100 は bf16 のみ**: `TrainingArguments(bf16=True)`。fp16 は型ミスマッチエラー
4. **stdin ブロック**: Unsloth の apt 確認プロンプトは `</dev/null` でスキップ
5. **disk 300GB 必須**: 70B GGUF 変換のピーク使用量 ~285GB。200GB では失敗する
6. **split GGUF**: `llama-quantize` は split GGUF を直接読めない。`llama-gguf-split --merge` 必須
7. **Adapter stacking**: 複数 adapter を重ねた場合、GGUF 変換時に全スタックを手動でマージする

---

## 現状と次回

v3 GGUF は adapter stacking ミスにより使用不可（garbage 出力）。v3 SFT adapter 本体（531MB）はローカルに保存済みで、正しい変換手順が判明したので次回再挑戦する。

現在の学習データ: 38件（閾値 5.0 に変更）。  
M2DX は 159 件全処理済み。次の SFT は adapter を正しくスタックして GGUF 変換まで完走させる。

---

*シリーズ一覧: [（１）](https://zenn.dev/hakaru/articles/m2dx-local-llm-audit-zero-true-positives) [（２）](https://zenn.dev/hakaru/articles/m2dx-local-llm-audit-rag) [（３）](https://zenn.dev/hakaru/articles/m2dx-local-llm-agentic-harness-eval) [（４）](https://zenn.dev/hakaru/articles/swift-audit-lora-fp-reduction) [（５）](https://zenn.dev/hakaru/articles/m2lora-code-review-pipeline)*
