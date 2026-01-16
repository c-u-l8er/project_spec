# Signals and Directives (Portfolio Standard)

This is a portfolio-wide standard derived from FleetPrompt’s proven mental model.

## Definitions

- **Signal** = immutable fact (“what happened”). Durable, replayable.
- **Directive** = explicit intent (“what we want to happen”). Durable, auditable, retryable.

## Rules

1. **Facts → signals**
   - Use signals for events you may need to audit or replay.

2. **Side effects → directives**
   - Anything externally impactful or safety-sensitive must be performed via a directive runner.
   - Tools in chat UIs that mutate state should generally create directives, not perform writes directly.

3. **No secrets in durable records**
   - Signals, directives, and logs must not include credentials, auth headers, tokens, cookies, etc.
   - Store secrets in a dedicated encrypted store and reference them by id.

4. **Idempotency and dedupe are required**
   - Signals: use `dedupe_key` when sources can redeliver.
   - Directives: use `idempotency_key` so “double click” and retries are safe.

## Design intention

These primitives should be used consistently across:

- OpenSentience Core audit log and marketplace actions
- FleetPrompt skills/workflows (typed actions)
- A2A event routing (signal-like semantics)
- Delegatic orchestration (directive-driven)
- Graphonomous ingestion and indexing (directive-driven for write actions)
