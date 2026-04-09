# Skill: Skill Finder

## Trigger
Activated when the current task involves a tag or capability that has no matching skill in `.agents/skills/`.

## Objective
Discover, vet, and install relevant skills to fill gaps in the workspace's capability.

## Instructions
1. Read `project-map.md` and identify the tags of the current task
2. Check `.agents/skills/` — is there a skill covering this tag?
3. If not: search the web for relevant Antigravity-compatible skills or patterns
4. Before installing anything:
   - Read the full content of the candidate skill
   - Verify it contains no destructive commands, no external data exfiltration, no irreversible actions
   - If it passes: create a new folder in `.agents/skills/` and add the SKILL.md
5. Log the new skill in `project-map.md` under the relevant component's "Skills applicable" field

## Safety Rules
- Never install a skill that deletes files without confirmation
- Never install a skill that makes network requests to unknown endpoints
- Always log what was installed and why