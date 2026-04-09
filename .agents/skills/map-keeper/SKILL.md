# Skill: Map Keeper

## Trigger
Activated after any structural change — new component created, component deleted, integration changed, new skill installed.

## Objective
Keep `project-map.md` accurate and complete so all other agents can trust it.

## Instructions
1. Read the current `project-map.md`
2. Identify what changed in this session (new files, deleted files, modified connections)
3. Update the Component Registry:
   - Add new components with all fields filled
   - Update "Connects to" for any affected components
   - Remove or archive deleted components
   - Update "Skills applicable" if new skills were installed
4. Update the "Last updated" date at the top
5. Never delete existing entries — archive them with a `[DEPRECATED]` tag instead

## Rules
- Preserve all existing content — append and update, never overwrite
- If a component's path changes, update all references to it across the map