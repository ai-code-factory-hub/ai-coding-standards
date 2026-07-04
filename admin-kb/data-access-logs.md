# Admin · Data-Access (Read) Logs

> The record of **who viewed or exported sensitive records** — not just who changed them. Per-record access history, bulk-access alerts, and filtering by actor, record, and data type. Read-access logging is a legal requirement under HIPAA, DPDP, GDPR and similar regimes.

## What it is / when you need it

Most audit logs capture *writes* (create/update/delete). Regulations covering sensitive data — health records, financial data, personal data — additionally require logging *reads*: every time a user **views, opens, or exports** a protected record. This surface is that read-access ledger. You need it to answer a patient/customer's "who has looked at my data?", to detect snooping (an employee browsing records they have no business reason to see), and to satisfy accounting-of-disclosures obligations. It is deliberately separate from the write-audit log because read volume is far higher and its questions ("show every access to record X", "show everything actor Y viewed") are different.

## Data model

Tenant-scoped. `DataAccessLog` is **append-only / immutable**. Retention: per regulatory requirement — often 6 years (HIPAA) or per DPDP/GDPR accountability policy — on immutable/WORM storage. High volume; partition by tenant + time.

### DataAccessLog (append-only)

| Field | Type | Notes |
| --- | --- | --- |
| `id` | ulid | Time-ordered PK |
| `tenant_id` | fk | Partition key |
| `occurred_at` | timestamp | UTC |
| `actor_id` | fk | Who accessed |
| `actor_role` | string | Role at time of access |
| `on_behalf_of_id` | fk? | Set if accessed via impersonation/support |
| `access_type` | enum | `view` \| `list` \| `search_hit` \| `export` \| `print` \| `download` \| `api_read` |
| `resource_type` | enum | e.g. `patient_record` \| `invoice` \| `pii_profile` (project-defines the sensitive set) |
| `resource_id` | string | The specific record |
| `data_classification` | enum | `phi` \| `pii` \| `financial` \| `confidential` \| `restricted` |
| `fields_accessed` | string[]? | Which sensitive fields were rendered, where tracked |
| `purpose` | string? | Declared business reason, where required |
| `record_count` | int | >1 for list/export/bulk operations |
| `ip_address` | string | Masked in UI |
| `correlation_id` | string | Ties to the originating request |
| `source` | enum | `web` \| `api` \| `report` \| `export_job` |

### BulkAccessAlert

| Field | Type | Notes |
| --- | --- | --- |
| `id` | ulid | PK |
| `tenant_id` | fk | |
| `actor_id` | fk | Who triggered |
| `window_start` | timestamp | Detection window |
| `record_count` | int | Records accessed in window |
| `threshold` | int | Rule that fired |
| `resource_type` | enum | |
| `status` | enum | `open` \| `acknowledged` \| `justified` \| `flagged` |
| `note` | text? | Required to justify/flag |

### SensitiveResourcePolicy (what must be read-logged)

| Field | Type | Notes |
| --- | --- | --- |
| `tenant_id` | fk? | Null = platform default |
| `resource_type` | enum | |
| `classification` | enum | Drives retention + alert thresholds |
| `log_reads` | bool | Whether reads of this type are logged |
| `bulk_threshold` | int | Records/window that trips a bulk-access alert |

## Key screens & UX

1. **Access log (table)** — virtualized, reverse-chronological: time, actor (+role), access type, resource type + id, classification badge, record count, source. Filters: date range, actor, resource type/id, access type, classification, impersonated-only, bulk-only. *Loading:* skeleton rows. *Empty:* "No access records match these filters." *Error:* retry.
2. **Per-record access history** — "who accessed this record?" — enter/deep-link a `resource_id` to see every actor who viewed/exported it, in order, with purpose where captured. This is the *accounting-of-disclosures* view a data subject's request maps to.
3. **Per-actor activity** — "what did this person view?" — every record an actor touched in a window; the snooping-investigation view.
4. **Bulk-access alerts** — actors who read/exported more than the threshold in a window; triage with acknowledge / justify / flag (note required). Flagged alerts route to [`security-center.md`](security-center.md).
5. **Sensitive-resource policy** — which resource types are read-logged, their classification, and bulk thresholds.

## Rules & logic

- [MUST] Admin/compliance-role only, **RBAC-gated** and **tenant-scoped** — see [`../standards-kb/04-security.md`](../standards-kb/04-security.md).
- [MUST] Log **every read** of a sensitive/classified record (view, list-with-render, search-hit, export, print, download, api-read) — reads are logged, not only writes — per [`../standards-kb/09-compliance-privacy.md`](../standards-kb/09-compliance-privacy.md).
- [MUST] Records are **append-only / immutable**; no edit or delete except via governed retention.
- [MUST] Capture **actor, on-behalf-of (impersonation), record, classification, and time**; where impersonation is involved, both the impersonator and the target are recorded.
- [MUST] **Mask PII** in the log view itself (the log must not become a second copy of the sensitive data); unmask is permission-gated and audited per [`../standards-kb/20-logging-audit-and-traceability.md`](../standards-kb/20-logging-audit-and-traceability.md).
- [MUST] **Exports of the access log are themselves logged** (viewing the read-log is also a sensitive access) and permission-gated.
- [MUST] Retain per the strictest applicable regulation for the data class (e.g. HIPAA 6-year) on immutable storage.
- [SHOULD] Fire a **bulk-access alert** when an actor exceeds the per-window threshold; route flagged alerts to the security center and on-call.
- [SHOULD] Support **per-record disclosure export** so a data-subject request ("who saw my data?") can be answered directly.
- [SHOULD] Capture a **declared purpose** for access to high-classification records where the workflow allows.

## Applicable standards

- Access control, least privilege, tenant isolation — [`../standards-kb/04-security.md`](../standards-kb/04-security.md)
- Read-access logging, append-only, unmask audit, export audit — [`../standards-kb/20-logging-audit-and-traceability.md`](../standards-kb/20-logging-audit-and-traceability.md)
- HIPAA/DPDP/GDPR read-logging, accounting of disclosures, retention — [`../standards-kb/09-compliance-privacy.md`](../standards-kb/09-compliance-privacy.md)
- Bulk-access anomaly detection, alert routing — [`../standards-kb/07-observability-aiops.md`](../standards-kb/07-observability-aiops.md)
- Table/filter UX, states — [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md)
- Related: [`security-center.md`](security-center.md) · [`../feature-blueprints/audit-log-and-activity.md`](../feature-blueprints/audit-log-and-activity.md) · [`README.md`](README.md)

## Acceptance checklist

- [ ] Every read (view/list/search/export/print/download/api) of a classified record is logged, tenant-scoped.
- [ ] Records are append-only and immutable.
- [ ] Per-record access history answers "who viewed this record?" in order.
- [ ] Per-actor view answers "what did this person access?" over a window.
- [ ] Impersonated access records both impersonator and target.
- [ ] PII is masked in the log view; unmask is permission-gated and audited.
- [ ] Bulk-access alerts fire on threshold and can be acknowledged/justified/flagged; flags reach the security center.
- [ ] Access-log exports are permission-gated and themselves logged.
- [ ] Retention matches the strictest applicable regulation; loading/empty/error states present.
