# Chatbot Platform — Spec Document List

Spec-first approach. Individual specs to be written later; this is the index only.

**Existing (foundation):**

| # | Document | Purpose |
|---|----------|---------|
| — | `01-chatbot-platform-architecture.md` | High-level system shape, layers, features |
| — | `02-chatbot-platform-architecture-gaps.md` | What's missing from the architecture doc |

---

## Specs to Create

### P0 — Core platform (build first)

| # | Spec | Purpose |
|---|------|---------|
| 01 | `01-api-and-streaming-events.md` | Public REST/WS API, SSE event schema, idempotency, webhooks |
| 02 | `02-data-model-and-persistence.md` | Entities, schemas, storage tiers, retention |
| 03 | `03-auth-and-identity.md` | Users, tokens, embed JWT exchange, channel identity linking, RBAC |
| 04 | `04-multi-tenancy-and-config.md` | Tenant isolation, config store, quotas, feature flags, onboarding |

### P1 — Orchestration & intelligence

| # | Spec | Purpose |
|---|------|---------|
| 05 | `05-orch-turn-engine.md` | Turn loop, tool execution, cancel/concurrency, HITL pause/resume |
| 06 | `06-model-provider.md` | Provider abstraction, routing, fallback, parameters, cost caps |
| 07 | `07-context-assembler.md` | Token budget, source priority, tool pruning, ContextBundle |
| 08 | `08-memory.md` | Graph + vector stores, write policy, scopes, retrieval, forget |
| 09 | `09-knowledge-base-and-ingestion.md` | Upload, connectors, chunk/embed pipeline, citations |
| 10 | `10-mcp-integration.md` | MCP client, server contract, tools/resources/prompts, auth context |

### P1 — Integration surfaces

| # | Spec | Purpose |
|---|------|---------|
| 11 | `11-chat-app.md` | Standalone app UX, threads, attachments, settings |
| 12 | `12-embed-widget.md` | Script/iframe SDK, theming, postMessage, host contract |
| 13 | `13-channel-adapters.md` | Slack, Teams, WhatsApp, Telegram — adapter pattern & per-platform rules |

### P2 — Safety, ops & quality

| # | Spec | Purpose |
|---|------|---------|
| 14 | `14-guardrails.md` | Checkpoints, rules, versioning, false-positive handling |
| 15 | `15-security-and-compliance.md` | Encryption, residency, audit log, GDPR/delete, retention |
| 16 | `16-observability-and-slos.md` | Traces, metrics, alerts, cost attribution, replay |
| 17 | `17-attachments-and-media.md` | Upload storage, extraction, multimodal, channel limits |
| 18 | `18-testing-and-eval.md` | Unit/integration, eval harness, load/chaos, guardrail regression |

### P3 — Platform operations & future

| # | Spec | Purpose |
|---|------|---------|
| 19 | `19-admin-and-operations.md` | Admin console, support mode, deploy, DR, scaling |
| 20 | `20-billing-and-metering.md` | Billable units, plans, quota enforcement |
| 21 | `21-multi-agent.md` | Supervisor model, delegation, budgets, failure handling (future) |

---

## Suggested write order

```
01 → 02 → 03 → 04 → 05 → 06 → 07 → 08 → 10 → 11/12/13 → 09 → 14 → 15 → 16 → 17 → 18 → 19 → 20 → 21
```

MCP integration (10) before surface specs if tools drive UX. KB ingestion (09) can follow memory (08).
