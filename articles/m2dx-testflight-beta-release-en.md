---
title: "M2DX — DX7-compatible FM Synthesizer with Full MIDI 2.0 UMP Support, Now in Public Beta"
emoji: "🎹"
type: "tech"
topics: ["ios", "midi", "swift", "synthesizer", "testflight"]
published: true
---

**M2DX** is a DX7-compatible FM synthesizer for iOS/iPadOS, built entirely in Pure Swift 6 with full MIDI 2.0 UMP support. The public TestFlight beta is now open.

https://testflight.apple.com/join/BAtGszPw

The synthesis engine is based on [M2DX-Core](https://hakaru.net/M2DX-Core-support/), a standalone Swift library that ports the msfa/Dexed FM core using Int32 Q24 fixed-point arithmetic — the same approach that gives vintage Yamaha hardware its character.

Technical deep-dives (Japanese, English summaries in progress):

https://zenn.dev/hakaru/articles/m2dx-core-dx7-fm-synth-swift

https://zenn.dev/hakaru/articles/m2dx-midi2-korg-property-exchange

## How to Join the Beta

Requires **iOS / iPadOS 18 or later**:

1. Install Apple's **TestFlight** app from the App Store (first time only)
2. Open [testflight.apple.com/join/BAtGszPw](https://testflight.apple.com/join/BAtGszPw) on your iPhone or iPad
3. Tap "Accept" → "Install"

## Current Status — Why a Public Beta

M2DX is **not yet at production instrument quality**. I'm opening it up specifically to collect real-world feedback on the areas below.

### MIDI 2.0 validation is limited

- I haven't been able to test against a wide range of MIDI 2.0 hardware — UMP behavior has only been confirmed on a small set of devices
- macOS CoreMIDI behavior differences across OS versions are not fully mapped
- Third-party MIDI 2.0 profiles beyond KORG's implementation are untested

### Preset sound design

- 32 factory presets are included but haven't been fully validated against vintage DX7 hardware
- User `.syx` bank loading is planned for an upcoming TestFlight build

### FX chain

- The 6-stage FX chain (EQ → Drive → Chorus → Reverb → Stereo → Maximizer) works but parameter ranges and musical tuning still need refinement

If you encounter bugs or unexpected behavior, crash logs are automatically collected via Firebase Crashlytics (see [privacy policy](https://hakaru.net/M2DX-support/privacy)). Reproducible bugs can be reported directly to hirose@hakaru.net.

## Key Features

### Full MIDI 2.0 UMP Support

Native Universal MIDI Packet support with 16-bit velocity (65,536 steps), 32-bit CC, and 32-bit pitch bend. Automatically falls back to MIDI 1.0 with conventional controllers and DAWs.

### DX7-Compatible 6-Operator FM Engine

6 operators × 32 algorithms, implemented in Pure Swift using Int32 Q24 fixed-point arithmetic. 32 factory presets included. The synthesis core is the same engine used in the open-source [M2DX-Core](https://hakaru.net/M2DX-Core-support/) library.

### 16-Voice Polyphony

16-voice polyphony with sustain pedal (CC64), pitch bend (±2 semitones), and Padé approximation tanh soft-clipping to prevent digital harshness.

### 6-Stage FX Chain

EQ → Drive → Chorus → Reverb → Stereo → Maximizer signal path. Every parameter is MIDI Learn-mappable to any CC.

### MIDI-CI Property Exchange

155+ parameters exposed in a hierarchical structure. Compatible hosts and controllers can auto-discover all parameters and manage presets via JSON — no SysEx required. A self-describing instrument.

### Low Latency

Direct rendering via AVAudioSourceNode drives the FM engine inside the CoreAudio render callback. No buffer queuing overhead — effective latency is iOS's IOBufferDuration (~5ms).

## Requirements

- **iOS / iPadOS 18+** (TestFlight)
- macOS support planned for a future release

## FAQ

### The "Install" button is greyed out

TestFlight can take up to 24 hours to reflect a new build after Beta App Review approval. Try again after 24 hours.

### What if it crashes?

Crash logs are automatically collected from build v1.3.1 (build 5) onward via Firebase Crashlytics. Reproducing the crash helps identify the root cause quickly. See the [privacy policy](https://hakaru.net/M2DX-support/privacy) for details.

### Can I load DX7 SysEx presets?

32 factory presets are included. SysEx bank loading (.syx compatibility) is planned for an upcoming TestFlight build.

### Do I need a MIDI 2.0 DAW?

No. M2DX falls back to MIDI 1.0 automatically, so it works with any conventional MIDI controller or DAW. MIDI 2.0-capable setups unlock higher-resolution expression.

### Is AUv3 supported?

Not yet — M2DX is currently a standalone iOS/iPadOS app. AUv3 support is under consideration.

## Links

- Support page: https://hakaru.net/M2DX-support/
- GitHub (open source): https://github.com/hakaru/M2DX
- Synthesis library: [M2DX-Core](https://hakaru.net/M2DX-Core-support/) — Pure Swift DX7 FM engine
- [Privacy Policy](https://hakaru.net/M2DX-support/privacy)

## Contact

Questions, bug reports, and feature requests: hirose@hakaru.net
