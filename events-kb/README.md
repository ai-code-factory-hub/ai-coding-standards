# events-kb · Event-Driven & Async Messaging Patterns

Purpose: a project-agnostic, buildable playbook for **event-driven and asynchronous messaging** in an enterprise multi-tenant system. It turns the reliability primitives in the standards KB into concrete, implementable patterns: outbox/inbox, idempotent consumers, sagas, schema evolution, and eventing operations.

This KB **extends, and does not duplicate**, two standards domains:

- **[../standards-kb/05-reliability-resilience.md](../standards-kb/05-reliability-resilience.md)** — timeouts, retries with backoff + jitter, circuit breakers, idempotency keys, load shedding. Everything here about retries, DLQs, and idempotency is the messaging-specific expression of those rules. When 05 says "idempotency keys on creates/payments/external operations", this KB shows you how to build the dedup store behind them.
- **[../standards-kb/10-api-integration.md](../standards-kb/10-api-integration.md)** — synchronous API contracts, versioning, webhooks, pagination. Async messaging is the other half of integration: 10 covers the request/response edge; this KB covers the event/message edge. Envelope and schema-versioning rules here mirror the API-versioning discipline there.

> If a rule here contradicts 05 or 10, **05/10 are canonical** — file a gap note, don't fork the rule.

---

## When to go event-driven vs synchronous

Default to **synchronous** request/response. Reach for **asynchronous messaging** only when at least one of these is true:

| Signal | Prefer async / events |
|---|---|
| Caller doesn't need the result to proceed | Fire-and-forget work (email, indexing, thumbnails) |
| Work is slow or bursty | Report generation, batch imports, ML scoring |
| Multiple independent consumers care about one fact | "Order placed" → billing, inventory, analytics, notifications |
| Producer and consumer scale/deploy/fail independently | Decoupling service lifecycles |
| You need buffering / backpressure absorption | Traffic spikes that would overwhelm a downstream |
| You need durable retry across restarts | At-least-once delivery of important side effects |

Stay **synchronous** when:

- The caller **needs the answer now** to render a response or make a decision.
- The operation is a simple read or a fast, single-owner write.
- **Strong consistency** across the operation is required and a distributed saga would be overkill.
- The added operational surface (broker, DLQ, replay, schema registry) isn't justified — **[SHOULD]** not stand up messaging infrastructure for a handful of low-value events.

### The tradeoffs you are buying

Event-driven architecture (EDA) is not free. You trade **synchronous simplicity** for **operational and cognitive complexity**:

- **You gain:** decoupling, independent scaling, resilience to downstream outages, buffering, easy fan-out, an audit-friendly event log.
- **You pay with:** eventual consistency (no global transaction), harder debugging (causality spans services — see [../standards-kb/20-logging-audit-and-traceability.md](../standards-kb/20-logging-audit-and-traceability.md) for correlation IDs), duplicate/out-of-order delivery you must handle, schema evolution across independently-deployed services, and new failure modes (backlog, poison messages, DLQ storms).
- **The golden rule:** if you can't articulate why synchronous won't work, don't go async. EDA rewards systems with genuine decoupling needs and punishes those that adopt it for fashion.

---

## Index

| # | File | What it covers |
|---|---|---|
| 01 | [01-messaging-fundamentals.md](01-messaging-fundamentals.md) | Brokers, queue vs topic, pub/sub vs point-to-point, commands vs events, event design, delivery semantics, ordering & partitioning |
| 02 | [02-outbox-and-inbox.md](02-outbox-and-inbox.md) | The dual-write problem, transactional outbox, inbox dedup, CDC alternative |
| 03 | [03-idempotent-consumers.md](03-idempotent-consumers.md) | Idempotency keys, dedup store, retries + backoff, poison messages, DLQ + redrive |
| 04 | [04-saga-and-orchestration.md](04-saga-and-orchestration.md) | Distributed transactions, sagas (choreography vs orchestration), compensating transactions, process managers |
| 05 | [05-event-schema-and-versioning.md](05-event-schema-and-versioning.md) | Schema design, registry, compatible evolution, event catalog, envelope metadata (CloudEvents) |
| 06 | [06-eventing-operations.md](06-eventing-operations.md) | Monitoring (lag, throughput, DLQ depth), replay, backpressure, ordering in practice, multi-tenant fairness |

---

## How to read this KB

1. Start with **01** to fix vocabulary (at-least-once is the realistic default; "exactly-once" is a myth resolved by idempotency).
2. Read **02 + 03 together** — the outbox gets an event out reliably; the idempotent consumer takes it in safely. They are two ends of the same guarantee.
3. Read **04** only when a single business transaction spans multiple services.
4. **05** governs the contract; **06** governs running it in production.

## Related KBs

- **[../standards-kb/06-data-management.md](../standards-kb/06-data-management.md)** — the outbox table, retention of event logs, and per-tenant data isolation.
- **[../standards-kb/08-testing-qa.md](../standards-kb/08-testing-qa.md)** — contract tests for event schemas, consumer idempotency tests.
- **[../performance-kb/03-code-level-performance.md](../performance-kb/03-code-level-performance.md)** — efficient consumer loops, batching, and serialization cost.
- **[../sre-kb/runbooks/queue-backlog-and-dlq.md](../sre-kb/runbooks/queue-backlog-and-dlq.md)** — the on-call runbook when this all goes wrong.
- **[../admin-kb/job-and-queue-monitor.md](../admin-kb/job-and-queue-monitor.md)** — the operator-facing surface for queue depth, DLQ, and redrive.

## Acceptance checklist

- [ ] Team can state, for each async pathway, *why* it isn't synchronous.
- [ ] Every async pathway assumes **at-least-once** delivery and has an idempotent consumer (03).
- [ ] Producers use the **outbox** pattern (02) — no dual writes to DB and broker.
- [ ] Cross-service business transactions use a **saga** with compensations (04), not a distributed 2PC.
- [ ] Every event carries a standard **envelope** and is registered in a **catalog/registry** (05).
- [ ] Lag, throughput, and DLQ depth are monitored and alertable (06), wired to the queue-backlog runbook.
- [ ] Rules trace back to standards 05 and 10; no rule here silently contradicts them.
