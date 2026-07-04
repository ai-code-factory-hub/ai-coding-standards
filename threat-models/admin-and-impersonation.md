# Threat Model — Admin & Impersonation

> Filled STRIDE model for privileged admin consoles and "log in as user" (impersonation / support-view) features. Method & risk rating: [README.md](README.md). Blank copy: [threat-model-template.md](threat-model-template.md).
> **Owner:** Security Officer · **Extends:** [04 Security](../standards-kb/04-security.md), [20 Logging, Audit & Traceability](../standards-kb/20-logging-audit-and-traceability.md)

## Feature / asset

Internal admin/back-office tools, role management, and support impersonation ("view/act as this customer"). **Asset:** the most powerful accounts in the system — an admin compromise or an abused impersonation is a total, cross-tenant compromise with the fewest attackers needed. **Impersonation must be consented, time-boxed, and fully audited.**

## Data classification

Admin surfaces reach **all** tenants' data — treat as **Regulated/PII** by default.

## Actors

| Actor | Trusted? | Notes |
|---|---|---|
| Admin / support operator | Semi-trusted | insider-threat and account-takeover target |
| Impersonated customer | Data subject | must be protected + informed |
| Attacker who phishes an admin | **Untrusted** | inherits maximal blast radius |
| Lower-priv user probing for escalation | **Untrusted** | |

## Trust boundaries

- **B4 Privilege boundary** — user → admin. **B3** tenant → all-tenant. **B2** the admin login (should be stronger than user login).

## Attack surface

Admin authentication, role/permission assignment endpoints, the impersonation "start/stop" action, admin bulk operations (export/delete/config), the audit log itself, and any admin route reachable by a non-admin.

## STRIDE analysis

| # | STRIDE | Threat | How (attack path) | Mitigation `[MUST]`/`[SHOULD]` | Standards ref | Residual |
|---|---|---|---|---|---|---|
| S1 | Spoofing | **Admin account takeover** | Phished/stuffed admin credentials give full control | `[MUST]` mandatory phishing-resistant MFA (WebAuthn preferred) for all admin/staff; SSO with strong session policy; `[SHOULD]` IP allow-list / device attestation for admin console | [04](../standards-kb/04-security.md) | Med |
| S2 | Spoofing | **Impersonation as identity confusion** | Actions taken while impersonating look like the customer did them; attacker exploits the ambiguity | `[MUST]` maintain **dual identity** in the session (real operator + impersonated user); stamp both on every action and audit entry; persistent UI banner while impersonating | [20](../standards-kb/20-logging-audit-and-traceability.md) / [04](../standards-kb/04-security.md) | Low |
| T1 | Tampering | **Audit-log tampering** | Insider/attacker edits or deletes audit entries to cover tracks | `[MUST]` append-only, immutable, tamper-evident audit store (hash-chained / WORM); no app path can update/delete entries; ship to a separate write-once sink; alert on gaps | [20](../standards-kb/20-logging-audit-and-traceability.md) | Med |
| R1 | Repudiation | **Denied admin action** | Operator denies making a config change, export, or impersonated action | `[MUST]` audit every admin action + every impersonation start/stop with operator, target, reason, scope, timestamp, correlation id | [20](../standards-kb/20-logging-audit-and-traceability.md) | Low |
| I1 | Info disclosure | **Impersonation abuse (snooping)** | Operator impersonates or opens customer data with no business need | `[MUST]` require a **reason/ticket + customer consent** to start impersonation; **time-box** the session (auto-expire); scope to least privilege (read-only view where possible); notify the customer; `[SHOULD]` periodic access reviews + anomaly alerts on impersonation volume | [09](../standards-kb/09-compliance-privacy.md) / [04](../standards-kb/04-security.md) | Med |
| I2 | Info disclosure | **Mass data export by admin** | Operator (or hijacked account) bulk-exports all-tenant PII | `[MUST]` gate bulk export behind approval/dual-control + volume thresholds; rate-limit; audit with row counts; `[SHOULD]` DLP / anomaly alert on large exports | [06](../standards-kb/06-data-management.md) / [09](../standards-kb/09-compliance-privacy.md) | Med |
| D1 | DoS | **Destructive admin action** | Bulk delete / disruptive config change (accidental or malicious) takes down tenants | `[MUST]` guardrails on destructive ops (confirmation, dry-run, soft-delete + recovery window, dual-control for high-impact); scope blast radius | [05](../standards-kb/05-reliability-resilience.md) | Med |
| E1 | Elevation | **Privilege escalation** | Non-admin reaches an admin route (BFLA), or self-assigns a role, or impersonation grants operator > customer's own rights | `[MUST]` deny-by-default function-level authz on every admin operation; role changes require admin + are audited; impersonation grants **≤** the target's own permissions, never more; separation of duties (an operator can't grant themselves admin) | [04](../standards-kb/04-security.md) | Med |
| E2 | Elevation | **Standing privilege abuse** | Always-on admin rights widen the window for misuse | `[SHOULD]` just-in-time / time-bound privilege elevation with approval; least-privilege default roles; regular entitlement reviews | [04](../standards-kb/04-security.md) | Med |

## Top risks

1. **Impersonation abuse (I1)** — the defining risk; without consent + time-box + audit it's indistinguishable from spying.
2. **Admin account takeover (S1)** — one phished admin = whole-platform breach; enforce phishing-resistant MFA.
3. **Audit-log tampering (T1)** — if the log is mutable, every other control is deniable; make it append-only and externalized.

## Open risks / decisions

| Item | Decision | Owner | Due |
|---|---|---|---|
| Customer consent model for support access | Explicit opt-in per session vs. standing ToS consent + notification | | |
| JIT elevation | Adopt time-bound privilege with approval workflow | | |

---

## Mitigation checklist

- [ ] `[MUST]` Phishing-resistant MFA + SSO required for all admin/staff.
- [ ] `[MUST]` Deny-by-default function-level authz on every admin operation; separation of duties on role grants.
- [ ] `[MUST]` Impersonation requires reason/consent, is time-boxed/auto-expiring, least-privilege, and shows a persistent banner.
- [ ] `[MUST]` Impersonation grants ≤ the target's own permissions.
- [ ] `[MUST]` Dual identity (operator + target) stamped on every impersonated action and audit entry.
- [ ] `[MUST]` Append-only, immutable, tamper-evident audit externalized to a write-once sink; alert on gaps.
- [ ] `[MUST]` Bulk export/destructive ops gated by approval/dual-control + thresholds; soft-delete + recovery window.
- [ ] `[MUST]` Every admin action + impersonation start/stop audited → [20](../standards-kb/20-logging-audit-and-traceability.md).
- [ ] `[SHOULD]` JIT/time-bound elevation; periodic entitlement & impersonation reviews; anomaly alerts.
