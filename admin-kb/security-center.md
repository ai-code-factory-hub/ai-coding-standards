# Admin · Security Center

> The single pane for security posture: failed-login & lockout events, brute-force / anomaly detection, permission & role changes, MFA enrollment status, suspicious IPs, WAF / blocked requests, and **security alerts with acknowledgement** and triage.

## What it is / when you need it

Where a tenant's security team lives day-to-day. It aggregates signals that individual surfaces emit — auth failures from [`login-history-and-sessions.md`](login-history-and-sessions.md), permission changes from RBAC, edge blocks from the WAF — into **alerts** that can be triaged, acknowledged, and resolved. You need it for incident response, for continuous monitoring (SOC2/ISO expectations), and to prove MFA coverage. It is a read/triage surface: it does not itself change security config, but it links out to the surfaces that do. Every acknowledgement and every unmask is audited per [`../standards-kb/20-logging-audit-and-traceability.md`](../standards-kb/20-logging-audit-and-traceability.md).

## Data model

Tenant-scoped. `SecurityEvent` is **append-only**; `SecurityAlert` is a mutable triage wrapper over one or more events. Retention: events per security-retention policy (often 12–24 months, WORM where required); alerts retained through resolution + audit window.

### SecurityEvent (append-only)

| Field | Type | Notes |
| --- | --- | --- |
| `id` | ulid | Time-ordered PK |
| `tenant_id` | fk | |
| `occurred_at` | timestamp | UTC |
| `category` | enum | `auth_failure` \| `lockout` \| `brute_force` \| `permission_change` \| `role_change` \| `mfa_change` \| `suspicious_ip` \| `waf_block` \| `anomaly` |
| `severity` | enum | `info` \| `low` \| `medium` \| `high` \| `critical` |
| `actor_id` | fk? | Who triggered (may be null/unknown) |
| `target_id` | fk? | Affected user/resource |
| `ip_address` | string | Masked in UI, hashed at rest |
| `geo` | jsonb | Coarse location |
| `detail_json` | jsonb | Category-specific payload (e.g. old→new role, rule id) |
| `correlation_id` | string | Links related events |
| `source` | enum | `app` \| `edge_waf` \| `auth` \| `siem` |

### SecurityAlert (triage wrapper)

| Field | Type | Notes |
| --- | --- | --- |
| `id` | ulid | PK |
| `tenant_id` | fk | |
| `rule_id` | fk | Detection rule that fired |
| `title` | string | Human summary |
| `severity` | enum | Rolled up from events |
| `event_ids` | ulid[] | Contributing events |
| `status` | enum | `open` \| `acknowledged` \| `investigating` \| `resolved` \| `false_positive` |
| `acknowledged_by` | fk? | |
| `acknowledged_at` | timestamp? | |
| `resolved_by` | fk? | |
| `resolution_note` | text? | Required to resolve/dismiss |
| `first_seen_at` | timestamp | |
| `last_seen_at` | timestamp | Updated as events recur |
| `count` | int | Event count in the group |

### DetectionRule

| Field | Type | Notes |
| --- | --- | --- |
| `tenant_id` | fk? | Null = platform default |
| `category` | enum | Matches `SecurityEvent.category` |
| `threshold` | jsonb | e.g. `{failures: 5, window_min: 10}` |
| `severity` | enum | Alert severity when it fires |
| `enabled` | bool | |
| `notify_channels` | string[] | Routes to notifications/on-call |

### MfaEnrollmentStatus (projection, per user)

| Field | Type | Notes |
| --- | --- | --- |
| `user_id` | fk | |
| `enrolled` | bool | |
| `factors` | string[] | `totp` / `webauthn` / `sms` |
| `enforced` | bool | Whether policy requires MFA for this user's role |
| `last_verified_at` | timestamp? | |

## Key screens & UX

1. **Security overview** — posture tiles: open alerts by severity, failed-login rate (trend), MFA-enrollment coverage %, suspicious-IP count, WAF blocks (24h). Each tile links to its detail list. *Loading:* skeleton tiles. *Empty:* "No security events in this window." *Error:* retry.
2. **Alerts inbox** — triage queue sorted by severity then recency: title, category, severity, count, first/last seen, status. Actions: **Acknowledge**, **Assign / mark investigating**, **Resolve**, **Mark false-positive** (note required). Filters: severity, category, status, date. Bulk-acknowledge supported.
3. **Alert detail** — the grouped events timeline, contributing IPs/actors, correlation-id pivot, and links to the source surface (e.g. "view sessions for this user"). Acknowledgement/resolution audit trail inline.
4. **Failed-login & lockout list** — per-attempt events with brute-force clustering (N failures / window highlighted). Inline "revoke sessions" / "lock account" links out to identity.
5. **Permission & role-change log** — old→new role/permission diff, who changed it, when — high-severity by default.
6. **MFA enrollment board** — per-user enrollment vs. enforcement; filter to *not enrolled but enforced* (the risk list).
7. **Suspicious IPs & WAF blocks** — blocked/anomalous IPs with geo, rule id, block count; drill to the raw edge events.

## Rules & logic

- [MUST] Admin/security-role only, **RBAC-gated** and **tenant-scoped** — see [`../standards-kb/04-security.md`](../standards-kb/04-security.md).
- [MUST] Security events are **append-only**; alerts are the only mutable object and every status change is recorded.
- [MUST] **Acknowledge / resolve / false-positive emit their own audit entries** (actor, alert, note) per [`../standards-kb/20-logging-audit-and-traceability.md`](../standards-kb/20-logging-audit-and-traceability.md).
- [MUST] **Mask PII/IPs** by default; unmasking is permission-gated and audited.
- [MUST] Permission/role changes and MFA changes are **always captured** as high-severity events, regardless of where they originate.
- [MUST] **Exports are permission-gated and themselves audited**.
- [SHOULD] Run detection rules (brute-force, impossible travel, privilege escalation, new-country login) with configurable thresholds; firing an alert routes to on-call/notifications per rule.
- [SHOULD] De-duplicate recurring signals into a single alert (`count`, `last_seen_at`) rather than flooding the inbox.
- [SHOULD] Feed high/critical alerts to the **SIEM / observability** pipeline — see [`../standards-kb/07-observability-aiops.md`](../standards-kb/07-observability-aiops.md).
- [SHOULD] Surface MFA-enrollment gaps against enforcement policy so admins can chase non-compliant users.

## Applicable standards

- Auth, MFA, lockout, WAF, PII masking — [`../standards-kb/04-security.md`](../standards-kb/04-security.md)
- Audit of triage actions, append-only, export audit — [`../standards-kb/20-logging-audit-and-traceability.md`](../standards-kb/20-logging-audit-and-traceability.md)
- Anomaly detection, SIEM feed, alerting — [`../standards-kb/07-observability-aiops.md`](../standards-kb/07-observability-aiops.md)
- Incident response, alert routing/resilience — [`../standards-kb/05-reliability-resilience.md`](../standards-kb/05-reliability-resilience.md)
- Retention, minimization of security signals — [`../standards-kb/09-compliance-privacy.md`](../standards-kb/09-compliance-privacy.md)
- Inbox/triage UX, states — [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md)
- Related: [`login-history-and-sessions.md`](login-history-and-sessions.md) · [`../feature-blueprints/audit-log-and-activity.md`](../feature-blueprints/audit-log-and-activity.md) · [`../feature-blueprints/notifications-center.md`](../feature-blueprints/notifications-center.md) · [`README.md`](README.md)

## Acceptance checklist

- [ ] Security overview aggregates alerts, failed-login rate, MFA coverage, suspicious IPs, and WAF blocks — tenant-scoped.
- [ ] Alerts can be acknowledged, assigned, resolved, and marked false-positive with a required note.
- [ ] Every triage state change and every unmask emits an audit entry.
- [ ] Permission/role changes and MFA changes are captured as high-severity events.
- [ ] Failed logins cluster into brute-force detections per configurable thresholds.
- [ ] MFA-enrollment board flags users not enrolled where MFA is enforced.
- [ ] Suspicious IPs and WAF blocks are listed and drill to raw edge events.
- [ ] High/critical alerts route to on-call and feed the SIEM pipeline.
- [ ] PII/IPs masked by default; exports permission-gated and audited; loading/empty/error states present.
