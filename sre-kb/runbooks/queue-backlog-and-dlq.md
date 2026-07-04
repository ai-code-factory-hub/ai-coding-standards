# Runbook · Queue Backlog & Dead-Letter Queue

> Fires when a message/job queue backs up (consumer lag grows) or the dead-letter queue (DLQ) starts filling.

Template: [../runbook-template.md](../runbook-template.md) · Index: [../README.md](../README.md)

## Alert / Symptoms
- Queue depth / consumer lag growing monotonically; oldest-message age past SLA.
- DLQ message count > 0 and climbing.
- Consumer throughput drops to zero or well below produce rate.
- Downstream effects: delayed emails/webhooks/exports, stale derived data.

## Impact
- **Users:** delayed async outcomes (notifications, report generation, sync); if the queue drives user-visible state, it looks like "nothing happened."
- **Severity:** SEV-3 for a slow drain that will catch up; **SEV-2** if the backlog threatens data loss, breaches an SLA, or the DLQ is filling from a poison-message storm.

## Quick checks
1. **Consumers alive?** Are worker instances running, healthy, and connected to the broker? Check replica count and last heartbeat.
2. **Produce vs consume rate:** is this a demand spike or a stalled consumer?
3. **DLQ contents:** sample a few DLQ messages — same error signature = poison message; varied = downstream failure.
4. **Recent deploy** to producers or consumers (schema change, serialization change).
5. **Downstream dependency** the consumer calls — is it erroring/slow (which forces retries → backlog)?

## Diagnosis
1. **Consumer down/stuck:** check logs for crash loops, unhandled exceptions, or blocked threads. A crashed consumer = lag with zero consume rate; a slow one = lag with reduced rate.
2. **Poison message:** if one message repeatedly fails and blocks an ordered partition, throughput stalls. Inspect the DLQ / the stuck offset for a common error (deserialization failure, schema mismatch, bad payload).
3. **Downstream failure amplification:** consumer calls a failing dependency → messages retry → effective throughput collapses and retries pile up. Cross-check [third-party-dependency-outage](./third-party-dependency-outage.md).
4. **Capacity:** legitimate demand exceeds worker throughput — consume rate is healthy per-worker but too few workers.
5. **Ordering/partition skew:** one hot partition/key serializes work while others idle.

## Mitigation
_Safe / reversible first._
1. **Scale out consumers** (add worker replicas / raise concurrency) if the cause is capacity and messages are safe to process in parallel.
2. **Restart stuck consumers** to clear blocked threads / stale connections.
3. **Quarantine the poison message:** move it to the DLQ (or skip its offset) so the partition unblocks; capture the payload for offline analysis.
4. **Pause producers or shed low-priority traffic** to let consumers catch up if the backlog is runaway.
5. **Fix + redeploy the consumer** if a code bug (deploy regression) is dropping messages to DLQ; roll back if faster.
6. **Drain the DLQ** once the root cause is fixed: replay messages back to the main queue in controlled batches, monitoring for re-failure. Never blind-replay a DLQ into an unfixed consumer — it just re-poisons.
7. **Purge only truly unprocessable messages** after capturing them for audit — irreversible, do last and with sign-off.

## Escalation
- Page the **owning service team** for the consumer.
- Involve **platform/infra on-call** if the broker itself is unhealthy (partitions offline, broker down).
- Escalate to **data owner** before purging any DLQ messages that represent user data.

## Prevention / follow-up
- Set max-retry + DLQ policies; make consumers idempotent so replay is safe.
- Alert on consumer-lag age and DLQ depth, not just count.
- Add poison-message handling (validate/deserialize defensively; route bad payloads straight to DLQ without blocking).
- Autoscale consumers on lag.
- **Ties to:** [../../standards-kb/05-reliability-resilience.md](../../standards-kb/05-reliability-resilience.md).
