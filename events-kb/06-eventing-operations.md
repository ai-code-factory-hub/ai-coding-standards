# 06 · Eventing Operations

Purpose: run event-driven systems in production. Monitoring the right signals (lag, throughput, DLQ depth), replay/reprocessing, backpressure, ordering guarantees as they actually behave, and multi-tenant fairness. This is where the patterns in 01-05 meet on-call reality. Ties directly to [../sre-kb/runbooks/queue-backlog-and-dlq.md](../sre-kb/runbooks/queue-backlog-and-dlq.md) and the operator surface in [../admin-kb/job-and-queue-monitor.md](../admin-kb/job-and-queue-monitor.md); extends the SLO/observability discipline in [../standards-kb/05-reliability-resilience.md](../standards-kb/05-reliability-resilience.md) and [../standards-kb/07-observability-aiops.md](../standards-kb/07-observability-aiops.md).

## Monitoring: the signals that matter

A queue's health is invisible unless you watch the right numbers. The four golden eventing signals:

| Signal | What it tells you | Alert when |
|---|---|---|
| **Consumer lag / backlog** | Messages produced but not yet consumed (offset gap or queue depth) | Growing monotonically, or oldest-message age past SLA |
| **Throughput** (produce vs consume rate) | Whether consumers keep up with producers | Consume rate < produce rate sustained, or drops to ~0 |
| **DLQ depth** | Messages that failed all retries | > 0 and climbing (any DLQ growth is a signal) |
| **Processing latency / age** | End-to-end time and oldest un-acked message | p99 age past business SLA |

- **[MUST]** Monitor and alert on **lag, throughput, DLQ depth, and oldest-message age** per topic/queue **and per consumer group**. A healthy aggregate can hide one dead partition or one starved tenant.
- **[MUST]** Alert on **lag trend**, not just an absolute threshold — steadily-growing lag means consumers will never catch up, even if the current depth looks fine. This is the primary trigger for [../sre-kb/runbooks/queue-backlog-and-dlq.md](../sre-kb/runbooks/queue-backlog-and-dlq.md).
- **[MUST]** Any **DLQ growth is alertable** ([03](03-idempotent-consumers.md)) — a non-empty DLQ is delayed data loss until someone triages it.
- **[SHOULD]** Emit per-message structured logs and traces carrying `correlationId`/`causationId` ([05](05-event-schema-and-versioning.md), [../standards-kb/20-logging-audit-and-traceability.md](../standards-kb/20-logging-audit-and-traceability.md)) so one business flow is traceable across every hop.
- **[SHOULD]** Surface these on the operator dashboard ([../admin-kb/job-and-queue-monitor.md](../admin-kb/job-and-queue-monitor.md)): depth, lag, throughput, DLQ, consumer health, and redrive controls.
- **[SHOULD]** Track consumer **health/heartbeat and rebalance events** — a crash-looping or constantly-rebalancing consumer group produces lag with a healthy-looking broker.

## Replay & reprocessing

The ability to **re-consume** past events — to rebuild a read model, recover from a consumer bug, backfill a new consumer, or drain a DLQ ([03](03-idempotent-consumers.md) redrive).

- **[MUST]** Replay is only safe because consumers are **idempotent** ([03](03-idempotent-consumers.md)). Reprocessing a retained log re-delivers events already handled; without idempotency, replay corrupts state. Never enable replay against a non-idempotent consumer.
- **[SHOULD]** For log-based brokers, replay by **resetting the consumer group's offset** to a timestamp/position; for queue-based brokers, replay from an event store, an archive, or the DLQ.
- **[MUST]** Replay in a **controlled, rate-limited** way — a full-history replay at max speed becomes a self-inflicted backlog and can overwhelm downstreams. Throttle and, where possible, replay onto an isolated consumer/topic so live traffic isn't starved.
- **[SHOULD]** Watch for **side-effect replay hazards**: re-consuming `PaymentCharged` must not re-charge and must not re-send notifications. Guard non-idempotent external effects (email/PSP) behind their dedup keys ([03](03-idempotent-consumers.md)) before enabling replay, or replay in a mode that suppresses external effects.
- **[SHOULD]** Set **retention** deliberately ([../standards-kb/06-data-management.md](../standards-kb/06-data-management.md)) — replay is only possible within the retained window; longer retention = more recovery power but more storage and privacy surface (erasure obligations, [../standards-kb/09-compliance-privacy.md](../standards-kb/09-compliance-privacy.md)).

## Backpressure

When producers outrun consumers, something must give. Backpressure is the deliberate policy for *what*.

- **[MUST]** Have an explicit backpressure strategy per pathway — don't let unbounded buffering silently consume memory until an OOM crash. Options:
  - **Buffer** in the durable broker (the broker *is* your backpressure buffer — it absorbs spikes so downstreams don't fall over). Bound it with retention/quotas.
  - **Scale consumers** — add partitions/consumer instances to raise throughput (see below on the partition ceiling).
  - **Shed / defer** — under sustained overload, drop or defer low-value work to protect core flows ([../standards-kb/05-reliability-resilience.md](../standards-kb/05-reliability-resilience.md) load shedding).
  - **Rate-limit / slow producers** — apply flow control upstream (429/Retry-After at the API edge) so the queue doesn't grow faster than any consumer count can drain.
- **[MUST]** Consumers **pull at a sustainable rate** (bounded prefetch/in-flight) rather than accepting unbounded in-flight work — an over-eager prefetch is how a single consumer OOMs and stalls a partition.
- **[SHOULD]** Prefer the broker-as-buffer model: the reason to go async ([README](README.md)) is precisely that a durable queue absorbs bursts the synchronous path couldn't. Size retention/quota to the longest realistic consumer outage.
- **[SHOULD]** Use **circuit breakers** so a failing downstream causes consumers to slow/pause (letting the broker buffer) rather than hammer it into a deeper outage ([03](03-idempotent-consumers.md), [../standards-kb/05-reliability-resilience.md](../standards-kb/05-reliability-resilience.md)).

## Ordering guarantees in practice

Theory from [01](01-messaging-fundamentals.md) meets operations:

- **[MUST]** Remember order holds **only within a partition/queue**, and **retries + DLQ redrive can still reorder** even there (a retried message lands after later ones). Design consumers to tolerate it — **version your state** (apply-if-newer, not apply-on-arrival), don't assume arrival order equals event order ([01](01-messaging-fundamentals.md), [05](05-event-schema-and-versioning.md)).
- **[MUST]** Understand the **ordering-vs-parallelism trade-off**: strict per-key order caps parallelism at one consumer per partition/key. You scale throughput by adding partitions — but **partition count is your parallelism ceiling** for a consumer group, and re-partitioning a keyed topic is disruptive. Size partitions for peak parallelism up front.
- **[SHOULD]** Keep the ordered scope **as narrow as correctness allows** ([01](01-messaging-fundamentals.md)) — narrower keys = more partitions usable in parallel = higher throughput and better fairness.
- **[SHOULD]** For a poison message on an ordered partition, dead-letter it promptly ([03](03-idempotent-consumers.md)) so it doesn't head-of-line-block everything behind it — a stalled ordered partition is a common backlog root cause ([../sre-kb/runbooks/queue-backlog-and-dlq.md](../sre-kb/runbooks/queue-backlog-and-dlq.md)).

## Multi-tenant fairness

In a shared multi-tenant system ([../standards-kb/01-architecture-multitenancy.md](../standards-kb/01-architecture-multitenancy.md)), one tenant's flood must not starve everyone else's events — the **noisy-neighbor** problem on the message plane.

- **[MUST]** Prevent one tenant's burst (a bulk import, a runaway producer) from monopolizing consumers and starving other tenants' events. A single shared FIFO queue gives the loudest tenant the whole pipe.
- **[SHOULD]** Choose an isolation strategy by blast-radius need:
  - **Partition-key includes tenant** — spreads tenants across partitions, but a hot tenant still hogs its partition and can dominate a shared group.
  - **Per-tenant (or per-tier) queues/topics** — strong isolation; more topics to manage; good for large/paid tenants.
  - **Weighted/fair scheduling** — round-robin or quota across tenants within a consumer so no tenant exceeds its share.
  - **Per-tenant rate limits / quotas** at the producer edge ([../standards-kb/05-reliability-resilience.md](../standards-kb/05-reliability-resilience.md)) so no tenant can outproduce its allotment.
- **[MUST]** Monitor lag and DLQ depth **per tenant** (or per tier), not just globally — a global-green dashboard can hide one tenant whose events are hours behind.
- **[SHOULD]** Give **premium/critical** tenants or event classes their own lane (dedicated partitions/queues or higher weight) so a free-tier bulk job never delays a paying tenant's critical flow.

## Worked examples

> **Example (fintech/e-commerce) — Black Friday spike.** Order events surge 10x. The durable broker absorbs the burst (backpressure = buffer); consumer autoscaling adds instances up to the `orderId`-partition ceiling (sized for peak in advance). Lag is watched by **trend**: it climbs during the spike but the drain rate exceeds produce rate, so it's a catch-up, not a stall — no page. A batch of card declines fills the DLQ; on-call is alerted on DLQ growth, fixes a fraud-rule regression, and **redrives in throttled batches** so replay doesn't re-create the backlog. Idempotent consumers make the redrive safe from double-charges. A merchant running a bulk price import is rate-limited per tenant so its events don't starve live checkout traffic.

> **Example (healthcare) — analyzer batch + fairness.** A hospital lab uploads a 50,000-result batch overnight (one tenant). Because lag and DLQ are tracked **per tenant**, ops sees this tenant's queue grow without mistaking it for a global outage. Results are partitioned by `specimenId`, so per-patient order is preserved while the batch parallelizes across partitions. A dedicated lane for STAT (critical) results keeps urgent findings flowing ahead of the routine batch — multi-tenant *and* multi-priority fairness. When a firmware change makes one analyzer's messages un-parseable (poison), they're dead-lettered fast so they don't head-of-line-block the patient's ordered stream; after a schema fix ([05](05-event-schema-and-versioning.md)) the DLQ is replayed idempotently, re-upserting results with no duplicate patient alerts.

## Acceptance checklist

- [ ] Lag, throughput, DLQ depth, and oldest-message age are monitored and alerted per topic **and** per consumer group, on trend not just threshold.
- [ ] Alerts wire into the queue-backlog-and-DLQ runbook; operators have a dashboard with depth/lag/DLQ/redrive controls.
- [ ] Replay/reprocessing is possible, rate-limited, and safe (idempotent consumers, guarded external side effects); retention is set deliberately.
- [ ] Each pathway has an explicit backpressure strategy (buffer/scale/shed/rate-limit) with bounded consumer prefetch — no unbounded in-flight growth.
- [ ] Consumers tolerate reordering (versioned state); partition count sized for peak parallelism; poison messages dead-lettered promptly.
- [ ] Multi-tenant fairness enforced (per-tenant partitioning/queues/quotas/weighting); lag and DLQ monitored per tenant/tier; critical lanes isolated.
