# Feature Blueprint · RBAC + Policy Admin

> Roles, permissions, and a policy-admin UI combining **RBAC** (role → permission) with **ABAC** (attribute-condition rules), plus role assignment, a permission matrix, custom roles, and delegation.

## What it is / when you need it

The authorization control plane for your app: who can do what, under which conditions. RBAC handles the common case (assign a role, inherit its permissions); ABAC layers attribute conditions on top (e.g. "approve invoices only for own branch and amount ≤ 50,000"). You need this the moment more than one kind of user exists, or a customer asks for "custom roles." See [`../standards-kb/04-security.md`](../standards-kb/04-security.md) — this blueprint implements its authorization requirements.

## Data model

Tenant-scoped, standard audit columns. System/global roles may be tenant-null but are read-only to tenants.

### Role

| Field | Type | Notes |
| --- | --- | --- |
| `key` | string | Stable machine key, unique per tenant |
| `name` | string | Display name |
| `description` | text | |
| `is_system` | bool | Built-in, not editable/deletable |
| `is_custom` | bool | Tenant-created |
| `parent_role_id` | fk? | Optional inheritance (permissions union up the chain) |
| `scope_type` | enum | `tenant` \| `org_unit` \| `resource` — where the role applies |

### Permission

| Field | Type | Notes |
| --- | --- | --- |
| `key` | string | `resource:action`, e.g. `invoice:approve` (globally defined) |
| `resource` | string | e.g. `invoice` |
| `action` | string | e.g. `approve`, `read`, `export` |
| `description` | text | |
| `is_dangerous` | bool | Requires extra confirmation / step-up auth |

RolePermission is the join (`role_id`, `permission_id`, optional `effect` = `allow`/`deny`).

### RoleAssignment
Grants a role to a principal, optionally scoped and time-boxed.

| Field | Type | Notes |
| --- | --- | --- |
| `principal_type` | enum | `user` \| `group` \| `service_account` |
| `principal_id` | fk | The grantee |
| `role_id` | fk | → Role |
| `scope_type` | enum | `tenant` \| `org_unit` \| `resource` |
| `scope_id` | string? | e.g. branch/org-unit id (null = whole tenant) |
| `valid_from` | timestamp? | Time-boxed grants |
| `valid_until` | timestamp? | Auto-expiry (e.g. delegation) |
| `granted_by` | fk | Delegator/admin |
| `is_delegation` | bool | Granted by another user, not an admin |

### PolicyRule (ABAC)
Attribute conditions that further constrain a permission.

| Field | Type | Notes |
| --- | --- | --- |
| `name` | string | |
| `permission_key` | string | The permission this rule guards |
| `effect` | enum | `allow` \| `deny` (deny wins) |
| `condition_json` | jsonb | Expression tree over subject/resource/env attributes |
| `priority` | int | Lower = evaluated first |
| `is_active` | bool | |

`condition_json` example: `{ "all": [ {"attr":"resource.branch_id","op":"eq","value":"subject.branch_id"}, {"attr":"resource.amount","op":"lte","value":50000} ] }`.

## Key screens & UX

1. **Roles list** — system + custom roles, member counts, inheritance badge. Actions: Create custom role, Duplicate, Edit, Archive (custom only). *Empty:* "No custom roles — start from a system role." *Error:* retry.
2. **Role editor** — name/description, parent role, and a **permission picker** grouped by resource; dangerous permissions flagged. Shows effective (inherited + direct) permissions distinctly.
3. **Permission matrix** — grid of Roles × Permissions with allow/deny/inherited cells; filter by resource; toggle inline; read-only for system roles. *Loading:* skeleton grid. Large-grid virtualization.
4. **Assignments** — assign roles to users/groups/service accounts with scope + validity window; bulk assign; shows expiring grants. 
5. **Policy (ABAC) rules** — list + condition builder (subject/resource/environment attributes, operators, all/any groups), priority ordering, deny-wins explainer, and a **"test decision"** simulator (pick subject + resource → allow/deny + which rule decided).
6. **Delegation** — a user with delegation rights grants a subset of their own access to another user for a time window; auto-expires; fully audited.
7. **Access review** — periodic view of who has what, flag stale/over-broad grants, one-click revoke.

## Rules & logic

- [MUST] **Deny overrides allow** across RBAC and ABAC; explicit deny always wins.
- [MUST] Enforce authorization **server-side** on every request; the UI is a convenience, never the control.
- [MUST] Effective permission = union of role + inherited-role permissions, minus denies, then filtered by matching **ABAC PolicyRules** and **assignment scope/validity**.
- [MUST] System roles/permissions are immutable to tenants; custom roles cannot exceed the granting admin's own permissions (no privilege escalation).
- [MUST] Every role/permission/assignment/policy change is **audited** (who, before→after) — see [`../standards-kb/04-security.md`](../standards-kb/04-security.md).
- [MUST] Dangerous permissions (`is_dangerous`) require confirmation and may require step-up auth.
- [MUST] Delegated grants are time-boxed (`valid_until` required) and revocable; expiry is enforced at decision time, not by a batch job alone.
- [SHOULD] Cache permission decisions per request with a short TTL; invalidate on any policy/assignment change.
- [SHOULD] Provide a "test decision" simulator and a per-user "why can/can't I?" explainer.
- [SHOULD] Support groups/service accounts as first-class principals to avoid per-user sprawl.

## Applicable standards

- Authorization, least privilege, step-up auth, audit — [`../standards-kb/04-security.md`](../standards-kb/04-security.md)
- Tenant isolation of roles/scopes — [`../standards-kb/01-architecture-multitenancy.md`](../standards-kb/01-architecture-multitenancy.md)
- Policy-change events & audit trail — [`../standards-kb/07-observability-aiops.md`](../standards-kb/07-observability-aiops.md)
- Admin screens/UX — [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md)
- Authorization on API surface — [`../standards-kb/10-api-integration.md`](../standards-kb/10-api-integration.md)

## Acceptance checklist

- [ ] Roles (system + custom) with optional inheritance; custom cannot exceed granter's rights.
- [ ] Permission matrix renders/edits allow/deny/inherited correctly and is read-only for system roles.
- [ ] Assignments support principal type, scope, and validity window; expiry enforced at decision time.
- [ ] ABAC PolicyRules evaluate with deny-wins and correct priority; condition builder works.
- [ ] "Test decision" simulator returns the deciding rule and outcome.
- [ ] Authorization enforced server-side on every request; UI reflects the same decision.
- [ ] Delegation is time-boxed, revocable, and audited.
- [ ] All authz changes produce before→after audit records; dangerous perms require confirmation/step-up.
