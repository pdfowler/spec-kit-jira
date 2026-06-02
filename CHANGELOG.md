# Changelog

## [1.4.2] - 2026-05-29

### Added

- **`plan` / `apply` subcommands** — `taskstotickets plan ENG-6867` and
  `taskstotickets apply ENG-6867`. Parent key only (`ENG-6867`) defaults to **plan**.
- **Shorthand commands** — `speckit.jira.plan-tickets` and `speckit.jira.apply-tickets`
  (same behavior as `taskstotickets plan` / `apply`).
- **Apply guards** — apply stops when no `jira-map.md`, legacy map format, or no `TBD`
  rows to create.
- **Step 4a legacy map migration** — on **plan**, auto-converts `## Story Map` /
  `## Sub-task Map` to flat `## Map` (preserves keys; backs up `jira-map.legacy.md`).
  Apply on legacy format directs you to run **plan** first.

### Changed

- **Deprecated flag aliases** — `--dry-run` / `-n` → `plan`; `--apply-plan` → `apply`
  (one-line deprecation warning; prefer subcommands).
- **`after_tasks` hook** — runs `plan-tickets` only; apply is always separate.
- **Write phase** — runs only on **apply** (plan never creates Jira issues).

## [1.4.1] - 2026-05-29

### Added

- **Plan / apply workflow** — `--dry-run` plans into `jira-map.md`; live run or
  `--apply-plan` creates Jira issues from `TBD` rows only (Terraform-style). Prompts
  to **use existing map**, **regenerate**, or **add new tasks only** when a map exists.

- **Incremental sync** — after tickets exist, new `tasks.md` Task IDs are planned as
  additional `TBD` rows and created on apply without duplicating mapped Task IDs.
  New sub-tasks reuse an existing phase key when the phase is already in **Map**.

- **`--fresh` / `--regenerate`** — skip reuse prompts and replan from `tasks.md`
  (regenerate on `created` maps preserves existing Task ID → key bindings).

### Changed

- **Unified `jira-map.md` for dry-run and live-run** — `--dry-run` and `save-plan-only`
  write draft or pending rows in `jira-map.md` (`TBD` keys) instead of a separate
  `jira-plan.md`. Live runs set `**Status**: created` with real keys when fully applied.

- **Preview and Links sections** — draft maps include the Step 9 preview tree under
  `## Preview` and clickable Jira URLs under `## Links` (appendix; not parsed by
  `implement`). Created maps keep **Map** + deduplicated **Links**.

- **`jira-map.md` writes merge** — apply updates and appends rows instead of
  overwriting the full file. `**Status**: created` may include pending `TBD` rows
  until incremental apply completes.

- **`implement` rejects pending plans** — stops when `**Status**: draft` or any `TBD`
  key appears in the **Map** table; parses **Map** only (ignores Preview/Links).

### Removed

- **`jira-plan.md`** — no longer written. Legacy files may remain in feature dirs but
  are ignored by commands.

## [1.4.0] - 2026-05-27

### Added

- **Default Jira progress policy** at `config/jira-progress-policy.yml`. Projects can
  copy it to `.specify/jira-progress-policy.yml` to customize semantic statuses,
  monotonic transition behavior, task classification, commit requirements, local
  quality gates, PR review gates, and phase completion gates.

- **Event-based progress recording** in `implement`: phase/task start, task commit,
  local gate success, PR open, remote check success, and PR merge are modeled as
  separate events instead of collapsing everything into "Done" when a task is checked off.

- **Configurable TDD/test-task handling**: red-phase test tasks default to `in_review`
  on commit and move to `done` only when the configured done gate is satisfied.

### Changed

- `implement` now uses Jira workflow transitions (`getTransitionsForJiraIssue` +
  `transitionJiraIssue`) rather than treating status as a field update.

- Phase tickets no longer move to `Done` just because all local task checkboxes are
  checked. By default, sub-tasks require local gates and phase tickets require a
  merged PR before moving to `Done`.

- `implement` now explicitly requires task/phase commits before Jira review/done
  transitions. "Do not push" and "do not create PRs" are remote-operation restrictions;
  they are not interpreted as "do not commit" unless explicitly stated.

## [1.3.0] - 2026-05-26

### Added

- **Story-card format support** in `taskstotickets`: detects a second task format
  (`### PREFIX-NNN: Title` with Name / Description / Acceptance Criteria / Dependencies
  blocks) alongside the existing checklist format. Both formats map cleanly to the same
  Jira hierarchy; sub-tasks are created from acceptance criteria instead of task lines.
  Includes story-card-specific title rules (use Name field directly) and dependency
  linking via `createJiraIssueLink`.

- **`--parent` / PARENT_KEY flow** replaces the previous Epic create/confirm/reuse
  flow. If a Jira key is in `$ARGUMENTS` it is used directly; otherwise the user is
  prompted once with a plain "key or none" question. This works for any issue type
  (Epic, Story, Task) as the parent — not just Epics.

- **Checklist status gate** in `implement`: if `checklists/` exists, all files are
  scanned and a pass/fail table is shown before implementation begins.

- **Project setup verification** in `implement`: detect and create/verify ignore files
  (`.gitignore`, `.dockerignore`, `.eslintignore`, etc.) based on the detected tech stack.

### Changed

- **Flat `jira-map.md` schema** replaces the previous Story Map + Sub-task Map
  two-section format. The new format is a single table:
  `| Phase Ticket | Sub-task | Task ID | Description |`
  Description-only phases use `—` in the Sub-task column. This format is simpler
  to parse and scan at a glance.

- **`implement` command** updated to parse the flat `jira-map.md` schema and
  display a richer status table (Phase Ticket | Title | Task IDs | Progress | Status).

- **Step numbers in `taskstotickets`** renumbered to reflect the new flow:
  Steps 0–13 (was 1–14 with different structure).

- **`extension.yml` removes the `requires.commands` list** — the extension works
  alongside core Spec Kit commands but does not formally depend on them by name.

> **Migration note**: existing `jira-map.md` files written by v1.x use the old
> Story Map / Sub-task Map format. Re-run `/speckit.jira.taskstotickets` to
> generate a fresh `jira-map.md` in the new format before using `implement`.

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
