# 01 · The Per-Tenant Cost Model — Worksheet

> **The core artifact of this KB.** A concrete model for computing the cost to serve one tenant. Implements the **[MUST]** "track cost per tenant" from [../standards-kb/16-finops-cost.md](../standards-kb/16-finops-cost.md). Metric *definitions* (LTV/CAC/ARPA/gross margin) live in [../data-kb/02-metrics-and-churn.md](../data-kb/02-metrics-and-churn.md); this file produces the **cost** those metrics depend on.

## The model in one line

```
tenant_cost = Σ (driver_usageₜ × unit_costₜ)   +   allocated_shared_cost
              └─────── direct-metered ───────┘      └── proportional ──┘

contribution_margin = tenant_revenue − tenant_cost
contribution_margin% = contribution_margin / tenant_revenue
```

Everything below is the expansion of those two lines: **what the drivers are**, **how to get `driver_usage` per tenant** (metering), **how to allocate the shared part**, and **a worked example**.

## Cost drivers per tenant

Enumerate every driver that scales with tenant activity. Classify each as **directly meterable** (you can attribute the exact unit to a tenant) or **shared** (must be allocated). This classification decides the allocation method.

| # | Cost driver | Typical unit | Directly meterable? | Notes |
|---|---|---|---|---|
| 1 | **Compute** (app servers, workers, functions) | vCPU-hours, GB-hours, invocations, request-CPU-ms | Partly | Serverless/per-request → direct. Shared cluster → proportional by request count or CPU-time. |
| 2 | **Database** | provisioned instance $, IOPS, storage-GB, read/write units | Partly | Managed serverless DB (per-RU/RCU) → direct. Shared instance → proportional by query count / rows / connection-time. |
| 3 | **Storage (at rest)** | GB-month, by tier (hot/warm/cold) | **Yes** | Object storage keyed by `tenant_id` prefix → direct and exact. |
| 4 | **Egress / bandwidth** | GB transferred out, CDN GB | **Yes** (mostly) | Attribute by request logs / CDN logs carrying tenant. Inter-AZ/replication → shared. |
| 5 | **AI / LLM tokens** | input tokens, output tokens, per-model $/1K, embeddings, vector-store $ | **Yes** | Highest-variance driver. Meter per call; see [../standards-kb/15-ai-llm-governance.md](../standards-kb/15-ai-llm-governance.md). |
| 6 | **Messaging** | SMS segments, emails sent, push notifications | **Yes** | Per-send provider fees; meter at send time with `tenant_id`. |
| 7 | **Third-party per-call fees** | per API call, per verification (KYC), per lookup, per document | **Yes** | External metered APIs (payments, KYC, geocoding, enrichment). Attribute at call site. |
| 8 | **Support & success** | ticket-hours, CSM-hours, onboarding-hours | Partly | Loaded human cost. Meter from the ticketing/CRM system; else allocate proportionally by seats or plan. |
| — | **Shared platform** | observability, CI/CD, control-plane, security tooling, load balancers, NAT, licenses | No | Genuinely shared overhead. Allocate proportionally (see below). |

**[MUST]** Every driver that is *technically* attributable to a tenant (rows 3–7 above) is **direct-metered**, not allocated. Allocation is a fallback for genuinely shared cost, not a shortcut to avoid instrumentation.

## Allocation methods

### A. Direct-metered (preferred)

You have a usage record tagged with `tenant_id`, and a unit cost. Cost is exact.

```
driver_cost(tenant, driver) = usage(tenant, driver) × unit_cost(driver)
```

- Unit cost comes from the provider's rate card (LLM $/1K tokens, SMS $/segment, storage $/GB-month, egress $/GB) or from a reconciled internal rate.
- **[MUST]** Reconcile the sum of metered driver cost across all tenants against the provider invoice each period; drift > a set tolerance (e.g. 5%) means the meter is wrong or a driver is unattributed.

### B. Proportional (for shared cost only)

Shared cost that cannot be keyed to a tenant is allocated by an **allocation basis** — a per-tenant weight that correlates with consumption of that shared resource.

```
allocated_shared_cost(tenant) = shared_cost_total × ( basis(tenant) / Σ basis(all tenants) )
```

| Shared cost | Sensible allocation basis |
|---|---|
| Shared compute cluster | tenant's share of request count or CPU-time |
| Shared DB instance | tenant's share of query volume / rows stored |
| Observability, logging | tenant's share of log/metric volume (or request count) |
| Control-plane, LB, NAT, security tooling | tenant's share of requests, or seats, or a flat per-tenant + usage blend |
| Platform licenses (per-seat tools) | tenant's seat count |

- **[SHOULD]** Pick **one** basis per shared bucket and document it. Mixing bases per period makes trends meaningless.
- **[SHOULD]** Prefer a **usage-correlated** basis (requests, CPU-time) over a flat per-tenant split; flat splits make small tenants look unprofitable and hide heavy tenants.
- **[MUST NOT]** Allocate a cost proportionally when it is directly meterable — that launders a heavy tenant's real cost across everyone.

## Instrumenting per-tenant usage metering

Direct-metered drivers only exist if you emit the meter. This is the build cost of the whole model.

- **[MUST]** Every request path and background job carries a **`tenant_id` in context** and stamps it onto every cost-relevant event (LLM call, message send, third-party call, storage write, egress-generating response).
- **[MUST]** Emit a **usage event** at the point of spend containing `{tenant_id, driver, quantity, unit, timestamp, resource_ref}`. Do not reconstruct usage after the fact from bills — you cannot re-attribute an aggregate invoice to tenants.
- **[MUST]** Use the **same usage meters** for cost as for metered billing — the `UsageRecord` stream in [../feature-blueprints/subscription-billing.md](../feature-blueprints/subscription-billing.md) and the cost model read the same events. One meter, two consumers (bill the customer, cost the tenant).
- **[SHOULD]** Aggregate raw usage events into a **daily per-tenant per-driver rollup**; compute cost by joining the rollup to a **rate table** (versioned unit costs). Storing cost = usage × rate lets you re-cost history when rates change.
- **[SHOULD]** Store usage events **idempotently** (dedupe by event key) so retries don't double-count — same discipline as metered billing usage dedup.
- **[SHOULD]** Keep the **rate table versioned with effective dates** so a mid-period price change is applied correctly and history stays reproducible.
- **[SHOULD]** Surface the per-tenant cost rollup on the internal analytics surface **beside revenue** — see [../feature-blueprints/customer-analytics-churn.md](../feature-blueprints/customer-analytics-churn.md).

## Worked example — sample tenant "Acme Labs"

A single tenant on a $1,200/month plan, one billing period. Illustrative rates only.

### Direct-metered drivers

| Driver | Usage | Unit cost | Method | Cost |
|---|---|---|---|---|
| Compute (serverless) | 1.8M request-CPU-s | $0.0000133 / CPU-s | direct | $23.94 |
| Database (serverless RUs) | 42M read + 6M write units | $0.09 / 1M RU | direct | $4.32 |
| Storage (hot) | 340 GB-month | $0.023 / GB-mo | direct | $7.82 |
| Storage (cold/archive) | 1,200 GB-month | $0.004 / GB-mo | direct | $4.80 |
| Egress / CDN | 210 GB | $0.085 / GB | direct | $17.85 |
| **AI / LLM tokens** | 48M in + 12M out (mid model) | $3 in / $15 out per 1M | direct | $324.00 |
| Messaging (SMS) | 9,400 segments | $0.0075 / segment | direct | $70.50 |
| Messaging (email) | 60,000 sends | $0.40 / 1,000 | direct | $24.00 |
| Third-party KYC lookups | 320 verifications | $0.25 / verification | direct | $80.00 |
| Support | 2.5 ticket-hours | $65 / loaded hour | direct | $162.50 |
| **Subtotal — direct** | | | | **$719.73** |

### Allocated shared cost (proportional)

Acme's allocation basis = 3.1% of total request volume this period.

| Shared bucket | Total | Basis | Allocated |
|---|---|---|---|
| Observability + logging | $9,000 | 3.1% of log volume ≈ 3.1% | $279.00 |
| Control-plane / LB / NAT / security | $14,000 | 3.1% of requests | $434.00 |
| Platform licenses (per-seat) | $6,000 | Acme 40 of 4,000 seats = 1.0% | $60.00 |
| **Subtotal — allocated** | | | **$773.00** |

> ⚠️ Reality check: allocated shared cost ($773) exceeding direct cost ($720) is a signal the platform is **overhead-heavy** or the tenant is small. Watch that allocation doesn't dominate the model — if it does, either the business has a fixed-cost problem or the allocation basis is too coarse.

### Putting it together

```
tenant_cost          = 719.73 (direct) + 773.00 (allocated) = 1,492.73
tenant_revenue       = 1,200.00
contribution_margin  = 1,200.00 − 1,492.73 = −292.73
contribution_margin% = −292.73 / 1,200.00  = −24.4%
```

**Finding:** Acme Labs is **unprofitable at −24%** — driven by LLM tokens ($324) and a heavy share of shared overhead. Invisible in a blended 80%-margin dashboard; obvious the moment you model per tenant. Actions: cap/tier the AI usage (see [03](03-cost-allocation-and-tagging.md) and [../standards-kb/15-ai-llm-governance.md](../standards-kb/15-ai-llm-governance.md)), move them to a usage-metered plan, or reprice the tier (see [02](02-unit-economics-and-pricing.md)).

## Acceptance checklist

- [ ] Every cost driver in the taxonomy is classified direct-metered vs shared, and documented.
- [ ] All technically-attributable drivers (storage, egress, LLM, messaging, third-party) are **direct-metered**, not allocated.
- [ ] A usage event `{tenant_id, driver, quantity, unit, timestamp}` is emitted at each point of spend; events are idempotent.
- [ ] Cost is computed as `usage × versioned_rate` from a daily per-tenant per-driver rollup (re-costable history).
- [ ] Shared cost uses one documented, usage-correlated allocation basis per bucket.
- [ ] Sum of per-tenant cost reconciles to the total cloud + third-party bill within tolerance each period.
- [ ] The same usage meters feed both cost and metered billing ([subscription-billing.md](../feature-blueprints/subscription-billing.md)).
- [ ] `contribution_margin` and `contribution_margin%` are computed and shown per tenant beside revenue.
