# Admin · Background Jobs & Queue Monitor

> Operational visibility into background work: job runs with status, duration and retries; a **dead-letter queue (DLQ)** with requeue/cancel actions; live **queue depth**; a scheduled-job calendar; and failure alerts.

## What it is / when you need it

Everything asynchronous in the app — emails, exports, imports, webhooks, report generation, nightly rollups — runs as background jobs on queues. When something silently doesn't happen (a report never lands, an invoice never sends), this is where ops looks. It shows what ran, what's running, what's stuck, and what failed into the DLQ, and it gives safe controls to **retry, requeue, or cancel**. It reads from the worker/queue infrastructure and pairs with [`log-explorer.md`](log-explorer.md) (drill from a failed job to its logs) and [`communication-logs.md`](communication-logs.md) (a failed send job → the message record).

## Data model

Tenant-scoped where jobs carry tenant context; platform jobs are visible only to platform admins. `JobRun` is **append-only** (a run is never edited; a retry is a new run linked to the original). Retention: job runs kept for an operational window (commonly 30–90 days); DLQ items kept until resolved.

### JobRun (append-only)

| Field | Type | Notes |
| --- | --- | --- |
| `id` | ulid | Time-ordered PK |
| `tenant_id` | fk? | Null for platform-level jobs |
| `job_type` | string | e.g. `export.generate`, `email.send` |
| `queue` | string | Queue name |
| `status` | enum | `queued` \| `running` \| `succeeded` \| `failed` \| `retrying` \| `dead` \| `cancelled` |
| `priority` | int | Scheduling priority |
| `attempt` | int | 1-based attempt number |
| `max_attempts` | int | Retry cap |
| `parent_run_id` | ulid? | Prior attempt this run retries |
| `scheduled_for` | timestamp? | For delayed/scheduled jobs |
| `started_at` | timestamp? | |
| `finished_at` | timestamp? | |
| `duration_ms` | int? | Derived |
| `worker_id` | string? | Executing worker/pod |
| `correlation_id` | string | Ties to originating request + logs |
| `payload_ref` | string | Pointer to payload (PII masked/not inlined) |
| `error_class` | string? | On failure |
| `error_message` | text? | On failure (masked) |

### DeadLetterItem (DLQ)

| Field | Type | Notes |
| --- | --- | --- |
| `id` | ulid | PK |
| `tenant_id` | fk? | |
| `job_run_id` | fk | The run that exhausted retries |
| `job_type` | string | |
| `failed_at` | timestamp | |
| `error_class` | string | |
| `error_message` | text | Masked |
| `payload_ref` | string | For inspection/requeue |
| `status` | enum | `dead` \| `requeued` \| `discarded` |
| `resolved_by` | fk? | Admin who acted |
| `resolution_note` | text? | Required to discard |

### ScheduledJob (recurring definition)

| Field | Type | Notes |
| --- | --- | --- |
| `id` | ulid | PK |
| `tenant_id` | fk? | |
| `job_type` | string | |
| `cron` | string | Schedule expression |
| `timezone` | string | Schedule tz |
| `enabled` | bool | |
| `last_run_at` | timestamp? | |
| `next_run_at` | timestamp? | Computed |
| `last_status` | enum? | Outcome of last run |

### QueueStat (live/rolled-up)

| Field | Type | Notes |
| --- | --- | --- |
| `queue` | string | |
| `depth` | int | Pending jobs |
| `oldest_age_ms` | int | Age of oldest pending (backlog signal) |
| `in_flight` | int | Currently running |
| `failure_rate` | float | Rolling window |

## Key screens & UX

1. **Queue overview** — per-queue tiles: depth, oldest-pending age, in-flight, failure rate, DLQ count. Backlog/failure tiles turn warning/critical past thresholds. *Loading:* skeleton tiles. *Empty:* "No active queues." *Error:* retry.
2. **Job runs (table)** — virtualized: time, job type, queue, status (color-coded), attempt/max, duration, worker. Filters: date range, job type, queue, status, tenant (platform admins), failed-only, long-running. Row expands to timing, error, and **"open logs"** (correlation-id pivot) / **"view related record"**.
3. **DLQ inspector** — dead items with error class/message and payload preview (masked). Actions: **Requeue** (single/bulk, safe/idempotent), **Discard** (note required), **Copy correlation id**. Confirmation on bulk.
4. **Running job detail** — live status, elapsed time, worker, and **Cancel** for a stuck/runaway job (cooperative cancellation where supported).
5. **Scheduled-job calendar** — recurring jobs on a calendar/timeline with cron, timezone, next-run, last outcome; toggle **enable/disable**; "run now". *Empty:* "No scheduled jobs."
6. **Failure alerts** — surfaced when failure rate, backlog age, or DLQ count breach thresholds; route to on-call/notifications.

## Rules & logic

- [MUST] Admin/ops-role only, **RBAC-gated**; tenant admins see only their tenant's jobs, platform jobs are platform-admin-only — see [`../standards-kb/04-security.md`](../standards-kb/04-security.md).
- [MUST] Job runs are **append-only**; a retry is a **new run** linked via `parent_run_id`, never an in-place edit — the history stays intact.
- [MUST] **Requeue / cancel / discard emit their own audit entries** (actor, target run/DLQ item, reason) per [`../standards-kb/20-logging-audit-and-traceability.md`](../standards-kb/20-logging-audit-and-traceability.md).
- [MUST] Requeue must be **idempotent / safe** — re-running a job must not double-charge, double-send, or double-write; rely on idempotency keys — see [`../standards-kb/05-reliability-resilience.md`](../standards-kb/05-reliability-resilience.md).
- [MUST] **Payloads are not inlined raw**; sensitive fields are masked and only referenced by `payload_ref`; unmask is permission-gated and audited.
- [MUST] Every run carries a **`correlation_id`** so a failed job drills straight to its logs — see [`../standards-kb/07-observability-aiops.md`](../standards-kb/07-observability-aiops.md).
- [MUST] **Exports are permission-gated and themselves audited**.
- [SHOULD] Enforce retry with **backoff** and a `max_attempts` cap; exhaustion moves the run to the **DLQ** rather than retrying forever.
- [SHOULD] Emit **failure/backlog alerts** on threshold (failure rate, oldest-pending age, DLQ growth) to on-call/notifications.
- [SHOULD] Support cooperative **cancellation** of long-running jobs and mark them `cancelled`, not `failed`.
- [SHOULD] Show queue **depth and oldest-age** as the primary backlog health signals.

## Applicable standards

- Access control, tenant isolation — [`../standards-kb/04-security.md`](../standards-kb/04-security.md)
- Audit of requeue/cancel/discard, export audit — [`../standards-kb/20-logging-audit-and-traceability.md`](../standards-kb/20-logging-audit-and-traceability.md)
- Retries, backoff, idempotency, DLQ, graceful cancellation — [`../standards-kb/05-reliability-resilience.md`](../standards-kb/05-reliability-resilience.md)
- Queue metrics, correlation ids, failure alerting — [`../standards-kb/07-observability-aiops.md`](../standards-kb/07-observability-aiops.md)
- Payload masking, retention of job history — [`../standards-kb/09-compliance-privacy.md`](../standards-kb/09-compliance-privacy.md)
- Monitor/calendar UX, states — [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md)
- Related: [`log-explorer.md`](log-explorer.md) · [`communication-logs.md`](communication-logs.md) · [`../feature-blueprints/import-export-pipeline.md`](../feature-blueprints/import-export-pipeline.md) · [`README.md`](README.md)

## Acceptance checklist

- [ ] Queue overview shows depth, oldest-pending age, in-flight, failure rate, and DLQ count per queue.
- [ ] Job-runs table filters by type/queue/status/date and drills to logs via correlation id.
- [ ] Job runs are append-only; retries create linked new runs.
- [ ] DLQ inspector supports safe/idempotent requeue and note-required discard, single and bulk.
- [ ] Running jobs can be cancelled cooperatively and are marked `cancelled`.
- [ ] Requeue/cancel/discard emit audit entries; payloads are masked.
- [ ] Scheduled-job calendar shows cron/timezone/next-run/last-status with enable/disable and run-now.
- [ ] Failure/backlog alerts fire on threshold and route to on-call.
- [ ] Retention, RBAC/tenant scoping, export audit, and loading/empty/error states are all present.
