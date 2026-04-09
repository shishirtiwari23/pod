# Voice AI Industry Standards Reference
## Immutable Implementation Guide — Do Not Modify Without Explicit Instruction

**Status:** Locked reference document. This captures industry-standard approaches as of 2026. No implementation detail in this document should be changed, substituted, or "improved upon" unless the user explicitly requests a revision. Treat every specification here as a hard constraint, not a suggestion.

---

## 0. Why This Document Exists

Building a voice AI assistant that feels genuinely human requires solving a dense stack of interacting problems simultaneously. Each layer — audio capture, echo cancellation, speech detection, streaming transport, transcription, turn detection, LLM integration, speech synthesis, and interruption handling — has its own failure modes and its own industry-validated solutions. This document captures those solutions in full, including every edge case, the exact numbers that matter, and how the industry leaders handle each problem. Nothing here is opinion. Every specification is grounded in production deployments from OpenAI Realtime API, Deepgram, ElevenLabs, Google Gemini Live, Pipecat, LiveKit, Silero VAD, and peer-reviewed audio research published through IEEE and arXiv.

---

## 1. The Human Conversation Timing Constraint

This is the single most important number in the entire system. According to NIH research (Stivers et al., 2009), **the median gap between conversational turns in human speech is approximately 200 milliseconds**. Users begin perceiving a response as unnatural at delays beyond 300–500ms. Interactions feel broken beyond 800ms.

This creates a hard latency budget that every layer must respect:

| Component | Target Latency | Hard Maximum |
|---|---|---|
| VAD speech detection | <100ms | 150ms |
| STT transcript (final) | 150–300ms | 500ms |
| LLM time-to-first-token | 100–350ms | 600ms |
| TTS time-to-first-audio | 75–200ms | 300ms |
| **End-to-end voice-to-voice** | **<500ms** | **800ms** |
| Barge-in stop latency | <200ms | 300ms |

These are not aspirational targets. They are the thresholds at which the system is distinguishable from human conversation. Any architecture or component choice that blows past the 800ms total budget must be rejected or re-architected.

---

## 2. Audio Pre-Processing Pipeline

### 2.1 The Mandatory Processing Chain

Every audio frame captured from the microphone must pass through this exact sequence before it touches STT or VAD:

```
Microphone capture → AEC → Noise Suppression → Beamforming (if multi-mic) → VAD → STT
```

No step is optional. Skipping AEC means the assistant hears itself. Skipping noise suppression means the STT degrades in real-world environments. This chain operates in real time, frame by frame.

### 2.2 Acoustic Echo Cancellation (AEC)

**The problem:** When the assistant is speaking and the microphone is open simultaneously (full-duplex), the microphone picks up the speaker output. Without AEC, the STT receives both the user's voice and the assistant's own TTS audio, causing the assistant to transcribe itself, interrupt itself, or loop into feedback. This is the most commonly under-solved problem in voice AI implementations.

**The specific web implementation problem:** The browser's built-in AEC (`getUserMedia` with `echoCancellation: true`) is designed for human-to-human video calls. It works by monitoring what the browser plays through a known audio element or Web Audio API. When TTS audio is delivered as raw PCM chunks over a WebSocket, decoded manually, and scheduled through AudioContext buffers, the browser's AEC has no visibility into that audio stream and cannot cancel it. This is not a configuration issue — it is a fundamental architectural limitation of browser-native AEC.

**The industry solution for WebSocket-delivered TTS audio:**
1. Route TTS audio playback through an `AudioContext` output node that the browser's AEC can reference, OR
2. Implement a software AEC using the reference signal — maintain a digital copy of the audio being played, feed it as the "far-end reference" to an adaptive filter, and subtract the estimated echo from the microphone signal before it reaches STT.
3. Apply a two-tier RMS gate: a primary gate that mutes microphone input while TTS audio energy exceeds a threshold, and a cooldown period of 200–400ms after TTS audio ends to account for room resonance decay. This prevents the residual acoustic tail from triggering false speech detection.

**The adaptive filter approach (industry standard):**
Modern AEC uses the Normalized Least Mean Square (NLMS) adaptive algorithm. The system maintains a digital reference of the signal being played (the "far-end signal"), continuously updates filter coefficients to model the current acoustic path, and subtracts the predicted echo from the microphone input. Filter coefficients must update in real time because the acoustic environment changes — people move, doors open, rooms change. Static coefficients are insufficient.

For AI-based AEC (higher quality, higher compute): RNN-based and transformer-based models (e.g., DeepFilterNet, NVIDIA Maxine) can model nonlinear distortions and long-latency echo paths better than adaptive filters. These are appropriate for production deployments where compute is available.

**Key metrics to target:**
- Echo Return Loss Enhancement (ERLE): minimum 25 dB (ITU G.168 requirement)
- AEC processing latency contribution: <10ms
- Must handle double-talk (user speaking while assistant is speaking) without clipping the user

**Do not apply Acoustic Echo Suppression (AES) as post-processing to the STT audio stream.** AES is a nonlinear process and corrupts the signal for ASR neural networks. AES is acceptable only on the telephony/output path — never on the ASR input path.

### 2.3 Noise Suppression

Background noise causes two distinct problems: it degrades STT accuracy, and it triggers false-positive VAD events, preventing end-of-speech detection from firing.

Industry standard implementations use hybrid DSP-neural approaches:
- **RNNoise / PercepNet**: lightweight hybrid models (~100KB), suitable for edge/mobile
- **DeepFilterNet**: higher quality neural approach, suitable for server-side processing
- **NVIDIA Riva/Maxine**: proprietary, highest quality, requires GPU

Target SNR improvement: 20+ dB. The system must remain below 3dB SNR degradation on the VAD input signal even in environments like cafes, cars, and open offices.

### 2.4 Audio Format Standards

- **Sample rate**: 16 kHz minimum for STT (most production models including Whisper, Nova-3, and Gemini are trained on 16 kHz). 24 kHz for TTS output (higher quality, acceptable bandwidth).
- **Bit depth**: 16-bit PCM (Int16) for transmission, convert to Float32 for model inference.
- **Chunk size for streaming**: 20ms frames for VAD processing (standard telephony frame size). Larger chunks (100–200ms) for STT streaming to balance latency with transcription coherence.
- **PSTN/telephony caveat**: Standard phone networks use 8 kHz G.711 codec. This severely degrades both STT accuracy and TTS naturalness compared to 16 kHz web audio. For phone deployments, use telephony-specific STT models and do not use speech-to-speech APIs (their latency and quality advantages disappear over PSTN while their costs remain high).

---

## 3. WebSocket Streaming Architecture

### 3.1 Why WebSockets, Not REST APIs

REST API calls introduce TCP handshake overhead on every request, making them structurally incompatible with continuous audio streaming. For voice AI:

- **WebSocket** maintains a persistent bidirectional connection for the duration of the session, eliminating connection overhead per audio packet. This is the universal standard for server-to-AI-service connections.
- **WebRTC** is the standard for client-to-server audio transport in browsers and mobile apps. It runs over UDP (tolerates packet loss, prioritizes speed), includes built-in AEC/AGC/noise reduction, and handles jitter buffering. The typical production architecture combines both: **WebRTC from client to relay server, then WebSocket from relay server to AI services**.

### 3.2 Audio Packet Streaming Protocol

The standard pipeline:

1. **Client side**: `getUserMedia` captures microphone audio → resampled to 16 kHz Int16 PCM → divided into 20ms frames → sent over WebSocket as binary data.
2. **Server side**: Receives binary audio frames → converts Int16 to Float32 → feeds into VAD + STT pipeline in parallel.
3. **Interim results**: STT returns partial transcriptions as audio arrives. These are labeled `interim` and must not be sent to the LLM. Only `final` or `speech_final` flagged transcripts proceed to LLM.
4. **Reconnection**: WebSocket connections must implement automatic reconnection with exponential backoff. Voice sessions must survive network interruptions without losing conversation context.

### 3.3 Audio Multiplexing

For systems handling concurrent users: do not open one WebSocket per user at scale. Use connection multiplexing — route multiple isolated audio streams through shared WebSocket connections with per-stream identifiers. This is how production platforms (Deepgram, Together.ai) handle hundreds of simultaneous calls without exhausting connection limits.

### 3.4 Packet Loss Handling

- **Input (WebRTC)**: WebRTC handles packet loss through built-in loss concealment, filling missing chunks with plausible noise.
- **Input (WebSocket)**: WebSocket implementations lack loss concealment, but ASR systems tolerate minor gaps on reliable networks. For unreliable networks (mobile), implement redundant packets or forward error correction at the application layer.
- **Output (TTS playback)**: Missing output packets cause audio blips that break the naturalness illusion. Buffer a 200ms TTS audio prefetch to absorb jitter before playback begins.

---

## 4. Speech-to-Text (STT) Streaming

### 4.1 Architecture: Streaming Over Batch

Never use batch (request/response) STT in a voice AI pipeline. Always use streaming STT over WebSocket. The difference: batch STT waits for the full audio file before returning anything. Streaming STT returns partial and final transcripts as audio arrives, enabling the system to begin LLM processing before the user has finished speaking.

**The "flush trick"**: When VAD detects end of speech, immediately flush the STT audio buffer without waiting for the model's internal timeout. This cuts STT end-to-final latency from ~500ms to ~125ms. This single optimization is responsible for a majority of the latency reduction in production voice agents.

### 4.2 Interim vs. Final Transcripts

- **Interim results**: Partial transcriptions produced while the user is still speaking. Used for live display and to begin LLM context preparation. **Never send to LLM as the turn input.**
- **Final results (`is_final: true` or `speech_final: true`)**: The locked, complete transcription of a speech segment. This is what gets sent to the LLM.
- **Immutable transcripts**: Some STT providers (AssemblyAI Universal-Streaming) produce "immutable" transcripts in ~300ms — locked output that does not change as more audio arrives. These eliminate the problem of LLM receiving an outdated partial transcript.

### 4.3 Word Error Rate (WER) Reality

WER numbers in marketing materials are measured on clean studio audio with native English speakers. Production reality:

- Real-world WER is 2–4x higher than benchmark WER in noisy environments
- Accented speech adds additional 15–30% error rate without accent-specific fine-tuning
- Signal-to-noise ratio below 3 dB causes sharp accuracy degradation in all models
- Background chatter ("talking noise") is more damaging than stationary noise (hum, AC)

Target: <7% WER on real-world streaming audio for production deployment. Choose STT providers by benchmarking on audio samples representative of your actual users, not on public benchmarks.

### 4.4 Hallucination on Silence

A critical edge case: when played silence or background noise, some STT models (including older Whisper versions) generate hallucinated text. OpenAI's 2025-12-15 model snapshots achieve ~90% fewer hallucinations on silence compared to Whisper v2. Always test and benchmark hallucination behavior on silence inputs specifically, as these generate false triggers that fire the entire LLM+TTS pipeline for nothing.

---

## 5. Voice Activity Detection (VAD) and End-of-Turn Detection

### 5.1 VAD Is Not Just Silence Detection

This is the most under-specified component in most implementations. VAD has two distinct jobs:
1. **Speech onset detection** — detecting when the user starts talking
2. **End-of-turn detection** — detecting when the user has *finished* their thought and it is safe to respond

These are different problems with different solutions. Using only energy-based VAD for end-of-turn detection produces either premature cutoffs (interrupting the user) or excessive dead air (waiting for silence that already passed). Industry standard in 2026 uses a dual-signal approach.

### 5.2 VAD Implementation Standard

**Do not use energy/RMS threshold VAD alone.** Use probability-based VAD:

- **Silero VAD** (open source): Bayesian/probability-based, runs in real time on any hardware, 85–100ms detection latency, achieves 95%+ accuracy in distinguishing speech from background noise. Operates on 20ms audio frames, returns a probability score 0–1. Use a configurable threshold (default ~0.5) rather than a hard binary cut.
- **WebRTC's built-in VAD**: Adequate for basic applications, not sufficient for production voice AI with noisy environments.

**VAD parameters (tunable per deployment):**
- `threshold`: Activation threshold (0–1). Higher = requires louder audio, better for noisy environments. Default 0.5.
- `prefix_padding_ms`: Audio to include before detected speech onset. Prevents clipping the first syllable. Default 200–300ms.
- `silence_duration_ms`: Silence duration before declaring speech stopped. 300–500ms for natural conversation, up to 1000ms for voice journaling or deliberate speakers.

**Adaptive VAD**: Production systems must dynamically update VAD thresholds based on measured background noise at session start and during the session. A user in a coffee shop needs different thresholds than a user in a quiet office. Fixed thresholds fail half the user population.

### 5.3 Three-State VAD Machine

Implement VAD as an explicit state machine with exactly three states:

```
SILENCE → SPEECH → HANGOVER → SILENCE
```

- **SILENCE**: No user speech. Microphone active, AEC running, VAD polling.
- **SPEECH**: User is speaking. Pre-roll buffer active (preserves audio from before onset detection). STT receiving frames.
- **HANGOVER**: Post-speech window. Allows for natural pauses mid-sentence without triggering end-of-turn. Duration: 300–500ms configurable. Pre-roll buffer continues capturing.

When transitioning from HANGOVER to SILENCE: this is end-of-turn. Flush the audio buffer, send to STT for final transcript, pass to LLM.

### 5.4 Semantic VAD (End-of-Turn Intelligence)

Pure acoustic VAD fires on acoustic silence, not on semantic completion. A user saying "I want to book a flight to... um..." (natural pause) should not trigger end-of-turn. A user saying "Okay thanks, bye" should, even if they pause mid-sentence.

**Semantic VAD** (available in OpenAI Realtime API as `semantic_vad` mode) analyzes the *words* being said — not just the audio energy — to determine whether the speaker has completed their utterance. This is the 2026 industry standard for natural-feeling turn detection. Parameters:
- `eagerness`: Controls how quickly the model interprets silence as end-of-turn. `high` = faster response, more interruption risk. `low` = waits longer, more natural for deliberate speakers. `auto` (default) = adaptive.

**Hybrid end-of-turn detection**: Combine acoustic VAD with semantic analysis for best results:
1. Acoustic VAD detects silence onset
2. Semantic model evaluates whether the transcript so far appears complete
3. If semantic confidence is high + silence duration > threshold → end-of-turn
4. If semantic confidence is low (sentence feels incomplete) → extend HANGOVER window

### 5.5 VAD Edge Cases

| Edge Case | Problem | Solution |
|---|---|---|
| Background music/TV | VAD never reaches SILENCE, preventing end-of-turn | AEC + noise suppression before VAD; implement utterance_end_ms timeout as fallback |
| User coughs or laughs mid-sentence | False end-of-turn trigger | Minimum speech duration gate: require ≥300ms of speech before HANGOVER is eligible |
| User says "um", "uh", thinking | Premature response | Semantic VAD; increase hangover duration; detect disfluency markers |
| Drive-through / fast food noise | Continuous VAD trigger from background | Adaptive threshold calibration at session start |
| Multiple speakers in room | Confuses VAD and STT | Speaker diarization (adds latency); recommend single-speaker assumption for consumer voice AI |
| Phone ringing in background | False trigger | Narrow frequency analysis; noise suppression |

---

## 6. LLM Integration for Voice

### 6.1 The Context Problem

Voice AI fails in the same places: long conversations, silence edge cases, and tool-driven flows. These are the production failure modes confirmed by OpenAI in their 2025 developer updates.

**Instruction following degrades with context length.** GPT-4o achieves 72% overall accuracy on the Berkeley Function-Calling Leaderboard but drops to 50% on the multi-turn subset. GPT-4o-mini drops to 34% on multi-turn. This means the LLM that works well in demo conversations degrades significantly in production multi-turn voice interactions. Mitigation: implement in-session context engineering — compress and focus conversation context at defined points in the workflow. Do not let raw conversation history accumulate unboundedly.

**Context compression strategy:**
```
Keep: last 3 user goals + last confirmed constraint + rolling LLM-generated summary
Drop: verbose turn-by-turn history beyond a configurable window
```

### 6.2 LLM Must Know Its I/O Modality

Always include in the system prompt that:
1. Input comes from an ASR/STT model — the LLM should anticipate transcription errors, mispronunciations, and missing punctuation in the input.
2. Output will be converted to TTS — the LLM must never include markdown, code blocks, bullet characters, URLs, emoji, or any formatting that sounds unnatural when spoken aloud. Responses must be written as natural speech prose.

### 6.3 Response Design for Voice

Voice responses are fundamentally different from text responses:
- **Brevity**: Responses must be interruptible. Long monologues are bad conversation design. Structure for natural break points.
- **No lists or bullets**: Convert all lists to spoken prose ("There are three options. First... Second... Third...")
- **No URLs or markdown**: These are nonsensical when read aloud.
- **Filler/thinking sounds**: Insert "Let me check on that for you" or brief placeholder speech before long tool calls. Never leave dead air beyond 1 second.
- **Sentence boundaries for TTS chunking**: The LLM should structure output so sentence boundaries align with natural TTS streaming chunks. Short sentences stream faster than long complex sentences.

### 6.4 Async Tool Calling

External tool calls (database lookups, API calls, RAG) must never be made synchronously in the main conversation loop. They violate the latency budget. The production pattern:

1. LLM detects tool call is needed
2. Immediately output a placeholder message ("One moment while I look that up")
3. Execute tool call asynchronously on a parallel inference pipeline
4. When tool result arrives, inject it into the conversation context
5. Resume or re-trigger LLM generation with the result

Do not run tool calls from the main conversation loop. A dedicated parallel tool-calling pipeline that inserts context from outside the main loop is the correct architecture.

---

## 7. Text-to-Speech (TTS) Streaming

### 7.1 The Three TTS Modes

Never use Single Synthesis (mode 1) in a voice agent. Always use Dual Streaming (mode 3):

| Mode | Input | Output | Voice-to-Voice Time |
|---|---|---|---|
| **Single Synthesis** | Complete text | Complete audio file | 3+ seconds |
| **Output Streaming** | Complete text | Audio chunks streamed | 1.5+ seconds |
| **Dual Streaming** | Text fed token-by-token | Audio chunks as soon as enough context exists | 225–400ms |

**Dual Streaming** (also called PADRI — Parallel Async Dual-streaming Real-time Inference) is the 2026 industry standard for conversational AI. As the LLM generates tokens, they are fed immediately to the TTS engine word-by-word or character-by-character. The TTS engine begins audio synthesis as soon as it has enough context for natural-sounding phonemization (typically the first complete phrase or clause). Audio chunks of 50–100ms windows are returned and streamed to the user before the LLM has even finished generating the response. The user hears the first word within 225–300ms of speech detection end.

### 7.2 TTS Streaming Architecture

```
LLM token stream
    ↓
Token buffer (accumulate until punctuation or ~64 tokens)
    ↓
Phonemization (convert buffered text to phonetic units)
    ↓
Model inference (generate mel-spectrogram)
    ↓
Waveform generation (vocoder converts mel-spec to PCM)
    ↓
Audio chunk stream (50–100ms windows over WebSocket)
    ↓
Client playback with 200ms jitter buffer
```

**Token buffering strategy**: Buffer tokens until a sentence-ending punctuation mark (`.`, `?`, `!`) or until buffer exceeds 64 tokens. This ensures TTS has enough context for natural prosody without introducing unnecessary latency. Do not wait for full sentences on the first chunk — start synthesis at the first natural phrase boundary.

### 7.3 TTS Quality Standards

**Word-level timestamps are mandatory for voice AI.** Without word-level timestamps, you cannot align conversation context with what the user actually heard when they interrupt the agent. If the user interrupts at word 12 of a 20-word sentence, the LLM context must know that words 1–12 were heard and words 13–20 were not. Without this alignment, the conversation context becomes inconsistent. TTS providers with word-level timestamp support in 2026: Cartesia, ElevenLabs, Rime, Inworld.

**Prosody and naturalness**: TTS models trained on audiobook data produce "performative speech" — clear, formal, unnatural for casual conversation. For human-like conversational AI, use models trained on real conversational data including hesitations, backchannel responses ("mm-hmm"), natural pauses, and overlapping speech. This is what makes Rime's Mist v2 differentiating — it was trained on mundane conversations (drive-thrus, customer service calls) rather than studio recordings.

**Filler audio**: Never output silence while the LLM is processing. The user should hear something within 1 second of their speech ending. Options:
- Brief thinking sounds ("Hmm...", "Let me think...")
- Explicit placeholder phrases ("Just a moment...")
- Background ambient audio during long tool calls

Silence beyond 1 second is interpreted by users as system failure.

### 7.4 TTS Chunk Size and Interruption Alignment

Keep TTS output in small chunks (100–200ms) rather than large buffers. This ensures that when an interruption arrives, TTS can stop cleanly without cutting off mid-word by more than 100ms. Large TTS buffers (500ms+) make interruption stops feel laggy and unnatural.

---

## 8. Interruption Handling (Barge-In)

### 8.1 What Barge-In Requires

Barge-in is the capability for the user to interrupt the assistant mid-sentence and have the assistant immediately stop, listen, and respond. This is what separates demo-quality voice AI from production-quality voice AI.

Three components must execute simultaneously and in this order:

1. **AEC** — Remove the assistant's own voice from the incoming audio so VAD and STT are not confused by the assistant hearing itself
2. **VAD** — Detect user speech onset with probability-based detection (not energy threshold)
3. **Immediate TTS cancellation** — Stop the assistant's speech within 200ms of user speech detection

### 8.2 The Barge-In State Machine

```python
class ConversationState(Enum):
    IDLE = "idle"
    LISTENING = "listening"
    PROCESSING = "processing"    # STT final received, LLM generating
    SPEAKING = "speaking"        # TTS audio playing
    INTERRUPTED = "interrupted"  # User spoke during SPEAKING

Transitions:
    IDLE → LISTENING (session start / turn complete)
    LISTENING → PROCESSING (end-of-turn detected)
    PROCESSING → SPEAKING (first TTS chunk ready)
    SPEAKING → INTERRUPTED (VAD detects user speech)
    INTERRUPTED → LISTENING (after TTS stopped + AEC cooldown)
    PROCESSING → LISTENING (barge-in before TTS starts — cancel LLM generation)
```

### 8.3 Barge-In Guard Period

**Do not allow barge-in in the first 500–1000ms after the assistant begins speaking.** This prevents false triggers from:
- Room resonance of the assistant's first words
- The user saying "okay" or "yes" in agreement (not an interruption)
- Background noise spikes coinciding with speech onset

After the guard period expires, VAD operates continuously while TTS plays.

### 8.4 False Positive Filtering

Not all detected speech during assistant speech should trigger interruption. Apply three filters before treating audio as a barge-in:

1. **Energy threshold gate**: Is the detected speech energy meaningfully above background? (Prevents TV/radio triggering interruption)
2. **Duration gate**: Has speech been continuous for at least 200ms? (Prevents single cough or click)
3. **Semantic gate (optional, adds latency)**: Does the partial transcript contain meaningful intent? (Prevents "uh-huh", "yeah", "right" from stopping the assistant)

For customer service voice AI where users need to interrupt frequently, use gates 1 and 2 only. The semantic gate adds too much latency for rapid correction scenarios.

### 8.5 Interruption Strategies

**Hard interruption (default for conversational AI)**:
Stop TTS immediately on valid barge-in detection. Clear audio buffers. Transition to LISTENING state. Acknowledge with a brief sound or "Yes?" if appropriate. Best for: voice assistants, customer service, casual conversation.

**Sentence completion before interrupt**:
Finish the current sentence, then stop. Delays 1–3 seconds but ensures partial information delivered is coherent. Best for: educational content, step-by-step instructions, critical information delivery.

**Pause-and-resume**:
Pause TTS, save position, wait 2 seconds. If user continues → discard saved position, process interruption. If user stops → resume from saved position. Best for: cases where user may be speaking to someone else, not the assistant.

For most implementations, use hard interruption as the default. Add the pause-and-resume mode as an opt-in for specific content types where partial delivery is harmful.

### 8.6 Context Alignment After Interruption

When the user interrupts at word N of an M-word TTS response, update the LLM conversation context to reflect that only words 1 through N were heard. This requires word-level timestamps from the TTS engine (see section 7.3). Without this alignment, the LLM may repeat information the user already heard or reference content the user never received.

---

## 9. Conversation State Management

### 9.1 Design the Agent as a State Machine, Not a Stateless Chatbot

Stateless chatbots send a fresh context window on every turn. This is insufficient for voice AI because:
- Context grows unboundedly, degrading LLM instruction following
- Interruptions leave the context in mid-turn states that must be resolved
- Tool calls complete asynchronously and must be injected back into context
- Session reconnections must restore state without losing conversation position

Implement the voice agent as an explicit state machine. Every state has defined valid transitions. Every transition is logged. Invalid state combinations (e.g., SPEAKING while already SPEAKING) must throw an error, not silently proceed.

### 9.2 Memory Architecture

```
Session state (in-memory, fast):
    - Current conversation state (enum)
    - Active TTS position (word index with timestamps)
    - VAD state (SILENCE/SPEECH/HANGOVER)
    - Pending tool call results
    - Short-term turn history (last 5 turns)

Persistent state (database, cross-session):
    - User preferences (voice, speed, language)
    - Long-term memory (name, preferences, history summary)
    - Rolling conversation summary (LLM-generated, compressed)
    - Known entities and relationships
```

**Context decay policy**: Keep the last 3–5 user goals plus the last confirmed constraint plus a rolling LLM-generated summary. Drop verbose history beyond a 10-turn window. Recompress the summary every 10 turns to prevent context bloat.

### 9.3 Reconnection and Resume

When a WebSocket connection drops mid-conversation:
1. The client must reconnect within 30 seconds to resume (longer → restart session)
2. On reconnection, the server restores the last known conversation state
3. If TTS was playing when disconnection occurred, resume from last committed audio position (requires word-level timestamps)
4. If STT was receiving audio, discard incomplete audio and re-prompt user to repeat

---

## 10. Backchannel Responses and Conversational Naturalness

### 10.1 What Backchannels Are

Backchannels are brief vocal signals that indicate active listening without taking the conversation turn: "mm-hmm", "I see", "okay", "yeah, go on", "uh-huh". In human conversation, these occur during speaker pauses and signal that the listener is engaged. Their absence makes the conversation feel robotic — as if the assistant is simply waiting to reply rather than actually listening.

### 10.2 Backchannel Implementation

Detect natural pause points in user speech (mid-sentence pauses of 500ms–1s where semantic VAD indicates the sentence is not complete). During these pauses, inject a brief contextual backchannel:

- Acknowledge content: "I see", "Got it", "Okay"
- Affirmative: "Mm-hmm", "Yeah", "Right"
- Encouraging: "And then?", "Go on"

Backchannel responses must be:
1. Generated before the user finishes speaking (overlapping is acceptable)
2. Short enough not to interfere with the user's speech
3. Contextually appropriate (don't say "excellent!" to bad news)
4. Not loud enough to trigger the user's own AEC

NVIDIA PersonaPlex and Kyutai Moshi are 2026 reference implementations of backchannel-aware full-duplex models. Rime's Mist v2 was specifically trained on conversational data including backchannel patterns.

### 10.3 Turn Completion Signals

Beyond acoustic silence, human conversation uses multiple signals to indicate turn transfer:
- **Falling intonation** (tone drops at end of sentence) → speaker is yielding
- **Completed syntactic structure** (sentence grammatically complete) → semantic VAD signal
- **Gaze or gesture cues** (not applicable to voice-only systems)
- **Explicit yield phrases**: "So anyway...", "What do you think?", "Does that make sense?"

Implement recognition of explicit yield phrases as a high-confidence end-of-turn trigger that bypasses the normal silence timeout.

---

## 11. Network Protocol Selection

| Scenario | Protocol | Reason |
|---|---|---|
| Browser microphone to server | WebRTC | Built-in AEC/AGC, UDP for low latency, packet loss tolerance |
| Mobile app microphone to server | WebRTC | Same as browser, native SDK support |
| Server to AI service (STT, LLM, TTS) | WebSocket | Persistent connection, binary audio, bidirectional streaming |
| High-scale multiplexed connections | WebSocket with stream multiplexing | One connection per service, not per user |
| Phone/PSTN | Twilio/SIP WebSocket (8 kHz G.711) | Telephony network constraint; use cascaded STT→LLM→TTS, not speech-to-speech |
| Edge/local inference | Local IPC or UNIX socket | Eliminates network overhead entirely |

**Hybrid pattern (production standard)**: WebRTC from client to relay server (handles unpredictable client network conditions) + WebSocket from relay server to AI APIs (simplifies AI model interfacing). OpenAI Realtime API's reference architecture follows this hybrid pattern.

---

## 12. Architecture Decision: Cascaded Pipeline vs. Speech-to-Speech

### 12.1 Cascaded Pipeline (STT → LLM → TTS)

**When to use**: Default choice for most production voice AI today.

Advantages:
- Maximum flexibility — swap any component independently
- Better instruction following (text LLMs outperform audio LLMs on multi-turn tasks)
- Better debugging and observability — each stage is inspectable
- Better cost predictability — ~$0.15/min regardless of conversation length
- Better support for telephony (PSTN 8 kHz audio)
- Better support for specialized TTS voices (ElevenLabs, Cartesia, Rime)

Disadvantages:
- Higher latency than native speech-to-speech (500–800ms vs. 200–300ms)
- Serial pipeline creates compounding latency

### 12.2 Speech-to-Speech Models (GPT-4o Realtime, Gemini Live)

**When to use**: Narrative applications that don't require high instruction-following accuracy; mixed-language conversations; when end-to-end latency below 300ms is non-negotiable.

Advantages:
- Lower latency (200–300ms end-to-end)
- Natural emotion and prosody preservation
- True full-duplex capability

Disadvantages:
- 10x higher cost due to context accumulation (costs grow with conversation length)
- Worse multi-turn instruction following
- Tight vendor lock-in (migrating requires rewriting streaming + state management)
- No self-hosting option
- Limited voice customization

**Cost warning**: A 15-minute speech-to-speech session can cost $5–6 in audio input tokens alone due to context window accumulation. This is not a fixed per-minute rate. Budget accordingly.

---

## 13. Edge Cases Catalogue

These are the production failure modes confirmed across industry deployments. Every one must have a defined handling path in the implementation.

| Edge Case | Failure Mode | Required Handling |
|---|---|---|
| User speaks while assistant speaks (barge-in) | Assistant hears itself via echo, loops | AEC → guard period → hard interrupt → clear buffers |
| Background TV/radio during assistant speech | AEC cannot cancel unknown reference signal | Software AEC with reference signal, not browser-native AEC |
| Long silence (user thinking) | STT hallucination, false end-of-turn | Semantic VAD extends hangover; hallucination suppression in STT |
| User says "mm-hmm" / "yeah" mid-assistant speech | False barge-in trigger | Duration gate ≥200ms; semantic filter for non-intent signals |
| Network drop mid-speech | Session state lost | State persistence; 30-second reconnection window; resume from last committed position |
| User repeats themselves after mishearing | Duplicate intent processing | Deduplication by content hash within 5-second window |
| User changes topic mid-sentence | Intent conflict in LLM context | Clear pending tool calls; inject new intent as override context |
| Tool call takes >2 seconds | Dead air → user assumes failure | Immediate placeholder speech; async tool execution; watchdog timer at 5s |
| Very noisy environment | VAD never reaches SILENCE | `utterance_end_ms` fallback timer (1000ms+); adaptive threshold increase |
| Accented or non-native speech | High WER → confused LLM | Confidence score threshold; ask for confirmation on low-confidence transcripts |
| User speaks very slowly with long pauses | Premature end-of-turn | Semantic VAD; explicit yield phrase detection; user-configurable hangover duration |
| Multiple people speaking in room | Diarization failure, wrong speaker processed | Single-speaker mode for consumer apps; speaker diarization for multi-user scenarios (adds latency) |
| User gives number sequences or proper names | STT error on proper nouns | Keyterm prompting in STT (Deepgram keyterm feature); intent confirmation on critical data |
| User interrupts during critical information (confirmation number, medical info) | Partial delivery of critical content | Disable barge-in for specific utterance types tagged as critical; resume after |
| LLM generates markdown in response | Markdown read aloud by TTS | System prompt must forbid markdown; output validation layer strips markdown before TTS |
| TTS cold start on first utterance | 500ms+ latency on first response | Pre-warm TTS connection at session start with silent synthesis |
| Context window exceeds LLM limit | LLM truncates early context, forgets instructions | Rolling summary compression every 10 turns; system prompt at both top and bottom of context |
| User switches language mid-conversation | STT/TTS mismatch | Auto-language detection in STT; language switch event triggers TTS voice change |

---

## 14. Latency Optimization Checklist

Every item on this list must be verified in production before launch:

- [x] WebSocket connections stay alive (no per-request TCP handshake)
- [x] STT flush trick implemented (buffer flushed immediately on VAD end-of-speech, not waiting for model timeout)
- [x] LLM streaming enabled (tokens return as generated, not waiting for full response)
- [x] TTS dual streaming enabled (synthesis starts on first tokens, not complete text)
- [ ] TTS connection pre-warmed at session start (no cold start on first utterance)
- [ ] Tool calls execute asynchronously (not blocking the conversation loop)
- [ ] Placeholder speech fires within 1 second of STT final (no silent gaps)
- [ ] Models co-located in same region (eliminate cross-provider network hops)
- [ ] P95 latency tracked per stage (not just mean; tail latency is the user-visible problem)
- [x] Context compression active (prevents LLM degradation on long conversations)
- [x] Barge-in stop latency <200ms verified (stop watch from VAD trigger to last TTS audio byte)

---

## 15. Recommended Stack (2026 Industry Standard)

This is the production-validated stack composition. Component substitutions are acceptable if they meet the latency and quality requirements specified in this document, but require documented justification.

| Layer | Recommended | Alternatives |
|---|---|---|
| Client audio transport | WebRTC (LiveKit, Daily.co) | Native WebSocket (higher implementation burden) |
| AEC | WebRTC built-in + software AEC for WebSocket TTS | Krisp SDK, NVIDIA Maxine |
| Noise suppression | DeepFilterNet (server) / RNNoise (edge) | NVIDIA Riva, WebRTC built-in |
| VAD | Silero VAD (probability-based) | Picovoice Cobra, WebRTC VAD (fallback) |
| End-of-turn detection | Semantic VAD (OpenAI) or smart-turn-v2 (open source) | Deepgram `utterance_end_ms` as fallback |
| STT | Deepgram Nova-3 (150ms, industry WER leader) | AssemblyAI Universal-Streaming, ElevenLabs Scribe v2 Realtime |
| LLM | GPT-4o or Gemini 2.5 Flash (multi-turn instruction following) | Do not use mini models for voice without evaluating multi-turn WER |
| TTS | ElevenLabs Flash v2.5 (75ms) or Cartesia Sonic (40ms TTFA) | Rime Mist v2 (conversational prosody), Inworld (sub-250ms, open-source training) |
| Orchestration framework | Pipecat (open source, vendor-agnostic) | LiveKit Agents |
