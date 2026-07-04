# 01 · SLOs & Error Budgets

**Purpose:** make "how reliable is reliable enough?" a number the whole team agrees on, and
turn that number into a lever that governs how fast you ship. This operationalises the SLOs
required by [05 · Reliability & Resilience](../standards-kb/05-reliability-resilience.md) and
consumes the RED/USE metrics defined in
[07 · Observability & AIOps](../standards-kb/07-observability-aiops.md) — it does not
redefine either.

## SLI vs SLO vs SLA

Three related-but-distinct things people constantly conflate:

- **SLI — Service Level *Indicator*** — a *measured* number: the ratio of good events to
  total. e.g. `successful requests / total requests`, or `requests under 300ms / total`.
- **SLO — Service Level *Objective*** — the *target* for an SLI over a window, set by you.
  e.g. "99.9% of checkout requests succeed over 28 days." Internal, and the thing on-call
  defends.
- **SLA — Service Level *Agreement*** — a *contractual* promise to customers with
  consequences (credits, penalties) if missed. e.g. "99.5% monthly uptime or a service
  credit."

- **[MUST]** Set the **SLO stricter than the SLA** — the SLA is a legal floor; the SLO is
  the engineering target that keeps you comfortably above it. (Miss the SLO → warning; miss
  the SLA → refund.)
- **[SHOULD]** Have SLOs long before you have SLAs. You cannot promise externally what you
  don't measure internally.

## Choosing SLIs from the user's perspective

Measure what the **user experiences**, not what is easy to graph. CPU at 90% is not an
incident if requests are fast and succeeding; CPU at 30% while checkout 500s **is**.

- **[MUST]** Define SLIs on **user-facing journeys** (login, create order, record entry,
  publish output), measured as close to the user as practical (load balancer, synthetic
  probe, or client), not deep internal counters.
- **[SHOULD]** Cover the four common SLI shapes where they apply:
  **Availability** (success ratio), **Latency** (fraction under a threshold),
  **Correctness/Quality** (results right, not just returned), **Freshness** (data/jobs
  current within X).
- **[SHOULD]** Slice **per tenant** so one large tenant's pain isn't hidden in a healthy
  global average.
- **[MUST NOT]** Set an SLI you can't measure — an unmeasurable objective is a wish.

> **Example (e-commerce):** SLI = `checkout requests completing < 1s and returning 2xx /
> all checkout requests`, measured at the edge — captures both "did it work" and "was it
> fast" in one user-centric number.

## Error budgets

An SLO of 99.9% is permission to be **unavailable 0.1% of the time**. That 0.1% is your
**error budget** — a finite, spendable resource.

- Budget = `1 − SLO`. 99.9% over 28 days ≈ **~40 min** of allowed downtime; 99.95% ≈ ~20 min.
- **[MUST]** Treat the budget as **shared currency** between reliability and velocity:
  budget remaining → ship freely and take risks; budget spent → slow down and stabilise.
- **[SHOULD]** Spend budget **deliberately** — a risky migration or a planned load test that
  consumes budget on purpose is fine; unplanned burn from flaky deploys is the problem.

## Burn-rate alerting

Alert on **how fast you are spending the budget**, not on every blip. Burn rate = how many
times faster than "sustainable" you are consuming budget (1× burn exactly exhausts the
budget at window's end).

- **[MUST]** Use **multi-window, multi-burn-rate** alerts to balance fast detection against
  noise:
  - **Fast burn** (e.g. **14.4×** over 1h, confirmed by a 5m window) → **page** — this
    exhausts a month's budget in ~2 days.
  - **Slow burn** (e.g. **3×** over 6h) → **ticket** — real but not drop-everything.
- **[SHOULD]** Page only on **symptom/burn-rate** alerts tied to an SLO (per Domain 07),
  never on raw resource thresholds that may not affect users.
- **[SHOULD NOT]** Alert on every threshold crossing — that is how you get alert fatigue
  (see [02 · On-Call & Escalation](02-oncall-and-escalation.md)).

> **Example (fintech):** payment-success SLO 99.95%. A vendor hiccup pushes failures to
> ~2% for 20 min → the 1h fast-burn alert pages on-call well before the monthly budget is
> gone, while a chronic 0.1% elevation only opens a ticket.

## When the budget is exhausted

The budget existing without a **policy** attached to it is theatre. Agree the policy *before*
you need it, in writing.

- **[MUST]** On exhaustion, trigger a pre-agreed **feature freeze**: non-essential feature
  work stops and the team's default priority shifts to reliability (fix flaky deploys,
  add tests, harden the top burn sources) until the budget recovers.
- **[MUST]** Only **reliability, security, and incident** changes ship during a freeze; the
  freeze lifts when the trailing SLO is back in budget.
- **[SHOULD]** Escalate **repeated** exhaustion to engineering leadership — a chronically
  blown budget means the SLO is wrong *or* the system needs real investment, not another
  freeze. Reliability spend is a [FinOps](../standards-kb/16-finops-cost.md) trade-off:
  buying more nines (standby capacity, multi-region) costs money — the budget makes that
  conversation explicit.
- **[MAY]** Tighten or loosen the SLO if data shows users don't notice the difference —
  don't burn engineering effort defending nines nobody perceives.

## Example SLO targets (starting points — tune per journey)

| Journey type | SLI | Example SLO | Window |
|---|---|---|---|
| Critical user flow (login, checkout, record entry) | success ratio | 99.9% | 28 days |
| Same, latency | fraction < 1s | 99% < 1s | 28 days |
| Read/dashboard | success ratio | 99.5% | 28 days |
| Async job / data freshness | processed < 5 min | 99% | 28 days |
| Internal/back-office tool | availability | 99% | 28 days |

Tiers of criticality get different nines — **[SHOULD NOT]** demand 99.99% of everything;
each nine is exponentially more expensive and most journeys don't need it.

## SLO checklist

- [ ] Each critical journey has a user-perspective SLI you actually measure.
- [ ] SLOs set stricter than any SLA; per-tenant slices exist for key journeys.
- [ ] Error budget derived from each SLO and visible on a dashboard.
- [ ] Multi-window burn-rate alerts: fast-burn pages, slow-burn tickets.
- [ ] A written budget-exhaustion policy (feature freeze) is agreed **before** it triggers.
- [ ] Repeated exhaustion escalates to a fix-the-SLO-or-invest decision, not just another freeze.
