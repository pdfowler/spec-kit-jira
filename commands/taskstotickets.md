---
description: Convert existing tasks into a three-tier Jira hierarchy (Epic → Story per phase → Sub-tasks per task) for the feature based on available design artifacts.
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

The user may optionally provide an **Epic ticket key** (e.g., `PROJ-100`). When provided, all Stories are created under that Epic. When omitted, the command will confirm or create an Epic at runtime.

## Hierarchy

This command produces a three-tier Jira structure:

```
Epic  (one per spec version — organizes the sprint/time allocation)
  └── Story  (one per ## Phase N: header — primary board visibility)
        └── Sub-task  (one per T00X item — granular implementation tracking)
```

Sub-tasks are **not** created for every phase. They are skipped automatically for
setup/infrastructure phases and phases with ≤ 3 tasks. All other phases are
presented for confirmation before sub-tasks are created.

---

## Outline

### Step 1 — Load Feature Context

Run `{SCRIPT} --json --require-tasks --include-tasks` from repo root and parse
`FEATURE_DIR` and `AVAILABLE_DOCS` list. All paths must be absolute. For single
quotes in args like "I'm Groot", use escape syntax: e.g. `'I'\''m Groot'`
(or double-quote if possible: `"I'm Groot"`).

From the executed script, extract the path to **tasks.md** and read it.

### Step 2 — Connect to Jira

Call `getAccessibleAtlassianResources` to obtain the Jira Cloud ID.

> [!CAUTION]
> ONLY PROCEED IF A VALID JIRA CLOUD RESOURCE IS RETURNED.

### Step 3 — Confirm or Create the Epic

> **The Epic is the organizing unit for the entire spec version.**
> One Epic should exist per `specs/{feature}/v{N}` directory.

**A. Check for existing mapping**: Read `FEATURE_DIR/jira-map.md` if it exists.
   - If the file contains `**Epic**: PROJ-NNN`, extract that key.
   - Present: `"Found existing Epic PROJ-NNN from jira-map.md. Use this? (yes / enter different key / create new)"`

**B. If no mapping exists** and the user provided an Epic key as argument, validate
   it via `getJiraIssue`. If valid, use it. If not found, **STOP** and report.

**C. If no mapping and no argument**, ask:
   `"No Epic found for this spec. Enter an existing Epic key (e.g., PROJ-100) or a name to create a new one:"`

   - If the user provides a key (matches `[A-Z]+-\d+`): validate via `getJiraIssue` and use it.
   - If the user provides a name: call `createJiraIssue` with the Epic issue type and a summary
     derived from the spec directory name and version (e.g., `"Two-Way SMS v1"`). Use the
     created key going forward.

Store the confirmed Epic key as `EPIC_KEY`.

### Step 4 — Determine Project Key

Extract the project key from `EPIC_KEY` (the part before the hyphen). All Stories
and Sub-tasks will be created in this same project.

### Step 5 — Discover Issue Types

Call `getJiraProjectIssueTypesMetadata` for the project. Identify the exact names for:
- `EPIC_TYPE` — the Epic issue type (for reference/validation)
- `STORY_TYPE` — the Story issue type (typically `"Story"` or `"Task"` if Story is unavailable)
- `SUBTASK_TYPE` — the Sub-task issue type (typically `"Sub-task"` or `"Subtask"`)

> [!CAUTION]
> Use the **exact** names returned by the API. Never hardcode `"Story"` or `"Sub-task"`.
> If no Story type exists, use the non-Epic, non-Sub-task task type for the middle tier.

### Step 6 — Estimate and Set Story Points on the Epic

**Read current points**: From the Epic ticket fetched in Step 3, extract the story
points field (may be `story_points`, `customfield_10016`, or another custom field —
use the value returned by `getJiraIssue`).

**Estimate effort (AI-assisted baseline)**: Analyze ALL tasks from tasks.md to produce
an estimate that assumes the developer is using Spec Kit and an AI coding agent to implement.

| Signal | Points contribution |
|--------|---------------------|
| **Total incomplete phases** | Base: 1 pt per phase |
| **Task volume** (incomplete tasks across all phases) | 1–10 tasks: +0, 11–20: +1, 21–30: +2, 31+: +3 |
| **Integration complexity** — tasks involving external services, databases, auth, or third-party APIs | +1 per distinct integration |
| **Testing depth** — dedicated test phases, contract tests, E2E, or performance benchmarks | +1 if significant test coverage required |
| **Novelty / research** — tasks referencing research.md, spikes, or unfamiliar tech | +1 if present |
| **Data model complexity** — >5 entities, migrations, or seed data | +1 if present |

Snap the raw total to the nearest Fibonacci value: **1, 2, 3, 5, 8, 13, 21**.

> The heuristic intentionally deflates compared to manual estimation because AI handles
> boilerplate, scaffolding, test generation, and repetitive implementation. Human effort
> concentrates on review, integration debugging, and design decisions.

**Apply or suggest**:

- **Epic has NO story points (null/0)**:
  - Present the estimate with a brief breakdown of contributing signals.
  - Ask the user to confirm or adjust before setting.
  - On confirmation, call `updateJiraIssue` to set the story points on the Epic.

- **Epic already HAS story points**:
  - Compare the existing value against the AI-assisted estimate.
  - **If they match**: report "Current estimate aligns with AI-assisted analysis" and move on.
  - **If they differ**: present both values with reasoning and offer to update.

### Step 7 — Discover Naming Convention

Search for existing Stories under the Epic using JQL:
`issueType = {STORY_TYPE} AND "Epic Link" = {EPIC_KEY} ORDER BY key ASC`
(Some Jira configs use `parent = {EPIC_KEY}` — try both if the first returns no results.)

- If sibling Stories exist: analyze their summary titles to extract the naming pattern
  (prefix style, voice, length) and match it for all new Stories.
- If no siblings: use the default convention below.

**Default naming convention**:
- Format: `Phase N: Concise action-oriented description`
- Keep the `Phase N:` prefix — it preserves execution order at a glance
- **NEVER** copy the raw phase heading from tasks.md verbatim
- Rewrite the part after `Phase N:` into a concise deliverable summary (≤ 10 words)
- Start with an action verb: Create, Implement, Add, Verify, Harden, etc.
- Describe the **outcome**, not internal artifacts

### Step 8 — Create Stories (one per incomplete Phase)

Parse tasks.md for `## Phase N:` headers. For each phase that contains at least one
incomplete task (`- [ ]`):

> [!CAUTION]
> SKIP any phase where ALL tasks are already completed (`- [x]` or `- [X]`).
> A phase with a mix of complete and incomplete tasks still gets a Story.

Create one `STORY_TYPE` issue per qualifying phase:
- **Summary**: Concise title per naming convention from Step 7.
- **Description** (Markdown): Phase goal/purpose, checkpoint criteria (if present),
  and the full list of task lines (with their IDs, `[P]`/`[US#]` markers, file paths).
  This gives the implementer full context without leaving the ticket.
- **Issue type**: `STORY_TYPE`
- **Parent / Epic Link**: `EPIC_KEY` (use the field name the project requires —
  `parent` for next-gen projects, `customfield_10014` / Epic Link for classic)

Record the created Story key for each phase: `phase_story_map[phase_N] = PROJ-NNN`.

### Step 9 — Sub-task Creation (selective, confirmed)

After all Stories are created, evaluate each phase for sub-task creation.

**Auto-skip** sub-tasks for a phase if ANY of these are true:
- The phase name (case-insensitive) contains any of: `setup`, `foundation`, `foundational`,
  `prereq`, `prerequisite`, `infrastructure`, `infra`
- The phase has ≤ 3 incomplete tasks total

For the remaining **candidate phases**, build a preview table and present it to the user:

```text
These phases qualify for sub-task breakdown:

| Story Key | Phase                  | Tasks | Sub-tasks will be created |
|-----------|------------------------|-------|---------------------------|
| PROJ-456  | Phase 3: User Story 1  |   9   | T006, T007, T008, ...     |
| PROJ-457  | Phase 4: User Story 2  |   7   | T015, T016, T017, ...     |

Create sub-tasks for: (all / select / skip)
```

- **all**: create sub-tasks for all candidate phases.
- **select**: ask the user to list which Story keys or phase numbers to include.
- **skip**: proceed without creating any sub-tasks.

For each approved phase, create one `SUBTASK_TYPE` issue per T00X line:
- **Summary**: The task description text (strip the `T00X`, `[P]`, `[US#]` markers;
  keep the human-readable action description).
- **Description**: The full raw task line as written in tasks.md, for traceability.
- **Issue type**: `SUBTASK_TYPE`
- **Parent**: The Story key for that phase (`phase_story_map[phase_N]`).

Record `subtask_map[T00X] = PROJ-NNN` for every sub-task created.

### Step 10 — Persist jira-map.md

Write a `jira-map.md` file to `FEATURE_DIR` (alongside tasks.md, plan.md, etc.):

```markdown
# Jira Task Map

**Project**: PROJ | **Epic**: PROJ-100 | **Created**: YYYY-MM-DD

## Story Map

| Story Key | Task IDs | Phase |
|-----------|----------|-------|
| PROJ-456  | T006, T007, T008, T009, T010, T011, T012, T013, T014 | Phase 3: User Story 1 |
| PROJ-457  | T001, T002 | Phase 1: Setup |

## Sub-task Map

| Sub-task Key | Task ID | Story Key |
|--------------|---------|-----------|
| PROJ-460     | T006    | PROJ-456  |
| PROJ-461     | T007    | PROJ-456  |
```

- `**Epic**`: the confirmed `EPIC_KEY`.
- **Story Map**: one row per created Story. Task IDs lists all T00X in that phase, comma-separated.
- **Sub-task Map**: one row per created Sub-task, mapping `Sub-task Key → Task ID → Story Key`.
  Omit this section entirely if no sub-tasks were created.
- The Story Map table is the primary index consumed by `/speckit.jira.implement`.

### Step 11 — Report

After all tickets are created and the mapping is saved, output a summary:

- Path to `jira-map.md`
- Epic key and title
- One line per Story: `PROJ-NNN → Phase N: <title> (<N> tasks, <M> sub-tasks or "no sub-tasks")`
- Total Stories created, total Sub-tasks created
- Story points set/updated/unchanged on the Epic
- Clickable link to the Epic on the Jira board
