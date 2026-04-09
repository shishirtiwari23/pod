---
trigger: always_on
---

- After every build: trigger qa-runner skill
- After every new component: trigger map-keeper skill
- When a task tag has no matching skill: trigger skill-finder skill
- When planning a feature: trigger architect skill first
- When running /ship: run /qa first, abort if it fails
- Never build without consulting project-map.md first