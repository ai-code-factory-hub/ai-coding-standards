# Feature Blueprint · Onboarding & KYC

A progressive tenant + user onboarding wizard with identity/KYC verification, malware-scanned document upload, verification states, activation tracking (time-to-first-value), and guided tours.

## What it is / when you need it

The gap between "signed up" and "getting value" is where most enterprise SaaS loses customers. This blueprint covers **tenant onboarding** (workspace setup, config, seed data), **user onboarding** (invite → profile → first meaningful action), and — where the domain requires it (fintech, regulated marketplaces, healthcare) — **identity / KYC verification** with document upload and verification states. It is progressive (never a 20-field wall), resumable, and instrumented so you can measure and improve **activation** and **time-to-first-value (TTFV)**.

## Data model

> All tables carry the standard audit columns (`id`, `tenant_id`, `created_at`, `created_by`, `updated_at`, `updated_by`, `deleted_at`, `version`) and are filtered by `tenant_id` on every query.

| Entity | Key fields | Notes |
| --- | --- | --- |
| `onboarding_flow` | `audience` (tenant/user/role), `steps[]` (ordered: key, title, required, component), `version`, `is_active` | Definition of a wizard; versioned so in-flight users keep their flow. |
| `onboarding_progress` | `subject_type` (tenant/user), `subject_id`, `flow_id` FK, `current_step`, `completed_steps[]`, `status` (not_started/in_progress/completed/skipped/abandoned), `started_at`, `completed_at`, `ttfv_at` | Per-subject progress; `ttfv_at` = first meaningful action timestamp. |
| `onboarding_step_result` | `progress_id` FK, `step_key`, `data` (JSON), `status`, `completed_at` | Captured input per step (idempotent upsert). |
| `kyc_case` | `subject_type` (tenant/user/beneficial_owner), `subject_id`, `level` (basic/enhanced), `status` (unstarted/pending/in_review/verified/rejected/expired), `provider`, `provider_ref`, `risk_score`, `verified_at`, `expires_at`, `reject_reason` | One verification case per subject; state machine below. |
| `kyc_document` | `kyc_case_id` FK, `doc_type` (id/passport/address_proof/business_reg), `storage_ref`, `scan_status` (pending/clean/infected/failed), `provider_status`, `expires_at` | Uploaded evidence; **not served until `scan_status=clean`**. |
| `identity_check` | `kyc_case_id` FK, `type` (document/biometric/liveness/sanctions/pep/address), `result` (pass/fail/manual), `provider_payload_ref`, `checked_at` | Individual provider checks composing a case decision. |
| `activation_event` | `subject_type`, `subject_id`, `milestone_key` (e.g. `first_report_created`), `occurred_at` | Activation funnel signals feeding TTFV/activation metrics. |
| `guided_tour_state` | `user_id`, `tour_key`, `status` (available/started/completed/dismissed), `last_step`, `updated_at` | In-product tour / checklist progress. |

## Key screens & UX

| Screen | Primary fields / actions | States |
| --- | --- | --- |
| **Welcome / setup wizard** | Progressive steps (org profile → team invites → key config → seed/import → first action); progress bar; save & continue later; skip optional | not-started / in-progress (resumable) / completed |
| **Setup checklist** (persistent) | "Get started" cards with % complete, deep-links to remaining steps, dismiss | empty (all done) |
| **KYC / verification** | Choose id type, capture/upload document, liveness/selfie step, review & submit; status banner | unstarted / pending upload / scanning / in-review / verified / rejected (with reason + re-submit) / expired |
| **Document upload** | Drag-drop, allowed types/size, scanning spinner, preview after clean | uploading / scanning / clean / infected (blocked) / failed |
| **Verification status** (admin/reviewer) | Case queue, per-check results, risk score, approve/reject with reason, request more docs | pending / in-review / decided |
| **User invite & profile** | Invite by email/role, accept flow, set name/avatar/preferences, verify email | invited / accepted / active |
| **Guided tour** | Contextual tooltips/coach-marks, "next/skip", replayable from help menu | available / running / completed / dismissed |

## Rules & logic

- **[MUST]** Onboarding is **progressive and resumable**: capture the minimum to start, defer the rest, and let users leave and return without losing progress. No single blocking wall of fields.
- **[MUST]** Every uploaded document is **malware-scanned** before it is stored-as-usable or shown; `scan_status != clean` files are quarantined and never served. See security standard.
- **[MUST]** KYC follows an explicit **state machine**: `unstarted → pending → in_review → verified | rejected | expired`. Transitions are audited; a rejected case shows the reason and supports controlled re-submission; verified cases can **expire** and require re-verification.
- **[MUST]** Access to gated capability is driven by **verification state + entitlements**, enforced **server-side** — never by hiding a button. A tenant/user that isn't `verified` cannot perform the gated action even via direct API.
- **[MUST]** KYC/identity data is **PII/sensitive**: encrypted at rest, access-controlled, retained per policy, and minimised (store the decision + provider ref, not more raw documents than required). Support deletion/retention obligations.
- **[MUST]** Verification providers sit behind a **swappable adapter** (timeouts, retries, verified idempotent callbacks); provider webhooks update case/check state.
- **[MUST]** Instrument the funnel: record step completion, drop-off, and **activation milestones**; compute **TTFV** and activation rate per cohort.
- **[SHOULD]** Pre-fill from invite/SSO/domain data to shorten steps; tailor the flow by `role`/plan.
- **[SHOULD]** Guided tours are dismissible, replayable, and never block the UI; respect a "don't show again".
- **[SHOULD]** Nudge stalled onboarders (reminder notification) and surface admin-visible onboarding health.

## Applicable standards

- [`../standards-kb/04-security.md`](../standards-kb/04-security.md) — malware scanning, quarantine, server-side gating, document access control.
- [`../standards-kb/09-compliance-privacy.md`](../standards-kb/09-compliance-privacy.md) — KYC/AML obligations, PII minimisation, retention, consent, data-subject rights.
- [`../standards-kb/06-data-management.md`](../standards-kb/06-data-management.md) — sensitive-data storage, encryption at rest, retention/expiry.
- [`../standards-kb/10-api-integration.md`](../standards-kb/10-api-integration.md) — verification-provider adapters + verified idempotent callbacks.
- [`../standards-kb/07-observability-aiops.md`](../standards-kb/07-observability-aiops.md) — activation/TTFV funnel metrics, drop-off tracking.
- [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md) — wizard/upload/tour states, progressive disclosure.
- [`../feature-blueprints/README.md`](../feature-blueprints/README.md) — shared conventions.

## Acceptance checklist

- [ ] Onboarding wizard is progressive, resumable, and versioned; progress survives leave/return.
- [ ] Persistent setup checklist with % complete and deep-links to remaining steps.
- [ ] All document uploads are malware-scanned; non-clean files are quarantined and never served.
- [ ] KYC state machine (unstarted→pending→in_review→verified|rejected|expired) enforced and audited.
- [ ] Rejected cases show reason + support re-submit; verified cases can expire and re-verify.
- [ ] Gated capability enforced server-side by verification state + entitlements (not UI-hidden).
- [ ] KYC/PII encrypted at rest, access-controlled, minimised, retained per policy.
- [ ] Verification providers behind swappable adapters with verified idempotent callbacks.
- [ ] Activation milestones + TTFV instrumented; funnel drop-off observable.
- [ ] Guided tours dismissible, replayable, non-blocking.
- [ ] Loading / empty / error states on wizard, upload, and verification screens.
- [ ] All tables tenant-scoped with standard audit columns.
