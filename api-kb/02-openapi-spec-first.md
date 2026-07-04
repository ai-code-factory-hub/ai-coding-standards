# 02 · OpenAPI Spec-First

**Purpose:** operationalize the [10 · API & Integration](../standards-kb/10-api-integration.md)
rule that "the OpenAPI schema is the source of truth." This doc defines the **spec-first
workflow**: the contract is authored and reviewed *before* server code, and every downstream
artifact — server stubs, client SDKs, mock server, request validation, contract tests, the
developer portal — is **generated from that one spec**. Code cannot drift from docs because the
docs *are* the contract.

## The contract is the source of truth

- **[MUST]** A single **OpenAPI 3.1** document (per API/version) is the authoritative contract.
  It is versioned in the repo, reviewed like code, and **published** to the portal
  ([07](07-developer-portal-and-sdks.md)).
- **[MUST]** The spec **[MUST NOT]** be generated *from* server annotations as the primary source
  (code-first) for public/partner APIs — that lets implementation drive the contract. Author the
  spec first; generate the server *from* it.
- **[SHOULD]** Split large specs into `$ref`-linked component files (schemas, parameters,
  responses, examples) and bundle for publish, so review diffs stay readable.

## Spec-first workflow

The pipeline, in order — each stage gates the next:

1. **Design** — author the spec (paths, schemas, examples, error envelope from
   [01](01-api-design-guide.md)). **[MUST]** lint it in CI (Spectral or equivalent) against the
   house style ruleset.
2. **Review** — humans review the *contract* in a PR: naming, pagination, breaking-change check
   against the previous version (see [05 · Versioning](05-versioning-and-lifecycle.md)). No
   implementation exists yet, so the argument is only about the interface.
3. **Generate** — from the approved spec, **generate**: (a) **server stubs / request-response
   validation middleware**, (b) **client SDKs** in each supported language
   ([07](07-developer-portal-and-sdks.md)), (c) typed models. Hand-written code fills the stubs;
   the interface is not hand-edited.
4. **Mock server** — stand up a **mock/prototype server from the spec** (e.g. Prism) so client
   teams and the sandbox can integrate **before** the real backend is built.
5. **Contract tests** — **[MUST]** run automated contract tests in CI asserting the running
   implementation matches the spec (schemathesis/Dredd or provider-contract tests); the build
   fails on divergence. Request **and** response validation run against the spec at the edge in
   non-prod, and optionally in prod.

- **[MUST]** All of steps 1–5 run in the **CI/CD pipeline** ([12 · DevOps & CI/CD](../standards-kb/12-devops-cicd.md)) —
  lint, breaking-change gate, generate, contract-test — on every change to the spec.

## Examples & sandbox

- **[MUST]** Every operation carries at least one **request and response `example`** in the spec.
  These examples power the interactive docs, the mock server responses, and the sandbox — write
  them once, reuse everywhere.
- **[SHOULD]** Examples use **realistic but synthetic** data (never real PII/secrets) and cover
  the **error envelope** too, so consumers see the failure shape, not only the happy path.
- **[SHOULD]** The sandbox ([07](07-developer-portal-and-sdks.md)) is backed by the mock server or
  a seeded test tenant so developers can call the API before signing anything.

## Changelog

- **[MUST]** Every published spec change produces a **human-readable changelog entry** (added /
  changed / deprecated / removed) tied to the version and surfaced in the portal — this is the
  same changelog the lifecycle process in [05](05-versioning-and-lifecycle.md) drives.
- **[SHOULD]** Generate the changelog **from the spec diff** (e.g. `oasdiff`) so it cannot omit a
  change; annotate each entry as **breaking / non-breaking**.

## Short OpenAPI snippet

> **Example (labelled — spec-first operation with error envelope + cursor page):**
> ```yaml
> paths:
>   /lab-orders:
>     get:
>       operationId: listLabOrders
>       summary: List lab orders
>       parameters:
>         - { name: limit,  in: query, schema: { type: integer, maximum: 100, default: 50 } }
>         - { name: cursor, in: query, schema: { type: string } }   # opaque keyset cursor
>         - { name: status, in: query, schema: { type: string, enum: [pending, resulted, cancelled] } }
>       responses:
>         '200':
>           description: A page of lab orders
>           content:
>             application/json:
>               schema: { $ref: '#/components/schemas/LabOrderPage' }
>               examples:
>                 default: { $ref: '#/components/examples/LabOrderPageExample' }
>         '429': { $ref: '#/components/responses/RateLimited' }
>         default:
>           description: Error
>           content:
>             application/json:
>               schema: { $ref: '#/components/schemas/Error' }   # the one standard envelope
> components:
>   schemas:
>     Error:
>       type: object
>       required: [error]
>       properties:
>         error:
>           type: object
>           required: [code, message, correlation_id]
>           properties:
>             code:           { type: string }
>             message:        { type: string }
>             correlation_id: { type: string }
>             errors:
>               type: array
>               items:
>                 type: object
>                 properties:
>                   field:   { type: string }
>                   code:    { type: string }
>                   message: { type: string }
> ```

## Acceptance checklist

- [ ] One OpenAPI 3.1 document per API/version is the authoritative, version-controlled contract.
- [ ] Spec is authored and PR-reviewed **before** implementation (spec-first, not code-first).
- [ ] CI lints the spec against a house-style ruleset on every change.
- [ ] Server stubs/validation, client SDKs, and mock server are all **generated from the spec**.
- [ ] A mock server lets clients integrate before the backend exists.
- [ ] Contract tests in CI fail the build when implementation diverges from the spec.
- [ ] Every operation has request + response examples (including the error shape) using synthetic data.
- [ ] A breaking/non-breaking changelog is generated from the spec diff and published to the portal.
