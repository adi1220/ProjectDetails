# audiosvc — Ultrasound-Based User Presence Detection

## One-Liner

An Android system service that uses the device's own speaker and microphone to detect
human presence in a room via inaudible ultrasound and Doppler shift analysis —
with no additional hardware required.

---

## Problem Statement

Modern smart devices — TVs, tablets, smart displays — need to know if a user is
present in front of them to make intelligent decisions: wake the screen, dim when
unattended, adjust brightness, or trigger ambient modes. Existing approaches have
hard limitations:

| Approach | Limitation |
|---|---|
| Camera-based | Privacy concerns, high power draw, requires line-of-sight |
| PIR (infrared) sensor | Requires dedicated hardware not present on commodity devices |
| Touch / button | Requires active user interaction — useless for passive detection |
| Wi-Fi CSI | Requires router cooperation, not universally available |

`audiosvc` solves this using hardware already present on every Android device —
the speaker and microphone — making it deployable on existing products with a
software update alone.

---

## How It Works

The system emits an inaudible **23 kHz ultrasonic tone** from the loudspeaker while
simultaneously recording the microphone at 48 kHz. A human body in the room reflects
that tone back toward the microphone with a small **Doppler frequency shift** caused
by micro-movements (breathing, posture). The native **RangeFinder C++ engine** analyzes
the power distribution across multiple frequency bins around the tone frequency,
detects this shift, and returns a scalar `distanceChange` metric per frame.

A Java pipeline accumulates 30 such measurements, applies outlier-resistant averaging,
and classifies the environment as `PRESENT` or `ABSENT`. Detection is **gated by an
SPL monitor**, so the ultrasound emitter only activates when ambient sound crosses a
configurable threshold — keeping power consumption minimal in silent rooms.

The system is delivered as a **standalone bound service APK** exposing its full API
cross-process over **AIDL**, so any third-party app can consume presence events without
linking against the library source or `.so`.

---

## Architecture

```
Client App (any package)
    │  AIDL / Binder IPC
    ▼
AudioService (bound Service — standalone APK)
    │
    ├── SPLMonitorThread  → AudioRecord (UNPROCESSED, 48kHz) → RmsSPL → threshold gate
    │
    ├── PlaybackThread    → AudioTrack (LOW_LATENCY, MODE_STREAM) → 23kHz PCM → speaker
    │
    └── RecorderThread    → AudioRecord (UNPROCESSED, 48kHz)
              │
              ▼
         PresenceDetector
           accumulate 12,288 samples → de-interleave L/R → JNI
              │
              ▼
         RangeFinderJNI → librangefinder.so (C++17 / NDK)
           Goertzel multi-bin → Doppler shift → distanceChange float
              │
              ▼
         Queue[30] → outlier filter → PRESENT / ABSENT
              │
              ▼
         BridgeDecisionModule → IPresenceCallback (AIDL oneway)
              │
              ▼
         Client app receives presence state
```

**State machine:** `IDLE → MONITORING → TRIGGERED → RECORDING → MONITORING (loop)`

---

## Engineering Challenges Solved

### 1. Raw Audio Path Without Signal Corruption
Used `AudioRecord` with `MediaRecorder.AudioSource.UNPROCESSED` — the only Android
audio source that bypasses the framework's noise suppression, AGC, and echo
cancellation chain entirely. Any of those effects at 23 kHz would attenuate or
eliminate the reflected ultrasound signal. Fallback to `MIC` source is implemented
for devices that don't expose `UNPROCESSED`.

### 2. Phase-Accurate Tone Playback
Used `AudioTrack` in `MODE_STREAM` with `PERFORMANCE_MODE_LOW_LATENCY` and
`FLAG_LOW_LATENCY`. The tone file is **raw headerless PCM** — not WAV, not MP3 —
because `AudioTrack` has no decoder; any header bytes played as audio would corrupt
the phase reference model the Doppler correlator depends on.

### 3. Lock-Free Audio Buffer on the Hot Path
Implemented a **lock-free SPSC (single-producer / single-consumer) circular buffer**
using monotonically increasing `AtomicInteger` indices and `lazySet` (store-store
fence only — no full `dmb` barrier on ARM). A `BlockingQueue` would stall the
`THREAD_PRIORITY_URGENT_AUDIO` recorder thread and cause `AudioRecord` buffer
overruns. The non-blocking write correctly drops frames rather than blocking.

### 4. Cross-Process API With Automatic Dead-Client Cleanup
The full API surface is exposed over **AIDL** using `RemoteCallbackList<IStateObserver>`
and `RemoteCallbackList<IPresenceCallback>`. `RemoteCallbackList` automatically
unregisters callbacks when client processes die — no manual death-notification
bookkeeping required. Clients only need three `.aidl` stub files — no AAR, no `.so`,
no source.

### 5. Outlier-Resistant Presence Classification
The `calQuePresence` algorithm accumulates 30 distance measurements, removes the
single largest outlier (if `> 15.0`), and averages absolute values. A
**mixed-signs guard** (`pCount > 0 && nCount > 0`) catches environmental noise that
produces incoherent positive/negative Doppler fluctuations and correctly classifies
as `ABSENT` regardless of magnitude.

### 6. Warm-Up Suppression
The first 3 detection cycles are silently discarded. The native IIR filter inside
`RangeFinder` needs time to build a stable acoustic baseline before distance deltas
are meaningful. Clients never see these unreliable early results.

---

## Key Design Decisions

| Decision | Alternative | Why This Choice |
|---|---|---|
| Standalone service APK | AAR embedded per client | One process owns audio hardware — prevents multi-app `AudioRecord` conflicts and shares DSP memory |
| `UNPROCESSED` audio source | `MIC` source | Bypasses noise suppression / AGC that would destroy the 23 kHz reflection signal |
| Raw PCM tone (no WAV header) | WAV / encoded file | `AudioTrack` is a raw PCM pipe — header bytes played as audio corrupt the phase reference |
| Lock-free SPSC buffer | `BlockingQueue` | Non-blocking write prevents stalling the `URGENT_AUDIO` thread |
| `lazySet` on `AtomicInteger` | `set()` | Avoids full ARM `dmb` barrier on every write — safe because SPSC guarantees a single writer |
| Multi-bin Goertzel analysis | Single-frequency power | Catches Doppler shifts in any direction without knowing the shift magnitude in advance |
| SPL gating before ultrasound phase | Always-on emitter | Power efficiency — emitter only runs when ambient sound crosses threshold |
| AIDL `oneway` callbacks | Synchronous callbacks | Fire-and-forget — service never blocks waiting for client to process a callback |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Java 17 (Android), C++17 (NDK) |
| Build system | Gradle 8.4, AGP 8.2.2, CMake 3.22.1 |
| Audio capture | `android.media.AudioRecord` — `UNPROCESSED` source, 48 kHz, 16-bit mono |
| Audio playback | `android.media.AudioTrack` — `MODE_STREAM`, `PERFORMANCE_MODE_LOW_LATENCY` |
| IPC | Android AIDL (Binder) — `IAudioControl`, `IStateObserver`, `IPresenceCallback` |
| Native DSP | C++17 `RangeFinder` via JNI — Goertzel multi-bin Doppler analysis |
| Concurrency | `AtomicReference`, `AtomicInteger`, `RemoteCallbackList`, lock-free ring buffer |
| Min SDK | 26 (Android 8.0 Oreo) / Target SDK 34 (Android 14) |

---

## Interview Pitch — STAR Format (~2 Minutes)

**SITUATION**

Smart devices like TVs and tablets need to know if someone is sitting in front of
them — to wake the screen, dim when unattended, or trigger ambient modes. Common
solutions are cameras (privacy problem), PIR sensors (extra hardware), or Wi-Fi CSI
(needs router support). None of these work on a commodity Android device out of the box.

**TASK**

I built a system that solves this using only the speaker and microphone already
present on any Android device — no additional hardware, no camera, no external sensors.
The goal was a production-ready, cross-process Android service that any app could bind
to and receive real-time human presence events from.

**ACTION**

The approach is ultrasound Doppler detection. I emit an inaudible 23 kHz tone through
the loudspeaker and simultaneously record the microphone. When a person is in the room,
micro-movements like breathing shift the reflected tone by a few Hz — classic Doppler
effect. A native C++ engine running Goertzel frequency analysis detects this shift and
returns a distance-change metric each frame.

On the Java side, I built a pipeline that accumulates 30 such measurements, applies
outlier-resistant averaging with a mixed-signs guard, and classifies the environment
as PRESENT or ABSENT. I gated the entire ultrasound phase behind an SPL monitor —
so the emitter only runs when ambient sound crosses a threshold, saving power in
silent rooms.

The whole system is delivered as a standalone bound service APK exposing its API over
AIDL, so any third-party app gets presence callbacks cross-process with just three stub
files — no AAR, no native library to redistribute.

The hardest engineering problems were: getting raw audio with zero signal processing
using `AudioSource.UNPROCESSED`; writing a lock-free single-producer single-consumer
ring buffer with ARM-safe `lazySet` to avoid stalling the urgent-audio-priority
recorder thread; and designing the `RemoteCallbackList`-based observer pattern so
client process deaths are handled automatically without any manual bookkeeping.

**RESULT**

The result is a full end-to-end presence detection stack — Java pipeline, AIDL
cross-process API, native DSP engine, threading model, and SPL gating — built
entirely by me as a solo developer. The system runs continuously as a background
service with minimal battery impact, delivers sub-10-second detection latency after
warm-up, and is designed to be integrated into any Android app without the client
knowing anything about the audio hardware underneath.

---

## Key Lines to Use in Interview

- *"I chose `UNPROCESSED` audio source specifically to bypass Android's noise suppression — any AGC or echo canceller operating at 23 kHz would destroy the Doppler signal I'm trying to measure."*
- *"The circular buffer uses `lazySet` instead of a full atomic write — on ARM that avoids a `dmb` instruction on every sample, which matters when you're processing 48,000 samples per second."*
- *"The service APK pattern means one process owns the audio hardware — if the library were an AAR embedded in each client, you'd have multiple apps fighting over the `AudioRecord` stream."*
- *"I gated detection behind SPL monitoring — the 23 kHz emitter only spins up when there's ambient sound above a threshold. In a silent room, only a single lightweight `AudioRecord` stream runs."*
