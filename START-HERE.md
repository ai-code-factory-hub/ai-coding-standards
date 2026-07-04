# Enterprise Starter Kit — START HERE

A **portable, project-agnostic** foundation you copy into every new project. It carries
everything that stays the same across projects, so you only ever write the part that
changes — the requirements.

> Copy this whole `_starter-kit/` folder into a new project, then follow the 6 steps below.

## What's reusable (in this kit) vs. what's new (per project)

| Reusable — lives here, copy as-is | New — you create per project |
|---|---|
| Engineering standards / NFRs (`standards-kb/`) | The requirements knowledge base |
| Design system + tokens (`design-system-kb/`) | The delivery plan (from the template) |
| Process templates (`templates/`) | The locked decisions (from the template) |
| Agent role files (`agents/`) | The actual application code |
| The prompt sequence (`PROMPT-PLAYBOOK.md`) | |

## The kit

```
_starter-kit/
├── START-HERE.md              ← you are here
├── PROMPT-PLAYBOOK.md         ← the exact prompts, in order, with [VARIABLES]
├── standards-kb/              ← 20 engineering-standard domains + release gate (GENERAL)
├── performance-kb/            ← deep performance playbook (extends standards Domain 03)
├── engineering-doctrine/      ← Lean Code Doctrine (YAGNI, anti-duplication)
├── data-kb/                   ← governed analytics data layer + SaaS metric/churn definitions
├── compliance-kb/             ← regime-specific: privacy (GDPR/DPDP/CCPA), attestations
│                                (SOC2/ISO/PCI), sector (HIPAA) + a controls matrix
├── feature-blueprints/        ← buildable specs: report-builder, rbac-admin, subscription,
│                                churn analytics, audit log, admin console, notifications,
│                                search, feature-flags, onboarding/KYC, integrations, import/export
├── admin-kb/                  ← the Admin Area: log explorer, login history, security center,
│                                jobs monitor, data-access logs, comms logs, consent/DSR,
│                                retention, tenant mgmt + a master index (references blueprints)
├── design-system-kb/          ← design tokens + theming (brand-neutral, re-skinnable)
├── component-library-kb/      ← component catalog + React/Tailwind recipes (built from tokens)
├── sre-kb/                    ← on-call: incident response, SLOs, runbooks, postmortems
├── threat-models/             ← STRIDE templates per feature type (auth, upload, API, AI…)
├── finops-kb/                 ← per-tenant cost model, unit economics, cost allocation
├── api-kb/                    ← OpenAPI spec-first: design, security, gateway, versioning,
│                                access logs & usage analytics, developer portal + SDKs
├── mcp-kb/                    ← expose the app to AI agents via a secure MCP server
│                                (per-tool authz, tenant isolation, tool-call logs, rate/cost caps)
├── events-kb/                 ← event-driven / async patterns: outbox, saga, pub/sub,
│                                idempotent consumers, event schema/versioning, eventing ops
├── templates/                 ← CLAUDE.md, DECISIONS.md, DELIVERY-PLAN, KB, gap analysis...
└── agents/                    ← 16 reusable expert roles (Architect, Builder, QA, CEO,
                                 Performance, Simplicity/Lean, Compliance/DPO, Data, Feature...)
```

**How the KBs relate (no duplication):** `standards-kb/` holds the cross-cutting NFR rules
(the *bar*). `performance-kb/`, `compliance-kb/`, and `engineering-doctrine/` go *deeper* on
three of those bars. `feature-blueprints/` + `data-kb/` are *buildable feature specs* that each
inherit the relevant standards. Every file cross-links the standards domain(s) that govern it.

## How to start a new project (6 steps)

1. **Copy** `_starter-kit/` into the new project's empty folder.
2. **Add** your requirement doc(s) + any design brief to the folder.
3. **Ingest** — run `PROMPT-PLAYBOOK.md` Step 1: I read your requirements and build a
   project-specific `knowledge-base/`, referencing `standards-kb/` for the NFRs.
4. **Decide** — Step 3: I fill in `templates/CLAUDE.md.template` and
   `templates/DECISIONS.md.template` for this project (stack, tenancy, conventions).
5. **Plan** — Step 4: I fill in `templates/DELIVERY-PLAN.md.template` — every module and
   every standards-kb domain mapped to a phase.
6. **Build** — Step 6 onward: I build slice by slice, governed by `standards-kb/`,
   styled by `design-system-kb/`, behaving like the roles in `agents/`.

## The three inputs, three jobs (why gaps disappear)

- **Requirements** = WHAT to build → your project KB.
- **Standards (this kit)** = HOW WELL → `standards-kb/` (never rewritten per project).
- **Design tokens (this kit)** = HOW IT LOOKS → `design-system-kb/` (re-skin per brand).

## Re-skinning the design system

`design-system-kb/` ships a **brand-neutral placeholder palette** plus the full **Marg /
MargHR token set** included as a ready default. To re-brand: change the primitive color +
font tokens only; every component inherits through semantic tokens. See
`design-system-kb/01-design-tokens.md`.

## The agents

Reusable expert roles you can ask me to "act as." Each maps to the standards domains it owns:
Solutions Architect · Sr Full-Stack Developer · UI/UX Designer · Business Analyst ·
Project Manager · DevOps Engineer · Security Officer · Code Reviewer · QA/Verifier ·
CEO (Plan Review) · CEO (Final Review). See `agents/`.
