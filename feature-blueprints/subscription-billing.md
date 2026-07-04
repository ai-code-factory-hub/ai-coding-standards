# Feature Blueprint · Subscription & Billing

> Plans → entitlements/features → quotas/limits → subscriptions → usage → invoices/payments, with full lifecycle, metered billing, proration, dunning, coupons, tax, and idempotent gateway webhooks. **Gate features by entitlement, never by plan name.**

## What it is / when you need it

The commercial engine of the product: what a customer bought, what they're allowed to do, how much they've used, and what they owe. You need it as soon as you charge money and have more than one plan. The golden rule: **features are gated on entitlements**, so pricing/packaging can change without touching feature code.

## Core concept chain

`Plan` bundles `Entitlements` (feature flags) and `Quotas` (limits). A `Subscription` links a customer to a plan and holds lifecycle state. `UsageRecord`s accrue against metered entitlements. Periodically these roll into an `Invoice`, settled by `Payment`s via a gateway whose `WebhookEvent`s drive state — idempotently.

## Data model

Tenant-scoped (the tenant *is* usually the customer here); standard audit columns.

### Plan

| Field | Type | Notes |
| --- | --- | --- |
| `key` | string | Stable, e.g. `pro_monthly` |
| `name` | string | Display |
| `billing_interval` | enum | `month` \| `year` \| `week` \| `one_time` |
| `price_amount` | int | Minor units |
| `currency` | string | ISO-4217 |
| `trial_days` | int | 0 = no trial |
| `is_public` | bool | Self-serve vs sales-only |
| `status` | enum | `active` \| `grandfathered` \| `retired` |

### Entitlement / Feature

| Field | Type | Notes |
| --- | --- | --- |
| `plan_id` | fk | → Plan |
| `feature_key` | string | e.g. `api_access`, `advanced_reports` |
| `type` | enum | `boolean` \| `metered` \| `numeric_limit` |
| `is_enabled` | bool | For boolean features |
| `metered_unit` | string? | e.g. `api_call`, `seat`, `gb` |

### Quota / Limit

| Field | Type | Notes |
| --- | --- | --- |
| `plan_id` | fk | → Plan |
| `feature_key` | string | Feature this caps |
| `limit_value` | bigint | e.g. 10000 |
| `period` | enum | `day` \| `month` \| `billing_cycle` \| `total` |
| `overage_policy` | enum | `block` \| `allow_and_bill` \| `soft_warn` |
| `overage_price` | int? | Per-unit price when `allow_and_bill` |

### Subscription

| Field | Type | Notes |
| --- | --- | --- |
| `customer_id` | fk | The account/tenant |
| `plan_id` | fk | Current plan |
| `status` | enum | `trialing` \| `active` \| `past_due` \| `suspended` \| `cancelled` \| `paused` |
| `current_period_start` | timestamp | |
| `current_period_end` | timestamp | Renewal boundary |
| `trial_end` | timestamp? | |
| `cancel_at_period_end` | bool | Scheduled cancel |
| `cancelled_at` | timestamp? | |
| `gateway_subscription_id` | string | External ref |
| `coupon_id` | fk? | Applied discount |
| `seats` | int? | For per-seat plans |

### UsageRecord (metered)

| Field | Type | Notes |
| --- | --- | --- |
| `subscription_id` | fk | |
| `feature_key` | string | Metered feature |
| `quantity` | bigint | Units consumed |
| `occurred_at` | timestamp | Event time |
| `idempotency_key` | string | Dedupe repeat submissions |
| `billed` | bool | Rolled into an invoice yet |

### Invoice

| Field | Type | Notes |
| --- | --- | --- |
| `subscription_id` | fk | |
| `number` | string | Sequential per tenant, gap-free |
| `status` | enum | `draft` \| `open` \| `paid` \| `uncollectible` \| `void` |
| `subtotal_amount` | int | Minor units |
| `discount_amount` | int | From coupons |
| `tax_amount` | int | |
| `total_amount` | int | |
| `currency` | string | |
| `period_start` / `period_end` | timestamp | |
| `due_at` | timestamp | |
| `issued_at` | timestamp? | |

InvoiceLineItem (`invoice_id`, `description`, `type` = `subscription`/`usage`/`proration`/`discount`/`tax`, `quantity`, `unit_amount`, `amount`).

### Payment

| Field | Type | Notes |
| --- | --- | --- |
| `invoice_id` | fk | |
| `amount` | int | |
| `currency` | string | |
| `status` | enum | `pending` \| `succeeded` \| `failed` \| `refunded` \| `partially_refunded` |
| `gateway_payment_id` | string | |
| `method` | enum | `card` \| `bank` \| `wallet` \| `manual` |
| `failure_code` | string? | Drives dunning |
| `retry_count` | int | Dunning attempts |

### Coupon / Discount

| Field | Type | Notes |
| --- | --- | --- |
| `code` | string | Redemption code |
| `type` | enum | `percent` \| `fixed_amount` |
| `value` | int | % or minor units |
| `duration` | enum | `once` \| `repeating` \| `forever` |
| `duration_months` | int? | For `repeating` |
| `max_redemptions` | int? | |
| `redeemed_count` | int | |
| `valid_until` | timestamp? | |

### WebhookEvent (from gateway)

| Field | Type | Notes |
| --- | --- | --- |
| `gateway_event_id` | string | **Unique** — dedupe key |
| `event_type` | string | e.g. `payment.succeeded` |
| `payload_json` | jsonb | Raw event |
| `signature_verified` | bool | Must be true to process |
| `processed_at` | timestamp? | Null until handled |
| `status` | enum | `received` \| `processed` \| `ignored` \| `failed` |

## Lifecycle

`trialing → active → past_due → suspended → cancelled`, with `→ active` reactivation and `paused` as an optional detour.

- **trial → active**: on first successful payment or trial end with valid payment method.
- **active → past_due**: a renewal payment fails → dunning begins.
- **past_due → active**: a retry/manual payment succeeds.
- **past_due → suspended**: dunning exhausted (retries + grace elapsed) → feature access restricted, data retained.
- **suspended → cancelled**: cancellation grace elapsed → access removed per retention policy.
- **any → cancelled**: user-initiated (respect `cancel_at_period_end`).
- **cancelled/suspended → active (reactivated)**: new payment; may create fresh period + proration.

## Key screens & UX

1. **Pricing / plan picker** — plans with entitlement comparison, trial CTA, coupon field. *Empty/Error* handled.
2. **Checkout** — payment method capture (via gateway SDK, PCI-scoped), tax preview, coupon apply, terms. *Loading* during gateway calls; clear error on decline.
3. **Subscription / account billing** — current plan, status badge, renewal date, seats, upgrade/downgrade (with **proration preview**), cancel (with retain-until date), reactivate.
4. **Usage** — metered features with consumed vs quota, overage warnings, period reset date. *Empty:* "No usage this period."
5. **Invoices** — list with status, download PDF, pay-now for `open`/`past_due`. 
6. **Payment methods** — add/remove/set-default; expiring-card warnings.
7. **Dunning banner / emails** — past-due state surfaced in-app with retry CTA; escalating email sequence.
8. **Admin: plans & coupons** — internal CRUD for plans, entitlements, quotas, coupons (see [admin-console.md](admin-console.md)).

## Rules & logic

- [MUST] **Gate features by entitlement**, resolved from the active subscription — never by matching plan name/id in feature code.
- [MUST] Process gateway webhooks **idempotently**: unique `gateway_event_id`, verify signature, no-op on replays; webhook is the source of truth for payment/subscription state, not the client redirect.
- [MUST] Money in minor units + explicit currency; invoice numbers sequential and gap-free per tenant.
- [MUST] **Proration** on mid-cycle plan changes: credit unused time on the old plan, charge prorated new plan; show a preview before confirm.
- [MUST] Metered usage is deduped by `idempotency_key`; overage honors `overage_policy` (`block` / `allow_and_bill` / `soft_warn`).
- [MUST] **Dunning**: on payment failure, schedule bounded retries with backoff; move to `suspended` only after retries + grace exhaust; every attempt recorded.
- [MUST] Coupons validate redemption limits, validity window, and stacking rules; discount reflected on the invoice, not silently.
- [MUST] Tax computed per jurisdiction at invoice time and stored as its own line/amount.
- [SHOULD] Trials require no charge until conversion; send pre-trial-end reminders.
- [SHOULD] Support downgrade-at-period-end vs immediate; grandfather retired plans for existing subs.
- [SHOULD] Emit domain events (`subscription.activated`, `invoice.paid`, `subscription.churned`) for analytics — feeds [customer-analytics-churn.md](customer-analytics-churn.md).

## Applicable standards

- Money handling, cost, FinOps — [`../standards-kb/16-finops-cost.md`](../standards-kb/16-finops-cost.md)
- Gateway webhooks, idempotency, integration — [`../standards-kb/10-api-integration.md`](../standards-kb/10-api-integration.md)
- PCI scope, payment data security — [`../standards-kb/04-security.md`](../standards-kb/04-security.md)
- Tenant/customer isolation — [`../standards-kb/01-architecture-multitenancy.md`](../standards-kb/01-architecture-multitenancy.md)
- Retention on cancel/suspend — [`../standards-kb/06-data-management.md`](../standards-kb/06-data-management.md)
- Billing UX/states — [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md)
- Billing events/observability — [`../standards-kb/07-observability-aiops.md`](../standards-kb/07-observability-aiops.md)

## Acceptance checklist

- [ ] Features gated by entitlement resolved from the active subscription (no plan-name checks in code).
- [ ] Full lifecycle transitions work: trial→active→past_due→suspended→cancelled→reactivated.
- [ ] Gateway webhooks are signature-verified and idempotent; replays are no-ops.
- [ ] Metered usage deduped; overage policy (block/allow_and_bill/soft_warn) enforced.
- [ ] Plan change produces correct proration with a pre-confirm preview.
- [ ] Dunning runs bounded retries with backoff; suspension only after grace exhausts.
- [ ] Coupons honor limits/validity/stacking; discount shown on invoice.
- [ ] Tax computed and stored per jurisdiction; invoice numbers gap-free per tenant.
- [ ] Money stored in minor units + currency; no floats anywhere in the money path.
- [ ] Domain events emitted for analytics consumption.
