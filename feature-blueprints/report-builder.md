# Feature Blueprint · Report Builder

> A drag-and-drop report/format designer that lets users build, save, share, schedule, and export reports over a **governed metadata layer** — without writing SQL.

## What it is / when you need it

An in-app, self-service reporting tool. Users drag governed fields onto a canvas, pick columns/filters/grouping/sort/aggregations, arrange a print layout (A4 / thermal), then save, share, schedule, and export (PDF/CSV/XLSX). You need it the moment customers start asking for "one more column" or exporting to spreadsheets to do their own analysis. The **field palette is not raw tables** — it comes from a curated semantic layer (see [`../data-kb/01-analytics-data-layer.md`](../data-kb/01-analytics-data-layer.md)) so users can only touch approved, correctly-joined, permission-safe fields.

## Data model

All tables are tenant-scoped and carry standard audit columns (`id`, `tenant_id`, `created_at/by`, `updated_at/by`, `deleted_at`, `version`).

### ReportDefinition
The saved report/format. PK `id`.

| Field | Type | Notes |
| --- | --- | --- |
| `name` | string | Display name; unique per tenant + folder |
| `description` | text | Optional |
| `dataset_key` | string | FK to a governed dataset in the metadata layer |
| `layout_type` | enum | `table` \| `grouped` \| `pivot` \| `print_template` |
| `page_format` | enum | `a4` \| `a5` \| `letter` \| `thermal_58mm` \| `thermal_80mm` |
| `orientation` | enum | `portrait` \| `landscape` |
| `visibility` | enum | `private` \| `shared_roles` \| `tenant_public` |
| `owner_user_id` | fk | Creator/owner |
| `row_limit` | int | Hard cap (governance); default from dataset |
| `is_template` | bool | System/starter template flag |
| `canvas_json` | jsonb | Full designer layout (positions, print blocks, headers/footers) |
| `status` | enum | `draft` \| `published` \| `archived` |

### ReportField
A column/field placed on the report.

| Field | Type | Notes |
| --- | --- | --- |
| `report_id` | fk | → ReportDefinition |
| `field_key` | string | FK to metadata-layer field (governed) |
| `label` | string | Override header text |
| `position` | int | Column order |
| `aggregation` | enum | `none` \| `sum` \| `avg` \| `min` \| `max` \| `count` \| `count_distinct` |
| `group_level` | int? | Grouping tier (null = not grouped) |
| `sort_dir` | enum? | `asc` \| `desc` \| null |
| `sort_priority` | int? | Multi-column sort order |
| `format_mask` | string | e.g. `#,##0.00`, date/currency mask |
| `width_px` | int | Rendered width |
| `is_visible` | bool | Hidden helper columns allowed |

### ReportFilter
A filter/parameter applied to the query.

| Field | Type | Notes |
| --- | --- | --- |
| `report_id` | fk | → ReportDefinition |
| `field_key` | string | Governed field |
| `operator` | enum | `eq` \| `neq` \| `in` \| `between` \| `gte` \| `lte` \| `contains` \| `is_null` |
| `value_json` | jsonb | Static value(s) |
| `is_parameter` | bool | If true, prompt user at run time |
| `param_label` | string | Prompt label |
| `required` | bool | Must be supplied to run |
| `default_json` | jsonb | Default value (e.g. `last_30_days`) |

### ReportSchedule
Unattended, recurring runs delivered by email/webhook/file drop.

| Field | Type | Notes |
| --- | --- | --- |
| `report_id` | fk | → ReportDefinition |
| `cron` | string | Timezone-aware schedule |
| `timezone` | string | IANA tz |
| `export_format` | enum | `pdf` \| `csv` \| `xlsx` |
| `delivery` | enum | `email` \| `webhook` \| `sftp` \| `in_app` |
| `recipients_json` | jsonb | Emails / channel refs |
| `param_values_json` | jsonb | Bound parameter values |
| `is_active` | bool | Pause without deleting |
| `last_run_at` | timestamp | |
| `next_run_at` | timestamp | Computed |

### ReportRun
An execution record (interactive or scheduled) — powers history, re-download, and governance.

| Field | Type | Notes |
| --- | --- | --- |
| `report_id` | fk | → ReportDefinition |
| `schedule_id` | fk? | Null if interactive |
| `triggered_by` | enum | `user` \| `schedule` \| `api` |
| `run_by_user_id` | fk? | Who ran it |
| `params_json` | jsonb | Bound params at run time |
| `status` | enum | `queued` \| `running` \| `succeeded` \| `failed` \| `truncated` \| `cancelled` |
| `row_count` | int | Rows returned |
| `duration_ms` | int | Query + render time |
| `bytes_scanned` | bigint | For cost/governance |
| `output_uri` | string | Signed URL to artifact (expiring) |
| `error_message` | text? | On failure |
| `expires_at` | timestamp | Artifact retention |

## Key screens & UX

1. **Report list / library** — folders, search, filter by owner/visibility/status, "My reports" vs "Shared". Actions: New, Duplicate, Run, Schedule, Share, Archive. *Loading:* skeleton rows. *Empty:* "No reports yet — start from a template or blank." *Error:* retry banner.
2. **Designer canvas** — three panes: **field palette** (governed, searchable, grouped by entity; shows type + description; disabled fields the user isn't entitled to are hidden), **canvas** (drag fields to columns/groups; drop zones for group-by and print blocks), **inspector** (per-field label/aggregation/format/sort). Live preview with sample/limited rows. *States:* palette loading skeleton; empty canvas prompt; invalid-field error toast.
3. **Filters & parameters** — build filter rows; mark as run-time parameter; set defaults/required. Preview parameter prompt.
4. **Layout / print template** — for `print_template`: header/footer, logo, page size (A4/thermal), margins, repeating group headers, page breaks, totals rows. Print-preview at true size.
5. **Run / result viewer** — bound parameter prompt → results grid with column resize, client sort within page, export buttons (PDF/CSV/XLSX), "truncated at N rows" banner when the row cap hits.
6. **Schedule dialog** — cron builder, timezone, format, recipients, bound params; shows next 3 run times.
7. **Run history** — per report: runs with status, rows, duration, who, re-download (until `expires_at`). *Empty:* "No runs yet."
8. **Share dialog** — set visibility, pick roles/users; explains that recipients still only see rows their own permissions allow (row-level security is enforced at run time, not baked in).

## Rules & logic

- [MUST] The field palette is sourced **only** from the governed metadata layer — no raw table/column access. Fields carry types, formats, and join semantics from that layer.
- [MUST] Enforce **query governance**: no full-table scans; every run has a hard `row_limit` and a query timeout; reject queries a governance check flags (missing indexed filter, cartesian join). Record `bytes_scanned` and `duration_ms` on every `ReportRun`.
- [MUST] Apply **row-level security at run time** for the *running* user — sharing a report never shares data the recipient couldn't otherwise see.
- [MUST] All definitions, runs, and outputs are **tenant-scoped**; output artifacts use expiring signed URLs and honor `expires_at` retention.
- [MUST] Aggregations must be consistent with grouping — a non-aggregated, non-grouped field alongside aggregates is rejected with a clear message.
- [MUST] Scheduled runs are idempotent per `(schedule_id, scheduled_time)`; a missed window does not double-fire.
- [SHOULD] Provide starter **templates** (`is_template`) users can duplicate.
- [SHOULD] Cache dataset metadata and sample previews; debounce live preview while dragging.
- [SHOULD] Support pivot/grouped totals and subtotals; export preserves formatting masks.
- [SHOULD] Large exports run **async** via `ReportRun` and notify on completion rather than blocking the UI.

## Applicable standards

- Governed metadata / semantic layer — [`../data-kb/01-analytics-data-layer.md`](../data-kb/01-analytics-data-layer.md)
- Data management, retention, PII in exports — [`../standards-kb/06-data-management.md`](../standards-kb/06-data-management.md)
- Multitenancy & row-level security — [`../standards-kb/01-architecture-multitenancy.md`](../standards-kb/01-architecture-multitenancy.md)
- Frontend/UX (drag-drop, states, print) — [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md)
- API/integration (export, webhook delivery) — [`../standards-kb/10-api-integration.md`](../standards-kb/10-api-integration.md)
- Query cost governance — [`../standards-kb/16-finops-cost.md`](../standards-kb/16-finops-cost.md)
- Run observability/audit — [`../standards-kb/07-observability-aiops.md`](../standards-kb/07-observability-aiops.md)

## Acceptance checklist

- [ ] Palette shows only governed fields the user is entitled to; each field shows type + description.
- [ ] Can build a report by drag-drop: columns, group-by, sort (multi), filters, aggregations.
- [ ] Run-time parameters prompt correctly, with defaults and required validation.
- [ ] Every run is capped by `row_limit` + timeout; truncation is surfaced, not silent.
- [ ] Row-level security enforced for the *running* user on shared reports.
- [ ] Export produces correct PDF/CSV/XLSX with formatting; large exports run async.
- [ ] Print template renders correctly at A4 and thermal widths.
- [ ] Schedules fire on cron in the right timezone, idempotently, with delivery.
- [ ] Run history records status/rows/duration/bytes and allows re-download until expiry.
- [ ] All entities tenant-scoped; artifacts use expiring signed URLs and honor retention.
