# Tool Calling and Execution (Portfolio Standard)

## Tool calling loop (conceptual)

1. User sends message
2. Model streams text
3. Model emits tool calls (typed)
4. System validates tool calls + permissions
5. System executes tool calls
6. System feeds tool results back into the model
7. System emits:
   - signals (facts)
   - directives (intent / side effects)
   - audit events (security-relevant actions)

## Guardrails

- **Mutating tools should usually create directives**
  - The tool returns `{ directive_id, status }`.
  - A runner performs the mutation.

- **Streaming is optional but preferred**
  - If supported, stream partial output to the UI.

- **Cancellation is a first-class capability**
  - Long-running runs should be cancelable.

## Integration boundary

OpenSentience Core should treat agents as separate processes/nodes and route tool calls over a protocol. Agents should not be loaded in-process.
