# 01 · API Design Guide

**Purpose:** make every endpoint feel like it was designed by one person on one day. This doc
sets the concrete design rules — resource modeling, naming, HTTP semantics, one error envelope,
pagination, filtering, dates, money — that turn the [10 · API & Integration](../standards-kb/10-api-integration.md)
requirement for a "consistent paradigm" into buildable conventions. **REST is the primary
style**; GraphQL notes appear where the rule differs.

## Resource modeling & naming

- **[MUST]** Model **nouns, not verbs**. Endpoints address *resources*; HTTP verbs express the
  action. `POST /invoices` — not `/createInvoice`.
- **[MUST]** **Plural, lowercase, kebab-case** collection names; resource identity via path
  segment. `GET /lab-orders/{orderId}`. Nest only to express ownership one level deep
  (`/patients/{patientId}/orders`); beyond that, use a top-level resource with a filter.
- **[MUST]** Use **opaque, non-sequential ids** in URLs (UUID/ULID) — never expose an
  auto-increment primary key (enumeration risk; see IDOR/BOLA in [03 · API Security](03-api-security.md)).
- **[SHOULD]** Keep the resource shape **stable and flat**; expose relationships as ids plus an
  optional `?expand=` allow-list rather than deeply nested payloads that vary per call.
- **[SHOULD]** Prefer **sub-resources for actions that don't fit CRUD** rather than verbs in the
  path: `POST /orders/{id}/cancellation` over `POST /orders/{id}/cancel` when the action has its
  own state/result worth addressing; a plain verb sub-path is acceptable when it does not.
- **[MAY]** *(GraphQL)* Model the same domain as a typed graph; the naming discipline (nouns,
  stable fields, opaque ids) still applies to types and fields.

## HTTP methods & status codes

- **[MUST]** Use methods per their contract: `GET` (safe, no side effects), `POST` (create /
  non-idempotent action), `PUT` (full idempotent replace), `PATCH` (partial update), `DELETE`
  (idempotent removal). `GET` **[MUST NOT]** mutate state.
- **[MUST]** Return the **narrowest correct status code**:

  | Class | Code | Use |
  |---|---|---|
  | Success | `200` / `201` / `202` / `204` | OK / created (+`Location`) / accepted-async / no-content |
  | Client | `400` | malformed/validation-failed request |
  | | `401` / `403` | unauthenticated / authenticated-but-forbidden |
  | | `404` | resource not found (or hidden for authz) |
  | | `409` | conflict (optimistic-concurrency / duplicate) |
  | | `422` | semantically invalid though well-formed *(if you distinguish from 400)* |
  | | `429` | rate-limited (see [04](04-api-gateway-and-rate-limiting.md)) |
  | Server | `500` / `503` | unexpected fault / dependency-unavailable-or-throttled |

- **[MUST]** `POST` that creates returns `201` with a `Location` header and the created body;
  long-running work returns `202` with a status/polling resource.
- **[SHOULD]** Support **`Idempotency-Key`** on retryable mutations and **optimistic concurrency**
  (`ETag` + `If-Match` → `409`/`412`) on edits, per Domain 10.

## One standard error envelope

- **[MUST]** **Every** non-2xx response uses **one** envelope: a stable machine `code`, a human
  `message`, optional field-level `errors`, and a `correlation_id` that matches the access log
  ([06](06-api-access-logs-and-analytics.md)). No stack traces, SQL, or internal hostnames leak.

> **Example (labelled — standard error envelope):**
> ```json
> {
>   "error": {
>     "code": "validation_failed",
>     "message": "One or more fields are invalid.",
>     "correlation_id": "01J8Z9K3P7QF2M4C",
>     "errors": [
>       { "field": "email", "code": "format", "message": "Not a valid email address." },
>       { "field": "amount", "code": "min", "message": "Must be greater than 0." }
>     ]
>   }
> }
> ```

- **[MUST]** `code` values are **stable, documented, and versioned** with the API — clients
  branch on `code`, never on `message` text (which may be localized/reworded).
- **[SHOULD]** Return a **problem-details-compatible** shape (RFC 9457 `application/problem+json`)
  if you want an ecosystem standard rather than a bespoke envelope — pick one and never mix.

## Pagination — cursor/keyset

- **[MUST]** **Every list endpoint paginates.** Default to **cursor/keyset** pagination (opaque
  `cursor` + `limit`), not offset/`page` — offset pagination skips or duplicates rows under
  concurrent writes and degrades on deep pages.
- **[MUST]** Return a stable envelope: the page's `data[]` plus a `next_cursor` (null at end).
  The cursor is **opaque** (base64 of the sort key + tiebreaker id) — clients never construct it.
- **[MUST]** Enforce a **max `limit`** (e.g. 100) and clamp requests above it.

> **Example (labelled — cursor page response):**
> ```json
> {
>   "data": [ { "id": "ord_01J...", "status": "resulted" } ],
>   "page": { "next_cursor": "eyJrIjoiMjAyNi0wNy0wNVQxMDoxMiIsImlkIjoib3JkXzAxSi4uLiJ9", "limit": 50 }
> }
> ```
> Next page: `GET /lab-orders?limit=50&cursor=eyJrIjoi...`

- **[MAY]** Offer offset pagination **only** for small, bounded admin lists where jump-to-page UX
  is required and data is near-static.

## Filtering, sorting & field selection — allow-lists

- **[MUST]** Filtering, sorting, and sparse field selection operate on an **explicit server-side
  allow-list** of fields — never reflect arbitrary column names from the query into a data store
  (injection / information-disclosure risk).
- **[MUST]** **Sort** via `?sort=` with a signed field list (`sort=-created_at,name`); reject
  unknown or non-allow-listed fields with `400`.
- **[SHOULD]** **Filter** via typed, documented params (`?status=active&created_after=...`) rather
  than a free-form query DSL; if you must offer operators, allow-list them (`eq,gte,lte,in`).
- **[SHOULD]** **Sparse fieldsets** via `?fields=id,status,total` to trim payloads, again against
  the allow-list; `?expand=` pulls related resources from a fixed set.

## Timestamps, money & consistency

- **[MUST]** **All timestamps are ISO-8601 in UTC** with an explicit offset/`Z`
  (`2026-07-05T10:12:03Z`) — never a local time, epoch-without-units, or ambiguous string. Field
  names end in `_at` for instants (`created_at`), `_date` for calendar dates.
- **[MUST]** **Money is a decimal string plus an ISO-4217 currency code**, never a float
  (`{ "amount": "1499.00", "currency": "INR" }`). Document whether the amount is major units or
  minor units and be consistent across the whole API.
- **[MUST]** **One casing convention** for all JSON keys (pick `snake_case` or `camelCase`) and
  one convention for enums (lowercase `snake_case` values) — applied everywhere, forever.
- **[SHOULD]** Booleans read as positive assertions (`is_active`, not `disabled`); nullable vs
  absent has one documented meaning; empty collections return `[]`, never `null`.
- **[SHOULD]** Represent enums as **strings**, not integers, so values are self-describing and
  extensible without a breaking change (see [05 · Versioning](05-versioning-and-lifecycle.md)).

## Consistency contract

- **[MUST]** These conventions live in the [OpenAPI spec](02-openapi-spec-first.md) and are
  enforced by a **spec linter in CI** (e.g. Spectral ruleset) so drift is caught at review, not
  in production. A reviewer should never have to re-argue naming or pagination style.

## Acceptance checklist

- [ ] Resources are nouns, plural/kebab-case, addressed by opaque non-sequential ids; nesting ≤ 1 level.
- [ ] HTTP methods respect safe/idempotent semantics; `GET` never mutates.
- [ ] Status codes are the narrowest correct value; `201`+`Location` on create; `202` for async.
- [ ] Exactly one error envelope with stable `code`, `message`, field `errors`, and `correlation_id`; no internal leakage.
- [ ] Every list uses cursor/keyset pagination with an opaque cursor and enforced max `limit`.
- [ ] Filter/sort/field-selection run against a server-side allow-list; unknown fields → `400`.
- [ ] All timestamps are ISO-8601 UTC; money is decimal-string + ISO-4217 code (no floats).
- [ ] One casing/enum convention applied everywhere and enforced by a spec linter in CI.
