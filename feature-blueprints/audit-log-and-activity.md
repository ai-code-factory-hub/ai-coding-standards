# Feature Blueprint · Audit Log & Activity

> An immutable audit log viewer (who/what/when/before→after/correlation id), a user-facing activity feed, and time-boxed **consented impersonation** ("view as") — all filterable and exportable, with the export itself audited.

## What it is / when you need it

Three related surfaces from one event backbone: a **compliance-grade audit log** (tamper-evident, admin-facing), a friendly **activity feed** (end-user facing, "here's what happened on your account"), and **impersonation** (support "views as" a user, consented and time-boxed). You need this for security, compliance (SOC2/ISO), debugging, and customer trust. Implements requirements in [`../standards-kb/04-security.md`](../standards-kb/04-security.md) and [`../standards-kb/07-observability-aiops.md`](../standards-kb/07-observability-aiops.md).

## Data model

Tenant-scoped. Audit records are **append-only** — no update/delete; `deleted_at` never set.

### AuditEvent (immutable)

| Field | Type | Notes |
| --- | --- | --- |
| `id` | ulid | Time-ordered PK |
| `tenant_id` | fk | |
| `occurred_at` | timestamp | Event time (UTC) |
| `actor_type` | enum | `user` \| `service_account` \| `system` \| `impersonator` |
| `actor_id` | fk | Who acted |
| `on_behalf_of_id` | fk? | Set when impersonating (real actor + target) |
| `action` | string | `resource:action`, e.g. `invoice:update` |
| `resource_type` | string | e.g. `invoice` |
| `resource_id` | string | Affected entity |
| `before_json` | jsonb? | State before (redacted PII) |
| `after_json` | jsonb? | State after |
| `outcome` | enum | `success` \| `failure` \| `denied` |
| `correlation_id` | string | Ties events across a request/workflow |
| `request_id` | string | Single request |
| `ip_address` | string | Redacted/hashed per policy |
| `user_agent` | string | |
| `source` | enum | `web` \| `api` \| `mobile` \| `system_job` |
| `hash_prev` | string | Hash-chain: hash of previous event (tamper evidence) |
| `hash_self` | string | Hash of this record incl. `hash_prev` |

### ActivityFeedItem (user-facing projection)

| Field | Type | Notes |
| --- | --- | --- |
| `tenant_id` | fk | |
| `user_id` | fk | Whose feed |
| `audit_event_id` | fk | Source event |
| `summary` | string | Human phrasing, e.g. "You updated Invoice #1032" |
| `icon` | string | |
| `is_read` | bool | |
| `occurred_at` | timestamp | |

### ImpersonationSession (consented "view as")

| Field | Type | Notes |
| --- | --- | --- |
| `tenant_id` | fk | |
| `impersonator_id` | fk | Support/admin acting |
| `target_user_id` | fk | User being viewed |
| `reason` | text | Required justification |
| `consent_ref` | string? | Consent token / ticket ref |
| `scope` | enum | `read_only` \| `full` |
| `started_at` | timestamp | |
| `expires_at` | timestamp | **Required**, time-boxed |
| `ended_at` | timestamp? | Manual end |
| `status` | enum | `active` \| `expired` \| `revoked` \| `ended` |

## Key screens & UX

1. **Audit log viewer (admin)** — dense, virtualized table: time, actor, action, resource, outcome, correlation id. Row expands to a **before→after diff**. Filters: date range, actor, action/resource type, outcome, correlation id, source, impersonation-only. *Loading:* skeleton rows. *Empty:* "No events match these filters." *Error:* retry. Export (CSV/JSON) — the export action itself writes an AuditEvent.
2. **Event detail / diff** — side-by-side or inline JSON diff (PII redacted), full metadata (IP, UA, request/correlation ids), and a "show related events" link that pivots on `correlation_id`.
3. **Activity feed (end user)** — reverse-chronological, human-readable, grouped by day, unread markers, "load more". *Empty:* "No recent activity."
4. **Impersonation start** — support enters target user, reason, scope (read-only default), duration; requires consent per policy; shows a persistent **"You are viewing as <user>"** banner throughout the session with an "Exit" control.
5. **Impersonation history** — who impersonated whom, when, why, scope, duration — always visible to tenant admins.

## Rules & logic

- [MUST] Audit events are **append-only and immutable**: no update/delete path, enforced at the storage layer; maintain a **hash chain** (`hash_prev`/`hash_self`) for tamper evidence.
- [MUST] Capture who/what/when + **before→after** + `correlation_id` for every state-changing action.
- [MUST] Redact/mask PII and secrets in `before_json`/`after_json` per data policy — see [`../standards-kb/06-data-management.md`](../standards-kb/06-data-management.md).
- [MUST] Impersonation is **time-boxed** (`expires_at` required), **consented** (`reason`, and `consent_ref` where policy demands), and defaults to **read-only**; every action during a session is stamped with both `actor_id` (impersonator) and `on_behalf_of_id` (target).
- [MUST] A visible banner is shown for the entire impersonation session; the session auto-expires and can be revoked; expiry is enforced server-side.
- [MUST] Exports are permission-gated and **themselves audited** (who exported what filter, when).
- [MUST] All views are tenant-scoped; end users see only their own activity, admins see the tenant's audit log.
- [SHOULD] Retain audit events per compliance retention (often years) on cheaper immutable/WORM storage; feed provides a shorter window.
- [SHOULD] Support correlation-id pivoting to reconstruct a full workflow across services.
- [SHOULD] Emit high-severity security events (e.g. permission changes, impersonation start) to the security/SIEM pipeline — see [`../standards-kb/07-observability-aiops.md`](../standards-kb/07-observability-aiops.md).

## Applicable standards

- Audit, immutability, impersonation, PII redaction — [`../standards-kb/04-security.md`](../standards-kb/04-security.md)
- Observability, correlation ids, SIEM feed — [`../standards-kb/07-observability-aiops.md`](../standards-kb/07-observability-aiops.md)
- Retention, WORM storage, PII masking — [`../standards-kb/06-data-management.md`](../standards-kb/06-data-management.md)
- Tenant isolation — [`../standards-kb/01-architecture-multitenancy.md`](../standards-kb/01-architecture-multitenancy.md)
- Viewer UX/states, impersonation banner — [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md)
- API audit for machine actors — [`../standards-kb/10-api-integration.md`](../standards-kb/10-api-integration.md)

## Acceptance checklist

- [ ] Audit events are append-only and immutable, with a verifiable hash chain.
- [ ] Every state-changing action records who/what/when + before→after + correlation id.
- [ ] PII/secrets are redacted in stored before/after payloads.
- [ ] Log viewer filters (date/actor/action/resource/outcome/correlation/source) work; diff view renders.
- [ ] Correlation-id pivot reconstructs a workflow.
- [ ] Activity feed shows human-readable, per-user, tenant-scoped items with read state.
- [ ] Impersonation is consented, time-boxed, read-only by default, banner-visible, and revocable.
- [ ] Impersonated actions carry both impersonator and target ids; impersonation history is visible to admins.
- [ ] Exports are permission-gated and audited.
