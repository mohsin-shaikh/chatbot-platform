# Billing & Metering

**Status:** Draft  
**Priority:** P3  
**Closes gaps:** Architecture gaps §18 (Billing & Commercial)

---

## Purpose

Define **billable units**, **metering pipeline**, **plans and entitlements**, **quota enforcement**, and **invoicing integration**. Connects observability cost data to commercial limits and revenue.

**Related docs:**

- [`04-multi-tenancy-and-config.md`](04-multi-tenancy-and-config.md) — quotas, plans, enforcement
- [`16-observability-and-slos.md`](16-observability-and-slos.md) — per-turn cost records, rollups
- [`06-model-provider.md`](06-model-provider.md) — token metering
- [`19-admin-and-operations.md`](19-admin-and-operations.md) — billing UI in admin console
- [`15-security-and-compliance.md`](15-security-and-compliance.md) — invoice PII handling

---

## Scope

### In scope

- Billable event types
- Metering ingestion and rollups
- Plan tiers and entitlements
- Quota enforcement modes
- Stripe integration (or export)
- Usage API for tenants

### Out of scope

- Tax calculation (Stripe Tax or vendor)
- Enterprise custom contract negotiation workflow

---

## Billable Units

| Unit | Event source | Notes |
|------|--------------|-------|
| **Message** | User message accepted (turn started) | 1 per user message, not per assistant reply |
| **Input tokens** | Model completion | Sum per turn |
| **Output tokens** | Model completion | Sum per turn |
| **Tool call** | MCP invocation success or attempt | Configurable: bill attempts vs successes only |
| **Embedding tokens** | KB ingest, memory embed | Separate line item |
| **Storage (GB-month)** | Attachments + KB blobs | Daily snapshot |
| **Seat** | Active user with login in period | Monthly |
| **Premium model surcharge** | Model in premium tier list | Multiplier on tokens |

### Non-billable

- Failed turns before model call (guardrail block at input)
- Health checks, synthetic probes
- Admin replay (inspect mode)

---

## Metering Pipeline

```
Service emits usage event (Kafka / internal bus)
  → Metering aggregator (idempotent by eventId)
  → Real-time counter (Redis) for quota enforcement
  → Hourly rollup → usage_daily table
  → Daily export → billing provider (Stripe Usage Records)
```

### Usage event schema

```json
{
  "eventId": "usg_01H...",
  "tenantId": "ten_01H...",
  "userId": "usr_01H...",
  "timestamp": "2026-06-26T10:00:00Z",
  "type": "tokens",
  "quantity": 1540,
  "dimensions": {
    "direction": "input",
    "model": "gpt-4.1",
    "channel": "chat_app",
    "turnId": "turn_01H..."
  },
  "costUsd": 0.0031
}
```

Idempotency: `(eventId)` unique; retries safe.

### Rollups

| Table | Granularity | Use |
|-------|-------------|-----|
| `usage_hourly` | tenant × metric × hour | Dashboards, alerts |
| `usage_daily` | tenant × metric × day | Invoicing |
| `usage_monthly` | tenant × metric × month | Plan limits |

---

## Plans

### Standard tiers

| Feature | Free | Pro | Enterprise |
|---------|------|-----|------------|
| **Price** | $0 | $99/mo + usage | Custom |
| **Messages/month** | 1,000 | 50,000 | Custom |
| **Tokens/month** | 500K | 10M | Custom |
| **Seats** | 3 | 25 | Custom |
| **Storage** | 1 GB | 50 GB | Custom |
| **MCP servers** | 1 | 10 | Unlimited |
| **KB documents** | 50 | 5,000 | Custom |
| **SSO** | No | Yes | Yes |
| **Audit log export** | No | Yes | Yes |
| **Support SLA** | Community | Email 48 h | Dedicated |
| **Overage** | Hard block | Soft + overage billing | Contract |

Plan stored on `Tenant.plan`; entitlements derived at request time.

### Overage pricing (Pro)

| Unit | Overage rate |
|------|--------------|
| Messages | $0.01 each |
| Tokens (1K) | $0.002 input / $0.006 output |
| Storage (GB-mo) | $0.10 |
| Extra seat | $15/mo |

---

## Quota Enforcement

### Check points

| When | Check |
|------|-------|
| Message POST | messages/month, concurrent turns, tokens/month (soft) |
| Model call (ORCH) | tokens/month hard cap |
| Attachment upload | storage quota |
| KB upload | document count, storage |
| User invite | seat count |

### Enforcement modes

| Mode | Behavior | Default |
|------|----------|---------|
| **hard_block** | `429 quota_exceeded`; no processing | Free |
| **soft_warn** | Allow + email admin at 80%, 100% | — |
| **overage_bill** | Allow + meter overage | Pro |

Tenant response:

```json
{
  "error": {
    "code": "quota_exceeded",
    "message": "Monthly message limit reached. Upgrade or wait until July 1.",
    "details": {
      "quota": "messages_per_month",
      "limit": 1000,
      "used": 1000,
      "resetsAt": "2026-07-01T00:00:00Z"
    }
  }
}
```

### Grace period

New tenants: 7-day Pro trial (optional marketing). Downgrade to Free on expiry if no payment method.

---

## Stripe Integration

### Objects

| Stripe object | Platform mapping |
|---------------|------------------|
| Customer | `Tenant.stripeCustomerId` |
| Subscription | Plan (Pro) |
| Usage Records | Hourly token/message overage |
| Invoice | Monthly consolidated |

### Flow

```
Signup Pro → create Stripe Customer + Subscription
  → webhook: checkout.session.completed → tenant.plan = pro
  → hourly: report usage records for metered line items
  → invoice.paid / payment_failed webhooks → update tenant status
```

### Webhooks handled

- `customer.subscription.updated`
- `customer.subscription.deleted`
- `invoice.paid`
- `invoice.payment_failed` → tenant `suspended` after 7 d grace

### Enterprise

Manual invoicing or annual contract; `plan: enterprise` set by platform ops without Stripe subscription.

---

## Usage API

### Tenant admin

```
GET /v1/admin/usage?period=2026-06
```

```json
{
  "period": "2026-06",
  "plan": "pro",
  "usage": {
    "messages": { "used": 12400, "limit": 50000 },
    "tokens": { "input": 4200000, "output": 890000, "limit": 10000000 },
    "toolCalls": { "used": 3400 },
    "storageBytes": { "used": 2147483648, "limit": 53687091200 },
    "seats": { "used": 12, "limit": 25 }
  },
  "estimatedCostUsd": 142.50,
  "overageCostUsd": 12.30
}
```

```
GET /v1/admin/usage/export?period=2026-06&format=csv
```

---

## Cost Estimation

Real-time estimate for admin dashboard:

```
estimatedCost = sum(tokenEvents × rateCard) + fixedPlanFee + seatFee + storageFee
```

Rate card versioned; stored in platform config. Matches `16-observability-and-slos.md` per-turn costs.

---

## Dunning & Suspension

| Day | Action |
|-----|--------|
| 0 | Payment failed → email owner |
| 3 | Retry charge + reminder |
| 7 | `tenant.status = suspended`; API 403 except billing portal |
| 30 | Data retained; downgrade to delete queue if unresolved |

Reactivation on successful payment → immediate `active`.

---

## Reporting

| Report | Audience |
|--------|----------|
| Monthly invoice | Tenant owner |
| Usage trend | Tenant admin dashboard |
| Revenue by plan | Platform finance |
| Top tenants by usage | Platform ops |

---

## Compliance

- Invoices: minimal PII (company name, billing email)
- Usage records: no message content
- SOC2: access controls on billing admin functions

---

## Open Questions

1. Credits / prepaid packs — P2 product option.
2. Reseller / MSP billing (parent tenant) — P3.
3. Crypto payment — not planned.
