# API & Streaming Events

**Status:** Draft  
**Priority:** P0  
**Closes gaps:** Architecture gaps §1 (Platform API & Event Contracts)

---

## Purpose

Define the **public Chatbot API** that all integration surfaces (chat app, embed widget, channel adapters) call, plus the **streaming event protocol**, **internal service contracts**, **webhooks**, and **idempotency** rules. This spec is the single source of truth for integrators and SDK generation.

**Related docs:**

- [`01-chatbot-platform-architecture.md`](../01-chatbot-platform-architecture.md) — layer topology, turn lifecycle
- [`02-data-model-and-persistence.md`](02-data-model-and-persistence.md) — entity shapes referenced here
- [`03-auth-and-identity.md`](03-auth-and-identity.md) — authentication headers and token types

---

## Scope

### In scope

- REST API (`/v1/...`) for conversations, messages, turns, attachments, memory, settings
- SSE streaming for turn events (primary); WebSocket as optional upgrade path
- Internal Chatbot → ORCH and ORCH → MCP Client contracts
- Outbound webhooks for hosts and channel platforms
- Idempotency, versioning, error envelope
- OpenAPI / AsyncAPI artifact requirements

### Out of scope

- Channel-specific webhook *inbound* formats (see `13-channel-adapters.md`)
- Embed SDK surface (`init`, `postMessage`) — see `12-embed-widget.md`
- Admin / tenant management API — see `19-admin-and-operations.md`

---

## API Conventions

### Base URL & versioning

```
https://{host}/v1
```

- Version is path-prefix only (`/v1`). Breaking changes require `/v2`.
- Non-breaking additions (new optional fields, new endpoints, new event types clients may ignore) stay within `/v1`.

### Authentication

All endpoints require a valid bearer token. Token type and claims are defined in `03-auth-and-identity.md`.

```
Authorization: Bearer <token>
```

### Request tracing

Clients may send:

```
X-Request-Id: <uuid>          # echoed in response and events
X-Idempotency-Key: <string>   # required on message POST (see Idempotency)
```

### Error envelope

All non-2xx responses use a consistent JSON body:

```json
{
  "error": {
    "code": "conversation_not_found",
    "message": "Conversation conv_abc does not exist or is not accessible.",
    "details": {},
    "requestId": "req_xyz"
  }
}
```

| HTTP | When |
|------|------|
| 400 | Validation error, malformed body |
| 401 | Missing or invalid token |
| 403 | Authenticated but not authorized |
| 404 | Resource not found (or hidden by authZ) |
| 409 | Conflict (e.g. turn already in progress, idempotency replay) |
| 429 | Rate limit / quota exceeded |
| 500 | Internal error |

### Pagination

List endpoints use cursor pagination:

```
GET /v1/conversations?limit=20&cursor=eyJ...
```

Response:

```json
{
  "data": [ ... ],
  "hasMore": true,
  "nextCursor": "eyJ..."
}
```

Default `limit`: 20. Max `limit`: 100.

---

## REST Endpoints

### Conversations

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/conversations` | Create conversation |
| `GET` | `/conversations` | List conversations for current user |
| `GET` | `/conversations/{conversationId}` | Get conversation metadata |
| `PATCH` | `/conversations/{conversationId}` | Update title, metadata |
| `DELETE` | `/conversations/{conversationId}` | Soft-delete conversation |
| `POST` | `/conversations/{conversationId}/archive` | Archive conversation |

**Create conversation** — `POST /v1/conversations`

```json
{
  "title": "Billing question",
  "channel": "chat_app",
  "metadata": {
    "pageUrl": "https://app.example.com/invoices/123"
  }
}
```

Response `201`:

```json
{
  "id": "conv_01H...",
  "title": "Billing question",
  "channel": "chat_app",
  "status": "active",
  "createdAt": "2026-06-26T10:00:00Z",
  "updatedAt": "2026-06-26T10:00:00Z",
  "metadata": { ... }
}
```

### Messages & turns

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/conversations/{conversationId}/messages` | List messages (paginated) |
| `POST` | `/conversations/{conversationId}/messages` | Send user message; starts a turn |
| `GET` | `/conversations/{conversationId}/messages/{messageId}` | Get single message |
| `POST` | `/conversations/{conversationId}/messages/{messageId}/regenerate` | Regenerate assistant reply from prior user message |
| `POST` | `/conversations/{conversationId}/turns/{turnId}/cancel` | Cancel in-flight turn |

**Send message** — `POST /v1/conversations/{conversationId}/messages`

Headers: `X-Idempotency-Key` (required)

```json
{
  "content": [
    { "type": "text", "text": "What's my invoice status?" }
  ],
  "attachments": ["att_01H..."],
  "metadata": {}
}
```

Response `202 Accepted` (turn started asynchronously):

```json
{
  "turnId": "turn_01H...",
  "userMessageId": "msg_01H...",
  "streamUrl": "/v1/conversations/conv_01H.../turns/turn_01H.../stream"
}
```

The assistant reply is delivered only via the stream (and persisted; retrievable via `GET .../messages`).

### Turn stream (SSE)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/conversations/{conversationId}/turns/{turnId}/stream` | SSE event stream for turn |

**Content-Type:** `text/event-stream`

Clients reconnect with `Last-Event-ID` header. Server replays missed events from a short buffer (≥ 5 minutes) or sends `turn.complete` / `error` if the turn already finished.

### Attachments

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/attachments` | Initiate upload (returns presigned URL or multipart handle) |
| `GET` | `/attachments/{attachmentId}` | Get attachment metadata |
| `DELETE` | `/attachments/{attachmentId}` | Delete ephemeral attachment |

See `17-attachments-and-media.md` for upload pipeline details.

### Memory (user-facing)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/memory` | List memory records for current user |
| `DELETE` | `/memory/{memoryId}` | Delete a memory record ("forget") |
| `POST` | `/memory` | Explicitly remember a fact |

### Settings

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/settings` | User preferences (language, notifications) |
| `PATCH` | `/settings` | Update user preferences |

---

## Streaming Event Protocol (SSE)

Each SSE event has:

```
id: <monotonic-event-id>
event: <event-type>
data: <json>
```

### Event types

| Event | Description | Terminal? |
|-------|-------------|-----------|
| `turn.started` | Turn accepted, processing began | No |
| `message.created` | A persisted message (user or assistant) | No |
| `text.delta` | Incremental assistant text token(s) | No |
| `text.done` | Final text for current assistant message segment | No |
| `tool_call.started` | Model requested a tool call | No |
| `tool_call.result` | Tool execution completed | No |
| `tool_call.failed` | Tool execution failed | No |
| `guardrail.triggered` | Guardrail fired (block, warn, rewrite) | No |
| `approval.required` | Turn paused for human approval (HITL) | No |
| `approval.resolved` | User approved or denied pending action | No |
| `memory.written` | Memory extraction persisted new record | No |
| `turn.complete` | Turn finished successfully | **Yes** |
| `turn.cancelled` | Turn cancelled by user or timeout | **Yes** |
| `error` | Unrecoverable error | **Yes** |

### Event payloads

**`turn.started`**

```json
{
  "turnId": "turn_01H...",
  "conversationId": "conv_01H...",
  "traceId": "trace_01H...",
  "timestamp": "2026-06-26T10:00:01Z"
}
```

**`text.delta`**

```json
{
  "turnId": "turn_01H...",
  "messageId": "msg_01H...",
  "delta": "Your invoice ",
  "index": 0
}
```

**`tool_call.started`**

```json
{
  "turnId": "turn_01H...",
  "toolCallId": "tc_01H...",
  "name": "billing.get_invoice",
  "arguments": { "invoiceId": "inv_123" },
  "serverId": "mcp_billing"
}
```

**`tool_call.result`**

```json
{
  "turnId": "turn_01H...",
  "toolCallId": "tc_01H...",
  "name": "billing.get_invoice",
  "result": { "status": "paid", "amount": 500 },
  "durationMs": 342
}
```

**`guardrail.triggered`**

```json
{
  "turnId": "turn_01H...",
  "checkpoint": "output",
  "ruleId": "no_pii_leak",
  "action": "rewrite",
  "severity": "warning"
}
```

**`approval.required`**

```json
{
  "turnId": "turn_01H...",
  "approvalId": "appr_01H...",
  "prompt": "Confirm charge of $500 to card ending 4242?",
  "expiresAt": "2026-06-26T18:00:00Z",
  "actions": ["approve", "deny"]
}
```

**`turn.complete`**

```json
{
  "turnId": "turn_01H...",
  "conversationId": "conv_01H...",
  "assistantMessageIds": ["msg_01H...", "msg_01H..."],
  "usage": {
    "inputTokens": 1200,
    "outputTokens": 340,
    "toolCalls": 1
  },
  "durationMs": 4200
}
```

**`error`**

```json
{
  "turnId": "turn_01H...",
  "code": "model_timeout",
  "message": "Model did not respond within the turn SLA.",
  "retryable": true
}
```

### Client reconnection

1. Client opens `GET .../stream` with `Accept: text/event-stream`.
2. On disconnect, client reconnects with `Last-Event-ID: <last-id>`.
3. Server replays buffered events after that ID, then continues live stream.
4. If turn already terminal, server sends the terminal event immediately and closes.

### WebSocket (optional)

A WebSocket endpoint `GET /v1/stream` may be offered for clients that need bidirectional communication (e.g. cancel mid-stream without a separate HTTP call). Event schema is identical; frames are JSON `{ "event": "...", "data": { ... } }`. SSE remains the **required** transport; WebSocket is optional.

---

## Idempotency

### Message POST

`POST /conversations/{id}/messages` **requires** `X-Idempotency-Key`.

- Scope: `(tenantId, userId, idempotencyKey)` — unique for 24 hours.
- First request: processes normally, stores result keyed by idempotency key.
- Duplicate request with same key and **identical body**: returns `202` with the **same** `turnId` / `userMessageId` / `streamUrl` (no duplicate turn).
- Duplicate request with same key but **different body**: returns `409` with `idempotency_key_reused`.

### Tool calls (internal)

ORCH deduplicates identical tool invocations within a single turn (same name + canonicalized arguments). See `05-orch-turn-engine.md`.

---

## Internal Service Contracts

### Chatbot → ORCH: `TurnRequest`

```json
{
  "turnId": "turn_01H...",
  "conversationId": "conv_01H...",
  "tenantId": "ten_01H...",
  "userId": "usr_01H...",
  "channel": "chat_app",
  "userMessage": {
    "id": "msg_01H...",
    "content": [ { "type": "text", "text": "..." } ],
    "attachments": []
  },
  "conversationHistory": [],
  "authContext": {
    "scopes": ["tools:billing:read"],
    "externalUserId": "host_user_42"
  },
  "traceId": "trace_01H..."
}
```

### ORCH → Chatbot: `TurnStream`

ORCH emits the same event types defined in the SSE section. Chatbot persists messages and forwards events to the connected client. Transport between Chatbot and ORCH is an internal message bus or gRPC stream (implementation choice); **event schema is fixed**.

### ORCH → MCP Client: `ToolInvocation`

```json
{
  "invocationId": "inv_01H...",
  "turnId": "turn_01H...",
  "serverId": "mcp_billing",
  "toolName": "get_invoice",
  "arguments": { "invoiceId": "inv_123" },
  "authContext": {
    "tenantId": "ten_01H...",
    "userId": "usr_01H...",
    "scopes": ["tools:billing:read"]
  },
  "timeoutMs": 30000
}
```

### MCP Client → ORCH: `ToolResult`

```json
{
  "invocationId": "inv_01H...",
  "status": "success",
  "result": { "status": "paid" },
  "durationMs": 342
}
```

Error variant:

```json
{
  "invocationId": "inv_01H...",
  "status": "error",
  "error": {
    "code": "tool_timeout",
    "message": "MCP server did not respond",
    "retryable": true
  },
  "durationMs": 30000
}
```

---

## Outbound Webhooks

Tenants configure webhook endpoints for async notification (embed hosts, integrations).

### Configuration

Managed via tenant config (`04-multi-tenancy-and-config.md`):

```json
{
  "webhooks": [
    {
      "url": "https://host.example.com/hooks/chatbot",
      "events": ["turn.complete", "approval.required"],
      "secret": "whsec_..."
    }
  ]
}
```

### Delivery

- `POST` to configured URL with `Content-Type: application/json`.
- Header: `X-Chatbot-Signature: sha256=<hmac>` (HMAC of body with webhook secret).
- Header: `X-Chatbot-Event: turn.complete`
- Retries: exponential backoff, 5 attempts over 24 hours.
- Payload mirrors the SSE event `data` object, plus envelope:

```json
{
  "id": "evt_01H...",
  "type": "turn.complete",
  "createdAt": "2026-06-26T10:00:05Z",
  "tenantId": "ten_01H...",
  "data": { ... }
}
```

### Webhook events (subset)

| Event | Use case |
|-------|----------|
| `turn.complete` | Host app updates UI, analytics |
| `approval.required` | Host surfaces approval in native UI |
| `tool_call.result` | Audit, side-effect triggers |
| `conversation.created` | CRM sync |

---

## Machine-Readable Artifacts

| Artifact | Location | Generator |
|----------|----------|-----------|
| OpenAPI 3.1 | `api/openapi.yaml` | Source of truth for REST |
| AsyncAPI 3.0 | `api/asyncapi.yaml` | SSE / webhook event schemas |

CI validates that implementation matches these specs. SDKs (TypeScript, Python) are generated from OpenAPI.

---

## Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| Stream time-to-first-byte | < 500 ms p99 after message POST |
| SSE buffer retention | ≥ 5 min for replay |
| API availability | 99.9% (see `16-observability-and-slos.md`) |
| Max concurrent streams per user | Configurable per tenant (default 3) |

---

## Open Questions

1. GraphQL layer in addition to REST — defer unless a concrete consumer needs it.
2. Batch message API for offline sync (mobile) — P2, chat app spec.
3. gRPC for internal Chatbot ↔ ORCH — implementation detail; event schema is fixed regardless.
