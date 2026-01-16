# Phase 1 Work Breakdown — OpenSentience Core MVP
## Module layout, storage schema, CLI/UI tasks, and minimal protocol regression suite plan

This document breaks Phase 1 into concrete, buildable tasks that minimize rework and set up Phase 2 (runtime protocol + ToolRouter) cleanly.

Phase 1 target (from `PROJECT_PHASES.md` / `PHASE_ACCEPTANCE.md`):
- Catalog + discovery of agent manifests (no code execution)
- Install/build/enable/run lifecycle (explicit trust boundary)
- Durable audit log (security-relevant)
- Launcher (separate OS process) + log capture
- Minimal admin UI skeleton on `127.0.0.1:6767` (token + CSRF + safe-by-default)
- CLI covering core lifecycle operations

This phase does **not** require full runtime protocol implementation, but it should scaffold for it.

---

## 0) Repository/module layout (recommended)

This layout assumes an Elixir/Phoenix Core project, but the concepts apply regardless.

### 0.1 Core application boundary
- `OpenSentience.Core` is the internal API boundary for:
  - catalog
  - discovery
  - install/build
  - enablement (permission approvals)
  - launcher
  - audit log
  - (Phase 2) protocol server + tool router

### 0.2 Suggested modules (Phase 1)
#### Catalog
- `OpenSentience.Catalog`
  - top-level API for querying and mutating agent records
- `OpenSentience.Catalog.AgentRecord`
  - struct/schema representing an agent in the catalog
- `OpenSentience.Catalog.Store`
  - persistence abstraction (SQLite recommended)
- `OpenSentience.Catalog.Query`
  - filtering/search helpers

#### Discovery
- `OpenSentience.Discovery`
  - orchestrates scans of roots, schedules rescans
- `OpenSentience.Discovery.Scanner`
  - filesystem walk logic (pure; no code execution)
- `OpenSentience.Discovery.ManifestReader`
  - reads/parses `opensentience.agent.json` safely
- `OpenSentience.Discovery.Hashing`
  - computes manifest hashes for change detection

#### Install/Build
- `OpenSentience.Install`
  - install orchestration: clone/fetch
- `OpenSentience.Install.Git`
  - runs `git clone/fetch/checkout` with sanitized args
- `OpenSentience.Build`
  - build orchestration: `mix deps.get`, `mix deps.compile` (explicit trust boundary)
- `OpenSentience.Build.Sandbox`
  - ensures build env is controlled (limited env vars, working dir)

#### Enablement / Permissions
- `OpenSentience.Permissions`
  - permission string parsing/validation utilities
  - matching helpers (globs/patterns where applicable)
- `OpenSentience.Enablement`
  - stores approved permissions per agent id + version/ref (policy decision)
- `OpenSentience.Enablement.Approvals`
  - CRUD operations for approvals
- `OpenSentience.Enablement.Validation`
  - validates that approved permissions are subset of requested manifest permissions

#### Launcher + Process Management
- `OpenSentience.Launcher`
  - start/stop/restart agent processes
- `OpenSentience.Launcher.CommandBuilder`
  - constructs safe command invocations
- `OpenSentience.Launcher.Env`
  - builds controlled environment, including session token (Phase 2 uses token)
- `OpenSentience.Launcher.LogCapture`
  - captures stdout/stderr, redacts best-effort, stores in log sink
- `OpenSentience.Launcher.Supervisor`
  - supervises per-agent process monitors, restarts (minimal in Phase 1)

#### Audit Log
- `OpenSentience.AuditLog`
  - append-only event writer and query API
- `OpenSentience.AuditLog.Event`
  - struct/schema: event_type, actor, subject, metadata, timestamps
- `OpenSentience.AuditLog.Redaction`
  - best-effort redaction utilities (defense-in-depth)
- `OpenSentience.AuditLog.Store`
  - persistence (same SQLite or separate store)

#### Admin UI (Phase 1 skeleton)
- `OpenSentience.Web` (Phoenix endpoint)
- `OpenSentience.Web.Auth`
  - token auth for state-changing actions
- `OpenSentience.Web.CSRF`
  - standard CSRF protection (Phoenix defaults)
- LiveView pages (or controllers):
  - `AgentsLive.Index` (list)
  - `AgentsLive.Show` (detail)
  - `AuditLive.Index` (basic audit listing)

#### CLI
- `Mix.Tasks.Opensentience.*` (Phase 1 is fine using Mix tasks)
  - `agents.list`
  - `agents.info`
  - `agents.install`
  - `agents.build`
  - `agents.enable`
  - `agents.run`
  - `agents.stop`
  - `registry.sync` (optional; can be Phase 1.5)

---

## 1) Storage schema (SQLite recommended)

Phase 1 storage should be minimal but durable and queryable.

### 1.1 Tables (recommended)
#### `agents`
One row per known agent (local discovered and/or installed).
- `agent_id` TEXT PRIMARY KEY
- `name` TEXT
- `version` TEXT
- `description` TEXT
- `source_git_url` TEXT
- `source_ref` TEXT NULL
- `manifest_path` TEXT (path to manifest on disk)
- `manifest_hash` TEXT (hash of manifest file contents)
- `discovered_at` TEXT (RFC3339)
- `last_seen_at` TEXT (RFC3339)
- `status` TEXT (enum-ish): `local_dev | local_uninstalled | installed | enabled | running | stopped | error`
- `install_path` TEXT NULL (e.g., `~/.opensentience/agents/<agent_id>/src`)
- `build_status` TEXT NULL: `not_built | building | built | failed`
- `build_last_at` TEXT NULL
- `last_error` TEXT NULL (safe, non-secret)

Notes:
- Store only safe summaries in `last_error`.
- Do not store secrets; never store session tokens.

#### `permission_approvals`
Approved permissions per agent id (and optionally per ref/version).
- `id` TEXT PRIMARY KEY
- `agent_id` TEXT (FK agents.agent_id)
- `approved_permissions_json` TEXT (JSON array)
- `approved_at` TEXT
- `approved_by` TEXT (actor id, safe)
- `requested_permissions_hash` TEXT (hash of requested permissions list for drift detection)
- `status` TEXT: `active | revoked`
- `revoked_at` TEXT NULL
- `revoked_by` TEXT NULL

Policy decision:
- You may want to key approvals by `(agent_id, source_ref)` to avoid “approval drift” across upgrades. If so, include `source_ref` or `manifest_hash` in this table and validate on run.

#### `runs`
Tracks launcher-level lifecycle state for agents.
- `run_id` TEXT PRIMARY KEY
- `agent_id` TEXT
- `started_at` TEXT
- `stopped_at` TEXT NULL
- `status` TEXT: `starting | running | stopped | crashed`
- `pid` INTEGER NULL (if available)
- `exit_code` INTEGER NULL
- `reason` TEXT NULL (safe)
- `last_heartbeat_at` TEXT NULL (Phase 2+)
- `session_id` TEXT NULL (Phase 2+)

#### `audit_events`
Append-only audit log.
- `event_id` TEXT PRIMARY KEY
- `at` TEXT (RFC3339)
- `event_type` TEXT (e.g., `agent.discovered`, `agent.installed`, `agent.enabled`, `agent.run_started`)
- `actor_type` TEXT: `human | system | agent`
- `actor_id` TEXT
- `subject_type` TEXT (e.g., `agent`, `permission_approval`, `run`)
- `subject_id` TEXT
- `correlation_id` TEXT NULL
- `causation_id` TEXT NULL
- `metadata_json` TEXT (JSON object; must be secret-free)

Optional:
- `severity` TEXT: `info|warn|error|security`

#### `logs` (optional Phase 1; can be file-backed)
If stored in SQLite:
- `log_id` TEXT PRIMARY KEY
- `at` TEXT
- `agent_id` TEXT
- `run_id` TEXT NULL
- `stream` TEXT: `stdout|stderr|core`
- `line` TEXT (redacted; bounded length)

MVP alternative:
- file-backed logs under `~/.opensentience/logs/<agent_id>/<run_id>.log` and only index recent lines.

---

## 2) Phase 1 task list (work items)

### 2.1 Catalog + discovery (no code execution)
1. Configurable scan roots (defaults: `~/Projects`, `~/.opensentience/agents`).
2. Scanner finds `opensentience.agent.json` (repo root).
3. Manifest parser:
   - validate required fields (per `project_spec/standards/agent-manifest.md`)
   - compute `manifest_hash`
4. Upsert into `agents` table.
5. Emit audit event:
   - `agent.discovered` / `agent.updated` with safe metadata
6. (Optional) file watcher for manifests (can be Phase 1.5).

Acceptance checkpoints:
- Discovery never runs `mix`, never executes code.
- Errors are actionable and do not leak secrets.

### 2.2 Install (git clone/fetch/checkout)
1. Implement “install” as:
   - clone repo into `~/.opensentience/agents/<agent_id>/src`
   - checkout desired ref (default or specified)
2. Persist `install_path`, `source_ref`.
3. Emit audit event `agent.installed` with:
   - git_url, ref, destination path (safe)
4. Capture stdout/stderr of git commands (but do not persist secrets; generally none expected).

Acceptance checkpoints:
- Install is explicit user action (CLI or UI).
- Install does not auto-run agent.
- Install is audited.

### 2.3 Build (explicit trust boundary)
1. Implement “build” step that runs:
   - `mix deps.get`
   - `mix deps.compile`
2. Strongly label and audit as “compilation executes code”.
3. Controlled env:
   - minimal env vars
   - controlled working directory
4. Persist build status + timestamps.
5. Emit audit event `agent.built` or `agent.build_failed`.

Acceptance checkpoints:
- Build is separate from install.
- Build is audited, including explicit “trust boundary” classification.

### 2.4 Enablement (approve permissions)
1. Read requested permissions from manifest.
2. CLI/UI flow to approve:
   - approve all, or approve subset
3. Validate:
   - approved ⊆ requested
4. Persist approval record in `permission_approvals`.
5. Emit audit event `agent.enabled` including:
   - requested_permissions_hash
   - approved_permissions summary (safe)
6. Support revoke:
   - mark approval revoked
   - emit `agent.permissions_revoked`.

Acceptance checkpoints:
- Deny-by-default.
- No “implicit enable”.
- Permission drift detection:
  - if manifest changes, require re-approval.

### 2.5 Launcher (run/stop)
1. Construct run command from manifest entrypoint:
   - Phase 1 can support `mix_task` first (per manifest standard)
2. Start process with:
   - controlled env
   - working directory
3. Capture stdout/stderr and attach to run_id.
4. Persist a `runs` record and update agent status.
5. Stop process (graceful then kill after timeout).
6. Emit audit events:
   - `agent.run_started`, `agent.run_stopped`, `agent.run_crashed` as applicable.

Acceptance checkpoints:
- Agents run as separate OS processes.
- Output capture works and is bounded.
- Stop works reliably.

### 2.6 Admin UI skeleton (localhost-only, safe)
1. Phoenix endpoint bound to `127.0.0.1:6767` by default.
2. Auth token model:
   - generate/store token under `~/.opensentience/` with restricted perms
   - require token for state-changing actions (install/build/enable/run/stop)
3. CSRF enabled (standard Phoenix).
4. Clickjacking protection:
   - `frame-ancestors 'none'` / deny framing.
5. CORS:
   - do not enable permissive CORS.
6. Screens:
   - Agents list (status, installed/enabled/running)
   - Agent detail (manifest, permissions requested/approved, actions)
   - Audit log view (recent events, filters)

Acceptance checkpoints:
- Drive-by protections in place.
- Actions are auditable.
- UI reuses Core APIs (no duplicated business logic).

### 2.7 CLI tasks (Mix tasks acceptable)
Implement CLI actions that call Core APIs:
- list agents
- show agent info
- install
- build
- enable (approve perms)
- run
- stop
- audit log tail (optional)

Acceptance checkpoints:
- CLI can fully operate Core without UI (important for early velocity).
- Errors are structured and safe.

---

## 3) Audit event taxonomy (Phase 1 minimum)

Define a minimal set of audit event types (strings), used consistently:
- `agent.discovered`
- `agent.updated`
- `agent.installed`
- `agent.build_started`
- `agent.built`
- `agent.build_failed`
- `agent.enable_requested`
- `agent.enabled`
- `agent.permissions_revoked`
- `agent.run_started`
- `agent.run_stopped`
- `agent.run_crashed`
- `security.denied` (for authorization failures in UI/CLI actions)

Each audit event should include:
- actor: `human|system`, `actor_id`
- subject: `agent:<agent_id>` or `run:<run_id>`
- safe metadata (no secrets)
- correlation_id (e.g., UI session id) where available

---

## 4) Minimal protocol regression suite plan (Phase 1 scaffolding)

Phase 2 will implement the runtime protocol; Phase 1 should still set up the scaffolding so Phase 2 is straightforward.

### 4.1 What to implement in Phase 1 (scaffolding only)
- [ ] Define a `protocol` test harness interface in Core (even if not implemented yet), e.g.:
  - “launch agent”
  - “wait for connection”
  - “send framed message”
  - “receive framed message”
- [ ] Reserve storage fields in `runs` for `session_id` and `last_heartbeat_at` (nullable).
- [ ] Ensure launcher passes a “session token” and socket path env vars (even if agent ignores them in Phase 1).

### 4.2 Regression suite cases (to run in Phase 2+, plan now)
When the protocol server exists, these should become automated integration tests:
1. Handshake success:
   - agent.hello → core.welcome
2. Tool registration:
   - agent.tools.register → core.tools.registered
3. Tool call success:
   - core.tool.call → agent.tool.result
4. Invalid input → structured error:
   - agent.tool.result.status=failed with error.code
5. Streaming:
   - agent.tool.stream chunks appear in order with seq
6. Cancellation:
   - core.tool.cancel results in agent.tool.result.status=canceled
7. Heartbeat:
   - agent.heartbeat received; missed heartbeat marks unhealthy

Phase 1 success is: the launcher and environment plumbing won’t need redesign to support these.

---

## 5) Critical path (dependency ordering within Phase 1)

1. Catalog storage + audit log store (foundation)
2. Manifest discovery + parser (feeds catalog)
3. Install (git) (feeds build)
4. Build (mix) (feeds run)
5. Enablement approvals (gates run)
6. Launcher run/stop + log capture (end-to-end lifecycle)
7. CLI surface (fast iteration)
8. UI skeleton (visibility + safer operator workflow)

---

## 6) Phase 1 “Done means done” (final exit criteria)

Phase 1 is complete when:
- You can discover, install, build, enable, run, and stop an agent.
- Every privileged action is recorded in the audit log.
- UI is localhost-only and state-changing actions require token + CSRF.
- No secrets are persisted in durable artifacts.
- Launcher runs agents as separate OS processes with bounded log capture.
- The codebase structure clearly supports adding the protocol server and ToolRouter in Phase 2 without refactors.