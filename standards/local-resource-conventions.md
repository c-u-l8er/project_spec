# Local Resource Conventions (Host VM)

These conventions come from `opensentience.org/project_spec/portfolio-integration.md` and are treated as shared portfolio standards.

## Directories

- `~/.opensentience/`
  - OpenSentience Core runtime state
  - Agent installs/clones/build artifacts
  - Catalog database
  - Audit log

- `~/.fleetprompt/`
  - Optional/legacy location (avoid depending on it for new work)
  - Not the source of truth for project resources; use repo `.fleetprompt/` instead

- `~/Projects/*/`
  - Developer projects and agent source repos

## Per-project resources: `.fleetprompt/`

**Source of truth:** the repo-local `.fleetprompt/` directory is authoritative and should be committed with the project.

**Derived state (optional):** OpenSentience Core may keep indexed/cached/normalized copies of these resources under `~/.opensentience/` for performance and auditing, but those caches must be treated as derived (rebuildable) data.

A project may include a `.fleetprompt/` folder containing:

- `config.toml` — project-level registry of skills/workflows/integrations
- `skills/` — reusable skill scripts/modules
- `workflows/` — workflow definitions (pipeline-like)
- `graphonomous/` — knowledge collections + schemas
- `delegatic/` — “company” configuration (teams, roles, policies)
- `a2a/` — subscriptions/publishing configuration

### Rule: safe indexing

OpenSentience Core must be able to **index** `.fleetprompt/` and `opensentience.agent.json` without executing any agent code.
