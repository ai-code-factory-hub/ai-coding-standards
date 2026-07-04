# SOC 2 (Trust Services Criteria)

Scope: a regime-specific attestation layer on top of the platform controls in [standards-kb](../../standards-kb/) Domains 04 (Security) & 09 (Compliance & Privacy). SOC 2 is an AICPA attestation, performed by a licensed CPA firm, that reports on how well your service organization's controls meet the Trust Services Criteria (TSC).

> **Engineering guidance, not legal or audit advice.** This file helps engineers build and evidence controls that map cleanly to SOC 2. Your scope, criteria selection, control language, and audit outcome are decisions owned by your auditor, security lead, and legal/compliance function. Nothing here substitutes for a signed engagement with a CPA firm.

See also: [compliance-kb README](../README.md) · [controls matrix](../controls-matrix.md)

---

## The 5 Trust Services Criteria

Every SOC 2 report covers **Security** (the mandatory "Common Criteria", CC-series). The other four are optional and included only if you commit to them.

| TSC | What it attests | Include when |
| --- | --- | --- |
| **Security** (required) | System is protected against unauthorized access (physical & logical). The CC1–CC9 "Common Criteria" underpin everything. | Always. |
| **Availability** | System is available for operation and use as committed (SLOs, DR, capacity). | You make uptime commitments to customers. |
| **Processing Integrity** | Processing is complete, valid, accurate, timely, and authorized. | Correctness of processing is part of your value (billing, calculations, results). |
| **Confidentiality** | Information designated confidential is protected (encryption, access limits, disposal). | You hold customer-confidential (non-personal) data — IP, contracts, configs. |
| **Privacy** | Personal information is collected, used, retained, disclosed, and disposed of per your notice and criteria (GAPP-aligned). | You process personal data and want privacy in scope (often deferred to GDPR/DPDP programs instead). |

- **[MUST]** Put **Security** in scope; it is non-negotiable.
- **[SHOULD]** Add **Availability** and **Confidentiality** early — they are cheap if Domains 04/05/06 are already implemented.
- **[SHOULD]** Scope **Privacy** deliberately; much of it overlaps [Domain 09](../../standards-kb/09-compliance-privacy.md) and dedicated privacy law, so avoid double-attesting the same controls without reason.

## Type I vs Type II

| | **Type I** | **Type II** |
| --- | --- | --- |
| Question answered | Are controls *designed* suitably at a point in time? | Did controls *operate effectively* over a period? |
| Evidence window | A single date | An observation period, typically **3–12 months** |
| Auditor tests | Design only | Design **and** operating effectiveness (samples across the period) |
| Buyer trust | Low–moderate ("day-one snapshot") | High — the report enterprise buyers actually ask for |

- **[SHOULD]** Treat **Type I as a milestone, not the goal.** Use it to prove readiness, then run a Type II observation window (start with 3 months, move to a recurring 12-month cadence).
- **[MUST]** For Type II, controls must produce **evidence continuously across the whole window** — a control you configured the week before fieldwork will fail sample testing.

## Common controls (and where they live in standards-kb)

- **Access control** — SSO/MFA, RBAC/ABAC, deny-by-default, joiner/mover/leaver process, **quarterly access reviews**, least-privilege for infra/DB. Built per [Domain 04](../../standards-kb/04-security.md).
- **Change management** — all production change flows through **CI/CD with peer review, required status checks, protected branches, and traceability from ticket → PR → deploy**. Built per [Domain 12](../../standards-kb/12-devops-cicd.md); tested per [Domain 08](../../standards-kb/08-testing-qa.md).
- **Logging & monitoring** — immutable audit trail, centralized logs, alerting, and an on-call path. Built per [Domain 04](../../standards-kb/04-security.md) (audit trail) and [Domain 07](../../standards-kb/07-observability-aiops.md).
- **Vendor / sub-processor risk** — inventory, risk tiering, DPAs, annual review of critical vendors' own SOC 2 reports. Built per [Domain 09](../../standards-kb/09-compliance-privacy.md).
- **Incident response** — documented IR plan, severity levels, roles, comms, and **post-incident reviews**; periodic tabletop exercise. Built per [Domain 04](../../standards-kb/04-security.md) / [Domain 05](../../standards-kb/05-reliability-resilience.md).
- **BCP / DR** — backups, tested restores, RPO/RTO targets, failover runbooks. Built per [Domain 05](../../standards-kb/05-reliability-resilience.md) and [Domain 06](../../standards-kb/06-data-management.md).

## Control → standards-kb domain → evidence artifact

| Control (SOC 2 CC ref) | standards-kb domain that implements it | Evidence artifact auditors request |
| --- | --- | --- |
| Logical access, MFA, RBAC (CC6.1–6.3) | [04-security](../../standards-kb/04-security.md) | IdP config export, MFA-enforcement screenshot, role matrix |
| Periodic access review (CC6.2) | [04-security](../../standards-kb/04-security.md) | Signed-off quarterly access-review record per system |
| Encryption in transit & at rest (CC6.7) | [04-security](../../standards-kb/04-security.md) / [06-data-management](../../standards-kb/06-data-management.md) | TLS scan, KMS key policy, at-rest encryption config |
| Change management (CC8.1) | [12-devops-cicd](../../standards-kb/12-devops-cicd.md) | PR with approval + passing checks; ticket→PR→deploy trace; branch-protection settings |
| Testing before release (CC8.1) | [08-testing-qa](../../standards-kb/08-testing-qa.md) | CI run logs, coverage report, release gate record |
| Audit logging & monitoring (CC7.2) | [04-security](../../standards-kb/04-security.md) / [07-observability](../../standards-kb/07-observability-aiops.md) | Immutable log samples, alert config, alert-fired examples |
| Incident response (CC7.3–7.5) | [04-security](../../standards-kb/04-security.md) | IR policy, an incident ticket with timeline + post-mortem, tabletop notes |
| Vendor risk (CC9.2) | [09-compliance-privacy](../../standards-kb/09-compliance-privacy.md) | Vendor inventory, DPAs, subservice-org SOC 2 review log |
| Availability / DR (A1.1–A1.3) | [05-reliability-resilience](../../standards-kb/05-reliability-resilience.md) | SLO dashboard, backup config, **restore-test record**, DR runbook |
| Processing integrity (PI1.x) | [08-testing-qa](../../standards-kb/08-testing-qa.md) / [06-data-management](../../standards-kb/06-data-management.md) | Input-validation tests, reconciliation jobs, data-quality checks |
| Confidentiality & disposal (C1.1–1.2) | [06-data-management](../../standards-kb/06-data-management.md) | Data-classification policy, retention/disposal job logs |
| Onboarding/offboarding (CC1.4) | [14-devex-documentation](../../standards-kb/14-devex-documentation.md) | HR checklist, deprovisioning tickets |

## Evidence auditors ask for (the "evidence pack")

- **Policies** — Information Security, Access Control, Change Management, Incident Response, BCP/DR, Vendor Management, Risk Assessment, Data Classification & Retention (versioned, dated, approved).
- **Population + samples** — a complete list (e.g. all production changes in the period) so the auditor can pull a random sample. Your tooling must be able to produce the *full population*, not curated examples.
- **Operating evidence** — access-review sign-offs, PR approvals, CI runs, alert firings, incident tickets, restore-test results, tabletop notes, onboarding/offboarding tickets — **each timestamped inside the observation window**.
- **Org evidence** — org chart, background-check confirmations, security-training completion, risk assessment.

## How continuous-compliance tooling helps

- **[SHOULD]** Adopt a continuous-compliance platform (Vanta, Drata, Secureframe, or equivalent) to:
  - **auto-collect evidence** from cloud, IdP, MDM, ticketing, and CI via read-only integrations (turns point-in-time screenshots into continuous checks);
  - **monitor drift** (e.g. an S3 bucket goes public, MFA disabled, a check fails) and alert before it becomes an audit exception;
  - **map one control to many frameworks** (SOC 2 / ISO 27001 / HIPAA) so evidence is collected once;
  - **maintain the population** automatically, which is what makes Type II sampling painless.
- **[MUST NOT]** Treat the tool as the control. The tool *evidences* the controls that Domains 04/05/06/08/12 actually implement; an empty dashboard over missing engineering controls will still fail.

## Acceptance checklist

- [ ] Security (Common Criteria) in scope; optional TSC (Availability/Confidentiality/Processing Integrity/Privacy) chosen deliberately and justified.
- [ ] Type I achieved as a readiness milestone; Type II observation window running (≥3 months, cadence to 12).
- [ ] Access control, quarterly access reviews, and joiner/mover/leaver evidenced from the IdP.
- [ ] 100% of production change flows through peer-reviewed CI/CD with traceable ticket→PR→deploy.
- [ ] Immutable audit logging + monitoring + alerting live, with fired-alert examples in the window.
- [ ] IR plan tested (tabletop) and at least one incident documented with a post-mortem.
- [ ] BCP/DR with a **successful restore-test record** inside the period; RPO/RTO stated.
- [ ] Vendor inventory + DPAs + annual subservice-org SOC 2 review.
- [ ] Full evidence populations producible (changes, accesses) — not curated samples.
- [ ] Continuous-compliance tooling collecting evidence and monitoring drift, mapped to the chosen TSC.
