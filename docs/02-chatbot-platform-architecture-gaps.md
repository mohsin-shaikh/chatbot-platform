# Chatbot Platform Architecture — Gap Analysis

This document lists what is **missing or underspecified** in [`chatbot-platform-architecture.md`](chatbot-platform-architecture.md). It is a companion checklist for turning the high-level architecture into an implementable platform spec.

---

## Summary

The architecture doc defines the **right shape**: surfaces → chatbot → ORCH → MCP client → MCP servers, with memory, guardrails, context assembly, and observability called out. What it lacks is most of the **operational, contractual, and data-layer detail** needed to build, integrate, secure, and run the platform in production.

| Area | Status in architecture doc |
|------|----------------------------|
| Layer topology & turn loop | Covered |
| Integration surfaces (conceptual) | Covered |
| Memory, guardrails, context, observability (conceptual) | Covered |
| Multi-agent (future) | Covered at vision level |
| API contracts | Missing |
| Data models & persistence | Missing |
| Auth & identity (detailed) | Underspecified |
| Multi-tenancy & config | Underspecified |
| Model layer | Missing as a first-class component |
| Knowledge ingestion (RAG) | Underspecified |
| Human-in-the-loop & long-running work | Missing |
| Reliability, concurrency, cancellation | Underspecified |
| Compliance & data lifecycle | Missing |
| Admin, onboarding, developer experience | Missing |
| Non-functional requirements & SLOs | Missing |

---

## 1. Platform API & Event Contracts

The doc describes layers but not the **interfaces between them** or the **public API** integrators call.

**Missing:**

- **Chatbot public API** — REST (and/or GraphQL) for threads, messages, memory, settings; versioning strategy (`/v1/...`).
- **Streaming protocol** — SSE or WebSocket event schema: `text_delta`, `tool_call_start`, `tool_call_result`, `guardrail_block`, `turn_complete`, `error`; client reconnection and `Last-Event-ID` behavior.
- **Internal contracts** — request/response shapes for Chatbot → ORCH (`TurnRequest`, `TurnStream`) and ORCH → MCP Client (`ToolInvocation`, `ToolResult`).
- **Webhook outbound** — events for embed hosts and messaging platforms (message completed, tool executed, escalation needed).
- **Idempotency** — `Idempotency-Key` on message POST to prevent duplicate turns on retry.
- **OpenAPI / AsyncAPI** — machine-readable specs for SDK generation and partner onboarding.

**Why it matters:** Without this, each surface (chat app, widget, Slack) will invent its own protocol and drift.

---

## 2. Conversation & Message Data Model

"Session" and "conversation threads" are mentioned but never defined.

**Missing:**

- **Entities** — `Tenant`, `User`, `Conversation`, `Message`, `Turn`, `ToolCall`, `Attachment`, `MemoryRecord`.
- **Message structure** — roles (user/assistant/system/tool), content parts (text, image, file ref), metadata (model, latency, token count).
- **Thread lifecycle** — create, list, archive, delete, rename, fork/branch from a message.
- **Turn vs message** — one user message may produce multiple assistant messages (tool loops); how IDs and ordering work.
- **Edits & regeneration** — edit user message, regenerate assistant reply; impact on memory and context window.
- **Cross-channel identity** — same `Conversation` reachable from chat app and Slack, or separate per channel?
- **Storage split** — hot conversation store (Postgres/Dynamo) vs long-term memory (graph + vector); retention per tier.

**Why it matters:** Memory, observability, and replay all depend on a stable conversation model.

---

## 3. Authentication & Identity

Auth is listed as "user accounts, SSO, or API keys" and embed "short-lived tokens" without detail.

**Missing:**

- **Identity model** — platform user vs external user (`external_user_id` from host app); anonymous/guest sessions.
- **Embed token exchange** — host backend signs JWT (shared secret or JWKS); payload fields, TTL, refresh; widget never holds host session cookie.
- **Channel identity linking** — map Slack `U123` / Teams AAD ID / WhatsApp phone to platform `User`; OAuth install flows per platform.
- **Authorization (authZ)** — RBAC: who can use which tools, see which conversations, manage tenant config.
- **Service-to-service auth** — ORCH → MCP servers, MCP client credential vault, mTLS or signed requests.
- **Session fixation / token theft** — binding widget token to origin, rotation, revoke on logout.

**Why it matters:** Embed and multi-channel are the hardest integration paths; the architecture doc stops at "auth handoff."

---

## 4. Multi-Tenancy & Configuration

`tenantId` appears in embed contract and observability but tenancy is not a designed subsystem.

**Missing:**

- **Tenant isolation** — data partition strategy (schema per tenant vs row-level `tenant_id`); noisy-neighbor limits.
- **Tenant config store** — system prompt, guardrail rules, allowed MCP servers, model defaults, branding, feature flags.
- **Per-tenant MCP registry** — which servers/tools are enabled; sandbox vs production tool sets.
- **Quotas & entitlements** — messages/day, tokens/month, max concurrent turns, storage limits.
- **Onboarding flow** — create tenant, issue API keys, register embed origins, connect Slack workspace.

**Why it matters:** A platform serving many customers needs tenancy as a first-class design, not a field in a JWT.

---

## 5. Model Layer (First-Class Component)

The sequence diagram includes **Model** but the architecture stack diagram and layer table omit it.

**Missing:**

- **Provider abstraction** — OpenAI, Anthropic, Azure, self-hosted; unified completion + tool-call interface.
- **Routing** — default model per tenant/channel; fallback on provider outage or rate limit.
- **Parameters** — temperature, max tokens, stop sequences; per-turn overrides vs tenant defaults.
- **Cost control** — token budget per turn/tenant; truncate context before call vs fail turn.
- **Structured output** — when ORCH needs JSON (memory extraction, guardrail classifier) vs free text.
- **Multimodal** — vision/audio models; which surfaces support attachments.

**Why it matters:** ORCH cannot be implemented without explicit model integration design.

---

## 6. Knowledge Base & Document Ingestion

Vector memory is described for "document chunks and knowledge base content" but there is no **ingestion path**.

**Missing:**

- **Upload API** — files, URLs, connectors (Google Drive, Confluence, S3).
- **Processing pipeline** — parse, chunk, embed, index; async job status and errors.
- **Scoped KBs** — tenant-wide vs user-private vs conversation-attached documents.
- **Freshness** — re-index on source change, tombstone deleted docs in vector store.
- **Citation** — return source references in assistant replies for trust and compliance.
- **Separation from episodic memory** — KB chunks vs "what we said last Tuesday" retrieval rules.

**Why it matters:** RAG is a product feature, not just a vector DB in the deployment diagram.

---

## 7. Attachments & Rich Media

The chat app section mentions file uploads in passing; nothing else does.

**Missing:**

- **Storage** — object store, virus scan, size/type allowlist, TTL for ephemeral uploads.
- **Pipeline** — image → vision model; PDF → text extraction → chunk; audio → transcription.
- **Channel limits** — WhatsApp media caps vs chat app large files; adapter normalization.
- **PII in files** — guardrail scan on extracted text before model or memory write.

---

## 8. Human-in-the-Loop & Long-Running Work

Pre-tool guardrails mention "destructive-action confirmation" but there is no **interrupt/resume** design.

**Missing:**

- **Approval flows** — pause turn, surface "Confirm charge $500?" in UI, resume on approve/deny.
- **Pending action store** — durable state for multi-hour approvals (Slack button, email link).
- **Async turns** — queue heavy work (large report, batch MCP calls); notify user when done (proactive message).
- **Escalation to human agent** — hand off to support queue with transcript export.

**Why it matters:** Enterprise and messaging channels require pausable, resumable turns—not only synchronous streaming.

---

## 9. Turn Execution — Edge Cases & Reliability

The turn lifecycle is the happy path only.

**Missing:**

- **Cancellation** — user clicks Stop; abort model stream and in-flight MCP calls.
- **Concurrency** — two messages in same thread; queue vs reject vs merge.
- **Partial failure** — one MCP tool fails, others succeed; retry policy per tool.
- **Timeouts** — turn-level SLA; degrade to "I couldn't finish" with partial results.
- **Parallel tool calls** — when model emits multiple tools; execution order and result aggregation.
- **Tool output limits** — truncate/summarize large JSON before next model call.
- **Idempotent tools** — dedupe duplicate invocations within a turn.

---

## 10. MCP — Deeper Specification

MCP client/server are outlined; several MCP capabilities are unused in the turn loop diagram.

**Missing:**

- **Resources** — when ORCH reads MCP resources vs calling tools; cache TTL.
- **Prompts** — MCP server-provided prompt templates; how they merge with tenant system prompt.
- **Transport & deployment** — stdio vs SSE vs streamable HTTP; where MCP client runs (sidecar vs embedded in ORCH).
- **Tool schema lifecycle** — discovery refresh, breaking schema changes, version pinning per tenant.
- **Authorization context** — exact claims passed to each MCP server (user, tenant, scopes).
- **Sandbox** — dry-run or mock MCP for integrator testing.

---

## 11. Memory — Additional Design Decisions

Graph + vector split is clear; operational rules are not.

**Missing:**

- **Scopes** — user memory vs team vs org-shared graph; who can read/write/delete.
- **Conflict resolution** — user says "I prefer email" then "use Slack"; last-write vs explicit override.
- **Decay & TTL** — episodic vector entries expire; preferences persist until deleted.
- **Right to forget** — delete user cascades graph + vector + conversations; audit log retention.
- **Quality** — false extractions, user correction UI ("that's wrong, forget it").
- **Evaluation** — metrics for retrieval precision/recall on benchmark queries.

---

## 12. Guardrails — Additional Design Decisions

Checkpoints are listed; operating the system is not.

**Missing:**

- **Rule authoring** — who writes rules (tenant admin vs platform); UI vs YAML/API.
- **Versioning & rollout** — staging rules, canary tenant, rollback.
- **LLM-judge cost/latency** — when to use vs keyword/regex only.
- **False positive handling** — user appeal, temporary override, support tooling.
- **Red team / eval suite** — regression tests when rules or base model change.
- **Jailbreak catalog** — maintained threat list vs ad hoc input rules.

---

## 13. Context Assembler — Additional Design Decisions

Priority list exists; mechanics for production tuning do not.

**Missing:**

- **Budget allocation** — fixed percentages per source vs dynamic based on query type.
- **Tool selection** — intent classifier vs embed-all vs MCP server filter; max tools in context.
- **Caching** — cache assembled context for retry after tool failure without re-fetching memory.
- **Debug mode** — expose `ContextBundle` to tenant admins (with PII redaction policy).

---

## 14. Observability — Additional Design Decisions

Signals and dashboards are named; running the system is not.

**Missing:**

- **SLOs & alerting** — p99 turn latency, error rate, MCP availability; pager rules.
- **PII in telemetry** — what is logged vs hashed vs omitted; retention periods.
- **Cost attribution** — tokens and tool calls per tenant/user/conversation for chargeback.
- **Sampling** — full trace for 1% vs always-on for errors.
- **Feedback loop** — thumbs up/down tied to trace ID for quality analysis.
- **Audit log** — separate from debug logs; immutable record of tool calls and config changes (compliance).

---

## 15. Security & Compliance (Beyond the Security Table)

The security table covers layers at a glance; compliance programs need more.

**Missing:**

- **Encryption** — at rest (DB, object store) and in transit; key management (KMS).
- **Data residency** — tenant region pinning for DB, vector store, model routing.
- **Retention policies** — conversation TTL, log retention, backup retention.
- **SOC2 / GDPR / HIPAA** — which regimes are in scope; DPA, subprocessors list.
- **Secrets management** — rotation, no secrets in traces or `ContextBundle` dumps.
- **Supply chain** — MCP server allowlist, signed artifacts, dependency scanning.

---

## 16. Integration Surfaces — Per-Surface Gaps

Conceptual coverage exists; production details per surface do not.

### Chat app

- Mobile web / native apps; offline draft messages.
- Accessibility (WCAG); keyboard navigation, screen reader for streaming.
- i18n/l10n for UI; RTL layouts.
- Settings UI for memory review, connected integrations, export data.

### Embed (script / iframe)

- Widget SDK surface — `init`, `open`, `close`, `setContext`, `destroy`, event callbacks.
- Theming — CSS variables, dark mode, z-index, mobile viewport.
- Parent ↔ iframe `postMessage` protocol and origin validation.
- CSP, cookie-less operation, Safari ITP implications.
- License key vs tenant key; domain allowlist enforcement.

### Messaging platforms

- Webhook signature verification and replay protection.
- Message ordering and deduplication (platform retries).
- Typing indicators and read receipts where supported.
- Proactive / outbound messages (not only reply-to-inbound).
- App install OAuth, workspace uninstall cleanup.
- Group chat vs DM policy (who triggers the bot, @mention rules).
- Platform-specific rate limits and message length splitting.

---

## 17. Admin, Operations & Developer Experience

Not covered in the architecture doc.

**Missing:**

- **Admin console** — tenants, users, usage, guardrails, MCP connections, conversation search (support).
- **Impersonation / support mode** — read-only view of tenant config and traces (with consent).
- **CI/CD** — deploy units (chatbot API, ORCH, MCP servers independently); migration strategy.
- **Environments** — dev/staging/prod; test tenants; synthetic monitoring.
- **Disaster recovery** — RPO/RTO for conversation store and memory indexes.
- **Horizontal scaling** — stateless ORCH workers, sticky sessions for WebSocket, queue-backed turns.
- **Feature flags** — gradual rollout of models, tools, multi-agent (future).
- **Partner docs** — integration guides, sandbox API keys, status page.

---

## 18. Billing & Commercial

Observability mentions "per-tenant usage and cost" but there is no commercial model.

**Missing:**

- **Metering** — what is billable (messages, tokens, tool calls, storage, seats).
- **Plans & limits** — free tier, enforcement when quota exceeded (hard block vs soft warn).
- **Invoicing integration** — Stripe or internal billing export.

---

## 19. Testing & Quality Assurance

Not mentioned.

**Missing:**

- **Unit/integration tests** — ORCH turn loop with mocked model and MCP.
- **Eval harness** — golden conversations, tool-call accuracy, guardrail regression.
- **Load testing** — concurrent streams, MCP fan-out, memory retrieval under load.
- **Chaos** — MCP server down, model timeout, partial network partition.

---

## 20. Future Multi-Agent — Gaps Even at Vision Level

The future section is directionally correct but leaves open questions to resolve before implementation.

**Missing:**

- **Delegation triggers** — when supervisor splits vs single-agent handles.
- **Budget** — max sub-agent turns, max total tokens per user message.
- **Failure** — one specialist fails; supervisor recovery strategy.
- **User visibility** — show sub-agent steps in UI or hide behind single reply.
- **Tool ownership** — same MCP registry or per-agent allowlists enforced how.

---

## Recommended Next Documents

To close the gaps above, consider adding focused specs (order is suggestive):

| Priority | Document | Closes gaps |
|----------|----------|-------------|
| P0 | **API & streaming events** | §1, chat app/embed integration |
| P0 | **Data model & persistence** | §2, memory, observability replay |
| P0 | **Auth & tenancy** | §3, §4, embed + channels |
| P1 | **ORCH turn engine spec** | §5, §8, §9, tool loop edge cases |
| P1 | **MCP integration spec** | §10, tool registry, auth context |
| P1 | **Memory & KB spec** | §6, §11, ingestion + retrieval |
| P2 | **Guardrails & compliance** | §12, §15, retention, audit |
| P2 | **Observability & SLOs** | §14, alerting, cost attribution |
| P2 | **Channel adapter guide** | §16, per-platform checklist |
| P3 | **Admin & billing** | §17, §18 |
| P3 | **Multi-agent design** | §20, when ready |

---

## Quick Checklist (for architecture doc updates)

Use this when revising `chatbot-platform-architecture.md`:

- [ ] Add **Model layer** to stack diagram and layer table
- [ ] Add **Conversation/message model** section (even if schema is appendix)
- [ ] Add **Public API + streaming events** subsection under Chatbot
- [ ] Expand **Auth** — embed JWT exchange, channel identity linking
- [ ] Add **Multi-tenancy** section (config, isolation, quotas)
- [ ] Add **Knowledge ingestion** pipeline adjacent to vector memory
- [ ] Add **Human-in-the-loop** to turn lifecycle (pause/resume)
- [ ] Add **Cancellation, concurrency, idempotency** to turn lifecycle
- [ ] Document **MCP resources & prompts** usage in ORCH
- [ ] Add **Compliance & data lifecycle** (retention, delete, residency)
- [ ] Add **Non-functional requirements** (latency targets, availability)
- [ ] Add **Admin / DX** paragraph or pointer to ops doc
