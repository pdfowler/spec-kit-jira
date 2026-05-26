---
description: Run implementation scoped to a single phase ticket's tasks from jira-map.md; closes sub-tasks and the phase ticket in Jira as work completes.
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

The user may optionally provide a **Jira phase ticket key** (e.g., `PROJ-456`). When
provided, only tasks mapped to that ticket are implemented. When omitted, the command
auto-selects the next incomplete ticket.

---

## Outline

### Step 1 — Load Feature Context

Run `{SCRIPT} --json --require-tasks --include-tasks` from repo root and parse
`FEATURE_DIR` and `AVAILABLE_DOCS`. All paths must be absolute.

---

### Step 2 — Load and Validate jira-map.md

Read `FEATURE_DIR/jira-map.md`.

If the file does not exist, **STOP**:
> "No Jira mapping found. Run `/speckit.jira.taskstotickets` first to create tickets
> and generate the mapping."

Parse the flat table:
- **Phase tickets**: unique Phase Ticket column values (first space-delimited token
  is the key; the rest is the title)
- **Sub-task index**: Sub-task key keyed by Task ID (skip rows where Sub-task is `—`)

Extract `**Parent**: PROJ-NNN` from the header for reference.

**Validate the map is not stale**: call `getJiraIssue` on the first Phase Ticket key.
- If it exists: proceed.
- If it does NOT exist: **STOP**:
  > "⚠ jira-map.md references {KEY} but that issue was not found in Jira. The map
  > may be from a failed or partial previous run. Run `/speckit.jira.taskstotickets`
  > to create tickets and regenerate the mapping."

---

### Step 3 — Determine Target Ticket

**If the user provided a key**: look it up in the Phase Ticket column.
- If not found: **STOP** and list available phase ticket keys.

**If no key provided**: cross-reference each phase ticket's Task IDs against
`tasks.md` completion status and display a status table:

```text
| Phase Ticket | Title                                | Task IDs  | Progress | Status    |
|--------------|--------------------------------------|-----------|----------|-----------|
| PROJ-450     | Bootstrap project dependencies       | T001–T002 | 2/2      | ✓ Done    |
| PROJ-456     | Implement SMS delivery pipeline      | T003–T011 | 0/9      | ○ Next    |
| PROJ-457     | Add delivery audit trail             | T012–T015 | 0/4      | ○ Pending |
```

Auto-select the first ticket with incomplete tasks (by order in jira-map.md) and
confirm with the user before proceeding.

---

### Step 4 — Check Checklists

If `FEATURE_DIR/checklists/` exists, scan all checklist files:

```text
| Checklist   | Total | Completed | Incomplete | Status  |
|-------------|-------|-----------|------------|---------|
| ux.md       | 12    | 12        | 0          | ✓ PASS  |
| test.md     | 8     | 5         | 3          | ✗ FAIL  |
```

- All pass → proceed automatically.
- Any fail → display table and ask:
  **"Some checklists are incomplete. Proceed with implementation anyway? (yes/no)"**

---

### Step 5 — Load Implementation Context

- **REQUIRED**: Read `tasks.md` (full task list and execution plan)
- **REQUIRED**: Read `plan.md` (tech stack, architecture, file structure)
- **IF EXISTS**: Read `data-model.md`, `contracts/`, `research.md`, `quickstart.md`

---

### Step 6 — Project Setup Verification

Create or verify ignore files based on detected project setup:

- `git rev-parse --git-dir` succeeds → verify `.gitignore`
- Dockerfile present or Docker mentioned in `plan.md` → verify `.dockerignore`
- `.eslintrc*` or `eslint.config.*` present → verify `.eslintignore` / `ignores` entries
- `.prettierrc*` present → verify `.prettierignore`
- `package.json` present (if publishing) → verify `.npmignore`
- `*.tf` files present → verify `.terraformignore`
- Helm charts present → verify `.helmignore`

**If ignore file exists**: append only missing critical patterns.
**If ignore file missing**: create with full pattern set for the detected technology.

Common patterns by technology:
- **Node.js/TypeScript**: `node_modules/`, `dist/`, `build/`, `*.log`, `.env*`
- **Python**: `__pycache__/`, `*.pyc`, `.venv/`, `dist/`, `*.egg-info/`
- **Java**: `target/`, `*.class`, `*.jar`, `.gradle/`, `build/`
- **Go**: `*.exe`, `*.test`, `vendor/`, `*.out`
- **Universal**: `.DS_Store`, `Thumbs.db`, `*.tmp`, `*.swp`

---

### Step 7 — Verify Phase Prerequisites

Before executing, verify all earlier phase tickets in `jira-map.md` are fully complete
(`- [x]` for every Task ID in those rows).

If any earlier phase has incomplete tasks, **STOP**:
> "Phase ticket PROJ-NNN must be completed before starting this phase."

---

### Step 8 — Execute Tasks

Filter `tasks.md` to the Task IDs in the target ticket's rows. Preserve execution
order and dependency/parallel rules from `tasks.md`.

- **Respect dependencies**: sequential tasks in order; parallel tasks `[P]` can run together.
- **TDD approach**: write failing tests before their corresponding implementation.
- **File coordination**: tasks affecting the same files run sequentially.

Implementation order:
1. Setup — dependencies, configuration
2. Tests — write failing tests before implementation code
3. Core — models, services, endpoints
4. Integration — external services, middleware, logging
5. Polish — cleanup, docs, performance

---

### Step 9 — Track Progress and Update Jira

After **each task** (T00X) is marked `[x]` in tasks.md:

**A. Close sub-task in Jira** (if a sub-task key exists for this Task ID):
- Call `updateJiraIssue` to transition the sub-task to "Done".
- If the transition fails, log a warning and continue — do not halt over a status failure.

**B. Report one-line progress**:
```text
✓ T006 done — PROJ-460 closed  (3/9 tasks in PROJ-456)
✓ T001 done  (1/2 tasks in PROJ-450)           ← no sub-task mapped
```

Halt on any non-parallel task failure. For parallel tasks `[P]`, continue with
successful ones and report failures at the end.

---

### Step 10 — Close Phase Ticket on Completion

When ALL Task IDs for the target ticket are marked `[x]` in tasks.md:

- Call `updateJiraIssue` to transition the **phase ticket** to "Done".
  - If an intermediate status is required (e.g., "In Review"), call `getJiraIssue`
    to inspect available transitions and apply the correct one.

Report:
```text
✓ All tasks complete — PROJ-456 (Implement SMS delivery pipeline) closed.
```

---

### Step 11 — Completion Report

Output final status:
- Phase ticket key, title, tasks completed:
  `PROJ-456 — Implement SMS delivery pipeline — 9/9 tasks`
- Sub-tasks closed (if any): `9/9 sub-tasks closed`
- Summary of work done
- Suggested next ticket (next incomplete row in `jira-map.md`)

Note: This command requires `jira-map.md`. If missing, run
`/speckit.jira.implement` first. For full (unscoped) implementation without
Jira integration, use `/speckit.implement` instead.
