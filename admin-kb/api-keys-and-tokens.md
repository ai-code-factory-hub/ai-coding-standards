# Admin · API Keys & Tokens

> Issue, rotate, and revoke scoped API keys and access tokens per tenant — with granular scopes, expiry, last-used tracking, a secret shown exactly once, and a usage + rate-limit view.

## What it is / when you need it

The credential surface for programmatic access. As soon as customers, internal services, or integrations call your API, you need first-class, revocable, least-privilege credentials — not shared passwords or long-lived god-tokens. This screen lets a tenant admin mint a key scoped to exactly what a client needs, see when it was last used, rotate it without downtime, and kill it instantly if leaked. It is the operational front-end to the auth story in [`../standards-kb/04-security.md`](../standards-kb/04-security.md) and the API contract in [`../standards-kb/10-api-integration.md`](../standards-kb/10-api-integration.md).

## Data model

Tenant-scoped; standard audit columns (`created_by`, `created_at`, `updated_at`). **Only a salted hash of the secret is stored** — the plaintext is shown once at creation and never again.

### ApiKey

| Field | Type | Notes |
| --- | --- | --- |
| `id` | uuid | PK |
| `tenant_id` | fk | Isolation boundary |
| `name` | string | Human label, e.g. `zapier-prod` |
| `prefix` | string | Non-secret display prefix, e.g. `mk_live_a1b2` — safe to show |
| `secret_hash` | string | Argon2/bcrypt hash of the full secret; plaintext never stored |
| `key_type` | enum | `secret` \| `publishable` \| `service` |
| `environment` | enum | `live` \| `test` |
| `status` | enum | `active` \| `rotating` \| `revoked` \| `expired` |
| `expires_at` | timestamp? | Null = no expiry (discouraged) |
| `last_used_at` | timestamp? | Updated async on use |
| `last_used_ip` | string? | Coarse source, for anomaly review |
| `created_by` | fk | Actor |
| `revoked_at` / `revoked_by` / `revoke_reason` | mixed | Populated on revoke |

### ApiKeyScope

| Field | Type | Notes |
| --- | --- | --- |
| `api_key_id` | fk | → ApiKey |
| `scope` | string | e.g. `invoices:read`, `webhooks:write`, `reports:export` |
| `resource_constraint_json` | jsonb? | Optional narrowing (e.g. single project id) |

Scopes are least-privilege and additive; the effective permission set is the intersection of the key's scopes and the owning principal's RBAC (see [`../feature-blueprints/rbac-admin.md`](../feature-blueprints/rbac-admin.md)) — a key can never exceed its creator's grants.

### AccessToken (short-lived / OAuth)

| Field | Type | Notes |
| --- | --- | --- |
| `id` | uuid | PK |
| `tenant_id` | fk | |
| `api_key_id` | fk? | Parent key if machine-to-machine |
| `token_hash` | string | Hash only |
| `grant_type` | enum | `client_credentials` \| `authorization_code` \| `refresh` |
| `scopes` | string[] | Subset of parent scopes |
| `issued_at` / `expires_at` | timestamp | Short TTL (minutes–hours) |
| `revoked` | bool | Supports token blacklist / logout-everywhere |

### RateLimitPolicy

| Field | Type | Notes |
| --- | --- | --- |
| `api_key_id` | fk | |
| `window` | enum | `second` \| `minute` \| `hour` \| `day` |
| `limit` | int | Max requests per window |
| `burst` | int? | Token-bucket burst allowance |
| `quota_period` | enum? | `month` \| `billing_cycle` for hard quotas |

### ApiUsageRecord (rollup)

| Field | Type | Notes |
| --- | --- | --- |
| `api_key_id` | fk | |
| `bucket_start` | timestamp | Hour/day bucket |
| `request_count` | bigint | |
| `error_count` | bigint | 4xx/5xx |
| `throttled_count` | bigint | 429s |
| `p95_latency_ms` | int? | |

## Key screens & UX

1. **Keys list** — table of keys with `name`, masked identifier (`prefix••••`), `key_type`/`environment` badge, scopes summary, `last_used_at` (relative), `status`, and expiry countdown. Filter by status/environment. *Loading:* row skeletons. *Empty:* "No API keys yet — create one to start calling the API." *Error:* retry banner.
2. **Create key** — name, environment, scope picker (grouped, search, least-privilege hints), optional expiry (with a recommended default), optional rate-limit policy. On submit → **one-time secret reveal modal**: full secret shown once with copy button and a "Store this now — you won't see it again" warning; a required "I've copied it" acknowledgment before dismissal. *Error:* scope-exceeds-your-permissions message.
3. **Key detail** — scopes, expiry, `last_used_at`/`last_used_ip`, revoke reason (if any), and a **usage panel**: requests/errors/throttled over time, current rate-limit consumption, and top endpoints. *Empty usage:* "No calls in this period."
4. **Rotate** — issues a new secret while keeping the same key identity/scopes; old secret enters a configurable **grace overlap** (`rotating` status) so clients cut over without downtime, then auto-expires. Shows the new secret via the same one-time reveal.
5. **Revoke** — immediate, irreversible; requires a reason; confirmation dialog names the key and warns that dependent integrations will start failing. *State:* key flips to `revoked`, calls rejected at the edge within seconds.
6. **Usage & rate-limits** — per-key and tenant-wide charts of volume, error rate, and 429s; surfaces keys nearing quota and keys unused for N days (rotation/cleanup candidates).

## Rules & logic

- [MUST] Secrets are **generated with a CSPRNG, stored only as a salted hash**, and the plaintext is **shown exactly once** at creation/rotation — never retrievable afterward. Only the non-secret `prefix` is ever displayed again.
- [MUST] Every key is **tenant-scoped**; a key issued in tenant A can never authenticate against tenant B's data.
- [MUST] Keys carry **explicit least-privilege scopes**; the effective permission is the intersection of key scopes and the creating principal's RBAC — a key **cannot exceed its creator's grants**.
- [MUST] Issue, rotate, and revoke are **RBAC-gated** (admin/owner) and **audited** with before→after (scopes, expiry, status) per [`../standards-kb/20-logging-audit-and-traceability.md`](../standards-kb/20-logging-audit-and-traceability.md). Revoke records actor + reason.
- [MUST] Revocation takes effect **immediately** at the auth edge (no cached grace for revoke); a revoked or expired key returns `401`, out-of-scope calls return `403`.
- [MUST] Enforce **rate limits per key** and return `429` with `Retry-After` and standard rate-limit headers per [`../standards-kb/10-api-integration.md`](../standards-kb/10-api-integration.md).
- [MUST] Secrets are **never logged**, echoed in errors, or included in audit payloads — only the `prefix`/`id`.
- [SHOULD] Rotation supports a **grace-overlap window** so both old and new secrets validate briefly for zero-downtime cutover, then the old auto-expires.
- [SHOULD] Default keys to a **finite expiry** and warn on keys with no expiry or unused for N days (cleanup nudges).
- [SHOULD] Track `last_used_at`/`last_used_ip` asynchronously and flag anomalous source IPs or sudden spikes for review.
- [SHOULD] Support **environment separation** (`live` vs `test`) so test keys can never touch production data.

## Applicable standards

- Credential storage, hashing, least-privilege, revocation, secrets handling — [`../standards-kb/04-security.md`](../standards-kb/04-security.md)
- API scopes, rate limiting, 401/403/429 semantics, headers — [`../standards-kb/10-api-integration.md`](../standards-kb/10-api-integration.md)
- Admin-action audit trail (issue/rotate/revoke) — [`../standards-kb/20-logging-audit-and-traceability.md`](../standards-kb/20-logging-audit-and-traceability.md)
- Tenant isolation for keys and usage — [`../standards-kb/01-architecture-multitenancy.md`](../standards-kb/01-architecture-multitenancy.md)
- Admin UX / loading-empty-error states — [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md)
- Usage cost / quota economics — [`../standards-kb/16-finops-cost.md`](../standards-kb/16-finops-cost.md)
- Related admin surfaces — [`../admin-kb/README.md`](../admin-kb/README.md), RBAC — [`../feature-blueprints/rbac-admin.md`](../feature-blueprints/rbac-admin.md)

## Acceptance checklist

- [ ] Secret generated with CSPRNG, stored as a salted hash, and shown exactly once with a copy-and-acknowledge modal.
- [ ] Only the non-secret `prefix` is ever displayed after creation; plaintext is unrecoverable.
- [ ] Keys are tenant-scoped and carry explicit least-privilege scopes that cannot exceed the creator's RBAC.
- [ ] Issue / rotate / revoke are RBAC-gated and audited with before→after; revoke captures actor + reason.
- [ ] Revocation and expiry take effect immediately at the auth edge (401/403 as appropriate).
- [ ] Per-key rate limits enforced with 429 + `Retry-After` and standard headers.
- [ ] Rotation supports a zero-downtime grace overlap that auto-expires the old secret.
- [ ] Usage view shows requests/errors/throttled, rate-limit consumption, and `last_used_at`/`last_used_ip`.
- [ ] Secrets never appear in logs, errors, or audit payloads.
- [ ] Loading / empty / error states designed for the list, detail, and usage panels.
