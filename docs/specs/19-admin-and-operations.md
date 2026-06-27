# Admin & Operations

**Status:** Draft  
**Priority:** P3  
**Closes gaps:** Architecture gaps §17 (Admin, Operations & Developer Experience)

---

## Purpose

Define the **admin console**, **platform support tooling**, **deployment topology**, **scaling**, **disaster recovery**, and **developer experience** for operating the chatbot platform in production.

**Related docs:**

- [`04-multi-tenancy-and-config.md`](04-multi-tenancy-and-config.md) — tenant config, onboarding
- [`03-auth-and-identity.md`](03-auth-and-identity.md) — RBAC, admin roles
- [`14-guardrails.md`](14-guardrails.md) — rule management UI
- [`15-security-and-compliance.md`](15-security-and-compliance.md) — audit log, impersonation policy
- [`16-observability-and-slos.md`](16-observability-and-slos.md) — dashboards, replay, SLOs
- [`18-testing-and-eval.md`](18-testing-and-eval.md) — CI/CD gates
- [`20-billing-and-metering.md`](20-billing-and-metering.md) — usage views

---

## Scope

### In scope

- Tenant admin console features
- Platform ops console
- Support / impersonation mode
- Environments and deploy units
- CI/CD and migrations
- Horizontal scaling and queues
- Disaster recovery (RPO/RTO)
- Partner developer experience

### Out of scope

- Detailed runbook prose (separate ops repo)
- Kubernetes manifest specifics (implementation)

---

## Console Roles

| Role | Console | Capabilities |
|------|---------|--------------|
| **Tenant owner/admin** | Tenant admin | Config, users, usage, guardrails, MCP, API keys |
| **Tenant member** | Chat app only | No admin console |
| **Platform support** | Platform ops | Cross-tenant read, impersonation with consent |
| **Platform engineer** | Platform ops | Deploy, flags, tenant lifecycle |

---

## Tenant Admin Console

### Navigation

```
Dashboard
├── Overview (usage, health, quota)
├── Conversations (search, support view)
├── Users & roles
├── Configuration
│   ├── General (name, branding, region)
│   ├── Model defaults
│   ├── Embed & channels
│   ├── MCP servers
│   ├── Guardrails
│   ├── Webhooks
│   └── Feature flags (plan-gated)
├── Knowledge bases
├── API keys
├── Audit log
└── Billing (link to usage / plan)
```

### Key features

| Feature | Description |
|---------|-------------|
| **Usage dashboard** | Messages, tokens, tool calls, cost estimate (7 d / 30 d) |
| **Conversation search** | Full-text search by user, date, keyword; read-only transcript |
| **Turn replay** | Inspect/replay turns (`16-observability-and-slos.md`) |
| **Config editor** | Form + JSON view; validate before save; version history |
| **MCP manager** | Register servers, test connection, refresh tools, view schemas |
| **Guardrail manager** | CRUD rules, test panel, appeals queue |
| **User management** | Invite, role assign, deactivate, force logout |
| **API keys** | Create, scope, revoke, rotate |
| **Onboarding checklist** | Progress widget for new tenants |

### Conversation support view

Read-only access to conversation messages for tenant admins resolving user issues.

- PII visible to authorized admins only
- All views audit logged (`15-security-and-compliance.md`)
- Optional end-user consent banner (tenant config): "Support may view this conversation"

---

## Platform Ops Console

### Capabilities

| Feature | Description |
|---------|-------------|
| **Tenant list** | Search, filter by plan/status/region |
| **Tenant detail** | Config, usage, SLO view, recent errors |
| **Tenant lifecycle** | Create, suspend, delete (with grace) |
| **Support impersonation** | Read-only tenant view (see below) |
| **Feature flags** | Global rollout controls |
| **Synthetic monitor status** | Probe results per region |
| **Incident banner** | Status page integration |

Restricted to platform staff with MFA and IP allowlist.

---

## Support / Impersonation Mode

### Flow

```
1. Support ticket references tenantId + userId
2. Platform engineer requests impersonation session
3. Tenant notified (email to owner) OR pre-authorized in enterprise contract
4. Time-boxed read-only session (max 1 h)
5. All actions audit logged with support actor ID
```

### Allowed (read-only)

- View tenant config, conversations, traces, guardrail hits
- Replay turns (inspect mode)
- Export redacted debug bundle for ticket

### Not allowed

- Send messages as user
- Modify config without elevating to tenant admin invite
- View full API keys or secrets (masked)

Enterprise tenants may disable impersonation entirely.

---

## Environments

| Environment | Purpose | Data |
|-------------|---------|------|
| **dev** | Engineer local / shared | Synthetic only |
| **staging** | Pre-prod, load test, eval | Anonymized or synthetic |
| **prod** | Customer traffic | Real |

- Separate AWS/GCP accounts or strong namespace isolation per env
- Test tenants: `ten_test_*` in staging with sandbox MCP
- Prod deploys require staging pass + approval

---

## Deploy Units

Services deployed **independently**:

| Unit | Scales | Notes |
|------|--------|-------|
| Chatbot API | Horizontal | REST + SSE |
| ORCH workers | Horizontal | Queue consumers |
| Channel adapters | Per platform | Slack, Teams, etc. |
| MCP Client | Embedded in ORCH or sidecar | Version with ORCH |
| Admin console | Static + BFF | CDN |
| Chat app | Static + CDN | |
| Embed widget | CDN | |
| Ingestion workers | Horizontal | KB pipeline |
| Migration jobs | One-off | DB schema |

### CI/CD pipeline

```
PR → unit + integration + eval core
  → merge → build container images (signed)
  → deploy staging → smoke + synthetic
  → manual/auto promote prod (canary 10% → 100%)
```

Database migrations: forward-only, backward-compatible expand phase, run before code deploy.

---

## Horizontal Scaling

### Chatbot API

- Stateless behind load balancer
- SSE: sticky sessions optional; prefer reconnect with `Last-Event-ID` over stickiness
- WebSocket (if used): sticky session required

### ORCH workers

- Pull turns from **message queue** (SQS, RabbitMQ, Redis Streams)
- Stateless; no in-memory turn state except active execution
- Paused HITL turns: state in DB; any worker can resume
- Auto-scale on queue depth + CPU

```
Message POST → enqueue TurnJob → worker executes → stream events to Redis/pubsub → Chatbot forwards SSE
```

### Memory / vector / graph

- Managed services with read replicas for retrieval scale
- Write path through ORCH only

### Rate limiting

- Redis token bucket at API gateway per tenant/user

---

## Disaster Recovery

### Targets

| Component | RPO | RTO |
|-----------|-----|-----|
| Conversation DB (Postgres) | 5 min | 1 h |
| Object store | 0 (replicated) | 15 min |
| Vector index | 1 h | 4 h |
| Graph DB | 1 h | 4 h |
| Redis (cache/queue) | Best effort | 15 min (rebuild) |

### Backups

- Postgres: continuous WAL + daily snapshots, 35-day retention
- Cross-region replica for prod (async)
- Object store: versioning + cross-region replication (enterprise)

### DR drill

- Quarterly restore test to isolated env
- Documented failover runbook for regional outage

---

## Feature Flags (platform)

Managed in platform ops console; integrates with `04-multi-tenancy-and-config.md`.

| Flag type | Example |
|-----------|---------|
| Global percentage | `new_context_assembler` 10% |
| Tenant allowlist | `multi_agent` for beta tenants |
| Plan gate | `sso` enterprise only |

Flag state in ORCH turn logs for debugging.

---

## Developer Experience

### Partner documentation

| Asset | Location |
|-------|----------|
| API reference | OpenAPI-generated docs site |
| Embed guide | Quickstart + SDK reference |
| Channel setup | Per-platform guides |
| MCP server guide | Build and register a server |
| Postman collection | Generated from OpenAPI |

### Sandbox

- Test API keys (`cbp_test_...`) → sandbox MCP, lower quotas
- Synthetic tenant auto-provisioned on signup

### Status page

- Public statuspage.io or self-hosted
- Components: API, ORCH, model providers (aggregate), MCP (tenant-specific optional)
- Incident history + subscribe

---

## Admin API Summary

Extends tenant admin endpoints from prior specs:

| Area | Base path |
|------|-----------|
| Config | `/v1/admin/config` |
| Users | `/v1/admin/users` |
| MCP | `/v1/admin/mcp` |
| Guardrails | `/v1/admin/guardrails` |
| Usage | `/v1/admin/usage` |
| Audit | `/v1/admin/audit-log` |
| Conversations | `/v1/admin/conversations` |
| Turn replay | `/v1/admin/turns/{id}/replay` |

Platform-only: `/v1/platform/tenants`, `/v1/platform/impersonate`

---

## Monitoring & On-Call

- PagerDuty rotation for platform SRE
- Runbooks linked from alerts (`16-observability-and-slos.md`)
- Weekly ops review: SLO burn, top errors, cost trend

---

## Open Questions

1. Self-hosted / single-tenant deployment bundle — P3 enterprise offering.
2. In-console live chat support widget — product decision.
3. GitOps for tenant config — P2 enterprise feature.
