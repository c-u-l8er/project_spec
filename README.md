# Project[&] — Portfolio Project Spec

This folder is the **portfolio-level specification** for Project[&]. It exists to keep shared principles consistent across all domains and codebases.

## Canonical specs you should treat as “source of truth”

- OpenSentience core + marketplace: `opensentience.org/project_spec/agent_marketplace.md`
- Portfolio integration architecture: `opensentience.org/project_spec/portfolio-integration.md`

## Implementation plan (build order)

- `project_spec/PROJECT_PHASES.md` — phased implementation plan in optimal dependency order
- `project_spec/PHASE_ACCEPTANCE.md` — phase-by-phase acceptance checklist

## Components

The portfolio currently includes these system components (each has its own `project_spec/`):

- `opensentience.org` — OpenSentience Core (always-running) + local Admin UI
- `fleetprompt.com` — FleetPrompt (reimplemented as an OpenSentience Agent, with portfolio integration in mind)
- `graphonomous.com` — Graphonomous (Arcana RAG / knowledge graph agent)
- `delegatic.com` — Delegatic (multi-agent “company” orchestration agent)
- `a2atraffic.com` — A2A Traffic (event routing / inter-agent messaging agent)
- `bendscript.com` — BendScript (marketing site now; reserved for future workflow/DSL tooling)

## Shared stance (learned from legacy FleetPrompt)

FleetPrompt legacy contains “hidden gems” worth preserving as portfolio-wide standards:

- **Signals vs Directives**
  - Facts are immutable and durable (**signals**).
  - Side effects are explicit, typed, auditable (**directives**).
- **Idempotency and dedupe** are non-optional for retry safety and replay.
- **No secrets** in durable records (signals/directives/logs).
- **Tool calling must not bypass intent**: mutating tool calls should create directives; a runner performs the mutation.

See the standards in `project_spec/standards/`.
