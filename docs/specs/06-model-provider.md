# Model Provider

**Status:** Draft  
**Priority:** P1  
**Closes gaps:** Architecture gaps §5 (Model Layer)

---

## Purpose

Define the **model provider abstraction** that ORCH uses for completions, tool calls, and structured output. Covers provider routing, fallback, parameters, cost caps, and multimodal support.

**Related docs:**

- [`05-orch-turn-engine.md`](05-orch-turn-engine.md) — when and how model is called
- [`07-context-assembler.md`](07-context-assembler.md) — prompt/token budget input
- [`04-multi-tenancy-and-config.md`](04-multi-tenancy-and-config.md) — per-tenant model defaults
- [`16-observability-and-slos.md`](16-observability-and-slos.md) — token metrics, cost attribution
- [`17-attachments-and-media.md`](17-attachments-and-media.md) — vision/audio inputs

---

## Scope

### In scope

- Unified completion interface across providers
- Provider registry and routing
- Fallback and retry
- Parameters (temperature, max tokens, etc.)
- Cost/token budget enforcement
- Structured output mode
- Multimodal message content

### Out of scope

- Fine-tuning or training pipelines
- Provider account billing setup
- Model evaluation benchmarks — `18-testing-and-eval.md`

---

## Provider Abstraction

### Interface

```
ModelProvider.complete(request: CompletionRequest): CompletionResponse
ModelProvider.completeStream(request): AsyncStream<CompletionChunk>
ModelProvider.cancel(requestId): void
ModelProvider.countTokens(messages, tools): int
```

### CompletionRequest

```json
{
  "requestId": "mreq_01H...",
  "tenantId": "ten_01H...",
  "turnId": "turn_01H...",
  "model": "gpt-4.1",
  "messages": [
    { "role": "system", "content": "..." },
    { "role": "user", "content": [ { "type": "text", "text": "..." } ] }
  ],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "billing.get_invoice",
        "description": "...",
        "parameters": { "type": "object", "properties": {} }
      }
    }
  ],
  "parameters": {
    "temperature": 0.7,
    "maxOutputTokens": 4096,
    "stop": [],
    "toolChoice": "auto"
  },
  "responseFormat": null,
  "stream": true
}
```

### CompletionResponse (non-streaming)

```json
{
  "requestId": "mreq_01H...",
  "model": "gpt-4.1",
  "content": [
    { "type": "text", "text": "Your invoice was paid." }
  ],
  "toolCalls": [],
  "usage": {
    "inputTokens": 1200,
    "outputTokens": 45,
    "cachedTokens": 0
  },
  "finishReason": "stop",
  "providerRequestId": "openai-req-abc",
  "latencyMs": 890
}
```

### CompletionChunk (streaming)

```json
{
  "requestId": "mreq_01H...",
  "delta": { "type": "text", "text": "Your " },
  "toolCallDelta": null,
  "finishReason": null
}
```

Tool call chunks stream incrementally (`toolCallDelta`) until complete.

---

## Supported Providers (v1)

| Provider ID | Models (examples) | Tool calling | Vision | Notes |
|-------------|-------------------|--------------|--------|-------|
| `openai` | gpt-4.1, gpt-4.1-mini | Yes | Yes | Primary |
| `anthropic` | claude-sonnet-4, claude-haiku | Yes | Yes | |
| `azure_openai` | Deployment-mapped | Yes | Yes | Per-tenant deployment config |
| `self_hosted` | vLLM, TGI endpoints | Yes* | Varies | OpenAI-compatible API |

*Self-hosted must expose OpenAI-compatible `/v1/chat/completions`.

### Provider credentials

Stored per tenant in secrets vault (encrypted). Platform may provide shared keys for free tier with rate limits.

```json
{
  "providerId": "openai",
  "apiKeyRef": "vault://ten_01H.../openai",
  "organizationId": "org-..."
}
```

---

## Routing

### Selection order

1. Explicit `model` in turn metadata (if permitted by role)
2. Channel override (`model.perChannel.slack`)
3. Tenant default (`model.default`)
4. Platform default (`gpt-4.1-mini`)

### Task-based routing (optional)

Tenant config may map intent → model:

```json
{
  "routing": {
    "structured_extraction": "gpt-4.1-mini",
    "complex_reasoning": "gpt-4.1",
    "default": "gpt-4.1"
  }
}
```

ORCH sets `request.task` for memory extraction and guardrail classifiers.

---

## Fallback

```
Primary model call fails
  → if retryable (429, 503, timeout):
       retry once with exponential backoff
  → if still failing OR non-retryable provider error:
       if tenant.model.fallback configured:
         call fallback model
       else:
         fail turn
```

| Error | Retryable | Fallback |
|-------|-----------|----------|
| 429 rate limit | Yes | Yes |
| 503 unavailable | Yes | Yes |
| Timeout | Yes | Yes |
| 400 bad request (context) | No | No — truncate or fail |
| 401 auth | No | No — alert tenant admin |
| Content policy violation | No | No |

Fallback emits observability event `model.fallback` with primary/fallback model IDs.

---

## Parameters

### Resolution

```
effective = merge(platformDefaults, tenant.model.parameters, turnOverrides)
```

| Parameter | Default | Range | Notes |
|-----------|---------|-------|-------|
| `temperature` | 0.7 | 0–2 | Lower for tool-heavy turns |
| `maxOutputTokens` | 4096 | 1–128K | Capped by plan |
| `topP` | 1.0 | 0–1 | Provider-specific |
| `stop` | [] | — | Stop sequences |
| `toolChoice` | `auto` | `auto` \| `none` \| `required` \| `{name}` | |
| `presencePenalty` | 0 | — | OpenAI only |
| `frequencyPenalty` | 0 | — | OpenAI only |

### Per-turn overrides

API may pass `metadata.modelOverrides` on message POST. Allowed only if token has `admin:config` or tenant enables `allowUserModelOverride`.

---

## Cost & Token Budgets

### Pre-call check

```
inputTokens = provider.countTokens(messages, tools)
if inputTokens > contextBudget:
  context assembler must truncate first (07-context-assembler.md)
if inputTokens still > hardLimit:
  fail turn (context_overflow)
if tenant monthly tokens exhausted:
  fail or degrade per quota policy (04-multi-tenancy-and-config.md)
```

### Per-turn cap

`maxInputTokensPerTurn` + `maxOutputTokensPerTurn` from tenant config. ORCH stops stream when output cap reached (`finishReason: length`).

### Metering

Every completion records:

```json
{
  "tenantId": "ten_01H...",
  "turnId": "turn_01H...",
  "model": "gpt-4.1",
  "inputTokens": 1200,
  "outputTokens": 340,
  "estimatedCostUsd": 0.0042
}
```

Cost rates in platform config table; updated without deploy.

---

## Structured Output

For ORCH internal tasks (memory extraction, guardrail classification):

```json
{
  "responseFormat": {
    "type": "json_schema",
    "jsonSchema": {
      "name": "memory_extraction",
      "schema": {
        "type": "object",
        "properties": {
          "candidates": { "type": "array" }
        }
      },
      "strict": true
    }
  }
}
```

- Uses smaller/cheaper model by default (`gpt-4.1-mini`).
- Parse failure → retry once → else skip extraction (log warning).

---

## Multimodal

### Content parts (user/assistant messages)

| Type | Providers | Notes |
|------|-----------|-------|
| `text` | All | |
| `image` | OpenAI, Anthropic, Azure | URL or base64; max size per `17-attachments-and-media.md` |
| `audio` | OpenAI (selected) | Future; transcription pre-pass preferred |

ORCH passes attachment references from context assembler; provider adapter resolves to provider-native format.

### Vision routing

If message contains images and selected model lacks vision → auto-upgrade to tenant `model.visionDefault` or fail with clear error.

---

## Streaming

- ORCH always requests streaming for user-facing turns.
- Provider adapter maps provider stream → `CompletionChunk` → ORCH emits `text.delta`.
- **Cancel:** `provider.cancel(requestId)` on turn cancellation.
- Time-to-first-token target: < 2 s p99 (excluding queue wait).

---

## Caching

Providers supporting prompt caching (e.g. Anthropic, OpenAI prefix cache):

- Context assembler orders messages to maximize cache hits (static system prompt first).
- Track `cachedTokens` in usage metrics.

---

## Security

- API keys never logged or included in `ContextBundle` dumps.
- Model providers are subprocessors; tenant DPAs list them (`15-security-and-compliance.md`).
- User content sent to providers respects tenant `dataProcessing` config (opt-out regions).

---

## Open Questions

1. Bring-your-own-key (BYOK) vs platform-managed keys — support both; BYOK for enterprise.
2. Local/on-device models — out of scope v1.
3. Model A/B testing per tenant — P2 via feature flags + routing weights.
