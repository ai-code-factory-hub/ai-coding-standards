# 16 · FinOps & Cloud Cost Management

Make cloud and third-party spend attributable, bounded, and part of every design decision, so unit economics stay ahead of revenue.

## Cost attribution

- **[MUST]** Tag every resource (owner / environment / service / tenant) so all spend is attributable. No untagged resource ships to production.
- **[MUST]** Track **cost per tenant** across compute, storage, bandwidth, and **metered third-party usage** (per-message notifications, model/inference tokens, external API calls) — you cannot know unit economics versus subscription revenue without it.
- **[SHOULD]** Surface per-tenant cost alongside per-tenant revenue in an internal dashboard so unprofitable tenants and plans are visible early.

## Budgets & guardrails

- **[MUST]** Set budgets with spike/anomaly alerts per environment and per service, so a runaway job or a metered-usage explosion is caught **before** the invoice arrives, not after.
- **[MUST]** Put **hard guardrails on every unbounded-cost path**: caps on metered third-party usage (token/message quotas), job-concurrency limits, storage quotas, and egress limits. An unbounded loop or a compromised credential must not be able to run up an unbounded bill.
- **[MUST]** Enforce guardrails **server-side and per tenant**, so one tenant's overrun cannot exhaust shared budget or degrade others (ties to the quota model in [01 · Architecture & Multi-Tenancy](01-architecture-multitenancy.md) and rate limits in [05 · Reliability & Resilience](05-reliability-resilience.md)).

## Efficiency

- **[MUST]** Right-size compute and databases to actual utilization; continuously identify and remove idle/orphaned resources (unattached volumes, dormant instances, forgotten environments).
- **[SHOULD]** Scale non-production environments to zero off-hours; use committed-use/reserved capacity for steady baseline and spot/preemptible capacity for interruptible batch work.
- **[SHOULD]** Tier storage and logs by access frequency with retention policies; compress cold data.
- **[SHOULD]** Cache deterministic results of metered third-party calls and apply per-tenant limits to reduce repeated spend; monitor egress and per-unit third-party costs (per message, per token, per request) as first-class metrics (see [07 · Observability & AIOps](07-observability-aiops.md)).

## Cost in the design loop

- **[MUST]** Include a cost consideration in design/architecture reviews for any feature that adds compute, storage, egress, or metered third-party usage. Cost is a non-functional requirement, not an afterthought.

### Examples

- **Healthcare:** a diagnostic-report PDF pipeline caches rendered artifacts and caps AI-narration tokens per tenant per day; a per-tenant cost dashboard shows storage-heavy imaging tenants sit above their plan margin, triggering a plan review.
- **Fintech / e-commerce:** every transaction-notification send (SMS/email/push) is metered per tenant with a monthly ceiling and anomaly alert; a misconfigured retry storm trips the concurrency guardrail instead of generating a five-figure messaging bill.

## Acceptance checklist

- [ ] Every resource tagged (owner/env/service/tenant); cost tracked per tenant including metered third-party usage.
- [ ] Budgets + spike/anomaly alerts per env/service.
- [ ] Hard guardrails on every unbounded-cost path (token/message/concurrency/storage/egress), enforced server-side per tenant.
- [ ] Compute/DB right-sized; idle/orphaned resources removed; storage/log tiering in place.
- [ ] Metered-usage cost controlled via caching + per-tenant limits; egress and per-unit costs monitored.
- [ ] Cost reviewed as part of design/architecture review.
