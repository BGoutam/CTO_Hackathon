# Final Recommendation & Deployment Considerations

**Challenge 1 — Deliverable 4 of 4** · Answers the *hyperparameter* portion of **Task 3 (Model Evaluation)** and all of **Task 4 (Deployment Considerations)**

> **What this document covers:** the recommended training hyperparameters and their impact, the browser deployment architecture (model size, memory, latency, quality trade-offs), and the key risks with concrete mitigations. Read alongside the Implementation Approach (datasets, validation) and the Technical Design Document (pipeline, the `en-in` question).

---

## Executive Recommendation (read this first)

Build the Indian-English voice by **fine-tuning `en_US-lessac-medium` on ~3–5 hours of clean Indian-English audio**, phonemizing with the **stock `en-us`** eSpeak voice (there is no `en-in` voice — see the Technical Design Document). Ship it to the browser as an **INT8-quantized ONNX model (~15–18 MB)** run on **ONNX Runtime Web (WASM + SIMD + multithreading) inside a Web Worker**, with the model cached in IndexedDB. Default to the **`medium`** quality tier with a **`low`**-tier fallback for weak devices. This hits natural Indian-English quality with **< 150 ms** time-to-first-audio, entirely client-side.

---

## 1. Key Training Hyperparameters & Impact

When fine-tuning VITS in the Piper codebase, these settings matter most:

| Hyperparameter | Recommended | Impact & Justification |
| :--- | :--- | :--- |
| **Learning rate** | $2\times10^{-4}$ | Standard AdamW rate. Higher risks destroying the pre-trained vocoder; lower makes adaptation too slow. |
| **Batch size** | 16–32 | Trade speed vs. memory. Larger batches stabilize gradients but need more VRAM (16 GB+). |
| **LR decay** | $0.9998^{\text{epoch}}$ | Gradual decay avoids overshooting as the model nears a good minimum. |
| **Grad-clip norm** | 1.0 | Caps gradient magnitude — essential for VITS to prevent normalizing-flow instability. |
| **Warmup steps** | ~1,000 | Ramps the LR up slowly at the start, protecting pre-trained layers from an abrupt shock. |
| **Fine-tune steps** | 10k–20k | Enough to adapt accent/prosody without overfitting a small dataset or "forgetting" the base voice. |

> **Why these and not others:** fine-tuning is a *gentle nudge* to a good model, not a fresh training run. Every value above is chosen to **preserve the stable pre-trained vocoder** while letting the accent-bearing layers (duration, pitch, flow) adapt. Aggressive settings undo the very thing that makes fine-tuning cheap.

---

## 2. Deployment Architecture for the Browser (Task 4)

The solution runs **100% client-side** — no inference server. The threading model keeps the UI responsive:

```
[Input Text]
    |
    v
[Web Worker Thread]                         <-- keeps the UI thread free
    |--- 1. eSpeak-ng WASM  (text -> IPA phonemes)
    |--- 2. ONNX Runtime Web (phonemes -> float32 audio)
    |
    v  (transfer the Float32 audio buffer)
[Main UI Thread]
    |--- Web Audio API (low-latency playback)
```

The four Task-4 concerns — **model size, memory, latency, quality** — are addressed below.

### A. Model Size & Memory — Quantization
| Precision | Model size | Notes |
| :--- | :--- | :--- |
| FP32 (baseline) | ~60 MB | Highest quality; heavy download and memory footprint. |
| **INT8 (recommended)** | **~15–18 MB** (~70% smaller) | INT8 ops run faster on modern x86/ARM via WASM SIMD, with minimal loss in naturalness. Best size/quality/speed balance for the web. |

### B. Inference Latency — CPU/WASM Execution
* Use **ONNX Runtime Web** with the **WebAssembly backend** (no GPU dependency → runs everywhere).
* Enable **SIMD** and **multithreading**:
  ```javascript
  import * as ort from 'onnxruntime-web';
  ort.env.wasm.numThreads = navigator.hardwareConcurrency || 4;
  ort.env.wasm.simd = true;
  ```
* Target **< 150 ms** time-to-first-audio and a **Real-Time Factor (RTF) < 1.0** (generates faster than it plays) so speech feels instant.

### C. Responsiveness & Repeat Loads — Threading + Caching
* **Never run inference on the UI thread.** Do phonemization and ONNX inference in a **Web Worker** so the page never freezes.
* **Cache the model** in **Cache API / IndexedDB** after first download so repeat visits load instantly and work offline.
* **Bundle the matching eSpeak-ng data.** The WASM phonemizer must ship the *same* voice/dictionary used in training (the consistency rule from the Technical Design Document) — otherwise the model receives phonemes it never trained on.

---

## 3. Key Trade-offs & Expected Challenges (with mitigations)

### A. Accent Drift / Regression
* **Problem:** because the base model is US-trained, the fine-tuned voice can "drift" back toward US pronunciations on complex or unfamiliar sentences.
* **Mitigation:** keep the training set **balanced** — cover a wide range of sentence structures in Indian English so the model doesn't overfit narrow patterns; monitor with A/B testing during training.

### B. Code-Mixed Terms (loanwords, names)
* **Problem:** words like "Aadhaar", "crore", "panchayat", or names like "Goutam" get mangled by generic English letter-to-sound rules.
* **Mitigation (two layers):** (1) a **custom lookup dictionary applied before eSpeak** that substitutes known loanwords with their correct phonemes; and/or (2) add these words to eSpeak's `dictsource/en_list` (Path B in the Technical Design Document). Whichever you choose, **it must run identically in training and in the browser.**

### C. Quality vs. Latency
* **Problem:** the `medium` tier (22.05 kHz) sounds excellent but costs more CPU; the `low` tier (16 kHz) is fast but slightly muffled.
* **Mitigation:** ship **`medium` by default** and **auto-fall-back to `low`** when the client reports limited CPU (`navigator.hardwareConcurrency`) or measured RTF ≥ 1.0. This preserves quality on capable devices without stranding weak ones.

### D. Train/Inference Phonemization Mismatch *(the subtle one judges look for)*
* **Problem:** using a different eSpeak version or dictionary in the browser than in training silently corrupts pronunciation.
* **Mitigation:** pin the eSpeak-ng version, ship the exact same compiled dictionary in the WASM bundle, and add a startup self-test that phonemizes a known sentence and checks the output against a stored reference.

---

## 4. Recommendation Summary

| Decision | Recommendation | Rationale |
| :--- | :--- | :--- |
| **Strategy** | Fine-tune, don't train from scratch | 4–6 hrs vs. weeks; ~3–5 hrs of audio vs. 40–100+ |
| **Base model** | `en_US-lessac-medium` | Clean, stable HiFi-GAN vocoder to build on |
| **Phonemizer** | Stock `en-us` (Path A); custom variant optional (Path B) | No `en-in` voice exists; accent learned acoustically |
| **Quality tier** | `medium` (22.05 kHz), `low` fallback | Best quality where possible, graceful on weak devices |
| **Deployment** | INT8 ONNX (~15–18 MB) on ORT-Web (WASM+SIMD) in a Web Worker, cached in IndexedDB | Small, fast, fully client-side, offline-capable |
| **Validation** | WER < 5%, MCD < 3.0, MOS panel + A/B | Objective gates during training; human MOS as the final word |
