# 03 · API Security

**Purpose:** the concrete authentication, authorization, and hardening mechanisms that make an
exposed API safe. This extends [10 · API & Integration](../standards-kb/10-api-integration.md)
("OAuth2/OIDC + scoped revocable keys + per-object authz") and the authorization model in
[04 · Security](../standards-kb/04-security.md). The operational credential UI lives in
[Admin · API Keys & Tokens](../admin-kb/api-keys-and-tokens.md).

## OAuth2 / OIDC flows

- **[MUST]** Use **OAuth2 / OIDC** for identity; pick the flow by client type:

  | Client | Flow | Notes |
  |---|---|---|
  | Web / mobile / SPA app acting **for a user** | **Authorization Code + PKCE** | No client secret in the browser/device; PKCE defeats code interception |
  | **Machine-to-machine** (backend, cron, partner service) | **Client Credentials** | App is the principal; no user; scoped to a service identity |
  | Refresh | **Refresh token** (rotating, short-lived) | Rotate on use; detect reuse and revoke the family |

- **[MUST NOT]** Use the **Implicit** or **Resource-Owner-Password** grants — both are deprecated
  and leak tokens/credentials. New integrations use Auth-Code+PKCE or Client-Credentials only.
- **[MUST]** Access tokens are **short-lived** (minutes) and **scoped**; long-lived access lives in
  refresh tokens or scoped API keys, never in a long-TTL access token.
- **[SHOULD]** Validate `iss`, `aud`, `exp`, `nbf`, and required scopes on every request at the
  edge; reject tokens whose `aud` is not this API.

## Scopes & least privilege

- **[MUST]** Every token/key carries **explicit, least-privilege scopes** (`invoices:read`,
  `orders:write`). The effective permission is the **intersection** of the credential's scopes and
  the principal's RBAC — a credential can never exceed its owner's grants.
- **[MUST]** Enforce scopes **server-side** on every operation (the gateway may pre-check, but the
  service is authoritative); a missing scope returns `403`, not `404`-by-accident.
- **[SHOULD]** Scopes map 1:1 to documented capabilities in the [OpenAPI spec](02-openapi-spec-first.md)
  (`security` + `scopes`) so the portal shows exactly what each scope grants.

## Scoped, revocable API keys

- **[MUST]** For non-OAuth machine access, issue **scoped, revocable API keys**: generated with a
  CSPRNG, **stored only as a salted hash**, plaintext shown **exactly once**, tenant-scoped, with
  an optional expiry. Full lifecycle rules live in
  [Admin · API Keys & Tokens](../admin-kb/api-keys-and-tokens.md).
- **[MUST]** **Revocation is immediate** at the auth edge (no cached grace for revoke); revoked or
  expired keys return `401`.
- **[SHOULD]** Support **rotation with a grace overlap** so clients cut over without downtime.

## JWT handling

- **[MUST]** Verify the **signature against a pinned issuer key set** (JWKS), the **algorithm
  against an allow-list** (reject `alg: none`; don't accept `HS*` where you expect `RS*`/`ES*`),
  and `exp`/`nbf`/`aud`/`iss`. Never trust an unverified JWT.
- **[MUST NOT]** Put secrets or sensitive PII in JWT claims — a JWT is signed, **not encrypted**;
  anyone holding it can read the payload.
- **[SHOULD]** Support **revocation** for stateless JWTs via short TTL + a revocation/blacklist
  check (or reference tokens) — a signed token is otherwise valid until `exp`.
- **[SHOULD]** Keep tokens out of URLs/query strings (they land in logs); send via the
  `Authorization: Bearer` header only.

## mTLS for partners

- **[SHOULD]** For **partner / B2B server-to-server** integrations, require **mutual TLS** — the
  client presents a certificate the gateway validates against a pinned CA/cert allow-list — in
  addition to (not instead of) OAuth client-credentials. This binds the transport to a known
  partner identity.
- **[MAY]** Use **certificate-bound access tokens** (RFC 8705) so a stolen token is unusable
  without the matching client certificate.

## Per-object authorization (defeat IDOR / BOLA)

- **[MUST]** **Authorize every object, on every request.** Having a valid token is *authentication*;
  the service **[MUST]** additionally check the caller is allowed **this specific resource** —
  `GET /invoices/{id}` verifies the invoice belongs to the caller's tenant/principal before
  returning it. This is the **#1 API vulnerability (BOLA/IDOR)**.
- **[MUST]** Use **opaque, non-guessable ids** ([01](01-api-design-guide.md)) so enumeration is
  useless, and **scope every query by tenant** at the data layer (not just the URL) so a
  mismatched id returns `404`/`403`, never another tenant's data.
- **[MUST NOT]** Rely on the client to send the tenant id — derive it from the authenticated token.

## Input validation

- **[MUST]** **Validate every input against the [OpenAPI schema](02-openapi-spec-first.md)** at
  the edge — types, formats, ranges, enums, max lengths, array bounds. Reject unknown fields
  (`additionalProperties: false`) rather than silently ignoring them.
- **[MUST]** Treat all input as hostile: parameterized queries (no string-built SQL), output
  encoding, and **payload size limits** to blunt injection and resource-exhaustion attacks.
- **[SHOULD]** Enforce a **request body size cap** and **max array/collection sizes** at the
  gateway before the request reaches application code.

## CORS

- **[MUST]** Set CORS to an **explicit origin allow-list** — never `Access-Control-Allow-Origin: *`
  on authenticated endpoints, and never reflect the `Origin` header unchecked.
- **[MUST]** Only enable `Access-Control-Allow-Credentials: true` with a specific origin, never a
  wildcard.

## Security headers

- **[MUST]** Serve **HSTS** (`Strict-Transport-Security`), `X-Content-Type-Options: nosniff`, and a
  restrictive `Cache-Control` on authenticated responses so tokens/PII aren't cached by
  intermediaries.
- **[SHOULD]** For any browser-rendered API surface (docs/portal), set `Content-Security-Policy`
  and `Referrer-Policy`; APIs should respond `application/json` and refuse content sniffing.

## Secrets in a manager

- **[MUST]** All signing keys, client secrets, DB credentials, and partner certs live in a
  **secrets manager** (Vault, cloud KMS/Secrets Manager) — never in code, env files in the repo,
  or logs. Rotate on a schedule and on suspected compromise. See [04 · Security](../standards-kb/04-security.md).
- **[MUST]** Secrets **[MUST NOT]** appear in access logs, error envelopes, or traces (masking is
  enforced in [06 · Access Logs](06-api-access-logs-and-analytics.md)).

> **Example (labelled — enforcing auth-code+PKCE for an app, client-credentials for a service):**
> A mobile app uses **Authorization Code + PKCE** to act for a logged-in user with scope
> `orders:read orders:write`; a nightly reconciliation job uses **Client Credentials** as a
> service principal with scope `orders:read` only. Both hit `GET /orders/{id}`, and both are
> **per-object authorized**: the service confirms the order's `tenant_id` matches the token's
> tenant before responding — a guessed id from another tenant returns `404`.

> **Example (labelled — rejecting a forged JWT):** A token arrives with `alg: none` and a valid
> `sub`. The verifier rejects it because `none` is not on the algorithm allow-list and the
> signature does not verify against the pinned JWKS — returned as `401` with the standard error
> envelope, logged with the correlation id but **without** the token.

## Acceptance checklist

- [ ] Auth-Code+PKCE for user-facing apps; Client-Credentials for M2M; Implicit/ROPC disabled.
- [ ] Access tokens short-lived and scoped; refresh tokens rotate with reuse detection.
- [ ] Every credential carries least-privilege scopes; effective perms = scopes ∩ RBAC; enforced server-side (403 on miss).
- [ ] API keys are CSPRNG-generated, hash-stored, shown once, tenant-scoped, and instantly revocable.
- [ ] JWTs verified against pinned JWKS with an algorithm allow-list; `alg:none`/wrong-`aud` rejected; no secrets/PII in claims.
- [ ] Partner integrations use mTLS against a pinned CA in addition to OAuth.
- [ ] Per-object authorization on every request; queries tenant-scoped at the data layer; opaque ids (no BOLA/IDOR).
- [ ] All input validated against the OpenAPI schema; unknown fields rejected; payload/array size caps enforced.
- [ ] CORS uses an explicit origin allow-list (no wildcard with credentials); security headers (HSTS, nosniff) set.
- [ ] All secrets in a secrets manager, rotated, and never present in logs/errors/traces.
