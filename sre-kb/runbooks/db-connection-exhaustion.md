# Runbook · Database Connection Exhaustion

> Fires when the app can no longer acquire DB connections — the pool is saturated or the server hit its `max_connections` ceiling.

Template: [../runbook-template.md](../runbook-template.md) · Index: [../README.md](../README.md)

## Alert / Symptoms
- Pool checkout wait time > 1s, or checkout timeouts (`pool timeout`, `connection acquire timeout`).
- DB server errors: `FATAL: sorry, too many clients already` / `remaining connection slots are reserved`.
- Active connections at or near `max_connections`; idle-in-transaction count climbing.
- Request latency spikes and 500s that correlate with a flat connection ceiling on the DB dashboard.

## Impact
- **Users:** requests hang then fail; writes and reads both affected; often site-wide.
- **Severity:** SEV-2 by default; escalate to **SEV-1** if primary DB is unreachable to all app instances or a tenant-wide outage.

## Quick checks
1. Current vs max connections: `SELECT count(*) FROM pg_stat_activity;` vs `SHOW max_connections;`.
2. Connections by state: `SELECT state, count(*) FROM pg_stat_activity GROUP BY state;` — watch `idle in transaction`.
3. Longest-running queries/txns (see Diagnosis) — is one query holding many connections?
4. Did anything change? Recent deploy, traffic surge, or a batch/cron job kicking off.
5. Pool config per instance × instance count vs DB ceiling — do the math; you may simply be over-provisioned.

## Diagnosis
1. **Find long transactions / leaks:**
   ```sql
   SELECT pid, state, now()-xact_start AS txn_age, now()-query_start AS query_age, left(query,120)
   FROM pg_stat_activity
   WHERE state <> 'idle'
   ORDER BY xact_start ASC NULLS LAST LIMIT 20;
   ```
   `idle in transaction` with old `xact_start` = a **connection leak** (transaction opened, never committed/closed).
2. **Attribute to a source:** group by `application_name`, `usename`, client addr to find the offending service/instance.
3. **Pool math:** `pool_size × app_instances (+ overhead)` must stay under `max_connections` minus reserved slots. If it exceeds, the pool config is the root cause.
4. **Slow queries starving the pool:** a few slow queries (missing index, lock contention) hold connections longer, so effective capacity drops. Cross-check with [high-latency-p99](./high-latency-p99.md).
5. **Traffic vs capacity:** if connections are healthy-but-full under legitimate load, this is a scaling problem, not a leak.

## Mitigation
_Safe / reversible first._
1. **Kill idle-in-transaction offenders** (reversible, targeted):
   ```sql
   SELECT pg_terminate_backend(pid) FROM pg_stat_activity
   WHERE state = 'idle in transaction' AND now()-xact_start > interval '5 min';
   ```
2. **Pause the batch/cron job** that spawned the surge; let the pool drain.
3. **Roll back the recent deploy** if the leak appeared right after it (see [failed-deploy-and-rollback](./failed-deploy-and-rollback.md)).
4. **Front the DB with a transaction-mode pooler (PgBouncer)** so hundreds of app connections multiplex onto a small server pool — the durable fix for connection fan-out.
5. **Offload reads to a read replica** to cut primary connection pressure.
6. **Right-size the pool** (lower per-instance `pool_size`, or add a global cap) so total demand fits under `max_connections`. Restart/rolling-deploy to apply.
7. **Last resort — raise `max_connections`:** costs memory per connection and can destabilize the DB; prefer a pooler. Requires DB restart on most engines.

## Escalation
- Page the **DBA / data-platform on-call** if killing offenders doesn't recover the pool within ~10 min, or if the primary is unreachable.
- Involve the **owning service team** for the leaking service (identified via `application_name`).
- Declare a SEV and open an incident channel if user-facing outage exceeds your SLO error budget alert.

## Prevention / follow-up
- Enforce short transactions and always-close/`with`-scoped connections; add a leak detector (log/kill idle-in-transaction beyond a threshold).
- Adopt a pooler and read-replica strategy as standard.
- Add a pre-deploy check that validates `pool_size × instances < max_connections`.
- **Ties to:** [../../standards-kb/03-performance-scalability.md](../../standards-kb/03-performance-scalability.md) and [../../performance-kb/01-pooling-and-resources.md](../../performance-kb/01-pooling-and-resources.md).
