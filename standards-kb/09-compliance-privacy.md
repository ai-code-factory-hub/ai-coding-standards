# 09 · Compliance & Privacy

The engineering program that makes privacy and regulatory obligations real in the product — data-subject rights, consent, sub-processor governance, and audit-ready controls.

> Engineering guidance, not legal advice. This domain covers the *engineering program* around privacy and compliance; the specific regimes that apply to your product are a legal/product decision layered on top.
>
> **Example:** e.g. GDPR / DPDP-style privacy law for personal data, HIPAA-style rules for health data, PCI-DSS for payment cards — mention the regime only to scope which controls below are mandatory.

## Privacy by design

- **[MUST]** **Data minimization** — collect and retain only what a lawful basis supports.
- **[MUST]** Maintain a **Record of Processing Activities (RoPA) / data inventory** and **privacy by default**.
- **[MUST]** **Data classification** (public / internal / confidential / PII / **sensitive personal data**) with controls enforced per class.

## Data-subject rights (build them as real features)

- **[MUST]** Implement data-subject rights as **shipped features** that resolve within statutory deadlines:
  - **access / portability** — machine-readable export,
  - **rectification**,
  - **erasure** — propagated across DB, backups (on their cycle), logs, search, caches, and processors, reconciling any legal-hold or regulated-retention exceptions,
  - **restriction / objection**.

> **Example (healthcare):** an erasure request cannot delete records under a mandated retention/legal hold — those are excluded and the exclusion is recorded and explained to the subject.

## Consent management

- **[MUST]** **Consent is granular, freely given, withdrawable, and logged with the policy version** in force at capture; re-consent is triggered on material change. A single product may carry multiple consent surfaces (e.g. processing consent and data-sharing consent) — each independently controllable by the data subject.

## Residency, sub-processors & breach

- **[MUST]** Enforce **data residency** per the applicable regime.
- **[MUST]** Hold a **DPA with every sub-processor** (messaging, email, payment, AI, cloud, and other integration vendors) and publish a **sub-processor list**.
- **[MUST]** Operate a **breach-notification process** (regulator + affected subjects) within the legally required window; support grievance redressal and any regulator-specific reporting and children's-data protections that apply.

## Assurance & sector controls

- **[MUST]** Run a **SOC 2 / ISO 27001**-style program: control mapping, least-privilege access with periodic reviews, audit logging, change management via CI/CD, vendor/risk management, an incident-response plan, and BCP/DR.
- **[MUST]** Where payment cards are handled, be **PCI-DSS** compliant — use a tokenizing gateway and **never store the PAN**. Where regulated data (e.g. health data) is handled, meet the applicable sector standard.
- **[SHOULD]** Use continuous-compliance tooling (e.g. Vanta / Drata) for evidence collection.

## Legal notices

- **[MUST]** **Privacy Policy + Terms + cookie/consent notice**, generated and versioned, **accepted at registration** (explicit, not pre-ticked), with the accepted version logged; re-acceptance on change; always accessible.
- **[MAY]** Per-tenant white-label policies.

See [Security](04-security.md) for the audit trail and access controls these obligations depend on, and [Data Management](06-data-management.md) for retention and residency mechanics.

## Acceptance checklist

- RoPA + data classification + minimization in place.
- Data-subject-rights features live and resolving within deadlines.
- Consent granular, withdrawable, and logged with policy version.
- Residency enforced; DPA + public list for every sub-processor; breach process defined.
- Consent-manager / grievance / regulator-reporting supported where the regime requires.
- SOC 2 / ISO 27001-style controls with evidence; PCI-DSS where cards are handled.
- Versioned policies explicitly accepted at signup and re-accepted on change.
