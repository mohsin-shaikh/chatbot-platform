# Security & Compliance

**Status:** Draft  
**Priority:** P2  
**Closes gaps:** Architecture gaps §15 (Security & Compliance)

---

## Purpose

Define **security controls** and **compliance requirements** for the platform: encryption, data residency, retention, audit logging, GDPR/privacy rights, secrets management, and supply chain security.

**Related docs:**

- [`03-auth-and-identity.md`](03-auth-and-identity.md) — authentication, token security
- [`04-multi-tenancy-and-config.md`](04-multi-tenancy-and-config.md) — tenant isolation, region pinning
- [`02-data-model-and-persistence.md`](02-data-model-and-persistence.md) — retention defaults, soft delete
- [`08-memory.md`](08-memory.md) — forget/delete cascades
- [`14-guardrails.md`](14-guardrails.md) — policy enforcement
- [`16-observability-and-slos.md`](16-observability-and-slos.md) — PII in telemetry, audit vs debug logs

---

## Scope

### In scope

- Encryption at rest and in transit
- Key management
- Data residency and cross-border transfer
- Retention and deletion (GDPR, right to erasure)
- Audit log (immutable)
- Compliance regimes in scope
- Secrets management
- Supply chain security
- Incident response hooks

### Out of scope

- Detailed pen-test procedures
- Legal DPA text (legal team owns templates)

---

## Compliance Regimes

| Regime | In scope (v1) | Notes |
|--------|---------------|-------|
| **GDPR** | Yes | EU tenants; DPA available |
| **SOC 2 Type II** | Target year 1 | Control mapping documented |
| **HIPAA** | Optional / BAA | Enterprise add-on; extra controls |
| **CCPA** | Yes | California residents |

Subprocessors list published (model providers, cloud, vector DB). Tenants notified 30 days before new subprocessor.

---

## Encryption

### In transit

| Path | Requirement |
|------|-------------|
| Client → API | TLS 1.2+ (TLS 1.3 preferred) |
| Service → service | TLS 1.2+ or mTLS |
| MCP Client → MCP Server | TLS required for remote transports |
| Database connections | TLS + cert validation |

HSTS enabled on all public endpoints. Certificate rotation automated (Let's Encrypt or ACM).

### At rest

| Store | Encryption |
|-------|------------|
| Relational DB (Postgres) | AES-256 (cloud provider managed or TDE) |
| Object store (S3/GCS) | SSE-KMS or SSE-S3 |
| Vector DB | Provider encryption at rest |
| Graph DB | Provider encryption at rest |
| Redis (cache, rate limits) | Encryption at rest if persistent |
| Backups | Encrypted with separate KMS key |

### Key management

- Cloud KMS (AWS KMS, GCP Cloud KMS, Azure Key Vault)
- **DEK per tenant** for conversation content (enterprise tier) — optional envelope encryption
- Key rotation: annual for CMK; automatic for provider-managed keys
- No keys in source code, env files in repos, or logs

---

## Data Residency

Each tenant has `region` at creation (`04-multi-tenancy-and-config.md`).

| Component | Pinning |
|-----------|---------|
| Conversation DB | Tenant region only |
| Object store | Tenant region bucket |
| Vector / graph indexes | Tenant region |
| Model API | Route to region-local endpoint where available; document cross-border if not |
| Logs / traces | Tenant region or EU-only global pipeline (config) |

**Cross-border:** EU tenants default to no US model routing unless explicit opt-in with SCCs documented.

---

## Retention Policies

### Defaults (overridable per tenant)

| Data class | Default retention | After expiry |
|------------|-------------------|--------------|
| Active conversations | Indefinite | — |
| Archived conversations | 1 year | Cold archive → delete |
| Soft-deleted conversations | 30 days | Hard delete |
| Debug logs | 30 days | Delete |
| Traces / metrics | 90 days | Aggregate → delete detail |
| Audit log | 7 years | Archive immutable store |
| Backups | 35 days | Rotate |
| Attachments (unlinked) | 7 days | Delete blob |

### Tenant-configurable TTL

`conversation.retentionDays` in tenant config. Compliance minimums enforced (cannot set below legal hold requirements).

### Legal hold

`legal_hold` flag on tenant or user suspends delete jobs until released. Audit logged.

---

## GDPR & Privacy Rights

### Data subject rights

| Right | Implementation |
|-------|----------------|
| **Access** | `GET /v1/users/me/export` → async JSON/ZIP job |
| **Rectification** | User profile PATCH; memory correction UI |
| **Erasure** | `DELETE /v1/users/me` → cascade delete job |
| **Portability** | Export includes conversations, memory, attachments metadata |
| **Restrict processing** | Tenant suspend or user `processing_restricted` flag |
| **Object** | Opt-out of memory extraction (tenant config) |

### Erasure cascade

```
deleteUser(userId):
  1. Revoke all tokens
  2. Soft-delete conversations → hard delete job
  3. Delete graph nodes/edges (08-memory.md)
  4. Delete vector docs
  5. Delete attachment blobs
  6. Anonymize audit log entries (replace PII with hash); retain event record
  7. Notify subprocessors where required (model providers: no stored training on API data per DPA)
```

Completion SLA: **30 days** max; target **72 hours** for automated path.

### Consent

- Memory extraction: implied for active use; explicit for sensitive categories (post_turn guardrail)
- Marketing emails: separate consent (out of chatbot scope)
- Cookie banner: chat app only if analytics cookies used

### DPA

Standard DPA template covers: processing purposes, subprocessor list, breach notification (72 h), data location.

---

## Audit Log

Separate from debug/application logs. **Append-only**, tamper-evident.

### Events recorded

| Category | Events |
|----------|--------|
| Auth | login, logout, token issued/revoked, embed exchange |
| Data access | admin viewed conversation, export requested |
| Config | tenant config change, guardrail publish, MCP server add/remove |
| Tools | tool invoked (name, status, userId — not sensitive args) |
| Guardrails | block, approval required |
| Privacy | erasure requested/completed |

### Audit entry schema

```json
{
  "id": "aud_01H...",
  "tenantId": "ten_01H...",
  "actorId": "usr_01H...",
  "actorType": "user",
  "action": "config.updated",
  "resourceType": "tenant_config",
  "resourceId": "ten_01H...",
  "metadata": { "version": 12, "changedKeys": ["model.default"] },
  "ip": "203.0.113.1",
  "timestamp": "2026-06-26T10:00:00Z",
  "integrityHash": "sha256:..."
}
```

### Access

- Tenant admin: own tenant audit log (filtered)
- Platform support: read-only with impersonation consent (`19-admin-and-operations.md`)
- Export for compliance audits: CSV/JSON API

---

## Secrets Management

| Secret type | Storage |
|-------------|---------|
| API keys (tenant) | Hashed; prefix only in UI |
| Provider API keys | Vault / KMS sealed secrets |
| MCP server credentials | Per-tenant vault path |
| Embed signing secrets | Vault; rotatable in admin |
| Webhook HMAC secrets | Encrypted column |

### Rules

- Never in git, CI logs, traces, or `ContextBundle` dumps
- Rotation: API keys manual; platform secrets quarterly
- Access: least privilege IAM; break-glass audited

---

## Supply Chain

| Control | Implementation |
|---------|----------------|
| MCP server allowlist | Tenant registry only; no arbitrary URLs |
| Container images | Signed (cosign); scan in CI (Trivy/Snyk) |
| Dependencies | Lock files; automated PR for CVEs |
| SBOM | Generated per release |
| Widget CDN | SRI hashes documented for hosts |

---

## PII Handling

| Location | Policy |
|----------|--------|
| Application logs | Redact email, phone, SSN patterns |
| Traces | Hash userId in sampled traces; full ID on error spans only |
| ContextBundle debug | Redact PII fields; admin role required |
| Model prompts | Sent to providers per DPA; no training opt-out contract required |
| Error reports | Strip message content; include traceId only |

Classification tags on fields: `pii`, `sensitive`, `public` in data model docs.

---

## Security Headers (public API)

```
Strict-Transport-Security: max-age=31536000; includeSubDomains
Content-Security-Policy: (embed page restrictive policy)
X-Content-Type-Options: nosniff
X-Frame-Options: DENY (except embed routes)
Referrer-Policy: strict-origin-when-cross-origin
```

---

## Incident Response

| Severity | Example | Response |
|----------|---------|----------|
| P1 | Data breach, key leak | 1 h internal, 72 h customer notification (GDPR) |
| P2 | Tenant cross-talk bug | Immediate patch, tenant notify |
| P3 | Single service degradation | Status page |

Runbooks in ops repo. Post-incident: root cause, audit log review, guardrail/rule updates.

---

## HIPAA (enterprise add-on)

When BAA signed:

- PHI tagging on conversations (`metadata.phi: true`)
- Stricter audit retention
- No LLM providers without BAA
- Access logging on every PHI read
- Minimum necessary in tool args

---

## Open Questions

1. Customer-managed keys (CMK/BYOK) — enterprise P2.
2. FedRAMP — out of scope v1.
3. Data residency certification per region — SOC2 evidence package.
