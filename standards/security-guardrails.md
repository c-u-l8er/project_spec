# Security Guardrails (Portfolio Standard)

This document consolidates security posture into cross-component guardrails.

Primary source of lessons: FleetPrompt legacy (`fleetprompt.com/project_guide/07_SECURITY_GUARDRAILS.md`).

## Non-negotiable invariants

1. **Local admin surfaces are localhost-only by default**
   - Bind OpenSentience Admin UI to `127.0.0.1`.
   - Protect against drive-by actions (auth token + CSRF).

2. **No secrets in durable artifacts**
   - Signals, directives, and logs must never contain secrets.

3. **Side effects require explicit intent**
   - Side-effectful operations should be directive-backed and auditable.

4. **Dedupe + idempotency are security features**
   - Prevent replay/spoofing and “cost explosions”.

## Webhooks / external events

Inbound webhooks must:

1. verify signature
2. dedupe by provider event id
3. emit a persisted signal (secret-free)
4. enqueue durable processing (directive/job)

## Prompt injection posture

Assume prompt injection occurs.

Defense is architectural:

- model output does not directly cause side effects
- explicit typed actions
- user confirmation for high-impact directives

## Operational controls (minimum viable)

- rate limits / quotas for high-cost operations (LLM, network)
- per-agent tool allowlists
- kill-switch to disable an agent/integration quickly
