# Project[&] — Portfolio Implementation Phases (Optimal Dependency Order)

This document is the **phased implementation plan** for building the Project[&] portfolio in an optimal dependency order, minimizing rework and maximizing early end-to-end validation.

It assumes the following portfolio-wide decisions are locked:
- **Canonical tool identifiers are namespaced** as `<agent_id>/<tool_name>`.
- **Pub/sub permissions are bus-agnostic** using `event:publish:<pattern>` / `event:subscribe:<pattern>`.
- **Repo-first resources**: `.fleetprompt/` in a repo is the source of truth; Core may keep derived caches under `~/.opensentience/`.
- **Runtime protocol v1** is concrete and uses **UDS + length-prefixed JSON frames**:
  - `opensentience.org/project_spec/RUNTIME_PROTOCOL.md`

Canonical specs to keep open while implementing:
- `opensentience.org/project_spec/agent_marketplace.md`
- `opensentience.org/project_spec/portfolio-integration.md`
- `project_spec/standards/*`
- Component specs:
  - FleetPrompt: `fleetprompt.com/project_spec/*`
  - Graphonomous: `graphonomous.com/project_spec/*`
  - A2A Traffic: `a2atraffic.com/project_spec/*`
  - Delegatic: `delegatic.com/project_spec/*`

---

## Guiding principles for build order

1. **Build the governance plane first**
   - OpenSentience Core (catalog, enablement, permissions, audit log, protocol, tool routing) is the dependency root.

2. **Prove the protocol with a minimal agent before building real agents**
   - A tiny “hello tools” agent validates the runtime protocol, tool routing, streaming, and cancellation without large domain complexity.

3. **Add one “vertical slice” at a time**
   - Each major component should be integrated into Core UI/CLI and audit timeline as it lands.

4. **Keep side effects behind explicit intent boundaries**
   - Mutating operations should cross directive boundaries (either Core-wrapped or agent-requested), and must be auditable.

5. **Never block progress on perfect UI**
   - Prefer CLI + minimal admin UI first; incrementally upgrade to richer views.

---

## Phase 0 — Portfolio scaffolding and invariants (Week 0)

### Goals
- Freeze cross-portfolio standards that reduce churn.
- Ensure all specs are internally consistent enough to implement.

### Deliverables
- Confirmed standard taxonomies:
  - `event:*` pub/sub permissions (bus-agnostic)
  - `network:egress:<host-or-tag>` (consistent with standards)
  - namespaced tool IDs
- Runtime protocol v1 is documented (already done): `opensentience.org/project_spec/RUNTIME_PROTOCOL.md`
- A “definition of done” checklist template for each agent:
  - manifest, permissions, tools, signals/directives, audit, tests

### Exit criteria
- You can hand the specs to an engineer and they can implement Phase 1 without additional design meetings.

---

## Phase 1 — OpenSentience Core MVP: catalog + enablement + audit + launcher (Weeks 1–2)

### Dependencies
- Portfolio standards + runtime protocol v1.

### Goals
- Core can discover agents, install/build (explicit trust boundary), enable (approve permissions), run (separate process), and record an audit log.

### Scope
1. **Catalog & discovery**
   - Scan configured roots for `opensentience.agent.json`
   - Store catalog records (SQLite recommended per spec)
2. **Install/build workflow**
   - `install`: clone/fetch agent source
   - `build`: explicit compile step (logged as trust boundary)
3. **Enablement**
   - Persist requested vs approved permissions
   - Deny-by-default and explicit approval UX (CLI first)
4. **Launcher**
   - Start agent processes with controlled env and session token
   - Capture stdout/stderr logs
5. **Audit log**
   - Durable, queryable record of security-relevant actions:
     - install/build/enable/run/stop
     - tool calls (with redaction best-effort)
6. **Admin UI skeleton (optional but recommended even in MVP)**
   - Localhost-only, token-protected, CSRF enabled
   - Agents list + agent detail (read-only is fine for Week 2)

### Exit criteria
- From CLI and/or UI, you can:
  - discover an agent manifest
  - install + build it (explicit user action)
  - enable it (approve permissions)
  - run it and see logs
  - see audit entries for each action

---

## Phase 2 — Runtime protocol implementation + ToolRouter MVP (Weeks 2–3)

### Dependencies
- Phase 1 launcher and audit log.

### Goals
- Implement the concrete v1 runtime protocol end-to-end:
  - handshake, tool registration, tool calls, streaming, cancellation, heartbeats

### Scope
1. **Protocol server in Core**
   - UDS listener
   - Length-prefixed JSON framing
   - Validate message envelope fields
2. **Session authentication**
   - Session token generation on launch
   - Validate `agent.hello` token; close on failure
3. **Tool registration**
   - Accept `agent.tools.register`
   - Build a routing table keyed by canonical `tool_id`
4. **Tool invocation**
   - `core.tool.call` → `agent.tool.result`
   - Optional `agent.tool.stream` support
5. **Cancellation**
   - `core.tool.cancel` propagation
6. **Heartbeats/health**
   - Mark agents unhealthy on missed heartbeats
7. **Audit integration**
   - Every tool call routed is audited (inputs/outputs redacted best-effort, secret-free persistence enforced)

### Exit criteria
- You can run a minimal test agent and:
  - register at least one tool
  - invoke it successfully via Core
  - observe streaming output (if supported)
  - cancel a long-running tool call
  - see health status and audit entries

---

## Phase 3 — Agent SDK + Example Agents (Week 3)

### Dependencies
- Phase 2 protocol implementation.

### Goals
- Make it easy to implement agents consistently and safely.

### Scope
1. **Agent SDK (Elixir)**
   - Client for protocol framing/envelope
   - Behaviours for:
     - tool definition registration
     - tool call dispatch
     - streaming helpers
     - cancellation hooks
     - heartbeat loop
2. **Example agents**
   - `hello_world_agent`
     - one pure function tool
     - one long-running tool demonstrating cancellation + streaming
   - `echo_schema_agent`
     - registers tools with input schema validation examples

### Exit criteria
- Creating a new agent takes minutes (manifest + SDK + tool module).
- Example agents serve as integration tests for protocol evolution.

---

## Phase 4 — FleetPrompt MVP Agent (Weeks 4–6)

### Dependencies
- Phases 1–3.

### Goals
- FleetPrompt can index `.fleetprompt/` resources (repo-local), validate them, and run skills/workflows with auditable execution records.

### Scope (MVP order)
1. **Resource validation**
   - Implement `com.fleetprompt.core/fp_validate_project_resources`
   - Validate `.fleetprompt/config.toml` and referenced files
2. **Skill discovery**
   - `fp_list_skills`, `fp_describe_skill`
3. **Skill execution**
   - `fp_run_skill` with:
     - input validation
     - idempotency key support
     - structured logs (secret-free)
     - cancellation (best-effort)
4. **Workflow discovery/execution**
   - `fp_list_workflows`, `fp_describe_workflow`, `fp_run_workflow`
5. **Audit integration**
   - Emit execution lifecycle signals
   - Ensure side-effectful steps are directive-backed (Core-wrapped or agent-requested; choose one consistent mechanism)

### Exit criteria
- From Core UI/CLI you can:
  - validate a repo’s `.fleetprompt/`
  - run a skill and see execution records and signals in the audit timeline
  - cancel a run and see terminal state correctly recorded

---

## Phase 5 — Graphonomous MVP Agent (Weeks 6–8)

### Dependencies
- Core ToolRouter + FleetPrompt optional (Graphonomous can ship independently).

### Goals
- Provide graph-native RAG search + ingestion with provenance/citations, permissioned by collection scope.

### Scope (MVP order)
1. **Repo-local config indexing**
   - `.fleetprompt/graphonomous/collections.json` validation (Core indexing + runtime defense-in-depth)
2. **Collections**
   - list collections visible to caller (permission-filtered)
3. **Search**
   - `com.graphonomous.core/graph_search` across specified collections
   - Return citations/provenance objects
4. **Ingestion**
   - `com.graphonomous.core/graph_ingest`
   - Enforce directive boundary for writes in chat/tool-calling contexts
   - Idempotency/dedupe (dedupe_key or computed fingerprint)
5. **Audit**
   - ingest accepted/succeeded/failed signals
   - query performed signal (without persisting raw query text by default)

### Exit criteria
- An agent can ingest a document (directive-backed if needed) and later retrieve it with citations.
- Unauthorized collections are inaccessible (no leakage in errors or listings).

---

## Phase 6 — A2A Traffic MVP Agent (Weeks 8–9)

### Dependencies
- Core protocol + audit + permissions enforcement.

### Goals
- Provide an auditable, permissioned event bus with at-least-once delivery semantics and dedupe.

### Scope (MVP order)
1. **Subscriptions**
   - `com.a2atraffic.core/a2a_subscribe`, `a2a_list_subscriptions`, `a2a_unsubscribe`
2. **Publish**
   - `com.a2atraffic.core/a2a_publish` with dedupe_key handling
3. **Delivery semantics**
   - at-least-once delivery attempt tracking
   - bounded retries + DLQ-like terminal state visibility
4. **Repo-local resource surfaces**
   - `.fleetprompt/a2a/subscriptions.json` and `publications.json` indexing/validation
5. **Audit**
   - publish accepted, delivery attempted, subscription created/removed signals

### Exit criteria
- Two agents can coordinate via events with:
  - enforced `event:*` permissions
  - visible publish + delivery attempts in the audit timeline
  - dedupe preventing fanout duplication

---

## Phase 7 — Delegatic MVP Agent (Weeks 9–11)

### Dependencies
- Core + FleetPrompt + (optional) Graphonomous + (optional) A2A.

### Goals
- Provide safe multi-agent “company” orchestration with strict policy gating.

### Scope (MVP order)
1. **Company config**
   - `.fleetprompt/delegatic/company.json` validation and description
2. **Missions**
   - start mission with idempotency_key
   - mission state machine (created/planning/running/waiting/terminal)
3. **Policy enforcement**
   - role-based tool allowlists and directive allowlists
   - resource scope constraints (graph collections, filesystem, event patterns)
4. **Orchestration template**
   - Keep MVP plan simple:
     - run a FleetPrompt workflow
     - optionally read from Graphonomous
     - optionally publish a completion event
5. **Cancellation**
   - cancel mission with best-effort downstream cancellation
6. **Audit**
   - mission and step lifecycle signals with correlation/causation linking

### Exit criteria
- You can define a company in repo config and run a mission that:
  - respects role policies (deny-by-default)
  - produces an auditable timeline
  - is cancelable and idempotent

---

## Phase 8 — Full portfolio demo slice + “unified timeline” UI (Weeks 11–12)

### Dependencies
- Phases 1–7.

### Goals
- Prove end-to-end integration in one realistic scenario and make it inspectable in Core UI.

### Demo scenario (recommended)
A “company” runs a mission that:
1. Uses Graphonomous search for context
2. Runs a FleetPrompt workflow that produces an artifact
3. Publishes an event indicating completion
4. A subscriber (another agent or workflow) reacts

### Core UI deliverables (minimum)
- Agents list + detail including:
  - requested vs approved permissions
  - running/healthy status
  - registered tools (namespaced)
- Unified timeline view:
  - tool calls
  - signals
  - directives (if modeled explicitly)
  - executions (FleetPrompt, Delegatic)
- Basic search/filter:
  - by `correlation_id` (chat session, mission id)
  - by agent id/tool id

### Exit criteria
- You can click through a single correlation_id and understand:
  - what was requested (intent)
  - what happened (facts)
  - who/what executed it
  - what failed and why (secret-free)

---

## Phase 9 — Hardening, reliability, and polish (Weeks 12–16, ongoing)

### Goals
- Make the system safe and robust under adversarial inputs and operational failures.

### Workstreams
1. **Security hardening**
   - stricter redaction and “no secrets” enforcement
   - deny-by-default audits for questionable payloads
   - improved localhost UI protections
2. **Reliability**
   - restart policies, crash recovery, replay-safe idempotency
   - backpressure and queueing (A2A, ingestion)
3. **Performance**
   - connection pooling and tool call latency improvements
   - indexing performance for `.fleetprompt/` resources
4. **Protocol evolution**
   - schema validation strictness policy
   - message size negotiation
   - optional compression (only if safe and observable)
5. **Documentation**
   - “How to build an agent” guide with SDK examples
   - “How to write `.fleetprompt/` resources” examples
   - threat model + security posture summary

### Exit criteria
- You trust it enough to run continuously on your machine without surprises:
  - stable uptime
  - clear auditability
  - safe defaults
  - predictable performance

---

## Recommended parallelization (if you want to move faster)

If you have time to do workstreams in parallel (even as a solo dev, alternating):
- While implementing Core protocol (Phase 2), start Agent SDK skeleton (Phase 3).
- While implementing FleetPrompt (Phase 4), start Graphonomous config surface parsing (Phase 5).
- While implementing A2A (Phase 6), finalize Delegatic company schema and validation UX (Phase 7).
- UI work can be incremental; prioritize “unified timeline” primitives early.

---

## “Are we ready to move on?”
Yes: the project_spec set is now sufficient to proceed to implementation phasing because:
- Protocol envelope/framing is concrete.
- Tool naming and permission taxonomy are consistent.
- Major agents now have: architecture, interfaces, execution model, resource surfaces, permissions, security, and tests.
- Remaining open questions are mostly tuning/implementation choices, not blockers for Phase 1–3.

If you want, the next planning artifact after this file is a **Phase-by-phase acceptance checklist** that maps directly to:
- CLI commands/screens
- audit events
- integration tests
- security invariants