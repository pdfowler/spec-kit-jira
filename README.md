# spec-kit-jira

A [Spec Kit](https://github.com/github/spec-kit) extension that integrates Jira into the spec-to-implementation workflow. Converts spec tasks into a three-tier Jira hierarchy and scopes implementation to individual tickets.

## What it does

This extension adds two commands:

| Command | Description |
|---------|-------------|
| `/speckit.jira.taskstotickets` | Converts `tasks.md` phases into a three-tier Jira hierarchy (Epic → Story → Sub-task) with a mandatory preview before any writes. Supports `--dry-run`. |
| `/speckit.jira.implement` | Runs implementation scoped to a single Jira Story's tasks; closes Sub-tasks and the Story in Jira as work completes. |

The workflow is:

1. **Generate tasks** with `/speckit.tasks` (core command)
2. **Preview and create Jira tickets** with `/speckit.jira.taskstotickets` — see exactly what will be created before committing
3. **Implement by ticket** with `/speckit.jira.implement PROJ-456` — executes only the tasks mapped to that Story

## Hierarchy

```
Epic   (one per spec version — the sprint/time organizing unit)
  └── Story    (one per ## Phase N: in tasks.md — primary board visibility)
        └── Sub-task  (one per T00X item — granular implementation tracking)
```

Sub-tasks are created selectively. Phases are auto-skipped when their name contains
`setup`, `foundation`, `prereq`, `infra`, or similar, or when they have ≤ 3 tasks.
All other phases are shown in the preview and created when you confirm.

## Requirements

- [Spec Kit](https://github.com/github/spec-kit) >= 0.8.3
- [Atlassian MCP server](https://www.npmjs.com/package/@anthropic/atlassian-mcp-server) configured and enabled
- A Jira Cloud instance accessible via the MCP server

## Installation

### From this fork's catalog

```bash
specify extension catalog add https://raw.githubusercontent.com/pdfowler/spec-kit-jira/main/catalog.json \
  --name pdfowler-spec-kit-jira \
  --install-allowed

specify extension add jira
```

Or install from the repository directly:

```bash
specify extension add jira --from https://github.com/pdfowler/spec-kit-jira/archive/refs/heads/main.zip
```

### From a local clone

```bash
git clone https://github.com/pdfowler/spec-kit-jira.git
cd /path/to/your-speckit-project
specify extension add --dev /path/to/spec-kit-jira
```

After installation, verify:

```bash
specify extension list

# Should show:
#  ✓ Jira Integration (v1.1.0)
#     Three-tier Jira hierarchy (Epic → Story → Sub-task) from spec tasks, with dry-run preview
#     Commands: 2 | Hooks: 1 | Status: Enabled
```

## Usage

### Dry run — preview without creating tickets

```
/speckit.jira.taskstotickets --dry-run
```

Builds the full plan and saves a `jira-plan.md` preview file. No Jira issues are
created. Useful for reviewing the ticket structure before committing.

You can also pass an existing Epic key:

```
/speckit.jira.taskstotickets PROJ-100 --dry-run
```

### Creating Jira tickets from tasks

```
/speckit.jira.taskstotickets
```

The command will:

1. Ask you to confirm or create the Epic for this spec version
2. Plan Stories (one per `## Phase N:` header) and Sub-tasks (for substantial phases)
3. Show a full preview table with proposed titles, task IDs, and story point estimate
4. Ask: **Proceed? (yes / no / save-plan-only)**
5. On confirmation: create Epic (if new), Stories, Sub-tasks, write `jira-map.md`

You can skip the Epic prompt by providing the key upfront:

```
/speckit.jira.taskstotickets PROJ-100
```

### Implementing by ticket

```
/speckit.jira.implement
```

With no arguments, the command shows a status table and auto-selects the next
incomplete Story. You can also target a specific Story:

```
/speckit.jira.implement PROJ-456
```

As tasks complete, the command:
- Marks each `T00X` as `[x]` in `tasks.md`
- Closes the corresponding Sub-task in Jira (if one was created)
- Closes the Story in Jira when all tasks in the phase are done

### Hook: after_tasks

When enabled, the extension prompts you to create Jira tickets automatically
after `/speckit.tasks` completes.

## Artifacts

### jira-plan.md

Written by `--dry-run` or the `save-plan-only` response to the preview prompt.
Shows the full planned ticket hierarchy with proposed titles. Does **not** contain
real Jira keys — for review only.

### jira-map.md

Written after tickets are successfully created. Maps Story keys to task IDs and
phases; consumed by `/speckit.jira.implement`.

```markdown
# Jira Task Map

**Project**: PROJ | **Epic**: PROJ-100 | **Created**: 2026-05-26

## Story Map

| Story Key | Task IDs | Phase |
|-----------|----------|-------|
| PROJ-456  | T006, T007, T008 | Phase 3: Implement delivery pipeline |
| PROJ-457  | T001, T002       | Phase 1: Set up infrastructure       |

## Sub-task Map

| Sub-task Key | Task ID | Story Key |
|--------------|---------|-----------|
| PROJ-460     | T006    | PROJ-456  |
| PROJ-461     | T007    | PROJ-456  |
```

## License

MIT
