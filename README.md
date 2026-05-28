# spec-kit-jira

A [Spec Kit](https://github.com/github/spec-kit) extension that bridges your spec-driven development workflow with Jira. After you generate tasks with `/speckit.tasks`, this extension creates a three-tier Jira hierarchy, provides a mandatory preview before writing anything, and records implementation progress with a configurable Jira status policy.

## Commands

| Command | Description |
|---------|-------------|
| `/speckit.jira.taskstotickets` | Converts `tasks.md` phases into Jira tickets (Epic → Story → Sub-task) with a dry-run preview gate |
| `/speckit.jira.implement` | Runs implementation scoped to a single Jira phase ticket; records sub-task and phase progress using policy-driven gates |

## How it works

Two task formats are supported:

### Checklist format (`/speckit.tasks` default output)

```
tasks.md phases                  Jira hierarchy
────────────────                 ──────────────────────────────────────────
## Phase 1: Setup          →     (description-only: tasks embedded in ticket)
## Phase 2: Core logic     →     Ticket: Implement core business logic
                                   └── Sub-task: Add input validation
                                   └── Sub-task: Wire service layer
## Phase 3: API layer      →     Ticket: Expose REST endpoints
                                   └── Sub-task: Create route handlers
                                   └── Sub-task: Add request DTOs
## Phase 4: Polish/QA      →     (description-only: tasks embedded in ticket)
```

**Sub-tasks are created selectively.** Phases are description-only when:
- Phase has ≤ 3 incomplete tasks, **or**
- Phase name contains: `setup`, `foundation`, `foundational`, `prereq`,
  `prerequisite`, `infrastructure`, `infra`, `polish`, `cross-cutting`, `cleanup`,
  `qa`, `validation`, `hardening`, `final`

**Ticket titles are clean, action-oriented summaries** — no "Phase N:" prefix.

### Story-card format

```
tasks.md story cards                        Jira hierarchy
───────────────────────────────             ────────────────────────────────────
### AUTH-001: Auth middleware setup    →    Ticket: Auth middleware setup
**Acceptance Criteria**:                     └── Sub-task: Token validation passes
- [ ] Token validation passes                └── Sub-task: 401 for expired tokens
- [ ] 401 returned for expired tokens        └── Sub-task: Middleware in NestJS
- [ ] Middleware integrated                **Dependencies** → Jira issue links
**Dependencies**: AUTH-000
```

Sub-tasks are created from acceptance criteria. Dependencies become Jira issue links.

## Requirements

- [Spec Kit](https://github.com/github/spec-kit) >= 0.8.3
- An [Atlassian MCP server](https://marketplace.atlassian.com/apps/1234567/atlassian-remote-mcp-server)
  configured in your AI agent (Cursor, Claude Code, etc.)
- A Jira Cloud instance accessible via the MCP server

### Setting up the Atlassian MCP

The extension uses the Atlassian Remote MCP Server. To configure it:

1. Visit [id.atlassian.com](https://id.atlassian.com) → **Security** → **API tokens**
   (or use OAuth — recommended for broader tool access)
2. In your AI agent, add the MCP server and authenticate
3. Verify the agent can see your Jira projects before installing this extension

> **Note:** OAuth authentication gives access to the full set of Jira MCP tools.
> API token scopes can be narrower — if you see only a limited tool set, switch to OAuth.

## Installation

### Via catalog (recommended)

```bash
# Add the catalog
specify extension catalog add https://raw.githubusercontent.com/pdfowler/spec-kit-jira/main/catalog.json \
  --name pdfowler-jira

# Install the extension
specify extension add jira
```

### Directly from repository

```bash
specify extension add jira \
  --from https://github.com/pdfowler/spec-kit-jira/archive/refs/heads/main.zip
```

### From a local clone

```bash
git clone https://github.com/pdfowler/spec-kit-jira.git
cd /path/to/your-speckit-project
specify extension add --dev /path/to/spec-kit-jira
```

### Verify installation

```bash
specify extension list
# ✓ Jira Integration (v1.4.0)
#     Three-tier Jira hierarchy (Epic → Story → Sub-task) with dry-run preview
#     Commands: 2 | Hooks: 1 | Status: Enabled
```

### Namespace customization

By default the commands are registered as `speckit.jira.*`. If your project uses a
custom command prefix (e.g. `sdd`), add a `.specify/config.yml` at your project root:

```yaml
namespace:
  command_prefix: sdd   # commands become /sdd.jira.taskstotickets, /sdd.jira.implement
```

Then re-run your init script (or `specify integration install`) to apply the rename.

## Usage

### 1. Create tickets from tasks

```
/speckit.jira.taskstotickets
```

The command walks you through:

1. **Parent ticket** — provide an Epic/Story key to nest tickets under it, or `none`
   for standalone tickets (can also be passed as a direct argument)
2. **Plan** — detects format (checklist or story-card), builds the full ticket hierarchy,
   estimates story points
3. **Preview** — shows everything that will be created before any Jira writes

   ```
   Epic: ENG-100 "Two-Way SMS v1" (existing)
   Estimated story points: 5  (3 phases, 17 tasks, 1 external integration)

   | Level       | Title                               | Phase   | Task IDs  |
   |-------------|-------------------------------------|---------|-----------|
   | Story       | Bootstrap project dependencies      | Phase 1 | T001–T002 |
   | (desc-only) | ← ≤3 tasks, tasks in description    | Phase 2 | T003–T005 |
   | Story       | Implement SMS delivery pipeline     | Phase 3 | T006–T014 |
   | Sub-task    |   ↳ Implement adapter send method   | Phase 3 | T006      |
   | Sub-task    |   ↳ Add retry with backoff          | Phase 3 | T007      |
   | Story       | Add delivery audit trail            | Phase 4 | T015–T019 |

   → Proceed? (yes / no / save-plan-only)
   ```

4. **Create** — on `yes`, creates Epic (if new), Stories, and Sub-tasks; writes `jira-map.md`

Pass an Epic key to skip the prompt:

```
/speckit.jira.taskstotickets ENG-100
```

### 2. Dry run — preview without creating

```
/speckit.jira.taskstotickets --dry-run
/speckit.jira.taskstotickets ENG-100 --dry-run
```

Saves the preview as `jira-plan.md` in your feature directory. No Jira issues are
created and no `jira-map.md` is written.

### 3. Implement by ticket

```
/speckit.jira.implement
```

Shows a status table and auto-selects the next incomplete Story. Provide a key to
target a specific Story:

```
/speckit.jira.implement ENG-456
```

Implementation progress is controlled by `jira-progress-policy.yml` rather than a
hardcoded "done when checked off" rule. By default:

- Starting a phase or task moves the mapped Jira issue to `In Progress` when possible.
- A local commit moves a mapped sub-task to `In Review`.
- Passing local quality gates can move a mapped sub-task to `Done`.
- A non-draft PR moves the phase ticket to `In Review`.
- A merged PR moves the phase ticket to `Done`.
- Done tickets are never moved backward by default.

```
✓ T006 committed — ENG-460 moved to In Review (3/9 tasks in ENG-456)
✓ T006 validated — ENG-460 moved to Done
...
✓ PR merged — ENG-456 moved to Done.
```

### Progress policy

The extension ships with a default policy at:

```text
config/jira-progress-policy.yml
```

Projects can copy it to:

```text
.specify/jira-progress-policy.yml
```

and customize status names, gates, and transition behavior without changing the
extension. The implement command loads the project override first, then falls back
to the extension default.

Key policy knobs include:

- Semantic Jira statuses: `todo`, `in_progress`, `in_review`, `done`
- Monotonic transitions: prevent `Done` from moving backward
- Sub-task done gate: `committed`, `local_gates_passed`, `remote_checks_passed`, or `explicit_confirm`
- Phase done gate: `pr_merged`, `remote_checks_passed`, `local_gates_passed`, or `explicit_confirm`
- Draft PR behavior: whether a draft PR counts as review
- Test-task behavior: whether red-phase tests move to review or done
- Local gate commands: project-specific test/lint/typecheck/build commands

### Hook: after_tasks

The extension registers an optional `after_tasks` hook. When enabled, your agent
will prompt you to create Jira tickets immediately after `/speckit.tasks` completes —
no separate command invocation needed.

```yaml
# .specify/extensions.yml (auto-generated)
hooks:
  after_tasks:
  - extension: jira
    command: speckit.jira.taskstotickets
    enabled: true
    optional: true
    prompt: "Create Jira tickets for the generated tasks?"
```

## Artifacts

### `jira-plan.md`

Written by `--dry-run` or the `save-plan-only` response. Human-readable preview of
the planned ticket hierarchy — safe to share or commit for review before creating
anything in Jira.

### `jira-map.md`

Written after tickets are created. Maps phase ticket keys, sub-task keys, and task
IDs; consumed by `/speckit.jira.implement` to scope implementation to individual
tickets.

```markdown
# Jira Task Map

**Project**: ENG | **Parent**: ENG-100 | **Created**: 2026-05-26

| Phase Ticket | Sub-task | Task ID | Description |
|--------------|----------|---------|-------------|
| ENG-456 Implement SMS delivery pipeline | ENG-460 | T006 | Create adapter |
| ENG-456 Implement SMS delivery pipeline | ENG-461 | T007 | Add retry logic |
| ENG-457 Bootstrap project dependencies | — | T001-T002 | (description-only) |
```

## Changelog

See [CHANGELOG.md](./CHANGELOG.md).

## License

MIT — [Patrick Fowler](https://github.com/pdfowler)
