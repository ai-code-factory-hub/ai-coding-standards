# 20 · Logging, Audit & Traceability

The definitive taxonomy of **which logs must exist**, so every developer emits them by default rather than by memory — one standard for what to capture, how to protect it, how long to keep it, and where an admin can see it.

This domain **extends** [04 · Security](04-security.md) (the immutable audit trail) and [07 · Observability & AIOps](07-observability-aiops.md) (structured, masked, correlation-tagged logs). It does not restate those rules — it consolidates them into a per-class contract and cross-references them. The **viewing surfaces** for every log class live in the [Admin Area](../admin-kb/README.md).

## Global logging principles

- **[MUST]** Every log entry is **structured JSON** with a consistent field set — inherited from [07 · Observability](07-observability-aiops.md): UTC timestamp, level, message, `tenant_id`, `actor` (user/service id + type), `correlation_id`/`trace_id`, service, version, environment.
- **[MUST]** A **correlation/trace id is present on every entry** and propagated across services, background jobs, queues, and outbound integration calls, so a single request can be reconstructed end-to-end across all log classes.
- **[MUST]** All timestamps are **UTC** (ISO-8601); local time is a presentation concern only.
- **[MUST]** Every entry carries **`tenant_id` and `actor`** (who or what caused it) — tenant isolation is a logging invariant, not just an authorization one.
- **[MUST NOT]** Log **secrets, credentials, tokens, PII, or PHI in plaintext**. Mask, tokenize, or field-encrypt sensitive values at the point of emission (see [04 · Security](04-security.md) crypto rules). Masking is the default; unmasking is a privileged, audited action.
- **[MUST]** **Audit-class logs are append-only and tamper-evident** — no update/delete on rows, hash-chained or WORM-backed, so any alteration is detectable (see [04 · Security](04-security.md) audit trail).
- **[MUST]** **Retention is set per data class**, not per convenience — driven by compliance, security-forensics, or operational need (see the [matrix](#retention--masking-matrix) and [09 · Compliance & Privacy](09-compliance-privacy.md)).
- **[MUST]** Logs are **centralized, searchable, and time-synchronized** across all services.
- **[MUST]** **Every log class has an admin surface** (a screen where an authorized admin can view/filter it) **and an export**, and **the export is itself an audited event** (a Data-export log entry — class 15).
- **[SHOULD]** Emit logs asynchronously so logging never blocks the request path; buffer and back-pressure rather than drop, and alarm on log-pipeline loss.

## The log taxonomy

Fifteen log classes. Each is a distinct contract: developers must emit the class whenever its trigger occurs, and each class maps to exactly one admin surface.

### 1 · Authentication / login history

- **Captures:** login success/failure, logout, MFA challenges, SSO/SAML/OIDC assertions, password/credential changes, token issuance and refresh, account recovery.
- **[MUST]** Record actor, tenant, timestamp (UTC), source IP, device/user-agent, method (password/MFA/SSO), and outcome. **[MUST]** Never log the password, OTP, or token value — only a reference/hash.
- **Retention driver:** security forensics + compliance (account-takeover investigations).
- **PII/masking:** IP and device are PII — retain but access-gate; never store credentials.
- **Tamper-evident:** **Yes** (audit-class).
- **Admin surface:** [../admin-kb/login-history-and-sessions.md](../admin-kb/login-history-and-sessions.md).

### 2 · Session logs

- **Captures:** session lifecycle — creation, refresh, idle/absolute expiry, explicit revocation, concurrent-session and impersonation start/stop.
- **[MUST]** Bind each session record to its auth event via `correlation_id`; record device, IP, and revocation reason. **[MUST]** Support "terminate all sessions" and log that action.
- **Retention driver:** security forensics + active-session management.
- **PII/masking:** device/IP are PII; session tokens are secrets — store only a session id, never the token.
- **Tamper-evident:** **Yes** (audit-class).
- **Admin surface:** [../admin-kb/login-history-and-sessions.md](../admin-kb/login-history-and-sessions.md).

### 3 · Audit trail (data mutations)

- **Captures:** create/update/delete and state transitions on business records — the canonical "who changed what, when, from → to" (see [04 · Security](04-security.md) audit trail).
- **[MUST]** Record actor+tenant, entity type+id, action, **before→after** diff, timestamp (UTC), source IP, and `correlation_id`. **[MUST]** Cover record amendments, approval sign-offs, acknowledgements, consent changes, config changes, high-value approvals, and deletions.
- **Retention driver:** compliance + legal defensibility (often the longest retention).
- **PII/masking:** before/after values may contain PII/PHI — field-mask sensitive columns in the stored diff and in the viewer.
- **Tamper-evident:** **Yes** (append-only, hash-chained).
- **Admin surface:** [../feature-blueprints/audit-log-and-activity.md](../feature-blueprints/audit-log-and-activity.md).

### 4 · Authorization / access-control events

- **Captures:** permission grants/denials, role/RBAC changes, ABAC policy decisions, escalation attempts, and **denied** access (403/BOLA/IDOR attempts).
- **[MUST]** Log every **denial** with actor, tenant, object id, permission required, and policy that denied it — denials are a primary security signal. **[MUST]** Log role and permission changes as audit-class events.
- **Retention driver:** security forensics + compliance (privilege review).
- **PII/masking:** actor is PII; mask any object payload.
- **Tamper-evident:** **Yes** for grant/role changes; access-decision telemetry may be operational-class.
- **Admin surface:** [../admin-kb/security-center.md](../admin-kb/security-center.md) (with role changes surfaced in [../feature-blueprints/audit-log-and-activity.md](../feature-blueprints/audit-log-and-activity.md)).

### 5 · Data-access / read logs (who viewed or exported sensitive records)

- **Captures:** **reads** of sensitive records — who **viewed, searched, printed, or exported** PII/PHI/financial data. This is distinct from the mutation audit trail: it records access with no change.
- **[MUST]** For regulated data classes, log every read of an individual sensitive record (actor, tenant, record id, purpose/context, timestamp). **[MUST]** Log bulk views and exports explicitly and flag anomalous volumes.
- **Retention driver:** compliance (HIPAA/GDPR "accounting of disclosures") — mandatory where regulated.
- **PII/masking:** the record id is enough; do not copy the record contents into the access log.
- **Tamper-evident:** **Yes** (audit-class — this is a disclosure record).
- **Admin surface:** [../admin-kb/data-access-logs.md](../admin-kb/data-access-logs.md).

### 6 · Activity / usage logs

- **Captures:** general user activity and feature usage — navigation, actions taken, feature adoption — feeding the activity feed and product analytics.
- **[MUST]** Tag with tenant + actor + `correlation_id`. **[SHOULD]** Respect analytics consent (see class 14) before enriching for product analytics.
- **Retention driver:** operational + product analytics (shorter than audit).
- **PII/masking:** aggregate where possible; pseudonymize actor for analytics pipelines.
- **Tamper-evident:** **No** (operational-class).
- **Admin surface:** [../feature-blueprints/audit-log-and-activity.md](../feature-blueprints/audit-log-and-activity.md).

### 7 · Application / error logs

- **Captures:** application diagnostics — errors, exceptions, stack traces, warnings, and debug traces across services and clients.
- **[MUST]** Structured JSON with `correlation_id`, service, version, environment (per [07 · Observability](07-observability-aiops.md)). **[MUST NOT]** Leak secrets/PII in messages or stack traces — scrub before shipping; user-facing errors stay generic (see [04 · Security](04-security.md)).
- **Retention driver:** operational debugging (short, high-volume).
- **PII/masking:** scrub request bodies and headers; redact tokens and identifiers.
- **Tamper-evident:** **No** (operational-class).
- **Admin surface:** [../admin-kb/log-explorer.md](../admin-kb/log-explorer.md).

### 8 · Security events (failed logins, lockouts, anomalies, WAF)

- **Captures:** security-relevant signals — repeated failed logins, lockouts, brute-force/credential-stuffing, rate-limit trips, WAF blocks, injection/SSRF attempts, and anomaly detections.
- **[MUST]** Feed a centralized security view and alerting; correlate with classes 1, 4, and 5. **[MUST]** Alert on thresholds (symptom-based, deduplicated — per [07 · Observability](07-observability-aiops.md)).
- **Retention driver:** security forensics + compliance (incident response).
- **PII/masking:** IP/device retained for forensics; mask any captured payloads.
- **Tamper-evident:** **Yes** (audit-class).
- **Admin surface:** [../admin-kb/security-center.md](../admin-kb/security-center.md).

### 9 · Admin action logs

- **Captures:** actions taken in the admin/back-office — settings changes, tenant provisioning, entitlement edits, impersonation, feature-flag toggles, forced logouts, data exports.
- **[MUST]** **Every admin surface emits its own audit entries** (admin actions are inherently sensitive — see [../admin-kb/README.md](../admin-kb/README.md)). **[MUST]** Log impersonation start/stop with the impersonated subject and reason.
- **Retention driver:** compliance + insider-threat forensics (long).
- **PII/masking:** mask any PII shown in the admin action payload.
- **Tamper-evident:** **Yes** (audit-class).
- **Admin surface:** [../feature-blueprints/audit-log-and-activity.md](../feature-blueprints/audit-log-and-activity.md) (impersonation via [../feature-blueprints/support-and-impersonation.md](../feature-blueprints/support-and-impersonation.md)).

### 10 · Background job / queue logs

- **Captures:** async work — job enqueue/start/finish, retries, dead-letter, schedule (cron) runs, queue depth and latency, worker failures.
- **[MUST]** Propagate the **originating request's `correlation_id`** into the job so async work traces back to its trigger. **[MUST]** Record outcome, attempt count, and duration; surface dead-letter items for replay.
- **Retention driver:** operational + reliability (see [05 · Reliability](05-reliability-resilience.md)).
- **PII/masking:** mask job payloads that carry PII; log ids, not full documents.
- **Tamper-evident:** **No** (operational-class).
- **Admin surface:** [../admin-kb/job-and-queue-monitor.md](../admin-kb/job-and-queue-monitor.md).

### 11 · Integration / API / webhook logs

- **Captures:** inbound API calls, outbound third-party calls, and webhook send/receive — request/response metadata, status, latency, retries, signature verification, and idempotency keys.
- **[MUST]** Log correlation id, endpoint, direction, status, and latency for every integration hop; record webhook delivery attempts and signature results. **[MUST]** Redact secrets and bearer tokens in captured headers/bodies.
- **Retention driver:** operational + partner dispute resolution.
- **PII/masking:** truncate/redact bodies; store hashes of large payloads, not the payloads.
- **Tamper-evident:** **No** (operational-class); webhook delivery records **[SHOULD]** be retained for replay/dispute.
- **Admin surface:** [../feature-blueprints/integrations-and-webhooks.md](../feature-blueprints/integrations-and-webhooks.md).

### 12 · Communication / delivery logs (email / SMS / push / in-app)

- **Captures:** every outbound message — email, SMS, push, in-app — with template, channel, recipient, provider message id, and delivery state (queued/sent/delivered/bounced/failed/opened).
- **[MUST]** Log the delivery lifecycle per message with `correlation_id`; capture provider callbacks (bounces/complaints). **[MUST]** Mask recipient contact details (email/phone) in the viewer.
- **Retention driver:** operational + compliance proof-of-notice (e.g. proof a required notice was sent).
- **PII/masking:** recipient address is PII — store masked/tokenized; never log full message body if it contains PII/PHI.
- **Tamper-evident:** **Yes** where the log is legal proof-of-delivery; otherwise operational-class.
- **Admin surface:** [../admin-kb/communication-logs.md](../admin-kb/communication-logs.md).

### 13 · Payment / billing event logs

- **Captures:** billing lifecycle — charges, refunds, invoices, subscription changes, plan/entitlement changes, dunning, gateway responses, and reconciliation.
- **[MUST]** Log each financial event with amount, currency, gateway reference, and outcome; keep it reconcilable and audit-class. **[MUST NOT]** Store PAN/CVV or full card data — reference the gateway token only (PCI-DSS).
- **Retention driver:** compliance + financial/tax record retention (typically multi-year).
- **PII/masking:** card data tokenized; store last-four only where needed.
- **Tamper-evident:** **Yes** (financial audit-class).
- **Admin surface:** [../feature-blueprints/subscription-billing.md](../feature-blueprints/subscription-billing.md).

### 14 · Consent & privacy logs (consent, DSR, erasure)

- **Captures:** consent grant/withdrawal, privacy-preference changes, and data-subject-request (DSR) lifecycle — access, rectification, portability, and erasure requests and their fulfillment.
- **[MUST]** Log consent state changes and every DSR step with actor, subject, timestamp, and outcome — this log is the **evidence of compliance** (see [09 · Compliance & Privacy](09-compliance-privacy.md)). **[MUST]** Record erasure as a tamper-evident event even after the underlying PII is deleted (log the *fact* of erasure, not the erased data).
- **Retention driver:** compliance (regulatory — often outlives the data it concerns).
- **PII/masking:** the log records the request and its resolution, not the personal data itself.
- **Tamper-evident:** **Yes** (audit-class — regulatory evidence).
- **Admin surface:** [../admin-kb/consent-and-dsr-center.md](../admin-kb/consent-and-dsr-center.md).

### 15 · Data export / import logs

- **Captures:** every bulk export and import — who, what dataset, row counts, format, filters, destination, and outcome. **Exports of any other log class land here too** (per the global principle that every export is audited).
- **[MUST]** Log the initiator, tenant, dataset/scope, record count, and destination for every export/import; flag exports of sensitive data classes for review. **[MUST]** Treat an export of PII/PHI as a disclosure (cross-reference class 5).
- **Retention driver:** compliance + data-governance (bulk data movement is high-risk).
- **PII/masking:** log the manifest (what/where/how many), not the exported rows.
- **Tamper-evident:** **Yes** (audit-class).
- **Admin surface:** [../feature-blueprints/import-export-pipeline.md](../feature-blueprints/import-export-pipeline.md).

> **Example (healthcare):** A clinician opens a patient's diagnostic report. A **data-access log** (class 5) records the read as a disclosure — even though nothing changed — satisfying HIPAA "accounting of disclosures." When the report PDF is emailed, a **communication log** (class 12) captures delivery with the recipient masked, and if the patient later requests erasure, a **consent/DSR log** (class 14) records the erasure as a tamper-evident event without retaining the erased PHI. All three share one `correlation_id`, so the full episode is reconstructable.

> **Example (fintech):** A merchant admin issues a bulk refund. The **audit trail** (class 3) records each order's before→after state, **payment logs** (class 13) capture gateway references and amounts with only the card last-four stored, **authorization logs** (class 4) prove the admin held the "approve refund ≤ $X" permission, and the CSV the admin then downloads is written to the **data-export log** (class 15) — which is itself audited. No PAN, token, or CVV appears in any log.

## Retention & masking matrix

Retentions are **suggested defaults**; the governing compliance regime overrides them (see [09 · Compliance & Privacy](09-compliance-privacy.md)). "Tamper-evident?" = append-only + hash-chained/WORM (audit-class).

| # | Log class | Suggested retention | PII masking | Tamper-evident? |
|---|---|---|---|---|
| 1 | Authentication / login history | 1–2 years | IP/device access-gated; no credentials | Yes |
| 2 | Session logs | 90 days – 1 year | IP/device masked; no tokens | Yes |
| 3 | Audit trail (data mutations) | 3–7 years (per regime) | before/after field-masked | Yes |
| 4 | Authorization / access-control | 1–2 years | actor PII; object payload masked | Yes (grants/roles) |
| 5 | Data-access / read logs | 3–6 years (regulated) | record id only, no contents | Yes |
| 6 | Activity / usage logs | 90 days – 1 year | pseudonymized for analytics | No |
| 7 | Application / error logs | 30–90 days | request/headers scrubbed | No |
| 8 | Security events | 1–2 years | payloads masked; IP retained | Yes |
| 9 | Admin action logs | 3–7 years | admin payload PII masked | Yes |
| 10 | Background job / queue logs | 30–90 days | job payloads masked | No |
| 11 | Integration / API / webhook logs | 90 days – 1 year | bodies redacted; secrets stripped | No (webhooks: retain for replay) |
| 12 | Communication / delivery logs | 1–2 years | recipient contact masked | Yes (proof-of-notice) |
| 13 | Payment / billing event logs | 5–10 years (financial) | card tokenized; last-four only | Yes |
| 14 | Consent & privacy logs | Life of subject + regulatory tail | request/outcome only, not the data | Yes |
| 15 | Data export / import logs | 3–7 years | manifest only, not the rows | Yes |

## Acceptance checklist

- [ ] Every log entry is structured JSON with UTC time, `tenant_id`, `actor`, and a propagated `correlation_id`/`trace_id`.
- [ ] No secrets, PII, or PHI in plaintext anywhere — masked/tokenized at emission; unmasking is privileged and audited.
- [ ] Audit-class logs (classes 1–5, 8, 9, 12–15) are append-only and tamper-evident.
- [ ] All 15 log classes are emitted at their triggers, each with retention set per its data class (per the matrix).
- [ ] Logs are centralized, searchable, and time-synchronized across all services and background jobs.
- [ ] Every log class has a named admin surface, and every export writes a class-15 record (exports are self-auditing).
- [ ] Data reads of regulated records (class 5) and bulk exports (class 15) are recorded as disclosures.
- [ ] Consent/DSR/erasure (class 14) is logged as compliance evidence without retaining erased personal data.
- [ ] Cross-references to [04 · Security](04-security.md) and [07 · Observability](07-observability-aiops.md) hold — no duplicated or conflicting rules.
