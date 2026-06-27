# Observability & SLOs

**Status:** Draft  
**Priority:** P2  
**Closes gaps:** Architecture gaps §14 (Observability)

---

## Purpose

Define **telemetry signals**, **SLOs**, **alerting**, **cost attribution**, **turn replay**, and **user feedback** integration. Operators and tenants must answer: what happened, why, how long, and what did it cost?

**Related docs:**

- [`01-api-and-streaming-events.md`](01-api-and-streaming-events.md) — correlation IDs, events
- [`05-orch-turn-engine.md`](05-orch-turn-engine.md) — turn checkpoints stored for replay
- [`06-model-provider.md`](06-model-provider.md) — token usage metering
- [`07-context-assembler.md`](07-context-assembler.md) — ContextBundle logging
- [`15-security-and-compliance.md`](15-security-and-compliance.md) — PII in logs, audit separation
- [`20-billing-and-metering.md`](20-billing-and-metering.md) — usage rollups for billing

---

## Scope

### In scope

- Traces, metrics, logs, product events
- SLO definitions and error budgets
- Alerting rules
- Cost attribution dimensions
- Sampling strategy
- Turn replay / debugging
- Thumbs up/down feedback loop

### Out of scope

- Vendor-specific dashboard JSON (Grafana/Datadog templates — implementation)
- On-call rotation management

---

## Correlation IDs

Every request carries:

| ID | Scope | Propagated to |
|----|-------|---------------|
| `traceId` | End-to-end turn | All services, logs, spans |
| `requestId` | Single HTTP request | API gateway, response header |
| `tenantId` | Tenant | All signals |
| `conversationId` | Conversation | Chatbot, ORCH, memory |
| `turnId` | Single turn | ORCH, model, MCP, guardrails |

W3C Trace Context (`traceparent`) supported for external integration.

---

## Traces

### Span hierarchy

```
turn (root)
├── chatbot.receive_message
├── orch.execute_turn
│   ├── memory.retrieve
│   ├── context.assemble
│   ├── guardrails.input
│   ├── model.completion [iteration 1]
│   ├── mcp.invoke (parallel children)
│   ├── guardrails.pre_tool
│   ├── model.completion [iteration 2]
│   ├── guardrails.output
│   └── memory.extract
└── chatbot.stream_response
```

### Span attributes (required)

```
tenant.id, user.id (hashed in sampled traces), conversation.id, turn.id,
channel, model.name, mcp.server_id, mcp.tool_name,
guardrail.rule_id, error.type
```

### Retention

- Full traces: **7 days** (hot)
- Sampled traces: **90 days** (warm)
- Error traces: always retained 90 days

---

## Metrics

### Platform metrics (Prometheus-style)

| Metric | Type | Labels |
|--------|------|--------|
| `turn_duration_seconds` | histogram | tenant, channel, status |
| `turn_count_total` | counter | tenant, channel, status |
| `model_tokens_total` | counter | tenant, model, direction (input/output) |
| `model_latency_seconds` | histogram | tenant, model |
| `mcp_invoke_duration_seconds` | histogram | tenant, server, tool, status |
| `mcp_circuit_open_total` | counter | tenant, server |
| `guardrail_triggered_total` | counter | tenant, checkpoint, rule, action |
| `memory_retrieval_duration_seconds` | histogram | tenant, store (graph/vector) |
| `api_request_duration_seconds` | histogram | method, path, status |
| `sse_active_streams` | gauge | tenant |
| `quota_exceeded_total` | counter | tenant, quota_type |

### SLI derivation

| SLI | Formula |
|-----|---------|
| Turn success rate | `turns completed / turns started` |
| Turn latency | `turn_duration_seconds` p50/p99 |
| API availability | `1 - (5xx / total requests)` |
| MCP success rate | `mcp success / mcp invocations` |
| Stream TTFB | time from message POST to first `text.delta` |

---

## SLOs

### Platform SLOs (default)

| SLO | Target | Window |
|-----|--------|--------|
| API availability | 99.9% | 30 d |
| Turn success rate | 99.5% | 30 d |
| Turn latency p99 | < 15 s | 30 d |
| Stream TTFB p99 | < 500 ms | 30 d |
| MCP invoke success | 99.0% | 30 d |

### Error budget

When SLO burn exceeds 50% in window:

1. Freeze non-critical deploys
2. Page on-call if burn rate > 2x
3. Postmortem if budget exhausted

### Tenant-facing SLOs (enterprise)

Custom targets in contract; dedicated dashboard per tenant.

---

## Alerting

### Pager (P1)

| Alert | Condition | For |
|-------|-----------|-----|
| API high error rate | 5xx > 1% | 5 min |
| Turn failure spike | failed > 5% | 10 min |
| MCP server down | circuit open > 3 servers | 5 min |
| DB connection exhaustion | pool > 90% | 2 min |

### Ticket (P2)

| Alert | Condition |
|-------|-----------|
| Elevated latency | turn p99 > 20 s for 30 min |
| Guardrail block spike | 3x baseline |
| Quota exhaustion trend | tenant at 90% monthly tokens |
| Disk / storage growth | > 80% capacity |

### Routing

- Platform ops: PagerDuty / Opsgenie
- Tenant admins: email/webhook for quota and MCP degradation (optional)

---

## Logs

Structured JSON to centralized log store.

```json
{
  "timestamp": "2026-06-26T10:00:01Z",
  "level": "info",
  "message": "turn.completed",
  "traceId": "trace_01H...",
  "tenantId": "ten_01H...",
  "turnId": "turn_01H...",
  "durationMs": 4200,
  "usage": { "inputTokens": 1200, "outputTokens": 340 }
}
```

### Log levels

| Level | Use |
|-------|-----|
| ERROR | Failures requiring attention |
| WARN | Degraded, retries, guardrail warn |
| INFO | Turn lifecycle, config changes |
| DEBUG | Context assembly detail (disabled prod default) |

### PII policy

- Do not log message body at INFO+
- DEBUG message content: tenant flag + 24 h retention max
- See `15-security-and-compliance.md`

---

## Product Events

Business events for analytics (separate from audit log):

| Event | Properties |
|-------|------------|
| `turn.started` | channel, model |
| `turn.completed` | duration, tokens, toolCalls |
| `turn.failed` | errorCode |
| `tool.invoked` | server, tool, status, durationMs |
| `guardrail.triggered` | checkpoint, ruleId, action |
| `memory.written` | type, store |
| `feedback.submitted` | rating, turnId |

Emitted to event pipeline (Segment, internal Kafka) for product analytics.

---

## Cost Attribution

### Per-turn cost record

```json
{
  "turnId": "turn_01H...",
  "tenantId": "ten_01H...",
  "userId": "usr_01H...",
  "conversationId": "conv_01H...",
  "channel": "chat_app",
  "modelCostUsd": 0.0042,
  "embeddingCostUsd": 0.0001,
  "toolCostUsd": 0,
  "totalCostUsd": 0.0043,
  "inputTokens": 1200,
  "outputTokens": 340,
  "toolCalls": 1
}
```

### Rollups

| Granularity | Use |
|-------------|-----|
| Per turn | Debugging, replay |
| Hourly per tenant | Dashboards |
| Daily per tenant | Billing (`20-billing-and-metering.md`) |
| Per user | Chargeback reports (enterprise) |

Model rates table versioned; historical cost recalculated on rate updates optional.

---

## Sampling

| Signal | Strategy |
|--------|----------|
| Traces (success) | 1% head sampling |
| Traces (error) | 100% |
| Traces (slow > 30 s) | 100% |
| Logs DEBUG | 0% prod default |
| Metrics | 100% (aggregated) |
| ContextBundle | 100% stored 7 d for completed turns; PII redacted |

Tenant `traceSampleRate` override for enterprise debugging (max 100% for 1 h bursts).

---

## Turn Replay

### Purpose

Debug production issues without re-executing side-effect tools.

### Stored artifacts (per turn)

- `ContextBundle` per model iteration
- Model request/response (redacted)
- Tool invocation records (args + results)
- Guardrail results

### Replay modes

| Mode | Behavior |
|------|----------|
| `inspect` | Load artifacts in admin UI; no execution |
| `replay_model` | Re-run model with stored context; no MCP |
| `full_simulate` | Mock MCP with recorded results |

API: `POST /v1/admin/turns/{turnId}/replay` (admin only, audit logged).

Access: tenant admin own tenant; platform support with consent.

---

## User Feedback

### API

```json
POST /v1/turns/{turnId}/feedback
{
  "rating": "positive" | "negative",
  "comment": "optional text",
  "tags": ["incorrect", "slow", "unhelpful"]
}
```

### Linkage

Feedback stored with `turnId`, `traceId`, `conversationId`, `tenantId`.

### Analytics

- Negative rate by model, channel, tool
- Feed eval harness (`18-testing-and-eval.md`) for regression cases
- Weekly quality report per tenant (enterprise)

---

## Dashboards

### Platform ops

- SLO burn rate
- Error rate by service
- Latency heatmap by channel
- MCP health matrix
- Cost burn daily

### Tenant admin

- Messages and tokens (7 d / 30 d)
- Tool reliability
- Guardrail hit rate
- Top errors
- Estimated cost

---

## Synthetic Monitoring

Probes every 5 min per region:

1. Auth + create conversation
2. Send canned message
3. Assert `turn.complete` within 30 s
4. Alert on failure

Runs against dedicated synthetic tenant.

---

## Open Questions

1. OpenTelemetry collector topology — single vs per-region collectors.
2. Real-time tenant cost alerts — P2 billing integration.
3. LLM output quality scoring automated — tie to feedback + eval harness.
