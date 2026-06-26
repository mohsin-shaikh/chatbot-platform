# Multi-Tenancy & Configuration

**Status:** Draft  
**Priority:** P0  
**Closes gaps:** Architecture gaps §4 (Multi-Tenancy & Configuration)

---

## Purpose

Define **tenant isolation**, the **configuration store**, **MCP registry**, **quotas**, **feature flags**, and **onboarding** so the platform can serve many customers safely on shared infrastructure.

**Related docs:**

- [`02-data-model-and-persistence.md`](02-data-model-and-persistence.md) — `Tenant` entity, row-level isolation
- [`03-auth-and-identity.md`](03-auth-and-identity.md) — tenant-scoped auth, API keys
- [`06-model-provider.md`](06-model-provider.md) — model defaults per tenant
- [`10-mcp-integration.md`](10-mcp-integration.md) — MCP server registration detail
- [`14-guardrails.md`](14-guardrails.md) — guardrail rule sets
- [`20-billing-and-metering.md`](20-billing-and-metering.md) — plans and billing

---

## Scope

### In scope

- Tenant lifecycle and isolation strategy
- Configuration schema and resolution
- Per-tenant MCP tool registry
- Quotas, entitlements, enforcement
- Feature flags
- Tenant onboarding flow

### Out of scope

- Admin console UI — see `19-admin-and-operations.md`
- Billing integration (Stripe) — see `20-billing-and-metering.md`
- Infrastructure provisioning (K8s, DB instances) — ops runbooks

---

## Tenant Isolation

### Strategy

**Row-level isolation** with `tenantId` on every tenant-scoped table and index. Single shared database per environment (v1).

| Approach | v1 | Future option |
|----------|-----|---------------|
| Data partition | `tenantId` column + RLS policies | Schema-per-tenant for enterprise |
| Compute | Shared ORCH workers; tenant in request context | Dedicated worker pools per tier |
| Memory indexes | Namespace prefix `ten_{id}` in graph + vector | Dedicated indexes for enterprise |
| Object storage | Prefix `s3://bucket/{tenantId}/` | Separate buckets per region/tier |

### Noisy-neighbor controls

| Control | Mechanism |
|---------|-----------|
| Rate limits | Per-tenant requests/min, messages/day |
| Concurrent turns | Max in-flight turns per tenant |
| Token budget | Monthly token cap; hard stop or overage billing |
| MCP fan-out | Max parallel tool calls per turn |
| Retrieval | Max vector queries per turn |

Exceeded limits return `429` with `quota_exceeded` and `retryAfter` where applicable.

### Region pinning

Each tenant has a `region` (e.g. `us-east-1`, `eu-west-1`). All data stores and model routing stay in-region. Cross-region requests rejected at API gateway.

---

## Configuration Store

Tenant configuration is a versioned JSON document. Changes are audited; effective config is cached (TTL 60 s) with invalidation on update.

### Config document schema

```json
{
  "tenantId": "ten_01H...",
  "version": 12,

  "branding": {
    "name": "Acme Assistant",
    "logoUrl": "https://cdn.example.com/acme/logo.png",
    "primaryColor": "#0066CC",
    "widgetPosition": "bottom-right"
  },

  "systemPrompt": {
    "template": "You are {{branding.name}}, a helpful assistant for {{tenant.name}}...",
    "variables": {}
  },

  "model": {
    "default": "gpt-4.1",
    "fallback": "gpt-4.1-mini",
    "parameters": {
      "temperature": 0.7,
      "maxOutputTokens": 4096
    },
    "perChannel": {
      "slack": { "default": "gpt-4.1-mini" }
    }
  },

  "guardrails": {
    "ruleSetId": "grs_01H...",
    "inputEnabled": true,
    "outputEnabled": true,
    "preToolEnabled": true
  },

  "mcp": {
    "servers": [ ... ]
  },

  "embed": {
    "allowedOrigins": ["https://app.acme.com", "https://*.acme.com"],
    "authMode": "jwt_exchange",
    "guestAllowed": false
  },

  "channels": {
    "slack": { "enabled": true, "workspaceId": "T123" },
    "teams": { "enabled": false }
  },

  "memory": {
    "extractionEnabled": true,
    "guestPersistence": false,
    "scopes": ["user"]
  },

  "conversation": {
    "concurrencyPolicy": "queue",
    "retentionDays": null,
    "unifiedInbox": false
  },

  "features": {
    "attachments": true,
    "hitlApprovals": true,
    "knowledgeBase": true,
    "multiAgent": false
  },

  "webhooks": [ ... ],

  "quotas": {
    "messagesPerDay": 10000,
    "tokensPerMonth": 5000000,
    "maxConcurrentTurns": 50,
    "maxStorageBytes": 10737418240
  }
}
```

### Resolution order

When ORCH or Chatbot needs a config value:

1. **Turn-level override** (if allowed, e.g. model override in API request metadata)
2. **Channel override** (`model.perChannel.slack`)
3. **Tenant config** (document above)
4. **Plan defaults** (from `20-billing-and-metering.md`)
5. **Platform defaults**

### Versioning

- Each update increments `version`.
- ORCH logs `configVersion` on each turn for replay/debugging.
- Rollback: admin restores prior version from history (last 50 versions retained).

---

## MCP Registry (per tenant)

Each tenant configures which MCP servers and tools are available.

```json
{
  "mcp": {
    "servers": [
      {
        "id": "mcp_billing",
        "name": "Billing",
        "transport": "streamable_http",
        "url": "https://mcp-billing.internal/acme",
        "auth": {
          "type": "forwarded_jwt"
        },
        "enabled": true,
        "environment": "production",
        "toolFilter": {
          "mode": "allowlist",
          "tools": ["get_invoice", "list_invoices"]
        },
        "timeoutMs": 30000,
        "retryPolicy": { "maxAttempts": 2, "backoffMs": 1000 }
      },
      {
        "id": "mcp_crm",
        "name": "CRM",
        "transport": "stdio",
        "command": "/opt/mcp/crm-server",
        "enabled": true,
        "environment": "production",
        "toolFilter": { "mode": "all" }
      }
    ],
    "sandbox": {
      "enabled": true,
      "serverId": "mcp_mock",
      "appliesTo": ["api_key_prefix:cbp_test_"]
    }
  }
}
```

| Field | Description |
|-------|-------------|
| `toolFilter.mode` | `all` \| `allowlist` \| `blocklist` |
| `environment` | `production` \| `sandbox` — sandbox servers for test API keys |
| `sandbox` | Route test keys to mock MCP server |

Tool discovery results are cached per `(tenantId, serverId)` with TTL 5 min. Admin "refresh tools" invalidates cache.

---

## Quotas & Entitlements

### Plan tiers (defaults)

| Quota | Free | Pro | Enterprise |
|-------|------|-----|------------|
| Messages/day | 100 | 10,000 | Custom |
| Tokens/month | 100K | 5M | Custom |
| Max users | 5 | 100 | Custom |
| MCP servers | 1 | 10 | Unlimited |
| Storage | 1 GB | 10 GB | Custom |
| Concurrent turns | 3 | 50 | Custom |
| Regions | 1 | 1 | Multi |

Plan sets **ceiling**; tenant `quotas` in config may be lower but not higher without plan upgrade.

### Enforcement

```
Request arrives
  → load tenant config + plan
  → check rate limiter (Redis): messages/min, concurrent turns
  → check usage meter (monthly tokens, storage)
  → if exceeded:
       hard block (429) OR soft warn (tenant config, log + notify admin)
  → proceed
```

Usage counters: real-time approximate (Redis) + durable rollups (hourly) for billing.

### Entitlement flags

Derived from plan + `features` config:

- `knowledgeBase`, `hitlApprovals`, `attachments`, `multiAgent`, `sso`, `auditLogExport`

Disabled features return `403 feature_not_enabled` on relevant endpoints.

---

## Feature Flags

### Platform flags (global)

Managed by platform ops. Examples: `new_context_assembler`, `model_routing_v2`.

Rollout: percentage → allowlist tenants → 100%.

### Tenant flags

In tenant `features` config. Tenant admin toggles where permitted by plan.

### Evaluation

```
effective = platform_flag AND (tenant_flag if tenant-scoped) AND plan_entitlement
```

ORCH and Chatbot check flags at runtime; flag state logged on each turn.

---

## Onboarding Flow

### Self-serve (Free / Pro)

```
1. Sign up (email or SSO)
   → create Tenant (status: provisioning)
   → create User (role: owner)

2. Provision defaults
   → seed config (platform default system prompt, model, quotas from plan)
   → create sandbox API key (cbp_test_...)
   → assign region from signup geo or selection

3. Wizard steps (console)
   a. Branding (name, logo, color)
   b. Embed origins OR connect Slack
   c. Register first MCP server (optional)
   d. Copy API key / embed snippet

4. Tenant status → active
```

### Enterprise / sales-led

- Tenant created via admin API with custom plan, region, and SLA.
- SSO pre-configured before invite.

### API: create tenant (admin only)

`POST /v1/admin/tenants` — see `19-admin-and-operations.md`.

### Post-onboarding checklist (automated)

| Check | Pass criteria |
|-------|---------------|
| API key issued | ≥ 1 active key |
| First conversation | ≥ 1 message sent |
| Embed configured | `allowedOrigins` non-empty OR channel connected |
| MCP connected | ≥ 1 server enabled (optional) |

---

## Tenant Lifecycle

| Status | Meaning |
|--------|---------|
| `provisioning` | Being set up |
| `active` | Normal operation |
| `suspended` | Billing issue or ToS violation; read-only |
| `deleting` | Grace period before purge |
| `deleted` | Hard delete scheduled complete |

**Suspend:** API returns `403 tenant_suspended`; existing streams complete.

**Delete:** 30-day grace (`15-security-and-compliance.md`); then cascade delete all tenant data.

---

## Config API (tenant admin)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/v1/admin/config` | Get effective tenant config |
| `PATCH` | `/v1/admin/config` | Partial update (validated against schema) |
| `GET` | `/v1/admin/config/history` | List config versions |
| `POST` | `/v1/admin/config/rollback` | Rollback to version N |
| `GET` | `/v1/admin/usage` | Current quota consumption |
| `GET` | `/v1/admin/mcp/servers` | List MCP servers |
| `POST` | `/v1/admin/mcp/servers` | Register MCP server |
| `POST` | `/v1/admin/mcp/servers/{id}/refresh` | Refresh tool discovery |

Requires `admin:config` scope or `admin`/`owner` role.

---

## Caching & Consistency

| Cache | Key | TTL | Invalidation |
|-------|-----|-----|--------------|
| Tenant config | `config:{tenantId}` | 60 s | On PATCH, webhook to workers |
| MCP tool list | `mcp_tools:{tenantId}:{serverId}` | 5 min | Manual refresh, server register |
| Quota counters | `quota:{tenantId}:{metric}` | Rolling window | N/A |
| Feature flags | `flags:global`, `flags:{tenantId}` | 30 s | Flag service push |

ORCH must tolerate up to 60 s stale config; critical security changes (embed origin revoke) invalidate immediately via pub/sub.

---

## Open Questions

1. Schema-per-tenant for enterprise — trigger: contractual or >10M rows per tenant.
2. Tenant-level custom domains for chat app (`acme.chatbot.example.com`) — P2.
3. Config-as-code (Git sync for guardrails/MCP) — P2, enterprise feature.
