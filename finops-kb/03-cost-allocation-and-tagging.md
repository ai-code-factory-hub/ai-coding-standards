# 03 · Cost Allocation, Tagging & Guardrails

> The mechanics that make the [01](01-per-tenant-cost-model.md) model possible and keep spend bounded. Implements the Domain 16 **[MUST]**s on tagging, budgets/anomaly alerts, and hard guardrails from [../standards-kb/16-finops-cost.md](../standards-kb/16-finops-cost.md) — at build depth.

## Resource tagging strategy

You cannot allocate what you cannot attribute. Tags are the substrate for the entire cost model: shared-cost allocation ([01](01-per-tenant-cost-model.md) method B) and per-service/per-env budgets both read tags.

### Mandatory tag schema

**[MUST]** Every provisioned cloud resource carries these four tags. No untagged resource ships to production (Domain 16 **[MUST]**).

| Tag | Purpose | Example values | Cardinality |
|---|---|---|---|
| `owner` | Accountable team/person for the spend | `team-payments`, `team-platform` | low |
| `env` | Environment — drives budgets & scale-down | `prod` \| `staging` \| `dev` \| `preview` | fixed set |
| `service` | Logical service/component | `api`, `worker`, `search`, `llm-gateway` | low |
| `tenant` | Tenant attribution where a resource is tenant-dedicated | `tenant-<id>` \| `shared` | high or `shared` |

- **[MUST]** Enforce tags at provision time — deny/flag untagged resources via policy (IaC policy check, cloud org policy, or admission control). Tagging *after the fact* never happens.
- **[SHOULD]** For **shared** resources (`tenant=shared`), per-tenant attribution comes from the **application usage meters** in [01](01-per-tenant-cost-model.md), not from cloud tags — tags can't split a shared cluster by tenant, meters can.
- **[SHOULD]** Add optional `cost-center` / `component` tags for finance rollups; keep the mandatory four small so compliance stays ~100%.
- **[SHOULD]** Reconcile tag coverage weekly; untagged spend > tolerance is a defect. Untagged cost is unallocatable cost, and unallocatable cost silently subsidizes heavy tenants.

## Showback vs chargeback

Two operating models for putting cost in front of the people who cause it.

| | **Showback** | **Chargeback** |
|---|---|---|
| What | Report each team/tenant's cost; no money moves | Bill the cost to the team's/tenant's budget |
| Accountability | Visibility, social pressure | Hard budget ownership |
| Prereq | Tagging + allocation ([01](01-per-tenant-cost-model.md)) | Same, plus org buy-in and trusted numbers |
| Risk | Ignored if no consequences | Gaming/disputes if allocation is coarse or wrong |
| Good first step | ✅ Start here | Graduate to this once numbers are trusted |

- **[SHOULD]** Start with **showback** — per-team and per-tenant cost visible on an internal dashboard beside revenue ([../feature-blueprints/customer-analytics-churn.md](../feature-blueprints/customer-analytics-churn.md)). Most waste dies under visibility alone.
- **[SHOULD]** Move to **chargeback** only when allocation is trusted (reconciles to the bill) — otherwise you'll spend more on disputes than you save.
- **[MUST]** Whichever model, the *tenant-facing* commercial consequence flows through **metered billing** ([../feature-blueprints/subscription-billing.md](../feature-blueprints/subscription-billing.md)), never an ad-hoc invoice.

## Budgets & anomaly alerts

- **[MUST]** Set a **budget per `env` and per `service`** with threshold alerts (e.g. 50/80/100% of forecast), so a runaway is caught **before** the invoice (Domain 16 **[MUST]**).
- **[MUST]** Enable **anomaly detection** on spend, per service and per metered driver — a spike in LLM tokens, egress, or message volume alerts within hours, not at month-end. The biggest bills come from *new* behavior, which a fixed budget threshold misses until late.
- **[SHOULD]** Also alert on **per-tenant cost anomalies** — a tenant whose daily cost jumps (retry storm, abuse, misconfigured integration). This is both a cost control and a security signal (compromised credential running up spend).
- **[SHOULD]** Route budget/anomaly alerts to the `owner` tag's team, not a black-hole channel.

## Hard guardrails on unbounded-cost paths

Alerts tell you **after** money is spent. Guardrails stop it **at spend time**. Domain 16 **[MUST]**: hard guardrails on every unbounded-cost path, enforced **server-side and per tenant**. Concrete caps:

| Unbounded path | Hard guardrail | Enforced |
|---|---|---|
| **AI / LLM tokens** | Per-tenant per-day token cap; max tokens/request; model allow-list; kill-switch. See [../standards-kb/15-ai-llm-governance.md](../standards-kb/15-ai-llm-governance.md). | server-side, per tenant, pre-call |
| **Job / worker concurrency** | Max concurrent jobs per tenant; queue depth cap; per-tenant rate limit | server-side, per tenant |
| **Storage** | Per-tenant storage quota; reject/tier writes past cap; lifecycle-expire temp data | at write path |
| **Egress / bandwidth** | Per-tenant egress cap / throttle; response-size limits; pagination caps | at response path |
| **Messaging** | Per-tenant per-period send ceiling (SMS/email/push); throttle + alert on breach | at send path |
| **Third-party per-call fees** | Per-tenant call budget/day; cache deterministic results; circuit-break on error/cost spike | at call site |
| **Compute / requests** | Per-tenant API rate limits + quotas (ties to [../standards-kb/16-finops-cost.md](../standards-kb/16-finops-cost.md) and reliability) | edge / gateway |

- **[MUST]** Guardrails are **per tenant** so one tenant's overrun cannot exhaust shared budget or degrade others.
- **[MUST]** Guardrails are **server-side** — never trust a client-side cap. A compromised credential or an unbounded loop must hit a hard server ceiling.
- **[MUST]** A cap breach has a **defined behavior** consistent with the billing overage policy (`block` / `allow_and_bill` / `soft_warn` from [../feature-blueprints/subscription-billing.md](../feature-blueprints/subscription-billing.md)) — the guardrail and the entitlement quota are the same limit, enforced in one place.
- **[SHOULD]** Provide a **global kill-switch** per expensive driver (especially LLM) to stop spend during an incident without a deploy.
- **[SHOULD]** Cache deterministic outputs of metered third-party/LLM calls per tenant to cut repeat spend (Domain 16 **[SHOULD]**).

## Non-prod scale-down & efficiency

Non-production is pure cost with no revenue — the cheapest savings in FinOps.

- **[MUST]** **Scale non-prod (`env` ≠ `prod`) to zero off-hours** — nights/weekends. A dev/staging fleet running 168 hrs/week when used ~50 is ~70% waste. Schedule stop/start on the `env` tag.
- **[SHOULD]** **Ephemeral preview environments** (`env=preview`) auto-expire on PR close/TTL — no orphaned stacks.
- **[SHOULD]** Continuously find and remove **idle/orphaned resources** (unattached volumes, dormant instances, forgotten environments) via the `owner`/`env` tags (Domain 16 **[MUST]**).
- **[SHOULD]** Right-size prod to actual utilization; committed/reserved capacity for steady baseline, spot/preemptible for interruptible batch (Domain 16 **[SHOULD]**).

### Example — messaging retry storm

A misconfigured retry loop starts re-sending a notification. **Without guardrails:** anomaly detection alerts at month-end after a five-figure SMS bill. **With this KB:** the per-tenant send ceiling **blocks** further sends at the send path within the tenant's cap, a per-tenant cost anomaly alert fires within the hour to the `owner` team, and the LLM/messaging kill-switch is available if it spreads — bounded loss, same day.

## Acceptance checklist

- [ ] Every prod resource carries `owner`/`env`/`service`/`tenant`; untagged provisioning is denied; coverage reconciled weekly.
- [ ] Shared-resource per-tenant attribution comes from app usage meters ([01](01-per-tenant-cost-model.md)), not cloud tags.
- [ ] Showback dashboard live (per team + per tenant, beside revenue); chargeback only once numbers reconcile.
- [ ] Budgets per env & per service with threshold alerts; anomaly detection per service, per driver, and per tenant.
- [ ] Every unbounded-cost path (LLM/concurrency/storage/egress/messaging/third-party/requests) has a hard, per-tenant, server-side cap with defined breach behavior matching the billing overage policy.
- [ ] Global kill-switch exists per expensive driver (esp. LLM).
- [ ] Non-prod scales to zero off-hours; preview envs auto-expire; idle/orphaned resources reaped.
