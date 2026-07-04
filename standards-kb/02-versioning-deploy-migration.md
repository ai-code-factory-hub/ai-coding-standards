# 02 · Versioning, Deployment & Migration

SemVer everywhere, zero-downtime deploys, expand→migrate→contract schema changes, and controlled per-tenant rollouts.

## Versioning

- **[MUST]** Use SemVer (MAJOR.MINOR.PATCH) for every shippable artifact: API, UI, DB schema, and each plugin. Version the API in the contract (`/v1/`); support **N and N-1** with a published deprecation window (6–12 months).
- **[MUST]** Version the DB schema with a migration tool (Flyway/Liquibase/Alembic/Prisma/EF); the app declares its compatible schema range. Maintain and test a **compatibility matrix** (UI × API × DB).

## Deployment

- **[MUST]** **Zero-downtime deploys** (rolling/blue-green/canary); old and new versions run simultaneously against the same DB; every deploy is rollback-able in minutes; health checks gate traffic.

## Schema migration

- **[MUST]** **Expand → Migrate → Contract** for every schema change (never a breaking change in one step). Migrations are forward-only in prod but each has a tested down path; idempotent and re-runnable; backfills are batched/resumable/throttled as background jobs. No destructive operation runs until telemetry confirms zero readers/writers.
- **[MUST NOT]** Couple a destructive migration to the deploy that introduces the new code path.
- **[SHOULD]** Take a verified backup immediately before any risky migration, and know the restore time.

> **Example (healthcare):** splitting a patient `name` column into structured parts — add new columns (expand), dual-write and backfill in throttled batches (migrate), and only drop the old column after telemetry shows no readers (contract).
>
> **Example (fintech/e-commerce):** renaming `order_total` to `amount_minor` for integer-money correctness follows the same three steps; the destructive drop ships in a *later* release than the code that stopped reading the old field.

## Tenant rollout

- **[MUST]** Tenant upgrades are a **controlled rollout** (internal → canary → cohorts → all), resumable/idempotent, with per-tenant rollback, audited, and tenants notified.
- **[SHOULD]** Enterprise tenants can pin a version or schedule maintenance windows in their own timezone.

See [Reliability & Resilience](05-reliability-resilience.md) for health-gated failover and [Data Management](06-data-management.md) for backup/restore guarantees.

## Acceptance checklist

- [ ] SemVer on every artifact; version surfaced in `/about` and the UI footer.
- [ ] API supports N and N-1 with a published deprecation window.
- [ ] Compatibility matrix (UI × API × DB) maintained and tested.
- [ ] Every schema change follows expand → migrate → contract; destructive steps decoupled from the deploy that adds the new code path.
- [ ] Backfills are batched, idempotent, resumable, and throttled.
- [ ] Zero-downtime deploys with health-gated traffic and rollback tested.
- [ ] Tenant upgrades are a controlled, resumable, audited rollout with per-tenant rollback.
