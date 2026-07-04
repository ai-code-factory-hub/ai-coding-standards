# 03 · Performance & Scalability

Enforced performance budgets, progressive disclosure, async job queues, pooling, indexing, and stateless horizontal scale.

## Performance budgets (set + enforce in CI)

- **[MUST]** API p95 < 300 ms, p99 < 1 s for interactive reads; writes < 1 s p95. FCP < 1.8 s, LCP < 2.5 s, INP < 200 ms on a mid-range device. Initial JS bundle < 250 KB gzipped. No interactive DB query > 100 ms without an index or documented justification. Budgets are tested in CI (Lighthouse CI / load-test thresholds); regressions fail the build.

## The rules

- **[MUST]** **Minimize fields per screen** — progressive disclosure; field visibility is config-driven per tenant/role. Paginate/virtualize all lists (server-side pagination + windowed scroll). Lazy-load routes and heavy components.
- **[MUST]** **Recent-first search** — lookups default to recently/frequently used items; full-table search only on explicit action and via a proper index — **never `LIKE '%x%'` scans**.
- **[MUST]** Slow or third-party work runs in a **retried, idempotent job queue with a DLQ** (document/PDF generation, messaging sends, external-integration pushes, imports/exports). Long operations return a job id; the UI polls or subscribes.
- **[MUST]** **Stream, don't buffer** large payloads (bulk CSV import, exports, large result sets). Use bounded batches; deterministically close resources; set process memory limits; bound and evict caches (LRU + TTL + max size).
- **[MUST]** All external connections are pooled (DB, cache, broker, HTTP) with sized limits, acquire timeouts, and max lifetime. **[SHOULD]** front the DB with a pooler (PgBouncer/RDS Proxy) and route reads to replicas.
- **[MUST]** Index for query patterns (`tenant_id` first in composite indexes); no N+1; select only needed columns; keep transactions short (never hold one across a third-party call). **[SHOULD]** use materialized views / pre-aggregated tables for dashboards and keyset (cursor) pagination.
- **[MUST]** The app tier is **stateless** → horizontal scale-out. **[SHOULD]** auto-scale on CPU/latency/queue-depth; plan around hot-tenant/hot-row contention.

> **Example (healthcare):** a morning specimen-registration rush hits a shared search box — recent-first, indexed lookup keeps p95 under budget where a `LIKE '%name%'` scan would collapse under load.
>
> **Example (fintech/e-commerce):** a flash-sale checkout surge pushes payment-capture and receipt generation onto an idempotent job queue with a DLQ, so a slow payment gateway never blocks the interactive cart.

See [Reliability & Resilience](05-reliability-resilience.md) for timeouts/breakers on those external calls and [Data Management](06-data-management.md) for indexing and object storage.

## Acceptance checklist

- [ ] Performance budgets defined and enforced in CI; regressions fail the build.
- [ ] Screens use progressive disclosure with config-driven fields; all lists paginated/virtualized.
- [ ] Search is recent-first and index-backed — no full-table `LIKE '%x%'` scans.
- [ ] Slow/third-party work runs in a retried, idempotent job queue with a DLQ.
- [ ] Large payloads are streamed/batched with bounded memory and evicting caches.
- [ ] Connections pooled with sized limits; indexes lead with `tenant_id`; no N+1; short transactions.
- [ ] App tier is stateless and horizontally scalable.
