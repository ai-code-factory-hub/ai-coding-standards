# Threat Model — Multi-Tenant Data Access

> Filled STRIDE model for isolating tenants that share the same application, database, cache, and search indices. Method & risk rating: [README.md](README.md). Blank copy: [threat-model-template.md](threat-model-template.md).
> **Owner:** Security Officer · **Extends:** [01 Architecture & Multitenancy](../standards-kb/01-architecture-multitenancy.md), [06 Data Management](../standards-kb/06-data-management.md)

## Feature / asset

Any data path in a shared-infrastructure SaaS where records belong to a tenant. **Asset:** tenant isolation itself — the invariant that Tenant A can never see or affect Tenant B's data. A single missed filter is a cross-tenant breach that can affect every customer at once. **Tenant isolation is an authorization invariant, not a query convention.**

## Data classification

Assume up to **Confidential/PII/Regulated** per tenant. Cross-tenant leakage is almost always a reportable incident.

## Actors

| Actor | Trusted? | Notes |
|---|---|---|
| Tenant A user | Trusted **within tenant A only** | the boundary being enforced |
| Tenant B user | **Untrusted w.r.t. A's data** | legitimate elsewhere, adversary here |
| Platform/background job | Semi-trusted | runs cross-tenant; easy to forget the filter |
| Shared cache / search index | Semi-trusted | a leaky store that spans tenants |

## Trust boundaries

- **B3 Tenant boundary** — the central boundary. Present in every query, cache key, index, export, and background job.

## Attack surface

Every query (esp. joins, aggregates, admin/reporting queries), background jobs and migrations, cache keys, search indices, generated export files, object ids in URLs, and any denormalized/duplicated store.

## STRIDE analysis

| # | STRIDE | Threat | How (attack path) | Mitigation `[MUST]`/`[SHOULD]` | Standards ref | Residual |
|---|---|---|---|---|---|---|
| S1 | Spoofing | **Tenant-context spoofing** | Client supplies/overrides `tenant_id` (header, JWT claim, body) to act as another tenant | `[MUST]` derive tenant **server-side from the authenticated session**, never from client input; reject requests where a client-supplied tenant differs from the session's | [04](../standards-kb/04-security.md) / [01](../standards-kb/01-architecture-multitenancy.md) | Low |
| T1 | Tampering | **Cross-tenant write** | IDOR on a write path lets A modify/delete B's record | `[MUST]` every write authorizes owner+tenant on the target object; deny by default; foreign-key scoping to the tenant | [04](../standards-kb/04-security.md) | Med |
| R1 | Repudiation | **Untraceable cross-tenant access** | A cross-tenant read happens but can't be attributed/detected | `[MUST]` audit include tenant id on every sensitive access; alert on any query returning rows outside the session tenant | [20](../standards-kb/20-logging-audit-and-traceability.md) | Low |
| I1 | Info disclosure | **Missing tenant filter** | A query (often a new report, join, or aggregate) omits the `WHERE tenant_id = ?` predicate and returns all tenants' rows | `[MUST]` enforce tenant scoping in a **central data layer / query filter**, not per-query `if`s; default-scope all models; code-review + tests specifically for the filter; `[SHOULD]` a lint/analyzer that flags unscoped queries | [06](../standards-kb/06-data-management.md) / [01](../standards-kb/01-architecture-multitenancy.md) | **Med** |
| I2 | Info disclosure | **RLS / policy bypass** | Row-Level Security disabled, a `BYPASSRLS`/superuser role used, or the session tenant variable unset for a connection | `[MUST]` if using DB RLS, force it on every table + connection role; never run app traffic as an RLS-bypassing role; set + verify the tenant session variable per request; fail closed if unset | [06](../standards-kb/06-data-management.md) | Med |
| I3 | Info disclosure | **Shared cache / index bleed** | Cache key or search index omits tenant, so A's cached result or search hit is served to B | `[MUST]` include tenant id in **every** cache key and as a mandatory filter on every search query; separate indices/namespaces per tenant where feasible; never cache a query result without its tenant scope | [06](../standards-kb/06-data-management.md) / [03](../standards-kb/03-performance-scalability.md) | Med |
| I4 | Info disclosure | **Export crossing tenants** | A generated report/CSV/backup includes rows from other tenants, or an export link is guessable | `[MUST]` apply tenant scope to the export query itself; authorize the download by tenant; signed, expiring, non-guessable export URLs; verify row-tenant before write (see [Search & Export](search-and-data-export.md)) | [06](../standards-kb/06-data-management.md) / [09](../standards-kb/09-compliance-privacy.md) | Med |
| D1 | DoS | **Noisy-neighbor exhaustion** | One tenant's load/heavy query starves others sharing the pool | `[MUST]` per-tenant rate limits and query/resource quotas; isolate or throttle expensive operations | [03](../standards-kb/03-performance-scalability.md) / [05](../standards-kb/05-reliability-resilience.md) | Med |
| E1 | Elevation | **Background job runs cross-tenant** | A job/migration/admin task iterates all tenants but writes/reads without re-scoping, or a bug elevates a user across tenants | `[MUST]` background jobs carry an explicit tenant context and re-apply scoping per tenant; no ambient "all tenants" access in request paths | [01](../standards-kb/01-architecture-multitenancy.md) / [04](../standards-kb/04-security.md) | Med |

## Top risks

1. **Missing tenant filter (I1)** — the single most common multi-tenant breach; centralize scoping so a forgotten `WHERE` isn't possible.
2. **Shared cache / index bleed (I3)** — filters on the DB but not the cache/search layer; tenant-key everything.
3. **Export crossing tenants (I4)** — bulk reads amplify a small bug into a mass leak.

## Open risks / decisions

| Item | Decision | Owner | Due |
|---|---|---|---|
| Isolation model | Shared-DB + RLS vs. schema/DB-per-tenant — record the choice | | |
| Query-filter enforcement | Central data layer + automated test/lint for unscoped queries | | |

---

## Mitigation checklist

- [ ] `[MUST]` Tenant derived from the session server-side; client-supplied tenant rejected.
- [ ] `[MUST]` Tenant scoping enforced centrally (data layer / default scope), not per-query.
- [ ] `[MUST]` Every read **and** write authorized by tenant (no cross-tenant IDOR).
- [ ] `[MUST]` RLS (if used) forced on all tables + roles; app never runs as bypass role; fail closed if context unset.
- [ ] `[MUST]` Tenant id in every cache key and mandatory in every search query (no bleed).
- [ ] `[MUST]` Exports scoped + authorized by tenant; signed expiring URLs.
- [ ] `[MUST]` Background jobs re-apply tenant scope; no ambient all-tenant access in request paths.
- [ ] `[MUST]` Per-tenant rate/resource quotas (no noisy neighbor).
- [ ] `[MUST]` Tenant id on every audit entry; alert on out-of-tenant result sets → [20](../standards-kb/20-logging-audit-and-traceability.md).
