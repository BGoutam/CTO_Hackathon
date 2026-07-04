# Performance Analysis Report: Client-Side VAD & ASR

**Challenge 2 — Deliverable (Performance Analysis)** · Answers **Task 4 (Performance Analysis)**

> **What this document covers:** where latency comes from, which stage dominates, the bottlenecks, and concrete optimizations. The PoC (`vad_asr_poc.html`) instruments and displays the two latencies you can actually measure in a browser: **speech-detection latency** and **end-of-speech → transcript latency**.

---

## 0. Bottom Line

End-to-end responsiveness is dominated by **one tunable knob — the silence timeout (~800 ms)** — not by model inference. VAD inference is ~1–8 ms and effectively free; the recognizer is the second-largest cost. Optimize the silence timeout (make it adaptive) and the perceived speed problem is largely solved.

---

## 1. End-to-End Latency Breakdown

We split latency into two user-visible measurements:

- **Speech-detection latency** — mic captures speech → system registers "user is speaking" (drives the "Listening" indicator).
- **End-of-speech latency** — user stops talking → final transcript appears (the number users *feel* as "slow").

$$\text{Latency}_{E2E} = \text{Latency}_{Chunk} + \text{Latency}_{VAD} + \text{Latency}_{Silence} + \text{Latency}_{ASR}$$

| Stage | Latency | Source & Notes |
| :--- | :--- | :--- |
| **Audio framing** | ~32 ms | Time to collect one 512-sample Silero frame at 16 kHz. |
| **VAD inference** | 1–8 ms | ONNX Runtime Web scoring one frame. Depends on WASM SIMD/threads. Effectively negligible. |
| **Start debounce** | ~64 ms | 2 consecutive frames required before "Speech Start" (false-trigger rejection). Affects *detection*, not end-of-speech. |
| **Silence timeout** | 600–1000 ms | The wait before declaring the turn over. **The single largest contributor** to end-of-speech latency. |
| **ASR processing** | 0–1500 ms | **Web Speech API** (cloud): ~100–300 ms, streams incrementally + network RTT. <br>**Local Whisper ONNX**: ~400–1500 ms, depends on model size and WASM/WebGPU acceleration. |
| **End-of-speech total** | **~750 ms – 2600 ms** | From end of speech to final transcript. |

> **Reading the table:** everything except the silence timeout and ASR is in the tens of milliseconds. The user's perception of "fast" or "slow" is set almost entirely by how long we wait in silence before finalizing, plus recognizer time.

---

## 2. Key Bottlenecks

### A. The Silence-Timeout Dilemma *(the dominant trade-off)*
- **Too short (<500 ms):** speakers who pause between clauses get cut off mid-sentence.
- **Too long (>1000 ms):** the system feels sluggish and unresponsive.
- There is **no single correct value** — hence the adaptive strategy in §3A.

### B. Audio Resampling on the Main Thread
- The mic is typically 44.1/48 kHz; Silero needs 16 kHz, so we resample every block.
- Legacy **`ScriptProcessorNode` runs on the main UI thread** → visible stutter under load.
- **`AudioWorkletNode`** moves this off-thread and fixes the jank, but requires shipping a separate worklet `.js` file — more deployment complexity for a single-file PoC.

### C. Model Loading (local Whisper path)
- A ~78 MB `whisper-tiny` download is slow on poor networks and lengthens first-load.
- Large WASM heaps can crash tabs on low-end mobile.
- **Mitigation:** cache weights in the Cache API after first load (offline thereafter); prefer the smallest acceptable model; lazy-load ASR only when first needed.

> **Note:** Silero VAD itself is *not* a bottleneck — the model is ~1.3–2.3 MB and inference is single-digit milliseconds.

---

## 3. Recommended Optimizations

### A. Adaptive Silence Timeout *(highest impact)*
Replace the flat 800 ms with a context-sensitive window:
- **Long, structurally complete utterance** → shorten to **~500 ms** (they're likely done).
- **Single short word** ("Yes", "No") → extend to **~1000 ms** (give them room to elaborate).
This directly attacks the largest latency contributor without increasing clipping.

### B. Hardware Acceleration in ONNX Runtime Web
- Enable **WASM SIMD + multi-threading** (`ort.env.wasm.simd = true`, `numThreads = hardwareConcurrency`). For the tiny VAD model, **WASM is the right backend** (WebGL/WebGPU overhead isn't worth it for a model this small).
- Feature-detect SIMD before relying on it:
  ```javascript
  const hasSimd = await WebAssembly.validate(new Uint8Array([
    0,97,115,109,1,0,0,0,1,5,1,96,0,1,123,3,2,1,0,10,9,1,7,0,65,0,253,15,11
  ]));
  ```

### C. Hybrid ASR Routing
- **Client-side-first:** default to local **Whisper (transformers.js)** to satisfy the no-server requirement and work offline.
- **Fallback:** if the device is too weak to run Whisper acceptably, and the user accepts cloud STT, route to the **Web Speech API** (lower footprint, faster, but cloud). Feature-detect and let the user choose.

### D. Off-Thread Audio (production)
- Migrate resampling + framing to an **AudioWorklet** so the UI thread stays smooth even during heavy rendering.

---

## 4. What the PoC Actually Measures

The demo instruments and displays, per utterance:
- **Speech-detection latency** — timestamp of first "Speech Start" minus the moment audio energy first rose.
- **End-of-speech → transcript latency** — timestamp of final transcript minus the Speech-End event.

These are the two latencies observable without server timing, and they let a reviewer see the silence-timeout effect live by adjusting it.
