# Admin · API Usage & Analytics

> The operational dashboard for **how the API is actually being used** — calls over time, top
> consumers and endpoints, error rates, latency percentiles, rate-limit and quota hits, and
> per-tenant usage — every chart drilling down to the underlying access log. This is the
> human-facing read-out of the metering pipeline in [`../api-kb/06-api-access-logs-and-analytics.md`](../api-kb/06-api-access-logs-and-analytics.md).

## What it is / when you need it

The moment your API has more than a handful of consumers, "is it healthy and who is using it?"
becomes a daily question — for support (a customer says calls are failing), for capacity (which
endpoint is hot), for revenue (who is near quota and should upgrade), and for security (a key
behaving anomalously). This surface answers all of them from one place. It reads the aggregated
usage rollups produced by the access-log pipeline and lets an admin slice by tenant, consumer,
endpoint, and version, then **drill from any chart straight into the raw access-log rows** behind
it. It complements [`api-keys-and-tokens.md`](api-keys-and-tokens.md) (the credential side) and
[`data-access-logs.md`](data-access-logs.md) (the sensitive-read side) — this one is about
**volume, performance, and economics**, not individual-record disclosure.

## Data model

Tenant-scoped views over the append-only access log; the admin surface reads **rollups** for
charts and the **raw log** for drill-down. Nothing here is a second copy of PII — see masking in
[`../api-kb/06-api-access-logs-and-analytics.md`](../api-kb/06-api-access-logs-and-analytics.md).

### ApiUsageRollup (aggregate — what the charts read)

| Field | Type | Notes |
| --- | --- | --- |
| `id` | ulid | PK |
| `tenant_id` | fk | Isolation + per-tenant slicing |
| `consumer_id` / `api_key_id` | fk? | Which app/key (null = all) |
| `route` | string | Templated, e.g. `GET /v1/lab-orders/{id}` — never raw ids |
| `api_version` | string | `v1`, `v2` |
| `bucket_start` | timestamp | Hour/day bucket (UTC) |
| `request_count` | bigint | Total calls |
| `error_4xx_count` / `error_5xx_count` | bigint | Split client vs server errors |
| `throttled_429_count` | bigint | Rate-limit / quota rejections |
| `p50_ms` / `p95_ms` / `p99_ms` | int | Latency percentiles |
| `request_bytes` / `response_bytes` | bigint | Bandwidth, for cost attribution |

### QuotaStatus (per consumer, current period)

| Field | Type | Notes |
| --- | --- | --- |
| `consumer_id` / `api_key_id` | fk | |
| `tenant_id` | fk | |
| `plan_tier` | enum | `free` \| `pro` \| `enterprise` |
| `quota_limit` | bigint | Calls allowed this period |
| `quota_used` | bigint | Consumed so far |
| `period_reset_at` | timestamp | When the quota resets |
| `projected_exhaustion_at` | timestamp? | Forecast from current rate |

### UsageAnomaly (fired by the pipeline)

| Field | Type | Notes |
| --- | --- | --- |
| `id` | ulid | PK |
| `tenant_id` / `consumer_id` | fk | |
| `type` | enum | `traffic_spike` \| `error_surge` \| `latency_regression` \| `quota_near_exhaustion` \| `auth_anomaly` |
| `detected_at` | timestamp | |
| `baseline` / `observed` | numeric | For context in the UI |
| `sample_correlation_ids` | string[] | Jump-to-log / trace |
| `status` | enum | `open` \| `acknowledged` \| `resolved` |

### AccessLogRecord (raw — drill-down target)

The append-only per-call record defined in [`../api-kb/06-api-access-logs-and-analytics.md`](../api-kb/06-api-access-logs-and-analytics.md)
(correlation id, tenant, consumer/key, method, templated route, version, status, latency, bytes,
masked IP, rate-limit state). This surface **reads** it; it does not own its schema.

## Key screens & UX

1. **Overview dashboard** — top-line tiles (total calls, error rate, p95 latency, 429 rate) plus a
   **calls-over-time** chart with selectable interval and version/consumer filters. *Loading:* tile
   + chart skeletons. *Empty:* "No API traffic in this period." *Error:* retry banner.
2. **Top consumers / top endpoints** — ranked tables (by calls, errors, or bytes) of the busiest
   consumers/keys and endpoints, each row linking to its detail. Answers "who/what is driving load?"
3. **Error-rate view** — 4xx / 5xx / 429 over time, sliced by endpoint and consumer, with the most
   common error `code`s; a spike links to the failing calls. *Empty:* "No errors in this period."
4. **Latency view** — **p50 / p95 / p99** per endpoint over time (percentiles, not averages), with
   SLO threshold lines and a highlight when p95/p99 breaches ([`../standards-kb/17-nfr-coverage-gaps.md`](../standards-kb/17-nfr-coverage-gaps.md)).
5. **Rate-limit & quota view** — 429 counts, consumers near or over quota, `projected_exhaustion_at`,
   and plan tier — the upgrade/throttle-candidate list ([`../api-kb/04-api-gateway-and-rate-limiting.md`](../api-kb/04-api-gateway-and-rate-limiting.md)).
6. **Per-tenant usage** — usage and bandwidth broken down by tenant for multi-tenant fairness and
   cost attribution ([`../standards-kb/16-finops-cost.md`](../standards-kb/16-finops-cost.md)).
7. **Anomalies** — open `UsageAnomaly` items with baseline-vs-observed context; acknowledge/resolve,
   and jump to sample correlation ids.
8. **Drill to access log** — from **any** chart or table, click through to the filtered raw
   access-log rows (same correlation id joins to traces), respecting masking and RBAC.

## Rules & logic

- [MUST] Admin/analyst-role only, **RBAC-gated** and **tenant-scoped** — a tenant admin sees only
  their tenant's usage; platform admins may see cross-tenant. See [`../standards-kb/04-security.md`](../standards-kb/04-security.md).
- [MUST] Charts read **precomputed rollups**, not raw logs, so the dashboard is fast at scale;
  drill-down queries the raw access log within the retention window.
- [MUST] **No PII or secrets** are shown — routes are templated, ids are opaque, IPs masked; this
  surface inherits the masking guarantees of [`../api-kb/06-api-access-logs-and-analytics.md`](../api-kb/06-api-access-logs-and-analytics.md)
  and [`../standards-kb/20-logging-audit-and-traceability.md`](../standards-kb/20-logging-audit-and-traceability.md).
- [MUST] Latency is reported as **p50/p95/p99 percentiles**, never only averages (averages hide the
  tail that consumers actually feel).
- [MUST] Every chart supports **drill-down to the underlying access-log rows** so a number is always
  traceable to the calls that produced it.
- [MUST] Metering shown here is **consistent with the quota-enforcement counters** at the gateway —
  the dashboard and the enforcement path do not disagree.
- [SHOULD] Surface **quota near-exhaustion** and **429-heavy** consumers as an actionable list
  (upgrade or throttle candidates), linkable to [`api-keys-and-tokens.md`](api-keys-and-tokens.md).
- [SHOULD] Show **deprecated-endpoint usage per consumer** to drive migration outreach
  ([`../api-kb/05-versioning-and-lifecycle.md`](../api-kb/05-versioning-and-lifecycle.md)).
- [SHOULD] Let anomalies be **acknowledged/resolved** and route high-severity ones (auth anomalies,
  error surges) to the security/on-call surfaces.
- [SHOULD] Support **export** (CSV/scheduled) of usage rollups for finance/capacity reporting;
  exports are themselves logged.

## Applicable standards

- Access-log schema, metering, percentiles, anomaly detection — [`../api-kb/06-api-access-logs-and-analytics.md`](../api-kb/06-api-access-logs-and-analytics.md)
- Rate-limit / quota semantics and tiers — [`../api-kb/04-api-gateway-and-rate-limiting.md`](../api-kb/04-api-gateway-and-rate-limiting.md)
- Structured logging, correlation ids, PII masking, export audit — [`../standards-kb/20-logging-audit-and-traceability.md`](../standards-kb/20-logging-audit-and-traceability.md)
- RBAC, tenant isolation — [`../standards-kb/04-security.md`](../standards-kb/04-security.md)
- Latency SLOs / NFRs — [`../standards-kb/17-nfr-coverage-gaps.md`](../standards-kb/17-nfr-coverage-gaps.md)
- Usage → cost attribution / showback — [`../standards-kb/16-finops-cost.md`](../standards-kb/16-finops-cost.md)
- Table/chart/filter UX, states — [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md)
- Related admin surfaces — [`api-keys-and-tokens.md`](api-keys-and-tokens.md) · [`data-access-logs.md`](data-access-logs.md) · [`README.md`](README.md)

## Acceptance checklist

- [ ] Admin/analyst-only, RBAC-gated, tenant-scoped access.
- [ ] Overview shows calls over time plus total-calls / error-rate / p95-latency / 429-rate tiles.
- [ ] Top-consumers and top-endpoints rankings are available and drillable.
- [ ] Error-rate view splits 4xx/5xx/429 by endpoint and consumer with common error codes.
- [ ] Latency view shows p50/p95/p99 per endpoint with SLO thresholds — not averages.
- [ ] Rate-limit/quota view lists near-exhaustion and 429-heavy consumers with projected exhaustion and plan tier.
- [ ] Per-tenant usage and bandwidth breakdown supports fairness and cost attribution.
- [ ] Every chart drills down to the underlying (masked) access-log rows via correlation id.
- [ ] Charts read precomputed rollups; metering is consistent with gateway quota counters.
- [ ] Anomalies can be acknowledged/resolved and routed; loading/empty/error states designed for every screen.
