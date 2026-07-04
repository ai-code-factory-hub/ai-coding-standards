# DPDP Act — India Digital Personal Data Protection Act, 2023

Scope: digital personal data of individuals ("Data Principals") in India, refining the generic program in [standards-kb Domain 09](../../standards-kb/09-compliance-privacy.md).

> **Engineering guidance, not legal advice.** The DPDP Act 2023 is operationalized through Rules (some still being finalized). Deadlines, thresholds, and residency specifics may shift as Rules are notified — confirm the current position with counsel before relying on it.

## Scope & applicability

- **[MUST]** Applies to processing of **digital personal data** within India, and to processing **outside India** where it relates to offering goods/services to Data Principals in India.
- **[MUST]** Identify your role: **Data Fiduciary** (decides purpose/means — analogous to GDPR controller), **Data Processor** (processes on a Fiduciary's behalf), or **Significant Data Fiduciary (SDF)** (designated by the Government based on volume/sensitivity/risk).
- Key terms: **Data Principal** (the individual), **Consent Manager** (a registered intermediary through which consent is given/managed).

## Notice requirements (Sec. 5)

- **[MUST]** Provide a **clear, plain-language notice** at or before collection stating: the **personal data** collected, the **purpose**, how the Data Principal can **exercise rights**, and how to **complain to the Data Protection Board**.
- **[MUST]** Offer the notice in **English or any language in the Eighth Schedule** of the Constitution (support i18n — see [13-i18n-accessibility.md](../../standards-kb/13-i18n-accessibility.md)).
- **[MUST]** For consent obtained before the Act, provide a fresh notice as soon as reasonably practicable.

## Consent + Consent Manager (Sec. 6–7)

- **[MUST]** Consent must be **free, specific, informed, unconditional, unambiguous** with a clear affirmative action, and **limited to the data necessary** for the stated purpose.
- **[MUST]** Make **withdrawal as easy as giving** consent; on withdrawal, cease processing (and cause processors to cease) within a reasonable time.
- **[MUST]** Log consent with **purpose, policy version, timestamp, and language** (audit trail, [04-security.md](../../standards-kb/04-security.md)).
- **[SHOULD]** Integrate with a registered **Consent Manager** so Data Principals can give, manage, review, and withdraw consent through an interoperable platform.
- **[MAY]** Process without consent for **"Certain Legitimate Uses"** (Sec. 7) — e.g. voluntarily-provided data for the specified purpose, employment, legal obligations, medical emergencies. Record the basis in the RoPA.

## Data Principal rights → engineering feature that satisfies it

| Right (Section) | Engineering feature that satisfies it |
| --- | --- |
| **Access to information** (Sec. 11) | "My data" view: summary of personal data processed + identities of processors it was shared with |
| **Correction & erasure** (Sec. 12) | Self-serve correct/complete/update + delete flow; erasure propagated across DB, search, caches, logs, processors |
| **Grievance redressal** (Sec. 13) | In-product grievance channel with tracked SLA, escalating to the Data Protection Board |
| **Nomination** (Sec. 14) | Nominee designation so rights can be exercised on death/incapacity |

- **[MUST]** Provide a **readily available grievance mechanism**; respond within the period the Rules prescribe. The Data Principal must exhaust it **before** approaching the Board.
- **[MUST]** Data Principals have corresponding **duties** (e.g. not filing false complaints) — surface these in the notice.

## Children's & disabled-persons data (Sec. 9)

- **[MUST]** For **children (< 18)**, obtain **verifiable parental/guardian consent** before processing.
- **[MUST NOT]** Undertake **tracking, behavioral monitoring, or targeted advertising** directed at children.
- **[MUST]** Apply the same guardian-consent rule to persons with disabilities who have a lawful guardian.
- **[SHOULD]** Build an **age-gate + verifiable-consent flow**; record the verification method.

## Significant Data Fiduciary duties (Sec. 10)

If designated an **SDF**, additionally:

- **[MUST]** Appoint a **Data Protection Officer based in India**, responsible to the board of directors, as the grievance point of contact.
- **[MUST]** Appoint an **independent Data Auditor** and conduct periodic **Data Protection Impact Assessments** + audits.
- **[MUST]** Undertake other measures (e.g. algorithmic-risk review) the Government specifies.

## Breach reporting to the Data Protection Board (Sec. 8(6))

- **[MUST]** On a **personal data breach**, notify **both the Data Protection Board of India and each affected Data Principal**, in the form and time the Rules prescribe (intimation expected to be prompt / without delay — treat as time-critical).
- **[MUST]** Maintain a **breach register** and a runbook triggered by detection signals ([07-observability-aiops.md](../../standards-kb/07-observability-aiops.md)).

## Data residency & cross-border transfer (Sec. 16)

- **[MUST]** Cross-border transfer is permitted **except to countries/territories the Government restricts** via notification (a negative-list / blacklist model, in contrast to GDPR's adequacy model).
- **[MUST]** Honor **sector-specific localization** rules that override the Act (e.g. RBI payment-data localization) where applicable.
- **[SHOULD]** Support **India region pinning** so restricted-transfer obligations and sector localization can be met ([06-data-management.md](../../standards-kb/06-data-management.md)).

## Required engineering controls → standards-kb

| Obligation | Control | standards-kb |
| --- | --- | --- |
| Notice + rights | Multilingual notice, "my data" view, correction/erasure/grievance features | [09](../../standards-kb/09-compliance-privacy.md), [13](../../standards-kb/13-i18n-accessibility.md) |
| Consent | Versioned consent store + Consent Manager integration + withdrawal | [09](../../standards-kb/09-compliance-privacy.md), [04](../../standards-kb/04-security.md) |
| Children's data | Age-gate + verifiable parental consent; no targeted ads to minors | [09](../../standards-kb/09-compliance-privacy.md) |
| Security safeguards (Sec. 8(5)) | TLS, encryption at rest, field-level encryption, access control | [04](../../standards-kb/04-security.md) |
| Breach reporting | Detection + Board/Principal notification runbook + register | [07](../../standards-kb/07-observability-aiops.md), [09](../../standards-kb/09-compliance-privacy.md) |
| Residency/transfer | India region pinning + restricted-country enforcement | [06](../../standards-kb/06-data-management.md) |
| Retention | Erase-when-purpose-served retention schedule | [06](../../standards-kb/06-data-management.md) |

## Evidence artifacts

- RoPA / data inventory noting purpose and consent/legitimate-use basis.
- Consent logs (purpose, version, language, timestamp) + Consent Manager records.
- DSR + grievance logs with SLA tracking.
- Verifiable-parental-consent records for any child data.
- SDF pack (if designated): DPO appointment, Data Auditor reports, DPIAs.
- Breach register + Board/Principal notification runbook.
- Residency configuration + restricted-transfer enforcement records.

## Acceptance checklist

- [ ] Fiduciary/Processor/SDF role determined; DPO-in-India appointed if SDF.
- [ ] Plain-language notice at collection, available in supported Indian languages, stating purpose + rights + Board complaint path.
- [ ] Consent free/specific/informed/unambiguous, withdrawable as easily as given, versioned and logged; Consent Manager integration path defined.
- [ ] Data Principal rights (access, correction, erasure, nomination) shipped as features; grievance channel with tracked SLA.
- [ ] Children: age-gate + verifiable parental consent; no tracking/targeted ads to minors.
- [ ] Breach runbook notifying both the Data Protection Board and affected Principals; breach register maintained.
- [ ] Cross-border transfers checked against the Government restricted list; sector localization (e.g. RBI) honored; India residency option available.
- [ ] Security safeguards in place: TLS, encryption at rest, field-level encryption, deny-by-default access ([04](../../standards-kb/04-security.md)).
