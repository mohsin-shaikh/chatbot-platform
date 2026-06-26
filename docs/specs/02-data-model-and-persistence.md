# Data Model & Persistence

**Status:** Draft  
**Priority:** P0  
**Closes gaps:** Architecture gaps §2 (Conversation & Message Data Model)

---

## Purpose

Define the **canonical entities**, **relationships**, **field schemas**, and **storage tiers** for the chatbot platform. Every layer — API, ORCH, memory, observability replay — depends on this model.

**Related docs:**

- [`01-api-and-streaming-events.md`](01-api-and-streaming-events.md) — REST shapes that expose these entities
- [`03-auth-and-identity.md`](03-auth-and-identity.md) — `User`, identity linking
- [`04-multi-tenancy-and-config.md`](04-multi-tenancy-and-config.md) — `Tenant`, isolation
- [`08-memory.md`](08-memory.md) — graph + vector memory detail
- [`15-security-and-compliance.md`](15-security-and-compliance.md) — retention, delete cascades

---

## Scope

### In scope

- Core entities: Tenant, User, Conversation, Message, Turn, ToolCall, Attachment, MemoryRecord
- ID format, timestamps, soft-delete conventions
- Turn vs message semantics, ordering, edits, regeneration
- Cross-channel conversation policy
- Storage tier assignment and retention defaults

### Out of scope

- Memory graph/vector schema internals — see `08-memory.md`
- KB document entities — see `09-knowledge-base-and-ingestion.md`
- Billing meter records — see `20-billing-and-metering.md`

---

## Conventions

### Identifiers

All primary keys are **prefixed ULIDs** (sortable, URL-safe):

| Entity | Prefix | Example |
|--------|--------|---------|
| Tenant | `ten_` | `ten_01HXYZ...` |
| User | `usr_` | `usr_01HXYZ...` |
| Conversation | `conv_` | `conv_01HXYZ...` |
| Message | `msg_` | `msg_01HXYZ...` |
| Turn | `turn_` | `turn_01HXYZ...` |
| ToolCall | `tc_` | `tc_01HXYZ...` |
| Attachment | `att_` | `att_01HXYZ...` |
| MemoryRecord | `mem_` | `mem_01HXYZ...` |

### Timestamps

Every entity has `createdAt` and `updatedAt` (ISO 8601 UTC). Soft-deleted records add `deletedAt`.

### Multi-tenancy

Every tenant-scoped row includes `tenantId`. Queries always filter by `tenantId` from the auth context. See `04-multi-tenancy-and-config.md` for isolation strategy.

### Soft delete

`Conversation`, `Message`, `MemoryRecord`, and `Attachment` support soft delete. Hard delete runs asynchronously via compliance jobs (`15-security-and-compliance.md`).

---

## Entity Definitions

### Tenant

Owned by platform. See `04-multi-tenancy-and-config.md` for config fields.

```json
{
  "id": "ten_01H...",
  "name": "Acme Corp",
  "slug": "acme",
  "status": "active",
  "plan": "pro",
  "region": "us-east-1",
  "createdAt": "...",
  "updatedAt": "..."
}
```

### User

A person (or service principal) within a tenant.

```json
{
  "id": "usr_01H...",
  "tenantId": "ten_01H...",
  "email": "jane@acme.com",
  "displayName": "Jane Doe",
  "role": "member",
  "status": "active",
  "externalIds": {
    "slack": "U123ABC",
    "teams": "aad-object-id",
    "host_app": "user_42"
  },
  "preferences": {
    "language": "en",
    "timezone": "America/New_York"
  },
  "createdAt": "...",
  "updatedAt": "..."
}
```

- `externalIds`: map of channel/provider → platform-external identifier. Populated by channel identity linking (`03-auth-and-identity.md`).
- Anonymous/guest users get a `usr_` ID with `status: "guest"` and optional TTL.

### Conversation

A durable thread of messages between a user and the assistant.

```json
{
  "id": "conv_01H...",
  "tenantId": "ten_01H...",
  "userId": "usr_01H...",
  "title": "Billing question",
  "channel": "chat_app",
  "status": "active",
  "metadata": {
    "pageUrl": "https://app.example.com/invoices/123",
    "slackChannelId": "C123",
    "slackThreadTs": "1234.5678"
  },
  "messageCount": 12,
  "lastMessageAt": "...",
  "createdAt": "...",
  "updatedAt": "...",
  "deletedAt": null
}
```

**`channel` values:** `chat_app` | `embed` | `slack` | `teams` | `whatsapp` | `telegram` | `api`

**`status` values:** `active` | `archived` | `deleted`

#### Cross-channel policy

**Default (v1):** one conversation per `(tenantId, userId, channel)` thread. A Slack thread and a chat-app thread are **separate** conversations even for the same user.

**Optional (future flag):** `unified_inbox` tenant feature merges channels under one conversation ID with `metadata` tracking per-channel thread handles. Not required for v1.

#### Lifecycle

| Action | Effect |
|--------|--------|
| Create | New `conv_` with `status: active` |
| Archive | `status → archived`; no new messages |
| Delete (soft) | `status → deleted`, `deletedAt` set; hidden from list |
| Rename | Update `title` |
| Fork | Create new conversation copying messages up to a given `messageId` |

### Message

An atomic unit of content in a conversation.

```json
{
  "id": "msg_01H...",
  "tenantId": "ten_01H...",
  "conversationId": "conv_01H...",
  "turnId": "turn_01H...",
  "role": "assistant",
  "content": [
    { "type": "text", "text": "Your invoice was paid on June 1." },
    { "type": "tool_call", "toolCallId": "tc_01H...", "name": "billing.get_invoice", "arguments": {} }
  ],
  "attachments": ["att_01H..."],
  "metadata": {
    "model": "gpt-4.1",
    "finishReason": "stop",
    "inputTokens": 800,
    "outputTokens": 120,
    "guardrailActions": []
  },
  "sequence": 7,
  "parentMessageId": null,
  "createdAt": "...",
  "updatedAt": "...",
  "deletedAt": null
}
```

**`role` values:** `user` | `assistant` | `system` | `tool`

**Content parts:**

| `type` | Fields | Notes |
|--------|--------|-------|
| `text` | `text` | Markdown supported in chat app |
| `image` | `attachmentId` or `url` | Vision model input |
| `file` | `attachmentId` | Reference to uploaded file |
| `tool_call` | `toolCallId`, `name`, `arguments` | Assistant requested tool |
| `tool_result` | `toolCallId`, `result` | Tool output fed back to model |

**`sequence`:** monotonically increasing integer per conversation. Defines display order. Gaps are not reused.

**`parentMessageId`:** set when message is an edit or regeneration branch.

### Turn

One user message through to the assistant's complete reply. A turn may produce **multiple** assistant messages (tool-call loops).

```json
{
  "id": "turn_01H...",
  "tenantId": "ten_01H...",
  "conversationId": "conv_01H...",
  "userId": "usr_01H...",
  "userMessageId": "msg_01H...",
  "assistantMessageIds": ["msg_01H...", "msg_01H..."],
  "status": "completed",
  "traceId": "trace_01H...",
  "usage": {
    "inputTokens": 1200,
    "outputTokens": 340,
    "toolCalls": 1
  },
  "startedAt": "...",
  "completedAt": "...",
  "cancelledAt": null,
  "error": null
}
```

**`status` values:** `pending` | `running` | `paused` (HITL) | `completed` | `cancelled` | `failed`

#### Turn vs message

```
User sends message
  → creates Message (role=user) + Turn (status=running)
  → model responds with text
      → creates Message (role=assistant)
  → model responds with tool calls
      → creates Message (role=assistant, tool_call parts)
      → executes tools
      → creates Message (role=tool, tool_result parts)
  → model responds with final text
      → creates Message (role=assistant)
  → Turn (status=completed)
```

One turn, three assistant-facing messages (initial text, tool call, final text) plus tool result messages.

### ToolCall

Durable record of a tool invocation within a turn.

```json
{
  "id": "tc_01H...",
  "tenantId": "ten_01H...",
  "turnId": "turn_01H...",
  "messageId": "msg_01H...",
  "serverId": "mcp_billing",
  "name": "get_invoice",
  "arguments": { "invoiceId": "inv_123" },
  "result": { "status": "paid" },
  "status": "success",
  "durationMs": 342,
  "error": null,
  "createdAt": "..."
}
```

**`status` values:** `pending` | `running` | `success` | `error` | `cancelled`

### Attachment

```json
{
  "id": "att_01H...",
  "tenantId": "ten_01H...",
  "userId": "usr_01H...",
  "conversationId": "conv_01H...",
  "filename": "invoice.pdf",
  "mimeType": "application/pdf",
  "sizeBytes": 1048576,
  "storageKey": "s3://bucket/ten_01H.../att_01H...",
  "status": "ready",
  "extractedText": "...",
  "expiresAt": null,
  "createdAt": "...",
  "deletedAt": null
}
```

**`status` values:** `uploading` | `processing` | `ready` | `failed` | `deleted`

### MemoryRecord

Pointer to a memory entry in graph or vector store. Full schema in `08-memory.md`.

```json
{
  "id": "mem_01H...",
  "tenantId": "ten_01H...",
  "userId": "usr_01H...",
  "type": "preference",
  "store": "graph",
  "storeRef": "node_01H...",
  "content": "Prefers email notifications",
  "scope": "user",
  "source": {
    "turnId": "turn_01H...",
    "conversationId": "conv_01H..."
  },
  "createdAt": "...",
  "deletedAt": null
}
```

**`type` values:** `preference` | `fact` | `episodic`  
**`store` values:** `graph` | `vector`  
**`scope` values:** `user` | `team` | `tenant`

---

## Edits & Regeneration

### Edit user message

1. Soft-delete messages after the edited message in the conversation (or mark as `superseded`).
2. Create new user message with `parentMessageId` pointing to the original.
3. Start a new turn from the edited message.
4. Memory written from superseded turns is **not** auto-deleted; user may "forget" via memory API. ORCH re-evaluates memory extraction on the new turn.

### Regenerate assistant reply

1. Soft-delete assistant messages from the target turn.
2. Re-run turn with same user message; new `turnId`, link via `parentMessageId` or `metadata.regeneratedFrom`.
3. Original turn marked `status: superseded`.

---

## Entity Relationship Diagram

```
Tenant 1──* User
User 1──* Conversation
Conversation 1──* Message
Conversation 1──* Turn
Turn 1──* Message (user + assistant + tool)
Turn 1──* ToolCall
Conversation 1──* Attachment
User 1──* MemoryRecord
```

---

## Storage Tiers

| Tier | Store | Entities | Access pattern |
|------|-------|----------|----------------|
| **Hot** | Relational DB (Postgres) | Tenant, User, Conversation, Message, Turn, ToolCall | OLTP, indexed by tenant + user + conversation |
| **Warm** | Object store (S3/GCS) | Attachment blobs, extracted text | Presigned URL access |
| **Memory** | Graph DB | Preferences, relationships, facts | Traversal queries per user |
| **Memory** | Vector DB | Episodic snippets, KB chunks | Similarity search |
| **Cold** | Archive object store | Expired conversations, audit exports | Infrequent, compliance retrieval |

### Indexing strategy (hot tier)

| Table | Primary index | Secondary indexes |
|-------|---------------|-------------------|
| conversations | `id` | `(tenantId, userId, status, lastMessageAt DESC)` |
| messages | `id` | `(conversationId, sequence)` |
| turns | `id` | `(conversationId, startedAt DESC)`, `(tenantId, status)` |
| tool_calls | `id` | `(turnId)` |

All tables: row-level `tenantId` column, enforced by application + DB policy (RLS).

---

## Retention Defaults

Configurable per tenant (`04-multi-tenancy-and-config.md`). Platform defaults:

| Data | Default retention | After expiry |
|------|-------------------|--------------|
| Active conversations | Indefinite while `status=active` | — |
| Archived conversations | 1 year | Move to cold archive, then hard delete |
| Deleted conversations | 30-day grace | Hard delete cascade |
| Attachments (ephemeral) | 7 days if not linked to message | Hard delete blob |
| Tool call audit | 2 years | Archive |
| Turn traces (observability) | 90 days | See `16-observability-and-slos.md` |

---

## Consistency & Concurrency

| Rule | Behavior |
|------|----------|
| Message sequence | Assigned atomically per conversation (DB sequence or `SELECT MAX + 1` with lock) |
| Concurrent messages in same thread | Second message queues (default) or returns `409 turn_in_progress` (tenant config). See `05-orch-turn-engine.md`. |
| Read-your-writes | Message list reflects persisted state; stream may be ahead of list until `turn.complete` |
| Cross-region | Tenant pinned to region; no cross-region conversation access in v1 |

---

## Migration Strategy

- Schema changes via versioned migrations (e.g. Flyway, Alembic).
- New fields are additive; removed fields are deprecated for one release then dropped.
- Entity ID format is stable; never reuse deleted IDs.

---

## Open Questions

1. Message branching UI (tree view) — data model supports `parentMessageId`; UX is chat-app scope.
2. Conversation sharing (multi-user threads) — not in v1; would add `conversation_participants` join table.
3. Event sourcing for messages — current model is state-based; audit log captures changes separately.
