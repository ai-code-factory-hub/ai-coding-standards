# 02 · Outbox & Inbox

Purpose: reliably get an event *out* of a service and safely take it *in*, without losing or corrupting state when the database and the broker are two separate systems. The transactional outbox pattern, the inbox pattern for consumer-side dedup, and CDC as an alternative. This is the messaging expression of the idempotency rule in [../standards-kb/05-reliability-resilience.md](../standards-kb/05-reliability-resilience.md) and pairs with idempotent consumers in [03](03-idempotent-consumers.md).

## The dual-write problem

You almost always need to do two things when something happens: **change state** (write the DB) and **tell the world** (publish an event). These live in two systems with no shared transaction.

```
        BEGIN tx
          UPDATE orders SET status='PAID'   ✓ committed
        COMMIT
        broker.publish("OrderPaid")          ✗ CRASH here
        ───────────────────────────────────────────────
   Result: DB says PAID, but nobody was ever told. Inventory
   never reserved, receipt never sent. Silent inconsistency.
```

Any ordering of "write DB" and "publish" has a failure window:

- **DB then publish:** crash after commit → state changed, event lost. **Under-notified.**
- **Publish then DB:** crash after publish → event sent, state not changed. **Over-notified / ghost event.**
- **[MUST]** Never do a naive dual write for anything that matters. There is no ordering that is safe, because the two systems can't commit atomically together. Wrapping them in a try/catch does not fix it — the crash can land between the two operations.

## The transactional outbox pattern

Turn the dual write into a **single local transaction** by writing the event into an **outbox table in the same database**, then relaying it to the broker asynchronously.

```
┌── Application, ONE local DB transaction ──────────────┐
│  BEGIN                                                │
│    UPDATE orders SET status='PAID' WHERE id=42        │
│    INSERT INTO outbox (id, type, payload, created_at) │
│           VALUES (uuid, 'OrderPaid', {...}, now())    │
│  COMMIT           ← state + intent-to-publish atomic  │
└───────────────────────────────────────────────────────┘
                         │
        ┌── Relay / message dispatcher (separate) ──┐
        │  loop:                                     │
        │    rows = SELECT * FROM outbox             │
        │           WHERE published_at IS NULL       │
        │           ORDER BY created_at LIMIT N      │
        │    for row in rows:                        │
        │       broker.publish(row.type, row.payload,│
        │                      key=row.id)           │
        │       UPDATE outbox SET published_at=now() │
        │              WHERE id=row.id               │
        └────────────────────────────────────────────┘
                         │
                    [ BROKER ] ──► consumers
```

- **[MUST]** Write the state change and the outbox row in the **same DB transaction**. Either both commit or neither does — the dual-write window is closed.
- **[MUST]** Publish from the outbox via a **relay** (a.k.a. message dispatcher/polling publisher) that runs independently and is safe to crash-and-resume.
- **[MUST]** The relay is **at-least-once**: it may publish a row, then crash before marking it `published_at`, and republish on restart. That's fine — it's expected. The *consumer* dedups (inbox / idempotency, [03](03-idempotent-consumers.md)). Do not try to make the relay exactly-once; it can't be (see [01](01-messaging-fundamentals.md)).
- **[MUST]** Carry the outbox row's **stable id as the broker message id / dedup key** so consumers can recognize a redelivery.
- **[SHOULD]** Preserve per-entity order by relaying in `created_at` order and using the entity id as the **partition key** ([01](01-messaging-fundamentals.md)).
- **[SHOULD]** Run the relay as a leader-elected singleton or with `SELECT ... FOR UPDATE SKIP LOCKED` so multiple relay instances don't double-scan; duplicates are still harmless but wasteful.
- **[MUST]** Prune or archive published outbox rows on a retention schedule ([../standards-kb/06-data-management.md](../standards-kb/06-data-management.md)) — an unbounded outbox table degrades write performance (see [../performance-kb/03-code-level-performance.md](../performance-kb/03-code-level-performance.md)).

### Relay implementation options

| Approach | How it works | Trade-off |
|---|---|---|
| **Polling publisher** | Relay polls `outbox` for unpublished rows | Simplest; adds poll latency + DB load; tune interval/batch |
| **Transaction log tailing (CDC)** | Read the DB's commit log; publish outbox inserts | Near-real-time, no poll load; needs CDC infra (see below) |
| **Listen/notify** | DB pub/sub (e.g. `LISTEN/NOTIFY`) wakes the relay | Low latency; still poll as a safety net for missed signals |

## The inbox pattern (consumer-side dedup)

The mirror image on the consuming side. Because delivery is at-least-once, the consumer records **which message ids it has already processed** in an **inbox table**, and processes the business change **in the same transaction**.

```
┌── Consumer, ONE local DB transaction ─────────────────┐
│  BEGIN                                                 │
│    -- dedup: skip if already seen                      │
│    INSERT INTO inbox (message_id, processed_at)        │
│           VALUES (msg.id, now())                       │
│           ON CONFLICT (message_id) DO NOTHING          │
│    IF inserted 0 rows: COMMIT; ack; return  (dupe)     │
│    -- first time: apply the business effect            │
│    UPDATE inventory SET reserved = reserved + qty ...  │
│  COMMIT                                                │
└────────────────────────────────────────────────────────┘
      then ack the message to the broker
```

- **[MUST]** Record the message id and apply the business effect in **one transaction**, keyed on the message/dedup id (unique constraint). A redelivery hits the unique constraint, applies nothing, and acks — **effectively-once** (see [03](03-idempotent-consumers.md)).
- **[MUST]** Ack **after** the transaction commits, never before — otherwise a crash between ack and commit loses the effect (back to at-most-once).
- **[SHOULD]** Retain inbox entries at least as long as the broker's maximum redelivery/retention window, then prune ([../standards-kb/06-data-management.md](../standards-kb/06-data-management.md)).
- **[MAY]** Collapse the inbox into a domain-natural idempotency key when the effect is already keyed (e.g. `payment_id`) — the inbox is one concrete way to build the dedup store in [03](03-idempotent-consumers.md), not a mandatory extra table.

## CDC as an alternative

**Change Data Capture** streams committed row changes directly from the database's transaction log to the broker (Debezium, native logical replication, DynamoDB Streams).

- **You can skip the application-level publish entirely:** commit your state change, and CDC turns the committed rows into events. This is a clean way to run the outbox — write outbox rows, let CDC tail them — or to emit events straight from domain tables.
- **[SHOULD]** Prefer **outbox + CDC tailing** over CDC directly on domain tables when you want **explicit, well-shaped events**. Publishing raw table-row diffs leaks your internal schema to consumers and couples them to your storage model (violating the contract discipline in [05](05-event-schema-and-versioning.md) and [../standards-kb/10-api-integration.md](../standards-kb/10-api-integration.md)).
- **[MUST]** If you emit events from raw table changes, still put a stable event id and envelope on each ([05](05-event-schema-and-versioning.md)) and keep it at-least-once — CDC does not give exactly-once to downstream systems.
- **Trade-off:** CDC removes polling latency and DB load but adds infra to operate (connector, offsets, snapshotting) and is sensitive to schema migrations ([../standards-kb/02-versioning-deploy-migration.md](../standards-kb/02-versioning-deploy-migration.md)).

## Worked example

> **Example (fintech/e-commerce) — outbox + inbox end-to-end.** Customer pays for order 42.
> 1. **Payments service** commits, in one tx: `UPDATE orders SET status='PAID'` **and** `INSERT INTO outbox('OrderPaid', {orderId:42, total, currency}, id=e1)`.
> 2. **Relay** reads row `e1`, publishes `OrderPaid` to the topic with message id `e1` and `partitionKey=42`, then marks it published. (It crashes once and republishes `e1` — harmless.)
> 3. **Inventory consumer** receives `e1` twice. First time: `INSERT INTO inbox(e1)` succeeds → reserves stock → commits → acks. Second time: `INSERT ... ON CONFLICT DO NOTHING` inserts 0 rows → no reservation → acks. Stock reserved exactly once despite two deliveries.
> No dual write, no lost event, no double reservation — the guarantee is composed, not bought from the broker.

> **Example (healthcare).** When a specimen result is finalized, the LIS commits `UPDATE results SET status='FINAL'` **and** an outbox row `LabResultFinalized(id=r-88)` in one transaction, so a crash can never leave a finalized result that no downstream (ordering system, patient portal, billing) was told about — a patient-safety-relevant invariant. CDC tails the outbox table to publish with sub-second latency. The portal consumer uses an inbox keyed on `event.id` so a redelivered finalization doesn't send the patient two "results ready" alerts.

## Acceptance checklist

- [ ] No code path writes the DB and publishes to the broker as two separate, unguarded operations.
- [ ] State change + outbox insert happen in **one** local DB transaction.
- [ ] A relay publishes from the outbox, is safe to crash/resume, and is explicitly at-least-once.
- [ ] The outbox row id travels as the broker message/dedup id and (where order matters) the partition key.
- [ ] Consumers dedup via an inbox (or domain idempotency key) and ack only after commit.
- [ ] Outbox and inbox tables have retention/pruning; growth is bounded.
- [ ] If CDC is used, events carry a proper envelope and don't leak raw internal table schemas.
