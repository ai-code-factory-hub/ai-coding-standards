# 05 · Event Schema & Versioning

Purpose: treat an event as a **published contract** and evolve it without breaking independently-deployed consumers. Envelope metadata, schema design, a schema registry, backward/forward-compatible evolution, and an event catalog. This is the async mirror of the API-versioning discipline in [../standards-kb/10-api-integration.md](../standards-kb/10-api-integration.md); envelope fields feed the traceability rules in [../standards-kb/20-logging-audit-and-traceability.md](../standards-kb/20-logging-audit-and-traceability.md).

## An event is a contract

Once you publish an event, unknown consumers depend on its shape. You usually **cannot** coordinate a synchronized deploy across all of them (that's the whole point of decoupling). So an event schema must evolve like a public API: additively, compatibly, versioned.

- **[MUST]** Treat every published event as a **versioned public contract**. Breaking it is a breaking API change and follows the same governance as [../standards-kb/10-api-integration.md](../standards-kb/10-api-integration.md).
- **[MUST]** Never repurpose a field's meaning or type in place. Old and new consumers must be able to read the wire form during the (possibly long) window where both run.

## Envelope metadata

Separate the **envelope** (transport/routing/observability metadata, standard across all events) from the **payload** (the domain data, specific to the event type).

- **[MUST]** Every event carries a standard envelope with at least:

| Field | Purpose |
|---|---|
| `id` | Globally unique event id — the **dedup/idempotency key** ([03](03-idempotent-consumers.md)), also the outbox row id ([02](02-outbox-and-inbox.md)) |
| `type` | Event type + version, e.g. `com.acme.order.placed` / `OrderPlaced.v2` |
| `time` | When it occurred (event time, not publish time) |
| `source` | Producing service/context |
| `tenantId` | Tenant scope — for isolation, routing, fairness ([../standards-kb/01-architecture-multitenancy.md](../standards-kb/01-architecture-multitenancy.md), [06](06-eventing-operations.md)) |
| `correlationId` | Ties all events/spans of one business flow together ([../standards-kb/20-logging-audit-and-traceability.md](../standards-kb/20-logging-audit-and-traceability.md)) |
| `causationId` | The id of the message that directly caused this one |
| `version` / `dataschema` | Payload schema version / pointer to registry |

- **[SHOULD]** Adopt a standard envelope spec rather than inventing one. **CloudEvents** (CNCF) is the interoperable de-facto standard — its attributes (`id`, `source`, `type`, `time`, `subject`, `datacontenttype`, `dataschema`) map directly to the table above and have SDKs and broker bindings. Extend it with `tenantId`, `correlationId`, `causationId` as extension attributes.

```
{                                        ← ENVELOPE (standard)
  "specversion": "1.0",
  "id": "e-9f3c...",                     ← dedup key
  "source": "/payments",
  "type": "com.acme.order.placed",
  "time": "2026-07-05T10:00:00Z",
  "tenantid": "t-42",                    ← extension
  "correlationid": "c-1a2b",            ← extension
  "dataschema": "registry://order.placed/2",
  "data": {                              ← PAYLOAD (event-specific)
     "orderId": 42, "total": 199.00, "currency": "GBP"
  }
}
```

- **[MUST]** Put `tenantId` and `correlationId` in the envelope of **every** event so every downstream can enforce isolation and stitch traces without parsing the payload.

## Schema design

- **[MUST]** Use an explicit, machine-readable schema (JSON Schema, Avro, Protobuf) — not "whatever the serializer emits." It's the artifact contract tests ([../standards-kb/08-testing-qa.md](../standards-kb/08-testing-qa.md)) validate against.
- **[SHOULD]** Prefer schemas with strong evolution rules built in (**Avro**, **Protobuf**) for high-volume streams — they encode compatibility and are compact ([../performance-kb/03-code-level-performance.md](../performance-kb/03-code-level-performance.md)); JSON Schema is fine for lower-volume, human-readable events.
- **[MUST]** Name events in the **past tense** ([01](01-messaging-fundamentals.md)) and namespace them (`domain.aggregate.eventPast`), so the type is self-describing and collision-free.
- **[SHOULD]** Keep payloads **minimal and purposeful** — every field is a field you must keep supporting forever. Decide fat vs thin deliberately ([01](01-messaging-fundamentals.md)); don't dump internal DB rows onto the wire ([02](02-outbox-and-inbox.md) on CDC).
- **[MUST]** Redact/omit sensitive or regulated fields not every subscriber is authorized to see ([../standards-kb/06-data-management.md](../standards-kb/06-data-management.md), [../standards-kb/09-compliance-privacy.md](../standards-kb/09-compliance-privacy.md)).

## Schema registry

A **schema registry** is a central store of event schemas and their versions that producers and consumers validate against, enforcing compatibility **at publish time** before a bad schema reaches consumers.

- **[SHOULD]** Run a schema registry (Confluent Schema Registry, Apicurio, AWS Glue, or a repo-based JSON-Schema store) as the source of truth for event contracts.
- **[MUST]** Enforce a **compatibility policy** in the registry (e.g. `BACKWARD` or `FULL`) so an incompatible schema is **rejected at registration**, not discovered when a consumer crashes in production.
- **[SHOULD]** Reference the schema by id/URL from the envelope (`dataschema`) so a consumer can fetch and validate the exact version it received.
- **[MUST]** Version schemas in source control and gate changes through review/CI ([../standards-kb/12-devops-cicd.md](../standards-kb/12-devops-cicd.md)); the registry enforces at runtime, CI enforces at merge.

## Compatible evolution — add optional, never break

The core rule, stated three ways so nobody misses it:

- **[MUST]** **Add optional, never break.** Safe changes: add a new **optional** field (with a default); add a new event type; add a new enum value **only if** consumers treat unknown values gracefully. Breaking changes (require a new **major** version + parallel run): remove/rename a field, change a field's type, tighten a constraint, make an optional field required, change semantics.

Two compatibility directions:

- **Backward compatible** = **new** consumer can read **old** events (you only added optional fields / defaults). Lets you upgrade consumers first.
- **Forward compatible** = **old** consumer can read **new** events (it **ignores unknown fields** and tolerates unknown enum values). Lets you upgrade producers first.
- **[MUST]** Consumers **must ignore unknown fields** (tolerant reader) and handle unknown enum values without crashing — this is what makes producer-first rollout safe. A strict parser that rejects extra fields is a latent outage on the next additive change.
- **[SHOULD]** Aim for **FULL** compatibility (both directions) so producer and consumer deploy order doesn't matter — the natural fit for independently-deployed services.

### When you truly must break

- **[MUST]** Introduce a **new event version** (`OrderPlaced.v2` / new `type`) and **publish both v1 and v2 in parallel** during a migration window; migrate consumers off v1; then retire v1 with a deprecation notice — exactly the deprecation lifecycle of [../standards-kb/10-api-integration.md](../standards-kb/10-api-integration.md).
- **[MAY]** Run an **upcaster** that transforms stored v1 events to v2 on read (common in event-sourced systems) so historical events remain consumable.

## Event catalog & discovery

- **[SHOULD]** Maintain an **event catalog** — a discoverable inventory of every event type: schema, version history, owner, fat/thin, producers, subscribers, PII classification, and an example payload. Without it, consumers reverse-engineer contracts from traffic, and no one knows who breaks if a field changes.
- **[SHOULD]** Generate the catalog from the registry / schema repo so it can't drift from reality; link it from [../admin-kb/job-and-queue-monitor.md](../admin-kb/job-and-queue-monitor.md) and developer docs ([../standards-kb/14-devex-documentation.md](../standards-kb/14-devex-documentation.md)).
- **[MUST]** Record **ownership** per event type — a contract with no owner rots and blocks safe change.

## Worked examples

> **Example (fintech/e-commerce) — evolving `OrderPlaced`.** v1 payload: `{orderId, total, currency}`. Marketing wants the coupon used. Adding `couponCode` as an **optional** field is backward+forward compatible: old analytics consumers ignore it (tolerant reader), new ones use it — no coordinated deploy, registry accepts it under `FULL`. Later, `total` must split into `subtotal` + `tax` (a breaking change): the team publishes `OrderPlaced.v2` alongside v1, migrates the billing, analytics, and fraud consumers, then retires v1. Every event across both versions carries the CloudEvents envelope with `tenantId` and `correlationId`, so fraud can trace a flagged order end-to-end.

> **Example (healthcare) — `LabResultFinalized` schema.** Payload references `resultId`, `specimenId`, `panelCode`, and a list of analytes with values + units + reference ranges — deliberately **excluding** free-text clinical notes (regulated; only the authorized analytics consumer fetches them by id). The schema lives in the registry under `BACKWARD` compatibility. Adding an optional `criticalFlag` for auto-alerting is additive and safe. The event catalog lists the LIS as producer; the patient portal, ordering system, and billing as subscribers; and marks the payload's PHI classification so any change triggers a privacy review ([../standards-kb/09-compliance-privacy.md](../standards-kb/09-compliance-privacy.md)).

## Acceptance checklist

- [ ] Every event carries a standard envelope (`id`, `type`, `time`, `source`, `tenantId`, `correlationId`, `causationId`, `version`) — CloudEvents or equivalent.
- [ ] Schemas are explicit, machine-readable, past-tense-named, and namespaced; sensitive fields omitted/redacted.
- [ ] A schema registry enforces a compatibility policy at publish time; schemas are versioned in source control and gated in CI.
- [ ] Evolution is additive-optional; consumers are tolerant readers (ignore unknown fields/enum values).
- [ ] Breaking changes ship as a new event version, run in parallel, and retire the old one via a deprecation window.
- [ ] An event catalog inventories every event type with owner, producers, subscribers, and PII classification.
