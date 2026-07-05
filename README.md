# Groundwork

**An enterprise-grade knowledge base + agent library that makes AI "vibe coding" produce production-grade software — in any AI coding tool.**

You describe features in plain English; your AI assistant (Claude Code, Cursor, Copilot, Windsurf, Cline…) reads Groundwork first and builds to an enterprise standard automatically — multi-tenancy, security, audit logging, compliance, performance, accessibility, and clean architecture, baked in from line one.

> **New here? Read [TUTORIAL.md](TUTORIAL.md).** It walks you through install, per-tool setup, what to keep/drop, and the daily workflow.

## Why it exists

Vibe coding is fast but drifts: the AI guesses architecture, forgets tenancy and audit trails, and builds every feature differently. Groundwork is the **rulebook the AI reads before it writes code**, so the output is consistent and enterprise-ready — without you re-explaining your standards every session.

## What's inside

| Folder | What it gives the AI |
|---|---|
| `standards-kb/` | 20 NFR domains + release gate — the **bar** every feature must meet |
| `feature-blueprints/` | 15 buildable feature specs (report builder, RBAC, billing, workflow engine…) |
| `admin-kb/` | The full Admin Area (logs, security center, consent/DSR, API analytics…) |
| `design-system-kb/` + `component-library-kb/` | Design tokens + a 36-component catalog with recipes |
| `performance-kb/` · `events-kb/` · `sre-kb/` | Deep performance, async patterns, on-call runbooks |
| `compliance-kb/` · `threat-models/` | GDPR/DPDP/SOC2/ISO/PCI/HIPAA + STRIDE threat models |
| `engineering-doctrine/` · `finops-kb/` · `data-kb/` | Lean-code doctrine, per-tenant cost model, analytics layer |
| `api-kb/` · `mcp-kb/` | Connect via OpenAPI or expose the app to AI agents via MCP |
| `agents/` | 19 expert roles you invoke (Architect, Security Officer, QA…) |
| `templates/` | Blank project files (CLAUDE.md, DECISIONS, DELIVERY-PLAN…) |
| `PROMPT-PLAYBOOK.md` | The 6-step build workflow with copy-paste prompts |

Full inventory: [KIT-REPORT.md](KIT-REPORT.md). Orientation: [START-HERE.md](START-HERE.md).

## Quick start

```bash
# 1. Copy Groundwork into your project as a folder named `groundwork/`
git clone <this-repo-url> groundwork    # or download the ZIP and rename

# 2. Turn it on in your tool (pick yours — details in TUTORIAL.md §4):
#    Claude Code → CLAUDE.md points to groundwork/
#    Cursor      → cp groundwork/tool-configs/cursor.mdc .cursor/rules/groundwork.mdc
#    Copilot     → cp groundwork/tool-configs/copilot-instructions.md .github/copilot-instructions.md
#    Windsurf    → cp groundwork/tool-configs/windsurfrules.md .windsurfrules
#    Cline/Roo   → cp groundwork/tool-configs/clinerules.md .clinerules
#    Anything    → the root AGENTS.md

# 3. Build. Use groundwork/PROMPT-PLAYBOOK.md prompts. The AI reads the kit on demand.
```

## Supported tools

Claude Code · Cursor · GitHub Copilot · Windsurf · Cline / Roo Code · and any tool that reads `AGENTS.md`. It's plain markdown — tool-agnostic by design.

## Keep only what you need

Every folder is independent. A small internal tool doesn't need `compliance-kb/` or `mcp-kb/`; a fintech app does. See [TUTORIAL.md](TUTORIAL.md) §5 for the enable/disable decision table.

## License

Released under the **[MIT License](LICENSE)** — © 2026 AI Code Factory Hub.
Free to use, copy, modify, and distribute, including commercially. No warranty.
