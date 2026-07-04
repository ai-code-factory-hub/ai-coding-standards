# Admin · Announcements & Maintenance

> Publish in-app announcements/banners with audience targeting (tenant / role / %), schedule maintenance windows with a status-page message, and control publish/expire — all audited.

## What it is / when you need it

The broadcast surface for the product: tell users about new features, planned downtime, incidents, or policy changes without a code deploy. Two related jobs live here — lightweight **announcements/banners** targeted at a slice of users, and **maintenance windows** that schedule downtime, show a countdown, and post a status-page message. You need it the moment you have users to inform and changes to communicate; doing it deliberately (targeted, scheduled, expiring, audited) beats hard-coded banners and Slack-only heads-ups.

## Data model

Platform- or tenant-scoped depending on audience; standard audit columns.

### Announcement

| Field | Type | Notes |
| --- | --- | --- |
| `id` | uuid | PK |
| `title` | string | |
| `body` | text | Rich/markdown; localizable (see [`localization-and-content.md`](localization-and-content.md)) |
| `type` | enum | `info` \| `success` \| `warning` \| `critical` |
| `placement` | enum | `top_banner` \| `modal` \| `inbox` \| `toast` |
| `dismissible` | bool | Can users close it? |
| `cta_label` / `cta_url` | string? | Optional call-to-action |
| `status` | enum | `draft` \| `scheduled` \| `published` \| `expired` \| `archived` |
| `starts_at` / `ends_at` | timestamp | Publish window; auto-expire at `ends_at` |
| `priority` | int | Ordering when multiple are active |
| `created_by` / `published_by` | fk | Actors |

### AnnouncementTargeting

| Field | Type | Notes |
| --- | --- | --- |
| `announcement_id` | fk | |
| `scope` | enum | `all` \| `tenant` \| `role` \| `plan` \| `percentage` |
| `tenant_ids` | uuid[]? | For `tenant` scope |
| `roles` | string[]? | For `role` scope |
| `plans` | string[]? | For `plan` scope |
| `rollout_pct` | int? | For `percentage` (deterministic hash of user id) |
| `locales` | string[]? | Restrict to certain languages |

### AnnouncementDismissal

| Field | Type | Notes |
| --- | --- | --- |
| `announcement_id` | fk | |
| `user_id` | fk | |
| `dismissed_at` | timestamp | So it doesn't re-show |

### MaintenanceWindow

| Field | Type | Notes |
| --- | --- | --- |
| `id` | uuid | PK |
| `title` | string | e.g. "Scheduled database upgrade" |
| `scope` | enum | `platform` \| `tenant` \| `region` \| `service` |
| `affected_ref` | string[]? | Tenant/region/service identifiers |
| `starts_at` / `ends_at` | timestamp | Window |
| `status` | enum | `scheduled` \| `in_progress` \| `completed` \| `cancelled` |
| `mode` | enum | `notice_only` \| `read_only` \| `full_downtime` |
| `status_page_message` | text | Public-facing message |
| `notify_lead_time` | duration | Pre-window heads-up (e.g. 24h, 1h) |
| `created_by` | fk | |

### StatusPageIncident (optional, for unplanned)

| Field | Type | Notes |
| --- | --- | --- |
| `maintenance_window_id` | fk? | Links planned work |
| `severity` | enum | `minor` \| `major` \| `critical` |
| `state` | enum | `investigating` \| `identified` \| `monitoring` \| `resolved` |
| `updates_json` | jsonb | Timeline of posted updates |

## Key screens & UX

1. **Announcements list** — table with title, type, placement, status, target summary, window, and impression/dismissal counts. Filter by status/type. *Loading:* skeletons. *Empty:* "No announcements — create one." *Error:* retry.
2. **Compose announcement** — title/body (localizable), type + placement, dismissible toggle, optional CTA, and a **targeting builder** (all / specific tenants / roles / plans / % rollout / locales) with a **live estimated-reach count**. Schedule `starts_at`/`ends_at`. Preview across placements. *Validation:* CTA url, non-empty body, window sanity.
3. **Schedule / publish** — publish now or schedule; a scheduled item flips to `published` at `starts_at` and **auto-expires** at `ends_at`; manual expire/unpublish available. Confirmation for `critical`/modal types.
4. **Maintenance windows** — calendar/list of scheduled windows with scope, mode (notice / read-only / full downtime), window, and status. Create/edit with a status-page message and pre-window notification lead time.
5. **Maintenance runtime** — when `in_progress`, the app shows the banner/countdown and, for `read_only`/`full_downtime`, enforces the mode; status page reflects `status_page_message`. Operators can extend, complete early, or cancel.
6. **Status page view** — public/tenant-facing summary of active and upcoming maintenance and any incidents.

## Rules & logic

- [MUST] Publishing, scheduling, editing, expiring, and cancelling are **RBAC-gated** (comms/ops admin roles) and **audited** with before→after per [`../standards-kb/20-logging-audit-and-traceability.md`](../standards-kb/20-logging-audit-and-traceability.md).
- [MUST] Targeting is **tenant-scoped-aware**: a tenant-scoped announcement only reaches users of the named tenant(s); `percentage` rollout is **deterministic** (stable hash of user id) so a user's inclusion doesn't flicker across page loads.
- [MUST] Announcements **auto-expire** at `ends_at` (never linger after their window) and respect per-user dismissal so a dismissed, dismissible banner does not re-appear.
- [MUST] Maintenance windows enforce their `mode`: `read_only` blocks writes, `full_downtime` serves a maintenance page; the mode change and window boundaries are audited.
- [MUST] A public `status_page_message` is required before a maintenance window can go `in_progress`, and pre-window notifications fire at the configured lead time.
- [MUST] Announcement body/content is **localizable** and resolves via the fallback chain (see [`localization-and-content.md`](localization-and-content.md)); no raw keys shown.
- [SHOULD] When multiple announcements target the same user, resolve by `priority` then recency and cap simultaneous banners to avoid noise.
- [SHOULD] Track impressions and dismissals for reach/effectiveness reporting (privacy-respecting counts, not content of who-saw-what beyond what's needed).
- [SHOULD] Allow **cancel** of a scheduled window/announcement before it starts, and **early completion** of an in-progress window, both audited.
- [SHOULD] Critical/downtime announcements should be non-dismissible for the duration and surface on the status page.

## Applicable standards

- Announcement/maintenance audit trail — [`../standards-kb/20-logging-audit-and-traceability.md`](../standards-kb/20-logging-audit-and-traceability.md)
- Access control for publishing — [`../standards-kb/04-security.md`](../standards-kb/04-security.md)
- Tenant-scoped targeting — [`../standards-kb/01-architecture-multitenancy.md`](../standards-kb/01-architecture-multitenancy.md)
- Maintenance mode, status page, incident comms — [`../standards-kb/05-reliability-resilience.md`](../standards-kb/05-reliability-resilience.md), [`../standards-kb/07-observability-aiops.md`](../standards-kb/07-observability-aiops.md)
- Banner/modal UX & states, accessibility of alerts — [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md)
- Localized announcement copy — [`../standards-kb/13-i18n-accessibility.md`](../standards-kb/13-i18n-accessibility.md)
- Related — [`localization-and-content.md`](localization-and-content.md), [`../feature-blueprints/notifications-center.md`](../feature-blueprints/notifications-center.md), [`../admin-kb/README.md`](../admin-kb/README.md)

## Acceptance checklist

- [ ] Announcements support targeting by all / tenant / role / plan / % with a live estimated-reach count.
- [ ] Percentage rollout is deterministic per user (no flicker across loads).
- [ ] Scheduled announcements auto-publish at `starts_at` and auto-expire at `ends_at`.
- [ ] Dismissible banners honor per-user dismissal and don't re-appear.
- [ ] Announcement copy is localizable and resolves via the fallback chain.
- [ ] Maintenance windows enforce their mode (notice / read-only / full downtime).
- [ ] A status-page message is required before a window goes in-progress; pre-window notifications fire.
- [ ] Publish / schedule / edit / expire / cancel are RBAC-gated and audited with before→after.
- [ ] Multiple concurrent announcements resolve by priority and are capped to reduce noise.
- [ ] Loading / empty / error states designed for list, composer, and maintenance screens.
