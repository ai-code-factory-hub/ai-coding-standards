# Admin · Tenant / Organization Management & Provisioning

> Create, configure, and lifecycle tenants/organizations — with a provisioning workflow (schema/DB, seed, admin invite), plan/entitlement + status, region/data-residency, per-tenant feature flags, and branding lock.

## What it is / when you need it

The control plane for the multi-tenant product itself. Where an operator (platform/super-admin) onboards a new customer organization, provisions its isolated data home, assigns a plan and entitlements, sets its region for data residency, flips tenant-level feature flags, and manages its lifecycle (trial → active → suspended → offboarded). Every multi-tenant SaaS needs this the moment it has more than one customer; doing it deliberately — provisioning as an auditable, resumable workflow rather than manual SQL — is the difference between clean onboarding and a support fire. This is the operational face of [`../standards-kb/01-architecture-multitenancy.md`](../standards-kb/01-architecture-multitenancy.md).

## Data model

Platform-scoped for the operator view, but every child record is **strictly tenant-partitioned**. Standard audit columns throughout.

### Tenant / Organization

| Field | Type | Notes |
| --- | --- | --- |
| `id` | uuid | PK; the isolation key on every table |
| `slug` | string | URL/subdomain-safe, unique |
| `display_name` | string | Legal/brand name |
| `status` | enum | `trial` \| `active` \| `suspended` \| `provisioning` \| `offboarding` \| `archived` |
| `plan_id` | fk | → Plan (see [`../feature-blueprints/subscription-billing.md`](../feature-blueprints/subscription-billing.md)) |
| `region` | enum | Data-residency region, e.g. `eu` \| `us` \| `in` — **immutable after provisioning** |
| `isolation_model` | enum | `shared_schema` \| `schema_per_tenant` \| `db_per_tenant` |
| `data_home_ref` | string | Pointer to the schema/DB/cluster hosting this tenant |
| `parent_org_id` | fk? | For org hierarchies / resellers |
| `branding_locked` | bool | If true, tenant cannot override platform branding |
| `trial_ends_at` | timestamp? | |
| `created_by` / `created_at` | mixed | Operator + timestamp |

### Entitlement (tenant-level)

| Field | Type | Notes |
| --- | --- | --- |
| `tenant_id` | fk | |
| `feature_key` | string | e.g. `advanced_reports`, `sso`, `api_access` |
| `type` | enum | `boolean` \| `numeric_limit` \| `metered` |
| `value` | jsonb | Enabled flag or limit value |
| `source` | enum | `plan` \| `override` — overrides win, and are audited |

### TenantFeatureFlag (override)

| Field | Type | Notes |
| --- | --- | --- |
| `tenant_id` | fk | |
| `flag_key` | string | → platform FeatureFlag |
| `value` | jsonb | Per-tenant value |
| `reason` | string | Required justification |
| `expires_at` | timestamp? | Optional auto-revert |

### ProvisioningJob

| Field | Type | Notes |
| --- | --- | --- |
| `tenant_id` | fk | |
| `status` | enum | `queued` \| `running` \| `completed` \| `failed` \| `rolled_back` |
| `steps_json` | jsonb | Ordered steps + per-step status |
| `current_step` | string | e.g. `create_schema`, `run_migrations`, `seed_reference_data`, `invite_admin` |
| `idempotency_key` | string | Safe to retry/resume |
| `error_json` | jsonb? | Failure detail for the failed step |
| `completed_at` | timestamp? | |

### TenantAdminInvite

| Field | Type | Notes |
| --- | --- | --- |
| `tenant_id` | fk | |
| `email` | string | First admin |
| `role` | enum | `owner` \| `admin` |
| `token_hash` | string | One-time, hashed |
| `status` | enum | `pending` \| `accepted` \| `expired` \| `revoked` |
| `expires_at` | timestamp | |

## Key screens & UX

1. **Tenants list** — searchable/filterable table: `display_name`, `slug`, `status` badge, plan, region, created date, user count. Filter by status/plan/region. *Loading:* skeleton rows. *Empty:* "No tenants yet — create one." *Error:* retry banner.
2. **Create tenant (provisioning wizard)** — steps: identity (name/slug) → region (**warned: immutable**) → plan/entitlements → isolation model → first-admin email. On submit, kicks off a **ProvisioningJob** with a live step tracker (create data home → run migrations → seed reference data → send admin invite). Each step shows pending/running/done/failed; a failed step offers **retry** or **rollback**, never leaves a half-built tenant silently. *State:* tenant sits in `provisioning` until complete.
3. **Tenant detail** — overview, entitlements (plan vs override, source-tagged), feature-flag overrides (with reason + optional expiry), region/residency (read-only), branding-lock toggle, and lifecycle actions (suspend/reactivate/offboard).
4. **Entitlements & plan** — assign/change plan, override individual entitlements with a required reason; changes surface a before→after diff and confirmation.
5. **Feature flags (per tenant)** — override platform flags for this tenant with justification and optional auto-revert; shows effective value.
6. **Suspend / reactivate** — suspend restricts access while **retaining data**; requires reason; confirmation names the tenant and warns users will be locked out. Reactivate restores access.
7. **Offboard** — guarded, multi-confirm flow: export-then-delete per [`retention-and-erasure.md`](retention-and-erasure.md); moves tenant to `offboarding` → `archived`. Irreversible after grace window.

## Rules & logic

- [MUST] Provisioning is an **idempotent, resumable, audited workflow** — create data home, run migrations, seed reference data, invite the first admin — with per-step status; a failed step is retryable or rolled back, never leaving a partial tenant.
- [MUST] `region`/data-residency is set at creation and **immutable afterward**; all of a tenant's data (including backups/logs) stays in-region per [`../standards-kb/01-architecture-multitenancy.md`](../standards-kb/01-architecture-multitenancy.md).
- [MUST] Every tenant child record is **partitioned by `tenant_id`**; no query path can cross tenants, and the operator console's cross-tenant reads are themselves audited.
- [MUST] All lifecycle and config actions (create/suspend/reactivate/offboard, plan/entitlement/flag changes) are **RBAC-gated** (platform operator roles) and **audited** with before→after per [`../standards-kb/20-logging-audit-and-traceability.md`](../standards-kb/20-logging-audit-and-traceability.md).
- [MUST] **Entitlement/flag overrides win over plan defaults**, are source-tagged, and require a reason; features are gated on entitlements, never on plan name (see [`../feature-blueprints/subscription-billing.md`](../feature-blueprints/subscription-billing.md)).
- [MUST] `suspend` restricts access but **retains data**; `offboard` follows export-then-delete with a grace window per [`retention-and-erasure.md`](retention-and-erasure.md) and [`../standards-kb/09-compliance-privacy.md`](../standards-kb/09-compliance-privacy.md).
- [MUST] First-admin invites use **one-time, hashed, expiring tokens**; accepting bootstraps the tenant's owner without exposing platform credentials.
- [SHOULD] Support **branding lock** so a tenant on a controlled plan cannot override platform theme; unlock is an audited entitlement.
- [SHOULD] Support org hierarchies (`parent_org_id`) for resellers/enterprise groups with inherited defaults.
- [SHOULD] Expose per-tenant health (user count, storage, last activity, plan status) for operator triage.

## Applicable standards

- Tenancy model, isolation, data residency, defaults vs overrides — [`../standards-kb/01-architecture-multitenancy.md`](../standards-kb/01-architecture-multitenancy.md)
- Provisioning/lifecycle audit trail — [`../standards-kb/20-logging-audit-and-traceability.md`](../standards-kb/20-logging-audit-and-traceability.md)
- Suspend/offboard retention & residency — [`../standards-kb/06-data-management.md`](../standards-kb/06-data-management.md), [`../standards-kb/09-compliance-privacy.md`](../standards-kb/09-compliance-privacy.md)
- Operator access control & step-up — [`../standards-kb/04-security.md`](../standards-kb/04-security.md)
- Provisioning as pipeline / migrations — [`../standards-kb/12-devops-cicd.md`](../standards-kb/12-devops-cicd.md)
- Operator UX / states — [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md)
- Related — [`../feature-blueprints/subscription-billing.md`](../feature-blueprints/subscription-billing.md), [`retention-and-erasure.md`](retention-and-erasure.md), [`../admin-kb/README.md`](../admin-kb/README.md)

## Acceptance checklist

- [ ] Create-tenant runs an idempotent, resumable provisioning job with per-step status (schema → migrations → seed → invite).
- [ ] A failed provisioning step is retryable or rolls back cleanly — no partial tenants.
- [ ] Region/data-residency is set once and immutable; all tenant data stays in-region.
- [ ] Every tenant child record is partitioned by `tenant_id`; cross-tenant operator reads are audited.
- [ ] Plan/entitlement/flag changes are source-tagged, reason-required, and show before→after.
- [ ] Suspend retains data and locks access; offboard follows export-then-delete with a grace window.
- [ ] First-admin invite uses a one-time, hashed, expiring token.
- [ ] Branding lock enforced per plan/entitlement; unlock is audited.
- [ ] All lifecycle/config actions are RBAC-gated and audited with before→after.
- [ ] Loading / empty / error states designed for list, wizard, and detail.
