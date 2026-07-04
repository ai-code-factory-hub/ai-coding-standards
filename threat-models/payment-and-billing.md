# Threat Model — Payment & Billing

> Filled STRIDE model for checkout, charges, refunds, subscriptions, and payment-provider integration. Method & risk rating: [README.md](README.md). Blank copy: [threat-model-template.md](threat-model-template.md).
> **Owner:** Security Officer · **Extends:** [04 Security](../standards-kb/04-security.md), [09 Compliance & Privacy](../standards-kb/09-compliance-privacy.md)

## Feature / asset

Price calculation, checkout, charge capture, subscription/metering, refunds, and the payment-provider (PSP) integration + its webhooks. **Asset:** money and cardholder data. Errors here are directly monetizable, and card data is regulated (PCI-DSS). **The client is never the source of truth for price or entitlement.**

## Data classification

**Regulated** — payment data (PCI-DSS). Prefer to hold **tokens, not PANs**; keep raw card data out of your systems entirely (use PSP-hosted fields / tokenization) to shrink PCI scope. → [09 Compliance & Privacy](../standards-kb/09-compliance-privacy.md).

## Actors

| Actor | Trusted? | Notes |
|---|---|---|
| Paying customer | Semi-trusted | can manipulate client-side amounts |
| Anonymous checkout user | **Untrusted** | |
| **PSP webhook (payment succeeded/failed/refunded)** | **Untrusted until verified** | forgery = fake payments |
| Internal refund operator | Semi-trusted | refund-abuse / insider risk |

## Trust boundaries

- **B1** checkout edge. **B2** authn. **B3** tenant/merchant. **B5** PSP (charges + webhooks).

## Attack surface

Cart/price payloads, quantity and coupon fields, the amount/currency sent to the charge call, the PSP webhook receiver, refund endpoints, idempotency keys, and stored payment references.

## STRIDE analysis

| # | STRIDE | Threat | How (attack path) | Mitigation `[MUST]`/`[SHOULD]` | Standards ref | Residual |
|---|---|---|---|---|---|---|
| S1 | Spoofing | **Webhook forgery** | Attacker POSTs a fake "payment.succeeded" to fulfil an order without paying | `[MUST]` HMAC/signature-verify PSP webhooks (constant-time); also **confirm charge state via the PSP API** before fulfilment — never trust the webhook body alone; per-PSP secret | [10](../standards-kb/10-api-integration.md) / [04](../standards-kb/04-security.md) | Low |
| T1 | Tampering | **Price / amount tampering** | Client posts its own price, discount, quantity, or currency; negative amounts; coupon stacking | `[MUST]` compute price **server-side** from catalog + validated coupons; ignore any client-sent amount; reject negatives; validate coupon eligibility/limits server-side | [04](../standards-kb/04-security.md) | Low |
| T2 | Tampering | **Currency / rounding abuse** | Currency swap to a cheaper unit; rounding/precision exploited to skim fractions | `[MUST]` store money as integer minor units (never float); fix currency server-side per order; consistent rounding policy; reconcile totals == sum of lines | [06](../standards-kb/06-data-management.md) | Low |
| R1 | Repudiation | **Chargeback / denied transaction** | Customer denies a purchase or refund; no defensible trail | `[MUST]` immutable audit of charge/refund/subscription events with amount, currency, actor, PSP reference, idempotency key, correlation id | [20](../standards-kb/20-logging-audit-and-traceability.md) | Low |
| I1 | Info disclosure | **PAN / cardholder-data exposure** | Raw card numbers stored/logged; DB dump or log leak exposes them | `[MUST]` **tokenize** — never store PAN/CVV; use PSP-hosted fields so raw data never touches your servers; mask everywhere; never log payment data; encrypt any retained references | [04](../standards-kb/04-security.md) / [09](../standards-kb/09-compliance-privacy.md) | Low |
| I2 | Info disclosure | **Cross-account billing leakage** | Guessing an invoice/receipt id exposes another customer's billing data (IDOR) | `[MUST]` authorize every invoice/receipt/payment-method by owner+tenant; non-guessable ids | [04](../standards-kb/04-security.md) | Med |
| D1 | DoS | **Card testing / BIN attack** | Attacker uses checkout to validate stolen cards at high volume, racking up PSP fees + fraud flags | `[MUST]` rate-limit + velocity checks per user/IP/card fingerprint; CAPTCHA/step-up on anomaly; enable PSP fraud/3-D Secure | [10](../standards-kb/10-api-integration.md) | Med |
| E1/T3 | Elev./Tamper | **Replay of a successful charge/refund** | Retry or replay a charge or refund request to double-charge or double-refund | `[MUST]` idempotency key on every charge/refund (dedupe server-side); mark webhook delivery ids processed-once; state machine forbids double transitions | [10](../standards-kb/10-api-integration.md) | Low |
| E2 | Elevation | **Refund abuse** | Customer or insider issues refunds exceeding the paid amount, to the wrong destination, or repeatedly | `[MUST]` refund only against a real captured charge, capped at remaining refundable balance, to the original method; approval + limit thresholds for operators; audit every refund | [04](../standards-kb/04-security.md) / [20](../standards-kb/20-logging-audit-and-traceability.md) | Med |

## Top risks

1. **Price/amount tampering (T1)** — trusting any client-supplied price is a direct loss; recompute server-side, always.
2. **Webhook forgery + missing PSP re-check (S1)** — fulfilling on an unverified callback ships goods for free.
3. **Refund abuse & double-refund (E2, replay)** — insider and replay paths drain funds; cap, approve, and idempotency-key everything.

## Open risks / decisions

| Item | Decision | Owner | Due |
|---|---|---|---|
| PCI scope | Use PSP-hosted fields/tokenization to stay SAQ-A where possible | | |
| Refund approval thresholds | Define per-role limits + dual control above $X | | |

---

## Mitigation checklist

- [ ] `[MUST]` Price/discount/quantity computed server-side; client amounts ignored; no negatives.
- [ ] `[MUST]` Money as integer minor units; currency fixed server-side; totals reconciled.
- [ ] `[MUST]` PSP webhooks signature-verified **and** charge state re-confirmed via PSP API before fulfilment.
- [ ] `[MUST]` No PAN/CVV stored or logged; tokenization / PSP-hosted fields; references encrypted & masked.
- [ ] `[MUST]` Every invoice/receipt/payment-method authorized by owner+tenant.
- [ ] `[MUST]` Idempotency keys on charges/refunds; webhook deliveries processed once; state machine blocks double transitions.
- [ ] `[MUST]` Refunds capped to refundable balance, original method, with operator limits + approval.
- [ ] `[MUST]` Card-testing velocity/rate limits + 3-D Secure/fraud checks.
- [ ] `[MUST]` All charge/refund/subscription events immutably audited → [20](../standards-kb/20-logging-audit-and-traceability.md).
