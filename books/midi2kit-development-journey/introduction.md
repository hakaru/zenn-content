---
title: "はじめに"
---

iPhoneをMIDIコントローラーにするアプリを作りたかった。スライダーでCCを送って、プログラムチェンジで音色を切り替えて、XYパッドで遊ぶ。それだけのアプリ。

ところがMIDI 2.0に対応しようとした瞬間、世界が変わった。

MIDI 1.0は1983年にできた規格で、40年間ずっと「バイト列を送る」だけの素朴なプロトコルだった。MIDI 2.0はそこにProperty Exchange（JSONベースのデバイス間通信）、MIDI-CI（能力の自動交渉）、32ビットの高精度値といった現代的な機能を載せてきた。

仕様書を読むと綺麗に見える。実装すると地獄。

この本は、SimpleMidiControllerという小さなiOSアプリを作る過程でMIDI 2.0の闇にぶつかり、その経験からMIDI2Kitというswiftライブラリを生み出すまでの記録。

https://github.com/hakaru/SimpleMidiController
https://midi2kit.dev/

手元で検証できたのはKORG KeyStageとKORG Module Proだけ。他メーカーのMIDI 2.0実装がどうなってるかは分からない。この本に書いてあることは「KORGの場合はこうだった」であって、MIDI 2.0全体の話ではない点は先に断っておく。
