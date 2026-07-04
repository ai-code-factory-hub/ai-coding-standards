# Threat Model — API & Webhooks

> Filled STRIDE model for public/partner REST/GraphQL APIs and inbound/outbound webhooks. Method & risk rating: [README.md](README.md). Blank copy: [threat-model-template.md](threat-model-template.md).
> **Owner:** Security Officer · **Extends:** [04 Security](../standards-kb/04-security.md), [10 API & Integration](../standards-kb/10-api-integration.md)

## Feature / asset

Programmatic endpoints (mobile/partner/public API) and webhook plumbing (inbound callbacks you receive, outbound events you send). **Asset:** every object reachable through the API, and the trust relationship with integration partners. APIs have no UI to hide behind — the contract *is* the attack surface.

## Data classification

Whatever the endpoints expose — assume up to **Confidential/PII/Regulated**.

## Actors

| Actor | Trusted? | Notes |
|---|---|---|
| Authenticated API client | Semi-trusted | scoped by key/token + tenant |
| Anonymous caller | **Untrusted** | probes for broken auth |
| **Inbound webhook sender** | **Untrusted until verified** | anyone can POST to your callback URL |
| Outbound webhook target / URL the API is asked to fetch | **Untrusted** | SSRF pivot |

## Trust boundaries

- **B1** network edge. **B2** authn (key/OAuth). **B3** tenant. **B5** external (partner APIs, webhook peers).

## Attack surface

Every route + method, all parameters and JSON bodies (incl. unexpected fields), object identifiers in the path/query, auth headers, pagination/filter params, the webhook receiver, and any user-supplied URL the server dereferences.

## STRIDE analysis

| # | STRIDE | Threat | How (attack path) | Mitigation `[MUST]`/`[SHOULD]` | Standards ref | Residual |
|---|---|---|---|---|---|---|
| S1 | Spoofing | **Webhook spoofing** | Attacker POSTs forged events to your public callback URL (fake payment, fake state change) | `[MUST]` verify an **HMAC signature** over the raw body with a shared secret (constant-time compare); reject unsigned/invalid; pin sender source where possible; per-provider secrets | [10](../standards-kb/10-api-integration.md) / [04](../standards-kb/04-security.md) | Low |
| S2 | Spoofing | **Broken authentication** | Missing/weak auth on a route; accepting expired/`none`-alg tokens; API key in URL/logs | `[MUST]` authenticate every non-public route; validate token signature+expiry+audience; keys in headers only, never URLs; rotate/revoke keys; scope keys to least privilege | [04](../standards-kb/04-security.md) / [10](../standards-kb/10-api-integration.md) | Low |
| T1 | Tampering | **Mass assignment** | Client sends extra fields (`role`, `tenant_id`, `is_admin`, `balance`) that bind to the model | `[MUST]` bind to an explicit allow-list DTO; never auto-map request body to a persistence model; ignore/reject unknown fields | [04](../standards-kb/04-security.md) / [10](../standards-kb/10-api-integration.md) | Low |
| T2 | Tampering | **Payload/replay tampering** | Webhook or request captured and replayed, or body altered in transit | `[MUST]` TLS everywhere; include + verify a timestamp and nonce/idempotency key; reject stale (>N min) or already-seen deliveries | [10](../standards-kb/10-api-integration.md) | Low |
| R1 | Repudiation | **Denied API action** | Client denies making a state-changing call | `[MUST]` audit state-changing calls with client id, tenant, params digest, correlation id, timestamp; log webhook delivery ids | [20](../standards-kb/20-logging-audit-and-traceability.md) | Low |
| I1 | Info disclosure | **IDOR / BOLA** | Swap an object id in the path/body to read/modify another user's or tenant's object — the #1 API risk | `[MUST]` authorize **every object**, not just the route: verify the object belongs to the caller's tenant *and* the caller holds the permission on it; use non-guessable ids; deny by default | [04](../standards-kb/04-security.md) | **Med** |
| I2 | Info disclosure | **Excessive data exposure** | Endpoint returns full objects (internal fields, other users' PII) and the client filters | `[MUST]` shape responses with explicit output DTOs; never serialize raw entities; field-level authz for sensitive attributes | [04](../standards-kb/04-security.md) / [06](../standards-kb/06-data-management.md) | Low |
| I3 | Info disclosure | **SSRF to outbound targets** | API accepts a URL (webhook registration, import-from-URL, avatar-by-URL) and the server fetches it, hitting cloud metadata / internal services | `[MUST]` allow-list schemes/hosts; resolve DNS and block private/link-local/loopback ranges; disable redirects; no metadata-endpoint access; egress-proxy with deny-by-default | [04](../standards-kb/04-security.md) / [10](../standards-kb/10-api-integration.md) | Med |
| D1 | DoS | **Rate-limit abuse / scraping** | Brute force, credential stuffing, enumeration, or bulk scraping via the API; unbounded pagination/GraphQL query depth | `[MUST]` per-key/IP/tenant rate + quota limits; cap page size and GraphQL depth/complexity; pagination not "return all"; `[SHOULD]` anomaly detection | [10](../standards-kb/10-api-integration.md) / [05](../standards-kb/05-reliability-resilience.md) | Med |
| D2 | DoS | **Webhook amplification / retry storm** | Attacker triggers events, or a failing endpoint causes unbounded retries | `[MUST]` bounded exponential-backoff retries with a dead-letter queue; idempotent receivers; cap concurrent outbound calls | [10](../standards-kb/10-api-integration.md) | Low |
| E1 | Elevation | **Function-level authz bypass (BFLA)** | Client calls an admin/privileged operation not exposed in their UI | `[MUST]` enforce role/permission per operation server-side; deny by default; don't rely on the client hiding the action | [04](../standards-kb/04-security.md) | Med |

## Top risks

1. **IDOR/BOLA (I1)** — the most exploited API flaw; every object access must be authorized, not just the route.
2. **SSRF via user-supplied URLs (I3)** — webhook URLs and "fetch from URL" features are a direct path to internal infra.
3. **Webhook spoofing (S1)** — an unauthenticated callback that mutates state (e.g., "payment succeeded") is a full compromise; HMAC-verify it.

## Open risks / decisions

| Item | Decision | Owner | Due |
|---|---|---|---|
| Outbound egress control | Route all server-side fetches through a filtering proxy | | |
| GraphQL exposure | Depth/complexity limits + persisted queries | | |

---

## Mitigation checklist

- [ ] `[MUST]` Every object access authorized by owner+tenant+permission (no IDOR/BOLA).
- [ ] `[MUST]` Every route authenticated; tokens/keys validated; keys in headers, scoped, rotatable.
- [ ] `[MUST]` Request binding via explicit DTO allow-list (no mass assignment); responses via output DTOs (no over-exposure).
- [ ] `[MUST]` Inbound webhooks HMAC-verified (constant-time) + timestamp/nonce anti-replay.
- [ ] `[MUST]` User-supplied URLs: scheme/host allow-list, private-range block, no redirects (no SSRF).
- [ ] `[MUST]` Per-key/IP/tenant rate limits + quotas; bounded page size & query depth.
- [ ] `[MUST]` Outbound retries bounded with backoff + dead-letter; receivers idempotent.
- [ ] `[MUST]` Per-operation function-level authz (no BFLA); deny by default.
- [ ] `[MUST]` State-changing calls + webhook deliveries audited → [20](../standards-kb/20-logging-audit-and-traceability.md).
