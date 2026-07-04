# Admin · Communication / Delivery Logs

> The delivery ledger for **every message the system sent** — email, SMS, push, in-app, webhook. Template, recipient (masked), channel, delivery status, bounces, opt-outs, and **resend** — tied to the notifications center that produced them.

## What it is / when you need it

When a customer says "I never got the invoice / OTP / reminder", this surface answers definitively: was it sent, to whom, on what channel, from which template, and did the provider accept, deliver, bounce, or was the recipient opted out? It is the audit and troubleshooting layer beneath [`../feature-blueprints/notifications-center.md`](../feature-blueprints/notifications-center.md) — the notifications center decides *what to send*, this logs *what actually happened to each message* via the delivery providers (ESP/SMS gateway/push service/webhook endpoint). You need it for support, deliverability monitoring, and compliance with consent/opt-out law.

## Data model

Tenant-scoped. `MessageLog` is **append-only**; delivery status advances via linked immutable `DeliveryEvent`s (provider callbacks), not by mutating the message row. Retention: message logs kept per operational + compliance window (commonly 12–24 months); message *body* stored only as a template ref + variables, not indefinitely as rendered PII.

### MessageLog (append-only)

| Field | Type | Notes |
| --- | --- | --- |
| `id` | ulid | Time-ordered PK |
| `tenant_id` | fk | |
| `created_at` | timestamp | When queued |
| `channel` | enum | `email` \| `sms` \| `push` \| `in_app` \| `webhook` |
| `template_id` | fk | Source template |
| `template_version` | string | Version rendered |
| `recipient` | string | Address/number/device/endpoint — **masked in UI** |
| `recipient_user_id` | fk? | Resolved user, if known |
| `subject` | string? | Email/push title (masked if PII) |
| `variables_ref` | string | Pointer to merge data (not inlined raw) |
| `provider` | string | ESP / SMS gateway / push service / endpoint |
| `provider_message_id` | string? | Provider's id for reconciliation |
| `status` | enum | `queued` \| `sent` \| `delivered` \| `bounced` \| `failed` \| `deferred` \| `opened` \| `clicked` \| `suppressed` |
| `bounce_type` | enum? | `hard` \| `soft` \| `block` |
| `opt_out_suppressed` | bool | Send blocked because recipient opted out |
| `job_run_id` | fk? | The send job — pivots to [`job-and-queue-monitor.md`](job-and-queue-monitor.md) |
| `correlation_id` | string | Ties to the originating event |
| `cost` | decimal? | Provider cost, where metered |

### DeliveryEvent (provider callback, append-only)

| Field | Type | Notes |
| --- | --- | --- |
| `id` | ulid | PK |
| `message_id` | fk | |
| `occurred_at` | timestamp | Provider timestamp |
| `event` | enum | `accepted` \| `delivered` \| `bounce` \| `complaint` \| `open` \| `click` \| `unsubscribe` \| `failed` |
| `provider_payload` | jsonb | Raw callback (masked) |
| `reason` | text? | Bounce/failure detail |

### OptOut / Suppression

| Field | Type | Notes |
| --- | --- | --- |
| `tenant_id` | fk | |
| `channel` | enum | |
| `recipient_hash` | string | Hashed address/number |
| `scope` | enum | `all` \| `marketing` \| `category` |
| `reason` | enum | `user_unsubscribe` \| `hard_bounce` \| `complaint` \| `admin` |
| `created_at` | timestamp | |

## Key screens & UX

1. **Message log (table)** — virtualized, reverse-chronological: time, channel (icon), template, recipient (masked), status (color-coded), provider, bounce/suppression badge. Filters: date range, channel, template, status, bounce type, opted-out-only, recipient (masked search), correlation id. *Loading:* skeleton rows. *Empty:* "No messages match these filters." *Error:* retry.
2. **Message detail** — full delivery timeline (queued → sent → delivered/bounced/opened/clicked) from `DeliveryEvent`s, rendered template + version, masked recipient and variables, provider message id, and pivots: **"view send job"** and **"view template"**. Body shown as template ref, not a permanent rendered copy of PII.
3. **Resend** — re-dispatch a message (fixed address, transient provider failure) — **blocked if the recipient is opted out/suppressed**; requires reason; creates a new `MessageLog` linked to the original.
4. **Bounces & suppressions** — hard/soft bounces and complaints, and the opt-out/suppression list; add/remove suppression (admin) with audit.
5. **Deliverability overview** — per-channel sent/delivered/bounce/complaint rates and cost; thresholds flag deteriorating deliverability.

## Rules & logic

- [MUST] Admin-role only, **RBAC-gated** and **tenant-scoped** — see [`../standards-kb/04-security.md`](../standards-kb/04-security.md).
- [MUST] Log **every outbound message** across all channels with template, channel, status, and correlation id; message logs are **append-only** and status advances only via linked delivery events.
- [MUST] **Mask recipient PII** (email/phone/device) and message variables in the UI by default; unmask is permission-gated and audited per [`../standards-kb/20-logging-audit-and-traceability.md`](../standards-kb/20-logging-audit-and-traceability.md).
- [MUST] **Honor opt-outs / suppressions**: a send to an opted-out recipient is `suppressed`, not delivered; **resend is blocked** for suppressed recipients — see [`../standards-kb/09-compliance-privacy.md`](../standards-kb/09-compliance-privacy.md).
- [MUST] **Resend and suppression changes emit their own audit entries** (actor, message/recipient, reason).
- [MUST] **Exports are permission-gated and themselves audited**.
- [MUST] Do **not** store the fully rendered PII body indefinitely — reference `template_id` + `variables_ref`; apply retention to raw payloads.
- [SHOULD] Reconcile provider callbacks by `provider_message_id`; keep the delivery timeline authoritative from `DeliveryEvent`s.
- [SHOULD] Auto-suppress on **hard bounce / complaint** and record the reason.
- [SHOULD] Pivot each message to its **send job** ([`job-and-queue-monitor.md`](job-and-queue-monitor.md)) and its **template** ([`../feature-blueprints/notifications-center.md`](../feature-blueprints/notifications-center.md)) via `correlation_id`.
- [SHOULD] Track per-channel **cost** and deliverability rates; alert on bounce/complaint spikes.

## Applicable standards

- Access control, tenant isolation, PII masking — [`../standards-kb/04-security.md`](../standards-kb/04-security.md)
- Message-log append-only, unmask audit, resend/export audit — [`../standards-kb/20-logging-audit-and-traceability.md`](../standards-kb/20-logging-audit-and-traceability.md)
- Consent, opt-out/suppression, PII retention/minimization — [`../standards-kb/09-compliance-privacy.md`](../standards-kb/09-compliance-privacy.md)
- Provider retries/deferrals, delivery reconciliation — [`../standards-kb/05-reliability-resilience.md`](../standards-kb/05-reliability-resilience.md)
- Deliverability metrics, correlation ids, spike alerting — [`../standards-kb/07-observability-aiops.md`](../standards-kb/07-observability-aiops.md)
- Table/detail UX, states — [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md)
- Related: [`../feature-blueprints/notifications-center.md`](../feature-blueprints/notifications-center.md) · [`job-and-queue-monitor.md`](job-and-queue-monitor.md) · [`../feature-blueprints/audit-log-and-activity.md`](../feature-blueprints/audit-log-and-activity.md) · [`README.md`](README.md)

## Acceptance checklist

- [ ] Every outbound message (email/SMS/push/in-app/webhook) is logged, tenant-scoped and append-only.
- [ ] Message detail shows the full delivery timeline from provider callbacks, template + version.
- [ ] Recipient PII and variables are masked by default; unmask is permission-gated and audited.
- [ ] Opt-outs/suppressions are honored; sends to suppressed recipients are `suppressed` and resend is blocked.
- [ ] Resend creates a new linked message and emits an audit entry.
- [ ] Hard bounces/complaints auto-suppress with a recorded reason.
- [ ] Messages pivot to their send job and source template via correlation id.
- [ ] Rendered PII bodies are not stored indefinitely; retention applies to raw payloads.
- [ ] Exports are permission-gated and audited; deliverability overview and loading/empty/error states present.
