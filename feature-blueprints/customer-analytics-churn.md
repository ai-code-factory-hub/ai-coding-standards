# Feature Blueprint · Customer Analytics & Churn

> Revenue and retention dashboards for paid / trial / unpaid segments: MRR/ARR, churn (logo + revenue), retention cohorts, activation, DAU/MAU, LTV/CAC, and at-risk/churn-risk scoring — with a customer 360.

## What it is / when you need it

The lens on business health: are we growing, who's about to leave, and which customers are healthy. It consumes the governed metrics definitions in [`../data-kb/02-metrics-and-churn.md`](../data-kb/02-metrics-and-churn.md) — this blueprint is the **presentation + scoring layer**, not a redefinition of the metrics. You need it once you have recurring revenue and want to act on retention rather than just report it.

## Data model

Read-heavy: mostly derived/aggregated from source events (subscriptions, usage, payments). Snapshot tables are tenant-scoped with standard audit columns. Definitions come from the metrics layer; do not re-derive them ad hoc.

### CustomerMetricSnapshot
Daily/period snapshot per customer — the grain everything rolls up from.

| Field | Type | Notes |
| --- | --- | --- |
| `customer_id` | fk | |
| `snapshot_date` | date | Grain |
| `segment` | enum | `trial` \| `paid` \| `unpaid` \| `churned` |
| `mrr_amount` | int | Minor units, normalized to monthly |
| `arr_amount` | int | |
| `plan_key` | string | Current plan |
| `active_users` | int | Distinct active in period |
| `usage_score` | numeric | Normalized engagement 0–1 |
| `days_since_signup` | int | |
| `is_activated` | bool | Hit activation milestone |
| `payment_status` | enum | `current` \| `past_due` \| `failed` |

### RevenueMetric (aggregate)

| Field | Type | Notes |
| --- | --- | --- |
| `period` | date | Month grain |
| `mrr` / `arr` | int | |
| `new_mrr` / `expansion_mrr` / `contraction_mrr` / `churned_mrr` | int | MRR movement components |
| `logo_count` | int | Active customers |
| `logo_churn` | int | Customers lost |
| `revenue_churn_rate` | numeric | |
| `logo_churn_rate` | numeric | |
| `net_revenue_retention` | numeric | NRR |

### CohortRetention

| Field | Type | Notes |
| --- | --- | --- |
| `cohort_month` | date | Signup cohort |
| `period_offset` | int | Months since signup (0..n) |
| `retained_count` | int | Still active |
| `retained_pct` | numeric | |
| `retained_revenue` | int | Revenue retention variant |

### ChurnRiskScore

| Field | Type | Notes |
| --- | --- | --- |
| `customer_id` | fk | |
| `score` | numeric | 0–1, higher = more at risk |
| `risk_band` | enum | `low` \| `medium` \| `high` |
| `top_factors_json` | jsonb | Explainability: contributing signals |
| `computed_at` | timestamp | |
| `model_version` | string | Reproducibility |

Signals feeding the score (examples): declining `usage_score`, drop in `active_users`, `past_due` payment, support tickets, no activation, approaching renewal with low engagement.

## Key screens & UX

1. **Overview dashboard** — headline tiles (MRR, ARR, NRR, logo churn, revenue churn, DAU/MAU, active trials), MRR movement waterfall (new/expansion/contraction/churn), trend charts. Global date-range + segment filter. *Loading:* tile + chart skeletons. *Empty:* "No data for this range." *Error:* per-widget retry.
2. **Segments** — paid / trial / unpaid tabs with counts, MRR, conversion and churn per segment; filter by plan/geo/signup-date; drill into a customer list.
3. **Cohort retention** — triangle heatmap (cohort_month × period_offset), logo vs revenue retention toggle, hover for exact %; export.
4. **Churn drill-down** — churned customers over the range, churn reasons, revenue lost, "at-risk" leading indicators; segment by plan/tenure.
5. **At-risk / churn-risk** — customers ranked by `ChurnRiskScore`, risk band filter, top contributing factors per customer, and a recommended action; bulk export for CS outreach.
6. **Customer 360** — single-customer view: plan, MRR, lifecycle timeline, usage/engagement trend, payment status, activation state, risk score + factors, recent activity, LTV estimate. Deep-links to billing and activity feed.
7. **Unit economics** — LTV, CAC, LTV/CAC ratio, payback period, by cohort/channel where source data exists.

## Rules & logic

- [MUST] Use metric definitions from [`../data-kb/02-metrics-and-churn.md`](../data-kb/02-metrics-and-churn.md) verbatim (MRR normalization, churn formula, activation event) — no divergent local definitions.
- [MUST] All figures are **tenant-scoped** and respect the viewer's data-access permissions.
- [MUST] MRR is normalized to a monthly figure regardless of billing interval; annual plans divided by 12.
- [MUST] Distinguish **logo churn** (customers) from **revenue churn** (MRR); report both; NRR includes expansion/contraction.
- [MUST] Churn-risk scores are **explainable** (`top_factors_json`) and versioned (`model_version`); never present a bare score with no reason.
- [MUST] Dashboards read from snapshots/aggregates, not live full-table scans — see [`../standards-kb/16-finops-cost.md`](../standards-kb/16-finops-cost.md).
- [SHOULD] Snapshots refresh on a schedule; show "data as of" timestamp so users trust freshness.
- [SHOULD] Every dashboard filter is shareable via URL state; exports are permission-checked and audited.
- [SHOULD] Cohorts support both logo and revenue retention; overlay a benchmark line where available.
- [SHOULD] At-risk list feeds CS workflows (export / webhook) and links to Customer 360.

## Applicable standards

- Metric + churn definitions (source of truth) — [`../data-kb/02-metrics-and-churn.md`](../data-kb/02-metrics-and-churn.md)
- Analytics/semantic layer for the underlying fields — [`../data-kb/01-analytics-data-layer.md`](../data-kb/01-analytics-data-layer.md)
- Query cost / snapshot strategy — [`../standards-kb/16-finops-cost.md`](../standards-kb/16-finops-cost.md)
- Data management, aggregation, freshness — [`../standards-kb/06-data-management.md`](../standards-kb/06-data-management.md)
- Tenant isolation & access control — [`../standards-kb/01-architecture-multitenancy.md`](../standards-kb/01-architecture-multitenancy.md), [`../standards-kb/04-security.md`](../standards-kb/04-security.md)
- Dashboard UX/states — [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md)
- Model/metric observability — [`../standards-kb/07-observability-aiops.md`](../standards-kb/07-observability-aiops.md)

## Acceptance checklist

- [ ] MRR/ARR, NRR, logo + revenue churn match the governed metric definitions.
- [ ] Segment views (paid/trial/unpaid) with working filters and drill-through.
- [ ] Cohort retention heatmap supports logo and revenue variants.
- [ ] Churn drill-down shows lost customers, revenue lost, and reasons.
- [ ] At-risk list ranks by explainable, versioned risk score with top factors.
- [ ] Customer 360 assembles plan, MRR, usage, payment, risk, activity, LTV.
- [ ] All figures tenant-scoped and permission-aware; exports audited.
- [ ] Dashboards read from snapshots/aggregates with a visible "data as of" timestamp.
