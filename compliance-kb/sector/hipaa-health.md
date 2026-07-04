# HIPAA (US Health Data)

Scope: a sector-specific regime layer on top of [standards-kb](../../standards-kb/) Domains 04 (Security) & 09 (Compliance & Privacy). The US **Health Insurance Portability and Accountability Act** and its regulations (Privacy, Security, and Breach Notification Rules, as amended by HITECH) govern **Protected Health Information (PHI)** held by *covered entities* (providers, plans, clearinghouses) and their *business associates*.

> **Engineering guidance, not legal or audit advice.** HIPAA obligations, your status as covered entity vs business associate, and Business Associate Agreement terms are legal determinations owned by counsel and compliance. This file helps engineers implement and evidence the technical and administrative safeguards HIPAA requires. It does not make a product "HIPAA compliant" on its own.

See also: [compliance-kb README](../README.md) · [controls matrix](../controls-matrix.md)

---

## Who's covered, and PHI

- **Covered Entity (CE):** health providers, plans, clearinghouses.
- **Business Associate (BA):** a vendor that creates/receives/maintains/transmits PHI **on behalf of** a CE — **most B2B health SaaS is a BA** and is directly liable under HIPAA.
- **PHI** = individually identifiable health information (diagnoses, treatment, health-related identifiers) tied to any of the **18 identifiers** (name, dates, MRN, email, IP, device IDs, biometrics, etc.). **ePHI** is PHI in electronic form.
- **De-identified data** (Safe Harbor: remove all 18 identifiers, or Expert Determination) is **not** PHI and falls outside these rules.

## The three rules

- **Privacy Rule** — governs *uses and disclosures* of PHI; requires **minimum necessary**, individual rights (access, amendment, accounting of disclosures), and a Notice of Privacy Practices.
- **Security Rule** — requires **administrative, physical, and technical safeguards** for **ePHI** specifically, driven by a mandatory **risk analysis**.
- **Breach Notification Rule** — on a breach of unsecured PHI, notify affected individuals and HHS (and, for large breaches, media) **without unreasonable delay and within 60 days**. Properly **encrypted** PHI that is breached may qualify for **safe harbor** (no notification required).

## Required safeguards (Security Rule)

Each safeguard is *Required* or *Addressable* (addressable = implement, or document why an equivalent alternative is reasonable — **not** optional to ignore).

**Administrative safeguards**
- **[MUST]** Conduct and maintain a **Security Risk Analysis** and a risk-management plan (the linchpin of the Security Rule).
- **[MUST]** Assign a **Security Official**; run **workforce security** (authorization, clearance, termination/deprovisioning), **security awareness training**, an **incident-response** procedure, a **contingency plan** (backups, DR, emergency-mode operation), and periodic **evaluation**.

**Physical safeguards**
- **[MUST]** Facility access controls, workstation use/security, and **device & media controls** (disposal, re-use, media movement). Largely inherited from a HIPAA-ready cloud provider under **shared responsibility** — but you must configure and document it.

**Technical safeguards**
- **[MUST]** **Access control** — unique user IDs, emergency access, automatic logoff, and **encryption/decryption** of ePHI.
- **[MUST]** **Audit controls** — hardware/software mechanisms that **record and examine activity** in systems with ePHI.
- **[MUST]** **Integrity** controls — protect ePHI from improper alteration/destruction.
- **[MUST]** **Person/entity authentication** and **transmission security** (encryption in transit, integrity on the wire).

## Business Associate Agreements (BAAs)

- **[MUST]** Sign a **BAA with every upstream and downstream party** in the PHI chain: with your customer CEs, and with **every subcontractor/sub-processor that touches ePHI** (cloud host, email/SMS, logging/APM, backup, analytics, AI/LLM provider).
- **[MUST NOT]** Route ePHI through any vendor that will not sign a BAA — including LLM/AI APIs unless covered by a signed BAA and configured for no-training/no-retention.
- **[SHOULD]** Maintain a **PHI data-flow map** and vendor/BAA register (align with the sub-processor governance in [Domain 09](../../standards-kb/09-compliance-privacy.md)).

## Minimum necessary

- **[MUST]** Limit uses, disclosures, and requests of PHI to the **minimum necessary** for the purpose. In engineering terms this is **least-privilege + purpose-scoped access**: role/attribute-based access so a user (or service, or query) sees only the PHI needed for the task — enforced server-side per object (see [Domain 04](../../standards-kb/04-security.md)).

## Audit controls & encryption (the two engineers own most)

- **[MUST]** **Audit controls:** immutable, append-only logging of **every access to ePHI** — read, create, amend, export, delete — capturing who/what/when/where + before→after, retained and reviewable. This is the [Domain 04](../../standards-kb/04-security.md) audit trail applied to PHI, and it is what makes accounting-of-disclosures and breach investigation possible.
- **[MUST]** **Encryption:** ePHI encrypted **in transit (TLS 1.2+)** and **at rest** (DB, backups, files, logs), with **field-level encryption** for the most sensitive fields and KMS-managed keys. Encryption to the recognized standard is also your path to **breach safe harbor**.

## Control → standards-kb domain → evidence artifact

| HIPAA control | standards-kb domain that implements it | Evidence artifact |
| --- | --- | --- |
| Access control, unique IDs, auto-logoff (§164.312(a)) | [04-security](../../standards-kb/04-security.md) | IdP config, RBAC/ABAC matrix, session-timeout config |
| Minimum necessary / need-to-know | [04-security](../../standards-kb/04-security.md) | Purpose-scoped role definitions, per-object authz tests |
| Audit controls (§164.312(b)) | [04-security](../../standards-kb/04-security.md) / [07-observability](../../standards-kb/07-observability-aiops.md) | Immutable ePHI-access log samples, review records |
| Integrity (§164.312(c)) | [06-data-management](../../standards-kb/06-data-management.md) / [08-testing-qa](../../standards-kb/08-testing-qa.md) | Checksums/versioning, validation tests, tamper-evidence |
| Transmission security & encryption (§164.312(e), (a)(2)(iv)) | [04-security](../../standards-kb/04-security.md) / [06-data-management](../../standards-kb/06-data-management.md) | TLS scan, at-rest + field-level encryption config, KMS key policy |
| Contingency plan / backup & DR (§164.308(a)(7)) | [05-reliability](../../standards-kb/05-reliability-resilience.md) / [06-data-management](../../standards-kb/06-data-management.md) | Backup config, **restore-test record**, DR runbook, emergency-mode plan |
| Risk analysis & management (§164.308(a)(1)) | [09-compliance-privacy](../../standards-kb/09-compliance-privacy.md) | Security Risk Analysis document, remediation tracker |
| Incident response & breach notification | [04-security](../../standards-kb/04-security.md) / [09-compliance-privacy](../../standards-kb/09-compliance-privacy.md) | IR plan, breach-assessment procedure, notification templates |
| Workforce security & training (§164.308(a)(3),(5)) | [14-devex-documentation](../../standards-kb/14-devex-documentation.md) / [09](../../standards-kb/09-compliance-privacy.md) | Onboarding/offboarding tickets, training completion records |
| BAAs / sub-processor governance | [09-compliance-privacy](../../standards-kb/09-compliance-privacy.md) | Signed BAAs, vendor/BAA register, PHI data-flow map |
| Individual rights (access, amendment, accounting) | [09-compliance-privacy](../../standards-kb/09-compliance-privacy.md) | Data-subject-rights features, disclosure-accounting export |

## Evidence a HIPAA assessment asks for

- The **Security Risk Analysis** and risk-management plan (auditors ask for this first).
- Written **policies & procedures** for each safeguard, plus the Notice of Privacy Practices (for CEs).
- **Signed BAAs** for every party in the PHI chain and the PHI data-flow map.
- **Audit-log samples** proving ePHI access is recorded; **encryption configuration** proving in-transit + at-rest coverage.
- **Contingency evidence**: backups + a successful restore test; workforce training completion; an incident-response record.

## Notes

- **[SHOULD]** HIPAA has **no certification body** — no one issues a "HIPAA certificate." Assurance comes from your own risk analysis + documented safeguards, often evidenced through a **SOC 2 report scoped to include HIPAA** (see [soc2.md](../attestations/soc2.md)) and continuous-compliance tooling with a HIPAA control mapping.

## Acceptance checklist

- [ ] Status (covered entity vs business associate) determined; PHI/ePHI inventory and 18-identifier data map exist.
- [ ] **Security Risk Analysis** completed and maintained, with a risk-management plan.
- [ ] **BAA signed with every party** touching ePHI (customers, cloud, email/SMS, logging, backup, AI/LLM); no ePHI to a non-BAA vendor.
- [ ] Minimum-necessary enforced as purpose-scoped, per-object, least-privilege access.
- [ ] **Audit controls** log every ePHI access (read/create/amend/export/delete), immutable and reviewable.
- [ ] ePHI **encrypted in transit and at rest** (+ field-level for sensitive fields, KMS keys) — breach safe-harbor eligible.
- [ ] Contingency plan: backups + **successful restore test** + DR + emergency-mode operation.
- [ ] Security Official assigned; workforce onboarding/offboarding + security-awareness training in place.
- [ ] **Breach-notification** procedure ready (individuals + HHS, within 60 days) with templates.
- [ ] Individual-rights features (access, amendment, accounting of disclosures) shipped.
