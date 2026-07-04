# Threat Models · Library

A project-agnostic library of reusable **STRIDE threat-model templates**, one per common feature type. Use these to reason about *what can go wrong* before you build a sensitive feature — and to prove you did.

- **Extends:** [standards-kb · 04 Security](../standards-kb/04-security.md)
- **Owner:** Security Officer agent
- **Companion template:** [threat-model-template.md](threat-model-template.md) (blank, copy per feature)

---

## What is threat modeling?

Threat modeling is a structured, repeatable way to answer four questions about a feature:

1. **What are we building?** (assets, actors, data flows, trust boundaries)
2. **What can go wrong?** (enumerate threats)
3. **What are we going to do about it?** (mitigations, mapped to standards)
4. **Did we do a good job?** (residual risk, open decisions, checklist)

It is a *design-time* activity. Finding a missing tenant filter on a whiteboard costs minutes; finding it in a breach report costs the company.

---

## STRIDE — the method

STRIDE is a mnemonic for six threat categories. For each element that crosses a trust boundary, ask: can it be **S-T-R-I-D-E**'d?

| Letter | Threat | Violates | Ask yourself… |
|---|---|---|---|
| **S** | **Spoofing** | Authenticity | Can an attacker pretend to be someone/something else? |
| **T** | **Tampering** | Integrity | Can data or code be modified in transit or at rest? |
| **R** | **Repudiation** | Non-repudiation | Can an actor deny having done something, with no proof? |
| **I** | **Information disclosure** | Confidentiality | Can data leak to someone who shouldn't see it? |
| **D** | **Denial of service** | Availability | Can the feature be made unavailable or ruinously slow? |
| **E** | **Elevation of privilege** | Authorization | Can an actor gain rights they shouldn't have? |

Each feature template below works the STRIDE table left to right: **Threat → How (attack) → Mitigation `[MUST]`/`[SHOULD]` → Standards ref → Residual risk.**

---

## How to run a session (30–60 min)

1. **Draw the picture.** Sketch the data-flow: actors → processes → data stores, and mark every **trust boundary** (network edge, tenant boundary, auth boundary, third-party edge). A boundary is anywhere the level of trust changes.
2. **Classify the data.** What's the most sensitive thing flowing here? (Public / Internal / Confidential / PII / Regulated.) This sets the stakes. See [09 Compliance & Privacy](../standards-kb/09-compliance-privacy.md).
3. **List actors** as **trusted** (authenticated users, internal services) and **untrusted** (anonymous internet, uploaded content, third-party callbacks, the LLM itself).
4. **Walk STRIDE** per boundary-crossing element. Fill the table. Don't skip a letter — write "N/A + why" if it truly doesn't apply.
5. **Rate each risk** (see below) and decide: mitigate / accept / transfer / eliminate.
6. **Record residual risk and open decisions.** Assign owners. File follow-ups.
7. **Attach the finished model to the feature's PR / design doc.**

Keep it lightweight. A filled table plus a data-flow sketch beats a 30-page document nobody reads.

---

## Risk rating — likelihood × impact

Score each threat's **likelihood** and **impact** 1–3, multiply, and bucket the product.

| Likelihood | | Impact | |
|---|---|---|---|
| 1 Unlikely | needs rare conditions / deep access | 1 Low | minor, recoverable, no data loss |
| 2 Possible | plausible for a motivated attacker | 2 Moderate | limited data / single tenant / degraded service |
| 3 Likely | easy, scriptable, or already seen in the wild | 3 Severe | mass data loss, cross-tenant breach, funds loss, regulated-data exposure |

| Score (L×I) | Rating | Action |
|---|---|---|
| 6–9 | **High** | Must mitigate before merge. Blocks release. |
| 3–4 | **Medium** | Mitigate this cycle; explicit sign-off to defer. |
| 1–2 | **Low** | Track; accept with rationale. |

*Residual risk* is the rating that remains **after** mitigations are applied.

---

## When to threat-model

- **Feature kickoff** — for any feature touching auth, money, PII/regulated data, files, multi-tenant data, admin power, external integrations, or an LLM. Do it while the design is still cheap to change.
- **Pre-merge** — a mandatory gate for those same sensitive features: reviewer confirms the STRIDE table's `[MUST]` mitigations are implemented and residual High risks are gone.
- **On material change** — new trust boundary, new third party, new data class, or a new actor.
- **Post-incident** — fold the new attack path back into the relevant template.

Low-risk features (static content, internal read-only dashboards over already-authorized data) don't need a full model — note the decision and move on.

---

## Mapping to standards-kb

| Threat area | Primary standard |
|---|---|
| AuthN/AuthZ, sessions, crypto, secrets, IDOR/BOLA | [04 Security](../standards-kb/04-security.md) |
| Data classification, encryption at rest, retention, RLS | [06 Data Management](../standards-kb/06-data-management.md) |
| PII, consent, data-subject rights, regulated data | [09 Compliance & Privacy](../standards-kb/09-compliance-privacy.md) |
| API authz, webhooks/HMAC, SSRF, rate limits | [10 API & Integration](../standards-kb/10-api-integration.md) |
| Prompt injection, tool authz, provider data flow | [15 AI/LLM Governance](../standards-kb/15-ai-llm-governance.md) |
| Immutable audit trail, non-repudiation, traceability | [20 Logging, Audit & Traceability](../standards-kb/20-logging-audit-and-traceability.md) |

Threat models **operationalize** these standards: the standard says "no IDOR"; the threat model shows *where* an IDOR could occur in this feature and *how* you closed it.

---

## Template index

| Feature type | Headline threats |
|---|---|
| [Authentication & Sessions](authentication-and-sessions.md) | credential stuffing, session fixation/hijack, MFA bypass, token theft, account enumeration |
| [File Upload](file-upload.md) | malware, path traversal, XXE, decompression bomb, SSRF via processing, stored XSS via filename, oversized upload |
| [API & Webhooks](api-and-webhooks.md) | IDOR/BOLA, broken auth, mass assignment, rate-limit abuse, webhook spoofing, SSRF, replay |
| [Payment & Billing](payment-and-billing.md) | amount tampering, replay, webhook forgery, PAN exposure, refund abuse, currency/rounding |
| [Multi-Tenant Data Access](multi-tenant-data-access.md) | cross-tenant leakage, missing tenant filter, RLS bypass, cache/index bleed, export crossing tenants |
| [Admin & Impersonation](admin-and-impersonation.md) | privilege escalation, impersonation abuse, audit-log tampering, mass data export |
| [AI / LLM Feature](ai-llm-feature.md) | prompt injection, context exfiltration, tool abuse, hallucinated facts, PII leakage to provider |
| [Search & Data Export](search-and-data-export.md) | query injection, over-broad results, expensive-query DoS, export leakage, PII in exports |

Start from the closest template, then delete what doesn't apply and add what's specific to your build. When a feature is novel, copy [threat-model-template.md](threat-model-template.md).
