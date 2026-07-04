# API KB — API Design, Integration, Security & Access Logging (project-agnostic)

The deep, buildable layer that lets an application be **safely exposed and connected via an
OpenAPI contract**: how you design resources, ship a spec-first API, secure it, put it behind
a gateway with rate limits, version and deprecate it responsibly, log and meter every call,
and onboard developers through a portal with generated SDKs.

Written **generically** (applies to any enterprise SaaS) with **labelled examples** across
domains (healthcare, fintech, e-commerce) so each practice is concrete without being tied to
one product.

## How this extends the Standards KB

This KB **extends [10 · API & Integration](../standards-kb/10-api-integration.md)** — it does
not restate it. Domain 10 sets the non-negotiable API surface (consistent, versioned,
idempotent, tenant-scoped, single error envelope, signed webhooks, contract-as-source-of-truth).
This KB is the **how**: the design rules, the OpenAPI workflow, the security mechanisms, the
gateway config, the deprecation process, and the access-log/analytics pipeline you build to
satisfy that standard.

| Standards KB says (the requirement) | API KB says (how you build it) |
|---|---|
| [10 · API & Integration](../standards-kb/10-api-integration.md) — consistent, versioned, single error envelope, rate limits, OAuth/keys, OpenAPI source of truth | Every doc here — resource modeling, spec-first pipeline, OAuth flows, gateway policy, deprecation process, access-log schema |
| [04 · Security](../standards-kb/04-security.md) — authZ model, secrets, least privilege | [03 · API Security](03-api-security.md) — OAuth2/OIDC flows, scopes, per-object authz, mTLS |
| [20 · Logging, Audit & Traceability](../standards-kb/20-logging-audit-and-traceability.md) — structured logs, correlation, PII masking | [06 · API Access Logs & Analytics](06-api-access-logs-and-analytics.md) — log every call, meter usage, detect anomalies |
| [16 · FinOps & Cost](../standards-kb/16-finops-cost.md) — usage economics, quota | Metering + quota consumption feeding cost attribution |

Rule of thumb: if it is a property the API must **have**, Domain 10 owns it. If it is the
**design decision, mechanism, or pipeline** that delivers that property, it lives here.

## The "spec-first" philosophy

The **OpenAPI document is the single source of truth**, authored and reviewed **before** any
server code exists. Server stubs, client SDKs, mock servers, request/response validation,
contract tests, and the developer portal are all **generated from the same spec**. Code never
drifts from docs because docs *are* the code's contract. This is the spine that ties every
doc in this KB together — design ([01](01-api-design-guide.md)) produces the spec
([02](02-openapi-spec-first.md)), which the gateway ([04](04-api-gateway-and-rate-limiting.md)),
lifecycle process ([05](05-versioning-and-lifecycle.md)), and portal ([07](07-developer-portal-and-sdks.md))
all consume.

## How to read the rule language (RFC 2119)

- **[MUST] / [MUST NOT]** — hard requirement; the API is not safe to expose without it.
- **[SHOULD] / [SHOULD NOT]** — strong default; deviation needs a written reason.
- **[MAY]** — optional; use judgement.

## Index

| # | Doc | Owns |
|---|---|---|
| 01 | [API Design Guide](01-api-design-guide.md) | resource modeling, naming, methods/status, error envelope, pagination, filtering, dates, money |
| 02 | [OpenAPI Spec-First](02-openapi-spec-first.md) | spec as source of truth, design→generate→mock→contract-test pipeline, examples, changelog |
| 03 | [API Security](03-api-security.md) | OAuth2/OIDC flows, scopes, scoped/revocable keys, JWT, mTLS, per-object authz, CORS, headers, secrets |
| 04 | [API Gateway & Rate Limiting](04-api-gateway-and-rate-limiting.md) | gateway duties, per-tenant/consumer limits + quotas, spike arrest, tiered plans, caching, WAF |
| 05 | [Versioning & Lifecycle](05-versioning-and-lifecycle.md) | versioning in the contract, N/N-1, deprecation + sunset headers, breaking-change rules, migration comms |
| 06 | [API Access Logs & Analytics](06-api-access-logs-and-analytics.md) | log every call, usage metering + frequency, latency percentiles, anomaly detection, alerting |
| 07 | [Developer Portal & SDKs](07-developer-portal-and-sdks.md) | interactive docs, sandbox, self-service keys, SDK generation, onboarding, versioned docs |

## Related domains & surfaces

- [12 · DevOps & CI/CD](../standards-kb/12-devops-cicd.md) — spec linting, contract tests, and SDK publish belong in the pipeline.
- [17 · NFR Coverage Gaps](../standards-kb/17-nfr-coverage-gaps.md) — API NFRs (latency SLOs, availability) that this KB must satisfy.
- [Integrations & Webhooks](../feature-blueprints/integrations-and-webhooks.md) — the outbound/event side that complements this inbound API surface.
- [Admin · API Keys & Tokens](../admin-kb/api-keys-and-tokens.md) — the credential UI; [Admin · API Usage & Analytics](../admin-kb/api-usage-and-analytics.md) — the metering dashboard this KB feeds.

> **Source:** generalized API-platform practice (REST/OpenAPI, OAuth2/OIDC, API-gateway patterns).
> Reuse across projects; extend, don't rewrite.
