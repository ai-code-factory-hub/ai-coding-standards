# 03 · MCP Security & Authorization

Purpose: the security core. Extends [../standards-kb/04-security.md](../standards-kb/04-security.md) and [../api-kb/03-api-security.md](../api-kb/03-api-security.md) with MCP-specific rules. If you read one file in this KB, read this one.

## The LLM is not an authorization boundary

- **[MUST]** **Each tool enforces its own authorization and tenant scope on every call.** Do not rely on the model, the client, the prompt, or the advertised capability list to keep an agent within bounds — any of them can be manipulated (prompt injection, a compromised client, a confused agent). Authz is decided **server-side, per call, against the authenticated identity.**
- **[MUST]** Authorize the **acting identity**, not "the agent" in the abstract: resolve *which user/service this request runs as*, then check that principal's permissions for this specific tool and these specific arguments (e.g. can this user refund *this* order).
- **[MUST]** Enforce **tenant isolation** independently in the tool even though the underlying API also does — defence in depth. **Never** return, reference, or confirm the existence of another tenant's data. See [../standards-kb/01-architecture-multitenancy.md](../standards-kb/01-architecture-multitenancy.md).
- **[MUST]** Fail **closed** on any authz/tenant ambiguity; never widen scope as a fallback.

## Remote-server authentication

For any HTTP/SSE (remote) MCP server:

- **[MUST]** Require authentication using **OAuth 2.1 / bearer tokens** — no unauthenticated remote server, ever. Serve over TLS only.
- **[MUST]** Carry a **per-user (or per-service) identity** in the token — not a single shared server credential. The server must know *who* the agent is acting as to authorize correctly and to log it.
- **[MUST]** Use **scoped tokens**: the token grants only the tool scopes that identity needs (`orders:read`, `tickets:write`), and the server checks the scope on each tool. Prefer short-lived tokens with refresh; validate `aud`/`iss`/`exp` and reject tokens minted for another audience.
- **[SHOULD]** Bind tokens to the client where possible (sender-constrained / DPoP-style) so a leaked token can't be replayed by another client.
- **[MUST NOT]** accept the identity or tenant as a **tool argument** or trust anything the model asserts about who it is — identity comes only from the validated token/session.

## Consent for data access

- **[MUST]** Obtain and record **user consent** before an agent accesses that user's data or acts on their behalf, per [../standards-kb/09-compliance-privacy.md](../standards-kb/09-compliance-privacy.md). The agent's authority derives from a real user's granted scopes, not from being "an approved bot".
- **[SHOULD]** Make consent **granular and revocable** — per scope, per data category — and re-check it at call time (revocation takes effect immediately, fail closed).

## Least privilege on exposed resources & tools

- **[MUST]** Expose the **minimum** set of tools and resources needed; advertise them **filtered per identity + tenant** (a client only sees what its principal may use — see [01](01-mcp-concepts-and-architecture.md)).
- **[MUST]** Give the server itself least-privilege credentials to downstream systems — never a tenant-wide or admin service account that a tool bug could turn into cross-tenant access.
- **[SHOULD]** Separate high-risk/destructive tools behind additional scopes and confirmation (see [04-mcp-safety-and-prompt-injection.md](04-mcp-safety-and-prompt-injection.md)).

## Secrets handling

- **[MUST]** Keep API keys, downstream credentials, signing keys, and provider secrets in a **secrets manager / KMS** — never in tool code, tool descriptions, resource contents, prompts, logs, or error messages. See [../standards-kb/04-security.md](../standards-kb/04-security.md).
- **[MUST NOT]** return secrets, tokens, or credentials in **any tool output or resource** — the model would place them in context, where they can leak via injection or logging.
- **[SHOULD]** Rotate MCP client credentials and downstream tokens on a schedule and on suspected compromise; support instant revocation of a client/agent.

## Acceptance checklist

- Every tool enforces authz + tenant scope server-side, per call, against the authenticated acting identity; fails closed.
- Remote servers require OAuth 2.1 / bearer over TLS, carry per-user identity, use scoped short-lived tokens with validated `aud`/`iss`/`exp`.
- Identity and tenant are never taken from tool arguments or model claims.
- User consent is recorded, granular, revocable, and re-checked at call time.
- Minimal tools/resources exposed and advertised per identity; server holds least-privilege downstream credentials.
- Secrets live in a KMS/secrets manager; never in code, descriptions, outputs, resources, prompts, logs, or errors.
