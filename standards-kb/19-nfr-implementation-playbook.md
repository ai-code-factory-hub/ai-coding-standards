# 19 · NFR Implementation Playbook

How to enforce the non-functional requirements while coding — each expressed as **Build** (do while coding) · **Automate** (a CI gate that blocks merge) · **Done when** (definition of done). Wire it into the pipeline once.

## Pipeline wiring

- **Pre-commit** (e.g. a git-hook runner + staged-file linter): run the formatter, linter, and type-check on changed files before the commit lands.
- **CI stages, in order:** lint + types → unit/integration → build container → E2E (browser + mobile drivers) → load test → security + license scans → quality gate. **Merge is blocked unless every stage is green.**
- **Definition of Done** (PR template): the "Acceptance checklist" boxes for every domain the change touches are checked.

## Performance

- **Build:** paginate and index new lists; push slow/third-party work to async jobs; avoid N+1.
- **Automate:** load-test thresholds (e.g. p95 < 300 ms, error rate < 1%) plus a soak test in CI.
- **Done when:** budgets hold in CI and no new interactive query lacks an index.

## Capacity & elasticity

- **Build:** set container CPU/memory limits and declare autoscaling policy for new services.
- **Automate:** verify resource limits are present; run the load-tested ceiling scenario.
- **Done when:** the service scales under the target load without breaching limits.

## Usability

- **Build:** emit product-analytics events on core tasks and instrument time-to-first-value.
- **Automate:** assert the required analytics events fire in E2E.
- **Done when:** core-task metrics and TTFV are observable in analytics.

## Accessibility

- **Build:** semantic markup, keyboard operability, sufficient contrast (see [13 · Internationalization & Accessibility](13-i18n-accessibility.md)).
- **Automate:** an automated a11y scan (e.g. axe) in CI fails the build on violations.
- **Done when:** the a11y scan passes and the flow is keyboard-navigable.

## Compatibility

- **Build:** develop against the published support matrix.
- **Automate:** cross-browser E2E (Chromium / Firefox / WebKit) + mobile E2E on min and latest supported OS.
- **Done when:** critical journeys pass on every supported target.

## Portability / anti-lock-in

- **Build:** put provider SDKs behind adapters; keep business code provider-agnostic.
- **Automate:** a dependency-graph rule (e.g. `dependency-cruiser`) forbids importing vendor SDKs outside their adapter module.
- **Done when:** no business module imports a vendor SDK directly.

## Supportability

- **Build:** throw a typed application error with a stable code; propagate the correlation id end to end.
- **Automate:** lint/test that errors carry a stable code and correlation id is threaded through.
- **Done when:** every user-facing error has a stable code + correlation id.

## Maintainability

- **Build:** keep functions/files within complexity and length limits; delete dead code and unused deps.
- **Automate:** linter complexity caps + duplication detector (e.g. `jscpd`) + dead-code/unused-dep scans (e.g. `ts-prune` / `depcheck`).
- **Done when:** complexity, duplication, and dead-code gates are green.

## License & IP

- **Build:** add new dependencies only from the allow-listed licenses.
- **Automate:** generate an SBOM and run the license-allow-list check in CI.
- **Done when:** SBOM is current and the license check passes.

### Examples

- **Healthcare:** the analyzer/device integration SDK is confined to an adapter module and the dependency-graph rule fails any PR that imports it from business logic — keeping the door open to swap devices or run on-prem.
- **Fintech / e-commerce:** the checkout load test (p95 < 300 ms, error < 1%) and a soak run gate every merge, so a payment-page regression is caught in CI rather than during a sale.
