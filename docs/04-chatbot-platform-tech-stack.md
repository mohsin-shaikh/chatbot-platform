# Chatbot Platform — Tech Stack

Technology choices for implementing the platform defined in [`01-chatbot-platform-architecture.md`](01-chatbot-platform-architecture.md) and [`docs/specs/`](specs/). Specs define **behavior**; this doc defines **implementation defaults**. Deviations are fine if they meet spec requirements.

---

## Principles

| Principle | Implication |
|-----------|-------------|
| **TypeScript-first monorepo** | Shared types across API, ORCH, frontends, and OpenAPI codegen |
| **Postgres as system of record** | Conversations, tenants, config, audit — one OLTP database |
| **Stateless services + queues** | ORCH workers scale horizontally; turn state in DB |
| **Managed services over self-ops** | RDS, ElastiCache, S3 in v1; K8s complexity deferred where possible |
| **Spec-aligned observability** | OpenTelemetry from day one (`16-observability-and-slos.md`) |
| **MCP-native** | Official TypeScript MCP SDK; no custom tool protocol |

---

## Stack at a Glance

| Layer | Primary choice | Spec reference |
|-------|----------------|----------------|
| Language | **TypeScript 5.x** (Node.js 22 LTS) | — |
| Monorepo | **pnpm** + **Turborepo** | — |
| Chatbot API | **Fastify** | `01-api-and-streaming-events.md` |
| ORCH | Node worker + **BullMQ** | `05-orch-turn-engine.md` |
| MCP client | **`@modelcontextprotocol/sdk`** | `10-mcp-integration.md` |
| Hot data store | **PostgreSQL 16** + **Drizzle ORM** | `02-data-model-and-persistence.md` |
| Vector search | **pgvector** (v1) → Pinecone (scale) | `08-memory.md`, `09-knowledge-base-and-ingestion.md` |
| Graph memory | **Neo4j** (Aura or self-hosted) | `08-memory.md` |
| Cache / pub-sub | **Redis 7** (ElastiCache) | `03-auth-and-identity.md`, `04-multi-tenancy-and-config.md` |
| Turn queue | **AWS SQS** (+ DLQ) | `19-admin-and-operations.md` |
| SSE fan-out | **Redis pub/sub** | `19-admin-and-operations.md` |
| Object storage | **AWS S3** | `17-attachments-and-media.md` |
| Chat app | **Next.js 15** (App Router) | `11-chat-app.md` |
| Admin console | **Next.js 15** (shared design system) | `19-admin-and-operations.md` |
| Embed widget | **Vite** (library mode) | `12-embed-widget.md` |
| Channel adapters | **Node** (Slack Bolt, Bot Framework SDK) | `13-channel-adapters.md` |
| Model providers | OpenAI, Anthropic, Azure OpenAI SDKs | `06-model-provider.md` |
| Billing | **Stripe** | `20-billing-and-metering.md` |
| Observability | **OpenTelemetry** → Grafana Cloud | `16-observability-and-slos.md` |
| CI/CD | **GitHub Actions** + Docker | `18-testing-and-eval.md`, `19-admin-and-operations.md` |
| Cloud | **AWS** (primary region + EU replica) | `15-security-and-compliance.md` |

---

## Repository Layout

```
chatbot-platform/
├── apps/
│   ├── api/                 # Chatbot REST + SSE (Fastify)
│   ├── orch-worker/         # Turn engine consumer (BullMQ/SQS)
│   ├── ingest-worker/       # KB + attachment processing
│   ├── adapter-slack/       # Slack channel adapter
│   ├── adapter-teams/       # Microsoft Teams adapter
│   ├── chat-app/            # Next.js end-user app
│   ├── admin/               # Next.js tenant + platform admin
│   └── embed-widget/        # Vite CDN bundle (loader + widget)
├── packages/
│   ├── shared/              # Types, IDs, constants, Zod schemas
│   ├── db/                  # Drizzle schema + migrations
│   ├── orch/                # Turn engine, context assembler, guardrails
│   ├── mcp-client/          # MCP connection manager + invocation
│   ├── model-provider/      # Provider adapters + token counting
│   ├── memory/              # Graph + vector client wrappers
│   └── config/              # Tenant config loader + cache
├── api/
│   ├── openapi.yaml         # Public REST (source of truth)
│   └── asyncapi.yaml          # SSE + webhook events
├── infra/                   # Terraform / CDK (AWS)
├── eval/                    # Golden conversation eval suites
└── docs/
```

---

## Backend

### Chatbot API (`apps/api`)

| Concern | Choice | Notes |
|---------|--------|-------|
| Framework | **Fastify 5** | Schema validation, hooks, strong SSE support |
| Validation | **Zod** + `fastify-type-provider-zod` | Shared schemas with `packages/shared` |
| Auth | **jose** (JWT verify/sign) | Session, embed, API key tokens (`03-auth-and-identity.md`) |
| SSE | Native `reply.raw` stream | `Last-Event-ID` replay via Redis buffer |
| API spec | OpenAPI 3.1 → `@hey-api/openapi-ts` | TS client for frontends |
| Rate limiting | `@fastify/rate-limit` + Redis | Per-tenant buckets |

### ORCH worker (`apps/orch-worker`)

| Concern | Choice | Notes |
|---------|--------|-------|
| Queue consumer | **BullMQ** (Redis) or **SQS poller** | SQS recommended for prod durability |
| Turn engine | `packages/orch` | State machine from `05-orch-turn-engine.md` |
| Stream events | Publish to Redis channel `turn:{turnId}` | API subscribes and forwards SSE |
| Cancellation | AbortController + provider cancel API | Propagate to MCP in-flight calls |
| HITL pause | Postgres `approval_records`; worker exits | Resume on any worker |

### MCP client (`packages/mcp-client`)

| Concern | Choice | Notes |
|---------|--------|-------|
| SDK | **`@modelcontextprotocol/sdk`** | stdio, SSE, streamable HTTP transports |
| Circuit breaker | **opossum** | Per `(tenantId, serverId)` |
| Timeouts | `AbortSignal.timeout()` | Per `10-mcp-integration.md` |

### Model provider (`packages/model-provider`)

| Provider | SDK |
|----------|-----|
| OpenAI | `openai` |
| Anthropic | `@anthropic-ai/sdk` |
| Azure OpenAI | `openai` (Azure config) |
| Token counting | `tiktoken` (OpenAI); provider APIs elsewhere |

Structured output (memory extraction, guardrail judge): JSON schema mode via provider native support.

### Guardrails (`packages/orch/guardrails`)

| Engine | Library / approach |
|--------|-------------------|
| Keyword / regex | Custom evaluator (no deps) |
| Classifier | `@xenova/transformers` (local) or provider embedding |
| LLM judge | `model-provider` with `gpt-4.1-mini` |

---

## Data Stores

### PostgreSQL 16 (RDS)

**Role:** Hot tier — tenants, users, conversations, messages, turns, tool calls, attachments metadata, audit log, usage rollups.

| Concern | Choice |
|---------|--------|
| ORM | **Drizzle ORM** |
| Migrations | **drizzle-kit** (SQL migrations in repo) |
| Row-level security | Postgres RLS on `tenant_id` (defense in depth) |
| Extensions | `pgvector`, `pg_trgm` (search), `uuid-ossp` or ULID in app |

**Why Drizzle:** SQL-transparent, lightweight, excellent TypeScript inference; fits spec-driven schema in `02-data-model-and-persistence.md`.

### pgvector (same Postgres instance, v1)

**Role:** Episodic vector memory + KB chunks (`08-memory.md`, `09-knowledge-base-and-ingestion.md`).

| Concern | Choice |
|---------|--------|
| Embedding model | `text-embedding-3-small` (OpenAI) |
| Index | HNSW (`vector_cosine_ops`) |
| Scale path | Migrate KB index to **Pinecone** or **Weaviate** when > 10M chunks |

Separate tables/indexes for episodic vs KB (never merge — per spec).

### Neo4j

**Role:** Graph memory — preferences, relationships, facts (`08-memory.md`).

| Concern | Choice |
|---------|--------|
| Deployment | **Neo4j Aura** (managed) or AuraDB Free for dev |
| Tenant isolation | Label prefix or `tenantId` property on all nodes |
| Driver | **`neo4j-driver`** |

**v1 alternative:** Defer graph to Postgres JSONB + application logic only if Neo4j ops cost is prohibitive; spec prefers graph DB — plan migration early.

### Redis 7 (ElastiCache)

| Use | Pattern |
|-----|---------|
| Session / token blocklist | String keys + TTL |
| Rate limits & quotas | Token bucket (`INCR` + expiry) |
| Tenant config cache | JSON blob, 60 s TTL |
| MCP tool discovery cache | JSON blob, 5 min TTL |
| SSE event buffer | Stream or list, 5 min retention |
| Turn event pub/sub | `PUBLISH turn:{turnId}` |
| BullMQ backend | Dev/small staging only |

### AWS S3

| Use | Path pattern |
|-----|--------------|
| Attachments | `{tenantId}/attachments/{attId}/` |
| KB source files | `{tenantId}/kb/{docId}/` |
| Cold archive | `{tenantId}/archive/` |

Presigned PUT/GET via `@aws-sdk/client-s3`. SSE-KMS encryption (`15-security-and-compliance.md`).

---

## Frontends

### Chat app (`apps/chat-app`)

| Concern | Choice |
|---------|--------|
| Framework | **Next.js 15** (App Router) |
| UI | **Tailwind CSS** + **shadcn/ui** |
| Data fetching | **TanStack Query v5** |
| Streaming | Native **EventSource** + reconnect polyfill |
| Markdown | **react-markdown** + **remark-gfm** |
| Auth | httpOnly cookie (session) via API BFF routes |

### Admin console (`apps/admin`)

Same stack as chat app; shared `packages/ui` component library. Separate deploy, role-gated routes.

### Embed widget (`apps/embed-widget`)

| Concern | Choice |
|---------|--------|
| Bundler | **Vite** (library + IIFE loader) |
| Output | `loader.js` (~8KB) + `widget.js` on CDN |
| CDN | **CloudFront** |
| Styling | CSS variables (no Tailwind in host page) |

---

## Channel Adapters

| Platform | Runtime | SDK |
|----------|---------|-----|
| Slack | Node (`apps/adapter-slack`) | **@slack/bolt** |
| Teams | Node (`apps/adapter-teams`) | **botbuilder** (Bot Framework) |
| WhatsApp | Node | Meta Cloud API (HTTP) |
| Telegram | Node | **telegraf** or raw Bot API |

Each adapter: stateless webhook receiver → normalize → Chatbot API → subscribe turn events → platform reply.

---

## Ingestion Worker (`apps/ingest-worker`)

| Concern | Choice |
|---------|--------|
| Trigger | SQS message on attachment/KB upload complete |
| PDF / docs | **`pdf-parse`**, **`mammoth`** (docx); fallback **Apache Tika** sidecar if quality insufficient |
| Virus scan | **ClamAV** (sidecar or cloud API) |
| Audio STT | OpenAI Whisper API |
| Images | Sharp (thumbnails); vision describe via model provider |
| Chunking | LangChain `RecursiveCharacterTextSplitter` or custom token-based splitter |

**Optional Python worker:** If document extraction quality becomes a bottleneck, isolate Tika/unstructured.io in a small Python Lambda; contract via SQS. Not required for v1.

---

## Infrastructure (AWS)

```
                    ┌──────────────┐
                    │  CloudFront  │  chat app, admin, embed
                    └──────┬───────┘
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
     ┌─────────────────┐      ┌─────────────────┐
     │  ALB            │      │  API Gateway     │  (optional) channel webhooks
     │  ECS Fargate    │      │  → Lambda/ECS    │
     │  api, adapters  │      └─────────────────┘
     └────────┬────────┘
              │
     ┌────────┴────────┐
     ▼                   ▼
┌─────────┐      ┌───────────────┐
│   SQS   │ ───► │ orch-worker   │  ECS Fargate (auto-scale on depth)
│ turn.q  │      │ ingest-worker │
└─────────┘      └───────┬───────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
   ┌──────────┐   ┌──────────┐   ┌──────────┐
   │ RDS      │   │ Redis    │   │ S3       │
   │ Postgres │   │ElastiCache│  │          │
   └──────────┘   └──────────┘   └──────────┘
         │
         ▼
   ┌──────────┐   ┌──────────┐
   │ Neo4j    │   │ Secrets  │
   │ Aura     │   │ Manager  │
   └──────────┘   └──────────┘
```

| Service | Purpose |
|---------|---------|
| **ECS Fargate** | api, orch-worker, ingest-worker, adapters (v1 — simpler than EKS) |
| **RDS PostgreSQL** | Multi-AZ prod; read replica for analytics |
| **ElastiCache Redis** | Cluster mode disabled v1; single shard + replica |
| **SQS** | Turn queue + ingestion queue + DLQ |
| **S3** | Attachments, KB, exports |
| **Secrets Manager** | Provider keys, MCP creds, Stripe, JWT signing keys |
| **KMS** | S3 SSE, RDS encryption, tenant DEK (enterprise) |
| **CloudWatch** | Logs; metric alarms → PagerDuty |
| **Route 53** | DNS |

**Scale path:** Move to **EKS** when multi-region active-active or > 50 services (`19-admin-and-operations.md`).

**IaC:** **Terraform** or **AWS CDK (TypeScript)** in `infra/`.

---

## Observability

| Signal | Tool |
|--------|------|
| Traces | **OpenTelemetry SDK** (Node auto-instrumentation) |
| Collector | OTel Collector → **Grafana Cloud Tempo** |
| Metrics | OTel → **Prometheus** / Grafana Mimir |
| Logs | Structured JSON → **CloudWatch Logs** → Grafana Loki |
| Dashboards | Grafana (SLO burn, per-tenant usage) |
| Errors | **Sentry** (optional) |
| Uptime | Synthetic probes (`16-observability-and-slos.md`) |
| Audit log | Postgres append-only table → S3 archive (immutable) |

Correlation: propagate `traceId`, `tenantId`, `turnId` via OTel baggage.

---

## Security & Secrets

| Concern | Choice |
|---------|--------|
| TLS | ACM certificates on ALB / CloudFront |
| JWT signing | RS256; keys in Secrets Manager; rotation quarterly |
| API key storage | bcrypt hash in Postgres |
| Dependency scanning | **Dependabot** + **Trivy** in CI |
| Container signing | **cosign** |
| WAF | AWS WAF on ALB (rate limit, geo block optional) |

See `15-security-and-compliance.md` for full control matrix.

---

## Billing

| Concern | Choice |
|---------|--------|
| Payments | **Stripe** (Checkout, Subscriptions, Usage Records) |
| Metering events | Internal Kafka or SQS → aggregator → `usage_hourly` table |
| Customer portal | Stripe Customer Portal (embedded link in admin) |

---

## Testing & CI

| Layer | Tool |
|-------|------|
| Unit / integration | **Vitest** |
| API tests | **supertest** + Fastify inject |
| DB tests | **Testcontainers** (Postgres, Redis) |
| Eval harness | Custom runner in `eval/` (YAML fixtures) |
| Load tests | **k6** (weekly staging) |
| E2E (frontends) | **Playwright** |
| CI | **GitHub Actions** |
| Preview deploys | PR → staging namespace |

Gates per `18-testing-and-eval.md`.

---

## Local Development

```bash
# Prerequisites: Node 22, pnpm, Docker

docker compose up -d   # Postgres (pgvector), Redis, MinIO, Neo4j, LocalStack (SQS/S3 optional)
pnpm install
pnpm db:migrate
pnpm dev               # Turborepo: api + orch-worker + chat-app
```

| Service | Local substitute |
|---------|------------------|
| S3 | **MinIO** |
| SQS | **LocalStack** or BullMQ-only dev mode |
| Neo4j | Docker `neo4j:5` |
| Models | OpenAI/Anthropic API keys in `.env.local` |
| MCP | Local stdio servers or platform mock MCP |
| Stripe | Stripe CLI webhooks → localhost |

**`.env.example`** committed; secrets never in repo.

---

## SDK Generation

| Artifact | Generator | Consumers |
|----------|-----------|-----------|
| `api/openapi.yaml` | `@hey-api/openapi-ts` | chat-app, admin, embed, partners |
| `api/asyncapi.yaml` | Custom types or `asyncapi-generator` | Event consumers |
| Partner SDKs | OpenAPI → Python (`openapi-generator`) | CI nightly publish optional |

---

## Version Policy

| Component | Pin strategy |
|-----------|--------------|
| Node.js | 22 LTS (`.nvmrc`) |
| PostgreSQL | 16.x |
| TypeScript | `strict: true`; upgrade minor freely |
| Provider SDKs | Pin minor; test on provider changelog |
| Neo4j | Match Aura version |

---

## Alternatives Considered

| Area | Alternative | When to switch |
|------|-------------|----------------|
| API framework | Hono, NestJS | Team preference; Fastify default for SSE perf |
| ORM | Prisma | If team strongly prefers Prisma Migrate UX |
| Vector DB | Pinecone, Weaviate | > 10M vectors or dedicated search team |
| Graph DB | Amazon Neptune | Already on AWS graph workloads at scale |
| Queue | RabbitMQ, Redis Streams only | Multi-cloud or simpler single-node dev |
| Compute | Lambda for adapters | Low-volume channels only |
| Frontend | Vite SPA (no Next) | If SSR/auth BFF not needed |
| Cloud | GCP (Cloud Run, GCS) | Existing org standard on GCP |
| Language | Python ORCH (FastAPI) | Heavy ML team; loses TS monorepo sharing |

---

## Related Docs

| Doc | Relationship |
|-----|--------------|
| [`01-chatbot-platform-architecture.md`](01-chatbot-platform-architecture.md) | System shape |
| [`03-chatbot-platform-spec-doc-list.md`](03-chatbot-platform-spec-doc-list.md) | Spec index |
| [`docs/specs/`](specs/) | Behavioral requirements this stack implements |
| [`19-admin-and-operations.md`](specs/19-admin-and-operations.md) | Deploy units, scaling, DR |
| [`18-testing-and-eval.md`](specs/18-testing-and-eval.md) | CI gates and test tooling |

---

## Open Decisions

1. **EKS vs ECS Fargate for prod** — start Fargate; revisit at ~100K turns/day.
2. **Neo4j vs Postgres-only graph (v1 shortcut)** — Neo4j recommended; Postgres JSONB acceptable for MVP with migration plan.
3. **Kafka vs SQS for metering events** — SQS sufficient v1; Kafka if real-time analytics pipeline needed.
4. **Single Next.js app vs split chat-app + admin** — split recommended for deploy isolation and bundle size.
