# Feature Blueprint · Support & Impersonation

A helpdesk/ticketing surface plus **time-boxed, consented, fully-audited** impersonation ("view as tenant/user"), PII-redacted support-bundle export, and per-tenant health/status.

## What it is / when you need it

Support teams need to reproduce a customer's problem and to answer "what's wrong with my account?" fast. The dangerous shortcut is a "god mode" login that silently sees everything. This blueprint provides a **ticketing surface** (or a bridge to one), a **safe impersonation** capability that is **explicitly consented, time-boxed, scoped, and audited** on every action, a **PII-redacted support bundle** so engineers can debug without exposing raw personal data, and a **per-tenant health/status** view so support can see subscription, usage, integration, and incident state at a glance.

## Data model

> All tables carry the standard audit columns (`id`, `tenant_id`, `created_at`, `created_by`, `updated_at`, `updated_by`, `deleted_at`, `version`) and are filtered by `tenant_id` on every query. Support-agent identity is the platform (support) tenant; the acted-upon tenant is always recorded.

| Entity | Key fields | Notes |
| --- | --- | --- |
| `support_ticket` | `subject`, `body`, `requester_id`, `tenant_ref`, `category`, `priority`, `status` (open/pending/on_hold/resolved/closed), `assignee`, `sla_due_at`, `external_ref` | Native ticket or mirror of an external helpdesk (Zendesk/Freshdesk) via adapter. |
| `ticket_message` | `ticket_id` FK, `author`, `body`, `visibility` (public/internal), `attachments[]`, `at` | Threaded conversation; internal notes never shown to the customer. |
| `impersonation_grant` | `tenant_ref`, `target_user_id` (nullable = tenant-level view), `granted_by` (customer/admin), `consent_ref`, `scope` (read_only/limited_write), `reason`, `expires_at`, `status` (pending/active/expired/revoked) | The **consent + time-box + scope** authorising a session. |
| `impersonation_session` | `grant_id` FK, `agent_id`, `started_at`, `ended_at`, `ip`, `correlation_id` | An actual session opened under a grant. |
| `impersonation_action_log` | `session_id` FK, `action`, `entity_type`, `entity_id`, `method` (read/write), `redacted_payload`, `at` | **Append-only** log of every action taken while impersonating. Immutable. |
| `support_bundle` | `tenant_ref`, `requested_by`, `contents[]` (config/logs/metrics/diag), `redaction_profile`, `status` (generating/ready/expired), `file_ref`, `expires_at` | PII-redacted diagnostic export. |
| `tenant_health_snapshot` | `tenant_ref`, `subscription_state`, `usage_vs_quota`, `integration_health`, `error_rate`, `open_incidents`, `captured_at` | Per-tenant status roll-up for the support console. |

## Key screens & UX

| Screen | Primary fields / actions | States |
| --- | --- | --- |
| **Ticket queue** | Filter by status/priority/assignee/SLA; columns: subject, tenant, priority, SLA countdown, assignee; open ticket | loading / empty / error |
| **Ticket detail** | Threaded messages, internal-note toggle, status/assignee/priority, linked tenant health, "request impersonation" | open / pending / resolved / closed |
| **Tenant health / 360** | Subscription + plan, usage vs quota, integration health, recent errors/incidents, feature-flag state, quick actions | healthy / degraded / at-risk |
| **Request impersonation** | Select tenant/user, scope (read-only/limited-write), reason, duration, **consent capture** (customer approval or documented admin authorization) | pending-consent / approved / denied |
| **Active impersonation banner** | Persistent "You are viewing as <tenant/user> — read-only — ends in mm:ss", **exit** button, everything visibly marked | active (countdown) / expiring |
| **Impersonation audit** | Search by agent/tenant/date; per-session action log, reason, consent, duration | loading / empty |
| **Support bundle** | Choose contents + redaction profile, generate, download (expiring link) | generating / ready / expired |

## Rules & logic

- **[MUST]** Impersonation requires an **explicit grant**: recorded **consent** (customer approval, or documented authorization per policy), a **reason**, a **scope**, and a **hard expiry**. No standing "log in as anyone" capability.
- **[MUST]** Sessions are **time-boxed** and auto-terminate at `expires_at`; grants are **revocable** immediately by the customer or an admin.
- **[MUST]** **Default scope is read-only.** Any write during impersonation requires an explicitly elevated grant, is clearly attributed to the acting agent (not the impersonated user), and is separately audited. Certain actions (password/MFA change, billing changes, data export/deletion, viewing full secrets) are **never** permitted under impersonation.
- **[MUST]** **Every action** taken while impersonating is written to an **append-only, immutable log** capturing agent, tenant, target user, action, entity, read/write, and time. The customer can see when and by whom their account was accessed.
- **[MUST]** An impersonation session is **unmistakably visible**: a persistent banner with the target, scope, and countdown, on every screen. It never looks like a normal login.
- **[MUST]** **Support bundles are PII-redacted** by profile (mask emails/phones/identifiers/free-text) before export; raw PII is never included by default. Bundles are access-controlled and have **expiring** download links.
- **[MUST]** All support access is **tenant-scoped and least-privilege**; a support agent sees only tenants/tickets they are assigned or authorised for, and impersonation authorization is enforced **server-side**.
- **[SHOULD]** Bridge to an external helpdesk via a **swappable adapter** (sync tickets/status) rather than reinventing a full ITSM; keep internal notes internal.
- **[SHOULD]** Notify the tenant/user when impersonation starts and ends; surface a self-service "who accessed my account" view.
- **[SHOULD]** Tenant health snapshots refresh on a schedule and flag at-risk tenants for proactive support.

> **Cross-reference:** consented "view as" also appears in [`audit-log-and-activity.md`](audit-log-and-activity.md); this blueprint is the operational support surface around it. Keep the audit model consistent between the two.

## Applicable standards

- [`../standards-kb/04-security.md`](../standards-kb/04-security.md) — consented/time-boxed/scoped impersonation, server-side authorization, forbidden actions, least privilege.
- [`../standards-kb/09-compliance-privacy.md`](../standards-kb/09-compliance-privacy.md) — consent records, PII redaction, access-transparency, retention of logs/bundles.
- [`../standards-kb/07-observability-aiops.md`](../standards-kb/07-observability-aiops.md) — immutable action logging, correlation ids, tenant health signals.
- [`../standards-kb/06-data-management.md`](../standards-kb/06-data-management.md) — redaction, expiring artifacts, immutable audit storage.
- [`../standards-kb/10-api-integration.md`](../standards-kb/10-api-integration.md) — helpdesk adapter for external ticketing.
- [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md) — impersonation banner, queue/health screen states.
- [`../feature-blueprints/README.md`](../feature-blueprints/README.md) — shared conventions.

## Acceptance checklist

- [ ] Ticket queue + threaded detail with public vs internal notes and SLA tracking.
- [ ] Impersonation requires explicit consent + reason + scope + hard expiry; no standing god-mode.
- [ ] Sessions are time-boxed, auto-terminate, and are revocable immediately by customer/admin.
- [ ] Default read-only; writes need elevated grant, are agent-attributed and separately audited.
- [ ] Forbidden actions (credentials/MFA, billing, export/delete, full secrets) blocked under impersonation.
- [ ] Every impersonated action written to an append-only immutable log; customer can see access history.
- [ ] Persistent, unmistakable impersonation banner with target + scope + countdown + exit.
- [ ] Support bundles PII-redacted by profile; access-controlled with expiring links.
- [ ] Support access tenant-scoped + least-privilege; authorization enforced server-side.
- [ ] External helpdesk bridged via swappable adapter; tenant health/status console with at-risk flags.
- [ ] Start/end impersonation notifications to the tenant/user.
- [ ] All tables tenant-scoped with standard audit columns.
