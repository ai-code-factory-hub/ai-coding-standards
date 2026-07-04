# Feature Blueprint · Integrations & Webhooks

An integrations catalog/console plus outbound webhooks (HMAC-signed, retried, DLQ, replayable, SSRF-safe) and verified inbound webhooks — every provider behind a swappable adapter.

## What it is / when you need it

Enterprise apps live in an ecosystem: accounting, CRM, storage, messaging, payment, identity, and countless partner systems. This blueprint gives tenants a **catalog** of available integrations, a **console** to connect / configure / **test the connection** and manage OAuth app credentials, an **outbound webhook** system so external systems react to events in your app, and **inbound webhook** verification so you can safely receive events. Every external provider is wrapped in a **swappable adapter** so one can be replaced without touching business logic.

## Data model

> All tables carry the standard audit columns (`id`, `tenant_id`, `created_at`, `created_by`, `updated_at`, `updated_by`, `deleted_at`, `version`) and are filtered by `tenant_id` on every query.

| Entity | Key fields | Notes |
| --- | --- | --- |
| `integration_provider` | `key` (e.g. `stripe`, `slack`), `name`, `category`, `auth_type` (oauth2/api_key/basic), `scopes[]`, `capabilities[]`, `config_schema`, `is_enabled` | Platform-level catalog entry (definition of an available integration). |
| `tenant_integration` | `provider_key` FK, `status` (disconnected/connected/error/expired), `config` (JSON, per `config_schema`), `connected_by`, `last_test_at`, `last_test_result`, `health` | A tenant's installed instance of a provider. |
| `integration_credential` | `tenant_integration_id` FK, `type` (oauth_token/refresh_token/api_key), `secret_ref` (secrets-manager pointer), `scopes[]`, `expires_at`, `revoked_at` | Scoped, revocable credentials; **secrets never stored in plaintext in the DB row**. |
| `oauth_app` | `provider_key`, `client_id`, `client_secret_ref`, `redirect_uris[]`, `granted_scopes[]` | Your app's registration with the provider (or, if you're the provider, third-party apps registered against you). |
| `webhook_endpoint` | `tenant_integration_id` FK (nullable), `url`, `event_types[]`, `signing_secret_ref`, `status` (active/paused/disabled), `is_verified`, `created_by` | Outbound target the tenant registers; URL validated against SSRF allow-list. |
| `webhook_event` | `event_key`, `event_id` (idempotent, unique), `payload`, `occurred_at` | The domain event to deliver (source of truth for replays). |
| `webhook_delivery` | `endpoint_id` FK, `event_id` FK, `status` (pending/success/failed/exhausted), `attempts`, `next_attempt_at`, `response_code`, `signature`, `correlation_id` | One delivery attempt-set per endpoint × event; retried + replayable. |
| `webhook_dlq` | `delivery_id` FK, `reason`, `parked_at`, `replayable` | Exhausted deliveries; replay from console. |
| `inbound_webhook` | `provider_key`, `signature_valid` (bool), `event_id`, `dedup_key` (unique), `payload`, `processed_at`, `status` | Verified, idempotent inbound events from providers. |

## Key screens & UX

| Screen | Primary fields / actions | States |
| --- | --- | --- |
| **Integrations catalog** | Grid of providers by category, search/filter, "Connect", capability badges, connected indicator | loading / empty / error |
| **Connect flow** | OAuth consent redirect or API-key form (scoped), select scopes, **Test connection** before save | connecting / test-passed / test-failed / connected |
| **Integration detail / console** | Config fields (per schema), credential status + expiry, reconnect / **revoke**, health + last test, activity log | connected / error / expired |
| **Webhook endpoints** | Add/edit URL, select event types, view signing secret (rotate), pause/disable, send test event | active / paused / unverified / disabled |
| **Delivery log** | Per-event deliveries, status, attempts, response code, **resend/replay**, filter by status/date | loading / empty / error |
| **DLQ console** | Parked deliveries, failure reason, bulk **replay** / discard | empty / has-items |
| **OAuth apps** (admin) | Registered apps, client id, redirect URIs, granted scopes, **revoke** | active / revoked |

## Rules & logic

- **[MUST]** Every external provider is behind a **swappable adapter** with timeouts, retries, and circuit breakers; swapping a provider must not require changes to business logic.
- **[MUST]** Credentials are **scoped, revocable, and stored in a secrets manager** (DB holds only a `secret_ref`). OAuth tokens are refreshed automatically; revocation is immediate and propagates to the adapter.
- **[MUST]** **Connect** must offer a **Test connection** that makes a real, least-privilege call and reports pass/fail *before* the integration is marked connected.
- **[MUST]** Outbound webhooks are **HMAC-signed + timestamped**, carry an **idempotent `event_id`**, are **retried with exponential backoff + jitter**, land in a **DLQ** on exhaustion, and are **replayable** from the event source.
- **[MUST]** Webhook destination URLs are validated against an **SSRF-safe allow-list**: reject private/link-local/metadata IP ranges, non-HTTPS, and unresolved/rebinding hosts; re-validate on delivery (not just on save).
- **[MUST]** Inbound webhooks are **signature-verified and idempotent** (`dedup_key`); an unverified or replayed inbound event is rejected/ignored, never processed twice.
- **[MUST]** Endpoint ownership requires a **verification step** (challenge/echo) before deliveries begin; consistently failing endpoints are auto-paused with an alert.
- **[MUST]** All connect/disconnect/revoke/credential/endpoint changes are **audited** (actor, action, before/after).
- **[SHOULD]** Rotate signing secrets with an overlap window (accept old + new during rotation).
- **[SHOULD]** Rate-limit outbound deliveries per endpoint; expose per-integration **health** and delivery success rate.
- **[SHOULD]** Provide a sandbox / test-event button and a published event catalog + payload schemas.

## Applicable standards

- [`../standards-kb/10-api-integration.md`](../standards-kb/10-api-integration.md) — adapters, signed/retryable/replayable webhooks, verified inbound, idempotency (canonical source).
- [`../standards-kb/04-security.md`](../standards-kb/04-security.md) — SSRF allow-list, secrets management, scoped/revocable credentials, OAuth handling.
- [`../standards-kb/05-reliability-resilience.md`](../standards-kb/05-reliability-resilience.md) — retries, backoff, DLQ, circuit breakers, auto-pause.
- [`../standards-kb/07-observability-aiops.md`](../standards-kb/07-observability-aiops.md) — delivery metrics, integration health, correlation ids.
- [`../standards-kb/06-data-management.md`](../standards-kb/06-data-management.md) — event source-of-truth for replay, dedup keys.
- [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md) — catalog/console/log screen states.
- [`../feature-blueprints/README.md`](../feature-blueprints/README.md) — shared conventions.

## Acceptance checklist

- [ ] Catalog + console: connect / configure / test-connection / reconnect / revoke, with health.
- [ ] Test connection makes a real least-privilege call and gates the "connected" state.
- [ ] Every provider behind a swappable adapter with timeouts/retries/circuit breakers.
- [ ] Credentials scoped + revocable in a secrets manager; DB holds only refs; revocation is immediate.
- [ ] OAuth apps: scoped grants, redirect-URI allow-list, revocable.
- [ ] Outbound webhooks HMAC-signed + timestamped, idempotent event id, retried with backoff, DLQ, replayable.
- [ ] Destination URLs validated against an SSRF-safe allow-list, re-checked at delivery time.
- [ ] Inbound webhooks signature-verified + idempotent (dedup); replays/unverified rejected.
- [ ] Endpoint ownership verification before delivery; failing endpoints auto-paused + alerted.
- [ ] Signing-secret rotation with overlap window.
- [ ] All connect/revoke/endpoint changes audited.
- [ ] All tables tenant-scoped with standard audit columns.
