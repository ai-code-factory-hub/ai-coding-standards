# Admin · Retention & Erasure

> Define retention policies per data class, run automated purge/anonymize jobs (audited), and manage an erasure queue with legal-hold exceptions — soft-delete vs hard-delete, and offboarding that exports before it deletes.

## What it is / when you need it

The data-lifecycle enforcer. Personal and operational data must not live forever: regulations and good hygiene demand you delete or anonymize it once its purpose ends — **unless** a legal hold says otherwise. This screen lets an admin declare how long each class of data is kept, schedule the jobs that actually purge/anonymize it, watch the erasure queue (fed by DSRs and offboarding), and honor legal-hold exceptions so evidence under litigation is never destroyed. It operationalizes retention duties from [`../standards-kb/06-data-management.md`](../standards-kb/06-data-management.md) and [`../standards-kb/09-compliance-privacy.md`](../standards-kb/09-compliance-privacy.md).

## Data model

Tenant-scoped; standard audit columns. Every purge/anonymize is itself an **audited, irreversible** event with counts and scope recorded.

### RetentionPolicy

| Field | Type | Notes |
| --- | --- | --- |
| `id` | uuid | PK |
| `data_class` | string | e.g. `audit_log`, `invoice`, `user_pii`, `chat_message`, `access_log` |
| `retention_period` | duration | e.g. `P7Y`, `P30D` |
| `basis` | enum | `legal_minimum` \| `business_need` \| `consent_bound` |
| `action` | enum | `hard_delete` \| `anonymize` \| `pseudonymize` \| `archive` |
| `trigger` | enum | `age` \| `inactivity` \| `event` (e.g. account closed) |
| `min_retention` | duration? | Floor a legal minimum forbids deleting before |
| `enabled` | bool | Policy active |
| `region_overrides_json` | jsonb? | Per-region retention differences |

### PurgeJob

| Field | Type | Notes |
| --- | --- | --- |
| `id` | uuid | PK |
| `policy_id` | fk | → RetentionPolicy |
| `status` | enum | `scheduled` \| `dry_run` \| `running` \| `completed` \| `failed` \| `partial` |
| `scope_json` | jsonb | What was evaluated (data class, filters) |
| `candidate_count` | int | Rows matched |
| `held_count` | int | Skipped due to legal hold |
| `actioned_count` | int | Deleted/anonymized |
| `dry_run` | bool | Report without acting |
| `report_uri` | string? | Downloadable summary |
| `started_at` / `completed_at` | timestamp | |
| `run_by` | fk/system | Scheduler or operator |

### ErasureRequest (queue item)

| Field | Type | Notes |
| --- | --- | --- |
| `id` | uuid | PK |
| `subject_id` | string | |
| `source` | enum | `dsr` \| `offboarding` \| `policy` \| `manual` |
| `dsr_id` | fk? | Link to originating DSR (see [`consent-and-dsr-center.md`](consent-and-dsr-center.md)) |
| `status` | enum | `queued` \| `on_hold` \| `in_progress` \| `completed` \| `blocked` |
| `method` | enum | `hard_delete` \| `anonymize` |
| `scope_systems` | string[] | Stores to action, incl. backups schedule |
| `verified` | bool | Prereqs met before destructive action |
| `completed_at` | timestamp? | |
| `certificate_uri` | string? | Proof-of-erasure record |

### LegalHold

| Field | Type | Notes |
| --- | --- | --- |
| `id` | uuid | PK |
| `matter` | string | Case/reason reference |
| `scope_json` | jsonb | Subjects/data classes frozen |
| `status` | enum | `active` \| `released` |
| `placed_by` / `placed_at` | mixed | |
| `released_by` / `released_at` | mixed? | |

A `LegalHold` **overrides retention** — held data is skipped by purge jobs and erasure requests (marked `on_hold`/`held_count`) and can only be deleted after the hold is released.

### SoftDelete convention

| Field | Type | Notes |
| --- | --- | --- |
| `deleted_at` | timestamp? | Non-null = soft-deleted, hidden from normal queries |
| `deleted_by` | fk? | Actor |
| `purge_after` | timestamp? | When the grace window ends and hard-delete is eligible |

## Key screens & UX

1. **Retention policies** — table per `data_class`: retention period, action (delete/anonymize/archive), trigger, legal minimum, enabled. Add/edit with validation that `retention_period` respects any `min_retention`. *Loading:* skeleton; *Empty:* "No policies — data has no defined lifecycle yet." *Error:* retry.
2. **Purge job runner** — schedule or run-now per policy, always with a **dry-run first** showing candidate count and how many are skipped by holds; commit runs the job with progress and a downloadable report. *State:* `partial` allowed only with a report explaining skips/failures.
3. **Erasure queue** — items from DSRs/offboarding/policy with status, method, scope, and any `on_hold` flag. Filter by source/status/held. An item under legal hold cannot be actioned and shows the blocking matter.
4. **Legal holds** — place/release holds scoped to subjects or data classes; placing a hold immediately freezes matching erasure/purge; releasing re-eligibilizes them. Every place/release is audited with matter reference.
5. **Soft-delete / trash** — recently soft-deleted records with a restore option until `purge_after`; after the grace window they become hard-delete-eligible.
6. **Offboarding** — for an offboarding tenant/user, an **export-then-delete** flow: generate the data export (expiring signed URL), confirm delivery, then schedule erasure across systems with a proof-of-erasure certificate.

## Rules & logic

- [MUST] Every `data_class` has a defined retention policy (period + action + trigger); nothing personal is kept **indefinitely by default** (per [`../standards-kb/06-data-management.md`](../standards-kb/06-data-management.md)).
- [MUST] Purge/anonymize jobs run **dry-run first**, and every real run is **audited** with scope, candidate/held/actioned counts, and a report per [`../standards-kb/20-logging-audit-and-traceability.md`](../standards-kb/20-logging-audit-and-traceability.md).
- [MUST] **Legal holds override retention**: held subjects/classes are skipped by all purge and erasure operations and are only deletable after the hold is released; skipped counts are surfaced, never hidden.
- [MUST] A retention policy **cannot delete before its `min_retention`/legal minimum** (e.g. financial records kept 7 years); the editor enforces the floor.
- [MUST] Distinguish **soft-delete (reversible, grace window) from hard-delete (irreversible)**; hard-delete only after the grace window or an explicit, audited override, and it also removes the data from **backups on their next cycle** (documented, not silently retained forever).
- [MUST] **Anonymization is irreversible** and must break linkability (no re-identification via retained keys) where a policy chooses anonymize over delete.
- [MUST] **Offboarding is export-then-delete**: the data export completes and is confirmed before destructive erasure runs; a proof-of-erasure certificate is produced.
- [MUST] Erasure requests originating from DSRs stay in sync with the [`consent-and-dsr-center.md`](consent-and-dsr-center.md) and honor its statutory deadlines.
- [MUST] All policy edits, job runs, hold place/release, and manual erasures are **RBAC-gated** (data-governance/DPO roles), tenant-scoped, and audited.
- [SHOULD] Support **per-region retention overrides** so residency-specific rules apply (see [`../standards-kb/01-architecture-multitenancy.md`](../standards-kb/01-architecture-multitenancy.md)).
- [SHOULD] Alert on jobs that fail, skip large volumes, or would violate a minimum — never fail silently.
- [SHOULD] Retain a minimal, non-personal **tombstone/audit** of the erasure itself (that data was erased, when, by which request) even after the content is gone.

## Applicable standards

- Retention, purge, anonymization, backups, soft vs hard delete — [`../standards-kb/06-data-management.md`](../standards-kb/06-data-management.md)
- Erasure/right-to-be-forgotten, legal basis, deadlines — [`../standards-kb/09-compliance-privacy.md`](../standards-kb/09-compliance-privacy.md)
- Purge/erasure/hold audit trail (irreversible actions) — [`../standards-kb/20-logging-audit-and-traceability.md`](../standards-kb/20-logging-audit-and-traceability.md)
- Access control for destructive actions, export security — [`../standards-kb/04-security.md`](../standards-kb/04-security.md)
- Per-region retention & residency — [`../standards-kb/01-architecture-multitenancy.md`](../standards-kb/01-architecture-multitenancy.md)
- Scheduled jobs / pipelines for purge — [`../standards-kb/12-devops-cicd.md`](../standards-kb/12-devops-cicd.md)
- Broader obligations & registers — [`../compliance-kb/README.md`](../compliance-kb/README.md)
- Related — [`consent-and-dsr-center.md`](consent-and-dsr-center.md), [`tenant-org-management.md`](tenant-org-management.md), [`../admin-kb/README.md`](../admin-kb/README.md)

## Acceptance checklist

- [ ] Every data class has a retention policy (period + action + trigger); nothing personal is kept indefinitely by default.
- [ ] Purge/anonymize jobs dry-run first and audit every real run with scope + candidate/held/actioned counts.
- [ ] Legal holds override retention; held data is skipped and only deletable after release, with counts surfaced.
- [ ] Policies cannot delete before their legal minimum; the editor enforces the floor.
- [ ] Soft-delete (reversible, grace window) is distinct from hard-delete (irreversible); backups purge on next cycle.
- [ ] Anonymization is irreversible and breaks re-identification.
- [ ] Offboarding runs export-then-delete with a proof-of-erasure certificate.
- [ ] DSR-sourced erasures stay in sync with the DSR center and its deadlines.
- [ ] Per-region retention overrides supported; job failures/large skips alert, never silent.
- [ ] All actions RBAC-gated, tenant-scoped, audited; loading/empty/error states designed.
