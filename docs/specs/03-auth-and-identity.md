# Auth & Identity

**Status:** Draft  
**Priority:** P0  
**Closes gaps:** Architecture gaps §3 (Authentication & Identity)

---

## Purpose

Define how **users are identified**, **authenticated**, and **authorized** across all surfaces: chat app, embed widget, messaging channels, and service-to-service calls. Auth is the foundation for tenant isolation, tool scoping, and embed security.

**Related docs:**

- [`01-api-and-streaming-events.md`](01-api-and-streaming-events.md) — `Authorization` header usage
- [`02-data-model-and-persistence.md`](02-data-model-and-persistence.md) — `User` entity, `externalIds`
- [`04-multi-tenancy-and-config.md`](04-multi-tenancy-and-config.md) — tenant onboarding, API keys
- [`10-mcp-integration.md`](10-mcp-integration.md) — auth context passed to MCP servers

---

## Scope

### In scope

- Identity model (platform user, external user, guest)
- Token types and issuance
- Embed JWT exchange flow
- Channel identity linking (Slack, Teams, WhatsApp, Telegram)
- RBAC roles and permission scopes
- Service-to-service authentication
- Token security (rotation, revocation, origin binding)

### Out of scope

- SSO provider configuration UI — see `19-admin-and-operations.md`
- Channel OAuth install flows (step-by-step) — see `13-channel-adapters.md`
- Encryption at rest — see `15-security-and-compliance.md`

---

## Identity Model

### Platform User

Canonical identity within a tenant. Stored as `User` (`usr_...`). Attributes:

- Unique per `(tenantId, email)` for registered users
- `externalIds` map for linked channel identities
- `role` for RBAC
- `preferences` for user-level settings

### External User

An identity in a host application or messaging platform, not natively registered on the platform. Linked via:

- Embed: `externalUserId` in JWT claims → stored in `User.externalIds.host_app`
- Channels: platform-specific ID → stored in `User.externalIds.{slack|teams|whatsapp|telegram}`

**Resolution rule:** on first authenticated request with an external ID, create or match a `User`:

1. Look up `(tenantId, externalIds.{provider}, value)`.
2. If found → use existing `usr_`.
3. If not found → create `User` with `status: active` (or `guest` for anonymous embed).

### Guest / Anonymous

Embed and public widgets may allow unauthenticated use:

- Issue a `usr_` with `status: guest` and `expiresAt` (default 24 h).
- Guest users have restricted tool scopes and no cross-session memory persistence (tenant config).
- Upgrade path: host re-authenticates with signed JWT containing `externalUserId` → merge guest conversation into linked user (optional, tenant config).

---

## Token Types

| Token | Issuer | TTL | Use |
|-------|--------|-----|-----|
| **Session token** | Platform | 1 h (refresh: 30 d) | Chat app browser sessions |
| **API key** | Platform (tenant admin) | Until revoked | Server-to-server, CI, channel adapters |
| **Embed token** | Platform (exchanged from host JWT) | 15 min | Widget script/iframe |
| **Channel token** | Platform (post-OAuth install) | Until uninstall | Adapter → Chatbot API |
| **Service token** | Platform (internal) | 5 min | ORCH → MCP Client, internal services |

All tokens are JWTs (or opaque tokens resolved server-side) with standard claims:

```json
{
  "sub": "usr_01H...",
  "tenantId": "ten_01H...",
  "type": "session",
  "scopes": ["conversations:read", "conversations:write", "tools:billing:read"],
  "role": "member",
  "iat": 1719392400,
  "exp": 1719396000,
  "jti": "tok_01H..."
}
```

---

## Authentication Flows

### Chat app — email/password or SSO

```
User → IdP (SSO) or login form
     → Platform auth service
     → session token + refresh token (httpOnly cookie or response body)
     → Chat app stores token, sends Authorization: Bearer on API calls
```

- SSO: OIDC/SAML per tenant configuration.
- Refresh: `POST /v1/auth/refresh` with refresh token → new session token.
- Logout: `POST /v1/auth/logout` → revoke `jti`, clear refresh token.

### API key

```
Tenant admin creates key in console
  → key id + secret shown once
  → requests use: Authorization: Bearer <api_key_secret>
     or: X-Api-Key: <api_key_secret>
```

- API keys are scoped (read-only, full, custom scope list).
- Keys are hashed at rest; only prefix shown in admin UI (`cbp_live_abc...`).
- Rate limits apply per key (`04-multi-tenancy-and-config.md`).

### Embed — JWT exchange

Host backend holds a **tenant embed secret** (or signs with private key; platform holds JWKS). Widget never sees the host session cookie.

```
Host backend                          Platform                     Widget
     │                                    │                          │
     │  POST /v1/auth/embed/exchange      │                          │
     │  { hostJwt: "..." }                │                          │
     │ ─────────────────────────────────► │                          │
     │                                    │ verify signature, claims │
     │  { embedToken, expiresIn }       │                          │
     │ ◄───────────────────────────────── │                          │
     │                                    │                          │
     │  pass embedToken to widget init ─────────────────────────────►│
     │                                    │                          │
     │                                    │◄── API calls with token ─│
```

**Host JWT required claims:**

```json
{
  "iss": "https://host.example.com",
  "aud": "chatbot-platform",
  "sub": "host_user_42",
  "tenantId": "ten_01H...",
  "iat": 1719392400,
  "exp": 1719392700
}
```

**Optional claims:**

```json
{
  "name": "Jane Doe",
  "email": "jane@host.com",
  "scopes": ["tools:billing:read"],
  "context": {
    "pageUrl": "https://host.example.com/invoices/123",
    "entityIds": { "invoiceId": "inv_123" }
  }
}
```

**Verification:**

- Shared secret: HMAC-SHA256 with tenant embed secret.
- Asymmetric: RS256/ES256; platform fetches host JWKS from configured URL.
- `aud` must match `chatbot-platform`.
- `tenantId` in JWT must match the tenant making the exchange (or be derived from API key).
- `exp - iat` ≤ 5 minutes for host JWT.

**Embed token additional claims:**

```json
{
  "type": "embed",
  "origin": "https://host.example.com",
  "context": { ... }
}
```

`origin` is bound to the request `Origin` header on every widget API call. Mismatch → `403 origin_mismatch`.

### Channel adapters

After platform OAuth install (per `13-channel-adapters.md`):

1. Adapter receives platform-issued **channel token** scoped to `(tenantId, channel, workspaceId)`.
2. Inbound webhook: adapter resolves platform user via identity linking (below).
3. Outbound: adapter uses channel token to call Chatbot API or internal reply endpoint.

---

## Channel Identity Linking

Map external platform IDs to `usr_...`.

### Link table (logical)

```
channel_identities
  tenantId
  provider        -- slack | teams | whatsapp | telegram
  externalId      -- U123, AAD object id, phone number
  userId          -- usr_...
  workspaceId     -- Slack team id, Teams tenant id (nullable)
  linkedAt
  metadata        -- display name, avatar url
```

### Resolution on inbound message

```
Inbound Slack message (user U123, team T456)
  → lookup (ten_..., slack, U123, T456)
  → if miss: create User + channel_identity row
  → proceed with usr_...
```

### Account linking (optional)

Tenant may allow users to link multiple channels to one account:

- `POST /v1/auth/link/{provider}` — OAuth flow
- Verified email match auto-merges (tenant config)

---

## Authorization (RBAC)

### Roles

| Role | Description |
|------|-------------|
| `owner` | Full tenant control, billing, delete tenant |
| `admin` | Manage users, config, guardrails, MCP servers |
| `member` | Use chatbot, manage own conversations and memory |
| `viewer` | Read-only access to shared resources (future) |
| `service` | Machine principal for adapters and integrations |

Roles are assigned per `(tenantId, userId)`.

### Permission scopes

Fine-grained scopes for API keys, embed tokens, and tool access:

| Scope | Grants |
|-------|--------|
| `conversations:read` | List/get own conversations |
| `conversations:write` | Send messages, create conversations |
| `memory:read` | List own memory |
| `memory:write` | Create/delete memory |
| `tools:{server}:{tool}` | Invoke specific MCP tool |
| `tools:{server}:*` | All tools on a server |
| `admin:config` | Tenant configuration |
| `admin:users` | User management |

**Enforcement points:**

| Layer | Checks |
|-------|--------|
| Chatbot API | Token valid, scopes match endpoint |
| ORCH | Tool allowlist ∩ token scopes ∩ tenant MCP registry |
| MCP Server | Receives `authContext`; enforces domain authZ |

### Conversation access

Default: users access only their own conversations (`conversation.userId == token.sub`).

Tenant config `shared_conversations` (future): team-visible threads with explicit ACL.

---

## Service-to-Service Auth

### ORCH → MCP Client

Internal service token (short-lived, mTLS optional). Claims include `tenantId`, `userId`, `scopes` from the originating turn.

### MCP Client → MCP Server

Per-server credential strategy (configured in tenant MCP registry):

| Method | When |
|--------|------|
| Forwarded auth context | Server trusts platform-signed JWT in MCP metadata |
| Stored credential | Client attaches API key / OAuth token from vault |
| mTLS | Server requires client certificate |

Auth context passed on every tool call:

```json
{
  "tenantId": "ten_01H...",
  "userId": "usr_01H...",
  "scopes": ["tools:billing:read"],
  "externalUserId": "host_user_42",
  "channel": "embed"
}
```

Secrets never enter model prompts or `ContextBundle` dumps.

---

## Token Security

### Rotation & revocation

| Event | Action |
|-------|--------|
| Logout | Revoke session `jti` (blocklist in Redis, TTL = remaining exp) |
| Password change / SSO session end | Revoke all session tokens for user |
| API key rotate | Old key grace period 24 h, then revoked |
| Compromised embed secret | Tenant admin rotates; old host JWTs rejected immediately |
| User deleted | Revoke all tokens for `usr_` |

### Embed-specific protections

- **Origin binding:** embed token `origin` claim must match request `Origin`.
- **Domain allowlist:** tenant config lists allowed embed origins; exchange endpoint rejects others.
- **Short TTL:** embed tokens expire in 15 min; widget refreshes via host callback (`12-embed-widget.md`).
- **No secrets in widget:** host JWT is obtained server-side only.

### Session fixation

- Platform always issues a new `jti` on login/exchange; never accept client-supplied session IDs.
- Refresh tokens are one-time use (rotation on refresh).

---

## Auth API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/v1/auth/login` | Email/password login |
| `POST` | `/v1/auth/refresh` | Refresh session token |
| `POST` | `/v1/auth/logout` | Revoke session |
| `POST` | `/v1/auth/embed/exchange` | Host JWT → embed token |
| `GET` | `/v1/auth/me` | Current user profile |
| `POST` | `/v1/auth/link/{provider}` | Start channel OAuth link |

---

## Audit

All auth events are logged to the immutable audit log (`15-security-and-compliance.md`):

- `auth.login.success`, `auth.login.failure`
- `auth.token.issued`, `auth.token.revoked`
- `auth.embed.exchange`
- `auth.identity.linked`

Logs include `tenantId`, `userId`, `ip`, `userAgent`; never include token values or secrets.

---

## Open Questions

1. Passkeys / WebAuthn for chat app — P2 enhancement.
2. Fine-grained ABAC (attribute-based) beyond scopes — defer until enterprise requirement.
3. Cross-tenant service accounts for managed deployments — P3.
