# Standards KB — Engineering Standards & NFRs (project-agnostic)

The reusable, cross-cutting engineering standard every project built with this kit must
satisfy. Written **generically** (applies to any enterprise SaaS) with **labelled examples**
across domains (e.g. healthcare, fintech, e-commerce) so each rule is concrete without being
tied to one product.

## How to read the rule language (RFC 2119)

- **[MUST] / [MUST NOT]** — hard requirement; no ship without it (or a logged, approved waiver).
- **[SHOULD] / [SHOULD NOT]** — strong default; deviation needs a written reason.
- **[MAY]** — optional; use judgement.

Each domain ends with an **Acceptance checklist**. Run the **[Master Release Gate](99-master-release-gate.md)** before every production release.

## Global engineering principles (apply everywhere)

1. **Multi-tenant by default** — every query, file, cache key, and log line is tenant-scoped.
2. **Secure by default** — deny by default; least privilege; encrypt in transit and at rest.
3. **Observable by default** — every request carries a correlation id; every error is reproducible.
4. **Configurable, not hardcoded** — tenant behaviour is config/data, never `if tenant == X`.
5. **Reversible & migratable** — every schema change and deploy can roll back.
6. **Async for anything slow** — >~1s or third-party work runs as a background job.
7. **Fail safe & degrade gracefully** — a failing dependency degrades one feature, not the app.
8. **Accessible & compliant** — WCAG 2.2 AA and applicable data-protection law are requirements.

## The domains (01–20 + release gate)

| # | Domain | Owns |
|---|---|---|
| 01 | [Architecture & Multi-Tenancy](01-architecture-multitenancy.md) | tenancy model, RLS, tenant registry, config layers |
| 02 | [Versioning, Deployment & Migration](02-versioning-deploy-migration.md) | SemVer, zero-downtime, expand→migrate→contract |
| 03 | [Performance & Scalability](03-performance-scalability.md) | budgets, pagination, async queues, indexing |
| 04 | [Security](04-security.md) | authN/authZ, IDOR, crypto, secrets, audit trail |
| 05 | [Reliability & Resilience](05-reliability-resilience.md) | SLOs, HA/DR, timeouts/retries/breakers, offline sync |
| 06 | [Data Management](06-data-management.md) | schema hygiene, backups, retention, residency, files |
| 07 | [Observability & AIOps](07-observability-aiops.md) | structured logs, RED/USE metrics, tracing, alerting |
| 08 | [Testing & QA](08-testing-qa.md) | test pyramid, contract tests, E2E, tenant-isolation tests |
| 09 | [Compliance & Privacy](09-compliance-privacy.md) | privacy-by-design, data-subject rights, consent, DPAs |
| 10 | [API & Integration](10-api-integration.md) | versioned APIs, error envelope, idempotency, webhooks |
| 11 | [Frontend, UX & Product](11-frontend-ux.md) | responsive/a11y baseline, dashboards, engines, workflows |
| 12 | [DevOps & CI/CD](12-devops-cicd.md) | gated pipeline, IaC, progressive delivery, feature flags |
| 13 | [Internationalization & Accessibility](13-i18n-accessibility.md) | no hardcoded strings, UTC, WCAG 2.2 AA |
| 14 | [Developer Experience & Docs](14-devex-documentation.md) | linting, ADRs, runbooks, one-command setup |
| 15 | [AI / LLM Governance & Safety](15-ai-llm-governance.md) | untrusted I/O, grounding, human-in-loop, cost caps |
| 16 | [FinOps & Cloud Cost](16-finops-cost.md) | tagging, cost-per-tenant, budgets, cost guardrails |
| 17 | [NFR Coverage & Gaps](17-nfr-coverage-gaps.md) | portability, supportability, capacity, usability targets |
| 18 | [Theming & Branding](18-theming-branding.md) | token engine, theme catalog, white-label |
| 19 | [NFR Implementation Playbook](19-nfr-implementation-playbook.md) | Build/Automate/Done-when per NFR, CI wiring |
| 20 | [Logging, Audit & Traceability](20-logging-audit-and-traceability.md) | every log class, retention, PII masking, tamper-evidence, and its admin surface |
| 99 | [Master Release Gate](99-master-release-gate.md) | the Go/No-Go checklist |

Design tokens and the theme catalog referenced by Domain 18 live in
[../design-system-kb/](../design-system-kb/README.md).

> **Source:** generalized from a 20-domain production-SaaS engineering handbook (RFC 2119).
> Reuse across projects; extend, don't rewrite.
