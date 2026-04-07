---
title: "iPhoneの中にLA-2Aと1176を入れた話"
emoji: "🎛️"
type: "tech"
topics: ["iOS", "Swift", "DSP", "音声処理", "1Take"]
published: true
---

## 「LA-2A風」って書いてあるのにLA-2Aじゃなかった

1TakeというiOS録音アプリを個人で作ってる。プリセットに「Studio（LA-2A風）」「Studio+（1176風）」って並んでるんだけど、中身はAppleの`kAudioUnitSubType_DynamicsProcessor`だった。ただのデジタルコンプ。

パラメータの数値だけ寄せたって、LA-2Aの「音が前に出てくるあの感じ」は出ない。あれは回路の非線形性から来てるもので、ThresholdとRatioの話じゃない。

ずっと気になってた。v1.5.0でちゃんと作ることにした。

## LA-2Aのフォトセルがやってること

1960年代、Teletronixが作ったオプトコンプレッサー。中にフォトセル（光依存抵抗器）が入ってて、信号が大きくなるとLEDが光る。その光で抵抗値が変わって音量が下がる。

面白いのはフォトセルの戻り方。浅い圧縮だとすぐ戻る。深く圧縮すると、なかなか戻らない。リリースタイムが固定じゃなくて、圧縮の深さで勝手に変わる。

ボーカルに使うと声が前に張り付く感じになるし、アコギのアルペジオは粒が立つ。デジタルコンプで均一に潰した音とは全然違う。回路に真空管が入ってるから、通すだけで偶数次倍音が足される。温かい音の正体はこれ。

## 1176は殴って整えるタイプ

UREI（のちのUniversal Audio）が1967年に出したFETコンプ。LA-2Aとは真逆。

アタックが異常に速い。最速20μs。しかも入力が大きいほど速くなる。ドラムのスネアにかけると「パツン」って前に出る。エレキのリフがザクザク切れる。FETの非線形特性で奇数次倍音が出るから、LA-2Aの温かさとは違うザラッとした色が乗る。

「全押し」（All Buttons In）っていう有名な裏技がある。4つのレシオボタンを全部押し込むと、設計意図にない動作をする。Ratio 20:1くらいの強烈な圧縮に独特の歪み。ドラムのルームマイクでこれやると、あの「ロックのドラムの音」になる。

## iPhoneで動くのかという問題

懐疑的だった。アナログ回路の味をデジタルで出すのは、WavesやUAが何十年もかけてやってきた仕事だ。

結論から言うと、CPU的には全く問題なかった。48kHzで毎秒48,000回のサンプル単位処理を回しても、A15以降なら負荷1%以下。

問題はCPUじゃなくてスレッド制約だった。

## NSLock事件

iOSのオーディオはレンダーコールバック（`internalRenderBlock`）で処理する。リアルタイムスレッドなので、ここでメモリ確保もロック取得もやっちゃだめ。

最初の実装で`NSLock`使った。設定値の読み書きを保護するつもりだったんだけど、ロックの待ちが発生するとオーディオバッファが間に合わなくなってプチプチ鳴る。Codexのコードレビューで指摘されて気づいた。

直し方は単純で、設定値をSwiftの値型（`struct`）にした。メインスレッドから書き換えても、レンダースレッドはコピーを読むだけ。arm64だとstructの代入がアトミックなので、壊れた状態を読むことはない。

## DSPの中身

`BaseCompressorDSP.swift`の140行に全部入った。コアのループ：

```swift
for i in 0..<frameCount {
    let sidechainSample = max(abs(left[i]), abs(right[i]))
    
    if target > state.envelope {
        state.envelope = target + attackCoef * (state.envelope - target)
    } else {
        // LA-2Aは圧縮の深さでリリース速度が変わる
        var releaseCoef = releaseCoefBase
        if model == .opto {
            let grAmount = 1.0 - state.gainReduction
            releaseCoef = exp(-1.0 / ((releaseTime + grAmount * 2.0) * sampleRate))
        }
        state.envelope = target + releaseCoef * (state.envelope - target)
    }
    
    let desiredGR = calculateStaticCurve(inputDB: envDB, settings: settings)
    
    // ハーモニック彩色
    if model == .opto {
        left[i] = tanh(left[i] * 1.2) / 1.2  // 真空管風のソフトクリッピング
    }
    if model == .fet {
        left[i] = left[i] + 0.1 * left[i] * left[i] - 0.01 * left[i] * left[i] * left[i]
    }
}
```

LA-2Aモデルの肝は`releaseCoef`の計算。`grAmount`（圧縮量）が大きいほど係数が大きくなって、リリースが遅くなる。これだけでLA-2Aっぽさの大部分が出る。普通のコンプはリリースタイム固定だから、ここが決定的に違う。

ソフトニーは二次関数で閾値前後をなめらかにつなぐ。Studioプリセットはknee=24dBにしてて、圧縮がかかり始めるのがほとんどわからない。

ウェーブシェーピングは`tanh`（LA-2A）と多項式（1176）の2種。WavesのやつみたいにSPICEで部品単位のシミュレーションしてるわけじゃないけど、録音アプリとしてはこれで十分キャラが出る。

## 2段で使う

1Takeのチェーンにはコンプが2つ入ってる。

```
Input → Gate → EQ → Comp1 → Comp2 → Saturation → M/S → Maximizer
```

Comp1でダイナミクスを整えて、Comp2でLA-2A/1176の色を付ける。スタジオでもこの「整えてから色付け」は定番の構成。

弾き語りを実際に録ると、Comp1だけだとクリーンだけど味気ない。Comp2だけだとムラが出る。両方合わせてちょうどよくなった。

## パラメータ

| パラメータ | 何をするか | 迷ったら |
|-----------|-----------|---------|
| Threshold | 圧縮が始まるレベル | GRメーターが2-3dB振れるところに |
| Ratio | 圧縮の強さ | LA-2A風は3:1-4:1、1176風は8:1-12:1 |
| Knee | 圧縮の始まり方 | 0=パキッと、24=じわっと |
| Attack | 反応速度 | 遅くするとアタックが残る |
| Release | 戻り速度 | LA-2Aモデルは自動なので放置でいい |
| Makeup Gain | 音量補正 | GRメーターの平均と同じだけ上げる |

AttackとReleaseは「数値が小さい＝速い」なので直感と逆になりやすい。

## ハマったこと

`vDSP_maxv`で波形のピーク値を取ったら、負方向に振れてる波形で値がおかしかった。`vDSP_maxmgv`（絶対値の最大）が正解。レビュー指摘されるまで気づかなかった。

RecordingAnalysisにフィールド追加したら既存JSONとの後方互換が壊れた。`decodeIfPresent`でフォールバック入れて対応。

設定画面にローカリゼーションキーの生文字列が表示されてた。`"compressor.hint.threshold"`って画面に出てるのに気づいたときは結構ショックだった。全部ハードコード英語に切り替えた。オーディオ用語は英語のまま使う方が自然。

## これから

v1.5.0はApp Storeに提出した。審査待ち。

DSPコードはSPMパッケージとして分離してあるので、1Take以外でも使える準備はできてる。どこで使うかは考え中。

iPhoneのマイクとプロセッサで、どこまでいけるか試してみたい。

---

1Take: https://github.com/hakaru/1Take
