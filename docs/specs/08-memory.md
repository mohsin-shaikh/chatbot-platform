# Memory

**Status:** Draft  
**Priority:** P1  
**Closes gaps:** Architecture gaps §11 (Memory)

---

## Purpose

Define the **memory subsystem**: graph store for structured knowledge, vector store for semantic recall, write policy, scopes, retrieval, conflict resolution, decay, and user forget/delete flows.

**Related docs:**

- [`02-data-model-and-persistence.md`](02-data-model-and-persistence.md) — `MemoryRecord` entity
- [`05-orch-turn-engine.md`](05-orch-turn-engine.md) — retrieve at turn start, extract at end
- [`07-context-assembler.md`](07-context-assembler.md) — consumes retrieved memory
- [`09-knowledge-base-and-ingestion.md`](09-knowledge-base-and-ingestion.md) — KB vs episodic separation
- [`11-chat-app.md`](11-chat-app.md) — memory review UI
- [`15-security-and-compliance.md`](15-security-and-compliance.md) — GDPR delete cascades

---

## Scope

### In scope

- Graph memory schema and operations
- Vector memory schema and operations
- Write policy and extraction pipeline
- Scopes (user, team, tenant)
- Retrieval API for ORCH
- Forget, conflict resolution, decay
- User correction flows

### Out of scope

- KB document ingestion — `09-knowledge-base-and-ingestion.md`
- Conversation message storage — `02-data-model-and-persistence.md`

---

## Architecture

```
                    ┌─────────────────┐
                    │      ORCH       │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              ▼                             ▼
     ┌─────────────────┐          ┌─────────────────┐
     │   Graph Store   │          │  Vector Store   │
     │  (preferences,  │          │  (episodic,     │
     │   relations,    │          │   semantic      │
     │   facts)        │          │   recall)       │
     └─────────────────┘          └─────────────────┘
```

Namespace: all data scoped by `tenantId`; user/team scoping within namespace.

---

## Graph Memory

### Node types

| Type | Example | Properties |
|------|---------|------------|
| `User` | usr_01H... | displayName, linked |
| `Preference` | — | key, value |
| `Entity` | project, team | type, name, externalId |
| `Fact` | — | statement, confidence |
| `Topic` | billing | label |

### Edge types

| Edge | Example |
|------|---------|
| `prefers` | User → Preference |
| `member_of` | User → Entity (team) |
| `interested_in` | User → Topic |
| `knows` | User → Fact |
| `related_to` | Entity → Entity |
| `can_access` | User → Entity (permission) |

### Operations

```
graph.upsert(node | edge, scope)
graph.query(userId, traversalPattern)
graph.delete(nodeId | edgeId)
graph.deleteCascade(userId)  // compliance
```

### Query at turn time

Default traversal from `userId`:

1. All `prefers` edges (preferences)
2. `can_access` edges (permissions for tool context)
3. `knows` facts with `confidence >= 0.7`
4. `interested_in` topics (top 5 by recency)

Max depth: 2 hops. Max nodes: 50.

---

## Vector Memory

### Document schema

```json
{
  "id": "vec_01H...",
  "tenantId": "ten_01H...",
  "userId": "usr_01H...",
  "scope": "user",
  "type": "episodic",
  "text": "User discussed refund policy on June 10...",
  "embedding": [ ... ],
  "metadata": {
    "conversationId": "conv_01H...",
    "turnId": "turn_01H...",
    "timestamp": "2026-06-10T14:00:00Z",
    "source": "conversation_extraction"
  },
  "expiresAt": "2026-12-10T14:00:00Z",
  "createdAt": "..."
}
```

### Types

| Type | Store | TTL default |
|------|-------|-------------|
| `episodic` | Vector | 180 days |
| `conversation_summary` | Vector | Life of conversation + 90 days |
| `tool_output_summary` | Vector | 90 days |
| `knowledge_base` | Vector (separate index) | Until source deleted — see `09-...` |

### Retrieval

```
vector.search(
  tenantId,
  userId,
  queryEmbedding,
  filters: { scope, type, excludeTypes: ["knowledge_base"] },
  topK: 10,
  minScore: 0.75
)
```

KB chunks retrieved via separate path in context assembler (`09-knowledge-base-and-ingestion.md`).

---

## Write Policy

### When to write

Memory extraction runs **after** turn completes successfully (not on cancelled/failed turns unless partial facts flagged).

Skipped when:

- `tenant.memory.extractionEnabled = false`
- Guest user with `guestPersistence = false`
- Turn blocked by input guardrail

### Extraction pipeline

```
1. Model structured extraction (gpt-4.1-mini):
   Input: user message + assistant reply + tool summaries
   Output: { candidates: [ { type, content, confidence, scope } ] }

2. Classify each candidate:
   - preference → graph upsert (prefers edge)
   - fact → graph upsert (knows edge) if confidence >= 0.8
   - episodic → vector insert
   - discard → drop

3. Post-turn guardrail checkpoint on sensitive facts
   - block → skip write
   - require_consent → queue for user approval (chat app)

4. Persist + emit memory.written events
```

### Explicit user commands

Parsed from user message (model-assisted):

| Command | Action |
|---------|--------|
| "Remember that..." | Force graph/vector write, `confidence: 1.0` |
| "Forget..." | Delete matching records (graph + vector search) |
| "That's wrong" | Downrank or delete referenced memory from prior turn |

---

## Scopes

| Scope | Read | Write | Delete |
|-------|------|-------|--------|
| `user` | Owning user | Owning user, ORCH extraction | Owning user, admin |
| `team` | Team members | Team members, ORCH | Team admin |
| `tenant` | All tenant users | Admin, ORCH (policy facts only) | Admin |

Tenant config `memory.scopes` lists enabled scopes (default: `["user"]`).

---

## Conflict Resolution

| Scenario | Policy |
|----------|--------|
| Duplicate preference key | **Last-write-wins** with timestamp |
| Contradicting facts | Higher confidence wins; tie → latest; user explicit command always wins |
| Same episodic content | Dedup on insert (similarity > 0.95 → skip) |

User correction UI sets `confidence: 0` and tombstones old record.

---

## Decay & TTL

| Type | Behavior |
|------|----------|
| Episodic vector | `expiresAt` default 180 days; background job deletes |
| Preferences (graph) | No auto-expiry; persist until deleted |
| Conversation summaries | Expire 90 days after conversation archived |
| Low-confidence facts (< 0.6) | Auto-prune after 30 days |

---

## Forget & Delete

### User API

- `DELETE /v1/memory/{memoryId}` — single record
- `POST /v1/memory/forget` — `{ "query": "my phone number" }` — semantic search + delete matches

### Cascade (GDPR)

`deleteUser(userId)`:

1. Delete all graph nodes/edges for user
2. Delete all vector docs where `userId` matches
3. Soft-delete conversations (separate job)
4. Audit log entry (retained per compliance policy)

---

## ORCH Integration

### Retrieve (turn start)

```
MemoryContext retrieve(RetrieveRequest):
  graphFacts = graph.query(userId, defaultTraversal)
  episodic = vector.search(userId, embed(userMessage), topK=10)
  return { graphFacts, episodic, kbChunks: [] }  // kb from KB service
```

### Extract (turn end)

```
extractAndPersist(turn, conversationSlice):
  candidates = model.extract(...)
  apply write policy + guardrails
  persist approved candidates
```

---

## Quality & Evaluation

| Metric | Target | Measurement |
|--------|--------|-------------|
| Retrieval precision@5 | > 0.7 | Benchmark query set per tenant |
| False extraction rate | < 5% | User corrections / total writes |
| Forget completeness | 100% | Audit after delete requests |

Eval harness in `18-testing-and-eval.md`.

---

## Storage Backends (implementation)

| Store | Recommended | Notes |
|-------|-------------|-------|
| Graph | Neo4j, Amazon Neptune | Tenant namespace via label or graph name |
| Vector | Pinecone, pgvector, Weaviate | Separate indexes: episodic vs KB |

---

## Open Questions

1. Team-shared memory graph — v1.1 when `scope: team` enabled.
2. Memory export (user data portability) — format JSON; P2.
3. Federated memory across products — out of scope.
