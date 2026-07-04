# Admin · Application / Error Log Explorer

> A search UI over structured JSON application/error logs — filter by level, service, correlation id, tenant and time; **group errors** by fingerprint, drill from a log line to its distributed **trace**, and save queries. Logs are PII-masked and tamper-evident.

## What it is / when you need it

The engineer's and SRE's window into what the app is actually doing. Where the audit log answers *"who changed what"*, the log explorer answers *"what happened and why did it break"* — stack traces, request lifecycles, service errors. You need it for incident debugging, root-cause analysis, and post-mortems. It queries the structured log store (JSON lines), fingerprints repeated errors into groups so an inbox of 10,000 exceptions collapses to a handful of issues, and pivots on `correlation_id` / `trace_id` to reconstruct a request across services. It never mutates logs — logs are write-once.

## Data model

Tenant-scoped views over a shared, tenant-partitioned log store. `LogEntry` is **immutable / write-once**. Retention: hot searchable window (commonly 14–30 days) then rolled to cheaper cold storage; error groups retained longer for trend analysis.

### LogEntry (structured, immutable)

| Field | Type | Notes |
| --- | --- | --- |
| `id` | ulid | PK |
| `tenant_id` | fk | Partition key; enforced in every query |
| `timestamp` | timestamp | UTC, ms precision |
| `level` | enum | `trace` \| `debug` \| `info` \| `warn` \| `error` \| `fatal` |
| `service` | string | Emitting service/module |
| `env` | enum | `prod` \| `staging` \| `dev` |
| `message` | string | Human message |
| `logger` | string | Logger/source name |
| `correlation_id` | string | Ties a workflow across services |
| `trace_id` | string? | Distributed-trace id |
| `span_id` | string? | Span within the trace |
| `request_id` | string? | Single request |
| `actor_id` | fk? | User/service account, if in-request |
| `error_fingerprint` | string? | Hash of normalized stack/message → group key |
| `stack` | text? | Stack trace (present on errors) |
| `attributes` | jsonb | Arbitrary structured fields (PII-masked at ingest) |
| `host` | string | Instance/pod |
| `version` | string | Build/release |

### ErrorGroup (fingerprint rollup)

| Field | Type | Notes |
| --- | --- | --- |
| `fingerprint` | string | Group key |
| `tenant_id` | fk | |
| `title` | string | Representative message |
| `service` | string | |
| `level` | enum | Usually `error`/`fatal` |
| `first_seen_at` | timestamp | |
| `last_seen_at` | timestamp | |
| `count` | int | Occurrences in window |
| `status` | enum | `open` \| `muted` \| `resolved` |
| `sample_log_id` | ulid | Exemplar entry for drill-in |

### SavedQuery

| Field | Type | Notes |
| --- | --- | --- |
| `id` | ulid | PK |
| `tenant_id` | fk | |
| `owner_id` | fk | Creator |
| `name` | string | |
| `filter_json` | jsonb | Level/service/correlation/time/free-text |
| `shared` | bool | Visible to other admins in tenant |

## Key screens & UX

1. **Log search** — a query bar (free-text + structured filters) over a virtualized, reverse-chronological result table: time, level (color-coded), service, message, correlation id. Filters: level, service, env, time range, correlation id, trace id, actor, free-text on message/attributes. Live-tail toggle. *Loading:* streaming skeleton. *Empty:* "No logs match — widen the time range or filters." *Error:* "Query failed / timed out" with retry and a "simplify query" hint.
2. **Log line detail** — the full JSON entry pretty-printed, stack trace with frames, all attributes (PII masked), and pivots: **"show correlation trail"** (same `correlation_id`) and **"open trace"** (drill to the distributed trace via `trace_id`).
3. **Error groups (issues)** — fingerprinted errors as an inbox: title, service, count, first/last seen, trend sparkline, status. Actions: **Mute**, **Resolve**, open representative entry. Sorted by frequency or recency.
4. **Trace drill-in** — waterfall of spans for a `trace_id` with per-span timing and the log lines attached to each span.
5. **Saved queries** — personal and shared saved filters; one click re-runs; shared queries visible tenant-wide to admins.

## Rules & logic

- [MUST] Admin/engineer-role only, **RBAC-gated**; every query is **tenant-scoped** on `tenant_id` — no cross-tenant log leakage even for platform admins without explicit elevation — see [`../standards-kb/04-security.md`](../standards-kb/04-security.md).
- [MUST] Logs are **write-once / immutable** — no edit or delete from this UI; **no log tampering**. Deletion only via retention/erasure policy, never ad hoc.
- [MUST] **PII and secrets are masked at ingest** and never rendered raw; unmask (where lawful) is permission-gated and audited per [`../standards-kb/20-logging-audit-and-traceability.md`](../standards-kb/20-logging-audit-and-traceability.md).
- [MUST] Every log line carries a **`correlation_id`**; error entries carry an `error_fingerprint` so grouping and cross-service pivots work — see [`../standards-kb/07-observability-aiops.md`](../standards-kb/07-observability-aiops.md).
- [MUST] **Exports are permission-gated and themselves audited** (who exported which query/window).
- [MUST] Viewing logs is itself an **auditable admin action** where the query touches sensitive services.
- [SHOULD] Fingerprint errors into groups (normalize stack/message) and support **mute/resolve** to keep the issue inbox actionable.
- [SHOULD] Enforce query cost/time limits (bounded time range, result caps) to protect the log store; degrade gracefully with a clear message.
- [SHOULD] Support drill from log → trace → spans so a single request can be reconstructed end-to-end.
- [SHOULD] Roll hot logs to cold storage on schedule and make the retention boundary visible in the UI.

## Applicable standards

- Access control, tenant isolation, secret handling — [`../standards-kb/04-security.md`](../standards-kb/04-security.md)
- Structured logging, correlation ids, tamper-evidence, export audit — [`../standards-kb/20-logging-audit-and-traceability.md`](../standards-kb/20-logging-audit-and-traceability.md)
- Tracing, error grouping, observability pipeline — [`../standards-kb/07-observability-aiops.md`](../standards-kb/07-observability-aiops.md)
- Query cost limits, graceful degradation — [`../standards-kb/05-reliability-resilience.md`](../standards-kb/05-reliability-resilience.md)
- PII masking, log retention/erasure — [`../standards-kb/09-compliance-privacy.md`](../standards-kb/09-compliance-privacy.md)
- Search/table UX, live-tail, states — [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md)
- Related: [`data-access-logs.md`](data-access-logs.md) · [`job-and-queue-monitor.md`](job-and-queue-monitor.md) · [`../feature-blueprints/audit-log-and-activity.md`](../feature-blueprints/audit-log-and-activity.md) · [`README.md`](README.md)

## Acceptance checklist

- [ ] Search filters by level, service, env, time, correlation id, trace id, actor, and free-text — every query tenant-scoped.
- [ ] Log entries are immutable; no edit/delete path exists in the UI.
- [ ] PII/secrets are masked; unmask is permission-gated and audited.
- [ ] Log detail shows full JSON + stack; correlation-id pivot reconstructs the workflow.
- [ ] Drill from a log line to its distributed trace and spans works.
- [ ] Errors fingerprint into groups with count/first-seen/last-seen; mute/resolve work.
- [ ] Saved queries (personal + shared) re-run correctly.
- [ ] Query cost/time limits protect the store; failures degrade with a clear message.
- [ ] Exports are permission-gated and audited; loading/empty/error states present.
