# 99 · Master Release Gate (Go / No-Go)

Run this per-domain checklist before every production release. Treat any unchecked **[MUST]** item as a release blocker unless there is a logged, approved waiver.

## Per-domain gate

- **Architecture & Tenancy ([01](01-architecture-multitenancy.md)):** new tenant-scoped tables/queries carry `tenant_id` + row-level security; no cross-tenant path; behaviour is config/entitlement-driven, not a code branch; plugins/extensions declare permissions + versioned contracts.
- **Versioning / Deploy / Migration ([02](02-versioning-deploy-migration.md)):** SemVer bumped; compatibility matrix valid (N / N-1); expand→migrate→contract, online-safe, reversible; backfills batched/idempotent; pre-migration backup taken with a known restore time; zero-downtime + rollback tested.
- **Performance ([03](03-performance-scalability.md)):** budgets held in CI; new lists paginated + indexed; slow/third-party work async; no N+1.
- **Security ([04](04-security.md)):** authorization on every new endpoint and object (no IDOR/BOLA); input validated; no secrets in code/logs; sensitive data encrypted; audit entries for new sensitive actions; SCA green.
- **Reliability ([05](05-reliability-resilience.md)):** new external calls have timeout + retry + circuit breaker + idempotency; SLOs unaffected; offline/sync paths idempotent.
- **Data ([06](06-data-management.md)):** new tables have PK + `tenant_id` + audit columns + DB constraints; money stored as decimal/minor units; timestamps UTC; soft delete; data residency respected; files in object storage.
- **Observability ([07](07-observability-aiops.md)):** structured logs + correlation id + new business metrics; alerts for new critical paths; no sensitive data in logs or sent to AI providers.
- **Testing ([08](08-testing-qa.md)):** unit (business-logic boundaries) + integration + API/contract + E2E + tenant-isolation tests; load-tested if on a hot path; no production data in non-prod.
- **Compliance ([09](09-compliance-privacy.md)):** data-subject-rights / consent / retention unaffected or updated; any new sub-processor has a signed DPA; policy version current.
- **API ([10](10-api-integration.md)):** consistent error envelope + correct status codes; pagination; idempotency; rate limits; webhooks signed + retryable; API schema updated + contract-tested.
- **Frontend / UX ([11](11-frontend-ux.md)):** responsive + accessible + internationalized + themed; loading/empty/error states present; async/bulk operations show progress; exports audited.
- **DevOps ([12](12-devops-cicd.md)):** gated pipeline green + peer-reviewed; IaC changes reviewed; change feature-flagged; SBOM + artifact signing; post-deploy smoke tests pass.
- **i18n / a11y ([13](13-i18n-accessibility.md)):** no hardcoded strings; UTC + locale formatting; WCAG 2.2 AA; status never conveyed by color alone.
- **DevEx ([14](14-devex-documentation.md)):** ADR recorded for any significant decision; docs/runbooks updated.
- **AI / LLM ([15](15-ai-llm-governance.md)):** untrusted I/O handling + prompt-injection defense; schema-validated, grounded output; propose-only with human sign-off; sensitive data scrubbed; eval set run.
- **FinOps ([16](16-finops-cost.md)):** resources tagged; no unbounded-cost path; cost guardrails in place.
- **NFR gaps ([17](17-nfr-coverage-gaps.md)):** stable error codes + correlation id; complexity/duplication/license gates green; capacity and support/compatibility matrix updated if changed.
- **Theming ([18](18-theming-branding.md)):** token-driven (no raw hex); light + dark tested; contrast meets AA; white-label intact across app, email, print, and PDF.
- **Implementation playbook ([19](19-nfr-implementation-playbook.md)):** every relevant domain's CI gate (performance, capacity, usability, a11y, compatibility, portability, supportability, maintainability, license) is wired and green; PR "Definition of Done" boxes checked.
