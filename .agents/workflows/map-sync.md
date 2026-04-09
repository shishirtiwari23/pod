---
description: 
---

# Workflow: /map-sync

Use this to reconcile project-map.md with the actual current state of the codebase.

## Steps
1. Scan the full project file tree
2. Compare against Component Registry in `project-map.md`
3. Flag: missing components, renamed files, new files not in the map
4. Activate `map-keeper` skill to update all discrepancies
5. Present a Map Sync Artifact showing what was added, updated, or flagged