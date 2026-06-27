# Channel Adapters

**Status:** Draft  
**Priority:** P1  
**Closes gaps:** Architecture gaps §16 (Messaging platforms)

---

## Purpose

Define the **channel adapter pattern** and **per-platform rules** for integrating messaging platforms (Slack, Microsoft Teams, WhatsApp, Telegram). Adapters normalize inbound/outbound messages between platform protocols and the Chatbot API.

**Related docs:**

- [`01-api-and-streaming-events.md`](01-api-and-streaming-events.md) — Chatbot API, webhooks
- [`02-data-model-and-persistence.md`](02-data-model-and-persistence.md) — `InboundMessage`, conversation metadata
- [`03-auth-and-identity.md`](03-auth-and-identity.md) — channel identity linking, OAuth
- [`05-orch-turn-engine.md`](05-orch-turn-engine.md) — HITL via platform buttons
- [`17-attachments-and-media.md`](17-attachments-and-media.md) — media limits per channel

---

## Scope

### In scope

- Adapter architecture and `InboundMessage` / `OutboundMessage` shapes
- Slack, Teams, WhatsApp, Telegram specifics
- Webhook security, deduplication, ordering
- OAuth install/uninstall flows
- Proactive messaging, typing indicators
- Group vs DM policies

### Out of scope

- Chat app and embed — `11-chat-app.md`, `12-embed-widget.md`
- Platform app store submission processes

---

## Adapter Architecture

```
Platform (Slack, etc.)
        │ webhook / event
        ▼
┌───────────────────┐
│ Channel Adapter   │  verify signature, dedupe, normalize
│ (stateless worker)│
└─────────┬─────────┘
          │ Chatbot API (REST + SSE or internal bus)
          ▼
┌───────────────────┐
│ Chatbot Backend   │
└─────────┬─────────┘
          │ ORCH ...
          ▼
┌───────────────────┐
│ Channel Adapter   │  render OutboundMessage → platform format
└─────────┬─────────┘
          │ platform API
          ▼
Platform reply
```

Each platform has a dedicated adapter service. Shared library for normalization, identity resolution, and rate limiting.

---

## Common Message Shapes

### InboundMessage

```json
{
  "id": "plat_msg_abc",
  "provider": "slack",
  "tenantId": "ten_01H...",
  "workspaceId": "T123",
  "channelId": "C456",
  "threadId": "1234.5678",
  "sender": {
    "externalId": "U789",
    "displayName": "Jane",
    "email": "jane@acme.com"
  },
  "content": [
    { "type": "text", "text": "What's my invoice status?" }
  ],
  "attachments": [],
  "timestamp": "2026-06-26T10:00:00Z",
  "metadata": {
    "isMention": true,
    "isDM": false
  }
}
```

### OutboundMessage

```json
{
  "conversationId": "conv_01H...",
  "turnId": "turn_01H...",
  "content": [
    { "type": "text", "text": "Your invoice was paid on June 1." }
  ],
  "attachments": [],
  "interactive": {
    "type": "approval",
    "approvalId": "appr_01H...",
    "actions": [
      { "id": "approve", "label": "Approve" },
      { "id": "deny", "label": "Deny" }
    ]
  },
  "metadata": {
    "replyToThread": "1234.5678"
  }
}
```

### Adapter responsibilities

| Step | Action |
|------|--------|
| Inbound | Verify signature → dedupe → resolve user → map/create conversation → `POST .../messages` |
| Stream | Subscribe to turn events (SSE or internal bus) |
| Outbound | Map events to platform messages; split long text |
| Interactive | Route button clicks to approval resolve API |

---

## Cross-Cutting Rules

### Webhook security

- Verify platform signature on every inbound request (HMAC or platform-specific).
- Reject requests older than **5 min** (replay protection).
- Return 200 quickly; process async if needed (> 3 s).

### Deduplication

- Key: `hash(provider, workspaceId, platformMessageId)`
- Store in Redis TTL 24 h; duplicates ack without reprocessing.

### Message ordering

- Per-thread serialization: process messages in `timestamp` order.
- Out-of-order: queue briefly (500 ms) or process with sequence guard.

### Rate limits

- Respect platform API limits; adapter maintains token bucket per workspace.
- Split long responses (see per-platform limits below).

### Conversation mapping

```
conversation.metadata = {
  slackChannelId, slackThreadTs
  // or teams conversationId, whatsapp phone, telegram chatId
}
```

One conversation per platform thread (default). New thread → new conversation.

---

## Slack

### Stack

- Slack Bolt (Node/Python) or Events API + Web API
- OAuth 2.0 for workspace install

### Inbound events

| Event | Handling |
|-------|----------|
| `message.im` | DM to bot |
| `app_mention` | @mention in channel |
| `message.channels` | If tenant enables channel listening |

### Policies

| Setting | Default |
|---------|---------|
| Respond in channels | Only when @mentioned |
| Respond in DMs | Always |
| Thread replies | Always reply in thread |

### Outbound rendering

| Content | Slack format |
|---------|--------------|
| Text | `chat.postMessage` with `mrkdwn` |
| Tool card | Block Kit: section + context |
| Approval | Block Kit: actions block with `approve`/`deny` buttons |
| Code | ` ``` ` fenced blocks |

### Limits

- Max message: 4000 chars → split into threaded replies
- Max blocks: 50 per message

### Install / uninstall

- OAuth scopes: `chat:write`, `im:history`, `app_mentions:read`, `users:read`
- Uninstall event → disable tenant channel config, revoke tokens

---

## Microsoft Teams

### Stack

- Bot Framework SDK
- Azure Bot registration per tenant or multi-tenant app

### Inbound

| Activity | Handling |
|----------|----------|
| `message` | Text, attachments |
| `invoke` | Adaptive Card action (approvals) |

### Outbound rendering

| Content | Teams format |
|---------|--------------|
| Text | Plain or HTML limited subset |
| Rich | Adaptive Cards |
| Approval | Adaptive Card `Action.Submit` |

### Policies

- Respond in channels: @mention required (default)
- Group chat: tenant configurable

### Limits

- ~28 KB per activity payload
- Adaptive Card: 28 actions max

---

## WhatsApp

### Stack

- WhatsApp Business Cloud API (Meta)
- Webhook verification + message templates for proactive

### Inbound

- Webhook `messages` object
- Text, image, document, audio, location

### Outbound

| Content | WhatsApp format |
|---------|-----------------|
| Text | Text message |
| Media | Image/document by media ID |
| Interactive | Reply buttons (≤ 3), list messages |

### Policies

- **24-hour session window:** free-form replies only within 24 h of user message
- Outside window: approved **template messages** only
- Proactive notifications: template required

### Limits

| Media | Max size |
|-------|----------|
| Image | 5 MB |
| Document | 100 MB |
| Audio | 16 MB |

Adapter uploads/downloads via platform media API → attachment pipeline (`17-attachments-and-media.md`).

---

## Telegram

### Stack

- Telegram Bot API (webhook or long polling)

### Inbound

- `message`, `callback_query` (inline keyboard)

### Outbound

| Content | Telegram format |
|---------|-----------------|
| Text | `sendMessage` with MarkdownV2 |
| Buttons | Inline keyboard |
| Media | `sendPhoto`, `sendDocument` |

### Policies

| Setting | Default |
|---------|---------|
| Group chats | Respond only on @mention or reply to bot |
| DMs | Always respond |
| Commands | `/start`, `/help` mapped to welcome flow |

### Limits

- Message text: 4096 chars → split messages
- File upload: 50 MB (bot API)

---

## Proactive / Outbound Messages

Initiated by platform (not direct reply to inbound):

| Trigger | Flow |
|---------|------|
| Async turn complete | Adapter receives webhook → push to channel |
| Approval timeout reminder | Scheduled job → template/bot message |
| Broadcast (admin) | Out of scope v1 |

Requires stored `channelId` + `threadId` in conversation metadata and valid platform credentials.

---

## Typing Indicators

| Platform | Support |
|----------|---------|
| Slack | `assistant.threads.setStatus` or typing in DM |
| Teams | `typing` activity |
| WhatsApp | Not available |
| Telegram | `sendChatAction: typing` |

Emit on `turn.started`; clear on `turn.complete`.

---

## HITL on Channels

```
1. ORCH emits approval.required
2. Adapter renders platform-specific interactive message
3. User clicks Approve/Deny
4. Adapter calls POST .../approvals/{id}/resolve
5. ORCH resumes; adapter streams final reply
```

Approval messages must include `approvalId` in callback payload for correlation.

---

## Error Handling

| Error | User-facing |
|-------|-------------|
| Turn failed | "Sorry, I couldn't process that. Please try again." |
| Quota exceeded | "This bot has reached its usage limit." |
| Bot not in channel | Ephemeral (Slack) / silent drop |

Never expose internal error codes or stack traces.

---

## Configuration (per tenant)

```json
{
  "channels": {
    "slack": {
      "enabled": true,
      "workspaceId": "T123",
      "botUserId": "U_BOT",
      "policies": {
        "requireMention": true,
        "allowedChannelIds": ["C456"]
      }
    }
  }
}
```

---

## Open Questions

1. Discord adapter — P2, same pattern.
2. SMS (Twilio) — P3.
3. Unified inbox across channels — `02-data-model` future flag.
