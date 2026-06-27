# Embed Widget

**Status:** Draft  
**Priority:** P1  
**Closes gaps:** Architecture gaps §16 (Embed)

---

## Purpose

Define the **embeddable chat widget**: script-tag and iframe deployment, JavaScript SDK, theming, `postMessage` protocol, and host application contract. Enables third-party sites to embed the chatbot securely.

**Related docs:**

- [`01-api-and-streaming-events.md`](01-api-and-streaming-events.md) — API, SSE events
- [`03-auth-and-identity.md`](03-auth-and-identity.md) — embed JWT exchange, origin binding
- [`04-multi-tenancy-and-config.md`](04-multi-tenancy-and-config.md) — `embed.allowedOrigins`, branding
- [`11-chat-app.md`](11-chat-app.md) — shared message rendering patterns

---

## Scope

### In scope

- Script widget and iframe modes
- SDK API (`init`, `open`, `close`, etc.)
- Host ↔ widget `postMessage` protocol
- Theming and branding
- Token lifecycle in widget
- CSP, security, domain allowlist

### Out of scope

- Host backend JWT signing implementation (document contract only)
- WordPress/Shopify plugins — sample integrations only

---

## Deployment Modes

### Script widget (recommended)

```html
<script
  src="https://cdn.chatbot.example.com/widget/v1/loader.js"
  data-tenant-id="ten_01H..."
  async
></script>
```

Loader mounts floating chat bubble + panel on `document.body`.

### Iframe embed

```html
<iframe
  src="https://embed.chatbot.example.com/v1?tenant=ten_01H..."
  sandbox="allow-scripts allow-same-origin allow-popups"
  style="width:400px;height:600px;border:none;"
></iframe>
```

Stricter isolation for untrusted hosts. Parent communicates via `postMessage` only.

| Mode | Pros | Cons |
|------|------|------|
| Script | Richer UX, shared DOM theming | Requires script CSP allowance |
| Iframe | Strong sandbox, CSP-friendly | Less flexible layout integration |

---

## SDK API

### Initialization

```javascript
const chatbot = ChatbotWidget.init({
  tenantId: 'ten_01H...',
  token: embedToken,           // from host backend exchange
  container: '#chat-root',     // optional; default floating
  mode: 'bubble',              // bubble | inline | fullscreen
  context: {
    pageUrl: window.location.href,
    entityIds: { orderId: 'ord_123' }
  },
  theme: {
    primaryColor: '#0066CC',
    position: 'bottom-right',
    zIndex: 9999
  },
  onReady: () => {},
  onError: (err) => {},
  onEvent: (event) => {}        // mirrors SSE events
});
```

### Methods

| Method | Description |
|--------|-------------|
| `init(config)` | Initialize widget; idempotent |
| `open()` | Open chat panel |
| `close()` | Close panel |
| `toggle()` | Toggle open/close |
| `setContext(ctx)` | Update host context mid-session |
| `setToken(token)` | Refresh embed token before expiry |
| `destroy()` | Remove DOM, close connections, clear listeners |
| `sendMessage(text)` | Programmatic send (optional) |

### Events (callback / `postMessage`)

| Event | Payload |
|-------|---------|
| `ready` | `{ version }` |
| `open` | `{}` |
| `close` | `{}` |
| `message.sent` | `{ messageId }` |
| `message.received` | `{ messageId, preview }` |
| `turn.complete` | `{ turnId, usage }` |
| `approval.required` | `{ approvalId, prompt }` |
| `error` | `{ code, message }` |

---

## Authentication Flow

Widget **never** holds host session cookies.

```
Host backend                          Platform
     │                                    │
     │  POST /v1/auth/embed/exchange      │
     │  { hostJwt }                       │
     │ ─────────────────────────────────► │
     │  { embedToken, expiresIn: 900 }  │
     │ ◄───────────────────────────────── │
     │                                    │
Host passes embedToken to ChatbotWidget.init()
Widget sends Authorization: Bearer on all API calls
```

### Token refresh

- Widget tracks `expiresAt`; refresh at T-2 min.
- Host registers refresh callback:

```javascript
ChatbotWidget.init({
  getToken: async () => {
    const res = await fetch('/api/chatbot-token');
    return res.json().then(d => d.embedToken);
  }
});
```

- On 401: call `getToken` once and retry; else `onError`.

---

## postMessage Protocol (iframe)

### Origin validation

- Widget accepts messages only from `parentOrigin` in config.
- Parent must verify `event.origin === 'https://embed.chatbot.example.com'`.

### Parent → iframe

```javascript
iframe.contentWindow.postMessage({
  type: 'chatbot:init',
  token: embedToken,
  context: { pageUrl: '...' },
  theme: { primaryColor: '#0066CC' }
}, 'https://embed.chatbot.example.com');
```

| type | Description |
|------|-------------|
| `chatbot:init` | Initial config |
| `chatbot:setContext` | Update context |
| `chatbot:setToken` | Token refresh |
| `chatbot:open` | Open panel |
| `chatbot:close` | Close panel |
| `chatbot:destroy` | Tear down |

### iframe → parent

```javascript
{ type: 'chatbot:ready', version: '1.0.0' }
{ type: 'chatbot:event', event: 'turn.complete', data: { ... } }
{ type: 'chatbot:resize', height: 520 }
{ type: 'chatbot:approval.required', data: { approvalId, prompt } }
```

Host may surface approval in native UI and call resolve via API.

---

## Theming

### CSS variables (script mode)

```css
:root {
  --cb-primary: #0066CC;
  --cb-primary-hover: #0052A3;
  --cb-bg: #ffffff;
  --cb-text: #1a1a1a;
  --cb-radius: 12px;
  --cb-font-family: system-ui, sans-serif;
  --cb-z-index: 9999;
}
```

### Tenant branding

From tenant config: logo, name, colors. Widget fetches on init (`GET /v1/embed/config?tenantId=...`) — public endpoint, returns only non-sensitive branding.

### Dark mode

- `theme.mode: 'light' | 'dark' | 'auto'`
- `auto` follows `prefers-color-scheme`

### Layout

| `mode` | Behavior |
|--------|----------|
| `bubble` | Fixed FAB + slide-up panel (default) |
| `inline` | Renders inside `container` element |
| `fullscreen` | Mobile-friendly full viewport |

---

## Security

| Control | Implementation |
|---------|----------------|
| Domain allowlist | `embed.allowedOrigins` in tenant config; enforced at exchange + API |
| Origin binding | Embed token `origin` claim vs request `Origin` header |
| CSP (host) | `script-src https://cdn.chatbot.example.com`; `connect-src` API host |
| CSP (iframe) | Platform sets strict CSP on embed page |
| Cookie-less | Widget uses bearer token only; no third-party cookies |
| Safari ITP | No reliance on cross-site cookies |
| XSS | Sanitize all rendered markdown; no `eval` in SDK |

### License / tenant key

- `data-tenant-id` is public (like Stripe publishable key).
- Sensitive operations require valid embed token from exchange.
- Optional `data-site-key` for additional domain verification.

---

## Feature Parity

| Feature | Script | Iframe |
|---------|--------|--------|
| Streaming messages | Yes | Yes |
| File upload | Yes | Yes (within sandbox) |
| HITL approval | Inline or host native | Host via postMessage |
| Conversation history | Session + user linked | Same |
| Guest mode | If tenant allows | If tenant allows |

---

## CDN & Loading

```
loader.js (~8KB gzipped)
  → async load widget.js + styles
  → init on DOMContentLoaded or manual
```

- SRI hash optional for security-conscious hosts.
- Versioned path: `/widget/v1/` for cache busting on major releases.

---

## Host Integration Checklist

1. Register allowed origins in tenant admin
2. Implement server-side JWT signing (≤ 5 min TTL)
3. Implement token exchange endpoint for frontend
4. Add loader script or iframe snippet
5. Handle `approval.required` webhook or postMessage (if using destructive tools)
6. Test CSP in staging

---

## Open Questions

1. Custom launcher icon upload — tenant branding P2.
2. Co-browsing / screen share — out of scope.
3. Widget inside Shadow DOM for CSS isolation — P2 script mode option.
