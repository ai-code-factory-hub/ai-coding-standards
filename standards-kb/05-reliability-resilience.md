# 05 · Reliability & Resilience

User-perspective SLOs, HA/DR, timeouts/retries/breakers/idempotency on every external call, load shedding, and offline sync.

## Service levels

- **[MUST]** Define **SLOs** (e.g. 99.9% monthly availability, p99 latency, error rate) measured from the user's perspective — synthetic checks of real flows (login, create order, record entry, publish output). Track error budgets and publish a status page.

## High availability

- **[MUST]** **HA:** ≥2 app instances across availability zones behind a load balancer; stateless app; DB replicas/failover; failover tested. **[SHOULD]** multi-AZ by default, multi-region where data residency permits.

## Disaster recovery

- **[MUST]** **DR:** document **RTO/RPO per data class**; automated, encrypted, **restore-tested** backups (a backup you have never restored is not a backup); offsite / separate-account; per-tenant restore/export; a rehearsed DR runbook; and PITR for the primary datastore.

## Fault tolerance

- **[MUST]** Every outbound call (DB, cache, queue, external integration, payment, messaging) has **timeouts**; **retries with exponential backoff + jitter** on idempotent operations only; **circuit breakers** around flaky dependencies (a down dependency must not cascade); and **idempotency keys** on creates/payments/external operations. **[SHOULD]** bulkheads isolate pools per dependency/tenant.
- **[MUST]** **Rate limiting** per tenant/user/IP (429 + Retry-After); **load shedding** protects core business flows (defer reports/exports under overload); non-critical failures (a dashboard widget) degrade locally.

> **Example (healthcare):** a down external health-records gateway trips a circuit breaker and the push is queued for retry — record entry and result review keep working; the outage degrades one integration, not the app.
>
> **Example (fintech/e-commerce):** under a traffic spike, load shedding defers analytics exports so checkout and payment capture stay within SLO; idempotency keys ensure a retried charge never double-bills.

## Offline & sync resilience

- **[MUST]** For field-worker / intermittently-connected apps: queue actions locally and sync on reconnect with **idempotency keys** (no duplicate records) and conflict resolution (last-write-wins is wrong for critical data — use versioning/merge). Surface sync status; never silently drop unsynced data. Capture offline errors with full context and auto-sync them for diagnosis.

## Health checks

- **[MUST]** Liveness + readiness endpoints gate traffic and restarts; readiness reflects dependency health (e.g. not ready if the DB pool is exhausted).

See [Performance & Scalability](03-performance-scalability.md) for the job queues these calls run on, [Security](04-security.md) for rate-limit enforcement, and [Versioning, Deployment & Migration](02-versioning-deploy-migration.md) for health-gated rollback.

## Acceptance checklist

- [ ] User-perspective SLOs defined with error budgets and a public status page.
- [ ] HA across ≥2 AZs with a stateless app and tested DB failover.
- [ ] RTO/RPO per data class; encrypted, offsite, **restore-tested** backups + PITR; rehearsed DR runbook; per-tenant restore.
- [ ] Every external call has timeout + backoff-with-jitter retry + circuit breaker + idempotency key.
- [ ] Rate limiting per tenant/user/IP; load shedding protects core flows; non-critical failures degrade locally.
- [ ] Offline apps queue locally and sync idempotently with real conflict resolution; no silent data loss.
- [ ] Liveness/readiness endpoints gate traffic and reflect dependency health.
