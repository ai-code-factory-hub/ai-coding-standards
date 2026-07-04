# 02 · Metrics & Churn

**Purpose.** Canonical, unambiguous definitions of the SaaS metrics that report builders and churn dashboards expose — and how to compute each one correctly. Each metric here should be implemented **once**, as a measure in the semantic layer ([01-analytics-data-layer.md](01-analytics-data-layer.md)), so every surface returns the same number. Definitions are project-agnostic; adapt currency/period conventions per product.

## Ground rules

- **[MUST]** Define **one** authoritative formula per metric; all dashboards, exports, and the board deck read that measure. No local recomputation. See the Engineering Doctrine on single-source-of-truth: [../engineering-doctrine/02-anti-duplication-and-refactoring.md](../engineering-doctrine/02-anti-duplication-and-refactoring.md).
- **[MUST]** Fix and document each metric's **grain** (per customer / per account / per tenant), **period** (calendar month vs rolling 30 days), and **currency normalization** (annual plans → monthly, multi-currency → reporting currency at a stated FX policy).
- **[MUST]** State the **population** every metric is computed over — see segmentation below. A churn number without a segment is meaningless.

## Revenue metrics

- **MRR (Monthly Recurring Revenue)** — normalized recurring subscription revenue per month. **[MUST]** Normalize annual/quarterly plans to a monthly figure (`annual_price / 12`); **exclude** one-time fees, usage overages you don't treat as recurring, and taxes. Decompose the monthly change into **new / expansion / contraction / churned / reactivation** MRR — this bridge is what dashboards chart.
- **ARR (Annual Recurring Revenue)** — `MRR × 12`. A view, not a separate source of truth.
- **ARPU / ARPA** — Average Revenue Per User (or Account) = `MRR / active_customers` for the chosen population. **[MUST]** State whether the denominator is paying customers only.

## Churn & retention

Define the window and the denominator explicitly; churn is the most misreported SaaS metric.

- **Logo (customer) churn** = `customers_lost_in_period / customers_at_start_of_period`. Counts **accounts**, ignores their size.
- **Revenue churn (gross)** = `MRR_lost_from_churn_and_contraction / MRR_at_start_of_period`. **Gross** never nets in expansion — it is the pure leak.
- **Net revenue churn** = `(churned_MRR + contraction_MRR − expansion_MRR) / starting_MRR`. Can be **negative** (good) when expansion outpaces losses.
- **Net Revenue Retention (NRR / NDR)** = `(starting_MRR + expansion − contraction − churn) / starting_MRR`. **[MUST]** Compute over a **fixed starting cohort** — new customers acquired mid-period are **excluded** from both numerator and denominator. > 100% means the existing base grows on its own.
- **Gross Revenue Retention (GRR)** = `(starting_MRR − contraction − churn) / starting_MRR`. Never exceeds 100%.
- **[MUST]** Distinguish **logo vs revenue** churn everywhere — losing many tiny accounts (high logo churn) can coincide with low revenue churn, and vice versa. Show both.
- **[SHOULD]** Report churn **net of reactivations** consistently, and decide **voluntary vs involuntary** (failed-payment/dunning) churn as separate dimensions.

> **Example A — gross vs net revenue churn (same month).**
> Start MRR ₹100,000. Lost ₹8,000 to cancellations, ₹2,000 to downgrades, gained ₹6,000 from upgrades.
> `Gross revenue churn = (8,000 + 2,000) / 100,000 = 10%`
> `Net revenue churn = (8,000 + 2,000 − 6,000) / 100,000 = 4%`
> Reporting only the 4% hides that 10% of the base is actually leaking.

## Cohorts & retention curves

- **[MUST]** Build **retention/cohort** analysis by grouping customers on their **acquisition (or activation) period**, then measuring the share still active (logo) or the retained MRR (revenue) at month N. Present as a triangle/heatmap.
- **[SHOULD]** Prefer **revenue-weighted** cohorts for expansion stories and **logo** cohorts for product-market-fit/stickiness; label which one a chart shows.

## Engagement & activation

- **Activation** — the share of new signups reaching a defined **"aha" milestone** within a set window (e.g. "completed first report within 7 days"). **[MUST]** Pin the milestone event and window; activation is meaningless without both.
- **DAU / WAU / MAU** — distinct active users per day/week/month. **[MUST]** Define **"active"** as a specific meaningful event (not merely a page load), and choose per-user vs per-account grain.
- **Stickiness** = `DAU / MAU` (roughly, days-per-month a user shows up). **[SHOULD]** Track alongside retention, not instead of it.

## Unit economics

- **LTV (Lifetime Value)** — for a segment: `ARPA × gross_margin% / revenue_churn_rate` (a steady-state approximation), or sum of discounted retained margin over the cohort's life. **[MUST]** State which method and always apply **gross margin**, not revenue.
- **CAC (Customer Acquisition Cost)** = `fully_loaded_sales_and_marketing_spend / new_customers_acquired` in the same period. **[MUST]** Include tooling and headcount, not just ad spend.
- **LTV:CAC** and **CAC payback** (`CAC / (ARPA × gross_margin%)`, in months) — **[SHOULD]** report both; a healthy ratio with a 24-month payback still strains cash.

## Segmentation & filter dimensions

Every metric above **[MUST]** be sliceable by population and by standard dimensions — this is where the semantic layer's dimensions earn their keep.

- **Billing status (population):** **paid** (active paying), **trial** (in trial, not yet paying), **free** (free tier / freemium), **past-due/dunning**, **churned**. **[MUST]** Never mix trial/free into revenue-metric denominators unless explicitly reporting a blended figure.
- **Common filter dimensions:** plan/tier, billing period (monthly vs annual), segment (SMB/mid/enterprise), region/geo, acquisition channel, cohort month, industry/vertical, voluntary vs involuntary.

> **Example B — segmentation changes the story.**
> Blended monthly logo churn = 5%. Split by segment: enterprise = 1%, SMB = 9%. The blended number hides that the enterprise base is healthy and SMB is the problem — the dashboard **[MUST]** allow the split.

## Acceptance checklist

- [ ] Each metric has one canonical formula implemented once as a semantic-layer measure; no surface recomputes locally.
- [ ] Grain, period, currency normalization, and population are documented per metric.
- [ ] Logo vs revenue churn, and gross vs net, are shown distinctly; NRR/retention use a fixed starting cohort.
- [ ] Cohort/retention analysis labels logo vs revenue weighting; activation pins a milestone event and window.
- [ ] "Active" for DAU/WAU/MAU is a defined meaningful event; LTV uses gross margin and states its method; CAC is fully loaded.
- [ ] Every metric is sliceable by billing status (paid/trial/free/past-due/churned) and the standard filter dimensions.
