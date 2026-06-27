# MCP Integration

**Status:** Draft  
**Priority:** P1  
**Closes gaps:** Architecture gaps §10 (MCP)

---

## Purpose

Define the **MCP client** implementation, **server contract**, and how ORCH uses **tools**, **resources**, and **prompts**. Covers transport, discovery, auth context, error handling, and sandbox testing.

**Related docs:**

- [`04-multi-tenancy-and-config.md`](04-multi-tenancy-and-config.md) — per-tenant MCP registry
- [`05-orch-turn-engine.md`](05-orch-turn-engine.md) — tool invocation in turn loop
- [`03-auth-and-identity.md`](03-auth-and-identity.md) — auth context claims
- [`07-context-assembler.md`](07-context-assembler.md) — tool definitions in context
- [`01-api-and-streaming-events.md`](01-api-and-streaming-events.md) — `ToolInvocation`, `ToolResult`

---

## Scope

### In scope

- MCP client architecture
- Server registration and transport
- Tool discovery, invocation, caching
- Resources and prompts usage in ORCH
- Auth context forwarding
- Schema lifecycle, circuit breaker, sandbox

### Out of scope

- Building MCP servers (domain teams own servers)
- MCP protocol spec itself (reference: modelcontextprotocol.io)

---

## Client Architecture

```
ORCH
  └── MCP Client (per tenant pool or shared with tenant context)
        ├── Connection Manager (per serverId)
        ├── Discovery Cache
        ├── Invocation Executor
        └── Circuit Breaker Registry
```

MCP Client runs **embedded in ORCH workers** (v1). Sidecar deployment optional for enterprise.

---

## Server Registration

From tenant config (`04-multi-tenancy-and-config.md`):

```json
{
  "id": "mcp_billing",
  "name": "Billing",
  "transport": "streamable_http",
  "url": "https://mcp-billing.example.com/mcp",
  "auth": {
    "type": "forwarded_jwt",
    "audience": "mcp-billing"
  },
  "enabled": true,
  "timeoutMs": 30000,
  "retryPolicy": { "maxAttempts": 2, "backoffMs": 1000 }
}
```

### Transport types

| Transport | Config | Use case |
|-----------|--------|----------|
| `stdio` | `command`, `args`, `env` | Local/sidecar processes |
| `sse` | `url` | Remote SSE endpoint (legacy MCP) |
| `streamable_http` | `url` | Remote HTTP (preferred) |

### Connection lifecycle

```
On tenant config load / server register:
  connect(transport)
  initialize() → server capabilities
  list_tools() → cache
  list_resources() → cache (optional)
  list_prompts() → cache (optional)

Heartbeat: ping every 60s; reconnect on failure
```

---

## Tool Discovery

### Discovery flow

```
list_tools() per enabled server
  → merge into tenant tool registry
  → apply toolFilter (allowlist/blocklist)
  → namespace tool names: {serverId}.{toolName}  // e.g. billing.get_invoice
  → cache with TTL 5 min
```

### Cached tool schema

```json
{
  "serverId": "mcp_billing",
  "name": "get_invoice",
  "qualifiedName": "billing.get_invoice",
  "description": "Retrieve invoice by ID",
  "inputSchema": { "type": "object", "properties": { "invoiceId": { "type": "string" } } },
  "metadata": {
    "destructive": false,
    "requiresApproval": false,
    "sequential": false
  },
  "discoveredAt": "...",
  "schemaVersion": "1.0.0"
}
```

`metadata` from MCP `annotations` or platform defaults.

### Schema refresh

- Auto: TTL expiry
- Manual: `POST /v1/admin/mcp/servers/{id}/refresh`
- On invocation schema mismatch error: force refresh + single retry

### Version pinning

Enterprise tenants may pin `schemaVersion` per server; mismatch at discovery → admin alert, server marked degraded.

---

## Tool Invocation

### Request

```json
{
  "invocationId": "inv_01H...",
  "serverId": "mcp_billing",
  "toolName": "get_invoice",
  "arguments": { "invoiceId": "inv_123" },
  "authContext": {
    "tenantId": "ten_01H...",
    "userId": "usr_01H...",
    "scopes": ["tools:billing:read"],
    "externalUserId": "host_user_42",
    "channel": "embed"
  },
  "timeoutMs": 30000,
  "traceId": "trace_01H..."
}
```

### Auth forwarding

| `auth.type` | Behavior |
|-------------|----------|
| `forwarded_jwt` | MCP Client signs short-lived JWT with `authContext` claims; `Authorization: Bearer` |
| `api_key` | Client attaches key from vault |
| `oauth` | Client uses stored refresh token for server |
| `none` | Public read-only tools only (discouraged) |

JWT claims:

```json
{
  "sub": "usr_01H...",
  "tenantId": "ten_01H...",
  "scopes": ["tools:billing:read"],
  "aud": "mcp-billing",
  "exp": 300
}
```

### Response normalization

```json
{
  "invocationId": "inv_01H...",
  "status": "success",
  "content": [
    { "type": "text", "text": "{\"status\":\"paid\"}" }
  ],
  "structuredContent": { "status": "paid" },
  "durationMs": 342
}
```

Client parses `structuredContent` when available; else parses text JSON.

### Error normalization

| MCP error | Platform code | retryable |
|-----------|---------------|-----------|
| Timeout | `tool_timeout` | true |
| Connection refused | `server_unavailable` | true |
| 401/403 | `tool_unauthorized` | false |
| Invalid args | `tool_invalid_args` | false |
| Tool internal error | `tool_error` | depends on server hint |

---

## Resources

MCP resources provide **read-only context** (files, configs, live state).

### When ORCH reads resources

| Trigger | Example |
|---------|---------|
| Context assembler prefetch | Wiki page before answering doc question |
| Tool-less retrieval | `read_resource` when model lacks tool for static data |
| Tenant config | `resourcePrefetch: ["wiki://policy/refunds"]` on certain intents |

### Flow

```
1. list_resources() at discovery — cache URI templates
2. At assemble time: select URIs by intent/keyword match
3. read_resource(uri) with auth context
4. Inject content into ContextBundle as `resource` source (budget from KB allocation)
```

### Cache

- Resource content TTL: **5 min** default (tenant-configurable).
- ETag support when server provides versioning.

---

## Prompts

MCP servers may expose **prompt templates** (parameterized).

### Merge strategy

```
1. list_prompts() at discovery
2. If tenant config maps intent → promptName, fetch prompt
3. Merge with tenant systemPrompt:
     - MCP prompt as "domain instructions" section
     - Tenant system prompt takes precedence on conflict
4. Render with arguments from user message extraction or channel context
```

Example: CRM server prompt `create_ticket_template` merged under system instructions for support intents.

---

## Circuit Breaker

Per `(tenantId, serverId)`:

| State | Behavior |
|-------|----------|
| `closed` | Normal |
| `open` | Fail fast; tools from server excluded from context |
| `half_open` | Allow 1 probe call after cooldown |

Open after **5 failures in 60 s**. Cooldown **30 s**. Emit `mcp.circuit_open` metric.

---

## Sandbox / Mock Server

For integrator testing (`cbp_test_*` API keys):

- Route to platform **mock MCP server** with canned tools.
- `dryRun: true` on invocation logs call but returns mock data (tenant flag).
- No side effects on production MCP servers.

---

## Multi-Server Topology

```
Tenant tool registry (merged):
  billing.get_invoice    ← mcp_billing
  billing.list_invoices  ← mcp_billing
  crm.get_contact        ← mcp_crm
  wiki.search            ← mcp_wiki
```

Qualified names prevent collisions. Model always sees `billing.get_invoice`, not bare `get_invoice`.

---

## Security

- MCP servers on tenant allowlist only; no arbitrary URL from user input.
- Tool arguments validated against JSON Schema before invocation.
- Server responses scanned by output guardrails before model continuation.
- Audit log: every invocation with `tenantId`, `userId`, `toolName`, `status` (not full args if sensitive).

---

## Observability

| Event | Fields |
|-------|--------|
| `mcp.discovery` | serverId, toolCount, durationMs |
| `mcp.invoke` | serverId, toolName, status, durationMs |
| `mcp.circuit_open` | serverId, failureCount |

Spans nested under turn trace (`16-observability-and-slos.md`).

---

## Open Questions

1. MCP server marketplace / third-party registry — P3.
2. Bidirectional MCP (server calls back to platform) — not in v1.
3. Tool result streaming for long-running tools — P2 with async turns.
