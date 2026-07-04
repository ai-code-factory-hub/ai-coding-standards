# 06 · MCP Testing & Governance

Purpose: keep the MCP surface **trustworthy over time** — tested (correctness + safety), versioned, catalogued, approved before it goes live, and cleanly deprecated. Extends [../standards-kb/08-testing-qa.md](../standards-kb/08-testing-qa.md), [../standards-kb/02-versioning-deploy-migration.md](../standards-kb/02-versioning-deploy-migration.md), and the versioning/eval rules in [../standards-kb/15-ai-llm-governance.md](../standards-kb/15-ai-llm-governance.md).

## Testing tools

- **[MUST]** **Unit-test each tool**: input-schema validation (accepts valid, rejects invalid/extra fields), authz enforcement (allowed vs. denied identities), **tenant isolation** (an identity in tenant A can never read/act on tenant B), output-schema conformance, and error/fail-closed paths.
- **[MUST]** Maintain a **safety eval set** for the MCP surface — indirect prompt-injection payloads, cross-tenant probes, destructive-action coaxing, malformed/oversized args, secret-exfiltration attempts — and assert the server refuses or safely handles each. Ties to [04-mcp-safety-and-prompt-injection.md](04-mcp-safety-and-prompt-injection.md) and [../standards-kb/15-ai-llm-governance.md](../standards-kb/15-ai-llm-governance.md).
- **[MUST]** Run both suites **in CI on every change** to a tool, schema, or the wrapped API; block release on regression (accuracy, safety, format, cost).
- **[SHOULD]** Add **eval checks on tool selection/format** — that descriptions are unambiguous enough that an agent picks the right tool and produces schema-valid calls.

## Versioning tools & schemas

- **[MUST]** **Version tools and their schemas like code** — pinned, reviewed, roll-back-able (see [../standards-kb/15-ai-llm-governance.md](../standards-kb/15-ai-llm-governance.md)).
- **[MUST]** Treat schema changes with API-versioning discipline (per [../standards-kb/02-versioning-deploy-migration.md](../standards-kb/02-versioning-deploy-migration.md)): additive/optional changes are minor; renames, removed fields, tightened types, or changed semantics are **breaking** and get a new tool version.
- **[SHOULD]** Surface the tool version in capability negotiation and in logs (see [05](05-mcp-access-logs-and-rate-limiting.md)) so clients and audits can track exactly what ran.

## Registry / catalog of approved servers & tools

- **[MUST]** Keep a **registry/catalog of approved MCP servers and tools** — the source of truth for what exists, who owns it, its scopes, its risk class (read / write / destructive), version, and status (proposed / approved / deprecated / retired).
- **[MUST NOT]** allow a tool or server to be reachable by agents unless it is in the registry and marked approved. Unregistered = not exposed.
- **[SHOULD]** Record each tool's data classes touched, required consent/scopes, rate/cost limits, and links to its tests and eval results — so review and audit are self-service.

## Approval workflow before a tool goes live

- **[MUST]** Gate new/changed tools behind an **approval workflow**: design review of name/description/schema → security review (authz, tenant scope, secrets, injection surface per [03](03-mcp-security-and-authz.md)/[04](04-mcp-safety-and-prompt-injection.md)) → passing unit + safety evals → owner + security sign-off → registry entry marked approved → release.
- **[MUST]** Require explicit sign-off for **destructive/high-impact** tools, including their confirmation flow and tighter limits.
- **[SHOULD]** Fit this into the platform's release gate (see [../standards-kb/99-master-release-gate.md](../standards-kb/99-master-release-gate.md)); no tool reaches agents without passing it.

## Deprecation

- **[MUST]** Deprecate tools/versions on a **published timeline**: mark deprecated in the registry, stop advertising to new clients, keep serving existing clients through a defined window, then retire.
- **[SHOULD]** Provide a migration path (successor tool/version) and monitor usage of the deprecated tool via logs (see [05](05-mcp-access-logs-and-rate-limiting.md)) to confirm it's safe to retire.
- **[MUST]** On retirement, remove the tool from advertised capabilities and revoke its scopes so it can't be invoked.

## Acceptance checklist

- Each tool has unit tests (schema, authz, tenant isolation, output, fail-closed) plus a safety/injection eval set, run in CI on every change.
- Tools and schemas are versioned like code; breaking schema changes get a new version, surfaced in negotiation and logs.
- A registry/catalog is the source of truth (owner, scopes, risk class, version, status); nothing unregistered is exposed to agents.
- New/changed tools pass a design → security → eval → sign-off approval workflow before going live; destructive tools need explicit sign-off.
- Tools are deprecated on a published timeline with a migration path, then retired with scopes revoked and advertising removed.
