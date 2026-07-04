# Admin · Consent & DSR Center

> Record granular, versioned, withdrawable consent; intake and track data-subject requests (access / rectify / erase / portability) against statutory deadlines; and maintain a DPA / sub-processor register.

## What it is / when you need it

The privacy operations hub. Two jobs: (1) prove **what a data subject consented to, when, and under which policy version** — and let them withdraw it; and (2) run **data-subject requests (DSRs / DSARs)** end-to-end with the legal clock ticking. You need it the moment you process personal data under GDPR/CCPA/DPDP-style regimes, which is nearly every SaaS. It is the operational face of [`../standards-kb/09-compliance-privacy.md`](../standards-kb/09-compliance-privacy.md) and the [`../compliance-kb/README.md`](../compliance-kb/README.md) obligations.

## Data model

Tenant-scoped (the tenant is usually the data controller; the platform is a processor); standard audit columns. Consent and DSR records are themselves regulated evidence — **append-only, never silently edited**.

### ConsentDefinition

| Field | Type | Notes |
| --- | --- | --- |
| `purpose_key` | string | e.g. `marketing_email`, `analytics`, `data_sharing` |
| `name` / `description` | text | User-facing purpose text (localizable) |
| `legal_basis` | enum | `consent` \| `contract` \| `legitimate_interest` \| `legal_obligation` |
| `policy_version` | string | Version of the notice this purpose belongs to |
| `required` | bool | Service-essential vs optional |
| `default_state` | enum | `opt_in` \| `opt_out` — no pre-ticked opt-in |

### ConsentRecord

| Field | Type | Notes |
| --- | --- | --- |
| `id` | uuid | PK |
| `subject_id` | string | User/data-subject reference |
| `purpose_key` | fk | → ConsentDefinition |
| `state` | enum | `granted` \| `withdrawn` \| `expired` |
| `policy_version` | string | Version consented to (immutable) |
| `source` | enum | `signup` \| `preference_center` \| `import` \| `api` |
| `captured_at` | timestamp | When granted/changed |
| `withdrawn_at` | timestamp? | |
| `proof_json` | jsonb | IP, UA, exact copy shown, checkbox state — evidence bundle |
| `expires_at` | timestamp? | For time-boxed consent |

Consent is **versioned and append-only**: a change writes a new record/row rather than mutating history, so the full timeline is reconstructable.

### DataSubjectRequest (DSR)

| Field | Type | Notes |
| --- | --- | --- |
| `id` | uuid | PK; user-visible reference |
| `subject_id` | string | |
| `type` | enum | `access` \| `rectification` \| `erasure` \| `portability` \| `restriction` \| `objection` |
| `channel` | enum | `self_service` \| `email` \| `support` \| `regulator` |
| `status` | enum | `received` \| `identity_pending` \| `verified` \| `in_progress` \| `awaiting_info` \| `fulfilled` \| `rejected` \| `partially_fulfilled` |
| `received_at` | timestamp | Starts the statutory clock |
| `due_at` | timestamp | Computed from regime deadline (e.g. 30/45 days) |
| `identity_verified_at` | timestamp? | Gate before any data is disclosed/changed |
| `assignee` | fk? | Owner |
| `resolution_note` | text? | Especially required for `rejected` (with lawful ground) |
| `export_uri` | string? | Expiring signed URL for access/portability output |

### DsrTask (fulfillment subtasks)

| Field | Type | Notes |
| --- | --- | --- |
| `dsr_id` | fk | |
| `system` | string | Source system/data store to action |
| `action` | enum | `collect` \| `redact` \| `correct` \| `erase` \| `confirm` |
| `status` | enum | `pending` \| `done` \| `blocked` |
| `blocked_reason` | string? | e.g. legal hold (see [`retention-and-erasure.md`](retention-and-erasure.md)) |

### ProcessorRegister (DPA / sub-processors)

| Field | Type | Notes |
| --- | --- | --- |
| `name` | string | Vendor/sub-processor |
| `role` | enum | `sub_processor` \| `processor` \| `controller` |
| `purpose` | text | What data, why |
| `data_categories` | string[] | PII categories shared |
| `region` | string | Processing location (residency) |
| `dpa_ref` | string | Executed agreement pointer |
| `status` | enum | `active` \| `pending_review` \| `terminated` |
| `valid_from` / `review_due` | timestamp | |

## Key screens & UX

1. **Consent overview** — per subject or per purpose: current state, policy version, capture source, and full **timeline** of grants/withdrawals. *Loading:* skeleton; *Empty:* "No consent records for this subject." *Error:* retry.
2. **Preference center (subject-facing)** — granular toggles per purpose with clear descriptions, showing current state and letting the subject **withdraw as easily as they granted**; writes a new versioned record.
3. **DSR intake** — create/receive a request: subject identity, type, channel; auto-computes `due_at` from the applicable regime. *State:* `received` with a countdown to the deadline.
4. **DSR queue** — table of open requests with type, status, **days-to-deadline (color-coded, overdue flagged)**, assignee. Filter by type/status/overdue. SLA breaches surfaced prominently.
5. **DSR workspace** — identity verification gate, per-system fulfillment tasks (collect/redact/correct/erase/confirm), generated export bundle (expiring signed URL), and a decision log; `reject` requires a lawful-ground note.
6. **Processor register** — searchable list of sub-processors with purpose, data categories, region, DPA reference, and review-due dates; add/edit/terminate is audited.

## Rules & logic

- [MUST] Consent is **granular per purpose, versioned to a policy version, and withdrawable as easily as it was given**; withdrawal writes a new **append-only** record — history is never overwritten (per [`../standards-kb/09-compliance-privacy.md`](../standards-kb/09-compliance-privacy.md)).
- [MUST] No **pre-ticked opt-ins**; optional purposes default per `default_state`, and service-essential purposes rely on a non-consent `legal_basis`, clearly labeled.
- [MUST] Each consent record stores a **proof bundle** (timestamp, source, policy version, exact copy shown) sufficient to demonstrate valid consent to a regulator.
- [MUST] DSRs **verify the requester's identity before any personal data is disclosed, changed, or erased**; the verification and every fulfillment step are audited per [`../standards-kb/20-logging-audit-and-traceability.md`](../standards-kb/20-logging-audit-and-traceability.md).
- [MUST] `due_at` is computed from the applicable **statutory deadline**; the queue tracks days-remaining, flags overdue, and never lets a request silently miss its clock.
- [MUST] `access`/`portability` outputs are delivered via **expiring signed URLs** in a portable, machine-readable format; the export is itself an audited event with PII handled per policy.
- [MUST] `erasure` requests route through the **erasure queue and honor legal-hold exceptions** (see [`retention-and-erasure.md`](retention-and-erasure.md)); a blocked task records the lawful reason rather than silently completing.
- [MUST] Rejections record a **lawful ground** and are communicated to the subject; partial fulfillment is explicit, not silent.
- [MUST] All actions are **RBAC-gated** (DPO/privacy roles), **tenant-scoped**, and audited.
- [SHOULD] Consent state is **enforced downstream** — e.g. a withdrawn `marketing_email` consent suppresses those sends (see [`../feature-blueprints/notifications-center.md`](../feature-blueprints/notifications-center.md)).
- [SHOULD] The processor register drives a **sub-processor change notification** obligation and periodic DPA review reminders.
- [SHOULD] Support consent expiry / re-consent prompts on policy-version bumps.

## Applicable standards

- Lawful basis, consent, DSR/DSAR obligations, deadlines, portability — [`../standards-kb/09-compliance-privacy.md`](../standards-kb/09-compliance-privacy.md)
- Consent + DSR action audit trail (regulated evidence) — [`../standards-kb/20-logging-audit-and-traceability.md`](../standards-kb/20-logging-audit-and-traceability.md)
- Identity verification, access control, export security — [`../standards-kb/04-security.md`](../standards-kb/04-security.md)
- PII handling, export, residency of processors — [`../standards-kb/06-data-management.md`](../standards-kb/06-data-management.md), [`../standards-kb/01-architecture-multitenancy.md`](../standards-kb/01-architecture-multitenancy.md)
- Localized consent/notice copy — [`../standards-kb/13-i18n-accessibility.md`](../standards-kb/13-i18n-accessibility.md)
- Broader obligations & registers — [`../compliance-kb/README.md`](../compliance-kb/README.md)
- Related — [`retention-and-erasure.md`](retention-and-erasure.md), [`../feature-blueprints/notifications-center.md`](../feature-blueprints/notifications-center.md), [`../admin-kb/README.md`](../admin-kb/README.md)

## Acceptance checklist

- [ ] Consent is granular per purpose, versioned to a policy version, and withdrawable as easily as granted.
- [ ] Consent history is append-only with a proof bundle sufficient for a regulator; no pre-ticked opt-ins.
- [ ] DSR intake computes `due_at` from the applicable statutory deadline and tracks days-remaining/overdue.
- [ ] Identity is verified before any personal data is disclosed, changed, or erased.
- [ ] Access/portability output is a portable format delivered via an expiring signed URL and audited.
- [ ] Erasure requests route through the erasure queue and honor legal-hold exceptions.
- [ ] Rejections record a lawful ground; partial fulfillment is explicit.
- [ ] Withdrawn consent is enforced downstream (e.g. marketing suppression).
- [ ] Processor/DPA register tracks purpose, data categories, region, DPA ref, and review dates.
- [ ] All actions RBAC-gated, tenant-scoped, audited; loading/empty/error states designed.
