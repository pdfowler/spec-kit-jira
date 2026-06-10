# spec-kit-jira

A [Spec Kit](https://github.com/github/spec-kit) extension that bridges your spec-driven development workflow with Jira. After you generate tasks with `/speckit.tasks`, this extension plans and applies Jira tickets, persists a reviewable `jira-map.md`, links phase dependencies with Jira `Blocks`, and records implementation progress with configurable gates.

## Commands

| Command | Description |
|---------|-------------|
| `/speckit.jira.taskstotickets` | `plan` or `apply` subcommands; flat `jira-map.md` |
| `/speckit.jira.plan-tickets` | Shorthand for `taskstotickets plan` |
| `/speckit.jira.apply-tickets` | Shorthand for `taskstotickets apply` |
| `/speckit.jira.implement` | Runs implementation scoped to a phase ticket; policy-driven Jira progress |

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

## Dependencies
- Phase 3 depends on Phase 2       →     Phase 2 blocks Phase 3
```

**Sub-tasks are created selectively.** Phases are description-only when:
- Phase has ≤ 3 incomplete tasks, **or**
- Phase label exactly matches setup/polish-style labels such as `setup`, `foundation`,
  `infrastructure`, `polish`, `cross-cutting`, `cleanup`, `qa`, or `final`

Phases with 4+ incomplete tasks always get sub-tasks. The matcher does not substring
match deliverable words such as "hardening" inside longer titles.

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

- [Spec Kit](https://github.com/github/spec-kit) >= 0.9.2
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
# ✓ Jira Integration (v1.5.0)
#     Jira ticket planning with phase dependency links and configurable gates
#     Commands: 4 | Hooks: 1 | Status: Enabled
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

### 1. Plan and apply (Terraform-style)

**Plan** — write or update `jira-map.md` with `TBD` keys (no Jira creates):

```
/speckit.jira.taskstotickets plan ENG-6867
/speckit.jira.taskstotickets ENG-6867              # plan is the default
```

Review and edit the flat **Map** table (and **Preview** / **Links** sections), then apply.

**Apply** — create Jira issues for `TBD` rows only (requires an existing plan):

```
/speckit.jira.taskstotickets apply ENG-6867
```

Apply **fails** if `jira-map.md` is missing, uses the legacy Story/Sub-task map format,
or has no `TBD` rows. After amending `tasks.md`, run **plan** again (delta), then **apply**.

Optional: `--regenerate` or `--fresh` on **plan** to skip “use existing map?” prompts.

**Legacy map** (Story Map / Sub-task Map): run **plan** once — migrates to flat **Map**
(backup at `jira-map.legacy.md`), then **apply**.

> **v1.4.2:** Prefer `plan` / `apply`. Deprecated flags `--dry-run`, `-n`, `--apply-plan`
> still work with a warning.

### 2. Implement by ticket

```
/speckit.jira.implement
```

Shows a status table and auto-selects the next incomplete Story. Provide a key to
target a specific Story:

```
/speckit.jira.implement ENG-456
```

Implementation progress is controlled by `jira-config.yml` rather than a hardcoded
"done when checked off" rule. By default:

- Starting a phase or task moves the mapped Jira issue to `In Progress` when possible.
- A local commit moves a mapped sub-task to `In Review`.
- Passing local quality gates can move a mapped sub-task to `Done`.
- A non-draft PR moves the phase ticket to `In Review`.
- A merged PR moves the phase ticket to `Done`.
- A phase blocked by another phase may start once the blocker reaches `In Review`;
  the blocker still moves to `Done` only when its done gate is satisfied.
- Done tickets are never moved backward by default.

```
✓ T006 committed — ENG-460 moved to In Review (3/9 tasks in ENG-456)
✓ T006 validated — ENG-460 moved to Done
...
✓ PR merged — ENG-456 moved to Done.
```

### Configuration

The extension ships with defaults in `extension.yml` under `config.defaults` and a
reference template at:

```text
config/jira-config.template.yml
```

Projects can copy it to:

```text
.specify/extensions/jira/jira-config.yml
```

and customize status names, dependency gates, and transition behavior without changing
the extension.

Configuration is deep-merged in this order:

1. Extension defaults: `.specify/extensions/jira/extension.yml` → `config.defaults`
2. Optional user-global override: `$HOME/.specify/extensions/jira/jira-config.yml`
3. Repo/team config: `.specify/extensions/jira/jira-config.yml`
4. Repo-local override: `.specify/extensions/jira/local-config.yml`
5. Environment variables: `SPECKIT_JIRA_*`

The user-global layer is extension-specific; Spec Kit core does not manage it. Add
`.specify/extensions/jira/local-config.yml` to your repo's `.gitignore` if your
project does not already ignore local extension config.

Legacy `.specify/jira-progress-policy.yml` files are still read for one compatibility
window and merged under `jira.progress_policy` with a deprecation warning.

Key policy knobs include:

- Semantic Jira statuses: `todo`, `in_progress`, `in_review`, `done`
- Monotonic transitions: prevent `Done` from moving backward
- Sub-task done gate: `committed`, `local_gates_passed`, `remote_checks_passed`, or `explicit_confirm`
- Phase done gate: `pr_merged`, `remote_checks_passed`, `local_gates_passed`, or `explicit_confirm`
- Phase dependency start gate: `semantic_status_at_least`, `tasks_complete`, `done`, or `none`
- Draft PR behavior: whether a draft PR counts as review
- Test-task behavior: whether red-phase tests move to review or done
- Local gate commands: project-specific test/lint/typecheck/build commands

For stacked PR workflows, the default `jira.phase_dependencies.start_when_blocker`
allows dependent implementation to start when the blocker is `in_review`, while merge
ordering remains enforced by the stack and GitHub.

### Hook: after_tasks

The extension registers an optional `after_tasks` hook. When enabled, your agent
will prompt you to plan Jira tickets immediately after `/speckit.tasks` completes.
Apply remains a separate explicit step.

```yaml
# .specify/extensions.yml (auto-generated)
hooks:
  after_tasks:
  - extension: jira
    command: speckit.jira.plan-tickets
    enabled: true
    optional: true
    prompt: "Plan Jira tickets from tasks?"
```

## Artifacts

### `jira-map.md`

Single artifact for dry-run and live-run. Consumed by `/speckit.jira.implement`.

| `**Status**` | Meaning |
|---------------|---------|
| `draft` | Full preview — all keys `TBD`, nothing created yet |
| `created` | At least one real key; may include pending `TBD` rows after a spec amend |
| `created` (no `TBD`) | Fully applied — safe for `/speckit.jira.implement` |

Sections:

- **Preview** (draft) — ASCII tree from the preview gate; optional after create
- **Map** — flat table of phase tickets, sub-tasks, and task IDs (implementation scope)
- **Review Unit** column in **Map** — optional sync/review metadata for a PR, repo,
  branch, or other review boundary; defaults to `—` and is preserved by plan/apply
- **Phase Dependencies** — phase-level `Blocks` relationships parsed from `tasks.md`
- **Links** — clickable Jira URLs in an appendix (navigation only; not parsed by implement)

```markdown
# Jira Task Map

**Status**: created
**Project**: ENG | **Parent**: ENG-100 | **Format**: checklist | **Created**: 2026-05-26

## Map

| Phase Ticket | Sub-task | Task ID | Description | Review Unit |
|--------------|----------|---------|-------------|-------------|
| ENG-456 Implement SMS delivery pipeline | ENG-460 | T006 | Create adapter | monorepo#123 |

## Phase Dependencies

| Blocker | Blocked | Status | Source |
|---------|---------|--------|--------|
| ENG-455 Bootstrap project dependencies | ENG-456 Implement SMS delivery pipeline | linked | Phase 3 depends on Phase 2 |

## Links

| Key | Title | Link |
|-----|-------|------|
| ENG-100 | Two-Way SMS | [ENG-100](https://your-site.atlassian.net/browse/ENG-100) |
| ENG-456 | Implement SMS delivery pipeline | [ENG-456](https://your-site.atlassian.net/browse/ENG-456) |
```

Legacy `jira-plan.md` from older extension versions is deprecated; use draft `jira-map.md` instead.

## Changelog

See [CHANGELOG.md](./CHANGELOG.md).

## License

MIT — [Patrick Fowler](https://github.com/pdfowler)
