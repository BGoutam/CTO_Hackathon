# Speech Synthesis (TTS) Glossary & Presentation Companion

Your one-stop reference for presenting **Challenge 1**. It defines every term in the deliverables, explains **why each matters**, gives **plain-English analogies** you can say out loud, and ends with a **Q&A cheat-sheet** and **common misconceptions** so you're never caught off guard.

> **The 15-second pitch:** *"Piper turns text into speech in two steps. First, eSpeak-ng converts letters into phonemes — the sounds. Then a neural network called VITS turns those sounds into an actual voice. To get Indian English, we fine-tune the neural network on Indian speech, and it all runs offline in the browser."*

---

## 1. The Big Picture — How the Pipeline Fits Together

Say this whole flow in one breath during the demo:

```
Text  →  [eSpeak-ng]  →  Phonemes (IPA)  →  [ID Map]  →  Numbers  →  [VITS neural net]  →  Audio
"crore"    phonemizer      "kroːr"          config       [11,17,..]     ONNX model          🔊
```

| Step | Who does it | Analogy |
| :--- | :--- | :--- |
| 1. Normalize text | JS / Piper | Turn "₹150" into "one hundred fifty rupees" — read it as you'd *say* it |
| 2. Phonemize | **eSpeak-ng** | Hand the actor a **pronunciation script** |
| 3. Map to IDs | config.json | Translate the script into **numbers the computer understands** |
| 4. Synthesize | **VITS** | The **actor performs** the script in a specific voice |
| 5. Play | Web Audio API | The **speaker** in the theatre |

**The single most important idea:** eSpeak-ng decides *which* sounds (pronunciation); VITS decides *how they sound* (accent, tone, naturalness). Almost every design decision follows from that split.

---

## 2. Core Terminology

### A. General TTS
* **TTS (Text-to-Speech):** software that converts written text into spoken audio. *Why it matters:* it's the whole product.
* **Grapheme:** the smallest *written* unit — a letter. "ship" has graphemes s, h, i, p. *Analogy:* what you **see**.
* **Phoneme:** the smallest *sound* unit that changes meaning. "ship" has phonemes `/ʃ/ /ɪ/ /p/`. *Analogy:* what you **hear**. (Note "sh" = two letters, one sound.)
* **Phonemizer:** the module that turns graphemes into phonemes. In Piper, that's eSpeak-ng.
* **Prosody:** the *melody* of speech — rhythm, stress, and intonation. *Why it matters:* it's what makes a voice sound Indian vs. American even when the words are identical.

### B. Phonetic Representation
* **IPA (International Phonetic Alphabet):** a universal alphabet where each symbol is exactly one sound (`/t/` vs. `/ʈ/`). *Analogy:* a **standardized sound spelling** every language can share.
* **eSpeak-ng:** a fast, open-source, rule-based phonemizer. *Key point for the panel:* in Piper it does **not** make the final audio — it only produces phonemes for the neural model.
* **Alveolar vs. Retroflex consonants:** *alveolar* `/t/ /d/` = tongue tip near the gum ridge (US/UK English). *Retroflex* `/ʈ/ /ɖ/` = tongue curled back toward the roof of the mouth (typical Indian English "t"/"d"). *Why it matters:* this is a concrete, audible marker of the Indian accent — a great example to name on stage.

### C. Neural Network Architecture (VITS)
* **VITS** (*Variational Inference with adversarial learning for end-to-end TTS*): the neural model Piper uses. Takes phonemes → outputs the waveform directly, in one model. *Analogy:* a **voice actor** who reads the phonetic script.
* **Text encoder:** reads the phoneme IDs and turns them into rich internal representations.
* **Duration & pitch predictor:** decides how *long* each sound lasts and how the *pitch* moves. *Why it matters:* this is where Indian **rhythm and intonation** are learned.
* **Posterior encoder:** used only during training; learns structure from the real audio.
* **Normalizing flows:** a technique that maps simple text features to the rich variety of real speech (pitch wobble, pacing, expressiveness). *Why it matters:* it's what keeps the voice from sounding flat/robotic.
* **Vocoder (HiFi-GAN):** the final stage that turns the model's internal representation into an actual sound wave. *Analogy:* the **vocal cords** that produce the audio.
* **GAN / Discriminator:** during training, a "critic" network judges whether audio sounds real; the generator improves to fool it. *Analogy:* a **strict voice coach** pushing the actor toward realism.
* **Spectrogram / Mel-spectrogram:** a picture of sound — frequency (y) over time (x). *Analogy:* **sheet music** for a waveform; the model works with these internally.
* **F0 (fundamental frequency):** the base pitch of the voice, in Hz. Its movement over a sentence *is* intonation.

### D. Training & Optimization
* **Fine-Tuning:** take a model already trained on lots of data and adapt it with a *small* new dataset. *Analogy:* teaching a **fluent English speaker an Indian accent** — far faster than raising a child from birth. *Why it matters:* it's the core strategy — 4–6 hrs vs. weeks.
* **Training from scratch:** the opposite — start from random weights; needs 40–100+ hrs of audio and weeks of GPU time. We *don't* do this.
* **Transfer learning:** the general principle behind fine-tuning — reuse knowledge from one task on a related one.
* **Checkpoint:** a saved snapshot of model weights (e.g., `en_US-lessac-medium`), used as the fine-tuning starting point.
* **Epoch / Step / Batch:** an **epoch** = one full pass over the data; a **step** = one weight update; a **batch** = the number of samples per step. We fine-tune for ~10k–20k steps.
* **Learning rate:** how big each weight update is. Too high → wreck the model; too low → learns too slowly. We use $2\times10^{-4}$.
* **Warmup:** starting with a tiny learning rate and ramping up, so early updates don't shock the pre-trained model.
* **Gradient clipping:** capping update size to prevent training from "blowing up" — important for VITS's flows.
* **Overfitting:** when the model memorizes the small dataset instead of generalizing; a risk with only a few hours of audio.

### E. Deployment & Runtime
* **ONNX (Open Neural Network Exchange):** a standard file format that packages a trained model for fast, portable execution (including in browsers). *Analogy:* a **PDF for AI models** — export once, run anywhere.
* **ONNX Runtime Web (ORT-Web):** the engine that runs ONNX models inside a browser.
* **WebAssembly (WASM):** a way to run compiled (near-native-speed) code in the browser. Both eSpeak-ng and the ONNX runtime run as WASM. *Analogy:* lets the browser run **desktop-speed code**.
* **SIMD (Single Instruction, Multiple Data):** a CPU feature that does many math ops at once — big speedup for neural nets in WASM.
* **Quantization (INT8):** storing model weights as 8-bit integers instead of 32-bit floats — ~4× smaller and faster, with minimal quality loss. *Why it matters:* shrinks the model from ~60 MB to ~15–18 MB so it downloads fast.
* **Web Worker:** a background browser thread. We run inference here so the page/UI never freezes. *Analogy:* a **kitchen out back** so the dining room stays calm.
* **Web Audio API:** the browser's audio engine that plays the generated samples.
* **Cache API / IndexedDB:** browser storage where we save the model after first download, so it loads instantly next time and works offline.
* **Latency:** delay before the user hears audio. We target **< 150 ms** to first sound.
* **RTF (Real-Time Factor):** $\text{RTF}=\dfrac{\text{time to generate audio}}{\text{duration of that audio}}$. **RTF < 1.0** = faster than real time = feels instant.
* **Sample rate:** audio resolution in Hz. **22,050 Hz** = `medium`/`high` quality; **16,000 Hz** = `low`/`x-low` (faster, slightly muffled).
* **PCM (Pulse-Code Modulation):** the raw uncompressed audio format the model outputs (float32 samples).

### F. Evaluation Metrics *(say these confidently — judges love metrics)*
* **WER (Word Error Rate):** transcribe the synthesized speech with a separate speech-recognizer and compare to the script. **Low WER (< 5%) = intelligible.** *Measures:* clarity.
* **MCD (Mel-Cepstral Distortion):** acoustic distance between synthesized and real audio. **Lower = more similar; target < 3.0.** *Measures:* voice similarity.
* **MOS (Mean Opinion Score):** humans rate samples 1–5. *Measures:* the thing that ultimately matters — does it *sound* good and Indian? Judged on intelligibility, naturalness, and accent authenticity.
* **A/B preference test:** play base-model vs. fine-tuned clips; ask which sounds more Indian. *Measures:* whether the fine-tuning actually helped.

---

## 3. The `en-in` Nuance — Be Ready For This

**There is no `en-in` voice in eSpeak-ng.** It ships `en-us`, `en` (British), `en-029` (Caribbean), and a few UK regional variants — nothing Indian. If a judge asks *"so you just set the voice to Indian English?"*, the strong answer is:

> *"eSpeak-ng doesn't have an Indian English voice — so we don't rely on it for the accent. We phonemize with standard `en-us` and let the **neural model** learn the Indian accent acoustically from the dataset. Optionally, we can author a custom eSpeak variant to emit retroflex sounds for even higher fidelity, but the accent primarily comes from fine-tuning."*

This turns a would-be mistake into a demonstration of understanding — exactly what the challenge rewards.

---

## 4. Presentation Q&A Cheat-Sheet

| If they ask… | Say… |
| :--- | :--- |
| *"Why fine-tune instead of training from scratch?"* | "From scratch needs 40–100+ hours of audio and weeks of GPUs. Fine-tuning needs ~3–5 hours and 4–6 hours on one GPU, because the vocoder is already trained — we only adapt the accent." |
| *"How does it run in the browser with no server?"* | "The model is exported to ONNX, quantized to ~15 MB, and run with ONNX Runtime Web on WebAssembly inside a Web Worker. eSpeak-ng also compiles to WASM. Nothing leaves the device." |
| *"Where does the Indian accent actually come from?"* | "The neural model — VITS — learns Indian rhythm, pitch, and consonant realization from the training data. eSpeak just provides the phonemes." |
| *"How do you know it's good?"* | "Objective metrics — WER under 5% for clarity, MCD under 3.0 for similarity — plus human MOS ratings and A/B tests against the base model." |
| *"What's the hardest part?"* | "Accent drift (the model slipping back to US pronunciation), code-mixed loanwords like 'Aadhaar' or 'crore', and keeping browser phonemization identical to training." |
| *"What about words like 'lakh' or 'crore'?"* | "We add them to a custom dictionary so they're pronounced correctly, and we use Indian number formatting — 1,50,000 becomes 'one lakh fifty thousand', not 'one hundred fifty thousand'." |
| *"Quality vs. speed on a weak phone?"* | "We ship the medium 22 kHz model by default and auto-fall-back to the fast 16 kHz low model if the device is slow." |

---

## 5. Common Misconceptions to Avoid

* ❌ *"eSpeak-ng generates the speech."* → ✅ eSpeak-ng only makes **phonemes**; **VITS** makes the audio.
* ❌ *"Just use the `en-in` voice."* → ✅ It **doesn't exist**; we use `en-us` + neural fine-tuning.
* ❌ *"`en-029` is Indian English."* → ✅ `en-029` is **Caribbean**.
* ❌ *"A phoneme is a letter."* → ✅ A phoneme is a **sound**; "sh" is two letters but one phoneme.
* ❌ *"Adding retroflex symbols to the config makes the accent."* → ✅ Only helps if eSpeak actually **emits** them (the custom-variant path); otherwise the accent is learned acoustically.
* ❌ *"Lower WER means it sounds more natural."* → ✅ WER measures **clarity**, not naturalness — that's what **MOS** is for.
