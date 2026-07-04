# 11 · Frontend, UX & Product Features

Baseline frontend, UX, and horizontal product-feature standards every screen and interactive surface of a multi-tenant SaaS must satisfy.

## Frontend baseline

- **[MUST]** Every screen is **responsive** across the supported device/screen-size matrix, **accessible** (WCAG 2.2 AA — see [13 · Internationalization & Accessibility](13-i18n-accessibility.md)), and built on a **consistent design system**. Components consume design tokens; they never hardcode raw hex or ad-hoc spacing (see [../design-system-kb/](../design-system-kb/README.md)).
- **[MUST]** Every screen ships designed **loading, empty, and error states** — no blank spinners, no unhandled failures.
- **[MUST]** All user-facing strings are **i18n-externalized**; money, dates, and numbers are **locale/timezone-formatted** at render time.
- **[MUST]** **Theming / white-label per tenant** is supported end to end (app, login, email, print/PDF). Token architecture and theme catalog live in [../design-system-kb/](../design-system-kb/README.md).

## Dashboards

- **[MUST]** Dashboards use configurable, graph-capable widgets with a template library **and** a self-service builder: chart-type switching, 2D/3D, share / download-image / export-data, and drill-down.
- **[MUST]** Widgets are **permission- and tenant-scoped**, backed by **pre-aggregated** data, and load **independently** — one slow widget never blocks the page.
- **[SHOULD]** Support a broad chart catalog: bar/column/stacked/grouped/radar; scatter/bubble/histogram/box/heatmap; pie/donut/treemap/funnel/waterfall/sunburst; line/area/gantt/sparkline; KPI card/gauge/progress/bullet; optional 3D.

## Configurable fields (field on/off)

- **[MUST]** Field **visibility, requiredness, label, and order** are configurable per tenant/role as **metadata, not code**.
- **[MUST]** The data model and API stay **stable** when a field is hidden; custom/extension fields are supported without schema forks.

## Notification & template engine

- **[MUST]** A unified notification service fronts per-channel engines (email / SMS / in-app / push / **messaging channels with regional regulatory constraints**), each behind a **swappable adapter**, running **async with retry + DLQ**.
- **[MUST]** Enforce **consent/opt-out per channel + category**, **idempotent sends**, and delivery tracking.
- **[MUST]** Respect per-channel regulatory constraints for the operating region — approved-template requirements, session/consent windows, registered sender IDs, and quiet-hours rules — as **configuration**, not hardcoded logic. Templates are multi-language.

> **Example (healthcare):** patient result-ready alerts go out over a regional messaging channel that only permits pre-approved templates within a rolling consent window; opt-out is honored per category.
> **Example (fintech/e-commerce):** transactional payment receipts use a registered sender ID mandated by the local telecom regulator, while marketing pushes obey a separate opt-in and quiet-hours policy.

## Reports & report writer

- **[MUST]** Reports are **parameterized, permission-scoped, tenant-isolated**, run **async for large jobs**, and are **replica-backed**.
- **[MUST]** A no-code report builder operates over an **allow-listed, governed metadata layer** — safe parameterized queries only, **no full-table scans**.
- **[MUST]** Reports are saveable / schedulable / shareable / exportable, with guardrails (row and time limits).

## Print engine

- **[MUST]** Configurable print templates support multiple paper sizes (A4/A5/thermal/custom) plus margins, orientation, headers, and page numbers.
- **[MUST]** PDF is **server-side and deterministic**, tenant-configurable and localized, with barcode/QR + label printing; batch runs go through a bulk job.

## Import / Export

- **[MUST]** A guided pipeline: upload → column mapping → validate/preview → async bulk → result report.
- **[MUST]** Streamed / batched / resumable / idempotent processing with **partial-success handling** (never silently drop rows) and malware scanning on uploads.
- **[MUST]** Exports respect permissions + filters, are tenant-isolated, **audited**, and run async/streamed.

## Bulk-operation framework

- **[MUST]** A generic bulk framework (approve/assign/print/dispatch/…): scope → preview → async batched with progress → per-item result, applying the **same authz + validation as single-item operations**, **audited**, with per-tenant fairness.

## Workflow / approval engine

- **[MUST]** A **config-driven** workflow engine (no code per workflow): states/transitions; approvals (single / multi / sequential / parallel / quorum); if/else and **limit/range routing** (e.g. amount over a threshold routes to senior approval — ties to ABAC in the security domain); role/assignee routing + delegation.
- **[MUST]** **Escalation / SLA** support, including **time-critical escalation SLAs** and deadline-driven routing; actions on transition (notify / update / webhook); durable and race-safe (no double-approval); versioned, with an audit history and a visual designer.

> **Example (healthcare):** a time-critical result triggers a 30-minute escalation SLA that pages successive on-call tiers until acknowledged.
> **Example (fintech/e-commerce):** a refund above a configured amount routes to a manager queue with a same-day SLA before disbursement.

## Acceptance checklist

- Responsive + a11y + design-system + i18n + theming baseline on every screen, with loading/empty/error states.
- Configurable dashboards with independent widget loading and a chart catalog.
- Field on/off driven by metadata; data model/API stable when fields hidden.
- Unified notification engine with per-channel adapters, retry/DLQ, consent, and region-specific regulatory constraints as config.
- Async governed reports + no-code report writer over an allow-listed metadata layer.
- Deterministic server-side PDF/print engine, tenant-configurable and localized.
- Import/export pipeline with resumable batches, partial-success handling, and audited exports.
- Generic bulk-operation framework with per-item results and shared authz.
- Config-driven workflow/approval engine with SLA/escalation, durability, versioning, and audit.
