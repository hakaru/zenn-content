---
title: "ローカルLLMって本当に開発に使える？（５）開発しながらLoRAデータが自動で貯まる仕組みを作った"
emoji: "🔄"
type: "tech"
topics: ["llm", "lora", "ollama", "python", "swift"]
published: true
---

:::message
**この記事の対象プロジェクト**

- **M2DX** — iOS/macOS 向け MIDI 2.0 対応 DX7 互換 FM シンセサイザーアプリ。[TestFlight 公開ベータ](https://testflight.apple.com/join/BAtGszPw) で試せる
- **M2DX-Core** — M2DX の DX7 互換エンジン部分。Pure Swift、Apache 2.0 で OSS 公開
- **MIDI2Kit** — M2DX-Core が依存する Swift 製 MIDI 2.0 ライブラリ
:::

## 前回までのあらすじ

このシリーズでは M3 Ultra (96GB) + Ollama（= 手元のMacでLLMを動かすランタイム）を使って、自分の Swift/MIDI プロジェクトにローカル LLM を投入し続けている。

- **（１）監査**: llama3.3:70b 他10モデルで DX7 エンジンを監査 → 指摘52件・真陽性0件（TP=true positive、本当のバグの数）
- **（２）RAG**: Swift 仕様書を検索させる（= LLM に外部資料を読ませて答えさせる手法） → TP = 0 のまま
- **（３）aider**: ツール統合でコーディングアシスト → 実用的だが監査は依然ダメ
- **（４）LoRA**: 73件の手作りデータで Qwen2.5-Coder-14B をファインチューン（= 既存LLMに追加学習で味付けすること、LoRA はその効率版）→ **誤検知93%削減**

（４）で「量より密度」という教訓を得た。手作りデータは効いた。じゃあ**自動で高品質なデータを貯め続けたら？**

---

## 問題意識：手作りデータの限界

前回の LoRA では、実際に出た誤検知19件 + Swift仕様Q&A 36件 = 計73件を**手動**で作った。これで93%削減できたのだが、根本的な問題がある。

- **スケールしない**: 73件は1週間かかった。1,000件作るには？
- **一方向**: 新しいコードを書くたびにモデルの知識は古くなる
- **バイアス**: 作った人間の思い込みがデータに入る

理想は「**開発しながら自動的にデータが貯まり、LoRA が常に最新になる**」仕組みだ。

---

## アイデア：3つのAIを「採点官」にする

構想はシンプル。

```
コミットするたびに:
  1. ローカルLLM（llama3.3:70b）がコードをレビュー
  2. Claude Code / Gemini / Codex CLI が「そのレビューはどうか」を採点
  3. 3者の採点を合成して「理想のレビュー」を生成
  4. diff（= コードの変更箇所） → 理想レビュー の ペアを蓄積
  5. 十分溜まったら LoRA でファインチューン
```

ポイントは**クラウドAPIを使わない**こと。Claude Code / Gemini / Codex はすでに開発ツールとして手元で動いている（AnthropicとOpenAIとGoogleのコーディングCLI）。これらをサブプロセスとして呼べばいい。

```python
# API呼び出しではなく、CLIをサブプロセスで起動
proc = await asyncio.create_subprocess_exec(
    "claude", "-p", prompt,  # claude -p "..."
    stdout=asyncio.subprocess.PIPE,
)
```

`claude -p "..."` はノンインタラクティブモードで動く。`gemini -p` も `codex exec` も同様。**APIキー不要、課金なし**（すでに契約済みのツールを使うだけ）。

---

## 仕組み：M2LoRA パイプライン

作ったリポジトリ: https://github.com/hakaru/M2LoRA

### 全体フロー

```
MIDI2Kit / M2DX / M2DX-Core
        │
   git commit
        │
  post-commit フック（= コミット直後に走るgitスクリプト、バックグラウンド実行）
        │
┌───────▼────────────────────────────────────┐
│  llama3.3:70b (Ollama)                     │
│  → diff を受け取り、コードレビューを生成    │
└───────┬────────────────────────────────────┘
        │ local_review
┌───────▼────────────────────────────────────┐
│  Claude Code CLI ┐                         │
│  Gemini CLI      ├ asyncio で並列実行       │
│  Codex CLI       ┘                         │
│  それぞれが:                                │
│    SCORE: 7                                │
│    CORRECTION: エラーハンドリングが不足    │
│    REVIEW: （独立したレビュー）            │
└───────┬────────────────────────────────────┘
        │ 3者の評価
┌───────▼────────────────────────────────────┐
│  Claude Code CLI（合成）                   │
│  → 矛盾点を指摘しつつ、Swift慣習に         │
│     最も正しいレビューを生成               │
│  → synthesized_review                      │
└───────┬────────────────────────────────────┘
        │
   SQLite (WAL モード)
        │
   m2lora export → Alpaca JSONL（= 1行1JSONの学習用フォーマット）
        │
   VAST.ai（= GPU を時間貸ししてくれるサービス）
   + Unsloth（= LoRA 学習を高速化するライブラリ）
   → LoRA アダプタ（= 元モデルに足す小さな追加重み）
        │
   Ollama にインポート → 次のサイクルへ
```

### 評価の構造化出力

3つのCLIには必ずこのフォーマットで答えるよう指示する:

```
SCORE: [0-10の整数]
CORRECTION: [欠落・誤りの訂正。問題なければ「なし」]
REVIEW: [独立したSwiftコードレビュー]
```

正規表現でパースするので、CLIがマークダウンや前置きを混ぜても安定して取れる。

```python
def parse_eval_output(text: str) -> EvalResult:
    score_m = re.search(r"^SCORE:\s*(\d+(?:\.\d+)?)", text, re.MULTILINE)
    corr_m = re.search(
        r"^CORRECTION:\s*(.*?)(?=^REVIEW:|\Z)", text, re.MULTILINE | re.DOTALL
    )
    rev_m = re.search(r"^REVIEW:\s*(.*)", text, re.MULTILINE | re.DOTALL)
    return EvalResult(
        score=float(score_m.group(1)) if score_m else None,
        correction=corr_m.group(1).strip() if corr_m else "",
        independent_review=rev_m.group(1).strip() if rev_m else "",
    )
```

### SQLite スキーマの要点

| カラム | 意味 |
|--------|------|
| `code_diff` | 入力（Alpaca の input） |
| `synthesized_review` | 理想レビュー（Alpaca の output） |
| `claude_score` / `codex_score` / `gemini_score` | 各採点官のスコア |
| `score_stddev` | 3者のスコアばらつき（標準偏差） |
| `flagged` | stddev > 2.0（採点官が意見不一致） |
| `exported` | Alpaca 変換済みフラグ |

エクスポート時は**平均スコア6未満を除外**し、低品質データの混入を防ぐ。

---

## 試してみた：設計ドキュメントをレビューさせる

パイプラインを作る途中で、設計ドキュメント自体を3モデルにレビューさせてみた。

### llama3.3:70b のレビュー（スコア推定：5/10相当）

> エラーハンドリングの強化、セキュリティ対策の徹底、パフォーマンスの最適化（SQLite → PostgreSQL移行）…

**汎用的すぎる**。SQLite を PostgreSQL に移せという提案は、このユースケース（LoRA学習データ蓄積、数百〜数千件）では完全にオーバーエンジニアリング。Swift/MIDI ドメインへの言及がゼロ。

### Gemini CLI のレビュー（スコア：8.5/10）

> **CLIツールの出力パースの難易度**: CLIの出力はANSIエスケープシーケンスやMarkdownが含まれることが多く、正規表現でのパースは壊れやすい（Brittle）です。
>
> **SQLite の書き込みロック**: 連続コミット時に複数プロセスが同時書き込みして `database is locked` が発生する可能性。WALモードと `storage.py` でのリトライ処理を推奨。
>
> **「クラウドAPI不使用」の定義矛盾**: CLIツールは内部的にクラウドAPIを呼んでいます。

**段違いに具体的**。「CLIの出力パース」は実装時に確実に詰まるポイントで、設計段階で気づけたのは価値が高かった。「定義矛盾」の指摘も鋭い（CLIを通じている以上、厳密にはクラウド通信が発生する）。

この差が**そのままLoRAのゴール**を示している。llama3.3:70b に Gemini レベルの具体性と鋭さを学ばせたい。

---

## 実装上のポイント

### git hook はバックグラウンドで

コミットがブロックされてはいけないので `&` でデタッチ。

```bash
cd "$PIPELINE_DIR" && \
  uv run python -m pipeline.cli commit \
    --project "$PROJECT" \
    --repo "$REPO" \
    --commit "$COMMIT" \
    >> "$LOG" 2>&1 &
```

### Semaphore(2) で CLI の同時実行を制限

3つのCLIを一気に走らせるとレートリミットや設定ファイル競合が起きる可能性がある。

```python
_SEMAPHORE = asyncio.Semaphore(2)

async def _run_cli(cmd: list[str], prompt: str) -> str:
    async with _SEMAPHORE:
        proc = await asyncio.create_subprocess_exec(*cmd, prompt, ...)
        try:
            stdout, _ = await asyncio.wait_for(proc.communicate(), timeout=60.0)
            return stdout.decode()
        except asyncio.TimeoutError:
            proc.kill()
            await proc.wait()  # ゾンビプロセスを回収
            return ""
```

`proc.kill()` 後に `await proc.wait()` を忘れるとゾンビプロセスが残る。これはGeminiのレビューで「重要な問題」として指摘された（パイプライン自身のコードレビューで発見されたバグ）。

### 合成プロンプトの設計

3者の評価をただ混ぜるのではなく、「矛盾点を指摘してから」合成させる。

```
各評価の矛盾点を指摘した上で、Swift/MIDIライブラリ開発の
慣習に最も正しいレビューを生成してください。

矛盾する意見がある場合はSwiftの公式スタイルガイドと
慣習に基づいて判断してください。
```

3者が同じ間違いに同意している場合（Geminiが指摘した「3モデル一致でも誤診」リスク）の対策として、Swift慣習を判断基準として明示している。

---

## 現時点の状況

パイプラインは完成し、MIDI2Kit / M2DX / M2DX-Core の3リポジトリのコミットフックにインストール済み。

```
$ m2lora stats
Total: 0 | Exported: 0 | Flagged: 0
```

まだデータ収集フェーズ。LoRAのファインチューニングに必要なデータ量は経験的に**100〜300件程度**（前回の73件でも93%削減できた）。開発を続けながら貯めていく予定。

---

## このアプローチの「ミソ」

1. **開発コストゼロでデータが貯まる**: コミットするだけで自動収集される。専用の作業時間が不要。

2. **実際のコードでトレーニングされる**: プロジェクト固有の Swift/MIDI イディオム、設計パターン、命名規則がデータに反映される。汎用データセットにはない。

3. **常に最新のコードをカバー**: リファクタリング後や新機能追加後のコードもデータになる。

4. **「先輩AI」の合議制**: Claude / Gemini / Codex は互いに異なるアーキテクチャ・学習データを持つ。3者が同意するレビューは信頼性が高い。

5. **APIなし**: すでに開発ツールとして使っているCLIを転用するだけ。追加コスト不要。

---

## 次のステップ

- 100件貯まったら VAST.ai + Unsloth でファインチューン
- LoRA 前後の比較: 同じ diff に対してスコアがどう変わるか
- `flagged=1`（採点官の意見不一致）レコードの傾向分析

データが十分溜まったら続編を書く。

---

## まとめ

| 項目 | 内容 |
|------|------|
| ローカルLLM | llama3.3:70b（Ollama） |
| 採点官 | Claude Code CLI / Gemini CLI / Codex CLI |
| トリガー | git post-commit フック（自動）/ `/m2review` スラッシュコマンド（手動） |
| 学習形式 | Alpaca JSONL → Unsloth（VAST.ai） |
| リポジトリ | https://github.com/hakaru/M2LoRA |

「ローカルLLMに先輩AIのレビューを学ばせる」というループを作った。開発を続けながらモデルが育っていくはずで、（４）の手作りデータより持続可能なアプローチだと思っている。100件溜まったら結果を報告する。

---

## 追記：100件超えてLoRAを作った（2026-05-04）

数週間で 174 件溜まった（M2DX のみ）。「100件で報告」と書いた手前、やってみた結果を書く。

結論を先に。

- ✅ Stage 1 LoRA **学習成功**: vast.ai H100 で 35 分 / loss 1.13 → 0.46 / **$1.70**
- ✅ Gemini評価では明確改善: avg 3.85 → **5.32** / 高評価（≥7）率 **13.7% → 40.8%**
- ❌ Claude評価では横ばい〜微減: 3.76 → 3.42
- ❌ Codex評価器、長時間バックフィル中に **93%** NULL（死んだ）
- ❌ ペア比較できてない（vanilla期と LoRA期で違う commit を評価してた）

順番に。

### Stage 1: spec を先に覚えさせる

LoRA は **2段構え**にしてる。Stage 1 は **CPT**（continued pre-training、= 元モデルの「素の文章を読む」訓練を引き継いで継続させる方法）で**ドメイン知識**を入れ、Stage 2 は **SFT**（supervised fine-tuning、= 入力→正解出力のペアで「振る舞い」を教える方法）でレビュー作法を上乗せする想定。

レビュー直接ではなく、まず spec corpus（= 仕様書テキスト集、FM 合成・MIDI 2.0・CoreMIDI・CoreAudio・Swift）を 3266 chunk（= 文書を学習に流せるサイズに切ったかたまり）集めて Stage 1 をやった。

vast.ai で H100 SXM 80GB（NVIDIA の最新世代GPU、VRAM 80GB付き、$1.83/hr / チェコ）を借りて流したら **35:10 で完走**。

```
12%|███   | 50/409 [04:24<30:53, 5.16s/it]
{'loss': '1.136', 'learning_rate': '9.8e-05', 'epoch': '0.1225'}
{'loss': '0.8053', ...}
{'loss': '0.4572', 'learning_rate': '5.164e-07', 'epoch': '0.9798'}
{'train_runtime': '2111', 'train_loss': '0.6644', 'epoch': '1'}
```

`loss`（= 学習中の予測誤差、低いほど学べてる）が 1.136 → 0.4572 へ。learning_rate（= 重みを更新する勢い）も cosine（= 半周期の余弦カーブで滑らかに減らすスケジュール）でちゃんと下がってる。残高 $5.74 から $1.70 で済んだ。

### LoRA を Ollama に組み込む

学習済み adapter（= 追加重みファイル、556MB の `safetensors` 形式）を `convert_lora_to_gguf.py` で **gguf**（= llama.cpp / Ollama が読める形式）に変換して、Ollama の Modelfile（= モデルの組み合わせを書くレシピ）から読ませる。

```
FROM llama3.3:70b
ADAPTER /path/to/m2lora-domain.gguf
PARAMETER num_ctx 8192
```

```
$ ollama create llama3.3:70b-m2lora-v1 -f Modelfile
$ ollama run llama3.3:70b-m2lora-v1 "What is MIDI 2.0 Property Exchange?"
> MIDI 2.0 Property Exchange (PX) is a protocol that allows
> devices to dynamically discover, configure, and control
> each other's capabilities and settings over a MIDI connection.
```

CPTだけでもMIDI 2.0 PEを正しく説明する。素のllama3.3:70bは「PXはたぶん...」程度の曖昧な答えしか返さなかったので、*ちょっとだけ感動*。

### バックフィルで残り全コミット評価

LoRA入れた状態で、未処理だった 62 commit を遡って評価し直した（**バックフィル** = 過去のデータをあとから埋める作業）。狙いは Stage 2 用の高品質データを稼ぐこと。

これが思ったより**事故った**。

### 事故その1: 並列で詰まる

「2並列なら速くね？」と思って `OLLAMA_NUM_PARALLEL=2` にしたら、各リクエストが3倍遅くなって15分タイムアウト連発。

なぜかというと、`llama3.3:70b` を NUM_PARALLEL=2 にすると **約 160GB** 必要になって半分がCPUに退避する。VRAM（= GPU 専用メモリ）80GB の GPU で 70B モデルを **4-bit 量子化**（= 重みを4ビットに圧縮して容量を1/4にする手法）で動かしてる以上、**KVキャッシュ**（= 推論中の中間状態を保存しておくバッファ、文脈長 × 2並列で2倍要る）を載せた瞬間にオーバーフロー。素直に `=1` に戻した。

### 事故その2: LoRA入りモデルが固まる

NUM_PARALLEL=1 にしても別の問題が出た。

**LoRA adapter込みのモデルで、生成中の Python client を `kill -9` すると、Ollama 側のタスクが完全には解放されない**。次のリクエストがキューに詰まって永遠に応答待ち。

最初の数件は2-3分で順調だったのに、たまたまmarkdown比率が高い特定の commit でハングしたから kill した。以降5時間で **6件** しか保存できなかった。

```
[m2lora] reviewing with llama3.3:70b-m2lora-v1...
（35分経過、TCP は ESTABLISHED のまま）
aiohttp.client_exceptions.SocketTimeoutError: Timeout on reading data from socket
```

*いや、そういうことじゃなくて…*

aiohttp（= Python の HTTPクライアント）の `total=900s` タイムアウトはなぜか発火しない（後で `sock_read=120s`、= 「2分間データが来なかったら切断」のセーフティを追加した）。Ollama log を見ると `/api/generate` の完了エントリすら出ない。内部で止まってる。

### 復活：完全再起動 + pre-warm

`pkill -9 -f "ollama (serve|runner)"` で全部殺してから:

```bash
OLLAMA_MODELS=/Volumes/Media/ollama/models \
  OLLAMA_NUM_PARALLEL=1 OLLAMA_KEEP_ALIVE=24h \
  ollama serve > /tmp/ollama.log 2>&1 &

# 必ず pre-warm
curl http://localhost:11434/api/generate \
  -d '{"model":"llama3.3:70b-m2lora-v1","prompt":"hi","stream":false}'
```

これで復活。以降4時間ノンストップで70commitを保存できた。**3.5分/件で安定**。

教訓: *LoRA adapter込みmodelで client kill が起きたら、即 Ollama 完全再起動 + pre-warm*。30分待つくらいなら3分でリセットしたほうが傷が浅い。

### 結果：Geminiは大絶賛、Claudeはお気に召さず

LoRA入れる前後で、各評価CLI のスコア分布を比較した。同じ commit を両方で評価してないので**厳密比較じゃない**けど、傾向は出てる。

| 指標 | vanilla `llama3.3:70b`（95件） | `m2lora-v1`（76件） | 差 |
|---|---|---|---|
| Gemini 平均 | 3.85 | **5.32** | **+1.47** |
| Gemini ≥5 率 | 25.3% | **56.6%** | +31pt |
| Gemini ≥7 率 | 13.7% | **40.8%** | +27pt |
| Claude 平均 | 3.76 | 3.42 | -0.34 |
| Codex（参考） | NULL率 27.8% | NULL率 **93.4%** | — |

Gemini からは**明確に上がった**。Claude からは横ばい〜微減。Codex は途中で死んだのでサンプル不足。

### 評価者間でなぜこんなに乖離する？

仮説3つ。

1. LoRA-v1 のレビューが**長くて自信ありげ** → Gemini が冗長性を評価しがち、Claude は内容の正しさを見るので長さに惑わされない
2. LoRA-v1 が **MIDI/Swift 専門用語を頻出** → Gemini は「専門的」と評価、Claude は「内容次第」
3. **ペアになってない**: vanilla は新しめの100commit、LoRA-v1 は残り（古い commit）。古い commit は依存少なくテスト未整備で指摘点が多いから、自然と高評価になる傾向

たぶん全部影響してる。

### 何ができてないか

- **ペア比較できてない**: 同じ commit を両モデルで評価し直さないと、世代差か commit selection か切り分けられない
- **Codex 評価器の死活監視がなかった**: 長時間バックフィル中に 93% NULLになっても気づかなかった
- **Stage 2 用のサンプル数**: avg≥6.0（= 採点官3人の平均6点以上）でフィルタすると LoRA期の export は **7件** しかない。SFT には全然足りない

正直、Stage 2 用のデータ稼ぎという目的なら**まだ実用じゃない**。Geminiの数字だけ見ると改善してるけど、それが「いい review」なのか「Gemini好みのreview」なのかは保留。

### 次の一手

- Codex CLIに死活監視を入れる（30分ごとに `codex --version` 等で生存確認、死んでたら再起動）
- ペアテスト用ハーネスを作る: 同じ commit集合を両世代で評価し直して直接比較
- M2DX以外（MIDI2Kit / M2DX-Core）でも収集再開して、commit集合を増やす
- Stage 2: alpaca SFT は7件じゃ足りないので、CPT継続 or データ生成に方針転換するか考え中

### 更新したまとめ

| 項目 | 内容 |
|---|---|
| 収集件数 | M2DX 173件 |
| Stage 1 LoRA | `llama3.3:70b-m2lora-v1`（CPT / spec corpus 3266 chunks） |
| 学習コスト | vast.ai H100 80GB / 35分 / **$1.70** |
| Gemini基準改善 | ≥7 高評価率 **13.7% → 40.8%** |
| Claude基準改善 | なし（むしろ微減） |
| Stage 2 状況 | サンプル7件で再考中 |

「ローカルLLM自身を採点官3人に育てさせる」ループは**最低限動いた**。ただし*「ペアテストせずに改善を語るな」*という当たり前の罠に一度落ちたので、次回は同じ commit を両世代で評価する仕組みから作る。

---

## 追記その2：Mac M3 Ultra で同じことやってみたら全然ダメだった（2026-05-05）

vast.ai で $1.70 で済んだのは確かに安いけど、手元に **M3 Ultra 96GB unified RAM**（= CPU と GPU が同じメモリを共有してるアーキテクチャ）がある。70B 4-bit + LoRA は理論上ぜんぶ載る。MLX（= Apple 純正の機械学習フレームワーク、Metal GPU を直接使う）なら速いはず。**同じデータ・同じ設定で回したら何分で終わる？**

これがまた、地獄だった。

結論を先に。

- ❌ 3回挑戦して **3回とも Metal GPU error で死亡**（iter 400 / 70 / 10、`iter` = 学習ループの繰り返し回数）
- ❌ MLX-LM の `lr_schedule`（= 学習中に learning_rate を時間で動かす設定）で warmup（= LR を低い値から徐々に上げる助走区間） + cosine（= ピーク後に余弦カーブで下げる）が**設定通りに動かなかった**
- ❌ ベストなadapter (iter 200, val 0.822、`val` = 学習に使ってない検証データでの loss) は v1 final 0.46 に届かず
- ✅ **動くは動く** — 96GB unified RAM、70B+LoRA で peak 46GB、ハードは余裕
- ✅ コスト $0（電気代のみ）、**ただし 6 倍遅い**（v1 35min vs Mac 3.5h+）

順番に。

### 準備：mlx-lm + 40GB ダウンロード

```bash
uv add mlx-lm hf_transfer
HF_HUB_ENABLE_HF_TRANSFER=1 huggingface-cli download \
  mlx-community/Llama-3.3-70B-Instruct-4bit \
  --local-dir ~/models/Llama-3.3-70B-Instruct-4bit-MLX
```

40GB を ~6MB/s でダウンロード。**2.5時間**待った。`hf_transfer` を入れてもなぜか速度は変わらず…HF 側の速度規制？

### 罠1：warmup なしで発散

最初は v1 と同じ `lr=2e-4` をそのまま入れた。気づいたらこうなってた。

```
Iter 10: 1.966
Iter 30: 5.974
Iter 50: 19.652  ← peak
Iter 90: 10.078
```

vast.ai v1 は `warmup_steps=100` を入れてたから、これが効いてる気がする。MLX 版は `warmup` を YAML に入れずスタートしてしまった結果、LoRA重みが初期値 0 から一気にぶっとんで発散。

### 罠2：YAML で warmup+cosine を書いたが、LR が線形に上がり続ける

mlx-lm のドキュメント通りに書いた。

```yaml
lr_schedule:
  name: cosine_decay
  warmup: 100
  warmup_init: 1.0e-7
  arguments: [2.0e-4, 308]
```

期待: warmup100 で 0→2e-4、その後 cosine で 2e-4→0 に decay。

実際:

```
Iter 10:  LR 2.099e-06   ← warmup 12% のはずが
Iter 100: LR 4.808e-05   ← warmup 完了で 2e-4 のはずが
Iter 200: LR 9.805e-05
Iter 300: LR 1.480e-04   ← LR が線形に上がり続けてる
Iter 408: LR 2.000e-04   ← 終端でようやくピーク
```

**cosine_decay が一切効いてない**。

```
Iter 200: val 0.822  ← BEST
Iter 250: train 2.067 ← また発散開始
Iter 300: val 10.157  ← 完全に壊れた
```

iter 200 までは収束してたが、LR が想定外に上がり続けて再発散。

### 原因：`iters` は **micro-step** だった

ここで用語を整理させて:

- **iter / step**: 学習ループの1周。1バッチを処理して終わる単位。
- **batch**: 一度にまとめて流すデータの数。`batch_size: 2` なら2サンプル同時。
- **gradient accumulation（grad_accum）**: メモリ節約のテク。`grad_accum_steps: 4` ならバッチ4回ぶん勾配を貯めてから1回だけ重み更新する。**実質バッチサイズは batch_size × grad_accum_steps**。
- **micro-step**: バッチ1回を処理する最小単位。
- **opt step（optimizer step）**: 重みが実際に更新される単位（= grad_accum 単位ぶんの micro-step ごとに1回）。

mlx-lm のソースを読んだ:

```python
# mlx_lm/tuner/trainer.py
for it, batch in zip(range(1, args.iters + 1), iterate_batches(...)):
    if it % grad_accum_steps == 0:
        optimizer.update(...)  # ← LR スケジュールはここで進む
```

つまり:

- `iters: 408` + `grad_accumulation_steps: 4` → 実際の **opt step は 102 のみ**
- `warmup: 100` は **opt step** で数える → 完了は iter 100×4 = **iter 400**
- iter 100 で観測 LR 4.8e-5 = 2e-4 × 25/100（opt step 25 = warmup 25% 地点）と完全一致

なるほど。HF Trainer（= HuggingFace の学習ライブラリ、Unsloth が内部で使ってるやつ）は `warmup_steps=100` が opt step で、`num_train_epochs=1`（= データ全体を1周）で計算した opt step 数のなかでの warmup。MLX は `iters` 自体が micro-step で、warmup や cosine は opt step。**同じ数字を入れたら 4倍ズレる**。

修正: `iters: 1636 = 409 × 4`。

### 罠3：直後 Metal GPU error で死亡

`iters: 1636` で再起動。warmup ramp が想定通り動き始めた。よし。

```
Iter 70: Train loss 0.903, Learning Rate 3.208e-05
libc++abi: terminating due to uncaught exception of type std::runtime_error: 
[METAL] Command buffer execution failed: Discarded 
(victim of GPU error/recovery) 
(00000005:kIOGPUCommandBufferCallbackErrorInnocentVictim)
```

**`kIOGPUCommandBufferCallbackErrorInnocentVictim`**。直訳すると「無実の犠牲者」。Metal（= Apple の GPU API、CUDA の Apple版） で他の処理が死んだ巻き添えで、こっちのコマンドバッファ（= GPU に投げた命令の塊）まで kill されたみたいな意味。

`save_every: 25` に縮めて再々挑戦したら **iter 10 で同じエラー**。劣化してる。

### 結局3連敗

| 試行 | iters設定 | 死因 | 到達 | best loss |
|---|---|---|---|---|
| 1 | 408（warmup無し） | Metal GPU error @ iter 408 | 完走時死亡 | 9.5（発散） |
| 2 | 408（warmup有、解釈ミス） | 自然終了 | 408 | val 0.822 @ iter 200 |
| 3 | 1636（修正版） | Metal GPU error @ iter 70 | 70 | (途中) |
| 4 | 1636（save_every=25） | Metal GPU error @ iter 10 | 10 | (途中) |

**ハードに問題はなかった**。peak メモリ 46GB（96GB unified の半分）、サーマル警告なし、`pmset -g therm` も `No thermal warning level has been recorded`。

それでも Metal の command buffer がランダムに殺される。MLX フォーラムや GitHub issues を漁ると、**長時間連続 GPU 計算 + 大きいモデル**で散発的に出るらしい。再起動で改善することもあるが根治はしてない。

### vast.ai と並べると

| 軸 | vast.ai H100 v1 | Mac M3 Ultra MLX |
|---|---|---|
| 動いた？ | ✅ 408/408 完走 | ❌ 3/4 で Metal error |
| Final loss | **0.46** | 0.822（iter 200 ベスト）/ 残りは発散か途中死 |
| Wall time | 35:10 | 3.5h（iter 200まで届いた回） |
| 直接コスト | $1.70 | $0（電気代） |
| セットアップ | torch/cu124/unsloth で依存解決の罠 | mlx-lm 入れるだけだが YAML config の罠 |
| 開発体験 | Unsloth が至れり尽くせり | 各設定の意味を毎回ソース確認 |
| 安定性 | ◎（H100 が悪さしない） | △（Metal が散発死） |

### 教訓

1. **`iters` の単位はフレームワーク依存**: HF Trainer は opt step、MLX-LM は micro-step。grad accumulation を入れると 4倍ズレる
2. **lr_schedule は実機で LR ログをまず見る**: 観測した LR の数字が想定と桁違いなら設定間違い。コードに書いた値ではなく、走らせた結果を信じる
3. **Metal GPU error は config では治らない**: ハードがイケても OS / driver レベルで kill される。長時間連続で 70B を回すのは M3 Ultra でもまだギリギリ
4. **MLX エコシステムは Unsloth に追いつけてない**: ドキュメント、エラーメッセージ、ベストプラクティス。**1〜2年は cloud 一択**

ベスト adapter（iter 200, val 0.822）は手元に残ったので、Stage 2 比較用に Ollama に組み込んで採点官3人に評価させれば、品質差が定量化できる。それは次回。

---

## 更新したまとめ（2026-05-05）

| 項目 | 内容 |
|---|---|
| 収集件数 | M2DX 173件 |
| Stage 1 LoRA v1 | vast.ai H100 / 35min / **$1.70** / loss 0.46 |
| Stage 1 LoRA v2 (Mac MLX) | M3 Ultra / 3.5h / $0 / loss 0.822（途中） / 後続2回はMetal死 |
| Gemini評価改善 (v1) | ≥7 高評価率 13.7% → 40.8% |
| 教訓 | クラウド圧勝。ハードはMacでも回るが、ツールチェーンが追いついてない |

リポジトリ: https://github.com/hakaru/M2LoRA
