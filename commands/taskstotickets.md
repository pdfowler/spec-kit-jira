---
description: Convert existing tasks into a three-tier Jira hierarchy (Epic → Story per phase → Sub-tasks per task), with a mandatory preview gate before any writes. Supports --dry-run to preview without creating tickets.
tools:
  - 'Atlassian/getAccessibleAtlassianResources'
  - 'Atlassian/getVisibleJiraProjects'
  - 'Atlassian/getJiraIssue'
  - 'Atlassian/getJiraProjectIssueTypesMetadata'
  - 'Atlassian/createJiraIssue'
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

Recognized arguments:
- **Epic key** (e.g., `PROJ-100`) — use this Epic as the parent for all Stories.
- **`--dry-run`** or **`-n`** — build the full plan and save `jira-plan.md`, but do **not** create any Jira issues or write `jira-map.md`.

Arguments may be combined: `/speckit.jira.taskstotickets PROJ-100 --dry-run`

---

## Hierarchy

```
Epic   (one per spec version — organizes the sprint/time allocation)
  └── Story    (one per ## Phase N: header — primary board visibility)
        └── Sub-task  (one per T00X item — granular implementation tracking)
```

Sub-tasks are **not** created for every phase. They are skipped automatically for
setup/infrastructure phases and phases with ≤ 3 tasks.

---

## Outline

### ── PLAN PHASE (read-only Jira calls) ──────────────────────────────────────

### Step 1 — Load Feature Context

Run `{SCRIPT} --json --require-tasks --include-tasks` from repo root and parse
`FEATURE_DIR` and `AVAILABLE_DOCS` list. All paths must be absolute.

Extract the path to **tasks.md** and read it.

### Step 2 — Connect to Jira

Call `getAccessibleAtlassianResources` to obtain the Jira Cloud ID.

> [!CAUTION]
> ONLY PROCEED IF A VALID JIRA CLOUD RESOURCE IS RETURNED.

### Step 3 — Identify the Epic

> The Epic is the organizing unit for the entire spec version.

**A. Check for existing mapping**: Read `FEATURE_DIR/jira-map.md` if it exists.
   - If the file contains `**Epic**: PROJ-NNN`, extract that key and **validate it
     via `getJiraIssue`**.
   - **If the Epic key does NOT exist in Jira** (404 / not visible):
     - Warn: `"⚠ jira-map.md references PROJ-NNN but that issue was not found in Jira. The map may be stale or from a failed previous run. Treating as no existing mapping."`
     - **Rename** `jira-map.md` to `jira-map.stale.md` so it is not silently reused.
     - Continue as if no mapping exists (fall through to B or C).
   - **If the Epic key exists**: present `"Found existing Epic PROJ-NNN from jira-map.md. Use this? (yes / enter different key / create new)"`

**B. If no mapping exists** and the user provided an Epic key as argument:
   - Validate via `getJiraIssue`. If not found, **STOP** and report.
   - Use the validated key.

**C. If no mapping and no argument**:
   - Ask: `"No Epic found for this spec. Enter an existing Epic key (e.g., PROJ-100) or a name to create a new one:"`
   - If key (matches `[A-Z]+-\d+`): validate via `getJiraIssue`.
   - If name: note it as `EPIC_TO_CREATE = "<name>"` — **do not create yet** (creation happens in the Write Phase).

Store as `EPIC_KEY` (existing key) or `EPIC_TO_CREATE` (name for new Epic).

### Step 4 — Determine Project Key and Issue Types

Extract the project key from `EPIC_KEY` or from the project the user confirmed.
If only `EPIC_TO_CREATE` is set (no existing key), call `getVisibleJiraProjects`
to confirm which project to use.

Call `getJiraProjectIssueTypesMetadata` to identify:
- `EPIC_TYPE` — Epic issue type name
- `STORY_TYPE` — Story issue type name (use `Task` if no Story type exists)
- `SUBTASK_TYPE` — Sub-task issue type name

> [!CAUTION]
> Use the **exact** names returned by the API. Never hardcode type names.

### Step 5 — Compute Story Point Estimate

Analyze ALL incomplete tasks in tasks.md using this heuristic (do **not** write
to Jira yet — this is computed for the preview):

| Signal | Points |
|--------|--------|
| Total incomplete phases | Base: 1 pt per phase |
| Task volume: 11–20 tasks | +1; 21–30: +2; 31+: +3 |
| External integrations (services, DB, auth, APIs) | +1 per distinct integration |
| Testing depth (dedicated test phases, E2E, contracts) | +1 if significant |
| Novelty / research (spikes, unfamiliar tech) | +1 if present |
| Data model complexity (>5 entities, migrations) | +1 if present |

Snap to the nearest Fibonacci: **1, 2, 3, 5, 8, 13, 21**.

Store as `ESTIMATED_POINTS` with a one-line breakdown (e.g., `"3 phases + 1 integration = 4 → 5"`).

> This estimate intentionally deflates vs. manual effort because AI handles boilerplate,
> scaffolding, and test generation. Human effort concentrates on review and integration.

### Step 6 — Discover Naming Convention

Search for existing Stories under the Epic using JQL:
`issueType = {STORY_TYPE} AND "Epic Link" = {EPIC_KEY} ORDER BY key ASC`
(Try `parent = {EPIC_KEY}` if the first returns nothing — next-gen vs. classic projects.)

- If siblings exist: extract the naming pattern and match it.
- If no siblings: use the default below.

**Default naming convention** (applies to Story summaries):
- Format: `<concise deliverable>` — **max 60 chars, no "Phase N:" prefix**
- Start with an action verb (Implement, Add, Create, Deliver, Verify…)
- Describe the **outcome**, not internal artifacts or phase numbers
- No task IDs, no phase numbers, no "User Story N —" labels, no file paths

| ❌ Bad | ✅ Good |
|--------|---------|
| `Phase 3: User Story 1 — Browse Filing Guides` | `Implement filing guide browse by state` |
| `Phase 4: T015-T018 Auth middleware` | `Add JWT auth middleware and token refresh` |
| `Phase 1: Setup` | `Bootstrap project and install dependencies` |

### Step 7 — Build the Full Planned Ticket Set

Parse tasks.md for `## Phase N:` headers. For each phase, determine:

**Story title**: apply the naming convention from Step 6.

**Sub-task eligibility** — mark a phase as **skip-subtasks** if ANY of:
- Phase name (case-insensitive) contains: `setup`, `foundation`, `foundational`,
  `prereq`, `prerequisite`, `infrastructure`, `infra`, `polish`, `cross-cutting`,
  `cleanup`, `qa`, `validation`, `hardening`, `final`
- Phase has ≤ 3 incomplete tasks

Phases marked **skip-subtasks** are **description-only**: embed the full task list
as a markdown checklist in the Story description instead of creating Sub-tasks.

For subtask-eligible phases, build the list of Sub-task summaries:
- Strip `T00X`, `[P]`, and `[US#]` markers from each task line
- Extract the core action in **≤ 7 words**, starting with a verb
- Move file paths to the sub-task description; keep them out of the summary

| ❌ Bad | ✅ Good |
|--------|---------|
| `T001 Create project structure per implementation plan in src/app/` | `Create project structure` |
| `T012 [P] [US1] Implement User model in src/models/user.py` | `Implement User model` |

Store the full planned set as:
```
planned_tickets = [
  { phase: "Phase 1", story_title: "...", tasks: ["T001","T002"], subtasks: [] },
  { phase: "Phase 3", story_title: "...", tasks: ["T006",...], subtasks: [
      { task_id: "T006", summary: "Implement TelnyxAdapter.send()" },
      ...
  ]},
  ...
]
```

Also note any phases where ALL tasks are already completed (`- [x]`) — these are
**skipped entirely** (no Story created).

---

### ── PREVIEW GATE ────────────────────────────────────────────────────────────

### Step 8 — Show Preview and Confirm

Build and display the consolidated preview table:

```
Epic: PROJ-100 "Two-Way SMS v1" (existing)
  — OR —
Epic: [New] "Two-Way SMS v1" (will be created in project PROJ)

Estimated story points: 5  (3 phases, 17 tasks, 1 external integration)

| Level       | Title                                       | Phase   | Task IDs  |
|-------------|---------------------------------------------|---------|-----------|
| Story       | Bootstrap project and install dependencies  | Phase 1 | T001–T002 |
| (desc-only) | ← skip-subtasks: ≤3 tasks                  | Phase 2 | T003–T005 |
| Story       | Implement SMS delivery pipeline             | Phase 3 | T006–T014 |
| Sub-task    |   ↳ Implement TelnyxAdapter send method     | Phase 3 | T006      |
| Sub-task    |   ↳ Add retry with exponential backoff      | Phase 3 | T007      |
| Sub-task    |   ↳ ...                                     | Phase 3 | ...       |
| Story       | Add delivery audit trail                    | Phase 4 | T015–T019 |
| Sub-task    |   ↳ Create DeliveryAuditRecord model        | Phase 4 | T015      |

3 Stories · 11 Sub-tasks will be created.  (Phase 2 tasks embedded in Story description.)
Story points: 5 will be set on the Epic.
```

**If `--dry-run` or `-n` was in the user's arguments**:
- Save the preview table to `FEATURE_DIR/jira-plan.md` (format below).
- Report: `"Dry run complete. Plan saved to jira-plan.md. No Jira tickets were created."`
- **STOP** — do not proceed to the Write Phase.

**Otherwise** — ask the user:

```
→ Proceed? (yes / no / save-plan-only)
```

- **`no`**: abort. No Jira writes, no files written.
- **`save-plan-only`**: save `FEATURE_DIR/jira-plan.md` and stop. No Jira writes.
- **`yes`**: continue to the Write Phase.

#### jira-plan.md format

> [!IMPORTANT]
> `jira-plan.md` **never contains real Jira keys** — no tickets have been created at
> this point. The file is a planning artifact only. Do not use it as evidence that
> tickets exist in Jira.

```markdown
# Jira Ticket Plan  ⚠ DRAFT — no tickets have been created

**Project**: PROJ | **Epic**: PROJ-100 (existing, not yet linked) | **Generated**: YYYY-MM-DD
**Estimated story points**: 5  (3 phases, 17 tasks, 1 external integration)
**Status**: DRAFT — run `/speckit.jira.taskstotickets` without `--dry-run` to create tickets

| Level       | Title                                       | Phase   | Task IDs  | Jira Key  |
|-------------|---------------------------------------------|---------|-----------|-----------|
| Story       | Bootstrap project and install dependencies  | Phase 1 | T001–T002 | (not created) |
| (desc-only) | Foundation (≤3 tasks — tasks in description)| Phase 2 | T003–T005 | (not created) |
| Story       | Implement SMS delivery pipeline             | Phase 3 | T006–T014 | (not created) |
| Sub-task    | ↳ Implement TelnyxAdapter send method       | Phase 3 | T006      | (not created) |
...

_No Jira tickets were created. To create them, run `/speckit.jira.taskstotickets` without `--dry-run`._
```

---

### ── WRITE PHASE (Jira writes — only reached after "yes") ───────────────────

### Step 9 — Create or Confirm Epic

**If `EPIC_KEY` is set** (existing Epic): use it directly.

**If `EPIC_TO_CREATE` is set** (new Epic): call `createJiraIssue` with:
- Issue type: `EPIC_TYPE`
- Summary: `EPIC_TO_CREATE`

Update `EPIC_KEY` with the newly created key.

### Step 10 — Set Story Points on Epic

Read current story points from the Epic via `getJiraIssue`:
- **No points (null/0)**: present `ESTIMATED_POINTS` with breakdown, ask to confirm,
  then call `updateJiraIssue` to set.
- **Points already set, matching estimate**: report "Aligned" and move on.
- **Points already set, differing**: present both values with reasoning and offer
  to update or keep existing.

### Step 11 — Create Stories

For each phase in `planned_tickets` (skip fully-completed phases):

Call `createJiraIssue` with:
- **Summary**: the Story title from Step 7
- **Description** (Markdown):
  - *Subtask-eligible phase*: phase goal/purpose and checkpoint criteria only —
    **do not include the full task list** (individual tasks are captured as Sub-tasks)
  - *Description-only phase* (skip-subtasks): phase goal/purpose, checkpoint criteria,
    **and** the full task list as a markdown checklist (since no Sub-tasks will be created)
- **Issue type**: `STORY_TYPE`
- **Parent / Epic Link**: `EPIC_KEY`

Record `phase_story_map[phase_N] = STORY_KEY`.

> [!CAUTION]
> SKIP phases where ALL tasks are `- [x]`.

### Step 12 — Create Sub-tasks

For each phase in `planned_tickets` where `subtasks` is non-empty:

For each sub-task entry, call `createJiraIssue` with:
- **Summary**: the sub-task summary from Step 7
- **Description**: the full raw task line from tasks.md (for traceability)
- **Issue type**: `SUBTASK_TYPE`
- **Parent**: `phase_story_map[phase_N]`

Record `subtask_map[T00X] = SUBTASK_KEY`.

### Step 13 — Persist jira-map.md

> [!CAUTION]
> **Only write `jira-map.md` after ALL `createJiraIssue` calls in Steps 11–12
> have returned successfully.** If any create call fails, report the error and
> **STOP** — do not write a partial map. A partial `jira-map.md` will cause
> `implement` to reference tickets that do not exist.
>
> If a `jira-map.md` already exists from a previous run, **overwrite it in full**
> (never append). Partial or appended maps are a source of stale-key bugs.

Write `FEATURE_DIR/jira-map.md`:

```markdown
# Jira Task Map

**Project**: PROJ | **Epic**: PROJ-100 | **Created**: YYYY-MM-DD

## Story Map

| Story Key | Task IDs | Phase |
|-----------|----------|-------|
| PROJ-456  | T006, T007, T008, T009, T010, T011, T012, T013, T014 | Phase 3: Implement SMS delivery pipeline |
| PROJ-457  | T001, T002 | Phase 1: Set up notification infrastructure |

## Sub-task Map

| Sub-task Key | Task ID | Story Key |
|--------------|---------|-----------|
| PROJ-460     | T006    | PROJ-456  |
| PROJ-461     | T007    | PROJ-456  |
```

- **Story Map**: one row per created Story. Primary index consumed by `/speckit.jira.implement`.
- **Sub-task Map**: one row per Sub-task. Omit the section entirely if no sub-tasks were created.

### Step 14 — Report

Output a summary:

- Path to `jira-map.md`
- Epic key and title
- One line per Story: `PROJ-NNN → Phase N: <title> (<N> tasks, <M> sub-tasks or "no sub-tasks")`
- Total Stories created, total Sub-tasks created
- Story points set/updated/unchanged on the Epic
- Clickable link to the Epic on the Jira board
