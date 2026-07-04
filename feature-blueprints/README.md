# Feature Blueprints

Reusable, **project-agnostic** specs for features that most enterprise SaaS apps need. Each blueprint is *buildable*: it gives you a data model, key screens, rules, the governing standards, and an acceptance checklist — enough to implement directly.

Use `[PROJECT]` as a placeholder for your product/tenant naming.

## Index

| Blueprint | What you get |
| --- | --- |
| [report-builder.md](report-builder.md) | Drag-and-drop report/format designer over a governed metadata layer; save/share/schedule/export. |
| [rbac-admin.md](rbac-admin.md) | Roles, permissions, RBAC + ABAC policy admin, permission matrix, custom roles, delegation. |
| [subscription-billing.md](subscription-billing.md) | Plans → entitlements → quotas → subscription lifecycle, metered usage, proration, dunning, tax, webhooks. |
| [customer-analytics-churn.md](customer-analytics-churn.md) | MRR/ARR, churn, retention/cohorts, activation, LTV/CAC, at-risk scoring, customer 360. |
| [audit-log-and-activity.md](audit-log-and-activity.md) | Immutable audit log viewer, user activity feed, consented impersonation ("view as"). |
| [admin-console.md](admin-console.md) | System settings, master-data + bulk import/export, feature flags, tenant config, integration health. |

## How to use a blueprint

1. **Pick the blueprint** that matches the feature you're building.
2. **Copy the Data model + Key screens** sections into your project KB, then adapt names/fields to your domain. Keep the tenant-scoping, PK, and audit columns — they are load-bearing.
3. **Read the linked standards** (`../standards-kb/*`) and data layers (`../data-kb/*`) before you design the schema or queries. Blueprints defer to those documents where they overlap.
4. **Treat `[MUST]` as blocking** and `[SHOULD]` as strong defaults you deviate from only with a reason.
5. **Use the Acceptance checklist** as your definition-of-done and as the seed for QA/test cases.

## Conventions used in every blueprint

- **Tenant-scoped**: every business table carries `tenant_id` and is filtered by it on every query. See [`../standards-kb/01-architecture-multitenancy.md`](../standards-kb/01-architecture-multitenancy.md).
- **Standard audit columns** on every table: `id` (PK), `tenant_id`, `created_at`, `created_by`, `updated_at`, `updated_by`, `deleted_at` (soft delete), `version` (optimistic lock).
- **IDs**: opaque, non-sequential (UUID/ULID). Never leak sequential integers across a tenant boundary.
- **Money**: store minor units (integer) + ISO-4217 `currency`; never floats. See [`../standards-kb/16-finops-cost.md`](../standards-kb/16-finops-cost.md).
- **Time**: UTC in storage, tenant/user timezone at the edge.
- **States**: every screen defines **loading / empty / error** explicitly. See [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md).

## Canonical standards referenced

- [`../standards-kb/01-architecture-multitenancy.md`](../standards-kb/01-architecture-multitenancy.md)
- [`../standards-kb/04-security.md`](../standards-kb/04-security.md)
- [`../standards-kb/06-data-management.md`](../standards-kb/06-data-management.md)
- [`../standards-kb/07-observability-aiops.md`](../standards-kb/07-observability-aiops.md)
- [`../standards-kb/10-api-integration.md`](../standards-kb/10-api-integration.md)
- [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md)
- [`../standards-kb/16-finops-cost.md`](../standards-kb/16-finops-cost.md)
