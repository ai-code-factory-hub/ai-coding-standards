# FinOps KB · The Per-Tenant Cost Model

**Purpose.** A reusable, project-agnostic knowledge base for building **per-tenant cost visibility** in an enterprise SaaS product. Where [standards-kb Domain 16 · FinOps & Cost](../standards-kb/16-finops-cost.md) states *what must be true* (spend is attributable, bounded, and part of every design review), this KB is the **buildable, worksheet-level companion**: the actual cost model, the formulas, the metering you instrument, and the guardrails you wire up.

## Why per-tenant cost visibility matters

You can know your **total** cloud bill to the penny and still not know a single thing that matters commercially: **is this customer profitable?**

Blended margins hide the truth. A SaaS business with a healthy 80% gross margin in aggregate almost always contains tenants that cost 5–20× the average — a storage-heavy tenant, a tenant hammering an expensive third-party API, a tenant whose AI/LLM usage dwarfs their subscription. Without per-tenant cost you cannot:

- **Know if a customer is profitable** — the single most important FinOps question. Revenue is on the invoice; cost-to-serve is invisible until you model it.
- **Price correctly** — pricing tiers and entitlements set *before* you know cost-to-serve are guesses. One unprofitable tier can silently eat the margin of ten good ones.
- **Catch runaway tenants early** — a metered-usage explosion or a compromised credential shows up as a per-tenant cost anomaly long before it shows up on the monthly invoice.
- **Have an honest LTV** — LTV must use **gross margin**, not revenue (see [../data-kb/02-metrics-and-churn.md](../data-kb/02-metrics-and-churn.md)). Gross margin needs cost-to-serve. No per-tenant cost, no trustworthy LTV, no trustworthy LTV:CAC.

Per-tenant cost turns FinOps from an accounting exercise into a **product and pricing input**.

## How this extends standards-kb Domain 16

| [standards-kb/16-finops-cost.md](../standards-kb/16-finops-cost.md) — the standard | This KB — the build |
|---|---|
| **[MUST]** track cost per tenant across compute/storage/bandwidth/metered usage | **How:** the cost-driver taxonomy, allocation methods, and the tenant-cost formula → [01](01-per-tenant-cost-model.md) |
| **[SHOULD]** surface per-tenant cost alongside revenue; know unit economics vs subscription | **How:** contribution margin, cost-to-serve → pricing/entitlements, guardrails on unprofitable tiers → [02](02-unit-economics-and-pricing.md) |
| **[MUST]** tag every resource; budgets + anomaly alerts; hard guardrails on unbounded-cost paths | **How:** tagging schema, showback vs chargeback, budget/anomaly wiring, concrete caps → [03](03-cost-allocation-and-tagging.md) |

This KB **does not restate** the Domain 16 policy statements or its acceptance gate — it assumes them and shows the mechanics. Metric *definitions* (LTV, CAC, churn, ARPA) are **owned by** [../data-kb/02-metrics-and-churn.md](../data-kb/02-metrics-and-churn.md); this KB references them and adds the **cost side** those metrics depend on.

## Index

- **[01-per-tenant-cost-model.md](01-per-tenant-cost-model.md)** — *the worksheet.* Enumerates every cost driver per tenant (compute, DB, storage, egress, AI/LLM tokens, messaging, third-party per-call fees, support), the two allocation methods (direct-metered vs proportional), the tenant-cost and contribution-margin formulas, a worked example table, and the metering you must instrument.
- **[02-unit-economics-and-pricing.md](02-unit-economics-and-pricing.md)** — CAC, LTV, LTV:CAC, gross margin, CAC payback, contribution margin; how cost-to-serve informs pricing tiers and entitlements; guardrails that catch an unprofitable tier early; the link from usage-metered cost to metered/entitlement billing.
- **[03-cost-allocation-and-tagging.md](03-cost-allocation-and-tagging.md)** — cloud resource tagging strategy (owner/env/service/tenant), showback vs chargeback, budgets + anomaly alerts, hard guardrails on unbounded-cost paths (AI token caps, job concurrency, storage quotas, egress), and non-prod scale-down.

## Related KBs

- [../standards-kb/16-finops-cost.md](../standards-kb/16-finops-cost.md) — the FinOps standard this KB implements.
- [../standards-kb/15-ai-llm-governance.md](../standards-kb/15-ai-llm-governance.md) — AI/LLM token accounting and per-tenant caps; the biggest unbounded-cost driver in most modern products.
- [../data-kb/02-metrics-and-churn.md](../data-kb/02-metrics-and-churn.md) — canonical definitions of LTV, CAC, ARPA, gross margin, churn. This KB supplies the cost input those metrics consume.
- [../feature-blueprints/customer-analytics-churn.md](../feature-blueprints/customer-analytics-churn.md) — the internal analytics surface where per-tenant cost sits **beside** per-tenant revenue and churn signals.
- [../feature-blueprints/subscription-billing.md](../feature-blueprints/subscription-billing.md) — plans, entitlements, quotas, and metered `UsageRecord`s. The same usage meters that drive cost drive metered billing.

## Acceptance checklist

- [ ] Team can answer "is tenant X profitable?" with a number, not a guess.
- [ ] The per-tenant cost model in [01](01-per-tenant-cost-model.md) is instrumented and reconciles (within tolerance) to the total cloud + third-party bill.
- [ ] Per-tenant contribution margin is visible **beside** per-tenant revenue on an internal surface.
- [ ] Cost-to-serve is an explicit input to every pricing-tier and entitlement decision (see [02](02-unit-economics-and-pricing.md)).
- [ ] Every unbounded-cost path has a hard, per-tenant, server-side guardrail (see [03](03-cost-allocation-and-tagging.md)).
- [ ] This KB is understood as the *build* for standards-kb Domain 16, not a duplicate of it.
