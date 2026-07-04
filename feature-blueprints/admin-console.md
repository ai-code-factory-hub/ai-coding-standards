# Feature Blueprint · Admin Console

> The operational control panel: system settings, master-data management with bulk import/export, a feature-flag surface, tenant configuration, and integration health/status with "test connection".

## What it is / when you need it

The place where admins configure the product without engineering: tune settings, manage reference/master data, flip feature flags, set tenant-level config, and check that integrations are healthy. Every enterprise app grows one; building it deliberately (audited, permission-gated, tenant-scoped) beats a pile of ad-hoc admin pages.

## Data model

Tenant-scoped (plus a `scope` = `system`/`tenant` distinction where global defaults exist); standard audit columns. Every change here is high-value — audit all of it (see [audit-log-and-activity.md](audit-log-and-activity.md)).

### SystemSetting

| Field | Type | Notes |
| --- | --- | --- |
| `key` | string | Namespaced, e.g. `billing.invoice_prefix` |
| `scope` | enum | `system` \| `tenant` |
| `value_json` | jsonb | Typed value |
| `data_type` | enum | `string` \| `number` \| `bool` \| `enum` \| `json` |
| `is_secret` | bool | Masked in UI, encrypted at rest |
| `validation_json` | jsonb | Constraints for the editor |
| `default_json` | jsonb | Fallback |

### MasterDataEntity + MasterDataRecord
Generic reference/lookup data (units, categories, tax rates, branches, etc.).

| Field | Type | Notes |
| --- | --- | --- |
| `entity_key` | string | e.g. `tax_rate`, `branch` |
| `code` | string | Business code, unique per entity+tenant |
| `label` | string | Display |
| `attributes_json` | jsonb | Entity-specific fields |
| `is_active` | bool | Soft enable/disable |
| `sort_order` | int | |
| `source` | enum | `manual` \| `import` \| `sync` |

### FeatureFlag

| Field | Type | Notes |
| --- | --- | --- |
| `key` | string | e.g. `new_dashboard` |
| `description` | text | |
| `type` | enum | `boolean` \| `percentage` \| `targeted` |
| `default_value` | bool | |
| `rollout_pct` | int? | For `percentage` |
| `targeting_json` | jsonb? | Rules (tenant/user/attribute) |
| `is_permanent` | bool | Ops flag vs temporary experiment |

FeatureFlagOverride (`flag_key`, `scope` = `tenant`/`user`, `scope_id`, `value`, `reason`).

### TenantConfig

| Field | Type | Notes |
| --- | --- | --- |
| `tenant_id` | fk | |
| `timezone` | string | IANA |
| `locale` | string | |
| `currency` | string | ISO-4217 |
| `branding_json` | jsonb | Logo, colors |
| `fiscal_year_start` | int | Month 1–12 |
| `config_json` | jsonb | Misc tenant knobs |

### Integration + IntegrationHealthCheck

| Field | Type | Notes |
| --- | --- | --- |
| `provider_key` | string | e.g. `payment_gateway`, `sms`, `smtp` |
| `name` | string | |
| `config_json` | jsonb | Non-secret config |
| `credentials_ref` | string | Pointer to secrets store (never inline secrets) |
| `status` | enum | `connected` \| `degraded` \| `error` \| `disabled` |
| `last_checked_at` | timestamp | |
| `last_result_json` | jsonb | Latest test-connection result |

### BulkJob (import/export)

| Field | Type | Notes |
| --- | --- | --- |
| `type` | enum | `import` \| `export` |
| `entity_key` | string | Target master-data/entity |
| `file_uri` | string | Source/result (expiring signed URL) |
| `status` | enum | `queued` \| `validating` \| `running` \| `completed` \| `failed` \| `partial` |
| `total_rows` / `success_rows` / `error_rows` | int | |
| `errors_uri` | string? | Downloadable error report |
| `dry_run` | bool | Validate without committing |

## Key screens & UX

1. **Settings** — grouped, searchable settings with typed editors and inline validation; secrets masked with reveal-on-permission; "reset to default"; diff-on-save confirmation for sensitive keys. *States:* loading skeleton, save error toast, unsaved-changes guard.
2. **Master data management** — per entity: searchable/sortable table, add/edit/deactivate, and **bulk import/export**. Import flow: upload → **dry-run validation** (row-level errors downloadable) → commit → job progress → summary. Export to CSV/XLSX. *Empty:* "No records — import or add one." *Error:* row-level error report.
3. **Feature flags** — list with type/rollout/targeting, per-tenant/per-user overrides with a required reason, and current effective value; toggle with confirmation for permanent/ops flags.
4. **Tenant configuration** — timezone, locale, currency, branding, fiscal year, and misc knobs; live preview of branding.
5. **Integrations** — provider cards showing status badge, last-checked, and a **"Test connection"** button that runs a live check and shows a clear pass/fail with diagnostics; edit config; credentials handled via secrets store, never shown.
6. **Health / status** — system + integration health dashboard, recent bulk jobs, and links into the audit log for recent admin changes.

## Rules & logic

- [MUST] Every admin action is **permission-gated** (see [rbac-admin.md](rbac-admin.md)) and **audited** with before→after (see [audit-log-and-activity.md](audit-log-and-activity.md)).
- [MUST] Secrets are stored in a secrets manager and referenced by pointer (`credentials_ref`); never rendered or logged. Masked in UI, encrypted at rest.
- [MUST] Bulk import is **validated dry-run first**; commit is transactional or produces a precise per-row error report (`partial` allowed only with an error file).
- [MUST] Settings and config are **tenant-scoped**; `system` scope provides defaults a tenant may override only where allowed.
- [MUST] "Test connection" performs a real, side-effect-free check and reports actionable diagnostics; a failing integration surfaces as `error`/`degraded` in health, not silently.
- [MUST] Feature-flag evaluation is deterministic and resolves overrides in a defined order (user > tenant > percentage > default).
- [SHOULD] Bulk export and import files use expiring signed URLs and honor retention.
- [SHOULD] Sensitive settings changes require confirmation and, where policy dictates, step-up auth.
- [SHOULD] Provide a settings/config change history view (backed by the audit log).

## Applicable standards

- Tenant config isolation, defaults vs overrides — [`../standards-kb/01-architecture-multitenancy.md`](../standards-kb/01-architecture-multitenancy.md)
- Secrets handling, permission gating, step-up — [`../standards-kb/04-security.md`](../standards-kb/04-security.md)
- Master data, bulk import/export, validation, retention — [`../standards-kb/06-data-management.md`](../standards-kb/06-data-management.md)
- Integration config, test connection, webhooks — [`../standards-kb/10-api-integration.md`](../standards-kb/10-api-integration.md)
- Integration/system health, status — [`../standards-kb/07-observability-aiops.md`](../standards-kb/07-observability-aiops.md)
- Admin UX/states — [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md)
- Cost of bulk jobs/exports — [`../standards-kb/16-finops-cost.md`](../standards-kb/16-finops-cost.md)

## Acceptance checklist

- [ ] Settings editor is typed, validated, tenant-scoped, with secrets masked and audited on save.
- [ ] Master-data CRUD + bulk import with dry-run validation and a per-row error report.
- [ ] Bulk export/import use expiring signed URLs and honor retention.
- [ ] Feature-flag surface resolves overrides deterministically (user > tenant > % > default).
- [ ] Tenant config (tz/locale/currency/branding/fiscal) editable with live preview.
- [ ] Integrations show health status; "Test connection" runs a real check with diagnostics.
- [ ] Secrets stored in secrets manager, referenced by pointer, never rendered or logged.
- [ ] Every admin action is permission-gated and audited with before→after.
