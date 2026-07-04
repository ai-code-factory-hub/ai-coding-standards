# 01 · Architecture & Multi-Tenancy

Isolation model, tenant registry, identity separation, layered config, and platform billing for any enterprise multi-tenant SaaS.

## Tenancy model

- **[MUST]** Pick and document one isolation model per data class: **Silo** (DB per tenant — highest isolation, for enterprise/regulated tenants), **Bridge** (shared DB, schema per tenant), **Pool** (shared DB+schema, `tenant_id` discriminator).
- **[MUST]** Default to a **hybrid**: Pool for small tenants, Silo for large/enterprise/regulated tenants. Store the chosen model per tenant in a **tenant registry**.
- **[MUST]** Every tenant-scoped table has a non-null, indexed `tenant_id`; every query filters by it. Enforce with **DB Row-Level Security (RLS)** — never rely on app code alone.
- **[MUST]** Tenant context resolved once per request (subdomain/JWT claim/API key) into an immutable request-scoped value; background jobs carry tenant context explicitly.
- **[MUST NOT]** Allow any cross-tenant "list all" query outside an isolated, audited admin/ops path.
- **[SHOULD]** Tenant-prefix cache keys, object-storage paths, search indexes, and log fields (`tenant_{id}:...`).

> **Example (healthcare):** a hospital-owned diagnostics lab handling sensitive personal data is provisioned as a **Silo** (dedicated DB, regulated isolation), while a single-location clinic runs in the shared **Pool**.
>
> **Example (fintech/e-commerce):** an enterprise merchant with a compliance mandate gets a **Silo**; thousands of small storefronts share the **Pool** with `tenant_id` RLS. Same registry, same code path.

## Tenant registry (control plane)

- **[MUST]** A central registry stores: tenant id, name, isolation model, DB connection ref, **region/data residency**, subscription plan, status (active/suspended/trial), feature flags, and branding config.
- **[SHOULD]** New-tenant provisioning is a single automated workflow (create DB/schema, seed reference/domain data, create admin, send welcome).

## Identity vs. tenant data separation

- **[MUST]** Separate the identity/auth store (users, credentials, sessions, MFA, SSO links) from tenant business data. Model membership as **user ↔ tenant ↔ role** (one user may belong to multiple tenants).
- **[SHOULD]** The identity store is replaceable by an external IdP (Keycloak/Auth0/Cognito/Azure AD) without touching business schema.

> **Example (healthcare):** a locum specialist works across three labs; one identity, three tenant memberships, distinct roles per lab.
>
> **Example (fintech/e-commerce):** an accountant administers several client merchant accounts under one login, each with its own scoped role.

## Configuration management

- **[MUST]** Three config layers, later wins: **system defaults → tenant config → user preferences**. No tenant behaviour as a code branch. Config is typed and validated at write time; secrets only as references to a secrets manager.
- **[SHOULD]** Config changes are versioned and audited (who/what/when, old→new) and roll-back-able. A "Control Room / Setup" UI exposes safe config to tenant admins.

## Subscription & billing (of the SaaS platform itself)

- **[MUST]** Model: Plan → Entitlements/Features → Quotas/Limits → Subscription → Usage → Invoice/Payment. **Gate features by entitlement, not plan name.** Enforce quotas server-side (tenants, users, records/month, storage). Handle the full lifecycle (trial → active → past-due → suspended → cancelled → reactivated), audited. Use a PCI-compliant provider (Stripe/Razorpay/Adyen); verify and idempotently handle payment webhooks.

> ⚠️ Do not confuse this **platform subscription billing** (what the tenant pays you to use the SaaS) with any **in-product billing module** a tenant uses to invoice its own customers — different domains, different data.

See [Security](04-security.md) for the authorization invariants that keep tenants isolated, [Data Management](06-data-management.md) for the schema hygiene that backs `tenant_id`, and design tokens in [../design-system-kb/](../design-system-kb/README.md).

## Acceptance checklist

- [ ] Isolation model documented per data class and enforced with **DB Row-Level Security**.
- [ ] No cross-tenant data path outside an isolated, audited admin/ops route.
- [ ] Identity/auth store separated from tenant business data; membership modelled as user ↔ tenant ↔ role.
- [ ] Tenant registry records isolation model, region/residency, plan, status, and feature flags.
- [ ] Config is layered (system → tenant → user), typed, validated, and audited — no `if tenant == X` branches.
- [ ] Platform subscription features gated by **entitlement**, with quotas enforced server-side.
