---
description: Run implementation scoped to a single phase ticket's tasks from jira-map.md; records Jira progress using a configurable event-based policy.
tools:
  - 'Atlassian/getAccessibleAtlassianResources'
  - 'Atlassian/getJiraIssue'
  - 'Atlassian/getTransitionsForJiraIssue'
  - 'Atlassian/transitionJiraIssue'
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

### Step 2 — Load Jira Progress Policy

Load progress policy in this order:

1. Project override: `.specify/jira-progress-policy.yml`
2. Extension default: `.specify/extensions/jira/config/jira-progress-policy.yml`

If neither exists, use the built-in default described below and offer to write it
to `.specify/jira-progress-policy.yml` for customization.

The policy defines:
- semantic status mappings (`todo`, `in_progress`, `in_review`, `done`)
- monotonic transition rules (never move Done back to In Progress)
- which events move sub-tasks and phase tickets
- commit, local gate, PR open, and PR merge requirements
- whether draft PRs count as review (default: false)

Use policy defaults unless the project override explicitly changes them:
- Start phase/task -> `in_progress` only from `todo`
- Task commit -> sub-task `in_review`
- Local gates passed -> sub-task `done`
- Non-draft PR opened -> phase ticket `in_review`
- PR merged -> phase ticket `done`

When applying a semantic status:
1. Call `getTransitionsForJiraIssue` for the issue.
2. Match the policy's semantic status to a real Jira transition target status.
3. Call `transitionJiraIssue` with the selected transition.
4. If no matching transition exists, follow `on_missing_transition` (`warn`, `stop`, `skip`).

> [!CAUTION]
> Do not use `updateJiraIssue` as a substitute for workflow transitions. Jira status
> changes must use the project's actual transition IDs.

---

### Step 3 — Load and Validate jira-map.md

Read `FEATURE_DIR/jira-map.md`.

If the file does not exist, **STOP**:
> "No Jira mapping found. Run `/speckit.jira.taskstotickets` first to create tickets
> and generate the mapping."

Extract `**Status**` from the header (`draft` or `created`). If missing, infer:
- `draft` when any key in the **Map** table is `TBD`
- `created` otherwise

**If `Status` is `draft`**, or any Phase Ticket / Sub-task cell starts with `TBD`, **STOP**:
> "jira-map.md has pending plan rows (`TBD` keys). Apply them first:
> `/speckit.jira.taskstotickets apply {PARENT_KEY}`, then retry implement."

If `FEATURE_DIR/jira-plan.md` exists (legacy), ignore it; only `jira-map.md` is authoritative.

If the file has `## Story Map` / `## Sub-task Map` but no `## Map` table, **STOP**:
> "jira-map.md uses the legacy format. Run `taskstotickets plan` to migrate to the flat
> **Map** table (preserve existing keys), then apply before implementing."

Parse the **Map** section table only (ignore **Preview** and **Links**):
- **Phase tickets**: unique Phase Ticket column values (first space-delimited token
  is the key; the rest is the title). Keys must match `[A-Z]+-\d+`.
- **Sub-task index**: Sub-task key keyed by Task ID (skip rows where Sub-task is `—`).
  Sub-task keys must match `[A-Z]+-\d+`.

Extract `**Parent**: PROJ-NNN` from the header for reference.

**Validate the map is not stale**: call `getJiraIssue` on the first real phase key
from the **Map** table.
- If it exists: proceed.
- If it does NOT exist: **STOP**:
  > "⚠ jira-map.md references {KEY} but that issue was not found in Jira. The map
  > may be from a failed or partial previous run. Run `/speckit.jira.taskstotickets`
  > to create tickets and regenerate the mapping."

---

### Step 4 — Determine Target Ticket

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

### Step 5 — Check Checklists

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

### Step 6 — Load Implementation Context

- **REQUIRED**: Read `tasks.md` (full task list and execution plan)
- **REQUIRED**: Read `plan.md` (tech stack, architecture, file structure)
- **IF EXISTS**: Read `data-model.md`, `contracts/`, `research.md`, `quickstart.md`

---

### Step 7 — Project Setup Verification

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

### Step 8 — Verify Phase Prerequisites

Before executing, verify all earlier phase tickets in `jira-map.md` are fully complete
(`- [x]` for every Task ID in those rows).

If any earlier phase has incomplete tasks, **STOP**:
> "Phase ticket PROJ-NNN must be completed before starting this phase."

---

### Step 9 — Start Phase and Execute Tasks

Filter `tasks.md` to the Task IDs in the target ticket's rows. Preserve execution
order and dependency/parallel rules from `tasks.md`.

- Before editing implementation files, apply `transitions.on_phase_start` to the
  selected phase ticket if the current status is allowed by policy.
- Before starting each mapped task, apply `transitions.on_task_start` to its sub-task
  if a sub-task key exists and the current status is allowed by policy.

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

### Step 10 — Commit Task or Phase Changes

Before moving any sub-task to review or done:

- Stage only files related to the completed task/phase. Never use `git add .`.
- Create a local commit if the task or phase changed implementation files.
- Treat "do not push" or "do not create PRs" as remote-operation restrictions only.
  Do not interpret them as "do not commit" unless the user explicitly says so.
- Use a conventional commit message and include the Jira key when available:
  `feat(scope): [PROJ-456] implement delivery route resolver`
- If there are task-related working tree changes after the commit, **STOP** and
  report them. Do not transition Jira while task work remains uncommitted.

For test-only tasks:
- If the policy classifies the task as a test task, follow `test_tasks.on_commit`.
- A red-phase failing test can move to `in_review` on commit, but it should not move
  to `done` until the policy's `test_tasks.done_gate` is satisfied.
- If the repository's pre-commit hooks reject failing tests, keep the task open and
  report that the repo does not support committing red-phase failures.

After a successful commit, apply `transitions.on_task_commit` to the sub-task when
the policy allows it.

---

### Step 11 — Run Local Quality Gates

Run the policy's `local_gates.commands` when configured. If no commands are configured,
infer the narrowest reasonable validation commands from `plan.md`, project metadata,
and existing task instructions.

Examples:
- `pnpm nx run <project>:test`
- `pnpm nx run <project>:lint`
- `pnpm nx run <project>:typecheck`
- `pnpm nx run <project>:build`

If local gates fail:
- **STOP** before marking any sub-task or phase ticket `done`.
- Leave the affected Jira issue in its current state (usually `in_progress` or `in_review`).
- Report the failing command and the next action needed.

If local gates pass, apply `transitions.on_task_local_gates_passed` for completed
sub-tasks whose policy `done_gate` is `local_gates_passed`.

---

### Step 12 — Record Sub-task Progress

After **each task** (T00X) is marked `[x]` in tasks.md:

**A. Update sub-task in Jira** (if a sub-task key exists for this Task ID):
- Use the configured policy event for the task's current evidence:
  - `on_task_commit` after commit evidence exists
  - `on_task_local_gates_passed` after local gates pass
  - `on_remote_checks_passed` only when remote checks are available and policy requires them
- Never move a `done` sub-task backward when `monotonic: true`.
- If transition fails, follow `on_missing_transition`.

**B. Report one-line progress**:
```text
✓ T006 committed — PROJ-460 moved to In Review  (3/9 tasks in PROJ-456)
✓ T006 validated — PROJ-460 moved to Done
✓ T001 done  (1/2 tasks in PROJ-450)            ← no sub-task mapped
```

Halt on any non-parallel task failure. For parallel tasks `[P]`, continue with
successful ones and report failures at the end.

---

### Step 13 — Phase Review Gate

When ALL Task IDs for the target ticket are marked `[x]` in tasks.md:

- Verify there are no task-related uncommitted changes.
- Verify local gates passed for the phase.
- If `require_remote_push_for_phase_review` is true, push the branch before moving
  the phase ticket to review.
- If `pull_requests.required_before_phase_review` is true:
  - Create or locate the PR for this branch.
  - Draft PRs count only if `draft_pr_counts_as_review: true`.
  - If no qualifying PR exists, leave the phase ticket in progress and report the
    missing PR requirement.
- Once review evidence exists, apply `transitions.on_pr_opened` to the phase ticket.

---

### Step 14 — Phase Done Gate

Move the phase ticket to `done` only when the configured `phase_done_gate` is satisfied:

- `pr_merged` (default): verify the PR is merged.
- `remote_checks_passed`: verify remote checks passed and policy permits this.
- `local_gates_passed`: verify local gates passed and policy explicitly permits this.
- `explicit_confirm`: ask the user to confirm the phase should be marked done.

If the gate is not satisfied, **do not close the phase ticket**. Report the current
evidence and the remaining gate.

When the gate is satisfied:
- Apply `transitions.on_pr_merged` or the configured equivalent to the phase ticket.
- Never move a `done` phase ticket backward when `monotonic: true`.

Report:
```text
✓ All tasks complete — PROJ-456 (Implement SMS delivery pipeline) moved to Done.
```

---

### Step 15 — Completion Report

Output final status:
- Phase ticket key, title, tasks completed:
  `PROJ-456 — Implement SMS delivery pipeline — 9/9 tasks`
- Sub-tasks closed (if any): `9/9 sub-tasks closed`
- Summary of work done
- Suggested next ticket (next incomplete row in `jira-map.md`)

Note: This command requires `jira-map.md`. If missing, run
`/speckit.jira.taskstotickets` first. For full (unscoped) implementation without
Jira integration, use `/speckit.implement` instead.
