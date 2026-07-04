# 05 · Profiling & Benchmarks

**Purpose:** measure before optimizing, prove where time goes, and turn every budget into an
automated CI gate so regressions fail the build. Implements the "budgets are tested in CI" rule of
[../standards-kb/03-performance-scalability.md](../standards-kb/03-performance-scalability.md) and
lives alongside [../standards-kb/08-testing-qa.md](../standards-kb/08-testing-qa.md) and
[../standards-kb/12-devops-cicd.md](../standards-kb/12-devops-cicd.md).

## First principle: measure, don't guess

- **[MUST]** Never optimize on intuition. Reproduce, measure, form a hypothesis, change one thing,
  re-measure. Track **p50/p95/p99**, not the mean — tail latency is what users and budgets feel.
- **[MUST]** Measure on **representative data volume and hardware** — a query fast on 1k rows can
  be catastrophic on 10M. Seed benchmarks with production-scale, multi-tenant data.

## How to measure

### APM / distributed tracing

- **[MUST]** Instrument with **distributed tracing** so a slow request decomposes into spans
  (app → DB → cache → upstream). Carry a correlation id (ties to
  [../standards-kb/07-observability-aiops.md](../standards-kb/07-observability-aiops.md)).
- **[SHOULD]** Dashboard **RED** (Rate, Errors, Duration) per endpoint and **USE** (Utilization,
  Saturation, Errors) per resource. Alert on p95 breaching budget, not just on errors.
  > **Example:** an APM trace shows a 900 ms endpoint is 40 ms app + 820 ms in one repeated
  > child query → an N+1 (fix per [02 · Query & Statement](02-query-and-statement.md)).

### CPU / allocation profiling (flame graphs)

- **[SHOULD]** Profile hot paths with a **sampling profiler** and read the **flame graph**: width =
  total time in a call stack. Wide frames are where to optimize; use a differential/off-CPU profile
  to find blocking waits vs CPU.
- **[SHOULD]** Capture an **allocation profile** to find GC pressure (see
  [03 · Code-Level Performance](03-code-level-performance.md)); watch GC pause p99.

### Database (EXPLAIN ANALYZE)

- **[MUST]** For any slow or new hot query, read **`EXPLAIN (ANALYZE, BUFFERS)`**. Look for:
  **Seq Scan** on a large table where an index is expected, **rows estimate vs actual** skew (stale
  stats / bad plan), nested-loop N+1 shapes, and high buffer/heap fetches (missing covering index).

  ```
  EXPLAIN (ANALYZE, BUFFERS)
  SELECT id, status FROM orders
   WHERE tenant_id = $1 AND status = 'open'
   ORDER BY created_at DESC LIMIT 50;
  -- WANT: Index Scan using ix_orders_tenant_status_created ... rows≈actual, no Sort, no Seq Scan
  -- SMELL: Seq Scan on orders  (actual rows=2,000,000)  -> add/fix the composite index
  ```

- **[SHOULD]** Track slow queries continuously (slow-query log / statement stats view) and review
  the top offenders by total time each release.

### Frontend

- **[MUST]** Measure Web Vitals in **lab** (synthetic, throttled mid-range profile) *and* **field**
  (RUM), segmented by device class and tenant. Use browser performance traces to find long tasks
  and layout shifts (see [04 · Frontend Performance](04-frontend-performance.md)).

## Load & stress testing

- **[MUST]** Load-test critical flows before launch and before major releases, at expected **and**
  2–3× peak. Define **thresholds as pass/fail**, not just charts.
  > **Example (k6):** thresholds encode the budget so the run exits non-zero on breach:
  > ```js
  > export const options = {
  >   scenarios: { steady: { executor: 'constant-arrival-rate', rate: 400, timeUnit: '1s',
  >                          duration: '5m', preAllocatedVUs: 100 } },
  >   thresholds: {
  >     http_req_duration: ['p(95)<300', 'p(99)<1000'],  // API budget
  >     http_req_failed:   ['rate<0.01'],                // <1% errors
  >   },
  > };
  > ```
- **[SHOULD]** Cover the shapes that break systems: **soak** (memory leaks / pool exhaustion over
  hours), **spike** (autoscale + breaker behavior), and **hot-tenant** (one tenant at 10× — verify
  bulkheads from [01 · Pooling](01-pooling-and-resources.md)).
- **[MUST]** Test **multi-tenant isolation under load** — one tenant's load must not push another's
  p95 past budget.

## Micro-benchmarks

- **[SHOULD]** Micro-benchmark isolated hot functions with a real harness (warm-up, multiple
  iterations, statistical output) — never a single `start/stop` timer. Beware compiler dead-code
  elimination and cold-cache artifacts.
- **[MUST NOT]** Ship a micro-benchmark win without an end-to-end measurement confirming it moves
  the real p95 — local speedups often vanish behind I/O.

## Enforcing budgets in CI

- **[MUST]** Every budget is a **failing CI gate**, not a dashboard:
  - Frontend: bundle-size check (fail if initial chunk > 250 KB gz) + Lighthouse-CI assertions
    (LCP/INP/CLS thresholds).
  - Backend: a load-test step with pass/fail thresholds on p95/p99/error-rate for key endpoints.
  - DB: fail CI if a migration adds an interactive query path with no supporting index / an
    unexpected Seq Scan on a large table (plan check on seeded data).
- **[MUST]** Gates run on a **stable, representative runner** (fixed resources, seeded data) so
  results are comparable run-to-run. See [../standards-kb/12-devops-cicd.md](../standards-kb/12-devops-cicd.md).

## Regression detection

- **[MUST]** Store per-build results (bundle size, endpoint p95, key query timings) as a **time
  series** and **fail on relative regression** beyond a threshold (e.g. p95 +10% or bundle +5% vs
  the rolling baseline), not only on the absolute budget — catch the slow creep before it breaches.
- **[SHOULD]** Use noise-tolerant comparison (compare medians of N runs, allow a small band) to
  avoid flaky failures; auto-open an issue with the flame graph / trace on regression.
- **[SHOULD]** Post-deploy, **canary** and watch RUM + APM p95 against budget; auto-rollback on
  sustained breach (ties to [../standards-kb/05-reliability-resilience.md](../standards-kb/05-reliability-resilience.md)).

> **Example (regression caught in CI):** a dependency bump added 60 KB to the initial chunk
> (238 → 298 KB). The bundle gate failed the PR; the team lazy-loaded the offending module and
> landed at 231 KB — the budget never regressed in production.

## Acceptance checklist

- [ ] Latency reported at p50/p95/p99 on representative, production-scale multi-tenant data.
- [ ] Distributed tracing + RED/USE dashboards; alerts fire on p95 budget breach.
- [ ] Hot paths profiled (flame + allocation); slow/new queries checked with EXPLAIN ANALYZE.
- [ ] Web Vitals measured in lab and field, segmented by device and tenant.
- [ ] Load tests (steady, spike, soak, hot-tenant) with pass/fail thresholds encoding the budgets.
- [ ] Multi-tenant isolation validated under load — no cross-tenant p95 impact.
- [ ] All budgets are failing CI gates (bundle size, Lighthouse, load-test, query-plan) on a stable runner.
- [ ] Per-build results tracked as a time series; relative regressions fail the build; canary + auto-rollback post-deploy.
