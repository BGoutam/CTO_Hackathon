# Technical Design Document: Client-Side VAD & Speech Recognition Pipeline

**Challenge 2 — Deliverable (Technical Documentation)** · Primarily answers **Task 1 (VAD Design)**, **Task 2 (ASR Integration)**, and **Task 3 (Reliability Strategy)**

> **What this document covers:** the full browser-only voice pipeline — how we detect speech start/end, handle silence and noise, wire the VAD to the recognizer, manage sessions, and stay reliable across long conversations, failures, interruptions, and browser limits. Latency numbers are in the Performance Analysis report; the runnable demos are described below.

---

## Proof-of-Concept: Two Runnable Demos

Two working single-file demos ship with this document. They share an **identical VAD core** (Silero VAD v5 with 512-sample framing + energy fallback), the same UI, TTS speak-back, live barge-in, latency panel, and silence-timeout slider. They differ **only in the recognizer** — which is exactly the design trade-off in §2.0.

| File | ASR engine | Client-side? | When to use |
| :--- | :--- | :--- | :--- |
| **`vad_asr_poc.html`** | Web Speech API (streaming) | VAD on-device; **ASR is cloud** in Chrome | Snappy demo (~100–300 ms ASR) where cloud STT is acceptable |
| **`vad_asr_poc_whisper.html`** | Whisper-tiny.en via transformers.js (ONNX/WASM) | **100% on-device**, offline & private after a one-time model download | The **strict "no server-side processing"** requirement (~0.4–1.5 s ASR) |

> **Why two?** The Whisper build is the honest answer to *"is it truly client-side?"* — you can run it with the network tab open and show **zero audio leaving the browser** after the initial weight download. Because Whisper is batch (not streaming), it also demonstrates the **~500 ms pre-roll ring buffer** for real (see §2B) — the first word is captured and prepended, which the Web Speech API cannot do. The Web Speech build is the fast, low-footprint counterpoint.
>
> **Run either demo** over a secure context (`https://` or `http://localhost`, since the mic is blocked on `file://`): `python -m http.server 8000`, then open the file and click the mic. Run instructions are in a comment at the top of each HTML file.

---

## 0. The One Idea That Frames Everything

A conversational voice UI is a **state machine driven by a fast, cheap gatekeeper**:

| Layer | Job | Cost | Runs |
| :--- | :--- | :--- | :--- |
| **VAD (Silero)** | *Is someone speaking right now?* | Tiny (~2 ms/frame) | Continuously, on every 32 ms frame |
| **ASR (recognizer)** | *What did they say?* | Expensive | Only while the VAD says "speech" |

The VAD is the **traffic cop**: it decides *when* to turn the expensive recognizer on and off. Get the VAD's timing right and everything downstream — latency, cost, barge-in — falls into place. Almost every design decision below flows from this split.

> **Client-side guarantee (read this):** the VAD (Silero VAD ONNX) is genuinely 100% on-device. The recognizer is the subtle part — see §2.0. The browser's default Web Speech API is **not** on-device, so we treat the fully-local Whisper path as the compliant default.

---

## 1. Voice Activity Detection (VAD) Strategy — Task 1

We use **Silero VAD v5** as an ONNX model in the browser: ~1.3–2.3 MB, runs on ONNX Runtime Web (WASM + SIMD), and returns a speech probability per audio frame.

```
+------------+   Raw Audio    +-----------------------+   16kHz mono    +-------------+
| Microphone | ------------> | Web Audio: resample + | -------------->  | Silero VAD  |
| (48 kHz)   |                | 512-sample framing    |   512-sample     | (ONNX v5)   |
+------------+                +-----------------------+   frames         +------+------+
                                                                                |
                                                                    P_speech ∈ [0,1] per 32 ms frame
```

> **Frame size — the fixed constraint:** Silero **v5 requires exactly 512 samples per frame at 16 kHz (= 32 ms)**. This is not adjustable in v5 (the flexible window of v3/v4 is gone). Because the browser mic is usually 48 kHz, we resample to 16 kHz and slice fixed 512-sample frames from a ring buffer — the audio callback's block size is *decoupled* from the model's frame size.

### A. Speech Start Detection
* **Mechanism:** each 512-sample (32 ms) frame is scored by Silero → `P_speech ∈ [0.0, 1.0]`.
* **Trigger:** fire **Speech Start** when `P_speech > 0.5` for **2 consecutive frames (~64 ms)**. Requiring two frames rejects transient clicks, keyboard taps, and lip smacks that would otherwise false-trigger the mic.

### B. Speech End (Silence) Detection
* **Mechanism:** once in the speaking state, watch for a sustained probability drop.
* **Trigger:** fire **Speech End** after `P_speech < 0.35` continuously for **~800 ms** (tunable). Using a **lower end threshold (0.35) than start (0.5)** is deliberate **hysteresis** — it prevents flicker between speaking/silent on the natural amplitude dips *within* a sentence.
* **Why 800 ms:** short enough to feel responsive, long enough not to clip a speaker who pauses briefly between clauses. This single value dominates end-to-end latency (see Performance Analysis).

### C. Silence Handling
* **Intra-utterance pauses** (commas, "um") are absorbed by the 800 ms window — we don't cut the user off mid-thought.
* **Trailing silence** beyond the window ends the turn and triggers transcription.

### D. Background Noise Handling
* **Spectral pre-filtering:** a Web Audio **band-pass filter (≈300 Hz–3400 Hz, the speech band)** removes low-frequency hum (AC, fans) and high-frequency hiss before the signal reaches the model.
* **Adaptive noise floor:** in noisy rooms the probability baseline drifts up. We track a moving average of the *quietest* recent frames and raise the start threshold accordingly:
  $$\text{Threshold}_{start} = \max\big(0.5,\ \text{NoiseFloor} + 0.15\big)$$
* **Why Silero over energy/RMS:** a neural VAD distinguishes *speech* from *loud non-speech* (music, door slams, typing) — a plain RMS/energy gate cannot. (The PoC still ships an RMS gate as a guaranteed-available fallback; see §3.)

---

## 2. Speech Recognition (ASR) Integration — Task 2

### 2.0. Choosing the Recognizer — the Client-Side Decision
The challenge requires **no server-side processing**. That eliminates the naive choice:

| Backend | Truly on-device? | Notes |
| :--- | :--- | :--- |
| **Whisper via transformers.js (ONNX/WASM)** ✅ *compliant default* | **Yes** — weights fetched once, cached, then fully offline | `whisper-tiny` ≈ ~78 MB download; ~400–1500 ms inference. Meets the strict requirement. |
| **Web Speech API** (Chrome default) ⚠️ | **No** — streams audio to Google's servers; needs network | Fast and accurate, but **violates "no server-side processing."** Fine only where cloud STT is acceptable. |
| **Web Speech API — Chrome on-device** ◐ | Yes, *if* enabled | Opt-in via `processLocally=true` + a downloaded language pack; experimental, Chrome-only. |

> **Design decision:** treat **Whisper (transformers.js)** as the compliant, private, offline default; offer **Web Speech API** as an optional low-footprint fallback *only* when the user accepts cloud recognition (or has Chrome's on-device pack). Both are provided as runnable demos: `vad_asr_poc_whisper.html` (compliant, on-device) and `vad_asr_poc.html` (Web Speech, cloud) — see the *Proof-of-Concept* note above.

### A. Event Flow
```
[IDLE]
   | VAD: Speech Start (P>0.5 × 2 frames)
   v
[USER_SPEAKING] ---> start/stream ASR   (prepend pre-roll buffer, see §B)
   | VAD: Speech End (P<0.35 for 800 ms)
   v
[TRANSCRIBING] ---> finalize ASR, request transcript
   | transcript received
   v
[PROCESSING] ---> hand transcript to app / assistant logic
```

### B. Session Management
* **ASR startup lag & the pre-roll buffer:** recognizers take 100–300 ms to spin up, which can swallow the first word. We keep a **~500 ms circular ring buffer of raw audio** and prepend it when the turn starts, so no leading audio is lost. *(Applies to the Whisper path, which accepts raw audio; the Web Speech API does not accept injected audio, so with it the pre-roll is a best-effort early-start rather than a true prepend.)*
* **State ownership:** the VAD controller — not the recognizer — is the single source of truth for turn boundaries.

### C. Error Handling
* **Empty transcript** (cough, laugh, click that slipped through): silently return to `IDLE`.
* **Mic disconnect / permission loss:** detect the dropped stream, surface a clear "check microphone permissions" prompt, and halt the graph cleanly.
* **ASR error / timeout:** log, return to `IDLE`, and keep the VAD alive so the next utterance still works.

---

## 3. Reliability & Edge Cases — Task 3

* **Long-running conversations.** Browsers cap recognizer sessions (Chrome's Web Speech auto-stops after a few seconds of silence). The **VAD controller acts as the master state machine**, explicitly cycling `.stop()`/`.start()` to refresh sessions cleanly so a 30-minute conversation never silently dies.
* **Recognition failures.** Empty/garbled results revert to `IDLE` without crashing the loop; the VAD keeps listening. Optionally re-prompt the user.
* **User interruption (barge-in).** While the assistant is speaking (TTS), the VAD keeps monitoring the mic. If `P_speech > 0.5`, we **immediately cancel TTS playback** and hand control back to the user. *(The PoC demonstrates this end-to-end using `speechSynthesis` for the assistant voice.)*
* **Browser limitations.**
  * If WASM SIMD/threads are unavailable, ONNX Runtime Web falls back to single-threaded WASM; we reduce concurrency to avoid UI jank.
  * If the Silero model can't load (offline/CORS), the pipeline **degrades to an RMS energy-gate VAD** so the demo always functions.
  * **`ScriptProcessorNode` is deprecated** (though still supported); production should use **`AudioWorklet`** (off-main-thread). The single-file PoC uses ScriptProcessor for portability and notes this trade-off.
  * **AudioContext at 16 kHz** works in Chrome (auto-resamples the mic) but is **ignored by Firefox** — so we resample explicitly from the hardware rate to 16 kHz rather than trusting the context rate.

---

## 4. Task Summary

- **VAD:** Silero v5 ONNX, **512-sample/32 ms frames**, start at `P>0.5 ×2 frames`, end at `P<0.35 for 800 ms` (hysteresis), band-pass pre-filter + adaptive noise floor.
- **ASR:** Whisper/transformers.js is the **client-side-compliant** default; Web Speech API is a cloud-dependent convenience fallback (flagged, not default).
- **Reliability:** VAD controller owns all turn state — refreshes ASR sessions, absorbs failures, drives barge-in, and degrades to energy VAD if the model can't load.
