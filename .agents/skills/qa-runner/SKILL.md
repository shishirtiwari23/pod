# Skill: QA Runner

## Trigger
Activated after any significant build event — new component, feature addition, or refactor.

## Objective
Test the thing that was just built AND every connected component, using the project map to find all integration points.

## Instructions
1. Identify what was just built or changed
2. Read `project-map.md` — find every component listed under "Connects to" for the changed component
3. For each connected component:
   - Run its listed tests
   - Open in browser and verify core functionality if it has a UI
   - Check that inputs/outputs still match expected shape
4. If anything fails: create a QA Failure Artifact listing exactly what broke and where
5. If everything passes: create a QA Pass Artifact with components tested and results

## Tools Used
- Terminal (run test suites)
- Browser subagent (visual verification)
- Editor (read test files)

## Output
QA Artifact: Pass or Fail, with component-by-component results