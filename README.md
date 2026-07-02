# VoxrtAsr for iOS

Streaming on-device speech recognition on the **VoxRT** custom inference runtime. NeMo FastConformer (32M parameters), 16 kHz mono in, P&C-aware text out, cache-aware streaming with ~1.1 s chunks.

- Current version: `v0.1.2`
- Minimum iOS: 16.0
- Architectures shipped: `arm64` (iPhone / iPad, NEON-accelerated)
- License: Apache-2.0 (Swift wrapper) · proprietary (compiled runtime, redistribution allowed via this Swift Package)
- Upstream model license: CC-BY-4.0 (NVIDIA NeMo)

---

## What is VoxRT?

VoxRT is a from-scratch inference runtime for on-device speech models. No ONNX Runtime, no PyTorch Mobile, no LiteRT — a custom Rust core sized and tuned for streaming voice workloads on phone-class hardware.

`VoxrtAsr` is the streaming-ASR product on that runtime, alongside [`VoxrtSilero`](https://github.com/VoxRT/voxrt-silero-ios) (VAD) and [`VoxrtWakeWord`](https://github.com/VoxRT/voxrt-wake-word-ios) (always-on wake-phrase detection). All three share the same runtime crate and the same NEON kernel set. The runtime is the product; the models are what it runs.

Commercial custom-phrase wake-word / KWS / domain-specific ASR models built on the same runtime live at [voxrt.com](https://voxrt.com).

## Performance

Measured at ship time, `arm64` device build, post-warmup, RTF = wall-time-per-chunk ÷ chunk audio duration (lower is better):

| Device                | RTF       | per-chunk latency |
| --------------------- | --------- | ----------------- |
| iPhone 13 Pro Max (A15 Bionic) | **0.08–0.10** | ~90 ms / 1.12 s chunk |

At RTF ≈ 0.10 you've got ~90 % of one core free during live transcription. CTC mode is ~15 % cheaper per chunk than RNN-T at the cost of marginally lower accuracy (CTC: 4.895 % WER on LibriSpeech test-clean vs 3.267 % for RNN-T).

## How it compares

VoxrtAsr targets the same on-device streaming-ASR slot as Picovoice Cheetah and the various Whisper.cpp deployments:

| | **VoxrtAsr (RNN-T)** | Picovoice Cheetah | Whisper.cpp base.en |
|---|---|---|---|
| WER on LibriSpeech test-clean | **3.27 %** | 5.4 % (vendor benchmark) | ~4.4 % (upstream Whisper paper) |
| Mobile RTF disclosed | ✅ measured on Snapdragon 662 + iPhone | ❌ desktop Ryzen + Raspberry Pi Zero only | ❌ iPhone 13 demo video only |
| Streaming granularity | 80 ms cache-aware chunk | 590 ms word emission latency | chunked (variable) |
| Model file | ~61 MB fp16 (.vxrt) | 34 MB | 142 MiB tiny.en · 466 MiB small.en |
| Runtime memory | ~150 MB | not published | ~388 MB base.en · ~852 MB small.en |
| License | Apache-2.0 wrapper + proprietary runtime + CC-BY-4.0 weights (NVIDIA NeMo) | Commercial freemium (paid tier opaque) | MIT runtime + OpenAI weight terms (separate due diligence) |

Where we win unambiguously: published WER on LibriSpeech test-clean (3.27 % vs Cheetah 5.4 % vs Whisper base.en ~4.4 %), measured cheap-Android RTF (no other vendor publishes one), and streaming chunk granularity (80 ms vs Cheetah's 590 ms word emission latency). Cheetah has a smaller model file (34 MB vs our 61 MB); we have a more accurate model with disclosed mobile speed.

Full sourced analysis: [voxrt.com](https://voxrt.com).

## Binary footprint

- Swift wrapper source: ~20 KB total
- `VoxrtAsrNative.xcframework` (compressed): ~5 MB device slice
- Streaming model `streaming_medium_pc.vxrt`: ~61 MB fp16 on disk (downloaded separately)
- Native heap at runtime: ~150 MB steady-state (weights expand to f32 for inference; mmap'd zero-copy at load time)

## Install

In Xcode: **File → Add Package Dependencies →** paste:

```
https://github.com/VoxRT/voxrt-asr-ios
```

…and pin to **v0.1.2**.

Or in `Package.swift`:

```swift
dependencies: [
    .package(url: "https://github.com/VoxRT/voxrt-asr-ios.git", from: "0.1.2"),
],
```

## Get the streaming model

The model weights are NOT bundled — you fetch them once from
[`voxrt-asr-models`](https://github.com/VoxRT/voxrt-asr-models/releases/tag/v0.1.2):

```
https://github.com/VoxRT/voxrt-asr-models/releases/download/v0.1.2/streaming_medium_pc.vxrt
```

SHA-256: `0d723e429157a8a8cb58739a1f090574f2f23db311ca7916b43411f5f727c79c`

You decide where it lives. Three common patterns:

- **Bundle in app resources** — drag `streaming_medium_pc.vxrt` into your Xcode project. Works offline from first launch. Adds ~61 MB to your app.
- **Download on first run** — `URLSession` fetch into `FileManager.default.urls(for: .applicationSupportDirectory, ...)`. Smaller App Store binary; needs network at first launch.
- **App Thinning / On-Demand Resources** — Apple's per-asset delivery if you want App Store to host the file.

## Quick start

```swift
import VoxrtAsr

// 1. Resolve the bundled model URL.
guard let modelURL = Bundle.main.url(forResource: "streaming_medium_pc",
                                     withExtension: "vxrt") else {
    fatalError("streaming_medium_pc.vxrt not found in bundle")
}

// 2. Build the engine. `init(modelURL:)` memory-maps the file via
//    `Data(contentsOf:options: .mappedIfSafe)` under the hood — no
//    eager copy. RNN-T decoder is the recommended default
//    (higher accuracy); pass `.ctc` for the ~15 % cheaper head.
let engine = try VoxrtAsrStreamingEngine(modelURL: modelURL)

// (Convenience: same as above for the default bundle + name)
//    let engine = try VoxrtAsrStreamingEngine.fromBundleResource()

// 3. Feed PCM (Float32, 16 kHz, mono, [-1, 1]) blocks of any size.
//    processPcm returns the text emitted during this call — often
//    "" until ~1.12 s of audio has accumulated, then non-empty
//    every chunk boundary.
let delta = try engine.processPcm(pcmFloatArray)
if !delta.isEmpty {
    print("delta: \(delta)")
}

// 4. When the utterance ends, drain the tail.
let tail = try engine.stop()
```

`engine.processPcm` / `stop` / `reset` are **synchronous and stateful** — same shape as `VoxrtSileroVadEngine.processPcm` in the companion VAD library. The engine does NOT own a worker thread. You drive it from your own capture / IO thread.

## Live microphone example

The canonical streaming pattern — capture-thread owns the `AVAudioEngine` tap, engine is just a stateful function.

```swift
import AVFAudio
import VoxrtAsr

// NOTE: tap callbacks fire on a real-time audio thread. Pre-size
// + reuse buffers; do not allocate per callback in production.

let session = AVAudioSession.sharedInstance()
try session.setCategory(.playAndRecord, mode: .measurement)
try session.setActive(true)

let audioEngine = AVAudioEngine()
let input = audioEngine.inputNode
let hwFormat = input.outputFormat(forBus: 0)        // 44.1 / 48 kHz
let voxrtFormat = AVAudioFormat(                    // engine target
    commonFormat: .pcmFormatFloat32,
    sampleRate: 16_000,
    channels: 1,
    interleaved: true,
)!
let converter = AVAudioConverter(from: hwFormat, to: voxrtFormat)!

guard let modelURL = Bundle.main.url(forResource: "streaming_medium_pc",
                                     withExtension: "vxrt") else { fatalError() }
let asr = try VoxrtAsrStreamingEngine(modelURL: modelURL)

// 3200 samples @ 16 kHz = 200 ms — the recommended push block.
let scratchCapacity: AVAudioFrameCount = 3_200
let voxrtBuf = AVAudioPCMBuffer(pcmFormat: voxrtFormat,
                                 frameCapacity: scratchCapacity)!
var cumulativeTranscript = ""

input.installTap(
    onBus: 0,
    bufferSize: 4_096,
    format: hwFormat
) { hwBuf, _ in
    voxrtBuf.frameLength = 0
    var error: NSError?
    converter.convert(to: voxrtBuf, error: &error) { _, status in
        status.pointee = .haveData
        return hwBuf
    }
    if error != nil { return }
    guard let f32 = voxrtBuf.floatChannelData?[0] else { return }
    let n = Int(voxrtBuf.frameLength)
    let samples = Array(UnsafeBufferPointer(start: f32, count: n))

    do {
        let delta = try asr.processPcm(samples)
        if !delta.isEmpty {
            DispatchQueue.main.async {
                cumulativeTranscript += delta
                // update UI with cumulativeTranscript
            }
        }
    } catch {
        // surface error to UI
    }
}

try audioEngine.start()
// ... later, on stop:
audioEngine.stop()
input.removeTap(onBus: 0)
let tail = try asr.stop()
if !tail.isEmpty {
    DispatchQueue.main.async { cumulativeTranscript += tail }
}
```

## Audio contract

- **Sample rate:** 16 000 Hz. **No automatic resampling.** Phone mic hardware delivers 44.1 / 48 kHz to `AVAudioEngine`; convert via `AVAudioConverter` to 16 kHz Float32 mono before feeding `processPcm`. Feeding the wrong rate is the #1 source of "transcript is gibberish" bugs.
- **Sample format:** `[Float]` PCM in `[-1, 1]`, mono, native endian.
- **Buffer size:** any. The engine internally accumulates to its steady-state chunk size (17 920 samples ≈ 1.12 s) and emits text every chunk.
- **Latency:** one chunk (~1.12 s) of inherent buffering. Output text becomes available chunk-by-chunk from `processPcm` return values.

## Threading

- The engine is a **synchronous, stateful function**. It does NOT own a queue. Each `processPcm` call blocks on the calling thread for the duration of the inference work — typically the `AVAudioEngine` tap thread for live mic. Marshal text deltas back to UI via `DispatchQueue.main.async` (or your concurrency framework of choice).
- One instance is **single-thread-at-a-time**. Serialise `processPcm` / `stop` / `reset` against each other on a given instance.
- One engine instance handles a stream of utterances. Between utterances, call `engine.reset()` to zero the K/V cache + LSTM state without paying weight-load cost again.

## Permissions

iOS requires a usage-description string for microphone access. Add to your **app**'s `Info.plist`:

```xml
<key>NSMicrophoneUsageDescription</key>
<string>Used for on-device speech recognition.</string>
```

`AVAudioSession.requestRecordPermission(...)` triggers the user prompt the first time mic capture is initiated. Without the `Info.plist` key the app crashes with a privacy-violation exception on first request.

## Decoder selection

**Recommended: RNN-T** — higher accuracy, modest extra cost. This is the SDK default; you only need to pass an explicit decoder constant if you specifically want CTC.

| Decoder | Constant | WER on test-clean (500-utterance subset) | Per-chunk cost | When to use |
| ------- | -------- | ----------------------: | --------------: | ----------- |
| **RNN-T** ★ | `.rnnt` | **3.267 %** | ~50 ms | **Recommended default.** Higher accuracy. LSTM state survives chunk boundaries. |
| CTC | `.ctc` | 4.895 % | ~5 ms | Battery-constrained long sessions, or background transcription where the ~1.6 % WER hit is acceptable. |

Both decoders run the same Conformer encoder; the head is selected at session-create time. **TDT is not supported** on streaming-medium-pc (no duration head) — passing `.tdt` fails the session creation.

## Architectures roadmap

`v0.1.2` ships only `arm64` for physical devices, NEON-optimized. Simulator slices (arm64-sim + x86_64) are included for build convenience but are not part of the supported production target list.

| Target                       | Status     |
| ---------------------------- | ---------- |
| iOS arm64 (device)           | ✅ Shipped  |
| iOS arm64 simulator          | ✅ Shipped (build-time only) |
| iOS x86_64 simulator         | ✅ Shipped (build-time only) |
| macOS arm64                  | 🟡 Coming soon |
| macOS x86_64 (AVX)           | 🟡 Coming soon |
| visionOS / tvOS / watchOS    | ☁️ On request |

## Project layout

```
voxrt-asr-ios/
├── Package.swift                # SPM manifest (binaryTarget URL + checksum)
├── README.md                    # this file
├── LICENSE                      # Swift wrapper terms (Apache-2.0)
├── LICENSE-BINARY               # compiled runtime terms (proprietary)
└── Sources/
    └── VoxrtAsr/
        └── VoxrtAsr.swift       # Swift wrapper (open, Apache-2.0)
```

The compiled `VoxrtAsrNative.xcframework` lives on the GitHub Release page for this tag, downloaded by SwiftPM via the URL+checksum pinned in `Package.swift`.

## License

- The Swift wrapper (`Sources/VoxrtAsr/`) is licensed under **Apache-2.0**. See [`LICENSE`](LICENSE).
- The compiled `VoxrtAsrNative.xcframework` is proprietary VoxRT runtime code owned by Elephant Enterprises LLC, redistributable as part of this unmodified Swift Package. See [`LICENSE-BINARY`](LICENSE-BINARY).
- The streaming-medium-pc model weights are derived from [`nvidia/stt_en_fastconformer_hybrid_medium_streaming_80ms_pc`](https://huggingface.co/nvidia/stt_en_fastconformer_hybrid_medium_streaming_80ms_pc), released under **CC-BY-4.0**. Attribution travels with the model on the [voxrt-asr-models](https://github.com/VoxRT/voxrt-asr-models) repo.
- Commercial integration / custom-model packaging questions: help@voxrt.com.

## Links

- VoxRT runtime + commercial models: [voxrt.com](https://voxrt.com)
- Android counterpart: [voxrt-asr-android](https://github.com/VoxRT/voxrt-asr-android)
- ASR model weights & versions: [voxrt-asr-models](https://github.com/VoxRT/voxrt-asr-models)
- VAD companion: [voxrt-silero-ios](https://github.com/VoxRT/voxrt-silero-ios)
- Wake-word companion (iOS): [voxrt-wake-word-ios](https://github.com/VoxRT/voxrt-wake-word-ios) · [models](https://github.com/VoxRT/voxrt-wake-word-models)
- Bugs / questions: open an issue on this repo
