# 10 · API & Integration

How the system exposes and consumes contracts — consistent, versioned, idempotent, tenant-scoped, and safe against the integration failure modes that break enterprise SaaS.

## API surface

- **[MUST]** A **consistent paradigm** (REST or GraphQL), **versioned** (support N / N-1 with a published deprecation window), consistent naming, **ISO-8601 UTC** timestamps, and decimal money. **Pagination on every list** (cursor/keyset) with validated, allow-listed filter/sort/field-selection.
- **[MUST]** One **standard error envelope** — stable error code, human message, field-level errors, and correlation id — with correct HTTP status codes and **no internal leakage** (stack traces, SQL, internal hostnames).

## Correctness & fairness

- **[MUST]** `Idempotency-Key` on retryable mutations; **optimistic concurrency** (ETag / version) on edits.
- **[MUST]** **Rate limits + quotas** per tenant/user/key (`429` + `X-RateLimit-*` headers).

## AuthN / AuthZ

- **[MUST]** **OAuth2 / OIDC** bearer tokens for user-facing APIs; **signed, scoped, revocable API keys** for machine integrations — both **tenant-scoped and least-privilege**, with **per-object authorization** (no IDOR/BOLA).

See [Security](04-security.md) for the underlying authorization model.

## Webhooks

- **[MUST]** Outbound **webhooks** are **HMAC-signed and timestamped**, retried with a **DLQ**, carry an **idempotent event id**, are tenant-configurable and replayable, and target only an **SSRF-safe destination allow-list**.

> **Example (fintech/e-commerce):** payment and order-status webhooks are signed and replayable so a merchant can rebuild state after an outage without duplicate processing.
> **Example (healthcare):** order and result events are delivered as signed, idempotent webhooks that downstream systems can replay safely.

## External providers

- **[MUST]** Wrap every external provider (messaging, email, payment, AI, or any external integration/device) behind a **swappable adapter**. All external calls have **timeouts, retries, and circuit breakers**; credentials live in a secrets manager; inbound provider webhooks are **verified and idempotent**.

## Contract as source of truth

- **[MUST]** The **OpenAPI / GraphQL schema is the source of truth**: interactive docs with examples, a sandbox, and a changelog — all **contract-tested**. Any additional machine-to-machine integration contracts (device or partner protocols) are likewise **versioned integration contracts** held to this same standard.

## Acceptance checklist

- Consistent, versioned API (N/N-1) with cursor pagination and allow-listed query params.
- Single error envelope + correct status codes; no internal leakage.
- Idempotency keys + optimistic concurrency on mutations/edits.
- Rate limits + quotas per tenant/user/key.
- OAuth/OIDC + scoped revocable keys + per-object authz (no IDOR).
- Signed, retryable, replayable, SSRF-safe webhooks with a DLQ.
- External providers behind adapters with timeouts/retries/breakers; verified inbound webhooks.
- OpenAPI/GraphQL source-of-truth with sandbox, changelog, and contract tests.
