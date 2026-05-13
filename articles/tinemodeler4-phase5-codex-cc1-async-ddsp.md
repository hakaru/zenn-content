---
title: "Phase 4 で素通しした DDSP の RT 設計、Codex に CC1 で全否定された"
emoji: "🎹"
type: "tech"
topics: ["swift", "cpp", "auv3", "coreml", "ddsp"]
published: false
---

:::message
**この記事の対象プロジェクト**

- **TineModeler4 (RhoDex Mk II)** — Apple AUv3 / Standalone 向け Rhodes エレクトリックピアノシンセ。物理モデル (C++17) + stateful DDSP (CoreML) + ETI Roads MKII サンプルのハイブリッド合成
- macOS 15+ / iOS 18+、Swift 6.0 + C++17 Interop
- 設計上の縛り: `Shells → Bridge → Platform → Core` の 4 層、`Sources/Core/` は Apple SDK ゼロ
- 前回の記事: [AUv3 で Rhodes Piano: 並列 agent + dual reviewer で Phase 4 を閉じた](./tinemodeler4-phase4-hybrid-voice-engine)
:::

Phase 5 (Shell & UI) に着手する前に計画書のレビューを 3 段階かけた。**セルフレビュー → 4 並列 agent による Documentation Discovery → Codex 第三者レビュー**、の 3 回。

各段で別種の Critical が出てきて、計画書は v1 → v2 → v3 と 3 回書き直した。とくに Codex が最後に投げてきた **CC1 (Critical 1 番)** は、Phase 4 で素通しした設計上の致命傷を「お前 AU render thread で CoreML 推論まわしてるけど Logic で確実に underrun するぞ」と一発で指摘されたやつで…  *いや、それ動いてたじゃん?* と最初は思った。

動いてたのは「テストが短くて & 単発ノートで & 再生してなかった」という 3 条件が揃っていたからで、Phase 5 で AU host に挿した瞬間に死ぬパターンだった。

この記事はその CC1 を Phase 5 5a-RT サブフェーズとして 2 commit で潰した記録。

## レビュー 3 段階の中身

| 段階 | 出てきた指摘 | 計画書改訂 |
|---|---|---|
| セルフ | FX 配線忘れ / ペダル CC64 完全欠落 / maxFrames 拘束 | v1 → v2 |
| 4 並列 Discovery | `factoryFunction` キーが AUv3 で使えない / `Atomic<Float>` 非対応 / `implementorValueProvider` 無限再帰 | v2 + Discovery doc 4 件 |
| Codex 第三者 | CC1〜CC5 (5 つの Critical) | v2 → v3 |

セルフレビューと Discovery で見つけた分は実装の手前で潰せた話なのでまあいい。問題は Codex が **「自分でレビューしてもまだ穴がある」を実証してきた**こと。

5 つの Critical のうち、本記事の主役は CC1。残り 4 つは Phase 5a / 5c で潰す予定なので別記事になる。

## Codex CC1: DDSP/CoreML が AU render thread で同期実行されてる

要約:

```
[AU render thread] → HybridVoice::renderBlock → 100Hz hop で
    s.ddsp->runFrame(in, out)    ← 同期、CoreML 推論完了まで block
    → 約 42.6ms (STFT 2048 / 48kHz)
```

Logic は典型的に 256 frame buffer = 約 5.3ms ぶんを render しに来る。**1 回の buffer 中に 42ms の推論を待たされたら、その間に DAW のオーディオエンジンは音を出すサンプルが無くなる**。typical underrun。

Codex はこれを「review session 開始 30 秒で」発見していた。コメント原文:

> DDSP/CoreML が AU render thread で同期実行される。Phase 5 task から RT-safe worker 化が落ちている。

…完全に落ちてました。

## なぜ Phase 4 のテストでは引っかからなかったか

Phase 4 で実装した sync `runFrame` 経路には、`Tests/AudioRegressionTests/EndToEndDDSPTests.swift` で **DDSP 有効 vs 無効の出力差**を測る回帰テストが入ってる。これが PASS していたので「動いてる」と思っていた。

なんで underrun を検出できなかったか、改めて考えるとこれ。

- テストは `swift test` 上で render を **synchronously に走らせる** だけ。AU host に挿してないので buffer timing 制約がない (= 1 秒ぶんを 10 秒かけて render しても regression test は通る)
- 単発ノート 1 個 × 1 秒。CoreML 推論の累積遅延を実時間と比較する圧力がそもそもない
- Standalone も AUv3 もまだ配線してないので、実際の DAW で再生したことがない

つまり「動いてる」が「設計通り動いてる」じゃない、というのを *また* やったわけ。TineModeler3 で `coreml_update_state` op が消えていた件 (`||h(t) - h(t-1)||_2 = 0`) と同じ構造で、ただし症状の現れ方が違うだけ。

## Insight: 1025 → 1 のダウンサイズで設計が単純化した

Worker pool 化する前に、HybridVoice の sync 経路を細かく見直していて気付いたことがある。

```cpp
// HybridVoice.cpp:418-424 (sync 経路)
if (next.ampCorrection != nullptr && next.ampLen > 0) {
    float sum = 0.0f;
    for (std::size_t k = 0; k < next.ampLen; ++k) {
        sum += next.ampCorrection[k];
    }
    const float mean = sum / static_cast<float>(next.ampLen);
    s.cachedBroadbandGain = clampFloat(mean, 0.1f, 2.0f);  // ← ここ
    s.ddspHasResult       = true;
}
```

DDSP 推論が返してくる `ampCorrection[1025]` 配列を、HybridVoice は **mean に reduce して scalar 1 個**として使ってる (Phase 4b C8: broadband gain として physics の出力にかけるだけ)。

これは Codex P1 #2 + Copilot Critical の review fix で既に入っていた制約で、**voice ごとに float 1 つの atomic で同期するだけで render thread の DDSP 依存が表現できる**ということを意味する。

つまり worker pool ↔ render thread 間で渡すデータは:
- ✅ 入力: voice ID + (f0, loudness, midi) の 3 float = 必要な情報
- ✅ 出力: voice ID + broadband gain の 1 float = 1025 → 1 にダウンサイズ済

これに気づいたら設計が劇的に単純になった。`std::atomic<float>` か `ManagedAtomic<UInt32>` (bit pattern) を 16 voice ぶん用意するだけ。1025 個の浮動小数点配列を ferry する必要なし。

…まあ、こうやって書くと自明だけど、これに気づくまでは「STFT バッファをまるごと SPSC queue で投げ合う」みたいな複雑な設計を考えてた。*計画書 v2 の §5a-RT 第 1 案* がそれ。

## 実装: 2 つの commit に分けた

### Pass 1: async DDSP インフラ (uninstalled)

[commit de64eb8](https://github.com/hakaru/TineModeler4/commit/de64eb8)。HybridVoice / Engine は触らず、新規ファイルだけ。

```
Sources/Core/DDSPRunner/
├── IAsyncDDSPRunner.h               ← 抽象 interface (Core, no Apple SDK)
└── FunctionPtrAsyncDDSPRunner.{h,cpp} ← 関数ポインタトランポリン

Sources/Platform/Apple/CoreMLBackend/
└── AsyncCoreMLRunnerPool.swift      ← DispatchQueue + ManagedAtomic
```

`IAsyncDDSPRunner` の API は 4 メソッドだけ:

```cpp
class IAsyncDDSPRunner {
public:
    virtual bool prepare(double sampleRate) noexcept = 0;
    virtual void resetVoice(uint32_t voiceID) noexcept = 0;
    // RT-hot: non-blocking enqueue
    virtual void requestFrame(const DDSPInputFrame& in) noexcept = 0;
    // RT-hot: atomic read scalar (caller fallback if false)
    virtual bool latestBroadbandGain(uint32_t voiceID, float& outGain) noexcept = 0;
};
```

`AsyncCoreMLRunnerPool` (Swift) の RT パス:

```swift
public func requestFrame(voiceID: UInt32,
                         f0Norm: Float,
                         loudnessNorm: Float,
                         midiNorm: Float,
                         generation: UInt32) {
    let i = Int(voiceID)
    guard i < Self.maxVoices else { return }
    let slot = pending[i]
    slot.f0.store(f0Norm.bitPattern, ordering: .relaxed)
    slot.loudness.store(loudnessNorm.bitPattern, ordering: .relaxed)
    slot.midi.store(midiNorm.bitPattern, ordering: .relaxed)
    // generation token: high 32 bits = HybridVoice generation
    //                   low  32 bits = monotonic counter
    let prev = slot.request.load(ordering: .relaxed)
    let prevCounter = UInt32(prev & 0xFFFF_FFFF)
    let newToken = (UInt64(generation) << 32) | UInt64(prevCounter &+ 1)
    slot.request.store(newToken, ordering: .releasing)
    semaphore.signal()  // worker を起こす
}
```

drop-oldest セマンティクス。worker が追いつかないなら最新値で **上書きする** だけ。render thread は store + signal の 2 アトミック操作だけで戻る。

Worker thread 側:

```swift
private func drainAndProcess() {
    guard let runner = self.runner else { return }
    for i in 0..<Self.maxVoices {
        let slot = pending[i]
        let token = slot.request.load(ordering: .acquiring)
        if token == lastProcessedGen[i] { continue }  // 既に処理済
        lastProcessedGen[i] = token
        let generation = UInt32(token >> 32)
        let f0       = Float(bitPattern: slot.f0.load(ordering: .relaxed))
        let loudness = Float(bitPattern: slot.loudness.load(ordering: .relaxed))
        let midi     = Float(bitPattern: slot.midi.load(ordering: .relaxed))
        do {
            let out = try runner.run(forVoiceID: i, generation: generation,
                                     f0Norm: f0, loudnessNorm: loudness,
                                     midiNorm: midi)
            // mean(ampCorrection) → broadband gain (← ここで 1025 → 1 reduce)
            let amp = out.ampCorrection
            guard !amp.isEmpty else { continue }
            var sum: Float = 0
            for v in amp { sum += v }
            let gain = sum / Float(amp.count)
            latest[i].gain.store(gain.bitPattern, ordering: .releasing)
            latest[i].hasResult.store(true, ordering: .releasing)
        } catch {
            // 推論失敗: latest を更新せずスキップ (前回値を維持)
        }
    }
}
```

`mean(ampCorrection[1025])` の `for` ループが、Phase 4 では **render thread の hot path で 100 回 / 秒 × 1025 ops = 102,500 ops/s** だったやつ。これが worker thread に移動して、render thread は scalar 1 個読むだけになる。

### Pass 2: HybridVoice / Engine 配線

[commit ff7785f](https://github.com/hakaru/TineModeler4/commit/ff7785f)。

HybridVoice の ctor signature を `IDDSPRunner*` → `IAsyncDDSPRunner*` に変更して、Sustain stage の sync `runFrame()` を非同期に置換。

```cpp
// HybridVoice.cpp Sustain stage (after pass 2)
if (s.ddspFrameCounter >= s.ddspFrameInterval) {
    // 1. Pull: 直前 hop の結果 (worker 完了済みなら更新される)
    float latestGain = 0.0f;
    if (s.ddsp->latestBroadbandGain(s.voiceID, latestGain)) {
        s.cachedBroadbandGain = clampFloat(latestGain, 0.1f, 2.0f);
        s.ddspHasResult       = true;
    }
    // 2. Request: 次 hop ぶんを enqueue (non-blocking)
    const DDSPInputFrame in{ s.f0Norm, s.loudnessNorm, s.midiNorm,
                              s.voiceID, s.generation };
    s.ddsp->requestFrame(in);
    s.ddspFrameCounter = 0;
}
++s.ddspFrameCounter;

if (s.ddspHasResult) physicsGain = s.cachedBroadbandGain;
```

Worker が間に合っていなければ `latestBroadbandGain` は false を返して、前回の `cachedBroadbandGain` を hold する。**初回 Sustain 突入時 (worker 未応答) は physics-only fallback** という graceful degradation がそのまま自然に効く。

## C++ ↔ Swift 関数ポインタトランポリン

`IAsyncDDSPRunner` の実装本体は Swift 側 (`AsyncCoreMLRunnerPool`) にあって、それを C++ の `HybridVoice` から呼びたい。Apple SDK は `Sources/Core/` には入れられない縛り (4 層ルール) があるので、Phase 4b C6 で確立した **関数ポインタトランポリン** パターンを再利用する。

```swift
// Sources/Bridge/AsyncIDDSPRunnerSwiftAdapter.swift
internal enum AsyncIDDSPRunnerSwiftAdapter {
    static let requestTrampoline: @convention(c) (
        UnsafeMutableRawPointer?,   // ctx (= Unmanaged AsyncCoreMLRunnerPool)
        Float, Float, Float, UInt32, UInt32
    ) -> Void = { ctx, f0, loudness, midi, voiceID, generation in
        guard let ctx = ctx else { return }
        let pool = Unmanaged<AsyncCoreMLRunnerPool>.fromOpaque(ctx)
                                                   .takeUnretainedValue()
        pool.requestFrame(voiceID: voiceID,
                          f0Norm: f0,
                          loudnessNorm: loudness,
                          midiNorm: midi,
                          generation: generation)
    }
    // prepare / reset / latest も同様に 4 つ
}
```

C++ 側の `FunctionPtrAsyncDDSPRunner` が 4 個の `void* fn` を受け取って、IAsyncDDSPRunner 経由の呼び出しを `reinterpret_cast` で関数ポインタ型に戻して dispatch する (`noexcept` 付き typedef を Swift Cxx interop が読まない問題は Phase 4b C6 と同じ)。

## TSAN sweep が一発 PASS した話

実装が終わった時点で `swift test --sanitize=thread` をかけた。

```
Test Suite 'TineModeler4PackageTests.xctest' passed at 2026-05-13 14:25:05.856.
Test Suite 'All tests' passed at 2026-05-13 14:25:05.856.
```

race 警告ゼロ。これは `ManagedAtomic` + `DispatchSemaphore.signal` (wait-free) + per-voice slot (cross-voice 競合なし) の lock-free 設計が、最初から狙い通り動いたという意味で。

…ただ正直、TSAN が *何も言わない* というのは「TSAN が見えてない race がある」可能性もそれなりに残るので、Phase 5f で sanitizer 全件 (TSAN/ASan/UBSan) + Instruments プロファイリングまで通した時点で本当に clean、と判定する予定。

## エンジン側の retrofit でテストが落ちた件

Pass 2 で Engine を `coreMLRunner` から `asyncPool` に置き換えた直後、`EndToEndDDSPTests.testDDSPChangesSustainOutput` が 1 件落ちた。

```
XCTAssertGreaterThan failed: ("0.0") is not greater than ("0.00022225...")
- DDSP enabled should perturb sustain by more than 0.1% relative RMS
```

diff = 0.0、つまり **DDSP 有効 / 無効で出力が完全に一致**。これ何が起きてるか:

- 旧 sync 経路: `CoreMLRunner.init` が throw → Engine init で catch → `coreMLRunner = nil` → `ddspEnabled = false` → テストは `XCTSkipUnless` で skip
- 新 async 経路: `AsyncCoreMLRunnerPool.init` は `try? CoreMLRunner(...)` で **常に成功** (pool 自体は失敗しない) → `ddspRunnerHandle != nil` → `ddspEnabled = true` のまま → でも実際の推論経路は壊れているので diff = 0

これが**「graceful degrade のしすぎ」**による回帰。

Fix は `fnPtrAsyncPrepare` の戻り値で handle を引き戻すこと。

```swift
if TineModeler.fnPtrAsyncPrepare(handle, sampleRate) {
    self.ddspRunnerHandle = handle
} else {
    // CoreML model load 失敗 → handle 捨てて ddspEnabled = false に倒す
    TineModeler.destroyFunctionPtrAsyncDDSPRunner(handle)
    self.ddspRunnerHandle = nil
    self.asyncPool?.stop()
    self.asyncPool = nil
}
```

これで sync 経路と同じ「DDSP が立ち上がらない環境では XCTSkip」というセマンティクスに復旧。テスト 23/23 (2 skip 既知) で Phase 4 完了時と同じ状態に戻った。

…細かいけど、こういう「graceful degrade を入れたら別のレイヤで隠蔽が起きてテストの skip 条件が成立しなくなる」のは Phase 4b 4 回めの reviewer cycle でも同種のパターンに遭遇したやつ。後段の整合性まで見ないと「とりあえず動くやつ」が積み上がる。

## 学び

3 つ。

- **「動いてる」は「設計通り動いてる」を保証しない**、はもう何度目か数えるのが嫌になるレベル
- **データを reduce してから ferry する** insight (1025 → 1) は、async 設計の規模を一桁変えた。最初に sync 実装をちゃんと書いておいたから境界の絞り込みが効いた、とも言える
- **第三者レビューは自分のセルフレビューよりも*別の角度*から穴を炙る**。Codex の CC1 は self / Discovery のどちらも掴めなかった。レビューの「層」を多重化すること自体に意味がある

次は Phase 5a (AUv3 配線本体)。Codex の残り 4 Critical (CC2-CC5) を一気に潰しに行く。
