# Admin KB — the Admin Area (project-agnostic)

The reusable blueprint for an enterprise app's **admin/back-office area**. Every project that
uses this kit inherits this checklist, so the admin panel is never an afterthought.

> **Reference, don't move:** several admin surfaces already exist as
> [feature blueprints](../feature-blueprints/README.md) — this folder adds the **admin-only**
> surfaces (mostly log/monitoring screens) and indexes everything into one Admin Area. Nothing
> is duplicated; each entry links to its source spec.

The **rules** for what every screen must log live in the NFR taxonomy:
[../standards-kb/20-logging-audit-and-traceability.md](../standards-kb/20-logging-audit-and-traceability.md).
The **Admin Auditor** agent ([../agents/admin-auditor.md](../agents/admin-auditor.md)) owns keeping the two in sync.

## The Admin Area — every surface, grouped

### Identity & access
| Surface | Spec | New? |
|---|---|---|
| Users & RBAC management | [../feature-blueprints/rbac-admin.md](../feature-blueprints/rbac-admin.md) | existing |
| Login history & sessions | [login-history-and-sessions.md](login-history-and-sessions.md) | ➕ new |
| API keys & tokens | [api-keys-and-tokens.md](api-keys-and-tokens.md) | ➕ new |
| Security center | [security-center.md](security-center.md) | ➕ new |

### Logs & observability
| Surface | Spec | New? |
|---|---|---|
| Audit log viewer & activity feed | [../feature-blueprints/audit-log-and-activity.md](../feature-blueprints/audit-log-and-activity.md) | existing |
| Application / error log explorer | [log-explorer.md](log-explorer.md) | ➕ new |
| Data-access (read) logs | [data-access-logs.md](data-access-logs.md) | ➕ new |
| Background jobs & queue monitor | [job-and-queue-monitor.md](job-and-queue-monitor.md) | ➕ new |
| Communication / delivery logs | [communication-logs.md](communication-logs.md) | ➕ new |
| Integration / webhook logs | [../feature-blueprints/integrations-and-webhooks.md](../feature-blueprints/integrations-and-webhooks.md) | existing |
| API usage & analytics (access logs, frequency, rate-limit hits) | [api-usage-and-analytics.md](api-usage-and-analytics.md) | ➕ new |

### Tenant & billing
| Surface | Spec | New? |
|---|---|---|
| Tenant / organization management & provisioning | [tenant-org-management.md](tenant-org-management.md) | ➕ new |
| Subscription, plans & entitlements | [../feature-blueprints/subscription-billing.md](../feature-blueprints/subscription-billing.md) | existing |

### Config & content
| Surface | Spec | New? |
|---|---|---|
| System settings & master data | [../feature-blueprints/admin-console.md](../feature-blueprints/admin-console.md) | existing |
| Feature flags | [../feature-blueprints/feature-flags-and-experimentation.md](../feature-blueprints/feature-flags-and-experimentation.md) | existing |
| Notification templates | [../feature-blueprints/notifications-center.md](../feature-blueprints/notifications-center.md) | existing |
| Localization & content | [localization-and-content.md](localization-and-content.md) | ➕ new |
| Announcements & maintenance banners | [announcements-and-maintenance.md](announcements-and-maintenance.md) | ➕ new |

### Compliance & data
| Surface | Spec | New? |
|---|---|---|
| Consent & DSR (data-subject request) center | [consent-and-dsr-center.md](consent-and-dsr-center.md) | ➕ new |
| Retention & erasure queue | [retention-and-erasure.md](retention-and-erasure.md) | ➕ new |
| Export / import history | [../feature-blueprints/import-export-pipeline.md](../feature-blueprints/import-export-pipeline.md) | existing |

### Reports, support & overview
| Surface | Spec | New? |
|---|---|---|
| Report builder | [../feature-blueprints/report-builder.md](../feature-blueprints/report-builder.md) | existing |
| Analytics / MIS / churn | [../feature-blueprints/customer-analytics-churn.md](../feature-blueprints/customer-analytics-churn.md) | existing |
| Helpdesk & impersonation | [../feature-blueprints/support-and-impersonation.md](../feature-blueprints/support-and-impersonation.md) | existing |
| Admin dashboard & system health | [../feature-blueprints/admin-console.md](../feature-blueprints/admin-console.md) | existing |

## Every admin surface must

- Be **RBAC-gated** (admin roles only) and **tenant-scoped** — see [../standards-kb/04-security.md](../standards-kb/04-security.md).
- **Emit its own audit entries** (admin actions are sensitive) per [../standards-kb/20-logging-audit-and-traceability.md](../standards-kb/20-logging-audit-and-traceability.md).
- **Mask PII/secrets** in every log view; export is itself an audited event.
- Have designed **loading / empty / error** states — see [../standards-kb/11-frontend-ux.md](../standards-kb/11-frontend-ux.md).
