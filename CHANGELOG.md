# Changelog

## [1.2.1] - 2026-05-26

### Fixed

- **Stale `jira-map.md` detection in `taskstotickets`**: When an existing
  `jira-map.md` references an Epic key that no longer exists in Jira, the map is
  now renamed to `jira-map.stale.md` with a clear warning instead of being silently
  reused. This prevents the command from building on a failed or partially-created
  previous run.

- **Stale map validation in `implement`**: On startup, `implement` now validates
  the first Story key in `jira-map.md` against Jira before doing any work. If the
  key is not found, execution stops with a clear explanation rather than attempting
  to close non-existent tickets.

- **`jira-plan.md` can no longer be mistaken for real ticket output**: The dry-run
  / `save-plan-only` artifact now has an explicit `⚠ DRAFT` header, a `(not created)`
  Jira Key column, and a prominent note that no tickets were created. Previously the
  file was structurally identical to `jira-map.md`, which caused confusion when an
  agent re-read it as evidence of existing tickets.

- **Atomic `jira-map.md` writes**: Added an explicit guard requiring all
  `createJiraIssue` calls (Steps 11–12) to succeed before `jira-map.md` is written.
  A failed mid-run previously risked writing a partial map that `implement` would
  then trust unconditionally.

## [1.2.0] - 2026-05-26

### Changed

- **Story title formatting**: Removed `Phase N:` prefix from Story summaries — titles are
  now concise deliverable statements starting with an action verb, max 60 chars
  (e.g., `Implement SMS delivery pipeline` rather than `Phase 3: Implement SMS delivery pipeline`)

- **Expanded skip-subtasks list**: Added `polish`, `cross-cutting`, `cleanup`, `qa`,
  `validation`, `hardening`, `final` to the set of phase name signals that suppress
  Sub-task creation. Those phases are now **description-only**: the full task checklist
  is embedded in the Story description instead of creating noisy Sub-tasks.

- **Story descriptions no longer repeat the task list** when Sub-tasks are being created.
  Sub-task-eligible Stories contain only the phase goal/purpose and checkpoint criteria;
  the individual tasks are captured as Sub-tasks. Description-only (skip-subtask) phases
  still include the full checklist in the description.

- **Sub-task title formatting**: Tightened to ≤ 7 words starting with an action verb.
  File paths and marker tokens (`T00X`, `[P]`, `[US#]`) are stripped from the summary
  and moved to the Sub-task description for traceability.

- **Preview table** updated to show `(desc-only)` instead of `(skip)` for phases that
  will have their task list embedded in the description.

## [1.1.0] - 2026-05-26

### Added

- **Three-tier Jira hierarchy**: `taskstotickets` now creates Epic → Story → Sub-task
  instead of a flat Task/Sub-task model
  - Epic is confirmed or created at runtime; the confirmed key persists in `jira-map.md`
    (not `feature.json`)
  - One `Story` per `## Phase N:` header, parented to the Epic
  - Sub-tasks created selectively: auto-skipped for setup/infra phases and phases
    with ≤ 3 tasks; all other phases shown in preview for confirmation

- **Mandatory preview gate** before any Jira writes
  - Shows a consolidated table: Level · Proposed Title · Phase · Task IDs
  - Includes story point estimate with breakdown
  - Prompts: `Proceed? (yes / no / save-plan-only)`
  - `save-plan-only` writes `jira-plan.md` without creating any tickets

- **`--dry-run` / `-n` argument** for `taskstotickets`
  - Skips directly to the preview table and saves `jira-plan.md`
  - No Jira issues created, no `jira-map.md` written

- **`jira-plan.md`** artifact: human-readable preview of the planned ticket hierarchy,
  suitable for sharing or reviewing before committing

- **`jira-map.md` format extended**:
  - `**Epic**: PROJ-NNN` in the header
  - Optional `## Sub-task Map` section mapping `Sub-task Key → Task ID → Story Key`
  - `## Story Map` heading added (Story Map table format unchanged — backward compatible)

- **`implement` command now closes tickets as work progresses**:
  - Closes the Sub-task in Jira when each `T00X` is marked `[x]` in tasks.md
  - Closes the Story in Jira when all phase tasks are complete
  - Status table gains a Sub-tasks column showing how many are mapped per Story

### Changed

- Story points are now estimated during the Plan Phase and set during the Write Phase
  (after preview confirmation), rather than being set before ticket creation
- `implement` command description updated to reflect Story/Sub-task terminology

## [1.0.0] - 2026-02-27

### Added

- `speckit.jira.taskstotickets` command — converts task phases into Jira tickets and writes `jira-map.md`
- `speckit.jira.implement` command — scopes implementation to a single Jira ticket's tasks
- `after_tasks` hook — optional prompt to create Jira tickets after `/speckit.tasks` completes
- Story point estimation for parent tickets (AI-assisted)
- Auto-selection of next incomplete ticket in implement command
