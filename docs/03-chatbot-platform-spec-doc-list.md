# Chatbot Platform — Spec Document List

Spec-first approach. All **21 implementation specs** are written (Draft) under [`specs/`](specs/).

**Foundation & planning:**

| # | Document | Status | Purpose |
|---|----------|--------|---------|
| — | [`01-chatbot-platform-architecture.md`](01-chatbot-platform-architecture.md) | Written | High-level system shape, layers, features |
| — | [`02-chatbot-platform-architecture-gaps.md`](02-chatbot-platform-architecture-gaps.md) | Written | What's missing from the architecture doc |
| — | [`04-chatbot-platform-tech-stack.md`](04-chatbot-platform-tech-stack.md) | Written | Technology choices and repo layout |
| — | [`05-chatbot-platform-infra-and-running-cost.md`](05-chatbot-platform-infra-and-running-cost.md) | Written | AWS sizing, monthly costs, unit economics |

---

## Specs

All specs live in [`specs/`](specs/). Status **Draft** unless noted otherwise.

### P0 — Core platform (build first)

| # | Spec | Status | Purpose |
|---|------|--------|---------|
| 01 | [`01-api-and-streaming-events.md`](specs/01-api-and-streaming-events.md) | Draft | Public REST/WS API, SSE event schema, idempotency, webhooks |
| 02 | [`02-data-model-and-persistence.md`](specs/02-data-model-and-persistence.md) | Draft | Entities, schemas, storage tiers, retention |
| 03 | [`03-auth-and-identity.md`](specs/03-auth-and-identity.md) | Draft | Users, tokens, embed JWT exchange, channel identity linking, RBAC |
| 04 | [`04-multi-tenancy-and-config.md`](specs/04-multi-tenancy-and-config.md) | Draft | Tenant isolation, config store, quotas, feature flags, onboarding |

### P1 — Orchestration & intelligence

| # | Spec | Status | Purpose |
|---|------|--------|---------|
| 05 | [`05-orch-turn-engine.md`](specs/05-orch-turn-engine.md) | Draft | Turn loop, tool execution, cancel/concurrency, HITL pause/resume |
| 06 | [`06-model-provider.md`](specs/06-model-provider.md) | Draft | Provider abstraction, routing, fallback, parameters, cost caps |
| 07 | [`07-context-assembler.md`](specs/07-context-assembler.md) | Draft | Token budget, source priority, tool pruning, ContextBundle |
| 08 | [`08-memory.md`](specs/08-memory.md) | Draft | Graph + vector stores, write policy, scopes, retrieval, forget |
| 09 | [`09-knowledge-base-and-ingestion.md`](specs/09-knowledge-base-and-ingestion.md) | Draft | Upload, connectors, chunk/embed pipeline, citations |
| 10 | [`10-mcp-integration.md`](specs/10-mcp-integration.md) | Draft | MCP client, server contract, tools/resources/prompts, auth context |

### P1 — Integration surfaces

| # | Spec | Status | Purpose |
|---|------|--------|---------|
| 11 | [`11-chat-app.md`](specs/11-chat-app.md) | Draft | Standalone app UX, threads, attachments, settings |
| 12 | [`12-embed-widget.md`](specs/12-embed-widget.md) | Draft | Script/iframe SDK, theming, postMessage, host contract |
| 13 | [`13-channel-adapters.md`](specs/13-channel-adapters.md) | Draft | Slack, Teams, WhatsApp, Telegram — adapter pattern & per-platform rules |

### P2 — Safety, ops & quality

| # | Spec | Status | Purpose |
|---|------|--------|---------|
| 14 | [`14-guardrails.md`](specs/14-guardrails.md) | Draft | Checkpoints, rules, versioning, false-positive handling |
| 15 | [`15-security-and-compliance.md`](specs/15-security-and-compliance.md) | Draft | Encryption, residency, audit log, GDPR/delete, retention |
| 16 | [`16-observability-and-slos.md`](specs/16-observability-and-slos.md) | Draft | Traces, metrics, alerts, cost attribution, replay |
| 17 | [`17-attachments-and-media.md`](specs/17-attachments-and-media.md) | Draft | Upload storage, extraction, multimodal, channel limits |
| 18 | [`18-testing-and-eval.md`](specs/18-testing-and-eval.md) | Draft | Unit/integration, eval harness, load/chaos, guardrail regression |

### P3 — Platform operations & future

| # | Spec | Status | Purpose |
|---|------|--------|---------|
| 19 | [`19-admin-and-operations.md`](specs/19-admin-and-operations.md) | Draft | Admin console, support mode, deploy, DR, scaling |
| 20 | [`20-billing-and-metering.md`](specs/20-billing-and-metering.md) | Draft | Billable units, plans, quota enforcement |
| 21 | [`21-multi-agent.md`](specs/21-multi-agent.md) | Draft (future) | Supervisor model, delegation, budgets, failure handling — not v1 |

---

## Suggested implementation order

```
01 → 02 → 03 → 04 → 05 → 06 → 07 → 08 → 10 → 11/12/13 → 09 → 14 → 15 → 16 → 17 → 18 → 19 → 20 → 21
```

MCP integration (10) before surface specs if tools drive UX. KB ingestion (09) can follow memory (08). Multi-agent (21) is design-only until v2.

---

## Next steps

- [ ] Review and promote specs from **Draft** → **Approved**
- [ ] Generate `api/openapi.yaml` from spec 01
- [ ] Begin P0 implementation per [`04-chatbot-platform-tech-stack.md`](04-chatbot-platform-tech-stack.md)
