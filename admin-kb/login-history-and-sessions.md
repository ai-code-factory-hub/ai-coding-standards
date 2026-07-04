# Admin · Login History & Sessions

> A tenant-scoped view of every authentication event (success/failure, IP, device, geo, method, MFA), the list of currently **active sessions**, and admin controls to **force-logout / revoke** — with concurrent-session limits and an impossible-travel flag.

## What it is / when you need it

The identity-forensics surface of the admin area: it answers *"who logged in, from where, how — and who is logged in right now?"*. Security teams use it to spot account takeover, ops uses it to end a stuck or shared session, and support uses it to confirm a user actually reached the app. It reads the same auth event stream that feeds [`security-center.md`](security-center.md) but presents it per-user and per-session rather than as alerts. Every admin action here (revoke, force-logout) is itself audited per [`../standards-kb/20-logging-audit-and-traceability.md`](../standards-kb/20-logging-audit-and-traceability.md).

## Data model

Tenant-scoped. `LoginEvent` is **append-only** (no update/delete). Sessions are mutable but every state change emits an event. Retention: login events retained per security-retention policy (commonly 12–24 months, longer on WORM for compliance); active sessions live only until expiry/revoke.

### LoginEvent (append-only)

| Field | Type | Notes |
| --- | --- | --- |
| `id` | ulid | Time-ordered PK |
| `tenant_id` | fk | |
| `user_id` | fk? | Null when the identifier didn't resolve to a user |
| `identifier` | string | Email/username attempted (masked in UI) |
| `occurred_at` | timestamp | Event time (UTC) |
| `outcome` | enum | `success` \| `failure` \| `locked_out` \| `challenge` |
| `failure_reason` | enum? | `bad_password` \| `unknown_user` \| `mfa_failed` \| `disabled` \| `rate_limited` |
| `method` | enum | `password` \| `sso_saml` \| `sso_oidc` \| `magic_link` \| `passkey` \| `api_token` |
| `mfa_used` | bool | Whether a second factor was satisfied |
| `mfa_type` | enum? | `totp` \| `sms` \| `push` \| `webauthn` |
| `ip_address` | string | Stored hashed/truncated per policy; masked in UI |
| `geo` | jsonb | `{country, region, city, lat, lng}` from IP (coarse) |
| `device` | jsonb | `{os, browser, device_type, client_app}` |
| `user_agent` | string | Raw UA |
| `session_id` | fk? | Set on success |
| `impossible_travel` | bool | Flagged when geo/time delta exceeds threshold vs. prior event |
| `correlation_id` | string | Ties to downstream requests |

### Session (active + historical)

| Field | Type | Notes |
| --- | --- | --- |
| `id` | ulid | Session PK |
| `tenant_id` | fk | |
| `user_id` | fk | |
| `created_at` | timestamp | Login time |
| `last_seen_at` | timestamp | Sliding activity marker |
| `expires_at` | timestamp | Absolute/idle expiry |
| `ip_address` | string | Masked in UI |
| `geo` | jsonb | Coarse location |
| `device` | jsonb | Device fingerprint summary |
| `method` | enum | How the session was established |
| `is_current` | bool | The session the admin/user is viewing from |
| `status` | enum | `active` \| `expired` \| `revoked` \| `logged_out` |
| `revoked_by` | fk? | Admin who force-logged-out |
| `revoke_reason` | text? | Required on manual revoke |

### ConcurrentSessionPolicy (per tenant/role)

| Field | Type | Notes |
| --- | --- | --- |
| `tenant_id` | fk | |
| `role` | string? | Null = tenant default |
| `max_sessions` | int | Concurrent-session cap |
| `on_exceed` | enum | `deny_new` \| `revoke_oldest` |
| `idle_timeout_min` | int | Idle expiry |
| `absolute_timeout_min` | int | Hard cap regardless of activity |

## Key screens & UX

1. **Login history (table)** — virtualized, reverse-chronological: time, user (masked identifier), outcome, method, MFA badge, device, geo, masked IP, impossible-travel flag. Filters: date range, user, outcome, method, MFA used, country, impossible-travel-only, failures-only. Row expands to full event metadata + "show this user's other logins". *Loading:* skeleton rows. *Empty:* "No login events match these filters." *Error:* inline retry.
2. **Session inspector (per user)** — all sessions for one user with active-first ordering; each row shows device, geo, last-seen, method, expiry. Actions: **Revoke session**, **Revoke all other sessions**. Confirmation requires a reason.
3. **Active sessions (tenant-wide)** — every live session across the tenant, sortable by last-seen; bulk **force-logout** selection; filter by role, device type, stale (>N min idle).
4. **Impossible-travel & anomaly strip** — flagged events surfaced at the top with the two locations and time delta; "acknowledge" or "revoke session" inline.
5. **Concurrent-session policy** — read/edit `max_sessions`, overflow behavior, idle/absolute timeouts per role. *Empty:* falls back to tenant default.

## Rules & logic

- [MUST] Admin-only, **RBAC-gated** and **tenant-scoped**; admins never see another tenant's sessions or events — see [`../standards-kb/04-security.md`](../standards-kb/04-security.md).
- [MUST] Login events are **append-only**; no edit/delete path exists.
- [MUST] **Mask PII** (IP, precise geo, full identifier) by default; unmasking is a permission-gated, audited action.
- [MUST] **Force-logout / revoke** takes effect server-side immediately (token/session invalidation), not just client-side; the revoked session's next request fails.
- [MUST] Every revoke/force-logout **emits its own audit entry** (actor, target session, reason) per [`../standards-kb/20-logging-audit-and-traceability.md`](../standards-kb/20-logging-audit-and-traceability.md).
- [MUST] **Exports are permission-gated and themselves audited** (who exported which filter, when).
- [MUST] Enforce **concurrent-session limits** per policy: `deny_new` blocks the login, `revoke_oldest` ends the least-recently-used session and records why.
- [SHOULD] Compute the **impossible-travel** flag from geo distance ÷ time delta exceeding a plausible-speed threshold; flagged events feed [`security-center.md`](security-center.md).
- [SHOULD] Emit high-severity auth events (repeated failures, impossible travel, new-country login) to the security/SIEM pipeline — see [`../standards-kb/07-observability-aiops.md`](../standards-kb/07-observability-aiops.md).
- [SHOULD] Show the user their own login history + let them self-revoke sessions (surfaces the same data, non-admin scope).
- [SHOULD] Retain events per security retention; expire raw IPs earlier than the aggregate event where policy requires.

## Applicable standards

- Authentication, session revocation, MFA, PII masking — [`../standards-kb/04-security.md`](../standards-kb/04-security.md)
- Audit of admin actions, append-only, export audit — [`../standards-kb/20-logging-audit-and-traceability.md`](../standards-kb/20-logging-audit-and-traceability.md)
- Anomaly/SIEM feed, correlation ids — [`../standards-kb/07-observability-aiops.md`](../standards-kb/07-observability-aiops.md)
- Session-limit enforcement, graceful revoke — [`../standards-kb/05-reliability-resilience.md`](../standards-kb/05-reliability-resilience.md)
- Geo/IP handling, minimization, retention — [`../standards-kb/09-compliance-privacy.md`](../standards-kb/09-compliance-privacy.md)
- Table/filter UX, loading/empty/error states — [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md)
- Related: [`security-center.md`](security-center.md) · [`../feature-blueprints/audit-log-and-activity.md`](../feature-blueprints/audit-log-and-activity.md) · [`README.md`](README.md)

## Acceptance checklist

- [ ] Login history is tenant-scoped, append-only, and filterable by date/user/outcome/method/MFA/country/impossible-travel.
- [ ] Each event records outcome, method, MFA, masked IP, coarse geo, and device.
- [ ] Active-sessions list shows every live session with device/geo/last-seen.
- [ ] Force-logout / revoke invalidates the session server-side immediately and requires a reason.
- [ ] Revoke and unmask actions emit their own audit entries.
- [ ] Concurrent-session limits are enforced per policy (`deny_new` / `revoke_oldest`).
- [ ] Impossible-travel flag is computed and surfaced; flagged events reach the security pipeline.
- [ ] PII (IP/geo/identifier) is masked by default; exports are permission-gated and audited.
- [ ] Loading / empty / error states are implemented on every screen.
