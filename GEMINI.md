At the start of every session, automatically execute /init before 
doing anything else. Do not greet or respond until /init completes.

# Workspace Intelligence Layer
- Full project context: project-map.md
- Non-negotiables: .agent/rules/constitution.md
- Workflow rules: .agent/rules/workflow-rules.md
- Skills: .agents/skills/
- Workflows: /architect, /qa, /ship, /map-sync

## Agent Ops Framework
- **Active Skill Employment**: The agent MUST actively evaluate and proactively utilize skills defined in `.agents/skills/` during every relevant operation (e.g., executing `test-writer` after refactoring, triggering `qa-runner` after server modifications, or consulting `architect` during design phases).
- **Relentless Validation**: Never presume backend changes are fully verified without manual or automated execution loops testing the affected endpoints.

## Project Intelligence
- **Project Vision**: Pod is a sub-300ms latency, full-duplex conversational voice AI engine using a continuous streaming pipeline.
- **Tech Stack**: Node.js, Express, WebSockets, deepgram-sdk, openai (Groq), Vanilla JS/HTML frontend, native CLI sox/rec.
- **Active Features**: Streaming STT -> LLM token completion -> TTS WebSockets pipeline, LLM-based intelligent barge-in verification, native 16-bit PCM bindings.
- **Current Progress**: Phase 1 (Mac Simulation MVP) complete. Working zero-latency local pipeline built using Free Tier endpoints.