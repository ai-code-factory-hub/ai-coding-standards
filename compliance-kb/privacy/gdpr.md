# GDPR — EU/EEA General Data Protection Regulation

Scope: personal data of individuals in the EU/EEA, with concrete obligations, engineering controls, and evidence layered on top of [standards-kb Domain 09](../../standards-kb/09-compliance-privacy.md).

> **Engineering guidance, not legal advice.** Deadlines, thresholds, and applicability below are the common engineering reading of the Regulation (EU 2016/679). Confirm with counsel before you rely on them.

## Scope & applicability

- **[MUST]** Determine whether you are a **controller** (you decide purposes/means) or a **processor** (you act on a controller's instructions) — obligations differ. In multi-tenant SaaS you are typically a **processor** for tenant data and a **controller** for your own account/billing data.
- **[MUST]** GDPR applies if you (a) are established in the EU/EEA, **or** (b) offer goods/services to, or monitor the behavior of, people in the EU/EEA — regardless of where you host.
- **[MUST]** If you have no EU establishment but are in scope, appoint an **EU Representative** (Art. 27).

## Lawful bases (Art. 6)

- **[MUST]** Record a lawful basis **per processing purpose** in the RoPA. One of: **consent**, **contract**, **legal obligation**, **vital interests**, **public task**, **legitimate interests**.
- **[MUST]** For **special-category data** (Art. 9 — health, biometrics, ethnicity, etc.) also satisfy an Art. 9 condition (usually explicit consent or a health/employment provision).
- **[SHOULD]** Prefer **contract** or **legitimate interests** for core service data; reserve **consent** for optional/marketing processing (it is withdrawable and raises the bar).

## Data-subject rights → engineering feature that satisfies it

Statutory response deadline: **1 month** from request (extendable by 2 months for complex/numerous requests, with notice). Build each right as a **shipped feature**, not a manual ticket.

| Right (Article) | Deadline | Engineering feature that satisfies it |
| --- | --- | --- |
| **Access** (Art. 15) | 1 month | Self-serve "download my data" → machine-readable export of all records + processing metadata |
| **Rectification** (Art. 16) | 1 month | User-editable profile/records + admin correction flow, audit-logged |
| **Erasure / "right to be forgotten"** (Art. 17) | 1 month | Delete pipeline propagating across DB, search, caches, logs, and processors; backups purged on their cycle; legal-hold/retention exceptions recorded |
| **Portability** (Art. 20) | 1 month | Export in structured, commonly-used, machine-readable format (JSON/CSV); direct controller-to-controller transfer where feasible |
| **Restriction** (Art. 18) | 1 month | "Freeze" flag that blocks processing while retaining the record |
| **Objection** (Art. 21) | 1 month | Opt-out toggle for legitimate-interest / direct-marketing processing |
| **Rights re: automated decisions** (Art. 22) | — | Human-review path + no solely-automated decisions with legal/significant effect without safeguards |

- **[MUST]** Verify requester identity before fulfilling (without collecting excessive new data).
- **[MUST]** Fulfillment is **free** for the first request; log every request and its resolution for evidence.

## Consent requirements (Art. 7)

- **[MUST]** Consent is **freely given, specific, informed, unambiguous**, via a clear affirmative act. **No pre-ticked boxes, no bundling** consent with T&Cs.
- **[MUST]** **Granular** per purpose; **as easy to withdraw as to give**; log the **policy version, timestamp, and scope** at capture (see audit trail, [04-security.md](../../standards-kb/04-security.md)).
- **[MUST]** Re-consent on material change of purpose.

## Governance — DPO, RoPA, DPA

- **[MUST]** Maintain a **Record of Processing Activities** (Art. 30): purposes, categories of data/subjects, recipients, transfers, retention, security measures.
- **[MUST]** Sign a **Data Processing Agreement** (Art. 28) with every sub-processor; publish a **sub-processor list** and notify tenants of changes.
- **[SHOULD]** Appoint a **Data Protection Officer** if you do large-scale monitoring or process special-category data at scale.
- **[SHOULD]** Run a **DPIA** (Art. 35) for high-risk processing before launch.

## Cross-border transfer (Chapter V)

- **[MUST]** For transfers outside the EU/EEA, rely on an **adequacy decision**, **Standard Contractual Clauses (SCCs)**, or **Binding Corporate Rules**, plus a **Transfer Impact Assessment**.
- **[MUST]** Support **data residency** so EU tenant data can be pinned to EU regions (see [06-data-management.md](../../standards-kb/06-data-management.md)).

## Breach notification (Art. 33–34)

- **[MUST]** Notify the **supervisory authority within 72 hours** of becoming aware of a breach likely to risk individuals' rights.
- **[MUST]** Notify **affected data subjects without undue delay** where risk is high.
- **[MUST]** Keep an internal **breach register** (all breaches, even non-notifiable) — see [07-observability-aiops.md](../../standards-kb/07-observability-aiops.md) for detection signals.

## Required engineering controls → standards-kb

| Obligation | Control | standards-kb |
| --- | --- | --- |
| Rights as features | DSR export/delete/rectify pipeline | [09](../../standards-kb/09-compliance-privacy.md), [06](../../standards-kb/06-data-management.md) |
| Consent | Versioned consent store + audit log | [09](../../standards-kb/09-compliance-privacy.md), [04](../../standards-kb/04-security.md) |
| Security of processing (Art. 32) | TLS, encryption at rest, field-level encryption, access control | [04](../../standards-kb/04-security.md) |
| Accountability | Immutable audit trail of consent/exports/deletions | [04](../../standards-kb/04-security.md), [07](../../standards-kb/07-observability-aiops.md) |
| Residency / transfer | Region pinning + SCC records | [06](../../standards-kb/06-data-management.md) |
| Retention | Retention schedule + auto-deletion | [06](../../standards-kb/06-data-management.md) |
| Notices | Versioned Privacy Policy accepted at signup | [09](../../standards-kb/09-compliance-privacy.md), [11](../../standards-kb/11-frontend-ux.md) |

## Evidence artifacts

- RoPA document (current, dated).
- Consent logs with policy version + timestamp.
- DSR request/resolution log showing within-deadline fulfillment.
- Sub-processor list + signed DPAs; SCCs for transfers + TIA.
- DPIA(s) for high-risk processing.
- Breach register + 72h notification runbook.
- Encryption/KMS config and access-review records (from [04-security.md](../../standards-kb/04-security.md)).

## Acceptance checklist

- [ ] Controller/processor role documented; EU Representative appointed if required.
- [ ] Lawful basis recorded per purpose in a current RoPA; Art. 9 condition for special-category data.
- [ ] All six data-subject rights shipped as features, resolving within 1 month, with identity verification and request logging.
- [ ] Consent granular, unbundled, withdrawable, versioned, and audit-logged; no pre-ticked boxes.
- [ ] Sub-processor list published; DPA signed with each; tenants notified of changes.
- [ ] Cross-border transfers covered by adequacy/SCCs/BCRs + TIA; EU residency option available.
- [ ] 72h supervisory-authority notification runbook + subject notification path + breach register.
- [ ] Security of processing: TLS, encryption at rest, field-level encryption, deny-by-default access ([04](../../standards-kb/04-security.md)).
- [ ] DPIA completed for high-risk processing; DPO appointed if thresholds met.
