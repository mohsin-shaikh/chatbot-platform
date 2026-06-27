# Chatbot Platform — Infrastructure & Running Cost

Infrastructure sizing, monthly cost estimates, and unit economics for the platform described in [`04-chatbot-platform-tech-stack.md`](04-chatbot-platform-tech-stack.md). Figures are **order-of-magnitude** for planning (AWS **us-east-1**, mid-2026 list pricing). Actuals vary with utilization, reserved capacity, and provider list-price changes.

**Related docs:**

- [`04-chatbot-platform-tech-stack.md`](04-chatbot-platform-tech-stack.md) — service choices
- [`specs/19-admin-and-operations.md`](specs/19-admin-and-operations.md) — deploy units, DR, scaling
- [`specs/20-billing-and-metering.md`](specs/20-billing-and-metering.md) — plans, overage, revenue model
- [`specs/16-observability-and-slos.md`](specs/16-observability-and-slos.md) — SLOs, cost attribution

---

## Summary

| Cost type | Typical share at scale | Who pays |
|-----------|------------------------|----------|
| **LLM / embedding API** | 60–85% of variable cost | Passed through to tenants (metered) |
| **AWS compute (ECS)** | 10–20% | Platform |
| **RDS + Redis** | 5–15% | Platform |
| **Neo4j, observability, CDN, misc** | 5–10% | Platform |
| **Stripe fees** | ~2.9% + $0.30 on subscription | Platform |

**Key insight:** Platform **fixed infra** must be covered by subscription revenue (Pro/Enterprise). **LLM usage** is largely a pass-through — price tokens/messages with margin in overage rates (`20-billing-and-metering.md`), not subsidize from base fee alone.

---

## Deployment Profiles

Three AWS profiles aligned to lifecycle stage. All assume single **primary region**; EU region adds ~80–100% to fixed infra for data residency.

### Profile comparison

| | **Dev / Local** | **Staging** | **Production — Launch** | **Production — Growth** | **Production — Scale** |
|---|-----------------|-------------|-------------------------|-------------------------|------------------------|
| **Purpose** | Engineer laptops + CI | Pre-prod, load test | First paying customers | 50+ tenants, 500K msg/mo | 500+ tenants, 5M+ msg/mo |
| **Tenants** | 0 | Test only | 5–20 | 50–200 | 500+ |
| **Messages/mo** | — | Synthetic | ~50K | ~500K | ~5M+ |
| **Fixed infra/mo** | **$0–80** | **$350–550** | **$750–1,100** | **$2,500–4,000** | **$8,000–15,000** |
| **Variable (LLM)/mo** | Usage-based | Low | **$150–500** | **$1,500–5,000** | **$15,000–50,000+** |

---

## Environment Details

### Dev / local

| Component | Implementation | Cost |
|-----------|----------------|------|
| Compute | Docker Compose on laptop | $0 |
| Postgres + pgvector | Docker | $0 |
| Redis, MinIO, Neo4j | Docker | $0 |
| Models | Developer API keys | ~$20–100/mo per active dev |
| Optional shared sandbox | `db.t4g.micro` RDS, single ECS task | ~$50–80/mo |

No production data. ClamAV and LocalStack optional in Compose.

---

### Staging

Single-AZ, smaller instances, no cross-region replica. Mirrors prod topology.

| Service | Sizing | Est. monthly |
|---------|--------|--------------|
| **ECS Fargate** — api | 1 task, 0.5 vCPU, 1 GB | $15 |
| **ECS Fargate** — orch-worker | 1 task, 1 vCPU, 2 GB | $30 |
| **ECS Fargate** — ingest + adapters | 1 shared task | $20 |
| **RDS PostgreSQL** | `db.t4g.medium`, 50 GB gp3, single-AZ | $55 |
| **ElastiCache Redis** | `cache.t4g.micro` | $12 |
| **S3** | ~50 GB | $2 |
| **CloudFront** | Minimal traffic | $5 |
| **SQS, Secrets, CloudWatch** | Low volume | $15 |
| **Neo4j Aura** | Free tier or smallest paid | $0–65 |
| **Grafana Cloud** | Free tier | $0 |
| **ALB** | 1 | $20 |
| **NAT Gateway** | 1 (or VPC endpoints −$32) | $0–35 |
| | **Total** | **~$350–550** |

Load tests (k6) run here weekly; may spike ECS autoscaling temporarily (+$20–50).

---

### Production — Launch

First production region. Multi-AZ for RDS. Target **99.9%** API availability (`16-observability-and-slos.md`).

#### Compute (ECS Fargate)

| Service | Tasks | vCPU | Memory | Hours/mo | Est. monthly |
|---------|-------|------|--------|----------|--------------|
| **api** | 2 | 0.5 | 1 GB | 1,460 | $55 |
| **orch-worker** | 2 | 1 | 2 GB | 1,460 | $110 |
| **ingest-worker** | 1 | 0.5 | 1 GB | 730 | $28 |
| **adapter-slack** (+ others bundled v1) | 1 | 0.5 | 1 GB | 730 | $28 |
| | | | | **Subtotal** | **~$220** |

Fargate pricing ≈ $0.04048/vCPU-hr + $0.004445/GB-hr (us-east-1).

Autoscaling: orch-worker 2→4 tasks when SQS depth > 50 (+~$110/mo at sustained peak).

#### Data & storage

| Service | Sizing | Est. monthly |
|---------|--------|--------------|
| **RDS PostgreSQL 16** | `db.r6g.large`, Multi-AZ, 100 GB gp3, pgvector | $280–350 |
| **RDS backups** | 100 GB snapshot storage | $10 |
| **ElastiCache Redis 7** | `cache.r6g.large`, 1 replica | $150–180 |
| **S3** | 200 GB attachments/KB + requests | $8–15 |
| **S3 cross-AZ transfer** | Minimal at launch | $5 |

#### Networking & edge

| Service | Est. monthly |
|---------|--------------|
| **ALB** (api) | $25 + ~$10 LCU |
| **CloudFront** (chat app, admin, embed) — 500 GB egress | $45–85 |
| **Route 53** hosted zone | $1 |
| **NAT Gateway** ×1 (or egress-only + VPC endpoints) | $35–45 |
| **AWS WAF** (basic rules) | $10–15 |
| **ACM** | $0 |

#### Messaging & ops

| Service | Est. monthly |
|---------|--------------|
| **SQS** (turn + ingest queues) | $5–10 |
| **Secrets Manager** (~15 secrets) | $8 |
| **KMS** | $2–5 |
| **CloudWatch Logs** (~50 GB ingest) | $30–50 |
| **CloudWatch alarms** | $5 |

#### Third-party managed

| Service | Sizing | Est. monthly |
|---------|--------|--------------|
| **Neo4j Aura** | Professional, smallest (~4 GB) | $65–250 |
| **Grafana Cloud** | Pro (traces + metrics) | $50–200 |
| **Sentry** (optional) | Team tier | $0–26 |
| **PagerDuty** | 1–3 users | $0–60 |
| **GitHub Actions** | Team CI minutes | $0–20 |

#### Launch total

| Category | Monthly |
|----------|---------|
| Compute | $220–330 |
| Data stores | $450–560 |
| Network & CDN | $120–170 |
| Messaging & AWS ops | $50–80 |
| Third-party | $115–530 |
| **Fixed infra total** | **~$750–1,100/mo** |

*Excludes LLM, Stripe, domain, support headcount.*

---

### Production — Growth

Triggers: >100K messages/mo, >30 concurrent turns p99, RDS CPU >60% sustained.

| Change from Launch | Est. delta |
|--------------------|------------|
| api tasks 2→4 | +$55 |
| orch-worker 2→6 | +$220 |
| RDS → `db.r6g.xlarge` Multi-AZ | +$250 |
| RDS read replica (analytics) | +$180 |
| Redis → `cache.r6g.xlarge` | +$120 |
| CloudFront 2 TB egress | +$100 |
| Neo4j Aura scale up | +$100 |
| Grafana / logs volume | +$100 |
| **Incremental** | **+~$1,700/mo** |

**Growth fixed total: ~$2,500–4,000/mo** (one region).

---

### Production — Scale

Triggers: >2M messages/mo, multi-region requirement, Pinecone migration for vectors.

| Change | Notes |
|--------|-------|
| orch-worker autoscale 10–30 tasks | Queue-driven |
| RDS → `db.r6g.2xlarge` + 2 read replicas | Or Aurora PostgreSQL |
| **Pinecone** serverless | Offload pgvector at >10M chunks |
| ElastiCache cluster mode | Redis memory pressure |
| **Second region** (EU) | Full stack duplicate |
| Consider **EKS** | >50 microservices / advanced rollout |
| MSK or Kafka | Real-time metering analytics |

**Scale fixed total: ~$8,000–15,000/mo** (single region); **~$14,000–25,000** dual-region.

---

## Variable Costs (Usage-Driven)

### LLM — dominant variable cost

Assumptions for **average turn** (one user message, typical tool-light Q&A):

| Parameter | Conservative | Typical | Heavy (tools + RAG) |
|-----------|--------------|---------|---------------------|
| Model mix | 90% mini, 10% full | 70% mini, 30% full | 50% mini, 50% full |
| Model calls per turn | 1 | 1.5 | 3 |
| Input tokens / call | 2,000 | 3,500 | 8,000 |
| Output tokens / call | 300 | 500 | 800 |
| **LLM cost / turn** | **~$0.002** | **~$0.006** | **~$0.025** |

Reference list prices (verify at deployment time):

| Model | Input / 1M tokens | Output / 1M tokens |
|-------|-------------------|---------------------|
| gpt-4.1-mini | ~$0.40 | ~$1.60 |
| gpt-4.1 | ~$2.00 | ~$8.00 |
| claude-sonnet (comparable) | ~$3.00 | ~$15.00 |
| text-embedding-3-small | ~$0.02 / 1M tokens | — |

#### LLM cost per 1,000 messages

| Profile | Est. COGS |
|---------|-----------|
| Conservative | **$2–3** |
| Typical | **$5–8** |
| Heavy (support bot + tools) | **$20–40** |

Memory extraction (+mini model call): +~$0.0003/turn.  
Guardrail LLM-judge (10% of turns): +~$0.0005/turn average.

**These costs are billed to tenants** via token metering and overage (`20-billing-and-metering.md`), not absorbed by platform.

---

### Embeddings

| Use | Tokens/doc | Cost per 1,000 docs (10 chunks each) |
|-----|------------|--------------------------------------|
| KB ingest | ~5,000 avg | ~$1.00 |
| Memory episodic | ~500/turn | ~$0.01 / 1,000 turns |

---

### AWS variable (marginal)

| Resource | Driver | Cost at 500K msg/mo |
|----------|--------|---------------------|
| S3 storage | Attachments + KB | ~$5–30 |
| S3 / CloudFront egress | File downloads | ~$20–80 |
| SQS | 2× messages (enqueue + ingest) | ~$2 |
| CloudWatch logs | ~2 KB/turn | ~$15–40 |
| Fargate scale-out | Peak concurrent turns | ~$50–200 |
| **AWS variable subtotal** | | **~$100–350/mo** |

Marginal AWS cost per 1,000 messages ≈ **$0.20–0.70** (after base capacity).

---

### Stripe

| Item | Rate |
|------|------|
| Pro subscription ($99/mo) | 2.9% + $0.30 ≈ **$3.17** |
| Usage overage charges | 2.9% + $0.30 per invoice line |
| Enterprise (invoiced) | Lower via negotiated rates |

---

## Unit Economics

### Platform COGS per message (internal view)

| Component | Typical | Notes |
|-----------|---------|-------|
| LLM (pass-through) | $0.005 | Tenant-metered |
| AWS marginal | $0.0005 | Shared infra amortized |
| Embeddings amortized | $0.0002 | |
| Neo4j / observability amortized | $0.0003 | Fixed ÷ volume |
| **Total COGS** | **~$0.006/msg** | Before support |

At **500K messages/mo**: ~$3,000 LLM (tenant-paid) + ~$250 platform infra marginal + ~$3,000 fixed infra allocation.

---

### Pro plan economics ($99/mo)

From `20-billing-and-metering.md`: 50,000 messages, 10M tokens included.

| Scenario | Tenant usage | Platform revenue | Platform infra share* | LLM (tenant-funded) |
|----------|--------------|------------------|----------------------|---------------------|
| Light | 5K msg, 1M tokens | $99 | ~$15 | ~$25 (within token cap) |
| At quota | 50K msg, 10M tokens | $99 | ~$15 | ~$250–400 |
| Heavy + overage | 80K msg, 18M tokens | $99 + ~$390 overage† | ~$20 | ~$600 |

\* Allocated fixed infra ~$750/mo ÷ 50 Pro tenants ≈ **$15/tenant/mo**.  
† Overage example: 30K extra messages × $0.01 + 8M extra tokens at blended rates.

**Break-even (infra only):** ~**8–12 Pro tenants** at $99/mo cover Launch fixed stack (~$900/mo).  
**Sustainable SaaS margin:** target **50+ Pro tenants** or mix of Enterprise contracts; LLM overage provides usage margin without infra risk.

---

### Free tier economics

1,000 messages/mo, 500K tokens (`20-billing-and-metering.md`).

| Item | Per free tenant/mo |
|------|-------------------|
| LLM COGS | ~$5–8 |
| Infra allocation | ~$1–2 |
| **Total cost** | **~$6–10** |

Cap free tenants (waitlist, credit card for verification) or aggressive hard blocks at quota. **100 active free tenants ≈ $600–1,000/mo** unfunded LLM cost — budget explicitly or fund as marketing CAC.

---

## Cost by Deploy Unit

Maps to [`19-admin-and-operations.md`](specs/19-admin-and-operations.md) deploy units.

| Unit | Infra home | Scales with | Launch $/mo |
|------|------------|-------------|-------------|
| Chatbot API | ECS Fargate | Requests, SSE connections | ~$55 |
| ORCH worker | ECS Fargate | Queue depth, turn duration | ~$110 |
| Ingest worker | ECS Fargate | Upload volume | ~$28 |
| Channel adapters | ECS Fargate | Webhook rate | ~$28 |
| Chat app + admin | CloudFront + S3 (static) | Egress | ~$30 |
| Embed widget | CloudFront | Egress | ~$15 |
| Postgres | RDS | Data size, IOPS | ~$320 |
| Redis | ElastiCache | Memory, ops/sec | ~$165 |
| Neo4j | Aura | Graph size | ~$65–250 |
| Turn queue | SQS | Messages | ~$5 |
| Observability | Grafana + CloudWatch | Log/trace volume | ~$80 |

---

## Scaling Triggers & Actions

| Signal | Threshold | Cost impact |
|--------|-----------|-------------|
| SQS turn queue depth | >100 for 5 min | +orch-worker tasks (+$55/task/mo) |
| API p99 latency | >500 ms | +api tasks |
| RDS CPU | >70% 15 min | Instance class bump (+$150–300/mo) |
| Redis memory | >75% | Larger node (+$80–150/mo) |
| pgvector query p99 | >200 ms | Pinecone migration (+$70–500/mo) |
| CloudFront egress | >5 TB/mo | Review caching; +$400/TB |
| Monthly LLM spend | >30% MoM growth | Review model routing defaults |

---

## Disaster Recovery Cost

From `19-admin-and-operations.md`:

| DR component | Launch | Growth |
|--------------|--------|--------|
| RDS cross-region read replica | +$140–180/mo | Recommended for prod |
| S3 cross-region replication | +$5–20/mo | Per 100 GB |
| Standby ECS in DR region (cold) | +$0 (Terraform only) | RTO 1 h |
| Warm standby (minimal tasks) | +$100–200/mo | RTO 15 min |
| Neo4j Aura DR | Included in Aura Professional+ | — |

**Minimum prod DR add-on: ~$150–250/mo** (async replica + replicated S3).

---

## Cost Optimization Levers

| Lever | Savings | Tradeoff |
|-------|---------|----------|
| **Reserved RDS / ElastiCache** (1 yr) | 30–40% on DB/cache | Commitment |
| **Fargate Savings Plans** | 20–30% compute | Commitment |
| **VPC endpoints** (S3, SQS) vs NAT | ~$35–100/mo | Setup complexity |
| **Neo4j → Postgres JSONB graph (MVP)** | $65–250/mo | Weaker graph queries; migration later |
| **Grafana Cloud free / self-host** | $50–200/mo | Ops burden |
| **Default model → mini** | 60–80% LLM COGS | Quality for complex tasks |
| **Prompt caching** (provider feature) | 10–50% input tokens | Provider lock-in features |
| **Aggressive context truncation** | 20–40% input tokens | Recall quality |
| **pgvector vs Pinecone at launch** | Defer $70+/mo | Scale limit ~5M vectors |
| **Single NAT, single-AZ staging** | Obvious | Availability |

---

## Monthly Budget Template

Use for finance planning. Fill **Actual** monthly.

| Line item | Launch budget | Growth budget | Actual |
|-----------|---------------|---------------|--------|
| ECS Fargate (all services) | $220 | $600 | |
| RDS PostgreSQL | $320 | $700 | |
| ElastiCache Redis | $165 | $280 | |
| S3 + CloudFront | $100 | $250 | |
| ALB + NAT + WAF | $80 | $120 | |
| SQS, CloudWatch, Secrets | $60 | $120 | |
| Neo4j Aura | $150 | $300 | |
| Grafana / Sentry / PagerDuty | $100 | $200 | |
| **AWS + ops subtotal** | **$1,195** | **$2,570** | |
| LLM + embeddings (pass-through) | $400 | $4,000 | |
| Stripe fees | $50 | $400 | |
| **Grand total** | **~$1,650** | **~$7,000** | |

---

## Monitoring Cost

Track in admin dashboard (`16-observability-and-slos.md`, `20-billing-and-metering.md`):

| Metric | Alert if |
|--------|----------|
| `platform.infra_cost_daily` | >20% over budget |
| `platform.llm_cost_daily` | Anomaly +50% WoW |
| `cost_per_turn_p50` | >$0.02 |
| `cost_per_tenant_infra` | >$50/mo (unprofitable tenant) |
| `free_tier_llm_subsidy` | >$500/mo total |

---

## Related Docs

| Doc | Use |
|-----|-----|
| [`04-chatbot-platform-tech-stack.md`](04-chatbot-platform-tech-stack.md) | What to size |
| [`specs/20-billing-and-metering.md`](specs/20-billing-and-metering.md) | Pricing vs COGS |
| [`specs/19-admin-and-operations.md`](specs/19-admin-and-operations.md) | When to scale |
| [`specs/06-model-provider.md`](specs/06-model-provider.md) | Model routing affects LLM line |

---

## Assumptions & Caveats

1. Prices are **estimates** — run [AWS Pricing Calculator](https://calculator.aws/) before commit.
2. LLM list prices change frequently — maintain internal **rate card** synced to billing.
3. **Enterprise single-tenant** deployments (dedicated VPC/RDS) start at **~$2,000–5,000/mo** fixed before usage.
4. **Support and engineering headcount** excluded — largest real cost at early stage.
5. Dual-region and HIPAA BAA add **20–40%** to infra and compliance tooling.

---

## Open Decisions

1. **Neo4j Aura at Launch** vs Postgres-only graph — saves ~$150/mo; decide before prod.
2. **Cross-region replica at Launch** vs Growth — +$150/mo for RPO compliance story.
3. **Free tier LLM subsidy cap** — max N free tenants or require upgrade path at 500 messages.
