# 05 · MCP Access Logs & Rate Limiting

Purpose: make every agent action **observable and bounded**. Log every tool call, cap frequency and cost per user/agent/tool, and detect anomalies. Extends [../standards-kb/20-logging-audit-and-traceability.md](../standards-kb/20-logging-audit-and-traceability.md) and [../standards-kb/16-finops-cost.md](../standards-kb/16-finops-cost.md).

## Log every tool call

- **[MUST]** Emit a structured audit log for **every** tool invocation — success and failure. Agent actions are privileged actions and must be as auditable as any API call. See [../standards-kb/20-logging-audit-and-traceability.md](../standards-kb/20-logging-audit-and-traceability.md).
- **[MUST]** Each log record includes at minimum:
  - **who / agent** — acting user/service identity and the agent/client id
  - **tool** — tool name and version
  - **args (masked)** — arguments with sensitive fields masked/redacted
  - **result summary** — status, row/output size, error code (not full sensitive payloads)
  - **tenant** — the server-derived tenant id
  - **correlation id** — trace/session id linking the call across the API and downstream systems
  - **latency** — duration; plus timestamp
  - authz decision (allowed/denied + scope) and any confirmation reference for destructive tools
- **[MUST]** **Mask/redact secrets and PII** in logged args and results; never log tokens, credentials, or full sensitive payloads. See [../standards-kb/09-compliance-privacy.md](../standards-kb/09-compliance-privacy.md).
- **[SHOULD]** Make logs **tamper-evident** and retained per policy; correlation id must join the MCP call to the wrapped API's own audit trail so a single trace spans agent → tool → API → data.

```json
{
  "ts": "2026-07-05T10:12:04Z",
  "correlation_id": "trc_9f2c...",
  "agent_id": "copilot-svc-14",
  "user_id": "u_8821",
  "tenant_id": "t_003",
  "tool": "refund_order",
  "tool_version": "2.1.0",
  "args_masked": { "order_id": "o_5567", "amount": "***" },
  "authz": { "decision": "allow", "scope": "orders:refund" },
  "confirmation_ref": "appr_77a1",
  "result": { "status": "ok", "rows": 1 },
  "latency_ms": 214
}
```

## Rate & frequency limits

- **[MUST]** Enforce **rate/frequency limits per user, per agent, and per tool** — a single agent stuck in a loop must not hammer a tool or exfiltrate at volume. Limit both request rate and total-calls-per-window.
- **[MUST]** Apply **tighter limits to destructive/high-impact and write tools** than to read tools; scope limits per tenant so one tenant can't starve another.
- **[SHOULD]** Return a clear `rate_limited` error with retry semantics; degrade gracefully (queue or reject) — never fail open.
- **[SHOULD]** Cap **result volume** (rows/output size) alongside call rate to bound data egress per window (ties to [04](04-mcp-safety-and-prompt-injection.md)).

## Token / cost caps

- **[MUST]** Enforce **token and cost budgets per tenant / user / agent / tool**, consistent with [../standards-kb/16-finops-cost.md](../standards-kb/16-finops-cost.md) and the cost controls in [../standards-kb/15-ai-llm-governance.md](../standards-kb/15-ai-llm-governance.md). An MCP surface can drive large model + downstream cost; it must have a ceiling.
- **[SHOULD]** Meter cost per agent/tool, cache deterministic/read results where safe, and cap the size of context a resource/tool can inject. Alert before budgets are exhausted; stop (fail closed) when they are.

## Anomaly detection

- **[SHOULD]** Monitor the audit stream for anomalies: spikes in call volume, unusual tool sequences, repeated authz denials, cross-tenant probes, off-hours activity, sudden cost jumps, or large/repeated outputs (possible exfiltration).
- **[SHOULD]** Alert and be able to **revoke/disable** a specific agent, client credential, or tool quickly when an anomaly fires (kill-switch). Feed signals into your SIEM/observability per [../standards-kb/07-observability-aiops.md](../standards-kb/07-observability-aiops.md).

## Acceptance checklist

- Every tool call (success and failure) is logged with who/agent, tool+version, masked args, result summary, tenant, correlation id, latency, and authz decision.
- Secrets/PII masked; logs tamper-evident and retained per policy; correlation id joins MCP → API → downstream traces.
- Rate/frequency limits enforced per user, agent, and tool; tighter on write/destructive; per-tenant; fail closed.
- Token/cost budgets enforced per tenant/user/agent/tool with alerting and hard stops; result volume capped.
- Anomaly detection on the audit stream with a fast per-agent/credential/tool kill-switch.
