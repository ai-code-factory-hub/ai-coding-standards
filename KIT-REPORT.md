# Starter Kit — Build Report

What the reusable `_starter-kit/` now contains, what was deliberately adjusted, and what is
still worth adding later. Compiled 2026-07-05. **163 files · 1,557 internal cross-links · 0 broken.**

## What's in the kit (by category)

| Category | Files | Purpose |
|---|---|---|
| `standards-kb/` | 22 | 20 NFR domains incl. **20 · Logging, Audit & Traceability** + master release gate (the *bar*) |
| `admin-kb/` | 14 | The Admin Area: log explorer, login history, security center, jobs monitor, data-access logs, comms logs, **API usage & analytics**, consent/DSR, retention, tenant mgmt, API keys + master index |
| `api-kb/` | 8 | OpenAPI spec-first: design guide, spec-first workflow, API security, gateway + rate limiting, versioning/lifecycle, access logs & usage analytics, developer portal + SDKs |
| `mcp-kb/` | 7 | Expose the app to AI agents via a secure MCP server: concepts, building, security/authz, safety/prompt-injection, tool-call logs + rate/cost caps, testing/governance |
| `performance-kb/` | 6 | Deep performance playbook — pooling, query/statement, code-level, frontend, profiling (extends Domain 03) |
| `engineering-doctrine/` | 3 | Lean Code Doctrine (YAGNI, stdlib/native-first) + anti-duplication & refactoring |
| `data-kb/` | 3 | Governed analytics data layer + SaaS metric/churn definitions |
| `compliance-kb/` | 9 | privacy (GDPR, DPDP, CCPA) · attestations (SOC2, ISO 27001, PCI-DSS) · sector (HIPAA) · controls matrix |
| `events-kb/` | 7 | Event-driven / async patterns: messaging fundamentals, outbox/inbox, idempotent consumers, saga/orchestration, event schema/versioning, eventing ops |
| `feature-blueprints/` | 15 | Buildable feature specs incl. **workflow & approval engine** (see below) |
| `design-system-kb/` | 4 | Brand-neutral tokens + Indigo skin, theming, components/a11y |
| `component-library-kb/` | 6 | Component catalog (36 components) + React/Tailwind recipes built from the tokens |
| `sre-kb/` | 14 | On-call: incident response, SLOs/error budgets, escalation, postmortem + 8 runbooks |
| `threat-models/` | 10 | STRIDE method + template + 8 per-feature threat models (auth, upload, API, payment, AI…) |
| `finops-kb/` | 4 | Per-tenant cost model worksheet, unit economics, cost allocation/tagging |
| `templates/` | 8 | CLAUDE, DECISIONS, DELIVERY-PLAN, KB, gap, ADR, BUILD-LOG, DoD |
| `agents/` | 20 | 19 expert roles (incl. **Admin Auditor**, **API Platform Engineer**, **MCP Server Engineer**) + index |
| root | 2 | START-HERE, PROMPT-PLAYBOOK |

**Feature blueprints (15):** report-builder · rbac-admin · subscription-billing ·
customer-analytics-churn · audit-log-and-activity · admin-console · notifications-center ·
global-search · feature-flags-and-experimentation · onboarding-and-kyc ·
integrations-and-webhooks · import-export-pipeline · support-and-impersonation ·
**workflow-and-approval-engine** · README.

**Agents (19):** Solutions Architect · Business Analyst · Sr Full-Stack Developer · UI/UX Designer ·
Security Officer · DevOps Engineer · QA-Verifier · Code Reviewer · Project Manager ·
CEO-Plan-Review · CEO-Final-Review · Performance Engineer · Simplicity Reviewer (Lean Code) ·
Compliance Officer (DPO) · Data & Analytics Engineer · Feature Architect · **Admin Auditor** ·
**API Platform Engineer** · **MCP Server Engineer**.

## What was adjusted (Solutions-Architect calls)

1. **No duplicate Performance KB.** Pooling/N+1/indexing already lived in `standards-kb/03`.
   `performance-kb/` *extends* it with hands-on depth instead of repeating it.
2. **"gstack" not baked in as KB.** It's a specific external toolchain, not portable knowledge.
   Its capabilities (automated QA, dogfooding, design/devex review) live as **agents** instead.
3. **"ponytail" rewritten as original IP.** `engineering-doctrine/` is a fresh, more rigorous
   "Lean Code Doctrine" — not a copy of any existing tool.
4. **Compliance layered, not duplicated.** `standards-kb/09` is the generic privacy *program*;
   `compliance-kb/` adds the *regime-specific* controls on top, with a "do-it-once, satisfy-many"
   controls matrix.

## How it all connects (no overlap)

- `standards-kb/` = the rules (the bar every feature must clear).
- `performance-kb/`, `compliance-kb/`, `engineering-doctrine/` = **deeper** on three of those bars.
- `feature-blueprints/` + `data-kb/` = **buildable feature specs**, each inheriting the standards.
- Every file cross-links the standards domain(s) that govern it (verified: 0 broken links).

## Still worth adding later (optional — add only when a project needs them)

The load-bearing enterprise gaps are all closed. What remains is deliberately *not* built —
adding it speculatively would violate the Lean Code Doctrine. Add each when a real project needs it:

- **Mobile blueprint** — offline-first, sync, push, app-store release (only if you build native apps).
- **Localization content kit** — actual locale bundles + translation workflow (Domain 13 + admin-kb/localization define the rules; this would be the content).
- **Coded component package** — the `component-library-kb/` catalog compiled into a real `packages/ui` (belongs in the project codebase once the stack is scaffolded, not in the doc kit).
- **More feature blueprints** as patterns recur: scheduling/calendar, document generation,
  e-signature, multi-currency wallet/ledger, in-app chat/messaging, marketplace/commissions.

> Previously-listed gaps now **built**: SRE runbooks, threat-model library, per-tenant cost model,
> component catalog, API (OpenAPI) KB, MCP-server KB, logging taxonomy, admin area,
> event-driven patterns, and the workflow & approval engine.

## How to use it

Copy `_starter-kit/` (or unzip `starter-kit.zip`) into a new project, add the requirement doc,
and run `PROMPT-PLAYBOOK.md`. For a feature, ask the **Feature Architect** to adapt the matching
blueprint; for the bar, everything inherits `standards-kb/`.
