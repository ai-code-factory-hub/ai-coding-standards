# Agent Roles — Index

Reusable, **project-agnostic** expert personas for the enterprise starter kit. Ask the AI to act as any of these on **any** project. Each role owns specific domains from [../standards-kb/](../standards-kb/), consumes the [../design-system-kb/](../design-system-kb/) where relevant, and plugs into the 6-step build sequence in [../PROMPT-PLAYBOOK.md](../PROMPT-PLAYBOOK.md).

## The 19 roles

| # | Role | One-liner | Invoke when… |
|---|------|-----------|--------------|
| 1 | [Solutions Architect](solutions-architect.md) | Designs the frame: tenancy, stack, boundaries, contracts. | **Playbook Step 3** — locking architecture; producing CLAUDE.md + DECISIONS.md. |
| 2 | [Business Analyst](business-analyst.md) | Builds the requirements KB and gap analysis from source docs. | **Playbook Steps 1-2** — ingesting sources and finding the gaps at kickoff. |
| 3 | [Senior Full-Stack Developer](senior-fullstack-developer.md) | Builds production code in the selected stack; tests + audit + tenant scope by default. | **Playbook Step 6** — implementing features, fixes, and migrations. |
| 4 | [UI/UX Designer](uiux-designer.md) | Responsive-first, WCAG 2.2 AA, token-driven; designs all states. | **Playbook Steps 3-6** — defining flows, components, and states. |
| 5 | [Security Officer](security-officer.md) | Adversarial threat-modeler; owns security, compliance, AI governance. | **Playbook Steps 3, 6-8** — threat-modeling and security review. |
| 6 | [DevOps Engineer](devops-engineer.md) | Pipeline-as-product: CI/CD, IaC, observability, safe reversible deploys. | **Pipeline setup** and any deploy/env/monitoring concern. |
| 7 | [QA Verifier](qa-verifier.md) | Verifies behavior against requirements end-to-end; produces evidence. | **Playbook Step 8** — verifying a built, reviewed feature. |
| 8 | [Code Reviewer](code-reviewer.md) | Reviews the diff for correctness, security, and maintainability. | **Playbook Step 8** — reviewing any non-trivial change before merge. |
| 9 | [Project Manager](project-manager.md) | Sequences a de-risked, shippable delivery plan; riskiest-assumption-first. | **Playbook Step 4** — planning and re-baselining delivery. |
| 10 | [CEO — Plan Review](ceo-plan-review.md) | Pressure-tests the PLAN: scope, sequencing, ROI, riskiest assumption. | **Before build** — go/no-go on the plan. |
| 11 | [CEO — Final Review](ceo-final-review.md) | The ship gate: does it meet the bar and the 99 release gate? | **Before ship** — go/no-go on the release. |
| 12 | [Performance Engineer](performance-engineer.md) | Profiles, sets/enforces budgets, kills N+1/pooling/hot-path issues, load-tests. | **When latency/throughput/cost regresses** or before a scale event. |
| 13 | [Simplicity Reviewer](simplicity-reviewer.md) | Lean-Code reviewer: hunts over-engineering; outputs a delete/simplify list. | **Playbook Step 8** — after a feature is built, before merge. |
| 14 | [Compliance Officer (DPO)](compliance-officer.md) | Maps regimes (GDPR/DPDP/SOC2/ISO/PCI/HIPAA); ensures DSR/consent/retention; control→evidence map. | **When personal data, a new jurisdiction, or an audit** is in play. |
| 15 | [Data & Analytics Engineer](data-analytics-engineer.md) | Governed semantic layer, metric definitions (MRR/churn), marts. | **For reporting, dashboards, or churn analytics.** |
| 16 | [Feature Architect](feature-architect.md) | Turns a needed feature into a buildable spec (data model + screens + rules) from a blueprint. | **At feature kickoff** — Playbook Steps 3-4. |
| 17 | [Admin Auditor](admin-auditor.md) | Observability & admin-surface owner: every feature emits the right logs, every log class has a masked, audited admin screen. | **At feature kickoff and pre-merge** — "what does this log + which admin screen shows it?"; audit-readiness; wiring the Admin Area. |
| 18 | [API Platform Engineer](api-platform-engineer.md) | Spec-first OpenAPI APIs, gateway, rate limits/quotas, versioning/deprecation, developer portal + SDKs, access analytics. | **When exposing or evolving a public/partner API**, wiring rate limits/quotas, or standing up a developer portal. |
| 19 | [MCP Server Engineer](mcp-server-engineer.md) | Builds/secures MCP servers so approved AI agents can use the app: per-tool authz, tenant isolation, injection defense, tool logging + limits, tool governance. | **When making the app AI-agent-connectable** or reviewing an MCP server. |

## Typical flow (mapped to [../PROMPT-PLAYBOOK.md](../PROMPT-PLAYBOOK.md))

1. **Steps 1-2 · Ingest + Gap** → Business Analyst builds the requirements KB and gap analysis.
2. **Step 3 · Architecture** → Solutions Architect locks CLAUDE.md + DECISIONS.md; Security Officer threat-models; UI/UX Designer sets the design language.
3. **Step 4 · Plan** → Project Manager produces DELIVERY-PLAN.md.
4. **Before build** → **CEO — Plan Review** pressure-tests the plan (go / no-go / conditions).
5. **Step 6 · Build** → Senior Full-Stack Developer implements; UI/UX Designer specs states; DevOps Engineer wires the pipeline.
6. **Step 8 · Verify** → Code Reviewer reviews the diff; QA Verifier verifies against requirements; Security Officer reviews findings.
7. **Before ship** → **CEO — Final Review** enforces the [../standards-kb/99-master-release-gate.md](../standards-kb/99-master-release-gate.md) and calls go / no-go.

---

Each role file follows the same structure: Persona · When to invoke · Owns these standards · Operating principles · Checklist/heuristics · Output · Guardrails. Roles reference the kit by relative path so they stay reusable across projects — swap in your project's CLAUDE.md, DECISIONS.md, and requirements KB, and the personas apply unchanged.
