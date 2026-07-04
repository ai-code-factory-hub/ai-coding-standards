# Controls Matrix — do it once, satisfy many

The master map: **one engineering control → which regimes require it → which standards-kb domain implements it.** Build each control once against the strictest applicable bar; claim it across every regime that needs it.

> **Engineering guidance, not legal advice.** "Required" markers below reflect the common engineering interpretation of each regime for a typical enterprise SaaS. Confirm exact applicability and thresholds with counsel.

**Legend:** ● required · ◐ required above a threshold / conditionally · ○ not directly mandated (still good practice). Regime files: [GDPR](privacy/gdpr.md) · [DPDP](privacy/dpdp-act-india.md) · [CCPA/CPRA](privacy/ccpa.md) · SOC 2 / ISO 27001 / PCI-DSS (`attestations/`, sibling task) · HIPAA (`sector/`, sibling task).

## Master matrix

| # | Control | GDPR | DPDP | CCPA/CPRA | SOC 2 | ISO 27001 | PCI-DSS | HIPAA | Implemented by (standards-kb) |
| --- | --- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | --- |
| 1 | **Consent capture + policy versioning** (granular, withdrawable, logged) | ● | ● | ◐ | ○ | ○ | ○ | ◐ | [09](../standards-kb/09-compliance-privacy.md) · [04](../standards-kb/04-security.md) (audit) |
| 2 | **Data-subject / consumer rights** (access, delete, portability, opt-out) | ● | ● | ● | ○ | ○ | ○ | ● | [09](../standards-kb/09-compliance-privacy.md) · [06](../standards-kb/06-data-management.md) |
| 3 | **RoPA / data inventory + classification** | ● | ● | ◐ | ● | ● | ○ | ● | [09](../standards-kb/09-compliance-privacy.md) · [06](../standards-kb/06-data-management.md) |
| 4 | **Encryption in transit (TLS 1.2+)** | ● | ● | ● | ● | ● | ● | ● | [04](../standards-kb/04-security.md) |
| 5 | **Encryption at rest + field-level for sensitive data (KMS)** | ● | ● | ● | ● | ● | ● | ● | [04](../standards-kb/04-security.md) · [06](../standards-kb/06-data-management.md) |
| 6 | **Immutable, tamper-evident audit log** | ● | ● | ● | ● | ● | ● | ● | [04](../standards-kb/04-security.md) · [07](../standards-kb/07-observability-aiops.md) |
| 7 | **Breach notification process** (regulator + subjects, within window) | ● | ● | ◐ | ● | ● | ● | ● | [09](../standards-kb/09-compliance-privacy.md) · [07](../standards-kb/07-observability-aiops.md) |
| 8 | **DPA / sub-processor register + BAA** | ● | ● | ● | ● | ● | ◐ | ● | [09](../standards-kb/09-compliance-privacy.md) |
| 9 | **Retention schedule + deletion propagation** | ● | ● | ● | ● | ● | ◐ | ● | [06](../standards-kb/06-data-management.md) · [09](../standards-kb/09-compliance-privacy.md) |
| 10 | **Periodic access review + least privilege** | ◐ | ◐ | ○ | ● | ● | ● | ● | [04](../standards-kb/04-security.md) |
| 11 | **Data residency / cross-border transfer controls** | ● | ● | ○ | ○ | ◐ | ◐ | ◐ | [06](../standards-kb/06-data-management.md) |
| 12 | **Versioned Privacy Policy + notice at collection** | ● | ● | ● | ○ | ○ | ○ | ● | [09](../standards-kb/09-compliance-privacy.md) · [11](../standards-kb/11-frontend-ux.md) |
| 13 | **Children's-data / age handling** | ◐ | ● | ◐ | ◐ | ○ | ○ | ○ | [09](../standards-kb/09-compliance-privacy.md) |
| 14 | **Log masking / no sensitive data in logs** | ● | ● | ● | ● | ● | ● | ● | [04](../standards-kb/04-security.md) · [07](../standards-kb/07-observability-aiops.md) |
| 15 | **Vendor / risk management + change management (CI/CD)** | ◐ | ◐ | ○ | ● | ● | ● | ● | [12](../standards-kb/12-devops-cicd.md) · [04](../standards-kb/04-security.md) |

## Reading the matrix

- **Rows 4–6, 14 are universal.** TLS, encryption at rest, an immutable audit log, and log masking are demanded by every regime in scope. Build these first — they are pure infrastructure and they unlock most of the table.
- **Rows 1–3, 12–13 are the "privacy" cluster.** Driven by GDPR / DPDP / CPRA. See the `privacy/` files.
- **Rows 7–11, 15 are the "assurance" cluster.** Driven by SOC 2 / ISO / PCI / HIPAA. See `attestations/` and `sector/` (sibling task).
- **The overlap is the point:** e.g. control #6 (audit log) is one build in [04-security.md](../standards-kb/04-security.md) that satisfies **seven** regimes. Track it once; cite it seven times in evidence.

## From matrix to work

1. Determine applicable regimes (legal/product decision).
2. Collapse their columns → union of required (● / ◐) controls.
3. For each control, implement the referenced standards-kb domain **once**.
4. Record the evidence artifact named in each regime file; reuse it across every regime that shares the control.
