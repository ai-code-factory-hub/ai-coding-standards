# Feature Blueprint · Global Search

One search box across every entity — tenant-scoped, permission-filtered, ranked, typeahead, recent-first — backed by a real search index (never `LIKE '%x%'` table scans).

## What it is / when you need it

Once an app has more than a couple of list screens, users want a single place to find *anything* — a customer, an invoice, a report, a setting. Global search unifies those into one **tenant-scoped index** with **typeahead**, **recent-first** suggestions, **ranking**, and **permission filtering** so a result never leaks something the user can't open. It is deliberately backed by a full-text / search engine index kept fresh **asynchronously**, not by scanning source tables with wildcard `LIKE`.

## Data model

> All tables carry the standard audit columns (`id`, `tenant_id`, `created_at`, `created_by`, `updated_at`, `updated_by`, `deleted_at`, `version`) and are filtered by `tenant_id` on every query.

| Entity | Key fields | Notes |
| --- | --- | --- |
| `search_document` | `entity_type` (e.g. customer/invoice/report), `entity_id`, `title`, `body` (searchable text), `keywords[]`, `deep_link`, `acl_subjects[]` (roles/groups/users who may see it), `boost` (numeric), `indexed_at` | The denormalised, indexed projection of a source record. Lives in the search engine (FTS/OpenSearch/Elastic/pg tsvector). |
| `search_index_config` | `entity_type`, `searchable_fields[]`, `weights` (field→weight), `analyzer/locale`, `is_enabled` | Which entities are indexed and how they're ranked. |
| `search_recent` | `user_id`, `query` OR `entity_type`+`entity_id`, `context`, `last_at`, `hit_count` | Recent searches / recently opened per user, for recent-first suggestions. |
| `search_saved` | `user_id`, `name`, `query`, `filters`, `is_shared` | Optional saved searches. |
| `search_index_event` | `entity_type`, `entity_id`, `op` (upsert/delete), `enqueued_at`, `processed_at`, `status` | Async reindex queue: source change → index update. |
| `search_synonym` | `term`, `synonyms[]`, `scope` (tenant/global) | Optional query expansion. |

## Key screens & UX

| Screen | Primary fields / actions | States |
| --- | --- | --- |
| **Command/Search bar** (global) | Keyboard shortcut (`/` or ⌘K) opens overlay; typeahead as you type; grouped by entity type; keyboard nav; Enter opens deep link | idle (shows recent + suggestions) / typing / results / no-results ("No matches for '…'") / error |
| **Recent & suggestions** | Recent searches, recently opened items, pinned/frequent entities before any keystroke | empty ("Start typing to search") |
| **Full results page** | Query box, entity-type facets, filters, sort (relevance / recency), paginated results with title + snippet + highlighted match + breadcrumb/type badge | loading / empty / error |
| **Saved searches** (optional) | Name, run, edit, share within tenant | empty |
| **Search admin** (admin) | Toggle indexed entities, field weights, synonyms, reindex trigger, index health/lag | ok / reindexing / degraded |

## Rules & logic

- **[MUST]** Search is served from a **dedicated index** (FTS / search engine), **never** `LIKE '%term%'` or `ILIKE` scans over source tables — those don't scale and can't rank. See performance standard.
- **[MUST]** Every query is **tenant-scoped** at the index level (per-tenant index or a mandatory `tenant_id` filter that cannot be omitted).
- **[MUST]** Results are **permission-filtered**: the index stores `acl_subjects`, and the query intersects them with the caller's roles/groups/identity **before** returning. A user must never see a title/snippet for a record they cannot open (no existence leakage).
- **[MUST]** The index is updated **asynchronously** on source change (event/queue → upsert/delete in the index). Source writes never block on indexing; deletes and soft-deletes remove/hide the document.
- **[MUST]** **Typeahead is debounced and capped** (min chars, max results, short timeout); it degrades gracefully to "type more to search" rather than firing heavy queries per keystroke.
- **[MUST]** **Ranking** combines text relevance (field weights, exact/prefix boosts) with signals like recency and per-entity `boost`; recent/opened items surface first before a query is typed.
- **[SHOULD]** Highlight matched terms in snippets; group results by entity type with per-type "see all".
- **[SHOULD]** Support synonyms, basic typo tolerance/fuzziness, and locale-aware analysis.
- **[SHOULD]** Track query → click for relevance tuning; expose index lag as a health metric.
- **[SHOULD]** Cap query cost (result size, timeout) and rate-limit per user to protect the cluster.

## Applicable standards

- [`../standards-kb/03-performance-scalability.md`](../standards-kb/03-performance-scalability.md) — no wildcard scans, indexed search, query cost caps, pagination.
- [`../standards-kb/04-security.md`](../standards-kb/04-security.md) — permission-filtered results, no existence leakage, tenant isolation in the index.
- [`../standards-kb/06-data-management.md`](../standards-kb/06-data-management.md) — index as a projection, async reindex, delete propagation.
- [`../standards-kb/05-reliability-resilience.md`](../standards-kb/05-reliability-resilience.md) — async index queue, backpressure, graceful degradation.
- [`../standards-kb/07-observability-aiops.md`](../standards-kb/07-observability-aiops.md) — index lag, query latency, zero-result rate.
- [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md) — typeahead states, keyboard nav, empty/error states.
- [`../feature-blueprints/README.md`](../feature-blueprints/README.md) — shared conventions.

## Acceptance checklist

- [ ] Search served from a real index; zero `LIKE '%x%'` / `ILIKE` scans on source tables.
- [ ] Every query is tenant-scoped at the index level.
- [ ] Results are permission-filtered before return; no title/snippet leaks an inaccessible record.
- [ ] Index updates are async on create/update/delete; source writes never block; deletes propagate.
- [ ] Typeahead is debounced, min-char gated, capped, and degrades gracefully.
- [ ] Ranking blends relevance + recency + boost; recent/opened surface before typing.
- [ ] Matched terms highlighted; results grouped by entity type; pagination on the full page.
- [ ] Synonyms / fuzziness / locale analysis supported where enabled.
- [ ] Query latency, index lag, and zero-result rate are observable.
- [ ] Loading / empty / error / no-results states defined on every search surface.
- [ ] All tables tenant-scoped with standard audit columns.
