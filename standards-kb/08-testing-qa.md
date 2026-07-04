# 08 · Testing & QA

How changes are proven correct, safe, and non-regressive before they reach production — from unit boundaries to tenant-isolation and load.

## Test strategy

- **[MUST]** A **test pyramid**: many unit tests, fewer integration/API tests, fewest E2E. CI **blocks merge on red**. A coverage floor (**≥80% on business logic**) is a floor, not a goal. Tests are **deterministic** — no reliance on real time, network, or execution order.
- **[MUST]** Unit-test all high-stakes domain and financial logic at its **boundaries**: threshold detection, range/limit checks, rounding and tax/fee math, commission and discount rules, state-transition guards (property-based tests where apt).

> **Example (fintech/e-commerce):** boundary tests cover currency rounding, tax brackets, discount ceilings, refund limits, and interest/fee accrual edges.
> **Example (healthcare):** boundary tests cover reference-range flagging, delta/critical-threshold detection, and rule-based validation.

## Integration, API & contract

- **[MUST]** Integration tests against a **real DB/queue** (e.g. Testcontainers). API tests cover happy path, validation, authorization, pagination, and error-format, plus **contract tests** (OpenAPI/Pact) and **N / N-1 compatibility**.

## End-to-end

- **[MUST]** E2E automation (e.g. Playwright / Detox) for critical journeys end to end — the core create → process → approve → deliver flow, billing/payment, consent, and offline capture + sync where applicable.

## Security & isolation testing

- **[MUST]** Security in CI: SAST + SCA + secret scan + image scan (**fail on critical**); DAST on staging; periodic pen-test; and **tenant-isolation / IDOR regression tests** proving tenant A cannot read or modify tenant B's data.

## Performance & data testing

- **[MUST]** **Load-test** critical flows against the [performance budgets](03-performance-scalability.md) at production-scale data (including peak-load bursts); run soak/stress/spike tests; test bulk import/export and heavy generation jobs at large sizes.
- **[MUST]** Test migrations for **online-safety and reversibility** at prod-like volume.
- **[MUST]** **No production sensitive personal data in non-prod** environments — use synthetic or anonymized data; staging mirrors prod.
- **[SHOULD]** Accessibility (axe) and i18n checks in CI.

## Acceptance checklist

- Test pyramid + CI merge gate + coverage floor; deterministic tests.
- Domain/financial unit tests at boundaries.
- Integration (real DB/queue) + API + contract + N/N-1 compatibility tests.
- E2E on all critical journeys.
- SAST/SCA/DAST + secret/image scans + tenant-isolation/IDOR regression tests.
- Load/soak/spike at production scale; bulk jobs tested large.
- Migrations tested for online-safety and reversibility.
- No production sensitive data in non-prod; a11y + i18n checks in CI.
