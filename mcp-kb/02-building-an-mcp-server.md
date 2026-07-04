# 02 · Building an MCP Server

Purpose: how to design and implement the server itself — tool design, typed schemas, safe resources, stateless handlers with explicit tenant context, wrapping the governed API instead of the raw database, and error handling. Security and authz specifics live in [03-mcp-security-and-authz.md](03-mcp-security-and-authz.md); this file is the construction guide.

## Designing tools

- **[MUST]** Give each tool a **clear, action-oriented name** and a **precise description** — the model chooses tools from these. `search_orders` beats `orders`; `refund_order` beats `process`. The description states what it does, what it does **not** do, and when to use it.
- **[MUST]** Define **typed input and output schemas** (JSON Schema). Constrain types, enums, formats, min/max, and required fields. A tight schema is your first validation layer and cuts prompt-injection surface.
- **[MUST]** Follow **least privilege per tool**: one tool = one narrow capability. No "do-everything" tools, no free-form query/command tools (`run_sql`, `exec`, `http_request`) exposed to a model.
- **[SHOULD]** Make tools **idempotent where possible** and require explicit identifiers (IDs) rather than fuzzy references the model might get wrong.
- **[SHOULD]** Split **read** and **write** into separate tools so authz and rate limits can differ; flag destructive tools for confirmation (see [04](04-mcp-safety-and-prompt-injection.md)).

### Example tool definition (illustrative, project-agnostic)

```json
{
  "name": "search_orders",
  "description": "Search the CURRENT tenant's orders by status and date range. Read-only. Does not return other tenants' data. Does not modify anything.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "status": { "type": "string", "enum": ["open", "shipped", "cancelled"] },
      "from_date": { "type": "string", "format": "date" },
      "to_date": { "type": "string", "format": "date" },
      "limit": { "type": "integer", "minimum": 1, "maximum": 50, "default": 20 }
    },
    "required": ["status"],
    "additionalProperties": false
  },
  "outputSchema": {
    "type": "object",
    "properties": {
      "orders": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "id": { "type": "string" },
            "status": { "type": "string" },
            "total": { "type": "number" },
            "created_at": { "type": "string", "format": "date-time" }
          },
          "required": ["id", "status"]
        }
      }
    },
    "required": ["orders"]
  }
}
```

Note: **tenant is not a parameter.** It is derived server-side from the authenticated identity — never accepted from the model (see below).

## Wrap the governed API, not the raw DB

- **[MUST]** Implement each tool as a **thin adapter over your existing governed API** (or the same service layer the API uses). That layer already carries authn/authz, input validation, tenant scoping, business rules, and audit. Reusing it means the MCP surface inherits those controls.
- **[MUST NOT]** let a tool run raw SQL, raw shell, or arbitrary HTTP against internal services. If the model can compose the query, the schema cannot protect you.
- **[SHOULD]** If a needed capability doesn't exist as a governed API endpoint yet, **build the endpoint first**, then wrap it — don't reach around it. See [../api-kb/03-api-security.md](../api-kb/03-api-security.md) and [../standards-kb/10-api-integration.md](../standards-kb/10-api-integration.md).

## Stateless tools + explicit tenant context

- **[MUST]** Keep tool handlers **stateless** — no per-conversation state stored in the server between calls. All authority comes from the **authenticated request context** (identity + tenant + scopes), resolved fresh on every call.
- **[MUST]** Make **tenant context explicit and server-derived**: pull `tenant_id` and `user_id` from the validated token/session, inject them into every API call, and **ignore any tenant/user the model supplies** in tool arguments.
- **[MUST]** Enforce tenant scope **in every tool**, defensively, even though the underlying API also enforces it (defence in depth). Never return another tenant's data in a tool result or a resource. See [../standards-kb/01-architecture-multitenancy.md](../standards-kb/01-architecture-multitenancy.md).

```
handle(request):
    ctx = auth.verify(request.token)        # -> {user_id, tenant_id, scopes}
    require_scope(ctx, "orders:read")        # per-tool authz (see 03)
    args = schema.validate(request.args)     # typed input validation
    result = orders_api.search(              # wrap governed API
        tenant_id=ctx.tenant_id,             # server-derived, never from model
        status=args.status, ...
    )
    return schema.validate_output(result)    # validate what you return (see 04)
```

## Exposing resources safely

- **[MUST]** Scope every resource to the requesting identity + tenant; a resource URI must resolve **only** to data that identity may read. Never expose an enumerable/global resource namespace that crosses tenants.
- **[MUST]** Treat resource **contents as untrusted data**, not instructions — a document may carry injected text (see [04](04-mcp-safety-and-prompt-injection.md)).
- **[SHOULD]** Prefer returning data through **tools with typed output** over broad resource listings when the data is sensitive — tools give you per-call authz and logging.

## Error handling

- **[MUST]** Return **structured, minimal errors** — a stable error code and a safe message. **Never leak** stack traces, internal IDs, SQL, secrets, or other tenants' existence in an error.
- **[SHOULD]** Distinguish `unauthorized` / `not_found` / `invalid_input` / `rate_limited` so the client (and logs) can react, but do not reveal *why* something is unauthorized.
- **[MUST]** Fail **closed**: if authz, tenant resolution, or schema validation fails, deny — never fall back to a broader scope or an unscoped query.
- **[SHOULD]** Make errors **non-actionable for injection** — do not echo attacker-controlled input verbatim in a way that could steer the model.

## Acceptance checklist

- Tools have clear names, precise descriptions, and tight typed input/output schemas (`additionalProperties: false`).
- One narrow capability per tool; least privilege; read/write split; no free-form query/command tools.
- Every tool wraps the governed API/service layer — never raw DB, shell, or arbitrary HTTP.
- Handlers are stateless; tenant + user derived from the authenticated context and ignored if supplied by the model.
- Tenant scope enforced defensively in every tool and resource; no cross-tenant data ever returned.
- Errors are structured, minimal, leak nothing, and fail closed.
