# Feature Blueprint · Import / Export Pipeline

A guided bulk-data pipeline — upload → column mapping → validate/preview → async bulk apply → result report — that is streamed, resumable, idempotent, partial-success, malware-scanned, permission-respecting, tenant-isolated, and audited.

## What it is / when you need it

Every enterprise app eventually needs to get data **in** (migrate from a spreadsheet or legacy system, bulk-create records) and **out** (export for reporting, backup, or another tool). Naïve implementations load the whole file into memory, silently drop bad rows, block the request thread, and leak cross-tenant data. This blueprint is a **guided pipeline**: the user uploads a file, maps columns to fields, previews validation, then an **async job** applies rows in batches with **partial success** (good rows commit, bad rows are reported — never silently dropped) and a downloadable **result report**. Exports mirror the same rigor and always respect the caller's **permissions + filters**.

## Data model

> All tables carry the standard audit columns (`id`, `tenant_id`, `created_at`, `created_by`, `updated_at`, `updated_by`, `deleted_at`, `version`) and are filtered by `tenant_id` on every query.

| Entity | Key fields | Notes |
| --- | --- | --- |
| `import_job` | `entity_type`, `source_file_ref`, `scan_status` (pending/clean/infected), `mapping_id` FK, `status` (uploaded/mapping/validating/ready/running/paused/completed/failed), `mode` (create/upsert/update), `total_rows`, `processed_rows`, `success_rows`, `error_rows`, `idempotency_key`, `checkpoint`, `started_by` | The controlling job; `checkpoint` enables resume. |
| `import_mapping` | `entity_type`, `column_map` (source col → target field), `transforms[]`, `dedup_keys[]`, `saved_name`, `is_reusable` | Reusable saved mapping template. |
| `import_row_result` | `import_job_id` FK, `row_number`, `status` (success/error/skipped), `target_id` (created/updated), `raw_values`, `errors[]` (field, code, message) | Per-row outcome (drives the result report). |
| `import_error_report` | `import_job_id` FK, `file_ref`, `error_count`, `generated_at` | Downloadable file of failed rows + reasons for re-upload. |
| `export_job` | `entity_type`, `filter` (query the user is allowed to run), `columns[]`, `format` (csv/xlsx/json/pdf), `status` (queued/running/completed/failed/expired), `row_count`, `file_ref`, `download_expires_at`, `requested_by` | Async export honouring permissions + filters. |
| `data_job_audit` | `job_type` (import/export), `job_id`, `actor`, `action`, `record_counts`, `at` | Audit trail of who moved what data, when. |

## Key screens & UX

| Screen | Primary fields / actions | States |
| --- | --- | --- |
| **Import: upload** | Drag-drop file, format/size limits, template download, scanning indicator | idle / uploading / scanning / clean / infected (blocked) / error |
| **Import: column mapping** | Auto-detected header → field suggestions, required-field indicators, transforms, dedup key selection, save mapping | mapping / incomplete (required unmapped) |
| **Import: validate & preview** | Sample rows with per-cell pass/fail, error summary by type, counts (valid / invalid), "fix & re-upload" or "import valid rows" | validating / has-errors / ready |
| **Import: run & progress** | Live progress (processed / total, success / error), pause/resume/cancel, ETA | running / paused / completed / failed |
| **Import: result report** | Summary counts, per-row errors table, **download error report**, link to created records | completed (partial or full) |
| **Export: build** | Entity, filters (only ones the user can apply), column picker, format, "export" | idle / queued / running |
| **Export: history** | Past exports, status, row count, **download** (expiring link), re-run | empty / has-items / expired |

## Rules & logic

- **[MUST]** Uploaded files are **malware-scanned** before parsing; `scan_status != clean` blocks the job and the file is quarantined.
- **[MUST]** Files are **streamed/parsed in batches**, never fully loaded into memory; the pipeline handles large files without OOM.
- **[MUST]** Import runs as an **async job** with **checkpointing** so it is **resumable** after failure/restart and does not block the request thread.
- **[MUST]** Import is **idempotent**: an `idempotency_key` + `dedup_keys` prevent duplicate records if the file is re-submitted or the job retries; `upsert` mode keys on stable business identifiers.
- **[MUST]** **Partial success is first-class**: valid rows commit, invalid rows are captured with per-row/field errors, and a **downloadable error report** is produced. **Rows are never silently dropped** — every input row has a recorded outcome (success/error/skipped) and the counts reconcile (`processed = success + error + skipped`).
- **[MUST]** Every write is **validated against the same server-side rules** as the normal create/update path (types, required, referential integrity, business rules) — bulk is not a bypass.
- **[MUST]** **Exports respect the caller's permissions + filters**: an export can only contain rows and columns the requester is authorised to see; row/field-level authorization applies exactly as in the UI. No "export all" that leaks restricted data.
- **[MUST]** Both directions are **tenant-isolated** (`tenant_id` on every read/write) and **audited** (who imported/exported what, counts, when). Export files are stored access-controlled with **expiring** download links.
- **[SHOULD]** Mappings are **reusable/saveable**; provide a downloadable import template per entity.
- **[SHOULD]** Large exports are **async + notified** ("your export is ready") rather than a blocking download; format options include csv/xlsx/json.
- **[SHOULD]** Rate-limit and queue heavy jobs; surface progress and allow cancel; apply retention/TTL to generated files.

## Applicable standards

- [`../standards-kb/06-data-management.md`](../standards-kb/06-data-management.md) — batch/streamed processing, idempotency, validation parity, retention of generated files.
- [`../standards-kb/05-reliability-resilience.md`](../standards-kb/05-reliability-resilience.md) — async jobs, checkpoint/resume, retries, partial-success semantics.
- [`../standards-kb/04-security.md`](../standards-kb/04-security.md) — malware scanning, permission-respecting exports, access-controlled expiring files, no cross-tenant leakage.
- [`../standards-kb/03-performance-scalability.md`](../standards-kb/03-performance-scalability.md) — memory-safe streaming, batching, job rate-limits.
- [`../standards-kb/09-compliance-privacy.md`](../standards-kb/09-compliance-privacy.md) — PII in exports, data-movement audit, retention.
- [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md) — wizard steps, progress, error-report screen states.
- [`../feature-blueprints/README.md`](../feature-blueprints/README.md) — shared conventions.

## Acceptance checklist

- [ ] Guided flow: upload → column mapping → validate/preview → async run → result report.
- [ ] Uploads malware-scanned; non-clean files quarantined and blocked.
- [ ] Files streamed/batched, never fully in memory; large files don't OOM.
- [ ] Import runs async with checkpointing; resumable after failure/restart.
- [ ] Import idempotent (idempotency key + dedup keys); re-submit/retry doesn't duplicate.
- [ ] Partial success: valid rows commit, invalid rows reported per-field, error report downloadable.
- [ ] No row silently dropped; counts reconcile (processed = success + error + skipped).
- [ ] Bulk writes use the same server-side validation as normal create/update.
- [ ] Exports respect caller permissions + filters (row/field level); no restricted-data leakage.
- [ ] Tenant-isolated + audited both directions; export files access-controlled with expiring links.
- [ ] Reusable mappings + downloadable templates; large exports async + notified.
- [ ] Loading / empty / error states on every step.
- [ ] All tables tenant-scoped with standard audit columns.
