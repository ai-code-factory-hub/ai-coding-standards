# 07 · Developer Portal & SDKs

**Purpose:** the self-service front door that turns a published [OpenAPI spec](02-openapi-spec-first.md)
into a developer who is making real calls in minutes — interactive docs, a sandbox, self-service
key issuance, generated SDKs, guided onboarding, and versioned docs. This is the human-facing end
of the [10 · API & Integration](../standards-kb/10-api-integration.md) "docs + sandbox +
changelog" requirement and the delivery surface for [Admin · API Keys & Tokens](../admin-kb/api-keys-and-tokens.md).

## Developer portal

- **[MUST]** Publish **interactive API docs generated from the OpenAPI spec** (Redoc/Swagger-UI/
  Stoplight-style) — every operation, schema, error `code`, and example rendered from the single
  source of truth, so docs can never drift from the contract ([02](02-openapi-spec-first.md)).
- **[MUST]** Provide a **sandbox** — a "try it" console backed by the mock server or a seeded test
  tenant ([02](02-openapi-spec-first.md)) — so a developer can execute a real, safe call from the
  docs page with a test key. Sandbox data is synthetic; no production PII.
- **[MUST]** Offer **self-service API-key issuance**: a developer signs up, creates a scoped **test**
  key from the portal, and later a **live** key, following the full lifecycle (CSPRNG, hash-stored,
  shown once, scoped, revocable) in [Admin · API Keys & Tokens](../admin-kb/api-keys-and-tokens.md).
- **[MUST]** Surface the **changelog** ([05](05-versioning-and-lifecycle.md)), the **deprecation
  schedule**, the **rate-limit/quota tiers** ([04](04-api-gateway-and-rate-limiting.md)), and the
  **auth/scopes guide** ([03](03-api-security.md)) — everything a consumer needs to integrate and
  stay current.
- **[SHOULD]** Show the developer their **own usage** (calls, error rate, quota consumed) pulled
  from the analytics pipeline ([06](06-api-access-logs-and-analytics.md)) so they can self-diagnose
  before opening a ticket.

## SDK generation from OpenAPI

- **[MUST]** **Generate SDKs from the OpenAPI spec** (openapi-generator / Speakeasy / Fern) for the
  supported languages, versioned in lockstep with the API — so the SDK is always faithful to the
  contract and regenerated on every spec change in CI ([12 · DevOps & CI/CD](../standards-kb/12-devops-cicd.md)).
- **[SHOULD]** SDKs ship with **built-in correctness helpers**: auth handling, automatic retries
  with backoff on `429`/`5xx` honoring `Retry-After`, idempotency-key support, and typed models —
  so consumers get the [01](01-api-design-guide.md) semantics for free.
- **[SHOULD]** Publish SDKs to the standard package registries (npm, PyPI, Maven, etc.) with
  semver aligned to the API version, plus quickstart snippets in the portal per language.

> **Example (labelled — generate + publish SDKs in CI on spec change):**
> ```bash
> # runs when openapi.yaml changes; fails the build on breaking-change without a version bump
> openapi-generator-cli generate -i openapi.yaml -g typescript-axios -o sdk/ts
> openapi-generator-cli generate -i openapi.yaml -g python        -o sdk/python
> # ... contract tests pass -> publish
> npm publish ./sdk/ts   &&   twine upload ./sdk/python/dist/*
> ```

## Onboarding

- **[MUST]** Provide a **quickstart** that gets a developer from signup → test key → first
  successful call in a few minutes, with copy-paste snippets in each SDK language and cURL.
- **[SHOULD]** Include **guides and recipes** for the common flows (auth, pagination, webhooks —
  see [Integrations & Webhooks](../feature-blueprints/integrations-and-webhooks.md), error
  handling, going from test to live) beyond the raw reference.
- **[SHOULD]** Gate **live** access behind whatever verification the product requires (email,
  billing, plan selection) while keeping **test/sandbox** access frictionless.

## Versioned docs

- **[MUST]** Host **docs per API major version** ([05](05-versioning-and-lifecycle.md)) — a
  consumer on `v1` sees `v1` docs, examples, and SDKs; `v2` docs live alongside, with a version
  switcher — so deprecating a version deprecates its docs on the same timeline.
- **[SHOULD]** Clearly badge **deprecated** operations/versions in the docs with their **sunset
  date** and a link to the migration guide, mirroring the response headers from
  [05](05-versioning-and-lifecycle.md).

## Acceptance checklist

- [ ] Interactive docs are generated from the OpenAPI spec and cannot drift from the contract.
- [ ] A sandbox lets developers make real, safe calls with synthetic data and a test key.
- [ ] Self-service issuance of scoped test and live keys follows the one-time-secret lifecycle.
- [ ] The portal surfaces changelog, deprecation schedule, rate-limit tiers, auth/scopes, and the developer's own usage.
- [ ] SDKs are generated from the spec, versioned in lockstep, regenerated in CI, and published to package registries.
- [ ] SDKs include auth, `429`/`5xx` retry-with-backoff, idempotency, and typed models.
- [ ] A quickstart takes a developer from signup to first successful call in minutes; test access is frictionless, live access is gated.
- [ ] Docs are versioned per major API version with a switcher; deprecated operations are badged with sunset dates and migration links.
