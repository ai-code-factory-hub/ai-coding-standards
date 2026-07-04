# Data KB · The Governed Analytics & Reporting Layer

**Purpose.** A reusable, project-agnostic knowledge base for the **analytics/reporting data layer** that powers report builders, dashboards, and churn/retention analysis in an enterprise SaaS product. It defines *how* self-serve reporting reads data safely — tenant-scoped, permissioned, allow-listed, and fast — and *what* the canonical business metrics mean so every surface computes them the same way.

## Why a governed data layer matters

Left ungoverned, "just let users build reports over the tables" becomes a support and security fire:

- **Full-table scans** and unbounded joins issued from a UI melt the primary database.
- **Tenant leakage** — a report that forgets a `tenant_id` filter exposes one customer's data to another.
- **Metric drift** — the dashboard says churn is 4%, the board deck says 6%, finance says 5%, because each computed it differently.
- **Coupling** — reporting queries written against raw app tables break every time the app schema changes.

A **governed layer** fixes all four by putting a curated, allow-listed model between users and the data, with a **single source of truth for every metric definition**.

## OLTP app tables vs the analytics/reporting layer

These are two different jobs and should not be the same surface.

| | **OLTP (app tables)** | **Analytics/reporting layer** |
|---|---|---|
| Optimised for | Many small, low-latency reads/writes | Large scans, aggregation, grouping |
| Shape | Normalised, row-oriented | Denormalised marts / star schemas, often columnar |
| Access pattern | Point lookups by key, single tenant | Wide filters, group-by, time series |
| Who queries | Application code | Report builders, dashboards, analysts |
| Risk if mixed | Analytics queries starve OLTP; ad-hoc SQL leaks tenants | — |
| Governance | Enforced by app code + DB constraints | Enforced by a **semantic layer**: allow-listed fields, forced tenant scope, row/column permissions |

**[MUST]** Serve heavy analytics from a **separate read path** (read replica, warehouse, or pre-aggregated marts) — never let a user-built report run unbounded against the primary OLTP database. See [../standards-kb/03-performance-scalability.md](../standards-kb/03-performance-scalability.md).

## What this KB governs

- A **semantic/metadata layer**: the allow-listed set of fields, dimensions, and measures a report builder may touch — with tenant scope and permissions baked in, so no report can scan a full table or cross a tenant boundary.
- **Storage topology**: when to use a read replica vs a warehouse vs pre-aggregated marts, and how ETL/ELT and freshness SLAs feed them.
- **Canonical metrics**: one correct definition per SaaS metric (MRR, churn, retention, activation, LTV/CAC …) that every dashboard and export shares.

## Index

- [01-analytics-data-layer.md](01-analytics-data-layer.md) — the governed semantic layer (fields/dimensions/measures, allow-listed queryable model, tenant scope, permissions), storage topology (warehouse vs replica vs marts), and ETL/ELT + freshness. This is the backbone a Report Builder feature consumes.
- [02-metrics-and-churn.md](02-metrics-and-churn.md) — canonical SaaS metric definitions and how to compute them correctly, with paid/unpaid/trial segmentation and filter dimensions.

## Related standards

- [../standards-kb/03-performance-scalability.md](../standards-kb/03-performance-scalability.md) — no full-table scans; pre-aggregation, keyset pagination, replicas.
- [../standards-kb/06-data-management.md](../standards-kb/06-data-management.md) — modeling, indexing, retention, object storage, lineage.
- [../standards-kb/17-nfr-coverage-gaps.md](../standards-kb/17-nfr-coverage-gaps.md) — tenant isolation and data-governance non-negotiables.

## Acceptance checklist

- [ ] Analytics reads run on a separate path from OLTP; no user report scans the primary DB unbounded.
- [ ] A semantic layer allow-lists queryable fields/dimensions/measures with forced tenant scope and permissions.
- [ ] Every metric has exactly one canonical definition consumed by all surfaces.
- [ ] Storage topology (replica/warehouse/marts) and freshness SLAs are documented per report.
