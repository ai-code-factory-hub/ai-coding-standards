# 06 · API Access Logs & Analytics

**Purpose:** capture **every API call** as a structured, PII-safe access record, then turn those
records into **usage metering, frequency/rate analytics, latency percentiles, and anomaly
detection**. This extends [20 · Logging, Audit & Traceability](../standards-kb/20-logging-audit-and-traceability.md)
(structured logs, correlation, masking) and feeds both the cost story in
[16 · FinOps & Cost](../standards-kb/16-finops-cost.md) and the
[Admin · API Usage & Analytics](../admin-kb/api-usage-and-analytics.md) dashboard.

## Log every API call

- **[MUST]** Emit **one structured access-log record per request**, at the gateway
  ([04](04-api-gateway-and-rate-limiting.md)), for **every** call — success, error, and rejected
  (401/403/429) alike. A dropped or throttled request is still a call worth recording.
- **[MUST]** Each record captures, at minimum:

  | Field | Purpose |
  |---|---|
  | `timestamp` | ISO-8601 UTC, when the request was received |
  | `correlation_id` / `request_id` | Ties to traces, error envelopes, and downstream logs |
  | `tenant_id` | Isolation + per-tenant metering |
  | `consumer_id` / `api_key_id` | Which app/key made the call (never the secret) |
  | `principal` / `auth_type` | user vs service; oauth / api_key / mtls |
  | `method` + `route` (templated) | `GET /v1/lab-orders/{id}` — templated, **not** the raw path with ids |
  | `api_version` | Which major version was called ([05](05-versioning-and-lifecycle.md)) |
  | `status_code` | Outcome |
  | `latency_ms` | Server-processing duration (and upstream/gateway split if available) |
  | `request_bytes` / `response_bytes` | For metering and payload analysis |
  | `client_ip` (masked) + `user_agent` | Coarse source for anomaly review |
  | `rate_limit_state` | remaining/limit, and whether this call was throttled |
  | `error_code` | The envelope `code` on non-2xx, for error analytics |

- **[MUST]** **Mask/omit PII and secrets** — never log `Authorization` headers, tokens, API-key
  plaintext, request/response bodies containing PII, or query-string secrets. Log the **templated
  route**, not raw ids/PII in the path. Masking is enforced at the logging layer, not left to
  callers, per [20 · Logging, Audit & Traceability](../standards-kb/20-logging-audit-and-traceability.md).
- **[MUST]** Access logs are **append-only** and **tenant-attributable**; retain per policy on
  durable storage. High volume — partition by tenant + time and roll up (below) rather than
  querying raw indefinitely.

> **Example (labelled — one masked access-log record):**
> ```json
> {
>   "timestamp": "2026-07-05T10:12:03Z",
>   "correlation_id": "01J8ZB2C7QF2M4C",
>   "tenant_id": "t_9f2",
>   "api_key_id": "ak_live_a1b2",
>   "auth_type": "api_key",
>   "method": "GET",
>   "route": "/v1/lab-orders/{id}",
>   "api_version": "v1",
>   "status_code": 200,
>   "latency_ms": 42,
>   "request_bytes": 0,
>   "response_bytes": 1841,
>   "client_ip": "203.0.113.0/24",
>   "rate_limit_state": { "limit": 600, "remaining": 512, "throttled": false }
> }
> ```

## Usage metering & frequency

- **[MUST]** Aggregate raw records into **usage rollups** (per tenant / consumer / key / endpoint /
  version, per time bucket) covering:
  - **Call frequency / rate** — calls per minute/hour/day, per consumer and per endpoint.
  - **Quota consumption** — usage against the plan quota ([04](04-api-gateway-and-rate-limiting.md)),
    with % consumed and projected exhaustion.
  - **Error rate** — 4xx / 5xx / 429 as a share of calls, sliced by endpoint and consumer.
  - **Latency percentiles** — **p50 / p95 / p99** per endpoint (averages hide tail pain; percentiles
    are the SLO signal).
  - **Bandwidth** — request/response bytes per consumer, for cost attribution.
- **[MUST]** Metering must be **accurate enough to bill or enforce quota on** — reconcilable with
  the gateway's own counters, not a lossy sample, where usage drives revenue or limits.
- **[SHOULD]** Precompute rollups (streaming or scheduled) so dashboards and quota checks read
  aggregates, not raw logs; keep raw records for drill-down within the retention window.

## Anomaly detection & alerting

- **[MUST]** Detect and **alert** on anomalies against per-consumer baselines:
  - sudden **traffic spikes/drops** (possible abuse, runaway client, or an outage),
  - **error-rate surges** (a broken client or a regression),
  - **latency-percentile regressions** (p95/p99 breaching the SLO, [17 · NFR](../standards-kb/17-nfr-coverage-gaps.md)),
  - **quota near-exhaustion** and **repeated 429s** (a consumer needing an upgrade or throttling),
  - **auth anomalies** — spikes in 401/403, a key used from a new geo/IP, credential-stuffing
    patterns (route to the security surfaces).
- **[SHOULD]** Route alerts to the on-call/SRE channel and, for consumer-facing conditions (quota,
  deprecation usage), notify the account per [05 · Versioning](05-versioning-and-lifecycle.md)
  and [07 · Portal](07-developer-portal-and-sdks.md).

> **Example (labelled — anomaly alert):** Consumer `ak_live_a1b2`'s 5xx rate jumps from a 0.2%
> baseline to 34% over 5 minutes on `POST /v1/orders`, while p99 latency triples. The pipeline
> fires a `consumer_error_surge` alert with the correlation ids of a sample of failing calls so
> on-call can jump straight to the traces.

## Ties to other domains

- **[MUST]** Records use the **same `correlation_id`** as traces and the error envelope
  ([01](01-api-design-guide.md)) so a single id joins gateway log → service trace → error, per
  [20 · Logging, Audit & Traceability](../standards-kb/20-logging-audit-and-traceability.md).
- **[SHOULD]** Feed per-tenant/consumer usage and bandwidth into **cost attribution / showback** per
  [16 · FinOps & Cost](../standards-kb/16-finops-cost.md), so API usage maps to spend and to plan
  economics.

## Surfaces in the admin analytics screen

- **[MUST]** These metrics power the [Admin · API Usage & Analytics](../admin-kb/api-usage-and-analytics.md)
  dashboard — calls over time, top consumers/endpoints, error rate, latency percentiles,
  rate-limit/quota hits, per-tenant usage — with **drill-down from any chart to the underlying
  access-log rows**.

## Acceptance checklist

- [ ] One structured record is emitted for every call (including 401/403/429) with correlation id, tenant, consumer/key, method, templated route, version, status, latency, and bytes.
- [ ] PII and secrets are masked/omitted at the logging layer; routes are templated, not raw ids/PII; records are append-only.
- [ ] Rollups compute call frequency, quota consumption, error rate, p50/p95/p99 latency, and bandwidth per tenant/consumer/endpoint/version.
- [ ] Metering is accurate enough to bill or enforce quota on and reconciles with gateway counters.
- [ ] Anomaly detection alerts on traffic spikes, error surges, latency regressions, quota exhaustion, and auth anomalies.
- [ ] The same correlation id joins access log, trace, and error envelope.
- [ ] Per-consumer usage/bandwidth feeds cost attribution (FinOps).
- [ ] All metrics surface in the admin analytics dashboard with drill-down to raw access-log rows.
