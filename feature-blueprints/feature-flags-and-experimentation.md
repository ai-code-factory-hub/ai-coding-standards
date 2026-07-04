# Feature Blueprint · Feature Flags & Experimentation

Runtime flags (release / ops / permission / experiment) with targeting, kill-switch, lifecycle + cleanup, A/B assignment + metrics, and a full audit of every flag change.

## What it is / when you need it

Feature flags decouple **deploy** from **release**: ship code dark, turn it on for a tenant/percentage/role, and kill it instantly if it misbehaves. The same machinery powers **operational** toggles (throttle a subsystem), **permission/entitlement** gating, and **experiments** (A/B tests with metrics). This blueprint covers the flag registry, targeting rules, deterministic experiment assignment, metrics, and — critically — flag **lifecycle and cleanup** so flags don't rot into permanent tech debt. Every change is audited.

## Data model

> All tables carry the standard audit columns (`id`, `tenant_id`, `created_at`, `created_by`, `updated_at`, `updated_by`, `deleted_at`, `version`) and are filtered by `tenant_id` on every query. Global (platform) flags use a reserved system tenant.

| Entity | Key fields | Notes |
| --- | --- | --- |
| `feature_flag` | `key` (unique), `name`, `type` (release/ops/permission/experiment), `description`, `default_value`, `value_type` (bool/string/number/json), `status` (active/archived), `owner`, `temporary` (bool), `expires_at`, `stale_after` | The flag definition. `temporary` + `expires_at` drive cleanup. |
| `flag_environment_state` | `flag_id` FK, `environment` (dev/staging/prod), `enabled`, `rollout_percentage`, `serve_value` | Per-environment on/off + default served value. |
| `flag_targeting_rule` | `flag_id` FK, `environment`, `priority`, `conditions` (tenant/role/attribute predicates), `serve_value`, `percentage` | Ordered rules: first match wins; supports % bucketing. |
| `flag_evaluation_context` (logical) | `tenant_id`, `user_id`, `role`, `attributes{}` | The context passed to `evaluate(key, context)`. Not persisted per-eval (sampled). |
| `experiment` | `flag_id` FK, `hypothesis`, `variants[]` (name, weight), `primary_metric`, `guardrail_metrics[]`, `status` (draft/running/paused/concluded), `start_at`, `end_at`, `winner` | An A/B/n test bound to a flag. |
| `experiment_assignment` | `experiment_id` FK, `subject_id` (user/tenant), `variant`, `assigned_at`, `sticky_bucket` | Deterministic, sticky assignment per subject. |
| `experiment_metric_event` | `experiment_id`, `subject_id`, `variant`, `metric_key`, `value`, `occurred_at` | Exposure + conversion events for analysis. |
| `flag_change_audit` | `flag_id`, `environment`, `field`, `old_value`, `new_value`, `actor`, `reason`, `changed_at` | Append-only audit of every flag/targeting/experiment change. |

## Key screens & UX

| Screen | Primary fields / actions | States |
| --- | --- | --- |
| **Flag list** | Search/filter by type/status/owner/staleness; columns: key, type, prod state, rollout %, owner, last-changed, stale badge | loading / empty / error |
| **Flag detail** | Per-environment toggle + rollout %, ordered targeting rules editor (add/reorder/delete), served-value preview for a test context, change history, **kill-switch** button | active / archived; guarded confirm for prod changes |
| **Targeting rule editor** | Condition builder (tenant / % / role / attribute), serve-value, percentage split, priority | valid / invalid predicate |
| **Experiment console** | Variants + weights, primary + guardrail metrics, start/pause/conclude, live results (assignment counts, metric deltas, significance), declare winner → roll out | draft / running / paused / concluded |
| **Cleanup / staleness** | Flags past `expires_at` or `stale_after`, unreferenced-in-code flags, "archive" / "remove" actions with owner nudges | empty / has-stale |
| **Audit view** | Filter by flag/actor/date; who changed what, when, why | loading / empty |

## Rules & logic

- **[MUST]** Flag evaluation is a **fast, local, side-effect-free** `evaluate(key, context)` — served from an in-process cache with streaming/polled updates. A flag service outage **fails safe to `default_value`**, never breaks the request path.
- **[MUST]** Targeting rules are **ordered, first-match-wins**; percentage rollouts use **deterministic hashing** (`hash(flagKey + subjectId)`) so a subject's bucket is stable across evaluations.
- **[MUST]** Every flag/ops toggle has a **kill-switch**: a single action that forces the safe value in prod immediately, independent of rollout config.
- **[MUST]** **Permission/entitlement flags do not replace authorization** — they gate feature availability, but the server still enforces RBAC/ABAC. A flag must never be the only thing standing between a user and privileged data.
- **[MUST]** Every change to a flag, targeting rule, or experiment is **audited** (actor, before/after, reason) and immutable.
- **[MUST]** **Temporary flags carry an expiry/owner**; the system surfaces stale and past-due flags for cleanup so flags are removed, not left forever. Release flags are expected to be short-lived.
- **[MUST]** Experiment **assignment is sticky and deterministic** per subject; exposures are logged; assignment is skipped for excluded/opted-out subjects.
- **[SHOULD]** Guardrail metrics can **auto-pause** an experiment or auto-trigger a kill-switch on regression.
- **[SHOULD]** Changes are **environment-scoped**; prod changes require confirmation and optionally approval.
- **[SHOULD]** Provide SDK parity across services/clients and a "why did I get this value?" evaluation explainer for debugging.

## Applicable standards

- [`../standards-kb/05-reliability-resilience.md`](../standards-kb/05-reliability-resilience.md) — fail-safe evaluation, kill-switch, graceful degradation.
- [`../standards-kb/04-security.md`](../standards-kb/04-security.md) — flags don't replace authz; change control on prod flags.
- [`../standards-kb/07-observability-aiops.md`](../standards-kb/07-observability-aiops.md) — exposure/metric events, guardrails, flag evaluation telemetry.
- [`../standards-kb/17-nfr-coverage-gaps.md`](../standards-kb/17-nfr-coverage-gaps.md) — flag debt / cleanup as a tracked NFR.
- [`../standards-kb/10-api-integration.md`](../standards-kb/10-api-integration.md) — flag/experiment SDK + admin API contracts.
- [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md) — admin screen states, guarded prod actions.
- [`../feature-blueprints/README.md`](../feature-blueprints/README.md) — shared conventions.

## Acceptance checklist

- [ ] Four flag types supported: release, ops, permission, experiment.
- [ ] Evaluation is fast, local, side-effect-free, and fails safe to default on outage.
- [ ] Targeting by tenant / percentage / role / attribute; ordered first-match-wins rules.
- [ ] Percentage rollout uses deterministic hashing (stable buckets).
- [ ] Kill-switch forces the safe value in prod instantly.
- [ ] Permission flags gate availability but never replace server-side authz.
- [ ] Every flag/rule/experiment change is audited (actor, before/after, reason), immutable.
- [ ] Temporary flags have owner + expiry; stale/past-due flags surfaced for cleanup.
- [ ] Experiments: sticky deterministic assignment, exposure logging, variant weights, primary + guardrail metrics, conclude→roll-out.
- [ ] Guardrail regressions can auto-pause / trigger kill-switch.
- [ ] Prod changes are environment-scoped and confirmed.
- [ ] All tables tenant-scoped with standard audit columns.
