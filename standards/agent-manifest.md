# Agent Manifest Standard (`opensentience.agent.json`)

This document standardizes how agents describe themselves so OpenSentience Core can discover/index/install safely.

Canonical source: `opensentience.org/project_spec/agent_marketplace.md`.

## Goals

- Parse metadata without executing code
- Stable agent IDs
- Declared permissions up front
- Declared capabilities (e.g. tools/streaming/cancellation)

## File location

- Repo root: `opensentience.agent.json`

## Required fields (v1)

- `id` (string): stable reverse-DNS identifier (e.g. `com.fleetprompt.core`)
- `name` (string)
- `version` (string, semver)
- `description` (string)
- `source` (object): `{ "git_url": string, "ref"?: string }`
- `entrypoint` (object): `{ "type": "mix_task"|"release"|"command", "value": string }`
- `permissions` (array of strings)
- `capabilities` (array of strings)

## Optional fields

- `homepage_url`
- `repository_url`
- `manifest_version` (default `1`)
- `tool_summary` (display-only; runtime registration is authoritative)

## Example

```json
{
  "manifest_version": 1,
  "id": "com.graphonomous.core",
  "name": "Graphonomous Knowledge Agent",
  "version": "1.0.0",
  "description": "Arcana-backed knowledge graph RAG agent",
  "source": { "git_url": "https://github.com/your-org/graphonomous" },
  "entrypoint": { "type": "mix_task", "value": "opensentience.agent" },
  "capabilities": ["tools", "streaming"],
  "permissions": [
    "filesystem:read:~/.opensentience/**",
    "database:rw:graphonomous"
  ]
}
```

## Permissions format

Permissions should be simple, explicit strings. Prefer capability-like scopes:

- `filesystem:read:<glob>`
- `filesystem:write:<glob>`
- `network:egress:<host-or-tag>`
- `database:rw:<db-or-scope>`
- `event:publish:<pattern>` / `event:subscribe:<pattern>` (bus-agnostic pub/sub)
- `tool:invoke:<agent_id>/<tool_name>` / `tool:invoke:*/<tool_name>` (tool identifiers are namespaced as `<agent_id>/<tool_name>`)

Rule: permissions are **requested** in the manifest and **approved** during “Enable”.
