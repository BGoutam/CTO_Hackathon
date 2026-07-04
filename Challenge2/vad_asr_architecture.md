# Architecture Diagram: Client-Side VAD & Speech Recognition

**Challenge 2 — Deliverable (Architecture Diagram)** · Supports **Tasks 1–4**

> **What this document covers:** three views of the same system — the **audio node graph** (how sound flows), the **state machine** (how the conversation is governed), and the **event sequence** (what happens during one utterance). Read the Technical Design Document first for the VAD/ASR rationale.

---

## 1. Web Audio Node Graph

Raw microphone audio is filtered, resampled to 16 kHz, and sliced into fixed **512-sample** frames for Silero VAD. A ring buffer holds recent audio for pre-roll.

```mermaid
graph LR
    Mic([Microphone Input<br/>~48 kHz]) --> Source[MediaStreamAudioSourceNode]
    Source --> Filter[BiquadFilterNode<br/>Band-pass 300-3400 Hz]

    subgraph Processor [Audio Processing]
        Filter --> Proc["ScriptProcessor / AudioWorklet<br/>(power-of-2 block, e.g. 2048)"]
        Proc --> Resample[Resample to 16 kHz]
        Resample --> Ring[Ring buffer]
        Ring --> Frame[Slice fixed 512-sample frames]
    end

    Frame -->|Float32 512-sample frame| ONNX[ONNX Runtime Web<br/>Silero VAD v5]
    ONNX -->|P_speech per 32 ms| Controller[Pipeline State Controller]
    Ring -->|~500 ms pre-roll on trigger| ASR[Speech Recognizer<br/>Whisper / Web Speech]
    Controller -->|start / stop| ASR
    Proc --> Analyser[AnalyserNode -> waveform canvas]
```

> **Two things the original diagram got wrong, now fixed:** (1) frames are **512 samples**, not 1536 — Silero v5's fixed requirement; (2) the audio block size and the model frame size are **decoupled** via the ring buffer, because the ScriptProcessor block must be a power of two while the model needs exactly 512.

---

## 2. Pipeline State Machine

The controller governs every transition. Note the **barge-in edge** (`ASSISTANT_SPEAKING → USER_SPEAKING`) — the reliability feature the PoC demonstrates live.

```mermaid
stateDiagram-v2
    [*] --> IDLE : Init Audio + ONNX VAD

    IDLE --> USER_SPEAKING : Speech Start (P>0.5 x2 frames)
    USER_SPEAKING --> USER_SPEAKING : Stream / buffer audio

    USER_SPEAKING --> TRANSCRIBING : Speech End (P<0.35 for 800 ms)
    TRANSCRIBING --> PROCESSING : ASR returns text
    TRANSCRIBING --> IDLE : ASR empty (noise / click)

    PROCESSING --> ASSISTANT_SPEAKING : Assistant responds (TTS)
    PROCESSING --> IDLE : Error / timeout

    ASSISTANT_SPEAKING --> USER_SPEAKING : Barge-in (user speaks -> cancel TTS)
    ASSISTANT_SPEAKING --> IDLE : Playback complete
```

---

## 3. Real-Time Conversational Event Flow

One full turn, including the assistant's spoken reply and a possible barge-in.

```mermaid
sequenceDiagram
    participant Mic as Audio Hardware
    participant VAD as Silero VAD (ONNX)
    participant Ctrl as Pipeline Controller
    participant ASR as Recognizer
    participant TTS as Assistant Voice (TTS)
    participant UI as Browser UI

    Note over Mic,ASR: User starts speaking
    Mic->>VAD: 16 kHz audio, 512-sample frames
    VAD->>Ctrl: P_speech = 0.85
    Note over Ctrl: >0.5 for 2 consecutive frames
    Ctrl->>ASR: Start session (prepend ~500 ms pre-roll)
    Ctrl->>UI: "Listening..." + live meters

    Note over Mic,ASR: User pauses (end of sentence)
    Mic->>VAD: near-silent frames
    VAD->>Ctrl: P_speech = 0.12
    Note over Ctrl: <0.35 sustained 800 ms
    Ctrl->>ASR: Stop + request final transcript
    Ctrl->>UI: "Transcribing..."
    ASR->>Ctrl: "Hello voice assistant"
    Ctrl->>UI: Show transcript
    Ctrl->>TTS: Speak assistant reply
    TTS->>UI: "Assistant speaking..."

    opt User interrupts
        Mic->>VAD: speech during playback
        VAD->>Ctrl: P_speech = 0.90
        Ctrl->>TTS: Cancel playback (barge-in)
        Ctrl->>ASR: Start new turn
    end
```
