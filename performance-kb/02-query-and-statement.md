# 02 · Query & Statement

**Purpose:** make the database the fast tier. Concrete patterns for prepared statements, N+1
elimination, tenant-first indexing, keyset pagination, replicas, and materialized views —
implementing the query rules of [../standards-kb/03-performance-scalability.md](../standards-kb/03-performance-scalability.md)
and the schema hygiene of [../standards-kb/06-data-management.md](../standards-kb/06-data-management.md).

## Prepared / parameterized statements + statement caching

- **[MUST]** All queries are **parameterized** (bind variables), never string-concatenated.
  This is a correctness/security rule (SQL injection) *and* a performance rule — the DB reuses a
  cached plan instead of re-parsing/re-planning each variant.
- **[MUST]** Reuse **prepared statements** on hot paths so parse+plan happens once. Keep a bounded
  per-connection statement cache.

  ```
  # Bad: new plan every call, unbounded plan-cache churn, injectable
  q = "SELECT * FROM orders WHERE tenant_id = " + tid + " AND id = " + oid

  # Good: one cached plan, bound params
  stmt = prepare("SELECT id, status FROM orders WHERE tenant_id = $1 AND id = $2")
  row  = stmt.execute(tenant_id, order_id)
  ```

- **[SHOULD]** Watch for **parameter sniffing / skewed plans**: when one param value has wildly
  different selectivity, a cached plan can be wrong for others. Fix with per-value plans, a plan
  hint, or by splitting the query.
- **[MAY]** With a transaction-mode connection pooler that disables server-side prepared
  statements, rely on **protocol-level prepare** or plan-cache-friendly SQL (see
  [01 · Pooling](01-pooling-and-resources.md)).

## N+1 elimination (batching / dataloader)

- **[MUST]** Never issue one query per row of a parent result. Detect N+1 in tests by asserting a
  **query count** per endpoint.
- **[MUST]** Collapse child lookups into **one batched query** (`IN (...)` / join), or use a
  **dataloader** that coalesces per-request keys into a single round trip:

  ```
  # N+1: 1 query for orders + N queries for each order's customer
  for o in orders: o.customer = db.query("... WHERE id = ?", o.customer_id)   # BAD

  # Batched: 2 queries total
  ids       = distinct(o.customer_id for o in orders)
  customers = db.query("SELECT * FROM customers WHERE tenant_id=$1 AND id = ANY($2)", tid, ids)
  by_id     = index_by(customers, 'id')
  for o in orders: o.customer = by_id[o.customer_id]

  # Dataloader: batches + de-dupes keys accumulated during one request tick
  loader.load(customer_id)  ->  auto-batched into ANY($ids) on next tick
  ```

- **[SHOULD]** Prefer a single join for one-to-one/one-to-few; prefer a second batched query for
  one-to-many to avoid row multiplication (cartesian blow-up) across multiple child collections.

## Select only needed columns

- **[MUST]** `SELECT` an explicit column list, never `SELECT *` on hot paths — narrower rows mean
  less I/O, and covering indexes can serve the query without touching the heap. Ties to the
  frontend "minimal fields per screen" rule in [04 · Frontend Performance](04-frontend-performance.md).

## Indexing for query patterns (tenant_id first)

- **[MUST]** Composite indexes lead with **`tenant_id`**, then the columns used for equality, then
  the range/sort column. Order matters — the index is only usable left-to-right.

  ```
  -- Query: WHERE tenant_id=? AND status=? ORDER BY created_at DESC LIMIT 50
  CREATE INDEX ix_orders_tenant_status_created
      ON orders (tenant_id, status, created_at DESC);
  -- Serves equality (tenant_id, status) + ordered range (created_at) in one index scan.
  ```

- **[MUST]** Index every foreign key used in joins/filters. **[MUST]** no interactive query > 100 ms
  without a supporting index (Standard 03).
- **[SHOULD]** Use **covering indexes** (`INCLUDE` non-key columns) so hot reads are index-only.
- **[SHOULD]** Use **partial indexes** for skewed predicates (`WHERE status = 'active'`) to keep
  the index small and hot.
- **[SHOULD]** Verify usage with the DB planner (see EXPLAIN in
  [05 · Profiling & Benchmarks](05-profiling-and-benchmarks.md)); drop unused indexes — they tax
  every write.

## Keyset (cursor) pagination

- **[MUST]** Deep pagination uses **keyset**, not `OFFSET`. `OFFSET n` scans and discards `n`
  rows — cost grows linearly with page depth. Keyset is O(page size) at any depth.

  ```
  -- Page 1
  SELECT id, created_at, ... FROM orders
   WHERE tenant_id = $1
   ORDER BY created_at DESC, id DESC
   LIMIT 50;

  -- Next page: seek past the last row seen (cursor = last (created_at, id))
  SELECT id, created_at, ... FROM orders
   WHERE tenant_id = $1
     AND (created_at, id) < ($cursor_created_at, $cursor_id)   -- tuple comparison, stable tiebreak
   ORDER BY created_at DESC, id DESC
   LIMIT 50;
  ```

  The `ORDER BY` must exactly match the index and include a unique tiebreaker (`id`) so pages are
  stable and gap-free.

## Avoiding LIKE '%x%' scans

- **[MUST NOT]** Serve interactive search with a **leading-wildcard `LIKE '%x%'`** — it cannot use
  a b-tree index and forces a full scan (Standard 03). Alternatives:
  - **Prefix search** `LIKE 'x%'` → uses a normal index.
  - **Full-text search** (tokenized index / GIN) for word matching.
  - **Trigram index** for fuzzy/substring where genuinely required.
  - **Recent-first default**: show recently/frequently used items; hit the search index only on
    explicit query.

## Short transactions

- **[MUST]** Keep transactions **short and CPU/DB-only**. **[MUST NOT]** hold a transaction open
  across a third-party call, user think-time, or a slow computation — long transactions hold locks
  and bloat MVCC, cascading into lock waits for everyone.
- **[SHOULD]** Do the external call *before* or *after* the transaction; use the outbox pattern for
  "commit + then publish" (see [../standards-kb/05-reliability-resilience.md](../standards-kb/05-reliability-resilience.md)).
- **[SHOULD]** Choose the lowest sufficient isolation level; use narrow `SELECT ... FOR UPDATE`
  (or optimistic version columns) instead of coarse locks.

## Read replicas

- **[SHOULD]** Route read-heavy, replication-lag-tolerant queries (reports, list views, search) to
  **read replicas**; keep writes and read-your-write flows on the primary.
- **[MUST]** Account for **replication lag** — after a write, read from the primary (or a "read
  primary until N ms after write" rule) so the user sees their own change.

## Materialized views / pre-aggregation

- **[SHOULD]** Back dashboards and expensive aggregates with **materialized views or summary
  tables**, refreshed on a schedule or incrementally — not computed live per request.
- **[MUST]** Define a refresh cadence and staleness budget; use concurrent/online refresh so reads
  aren't blocked. Serve "as of <timestamp>" so users understand freshness.

> **Example (multi-tenant LIS dashboard):** a "tests-by-status today" tile aggregated live scans
> millions of result rows per load. A summary table keyed `(tenant_id, day, status)` refreshed
> every 5 minutes turns the tile into a single indexed lookup — p95 drops from seconds to < 20 ms.

> **Example (fintech ledger search):** analysts searched transactions with `description LIKE
> '%acme%'`, forcing full scans that broke the p95 budget. Switching to a full-text index plus a
> recent-first default keeps interactive search index-backed and under 300 ms.

## Acceptance checklist

- [ ] All queries parameterized; hot paths reuse prepared statements with a bounded cache.
- [ ] No N+1 — child lookups batched/dataloaded; endpoints assert a query-count ceiling in tests.
- [ ] Explicit column lists on hot paths; covering indexes where it pays.
- [ ] Composite indexes lead with `tenant_id`; every filtered FK indexed; no >100 ms query unindexed.
- [ ] Deep pagination uses keyset with a unique tiebreaker matching the index order.
- [ ] No interactive `LIKE '%x%'`; search uses prefix/full-text/trigram + recent-first default.
- [ ] Transactions are short; no external call inside a transaction.
- [ ] Reads routed to replicas where lag-tolerant, with read-your-write handled.
- [ ] Dashboards/aggregates served from materialized views/summary tables with a stated staleness budget.
