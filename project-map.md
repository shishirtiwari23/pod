# Project Map

_Last updated: 2026-04-09 by map-keeper_

## Vision
Pod is a sub-second, full-duplex voice AI engine. It eliminates sequential REST-based bottlenecks by executing a highly parallel, continuous streaming 3-stage pipeline (STT -> LLM -> TTS) for conversational voice capabilities matching human <300ms reaction times.

## Tech Stack
- **Frontend**: Vanilla HTML/CSS/JS (MediaRecorder API, WebSocket binary blobs)
- **Backend**: Node.js / Express / ws
- **Database**: None (Stateless MVP)
- **Testing**: Manual CLI inspection / `afplay` & `rec`
- **Deployment**: Localhost (Phase 1)
- **AI Services**: Deepgram (`nova-2` STT, `aura-2` TTS), Groq (`llama-3.1-8b-instant`)

## Conversational Boundary Cases (VAD Exception Matrix)
The backend enforces native handlers for these critical boundary conditions:
1. **Interim Bypass Bugs**: Timer-based Failsafes must actively flush `is_final: false` buffers instead of submitting stale state.
2. **False Positive Punctuation**: STT engines hallucinatory punctuation must be overridden by a local Semantic Heuristic that mathematically checks `continuationWords`.
3. **Passive Backchanneling**: Groq LLM explicitly ignores non-functional speech (e.g., "mhm", "yeah") via rigorous contextual prompting.
4. **Hardware Echo**: Hardcoded `cleanAI.includes(cleanUser)` substring intersection actively drops speaker bleed from the internal pipeline.
5. **Endpointing vs Natural Breathing**: Aggressive native `isFinal` endpointing actively strangles natural thought gaps. We explicitly enforce `speech_final` combined with `endpointing: 1000` to guarantee humans 1.0 seconds to breathe between logical sentence formations without bypassing architectural loops.

## Industry Benchmark Objectives
To match industry standards (e.g. OpenAI Realtime API, LiveKit), operations moving to Phase 2/3 must explicitly build toward:
- **Client-Side Silero VAD**: Dropping ~600ms round-trip lag by doing VAD explicitly on the client edge.
- **Interruption Injection Indices**: Sending text offsets back to the LLM upon interruption so it accurately understands context bounds.
- **Parallel JSON Function Calling**: WebSockets inherently processing native functions mid-conversation smoothly.

## Component Registry

### Backend Gateway
- **Type**: API / WebSocket Server
- **Path**: `backend/server.js`
- **Purpose**: Coordinates streaming audio between client mic/speakers and external LLM/TTS/STT providers. Handles `is_final` VAD fallbacks and semantic LLM verification of user interruptions.
- **Connects to**: Frontend Simulator, Hardware Client Simulator, Deepgram API, Groq API
- **Tests**: None (Phase 1)
- **Skills applicable**: architect, qa-runner, map-keeper, test-writer, core-debugger
- **Tags**: core, inference, websockets, mvp

### Frontend Simulator
- **Type**: UI
- **Path**: `simulator/index.html`
- **Purpose**: Browser-based client demonstrating voice capture via WebM blobs and visual state management (NeoPixel Orb simulation patterns).
- **Connects to**: Backend Gateway
- **Tests**: None
- **Skills applicable**: qa-runner, frontend-expert, core-debugger
- **Tags**: ui, debugging, browser

### Hardware Client Simulator
- **Type**: CLI Utility
- **Path**: `simulator/client.js`
- **Purpose**: Proves out sub-system native audio bindings via `rec` PCM streams mimicking the future Pi ALSA hardware layer.
- **Connects to**: Backend Gateway
- **Tests**: None
- **Skills applicable**: qa-runner, core-debugger, test-writer
- **Tags**: hardware, i/o, alsa-testing

---
_Add new components via `/architect` — never manually without running `/map-sync` after_