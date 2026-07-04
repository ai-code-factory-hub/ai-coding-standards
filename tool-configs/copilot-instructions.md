# GitHub Copilot Instructions — Groundwork

This project uses **Groundwork**, an enterprise-grade knowledge base + agent library
(`groundwork/START-HERE.md`, `groundwork/TUTORIAL.md`). Apply it to all generated code.

## Before writing code
1. Follow the NFR rules in `groundwork/standards-kb/` (multi-tenancy, security, reliability,
   data, logging/audit, API, frontend, i18n/a11y).
2. For known features, follow the matching spec in `groundwork/feature-blueprints/` or `groundwork/admin-kb/`.
3. For UI, use `groundwork/design-system-kb/` tokens and `groundwork/component-library-kb/` recipes; no hardcoded colors.
4. Obey this project's `CLAUDE.md` / `DECISIONS.md`.

## Always
- Scope every query/file/log to the tenant; enforce with database row-level security.
- Write an audit entry for every significant action (see `groundwork/standards-kb/20-logging-audit-and-traceability.md`).
- Validate all input server-side; never trust client input or model output.
- Meet WCAG 2.2 AA; include loading, empty, and error states.

## Definition of done
A change is done only when it passes the relevant acceptance checklists in `groundwork/standards-kb/`
and the release gate `groundwork/standards-kb/99-master-release-gate.md`.

Only folders present in `groundwork/` apply — removing a folder disables that module.
