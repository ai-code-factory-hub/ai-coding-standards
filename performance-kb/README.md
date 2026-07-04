# Performance KB — Buildable performance blueprint (project-agnostic)

Deep, code-level performance patterns for any enterprise SaaS built with this kit. This is a
**buildable blueprint**: concrete techniques, sizing formulas, and short language-neutral
pseudocode you can lift into a real service — not vague advice.

## How this extends the Standards KB

The standard **[../standards-kb/03-performance-scalability.md](../standards-kb/03-performance-scalability.md)**
sets the **rules and budgets** (what MUST be true to ship). This KB does **not** repeat those
rules — it shows **how** to implement them at depth. Read the standard first for the "what/why";
read here for the "how".

| Standard 03 says… | This KB shows how, in… |
|---|---|
| "All external connections are pooled with sized limits, acquire timeouts, max lifetime" | [01 · Pooling & Resources](01-pooling-and-resources.md) |
| "Index for query patterns; no N+1; short transactions; keyset pagination" | [02 · Query & Statement](02-query-and-statement.md) |
| "Stream don't buffer; bound and evict caches; async for slow work" | [03 · Code-Level Performance](03-code-level-performance.md) |
| "Minimize fields per screen; paginate/virtualize; lazy-load routes" | [04 · Frontend Performance](04-frontend-performance.md) |
| "Budgets are tested in CI; regressions fail the build" | [05 · Profiling & Benchmarks](05-profiling-and-benchmarks.md) |

## Rule language (RFC 2119)

- **[MUST] / [MUST NOT]** — hard requirement; no ship without it (or a logged, approved waiver).
- **[SHOULD] / [SHOULD NOT]** — strong default; deviation needs a written reason.
- **[MAY]** — optional; use judgement.

Tools are named only as `> **Example:**` illustrations — the rules themselves are language- and
vendor-neutral.

## Performance budgets (recap — canonical source is Standard 03)

- **[MUST]** **API p95 < 300 ms**, p99 < 1 s for interactive reads; writes < 1 s p95.
- **[MUST]** **LCP < 2.5 s**, **FCP < 1.8 s**, **INP < 200 ms** on a mid-range device; CLS < 0.1.
- **[MUST]** **Initial JS bundle < 250 KB gzipped**.
- **[MUST]** No interactive DB query > 100 ms without an index or documented justification.
- **[MUST]** Budgets are enforced in CI; a regression fails the build.

Treat every budget as a **per-request/per-screen error budget**: measure at p95/p99, not the mean.

## Index

1. **[01 · Pooling & Resources](01-pooling-and-resources.md)** — connection/thread/HTTP pools, sizing formulas, exhaustion, per-tenant bulkheads.
2. **[02 · Query & Statement](02-query-and-statement.md)** — prepared statements, N+1, indexing, keyset pagination, replicas, materialized views.
3. **[03 · Code-Level Performance](03-code-level-performance.md)** — hot paths, allocation/GC, streaming, memoization tiers, batch I/O, async.
4. **[04 · Frontend Performance](04-frontend-performance.md)** — bundle splitting, virtualization, render cost, Core Web Vitals, images, CDN.
5. **[05 · Profiling & Benchmarks](05-profiling-and-benchmarks.md)** — APM, flame graphs, EXPLAIN ANALYZE, k6 load tests, CI budget gates, regression detection.

## Related standards

- [03 · Performance & Scalability](../standards-kb/03-performance-scalability.md) — the parent rules/budgets.
- [05 · Reliability & Resilience](../standards-kb/05-reliability-resilience.md) — timeouts, retries, breakers on the calls you pool.
- [06 · Data Management](../standards-kb/06-data-management.md) — schema, indexing, object storage.
- [08 · Testing & QA](../standards-kb/08-testing-qa.md) — where load/perf tests live in the pyramid.
- [12 · DevOps & CI/CD](../standards-kb/12-devops-cicd.md) — running budget gates in the pipeline.
- [19 · NFR Implementation Playbook](../standards-kb/19-nfr-implementation-playbook.md) — sequencing NFR work.
