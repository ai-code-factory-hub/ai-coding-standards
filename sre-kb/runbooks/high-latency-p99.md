# Runbook · High p99 Latency

> Fires when tail latency (p99/p95) crosses SLO while p50 may look normal — a subset of requests is slow.

Template: [../runbook-template.md](../runbook-template.md) · Index: [../README.md](../README.md)

## Alert / Symptoms
- p99 (or p95) request latency > SLO threshold for N minutes; p50 often still healthy.
- Latency histogram develops a long tail / bimodal shape.
- Apdex drops; timeouts and retries increase downstream.
- Often scoped to one endpoint, one tenant, or one dependency.

## Impact
- **Users:** intermittent slowness — some requests fast, some very slow; timeouts on heavy pages/exports.
- **Severity:** SEV-3 if contained to a low-traffic endpoint; **SEV-2** if a core flow or many tenants are affected or timeouts cascade.

## Quick checks
1. **Scope it:** break p99 down by endpoint, tenant, region, and instance. Is it one dimension or global?
2. **Correlate with a change:** recent deploy, feature-flag flip, data growth, or traffic surge.
3. **Downstream latency:** are DB / cache / third-party call spans the ones that grew? Check trace waterfalls.
4. **Resource saturation:** CPU, memory, GC pause time, connection pool wait, thread/worker saturation.
5. **Hot tenant / hot key:** is one tenant or one key generating outsized, expensive work?

## Diagnosis
1. **Pull exemplar traces** at p99 for the slow endpoint. Identify the dominant span: DB, external call, serialization, or in-process compute.
2. **Slow queries & N+1:** check the DB slow-query log and per-request query counts. A loop issuing one query per row (N+1) is the classic tail-latency cause.
3. **Missing/!used index:** `EXPLAIN (ANALYZE, BUFFERS)` the slow query; look for seq scans on large tables, bad row estimates, or sorts spilling to disk.
4. **Downstream dependency:** if the tail lives in an external call span, jump to [third-party-dependency-outage](./third-party-dependency-outage.md).
5. **GC / runtime pauses:** correlate p99 spikes with GC pause metrics or major-collection frequency; a memory leak or under-sized heap produces periodic tail spikes.
6. **Hot tenant / cardinality:** group latency by tenant; a tenant with 100x the data can dominate the tail on shared endpoints.
7. **Lock/contention:** check DB lock waits and app-level mutex contention for bimodal latency.

## Mitigation
_Safe / reversible first._
1. **Add/warm a cache** for the hot read path (reversible; validate correctness of TTL).
2. **Flip off the offending feature flag** or recent code path if the tail started with a deploy/flag.
3. **Rate-limit or throttle the hot tenant/key** to protect shared capacity.
4. **Add the missing index** (use online/concurrent index build to avoid locking) or fix the N+1 with eager loading / batch fetch.
5. **Raise timeouts/retries carefully** only as a stopgap — this hides the problem and can worsen load; prefer fixing the slow path.
6. **Scale out / up** the saturated tier (workers, DB replica, larger heap) if the cause is genuine capacity.
7. **Roll back** the deploy if the regression is code-introduced and no quick fix exists.

## Escalation
- Engage the **owning service team** for the slow endpoint.
- Page the **DBA** for query/index issues that need production DDL.
- Involve the **dependency owner** (internal or vendor) if the tail is in a downstream call.

## Prevention / follow-up
- Add p99 SLOs and alerts per critical endpoint; track query counts per request in CI to catch N+1 regressions.
- Load-test with representative (hot-tenant) data, not uniform data.
- Establish index review as part of migration review.
- **Ties to:** [../../performance-kb/](../../performance-kb/) — esp. [02-query-and-statement.md](../../performance-kb/02-query-and-statement.md) and [03-code-level-performance.md](../../performance-kb/03-code-level-performance.md); standard [../../standards-kb/03-performance-scalability.md](../../standards-kb/03-performance-scalability.md).
