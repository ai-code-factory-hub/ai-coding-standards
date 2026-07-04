# Feature Blueprint · Notifications Center

A unified notification service that turns domain events into multi-channel messages (email / SMS / push / in-app / webhook) with templates, per-channel consent, async delivery, retries, DLQ, idempotent sends, and delivery tracking.

## What it is / when you need it

Almost every enterprise app must tell users things: "your report is ready", "payment failed", "someone shared a dashboard with you". Doing this ad-hoc (a `sendEmail()` call sprinkled through the codebase) produces duplicate sends, no opt-out, no audit, and no observability. This blueprint centralises **one** notification service: producers publish an **event**, the service resolves recipients, applies **consent**, renders **templates**, fans out to per-channel **engines**, and tracks delivery — all tenant-scoped, async, and idempotent.

## Data model

> All tables carry the standard audit columns (`id`, `tenant_id`, `created_at`, `created_by`, `updated_at`, `updated_by`, `deleted_at`, `version`) and are filtered by `tenant_id` on every query.

| Entity | Key fields | Notes |
| --- | --- | --- |
| `notification_event` | `event_key` (e.g. `report.ready`), `category` (transactional/marketing/security/system), `default_channels[]`, `payload_schema` (JSON schema), `is_active` | Catalog of event types the app can emit. |
| `notification_template` | `event_key` FK, `channel` (email/sms/push/in_app/webhook), `locale`, `subject`, `body` (templated), `variables[]`, `status` (draft/active/archived), `version_no` | One template per event × channel × locale. Versioned; render is sandboxed. |
| `notification_message` | `event_key`, `recipient_user_id`, `channel`, `category`, `idempotency_key` (unique), `dedup_hash`, `rendered_subject`, `rendered_body`, `status` (queued/sending/sent/delivered/bounced/failed/suppressed), `provider_message_id`, `attempts`, `next_attempt_at`, `correlation_id` | One row per channel send. `idempotency_key` unique per tenant. |
| `notification_delivery_event` | `message_id` FK, `type` (queued/sent/delivered/opened/clicked/bounced/complained/failed), `provider_status`, `occurred_at`, `raw_provider_payload` | Append-only delivery timeline (fed by provider webhooks). |
| `notification_preference` | `user_id`, `channel`, `category`, `opted_in` (bool), `source` (default/user/admin), `changed_at` | Consent per user × channel × category. Absence = fall back to category default. |
| `notification_inbox_item` | `user_id`, `message_id` FK, `title`, `snippet`, `deep_link`, `read_at`, `archived_at`, `priority` | In-app inbox projection. |
| `notification_channel_config` | `channel`, `provider` (e.g. SES/Twilio/FCM), `credentials_ref` (secrets-manager pointer), `from_identity`, `rate_limit`, `is_enabled` | Per-tenant/per-channel provider binding behind a swappable adapter. |
| `notification_dlq` | `original_message_id`, `event_key`, `channel`, `failure_reason`, `payload`, `retry_count`, `parked_at` | Dead-letter for exhausted retries; replayable. |

## Key screens & UX

| Screen | Primary fields / actions | States |
| --- | --- | --- |
| **In-app Inbox** | List (recent-first) with title/snippet/time, unread badge, mark read/unread, archive, "mark all read", deep-link to source | loading / empty ("You're all caught up") / error |
| **Notification Preferences** | Matrix of category × channel toggles; global "pause non-essential"; quiet hours; verify contact (email/phone/device) | loading / empty / saving / error; transactional & security categories shown as non-optional |
| **Template Manager** (admin) | Per event × channel × locale editor with variable palette, live preview with sample payload, send-test-to-me, publish/rollback version, diff vs active | draft / active / archived / invalid-variables |
| **Delivery Log** (admin) | Search by recipient/event/status/date; per-message timeline (queued→sent→delivered→opened); resend; view provider error | loading / empty / error |
| **Channel Settings** (admin) | Provider selection, from-identity, credential binding, enable/disable channel, test send | connected / disconnected / test-failed |
| **DLQ Console** (admin) | Parked messages, failure reason, bulk replay / discard | empty / has-items |

## Rules & logic

- **[MUST]** Producers **emit events, never call channels directly.** The service resolves channels via event config + user preference; no business code constructs an email.
- **[MUST]** Every send carries an **`idempotency_key`**; re-emitting the same logical event never double-sends. A `dedup_hash` (event + recipient + channel + payload digest) suppresses accidental duplicates within a window.
- **[MUST]** Delivery is **async** off a queue: enqueue → worker renders → channel adapter sends. API responses never block on provider latency.
- **[MUST]** Failed sends **retry with exponential backoff + jitter** up to a cap, then land in the **DLQ** (never silently dropped). DLQ items are replayable.
- **[MUST]** **Consent is enforced per channel + category.** `transactional`/`security`/`system` may be non-opt-out; `marketing` requires explicit opt-in. Suppression (unsubscribe, hard bounce, complaint) is honoured on every send and cannot be overridden by a producer flag.
- **[MUST]** Templates render in a **sandboxed** engine with escaping; recipient-supplied data is never executed. Missing required variables → template is invalid and cannot be published.
- **[MUST]** External providers sit behind **swappable adapters** with timeouts, retries, and circuit breakers; credentials live in a secrets manager, never in the DB row.
- **[MUST]** Inbound **provider delivery webhooks are verified (signature) and idempotent**; they update the delivery timeline and drive bounce/complaint suppression.
- **[SHOULD]** Support **digest / batching** (e.g. one daily summary instead of N emails) and **quiet hours** per user timezone.
- **[SHOULD]** Localise subject/body by user `locale`; fall back to tenant default locale, then a base locale.
- **[SHOULD]** Priority lanes: security/transactional bypass batching and quiet hours.

> **Example (regional messaging regs):** honour SMS sender-ID / DLT template registration and consent regimes where required (e.g. India TRAI DLT, US TCPA/CAN-SPAM, EU ePrivacy/GDPR). Marketing SMS/email must include opt-out, respect do-not-disturb windows, and use pre-registered templates where the regulator mandates it — model this via the `category` + registered-template-id on `notification_template`.

## Applicable standards

- [`../standards-kb/05-reliability-resilience.md`](../standards-kb/05-reliability-resilience.md) — async queue, retries, backoff, DLQ, circuit breakers.
- [`../standards-kb/10-api-integration.md`](../standards-kb/10-api-integration.md) — provider adapters, HMAC-verified inbound webhooks, idempotency.
- [`../standards-kb/09-compliance-privacy.md`](../standards-kb/09-compliance-privacy.md) — consent, opt-out, marketing regs, PII in message bodies.
- [`../standards-kb/07-observability-aiops.md`](../standards-kb/07-observability-aiops.md) — delivery metrics, correlation ids, alerting on send failure rate.
- [`../standards-kb/04-security.md`](../standards-kb/04-security.md) — secrets handling, template sandboxing, no PII leakage.
- [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md) — inbox/preferences screen states.
- [`../feature-blueprints/README.md`](../feature-blueprints/README.md) — shared conventions.

## Acceptance checklist

- [ ] Producers emit events only; the service owns channel resolution and rendering.
- [ ] Every send is idempotent (unique key + dedup window); re-emit never double-sends.
- [ ] Delivery is async; API never blocks on provider latency.
- [ ] Retries with backoff + jitter; exhausted sends land in a replayable DLQ (none dropped).
- [ ] Consent enforced per channel + category; suppression list honoured; marketing requires opt-in.
- [ ] Templates versioned, sandboxed, localised, previewable, with test-send; invalid templates cannot publish.
- [ ] Providers behind swappable adapters with timeouts/retries/breakers; credentials in secrets manager.
- [ ] Inbound provider webhooks verified + idempotent; drive delivery timeline and suppression.
- [ ] In-app inbox: recent-first, unread badge, mark-read/archive, deep-links; loading/empty/error states.
- [ ] Delivery log with per-message timeline and resend; DLQ console with replay.
- [ ] Regional messaging regs (DLT/TCPA/CAN-SPAM/ePrivacy) modelled where applicable.
- [ ] All tables tenant-scoped with standard audit columns.
