# 06 · Data Management

How every datastore models, protects, retains, and locates data so it stays correct, recoverable, and tenant-isolated.

## Schema hygiene

- **[MUST]** Every table has: a stable primary key (**UUID/ULID** for tenant safety — never an auto-increment integer that leaks volume or enables cross-tenant guessing), a non-null indexed `tenant_id`, `created_at`/`updated_at` (UTC), and `created_by`/`updated_by`.
- **[MUST]** Enforce integrity **in the database**, not only in app code: foreign keys, `NOT NULL`, unique constraints, and check constraints.
- **[MUST]** Represent **money as integer minor units or `DECIMAL`, never floating point**, and always store the currency code alongside it.
- **[MUST]** Store all timestamps in **UTC**; convert to local time only at display.
- **[SHOULD]** Use **append-only ledgers** for financial and other legally-significant record history — immutable entries plus corrections, never in-place edits.

> **Example (fintech/e-commerce):** the transactions ledger is append-only; a refund is a new compensating entry, so balances are always reconstructable from history.
> **Example (healthcare):** authoritative domain records (e.g. finalized results) are never overwritten — a superseded record is retained and a corrected version supersedes it.

## Backups & recovery

- **[MUST]** Automated, encrypted, **offsite** backups of all datastores, restore-tested to meet the target RTO (a backup you have never restored is not a backup). Support **per-tenant restore/export**.
- **[SHOULD]** Point-in-time recovery (PITR) on the primary datastore; immutable / object-locked backups to resist ransomware and accidental deletion.

See [Reliability & Resilience](05-reliability-resilience.md) for RTO/RPO targets and DR runbooks.

## Retention & deletion

- **[MUST]** A **retention policy per data type**, aligned to legal and compliance obligations; automated purge/anonymize jobs, audited. On tenant offboarding: export, then delete per policy and any contractual window.
- **[MUST]** **Soft delete** (`deleted_at` / status column) for user-facing deletions, excluded from default queries. Hard delete only via retention/purge jobs or an explicit data-subject erasure request.

> **Example (healthcare):** regulated domain records are **never** hard-deleted on user request — an erroneous record is marked superseded/entered-in-error and retained for the mandated audit period.

See [Compliance & Privacy](09-compliance-privacy.md) for data-subject erasure and legal-hold exceptions.

## Residency

- **[MUST]** **Data residency:** each tenant's data is stored and processed within its required jurisdiction; the tenant registry records the region; routing, storage, backups, and third-party processors all respect it.

## Consistency & search

- **[MUST]** Write-time validation plus transactional consistency for financial and other high-integrity data (optimistic locking / version columns); run periodic reconciliation checks.
- **[MUST]** Search uses a proper index (e.g. Postgres FTS/GIN or a dedicated search engine), **tenant-scoped**, asynchronously updated, and reconcilable against the source of truth. **Never** `LIKE '%x%'` full-table scans.

## Files & blobs

- **[MUST]** Store files (images, PDFs, documents, signatures, certificates) in **object storage**, not the database: tenant-partitioned paths, signed expiring URLs, encrypted at rest, and virus-scanned on upload.

## Acceptance checklist

- Every table has PK + `tenant_id` + audit columns; integrity enforced in the DB.
- Money stored as decimal/minor units with currency; timestamps in UTC.
- Restore-tested offsite backups with per-tenant restore; PITR where applicable.
- Retention policy + automated purge/anonymize; soft delete for user deletes.
- Data residency honored per tenant across storage, backups, and processors.
- Tenant-scoped indexed search; write-time validation + reconciliation.
- Files in object storage with signed URLs, encryption, and malware scanning.
