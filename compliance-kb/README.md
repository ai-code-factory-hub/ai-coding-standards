# Compliance KB

The regime-specific layer that turns generic privacy/compliance engineering into concrete, per-law obligations, engineering controls, and evidence — sitting **on top of** the generic program in [standards-kb Domain 09](../standards-kb/09-compliance-privacy.md).

> **Engineering guidance, not legal advice.** This KB translates well-known regulatory regimes into engineering-actionable controls, mappings, and checklists. It is **not** a legal opinion and does not establish an attorney–client relationship. Which regimes apply to *your* product, tenant, and data — and the authoritative interpretation of each obligation — is a legal/product decision. Have counsel confirm applicability, thresholds, and deadlines before you rely on anything here.

## Why this KB exists

standards-kb Domain 09 describes the **engineering program** for privacy and compliance in the abstract: RoPA, data-subject-rights features, consent, sub-processor governance, audit-ready controls. It deliberately stays regime-neutral.

This compliance-kb is the **regime-specific overlay**. For each law or framework it answers three engineering questions:

1. **What does this regime actually require?** — concrete obligations (rights, deadlines, notices, breach windows, residency).
2. **What must we build?** — required engineering controls, mapped back to a standards-kb domain that implements them.
3. **How do we prove it?** — evidence artifacts and an acceptance checklist.

It stays **project-agnostic**: no product names, no tenant specifics. Drop it into any enterprise SaaS starter kit and scope which files apply.

## Regimes overlap — do it once, satisfy many

Most regimes demand the **same underlying controls** wearing different names. Consent, an immutable audit log, encryption at rest, breach notification, retention, and access review each satisfy *multiple* regimes at once. Build the control once against the strictest applicable bar, then map it to every regime that needs it.

See **[controls-matrix.md](controls-matrix.md)** for the master "one control → many regimes → which standards-kb domain implements it" map. Read that first; it is the shortest path from "which laws apply" to "what we build."

Rough overlap intuition:

| Control | Roughly who wants it |
| --- | --- |
| Consent + policy versioning | GDPR, DPDP, CPRA (opt-out flavor) |
| Data-subject / consumer rights | GDPR, DPDP, CPRA |
| Immutable audit log | Everyone (GDPR, DPDP, SOC 2, ISO 27001, PCI, HIPAA) |
| Encryption at rest + in transit | Everyone |
| Breach notification | GDPR (72h), DPDP (Board), HIPAA, PCI, most US states |
| DPA / sub-processor governance | GDPR, DPDP, SOC 2, HIPAA (BAA) |
| Retention + deletion | GDPR, DPDP, CPRA, HIPAA |
| Periodic access review | SOC 2, ISO 27001, PCI, HIPAA |

## Folder layout

```
compliance-kb/
├── README.md              ← you are here
├── controls-matrix.md     ← control → regimes → standards-kb domain
├── privacy/               ← data-protection / privacy laws
│   ├── gdpr.md            ← EU/EEA General Data Protection Regulation
│   ├── dpdp-act-india.md  ← India Digital Personal Data Protection Act
│   └── ccpa.md            ← California CCPA/CPRA
├── attestations/          ← added by sibling task: soc2, iso-27001, pci-dss
└── sector/                ← added by sibling task: hipaa-health
```

> `attestations/` and `sector/` are populated by a **sibling task**. This task owns the `privacy/` files, `README.md`, and `controls-matrix.md`. The matrix already references SOC 2 / ISO / PCI / HIPAA columns so the two halves interlock.

## Index of compliance files

| File | Regime | Status |
| --- | --- | --- |
| [privacy/gdpr.md](privacy/gdpr.md) | GDPR (EU/EEA) | ✅ this task |
| [privacy/dpdp-act-india.md](privacy/dpdp-act-india.md) | DPDP Act 2023 (India) | ✅ this task |
| [privacy/ccpa.md](privacy/ccpa.md) | CCPA / CPRA (California) | ✅ this task |
| [controls-matrix.md](controls-matrix.md) | cross-regime control map | ✅ this task |
| attestations/soc2.md | SOC 2 | ⏳ sibling task |
| attestations/iso-27001.md | ISO/IEC 27001 | ⏳ sibling task |
| attestations/pci-dss.md | PCI-DSS | ⏳ sibling task |
| sector/hipaa-health.md | HIPAA (US health) | ⏳ sibling task |

## How this maps back to standards-kb

This KB names *what a regime demands*; standards-kb tells you *how the platform implements it*. The load-bearing links:

- **[../standards-kb/09-compliance-privacy.md](../standards-kb/09-compliance-privacy.md)** — the generic privacy-engineering program: RoPA, DSR-as-features, consent, sub-processors, breach process. Every privacy file here refines Domain 09.
- **[../standards-kb/04-security.md](../standards-kb/04-security.md)** — the audit trail, access control, encryption, and secrets hygiene that privacy obligations depend on.
- **[../standards-kb/06-data-management.md](../standards-kb/06-data-management.md)** — retention, residency, deletion propagation, backups.
- **[../standards-kb/07-observability-aiops.md](../standards-kb/07-observability-aiops.md)** — log masking, breach detection signals, evidence collection.

**Usage:** pick the regimes that apply → read those files' obligations + acceptance checklists → implement the referenced standards-kb controls once → record the evidence artifacts each file lists.
