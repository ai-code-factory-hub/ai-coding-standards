# 01 · MCP Concepts & Architecture

Purpose: give an enterprise engineer the mental model of MCP — the client/server split, the three server primitives, transports, capability negotiation — and a clear rule for **when to build an MCP server vs. just ship an API**.

## Client / server model

- An **MCP client** lives inside an AI application or agent runtime (a chat app, an IDE assistant, an orchestration service). It connects to one or more **MCP servers**.
- An **MCP server** is a process **you** own that exposes a bounded set of capabilities from your application to the client.
- The **model never talks to your systems directly.** It asks its client to invoke a tool or read a resource; the client sends a request to your server; your server does the work and returns a result. **[MUST]** Treat the server as the trust boundary — the client and the model behind it are outside it.

```
[ LLM ] -> [ MCP client (AI app) ] --transport--> [ MCP server (yours) ] -> [ governed API ] -> [ data ]
```

## Server primitives

- **Tools** — callable functions the model may invoke. Each has a name, a description, a typed **input schema**, and returns a typed result. Tools are the main action surface (read *and* write). Example: `get_invoice(invoice_id)`, `refund_order(order_id, amount)`.
- **Resources** — readable data/context identified by URI (e.g. `app://tenant/{id}/policy.md`). The client loads them into context. Resources are **data, not actions**; treat their contents as untrusted (see [04-mcp-safety-and-prompt-injection.md](04-mcp-safety-and-prompt-injection.md)).
- **Prompts** — reusable prompt templates the server offers (e.g. a "summarize this ticket" template). Optional; useful for standardizing how agents use your tools.

**[SHOULD]** Prefer **Tools** for anything with side effects or that needs authz enforcement per call. Use **Resources** only for read context that is already safe to expose to the requesting identity.

## Transports

| Transport | Shape | Auth | Use when |
|-----------|-------|------|----------|
| **stdio** | Local subprocess; client launches server, communicates over stdin/stdout | Inherits local process/user | Local dev tools, desktop copilots, single-user CLI agents |
| **Streamable HTTP / SSE** | Remote server over HTTP; server-sent events for streaming | **Real auth required** — OAuth 2.1 / bearer tokens, per-user identity | Multi-tenant SaaS, shared/remote agents, anything on a network |

- **[MUST]** Any server reachable over a network uses the HTTP transport with real authentication and TLS — never expose an unauthenticated remote MCP server. See [03-mcp-security-and-authz.md](03-mcp-security-and-authz.md).
- **[MUST]** For `stdio`, remember the server still runs with whatever local privileges/credentials it inherits — scope those to least privilege; do not bake in tenant-wide or admin credentials.

## Capability negotiation

- On connect, client and server **negotiate capabilities**: protocol version, and which primitives (tools/resources/prompts) and features (e.g. subscriptions, sampling) each supports. The client only uses what the server advertises.
- **[MUST]** Advertise only the tools/resources the **connecting identity** is allowed to use — capability lists themselves leak your surface area. Filter the advertised set per authenticated principal and tenant.
- **[SHOULD]** Pin and check protocol/tool **versions** during negotiation so a client can detect breaking changes (see [06-mcp-testing-and-governance.md](06-mcp-testing-and-governance.md)).

## When to build an MCP server vs. just an API

Building an MCP server is **additive** — it does not replace your API; it wraps it for agent consumption.

- **[SHOULD] Build an MCP server when:** approved AI agents need to *act on* or *read from* your app conversationally; you want a catalogued, per-tool-authorized, logged surface for agents; multiple agent clients would otherwise each hand-roll API glue.
- **[SHOULD] Just ship / keep the API when:** the consumer is deterministic software (not an LLM), you need fine-grained REST/GraphQL semantics, or the capability is too destructive/broad to hand to a model at all.
- **[MUST NOT]** expose a capability via MCP that you would not expose via a governed API. MCP is a *presentation* of the governed API for agents, not a bypass. See [../standards-kb/10-api-integration.md](../standards-kb/10-api-integration.md) and [../api-kb/README.md](../api-kb/README.md).

> **Example (SaaS):** a support copilot needs to look up a customer's open tickets and post a reply. Build two MCP tools (`list_tickets`, `post_ticket_reply`) that wrap existing governed API endpoints, each enforcing the agent-user's authz and tenant. Do **not** expose a generic `run_sql` tool.

## Acceptance checklist

- The server is the trust boundary; the model/client is outside it and never touches data directly.
- Tools used for actions/side-effects; Resources only for already-safe read context; Prompts optional and standardized.
- Remote/network servers use HTTP transport with real auth + TLS; stdio servers run least-privilege.
- Capability negotiation filters advertised tools/resources per authenticated identity + tenant.
- A documented build-vs-API decision; nothing exposed via MCP that wouldn't be exposed via the governed API.
