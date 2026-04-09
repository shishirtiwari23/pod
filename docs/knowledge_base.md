# Pod: Bidirectional Voice-AI Engine Knowledge Base
This document serves as the central knowledge repository for **Pod**, synthesizing the multi-phase product roadmap with the current state of the codebase. It defines the core architectural patterns and the strict development rules for transitioning from a local simulation to a production-grade physical product.

## 1. Core Architecture & Philosophy
Pod is a sub-second, full-duplex voice AI engine designed to run end-to-end with **< 300ms time-to-first-audio (TTFA)** and **< 800ms total round-trip latency**.

### The Latency Paradigm
Traditional voice agents use a serialized REST pipeline: `Record -> Wait -> STT -> Wait -> LLM -> Wait -> TTS -> Play`.
**Pod’s Paradigm:** Concurrent streaming.
1. Audio is continuously streamed in chunks.
2. STT streams intermediate transcripts natively.
3. LLM streams token-by-token completions.
4. TTS consumes text chunks on-the-fly via WebSocket and returns audio buffers immediately.

---

## 2. Current State Assessment (Phase 1: Mac Simulation MVP)

The workspace currently implements the **Phase 1** goals.

### Component Map
*   **Frontend Simulator (`simulator/index.html`)**: A vanilla HTML/CSS/JS client simulating hardware via a browser tab. Captures mic input in 200ms WebM chunks and plays back MPEG audio chunks. Uses CSS animations (breathing, pulsing) to represent device states (Orb).
*   **Hardware Client Simulator (`simulator/client.js`)**: A CLI tool using native MacOS `rec` and `afplay` to stream 16-bit linear PCM and playback responses natively to mimic physical hardware inputs without browser overhead.
*   **Backend Gateway (`backend/server.js`)**:
    *   **STT**: Deepgram `nova-2` via WebSocket using `smart_format` and `endpointing=800`.
    *   **LLM**: Groq API using `llama-3.1-8b-instant` via the OpenAI SDK, streaming response tokens.
    *   **TTS**: Deepgram `aura-2-zeus-en` accessed via the `@deepgram/sdk`'s `speak.live()` method.
    *   **VAD (Voice Activity Detection)**: Custom logic mapping Deepgram's `is_final` and `speech_final` events with an 2500ms `utteranceTimer` fallback.
    *   **Interrupt Handling (Barge-in)**: A secondary LLM call (Groq) evaluates intermediate partial transcripts to determine if the user is making a meaningful interruption or just providing backchannel noise ("uh hum"). If affirmative, it sends a `STOP_AUDIO` dispatch.

---

## 3. Product Direction Rules by Phase

To maintain velocity, stability, and unit economics, all development work must strictly adhere to the following phase-specific rules.

### Stage 1: Mac Simulation MVP (Current)
**Goal:** Prove the pipeline works end-to-end locally with $0 API spend.
*   **Rule 1.1: Enforce Free-Tier Constraints:** Exclusively use Deepgram Nova-2 (STT), Aura-2 (TTS), and Groq 8B Instant (LLM). No paid tier keys should be introduced at this stage. Keep within 6,000 req/day for Groq and 12,000 min/year for Deepgram.
*   **Rule 1.2: Embrace the Timer Hack:** The current `utteranceTimer` (2.5s) and LLM-based barge-in verification are suitable for Stage 1. Do not spend excessive engineering time building custom acoustic VAD models from scratch; this will be resolved by vendor upgrades in Stage 3.
*   **Rule 1.3: Strict Latency Logging:** Sub-second latency is the primary KPI. Ensure the `[LATENCY]` console traces in `server.js` stay intact to monitor performance against the 800ms round-trip budget. Total LLM TTFT (Time To First Token) must remain under ~150ms.
*   **Rule 1.4: Stateless Architecture:** Maintain simplicity. Avoid adding vector databases, user authentication, or persistent storage during this stage.

### Stage 2: Hardware MVP (Investor Demo)
**Goal:** Port the Stage 1 pipeline to physical hardware (Raspberry Pi 5) for tactile demonstration.
*   **Rule 2.1: Transition I/O, Not Intelligence:** Keep the Node.js backend hosted in the cloud (e.g., Railway). The Pi 5 should act strictly as an I/O client using the ReSpeaker 2-Mics Pi HAT (via ALSA/`node-audiorecord` and `aplay`).
*   **Rule 2.2: Map Software States to Physical LEDs:** The browser `index.html` visual orb must be adapted to a physical NeoPixel LED ring. The states must perfectly map:
    *   *Blue (Slow pulse):* Standby
    *   *Green (Chase):* Listening
    *   *Yellow (Fast pulse):* Processing
    *   *White (Steady glow):* Speaking
    *   *Red (Flash):* Barge-in intercepted
*   **Rule 2.3: Graceful Demo Degradation:** Ensure standard demo paths work seamlessly. If Wi-Fi drops, implement visual feedback via LEDs. The script should be tailored to confidently show barge-in (< 300ms interruption) and real-time streaming capability.

### Stage 3: Production Product
**Goal:** Build for scalability (1 to 100,000 users), minimal latency, and robust unit economics.
*   **Rule 3.1: Eliminate Custom VAD with Deepgram Flux:** Remove the `utteranceTimer` hack and transition STT to Deepgram Flux. Utilize its built-in `EndOfTurn` and `EagerEndOfTurn` events to reduce endpointing latency by 200–600ms.
*   **Rule 3.2: Switch TTS for Speed and Cost:** Migrate from Deepgram Aura-2 to Cartesia Sonic-3. The State Space Model (SSM) architecture will drop TTFA to 40ms while reducing character synthesis costs by ~30–73% compared to alternatives.
*   **Rule 3.3: Implement Provider Redundancy (HA):** Implement `LiteLLM`. Route primary traffic to Groq (Llama 3.3 70B), with Cerebras configured as an automatic hot-failover for 429 rate limits, securing < 200ms LLM reaction times across arbitrary scale.
*   **Rule 3.4: Shift Orchestration to Pipecat & WebRTC:** Rewrite custom Node.js `server.js` websocket loop into a Python-based **Pipecat** framework pipeline. Migrate transport to **LiveKit (WebRTC)** to natively handle jitter buffering, packet loss, and robust cross-platform echo cancellation, capping end-to-end P50 latency at < 500ms.
