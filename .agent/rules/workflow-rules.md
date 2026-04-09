---
trigger: always_on
---

- After every build: trigger qa-runner skill
- After every new component: trigger map-keeper skill
- When a task tag has no matching skill: trigger skill-finder skill
- When planning a feature: trigger architect skill first
- When running /ship: run /qa first, abort if it fails
- Never build without consulting project-map.md first
- Before modifying core architecture: Write and run tests using test-writer skill to ensure existing functionality remains unbroken.
- Continually evaluate and trigger existing skills (.agents/skills/*) to natively elevate code quality and reliability.
- **Before ANY functional pipeline modification or new feature implementation:** Perform an `Architectural Evaluation Matrix`. Explicitly list Pros vs Cons internally inside your plan, assign a strict functional score (1-100) based on parameter impact (e.g. UX gain vs Latency cost), and ONLY proceed if the modification crosses an 85/100 threshold ensuring stability.