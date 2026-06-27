# Attachments & Media

**Status:** Draft  
**Priority:** P2  
**Closes gaps:** Architecture gaps §7 (Attachments & Rich Media)

---

## Purpose

Define **file upload**, **storage**, **extraction pipelines**, **multimodal model input**, and **channel-specific media limits**. Attachments flow from surfaces → object store → ORCH/model with guardrail scanning.

**Related docs:**

- [`01-api-and-streaming-events.md`](01-api-and-streaming-events.md) — attachment API
- [`02-data-model-and-persistence.md`](02-data-model-and-persistence.md) — `Attachment` entity
- [`06-model-provider.md`](06-model-provider.md) — vision/multimodal content parts
- [`09-knowledge-base-and-ingestion.md`](09-knowledge-base-and-ingestion.md) — document parsing overlap
- [`11-chat-app.md`](11-chat-app.md) — upload UX
- [`13-channel-adapters.md`](13-channel-adapters.md) — per-platform media limits
- [`14-guardrails.md`](14-guardrails.md) — PII scan on extracted text

---

## Scope

### In scope

- Upload API and presigned URLs
- Storage, virus scan, TTL
- Text extraction by file type
- Image/audio/video handling for multimodal models
- Channel adapter normalization
- Size/type allowlists per tenant and channel

### Out of scope

- Video generation output
- Real-time voice conversation (STT/TTS streaming) — future

---

## Attachment Entity

See `02-data-model-and-persistence.md`. Key fields:

```json
{
  "id": "att_01H...",
  "tenantId": "ten_01H...",
  "userId": "usr_01H...",
  "filename": "invoice.pdf",
  "mimeType": "application/pdf",
  "sizeBytes": 1048576,
  "storageKey": "s3://bucket/ten_01H.../att_01H...",
  "status": "ready",
  "extractedText": "...",
  "thumbnailKey": null,
  "metadata": {
    "pageCount": 3,
    "width": null,
    "height": null,
    "durationSec": null,
    "virusScanStatus": "clean"
  },
  "expiresAt": null,
  "createdAt": "..."
}
```

**Status flow:** `uploading` → `processing` → `ready` | `failed`

---

## Upload API

### Initiate upload

```
POST /v1/attachments
{
  "filename": "invoice.pdf",
  "mimeType": "application/pdf",
  "sizeBytes": 1048576,
  "conversationId": "conv_01H...",
  "purpose": "message" | "knowledge_base"
}
```

Response:

```json
{
  "id": "att_01H...",
  "uploadUrl": "https://s3.../presigned...",
  "uploadMethod": "PUT",
  "uploadHeaders": { "Content-Type": "application/pdf" },
  "expiresAt": "2026-06-26T10:15:00Z"
}
```

### Complete upload

```
POST /v1/attachments/{id}/complete
```

Triggers processing pipeline. Returns `202`; client polls `GET /v1/attachments/{id}`.

### Direct multipart (alternative)

For large files: `POST /v1/attachments/multipart` → part URLs → `complete`.

---

## Allowlists

### MIME types (platform default)

| Category | Allowed types |
|----------|---------------|
| Documents | `application/pdf`, `text/plain`, `text/markdown`, `application/vnd.openxmlformats-officedocument.wordprocessingml.document` |
| Images | `image/jpeg`, `image/png`, `image/gif`, `image/webp` |
| Audio | `audio/mpeg`, `audio/wav`, `audio/ogg` |
| Spreadsheets | `text/csv`, `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` |

Blocked: executables, archives with executables (`application/x-msdownload`), `text/html` (XSS risk in previews).

### Size limits

| Surface | Default max | Enterprise |
|---------|-------------|------------|
| Chat app | 50 MB | 200 MB |
| Embed widget | 10 MB | 50 MB |
| Slack | 4 MB (platform) | — |
| Teams | 4 MB | — |
| WhatsApp | 100 MB doc / 5 MB image | — |
| Telegram | 50 MB | — |

Tenant config `attachments.maxSizeBytes` caps below platform max.

### Per-message limits

- Max attachments per message: **5** (default)
- Max total size per message: **100 MB** (chat app)

---

## Storage

| Property | Value |
|----------|-------|
| Backend | S3-compatible object store |
| Path | `{tenantId}/attachments/{att_id}/{filename}` |
| Encryption | SSE-KMS (`15-security-and-compliance.md`) |
| Access | Presigned GET URLs, TTL 1 h |
| Public URLs | Never; always authenticated or presigned |

### Ephemeral attachments

Unlinked to a message within **7 days** → delete blob + metadata.

Linked attachments persist per conversation retention policy.

---

## Processing Pipeline

```
upload complete
  → virus scan (ClamAV or cloud scanner)
  → if infected: status failed, notify user
  → route by mimeType:
       image/*     → validate dimensions, generate thumbnail, optional vision describe
       application/pdf → text extraction
       docx/csv/txt/md → text extraction
       audio/*     → transcription (Whisper or provider API)
  → guardrails.input on extractedText (PII, secrets)
  → status ready
```

### Text extraction

| Type | Method |
|------|--------|
| PDF | pdfplumber / Apache Tika; OCR fallback for scanned pages |
| DOCX | python-docx / Tika |
| CSV | Parse rows; sample first 100 rows for context |
| HTML | Blocked at upload unless KB connector |

`extractedText` capped at **500 KB**; larger docs truncated with notice for message context (full text available for KB ingestion).

### Images

| Step | Output |
|------|--------|
| Validate | Max 20 MP; reject decompression bombs |
| Thumbnail | 256px max edge for UI |
| Vision describe (optional) | Alt text for accessibility + model context if no user question about image |

### Audio

- Transcribe to text via STT model
- Store transcript in `extractedText`
- Original audio retained for replay; passed to audio-capable models if supported

### Processing SLA

| Size | Target processing time |
|------|------------------------|
| < 1 MB | < 5 s p99 |
| < 10 MB | < 30 s p99 |
| < 50 MB | < 120 s p99 |

---

## Multimodal Model Input

ORCH builds message content parts (`06-model-provider.md`):

```json
{
  "role": "user",
  "content": [
    { "type": "text", "text": "What's in this invoice?" },
    { "type": "image", "attachmentId": "att_01H..." },
    { "type": "file", "attachmentId": "att_02H...", "extractedText": "..." }
  ]
}
```

### Routing rules

| Attachment type | Model input |
|-----------------|-------------|
| Image + vision model | Native image part |
| Image + no vision | `extractedText` from vision describe pass |
| PDF/doc | `extractedText` in text part or file reference |
| Audio | Transcript as text; optional audio part if model supports |

Context assembler counts attachment tokens toward conversation budget.

---

## Channel Adapter Normalization

### Inbound (platform → Chatbot)

```
Platform media URL/id
  → adapter downloads (within platform size limit)
  → POST /v1/attachments (internal) or streaming upload
  → attach to InboundMessage.attachments
```

### Outbound (Chatbot → platform)

```
OutboundMessage attachment
  → if within channel limit: upload to platform API
  → if over limit: share presigned link (if allowed) or text summary only
```

| Channel | Outbound media |
|---------|----------------|
| Slack | `files.upload` or image block |
| Teams | Adaptive Card Image or file consent |
| WhatsApp | Media ID from re-upload |
| Telegram | `sendPhoto` / `sendDocument` |

---

## Security

| Control | Implementation |
|---------|----------------|
| Virus scan | Block malware before extraction |
| Content sniffing | Validate magic bytes vs declared MIME |
| PII in files | Guardrail scan on `extractedText` before model/memory |
| SSRF | URL fetch (KB) uses separate fetcher; attachment upload is client-direct to S3 |
| XSS in previews | Sanitize; PDF rendered in sandboxed viewer |

---

## KB vs Message Attachments

| `purpose` | Pipeline |
|-----------|----------|
| `message` | Extract for turn context; optional `indexInConversation` |
| `knowledge_base` | Hand off to `09-knowledge-base-and-ingestion.md` after scan |

Same upload API; routing at `complete` based on `purpose`.

---

## Errors

| Code | User message |
|------|--------------|
| `file_too_large` | "File exceeds size limit (50 MB)." |
| `type_not_allowed` | "This file type is not supported." |
| `virus_detected` | "File could not be uploaded for security reasons." |
| `processing_failed` | "We couldn't process this file. Try a different format." |

---

## Observability

| Event | Fields |
|-------|--------|
| `attachment.uploaded` | attId, mimeType, sizeBytes |
| `attachment.processed` | attId, durationMs, extractChars |
| `attachment.failed` | attId, errorCode |

---

## Open Questions

1. Client-side encryption before upload — enterprise P3.
2. Redaction preview (auto-blur PII in images) — P2.
3. Video file support — extract key frames + transcript P2.
