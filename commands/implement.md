---
description: Execute implementation scoped to a single Jira Story's tasks using the jira-map.md mapping produced by /speckit.jira.taskstotickets. Updates Sub-task and Story status in Jira as tasks complete.
tools:
  - 'Atlassian/getAccessibleAtlassianResources'
  - 'Atlassian/getJiraIssue'
  - 'Atlassian/updateJiraIssue'
  - 'Atlassian/searchJiraIssuesUsingJql'
scripts:
  sh: ../scripts/bash/check-prerequisites.sh
  ps: ../scripts/powershell/check-prerequisites.ps1
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

The user may optionally provide a **Jira Story key** (e.g., `PROJ-456`). When provided,
only the tasks mapped to that Story are implemented. When omitted, the command
auto-selects the next incomplete Story.

---

## Outline

### Step 1 — Load Feature Context

Run `{SCRIPT} --json --require-tasks --include-tasks` from repo root and parse
`FEATURE_DIR` and `AVAILABLE_DOCS` list. All paths must be absolute.

### Step 2 — Load jira-map.md

Read `FEATURE_DIR/jira-map.md`.

If the file does not exist, **STOP** and report:
> "No Jira mapping found. Run `/speckit.jira.taskstotickets` first to create tickets and generate the mapping."

Parse the file into two structures:
- **Story Map**: the `## Story Map` table — maps Story Key → Task IDs → Phase name.
- **Sub-task Map** (optional): the `## Sub-task Map` table — maps Sub-task Key → Task ID → Story Key.
  If this section is absent, sub-task status updates are skipped silently.

Also extract `**Epic**: PROJ-NNN` from the header for reference.

### Step 3 — Determine Target Story

**If the user provided a Story key**:
- Look it up in the Story Map to find the matching Task IDs and phase.
- If the key is not found, **STOP** and report the error with the list of available Story keys.

**If no Story key was provided**:
- Cross-reference each Story's Task IDs against `tasks.md` completion status.
- Display a status table:

  ```text
  | Story Key  | Phase                      | Progress | Sub-tasks | Status    |
  |------------|----------------------------|----------|-----------|-----------|
  | PROJ-450   | Phase 1: Setup             | 2/2      | none      | ✓ Done    |
  | PROJ-451   | Phase 2: Foundational      | 5/5      | none      | ✓ Done    |
  | PROJ-456   | Phase 3: User Story 1      | 0/9      | 9 mapped  | ○ Next    |
  | PROJ-457   | Phase 4: User Story 2      | 0/7      | 7 mapped  | ○ Pending |
  ```

  The `Sub-tasks` column shows "N mapped" when the Sub-task Map has entries for that Story,
  or "none" when sub-tasks were not created for that phase.

- Auto-select the first Story with incomplete tasks (by phase order in tasks.md)
  and confirm with the user before proceeding.

### Step 4 — Check Checklists (if present)

If `FEATURE_DIR/checklists/` exists, scan all checklist files and build a status table:

```text
| Checklist   | Total | Completed | Incomplete | Status  |
|-------------|-------|-----------|------------|---------|
| ux.md       | 12    | 12        | 0          | ✓ PASS  |
| test.md     | 8     | 5         | 3          | ✗ FAIL  |
```

- **All pass**: proceed automatically.
- **Any fail**: display the table, **STOP**, and ask:
  `"Some checklists are incomplete. Proceed with implementation anyway? (yes/no)"`

### Step 5 — Load Implementation Context

- **REQUIRED**: Read tasks.md
- **REQUIRED**: Read plan.md
- **IF EXISTS**: Read data-model.md, contracts/, research.md, quickstart.md

### Step 6 — Project Setup Verification

Create or verify ignore files based on detected project setup:

- `git rev-parse --git-dir` succeeds → verify `.gitignore`
- Dockerfile present or Docker mentioned in plan.md → verify `.dockerignore`
- `.eslintrc*` or `eslint.config.*` present → verify `.eslintignore` / `ignores` entries
- `.prettierrc*` present → verify `.prettierignore`
- `package.json` present → verify `.npmignore` (if publishing)
- `*.tf` files present → verify `.terraformignore`
- Helm charts present → verify `.helmignore`

**If ignore file already exists**: append only missing critical patterns.
**If ignore file missing**: create with full pattern set for detected technology.

Common patterns by technology:
- **Node.js/TypeScript**: `node_modules/`, `dist/`, `build/`, `*.log`, `.env*`
- **Python**: `__pycache__/`, `*.pyc`, `.venv/`, `dist/`, `*.egg-info/`
- **Java**: `target/`, `*.class`, `*.jar`, `.gradle/`, `build/`
- **Go**: `*.exe`, `*.test`, `vendor/`, `*.out`
- **Universal**: `.DS_Store`, `Thumbs.db`, `*.tmp`, `*.swp`

### Step 7 — Verify Phase Prerequisites

Before executing tasks for the target Story, verify that all earlier phases in the
Story Map are fully complete (`- [x]` for every Task ID in those rows).

If any earlier phase has incomplete tasks, **STOP** and report:
> "Phase N (PROJ-NNN) must be completed before starting this phase."

### Step 8 — Execute Tasks

Filter tasks.md to the Task IDs in the target Story's row. Preserve execution
order and dependency/parallel rules from tasks.md.

- **Respect dependencies**: sequential tasks in order; parallel tasks `[P]` can run together.
- **TDD approach**: execute test tasks before their corresponding implementation tasks.
- **File coordination**: tasks affecting the same files run sequentially.

Implementation rules:
1. Setup → dependencies/config first
2. Tests (write failing tests before implementation)
3. Core models, services, endpoints
4. Integration — external services, middleware, logging
5. Polish — cleanup, docs, performance

### Step 9 — Track Progress and Update Jira

After **each task** (T00X) is completed and marked `[x]` in tasks.md:

**A. Update Sub-task in Jira** (if Sub-task Map has an entry for this Task ID):
- Look up `T00X` in the Sub-task Map → get `SUBTASK_KEY`.
- Call `updateJiraIssue` to transition `SUBTASK_KEY` to the project's "Done" status.
- If the transition fails (e.g., workflow restriction), log a warning and continue —
  do not halt implementation over a status update failure.

**B. Report task progress**: after each task, output a one-line status:
```text
✓ T006 done — PROJ-460 closed (3/9 tasks in PROJ-456)
```
Or, if no sub-task mapped:
```text
✓ T001 done (1/2 tasks in PROJ-450)
```

If any non-parallel task fails, halt and provide a clear error with context.
For parallel tasks `[P]`, continue with successful ones and report failures at the end.

### Step 10 — Close Story on Phase Completion

When ALL Task IDs for the target Story are marked `[x]` in tasks.md:

- Call `updateJiraIssue` to transition the **Story** (`STORY_KEY`) to "Done".
- If the transition requires an intermediate status (e.g., "In Review"), use whatever
  the project's workflow requires — call `getJiraIssue` to inspect available transitions
  if needed.

Report:
```text
✓ All tasks complete — Story PROJ-456 (Phase 3: User Story 1) closed.
```

### Step 11 — Completion Report

Output final status:
- Story key, phase name, tasks completed: `PROJ-456 — Phase 3: User Story 1 — 9/9 tasks`
- Sub-tasks closed (if any): `9/9 sub-tasks closed`
- Summary of work done
- Suggested next Story to implement (next incomplete row in the Story Map)

Note: This command requires `jira-map.md` in the feature directory. If missing, run
`/speckit.jira.taskstotickets` first. For full (unscoped) implementation without Jira
integration, use `/speckit.implement` instead.
