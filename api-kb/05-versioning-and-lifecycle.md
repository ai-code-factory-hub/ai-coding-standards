# 05 · Versioning & Lifecycle

**Purpose:** let the API evolve without breaking the consumers who depend on it. This extends the
[10 · API & Integration](../standards-kb/10-api-integration.md) rule "versioned (support N / N-1
with a published deprecation window)" into a concrete versioning scheme, breaking-change ruleset,
deprecation process with sunset headers, and consumer-migration communication. It complements
[02 · Versioning, Deploy & Migration](../standards-kb/02-versioning-deploy-migration.md).

## Version in the contract

- **[MUST]** Carry the **major version in the URL path** — `/v1/lab-orders`. It is explicit,
  cacheable, unambiguous in logs, and trivial to route at the gateway. This is the default for
  public/partner APIs.
- **[MUST]** Only the **major** version appears in the contract; minor/patch changes are
  **additive and non-breaking** and ship without a new version (see rules below).
- **[MAY]** Use a **header-based** version (`Accept: application/vnd.api.v1+json`) or a date-based
  version (`API-Version: 2026-07-01`) instead — pick **one** scheme and apply it everywhere. Do not
  mix path and header versioning across the same API.
- **[SHOULD]** *(GraphQL)* Prefer **schema evolution** (add fields, `@deprecated` directive on old
  ones) over versioned endpoints — the same non-breaking discipline applies to types and fields.

## N / N-1 support

- **[MUST]** Support at least the **current (N) and previous (N-1)** major versions concurrently,
  each with its own published [OpenAPI spec](02-openapi-spec-first.md) and changelog, so consumers
  have a real window to migrate.
- **[SHOULD]** Publish the **support policy** (how many majors, for how long) in the portal so
  consumers can plan; enterprise/partner contracts may extend N-1 support by agreement.

## Breaking vs non-breaking changes

- **[MUST]** These are **non-breaking** (ship without a new version): adding a new endpoint, adding
  an **optional** request field, adding a field to a response, adding a new enum value **only if
  clients were told to tolerate unknown values**, adding an optional query param, relaxing a
  validation constraint.
- **[MUST]** These are **breaking** (require a new major version): removing/renaming a field or
  endpoint, changing a field's type or format, making an optional field required, changing default
  behavior, tightening validation, changing error `code`s or status codes, changing pagination or
  auth semantics.
- **[MUST]** The **breaking-change gate runs in CI** (spec-diff, e.g. `oasdiff`) and **fails the
  build** if a breaking change lands without a version bump — the contract cannot break silently.

## Deprecation policy, sunset headers & timelines

- **[MUST]** Deprecation is a **staged, time-boxed** process, never a silent removal:
  1. **Announce** — changelog + portal + direct comms; mark the operation `deprecated: true` in the
     spec.
  2. **Signal in responses** — emit the **`Deprecation`** header (RFC 8594) and a **`Sunset`**
     header with the removal date, plus a `Link` to migration docs, on every affected response.
  3. **Grace window** — keep the deprecated version live for the **published minimum** (e.g. ≥ 6
     months for public APIs, or per contract).
  4. **Sunset** — remove only after the window; removed endpoints return `410 Gone` (not `404`) with
     a pointer to the successor.

> **Example (labelled — deprecation signaling on a live response):**
> ```http
> HTTP/1.1 200 OK
> Deprecation: true
> Sunset: Sat, 03 Jan 2026 23:59:59 GMT
> Link: <https://developers.example.com/migrate/v2>; rel="deprecation"; type="text/html"
> ```

- **[SHOULD]** Track **which consumers still call deprecated endpoints** (from the access log,
  [06](06-api-access-logs-and-analytics.md)) so migration outreach is targeted at the accounts that
  actually need it, and the sunset date is data-informed.

## Changelog

- **[MUST]** Maintain a **published, versioned changelog** (generated from the spec diff,
  [02](02-openapi-spec-first.md)) marking each entry breaking / non-breaking / deprecated /
  removed, with dates. It is the consumer's single reference for "what changed and when do I have
  to act."

## Consumer migration communications

- **[MUST]** Reach deprecated-endpoint consumers through **more than the changelog** — email/portal
  notices to the technical contact on the account, with the sunset date, the migration guide, and
  the specific endpoints they use.
- **[SHOULD]** Provide a **migration guide** per major bump (old→new mapping, code samples, updated
  SDKs from [07](07-developer-portal-and-sdks.md)) and, where feasible, a **compatibility shim** at
  the gateway that adapts N-1 calls during the window.
- **[SHOULD]** Send **escalating reminders** as the sunset date approaches (e.g. T-90, T-30, T-7
  days) and confirm zero traffic on the old version before removal.

## Acceptance checklist

- [ ] Major version is explicit in the contract (path `/v1/` by default); one scheme used consistently.
- [ ] At least N and N-1 majors are supported concurrently, each with its own spec + changelog.
- [ ] The non-breaking vs breaking rules are documented; only breaking changes bump the major version.
- [ ] A CI spec-diff gate fails the build on an unversioned breaking change.
- [ ] Deprecation is staged: announce → `Deprecation`/`Sunset` headers → published grace window → `410 Gone`.
- [ ] Deprecated-endpoint usage is tracked per consumer from the access log to target migration.
- [ ] A versioned changelog (generated from spec diff) marks breaking/deprecated/removed with dates.
- [ ] Affected consumers are contacted directly with sunset dates, a migration guide, and updated SDKs; reminders escalate before sunset.
