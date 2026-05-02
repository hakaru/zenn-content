---
title: "ローカルLLMって本当に開発に使える？（５）開発しながらLoRAデータが自動で貯まる仕組みを作った"
emoji: "🔄"
type: "tech"
topics: ["llm", "lora", "ollama", "python", "swift"]
published: false
---

:::message
**この記事の対象プロジェクト**

- **M2DX** — iOS/macOS 向け MIDI 2.0 対応 DX7 互換 FM シンセサイザーアプリ。[TestFlight 公開ベータ](https://testflight.apple.com/join/BAtGszPw) で試せる
- **M2DX-Core** — M2DX の DX7 互換エンジン部分。Pure Swift、Apache 2.0 で OSS 公開
- **MIDI2Kit** — M2DX-Core が依存する Swift 製 MIDI 2.0 ライブラリ
:::

## 前回までのあらすじ

このシリーズでは M3 Ultra (96GB) + Ollama を使って、自分の Swift/MIDI プロジェクトにローカル LLM を投入し続けている。

- **（１）監査**: llama3.3:70b 他10モデルで DX7 エンジンを監査 → 指摘52件・真陽性0件
- **（２）RAG**: Swift 仕様書を検索させる → TP = 0 のまま
- **（３）aider**: ツール統合でコーディングアシスト → 実用的だが監査は依然ダメ
- **（４）LoRA**: 73件の手作りデータで Qwen2.5-Coder-14B をファインチューン → **誤検知93%削減**

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
  4. diff → 理想レビュー の ペアを蓄積
  5. 十分溜まったら LoRA でファインチューン
```

ポイントは**クラウドAPIを使わない**こと。Claude Code / Gemini / Codex はすでに開発ツールとして手元で動いている。これらをサブプロセスとして呼べばいい。

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
  post-commit フック（バックグラウンド実行）
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
   m2lora export → Alpaca JSONL
        │
   VAST.ai + Unsloth → LoRA アダプタ
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
| `score_stddev` | 3者のスコアばらつき |
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
