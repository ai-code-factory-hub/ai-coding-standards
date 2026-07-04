# 04 · Saga & Orchestration

Purpose: coordinate a single business transaction that spans multiple services when there is no global ACID transaction to lean on. Sagas (choreography vs orchestration), compensating transactions, process managers/orchestrators, and when to use each. Builds on the fault-tolerance rules in [../standards-kb/05-reliability-resilience.md](../standards-kb/05-reliability-resilience.md) and assumes idempotent consumers ([03](03-idempotent-consumers.md)) and reliable publishing ([02](02-outbox-and-inbox.md)).

## The distributed-transaction problem

One business goal ("place an order") touches several services, each with its own database: reserve inventory, charge payment, allocate shipping. You cannot wrap them in one transaction.

- **[MUST]** Do not use distributed **two-phase commit (2PC / XA)** across microservices for online flows. It couples availability (one slow participant blocks all), locks resources across the network, and most modern datastores/brokers don't support it well. It trades the whole point of decoupling for a fragile lock.
- **[MUST]** Accept **eventual consistency**: the business transaction is a **sequence of local transactions**, each in one service, stitched together with messages. Between steps the system is temporarily inconsistent, and that's by design — surface it in the UX (e.g. "order pending").

A **saga** is that sequence: a set of local transactions where each step publishes a message that triggers the next, and each step has a **compensating action** that semantically undoes it if a later step fails.

## Compensating transactions

You cannot `ROLLBACK` a committed local transaction in another service. Instead you run a **compensating transaction** that **semantically undoes** it.

- **[MUST]** Define a compensation for every step that has externally-visible effects: `ReserveInventory` ↔ `ReleaseInventory`, `ChargePayment` ↔ `RefundPayment`, `AllocateCourier` ↔ `CancelAllocation`.
- **[MUST]** Compensations are themselves **idempotent** and **retryable** ([03](03-idempotent-consumers.md)) — a compensation may be delivered twice, or you may crash mid-rollback and resume.
- **[MUST]** Compensation is **semantic, not physical**: you don't erase history, you post an offsetting action (a refund, a release, a cancellation record). Preserve the audit trail ([../standards-kb/20-logging-audit-and-traceability.md](../standards-kb/20-logging-audit-and-traceability.md)).
- **[SHOULD]** Order steps so the **hardest-to-compensate step runs last** (e.g. charge payment after inventory is reserved), minimizing the blast radius of rollback. Some effects (a sent email, a shipped parcel) are **not** compensable — design the flow so they come after the point of no failure, or make them tolerant of cancellation.
- **[MAY]** Use **semantic locks / pending states** (e.g. `reserved`, `pending`) so a partially-completed saga doesn't expose intermediate state as final.

## Choreography vs orchestration

Two ways to wire the steps.

### Choreography — services react to each other's events

No central coordinator. Each service listens for events and emits its own; the flow **emerges** from the reactions.

```
OrderPlaced ─► [Inventory] ─ InventoryReserved ─► [Payment]
                                                     │
                     PaymentCharged ─► [Shipping] ─ Shipped ─► done
   on failure: PaymentFailed ─► [Inventory] runs ReleaseInventory (compensate)
```

- **Pros:** maximal decoupling; no central bottleneck; easy to add a new reactor.
- **Cons:** the workflow exists **nowhere explicitly** — it's emergent, hard to see, hard to debug ("who reacts to what?"), and cyclic-dependency-prone. Compensation logic is scattered across services.
- **[SHOULD]** Use choreography for **short** flows (2-4 steps) with **few** participants and simple failure handling.

### Orchestration — a central coordinator drives the steps

A **process manager / orchestrator** owns the workflow: it sends commands, receives replies, tracks state, and decides the next step (including compensations on failure).

```
        ┌──────────── Order Saga Orchestrator ───────────┐
        │  state machine:  RESERVING → CHARGING →         │
        │                  SHIPPING → COMPLETED / FAILED  │
        └───┬─────────────┬──────────────┬────────────────┘
   ReserveInventory   ChargePayment   AllocateShipping   (commands out)
   InventoryReserved  PaymentCharged  Shipped            (replies in)
   on any failure → orchestrator issues compensations in reverse
```

- **Pros:** the workflow is **explicit and centralized** — one place to read, test, monitor, and change. Compensation logic lives in one state machine. Easy to answer "where is order 42?"
- **Cons:** the orchestrator is a component to build/operate; risk of it becoming a god-service with business logic that belongs in the participants.
- **[SHOULD]** Use orchestration for **longer** flows, **many** participants, or **complex** failure/branching logic. Keep the orchestrator focused on *coordination*; keep domain rules in the services.
- **[MUST]** Persist orchestrator state durably (it's a long-running entity); it must survive restarts and resume mid-saga. Drive its commands through the outbox ([02](02-outbox-and-inbox.md)) and make replies idempotent ([03](03-idempotent-consumers.md)). Consider a workflow engine (Temporal, Camunda, AWS Step Functions) rather than hand-rolling durable state and timers.

| | Choreography | Orchestration |
|---|---|---|
| Coordinator | None (emergent) | Central process manager |
| Visibility of flow | Low (scattered) | High (one place) |
| Coupling | Lowest | Higher (to coordinator) |
| Best for | Short, few steps | Long, many steps, complex |
| Failure logic | Distributed | Centralized |

## Failure handling in sagas

- **[MUST]** Every saga has an explicit outcome: **completed** or **compensated-and-failed**. There is no "stuck forever" — enforce with timeouts on each step.
- **[MUST]** On a step failure that can't be retried into success ([03](03-idempotent-consumers.md)), run compensations for all **previously completed** steps, in reverse order, then mark the saga failed and notify.
- **[MUST]** Handle the **timeout / no-reply** case: an orchestrator step that gets no reply within its deadline must either retry (idempotent) or trigger compensation — never wait indefinitely.
- **[SHOULD]** Make each step and each reply **idempotent** so retries and redeliveries during a chaotic partial failure don't double-apply or double-compensate.
- **[SHOULD]** Alert on sagas that exceed expected duration or land in a failed/compensating state ([06](06-eventing-operations.md); ties to the [../sre-kb/runbooks/queue-backlog-and-dlq.md](../sre-kb/runbooks/queue-backlog-and-dlq.md) when steps stall on a queue).

## Worked examples

> **Example (fintech/e-commerce) — Place Order saga (orchestrated).** Steps: (1) `ReserveInventory` → `InventoryReserved`; (2) `ChargePayment` → `PaymentCharged`; (3) `AllocateShipping` → `Shipped`. Payment is ordered **after** reservation because a refund is costlier to compensate than a release. Failure case: inventory reserved, payment **declined**. The orchestrator, in state `CHARGING`, receives `PaymentFailed`, transitions to `COMPENSATING`, issues `ReleaseInventory` (idempotent), then marks the order `FAILED` and notifies the customer. A redelivered `PaymentFailed` is deduped by the orchestrator, so it doesn't release twice. Order state is `PENDING` throughout — never shown as confirmed until `COMPLETED`.

> **Example (healthcare) — Register & schedule specimen (choreographed).** Short 3-step flow: `PatientRegistered` → billing creates a charge and emits `ChargeCreated` → scheduling books the collection slot and emits `CollectionScheduled`. If scheduling finds no slot, it emits `SchedulingFailed`, to which billing reacts by voiding the charge (compensation `ChargeVoided`) and registration marks the encounter `INCOMPLETE`. Choreography fits here because the flow is short, has three participants, and failure handling is simple. Each reactor is idempotent so an at-least-once redelivery of `SchedulingFailed` doesn't void the charge twice, and every void is an auditable offsetting entry, not a deletion.

## Acceptance checklist

- [ ] No 2PC/XA across services for online flows; cross-service transactions are modeled as sagas with eventual consistency surfaced in the UX.
- [ ] Every effectful step has an idempotent, retryable, semantic **compensation**; non-compensable steps run last.
- [ ] Choreography vs orchestration is a deliberate choice (short/simple → choreography; long/complex → orchestration).
- [ ] Orchestrator (if used) persists state durably, resumes after restart, and drives commands via the outbox.
- [ ] Every saga reaches a definite outcome; step timeouts trigger retry or compensation — never an indefinite wait.
- [ ] Failed/compensating/overrunning sagas are alertable and traceable via correlation ids.
