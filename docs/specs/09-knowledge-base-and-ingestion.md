# Knowledge Base & Ingestion

**Status:** Draft  
**Priority:** P1  
**Closes gaps:** Architecture gaps §6 (Knowledge Base & Document Ingestion)

---

## Purpose

Define the **knowledge base (KB)** subsystem: document upload, connectors, chunking, embedding, indexing, scoped retrieval, citations, and freshness. KB content feeds the context assembler separately from episodic memory.

**Related docs:**

- [`07-context-assembler.md`](07-context-assembler.md) — KB chunks in context budget
- [`08-memory.md`](08-memory.md) — separation from episodic vector memory
- [`09-knowledge-base-and-ingestion.md`](09-knowledge-base-and-ingestion.md) — (this doc)
- [`17-attachments-and-media.md`](17-attachments-and-media.md) — file parsing overlap
- [`01-api-and-streaming-events.md`](01-api-and-streaming-events.md) — upload APIs

---

## Scope

### In scope

- KB entities and scopes
- Upload API and connectors
- Processing pipeline (parse → chunk → embed → index)
- Retrieval and citations in assistant replies
- Re-indexing, tombstoning, freshness

### Out of scope

- Episodic conversation memory — `08-memory.md`
- Web crawl at internet scale — basic URL fetch only in v1

---

## KB Scopes

| Scope | Visibility | Use case |
|-------|------------|----------|
| `tenant` | All users in tenant | Company wiki, policies |
| `team` | Team members | Department docs |
| `user` | Single user | Private notes |
| `conversation` | One conversation | Ad-hoc upload in thread |

Tenant feature `knowledgeBase` must be enabled (`04-multi-tenancy-and-config.md`).

---

## Entities

### KnowledgeBase

```json
{
  "id": "kb_01H...",
  "tenantId": "ten_01H...",
  "name": "Support Docs",
  "scope": "tenant",
  "scopeRef": null,
  "documentCount": 42,
  "status": "active",
  "createdAt": "..."
}
```

### Document

```json
{
  "id": "doc_01H...",
  "knowledgeBaseId": "kb_01H...",
  "tenantId": "ten_01H...",
  "title": "Refund Policy",
  "source": {
    "type": "upload",
    "uri": "s3://.../refund-policy.pdf",
    "originalFilename": "refund-policy.pdf"
  },
  "mimeType": "application/pdf",
  "status": "indexed",
  "chunkCount": 12,
  "metadata": { "author": "Legal", "version": "2.1" },
  "indexedAt": "...",
  "createdAt": "...",
  "deletedAt": null
}
```

**`status` values:** `pending` | `processing` | `indexed` | `failed` | `deleted`

### Chunk (vector index)

```json
{
  "id": "chk_01H...",
  "documentId": "doc_01H...",
  "knowledgeBaseId": "kb_01H...",
  "tenantId": "ten_01H...",
  "text": "...",
  "embedding": [ ... ],
  "metadata": {
    "title": "Refund Policy",
    "page": 3,
    "section": "Eligibility",
    "chunkIndex": 2
  }
}
```

Stored in **dedicated KB vector index** (separate from episodic memory in `08-memory.md`).

---

## Ingestion API

### Knowledge bases

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/v1/knowledge-bases` | Create KB |
| `GET` | `/v1/knowledge-bases` | List KBs |
| `GET` | `/v1/knowledge-bases/{id}` | Get KB |
| `DELETE` | `/v1/knowledge-bases/{id}` | Delete KB + all documents |

### Documents

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/v1/knowledge-bases/{id}/documents` | Upload or register document |
| `GET` | `/v1/knowledge-bases/{id}/documents` | List documents |
| `GET` | `/v1/documents/{id}` | Get document status |
| `DELETE` | `/v1/documents/{id}` | Tombstone document |
| `POST` | `/v1/documents/{id}/reindex` | Force re-process |

### Upload flow

```
1. POST /documents { filename, mimeType, sizeBytes }
   → { documentId, uploadUrl }  (presigned PUT)

2. Client PUT file to uploadUrl

3. POST /documents/{id}/complete
   → enqueue ingestion job

4. Poll GET /documents/{id} until status = indexed | failed
```

### URL import

```json
POST /v1/knowledge-bases/{id}/documents
{
  "source": { "type": "url", "uri": "https://docs.example.com/policy" }
}
```

Fetcher respects robots.txt; max page size 5 MB HTML.

---

## Connectors (v1.1+)

| Connector | Sync mode |
|-----------|-----------|
| Google Drive | OAuth; folder watch |
| Confluence | API token; space sync |
| S3 bucket | Prefix poll on schedule |
| Notion | Integration token |

Connector framework:

```json
{
  "connectorId": "conn_01H...",
  "type": "google_drive",
  "knowledgeBaseId": "kb_01H...",
  "config": { "folderId": "..." },
  "syncSchedule": "0 */6 * * *",
  "lastSyncAt": "..."
}
```

v1: manual upload + URL only. Connector spec stub for future.

---

## Processing Pipeline

```
Upload complete
  → Job: ingest_document(docId)
      1. Fetch blob from object store
      2. Parse by mimeType:
           pdf → text extraction (pdfplumber / Apache Tika)
           docx → structured text
           html → readability extract
           md/txt → direct
           images → OCR (optional) + vision description
      3. Chunk text
      4. Embed chunks (same model as episodic memory)
      5. Upsert to KB vector index
      6. Update document status → indexed
      7. On failure → status failed + error reason
```

### Chunking strategy

| Parameter | Default |
|-----------|---------|
| `chunkSizeTokens` | 512 |
| `chunkOverlapTokens` | 64 |
| `splitter` | Recursive by heading → paragraph → sentence |

Preserve metadata: `page`, `section`, `title` from document structure.

### Embedding

- Model: tenant `embeddingModel` (default `text-embedding-3-small` or equivalent).
- Batch size: 100 chunks per API call.
- Store embedding version in chunk metadata for re-embed migrations.

---

## Retrieval

Called by context assembler (`07-context-assembler.md`):

```
kb.search(
  tenantId,
  knowledgeBaseIds,  // filtered by scope + user access
  queryEmbedding,
  topK: 8,
  minScore: 0.78
)
```

### Access control

| Scope | Filter |
|-------|--------|
| `tenant` | All tenant users |
| `team` | User ∈ team |
| `user` | `userId` match |
| `conversation` | `conversationId` match |

### Conversation-attached docs

Message with attachments flagged `indexInConversation: true` → ingest into `conversation`-scoped KB linked to `conv_`.

---

## Citations

### In context

Chunks injected with markers:

```
[KB-1] Refund Policy (p.3): Eligible refunds within 30 days...
```

### In assistant response

ORCH instructs model to cite sources. Parser extracts citations → message metadata:

```json
{
  "citations": [
    {
      "marker": "KB-1",
      "documentId": "doc_01H...",
      "chunkId": "chk_01H...",
      "title": "Refund Policy",
      "page": 3,
      "snippet": "Eligible refunds within 30 days..."
    }
  ]
}
```

Chat app renders clickable citations (`11-chat-app.md`).

---

## Freshness & Tombstoning

| Event | Action |
|-------|--------|
| Document updated (re-upload) | Delete old chunks → re-ingest |
| Document deleted | Tombstone: `status: deleted`, delete vectors async |
| Connector sync detects change | Upsert by `sourceContentHash` |
| Source unavailable | Mark document `failed`; keep last indexed version (tenant config) |

`sourceContentHash`: SHA-256 of normalized text; skip re-embed if unchanged.

---

## Separation from Episodic Memory

| Aspect | Knowledge Base | Episodic Memory |
|--------|----------------|-----------------|
| Index | `kb_vectors` | `episodic_vectors` |
| Content | Curated documents | Conversation extracts |
| Retrieval budget | 20% context (default) | 15% context |
| TTL | Until document deleted | 180 days default |
| User forget API | Deletes document | Deletes memory record |
| Citations | Required | Optional |

Context assembler queries both; never merges indexes.

---

## Quotas

| Limit | Free | Pro |
|-------|------|-----|
| Documents | 50 | 5000 |
| Total storage | 100 MB | 10 GB |
| Max file size | 10 MB | 50 MB |

Enforced at upload; `403 quota_exceeded`.

---

## Observability

| Event | Fields |
|-------|--------|
| `kb.ingest.started` | documentId, mimeType |
| `kb.ingest.completed` | documentId, chunkCount, durationMs |
| `kb.ingest.failed` | documentId, error |
| `kb.search` | knowledgeBaseIds, hitCount, topScore |

---

## Open Questions

1. Hybrid search (BM25 + vector) — P2 quality improvement.
2. Table/chart extraction from PDF — enhanced parser P2.
3. Real-time collaborative doc editing sync — out of scope.
