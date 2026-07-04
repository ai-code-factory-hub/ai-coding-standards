# 03 · Idempotent Consumers

Purpose: make consumers safe under at-least-once delivery — the same message delivered twice (or five times) must produce the same result as delivering it once. Idempotency keys, the dedup store, retries with backoff + jitter, poison messages, and dead-letter queues with redrive. This is the messaging build-out of the idempotency and retry rules in [../standards-kb/05-reliability-resilience.md](../standards-kb/05-reliability-resilience.md), paired with the inbox pattern in [02](02-outbox-and-inbox.md).

## Why this is not optional

From [01](01-messaging-fundamentals.md): delivery is **at-least-once**, "exactly-once delivery" is a myth, and the honest guarantee you build is **effectively-once processing = at-least-once delivery + idempotent consumers**. Retries, ack timeouts, consumer crashes after doing work but before acking, and partition rebalances **will** redeliver messages in normal operation. A non-idempotent consumer under at-least-once delivery is a correctness bug waiting for load.

- **[MUST]** Every consumer of an at-least-once stream is idempotent. No exceptions for "it rarely duplicates."

## Idempotency keys

An **idempotency key** is a stable identifier for the *unit of effect* that lets the consumer recognize "I've already done this."

- **[MUST]** Derive the key from something **stable across redeliveries** — the outbox/message `event.id` ([02](02-outbox-and-inbox.md)), or a domain-natural key (`paymentId`, `orderId + version`). Never derive it from arrival time, a random per-attempt value, or the broker's delivery count.
- **[SHOULD]** Prefer a **business-meaningful** key when the effect is naturally unique (one charge per `paymentId`) — it also dedups across *different* producers of the same intent, not just retries of one message.
- **[MUST]** Scope the key to a **tenant** in multi-tenant systems (`{tenantId}:{eventId}`) so keys never collide or leak across tenants ([../standards-kb/01-architecture-multitenancy.md](../standards-kb/01-architecture-multitenancy.md)).

## The dedup store

Where you remember "already processed." Three implementation shapes, strongest first:

1. **Same-transaction dedup (inbox):** insert the key with a **unique constraint** and apply the effect in the same DB transaction ([02](02-outbox-and-inbox.md)). Strongest — dedup and effect commit atomically.
2. **Natural idempotency:** design the write so re-applying is a no-op — `UPSERT ... ON CONFLICT DO NOTHING/UPDATE`, `SET status='PAID'` (absolute, not `+=`), `INSERT ... IF NOT EXISTS`. No separate store needed.
3. **External dedup store (cache/KV):** record processed keys in Redis/DynamoDB with a TTL. **Weakest** — the store and the effect are two systems (a mini dual-write, [02](02-outbox-and-inbox.md)); a crash between "mark done" and "do work" can drop the effect, or vice-versa. Use only when the effect is in a system with no transaction to join.

```
on message m:
  key = tenantId + ":" + m.event.id
  BEGIN tx
     INSERT INTO processed(key) VALUES(key)      -- unique
        ON CONFLICT DO NOTHING
     IF rows_affected == 0:                        -- seen before
        COMMIT; ack(m); return
     applyEffect(m)                                -- first time only
  COMMIT
  ack(m)                                           -- only after commit
```

- **[MUST]** Ack the message **only after** the effect (and dedup record) is durably committed. Acking first turns at-least-once into at-most-once (lost effects on crash).
- **[MUST]** Prefer options 1 or 2 over 3. If you must use an external store, size the **TTL ≥ the broker's max redelivery/retention window** so a late redelivery still finds the key.
- **[SHOULD]** For non-idempotent side effects you don't own (send email, call an external API), pass a provider **idempotency key** if supported; otherwise record "sent" in your own store *before* the call and treat a duplicate as best-effort-suppressed — accept that some externally-visible duplicates are unavoidable and design the UX to tolerate them.

## Retries with exponential backoff + jitter

When processing fails transiently (downstream 503, lock timeout), retry — but only safely.

- **[MUST]** Retry with **exponential backoff + jitter**, not fixed-interval retries (which synchronize into thundering herds). Matches [../standards-kb/05-reliability-resilience.md](../standards-kb/05-reliability-resilience.md).

```
attempt n:  delay = min(cap, base * 2^n)
            sleep(random_between(0, delay))     # full jitter
   e.g. base=1s cap=60s → ~1s, ~2s, ~4s, ~8s ... (randomized)
```

- **[MUST]** Only auto-retry **idempotent** operations. Since your consumer is idempotent (above), retrying the whole message is safe. If a step has a non-idempotent external effect without a dedup key, guard it explicitly before retrying.
- **[MUST]** Cap total attempts / total elapsed time. Infinite in-line retries block the partition and become a self-inflicted backlog ([06](06-eventing-operations.md)). After the cap → dead-letter (below).
- **[SHOULD]** Distinguish **transient** (retry: timeouts, 429/503, deadlocks) from **permanent** (do not retry, dead-letter immediately: validation failure, deserialization error, 400, business rejection). Retrying a permanent failure just burns the retry budget.
- **[SHOULD]** Use a **circuit breaker** around a flaky downstream so a mass-failing dependency doesn't drive every message into its full retry ladder ([../standards-kb/05-reliability-resilience.md](../standards-kb/05-reliability-resilience.md)); shed/park load until it recovers.

## Poison messages

A **poison message** fails every time — malformed payload, unsupported schema version, a referenced entity that will never exist, a bug in the handler. Left in place on an ordered partition, it **blocks every message behind it** (head-of-line blocking) and can spin retries forever.

- **[MUST]** Bound retries per message; after the cap, **remove it from the main flow** (dead-letter it) so the partition drains. One bad message must never halt a tenant's whole stream.
- **[MUST]** Detect and short-circuit obviously-permanent failures (deserialization/validation) on attempt 1 — don't spend the retry ladder on them.
- **[SHOULD]** Emit a structured, correlatable log/metric on every dead-letter with the reason and the full envelope ([../standards-kb/20-logging-audit-and-traceability.md](../standards-kb/20-logging-audit-and-traceability.md)) so on-call can triage without spelunking.

## Dead-letter queues (DLQ) + redrive

A **DLQ** is a separate queue/topic that holds messages the consumer could not process after exhausting retries. It's a quarantine, not a graveyard.

```
main queue ──► consumer ──(fail × maxAttempts)──► [ DLQ ]
                  ▲                                   │
                  └────────── redrive ◄───────────────┘
                     (after fixing bug / dependency,
                      replay DLQ back to main queue)
```

- **[MUST]** Every at-least-once consumer has a DLQ (or equivalent parking store). A message that can't be processed goes to the DLQ **with its full envelope and failure context** (error, attempt count, first-seen time) — never silently dropped.
- **[MUST]** **DLQ depth is monitored and alerted** — a non-zero, growing DLQ is an incident signal ([06](06-eventing-operations.md), and the [../sre-kb/runbooks/queue-backlog-and-dlq.md](../sre-kb/runbooks/queue-backlog-and-dlq.md) runbook).
- **[MUST]** Provide **redrive**: after fixing the bug or waiting out the dependency, replay DLQ messages back to the main queue. Because consumers are idempotent, replaying a message already partially processed is safe. Operators drive this from [../admin-kb/job-and-queue-monitor.md](../admin-kb/job-and-queue-monitor.md).
- **[SHOULD]** Redrive in **controlled batches** with backoff, not all at once — a full DLQ flushed instantly re-creates the backlog that filled it.
- **[SHOULD]** Keep a DLQ **retention/quota** and a triage SLA; a DLQ nobody drains is just delayed data loss.
- **[MAY]** Route different failure classes to different DLQs (poison vs downstream-outage) so redrive strategy can differ.

## Worked examples

> **Example (fintech/e-commerce) — idempotent charge.** The payments consumer handles `ChargePayment` (at-least-once). Key = `paymentId`. In one DB tx it inserts `payments(paymentId)` with a unique constraint and, only if newly inserted, calls the PSP with `paymentId` as the provider idempotency key; then commits and acks. A redelivery finds the row → no PSP call → acks. A PSP timeout is transient → exponential backoff + jitter, up to 5 attempts; a `card_declined` is permanent → straight to DLQ. On-call sees DLQ depth rise, finds a batch declined by an expired BIN rule, fixes the rule, and **redrives** in batches — idempotency guarantees no double charge on replay.

> **Example (healthcare) — result ingestion.** An analyzer-integration consumer ingests `LabResultFinalized`. A message referencing a specimen that was purged (permanent) is detected on attempt 1 and dead-lettered immediately with full context, so it doesn't block the patient's ordered partition. A transient DB deadlock is retried with backoff. Ingestion is naturally idempotent via `UPSERT` on `resultId`, so a redelivery after a consumer crash re-upserts the same values — no duplicate result, no duplicate "results ready" alert. DLQ depth feeds the queue-backlog runbook; a jammed analyzer feed shows as a poison-message cluster with one shared error signature.

## Acceptance checklist

- [ ] Every at-least-once consumer is idempotent; dedup uses a stable key (event id or domain key), tenant-scoped.
- [ ] Dedup + effect commit atomically (inbox/natural idempotency) where possible; external stores TTL ≥ redelivery window.
- [ ] Messages are acked only after the effect is durably committed.
- [ ] Retries use exponential backoff + jitter, are capped, and only wrap idempotent work; permanent failures skip the ladder.
- [ ] A circuit breaker protects flaky downstreams from driving mass full-retry ladders.
- [ ] Poison messages are bounded and can't block a partition/tenant.
- [ ] Every consumer has a monitored, alerted DLQ with full failure context and a controlled, batched redrive path.
