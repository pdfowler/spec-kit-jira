# Changelog

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
