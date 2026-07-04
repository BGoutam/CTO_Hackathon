# Implementation Approach: Voice Development Strategy

**Challenge 1 — Deliverable 3 of 4** · Primarily answers **Task 2 (Voice Development Strategy)** and the *validation* portion of **Task 3 (Model Evaluation)**

> **What this document covers:** which datasets to use and why, how to prepare the data, why we fine-tune rather than train from scratch, and how we validate the result. Hyperparameters and the remaining evaluation metrics live in the Final Recommendation (Deliverable 4).

---

## 1. Recommended Datasets (with Justification)

The dataset determines the ceiling on voice quality — no amount of training fixes noisy or inconsistent audio. We recommend one primary corpus and one alternative depending on the target profile.

### A. IndicTTS English Database, IIT Madras — *Recommended (single-speaker consistency)*
* **What it is:** studio-recorded, high-fidelity single-speaker English read with Indian pronunciation (separate male and female sets).
* **Why:**
  * **Acoustic consistency** — controlled studio conditions, professional talent, minimal noise, steady prosody. Ideal for a clean, natural single-speaker voice.
  * **Phonetic balance** — designed for speech-synthesis research, so it covers a wide range of phoneme combinations.
  * **Licensing** — available for research and development.

### B. Crowdsourced Indian-English speech (e.g., Mozilla Common Voice, India-accent subset) — *Alternative (multi-speaker variety, permissive license)*
* **What it is:** large crowdsourced English corpus; Common Voice tags speaker accent, so an India-accented subset can be filtered out.
* **Why:**
  * **Modern, realistic accents** — natural urban Indian English across many speakers.
  * **Permissive license (CC0)** — safe for commercial and hackathon use.
  * **Trade-off to manage** — crowdsourced audio is noisier and less consistent than studio data, so it needs heavier cleaning (Section 2) and is better suited to a robust multi-speaker voice than a pristine single-speaker one.

> **Choice rule:** for the demo, prioritize **IndicTTS (single-speaker)** — it gives the cleanest, most natural voice for the least effort. Use crowdsourced data when speaker diversity matters more than pristine fidelity.

---

## 2. Data Preparation Requirements

Raw audio and transcripts must be prepared carefully — misalignment or inconsistent audio is the most common cause of garbled fine-tuning results.

```
[Raw Data] → [Audio Audit & Normalize] → [Silence Trim] → [Text Normalize] → [Phoneme Check]
```

### A. Audio Standardization & Cleanup
1. **Resample** every file to exactly **22,050 Hz** (or 16,000 Hz for a low-profile target), 16-bit, mono PCM WAV.
2. **Loudness normalize** — target ≈ **-23 LUFS** (or a -1.0 dBFS peak) so volume is consistent across samples; uneven loudness destabilizes training.
3. **Trim silence** — clip leading/trailing silence to a fixed window (~50 ms). Excess boundary silence degrades text-to-audio alignment.
4. **Gentle denoise** — high-pass filter at ~80 Hz to remove low-frequency room rumble without touching speech.

### B. Transcript Normalization
1. **Expand digits, symbols, and abbreviations to spoken words** — and crucially, use **Indian number formatting**:
   * `₹1,50,000` → "one lakh fifty thousand rupees"  *(lakh/crore grouping, not million)*
   * `1.5 kg` → "one point five kilograms"
   * `IT Dept.` → "I T Department"
2. **Phonemization pre-flight** — run transcripts through `espeak-ng -v en-us -q --ipa` and scan for unmappable characters or unexpected fallbacks *before* training. (Use `en-us`, not `en-in` — the latter does not exist; see the Technical Design Document.)

---

## 3. Training & Adaptation Strategy — Fine-Tune, Don't Train From Scratch

We strongly recommend **fine-tuning** a pre-trained checkpoint (e.g., `en_US-lessac-medium`) via transfer learning.

```
+------------------------------------+
| Pre-trained Checkpoint             |  (en_US-lessac-medium)
| - General pronunciation framework  |
| - Stable, high-fidelity HiFi-GAN   |
+------------------------------------+
                  |  transfer learning
                  v
+------------------------------------+
| Fine-Tuning                        |  (~10k-20k steps on the en-in dataset)
| - Adapt to Indian acoustic accent  |
| - Adjust prosody, pitch, duration  |
+------------------------------------+
                  |
                  v
+------------------------------------+
| Indian English Voice (VITS)        |  -> export to ONNX for the browser
+------------------------------------+
```

### Why fine-tuning wins
* **Compute efficiency** — training VITS from scratch needs 40–100+ hours of studio audio and *weeks* on multiple GPUs. Fine-tuning converges in **~10k–20k steps (≈ 4–6 hours on a single GPU)**.
* **Modest data need** — a natural voice needs only **~3–5 hours** of clean target audio, versus the massive corpora scratch training demands.
* **Vocoder reuse** — the HiFi-GAN vocoder in the base checkpoint already produces clean audio; fine-tuning only has to shift the accent, intonation, and timing — not relearn how to make sound.

> **Recall the two-path decision (Technical Design Doc):** fine-tuning here delivers **Path A** (accent learned acoustically with stock `en-us` phonemes). A custom eSpeak variant (**Path B**) is an *optional* add-on for retroflex/loanword fidelity, layered on top of this same fine-tuning.

---

## 4. Validation Methodology

We use a hybrid of objective metrics (repeatable, cheap) and subjective metrics (the ultimate test of a voice). This directly supports **Task 3 (Model Evaluation)**.

### A. Quantitative (Objective)
1. **Word Error Rate (WER) — intelligibility.** Synthesize a held-out test set, transcribe it with an independent ASR model (e.g., Whisper-medium), and compare to the reference text. **Target WER < 5%** — low WER means the speech is clear and machine-intelligible.
2. **Mel-Cepstral Distortion (MCD) — acoustic similarity.** Measure the spectral distance between synthesized audio and matched real recordings. **Target MCD < 3.0** dB indicates high similarity to the target voice.

### B. Qualitative (Subjective)
1. **Mean Opinion Score (MOS).** A panel of native Indian-English listeners rates samples **1–5** on three axes:
   * **Intelligibility** (clarity), **Naturalness** (human-like flow), **Accent authenticity** (absence of US/UK coloring).
2. **A/B preference testing.** Present listeners with base-model vs. fine-tuned clips of the same sentence to confirm a real, perceptible accent improvement.

> **Why both:** objective metrics catch regressions automatically during training (cheap, fast), but only human MOS confirms the voice actually *sounds* right — the metric that matters for the demo. A model can score well on WER yet still sound foreign; A/B testing catches that.
