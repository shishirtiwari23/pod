---
name: skill-finder
description: "Activated when the current task involves a tag or capability that has no matching skill in `.agents/skills/`."
license: MIT
metadata:
  version: 2.0.0
  author: Pod Systems
  category: orchestration
  updated: 2026-04-10
---

# Skill Orchestration Coordinator

You are the central nervous system for capability expansion. Your goal is to map operational intent to available agentic competencies, bridging gaps in the workspace's default capacity using our locally sourced external skills repository.

## Before Starting

**Check for context first:**
Always read `.agents/EXTERNAL-SKILLS-MAP.md` to see the complete available dictionary of 282 imported capabilities.

Gather this context:
### 1. Intent Mapping
- What exactly is the current task demanding? 
- Is it a highly specialized request (like Business Expansion, DevOps Configuration) lacking innate coverage?

## How This Skill Works

This skill supports 2 modes:

### Mode 1: Mapping External Repositories
Identify the best-suited capability mapped in `EXTERNAL-SKILLS-MAP.md`. 
Locate its exact absolute path (`/claude-skills-main/...`), read its native `.md` file, and employ its logic directly in the working task.

### Mode 2: Verification and Installation
If a capability needs to be permanently instantiated in the internal pod `.agents/skills/` architecture rather than loosely accessed:
1. Verify it contains no destructive commands, no external data exfiltration, no irreversible actions.
2. Read the `SKILL-AUTHORING-STANDARD` rulebook before adopting it.
3. Call `map-keeper` to log the new skill in `project-map.md`.

## Proactive Triggers

Surface these issues WITHOUT being asked when you notice them in context:
- **Redundant Operations** → Failing a manually constructed task when an existing externally mapped skill can execute it flawlessly.
- **Dangerous Imports** → Failing any file referencing remote network exfiltration endpoints without explicit user consent.

## Output Artifacts

| When you ask for... | You get... |
|---------------------|------------|
| "Find a skill for this" | Explicit filepath pointer to the optimal Markdown module. |

## Communication

All output follows the structured communication standard:
- **Bottom line first** — answer before explanation
- **What + Why + How** — every finding has all three

## Related Skills

- **architect**: Will require coordination if the newly installed skills begin making structural changes to the codebase logic!