# Multi-Agent (Future)

**Status:** Draft — **Future / not v1**  
**Priority:** P3  
**Closes gaps:** Architecture gaps §20 (Future Multi-Agent), architecture doc Future section

---

## Purpose

Specify how **multi-agent orchestration** extends ORCH **without changing** Chatbot API, MCP Client, or integration surface contracts. Defines supervisor behavior, specialist agents, delegation, budgets, failure handling, and user visibility.

**Status note:** This spec is **design-only** for v1. Implementation gated by tenant feature flag `features.multiAgent` (default `false`).

**Related docs:**

- [`05-orch-turn-engine.md`](05-orch-turn-engine.md) — single-agent turn loop (baseline)
- [`07-context-assembler.md`](07-context-assembler.md) — per-agent context
- [`08-memory.md`](08-memory.md) — shared vs per-agent memory
- [`10-mcp-integration.md`](10-mcp-integration.md) — tool ownership
- [`16-observability-and-slos.md`](16-observability-and-slos.md) — nested agent spans
- [`21-multi-agent.md`](21-multi-agent.md) — (this doc)

---

## Scope

### In scope

- Supervisor + specialist agent model
- Delegation triggers and handoff protocol
- Token/turn budgets per agent
- Failure and recovery strategies
- Tool and memory scoping per agent
- User visibility options
- Observability requirements

### Out of scope (unchanged by multi-agent)

- Public Chatbot REST/SSE API
- MCP wire protocol
- Channel adapter interfaces
- Embed SDK

---

## Architecture

```
User message
     │
     ▼
┌─────────────┐
│ Supervisor  │  decomposes intent, delegates, merges
└──────┬──────┘
       │
   ┌───┴───┬───────────┐
   ▼       ▼           ▼
┌──────┐ ┌──────┐ ┌──────┐
│Research│ │Actions│ │Review│  specialist agents
└───┬──┘ └───┬──┘ └───┬──┘
    │        │        │
    └────────┴────────┘
              │
              ▼
        MCP Client (shared)
              │
              ▼
        MCP Servers
```

Supervisor replaces the inner loop of single-agent ORCH; outer turn envelope (turnId, SSE events) unchanged.

---

## Agent Definitions

### Supervisor

| Property | Value |
|----------|-------|
| Role | Plan, delegate, synthesize final user reply |
| Tools | Optional meta-tools: `delegate_to_agent`, `finalize_response` |
| Model | Tenant default or `model.routing.complex_reasoning` |
| Context | Full user message, conversation summary, agent results |

### Specialist agents (configurable per tenant)

| Agent ID | Purpose | Typical tools | Model |
|----------|---------|---------------|-------|
| `research` | Search docs, KB, web | wiki.*, kb.search, read_resource | Mini |
| `actions` | CRM, billing, tickets | crm.*, billing.* | Default |
| `review` | Output validation, policy | none (guardrails + LLM check) | Mini |

Tenant config:

```json
{
  "multiAgent": {
    "enabled": false,
    "agents": [
      {
        "id": "research",
        "systemPrompt": "You retrieve and summarize information...",
        "toolFilter": { "servers": ["mcp_wiki"], "mode": "all" },
        "maxIterations": 5,
        "maxTokens": 8000
      },
      {
        "id": "actions",
        "systemPrompt": "You execute user-requested actions...",
        "toolFilter": { "servers": ["mcp_crm", "mcp_billing"], "mode": "all" },
        "maxIterations": 8,
        "requiresApprovalTools": true
      }
    ],
    "supervisor": {
      "maxDelegations": 5,
      "maxTotalTokens": 50000,
      "parallelAgents": 2
    },
    "visibility": "collapsed"
  }
}
```

---

## Delegation Triggers

Supervisor decides single-agent vs multi-agent per turn.

### Single-agent (default path)

Use existing ORCH loop when:

- Simple Q&A, no tools needed
- ≤ 2 tools from same domain
- Tenant `multiAgent.enabled = false`
- User message below complexity threshold (heuristic or classifier)

### Multi-agent path

Delegate when:

- Message spans **multiple domains** (research + action): "Find invoice 123 and email it to my manager"
- Explicit tenant routing rules match intent
- Single-agent loop exceeded **3 tool iterations** without resolution (escalation to supervisor re-plan)
- `review` agent required by policy for outbound customer-facing content

### Delegation plan (structured)

Supervisor emits plan before execution:

```json
{
  "planId": "plan_01H...",
  "steps": [
    { "agentId": "research", "task": "Retrieve invoice inv_123 details", "dependsOn": [] },
    { "agentId": "actions", "task": "Send email with invoice PDF to manager", "dependsOn": ["research"] }
  ]
}
```

Logged and visible in debug mode; optional collapsed UI for users.

---

## Handoff Protocol

Agents communicate via **structured handoff messages**, not full chat history.

### AgentTask (supervisor → specialist)

```json
{
  "taskId": "task_01H...",
  "agentId": "research",
  "instruction": "Retrieve invoice inv_123 including payment status and PDF link",
  "contextRefs": {
    "userMessage": "msg_01H...",
    "entityIds": { "invoiceId": "inv_123" }
  },
  "constraints": {
    "maxToolCalls": 3,
    "timeoutMs": 30000
  }
}
```

### AgentResult (specialist → supervisor)

```json
{
  "taskId": "task_01H...",
  "agentId": "research",
  "status": "success",
  "summary": "Invoice inv_123 is paid; PDF at att_01H...",
  "artifacts": {
    "attachmentIds": ["att_01H..."],
    "facts": [{ "key": "invoice.status", "value": "paid" }]
  },
  "usage": { "inputTokens": 800, "outputTokens": 200, "toolCalls": 1 }
}
```

Specialists do not stream to user directly (except debug visibility mode).

---

## Execution Graph

```
1. Supervisor creates plan
2. Execute independent steps in parallel (up to parallelAgents)
3. Dependent steps wait for upstream AgentResult
4. Failed step → supervisor recovery (see Failures)
5. All steps complete → supervisor synthesizes final answer
6. Optional: review agent validates final draft
7. Stream final text to user (same SSE events as today)
```

Internal events (optional, debug):

- `agent.started`, `agent.completed`, `agent.failed` — extensions to SSE schema; clients ignore if unknown.

---

## Budgets

| Limit | Default | Scope |
|-------|---------|-------|
| `maxDelegations` | 5 | Per turn |
| `maxTotalTokens` | 50,000 | Sum all agents + supervisor |
| `maxIterations` per agent | 5–8 | Per specialist |
| `parallelAgents` | 2 | Concurrent specialists |
| Wall-clock | 180 s | Entire multi-agent turn |

Exceeded budget → supervisor produces best-effort partial answer + explanation.

Counts toward tenant monthly token quota same as single-agent.

---

## Tool Ownership

| Rule | Description |
|------|-------------|
| Registry | Same tenant MCP registry (`10-mcp-integration.md`) |
| Scoping | Each agent has `toolFilter`; intersection with user scopes still applies |
| No cross-agent tool steal | Agent only invokes tools in its filter |
| Supervisor meta-tools | Not MCP tools; internal ORCH functions |
| HITL | `actions` agent destructive tools still trigger `approval.required` |

---

## Memory

| Memory type | Scope |
|-------------|-------|
| Graph (preferences, facts) | **Shared** across agents |
| Vector episodic | **Shared**; metadata includes `agentId` optional |
| Agent working notes | **Ephemeral** per task; not persisted unless supervisor promotes to shared memory |
| Post-turn extraction | Runs once on final user-visible exchange; attributes source agent in metadata |

Specialists receive minimal memory context assembled for their task, not full conversation unless needed.

---

## Failure Handling

| Failure | Supervisor strategy |
|---------|---------------------|
| Specialist timeout | Retry once; else skip step with `partial: true` in plan |
| Specialist tool error | Include error in synthesis; ask user if critical |
| Research empty | Abort dependent action steps; explain missing info |
| Review rejects output | Supervisor revises or asks user clarifying question |
| All specialists fail | Fall back to single-agent ORCH if enabled; else apologetic failure |
| Budget exceeded | Stop delegation; summarize partial results |

User sees coherent single reply; internal failures in tool cards only if `visibility: verbose`.

---

## User Visibility

| Mode | UX |
|------|-----|
| `collapsed` (default) | User sees normal streaming reply; optional "Worked with 2 agents" footnote |
| `verbose` | Show agent steps as collapsible cards (like tool cards) |
| `hidden` | Indistinguishable from single-agent |

Chat app setting or tenant default. Embed/channel adapters use collapsed unless host enables verbose via config.

---

## Observability

Trace structure:

```
turn (root)
└── supervisor.plan
    ├── agent.research (child span)
    │   ├── model.completion
    │   └── mcp.invoke
    ├── agent.actions (child span)
    └── agent.review (child span)
```

Metrics:

- `agent_delegation_count` per turn
- `agent_success_rate` by agentId
- Token cost split supervisor vs specialists

---

## Migration Path

1. **v1:** Single-agent ORCH only; this spec dormant
2. **v1.5:** Beta flag; research + actions only; collapsed visibility
3. **v2:** Review agent, verbose mode, parallel graph execution
4. No API version bump if SSE events remain backward compatible

---

## Open Questions

1. Dynamic agent spawning (not pre-configured specialists) — research topic.
2. User-selectable agent personas — product UX.
3. Inter-agent negotiation (multiple rounds) — budget explosion risk; cap rounds at 2.
