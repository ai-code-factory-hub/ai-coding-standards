# Groundwork — The Tutorial

**Enterprise-grade "vibe coding," without the enterprise-grade regrets.**

Groundwork is a folder of Markdown you drop into any project. Once it's there, your
AI coding tool (Claude Code, Cursor, Copilot, Windsurf, Cline/Roo, or anything that
reads a rules file) writes code that already knows about multi-tenancy, security,
audit logging, performance budgets, accessibility, and compliance — because it read
the rules *before* it typed a line.

This tutorial takes you from zero to shipping. Grab a coffee; it's friendly and
skimmable.

---

## 0. How to use this tutorial

### Who this is for
You're a competent developer who has started using AI coding tools and loves how fast
they are — but you've noticed the output can be *naive*: no tenant scoping, no audit
trail, happy-path only, "we'll add auth later." Groundwork fixes that. You don't need
to be an architect or a security expert; that expertise now lives in the folder.

### Two paths through this doc

| Path | Time | Read these sections |
|---|---|---|
| **Quick start** — "just make my AI smarter, now" | ~5 min | 1 (skim) → 3 (install) → 4 (your tool only) → 6 (workflow) |
| **Full setup** — tailor the kit to your project | ~30 min | All of it, in order |

### Legend / conventions

| Convention | Meaning |
|---|---|
| `groundwork/standards-kb/…` | A path **inside the kit**. Always relative to your project root. |
| `> Prompt:` | A copy-paste prompt for your AI. Replace `[BRACKETS]` with your own values. |
| **@-mention** | Typing `@groundwork/some-file.md` in your AI's chat to force it to read that exact file. |
| **Rules file** | The small pointer file that tells your tool "read Groundwork." One per tool. |
| 🧩 | An optional module you can keep or drop. |

Throughout, "the AI," "your tool," and "the assistant" all mean the same thing:
whatever coding agent you're driving.

---

## 1. What Groundwork is & why it helps

**In one sentence:** Groundwork is the *brain and rulebook* your AI reads before it
writes code, so that "vibe coding" produces enterprise-grade output instead of a
demo that falls over in production.

Left to its own devices, an AI writes the most *obvious* code — the thing that makes
the screenshot work. Groundwork changes what "obvious" means. When the rules say
"every table is tenant-scoped, every mutation is audited, every screen is WCAG 2.2
AA," the AI builds *that* by default, without you nagging it every prompt.

### Before / after

| Aspect | Vibe coding **without** Groundwork | Vibe coding **with** Groundwork |
|---|---|---|
| Multi-tenancy | Single-tenant; `tenant_id` bolted on later (painfully) | Row-level tenant isolation from the first migration |
| Security | Auth "to be added"; secrets in code | Threat-modeled; authz + input validation by default |
| Audit / logging | `console.log` if you're lucky | Structured, masked audit events + an admin screen to view them |
| Performance | N+1 queries discovered in prod | Query/pooling budgets applied up front |
| Accessibility | Div soup, mouse-only | Keyboard + screen-reader states designed in |
| Compliance | "What's a DSR?" | GDPR/DPDP/SOC2/PCI/HIPAA controls mapped when relevant |
| Consistency | Every feature invents its own patterns | One shared bar; every feature inherits it |
| Your effort | Re-explaining standards every single prompt | Say it once; it's written down and reused |

You still drive. Groundwork just makes sure the car has seatbelts.

---

## 2. The mental model

Five kinds of thing live in the kit. Learn these five words and you understand the
whole system:

| Part | Folder(s) | It is… | Analogy |
|---|---|---|---|
| **The BAR** | `standards-kb/` (+ deep-dives: `performance-kb/`, `compliance-kb/`, `engineering-doctrine/`, `threat-models/`, `sre-kb/`, `finops-kb/`, `events-kb/`, `data-kb/`, `api-kb/`, `mcp-kb/`) | The non-negotiable rules every feature must clear | Building code & safety regulations |
| **BUILDABLE specs** | `feature-blueprints/`, `admin-kb/` | Ready-made specs for features you can actually build | Architect's blueprints |
| **The LOOK** | `design-system-kb/`, `component-library-kb/` | Design tokens + a 36-component catalog with React/Tailwind recipes | The interior design catalog |
| **ROLES** | `agents/` | 19 expert personas you invoke ("act as the Security Officer") | Specialists you call in |
| **The WORKFLOW** | `PROMPT-PLAYBOOK.md`, `templates/`, `START-HERE.md` | The 6-step build sequence + fill-in-the-blank docs | The project plan & forms |

### One picture (in text)

```
                    YOUR REQUIREMENTS  (what you type)
                             │
                             ▼
        ┌────────────────────────────────────────────────┐
        │                 PROMPT-PLAYBOOK                 │  ← the workflow
        │   Ingest → Gap → Decide → Plan → Build → Verify │
        └────────────────────────────────────────────────┘
                             │  the AI, while building, consults:
          ┌──────────────────┼───────────────────┬────────────────┐
          ▼                  ▼                   ▼                ▼
   ┌─────────────┐   ┌────────────────┐  ┌──────────────┐  ┌───────────┐
   │ standards-kb│   │feature-blueprint│  │ design-system│  │  agents/  │
   │  = THE BAR  │   │  admin-kb       │  │  component-lib│  │ = ROLES   │
   │ (rules)     │   │  = BUILDABLE    │  │  = THE LOOK   │  │ (personas)│
   └─────────────┘   └────────────────┘  └──────────────┘  └───────────┘
          │
          ▼
   ┌──────────────────────────────────────────────┐
   │  deep-dive KBs (only when a project needs it) │
   │  performance · compliance · threat-models ·   │
   │  sre · finops · events · data · api · mcp ·   │
   │  engineering-doctrine (Lean Code)             │
   └──────────────────────────────────────────────┘
                             │
                             ▼
                   ENTERPRISE-GRADE CODE
```

The golden rule of the workflow (from `groundwork/PROMPT-PLAYBOOK.md`):
**Ingest → Gap → Decide → PLAN → Build. Never jump from docs straight to "build it."**

---

## 3. Install (copy into your project)

Groundwork is just files. "Installing" means copying the folder into your project and
naming it `groundwork/`.

### Step 1 — get the kit

Pick one:

```bash
# Option A: clone, then copy the folder in
git clone <groundwork-repo-url> /tmp/groundwork-src
cp -R /tmp/groundwork-src /path/to/your-project/groundwork

# Option B: you already have it as a zip
unzip groundwork.zip -d /path/to/your-project/
# then rename the extracted folder to exactly:  groundwork
```

> The folder **must** be named `groundwork/` at your project root. Every rules file
> and every cross-link in the kit assumes that path.

### Step 2 — verify the layout

From your project root:

```bash
ls groundwork
```

You should see:

```
your-project/
├── groundwork/
│   ├── START-HERE.md
│   ├── PROMPT-PLAYBOOK.md          ← the 6-step build sequence
│   ├── KIT-REPORT.md               ← what's in the kit + changelog
│   ├── standards-kb/               ← 20 NFR domains + 99-master-release-gate.md
│   ├── performance-kb/             🧩 deep performance playbook
│   ├── engineering-doctrine/       🧩 Lean Code Doctrine (YAGNI)
│   ├── compliance-kb/              🧩 GDPR · DPDP · CCPA · SOC2 · ISO · PCI · HIPAA
│   ├── data-kb/                    🧩 governed analytics + churn/MRR definitions
│   ├── events-kb/                  🧩 async: outbox, saga, idempotent consumers
│   ├── sre-kb/                     🧩 runbooks, SLOs, incident response
│   ├── threat-models/              🧩 STRIDE templates per feature type
│   ├── finops-kb/                  🧩 per-tenant cost model
│   ├── api-kb/                     🧩 OpenAPI spec-first, gateway, versioning
│   ├── mcp-kb/                     🧩 expose the app to AI agents (MCP server)
│   ├── feature-blueprints/         15 buildable feature specs
│   ├── admin-kb/                   the Admin Area (log explorer, security center…)
│   ├── design-system-kb/           tokens + theming
│   ├── component-library-kb/       36-component catalog + React/Tailwind recipes
│   ├── agents/                     19 expert role files
│   ├── templates/                  CLAUDE.md, DECISIONS.md, DELIVERY-PLAN, ADR…
│   └── tool-configs/               ready-made per-tool rules files
└── (your app code)
```

Nothing runs. Nothing installs. It's Markdown. The magic is in **Step 4**, where you
point your tool at it.

---

## 4. Turn it on in your tool (per-tool setup)

### The shared mechanism (read this once)

Every AI coding tool supports some kind of **rules file** — a small file it reads
automatically at the start of a session. You do **not** paste the whole kit into that
file (that would blow your context window). Instead, the rules file is a short
**pointer**: "There's a `groundwork/` folder. Read `groundwork/START-HERE.md` and
`groundwork/PROMPT-PLAYBOOK.md`, treat `groundwork/standards-kb/` as the bar, and open
the specific file relevant to the task."

Then, for any given task, you **@-mention** the exact files you want in play (e.g.
`@groundwork/feature-blueprints/rbac-admin.md`). The AI reads only what's relevant —
that's why the kit is many small files instead of one giant one.

> Ready-made rules files ship in `groundwork/tool-configs/`. If that folder is empty in
> your copy, use the inline content below — it's the same thing, copy-paste ready.

Set up **only the tool you use**. Skip the rest.

---

### 4.1 Claude Code (VS Code extension or CLI)

**File to add:** `CLAUDE.md` at your **project root** (not inside `groundwork/`).

Claude Code reads `CLAUDE.md` automatically on every session. Point it at the kit:

```bash
# from your project root
cat > CLAUDE.md <<'EOF'
# Project Constitution

This project uses the **Groundwork** enterprise kit in `groundwork/`.

## Read before building
- `groundwork/START-HERE.md` — the mental model.
- `groundwork/PROMPT-PLAYBOOK.md` — the 6-step build sequence. Follow it.
- `groundwork/standards-kb/` — THE BAR. Every feature must clear these NFR domains,
  and `groundwork/standards-kb/99-master-release-gate.md` before "done."

## How to build
- One slice (module/screen) at a time — never "the whole app."
- Multi-tenant scope + audit logging + tests are ON by default, not optional.
- Style with `groundwork/design-system-kb/` and `groundwork/component-library-kb/`.
- For a feature, adapt the matching `groundwork/feature-blueprints/*.md`.
- Act as the relevant role in `groundwork/agents/` when asked.
- Log every assumption in DECISIONS.md; don't re-ask locked decisions.

## Precedence
This file and the project's DECISIONS.md WIN over anything in `groundwork/` when they
conflict. Groundwork is the default, not the override.
EOF
```

**Verify it's active:** start a session and ask:

> Prompt: `What does groundwork/standards-kb say is required for multi-tenancy? Cite the file.`

If it names `groundwork/standards-kb/01-architecture-multitenancy.md`, you're live.

**Optional power-ups (Claude Code only):**

- **Turn agents into subagents.** Copy role files into `.claude/agents/` so you can
  spin them up as dedicated subagents:
  ```bash
  mkdir -p .claude/agents && cp groundwork/agents/*.md .claude/agents/
  ```
  Then: "Use the security-officer subagent to threat-model this upload flow."
- **Turn playbook steps into slash commands.** Create `.claude/commands/` files (e.g.
  `build-slice.md`, `verify.md`) whose body is the matching prompt from
  `groundwork/PROMPT-PLAYBOOK.md`. Now `/build-slice` runs Step 6 every time.

---

### 4.2 Cursor

**File to add:** `.cursor/rules/groundwork.mdc`

```bash
mkdir -p .cursor/rules
cp groundwork/tool-configs/cursor.mdc .cursor/rules/groundwork.mdc
```

If `tool-configs/cursor.mdc` isn't present, create the file with this content:

```mdc
---
description: Groundwork enterprise kit — always apply
alwaysApply: true
---
This project uses the Groundwork kit in `groundwork/`.
- Read `groundwork/START-HERE.md` and `groundwork/PROMPT-PLAYBOOK.md`.
- Treat `groundwork/standards-kb/` as the non-negotiable bar; clear
  `groundwork/standards-kb/99-master-release-gate.md` before calling anything done.
- Tenant scope + audit + tests are on by default.
- Style from `groundwork/design-system-kb/` and `groundwork/component-library-kb/`.
- For a feature, adapt the matching file in `groundwork/feature-blueprints/`.
- @-mention specific groundwork files for the task at hand.
```

**Verify:** open Cursor's chat and ask it to cite a `groundwork/standards-kb/` file.
Cursor also shows applied rules in the composer — `groundwork.mdc` should appear.

---

### 4.3 GitHub Copilot

**File to add:** `.github/copilot-instructions.md`

```bash
mkdir -p .github
cp groundwork/tool-configs/copilot-instructions.md .github/copilot-instructions.md
```

If the source file isn't present, create it with:

```markdown
# Copilot Instructions

This repository uses the **Groundwork** enterprise kit in `groundwork/`.

When generating code or answering in chat:
- Follow `groundwork/PROMPT-PLAYBOOK.md`; read `groundwork/START-HERE.md` for context.
- Meet the NFRs in `groundwork/standards-kb/` (multi-tenancy, security, audit,
  performance, accessibility, testing). Clear `99-master-release-gate.md` before "done."
- Default to tenant-scoped queries, audit logging, and tests.
- Use `groundwork/design-system-kb/` + `groundwork/component-library-kb/` for UI.
- Adapt `groundwork/feature-blueprints/*.md` for features.
```

**Verify:** in Copilot Chat, ask "What standards does this repo require for logging?"
It should reference `groundwork/standards-kb/20-logging-audit-and-traceability.md`.
(Copilot reads this file automatically in VS Code, Visual Studio, and on github.com.)

---

### 4.4 Windsurf

**File to add:** `.windsurfrules` at the project root.

```bash
cp groundwork/tool-configs/windsurfrules.md .windsurfrules
```

If not present, create `.windsurfrules` with:

```
This project uses the Groundwork kit in `groundwork/`.
- Read `groundwork/START-HERE.md` + `groundwork/PROMPT-PLAYBOOK.md`.
- `groundwork/standards-kb/` is the bar; clear `99-master-release-gate.md` before done.
- Tenant scope, audit logging, and tests are on by default.
- UI from `groundwork/design-system-kb/` + `groundwork/component-library-kb/`.
- Features from `groundwork/feature-blueprints/`. @-mention specific files per task.
```

**Verify:** ask Cascade (Windsurf's assistant) to cite a standards-kb domain file.

---

### 4.5 Cline / Roo Code

**File to add:** `.clinerules` (Cline) or `.roorules` (Roo Code) at the project root.

```bash
cp groundwork/tool-configs/clinerules.md .clinerules
# Roo Code users:
cp groundwork/tool-configs/clinerules.md .roorules
```

If not present, use the same body as the Windsurf block above. Both tools read these
files as system rules at session start.

**Verify:** start a task and confirm it references `groundwork/` files unprompted.

---

### 4.6 Any other tool — the `AGENTS.md` standard

A growing number of tools (and some of the above) read a root **`AGENTS.md`** file —
an emerging cross-tool standard. Groundwork ships one at the repo root; copy it up:

```bash
cp groundwork/AGENTS.md ./AGENTS.md   # if present in your copy
```

If it isn't present, create `AGENTS.md` at your project root with the same pointer
content as the Claude Code `CLAUDE.md` above. Keeping both `CLAUDE.md` and `AGENTS.md`
in sync means almost any tool a teammate uses will pick up Groundwork.

---

### Quick reference — where each file goes

| Tool | File | Location |
|---|---|---|
| Claude Code | `CLAUDE.md` | project root |
| Cursor | `groundwork.mdc` | `.cursor/rules/` |
| GitHub Copilot | `copilot-instructions.md` | `.github/` |
| Windsurf | `.windsurfrules` | project root |
| Cline | `.clinerules` | project root |
| Roo Code | `.roorules` | project root |
| Anything else | `AGENTS.md` | project root |

---

## 5. Enable / disable — keep what you need, drop what you don't

This is the part people miss, and it matters. **Every folder in Groundwork is
independent.** The AI only uses what is (a) present on disk and (b) referenced. So you
can tailor the kit to your project by keeping some modules and dropping others — with
zero risk of breaking anything.

`standards-kb/`, `feature-blueprints/`, `design-system-kb/`, `component-library-kb/`,
`agents/`, `templates/`, and the playbook are the **core** — keep those. Everything
else (the 🧩 deep-dive KBs) is opt-in.

### Decision table — which modules matter for your project

Keep = ✅, Skip unless noted = ⬜.

| Module | Internal tool / MVP | Public SaaS | Fintech / payments | Healthcare | AI-heavy app | Mobile app |
|---|:--:|:--:|:--:|:--:|:--:|:--:|
| `standards-kb/` (core bar) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `engineering-doctrine/` (Lean Code) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `design-system-kb/` + `component-library-kb/` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `feature-blueprints/` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `admin-kb/` (Admin Area) | ⬜ | ✅ | ✅ | ✅ | ✅ | ⬜ |
| `performance-kb/` | ⬜ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `compliance-kb/` | ⬜ | ✅ (GDPR/DPDP) | ✅ (+PCI, SOC2) | ✅ (+HIPAA) | ✅ | ✅ |
| `threat-models/` | ⬜ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `sre-kb/` (runbooks/SLOs) | ⬜ | ✅ | ✅ | ✅ | ✅ | ⬜ |
| `finops-kb/` (per-tenant cost) | ⬜ | ✅ | ✅ | ⬜ | ✅ | ⬜ |
| `events-kb/` (async/outbox/saga) | ⬜ | ⬜ (if async) | ✅ | ✅ | ⬜ | ⬜ |
| `data-kb/` (analytics/churn) | ⬜ | ✅ | ✅ | ⬜ | ✅ | ⬜ |
| `api-kb/` (public/partner API) | ⬜ | ✅ (if public API) | ✅ | ⬜ | ✅ | ⬜ |
| `mcp-kb/` (expose app to AI agents) | ⬜ | ⬜ | ⬜ | ⬜ | ✅ | ⬜ |

Rules of thumb:
- **Payments** → always keep `compliance-kb/` (PCI-DSS) + `threat-models/` (payment) +
  `events-kb/` (ledgers love the outbox pattern).
- **Personal / health data** → keep `compliance-kb/` (HIPAA/GDPR/DPDP) + `admin-kb/`
  (consent/DSR, retention, data-access logs).
- **Your app is used by AI agents** → keep `mcp-kb/`.
- **Reporting / churn / dashboards** → keep `data-kb/`.

### How to disable a module (three ways)

You can use any one of these. They stack; belt-and-suspenders is fine.

**(a) Delete the folder** — simplest, most honest.
```bash
rm -rf groundwork/mcp-kb groundwork/finops-kb
```

**(b) Remove its mention from your rules file** — if your rules file listed it
explicitly, delete that line. (If your rules file just points at `groundwork/`
generically, there's nothing to remove — the AI simply won't find a deleted folder.)

**(c) Tell the AI in `DECISIONS.md`** — the durable, self-documenting way. Add:
```markdown
## Scope decisions
- NOT using `groundwork/mcp-kb/` — no AI-agent access to this app.
- NOT using `groundwork/events-kb/` — synchronous request/response only for v1.
- NOT using `groundwork/finops-kb/` — internal tool, no per-tenant billing.
```
Because the AI reads `DECISIONS.md` every session, it stops considering those modules.

### How to re-enable

- Copy the folder back from your kit source:
  ```bash
  cp -R /tmp/groundwork-src/events-kb groundwork/events-kb
  ```
- Delete the "NOT using" line from `DECISIONS.md` (and re-add the mention to your rules
  file if you'd removed it).

### Is disabling safe? Yes.

Groundwork files cross-link each other, but those links are just Markdown references.
If a file links to a folder you removed, the AI simply **won't follow a dead link** —
it doesn't error, crash, or corrupt anything. Nothing depends on a folder being
present. Keep the core, drop the rest, sleep well.

---

## 6. The daily workflow (step-by-step)

The whole method lives in `groundwork/PROMPT-PLAYBOOK.md`. Here's the loop you'll
actually run. Steps 1–4 happen **once per project**; Steps 6 & 8 repeat **per module**
and are where you'll spend 90% of your time.

Replace every `[BRACKET]` with your own value.

### Step 1 — Ingest (build the knowledge base) · once

> Prompt:
> ```
> New project: [PROJECT_NAME] — [ONE-LINE DOMAIN].
> Source docs in this folder: [REQUIREMENT_DOC], [DESIGN_BRIEF or "none"].
> Read them fully. Then build a cross-linked knowledge-base/ folder — ONE markdown
> file per topic, plus a README.md index — capturing every requirement, screen, and
> NFR, referencing groundwork/standards-kb/ for the "how well." Do NOT write app code
> yet. Show me the KB index when done.
> ```

### Step 2 — Gap analysis · once

> Prompt:
> ```
> Now produce knowledge-base/REQUIREMENTS-DIGEST-AND-GAPS.md: (A) what each doc
> defines, (B) inconsistencies to reconcile, (C) modules under-specified, (D) logic
> named but not defined. Rank gaps by build priority and list the decisions that are
> genuinely mine to make. Don't build.
> ```

### Step 3 — Decide (lock the constitution) · once

> Prompt:
> ```
> Act as the Solutions Architect (groundwork/agents/solutions-architect.md). Give me
> ONE recommendation each for: tech stack, multi-tenancy model, repo structure, auth
> model — one line of "why" each. Then fill in groundwork/templates/CLAUDE.md.template
> and DECISIONS.md.template for THIS project. Ask me only the decisions that are truly
> mine; pick sensible defaults for the rest and log them.
> ```

### Step 4 — Plan (the delivery plan) · once · always before building

> Prompt:
> ```
> Create DELIVERY-PLAN.md from groundwork/templates/DELIVERY-PLAN.md.template. Break
> the product into phases, foundation first (identity/tenancy/audit before anything
> that writes real data). Map EVERY module and EVERY standards-kb domain to a phase —
> nothing dropped. Don't build yet.
> ```

### Step 6 — Build one slice · repeat · your everyday workhorse

This is the prompt you'll use most. Same shape every time:

> Prompt:
> ```
> Build Phase [N] — Module [ID] ([NAME]).
> Source: knowledge-base/[PAGES] + groundwork/standards-kb/ +
>          groundwork/feature-blueprints/[BLUEPRINT].md.
> Done = [CONCRETE OUTCOME THAT PROVES IT WORKS].
> Autonomy: build it, follow CLAUDE.md, log assumptions in DECISIONS.md, write tests
> for all logic and state transitions, add audit + tenant scope by default. Don't stop
> unless a choice is irreversible, costs money, or is a product call only I can make.
> Hand me a git diff to review.
> ```

The **"Done ="** line is the discipline that makes this work. No "Done =", no build.

### Step 8 — Verify & review · after each slice

> Prompt:
> ```
> Verify Phase [N]/[ID] works end to end: run it, drive the real flow, show me the
> result — not just that tests pass. Then act as the Code Reviewer
> (groundwork/agents/code-reviewer.md) and the Simplicity Reviewer
> (groundwork/agents/simplicity-reviewer.md); fix what you find. Summarize changes and
> anything I should decide.
> ```

### Invoking an agent role — any time

The 19 personas in `groundwork/agents/` are experts you summon by name:

> Prompt: `Act as the Security Officer (groundwork/agents/security-officer.md) and threat-model this file-upload feature. Use groundwork/threat-models/ for the STRIDE method.`

Other favorites:
- `Act as the Performance Engineer…` when latency regresses.
- `Act as the Compliance Officer…` when personal data or a new jurisdiction appears.
- `Act as the CEO — Final Review…` for a go/no-go against the release gate before shipping.

See the full roster and when to use each in `groundwork/agents/README.md`.

---

## 7. Worked example — "Build a login + tenant-provisioning screen"

Let's make it concrete. You're on a public SaaS. You've done Steps 1–4. Now:

> Prompt:
> ```
> Build Phase 0 — Module M1 (Identity & Tenancy): a login screen + new-tenant
> provisioning.
> Source: knowledge-base/identity.md + groundwork/standards-kb/ +
>          groundwork/feature-blueprints/rbac-admin.md +
>          groundwork/feature-blueprints/onboarding-and-kyc.md.
> Done = a user can sign up, a new tenant + owner role is provisioned, they can log in
> with MFA, and every step is audit-logged and tenant-scoped. Include the sign-in
> history screen.
> Autonomy: follow CLAUDE.md, tests + audit + tenant scope by default. Git diff please.
> ```

**Which Groundwork files the AI reads (and why):**

| File | What it contributes |
|---|---|
| `standards-kb/01-architecture-multitenancy.md` | Row-level tenant isolation; how `tenant_id` scopes every query |
| `standards-kb/04-security.md` | Password/MFA handling, session security, input validation |
| `standards-kb/20-logging-audit-and-traceability.md` | Which events to emit; how to mask PII in logs |
| `standards-kb/13-i18n-accessibility.md` | Keyboard + screen-reader states for the forms |
| `feature-blueprints/rbac-admin.md` | The owner/admin/member role model to provision |
| `feature-blueprints/onboarding-and-kyc.md` | Sign-up + tenant-provisioning flow spec |
| `admin-kb/login-history-and-sessions.md` | The sign-in history / active-sessions screen spec |
| `admin-kb/security-center.md` | Where MFA status + suspicious-login surfaces live |
| `design-system-kb/` + `component-library-kb/` | Token-driven form, button, and input components |
| `threat-models/` (auth) | STRIDE checks: credential stuffing, session fixation, enumeration |

**What it produces (by default, because the rules said so):**

1. Migrations with **row-level tenant scoping** — every row carries `tenant_id`, and
   queries filter on it (RLS or equivalent).
2. **MFA** on login, not "later."
3. **Audit events** for signup, tenant-provision, login success/failure, MFA
   enrollment — structured and PII-masked.
4. A **sign-in history / active-sessions** screen (from `admin-kb/`).
5. **Accessible** forms: labels, focus order, error announcements, keyboard-only paths.
6. **Tests** for the state transitions (unverified → verified, no-MFA → MFA-enrolled,
   provisioning success/failure).
7. Assumptions (e.g. "chose TOTP over SMS for MFA") logged in `DECISIONS.md`.

Then run **Step 8** to verify it end-to-end and review it. That's a real,
enterprise-shaped login — from one prompt — because the AI read the bar first.

---

## 8. Troubleshooting & FAQ

**The AI is ignoring the rules.**
Two fixes, in order: (1) **@-mention the file** directly in your prompt —
`@groundwork/standards-kb/04-security.md` — which forces it into context. (2) Make your
rules file more forceful: short imperative sentences ("Tenant scope is mandatory," not
"consider tenant scoping"), and for Cursor set `alwaysApply: true`. If it still drifts,
your session context may be full — start a fresh session; the rules file reloads.

**Context feels too big / the AI seems overwhelmed.**
That's *exactly* why Groundwork is dozens of small files instead of one mega-doc. Don't
ask it to "read all of groundwork." Reference the two or three files a task needs. The
kit is designed for on-demand retrieval.

**Groundwork's guidance conflicts with my project's needs.**
Your project wins. Precedence is: **your `CLAUDE.md`/`AGENTS.md` and `DECISIONS.md` >
`groundwork/`.** Record the override in `DECISIONS.md` ("We deviate from
standards-kb/03 budget X because Y") and the AI will respect it going forward.

**How do I update Groundwork?**
Re-copy the folder (see Section 9). Check `groundwork/KIT-REPORT.md` for what changed.

**Does all this slow the AI down?**
No. The rules file is tiny, and the AI only opens the specific KB files a task needs —
it does not load the whole kit every prompt. You trade a few extra file reads for code
that doesn't need three rounds of "you forgot the audit log."

**Do I need every folder?**
No — see Section 5. Keep the core, drop the 🧩 deep-dives you don't need.

**Can my whole team use it?**
Yes. Commit the rules files (`CLAUDE.md`, `.cursor/rules/`, `.github/`, etc.) and the
`groundwork/` folder to your repo. Everyone's AI tool picks it up automatically.

**It's writing too much / over-engineering.**
Invoke the Lean Code roles: "Act as the Simplicity Reviewer
(`groundwork/agents/simplicity-reviewer.md`) using `groundwork/engineering-doctrine/`;
give me a delete/simplify list." Groundwork raises the *quality* bar, not the *quantity*.

---

## 9. Keeping it updated

Groundwork evolves — new blueprints, tightened standards, more agents. Pulling a newer
version into an existing project is a folder swap.

### Update in place

```bash
# get the latest kit
git clone <groundwork-repo-url> /tmp/groundwork-latest    # or unzip the new release

# back up your current copy just in case
mv groundwork groundwork.bak

# drop the new one in
cp -R /tmp/groundwork-latest groundwork
```

Then re-apply any **local edits** you'd made (e.g. deleted 🧩 modules — just delete
them again, or better, record the deletions in `DECISIONS.md` so updates don't
re-introduce them). Your project-level files (`CLAUDE.md`, `DECISIONS.md`,
`DELIVERY-PLAN.md`, `knowledge-base/`) live **outside** `groundwork/` and are untouched
by the swap.

### What changed?

Open `groundwork/KIT-REPORT.md` — it's the kit's changelog and inventory (file counts
per category, what was added, what's deliberately left out). Diff it against your backup
to see exactly what moved:

```bash
diff groundwork.bak/KIT-REPORT.md groundwork/KIT-REPORT.md
```

If you version-control your project (you should), commit the update as its own commit
("chore: update groundwork to <version>") so you can review the diff and roll back
cleanly. Once you're happy:

```bash
rm -rf groundwork.bak
```

That's it. Same rules files, newer brain.

---

### You're ready

- **Quick start?** Install (§3), turn on your tool (§4), run the Build-one-slice prompt
  (§6, Step 6).
- **Going deep?** Tailor the modules (§5), run the full 6-step loop, and invoke agent
  roles as you go.

Groundwork does the remembering so you can keep vibing — just with seatbelts on. Happy
building.
