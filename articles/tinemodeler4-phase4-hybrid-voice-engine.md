---
title: "AUv3 で Rhodes Piano: 並列 agent + dual reviewer で Phase 4 を閉じた"
emoji: "🎹"
type: "tech"
topics: ["swift", "cpp", "auv3", "coreml", "ddsp"]
published: false
---

:::message
**この記事の対象プロジェクト**

- **TineModeler4 (RhoDex Mk II)** — Apple AUv3 / Standalone 向け Rhodes エレクトリックピアノシンセ。物理モデル (C++17) + stateful DDSP (CoreML) + ETI Roads MKII サンプルのハイブリッド合成
- macOS 15+ / iOS 18+、Swift 6.0 + C++17 Interop、`-mcpu=apple-m1`
- 設計上の縛り: `Shells → Bridge → Platform → Core` の 4 層、`Sources/Core/` は Apple SDK ゼロ
- このシリーズは 4 度目の挑戦。`elepiano / iRhodes / TineModeler1..3` で同じところで全部こけてる
:::

26 commit、4 design doc、4 reviewer cycle、18 tests。Phase 4 を 1 セッションで閉じた。

ここでいう「1 セッション」は Claude Code のセッション 1 本という意味。**並列 agent dispatch** を 7 回、**Copilot + Codex の dual reviewer pipeline** を 4 回まわして、最後は Codex 単独でセカンドオピニオン。手で打ったコミットは 5 つくらいで、残り 21 commit は agent と reviewer fix の往復で勝手に積まれていった。

…と書くと自動化が走りきった話に聞こえるが、実際は *Swift Cxx interop の壁にこってり頭打ち* した記録でもある。最後の最後で Codex が「お前の Bundle.module fix、production AUv3 で動かないぞ」と P1 を投げ込んできた瞬間、4 番目の reviewer cycle がギリギリ間に合った、というのが正直なところ。

## TineModeler4 が何をするやつか

3 行で。

- ETI Roads MKII (96kHz/24bit/73鍵) の **サンプル**をアタックとリリースに置く
- 中間の sustain は **物理モデル** (Tine, Hammer, Pickup, Damper) で鳴らす
- 物理だけだと「らしくない」差分を **stateful DDSP (CoreML)** が学習済みの broadband gain で埋める

これを Apple AUv3 として Logic / GarageBand / AUM に挿せる形にして App Store に出したい、というのが 60 日プランの目標。Phase 4 はその合成パイプライン本体 (HybridVoice + Engine 配線) を埋めるフェーズ。

過去 3 度の iteration (`elepiano / iRhodes / TineModeler1..3`) で全部ここで詰んでる。 *特に* TineModeler3 では `.mlpackage` の GRU recurrence が静かに動いてなかった(`h_in/h_out` を input/output で宣言してしまって state が毎フレームリセットされてた、というやつ)。これに気づいたのが出荷直前で、心が折れた。

なので今回は **回帰テスト先行 + 4 層 enforcement + dual reviewer 必須** で着手した。

## Phase 4a と 4b の地形

ROADMAP.md ではざっくりこう。

```
Phase 4a: HybridVoice + Sample banks
  C1: dr_flac vendor + FlacDecoder
  C2: ManifestReader (手書き JSON parser) + ResourcePath + SpscQueue
  C3: AttackBank / ReleaseBank / MechanicalNoiseBank
  C4: SampleBanksTests (Swift Cxx interop)
  C5: HybridVoice 4 層 state machine + HybridVoiceTests

Phase 4b: Engine + DDSP wiring
  C6: FunctionPtrDDSPRunner (関数ポインタ trampoline)
  C7: TineModeler4Engine + IDDSPRunnerSwiftAdapter
  C8: HybridVoice Sustain に DDSP broadband gain 統合
  C9: docs/DESIGN/SHIP/ROADMAP/AGENTS 更新 + 最終 review
```

最初に詳細仕様を `docs/PHASE_4_PLAN.md` に Plan agent で書き出して、それを起点に 4a (T0-T9) と 4b (C6-C9) を進めた。

設計書のスタイルは `docs/PHASE_4B_C6_DESIGN.md` 以降ほぼ統一されてる:

- §1 何を作るか
- §2-3 API surface
- §4-7 内部実装の詳細
- §8 テスト戦略
- §9 Q1-Q? の確認待ち項目 (✅ or 修正案)
- §10 commit 分割
- §11 リスクと緩和
- §12 次フェーズへの引き継ぎ

agent に投げる前にこの §9 を埋めるのが一番重要だった。**Q を立てた瞬間、その答え通りに agent が書く**。逆に Q を立てそびれた論点は agent が好きに決めて、後で reviewer が「お前そこ自分で決めるなよ」と差し戻してくる。

## 並列 agent dispatch がどれくらい効いたか

数えてみたら 7 回使ってる。

| Phase | dispatch | 並列度 | 完了時間 |
|---|---|---|---|
| 4a T0-T4 (C1 + C2.a + C2.b) | 3 agent | 30 分くらい |
| 4a T5-T7 (3 Banks) | 3 agent | 25 分くらい |
| 4a T8-T9 (C4 + C5) | 2 agent | 20 分くらい |
| 4b C6 | 1 agent | 10 分 |
| 4b C7 (a/b/c シリアル) | 1 agent | 15 分 |
| 4b C8 (a/b シリアル) | 1 agent | 10 分 |
| 4b C9.a docs | 1 agent | 8 分 |

並列に投げたのは合計 8 agent。各 agent は **worktree 隔離** で独立した `.build/` を持つので、`swift build` を並走させても干渉しない。完了したら main worktree で `git cherry-pick` で 1 件ずつ取り込む。

ハマったのは **agent 間で前提が共有されない** こと。最初の 4a C3 で AttackBank agent が「manifest はフラットなファイル名で、実ファイルは `attack/` subdir に居る」と発見して subdir prefix を勝手に足した。これは正しい挙動だったが、後続の ReleaseBank / MechanicalNoiseBank agent には伝わらない。両 agent は smoke test の `loaded=0/73` を見て「あ、subdir が要るんだな」と各自で気づいて `release/` / `mechanical_noises/` を prepend した。 *結果的に* 同じ修正に収束したけど、これは smoke test の期待値を prompt に書いてあったから自力で気づけただけ。**smoke test の期待値は必ず prompt に入れる**、というのがこの phase で学んだ運用ルール。

## Swift Cxx interop が黙って嫌がってくるパターン 5 連発

Phase 4a から 4b にかけて、Swift から C++ Core を叩く経路で 5 種類の壁を踏んだ。順に。

### 1. `= delete` の copy ops があると Swift から見えない

`AttackBank` を `final : public ISampleBank` で書いて、copy をうっかり deleted のまま move を未定義にしてた。Swift Cxx interop が *無言で* class 全体を import 対象から外す。

```swift
// Swift 側
var bank = TineModeler.AttackBank()  // Error: 'AttackBank' is not a member of 'TineModeler'
```

`= delete` のメッセージは一切出ない。Swift が「このクラスは値型として安全に扱えない」と判断して、Cxx interop 経路から黙って消した。

修正は 2 行。

```cpp
AttackBank(AttackBank&&) noexcept = default;
AttackBank& operator=(AttackBank&&) noexcept = default;
```

非コピー可だが move 可、に揃えた瞬間 import される。 *…これに 30 分溶かした*。

### 2. pure-virtual override は Swift から呼べない

`AttackBank::find(SampleQuery)` は `ISampleBank::find` の override。これも Swift Cxx interop が「pure-virtual 経路は安全に呼べない」と判定して unavailable にする。

迂回策: 非仮想の薄い wrapper を足す。

```cpp
// AttackBank.h
__attribute__((swift_attr("import_unsafe")))
SampleSlice findSlice(const SampleQuery& q) const noexcept {
    return find(q);  // 中身は ISampleBank::find と同一
}
```

`import_unsafe` を付けないと `SampleSlice` 内の `const float*` が引っかかってこれも import されない。

### 3. C++ 抽象クラスを Swift から継承するのは不安定

最初の設計では `class CoreMLDDSPRunner: IDDSPRunner` を Swift で書いて inject するつもりだった。実際試したら Swift の compiler が「これは未対応」と諦める。docs にも書いてある現象。

代わりに採用したのが **関数ポインタ trampoline**。

```cpp
// FunctionPtrDDSPRunner.h
class FunctionPtrDDSPRunner final : public IDDSPRunner {
public:
    using PrepareFn = bool (*)(void* ctx, double sr) noexcept;
    using ResetFn   = void (*)(void* ctx, uint32_t voiceID) noexcept;
    using RunFn     = bool (*)(void* ctx, const DDSPInputFrame*,
                               float* ampOut, std::size_t ampCap, std::size_t* ampLen,
                               float* noiseOut, std::size_t noiseCap, std::size_t* noiseLen,
                               float* gateOut) noexcept;

    FunctionPtrDDSPRunner(void* ctx, PrepareFn, ResetFn, RunFn,
                          std::size_t ampCap = 1025,
                          std::size_t noiseCap = 65) noexcept;
    // ... override prepare / resetVoice / runFrame
};
```

Swift 側は `@convention(c)` クロージャに opaque な ctx を渡す。

```swift
static let runTrampoline: @convention(c) (
    UnsafeMutableRawPointer?, UnsafePointer<TineModeler.DDSPInputFrame>?,
    UnsafeMutablePointer<Float>?, Int, UnsafeMutablePointer<Int>?,
    UnsafeMutablePointer<Float>?, Int, UnsafeMutablePointer<Int>?,
    UnsafeMutablePointer<Float>?
) -> Bool = { ctx, inPtr, ampOut, ampCap, ampLen,
              noiseOut, noiseCap, noiseLen, gateOut in
    guard let ctx = ctx, let inPtr = inPtr else { return false }
    let runner = Unmanaged<CoreMLRunner>.fromOpaque(ctx).takeUnretainedValue()
    // CoreMLRunner.run() を try/catch で false に変換、結果を ampOut/noiseOut に書き戻す
    ...
}
```

`Unmanaged.passUnretained(coreMLRunner).toOpaque()` で `void*` を作って、`@convention(c)` クロージャは Swift キャプチャを持てないので ctx 経由でしか CoreMLRunner に触れない。**Engine が CoreMLRunner の強参照を保持する責務**、という暗黙の規約をここで導入する。

### 4. C++ class そのものを Swift から構築できないパターン

`HybridVoice` も同じく Cxx interop に拒否された。理由は構造体の中で `struct Impl*` を opaque pointer で持つ pImpl idiom と、copy/move のオプス組み合わせ。

ここで取った迂回は **C スタイルの `void*` ハンドル経由**。

```cpp
// HybridVoiceHandle.h (C7.a で新設)
void* makeHybridVoiceHandle(double sampleRate,
                            IDDSPRunner* ddsp,
                            uint32_t voiceID,
                            const AttackBank*  atk,
                            const ReleaseBank* rel) noexcept;
void  destroyHybridVoiceHandle(void* handle) noexcept;
void  hybridVoiceNoteOn(void* handle, int midiNote, float velocity) noexcept;
void  hybridVoiceNoteOff(void* handle) noexcept;
void  hybridVoiceRender(void* handle, float* out, int frameCount) noexcept;
bool  hybridVoiceIsActive(void* handle) noexcept;
```

`void*` を返すだけの factory + ハンドル操作の自由関数群。Swift からは `UnsafeMutableRawPointer` として扱う。AttackBank / ReleaseBank も同じパターンで C7.a 内で `makeAttackBankHandle()` / `attackBankUnwrap()` を生やした。

これで Swift 側の Engine が voice pool として 16 個のハンドルを保持して round-robin できる。

### 5. Bundle.module は SwiftPM 専用、XcodeGen では nil

これが Phase 4 最後の壁。最初の C9.c2 で Bundle.module 経由で Resources/samples の path を解決して `TINE_RESOURCES_DIR` に setenv する、という fix を入れた。`swift test` で動いてテストも 16+2 skip で通ったので push した。

…そしたら **Codex に怒られた**。

> When building the AUv3/TestHarness targets from `project.yml`, `Sources/Platform/Apple` is compiled directly and `Resources/samples` is copied into the app/appex bundle, not the SwiftPM `Bundle.module` resource bundle. This lookup therefore does not find the resources in the production packaging path, so `exportSamplesDirEnv()` leaves `ResourcePath` on its cwd fallback and the installed AUv3 can still throw `sampleBanksMissing`.

要点: **XcodeGen が `project.yml` から生成する Xcode project では、Swift Platform sources は直接 AUv3 .appex に static-link されて、Resources/samples は .appex の Contents/Resources に flatten される**。Bundle.module を生やすのは SwiftPM の auto-generated `resource_bundle_accessor.swift` だけで、XcodeGen 経路ではそれが無い。だから `Bundle.module.url(forResource: "samples", ...)` は nil を返す。

つまり俺の修正は *SwiftPM tests でしか効かない* fix だった。AUv3 host (Logic Pro) に挿した瞬間 `sampleBanksMissing` で throw する。

C9.c3 で 3 候補 chain に置き換えた。

```swift
@objc(TineModeler4BundledResourcesMarker)
private final class _BundleMarker: NSObject {}

private static func candidateBundles() -> [Bundle] {
    var bundles: [Bundle] = []
    #if SWIFT_PACKAGE
    bundles.append(Bundle.module)               // SwiftPM tests
    #endif
    bundles.append(Bundle(for: _BundleMarker.self))  // XcodeGen .appex / .app
    bundles.append(Bundle.main)                  // Standalone last resort
    return bundles
}
```

`Bundle(for: _BundleMarker.self)` は 「このクラスのバイナリが入っている bundle」を返す。AUv3 .appex に static-link されると .appex bundle そのものを指してくれる。`@objc` 属性が必須 (Bundle(for:) は AnyClass を要求するけど Swift の生クラスでは通らない)。

`#if SWIFT_PACKAGE` で Bundle.module をガードする理由はもう一段あって、XcodeGen 生成プロジェクトでは `resource_bundle_accessor.swift` が出力されないから、Bundle.module シンボルを使うとそもそもコンパイルが通らない。ここをミスると本番ビルドだけ落ちる。

…これ、3 番目の reviewer cycle (`phase4-complete`) では Copilot も Codex も両方とも見逃した。Codex 単独でセカンドオピニオンを別途走らせて初めて拾った。**reviewer は同じ commit に対して何度走らせても新しい角度を見つけてくれる**、というのがこの phase で得た一番大きい教訓。

## DDSP polyphony bug — 共有バッファの aliasing

`phase4-complete` の reviewer cycle で Codex が出してきた P1 #2 がこれ。

`FunctionPtrDDSPRunner` は **1 セットの `std::vector<float> ampBuf_` / `noiseBuf_`** を持つ。runFrame が返すポインタはこの内部 vector を指す。`DDSPOutputFrame::ampCorrection` 契約上、次の runFrame 呼び出しまでは有効。

最初の C8 実装はこれを `HybridVoice::Impl::lastDDSP` に「次の runFrame まで持ち越す cache」として保存していた。

```cpp
// 旧 (C8.a):
if (s.ddsp->runFrame(in, next)) {
    s.lastDDSP      = next;   // ← next.ampCorrection は runner 内部 buffer
    s.ddspHasResult = true;
}
// render loop:
for (std::size_t k = 0; k < s.lastDDSP.ampLen; ++k) {
    sum += s.lastDDSP.ampCorrection[k];  // ← 共有 buffer を読む
}
```

単一 voice なら問題ない。voice A が runFrame して buffer に書く → ampCorrection を読む → 次の hop まで放置 → 次の runFrame で上書き、というサイクル。

ところが **2 voice 以上同時に鳴ってる polyphonic 環境では破綻する**。voice A の `s.lastDDSP.ampCorrection` がまだ参照されてる最中に、voice B の runFrame が呼ばれて buffer を上書きする。voice A が render で読みに行った時には voice B の correction が乗ってる。**DDSP 出力が voice 順依存で変わる**。

加えて Copilot Critical で別途指摘された hot path 問題: `mean(ampCorrection)` を **per-sample で再計算** していた。1025 × 48000 ≈ 5×10^7 FLOPS、48 kHz レンダーで 50M ops/sec をただの平均値のために回してた。

両方とも **scalar への置換** で同時に解決できる。

```cpp
// 新 (C9.c1):
float cachedBroadbandGain = 1.0f;   // voice 所有の scalar
bool  ddspHasResult       = false;

// Sustain 段、100Hz hop で:
if (s.ddspFrameCounter >= s.ddspFrameInterval) {
    DDSPOutputFrame next{};
    if (s.ddsp->runFrame(in, next)) {
        // runFrame 戻り直後だけが next.ampCorrection の安全な読み出し窓
        if (next.ampCorrection != nullptr && next.ampLen > 0) {
            float sum = 0.0f;
            for (std::size_t k = 0; k < next.ampLen; ++k) {
                sum += next.ampCorrection[k];
            }
            const float mean = sum / static_cast<float>(next.ampLen);
            s.cachedBroadbandGain = clampFloat(mean, 0.1f, 2.0f);  // ← scalar に固める
            s.ddspHasResult       = true;
        }
    }
    s.ddspFrameCounter = 0;
}
++s.ddspFrameCounter;

// render loop:
if (s.ddspHasResult) {
    physicsGain = s.cachedBroadbandGain;   // ← 単発の乗算、ポインタ不参照
}
```

ポインタを voice に持ち越さず、runFrame の戻り直後に **その場で平均を計算して float に固める**。これで:

- voice 間の共有 buffer は次フレームまでに完全消費されるので aliasing しない
- per-sample 50M ops/sec → ほぼゼロ (100Hz hop に集約)
- runFrame failure 時は `cachedBroadbandGain` を hold する (前回値で繋ぐ、クリックしない)

*両方の P1 がたった 1 行の cache 戦略変更で消える*、という気持ちのいい収束。これは Copilot と Codex で別の理由で独立に同じ箇所を指摘してきた、というのが地味に効いた。

## 06_train.py との f0/loudness/midi 正規化合わせ

CoreMLRunner に渡す `f0_norm` / `loudness_norm` / `midi_norm` は **学習時の正規化と完全に一致してないと意味が無い**。当初 C8 設計書で適当に「`log2(hz/27.5) / log2(4186/27.5)` で 0-1 に」と書いてたが、これは推測。

`tools/ddsp-retraining/06_train.py` を grep したら出てきた:

```python
# 06_train.py L143-156
f0_safe       = np.maximum(f0_hz, 1e-6)
f0_norm       = np.log2(f0_safe / 440.0).astype(np.float32)   # ← A4=440 基準 log2
loudness_norm = ((loud_db - loud_db.mean())                    # ← per-clip standardization
                / (loud_db.std() + 1e-6)).astype(np.float32)
midi_norm_scalar = (midi - 52.0) / 36.0                        # ← (midi - 52) / 36
```

3 つ全部違ってた。

f0 は A4 基準 log2、clamp なし。midi 16 (E0, ~21Hz) で `log2(21/440) ≈ -4.4`、midi 88 で +3.58 という生値が入る。midi 正規化は midi 52 (E3) を 0、73 鍵 [16, 88] が概ね [-1, 1] にマップ。loudness は per-clip standardization で、これは **streaming inference 時に再現不可能** (clip の mean/std を計算する手段がリアルタイムにない)。

Phase 4 では loudness_norm = 0.0 (= clip 平均位置) で固定する妥協を取った。 *これは Phase 5 で streaming loudness standardization を入れる時に直す前提*。

```cpp
// HybridVoice.cpp noteOn():
const float hz = midiToHz(midiNote);
s.f0Norm       = std::log2(std::max(hz, 1e-6f) / 440.0f);
s.loudnessNorm = 0.0f;   // Phase 4 妥協、Phase 5 で streaming 化
s.midiNorm     = (static_cast<float>(midiNote) - 52.0f) / 36.0f;
```

設計書のコードコメントに `06_train.py L143-156 と一致` と書いて、Phase 5 着手時に grep し直さなくて済むようにした。

## 4 つの reviewer cycle が拾ったもの

`scripts/review.sh --title <name> --base <ref>` で Copilot CLI と Codex CLI を並列に走らせて、各々の出力を `docs/REVIEW_<name>_{COPILOT,CODEX}.md` に保存する仕組み。Phase 4 では 4 回まわした。

| Cycle | base..head | Critical | Important | P1 | P2 |
|---|---|---|---|---|---|
| phase4a-hybrid-voice | 4b08a24..3ba2c97 | 1 (data race) | 3 | - | - |
| phase4a-postreview | 4b08a24..7ae5ae4 | - | - | - | 2 (Codex) |
| phase4-complete | 4b08a24..abec6a0 | 1 (per-sample hot path) | 4 | 2 (Codex) | - |
| codex postfix (単独) | 6ee0c0f..8c7216e | - | - | 1 (Codex) | - |

合計 5 個の P1/Critical と 6 個の Important が出て、Phase 4 で即修正したのが 8 件、明示的に Phase 5 deferral にしたのが 3 件 (HybridVoice noteOn データレース / Engine.render の `[Float]` 再 alloc / loudness 妥協)。

Copilot と Codex は **見るところが違う**。

- **Codex**: データフロー / 設計整合性 / 実装の論理的破綻 (polyphony aliasing、Bundle.module の packaging path 不整合、duplicate noteOff の release sample 重再生)
- **Copilot**: per-function の細部 / パフォーマンス hotspot / 静的解析的な不整合 (per-sample mean loop、SpscQueue の static_assert 緩さ、ManifestReader の find ループ不足)

同じ事象を両方が違う角度で指摘するパターン(例: polyphony aliasing と per-sample mean loop は別問題に見えて同じ cache 戦略で両方解決) が複数回あった。 *…片方だけだと気づかなかったやつもあるはず*。

特に最後の **codex postfix** が効いた。`phase4-complete` で Copilot と Codex 両方とも見逃した「Bundle.module は SwiftPM only」という P1 を、同じ Codex で再走させたら拾ってきた。reviewer は同じ commit に対して何度走らせても良い、という運用が固まったのはこれが理由。

## 最終状態

```
$ git log --oneline 4b08a24..HEAD | wc -l
26

$ swift build
Build complete! (2.40s)

$ xcrun xctest .build/arm64-apple-macosx/debug/TineModeler4PackageTests.xctest
Executed 18 tests, with 2 tests skipped and 0 failures (0 unexpected) in 1.91 seconds

$ bash tools/check-layer-deps.sh
✓ Layer 1 (Sources/Core/) is clean (no Apple SDK / CoreML deps)
✓ Layer 2 (Sources/Platform/) is clean (no Bridge / Shells deps)
✓ Layer 3 (Sources/Bridge/) is clean (no Shells deps)
PASS: All layer dependencies are correct
```

2 つの skip は `EngineTests.testInitWithDDSPOrSkip` と `EndToEndDDSPTests.testDDSPChangesSustainOutput`。両方とも `swift test` 経路では `.mlpackage` を直接 load できない (xcodebuild の `MLModel.compileModel(at:)` を経由しないとダメ) 制約で意図的に skip にしてる。`xcodebuild test` 環境では走る。SHIP.md の `## Phase 4 で残った運用 TODO` に CI 対応として書いてある。

## Phase 5/6 への引き継ぎ

明示的に scope-out したもの:

- HybridVoice noteOn/noteOff のデータレース → **Phase 5** で `SpscQueue` command queue 化
- Engine.render の `[Float] voiceTmp` heap alloc → **Phase 5** で `UnsafeMutablePointer` に
- `loudness_norm = 0.0` 妥協 → **Phase 5** で streaming standardization
- voice stealing が rrNext 位置 stealing → **Phase 5** で LRU 化
- `.mlpackage` の CI 自動 compile → **Phase 5** か SHIP 直前
- `LoudnessGuard` (DESIGN §7-(2.1)) → **Phase 5** (AUv3 配線後の AB 評価依存)
- `JointExcitationInjector` (DESIGN §7-(5)) → **Phase 6**

reviewer 指摘の中には Phase 4 で「これ Phase 5 だな」と明示的に送ったものがいくつかあって、それは `AGENTS.md` の Session 5 ブロックと `docs/SHIP.md` に書いてある。Phase 5 着手時に最初に読むのは:

1. `docs/SHIP.md` の Phase 4 残運用 TODO
2. `AGENTS.md` Session 5 (Cxx interop 教訓 5 点)
3. `docs/PHASE_4B_C7_DESIGN.md` §11 R5/R6 (RT-safety の Phase 5 work)
4. `docs/REVIEW_phase4-complete_{COPILOT,CODEX}.md` 全部

## 雑感

Phase 4 で「**Plan agent → 詳細設計書 → multi-agent dispatch → multi-reviewer cycle**」のループが回るようになったのが一番大きい。設計書を起こす段階で `§9 Q1-Qn 確認待ち` のセクションを作っておくと、後の reviewer の指摘が「これ §9 で抜けてた論点」に綺麗に収まる。逆に §9 で立て損ねた論点は reviewer に拾われて初めて気づく、というパターンが何度かあった。

…と書くとフロー化された美しい話に聞こえるけど、実際は *Cxx interop の壁を踏み抜くたびに「またか」と思いながら回避策を仕込む* の繰り返し。1 で `= delete` の沼にハマり、2 で pure-virtual の沼にハマり、3 で抽象クラス継承の沼にハマり、4 で pImpl 構築の沼にハマり、5 で Bundle.module の SwiftPM 専用沼にハマった。 *5 個全部、設計書を書く時点では予想してなかった*。

次の Phase 5 は AUv3 / Standalone Shell の配線 + RT-safety hardening。最初の音が AUv3 host から鳴る瞬間まで、もうひと頑張り。
