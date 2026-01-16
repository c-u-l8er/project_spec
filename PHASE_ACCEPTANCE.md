# Project[&] — Phase Acceptance Checklist
## CLI/UI checkpoints, audit events, protocol tests, and security invariants (by phase)

This document is the phase-by-phase **acceptance checklist** for building the Project[&] portfolio in a safe, optimal dependency order.

It is meant to be used as:
- a “Definition of Done” for each phase
- a test planning guide (integration + regression)
- a security gate checklist
- a UI/CLI feature checklist

It assumes portfolio decisions are locked:
- Tool identifiers are namespaced as `<agent_id>/<tool_name>`
- Pub/sub permissions are bus-agnostic: `event:publish:<pattern>`, `event:subscribe:<pattern>`
- Repo-first `.fleetprompt/` is source of truth; Core may cache derived data under `~/.opensentience/`
- Runtime protocol v1 is concrete (UDS + length-prefixed JSON framing):
  - `opensentience.org/project_spec/RUNTIME_PROTOCOL.md`

---

## Cross-phase invariants (must hold in every phase)

### Security invariants (non-negotiable)
- **Localhost-only admin UI** by default (bind to `127.0.0.1`).
- **State-changing actions require an admin token** and are **CSRF-protected**.
- **No permissive CORS**; deny framing (`frame-ancestors 'none'` / `X-Frame-Options: DENY`).
- **No secrets in durable artifacts**:
  - audit log entries
  - signals/directives
  - persisted tool inputs/outputs and logs
- **Deny-by-default**:
  - agents not enabled cannot run
  - permissions not explicitly approved cannot be used
- **Side effects must cross explicit intent boundaries** (directive boundary; Core-wrapped or agent-requested).
- **Idempotency/dedupe is treated as a safety feature** for any operation that can be retried or replayed.

### Observability and operability invariants
- Every phase must add or preserve:
  - audit entries for privileged actions
  - safe structured errors (no raw secrets, minimal leakage)
  - “what happened” timeline linkage via `correlation_id` where applicable

### Protocol invariants
- Protocol framing and message envelope conform to v1:
  - length-prefixed UTF-8 JSON messages
  - `v`, `type`, `id`, `ts`, `payload`
  - correlation fields are echoed by agents throughout a request (`request_id`, `correlation_id`, `causation_id`)

---

## Common acceptance artifacts (what to capture each phase)

For each phase completion, capture:
1. Screenshots (or notes) of UI state (if UI exists)
2. CLI transcript of key commands
3. A “golden audit log excerpt” showing expected events
4. A short “threat regression checklist” (the security invariants above)
5. A minimal integration test report (even if manual)

---

## Phase 0 — Portfolio scaffolding and invariants (Week 0)

### Acceptance checklist
- [ ] Tool naming is standardized as `<agent_id>/<tool_name>` across specs and examples.
- [ ] Permission taxonomy is consistent across specs:
  - [ ] event bus permissions use `event:*`
  - [ ] network uses `network:egress:<host-or-tag>`
- [ ] Runtime protocol is documented and concrete:
  - [ ] `opensentience.org/project_spec/RUNTIME_PROTOCOL.md` exists and is treated as MVP baseline.
- [ ] A single source-of-truth exists for “repo-first `.fleetprompt/`” convention.
- [ ] Specs for FleetPrompt, Graphonomous, A2A Traffic, Delegatic include:
  - architecture, interfaces, execution model, resource surfaces, permissions, security, test plan

### Exit criteria
- [ ] An engineer can start Phase 1 without asking “what does the protocol look like?” or “what are tool ids?” or “what are event permissions?”

---

## Phase 1 — OpenSentience Core MVP: catalog + enablement + audit + launcher (Weeks 1–2)

### CLI acceptance (minimum viable commands)
- [ ] List discovered agents (local scan roots).
- [ ] Show agent detail (manifest display).
- [ ] Install (clone/fetch) an agent repo into `~/.opensentience/`.
- [ ] Build/compile (explicit trust boundary; separately invoked from install).
- [ ] Enable agent (approve requested permissions; store approved subset).
- [ ] Run agent (separate OS process).
- [ ] Stop agent.

*(Command names can vary; behavior is what matters.)*

### UI acceptance (if UI is included in Phase 1)
- [ ] UI binds to `127.0.0.1` by default.
- [ ] UI shows agents list + agent detail (read-only is acceptable).
- [ ] UI displays requested vs approved permissions.

### Audit acceptance
For each action below, a durable audit entry exists (secret-free):
- [ ] discovery/index update (manifest found/updated)
- [ ] install (source + ref)
- [ ] build/compile (explicit trust boundary)
- [ ] enable (permission approvals recorded)
- [ ] run (command + timestamps)
- [ ] stop (timestamp, reason if any)

Audit entries must include (where possible):
- [ ] actor identity (human/system)
- [ ] agent_id
- [ ] correlation_id when triggered by UI session

### Security acceptance
- [ ] Discovery/indexing does not execute agent code.
- [ ] Install/build is explicit and auditable.
- [ ] Enablement requires explicit approval; deny-by-default.
- [ ] No secrets are persisted in audit entries.
- [ ] UI state-changing actions require token + CSRF.

### Exit criteria
- [ ] Core can manage an agent lifecycle end-to-end and prove it via audit log.

---

## Phase 2 — Runtime protocol implementation + ToolRouter MVP (Weeks 2–3)

### Protocol acceptance (MVP messages)
Using a minimal test agent:
- [ ] Agent connects and sends `agent.hello` with session token.
- [ ] Core responds with `core.welcome` (negotiated version, session_id).
- [ ] Agent registers tools via `agent.tools.register`.
- [ ] Core acknowledges via `core.tools.registered`.
- [ ] Core invokes a tool via `core.tool.call`.
- [ ] Agent returns final result via `agent.tool.result`.

### Streaming acceptance (optional but recommended)
- [ ] Agent can send at least one `agent.tool.stream` message during a call.
- [ ] UI/CLI can display streaming output progressively.
- [ ] Streaming payloads remain secret-free and bounded.

### Cancellation acceptance (optional but recommended)
- [ ] Core can send `core.tool.cancel`.
- [ ] Agent acknowledges (optional) and produces a terminal `agent.tool.result` with status `canceled`.

### Health acceptance
- [ ] Agent sends `agent.heartbeat` at the negotiated cadence.
- [ ] Core marks agent unhealthy on missed heartbeats and stops routing tool calls.

### Tool routing acceptance
- [ ] Tools are registered and addressed by canonical namespaced tool ids.
- [ ] Tool calls are correlated:
  - [ ] Core sets `request_id` (and preferably `correlation_id`).
  - [ ] Agent echoes correlation fields back in stream/result messages.

### Audit acceptance
- [ ] Tool call audit record exists for each invocation:
  - tool_id, caller, outcome, durations
  - redaction best-effort applied to inputs/outputs
- [ ] Protocol errors are audited as security-relevant when appropriate (without logging raw payloads).

### Security acceptance
- [ ] Session token is never logged.
- [ ] Unauthorized handshake is rejected and connection closed.
- [ ] Oversized frames are rejected safely and audited without payload content.

### Exit criteria
- [ ] Core can route tool calls reliably with streaming/cancellation/health and produce an auditable record.

---

## Phase 3 — Agent SDK + Example Agents (Week 3)

### SDK acceptance
- [ ] SDK can:
  - connect using the runtime protocol framing/envelope
  - register tools
  - dispatch tool calls to handlers
  - emit streaming chunks
  - handle cancellation hooks
  - send heartbeats
- [ ] SDK makes it difficult to violate invariants:
  - provides helpers for safe structured errors
  - provides best-effort redaction utilities (optional but recommended)
  - encourages namespaced tool IDs

### Example agent acceptance
Implement at least two minimal agents:
1) **Hello agent**
- [ ] One pure tool returning deterministic output.
- [ ] One long-running tool demonstrating:
  - streaming + cancellation

2) **Schema agent**
- [ ] One tool with strict input schema validation.
- [ ] Demonstrates structured error on invalid input.

### Audit acceptance
- [ ] Tool calls from example agents produce expected audit entries.
- [ ] Streaming/cancellation events are visible in UI/timeline.

### Exit criteria
- [ ] Building new agents is fast and consistent, and serves as the regression suite for protocol changes.

---

## Phase 4 — FleetPrompt MVP Agent (Weeks 4–6)

### CLI/UI acceptance (Core integration)
- [ ] You can select a repo project and run:
  - `com.fleetprompt.core/fp_validate_project_resources`
  - list/describe skills and workflows
  - run a skill/workflow
  - cancel an execution
- [ ] `.fleetprompt/` repo-local resources are indexed without code execution.

### Execution semantics acceptance
- [ ] Idempotency:
  - repeated run with same idempotency key returns stable result/reference
- [ ] Cancellation:
  - long-running runs can be canceled (best-effort) and end in a terminal state
- [ ] Structured logs are secret-free and show progress (streaming preferred)

### Audit acceptance
- [ ] Execution record exists for each run:
  - started/finished timestamps, status, inputs/outputs (redacted), correlation_id
- [ ] Signals emitted for execution lifecycle:
  - started/succeeded/failed (progressed optional)
- [ ] Side effects are directive-backed:
  - either Core wraps mutating tool calls as directives, or FleetPrompt requests directives and returns directive ids

### Security acceptance
- [ ] Path traversal protections in resource validation (`../` etc.)
- [ ] Permission enforcement:
  - tool registration and execution reflect approved permissions
- [ ] No secrets in logs/signals/execution records

### Exit criteria
- [ ] FleetPrompt provides a dependable local workflow runner integrated into Core audit timeline.

---

## Phase 5 — Graphonomous MVP Agent (Weeks 6–8)

### Config/resource acceptance
- [ ] Core can index `.fleetprompt/graphonomous/collections.json` without executing code.
- [ ] Invalid configs are surfaced with actionable errors.

### Tool acceptance
- [ ] `com.graphonomous.core/graph_list_collections` returns permission-filtered collections.
- [ ] `com.graphonomous.core/graph_search`:
  - requires explicit collections list
  - returns citations/provenance objects
- [ ] `com.graphonomous.core/graph_ingest`:
  - enforces write permissions
  - is directive-backed in chat/tool calling contexts (Core-wrapped or agent-requested)

### Idempotency/dedupe acceptance
- [ ] repeated ingest converges (no duplicate docs/embeddings/graph artifacts) by:
  - explicit dedupe_key and/or computed fingerprint
- [ ] concurrent ingests do not create duplicates

### Security acceptance
- [ ] ingestion rejects or redacts secret-like content before persistence/embedding
- [ ] search does not leak unauthorized collection data (including in errors)
- [ ] query logging does not persist raw query text by default (hash/metadata only)

### Audit acceptance
- [ ] ingest accepted/succeeded/failed signals exist with correlation linkage
- [ ] query performed signals exist (safe metadata only)

### Exit criteria
- [ ] Knowledge ingest + retrieval works with citations and permission boundaries.

---

## Phase 6 — A2A Traffic MVP Agent (Weeks 8–9)

### Resource acceptance
- [ ] Core can index `.fleetprompt/a2a/subscriptions.json` and `publications.json` without executing code.
- [ ] Invalid patterns/filters are rejected with actionable messages.

### Tool acceptance
- [ ] `com.a2atraffic.core/a2a_subscribe` creates a subscription (permission enforced).
- [ ] `com.a2atraffic.core/a2a_publish` publishes an event (permission enforced).
- [ ] `a2a_list_subscriptions` and `a2a_unsubscribe` work correctly.

### Delivery semantics acceptance
- [ ] At-least-once delivery is implemented:
  - retries with bounded backoff
  - terminal failure visibility (DLQ-like state)
- [ ] Dedupe works:
  - repeated publishes with same dedupe_key do not create duplicate fanout
  - dedupe hits are auditable

### Audit acceptance
- [ ] signals exist for:
  - event published (including dedupe marker)
  - delivery attempted (per subscriber)
  - subscription created/removed

### Security acceptance
- [ ] event payloads are secret-free (reject or redact policy is consistent)
- [ ] permission checks are enforced at both publish and delivery time (defense-in-depth)

### Exit criteria
- [ ] Two agents can coordinate safely via events with auditable delivery.

---

## Phase 7 — Delegatic MVP Agent (Weeks 9–11)

### Resource acceptance
- [ ] Core indexes `.fleetprompt/delegatic/company.json` without executing code.
- [ ] Delegatic validates config and returns structured validation output.

### Tool acceptance
- [ ] Tools are namespaced and callable:
  - list companies, describe company
  - start mission with idempotency_key
  - status returns timeline and step summary
  - cancel mission works (best-effort)

### Policy enforcement acceptance (critical)
- [ ] Deny-by-default:
  - tool allowlists enforced
  - directive allowlists enforced
  - resource scopes enforced (graph/fs/event/secrets)
- [ ] Confused deputy resistance:
  - a role cannot cause actions outside its allowlist even if Core permissions are broader

### Mission execution acceptance
- [ ] Mission state machine behaves as specified (created/planning/running/waiting/terminal).
- [ ] Idempotency:
  - same `(company_id, idempotency_key)` returns existing mission reference
  - conflicts are rejected
- [ ] Cancellation:
  - transitions mission to terminal canceled
  - stops issuing new directives/tool calls

### Audit acceptance
- [ ] Mission timeline is coherent and linkable:
  - mission/step lifecycle signals
  - directive linkage (directive ids)
  - correlation_id (recommend mission_id)
- [ ] Policy denials are recorded as facts (signals/timeline entries)

### Security acceptance
- [ ] No secrets in company config, mission goal/state, step inputs/outputs, logs.
- [ ] Event-driven triggers are deduped (no double-run on redelivery).

### Exit criteria
- [ ] Delegatic can safely orchestrate a small mission across FleetPrompt (+ optional Graphonomous/A2A) with full auditability.

---

## Phase 8 — Full portfolio demo slice + unified timeline UI (Weeks 11–12)

### Demo acceptance (one correlation_id, fully explainable)
A single scenario should demonstrate:
- [ ] Graphonomous search (with citations)
- [ ] FleetPrompt workflow execution (with logs + cancel)
- [ ] An event publish + delivery (A2A Traffic)
- [ ] Delegatic mission orchestration (policy enforced)

### Unified timeline UI acceptance (Core)
- [ ] Timeline view can filter by:
  - correlation_id (chat session / mission id)
  - agent id
  - tool id
- [ ] Timeline shows:
  - tool calls
  - signals
  - directives (if modeled)
  - executions (FleetPrompt/Delegatic)
- [ ] You can click through and answer:
  - “what happened?”
  - “why?”
  - “who did it?”
  - “what failed?”
+ without seeing secrets.

### Security acceptance
- [ ] UI protections verified (token + CSRF + localhost binding).
- [ ] Kill switch workflows exist:
  - cancel mission
  - stop agent
  - disable agent (or revoke permissions)

### Exit criteria
- [ ] End-to-end demo is stable, inspectable, and safe by default.

---

## Phase 9 — Hardening, reliability, and polish (Weeks 12–16, ongoing)

### Reliability acceptance
- [ ] Restart safety:
  - Core restarts agents safely; audit shows restarts and causes
  - Delegatic missions resume without duplicating side effects (idempotency keys)
  - A2A resumes deliveries without fanout duplication
- [ ] Backpressure:
  - A2A and ingestion have bounded queues and visible saturation behavior
- [ ] Rate limits/quotas exist for high-cost operations (LLM/network/ingest)

### Security hardening acceptance
- [ ] Secret redaction/denial is consistent across:
  - tool inputs/outputs
  - logs
  - signals/directives
  - persisted records
- [ ] Permission revocation is effective immediately at run time (TOCTOU safe).

### Protocol evolution acceptance
- [ ] Backward/forward compatible handling of unknown fields/types is verified.
- [ ] Frame size limits and error handling are tested.
- [ ] Optional compression is only introduced if observable and safe.

### Exit criteria
- [ ] You trust it to run continuously on your machine with predictable behavior.

---

## Appendix A — Minimal “protocol regression suite” (recommended)
The following should be re-run after any change to protocol, tool routing, or audit logging:
- hello agent handshake + tool register
- tool call success
- tool call invalid input → structured error
- streaming tool call
- cancellation of long-running tool call
- heartbeat failure → unhealthy marking
- audit log records for tool call and failures (secret-free)

---

## Appendix B — Minimal “security regression suite” (recommended)
Re-run after any change to UI/auth/logging:
- verify UI binds to 127.0.0.1
- verify token required for state-changing actions
- verify CSRF enabled
- verify no permissive CORS
- verify secrets are not persisted when included accidentally in tool inputs
- verify permission denies are fail-closed and auditable