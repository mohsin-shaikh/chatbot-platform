# Chatbot Platform

A multi-tenant chatbot platform for building, deploying, and operating AI assistants across web, embed, and messaging channels. The orchestrator runs tool calls via [MCP](https://modelcontextprotocol.io), maintains graph + vector memory, and enforces guardrails with full observability.

**Status:** Spec-first — architecture and **21 implementation specs** are written; application code not yet started.

---

## What it does

- **Deploy anywhere** — standalone chat app, embeddable widget, Slack, Teams, WhatsApp, Telegram
- **Orchestrate turns** — model calls, parallel MCP tool execution, human-in-the-loop approvals, cancellation
- **Remember context** — graph memory (preferences, relationships) + vector memory (semantic recall) + knowledge base (RAG)
- **Stay safe** — input/output/pre-tool guardrails, tenant isolation, audit logging, GDPR delete flows
- **Operate at scale** — multi-tenancy, quotas, billing, admin console, SLOs and turn replay

```
Integration Surfaces (Chat App · Embed · Slack/Teams/…)
        │
        ▼
   Chatbot API (REST + SSE)
        │
        ▼
   Orchestrator (ORCH)
        ├── Context assembler · Memory · Guardrails
        ├── Model providers (OpenAI, Anthropic, Azure)
        └── MCP Client → MCP Servers
```

See [`docs/01-chatbot-platform-architecture.md`](docs/01-chatbot-platform-architecture.md) for the full architecture.

---

## Documentation

| Doc | Description |
|-----|-------------|
| [Architecture](docs/01-chatbot-platform-architecture.md) | System shape, turn lifecycle, features |
| [Architecture gaps](docs/02-chatbot-platform-architecture-gaps.md) | Gap analysis (companion to architecture) |
| [**Spec index**](docs/03-chatbot-platform-spec-doc-list.md) | Index of all 21 specs + implementation order |
| [Tech stack](docs/04-chatbot-platform-tech-stack.md) | TypeScript monorepo, AWS, Postgres, Redis, Neo4j |
| [Infra & cost](docs/05-chatbot-platform-infra-and-running-cost.md) | AWS sizing and monthly cost estimates |

### Specs (`docs/specs/`)

| Priority | Specs | Focus |
|----------|-------|-------|
| **P0** | [01](docs/specs/01-api-and-streaming-events.md)–[04](docs/specs/04-multi-tenancy-and-config.md) | API, data model, auth, multi-tenancy |
| **P1** | [05](docs/specs/05-orch-turn-engine.md)–[13](docs/specs/13-channel-adapters.md) | ORCH, models, memory, MCP, surfaces |
| **P2** | [14](docs/specs/14-guardrails.md)–[18](docs/specs/18-testing-and-eval.md) | Guardrails, security, observability, testing |
| **P3** | [19](docs/specs/19-admin-and-operations.md)–[21](docs/specs/21-multi-agent.md) | Admin, billing, multi-agent (future) |

All specs are **Draft**. Start implementation with P0 in the order listed in the [spec index](docs/03-chatbot-platform-spec-doc-list.md).

---

## Planned tech stack

| Layer | Choice |
|-------|--------|
| Language | TypeScript (Node.js 22) |
| Monorepo | pnpm + Turborepo |
| API | Fastify · SSE streaming |
| ORCH | ECS Fargate workers + SQS |
| Data | PostgreSQL 16 (pgvector) · Redis · Neo4j · S3 |
| Frontends | Next.js (chat app, admin) · Vite (embed widget) |
| MCP | `@modelcontextprotocol/sdk` |
| Cloud | AWS (ECS, RDS, ElastiCache, CloudFront) |

Details: [`docs/04-chatbot-platform-tech-stack.md`](docs/04-chatbot-platform-tech-stack.md)

---

## Repository layout (planned)

```
chatbot-platform/
├── apps/          # api, orch-worker, chat-app, admin, embed-widget, adapters
├── packages/      # shared, db, orch, mcp-client, model-provider, memory
├── api/           # openapi.yaml, asyncapi.yaml
├── infra/         # Terraform / CDK
├── eval/          # Golden conversation eval suites
└── docs/          # Architecture, specs, planning
```

---

## Next steps

1. Review specs and promote Draft → Approved
2. Generate `api/openapi.yaml` from [spec 01](docs/specs/01-api-and-streaming-events.md)
3. Scaffold monorepo and implement P0 (API, data model, auth, tenancy)

---

## License

[GNU General Public License v3.0](LICENSE) — Copyright © 2026 Mohsin Shaikh
