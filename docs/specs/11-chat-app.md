# Chat App

**Status:** Draft  
**Priority:** P1  
**Closes gaps:** Architecture gaps §16 (Chat app)

---

## Purpose

Define the **standalone chat application**: UX requirements, thread management, streaming display, attachments, settings, memory management, and accessibility. The chat app is the reference client for the platform API.

**Related docs:**

- [`01-api-and-streaming-events.md`](01-api-and-streaming-events.md) — API and SSE protocol
- [`03-auth-and-identity.md`](03-auth-and-identity.md) — login, SSO, session
- [`05-orch-turn-engine.md`](05-orch-turn-engine.md) — HITL approvals, cancellation
- [`08-memory.md`](08-memory.md) — memory review UI
- [`17-attachments-and-media.md`](17-attachments-and-media.md) — upload flow

---

## Scope

### In scope

- Core UX flows and layout
- Thread sidebar and conversation management
- Message rendering (markdown, code, tool cards)
- Streaming and reconnection
- Attachments and file upload
- Settings and preferences
- HITL approval UI
- Accessibility and i18n requirements

### Out of scope

- Embed widget — `12-embed-widget.md`
- Mobile native apps — future; mobile web required
- Admin console — `19-admin-and-operations.md`

---

## Layout

```
┌─────────────────────────────────────────────────────────────┐
│ Header: branding, user menu, settings                        │
├──────────────┬──────────────────────────────────────────────┤
│  Sidebar     │  Main chat area                               │
│  - New chat  │  - Message list (scrollable)                  │
│  - Thread    │  - Streaming indicator                        │
│    list      │  - Tool call cards (collapsible)              │
│  - Search    │  - Approval prompts (inline)                  │
│              │  - Composer (text + attach + send/stop)       │
└──────────────┴──────────────────────────────────────────────┘
```

Responsive: sidebar collapses to drawer on viewport < 768px.

---

## Core Flows

### New conversation

1. User clicks "New chat" → `POST /v1/conversations`
2. Navigate to empty thread; focus composer
3. First message auto-titles conversation (ORCH or client-side summary)

### Send message

1. User types + optional attachments → `POST .../messages` with `X-Idempotency-Key`
2. Optimistic UI: show user message immediately
3. Open SSE stream from `streamUrl`
4. Render `text.delta` incrementally
5. On `tool_call.started` → show tool card (spinner)
6. On `tool_call.result` → update card with result preview
7. On `turn.complete` → finalize message, update usage badge (optional)

### Stop generation

- Send button becomes **Stop** while turn running
- Click → `POST .../turns/{turnId}/cancel`
- Preserve partial assistant text with "Stopped" indicator

### Regenerate

- On assistant message: "Regenerate" action
- `POST .../messages/{id}/regenerate`
- Replace assistant message in UI (old version hidden, not deleted from API)

### Edit message

- On user message: "Edit" → inline editor
- Resubmit creates new branch per `02-data-model-and-persistence.md`
- UI shows latest branch only by default; "Show previous versions" expandable

---

## Thread Sidebar

| Feature | API |
|---------|-----|
| List threads | `GET /v1/conversations?cursor=...` |
| Search | `GET /v1/conversations?q=...` (server-side full-text) |
| Rename | `PATCH /v1/conversations/{id}` |
| Archive | `POST .../archive` |
| Delete | `DELETE ...` with confirm dialog |

Sort: `lastMessageAt` descending. Group by date (Today, Yesterday, Previous).

---

## Message Rendering

### Content types

| Type | Rendering |
|------|-----------|
| Text | Markdown (GFM): headings, lists, tables, links |
| Code | Syntax-highlighted fenced blocks; copy button |
| Tool call | Card: tool name, args (collapsed JSON), status, result |
| Tool result | Hidden in main flow; visible in tool card expand |
| Image | Inline preview from attachment URL |
| Guardrail notice | Subtle banner when `guardrail.triggered` with `action: rewrite` |

### Streaming

- Token batching: flush to DOM every 50 ms or 20 chars (whichever first) to reduce layout thrash
- Cursor blink during stream
- Auto-scroll if user near bottom; pause auto-scroll if user scrolls up ("Jump to latest" pill)

### Citations

When assistant cites KB sources (`metadata.citations`):

```
[1] Refund Policy, p.3
```

Click opens source preview panel.

---

## Attachments

1. Drag-drop or paperclip → `POST /v1/attachments` (presigned upload)
2. Show upload progress per file
3. Thumbnail in composer; sent with message `attachments` array
4. Max files per message: 5 (tenant config)
5. Supported types per `17-attachments-and-media.md`

---

## HITL Approvals

On `approval.required`:

```
┌─────────────────────────────────────────┐
│  Confirm charge of $500 to card •••4242? │
│  [Approve]  [Deny]                       │
└─────────────────────────────────────────┘
```

- `POST /v1/.../approvals/{id}/resolve`
- Disable composer while paused (or allow queue per concurrency policy)
- Show countdown to `expiresAt` if < 1 h

---

## Settings

### User settings (`GET/PATCH /v1/settings`)

- Language preference
- Theme: light / dark / system
- Notification preferences

### Memory management (`GET/DELETE /v1/memory`)

- List memories with search
- Delete individual items ("Forget")
- "Export my data" → triggers GDPR export job (`15-security-and-compliance.md`)

### Connected integrations (future)

- Link Slack, etc. via `POST /v1/auth/link/{provider}`

---

## Authentication UI

- Email/password login
- SSO button when tenant has OIDC configured
- Session refresh transparent to user
- Logout clears local state + `POST /v1/auth/logout`

---

## Accessibility (WCAG 2.1 AA)

| Requirement | Implementation |
|-------------|----------------|
| Keyboard navigation | Tab through sidebar, messages, composer; Enter to send (Shift+Enter newline) |
| Screen reader | `aria-live="polite"` on streaming region; announce tool completion |
| Focus management | Focus composer after send; trap focus in modals |
| Color contrast | 4.5:1 minimum; don't rely on color alone for status |
| Reduced motion | Respect `prefers-reduced-motion`; disable streaming animation |

---

## Internationalization

- UI strings externalized (i18n keys)
- RTL layout support (`dir=rtl` for Arabic, Hebrew)
- Date/time formatted per user locale
- Assistant language follows user preference + graph memory

---

## Offline / Resilience

- **Draft messages:** save composer text to `localStorage` per conversationId
- **SSE reconnect:** auto-reconnect with `Last-Event-ID` (exponential backoff, max 30 s)
- **Network error:** toast + retry button on failed send
- Native offline mode: not in v1; show offline banner

---

## Tech Stack (recommended)

| Layer | Choice |
|-------|--------|
| Framework | React + Next.js or Vite SPA |
| Streaming | EventSource API with fetch fallback polyfill |
| State | React Query for API; local state for stream |
| Markdown | react-markdown + remark-gfm |
| Auth | httpOnly cookie or secure memory token |

Implementation choice is not normative; behavior is.

---

## Open Questions

1. Voice input — P2.
2. Split view (compare model responses) — P3.
3. Shared team threads — when `shared_conversations` enabled.
