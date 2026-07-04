# 01 · The Analytics Data Layer

**Purpose.** Define the **governed semantic layer** and storage topology that let a Report Builder and dashboards query data *safely and fast*: an allow-listed model of fields, dimensions, and measures; forced tenant scoping and permissions; a separated read path (replica / warehouse / marts); and ETL/ELT with freshness SLAs. This is the backbone a `feature-blueprints/report-builder.md` consumes — the report UI generates queries **only** against this model, never raw SQL over app tables. See [README.md](README.md).

## The semantic layer — vocabulary

The semantic layer is metadata: a curated contract between "what the business means" and "what SQL runs."

- **Field** — a governed, human-named attribute (e.g. `customer.plan_name`), mapped to a physical column/expression, with a type, a description, and visibility rules.
- **Dimension** — a field you **group by / filter on** (time, plan, region, segment, cohort).
- **Measure** — an **aggregation** with a fixed formula (e.g. `mrr = SUM(subscription.mrr_amount)`, `active_customers = COUNT(DISTINCT customer.id) WHERE status='active'`). A measure encodes the *one* correct way to compute a number — see [02-metrics-and-churn.md](02-metrics-and-churn.md).
- **Model/Explore** — an allow-listed join graph of fields+measures that a report may traverse. If a field isn't in the model, it isn't queryable. Full stop.

> **Example A — a semantic model definition (illustrative YAML).**
> ```
> model: subscriptions
> tenant_key: tenant_id            # ALWAYS injected into the WHERE clause
> primary_source: mart_subscription_daily
> dimensions:
>   - name: plan_name    type: string   sql: ${source}.plan_name
>   - name: billed_month type: date     sql: date_trunc('month', ${source}.event_date)
>   - name: segment      type: string   sql: ${source}.segment   # paid|trial|free
> measures:
>   - name: mrr          type: sum      sql: ${source}.mrr_amount
>   - name: customers    type: count_distinct sql: ${source}.customer_id
>   - name: gross_churn  type: derived  sql: ${churned_mrr} / nullif(${starting_mrr},0)
> permissions:
>   row:    tenant_id = :current_tenant           # non-negotiable
>   column: segment != 'internal' unless role in (admin, finance)
> ```

## Rules — safe queryable model

- **[MUST]** A Report Builder emits queries **only** against the semantic model's allow-listed fields/measures. Users never write raw SQL against production tables; the layer compiles their selections to SQL.
- **[MUST]** **Force tenant scope** at compile time — every generated query gets `tenant_id = :current_tenant` injected server-side, not supplied by the client. A report cannot opt out. See [../standards-kb/17-nfr-coverage-gaps.md](../standards-kb/17-nfr-coverage-gaps.md).
- **[MUST]** **No full-table scans.** Every model declares an indexed/partitioned access path and a **default time bound**; unbounded queries are rejected. Enforce row limits, query timeouts, and byte-scanned caps. See [../standards-kb/03-performance-scalability.md](../standards-kb/03-performance-scalability.md).
- **[MUST]** **Permission-filter rows and columns** by role: sensitive dimensions (cost, PII, internal segments) are masked or withheld unless the user's role allows them. Enforce in the compiler, not the UI.
- **[SHOULD]** Version the semantic layer; a field rename/removal is a reviewed migration, because saved reports depend on it. Keep field names stable; deprecate before deleting.
- **[SHOULD]** Return **query cost estimates** and reject/queue expensive ad-hoc reports (async job + polling) rather than running them inline.

## Storage topology — where reports read from

Pick per workload; document the choice per report. Ordered roughly by scale/latency of the reporting need.

| Option | Best for | Freshness | Watch out for |
|--------|----------|-----------|---------------|
| **Read replica** of OLTP | Small tenants, operational reports, near-real-time | Seconds (replication lag) | Still row-oriented; heavy aggregation can lag replicas; keep queries bounded |
| **Data warehouse** (columnar) | Cross-tenant analytics, large scans, complex joins | Minutes–hours (batch/stream load) | Cost per byte scanned; needs the semantic layer to prevent runaway queries |
| **Pre-aggregated marts** (star schema / rollup tables / materialized views) | Dashboards, churn/MRR trends, high-traffic canned reports | As fresh as the refresh job | Combinatorial explosion if you pre-aggregate every dimension — roll up only the hot ones |

- **[MUST]** Serve high-traffic dashboards from **pre-aggregated marts/materialized views**, not live scans, so the same expensive group-by isn't recomputed on every page load.
- **[SHOULD]** Use a **replica for freshness-sensitive operational reports** and a **warehouse for heavy/cross-tenant analytics**; route by the report's freshness SLA.

## ETL/ELT and freshness

- **[MUST]** Define a **freshness SLA per report/mart** ("data as of ≤ 15 min / ≤ 24 h") and **surface the as-of timestamp** in the UI so users trust the number.
- **[MUST]** Make transform jobs **idempotent and incremental** (watermark/high-water-mark on `updated_at` or CDC), so a re-run doesn't double-count and a backfill is safe. See [../standards-kb/06-data-management.md](../standards-kb/06-data-management.md).
- **[SHOULD]** Prefer **ELT** (load raw, transform in the warehouse) for flexibility and lineage; use **ETL** when you must not land raw PII in the warehouse.
- **[SHOULD]** Monitor pipeline **freshness, row-count deltas, and null/skew anomalies**; alert when a mart misses its SLA. Refresh marts on a schedule sized to the SLA, incrementally where possible.
- **[SHOULD]** Keep **lineage** from mart → source columns so a metric can be traced and audited.

> **Example B — bounded, tenant-scoped compiled query (what the layer emits).**
> ```
> -- user picked: measure=mrr, dimension=billed_month, filter plan_name='Pro'
> SELECT date_trunc('month', event_date) AS billed_month,
>        SUM(mrr_amount)                 AS mrr
> FROM   mart_subscription_daily
> WHERE  tenant_id = :current_tenant           -- injected, non-optional
>   AND  event_date >= :from AND event_date < :to   -- default time bound
>   AND  plan_name = 'Pro'
> GROUP  BY 1 ORDER BY 1
> LIMIT  10000;                                 -- hard row cap
> ```

## Acceptance checklist

- [ ] Reports query only allow-listed semantic fields/dimensions/measures — never raw SQL over app tables.
- [ ] Tenant scope is injected server-side into every generated query and cannot be bypassed.
- [ ] Row/column permissions are enforced in the query compiler; every model has a bounded, indexed access path with row/timeout/bytes caps.
- [ ] Storage topology (replica/warehouse/marts) is chosen per report by its freshness SLA; hot dashboards read pre-aggregated marts.
- [ ] Each mart has a documented freshness SLA, an as-of timestamp shown in the UI, and idempotent incremental transforms with lineage.
- [ ] Measures encode the single canonical formula defined in [02-metrics-and-churn.md](02-metrics-and-churn.md).
