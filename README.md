# Agent Action Gate

A fail-closed execution boundary for consequential AI-agent actions.

Agent Action Gate sits immediately before a side effect. It snapshots the proposed action, obtains an authority decision, verifies that the action has not changed, and calls `execute` only for an unambiguous `allow`.

```js
import { runActionGate } from "./index.mjs";

const result = await runActionGate({
  action: { tool: "email.send", args: { to: "ops@example.com" } },
  decide: async (action, { signal }) => {
    const response = await fetch("https://api.interailabs.dev/verify", {
      method: "POST",
      signal,
      headers: {
        authorization: `Bearer ${process.env.INTERAI_API_KEY}`,
        "content-type": "application/json",
      },
      body: JSON.stringify({
        use_case: "agent-before-tool-execution",
        action: {
          schema: "interai-canonical-action/v1",
          tool_id: action.tool,
          type: "tool_call",
          operation: "execute",
          arguments: action.args,
          external_side_effect: true,
          irreversible: false,
        },
        context: { environment: "production" },
      }),
    });

    if (!response.ok) throw new Error(`InterAI ${response.status}`);
    return response.json();
  },
  execute: async (exactAction) => sendEmail(exactAction.args),
});
```

## Execution rule

The built-in authority reader implements the current InterAI authority contract. Both fields must be present, valid, and identical:

```text
recommended_action = allow | review_required | block
policy_result      = allow | review_required | block
```

Only `allow + allow` can reach `execute`.

`review_required`, `block`, provider exceptions, provider timeout, malformed responses, unknown authority values, missing fields, contradictory fields, or action mutation all keep the execution boundary closed.

## Why this exists

Agent frameworks often make it easy to call tools but leave enforcement semantics to the host. A timeout, 502, malformed policy response or swallowed hook exception must not accidentally turn uncertainty into permission for a consequential action.

```text
agent proposes action
       |
       v
 Agent Action Gate
       |
       +--> decision unavailable/invalid --> CLOSED
       +--> REVIEW_REQUIRED -------------> CLOSED
       +--> BLOCK -----------------------> CLOSED
       +--> action changed --------------> CLOSED
       |
       +--> ALLOW + exact action match
                    |
                    v
                 execute
```

## Exact-action binding

Agent Action Gate composes with [Exact Action Binding](https://github.com/InterAILabs/exact-action-binding) instead of maintaining a duplicate implementation. The dependency snapshots the action and detects mutation between decision and execution.

That public host-side binding is deliberately distinct from Risk Oracle's hosted `CanonicalExecutionIntent`, `execution_intent_digest`, `ExecutionAuthorization`, expiry, policy identity and single-use semantics.

## API

- `runActionGate({ action, decide, execute, timeoutMs? })`
- `readAuthorityDecision(response)`
- `GateClosedError` with machine-readable `code`

The `decide` callback receives `(actionSnapshot, { binding, signal })`.
The `execute` callback receives `(actionSnapshot, { binding, decision_response })`.

Default decision timeout: 5000 ms.

## Run tests

Requires Node.js 20 or newer and network access during dependency installation.

```bash
npm install
npm test
```

## Relationship to InterAI

This repository is an open-source execution-boundary tool from InterAI Labs. [InterAI Risk Oracle](https://github.com/InterAILabs/ai-risk-oracle) is the related hosted decision service and remains a separate product.

This repository contains enforcement/integration logic only. It does not contain Risk Oracle's private decision engine, scoring/trust intelligence, authoritative policy implementation, billing/storage internals, signing internals, account data or production operations.

Hosted Risk Oracle API: `https://api.interailabs.dev`

## License

Apache-2.0. See `LICENSE` and `NOTICE`.
