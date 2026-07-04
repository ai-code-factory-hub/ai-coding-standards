# 01 · Messaging Fundamentals

Purpose: fix the vocabulary and the physics of async messaging so the rest of this KB is unambiguous. Brokers, delivery models, the command-vs-event distinction, event design, delivery semantics (and why "exactly-once" is mostly a myth), and ordering. Builds on [../standards-kb/10-api-integration.md](../standards-kb/10-api-integration.md) (integration contracts) and [../standards-kb/05-reliability-resilience.md](../standards-kb/05-reliability-resilience.md) (retries/idempotency).

## Brokers

A **message broker** is durable middleware that accepts messages from **producers** and delivers them to **consumers**, decoupling the two in time, space, and failure.

- **[MUST]** Use a durable, replicated broker for any message whose loss is a business defect. Categories: **log-based** (Kafka, Pulsar, Kinesis — ordered, replayable, retained) and **queue-based** (RabbitMQ, SQS, Azure Service Bus, ActiveMQ — competing consumers, per-message ack). Choose by need, not fashion (see the table below).
- **[MUST]** Treat the broker as a managed dependency with its own SLO, capacity plan, and failover — a down broker halts every async pathway (see [../standards-kb/05-reliability-resilience.md](../standards-kb/05-reliability-resilience.md) on cascading failure).
- **[SHOULD]** Prefer a managed broker; self-hosting a distributed log is a standing operational commitment.

| Axis | Log-based (Kafka-style) | Queue-based (RabbitMQ/SQS-style) |
|---|---|---|
| Retention | Retained window; replayable | Consumed = gone (unless DLQ) |
| Ordering | Per-partition | Best-effort / per-group |
| Consumption | Offset per consumer group | Ack/nack per message |
| Fan-out | Cheap (many groups read same log) | Needs a topic/exchange per subscriber |
| Best for | Event streams, replay, analytics | Task queues, work distribution |

## Queue vs topic; point-to-point vs pub/sub

Two orthogonal ideas people constantly conflate:

- **Point-to-point (queue):** one message → exactly **one** consumer in a group processes it. Competing consumers share the load. Use for **work distribution** ("process this job").

```
Producer ──► [ QUEUE ] ──► Consumer A   (only ONE of A/B/C
                       └──► Consumer B    handles each message)
                       └──► Consumer C
```

- **Publish/subscribe (topic):** one message → **every** subscriber gets its own copy. Use for **fan-out of facts** ("this happened; whoever cares, react").

```
                    ┌──► Billing sub    (each subscriber gets
Producer ──► [TOPIC]├──► Inventory sub    its OWN copy of the
                    └──► Analytics sub    same event)
```

- **[MUST]** Pick the model from intent: distributing work → queue; broadcasting a fact → topic. A log-based broker gives pub/sub via multiple consumer groups on one topic; each group is internally point-to-point across its partitions.

## Commands vs events

The single most clarifying distinction in messaging.

| | **Command** | **Event** |
|---|---|---|
| Intent | "Do this" (imperative) | "This happened" (past tense) |
| Naming | `ChargePayment`, `SendInvoice` | `PaymentCharged`, `InvoiceSent` |
| Recipients | Exactly one owner | Zero-to-many subscribers |
| Coupling | Sender knows the receiver | Publisher doesn't know subscribers |
| Failure | Can be rejected | Cannot be "rejected" — it already happened |
| Typical transport | Queue (point-to-point) | Topic (pub/sub) |

- **[MUST]** Name events in the **past tense** and commands in the **imperative**. A mislabeled "event" that expects exactly one handler and can be rejected is really a command — model it as one.
- **[SHOULD]** Prefer events for cross-service integration (looser coupling); reserve commands for a known, single-owner action. Publishing an event and letting subscribers decide beats commanding named services.

## Event design: fat vs thin, and event-carried state transfer

How much data to put in an event.

- **Thin event (notification):** carries just identifiers — `{ orderId }`. Consumer calls back to the source of truth for details.
  - Pros: small; source stays authoritative; no stale data in the message.
  - Cons: every consumer must call back (chatty, couples them to the producer's API, load on the producer).
- **Fat event (event-carried state transfer / ECST):** carries the full relevant state — `{ orderId, lineItems, total, customer, ... }`.
  - Pros: consumers are self-sufficient and resilient to producer downtime; enables local read models.
  - Cons: larger payloads; snapshot can be stale by the time it's read; more schema surface to version (see [05](05-event-schema-and-versioning.md)).

- **[MUST]** Choose deliberately per event and document it in the catalog. Don't accidentally leak a thin event that forces N callbacks per message under load (a self-inflicted backpressure source — see [06](06-eventing-operations.md)).
- **[SHOULD]** Prefer **event-carried state transfer** when consumers need to build independent read models or must survive producer outages; prefer thin events when payloads are large, sensitive, or change faster than they're read.
- **[MUST]** Never put secrets or full regulated payloads in a broadly fan-out event beyond what each subscriber is authorized to see — minimize per [../standards-kb/06-data-management.md](../standards-kb/06-data-management.md) and [../standards-kb/09-compliance-privacy.md](../standards-kb/09-compliance-privacy.md). Reference by ID and let authorized consumers fetch, or scope topics per data class.

## Delivery semantics — and the exactly-once myth

Three theoretical guarantees:

- **At-most-once:** deliver 0 or 1 times. Simple, but **loses messages** on failure. Acceptable only for disposable telemetry.
- **At-least-once:** deliver 1-or-more times; **never lost, may be duplicated**. This is the realistic, achievable default.
- **Exactly-once:** deliver precisely once. The thing everyone wants.

- **[MUST]** Design every consumer for **at-least-once** delivery. It is what durable brokers actually provide once you account for redeliveries after crashes, ack timeouts, retries, and rebalances.

> **Debunking "exactly-once delivery":** true exactly-once *delivery* across a network with independent failures is impossible (the Two Generals problem: the ack for your delivery can be lost, so the sender can't know whether to resend). What brokers market as "exactly-once" is either (a) scoped to *within one broker's transaction* (e.g. Kafka read-process-write to the same cluster), which does **not** extend to your database or an external API, or (b) really **effectively-once processing** = at-least-once delivery **+ idempotent consumers**. The honest guarantee you build is: *the broker may deliver a message more than once; my consumer produces the same result as if it arrived once.* That is **effectively-once**, and it is achieved with idempotency, not with a broker checkbox. See [03](03-idempotent-consumers.md).

- **[MUST]** Do not rely on a broker's "exactly-once" flag to protect a **cross-system** side effect (DB write, payment, email). It doesn't. Make the side effect idempotent instead (03).

## Ordering and partitioning keys

- Ordering is only guaranteed **within a partition/queue**, never globally across a topic.
- **[MUST]** If order matters for an entity (e.g. all events for one account, one patient, one order), route them to the same partition via a **partition key** = the entity id. Same key → same partition → ordered.

```
partitionKey = orderId
  order-42 events ─► partition 1 : Created → Paid → Shipped   (ordered ✓)
  order-77 events ─► partition 3 : Created → Cancelled        (ordered ✓)
  (no ordering guarantee BETWEEN partition 1 and partition 3)
```

- **[MUST]** Choose the partition key = the smallest scope over which order is required. Too coarse (e.g. `tenantId`) creates hot partitions and serializes unrelated work (see multi-tenant fairness in [06](06-eventing-operations.md)); too fine loses the ordering you needed.
- **[SHOULD]** Prefer designing consumers to **tolerate reordering** (commutative or versioned updates) rather than depending on strict global order — it's more robust to retries, DLQ redrive, and partition rebalances.
- **[MUST]** Remember that at-least-once + retries can reorder even within a partition when a message is retried later. Version your state (last-write-by-version, not last-write-by-arrival) — see [03](03-idempotent-consumers.md) and [05](05-event-schema-and-versioning.md).

## Worked examples

> **Example (healthcare):** A LIS publishes `LabResultFinalized` events. Order for one patient's specimen matters (`Amended` must not overtake `Finalized`), so the **partition key is `specimenId`**. The event is **fat** (carries the panel, values, and reference ranges) so the ordering/notification service can render an alert without calling back into the LIS during a spike. Regulated free-text notes are **excluded** from the fan-out event; only the analytics subscriber, authorized for them, fetches by `resultId`. Delivery is at-least-once, so the notification consumer dedups on `event.id` (03).

> **Example (fintech/e-commerce):** `OrderPlaced` is a **fat event** on a pub/sub topic consumed independently by billing, inventory, and analytics groups. `ChargePayment` is a **command** on a point-to-point queue owned solely by the payments service — it can be rejected (insufficient funds) and has exactly one handler. Payment-critical ordering per order uses `partitionKey = orderId`. The payments consumer treats delivery as at-least-once and uses an idempotency key so a redelivered `ChargePayment` never double-charges (see [../standards-kb/05-reliability-resilience.md](../standards-kb/05-reliability-resilience.md)).

## Acceptance checklist

- [ ] Broker choice (log vs queue) justified by ordering/retention/fan-out needs, not defaults.
- [ ] Each pathway is explicitly point-to-point (work) or pub/sub (fan-out of facts).
- [ ] Messages are labeled correctly: past-tense **events** vs imperative **commands**, with matching transport.
- [ ] Each event is documented as fat or thin; ECST used only where consumers need self-sufficiency; no secrets/regulated data over-shared.
- [ ] Every consumer is designed for **at-least-once**; no cross-system side effect trusts a broker "exactly-once" flag.
- [ ] Partition keys chosen at the minimal scope where order is required; consumers tolerate reordering where feasible.
