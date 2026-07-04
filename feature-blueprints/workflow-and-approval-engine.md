# Feature Blueprint · Workflow & Approval Engine

> A generic, config-driven engine that runs workflows defined as **data** — states, transitions, guards, approvals, SLAs, escalations — so routing and sign-off are configured, never coded per workflow.

## What it is / when you need it

One reusable engine that executes **workflow definitions** authored as configuration (states + transitions + rules), not as bespoke code per business process. Any record that needs review, routing, or sign-off — an expense claim, a document, a refund, a leave request, a lab report — attaches to a workflow **instance** and moves through states as approvals and conditions are satisfied. You need this the moment a second approval flow appears: without it, every new "route X to Y when Z" becomes a code change and a deploy. With it, an admin draws the flow in a visual designer, publishes a version, and it runs — durably, race-safely, and fully audited.

This blueprint is the buildable spec for the workflow/approval engine **mandated** by [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md) (Domain 11 · Workflow / approval engine).

## Data model

Tenant-scoped. Every table has PK `id` (ulid, time-ordered), `tenant_id` (fk), and standard audit columns `created_at`, `created_by`, `updated_at`, `updated_by`. Definitions are **versioned and immutable once published**; in-flight instances pin the version they started on.

### WorkflowDefinition (versioned)

The authored process. A new `version` is a new row; `key` is stable across versions.

| Field | Type | Notes |
| --- | --- | --- |
| `key` | string | Stable machine key, unique per tenant (e.g. `expense_approval`) |
| `version` | int | Monotonic per `key`; published versions are immutable |
| `name` | string | Display name |
| `description` | text | |
| `entity_type` | string | What it governs, e.g. `expense`, `document` |
| `status` | enum | `draft` \| `published` \| `deprecated` \| `archived` |
| `initial_state_key` | string | Entry State on start |
| `graph_json` | jsonb | Canonical designer layout (node/edge positions) |
| `config_json` | jsonb | Engine-level config (default SLA calendar, timezone, on-cancel policy) |
| `published_at` | timestamp? | Set when `status` → `published` |
| `published_by` | fk? | |

### State

A node in the graph. Terminal states end the instance.

| Field | Type | Notes |
| --- | --- | --- |
| `definition_id` | fk | → WorkflowDefinition (version-scoped) |
| `key` | string | Unique within the definition, e.g. `pending_manager` |
| `name` | string | Display label |
| `type` | enum | `start` \| `task` \| `approval` \| `decision` \| `wait` \| `end` |
| `sla_rule_id` | fk? | Optional SLA/deadline attached to time in this state |
| `is_terminal` | bool | End state (`approved`, `rejected`, `cancelled`) |
| `metadata_json` | jsonb | UI hints (color, form to render, help text) |

### Transition (with guards)

A directed edge from one State to another, taken on an event, subject to a guard.

| Field | Type | Notes |
| --- | --- | --- |
| `definition_id` | fk | → WorkflowDefinition |
| `from_state_key` | string | Source State |
| `to_state_key` | string | Target State |
| `event` | string | Trigger: `approve` \| `reject` \| `submit` \| `escalate` \| `timer` \| custom |
| `guard_json` | jsonb? | Condition tree over instance/context attributes; transition only fires if it evaluates true |
| `priority` | int | Lower first — used to resolve if/else and range routing (first matching guard wins) |
| `actions_json` | jsonb? | Ordered actions to run on this transition (notify / update / webhook) |
| `is_automatic` | bool | Fires without human action when guard becomes true (used by `decision`/`wait` states) |

`guard_json` example (limit/range routing): `{ "all": [ {"attr":"instance.context.amount","op":"gt","value":50000} ] }`.

### WorkflowInstance

One running (or finished) execution, bound to one business record and one **pinned** definition version.

| Field | Type | Notes |
| --- | --- | --- |
| `definition_id` | fk | The exact **version** this instance runs (pinned at start) |
| `definition_key` | string | Denormalized for querying across versions |
| `entity_type` | string | e.g. `expense` |
| `entity_id` | string | The business record under workflow |
| `current_state_key` | string | Where it is now |
| `status` | enum | `active` \| `completed` \| `cancelled` \| `errored` |
| `context_json` | jsonb | Working variables the engine + guards read (amount, branch_id, requester_id, …) |
| `state_version` | int | **Optimistic-lock counter**; bumped on every transition (race safety) |
| `started_by` | fk | Initiator |
| `started_at` | timestamp | |
| `completed_at` | timestamp? | |
| `correlation_id` | string | Ties all events/tasks/audit across the instance |

### Task / Approval

A unit of work assigned to a principal at an `approval`/`task` state. Multiple open tasks per state model parallel/quorum approvals.

| Field | Type | Notes |
| --- | --- | --- |
| `instance_id` | fk | → WorkflowInstance |
| `state_key` | string | The state that created this task |
| `policy_id` | fk? | → ApprovalPolicy governing this batch of tasks |
| `assignee_type` | enum | `user` \| `role` \| `group` \| `dynamic` (resolved from context) |
| `assignee_id` | string | Resolved principal (post-delegation) |
| `original_assignee_id` | string? | Set if reassigned via Delegation |
| `sequence` | int | Order for **sequential** approvals (0 = first); same value = **parallel** |
| `decision` | enum? | `approve` \| `reject` \| `abstain` — null while open |
| `decided_by` | fk? | Actual acting principal |
| `decided_at` | timestamp? | |
| `comment` | text? | Justification / note |
| `due_at` | timestamp? | Task-level SLA deadline |
| `status` | enum | `pending` \| `completed` \| `skipped` \| `expired` \| `reassigned` |

### ApprovalPolicy

Declares how the decisions on a state's tasks combine into a state outcome.

| Field | Type | Notes |
| --- | --- | --- |
| `definition_id` | fk | → WorkflowDefinition |
| `state_key` | string | The approval State this policy governs |
| `mode` | enum | `single` \| `multi` \| `sequential` \| `parallel` \| `quorum` |
| `quorum_required` | int? | **M** in M-of-N (required when `mode=quorum`) |
| `quorum_total` | int? | **N** in M-of-N (N approvers) |
| `on_reject` | enum | `reject_immediately` \| `continue` \| `require_quorum_reject` — how one reject affects the batch |
| `allow_self_approval` | bool | If false, initiator cannot be an approver |
| `assignee_rule_json` | jsonb | How assignees are resolved (role, group, dynamic expression over context) |

### EscalationRule

Fires when an SLA deadline passes without resolution.

| Field | Type | Notes |
| --- | --- | --- |
| `definition_id` | fk | → WorkflowDefinition |
| `state_key` | string | State whose deadline is watched |
| `trigger` | enum | `deadline_breached` \| `pct_of_deadline` (e.g. warn at 80%) |
| `offset_minutes` | int | Deadline duration (business calendar aware) |
| `action` | enum | `escalate` \| `reassign` \| `notify` \| `auto_approve` \| `auto_reject` |
| `escalate_to_json` | jsonb? | Next-tier assignee (role/user/dynamic) for `escalate`/`reassign` |
| `repeat_every_minutes` | int? | Re-fire cadence (tiered paging) |
| `max_repeats` | int? | Cap re-fires |

### Delegation

Out-of-office / stand-in routing: reassign one principal's approvals to another for a window.

| Field | Type | Notes |
| --- | --- | --- |
| `delegator_id` | fk | Principal going out of office |
| `delegate_id` | fk | Stand-in who receives the tasks |
| `scope_definition_key` | string? | Limit to one workflow (null = all) |
| `valid_from` | timestamp | |
| `valid_until` | timestamp | **Required** — auto-expiry |
| `reason` | text? | |
| `is_active` | bool | Manual revoke |

### WorkflowAuditEntry (append-only)

Immutable record of every transition and decision. No update/delete.

| Field | Type | Notes |
| --- | --- | --- |
| `instance_id` | fk | → WorkflowInstance |
| `correlation_id` | string | Shared with the instance |
| `sequence` | int | Monotonic per instance (ordering + gap detection) |
| `event` | string | `transition` \| `decision` \| `escalation` \| `delegation_applied` \| `action_dispatched` |
| `from_state_key` | string? | |
| `to_state_key` | string? | |
| `actor_type` | enum | `user` \| `system` \| `timer` \| `service_account` |
| `actor_id` | fk? | Null for `system`/`timer` |
| `on_behalf_of_id` | fk? | Set when a Delegation was applied |
| `payload_json` | jsonb | Guard result, policy tally (e.g. `2/3 approved`), action outcomes |
| `state_version_after` | int | Instance `state_version` after this entry |
| `occurred_at` | timestamp | UTC |

## Key screens & UX

Every screen ships designed **loading, empty, and error** states per [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md).

1. **Workflow list** — all definitions by `key`, showing latest version, status badge (draft/published/deprecated), entity type, and count of active instances. Actions: New workflow, Edit draft, Publish, New version, Deprecate. *Loading:* skeleton rows. *Empty:* "No workflows yet — create one to start routing approvals." *Error:* retry.
2. **Visual designer** — a canvas to **drag states** (nodes) and **draw transitions** (edges). Selecting a node opens a panel for state type, attached ApprovalPolicy (mode + quorum M/N), and SLA rule; selecting an edge opens the guard builder (attribute/operator/value groups, priority order) and on-transition actions (notify/update/webhook). Live validation: unreachable states, missing initial/terminal, dangling edges. **Publish** snapshots the graph into an immutable version. *Empty:* starter template ("Submit → Approve → Done"). *Error:* inline validation banner listing problems.
3. **My Approvals inbox** — the actioning surface: pending tasks assigned to me (or delegated to me), with entity summary, requested-by, amount/context, due-in countdown, and Approve/Reject/Comment. Bulk approve where policy allows. Filters: workflow, overdue, delegated-to-me. *Empty:* "You're all caught up." *Loading:* skeleton list. *Error:* retry without losing draft comment.
4. **Instance detail + timeline** — current state, context values, open tasks with assignees and quorum tally (`2 of 3 approved`), and a **chronological timeline** rendered from WorkflowAuditEntry (each transition, decision, escalation, delegation, dispatched action). Admin actions (guarded): cancel, force-transition (audited with reason), reassign task. *Loading:* skeleton timeline. *Error:* retry.
5. **Approval-policy config** — per approval state: mode (single / multi / sequential / parallel / quorum), quorum **M-of-N** inputs, on-reject behavior, self-approval toggle, and assignee resolution rule. Preview shows "how many approvals complete this state."
6. **SLA / escalation config** — per state: deadline (business-calendar aware), warn threshold (% of deadline), and an ordered ladder of EscalationRules (escalate → reassign → notify → auto-decide) with repeat cadence for tiered paging. Preview a timeline of when each rule would fire.

## Rules & logic

### States, transitions & guards
- [MUST] A workflow is a directed graph of **States** and **Transitions**; exactly one `initial_state_key` and at least one terminal (`is_terminal`) state; the designer rejects unreachable states and dangling edges before publish.
- [MUST] A transition fires only if its `guard_json` evaluates **true** against instance `context_json` + environment; when multiple transitions match on the same event, the lowest `priority` wins (deterministic).
- [SHOULD] `decision`/`wait` states use `is_automatic` transitions that fire the instant a guard becomes satisfiable (no human task), enabling pure conditional routing.

### Approvals — single / multi / sequential / parallel / quorum
- [MUST] The **ApprovalPolicy.mode** determines how task decisions combine into the state outcome:
  - `single` — one approval completes the state.
  - `multi` / `parallel` — all tasks at the same `sequence` are open at once; state completes when the policy's completion condition is met.
  - `sequential` — tasks open in ascending `sequence`; the next opens only after the prior approves.
  - `quorum` — **M-of-N**: state approves once `quorum_required` (M) of `quorum_total` (N) tasks approve; it rejects once N−M+1 reject (approval becomes unreachable), per `on_reject`.
- [MUST] `on_reject` governs a single rejection's effect (`reject_immediately` short-circuits the batch; `continue` waits for the tally; `require_quorum_reject` needs a matching quorum to reject).
- [MUST] Enforce `allow_self_approval=false` server-side: the initiator (`started_by`) cannot be assigned or act as an approver.

### Conditional & limit/range routing
- [MUST] Support **if/else routing** via guarded transitions on a `decision` state (first matching guard by `priority`, else the default/fallback edge).
- [MUST] Support **limit/range routing** — numeric/threshold guards (e.g. `amount > 50000 → senior approval`). Routing conditions and approver eligibility tie into **ABAC** — evaluate against the same subject/resource/environment attributes as [`../standards-kb/04-security.md`](../standards-kb/04-security.md); the engine never grants access it wouldn't grant at the authorization layer.

### Role/assignee routing & delegation
- [MUST] Tasks resolve assignees by `user` / `role` / `group` / `dynamic` (expression over context, e.g. "the requester's manager"); role/group resolution expands to concrete principals at task creation.
- [MUST] **Delegation** (out-of-office): when an active Delegation matches the resolved assignee and workflow scope within its window, the task is routed to `delegate_id`, `original_assignee_id` is preserved, and a `delegation_applied` audit entry is written. Delegations are **time-boxed** (`valid_until` required) and revocable; expiry is enforced at resolution time, not by a batch job alone.

### Escalation & SLA timers
- [MUST] A State/Task deadline (`due_at`, business-calendar aware) drives **durable timers**; on `deadline_breached` (or `pct_of_deadline`) the ordered EscalationRules fire their `action` — `escalate` (next tier), `reassign`, `notify`, or `auto_approve`/`auto_reject`.
- [MUST] Timers are **durable** — they survive restarts and fire exactly once per (instance, rule, occurrence); tiered paging uses `repeat_every_minutes`/`max_repeats`. See [`../standards-kb/05-reliability-resilience.md`](../standards-kb/05-reliability-resilience.md).
- [SHOULD] Support **time-critical SLAs** (e.g. a 30-minute escalation that pages successive on-call tiers until acknowledged).

### Actions on transition
- [MUST] A transition may run an ordered list of `actions_json`: **notify** (via the notification service), **update record** (write back to the governed entity, e.g. set status), or **call webhook**. Actions are executed **after** the state/version commit and dispatched **asynchronously** with retry + DLQ — see the events layer ([`../events-kb/README.md`](../events-kb/README.md)); a failing action never corrupts instance state.
- [SHOULD] Record-update actions run through the same validation/authz as a normal write to that entity.

### Durability & race safety
- [MUST] Every transition is a single atomic operation guarded by **optimistic locking** on `WorkflowInstance.state_version`: read version → apply → write with `WHERE state_version = :read` and bump; a lost race retries against fresh state. This makes **double sign-off impossible** — two approvers acting on the last-needed task cannot both complete the transition.
- [MUST] Task decisions are **idempotent**: re-submitting a decided task is a no-op, not a second vote.

### Audit
- [MUST] Every transition, decision, escalation, delegation, and dispatched action writes an **append-only** WorkflowAuditEntry (immutable; no update/delete) carrying `actor`, `on_behalf_of`, guard result, policy tally, `correlation_id`, and `state_version_after`. This is the timeline. See [`../standards-kb/20-logging-audit-and-traceability.md`](../standards-kb/20-logging-audit-and-traceability.md).
- [MUST] Admin overrides (force-transition, cancel, manual reassign) are permitted only with permission + a recorded reason, and are audited like any other transition.

### Versioning
- [MUST] Publishing snapshots the definition into an **immutable version**; edits create a new `version`, never mutate a published one.
- [MUST] A WorkflowInstance **pins** its `definition_id` (version) at start and runs that version to completion even after newer versions publish — **in-flight instances keep their version**. New instances use the latest `published` version of the `key`.
- [SHOULD] Deprecating a version blocks new instances on it while letting existing ones finish; provide an optional, explicit, audited migration path for long-running instances.

## Labelled examples

**Example A — Expense approval (limit/range + quorum).** Definition `expense_approval` v3. `start → submitted`. A `decision` state routes on a guard: `amount ≤ 50,000 → pending_manager` (ApprovalPolicy `single`), else `→ pending_finance_board` (ApprovalPolicy `quorum`, M=2 of N=3, `on_reject=continue`, `allow_self_approval=false`). `pending_manager` has an SLA of 1 business day; EscalationRule: at 100% deadline `reassign` to the manager's delegate, else `escalate` to the department head. On the approving transition, `actions_json` = [notify requester, update expense.status=`approved`, webhook to the finance system]. In-flight v3 instances keep running even after v4 lowers the threshold to 25,000.

**Example B — Document sign-off (sequential).** Definition `doc_signoff` v1. `start → legal_review → compliance_review → approved`. Two `approval` states with `sequential` policy: compliance's task only opens after legal approves. A reject at either state transitions to a terminal `rejected` state and notifies the author. Each signer's Delegation (out-of-office) reroutes their task to a stand-in without changing the audit trail's `original_assignee_id`.

## Applicable standards

- **Mandating source** — Domain 11 · Workflow / approval engine — [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md)
- Approver eligibility & routing via **ABAC**, server-side enforcement, self-approval prevention — [`../standards-kb/04-security.md`](../standards-kb/04-security.md)
- Durable timers, exactly-once escalation, atomic transitions, retries/DLQ — [`../standards-kb/05-reliability-resilience.md`](../standards-kb/05-reliability-resilience.md)
- Append-only audit trail, correlation ids, timeline traceability — [`../standards-kb/20-logging-audit-and-traceability.md`](../standards-kb/20-logging-audit-and-traceability.md)
- Async transition actions (notify / webhook) via the events layer — [`../events-kb/README.md`](../events-kb/README.md)
- Tenant isolation of definitions/instances — [`../standards-kb/01-architecture-multitenancy.md`](../standards-kb/01-architecture-multitenancy.md)
- Screen states, designer/inbox UX — [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md)

## Acceptance checklist

- [ ] Workflows are authored as **data** (states/transitions/guards) — no code per workflow; a new flow needs no deploy.
- [ ] Visual designer creates/edits states + transitions, validates the graph, and publishes an immutable version.
- [ ] Guarded transitions support if/else and **limit/range routing**, with deterministic priority resolution.
- [ ] Approval modes work: **single, multi, sequential, parallel, and quorum (M-of-N)**, with configurable on-reject behavior.
- [ ] Assignees resolve by user/role/group/dynamic; **delegation** (time-boxed, revocable) reroutes tasks and is audited.
- [ ] SLA deadlines drive **durable timers**; EscalationRules escalate/reassign/notify/auto-decide, with tiered repeats.
- [ ] Transition actions (notify / update record / webhook) run async with retry + DLQ and never corrupt instance state.
- [ ] Transitions are atomic and **race-safe** via optimistic locking on `state_version`; **no double sign-off**; decisions idempotent.
- [ ] Every transition/decision/escalation/delegation is in an **append-only** audit trail; instance detail renders the timeline.
- [ ] Definitions are **versioned**; in-flight instances keep their pinned version; new instances use the latest published.
- [ ] Self-approval blocked server-side; admin force-transition/cancel require permission + recorded reason and are audited.
- [ ] All screens ship loading/empty/error states; everything tenant-scoped.
