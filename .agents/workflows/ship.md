---
description: 
---

# Workflow: /ship

Full pipeline: test → build → deploy.

## Steps
1. Run `/qa` — if it fails, STOP. Do not proceed.
2. Run build command for the current tech stack
3. Run map-sync to ensure project-map.md is current
4. Deploy using the deployment skill (install via skill-finder if not present)
5. Verify deployment in browser
6. Create a Ship Artifact with: build summary, QA results, deployment URL