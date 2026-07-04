# 04 · API Gateway & Rate Limiting

**Purpose:** the edge that fronts every service — where authentication, rate limiting, quotas,
caching, transformation, and WAF are enforced once, consistently, for all consumers. This makes
the [10 · API & Integration](../standards-kb/10-api-integration.md) rule "rate limits + quotas per
tenant/user/key (`429` + `X-RateLimit-*`)" real and centralizes the security controls from
[03 · API Security](03-api-security.md).

## Gateway responsibilities

The gateway is the **single enforcement point** at the edge. It **[MUST]** own:

- **AuthN/AuthZ pre-check** — validate the token/key/mTLS cert and required scopes before the
  request reaches a service (the service remains authoritative for per-object authz, [03](03-api-security.md)).
- **Rate limiting, quotas, and spike arrest** — see below.
- **Routing** to the correct service/version and **request/response transformation**.
- **TLS termination**, **WAF**, request size limits, and **correlation-id injection** (generate one
  if absent; propagate downstream) that ties into the access log ([06](06-api-access-logs-and-analytics.md)).
- **Response caching** for cacheable reads.

- **[MUST]** Business logic **[MUST NOT]** live in the gateway — it does policy, not domain rules.
- **[SHOULD]** The gateway config is **declarative and version-controlled** (deployed via CI,
  [12 · DevOps & CI/CD](../standards-kb/12-devops-cicd.md)), not hand-edited in a console.

## Rate limits & quotas

- **[MUST]** Enforce limits **per consumer** — keyed by **tenant, consumer/app, and API key**, not
  just source IP (shared NATs make IP unfair). A single tenant cannot exhaust capacity for others
  (noisy-neighbor / fairness).
- **[MUST]** Distinguish two controls:
  - **Rate limit** — requests per short window (per second/minute); protects capacity.
  - **Quota** — total requests per long window (per day/month/billing-cycle); enforces the plan.
- **[MUST]** On a limit breach return **`429 Too Many Requests`** with a **`Retry-After`** header
  and standard **`X-RateLimit-*`** headers on every response (not only on 429s):

> **Example (labelled — rate-limit response headers):**
> ```http
> HTTP/1.1 429 Too Many Requests
> Retry-After: 12
> X-RateLimit-Limit: 600
> X-RateLimit-Remaining: 0
> X-RateLimit-Reset: 1751710332
> Content-Type: application/json
>
> { "error": { "code": "rate_limited", "message": "Rate limit exceeded. Retry after 12s.",
>   "correlation_id": "01J8ZB2C..." } }
> ```

- **[SHOULD]** Quota responses use `429` with a distinct `code` (e.g. `quota_exceeded`) and a
  `Retry-After` pointing at the quota reset, so clients can tell "slow down" from "you're out for
  the period".

## Spike arrest, throttling, burst vs sustained

- **[MUST]** Implement limits with a **token-bucket / leaky-bucket** algorithm that separates
  **burst** (short spike allowance) from **sustained** (steady refill rate) — e.g. sustained
  600 req/min with a burst bucket of 100. This absorbs legitimate spikes without letting a runaway
  client saturate the backend.
- **[SHOULD]** Add **spike arrest** (a hard per-second smoothing cap) in front of the sustained
  limit so a thundering burst is flattened even within an allowed per-minute budget.
- **[SHOULD]** Apply **concurrency limits** (max in-flight requests per consumer) for expensive
  endpoints, and shed load with `503` + `Retry-After` when the backend is unhealthy rather than
  queueing unboundedly.

## Tiered limits by plan

- **[MUST]** Limits and quotas are **configurable per plan/tier** (free / pro / enterprise) and
  attach to the consumer's credential, so upgrading a plan raises limits without code changes.
- **[SHOULD]** Enforce tiers from a **central policy** the billing/subscription system updates, so
  the gateway and the plan stay in sync; expose current tier + remaining quota to the consumer via
  headers or a `GET /me/usage` endpoint ([Admin · API Usage & Analytics](../admin-kb/api-usage-and-analytics.md)).

> **Example (labelled — tiered policy):**
>
> | Tier | Sustained | Burst | Monthly quota |
> |---|---|---|---|
> | Free | 60 req/min | 20 | 100k |
> | Pro | 600 req/min | 100 | 5M |
> | Enterprise | 6,000 req/min | 1,000 | custom |

## Response caching

- **[MUST]** Cache **only** safe, cacheable `GET`s, keyed **including the tenant/authorization
  scope** — never serve one tenant's cached response to another. Vary on auth context.
- **[SHOULD]** Honor `Cache-Control`/`ETag`; use **short TTLs** for near-real-time data and
  explicit cache-busting on writes. Authenticated PII responses default to **no shared caching**.

## Request/response transformation

- **[MAY]** The gateway may **transform** at the edge: strip internal headers, inject correlation
  ids, normalize versions ([05 · Versioning](05-versioning-and-lifecycle.md)), or adapt legacy
  payload shapes — but transformation **[MUST NOT]** alter business semantics or hide validation.

## WAF

- **[MUST]** A **Web Application Firewall** fronts the gateway: OWASP-core-ruleset protection
  (injection, XSS, path traversal), IP/geo/bot reputation controls, and payload inspection — tuned
  to avoid blocking legitimate API traffic (JSON bodies, not just form posts).
- **[SHOULD]** Pair the WAF with **DDoS protection** at the edge and **anomaly-based blocking**
  informed by the access-log analytics in [06](06-api-access-logs-and-analytics.md).

## Acceptance checklist

- [ ] The gateway is the single edge enforcing authN pre-check, rate limits, quotas, TLS, WAF, and correlation-id injection; no business logic in it.
- [ ] Limits are keyed per tenant/consumer/key (not just IP); rate limit and quota are distinct controls.
- [ ] Breaches return `429` with `Retry-After` and `X-RateLimit-*` headers on every response.
- [ ] Token/leaky-bucket separates burst from sustained; spike arrest and concurrency limits protect the backend.
- [ ] Limits/quotas are configurable per plan tier and driven by central billing policy; consumers can read remaining quota.
- [ ] Response caching is tenant/scope-keyed with short TTLs; PII responses are not shared-cached.
- [ ] Edge transformation never changes business semantics or bypasses validation.
- [ ] A WAF with OWASP rules and DDoS protection fronts the gateway, tuned for JSON API traffic.
- [ ] Gateway config is declarative and deployed via CI, not hand-edited.
