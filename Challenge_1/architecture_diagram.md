# Architecture Diagram: Indian English Piper TTS Adaptation

**Challenge 1 — Deliverable 2 of 4** · Supports **Tasks 1–4**

> **What this document covers:** two block diagrams — the **offline** training/fine-tuning pipeline (how the voice is built) and the **online** browser inference pipeline (how it runs client-side) — plus component notes. Read the Technical Design Document first for the conceptual split (eSpeak-ng = phonemes, VITS = audio).

---

## 1. Offline Training & Fine-Tuning Pipeline

Raw audio + transcripts are transformed into a specialized VITS checkpoint and exported to ONNX for the browser.

```mermaid
flowchart TD
    %% Data preparation
    subgraph DataPrep [1. Data Preparation & Normalization]
        A[(Raw Audio: .wav)] --> B[Resample to 22.05 kHz / Mono / 16-bit]
        B --> C[Silence Trim & Loudness Normalize]
        D[(Raw Transcripts)] --> E[Text Normalizer: Numbers / Currency / Abbreviations]
    end

    %% Preprocessing & alignment
    subgraph PiperPre [2. Preprocessing & Alignment]
        C --> F[Audio Feature Extraction: Linear Spectrograms]
        E --> G["eSpeak-ng phonemizer<br/>(stock en-us — Path A, or custom en-in variant — Path B)"]
        G --> H[IPA Phoneme String]
        H --> I[Phoneme-to-ID Mapper]
        I --> J[Phoneme ID Tensor]
    end

    %% Model & training
    subgraph TrainingLoop [3. VITS Fine-Tuning Loop]
        K[(Pre-trained Checkpoint: en_US-lessac-medium)] --> L[Initialize VITS Generator & Discriminators]
        F --> M[Posterior Encoder]
        J --> N[Text Encoder + Duration/Pitch Predictor]
        M & N --> O[Normalizing-Flow Decoder]
        O --> P[HiFi-GAN Vocoder]
        P --> Q[Synthesized Audio]
        Q & B --> R[Discriminator + GAN + Reconstruction Loss]
        R -->|Gradient Backprop| L
    end

    %% Export
    L -->|Best checkpoint| S[PyTorch Checkpoint: .ckpt]
    S -->|ONNX Export| T[Optimized Model: .onnx]
    S -->|INT8 Quantization| T2[Quantized Model: .onnx ~15-18MB]
    I -->|Export config| U[Metadata Config: .onnx.json]
```

---

## 2. Client-Side Browser Inference Pipeline

Everything runs locally in the browser — no server round-trip. Inference happens on a Web Worker so the UI never blocks.

```mermaid
flowchart LR
    Input[Input Text String] --> Norm[JS Text Pre-processor: expand numbers/symbols]

    subgraph Worker [Web Worker - off the UI thread]
        subgraph WasmEngine [eSpeak-ng WASM Phonemizer]
            Norm --> EspeakWasm["eSpeak-ng WebAssembly<br/>(SAME voice + dictionary as training)"]
            EspeakWasm --> IPA[IPA Phoneme String]
        end
        subgraph OrtWeb [ONNX Runtime Web - WASM + SIMD]
            IPA --> Map[JS Phoneme-to-ID Mapper]
            Map --> Tensor[Phoneme ID Tensor]
            Tensor --> ONNX[VITS ONNX Engine]
            ONNX -->|float32 array| RawAudio[Raw Audio Buffer]
        end
    end

    RawAudio -->|Transfer to main thread| WebAudio[Web Audio API: AudioContext]
    WebAudio --> Speakers([Speakers / Headphones])

    Cache[(Cache API / IndexedDB<br/>model cached after first load)] -.-> ONNX
```

> **The two diagrams must agree at the eSpeak box.** The browser phonemizer (right) must use the *identical* eSpeak-ng voice and dictionary as the training pipeline (left). This is the single most important cross-pipeline invariant — a mismatch feeds the model phoneme IDs it never saw in training.

---

## 3. Component Details & Pipeline Flow

### A. Data Prep & Preprocessing
* **Audio resampler** — standardizes voice data to 22,050 Hz, mono, 16-bit PCM, matching Piper's `medium` quality tier.
* **eSpeak-ng (native & WASM)** — converts text to IPA phonemes. Training uses the native CLI/library; the browser uses an Emscripten-compiled WASM module of the *same* eSpeak version and dictionary.

### B. VITS Neural Network Structure
* **Text encoder** — projects phoneme IDs into hidden linguistic representations.
* **Duration / pitch predictor** — predicts per-phoneme timing and the F0 (pitch) contour; this is where Indian cadence and intonation are learned.
* **Posterior encoder** — used only in training, encodes structure from ground-truth audio spectrograms.
* **Normalizing flow** — bridges the simple text representation to the complex target audio distribution.
* **HiFi-GAN vocoder** — the decoder; turns features into a time-domain waveform.

### C. ONNX Runtime Web Execution
* **Model serialization** — the full VITS generator (flow decoder + HiFi-GAN) exports as one `.onnx` file; INT8-quantized to ~15–18 MB for the web.
* **Web Workers** — inference runs on a background thread so the UI never freezes.
* **Web Audio API** — the raw float32 output streams straight to the audio device with low latency (target < 150 ms to first audio).
* **Caching** — the model is cached in Cache API / IndexedDB after first download, so repeat visits load instantly and offline.
