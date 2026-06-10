---
description: Plan or apply Jira tickets from tasks.md (plan/apply subcommands; flat jira-map.md). Checklist and story-card formats supported.
tools:
  - 'Atlassian/getAccessibleAtlassianResources'
  - 'Atlassian/getVisibleJiraProjects'
  - 'Atlassian/getJiraIssue'
  - 'Atlassian/getJiraProjectIssueTypesMetadata'
  - 'Atlassian/createJiraIssue'
  - 'Atlassian/editJiraIssue'
  - 'Atlassian/searchJiraIssuesUsingJql'
  - 'Atlassian/getIssueLinkTypes'
  - 'Atlassian/createIssueLink'
scripts:
  sh: ../scripts/bash/check-prerequisites.sh
  ps: ../scripts/powershell/check-prerequisites.ps1
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

### Parse mode and parent key

Tokenize `$ARGUMENTS` (space-separated). Set defaults: `MODE` = `plan`, `PARENT_KEY` = unset.

| Token | Action |
|-------|--------|
| `plan` | `MODE` = `plan` (explicit) |
| `apply` | `MODE` = `apply` |
| `[A-Z]+-\d+` | `PARENT_KEY` = token |
| `--fresh` or `--regenerate` | skip “use existing map?” prompts; replan from `tasks.md` |
| `none` | `PARENT_KEY` = null (only when prompting; not combined with a Jira key) |
| `--dry-run` or `-n` | **Deprecated** — treat as `plan`; print one-line warning to use `plan` |
| `--apply-plan` | **Deprecated** — treat as `apply`; print one-line warning to use `apply` |

**Examples** (all valid):

```text
/speckit.jira.taskstotickets plan ENG-6867
/speckit.jira.taskstotickets apply ENG-6867
/speckit.jira.taskstotickets ENG-6867          # plan (default)
/speckit.jira.taskstotickets plan ENG-6867 --regenerate
```

> [!NOTE]
> **Deprecated flags (v1.4.2):** `--dry-run`, `-n` → `plan`; `--apply-plan` → `apply` (warning only).
> Prefer `plan` / `apply` subcommands or `/speckit.jira.plan-tickets` / `apply-tickets`.

| Mode | Invocation | Writes `jira-map.md` | Jira creates |
|------|------------|----------------------|--------------|
| **plan** | `plan KEY` or `KEY` alone | Yes (`TBD` / pending rows) | No |
| **apply** | `apply KEY` | Merges real keys | Yes (`TBD` rows only) |

Fine-tune the **Map** table while keys are `TBD`, then run **apply**.

When invoked from the `after_tasks` hook with no subcommand, treat as **`plan`**.

### Apply prerequisites (enforce before any Jira writes)

When `MODE` is `apply`, validate immediately after loading paths (before planning logic):

1. **`FEATURE_DIR/jira-map.md` must exist** — else **STOP**:
   > "No plan found. Run `/speckit.jira.taskstotickets plan {PARENT_KEY}` first."

2. **Flat map required** — if the file has `## Story Map` or `## Sub-task Map` but no `## Map`
   table, **STOP**:
   > "Legacy jira-map format. Run `taskstotickets plan {PARENT_KEY}` to migrate to flat **Map**
   > (preserves existing keys), then `apply`."
   On **plan**, run **Step 4a — Migrate legacy map** automatically instead of stopping.

3. **Plan must be applicable** — either:
   - the **Map** table contains at least one row where Phase Ticket or Sub-task starts
     with `TBD`, or
   - **Phase Dependencies** contains at least one row with `Status = pending` and both
     sides have real Jira keys.

   If `**Status**: created`, there are no `TBD` rows, and there are no pending
   dependency links, **STOP**:
   > "Nothing to apply. Run `plan` to add new tasks from tasks.md (delta), or confirm the map is complete."

4. Set `PLAN_SOURCE` = `apply-pending` for apply runs (no replan unless `--regenerate`).

Apply **never** runs a full naive ticket generation from `tasks.md` alone.

---

## Outline

### ── PLAN PHASE (read-only) ───────────────────────────────────────────────────

### Step 0 — Identify Parent Ticket

If `PARENT_KEY` is still unset after parsing, scan remaining tokens for `[A-Z]+-\d+`.
If none, ask the user:

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

### Step 1a — Load Jira Extension Configuration

Load and deep-merge Jira configuration in this order, where later layers override
earlier layers:

1. Extension defaults: `.specify/extensions/jira/extension.yml` → `config.defaults`
2. User-global override (extension-specific): `$HOME/.specify/extensions/jira/jira-config.yml`
3. Repo/team config: `.specify/extensions/jira/jira-config.yml`
4. Repo-local override: `.specify/extensions/jira/local-config.yml`
5. Environment variables: `SPECKIT_JIRA_*`

Missing files are skipped. The user-global layer is not managed by Spec Kit core;
this command reads it explicitly.

**Legacy compatibility**: if `.specify/jira-progress-policy.yml` exists, print a
one-line deprecation warning and merge its contents under `jira.progress_policy`
between layer 2 and layer 3. Do not rewrite or move the legacy file.

The task-to-ticket workflow uses:
- `jira.phase_dependencies.create_blocks_links_on_apply`
- `jira.progress_policy.semantic_statuses` only for preview/status wording when needed

---

### Step 2 — Connect to Jira

Call `getAccessibleAtlassianResources` to obtain the Jira Cloud ID and site URL.

Store `JIRA_BROWSE_BASE` as `{site-url}/browse/` (trailing slash optional; use
consistent URL form in the Links appendix).

> [!CAUTION]
> ONLY PROCEED IF A VALID JIRA CLOUD RESOURCE IS RETURNED.

---

### Step 3 — Determine Project Key

- If `PARENT_KEY` is set: extract the project key from it (the part before the hyphen).
- If `PARENT_KEY` is null: call `getVisibleJiraProjects`, list available projects,
  and ask the user to confirm the target project key before proceeding.

---

### Step 4 — Load Map, Validate Parent, Plan Source

If `PARENT_KEY` is set:
- Call `getJiraIssue` to confirm the parent ticket exists. If not, **STOP** and report.

If `FEATURE_DIR/jira-plan.md` exists (legacy), note it is deprecated — do not use it.

**Load `FEATURE_DIR/jira-map.md` when present** and set `MAP_MODE`:

| Condition | `MAP_MODE` |
|-----------|------------|
| No file | `none` |
| `**Status**: draft` (or all phase keys are `TBD`) | `draft` |
| `**Status**: created`, no `TBD` in **Map** | `created` |
| `**Status**: created`, some `TBD` rows in **Map** | `created-pending` |

Parse the **Map** table into `EXISTING_ROWS` (all rows) and index **mapped Task IDs**
(checklist: `T001`, ranges `T003–T005`; story-card: use the Task ID column as stored).
The `Review Unit` column is optional for maps written before v1.5.0; treat missing
values as `—`.

**Stale validation** (skip when `MAP_MODE` is `none` or `draft`):
- Call `getJiraIssue` on the first real phase key in **Map**.
- If missing in Jira: rename `jira-map.md` → `jira-map.stale.md` and set `MAP_MODE` to `none`.

---

### Step 4a — Migrate legacy `jira-map.md` (plan only)

Run when `MODE` is `plan` and `jira-map.md` has `## Story Map` and/or `## Sub-task Map`
but no `## Map` section.

1. Copy `jira-map.md` → `jira-map.legacy.md` (do not delete until migration succeeds).
2. Read **Parent** from header (`**Parent**` or legacy `**Epic**` line).
3. Build a **story index** from `## Story Map`: `Story Key` → `Task IDs` range + phase title
   (text after `Phase N:` in the Phase column, trimmed; drop `Phase N:` prefix for titles).
4. Build **flat Map rows**:
   - For each row in `## Sub-task Map` (`Sub-task Key | Task ID | Story Key`):
     - Phase Ticket column = `{Story Key} {title}` (title from story index or Jira `getJiraIssue` summary).
     - Sub-task = sub-task key; Task ID = task id; Description = from `tasks.md` line for that ID if available;
       Review Unit = `—`.
   - For each story in the story index with **no** sub-task rows in Sub-task Map (description-only phase):
     - One row: Phase Ticket = `{Story Key} {title}`; Sub-task = `—`; Task ID = range (e.g. `T001–T011`);
       Description = `(description-only)`; Review Unit = `—`.
5. Set header `**Status**: created` (keys are real, not `TBD`).
6. Preserve `**Project**`, `**Format**`, dates; add `**Migrated**`: YYYY-MM-DD.
7. Write `## Map` with the flat table; keep or omit legacy sections (prefer **omit** Story/Sub-task maps).
8. Rebuild `## Links` from all real keys (parent, stories, sub-tasks) using `JIRA_BROWSE_BASE`.
9. Report: `Migrated legacy map → flat Map ({N} rows). Backup: jira-map.legacy.md`
10. Set `MAP_MODE` from the new file (`created` or `created-pending` if `TBD` rows already present).

Do **not** create or modify Jira issues during migration.

---

**Plan source** — only when `MODE` is `plan`, unless `--fresh` / `--regenerate` is set:

| `MAP_MODE` | Prompt |
|------------|--------|
| `none` | Generate a new plan (no prompt). |
| `draft` | **Use existing draft**, **regenerate from tasks.md**, or **show existing only** (no write)? |
| `created` | **Add new tasks only** (delta as `TBD` rows), **regenerate full map** (keep bindings), or **show map only**? |
| `created-pending` | **Replan delta**, **show map only**, or remind: run **`apply`** for pending `TBD` rows? |

When `MODE` is `apply`, skip plan-source prompts (apply prerequisites already enforced).

Set `PLAN_SOURCE`:

- `use-existing` — use `jira-map.md` as-is (plan show-only).
- `regenerate` — rebuild from `tasks.md` (see Step 7 delta rules).
- `delta-only` — only unmapped Task IDs from `tasks.md`.
- `apply-pending` — **apply mode only**: create Jira for `TBD` rows (no replan).

When `PLAN_SOURCE` is `use-existing` on a **plan** run: display the map and **STOP**
(no file write unless the user asked to normalize Links/Preview).

When `PLAN_SOURCE` is `regenerate` and `MAP_MODE` is `created`: **preserve** every
**Map** row whose Task ID still exists in `tasks.md` (keep real keys). Recompute only
missing work units / Task IDs. Do **not** remove Jira issues for tasks dropped from
`tasks.md` — report them under **## Drift** in the written map (optional section).

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

### Step 7 — Parse Work Units (and Delta)

**If `MAP_MODE` is `created` or `created-pending` and `PLAN_SOURCE` is `delta-only` or
`regenerate`**: after parsing, **filter out** work units and task lines whose Task IDs
are already in the mapped Task ID index. A phase with **some** mapped and **some** new
Task IDs is kept; only the **new** lines become sub-tasks (reuse the existing phase key
from **Map** when applying).

**If `PLAN_SOURCE` is `apply-pending`**: skip parsing for planning; use **Map** rows
where Phase Ticket or Sub-task starts with `TBD` as the create queue.

**Checklist format** — group by `## Phase N: …` headers:
- Each phase = one work unit.
- Build a **phase index** for every phase header, even if the phase is later filtered
  out by delta rules:
  - `PHASE_REF`: the token after `Phase` (`1`, `2`, `6`, `6A`, `10`, …).
  - `PHASE_LABEL`: text after `## Phase N:` up to the first ` - `, when present.
  - `US_REF`: `USN` when the header contains `User Story N`.
  - `TITLE_SOURCE`: the full header text used by Step 8 to draft the Jira title.
  - `MAP_KEY`: the existing real phase key from **Map** when one is already mapped
    for the phase; otherwise `TBD`.
- Phase index refs are case-insensitive for matching (`6a` = `6A`, `us2` = `US2`)
  but should be written in normalized form in `jira-map.md`.
- If two phases resolve to the same `PHASE_REF` or `US_REF`, **STOP** during plan
  with an ambiguity message before writing `jira-map.md`.
- Collect all incomplete task lines (`- [ ] T00N …`). Skip phases where all tasks
  are `- [x]`.
- A phase with mixed complete/incomplete tasks is included; note already-done tasks
  in the ticket description.
- **Description-only decision** (store `DESCRIPTION_ONLY` and `DESCRIPTION_ONLY_REASON`
  on each work unit):

  1. If the phase has **≥ 4** incomplete task lines → **not** description-only; create
     sub-tasks (overrides bootstrap labels — e.g. "Contract Hardening" with T103–T106).
  2. Else if **≤ 3** incomplete tasks → description-only; reason:
     `≤3 incomplete tasks (N)`.
  3. Else extract **phase label**: text after `## Phase …:` up to (not including) the
     first ` - `. If that label **exactly equals** (case-insensitive) one of:
     `setup`, `foundation`, `foundational`, `prerequisites`, `prerequisite`,
     `infrastructure`, `infra`, `polish`, `cross-cutting`, `cleanup`, `qa`, `final`
     → description-only; reason: `phase label "{label}"`.
  4. Otherwise → sub-tasks.

  - **Never** substring-match keywords inside deliverable titles (`hardening`,
    `validation`, `teardown` in a long title are **not** bootstrap signals).
  Tasks for description-only phases are embedded in the ticket description.

**Story-card format** — group by `### PREFIX-NNN: Title` blocks:
- Each story card = one work unit.
- Collect all incomplete acceptance criteria lines (`- [ ] …`).
- Skip cards where Status is `done` or all criteria are checked.
- Also extract: **Name**, **Description**, **Files to modify**, **Dependencies**.

---

### Step 7b — Parse Phase Dependencies

Run for **checklist format** only.

Locate `## Dependencies` in `tasks.md` and read bullets until the next `##` heading.
If no section exists, set `PHASE_DEPENDENCIES` to an empty list; `jira-map.md` must
still include an empty **Phase Dependencies** table.

Resolve mentions using the phase index from Step 7:
- `Phase 6`, `Phase 6A`, `phase 6a` → matching `PHASE_REF`
- `US2`, `User Story 2`, `user story 2` → matching `US_REF`
- Ignore parenthesized task ranges like `(T103–T106)` for dependency linking
- Never resolve individual task IDs (`T083`, `T084`) as phase dependencies

Extract directed edges as **Blocker → Blocked**:

| Prose pattern | Edge |
|----------------|------|
| `Phase X depends on Phase Y` | Y blocks X |
| `USX depends on USY` / `User Story X depends on User Story Y` | Y blocks X |
| `Phase X must complete before Phase Y` | X blocks Y |
| `Phase X must merge before Phase Y` / `should be merged before` | X blocks Y |
| `Phase X depends on A, B, and C` | A, B, and C each block X |

For compound sentences, extract every resolvable phase/story mention in the dependent
or downstream clause. Example: `Phase 6A depends on Phase 6 and must merge before
Phase 6B or Phase 6C` yields `Phase 6 blocks Phase 6A`, `Phase 6A blocks Phase 6B`,
and `Phase 6A blocks Phase 6C`.

Deduplicate identical edges by normalized blocker/blocked refs. If a dependency
mention cannot be resolved, keep a row with `Status = unresolved` and the original
source bullet so the user can repair the map. If the graph contains a cycle, print a
preview warning; do not fail plan by default.

Do **not** parse `## Parallel Examples`, `## Implementation Strategy`, or inline task
phrases such as `after T092` for phase dependencies.

Story-card `**Dependencies**` fields keep their existing story-card behavior and are
not part of **Phase Dependencies**.

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
> **Plan** (`MODE` = `plan`): no Jira writes in this step; write `jira-map.md` at the end.
> **Apply** (`MODE` = `apply`): confirm once, then proceed to Write Phase for `TBD` rows only.

Build and display a preview tree. For each **description-only** work unit, append the
reason on the same line, e.g. `[Task]  ⚠ description-only — ≤3 incomplete tasks (2)`.

After the tree, if any work unit has `DESCRIPTION_ONLY`, print a **Description-only
warnings** block:

```text
⚠ Description-only (tasks will be embedded in the story description, not sub-tasks):
  · {phase title} — {DESCRIPTION_ONLY_REASON}
  · Polish and cross-cutting cleanup — ≤3 incomplete tasks (2)
```

Ask the user to confirm they intend description-only for each listed phase before
plan write (plan continues automatically unless they abort).

For checklist format, print a **Phase dependencies** block after description-only
warnings:

```text
Phase dependencies (Blocks):
  Phase 6 → Phase 6A  (Phase 6 blocks Phase 6A)
  Phase 6A → Phase 6B (Phase 6A blocks Phase 6B)
Unresolved: 0 | Planned: 2 | Warnings: none
```

If no phase dependencies were found:

```text
Phase dependencies (Blocks): none
```

*Checklist format:*
```
Project: PROJ  |  Parent: PROJ-100 (or "standalone")  |  Format: checklist
────────────────────────────────────────────────────────────────────────────
Bootstrap project and install dependencies   [Task]  2 sub-tasks
  ├─ Create project structure                [Sub-task]
  └─ Install and configure dependencies      [Sub-task]
Implement core data model                    [Task]  3 sub-tasks
  ├─ …
Polish and cross-cutting cleanup             [Task]  ⚠ description-only — ≤3 incomplete tasks (2)

⚠ Description-only (tasks embedded in story description, not sub-tasks):
  · Polish and cross-cutting cleanup — ≤3 incomplete tasks (2)

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

**If `PLAN_SOURCE` is `use-existing`** (`MODE` = `plan`): show the map and **STOP**.

**If `MODE` is `plan`** (and not `use-existing`):
- Write or **merge** `FEATURE_DIR/jira-map.md`:
  - First plan (`MAP_MODE` `none`): `**Status**: draft`, all keys `TBD`.
  - Incremental (`MAP_MODE` `created`): keep existing **Map** rows; append new rows with
    `TBD`; keep `**Status**: created` (pending apply).
- Write or **merge** **Phase Dependencies**:
  - Recompute edges from `tasks.md` on every plan.
  - Preserve existing `linked` rows when the same blocker/blocked edge is still present.
  - Add new edges as `planned` when either side is still logical (`Phase N` / `USN`) or
    `pending` when both sides already have real Jira keys.
  - Keep unresolved edges as `unresolved` with their source bullet.
  - On `--regenerate`, report previously linked rows that no longer appear in
    `tasks.md` as drift; do not remove Jira links.
- Report counts: existing mapped tasks, new `TBD` rows, skipped duplicates.
- Report dependency counts: `linked`, `pending/planned`, `unresolved`, and cycle warnings.
- Include description-only warnings in the **Preview** section of `jira-map.md` when any apply.
- Include the phase dependency summary in **Preview** for checklist format.
- Remind: edit **Map**, then run `apply {PARENT_KEY}`.
- **STOP** — do not proceed to the Write Phase.

**If `MODE` is `apply`**:
- Show count of `TBD` rows to be created.
- Ask once: **"Apply {N} planned tickets to Jira? (yes / no)"**
- `no` → abort; no Jira writes.
- `yes` → continue to Write Phase (additive rules below).

#### jira-map.md format (draft and created)

Dry-run and live-run use the **same file** (`jira-map.md`). Only `**Status**`, keys,
and the Links appendix differ.

> [!IMPORTANT]
> When `**Status**: draft`, phase and sub-task keys are `TBD` placeholders. Do not
> treat the map as evidence that phase/sub-task tickets exist. `/speckit.jira.implement`
> refuses draft maps.

**Draft** (`MODE` = `plan`):

```markdown
# Jira Task Map

**Status**: draft
**Project**: PROJ | **Parent**: PROJ-100 (or "None") | **Format**: checklist | **Generated**: YYYY-MM-DD

> **Draft** — Phase and sub-task keys are placeholders (`TBD`). No phase/sub-task Jira
> issues exist yet. Run `/speckit.jira.taskstotickets apply {PARENT_KEY}` to create
> tickets and set **Status** to `created`.

**Story points (planned)**: 5 — 3 phases · 17 tasks · 1 external integration

## Preview

Project: PROJ | Parent: PROJ-100 | Format: checklist
────────────────────────────────────────────────────────────────────────────
Bootstrap project and install dependencies   [Task]  2 sub-tasks
  ├─ Create project structure                [Sub-task]
  └─ Install and configure dependencies      [Sub-task]
Total: 3 tickets · 5 sub-tasks

## Map

| Phase Ticket | Sub-task | Task ID | Description | Review Unit |
|--------------|----------|---------|-------------|-------------|
| TBD Bootstrap project and install dependencies | TBD | T001 | Create project structure | — |
| TBD Bootstrap project and install dependencies | TBD | T002 | Install dependencies | — |
| TBD Polish and cross-cutting cleanup | — | T003–T005 | (description-only) | — |

## Phase Dependencies

| Blocker | Blocked | Status | Source |
|---------|---------|--------|--------|
| — | — | — | *(none)* |

## Links

**Browse base**: `https://your-site.atlassian.net/browse/`

| Key | Title | Link |
|-----|-------|------|
| PROJ-100 | Parent epic title | [PROJ-100](https://your-site.atlassian.net/browse/PROJ-100) |
| TBD | Bootstrap project and install dependencies *(planned phase)* | — |
| TBD | Create project structure *(planned sub-task)* | — |
```

- Use `TBD` as the key token in **Map** columns; append the human title after `TBD`
  and a space (same shape as live `KEY Title` rows).
- Always write `Review Unit` in **Map**:
  - Default value is `—`.
  - This is optional sync/review metadata, not ticket creation input.
  - Users may edit it while the map is draft or after apply to identify a finer-grained
    review unit such as a sub-task PR, repo slice, or branch/PR evidence.
  - Preserve existing values during replan/apply; do not overwrite user edits.
- Always write **Phase Dependencies** immediately after **Map**:
  - Use `Phase N` / `USN` refs while keys are `TBD`.
  - Use `Status = planned` for resolvable draft edges.
  - Use `Status = unresolved` for rows that could not resolve cleanly.
  - If no edges exist, write the placeholder row `— | — | — | *(none)*`.
- **Preview** holds the Step 9 ASCII tree and totals.
- **Links**: one row per distinct parent key (real URL when `PARENT_KEY` is set) plus
  one row per planned phase/sub-task (`TBD` in Key column, `—` in Link until created).
- Do **not** write `jira-plan.md`.

**Created** (after successful Step 11 — see Step 12):
- Set `**Status**: created` when no `TBD` rows remain in **Map**.
- Replace each applied `TBD` with the real key returned from Jira (preserve title text).
- Include `**Story points**`: N (applied to PARENT) when parent points were set.
- **Links**: deduplicated rows for every real key in the map (parent, phases, sub-tasks).

**Incremental / pending rows** (`MAP_MODE` was `created` or `created-pending`):
- **Map** = prior rows with real keys **plus** new rows (never duplicate Task ID).
- Only **new** `TBD` rows are written on plan; only those rows are created on apply.
- Reuse an existing phase key when new Task IDs belong to a phase already in **Map**
  (match by phase title text after the key).

---

### ── WRITE PHASE (`MODE` = `apply` only, after "yes") ───────────────────────

### Step 10 — Set Story Points

If `PARENT_KEY` is set, apply `ESTIMATED_POINTS` to the parent:
- No existing points → confirm, then call `editJiraIssue`.
- Existing points differ → show both, ask to update or keep.
- Existing points match → report "aligned" and move on.

---

### Step 11 — Create Tickets (additive)

> [!CAUTION]
> **Never create duplicate issues.** Before each create, confirm the Task ID is not
> already present in **Map** with a real key (`[A-Z]+-\d+`). Skip creates for Task IDs
> already mapped.

**Apply queue**: rows where Sub-task or Phase Ticket starts with `TBD`, or
`PLAN_SOURCE` is `apply-pending`. When adding sub-tasks to an **existing phase**, use
`createJiraIssue` with the **existing phase key** as parent — do not create a new
phase ticket.

If there are no `TBD` rows but **Phase Dependencies** has retryable `pending` rows,
skip ticket creation and proceed directly to Step 11b.

For each work unit in order (only units with at least one unmapped Task ID):

1. **Create the work-unit ticket** (`createJiraIssue`) **only when the phase has no
   real key yet**:
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
   **Dependencies** field, call `createIssueLink` to add "blocks"/"is blocked by" links.

Report progress after each ticket:
`✓ PROJ-456 Implement SMS delivery pipeline · 3 sub-tasks created`

---

### Step 11b — Link Phase Dependencies

Run after all Step 11 issue creation calls succeed and before persisting
`jira-map.md`.

Skip this step when:
- `FORMAT` is not `checklist`, or
- `jira.phase_dependencies.create_blocks_links_on_apply` is `false`, or
- **Phase Dependencies** has only the placeholder row.

Procedure:

1. Call `getIssueLinkTypes` once and confirm a link type named `Blocks` exists.
   If it does not exist, **STOP** with a clear project-configuration message.

2. For each **Phase Dependencies** row with `Status` = `planned` or `pending`:
   - Resolve **Blocker** and **Blocked** to real Jira keys from the merged **Map**.
   - Skip rows where either side is still `TBD`, unresolved, or the same issue key.
   - Call `createIssueLink` with:
     - `type`: `Blocks`
     - `inwardIssue`: blocker key
     - `outwardIssue`: blocked key
   - On success: set row `Status` to `linked`.
   - If Jira reports the link already exists: treat as success and set `linked`.
   - On any other link error: leave row `Status` as `pending`, report the error, and
     continue with remaining dependency rows.

`createIssueLink` direction: for "A is blocked by B", use `inwardIssue = B` and
`outwardIssue = A`.

---

### Step 12 — Persist jira-map.md

> [!CAUTION]
> **Only write `jira-map.md` after ALL `createJiraIssue` calls in the apply batch succeed.**
> If any call fails, report the error and **STOP** — do not write a partial map.

**Merge, do not blind overwrite:**
- Keep all prior **Map** rows whose Task IDs were not in this apply batch.
- Update rows that were applied: replace `TBD` with real keys in Phase Ticket / Sub-task.
- Append brand-new rows from this run.
- Merge **Phase Dependencies** after key replacement:
  - Replace logical refs (`Phase N`, `USN`) with `KEY Title` when both sides are now
    known from **Map**.
  - Preserve `linked` rows.
  - Keep failed or skipped link attempts as `pending` so a later apply can retry.
  - Keep unresolved rows unchanged.
- Rebuild **Links** from the merged **Map** (dedupe by key).
- Set `**Status**: created` only when **Map** has zero `TBD` keys; otherwise
  `**Status**: created` with pending `TBD` rows (implement blocks until apply completes).

First-time apply from `draft`: after success, set `**Status**: created` and
`**Created**` date (replace `**Generated**` if present).

```markdown
# Jira Task Map

**Status**: created
**Project**: PROJ | **Parent**: PROJ-100 (or "None") | **Format**: checklist | **Created**: YYYY-MM-DD

**Story points**: 5 (applied to PROJ-100)

## Map

| Phase Ticket | Sub-task | Task ID | Description | Review Unit |
|--------------|----------|---------|-------------|-------------|
| PROJ-456 Implement SMS delivery pipeline | PROJ-460 | T001 | Create adapter | — |
| PROJ-456 Implement SMS delivery pipeline | PROJ-461 | T002 | Add retry logic | monorepo#123 |
| PROJ-457 Bootstrap project dependencies | — | T003–T005 | (description-only) | — |

## Phase Dependencies

| Blocker | Blocked | Status | Source |
|---------|---------|--------|--------|
| PROJ-457 Bootstrap project dependencies | PROJ-456 Implement SMS delivery pipeline | linked | Phase 2 blocks Phase 3 |

## Links

**Browse base**: `https://your-site.atlassian.net/browse/`

| Key | Title | Link |
|-----|-------|------|
| PROJ-100 | Parent title | [PROJ-100](https://your-site.atlassian.net/browse/PROJ-100) |
| PROJ-456 | Implement SMS delivery pipeline | [PROJ-456](https://your-site.atlassian.net/browse/PROJ-456) |
| PROJ-460 | Create adapter | [PROJ-460](https://your-site.atlassian.net/browse/PROJ-460) |
```

- One row per sub-task in **Map**; description-only phases use `—` in the Sub-task column.
- The Phase Ticket column repeats `KEY Title` for each of its sub-tasks.
- `Review Unit` is optional metadata for review/sync workflows. Current ticket creation
  ignores it; future/manual sync-state reconciliation may use it to associate a row
  with a PR, repo, branch, or other review boundary.
- **Phase Dependencies** stores phase-level Jira Blocks relationships. `implement`
  reads this section when evaluating phase prerequisites.
- **Links** is for navigation only; `/speckit.jira.implement` reads **Map**, not Links.
- This file is consumed by `/speckit.jira.implement`.

---

### Step 13 — Report

- Path to `jira-map.md`
- One line per phase ticket: key → title · N sub-tasks created
- Total phase tickets and sub-tasks created
- Parent ticket and story points applied (if applicable)
- Clickable link to the parent ticket or Jira board
