# Context Assembler

**Status:** Draft  
**Priority:** P1  
**Closes gaps:** Architecture gaps §13 (Context Assembler)

---

## Purpose

Define how ORCH builds the **prompt context** for each model call under a fixed **token budget**. The assembler gathers, ranks, truncates, and deduplicates sources into a structured **`ContextBundle`** for debugging, replay, and observability.

**Related docs:**

- [`05-orch-turn-engine.md`](05-orch-turn-engine.md) — calls assembler each loop iteration
- [`06-model-provider.md`](06-model-provider.md) — `countTokens`, model limits
- [`08-memory.md`](08-memory.md) — graph + vector retrieval input
- [`09-knowledge-base-and-ingestion.md`](09-knowledge-base-and-ingestion.md) — KB chunks as vector source
- [`10-mcp-integration.md`](10-mcp-integration.md) — tool definitions input
- [`14-guardrails.md`](14-guardrails.md) — injected constraints
- [`16-observability-and-slos.md`](16-observability-and-slos.md) — bundle logging, replay

---

## Scope

### In scope

- Source priority and budget allocation
- `ContextBundle` schema
- Truncation, summarization, deduplication
- Tool pruning / selection
- Caching assembled context for retry
- Debug exposure policy

### Out of scope

- Memory retrieval algorithms — `08-memory.md`
- Guardrail rule content — `14-guardrails.md`

---

## Token Budget

### Budget resolution

```
contextBudget = min(
  model.contextWindow - reservedOutputTokens,
  tenant.maxInputTokensPerTurn,
  plan.maxInputTokens
)
reservedOutputTokens = tenant.model.parameters.maxOutputTokens (default 4096)
```

Typical effective input budget: **100K–128K tokens** minus output reserve (model-dependent).

### Allocation strategy

**Fixed percentages (default):**

| Source | Budget % | Min tokens | Max tokens |
|--------|----------|------------|------------|
| System + tenant config | 15% | 500 | 8000 |
| Guardrail constraints | 5% | 100 | 2000 |
| Graph memory | 10% | 200 | 4000 |
| Vector memory (episodic) | 15% | 500 | 12000 |
| Knowledge base chunks | 20% | 500 | 16000 |
| Recent conversation | 25% | 1000 | 32000 |
| Tool definitions | 10% | 500 | 8000 |
| Channel context | 5% | 100 | 2000 |
| **Reserve (unallocated)** | 5% | — | — |

Unused budget from a source rolls into **reserve**, then reallocated to conversation history (oldest dropped last).

**Dynamic mode (optional tenant flag):**

Intent classifier (lightweight model or heuristic) shifts budget — e.g. factual query → +10% KB, −10% conversation; chitchat → inverse.

---

## Source Priority

Assembly order (highest priority first for truncation — **lowest priority dropped first**):

1. System instructions and tenant config
2. Guardrail-injected constraints
3. Graph memory — preferences, permissions
4. Knowledge base chunks (RAG)
5. Vector memory — episodic recall
6. Recent conversation turns
7. Tool definitions
8. Channel-specific context (embed page metadata, Slack thread)

### System instructions

Rendered from tenant `systemPrompt.template` with variables:

```json
{
  "tenant.name": "Acme Corp",
  "branding.name": "Acme Assistant",
  "user.displayName": "Jane",
  "user.preferences.language": "en",
  "channel": "embed"
}
```

### Guardrail constraints

Active rules inject additional system messages (e.g. "Do not disclose internal pricing").

### Graph memory

Formatted as structured facts:

```
User preferences:
- Language: en
- Notification: email

Permissions:
- Can access billing tools
```

### Vector memory & KB

Each chunk:

```json
{
  "id": "chunk_01H...",
  "text": "...",
  "score": 0.87,
  "source": "conversation" | "knowledge_base",
  "metadata": { "documentTitle": "Refund Policy", "page": 3 }
}
```

Rendered with citation markers for model to reference.

### Recent conversation

Sliding window of turns from hot store. Oldest messages dropped first when over budget.

**Summarization:** when > 20 turns and over 50% conversation budget, distant history compressed via rolling summary (stored in vector memory, keyed `conversation_summary:{convId}`).

### Tool definitions

From MCP discovery, after pruning (below). OpenAI function-calling format.

### Channel context

```json
{
  "pageUrl": "https://app.example.com/invoices/123",
  "entityIds": { "invoiceId": "inv_123" },
  "slackThreadTs": "1234.5678"
}
```

Injected as system or user prefix: `Context from host application: ...`

---

## Tool Pruning

Full tool list may exceed budget. Pruning pipeline:

```
1. Start with tenant MCP allowlist (04-multi-tenancy-and-config.md)
2. Filter by user token scopes (03-auth-and-identity.md)
3. Filter by relevance:
     a. embed-all mode: skip to step 4
     b. intent mode (default): rank tools by embedding similarity to user message
     c. server filter: only tools from servers tagged in tenant config
4. Cap at maxToolsInContext (default 20)
5. If still over token budget: drop lowest-ranked tools until fit
```

| Mode | Config | Use case |
|------|--------|----------|
| `intent` | Default | General purpose |
| `embed_all` | `toolPruning: none` | Small tool sets (< 15) |
| `server_filter` | Per-channel server list | Slack bot with limited tools |

---

## Deduplication

Before injection:

- Merge overlapping vector/KB chunks (cosine similarity > 0.95 → keep higher score).
- Merge duplicate graph facts (same predicate → latest wins).
- Strip redundant conversation turns that repeat retrieved memory verbatim.

---

## ContextBundle Schema

Persisted per model call iteration for replay:

```json
{
  "id": "bundle_01H...",
  "turnId": "turn_01H...",
  "iteration": 1,
  "createdAt": "...",
  "budget": {
    "total": 120000,
    "used": 98432
  },
  "sources": {
    "system": { "tokens": 1200, "hash": "abc..." },
    "guardrails": { "tokens": 300, "ruleIds": ["gr_01"] },
    "graphMemory": { "tokens": 800, "nodeIds": ["node_01H..."] },
    "vectorMemory": { "tokens": 2400, "chunkIds": ["chk_01H..."] },
    "knowledgeBase": { "tokens": 3200, "chunkIds": ["kb_01H..."] },
    "conversation": { "tokens": 85000, "messageIds": ["msg_01H..."] },
    "tools": { "tokens": 4500, "toolNames": ["billing.get_invoice"] },
    "channel": { "tokens": 232 }
  },
  "messages": [ ... ],
  "tools": [ ... ],
  "truncation": {
    "droppedMessageIds": ["msg_old..."],
    "droppedTools": ["crm.delete_contact"],
    "summarized": true
  },
  "configVersion": 12
}
```

**PII policy:** bundles stored for replay redact email/phone patterns in debug views; full bundle restricted to platform support with tenant consent.

---

## Caching

### Retry cache

On tool failure and model retry within same turn iteration:

- Reuse cached `ContextBundle` if `userMessage`, memory versions, and config version unchanged.
- Invalidate on new tool results appended.

### Cross-turn cache

System prompt + tool schemas cached per `(tenantId, configVersion)` — token count cached to skip re-counting.

---

## Debug Mode

Tenant flag `contextDebugEnabled` (admin only):

- `GET /v1/admin/turns/{turnId}/context` returns redacted `ContextBundle`.
- Chat app developer panel shows source breakdown (token bars per source).

---

## API (internal)

```
POST /internal/context/assemble
{
  "turnId": "...",
  "tenantId": "...",
  "userId": "...",
  "conversationId": "...",
  "userMessage": { ... },
  "iteration": 1,
  "priorToolResults": []
}
→ ContextBundle
```

---

## Open Questions

1. Learned budget allocation from feedback — P2 ML optimization.
2. Per-tool token cost in ranking — include in pruning score.
3. Real-time context from MCP resources — see `10-mcp-integration.md`; inject as optional source.
