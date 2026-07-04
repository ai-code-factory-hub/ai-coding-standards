# 04 · Security

Authentication, deny-by-default per-object authorization, session/token/input/crypto/secret hygiene, and an immutable audit trail.

## Authentication (AuthN)

- **[MUST]** Passwords hashed with Argon2id/bcrypt/scrypt (+ per-user salt); enforce a password policy plus a breached-password check; lockout/rate-limit on failed logins. **MFA/2FA** (TOTP minimum; SMS fallback), which a tenant can mandate. Enterprise **SSO: OAuth2/OIDC + SAML 2.0**, with **SCIM** for provisioning. Recovery tokens are secure, single-use, and time-limited.

## Authorization (AuthZ)

- **[MUST]** **Deny by default**; every endpoint **and every object** is authorized server-side (defeat IDOR/BOLA — the #1 API risk): verify the object belongs to the caller's tenant and that the caller holds the right permission on it. **RBAC** baseline plus **ABAC** where needed. Tenant isolation is an authorization invariant. **[SHOULD]** centralize in a policy layer (OPA-style), not scattered `if`s.

> **Example (healthcare):** ABAC enforces "a field worker sees only records in their assigned zone" and "only an authorized reviewer can sign off a domain record" — checked server-side on every object.
>
> **Example (fintech/e-commerce):** ABAC enforces "a manager may approve a refund ≤ $X" and "a merchant may read only orders under their own tenant" — the same object-level check blocks an IDOR attempt to fetch another merchant's order by id.

## Sessions, tokens, input, crypto, secrets

- **[MUST]** Short-lived access tokens + rotating refresh tokens, revoked on logout/password/role change; secure cookies (HttpOnly/Secure/SameSite); CSRF protection; no secrets or sensitive personal data in a JWT.
- **[MUST]** Validate all input server-side (allow-list); parameterized queries (no SQLi); context-aware output encoding (no XSS); prevent SSRF/command/template injection; file uploads validated + malware-scanned + stored outside the webroot + served via signed URLs. Set security headers (CSP/HSTS/etc.).
- **[MUST]** **TLS 1.2+ everywhere**; **encryption at rest** (DB, backups, files, logs); **field-level encryption for sensitive personal/identity data** with KMS-managed keys. No home-grown crypto.
- **[MUST]** No secrets in code/config/logs — use a secrets manager; secret scanning in CI + pre-commit; distinct secrets per environment; rotate on exposure.

## Audit trail

- **[MUST]** Keep an immutable, append-only, tamper-evident audit log of sensitive events: logins (success/fail), permission/role changes, **record amendments, approval sign-offs, acknowledgements of critical events, consent changes, data exports, config changes, high-value approvals, and deletions**. Each entry records: who (user+tenant), what, when (UTC), where (IP/device), before→after, and a correlation id.
- **[MUST NOT]** Log secrets or sensitive personal data in plaintext — mask/tokenize.

## Platform security hygiene

- **[MUST]** Dependencies patched; SCA on every build (fail on critical CVEs); least privilege for DB/service/cloud/container users; network segmentation + WAF; generic user-facing errors (no stack traces). **[SHOULD]** periodic pen-tests and a documented incident-response plan.

See [Architecture & Multi-Tenancy](01-architecture-multitenancy.md) for the tenant-isolation invariant, [Reliability & Resilience](05-reliability-resilience.md) for rate limiting, and [Data Management](06-data-management.md) for at-rest encryption of files and backups.

## Acceptance checklist

- [ ] Passwords strongly hashed; MFA available and tenant-mandatable; SSO (OIDC/SAML) + SCIM for enterprise.
- [ ] Deny-by-default authorization on every endpoint **and object**; tenant isolation enforced; no IDOR/BOLA.
- [ ] Short-lived tokens + rotating refresh + secure cookies + CSRF protection; no sensitive data in JWTs.
- [ ] Server-side input validation; injection defenses; upload scanning; security headers set.
- [ ] TLS everywhere; encryption at rest; field-level encryption of sensitive data via KMS.
- [ ] Secrets in a manager, scanned in CI/pre-commit, rotated on exposure.
- [ ] Immutable, masked audit trail on all sensitive events; SCA green; WAF + IR plan in place.
