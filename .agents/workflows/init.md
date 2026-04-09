---
description: 
---

# Workflow: /init

## Auto-trigger
Runs automatically at the start of every session before anything else.

## Steps

1. Check GEMINI.md, project-map.md, and .agent/rules/constitution.md

2. For each file, check QUALITY not just existence:
   - GEMINI.md: Does it have a project vision, tech stack, and at least 
     one agent/skill listed?
   - project-map.md: Does it have at least one component in the registry 
     with all fields filled?
   - constitution.md: Does it have at least 3 non-negotiable rules?

3. If ALL pass quality check:
   - Silently load all three files as active context
   - Proceed ready to work. Say nothing.

4. If ANY file is missing, empty, or fails quality check:
   - Tell the user exactly which files need population
   - Ask: "Describe your project — vision, stack, and key components. 
     I'll build the missing context files from your description."
   - After description: create/update all failing files
   - Re-run quality check to confirm
   - Only then proceed

5. Never ask the user to run /init manually.