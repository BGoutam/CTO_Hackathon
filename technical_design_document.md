# Technical Design Document: Indian English Voice Adaptation for Piper TTS

**Challenge 1 — Deliverable 1 of 4** · Primarily answers **Task 1 (Pipeline Analysis)**

> **What this document covers:** how the Piper + eSpeak-ng pipeline works, which components control pronunciation, accent, and naturalness, how the two tools interact during training and inference, and exactly which files/configs must change to support Indian English. Concrete code changes are given in Section 4.

---

## 0. The One Idea That Frames Everything

Piper generates speech in **two clearly separated stages**, and understanding the split is the key to this challenge:

| Stage | Tool | Decides | Analogy |
| :--- | :--- | :--- | :--- |
| **1. Phonemization** (text → phonemes) | **eSpeak-ng** | *Which* sounds to make (the phoneme symbols) | The **script** an actor is handed |
| **2. Acoustic synthesis** (phonemes → audio) | **VITS neural model** | *How* those sounds are actually voiced — timbre, pitch, rhythm, accent | The **actor's performance** of that script |

Indian English differs from US/UK English in **both** stages: some sounds are chosen differently (e.g., retroflex consonants), and the same sounds are *realized* differently (syllable-timed rhythm, distinct intonation). A complete solution therefore has to address both stages — this is the backbone of the whole design.

---

## 1. Piper & eSpeak-ng Pipeline Interaction

The pipeline is a two-stage process: **Text → Phonemes** (`eSpeak-ng`) followed by **Phonemes → Audio** (`VITS`).

```
+-----------+   Text   +-----------+  IPA Phonemes  +---------------+  Integer IDs  +---------------+   Audio
| Input Text| -------> | eSpeak-ng | -------------> | Phoneme→ID    | ------------> | VITS Model    | --------> Waveform
|           |          | (voice)   |                | Map (config)  |               | (ONNX/PyTorch)|
+-----------+          +-----------+                +---------------+               +---------------+
   "150 crore"      "wʌ n hʌndrəd ..."           [22, 15, 14, ...]              float32 PCM samples
```

### A. Training-Phase Interaction
1. **Transcription normalization** — numbers, symbols, and abbreviations in the transcripts are expanded to spoken words (`₹150` → "one hundred fifty rupees").
2. **Phonemization** — normalized text is passed to `eSpeak-ng`, which emits an **IPA phoneme string**.
3. **Phoneme→ID mapping** — Piper maps each IPA symbol to an integer using the `phoneme_id_map` in the model's `config.json`.
4. **Model training** — VITS learns to map phoneme-ID sequences to the acoustic features extracted from the matching target-audio `.wav` files.

### B. Inference-Phase Interaction
1. **Client-side phonemization** — in the browser, a WebAssembly build of `eSpeak-ng` converts the input string to the same IPA phoneme string.
2. **Numerical conversion** — the IPA string is mapped to phoneme IDs using the *identical* map from training.
3. **Neural inference** — the phoneme IDs are fed as a tensor into the VITS ONNX model.
4. **Waveform production** — VITS outputs a raw float32 PCM audio array directly.

> ⚠️ **Critical consistency rule:** the eSpeak-ng configuration used at **inference must be byte-for-byte identical** to the one used at **training** — same voice, same dictionary, same version. If training phonemizes "crore" one way and the browser does it another, the model receives phoneme IDs it never trained on and the output is garbled. This single rule governs much of the deployment design (see Deliverable 4).

---

## 2. Correcting a Key Assumption: eSpeak-ng Has No `en-in` Voice

A natural first instinct is to "just set the eSpeak voice to `en-in`." **This does not exist.** eSpeak-ng ships only the following English variants (verified against the eSpeak-ng `docs/languages.md`):

`en-us` (American) · `en` (British, default) · `en-029` (Caribbean) · `en-gb-x-rp` (Received Pronunciation) · `en-gb-scotland` · `en-gb-x-gbclan` (Lancastrian) · `en-gb-x-gbcwmd` (West Midlands)

There is **no Indian-accented English identifier**. (India-related eSpeak voices such as `hi`, `ta`, `bn` are for *Indian languages*, not Indian-accented English; `en-029` is Caribbean, not Indian — a common mix-up.)

This constraint is not a dead end — it defines two legitimate strategies, and choosing between them is the central design decision:

### Path A — Acoustic Adaptation *(recommended default)*
- **Phonemize with the stock `en-us` (or `en`) voice.** The phoneme *inventory* stays standard.
- Fine-tune VITS on an **Indian-English dataset** so the model learns the Indian *acoustic realization* — retroflex-colored consonants, syllable-timed rhythm, and intonation — as properties tied to the standard phonemes.
- **Pros:** simple, robust, no eSpeak recompilation, no phoneme-map surgery, and the browser only needs the stock eSpeak-ng WASM build. **Cons:** the model cannot *contrast* sounds that eSpeak collapses into one symbol.

### Path B — Custom eSpeak-ng Indian-English Variant *(advanced, higher fidelity)*
- **Author a new variant voice** (a small voice file that inherits from `en` with phoneme-substitution rules) plus custom dictionary entries, so eSpeak *emits* Indian-specific IPA (e.g., retroflex `/ʈ/`, `/ɖ/`) and correct loanword pronunciations.
- This requires recompiling the eSpeak-ng dictionary **and** extending `phoneme_id_map` to include the new symbols — and the same custom data must ship in the browser WASM build.
- **Pros:** finer phonemic control, correct-by-construction loanwords. **Cons:** more engineering; train/inference consistency becomes harder to guarantee.

> **Design recommendation:** start with **Path A** to get a strong Indian-English voice quickly, and layer in **Path B** selectively — a custom dictionary for loanwords/names (high value, low risk) before full retroflex phoneme rules (high value, higher risk).

---

## 3. Components Influencing Speech Quality

### A. Pronunciation Quality — *Phonemic Correctness* (owned by eSpeak-ng)
* **Lexicon & rules:** correctness is decided almost entirely at phonemization. If eSpeak maps a word to the wrong IPA, VITS will faithfully synthesize the wrong sound — the neural model cannot "fix" a bad phoneme.
* **Letter-to-Sound (LTS) rules:** words not in the dictionary are pronounced via language rules in `en_rules`.
* **Files that control this:** `dictsource/en_list` (dictionary lookups / exceptions), `dictsource/en_rules` (grapheme→phoneme rules), and any custom user lexicon applied before eSpeak.

### B. Accent Characteristics — *Acoustic Realization* (owned by VITS, optionally aided by eSpeak)
* **Phoneme inventory:** Indian English uses retroflex plosives `/ʈ/`, `/ɖ/` where US/UK English uses alveolar `/t/`, `/d/`. Whether the model treats these as *distinct phonemes* depends on Path A vs. Path B above.
* **Prosody & rhythm:** Indian English is largely **syllable-timed** (syllables take roughly equal time) rather than **stress-timed**. This is learned acoustically by VITS from the dataset.
* **VITS duration & pitch predictors:** dedicated sub-networks predict per-phoneme duration and the fundamental-frequency (F0/pitch) contour. Fine-tuning teaches them the Indian speaker's cadence and pitch movement.

### C. Speech Naturalness — *Audio Fidelity* (owned by VITS)
* **HiFi-GAN vocoder (the VITS decoder):** converts the model's latent features into the final waveform. Naturalness scales with sample rate (22.05 kHz vs. 16 kHz) and the acoustic diversity of the training data.
* **Normalizing flows:** VITS uses a variational autoencoder with normalizing flows to model the one-to-many mapping from static text to expressive speech — this is what prevents flat, robotic output.

---

## 4. Files, Configurations & Modules Requiring Customization

| Module / Component | Target File / Parameter | Modification | Path |
| :--- | :--- | :--- | :--- |
| **Piper training + config** | `config.json` → `"espeak": {"voice": "en-us"}` | Use a **real** eSpeak voice. Rely on the dataset to teach the accent. | A |
| **eSpeak-ng (dictionary)** | `dictsource/en_list` | Append Indian loanwords, acronyms, and names ("crore", "lakh", "panchayat", "Aadhaar") with correct IPA. | A + B |
| **eSpeak-ng (custom variant)** | new file under `espeak-ng-data/lang/gmw/` (e.g. `en-in`) + `phsource/phonemes` | Define an Indian-English variant that inherits from `en` and emits retroflex `/ʈ/`, `/ɖ/`. | B only |
| **Piper config (phoneme map)** | `config.json` → `phoneme_id_map` | **Only if Path B:** add IDs for the new retroflex symbols eSpeak now emits. Under Path A the map stays standard. | B only |
| **Piper preprocessing** | `piper_train.preprocess` args | Set `--language en-us`; ensure text normalization handles Indian formats (e.g., `1,00,000` → "one lakh"). | A |

> **Note on `phoneme_id_map`:** adding retroflex IDs is *only meaningful* if eSpeak actually emits those symbols (Path B). Under Path A, eSpeak never produces `/ʈ/`, so extra map entries would sit unused. The two changes must be made together or not at all.

---

## 5. Concrete Code & Configuration Changes

### A. eSpeak-ng Dictionary Customization (`dictsource/en_list`) — Path A + B
Add Indian loanwords, acronyms, and names so they are pronounced correctly rather than guessed by letter-to-sound rules:

```
// File: dictsource/en_list
crore      kr'oː
lakh       l'aːk
aadhaar    aːdʱ'aːr
bazaar     bəz'aːr
paneer     pən'iːr
masala     məs'aːlaː
goutam     g'oːtəm
```

Recompile the English dictionary after editing (this produces `espeak-ng-data/en_dict`):
```bash
cd espeak-ng/dictsource
espeak-ng --compile=en
```

Verify a specific word phonemizes as intended:
```bash
espeak-ng -v en-us -q --ipa "150 crore"
```

---

### B. Piper Model Configuration (`config.json`)
Set a **real** eSpeak voice. The `phoneme_id_map` below shows the **Path B** case, where a custom variant emits retroflex symbols; under **Path A** you would keep the stock map and *omit* the retroflex additions.

```json
{
  "audio": { "sample_rate": 22050 },
  "espeak": { "voice": "en-us" },
  "phoneme_type": "espeak",
  "phoneme_id_map": {
    " ": [0], "^": [1], "$": [2],
    "a": [3], "b": [4], "d": [5], "e": [6], "f": [7], "g": [8],
    "h": [9], "i": [10], "k": [11], "l": [12], "m": [13], "n": [14],
    "o": [15], "p": [16], "r": [17], "s": [18], "t": [19], "u": [20],
    "v": [21], "w": [22], "z": [23],
    "æ": [24], "ð": [25], "ŋ": [26], "ɐ": [27], "ɑ": [28], "ɒ": [29],
    "ɔ": [30], "ə": [31], "ɛ": [32], "ɪ": [33], "ʃ": [34], "ʊ": [35],
    "ʌ": [36], "ʒ": [37], "θ": [38],

    "ʈ": [39], "ɖ": [40], "ɭ": [41], "ɳ": [42]
  }
}
```
*The last row (retroflex `/ʈ/`, `/ɖ/`, `/ɭ/`, `/ɳ/`) is added **only** for Path B, and only because a custom eSpeak variant will emit them. Adding IDs the phonemizer never produces has no effect.*

---

### C. Piper Preprocessing Command
Run inside the Piper training repo, using a valid eSpeak voice:

```bash
python3 -m piper_train.preprocess \
    --language en-us \
    --input-dir ./datasets/indian_english_voice/ \
    --output-dir ./preprocessed_data/ \
    --dataset-format ljspeech \
    --single-speaker \
    --sample-rate 22050
```
*`--language` refers to an **eSpeak-ng voice identifier**, so it uses the same name space as Section 2 — `en-in` is not valid here either. Output: `config.json`, `dataset.jsonl`, and per-utterance audio tensors that feed `piper_train`.*

---

### D. Phonemizing with libespeak-ng in Python (conceptual)
During preprocessing Piper calls the eSpeak C library. This illustrates setting a valid voice and extracting phonemes:

```python
# File: piper_train/phonemize.py (illustrative)
import ctypes

libespeak = ctypes.cdll.LoadLibrary("libespeak-ng.so")
libespeak.espeak_Initialize(0, 0, None, 0)

# Use a REAL eSpeak voice. For Path B, this would be your compiled custom "en-in" variant.
voice_name = b"en-us"
if libespeak.espeak_SetVoiceByName(voice_name) != 0:
    raise ValueError(f"Could not load espeak voice: {voice_name.decode()}")

sample_text = "Welcome to the bazaar."
libespeak.espeak_Synth(
    sample_text.encode("utf-8"),
    0, 0, 0, 0,
    0x1000,        # espeakCHARS_UTF8
    None, None,
)
# eSpeak returns an IPA phoneme string, e.g. "w'ɛlkʌm tə ðə bəz'aːr",
# which Piper then maps to integer IDs via phoneme_id_map.
```

---

## 6. Task-1 Summary

- **Pipeline:** two stages — eSpeak-ng (text→phonemes) then VITS (phonemes→audio); the same phonemization must be used in training and inference.
- **Pronunciation** is owned by eSpeak-ng (`en_list`, `en_rules`); **accent** and **naturalness** are owned by the VITS acoustic model (duration/pitch predictors, HiFi-GAN, normalizing flows).
- **There is no `en-in` voice.** The design therefore uses stock `en-us` phonemization plus dataset-driven acoustic fine-tuning (**Path A**), with an optional custom eSpeak variant for retroflex/loanword fidelity (**Path B**).
- **Files to change:** `config.json` (`espeak.voice`, and `phoneme_id_map` only under Path B), `dictsource/en_list` (loanwords), and the preprocessing invocation.
