---
title: "DDSPモデル出荷判定でハマった3つの罠 + Codex事前レビュー4イテレーションの記録"
emoji: "🪤"
type: "tech"
topics: ["dsp", "coreml", "macos", "auv3", "codex"]
published: false
---

# DDSPモデル出荷判定でハマった3つの罠 + Codex事前レビュー4イテレーションの記録

## はじめに

**TineModeler4** は Fender Rhodes Electric Piano を物理モデル + DDSP (CoreML) + 実録サンプルのハイブリッドで再現する AUv3 / スタンドアロンシンセです。前回までの記事で Phase 5 (HybridVoice 統合) まで完了済み、今回は **Phase 1.2** で再学習した DDSP モデル「MethodC」(Sine-Source NSF) の **出荷判定** と、その先の **Phase D ランタイム統合** の設計過程の話です。

今日 1 日でハマった罠を 3 つ紹介します。

1. **ETI 録音の基音不在で診断ゲートが不適合になっていた**
2. **試聴用 WAV のファイル名衝突で UI がキャッシュしていた**
3. **「-3dBFS 正規化が pitch 知覚を破壊する」という物理的にあり得ない仮説に走った**

加えて Phase D の **Codex 事前レビュー 4 イテレーション** の記録 (⛔ CRITICAL × 3 回 → ⚠️ APPROVED) も書きます。

---

## 背景: MethodC とは

TineModeler シリーズの DDSP 路線は過去 3 回失敗しています:

- **MethodA-v1**: シンプルな LSTM スペクトル補正 → 過学習
- **MethodA-v2**: 16-band mel gain → ノイズボコーダ化 (gate saturation)
- **MethodB**: NSF (Neural Source Filter) で physics audio を source として使う → モデルが phys を無視、cond pathway だけで学習

3 回連続で諦めかけた後、4 回目として **MethodC** を設計しました。Wang & Yamagishi 2019 のオリジナル NSF に近い構成で、**sine 波**を excitation source にします。

```
sine_excitation(f0_norm) → cumulative phase sine
                ↓
ConditioningEncoder(f0, loudness, midi) → 128-dim cond
                ↓
SineSourceFilter (10-layer causal dilated CNN, RF=2047) → audio output
                                                        = β · env(loud) · tanh(out_proj)
```

Vast.ai の RTX 4090 で 500 epoch、約 14 時間、$6 で学習が完了しました。最終 loss = 0.4718。

---

## 罠 1: 診断ゲートがデータ特性に合っていなかった

学習完了後、`diagnose_method_c.py` で出荷判定:

```
pred MSS  4.495   (MethodB 7.10, MethodA-v2 8.076, phys 18.18)
zero_sine 6.444   (pred + 1.95、閾値 +3.0 — uses_sine FAIL)
shuf_sine 6.950   (+2.46)
shuf_cond 8.053   (+3.56 — cond は効いている)
sine_share  12.6% (first conv layer)
output |abs| mean 0.0020  ← silence-collapse 疑惑

⛔ NO-GO: model ignores sine excitation
```

MSS では全ベースラインを圧倒しているのに、**`uses_sine` ゲートで不合格**になりました。さらに出力振幅が 0.002 平均 = ほぼ無音という結果。

「これは MethodB と同じ `cond pathway bypass` の失敗かもしれない」と最初は思いました。

### 真の原因: ETI 録音には基音がない

ETI Roads MKII (Rhodes 実機録音、73 鍵 × 多数 velocity 層) の持続音区間を FFT で分析してみると、衝撃的な事実が判明しました:

```
midi 40 E2 (期待基音 82 Hz, h2 165 Hz, h4 330 Hz):
  持続音 top peaks: 329 / 330 / 494 / 165(弱) / 164(弱)
  → 基音 82 Hz は top 3 圏外、エネルギー ~0.04

midi 50 D3 (期待基音 147 Hz):
  持続音 top peaks: 294 / 293 / 295 / 587 / 292
  → 基音 147 Hz は top 3 圏外

midi 60 C4 (期待基音 262 Hz):
  持続音 top peaks: 523 / 522 / 524 / 521 / 525
  → 基音 262 Hz はほぼゼロ
```

**Rhodes の物理特性**で、tine の振動を electromagnetic pickup で拾うとき、**pickup 位置が node に近いほど基音が打ち消される** という性質があります。さらに tonebar (共鳴体) が 2nd / 4th 倍音を増幅。結果として**持続音は h2/h4 が dominant、基音はほぼ存在しない**。これは録音アーティファクトではなく、Rhodes の正しい音色です。

つまり:

- **target audio に基音がない**
- MethodC の sine excitation は **基音そのもの**
- モデルは sine を抑制して cond pathway の倍音構造で target に合わせるのが正解
- `uses_sine` ゲート (`zero_sine - pred > 3.0`) は **構造的に不適合**

「sine を使わないと損失が大きく悪化するはず」と仮定したゲートが、データの実態 (基音なし) と矛盾していました。

### 教訓: 診断ゲートはデータ特性とともに設計せよ

これは Codex audit (codex CLI で別 LLM に検証してもらう自前パイプライン) に依頼して独立検証してもらいました。Codex も同じ HDF5 を独立に FFT 投影し、`h1/max ≈ 0` を確認。verdict:

```
⚠️ AGREE WITH CAVEATS — Ship MethodC as mixable layer

ETI no-fund 確認 (Codex 独立 FFT で h1/max ≈ 0)
uses_sine gate 不適合 — sine 抑制は正解、cond 強く効いてる
MSS 4.495 で全先行手法に勝ち
midi 40 standalone は不可、3-layer mix なら OK
```

5 つの caveats を添えて出荷適性を認めてもらいました。

---

## 罠 2: ファイル名衝突で UI が同じ音を 3 回再生していた

Codex の audit を受けて、ユーザーに **A/B 試聴** で確認してもらうことに。`render_ab_wavs_c.py` で midi 40 / 50 / 60 × 5 stems (target/phys/A-v2/B/C) = 15 WAV を生成、それぞれを `clip_{idx}_midi_{midi}/{stem}.wav` というディレクトリ構造で配置。

ユーザー初回フィードバック:

> 全部 E の音程に聞こえる。MethodC は MethodB より良い。

最初これを「MethodC が pitch collapse している」と解釈し、深い調査に入りました。

- f0 (CREPE 出力) が `repeat_interleave` でステップ状になるのが原因かも?
- → f0 を linear interpolate に置換した版を render → **「ほとんど変化が感じられない」**
- 「では cond pathway 由来?」「いや MethodB は中高音域では問題ないから...」
- 「Rhodes の基音不在で knee 倍音が混乱?」

しばらく悩んだ後、ユーザーが言いました:

> まず target.wav が全部同じ音。

え?

```bash
md5sum results/ab_wavs_c/clip_*/target.wav
# 6a0a92fb...  clip_1041_midi_40/target.wav  ← E2
# 6b08ea6c...  clip_1450_midi_50/target.wav  ← D3
# 73d92aac...  clip_1865_midi_60/target.wav  ← C4
```

**ハッシュが違う = ファイル内容が違う**。FFT 直接確認:

```
clip_1041 target: dominant 329 Hz (E4)
clip_1450 target: dominant 294 Hz (D4)
clip_1865 target: dominant 523 Hz (C5)
```

つまりファイル自体は正しく E2 / D3 / C4 の (Rhodes 特性で h2/h4 dominant の) audio が入っている。

なのにユーザーには「全部同じ音」に聞こえる ⁉️

### 真の原因: ファイル名

3 ディレクトリすべての WAV のファイル名が `target.wav` でした。**ユーザーの UI / プレイヤーが同名ファイルを cache / dedupe して、最初の 1 つだけを 3 回再生** していた疑い。

切り分け実験として、同じファイルを別名で再送:

```bash
cp clip_1041_midi_40/target.wav E2test/TGT_E2_normalized.wav
cp clip_1450_midi_50/target.wav E2test/TGT_D3_normalized.wav
cp clip_1865_midi_60/target.wav E2test/TGT_C4_normalized.wav
```

→ ユーザー報告: **「明らかに正しい音程。全部違う音程。」** 確定。

`render_ab_wavs_c_v2/` を作って **flat 構造 + ユニークなファイル名** (`midi40_E2_target.wav` 等) で再生成。ユーザーが clean な audition で再評価:

| midi | MethodA-v2 | MethodB | MethodC | 勝者 |
|---|---|---|---|---|
| 40 (E2) | NG | 発振 | 明らかに良い | **C** |
| 50 (D3) | (n/a) | 良い | ピッチ揺れ | **B** |
| 60 (C4) | (n/a) | (n/a) | さらに揺れ | **B** |

**相補的な強み**が見えました — MethodC は低音 (E2) で MethodB の発振を克服、MethodB は中高音 (D3/C4) でピッチ安定。3-layer mix architecture (Physics + Samples + DDSP) で各レイヤーの弱点をカバーする運用に決定。

### 教訓: A/B 試聴は必ずユニークなファイル名にする

これまでの会話中で「**全部 E の音程**」「**B より良い**」「**MethodC は pitch collapse している**」「**silence collapse 疑惑**」と進めた分析、**全部この副作用**でした。データもパイプラインもモデルも正常。試聴方法論のバグでした。

何時間か悩んで気づいたとき、ぐったり。

---

## 罠 3: 物理的にあり得ない仮説に走った

罠 2 の途中、「**`-3dBFS peak 正規化** が pitch 知覚を破壊しているのでは?**」**」という仮説を提示してしまいました。

低 vel クリップは生 peak が 0.043 程度。これを -3dBFS (0.708) に正規化するには 16 倍 boost が必要。「16 倍も boost したら何か壊れるかも...」

これに対してユーザーから一発:

> linear gain で pitch 知覚が変わるなんてあり得ない。仮説は完全に間違いです。

その通り。線形利得は周波数特性を変えないので、相対的な倍音バランスは保たれます。pitch 知覚が変わるはずがない。基本的な物理を踏み外していました。

### 教訓: デバッグ中も基礎物理を忘れない

何時間も悩んでいると焦って「もしかして... 何かが何かに影響しているのでは...」という曖昧な仮説に逃げがち。ユーザーが冷静に基本に戻してくれて助かりました。

---

## ship 判定: 3-layer mix architecture

3 つの罠を経て出荷判定が確定:

```
Physics  ████████████░░░░░  pitch authority (基音含む全帯域クリーン)
Samples  ██████████░░░░░░░  attack & release の本物感
DDSP     ████░░░░░░░░░░░░░  Rhodes 持続音の倍音色付け (低音域中心)
```

各レイヤーが独立な volume パラメータで mix される設計。MethodC は低音 (E2) で MethodB の発振を救い、中高音では薄く混ぜる運用。AU パラメータ `ddspMix` (addr 16, range [0, 2], default 0.5) で操作可能。

`docs/PHASE_1_METHOD_C_RESULTS.md` に正式記録、Phase 1 をクローズ。

---

## Phase D: ランタイム統合の Codex 事前レビュー 4 イテレーション

ship 判定が出たので、次は **AUv3 ランタイムへ統合**する Phase D。`rhodex_method_c.mlpackage` (CoreML、680KB fp16、full-clip 176400 sample 出力) を HybridVoice の 4 番目の additive branch として組み込みます。

過去 (MethodA postmortem L1) の教訓:

> **Codex pre-impl review を必ず通せ。MethodA は通さずに出荷 → 全敗。**

なので Phase D plan を書いてから **必ず Codex に事前レビュー**を依頼。今回は 4 イテレーションかかりました。設計が大きく進化したので記録します。

### v1: ⛔ STILL CRITICAL — 5 blockers

最初の plan は **`AsyncMethodCRenderPool`** (Swift `@objc` クラス) + **`MethodCRendererCxxBridge`** (ObjC++) という構成。Codex の指摘:

1. **RT-safety**: `MethodCRendererCxxBridge::readSample` が ObjC msg send 経由になる。CLAUDE.md「render thread で ObjC msg send 禁止」に違反。
2. **Buffer race**: `buffers[i] = newBuffer` の Swift 配列 slot 置換は atomic ではない。
3. **Sample-rate interpolation 抜け**: 訓練 44.1 kHz、ランタイム 48 kHz が混在しているのに、Sustain stage の mix コードでサンプル index を `++` で進めていた → 8.8% ピッチアップ。
4. **Task 6 test が compile しない**: `HybridVoice()` default ctor も `renderSample()` API も存在しない。Swift XCTest + Cxx interop で書くべき。
5. **Sources/Core/TineModeler4Engine が存在しない**: engine は実際は `Sources/Bridge/TineModeler4Engine.swift` (Swift)。

### v2: ⛔ STILL CRITICAL — 4 new blockers

5 つ全部対処して再提出:

- アーキテクチャ転換 → **Swift クラス + ObjC++ bridge を捨て、純 C++ `MethodCRenderCache` (atomic struct + 事前確保 double-buffer)** + **Swift `MethodCWorker` が C-linkage accessor 経由でアクセス**。render thread は pure C++ atomic load のみ。
- Sample-rate interp 追加
- テストを Swift XCTest に書き換え
- Engine path を Bridge layer に修正

新たな blocker:

1. **`readyPacket` length 16-bit では 176400 入らない** → `gen[32] | length[24] | idx[8]` に repack
2. **Worker teardown race**: `MethodCWorker.stop()` が plain Swift `Bool`、queue join しない
3. **Test seam (`methodC_cache_test_inject`)** が production header に漏出
4. v1 で出てきた `AsyncMethodCRenderPool` / `MethodCRendererCxxBridge` 名が Task 1 / 11 の説明に残存

### v3: ⛔ STILL CRITICAL — teardown race + 非 atomic ポインタ

v2 fix 全部適用 + 包装:

- Packet `gen[32] | length[24] | idx[8]`
- `MethodCWorker.stopAndJoin()` (`DispatchSemaphore` で worker thread 終了を待機)
- Test seam 削除、production C accessor のみで test 可能に
- 全タスクの古い名前を purge

しかし Codex から新 blocker:

```
⛔ CRITICAL: Still not resolved
allSoundOff() does not quiesce the audio render thread.
After it returns, the render thread may already be inside a Sustain
block with a loaded methodC pointer. The plain pointer write in
hybridVoiceSetMethodCRenderer(..., nil) races with any in-flight
methodC->readSample(...) call.
```

要点: 「`hybridVoiceSetMethodCRenderer(voice, nil)` は **plain pointer write**。audio thread が並行で読んでる場合 race。`std::atomic<IMethodCRenderer*>` にしろ。」

### v4: ⚠️ APPROVED WITH MINOR — ようやく approved

v4 で本気の修正:

1. **`HybridVoice::Impl::methodC` を `std::atomic<IMethodCRenderer*>`** に。setter は `store(release)`、reader は `load(acquire)`。これでポインタ書き込み race を排除。

   ```cpp
   // ----- MethodC layer (44.1 kHz cache → runtime SR linear interp) -----
   auto* methodCPtr = s.methodC.load(std::memory_order_acquire);
   float methodCVal = 0.0f;
   if (methodCPtr != nullptr) {
       const float pos = s.methodCPos;
       const uint32_t idx0 = static_cast<uint32_t>(pos);
       const float    frac = pos - static_cast<float>(idx0);
       float s0 = 0.0f, s1 = 0.0f;
       const bool ok0 = methodCPtr->readSample(s.voiceID, s.methodCGeneration,
                                                idx0,     &s0);
       const bool ok1 = methodCPtr->readSample(s.voiceID, s.methodCGeneration,
                                                idx0 + 1, &s1);
       if (ok0 && ok1) {
           methodCVal = s0 + (s1 - s0) * frac;
       } else if (ok0) {
           methodCVal = s0;  // last sample of cache, hold
       }
   }
   s.methodCPos += s.methodCStep;
   ```

2. **MethodC chain ライフサイクルを AU `allocateRenderResources()` / `deallocateRenderResources()` に紐付け**。Apple AU の契約で「`deallocateRenderResources()` 実行中は audio render thread は走らない」が保証されているので、その境界でのみ teardown する。これで use-after-free 不可能。

3. `engine` 変数の ARC deinit に頼らず、明示的な `shutdownMethodC()` メソッドで teardown (idempotent)。`deallocateRenderResources` から直接呼ぶ。

   ```swift
   public func shutdownMethodC() {
       // 1. 全 voice から renderer detach (atomic release-store nil)
       for v in voiceHandles {
           TineModeler.hybridVoiceSetMethodCRenderer(v, nil)
       }
       // 2. Worker stop + JOIN (500ms bounded timeout)
       methodCWorker?.stopAndJoin()
       methodCWorker = nil
       // 3. Renderer 解放
       methodC_cache_renderer_destroy(methodCRendererPtr)
       // 4. Cache 解放
       methodC_cache_destroy(methodCCachePtr)
   }
   ```

4. `methodC_cache_publish` で length clamp 追加 (defensive)

5. Codex 5 minor 全部適用 → ⚠️ APPROVED WITH MINOR

### 4 イテレーションの教訓

- **plan v1 の段階で気づくのは無理だった** blocker が多い。書いてみて初めて見える issue がある。
- **ObjC msg send / atomic pointer / 64-bit packet 設計 / AU lifecycle** といった concurrency の罠は、人間 1 人ではなかなか全部押さえきれない。
- Codex は確実に critical を見つけてくれた。**特に v3 で「`allSoundOff` は quiesce にならない」を指摘してきたのが鋭い**。AU ライフサイクルの正しい理解を要する判断で、自分一人なら気づかなかった可能性が高い。
- 結果として 4 イテレーションの過程で **architectural な質**が大きく上がった。最終形は pure C++ atomic cache + Swift worker via C accessors という、レイヤー分離も clean な設計。

---

## Phase D 実装: subagent-driven で Task 1-9 完了

Codex APPROVED 後は subagent-driven development (superpowers plugin) で実装。fresh subagent をタスクごとに dispatch、各タスク完了後に独立にレビュー。

| Task | 内容 | Test 結果 |
|---|---|---|
| 1 | `IMethodCRenderer` 抽象 interface (Core, dep-free) | build pass |
| 2 | `rhodex_method_c.mlpackage` Resources/ にバンドル | build pass |
| 3 | `MethodCRunner` Swift CoreML wrapper | 3 tests pass |
| 4 | `MethodCRenderCache` + `MethodCCacheRenderer` (pure C++) | 3 tests pass |
| 5 | `MethodCWorker` Swift worker | 3 tests pass |
| 6 | HybridVoice 4th additive branch + SR-aware interp | 3 tests pass |
| 7 | `ddspMix` AU param (addr 16) | EngineParameterTests 更新 |
| 8 | Engine が MethodC chain を所有 + shutdownMethodC | build pass |
| 9 | Full build + 4-layer dep + 80+ test gate | 89 tests, 3 pre-existing failures |

subagent 経由の興味深い発見:

- **Task 4 で namespace `tm4` → `TineModeler` にリネーム必要**。Swift Cxx interop が `tm4` 名前空間を見せないため。
- **`MethodCRenderCache.h` + `MethodCRenderCacheAPI.h` に header 分割必要**。std::atomic を含むヘッダを Swift から見ようとすると interop が破綻するので、Swift facing な C-linkage 宣言だけを別ヘッダに分離。
- **`MLModel.compileModel(at:)` が SwiftPM bundle の `.mlpackage` に必要**。`MLModel(contentsOf: url)` の直接読み込みは `.mlmodelc` だけ。Bundle.module は SwiftPM 経由で `.mlpackage` を raw コピーしてくるので、ランタイム compile が必要。

これらは plan に書いていなかった現場 issue。実装中 subagent がそれぞれ自律的に検知してアダプテーション。

---

## Phase D Task 10 進行中: Logic Pro 実機テスト

ここまで実装 + ユニットテストはグリーン。次は実機 AUv3 を Logic Pro で実演奏して、3-layer mix が想定通り鳴るかを確認します (Codex audit Caveat 4)。

ところがここで詰まり中:

```
Logic 警告: 「プラグインを安定して動作させることができません」
DDSP Volume スライダー: 効果なし
Sample Volume スライダー: 効果なし
Physics のみ音が出る
AUv3 process logs: 8192 IOSurfaces created (limit 16384)
                   ANE PowerOn 繰り返し
```

8192 IOSurface は明らかに過大 (1 inference あたり 4 IOSurface とすると 2048 回 inference 実行 = 異常)。あるいは `MLPrediction` が IOSurface を release していない可能性。

検証中:
- Sample silent の原因と DDSP silent の原因は別問題か
- AUv3 sandbox 内での `Bundle.module` 解決失敗の可能性
- 起動時の初期化エラーで一部レイヤーが死んでいる

これは別記事で続報を書きます。

---

## まとめ

今日の 3 つの罠を振り返ると:

**罠 1 (uses_sine ゲート不適合)**: 訓練データの **実態** (Rhodes の基音不在) を見ずに、**理論的に妥当そうな** 診断ゲートを設計してしまった。Codex audit が独立検証で救ってくれた。

**罠 2 (ファイル名衝突)**: A/B 試聴用 WAV を `clip_*/target.wav` という同名で 3 ディレクトリに置いた結果、UI が dedupe → 「全部同じ音」誤判定。何時間もモデルのバグを疑った末に方法論のバグだった。

**罠 3 (物理的にあり得ない仮説)**: デバッグで焦ったとき、「linear gain で pitch が変わる」という基本物理を無視した仮説に走った。ユーザーの即時否定で助かった。

**Codex 4 イテレーション**: plan v1 の 5 blocker → v4 で APPROVED まで進む過程で、architectural quality が大きく上がった。特に v3 の「AU lifecycle で render thread quiescence を保証する」洞察は単独では出なかった。

そして実装はテストレベルでは通過、しかし **実機統合**で別の罠 (IOSurface 過大) にハマっている、というのが現在地です。

今日の commit ログ:

```
467cd44  plan(MethodC Phase D): runtime integration plan v4 + 4-iter Codex pre-impl review
6fa487c  feat(MethodC Phase D): IMethodCRenderer abstract interface (Core, dep-free)
e1726fc  feat(MethodC Phase D): bundle rhodex_method_c.mlpackage into Platform target
e4b9b86  feat(MethodC Phase D): MethodCRunner single-shot CoreML inference
9f6676a  feat(MethodC Phase D): MethodCRenderCache + cache-backed IMethodCRenderer
702c224  feat(MethodC Phase D): MethodCWorker swift loop writing cache via C accessors
7b07120  feat(MethodC Phase D): HybridVoice 4th additive DDSP branch + SR-aware interp
af91c1d  feat(MethodC Phase D): ddspMix AU param (addr 16, range [0,2], def 0.5)
d343249  feat(MethodC Phase D): engine owns MethodC cache+worker+renderer chain
```

次回: 実機統合の続報 + IOSurface 問題の根本解析 + Caveat 4 mixed-output validation の本番。

リポジトリ: [hakaru/TineModeler4](https://github.com/hakaru/TineModeler4)
