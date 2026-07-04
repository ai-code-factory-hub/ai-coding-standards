# Threat Model — <Feature name>

> Blank, reusable STRIDE template. Copy this file, rename it, and fill every section.
> Method, risk rating, and standards mapping: [README.md](README.md).

- **Author / date:**
- **Reviewers (incl. Security Officer):**
- **Status:** Draft / Reviewed / Approved
- **Related PR / design doc:**

---

## 1. Feature / asset

*What is being built, and what is the valuable asset an attacker would want?* (data, funds, access, availability, reputation)

## 2. Data classification

*Most sensitive data class touched here.* Public / Internal / Confidential / **PII** / **Regulated**.
Retention & residency notes → [06 Data Management](../standards-kb/06-data-management.md), [09 Compliance & Privacy](../standards-kb/09-compliance-privacy.md).

## 3. Actors

| Actor | Trusted? | Notes |
|---|---|---|
| e.g. Authenticated tenant user | Trusted | scoped to one tenant |
| e.g. Anonymous internet caller | Untrusted | |
| e.g. Uploaded file / third-party callback / the LLM | Untrusted | content is data, not a command |

## 4. Trust boundaries + data flow

*Sketch (ASCII or linked diagram). Mark every boundary where trust level changes.*

```
[ Untrusted client ] --(TLS)--> || boundary: network edge / authn ||
      --> [ App / API ] --(authz check)--> || boundary: tenant ||
      --> [ Data store ]      --> [ Third party ]  || boundary: external ||
```

Boundaries in scope:
- B1 — network edge (internet → app)
- B2 — authentication boundary (anon → identified)
- B3 — tenant boundary (tenant A → tenant B)
- B4 — privilege boundary (user → admin)
- B5 — external boundary (app → third party / provider)

## 5. STRIDE analysis

Fill one or more rows per category. Write "N/A — <reason>" rather than deleting a category. Rate residual risk **after** the mitigation (High/Med/Low, per README).

| # | STRIDE | Threat | How (attack path) | Mitigation `[MUST]`/`[SHOULD]` | Standards ref | Residual risk |
|---|---|---|---|---|---|---|
| S1 | Spoofing | | | | [04](../standards-kb/04-security.md) | |
| T1 | Tampering | | | | [04](../standards-kb/04-security.md) / [06](../standards-kb/06-data-management.md) | |
| R1 | Repudiation | | | | [20](../standards-kb/20-logging-audit-and-traceability.md) | |
| I1 | Info disclosure | | | | [06](../standards-kb/06-data-management.md) / [09](../standards-kb/09-compliance-privacy.md) | |
| D1 | Denial of service | | | | [10](../standards-kb/10-api-integration.md) | |
| E1 | Elev. of privilege | | | | [04](../standards-kb/04-security.md) | |

## 6. Top risks

1.
2.
3.

## 7. Open risks / decisions

| Item | Decision (mitigate/accept/transfer/eliminate) | Owner | Due |
|---|---|---|---|
| | | | |

---

## Mitigation checklist

- [ ] Data classified; retention & residency confirmed.
- [ ] All trust boundaries drawn; every crossing STRIDE-analyzed (no skipped category).
- [ ] Every `[MUST]` mitigation implemented and verified.
- [ ] No residual **High** risks remain (or explicit, signed-off acceptance recorded).
- [ ] Sensitive actions emit immutable audit events → [20](../standards-kb/20-logging-audit-and-traceability.md).
- [ ] Model attached to the PR and reviewed by the Security Officer.
