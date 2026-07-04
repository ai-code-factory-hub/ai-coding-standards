# PCI DSS (Payment Card Data)

Scope: a regime-specific attestation layer on top of [standards-kb](../../standards-kb/) Domains 04 (Security) & 09 (Compliance & Privacy). The **Payment Card Industry Data Security Standard** (current major version: **PCI DSS v4.0.1**) applies to any organization that **stores, processes, or transmits cardholder data (CHD) or sensitive authentication data (SAD)**. It is a contractual requirement from the card brands, enforced by your acquirer/payment provider.

> **Engineering guidance, not legal or audit advice.** This file helps engineers architect for *minimum scope* and evidence the controls PCI cares about. Your merchant level, applicable SAQ, need for a QSA, and Attestation of Compliance are determined by your acquirer and (above certain volumes) a Qualified Security Assessor. Nothing here substitutes for that.

See also: [compliance-kb README](../README.md) · [controls matrix](../controls-matrix.md)

---

## When it applies

- **[MUST]** If your system ever touches a **Primary Account Number (PAN)** — alone or with cardholder name, expiry, or service code — PCI DSS applies to the systems that touch it and everything connected to them (the **Cardholder Data Environment, CDE**).
- **Cardholder Data (CHD):** PAN, cardholder name, expiration date, service code.
- **Sensitive Authentication Data (SAD):** full track data, CAV2/CVC2/CVV2/CID, PINs/PIN blocks. **[MUST NOT]** store SAD after authorization — ever, even encrypted.
- Storing the PAN, even encrypted, pulls you into the widest, most expensive scope.

## The smart default: tokenize and never store PAN

**[MUST] Do not build a CDE if you can avoid one.** Use a PCI-DSS-certified **tokenizing payment gateway** (Stripe, Adyen, Braintree, Razorpay, etc.) so raw card data goes **browser → gateway** and never touches your servers.

- **[MUST]** Capture card data with a **gateway-hosted field / iframe / redirect / hosted checkout** (e.g. Stripe Elements). Your DOM/JS must not read the PAN.
- **[MUST]** Store only the gateway's **token** + safe metadata (last 4, brand, expiry) — never the PAN, never SAD.
- **[MUST]** Charge/refund by referencing the token via the gateway API over TLS.
- **Result:** your scope collapses from "protect a full CDE" to "protect the integrity of the payment page and the token" — typically qualifying you for the smallest SAQ.

> This is the single highest-leverage PCI decision. Every requirement below is cheaper — or out of scope — when no PAN lives on your systems.

## SAQ types (Self-Assessment Questionnaires)

Merchants below the QSA threshold self-assess with the SAQ matching **how card data is captured**:

| SAQ | Applies when | Typical requirement count |
| --- | --- | --- |
| **A** | Fully outsourced: all CHD functions on a PCI-compliant third party; you only redirect/iframe. **This is the target for a tokenizing-gateway SaaS.** | Smallest |
| **A-EP** | E-commerce where your page is served by you but posts to a third party (e.g. JS-based fields you control). Page integrity matters. | Moderate |
| **B / B-IP** | Standalone terminals / imprint machines (no e-commerce). | Small |
| **C / C-VT** | Payment app connected to internet / virtual terminal. | Moderate |
| **D** | Everything else — **you store/process/transmit PAN**. All requirements apply. | Largest |
| **P2PE** | Validated point-to-point encryption solution. | Small |

- **[SHOULD]** Architect for **SAQ A**. Slipping to A-EP or D is usually an architecture regression (you started touching card data), not an inevitability.

## The 12 requirements (v4.0.1), summarized

**Build & maintain a secure network**
1. Install and maintain **network security controls** (firewalls, segmentation of the CDE).
2. Apply **secure configurations** — no vendor defaults, remove unnecessary services.

**Protect account data**
3. **Protect stored account data** — minimize storage, render PAN unreadable (strong crypto/truncation/tokenization), **never store SAD**.
4. **Encrypt CHD in transit** over open/public networks (strong TLS).

**Maintain a vulnerability-management program**
5. Protect systems against **malware**.
6. Develop and maintain **secure systems and software** — patching, secure SDLC, change control.

**Implement strong access control**
7. Restrict access to CHD by **business need-to-know** (least privilege).
8. **Identify users and authenticate** access — unique IDs, **MFA** into the CDE.
9. Restrict **physical access** to CHD (largely the cloud/gateway provider's responsibility under shared responsibility).

**Regularly monitor & test networks**
10. **Log and monitor** all access to system components and CHD; retain and review logs.
11. **Test security** regularly — vuln scans (ASV quarterly for external), pen tests, and (v4.0) integrity monitoring of the payment page.

**Maintain an information-security policy**
12. Maintain an **information-security policy** and program (risk assessment, awareness, incident response, vendor management).

## Network segmentation

- **[MUST]** If a CDE exists, **segment it** from the rest of the network so out-of-scope systems cannot reach it — segmentation is what keeps scope (and audit cost) small.
- **[MUST]** Validate segmentation (pen test) to prove out-of-scope systems truly are isolated.
- **[SHOULD]** With the tokenizing-gateway pattern there is **no CDE to segment** — the cleanest possible outcome.

## Control → standards-kb domain → evidence artifact

| PCI requirement | standards-kb domain that implements it | Evidence artifact |
| --- | --- | --- |
| Req 1 network controls / segmentation | [04-security](../../standards-kb/04-security.md) / [01-architecture](../../standards-kb/01-architecture-multitenancy.md) | Firewall/security-group config, segmentation test result, network diagram |
| Req 2 secure config | [12-devops-cicd](../../standards-kb/12-devops-cicd.md) | Hardening baseline / IaC, no-default-creds check |
| Req 3 protect stored data / tokenization | [06-data-management](../../standards-kb/06-data-management.md) | Data-flow diagram proving no PAN stored, gateway tokenization config |
| Req 4 encrypt in transit | [04-security](../../standards-kb/04-security.md) | TLS scan (strong ciphers), cert inventory |
| Req 5 anti-malware | [04-security](../../standards-kb/04-security.md) | Malware-protection config on in-scope systems |
| Req 6 secure SDLC / patching | [08-testing-qa](../../standards-kb/08-testing-qa.md) / [12-devops-cicd](../../standards-kb/12-devops-cicd.md) | SAST/SCA results, patch cadence, change-control records |
| Req 7 need-to-know access | [04-security](../../standards-kb/04-security.md) | RBAC matrix, least-privilege review |
| Req 8 auth + MFA | [04-security](../../standards-kb/04-security.md) | IdP config, MFA-into-CDE evidence, unique-ID proof |
| Req 9 physical | shared responsibility | Cloud/gateway AOC covering physical controls |
| Req 10 logging & monitoring | [04-security](../../standards-kb/04-security.md) / [07-observability](../../standards-kb/07-observability-aiops.md) | Immutable log samples, retention config, review records |
| Req 11 scanning / page integrity | [08-testing-qa](../../standards-kb/08-testing-qa.md) | ASV scan reports, pen-test report, payment-page script-integrity/CSP evidence |
| Req 12 security policy & vendor mgmt | [09-compliance-privacy](../../standards-kb/09-compliance-privacy.md) | InfoSec policy, risk assessment, vendor AOCs, IR plan |

## Evidence auditors / your acquirer ask for

- The completed **SAQ** and signed **Attestation of Compliance (AOC)**.
- A **data-flow diagram** and **network diagram** showing exactly where (or that) card data flows.
- **ASV quarterly external scan** reports (passing) and any pen-test report.
- Your **gateway/service providers' AOCs** (you inherit their compliance for outsourced functions).
- Evidence for the mapped controls above, scoped to in-scope systems only.

## Acceptance checklist

- [ ] Card capture uses a PCI-certified **tokenizing gateway** (hosted fields/iframe/redirect); **no PAN or SAD on our systems**.
- [ ] Only tokens + safe metadata (last 4, brand, expiry) are stored; SAD never stored.
- [ ] Correct **SAQ** identified (target **SAQ A**); AOC on file; provider AOCs collected.
- [ ] Data-flow + network diagrams accurately show card-data paths.
- [ ] TLS strong-cipher scan passes; MFA required into any in-scope system.
- [ ] Least-privilege access to any in-scope component; immutable logging + retention.
- [ ] ASV quarterly external scans passing; pen test and payment-page integrity (CSP/script control) in place.
- [ ] If a CDE exists: it is segmented and segmentation is validated.
- [ ] InfoSec policy, risk assessment, IR plan, and vendor management maintained (Req 12).
