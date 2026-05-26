---
description: Convert tasks.md into Jira tickets (parent ticket → phase tickets → sub-tasks) with a mandatory preview gate. Supports both checklist and story-card task formats. Supports --dry-run.
tools:
  - 'Atlassian/getAccessibleAtlassianResources'
  - 'Atlassian/getVisibleJiraProjects'
  - 'Atlassian/getJiraIssue'
  - 'Atlassian/getJiraProjectIssueTypesMetadata'
  - 'Atlassian/createJiraIssue'
  - 'Atlassian/updateJiraIssue'
  - 'Atlassian/searchJiraIssuesUsingJql'
  - 'Atlassian/createJiraIssueLink'
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
- **Parent ticket key** (e.g., `PROJ-123`) — nest all phase tickets under this parent
- **`--dry-run`** or **`-n`** — preview without creating any Jira issues

Arguments may be combined: `/speckit.jira.taskstotickets PROJ-100 --dry-run`

---

## Outline

### ── PLAN PHASE (read-only) ───────────────────────────────────────────────────

### Step 0 — Identify Parent Ticket

If `$ARGUMENTS` contains a key matching `[A-Z]+-\d+`, extract it as `PARENT_KEY`.

Otherwise ask the user:

> Do you have an existing Epic or Story to group these tickets under?
> - Provide the Jira key (e.g. `PROJ-123`) to nest phase tickets under it, **or**
> - Type `none` to create standalone phase tickets

Set `PARENT_KEY` to the provided key, or `null` if "none".

---

### Step 1 — Load Tasks and Detect Format

Run `{SCRIPT} --json --require-tasks --include-tasks` from repo root and parse
`FEATURE_DIR` and `AVAILABLE_DOCS`. All paths must be absolute. Extract the path
to **tasks.md** and read it.

**Detect format** — inspect `tasks.md` to determine which format was used:
- **Checklist**: phase headers (`## Phase N: …`) with task lines (`- [ ] T001 …`)
- **Story-card**: story card headings (`### PREFIX-NNN: Title`) with **Name** /
  **Description** / **Acceptance Criteria** / **Dependencies** blocks

Set `FORMAT` to `checklist` or `story-card`. All subsequent steps branch on `FORMAT`.

---

### Step 2 — Connect to Jira

Call `getAccessibleAtlassianResources` to obtain the Jira Cloud ID.

> [!CAUTION]
> ONLY PROCEED IF A VALID JIRA CLOUD RESOURCE IS RETURNED.

---

### Step 3 — Determine Project Key

- If `PARENT_KEY` is set: extract the project key from it (the part before the hyphen).
- If `PARENT_KEY` is null: call `getVisibleJiraProjects`, list available projects,
  and ask the user to confirm the target project key before proceeding.

---

### Step 4 — Validate Parent and Detect Stale Map

If `PARENT_KEY` is set:
- Call `getJiraIssue` to confirm the parent ticket exists. If not, **STOP** and report.

If `FEATURE_DIR/jira-map.md` exists from a previous run:
- Extract the first Phase Ticket key from the map table and call `getJiraIssue`.
- **If it does NOT exist in Jira**: the map is stale — rename `jira-map.md` →
  `jira-map.stale.md` with a warning and continue as if no prior mapping exists.

---

### Step 5 — Story Points on Parent

If `PARENT_KEY` is set, estimate and set story points (computed for preview, applied
after confirmation):

**Heuristic** (AI-assisted — assumes Spec Kit + AI coding agent):

| Signal | Points |
|--------|--------|
| Total incomplete phases / story cards | 1 pt each |
| Task volume (incomplete): 11–20 | +1 · 21–30: +2 · 31+: +3 |
| External integrations (APIs, DBs, auth) | +1 per distinct integration |
| Test depth (dedicated phases, E2E) | +1 if significant |
| Novelty / research tasks | +1 if present |
| Data model complexity (>5 entities, migrations) | +1 if present |

Snap to nearest Fibonacci: **1, 2, 3, 5, 8, 13, 21**.

Store as `ESTIMATED_POINTS` with a one-line breakdown.

> This estimate intentionally deflates vs. manual effort because AI handles
> boilerplate, scaffolding, and test generation.

---

### Step 6 — Discover Issue Types

Call `getJiraProjectIssueTypesMetadata` for the target project. Identify:
- **Task** (or Story) issue type — used for phase / story-card tickets
- **Sub-task** issue type — used for individual task / acceptance-criterion tickets

> [!CAUTION]
> Use the **exact** names returned by the API. Never hardcode type names.

---

### Step 7 — Parse Work Units

**Checklist format** — group by `## Phase N: …` headers:
- Each phase = one work unit.
- Collect all incomplete task lines (`- [ ] T00N …`). Skip phases where all tasks
  are `- [x]`.
- A phase with mixed complete/incomplete tasks is included; note already-done tasks
  in the ticket description.
- **Mark as description-only** (no sub-tasks) any phase whose heading contains any
  of these signals (case-insensitive):
  `setup`, `foundation`, `foundational`, `prereq`, `prerequisite`,
  `infrastructure`, `infra`, `polish`, `cross-cutting`, `cleanup`,
  `qa`, `validation`, `hardening`, `final`
  — or any phase with ≤ 3 incomplete tasks.
  Tasks for description-only phases are embedded in the ticket description.

**Story-card format** — group by `### PREFIX-NNN: Title` blocks:
- Each story card = one work unit.
- Collect all incomplete acceptance criteria lines (`- [ ] …`).
- Skip cards where Status is `done` or all criteria are checked.
- Also extract: **Name**, **Description**, **Files to modify**, **Dependencies**.

---

### Step 8 — Draft Ticket Titles

**Checklist — phase ticket titles**:
- Format: `<concise deliverable>` — **max 60 chars, no "Phase N:" prefix**
- Start with an action verb (Implement, Add, Create, Verify, Harden…)
- Describe the **outcome**, not internal artifacts or phase numbers

| ❌ Bad | ✅ Good |
|--------|---------|
| `Phase 3: User Story 1 — Browse Filing Guides` | `Implement filing guide browse by state` |
| `Phase 4: T015-T018 Auth middleware` | `Add JWT auth middleware and token refresh` |
| `Phase 1: Setup` | `Bootstrap project and install dependencies` |

**Checklist — sub-task titles** (one per T00N task line):
- Strip the task ID prefix (`T001`, `T012 [P]`, etc.) — never include it in the title
- Strip `[US#]` and `[P]` markers; move file paths to the sub-task description
- Extract the core action in ≤ 7 words, starting with a verb

| ❌ Bad | ✅ Good |
|--------|---------|
| `T001 Create project structure per plan in src/app/` | `Create project structure` |
| `T012 [P] [US1] Implement User model in src/models/` | `Implement User model` |

**Story-card — ticket titles**:
- Use the story card's **Name** field directly (3–8 words by spec)
- Do NOT use the `### PREFIX-NNN: Title` heading line as the summary
- If Name is missing, fall back to the heading title trimmed to 60 chars

**Story-card — sub-task titles** (one per acceptance criterion):
- Strip the `- [ ]` prefix; use the criterion text as-is if ≤ 60 chars
- If longer: trim to the first clause (before a comma or semicolon) + "…"

---

### ── PREVIEW GATE ─────────────────────────────────────────────────────────────

### Step 9 — Show Preview and Confirm

> [!IMPORTANT]
> No Jira write API calls may occur before the user confirms in this step.

Build and display a preview tree:

*Checklist format:*
```
Project: PROJ  |  Parent: PROJ-100 (or "standalone")  |  Format: checklist
────────────────────────────────────────────────────────────────────────────
Bootstrap project and install dependencies   [Task]  2 sub-tasks
  ├─ Create project structure                [Sub-task]
  └─ Install and configure dependencies      [Sub-task]
Implement core data model                    [Task]  3 sub-tasks
  ├─ …
Polish and cross-cutting cleanup             [Task]  description-only (5 tasks in description)
Total: 3 tickets · 5 sub-tasks  (1 description-only ticket)
Story points: 5 will be set on PROJ-100
```

*Story-card format:*
```
Project: PROJ  |  Parent: PROJ-100 (or "standalone")  |  Format: story-card
─────────────────────────────────────────────────────────────────────────────
Auth middleware setup                        [Task]  3 sub-tasks
  ├─ Token validation passes for valid JWT   [Sub-task]
  ├─ 401 returned for missing/expired token  [Sub-task]
  └─ Middleware integrated with NestJS guards [Sub-task]
Total: 2 tickets · 7 sub-tasks
Story points: 3 will be set on PROJ-100
```

**If `--dry-run` or `-n`**:
- Save the preview as `FEATURE_DIR/jira-plan.md` (format below).
- Report: `"Dry run complete. Plan saved to jira-plan.md. No Jira tickets were created."`
- **STOP** — do not proceed to the Write Phase.

**Otherwise** ask: **"Create these tickets? (yes / no / save-plan-only)"**
- `no` → abort; no Jira writes, no files written.
- `save-plan-only` → save `jira-plan.md` and stop; no Jira writes.
- `yes` → continue to the Write Phase.

#### jira-plan.md format

> [!IMPORTANT]
> `jira-plan.md` **never contains real Jira keys** — no tickets exist yet.
> Do not use it as evidence that tickets have been created.

```markdown
# Jira Ticket Plan  ⚠ DRAFT — no tickets have been created

**Project**: PROJ | **Parent**: PROJ-100 (not yet linked) | **Generated**: YYYY-MM-DD
**Status**: DRAFT — run `/speckit.jira.taskstotickets` without `--dry-run` to create tickets

| Level       | Title                                       | Jira Key      |
|-------------|---------------------------------------------|---------------|
| Task        | Bootstrap project and install dependencies  | (not created) |
| Sub-task    |   ↳ Create project structure               | (not created) |
| Sub-task    |   ↳ Install and configure dependencies     | (not created) |
| Task (desc) | Polish and cross-cutting cleanup            | (not created) |
...

_No Jira tickets were created. To create them, run without `--dry-run`._
```

---

### ── WRITE PHASE (only reached after "yes") ──────────────────────────────────

### Step 10 — Set Story Points

If `PARENT_KEY` is set, apply `ESTIMATED_POINTS` to the parent:
- No existing points → confirm, then call `updateJiraIssue`.
- Existing points differ → show both, ask to update or keep.
- Existing points match → report "aligned" and move on.

---

### Step 11 — Create Tickets

For each work unit in order:

1. **Create the work-unit ticket** (`createJiraIssue`):
   - **Summary**: title from Step 8
   - **Issue type**: Sub-task (if `PARENT_KEY` set) or Task
   - **Parent**: `PARENT_KEY` (if set)
   - **Description**:
     - *Checklist (sub-task-eligible)*: phase goal/purpose and checkpoint criteria only —
       **do not include the task list**; individual tasks are captured as sub-tasks
     - *Checklist (description-only)*: phase goal/purpose, checkpoint criteria, **and**
       the full task list as a markdown checklist
     - *Story-card*: full **Description** block and **Files to modify** list —
       **do not include acceptance criteria**; those are captured as sub-tasks

2. **Create one sub-task per item** under the work-unit ticket (`createJiraIssue`):
   - *Checklist*: one per incomplete `- [ ] T00N` task line, except description-only phases
   - *Story-card*: one per incomplete acceptance criterion (`- [ ] …`)
   - **Summary**: sub-task title from Step 8
   - **Issue type**: Sub-task
   - **Parent**: the work-unit ticket key just created
   - **Description**:
     - *Checklist*: full original task line including file paths and any `[P]`/`[US#]` markers
     - *Story-card*: the full criterion text

3. **Link dependencies** (story-card only): for each work unit with a non-empty
   **Dependencies** field, call `createJiraIssueLink` to add "blocks"/"is blocked by" links.

Report progress after each ticket:
`✓ PROJ-456 Implement SMS delivery pipeline · 3 sub-tasks created`

---

### Step 12 — Persist jira-map.md

> [!CAUTION]
> **Only write `jira-map.md` after ALL `createJiraIssue` calls in Step 11 succeed.**
> If any call fails, report the error and **STOP** — do not write a partial map.
> Overwrite any existing `jira-map.md` in full; never append.

Write `FEATURE_DIR/jira-map.md`:

```markdown
# Jira Task Map

**Project**: PROJ | **Parent**: PROJ-100 (or "None") | **Created**: YYYY-MM-DD

| Phase Ticket | Sub-task | Task ID | Description |
|--------------|----------|---------|-------------|
| PROJ-456 Implement SMS delivery pipeline | PROJ-460 | T001 | Create adapter |
| PROJ-456 Implement SMS delivery pipeline | PROJ-461 | T002 | Add retry logic |
| PROJ-457 Bootstrap project dependencies | — | T003–T005 | (description-only) |
```

- One row per sub-task; description-only phases use `—` in the Sub-task column.
- The Phase Ticket column repeats the key+title for each of its sub-tasks.
- This file is consumed by `/speckit.jira.implement`.

---

### Step 13 — Report

- Path to `jira-map.md`
- One line per phase ticket: key → title · N sub-tasks created
- Total phase tickets and sub-tasks created
- Parent ticket and story points applied (if applicable)
- Clickable link to the parent ticket or Jira board
