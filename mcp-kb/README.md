# MCP-KB · Exposing an Application to AI Agents (Securely)

Purpose: a project-agnostic knowledge base for exposing an enterprise application to approved AI agents through a **Model Context Protocol (MCP)** server — with per-tool authorization, tenant isolation, access logging, and rate/cost limits.

This KB **extends** [../standards-kb/15-ai-llm-governance.md](../standards-kb/15-ai-llm-governance.md) (AI/LLM Governance) and [../standards-kb/04-security.md](../standards-kb/04-security.md) (Security). It references those standards rather than re-stating them — read them first, then apply the MCP-specific guidance here.

## What is MCP

MCP is an open protocol for connecting AI applications/agents to external tools and data. An **MCP client** (an AI app or agent runtime) connects to one or more **MCP servers**. A server exposes three primitives:

- **Tools** — callable functions the model can invoke (e.g. `search_orders`, `create_ticket`).
- **Resources** — readable data/context the client can load (documents, records, config).
- **Prompts** — reusable prompt templates the server offers to the client.

Servers speak over a **transport**: `stdio` (local subprocess) or **streamable HTTP / SSE** (remote). Remote servers require real authentication (OAuth 2.1 / bearer tokens). See [01-mcp-concepts-and-architecture.md](01-mcp-concepts-and-architecture.md).

## Why an enterprise app exposes an MCP server

To let **approved** AI agents (internal copilots, customer-facing assistants, automation) use the application's capabilities **through a governed boundary** instead of scraping the UI or holding raw credentials. Done right, an MCP server gives you:

- One catalogued, versioned surface of what agents may do.
- Per-tool authorization and tenant scoping enforced server-side.
- Full access logging and rate/cost limits over every agent action.

## Core principle

**An LLM is not an authorization boundary.** The model may be tricked (prompt injection via a document, a resource, or a tool result). Therefore **every tool enforces its own authz + tenant scope**, and every input and tool output is treated as untrusted. This principle threads through the whole KB.

## How MCP relates to your API

MCP tools should **wrap the same governed API** the rest of the platform uses — not the raw database. The API already carries authn/authz, validation, tenant scoping, and audit; an MCP tool is a thin, well-described adapter over it. See [../api-kb/README.md](../api-kb/README.md) and [../api-kb/03-api-security.md](../api-kb/03-api-security.md). If a capability isn't safe as an API, it isn't safe as an MCP tool.

## Index

| # | File | Covers |
|---|------|--------|
| 01 | [01-mcp-concepts-and-architecture.md](01-mcp-concepts-and-architecture.md) | Client/server model, Tools/Resources/Prompts, transports, capability negotiation, build-vs-API decision |
| 02 | [02-building-an-mcp-server.md](02-building-an-mcp-server.md) | Designing tools & schemas, safe resources, stateless + explicit tenant context, wrapping the governed API |
| 03 | [03-mcp-security-and-authz.md](03-mcp-security-and-authz.md) | Per-tool authz, remote auth (OAuth 2.1/bearer), consent, least privilege, secrets |
| 04 | [04-mcp-safety-and-prompt-injection.md](04-mcp-safety-and-prompt-injection.md) | Untrusted inputs & resources, indirect injection, output validation, destructive-tool confirmation |
| 05 | [05-mcp-access-logs-and-rate-limiting.md](05-mcp-access-logs-and-rate-limiting.md) | Per-call audit logging, rate/frequency limits, token/cost caps, anomaly detection |
| 06 | [06-mcp-testing-and-governance.md](06-mcp-testing-and-governance.md) | Tool testing & evals, versioning, registry/catalog, deprecation, approval workflow |

## Related standards

- [../standards-kb/04-security.md](../standards-kb/04-security.md) — Security baseline
- [../standards-kb/09-compliance-privacy.md](../standards-kb/09-compliance-privacy.md) — Compliance & privacy
- [../standards-kb/10-api-integration.md](../standards-kb/10-api-integration.md) — API & integration
- [../standards-kb/15-ai-llm-governance.md](../standards-kb/15-ai-llm-governance.md) — AI/LLM governance
- [../standards-kb/16-finops-cost.md](../standards-kb/16-finops-cost.md) — FinOps & cost
- [../standards-kb/20-logging-audit-and-traceability.md](../standards-kb/20-logging-audit-and-traceability.md) — Logging, audit & traceability
