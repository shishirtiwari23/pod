---
trigger: always_on
---

# Project Constitution

These rules are non-negotiable. No agent overrides them.

1. `project-map.md` must be updated before any PR is merged
2. No production deployment without QA passing
3. No deletion of files in `project-map.md` without Architect review
4. Skill Finder must vet any external skill before it is added to `.agents/skills/`
5. All new components must be tagged and added to the Component Registry
6. **No Serialization in the Critical Path**: Audio pipeline must stream concurrently. Never wait for full mp3/wav files in active turns.
7. **Strict Latency Budget**: Sub-300ms time-to-first-audio (TTFA) constraint dictates all dependencies and middleware.
8. **Hardware Agnostic Core**: The Node.js backend must gracefully accept pure `linear16` PCM hardware streams natively.
9. **Progressive VAD Evolution**: Fallback to EOT or timers, do not write custom local acoustic VAD models from scratch.