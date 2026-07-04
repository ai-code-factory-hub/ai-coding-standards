# CCPA / CPRA — California Consumer Privacy Act (as amended by CPRA)

Scope: personal information of California residents ("consumers"), refining the generic program in [standards-kb Domain 09](../../standards-kb/09-compliance-privacy.md).

> **Engineering guidance, not legal advice.** Thresholds and definitions below reflect the CCPA as amended by the CPRA and CPPA regulations. Confirm current thresholds and applicability with counsel.

## Scope & applicability (thresholds)

- **[MUST]** A for-profit business doing business in California is in scope if it meets **any one** of:
  - annual gross revenue **> $25M**, **or**
  - buys/sells/shares the personal information of **≥ 100,000** consumers or households per year, **or**
  - derives **≥ 50%** of annual revenue from **selling or sharing** personal information.
- **[MUST]** Determine your role: **business** (decides purposes), **service provider / contractor** (processes under a compliant contract), or **third party**. Contracts must include the CPRA-required processing terms.
- Definitions: **sensitive personal information (SPI)** = SSN, government ID, financial account + credentials, precise geolocation, race/religion, health, sex life, biometric-for-ID, contents of private communications. **"Sale"** = disclosure for monetary/other valuable consideration; **"Share"** = disclosure for cross-context behavioral advertising.

## Consumer rights → engineering feature that satisfies it

Response deadline: **45 days** (extendable once by 45 days with notice). Provide **at least two** request methods (e.g. toll-free number and a web form; a web-only business may use an interactive form/email).

| Right | Engineering feature that satisfies it |
| --- | --- |
| **Right to know / access** | Self-serve report of categories + specific pieces of PI collected, sources, purposes, and third parties disclosed to (12-month+ lookback) |
| **Right to delete** | Delete pipeline across DB, search, caches, logs, and service providers; statutory exceptions recorded |
| **Right to correct** | User-editable data + admin correction flow, audit-logged |
| **Right to opt out of sale/sharing** | "Do Not Sell or Share My Personal Information" link/toggle + honoring the **Global Privacy Control (GPC)** browser signal |
| **Right to limit use of SPI** | "Limit the Use of My Sensitive Personal Information" link restricting SPI to permitted business purposes |
| **Right to non-discrimination** | No degraded price/service for exercising rights; financial-incentive programs disclosed |
| **Right re: automated decision-making / access to logic** | Opt-out + meaningful-information path (per CPPA ADMT rules) |

- **[MUST]** Verify the consumer's identity to a reasonable degree before fulfilling know/delete/correct.
- **[MUST]** Allow an **authorized agent** to submit requests on a consumer's behalf.

## "Do Not Sell or Share" + opt-out signals

- **[MUST]** If you sell or share PI, post a clear **"Do Not Sell or Share My Personal Information"** link on the homepage (or an Alternative Opt-Out Link).
- **[MUST]** **Honor the Global Privacy Control (GPC)** as a valid opt-out signal — detect it server-side/client-side and apply it without requiring a separate form.
- **[MUST]** For consumers **under 16**, sale/sharing requires **opt-in** (13–15 the consumer opts in; under 13 a parent opts in).
- **[MUST]** Do not re-ask for opt-in for at least **12 months** after an opt-out.

## Notice & retention

- **[MUST]** Publish a **Notice at Collection** (categories, purposes, whether sold/shared, retention) at or before the point of collection, plus a full **Privacy Policy** updated at least every 12 months.
- **[MUST]** Disclose and enforce a **retention period** per category — do not keep PI longer than reasonably necessary (see [06-data-management.md](../../standards-kb/06-data-management.md)).

## Required engineering controls → standards-kb

| Obligation | Control | standards-kb |
| --- | --- | --- |
| Rights as features | Access/delete/correct pipeline + 12-month lookback report | [09](../../standards-kb/09-compliance-privacy.md), [06](../../standards-kb/06-data-management.md) |
| Opt-out of sale/share | "Do Not Sell/Share" toggle + GPC signal handling | [09](../../standards-kb/09-compliance-privacy.md), [11](../../standards-kb/11-frontend-ux.md) |
| Limit SPI | SPI-use limitation flag enforced in processing paths | [09](../../standards-kb/09-compliance-privacy.md), [06](../../standards-kb/06-data-management.md) |
| Reasonable security (a breach of which creates statutory liability) | TLS, encryption at rest, field-level encryption, access control | [04](../../standards-kb/04-security.md) |
| Accountability | Audit log of requests, opt-outs, deletions | [04](../../standards-kb/04-security.md), [07](../../standards-kb/07-observability-aiops.md) |
| Service-provider contracts | Sub-processor register with CPRA terms | [09](../../standards-kb/09-compliance-privacy.md) |
| Retention | Per-category retention schedule + auto-deletion | [06](../../standards-kb/06-data-management.md) |

## Evidence artifacts

- Notice at Collection + Privacy Policy (dated, ≤12 months old).
- Request logs (know/delete/correct/opt-out) showing 45-day fulfillment + identity verification.
- GPC-handling test evidence.
- SPI-limitation and opt-out state records per consumer.
- Service-provider/contractor agreements with CPRA clauses.
- Retention schedule + reasonable-security controls (from [04-security.md](../../standards-kb/04-security.md)).

## Acceptance checklist

- [ ] Applicability assessed against the three thresholds; business/service-provider role and contracts confirmed.
- [ ] Notice at Collection + annually-updated Privacy Policy published, with categories, purposes, sale/share status, and retention.
- [ ] Know / delete / correct requests shipped as features, resolving within 45 days, with identity verification and authorized-agent support.
- [ ] "Do Not Sell or Share" link present; **GPC signal honored** automatically.
- [ ] "Limit the Use of My Sensitive Personal Information" control enforced in processing.
- [ ] Opt-in (not opt-out) for sale/sharing of minors' data; no re-solicitation within 12 months.
- [ ] Non-discrimination for exercising rights; financial incentives disclosed.
- [ ] Reasonable security: TLS, encryption at rest, field-level encryption, deny-by-default access, audit log ([04](../../standards-kb/04-security.md)).
- [ ] Per-category retention schedule enforced ([06](../../standards-kb/06-data-management.md)).
