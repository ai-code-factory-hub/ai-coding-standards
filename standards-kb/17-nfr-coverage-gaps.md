# 17 · NFR Coverage & Gaps

The catch-all non-functional requirements that fall between the major domains — portability, supportability, maintainability, capacity, usability, and compatibility — each made concrete and testable.

## Portability & anti-lock-in

- **[MUST]** Package as OCI containers following 12-factor principles; keep cloud-provider services behind adapters so a provider can be swapped without rewriting business logic.
- **[MUST]** Tenant data is exportable in **open, documented formats** (CSV/JSON/standard schemas), not a proprietary dump only your app can read.
- **[MUST]** Maintain a **lock-in register** listing every provider-specific dependency and its exit cost; document installability and, where the market demands it (e.g. regulated or self-hosting customers), an **on-prem / private-cloud deployment path**.

## Supportability

- **[MUST]** Surface stable **error codes** plus a correlation id to users, each mapped to a knowledge-base article, so support can act without a stack trace.
- **[MUST]** Support/impersonation ("view as tenant") is **audited, time-boxed, and consented** — never silent, never open-ended.
- **[MUST]** Provide a **PII-redacted support-bundle export** and per-tenant health/status with a "test connection" action for every integration.

## Maintainability

- **[MUST]** Enforce modularity; set **complexity and duplication budgets in CI** (cyclomatic complexity, file/function length, copy-paste detection, no circular dependencies) that block merge when exceeded.
- **[MUST]** Keep a **tech-debt register** with a budget burned down each cycle; remove dead code and unused dependencies routinely.

## Sustainability / green IT

- **[SHOULD]** Scale idle capacity to zero; schedule heavy batch/compute jobs off-peak; tier storage and compress logs to cut waste and cost.

## License & IP compliance

- **[MUST]** Maintain a dependency **license inventory** (derived from the SBOM) with an allow-list enforced in CI; ship NOTICE/attribution files; confirm all third-party assets (fonts, icons, models, media) are properly licensed for your use.

## Capacity & elasticity

- **[MUST]** Document expected scale — tenants, users, requests/sec, data volume, and a +12-month growth assumption (e.g. 3×) — in a `capacity` doc; define elasticity targets and a **load-tested ceiling** (N× current peak). Quotas protect shared capacity from any single tenant.

## Usability as a measurable NFR

- **[MUST]** Set measurable targets for core tasks — completion rate, time-on-task, ≤N clicks/steps, error rate, and **time-to-first-value** — instrument them with product analytics, and benchmark (e.g. SUS ≥ target). Usability is measured, not asserted.

## Compatibility matrix

- **[MUST]** Publish and test the supported browsers, operating systems, devices, and screen sizes (including minimum mobile OS); degrade gracefully on unsupported combinations; verify data-format interoperability with the systems you exchange data with.

### Examples

- **Healthcare:** the lock-in register flags a managed queue service and documents the self-hosted equivalent, because hospital customers require an on-prem option; impersonation into a patient-data tenant requires a consented, time-boxed, fully-audited session.
- **Fintech / e-commerce:** the compatibility matrix pins supported card-reader browsers and minimum mobile OS versions; a stable `PAYMENT_DECLINED_INSUFFICIENT_FUNDS` error code maps to a KB article so support resolves tickets without reading logs.

## Acceptance checklist

- [ ] Containerized + adapters + open-format export + lock-in register + on-prem path documented where required.
- [ ] Stable error codes + correlation id; audited/time-boxed/consented impersonation; PII-redacted support bundle.
- [ ] Complexity/duplication budgets + no-circular-deps in CI; tech-debt register with per-cycle budget.
- [ ] License inventory + CI allow-list + NOTICE; third-party assets licensed.
- [ ] Documented scale + load-tested ceiling + quotas.
- [ ] Measurable UX targets incl. time-to-first-value, instrumented and benchmarked.
- [ ] Published + tested compatibility matrix with graceful degradation.
