# spec-kit-jira

A [Spec Kit](https://github.com/github/spec-kit) extension that integrates Jira into the spec-to-implementation workflow. Create Jira tickets from your spec tasks and scope implementation to individual tickets.

## What it does

This extension adds two commands:

| Command | Description |
|---------|-------------|
| `/speckit.jira.taskstotickets` | Converts phases from `tasks.md` into Jira tickets (Tasks or Sub-tasks) and writes a `jira-map.md` mapping file |
| `/speckit.jira.implement` | Runs implementation scoped to a single Jira ticket's tasks, with auto-selection of the next incomplete ticket |

The workflow is:

1. **Generate tasks** with `/speckit.tasks` (core command)
2. **Create Jira tickets** with `/speckit.jira.taskstotickets` — one ticket per phase, optionally as sub-tasks under a parent
3. **Implement by ticket** with `/speckit.jira.implement PROJ-456` — executes only the tasks mapped to that ticket

## Requirements

- [Spec Kit](https://github.com/github/spec-kit) >= 0.1.0
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
#  ✓ Jira Integration (v1.0.0)
#     Create Jira tickets from spec tasks and scope implementation to individual tickets
#     Commands: 2 | Hooks: 1 | Status: Enabled
```

## Usage

### Creating Jira tickets from tasks

```
/speckit.jira.taskstotickets
```

Optionally provide a parent ticket to create sub-tasks:

```
/speckit.jira.taskstotickets PROJ-123
```

This will:
- Group tasks by phase
- Create one Jira ticket per incomplete phase
- Estimate story points for the parent ticket (AI-assisted, accounting for Spec Kit + AI implementation)
- Write a `jira-map.md` file to your feature directory

### Implementing by ticket

```
/speckit.jira.implement
```

With no arguments, the command shows a status table and auto-selects the next incomplete ticket. You can also target a specific ticket:

```
/speckit.jira.implement PROJ-456
```

The command:
- Validates prerequisite phases are complete
- Checks checklists (if any) before proceeding
- Loads full spec context (plan, data model, contracts, etc.)
- Executes only the tasks mapped to the target ticket
- Marks completed tasks in `tasks.md`

### Hook: after_tasks

When enabled, the extension prompts you to create Jira tickets automatically after `/speckit.tasks` completes.

## Artifacts

### jira-map.md

Created by `taskstotickets` in your feature directory. Maps Jira ticket keys to task IDs and phases:

```markdown
# Jira Task Map

**Project**: PROJ | **Parent**: PROJ-123 | **Created**: 2026-02-27

| Jira Key | Task IDs | Phase |
|----------|----------|-------|
| PROJ-456 | T006, T007, T008 | Phase 3: Implement API endpoint |
| PROJ-457 | T009, T010       | Phase 4: Add integration tests  |
```

## License

MIT
