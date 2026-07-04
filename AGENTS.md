# AGENTS.md — Groundwork Operating Rules

This project uses **Groundwork** (see `groundwork/START-HERE.md` and `groundwork/TUTORIAL.md`),
an enterprise-grade knowledge base + agent library. Follow it on every coding task.
Many AI tools read this `AGENTS.md` automatically; tool-specific copies live in `groundwork/tool-configs/`.

## Before you write code
1. Read the relevant domains in `groundwork/standards-kb/` — the non-functional **bar** every
   feature must meet (multi-tenancy, security, reliability, data, logging/audit, API, a11y…).
2. If building a known feature, **adapt the matching spec** in `groundwork/feature-blueprints/`
   or `groundwork/admin-kb/` instead of inventing one.
3. For UI, use `groundwork/design-system-kb/` tokens and `groundwork/component-library-kb/`
   recipes. Never hardcode colors.
4. Follow this project's `CLAUDE.md` / `DECISIONS.md`. Log assumptions; don't re-ask settled things.

## Always — non-negotiable
- Scope every query, file, cache key, and log line to the tenant; enforce with DB row-level security.
- Write an audit entry for every significant action (`groundwork/standards-kb/20-logging-audit-and-traceability.md`).
- Validate all input server-side; treat client and model output as untrusted.
- Meet WCAG 2.2 AA; design loading / empty / error states on every screen.

## Act as a role when asked
Use the expert roles in `groundwork/agents/` (e.g. Solutions Architect, Security Officer,
Feature Architect, QA-Verifier, Code Reviewer, Performance Engineer) when the user invokes them.

## Workflow
Use the 6-step sequence in `groundwork/PROMPT-PLAYBOOK.md` (Ingest → Gap → Decide → Plan → Build → Review).
The unit of work is one module/screen — never "the whole app".

## Before shipping
Run the release gate: `groundwork/standards-kb/99-master-release-gate.md`.

## Scope
Only the folders present in `groundwork/` and referenced here apply. If a folder was removed for
this project, simply don't use it — see `groundwork/TUTORIAL.md` §"Enable / disable".
