# Threat Model — Search & Data Export

> Filled STRIDE model for search/query features and bulk data export (CSV/Excel/PDF/API dumps, reports). Method & risk rating: [README.md](README.md). Blank copy: [threat-model-template.md](threat-model-template.md).
> **Owner:** Security Officer · **Extends:** [06 Data Management](../standards-kb/06-data-management.md), [09 Compliance & Privacy](../standards-kb/09-compliance-privacy.md)

## Feature / asset

Full-text/filtered search, reporting queries, and any feature that lets a user pull data out in bulk (export/download/report/API list). **Asset:** the data returned — search and export **amplify** authorization mistakes: one over-broad query returns thousands of records at once, and one export walks them out the door. **Filter for permissions and tenant *before* returning results, not after.**

## Data classification

Whatever is searchable/exportable — assume up to **Confidential/PII/Regulated**. Exports create new copies of data that live outside the app's controls → [06 Data Management](../standards-kb/06-data-management.md), [09 Compliance & Privacy](../standards-kb/09-compliance-privacy.md).

## Actors

| Actor | Trusted? | Notes |
|---|---|---|
| Authenticated searcher/exporter | Semi-trusted | may probe filters / craft queries |
| Anonymous caller | **Untrusted** | injection, scraping |
| Recipient of an export file | Varies | data leaves app controls once exported |
| Background report generator | Semi-trusted | easy to run with wrong scope |

## Trust boundaries

- **B1** query/search endpoint. **B3** tenant boundary (results must not cross it). **Data-egress boundary** — exported files leave the system.

## Attack surface

Search query strings and filter/sort/facet params, raw query passed to the search engine or DB, pagination limits, result-field selection, the export request (format, filters, columns), the generated file, and its download URL.

## STRIDE analysis

| # | STRIDE | Threat | How (attack path) | Mitigation `[MUST]`/`[SHOULD]` | Standards ref | Residual |
|---|---|---|---|---|---|---|
| T1 | Tampering | **Query / search injection** | User input concatenated into SQL / NoSQL / search-DSL / OS command runs attacker queries | `[MUST]` parameterize queries; use the engine's structured query builder, never string concatenation; allow-list sortable/filterable fields; validate operators | [04](../standards-kb/04-security.md) / [06](../standards-kb/06-data-management.md) | Low |
| T2 | Tampering | **Filter/scope manipulation** | Client changes a filter, field selector, or scope param to widen results beyond its rights | `[MUST]` derive the authorization scope server-side (user + tenant + role); apply it as a mandatory base predicate the client cannot remove or override | [04](../standards-kb/04-security.md) | Med |
| R1 | Repudiation | **Untraceable bulk pull** | User denies exporting a large PII dataset | `[MUST]` audit search-heavy and **all export** actions with actor, tenant, filters, row count, format, timestamp, correlation id | [20](../standards-kb/20-logging-audit-and-traceability.md) | Low |
| I1 | Info disclosure | **Over-broad results crossing permissions** | Search/report returns records the user shouldn't see — object- or field-level authz applied after retrieval (or not at all); ranking leaks existence of hidden docs | `[MUST]` apply per-object **and** tenant authorization as a pre-filter in the query itself; field-level redaction for sensitive columns; ensure no-access items don't leak via counts/facets/autocomplete | [04](../standards-kb/04-security.md) / [multi-tenant](multi-tenant-data-access.md) | **Med** |
| I2 | Info disclosure | **PII in exports** | Export includes more PII/columns than needed, or sensitive fields that should be masked | `[MUST]` default exports to the minimum necessary columns; mask/tokenize sensitive fields unless explicitly authorized; data-minimization + purpose limitation; `[SHOULD]` watermark/label exports | [09](../standards-kb/09-compliance-privacy.md) / [06](../standards-kb/06-data-management.md) | Med |
| I3 | Info disclosure | **Export file leakage** | Generated file stored at a guessable/public URL, kept forever, or emailed unencrypted | `[MUST]` serve exports via short-lived, signed, non-guessable URLs; authorize the download by owner+tenant; auto-expire/delete generated files; encrypt at rest; never email raw PII files | [06](../standards-kb/06-data-management.md) / [04](../standards-kb/04-security.md) | Med |
| I4 | Info disclosure | **Formula/CSV injection** | Exported cell like `=CMD()` executes when opened in a spreadsheet, exfiltrating data on the recipient's machine | `[MUST]` sanitize/escape leading `= + - @` in CSV/Excel cells | [06](../standards-kb/06-data-management.md) | Low |
| D1 | DoS | **Expensive-query DoS** | Wildcard/regex/deep-pagination/huge-aggregate or "export everything" query exhausts CPU/memory | `[MUST]` cap result size and page depth; timeouts + query cost limits; forbid unbounded regex/leading wildcards; run large exports as bounded async jobs with per-user concurrency limits | [03](../standards-kb/03-performance-scalability.md) / [05](../standards-kb/05-reliability-resilience.md) | Med |
| D2 | DoS | **Scraping via search/export** | Attacker enumerates the full dataset through paginated search or repeated exports | `[MUST]` per-user rate limits + export quotas; anomaly alerts on bulk pulls; `[SHOULD]` throttle deep pagination | [10](../standards-kb/10-api-integration.md) | Med |
| E1 | Elevation | **Export bypasses field-level authz** | A field hidden in the UI is included in the export/report path | `[MUST]` apply the same field-level authorization to export/report generation as to the interactive view; deny by default | [04](../standards-kb/04-security.md) | Med |

## Top risks

1. **Over-broad results crossing permissions (I1)** — post-filtering or missing object/tenant scope turns search into a data-leak engine; pre-filter in the query.
2. **Export file leakage (I3)** — a guessable or never-expiring download URL leaks the whole export; sign, scope, and expire it.
3. **PII in exports (I2)** — exports create uncontrolled copies of regulated data; minimize columns and mask by default.

## Open risks / decisions

| Item | Decision | Owner | Due |
|---|---|---|---|
| Export retention | Auto-delete generated files after N hours | | |
| Large-export approval | Threshold above which export needs approval / async job | | |

---

## Mitigation checklist

- [ ] `[MUST]` Queries parameterized / structured; sortable/filterable fields allow-listed (no injection).
- [ ] `[MUST]` Authorization scope (user+tenant+role) applied as a mandatory pre-filter the client can't override.
- [ ] `[MUST]` Per-object + field-level authz applied before results/exports; no leakage via counts/facets/autocomplete.
- [ ] `[MUST]` Exports default to minimum columns; sensitive fields masked; CSV/Excel cells escaped (no formula injection).
- [ ] `[MUST]` Export files via signed, expiring, non-guessable URLs; authorized download; encrypted at rest; auto-deleted.
- [ ] `[MUST]` Result-size/page-depth caps, query timeouts/cost limits; large exports as bounded async jobs.
- [ ] `[MUST]` Per-user rate limits + export quotas; anomaly alerts on bulk pulls.
- [ ] `[MUST]` Export/report path enforces the same field-level authz as the interactive view.
- [ ] `[MUST]` Search-heavy and all export actions audited with row counts → [20](../standards-kb/20-logging-audit-and-traceability.md).
