# 02 · Unit Economics & Pricing

> How the per-tenant cost from [01](01-per-tenant-cost-model.md) feeds unit economics and **pricing decisions**. Metric definitions are **canonical in** [../data-kb/02-metrics-and-churn.md](../data-kb/02-metrics-and-churn.md) — this file references them and adds the **cost-to-serve → pricing** loop that Domain 16 ([../standards-kb/16-finops-cost.md](../standards-kb/16-finops-cost.md)) requires.

## The metrics (definitions owned by data-kb)

These are the canonical definitions from [../data-kb/02-metrics-and-churn.md](../data-kb/02-metrics-and-churn.md), reproduced here only for the cost linkage. **[MUST]** use those definitions as the single source of truth.

| Metric | Definition (canonical) | Why cost-to-serve matters |
|---|---|---|
| **Gross margin %** | `(revenue − cost_to_serve) / revenue` | The whole point of [01](01-per-tenant-cost-model.md): `cost_to_serve` **is** `tenant_cost`. No per-tenant cost → margin is a blended guess. |
| **Contribution margin** | `tenant_revenue − tenant_cost` (per tenant/segment) | Direct output of the [01](01-per-tenant-cost-model.md) worksheet. |
| **CAC** | `fully_loaded_S&M_spend / new_customers` | Acquisition cost; independent of serving cost but paired with it below. |
| **LTV** | `ARPA × gross_margin% / revenue_churn_rate` | **Uses gross margin, not revenue** — so LTV is only as honest as your cost model. |
| **LTV:CAC** | `LTV / CAC` | Target ≥ 3:1. Overstated whenever gross margin is overstated (i.e. cost-to-serve ignored). |
| **CAC payback** | `CAC / (ARPA × gross_margin%)`, in months | Same gross-margin dependency; target ≤ 12–18 months. |

**Key coupling:** four of these six metrics collapse to garbage if `gross_margin%` is wrong, and `gross_margin%` is wrong whenever `cost_to_serve` is a blended average instead of the per-tenant number from [01](01-per-tenant-cost-model.md). **Per-tenant cost is the input that makes unit economics trustworthy.**

## Worked example — two tenants, same revenue, different truth

Both pay $1,200/mo. CAC $4,000. Revenue churn 2%/mo.

| | Tenant A (light) | Tenant B = "Acme Labs" (heavy, from [01](01-per-tenant-cost-model.md)) |
|---|---|---|
| Revenue / mo | $1,200 | $1,200 |
| Cost-to-serve / mo | $360 | $1,493 |
| Contribution margin | **+$840 (70%)** | **−$293 (−24%)** |
| LTV `= ARPA × GM% / churn` | `1,200 × 0.70 / 0.02` = **$42,000** | `1,200 × (−0.24) / 0.02` = **−$14,650** |
| LTV:CAC | 10.5:1 ✅ | negative ❌ |
| CAC payback | `4,000 / (1,200×0.70)` = **4.8 mo** | never |

**Blended view** (the trap): average GM = `(70% + −24%)/2` ≈ 23%, LTV ≈ $13,800, LTV:CAC ≈ 3.5:1 — looks *acceptable*, and completely hides that acquiring another Tenant B **destroys** value. The per-tenant model catches it; the blended model rewards it.

## How cost-to-serve informs pricing tiers & entitlements

Pricing is where the cost model pays for itself. The loop:

```
per-tenant cost model ([01]) → cost-to-serve per segment → price/entitlement design → guardrails → back to cost model
```

- **[MUST]** Set each tier's **price floor from its cost-to-serve**, not from competitor prices alone. A tier's list price must clear its modeled cost-to-serve **plus** target contribution margin at expected usage.
- **[MUST]** Attach **quotas/limits to the entitlements** that gate the expensive drivers (LLM tokens, messages, storage, third-party calls). Entitlements are defined in [../feature-blueprints/subscription-billing.md](../feature-blueprints/subscription-billing.md); this KB says *which* to bound and *at what level* (the level that keeps the tier profitable).
- **[SHOULD]** Design tiers so the **cost-driving dimension is the pricing dimension**. If LLM tokens dominate cost (as in the [01](01-per-tenant-cost-model.md) example), the tier should meter or cap tokens — not seats. Charge for what costs you money.
- **[SHOULD]** For high-variance drivers, prefer **usage-metered / overage billing** over all-you-can-eat, so revenue tracks cost per tenant instead of decoupling from it.
- **[SHOULD]** Model the **worst-case profitable tenant** per tier (max usage within entitlement) and confirm it still clears margin. If the entitlement ceiling allows a loss, lower the ceiling or raise the price.

### Metered cost ↔ metered billing — one meter, two ledgers

The usage events from [01](01-per-tenant-cost-model.md) are the same events that drive metered billing:

| Usage event `{tenant_id, driver, quantity}` | → Cost ledger ([01](01-per-tenant-cost-model.md)) | → Billing ledger ([subscription-billing.md](../feature-blueprints/subscription-billing.md)) |
|---|---|---|
| 60M LLM tokens | `× unit_cost` = internal cost | `× price / overage_rate` = invoice line |
| 9,400 SMS segments | provider fee | metered `UsageRecord` → invoice |
| 320 KYC lookups | vendor per-call | overage vs quota |

**[MUST]** Emit each billable/costable usage event **once**, idempotently, and let both ledgers consume it. Two separate meters for cost and billing **will** drift and you will bill for usage you didn't cost, or cost usage you didn't bill.

## Guardrails — catch an unprofitable tier early

The failure mode is a tier that loses money on every customer, scaled by sales. Detect it before, not after.

- **[MUST]** Compute **contribution margin per pricing tier** (not just per tenant) every period and alert when a tier's **median contribution margin goes negative** or below a floor. A whole tier underwater is a pricing bug, not a tenant problem.
- **[MUST]** Gate **new-tier / new-plan launch** on a cost-to-serve model: no plan ships without a modeled worst-case contribution margin ≥ target. Ties to the Domain 16 **[MUST]** that cost is reviewed in design/architecture review.
- **[SHOULD]** Alert on **per-tenant margin cliffs** — a tenant whose contribution margin drops (e.g. > 20 points) period-over-period, usually a usage/behavior change. Route to success/pricing before renewal.
- **[SHOULD]** Track **% of tenants below margin floor** as a first-class metric beside churn and NRR on the internal surface ([../feature-blueprints/customer-analytics-churn.md](../feature-blueprints/customer-analytics-churn.md)). Rising unprofitable-tenant count is an early pricing/packaging warning.
- **[SHOULD]** When a tenant is structurally unprofitable (like Acme in [01](01-per-tenant-cost-model.md)), the playbook is: **cap the driver** (guardrails, [03](03-cost-allocation-and-tagging.md)) → **meter the overage** (billing) → **reprice/migrate the tier** — in that order.

## Acceptance checklist

- [ ] Gross margin and LTV use **per-tenant cost-to-serve** from [01](01-per-tenant-cost-model.md), not a blended average.
- [ ] Metric definitions are taken from [../data-kb/02-metrics-and-churn.md](../data-kb/02-metrics-and-churn.md) (no divergent local definitions).
- [ ] Each pricing tier has a price floor derived from its modeled cost-to-serve + target margin.
- [ ] The cost-driving dimension is bounded by an entitlement/quota on every tier; worst-case in-entitlement usage still clears margin.
- [ ] Usage events feed both the cost ledger and metered billing from **one** idempotent meter.
- [ ] Contribution margin per **tier** is computed each period; a tier going negative alerts.
- [ ] No new plan/tier launches without a cost-to-serve model showing positive worst-case contribution margin.
- [ ] "% tenants below margin floor" is tracked beside churn/NRR on the internal surface.
