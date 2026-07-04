# Threat Model — Authentication & Sessions

> Filled STRIDE model for login, session, and token handling. Method & risk rating: [README.md](README.md). Blank copy: [threat-model-template.md](threat-model-template.md).
> **Owner:** Security Officer · **Extends:** [04 Security](../standards-kb/04-security.md)

## Feature / asset

Login, logout, session establishment, token issuance/refresh, MFA, and account recovery. **Asset:** user identity and the session — the keys to every other feature. Compromise here compromises everything downstream.

## Data classification

**Confidential / PII** — credentials, MFA secrets, session tokens, recovery tokens, and login metadata (IP/device).

## Actors

| Actor | Trusted? | Notes |
|---|---|---|
| Registered user | Trusted (post-auth) | the identity being protected |
| Anonymous internet caller | **Untrusted** | source of credential stuffing, enumeration |
| Attacker with a stolen token/cookie | **Untrusted** | bypasses the login step entirely |
| Identity provider (OIDC/SAML) | Semi-trusted | external boundary |

## Trust boundaries

- **B1 Network edge** — anonymous internet → login endpoint.
- **B2 Authentication boundary** — the moment anonymous becomes identified. The highest-value boundary in the system.
- **B5 External** — SSO/IdP redirect flows.

## Attack surface

Login form, refresh/token endpoint, MFA challenge, "forgot password" flow, SSO callback, session cookie, remember-me tokens, and every response's timing/error text (enumeration oracle).

## STRIDE analysis

| # | STRIDE | Threat | How (attack path) | Mitigation `[MUST]`/`[SHOULD]` | Standards ref | Residual |
|---|---|---|---|---|---|---|
| S1 | Spoofing | **Credential stuffing** | Attacker replays breached email/password pairs at scale | `[MUST]` breached-password check on set/login; per-account + per-IP rate limit and lockout with backoff; CAPTCHA/step-up on anomaly; `[SHOULD]` device fingerprint + impossible-travel detection | [04](../standards-kb/04-security.md) | Low |
| S2 | Spoofing | **Account enumeration** | Different response/timing/error for "no such user" vs "wrong password" or during signup/reset reveals valid accounts | `[MUST]` uniform generic responses and constant-ish timing across login, signup, and reset; identical "if an account exists, we sent a link" reset message | [04](../standards-kb/04-security.md) | Low |
| S3 | Spoofing | **MFA bypass** | Attacker skips the 2nd factor via a direct call to the post-MFA endpoint, brute-forces a 6-digit TOTP, or abuses a weak SMS/recovery fallback | `[MUST]` server-side enforce MFA state on the session (not a client flag); rate-limit + lock TOTP attempts; single-use, expiring codes; treat recovery codes as single-use; `[SHOULD]` phishing-resistant WebAuthn/passkeys where possible | [04](../standards-kb/04-security.md) | Med |
| T1 | Tampering | **Session fixation** | Attacker sets/knows a session id before login and reuses it after the victim authenticates | `[MUST]` issue a **new** session id on every privilege change (login, MFA pass, role change); never accept a client-supplied session id | [04](../standards-kb/04-security.md) | Low |
| T2 | Tampering | **JWT/token forgery** | Attacker forges or `alg:none`-downgrades a self-contained token | `[MUST]` verify signature with a pinned algorithm; reject `none`; short expiry; no sensitive data or authz decisions baked into an unverified client copy | [04](../standards-kb/04-security.md) | Low |
| R1 | Repudiation | **Denied login / takeover** | User (or attacker) denies a login, password change, or MFA reset happened | `[MUST]` immutable audit of login success/fail, MFA changes, password/email changes, new-device sign-in, logout — who/when/IP/device/correlation-id; `[SHOULD]` notify user on security-sensitive changes | [20](../standards-kb/20-logging-audit-and-traceability.md) | Low |
| I1 | Info disclosure | **Session hijack / token theft** | XSS steals a token from JS-readable storage; cookie sniffed over non-TLS; token leaked in URL/logs/referrer | `[MUST]` HttpOnly + Secure + SameSite cookies; TLS-only (HSTS); never place tokens in URLs; never log tokens; robust CSP to blunt XSS; bind session to a rotating server-side handle | [04](../standards-kb/04-security.md) | Med |
| I2 | Info disclosure | **Credential leak at rest** | DB dump exposes passwords/MFA seeds | `[MUST]` Argon2id/bcrypt/scrypt + per-user salt for passwords; encrypt MFA seeds with a KMS key; never store recovery answers in plaintext | [04](../standards-kb/04-security.md) / [06](../standards-kb/06-data-management.md) | Low |
| D1 | DoS | **Login flood / lockout weaponization** | Attacker exhausts auth backend, or triggers lockouts to deny legitimate users | `[MUST]` rate-limit per IP/account; use backoff + step-up rather than hard permanent lockout; `[SHOULD]` isolate auth capacity so a flood can't starve the rest of the app | [10](../standards-kb/10-api-integration.md) / [05](../standards-kb/05-reliability-resilience.md) | Med |
| E1 | Elevation | **Privilege carried by stale session** | Session keeps elevated rights after role downgrade/deactivation; refresh token not revoked | `[MUST]` short-lived access tokens + rotating refresh; revoke on logout, password change, role change, and deactivation; re-check authorization from source of truth, not the token snapshot | [04](../standards-kb/04-security.md) | Low |

## Top risks

1. **Session hijack via stolen token (I1)** — depends on XSS defenses holding; keep tokens out of JS-readable storage and enforce CSP.
2. **MFA bypass (S3)** — the most common way "we have MFA" still fails; enforce factor state server-side.
3. **Credential stuffing (S1)** — high-likelihood, scriptable, industry-wide; breached-password checks + rate limiting are non-negotiable.

## Open risks / decisions

| Item | Decision | Owner | Due |
|---|---|---|---|
| SMS fallback for MFA (SIM-swap risk) | Prefer TOTP/WebAuthn; SMS only as last resort | | |
| Remember-me lifetime vs. hijack window | Set max lifetime + reauth for sensitive actions | | |

---

## Mitigation checklist

- [ ] `[MUST]` Passwords hashed (Argon2id/bcrypt/scrypt) + breached-password check.
- [ ] `[MUST]` Per-IP/per-account rate limiting + backoff on login, MFA, and reset.
- [ ] `[MUST]` Uniform responses/timing — no account enumeration on login/signup/reset.
- [ ] `[MUST]` MFA state enforced server-side; TOTP rate-limited; recovery codes single-use.
- [ ] `[MUST]` New session id on every privilege change (no fixation).
- [ ] `[MUST]` HttpOnly + Secure + SameSite cookies; TLS/HSTS; tokens never in URLs or logs.
- [ ] `[MUST]` Short-lived access + rotating refresh; revoke on logout/password/role change/deactivation.
- [ ] `[MUST]` Immutable audit of all auth-sensitive events → [20](../standards-kb/20-logging-audit-and-traceability.md).
- [ ] `[SHOULD]` WebAuthn/passkeys; new-device and security-change notifications.
