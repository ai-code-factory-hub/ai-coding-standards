# 14 · Developer Experience & Documentation

Standards for code quality gates, decision records, documentation, local setup, and in-app help that keep a multi-tenant SaaS maintainable and onboardable.

## Code quality & review

- **[MUST]** Enforce a **formatter + linter + static typing** in CI and pre-commit; document project structure and naming conventions.
- **[MUST]** **Code review is mandatory** and checks correctness, security, **tenant-scoping**, and tests.
- **[SHOULD]** Prefer small PRs and conventional commits. Keep a **`CLAUDE.md` / agent-rules file at the repo root** that points the AI assistant to this standards knowledge base (this `standards-kb/`) so automated contributors follow the same rules.

## Decisions & documentation

- **[MUST]** Record **ADRs** for significant decisions (tenancy model, datastore, auth approach, external-integration mapping approach).
- **[MUST]** Auto-generate API docs from the OpenAPI/schema source and keep them **contract-synced**; maintain per-service README/runbook (build / run / test / deploy / config / on-call), documented config/env vars, and current architecture diagrams.

> **Example (healthcare):** an ADR captures why the platform chose a hybrid pooled/siloed tenancy model for regulated labs; the runbook documents the on-call escalation for result-delivery failures.
> **Example (fintech/e-commerce):** an ADR records the choice of an idempotent ledger for payments; API docs for the checkout endpoints are generated from OpenAPI and contract-tested.

## Local setup

- **[MUST]** **One-command local setup** (devcontainer or equivalent) that provisions seed **multi-tenant** data so a new developer can run realistic scenarios immediately.

## In-app help

- **[MUST]** In-app help, empty states, and tours, plus a searchable help center and usage-driven onboarding nudges.

## Acceptance checklist

- Formatter / linter / static types enforced in CI and pre-commit; mandatory code review covering tenant-scoping.
- ADRs for significant decisions.
- Auto-generated, contract-synced API docs; per-service runbooks; documented config; current architecture diagrams.
- One-command local setup with seed multi-tenant data.
- In-app help, empty states, tours, and searchable help center.
- Repo-root `CLAUDE.md` pointing the AI assistant to this `standards-kb/`.
