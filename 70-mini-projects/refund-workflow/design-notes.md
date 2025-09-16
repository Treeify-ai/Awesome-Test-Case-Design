# Design Notes

> “Refunds must be **predictable**, **auditable**, and **exactly-once**.”  
> These notes capture architecture, contracts, failure modes, and test hooks for **Refund Workflow v1**. Pair with the **Brief** for outcomes and acceptance.

---

## 1) Goals & non-goals

**Goals**
- Deterministic, auditable **state machine** for refunds (full/partial).
- **Exactly-once** provider submission and ledger posting.
- Strong **observability** and **reconciliation** with payment provider.
- Safe under retries, outages, and abuse attempts.

**Non-goals (v1)**
- Split-tender, multi-currency conversions, RMA logistics.
- Chargeback dispute automation (separate process).
- Subscription proration.

---

## 2) Architecture (logical)

```
[UI / Agent Console]
        |
        v
[Refund API]  --(RPC)-->  [Policy Service]
     |                    [Ledger Service]
     |                    [Provider Adapter(s)]
     |                    [Attachments Store]
     |                           |
     |                           +-- PSP(s) / Bank rails
     |
     +--(DB)-----> Refunds (state), Idempotency store, Outbox/Inbox
     +--(Cache)--> Short-lived reads (refund status)
     +--(Events)--> Event Bus (refund.*)
```

**Patterns**
- **Outbox**: queue provider submissions transactionally with refund state.
- **Inbox**: dedupe provider webhooks by `provider_refund_id` + hash.
- **Saga**: refund approval → submission → pending → completion/failure.

---

## 3) State machine (detailed)

`requested → approved → submitting → provider_pending → completed | failed | canceled`

**Transitions**
- `requested → approved` (auto or agent decision).
- `approved → submitting` (enqueue job; mark attempt).
- `submitting → provider_pending` (accepted by provider, async outcome).
- `provider_pending → completed/failed` (webhook or poll).
- `requested|approved → canceled` (if allowed before submit).

**Rules**
- Only **approved** refunds may be submitted.
- Amount cannot exceed **remaining refundable**.
- State transitions are **idempotent**; repeated webhooks are safe.

---

## 4) Data model (persistence)

```sql
CREATE TABLE refunds (
  refund_id TEXT PRIMARY KEY,
  order_id TEXT NOT NULL,
  user_id TEXT NOT NULL,
  amount_minor INTEGER NOT NULL,
  currency TEXT NOT NULL,
  reason TEXT NOT NULL,                         -- taxonomy in brief
  state TEXT NOT NULL CHECK (state IN (
    'requested','approved','submitting','provider_pending',
    'completed','failed','canceled'
  )),
  provider_refund_id TEXT,
  provider_attempts INTEGER NOT NULL DEFAULT 0,
  last_error_code TEXT,
  last_error_at TIMESTAMPTZ,
  idempotency_key TEXT,                         -- for create
  created_at TIMESTAMPTZ NOT NULL,
  updated_at TIMESTAMPTZ NOT NULL
);
CREATE UNIQUE INDEX refunds_order_remaining_guard
  ON refunds(order_id, refund_id);              -- combined with ledger guard

CREATE TABLE idempotency_keys (
  key TEXT PRIMARY KEY,
  request_hash TEXT NOT NULL,
  response_json JSONB NOT NULL,
  created_at TIMESTAMPTZ NOT NULL
);

CREATE TABLE inbox_webhooks (
  provider TEXT NOT NULL,
  event_id TEXT NOT NULL,
  hash TEXT NOT NULL,
  received_at TIMESTAMPTZ NOT NULL,
  PRIMARY KEY(provider, event_id)
);
```

**Ledger**
- Post **REFUND_PENDING** on approval; **REFUND_SETTLED** on completion.
- Guard remaining balance with **sum of completed + approved** ≤ captured.

---

## 5) Contracts (API)

### Create refund (idempotent)
`POST /v1/orders/{order_id}/refunds`

Headers: `Idempotency-Key`, `X-Correlation-Id`  
Body (simplified):
```json
{ "amount_minor": 2500, "currency": "USD", "reason": "not_received", "evidence": [] }
```
Success `202`:
```json
{ "refund_id":"rf_123","state":"approved","message_id":"refund.request.accepted" }
```
Errors:
- `400 ERR.VALIDATION.amount.range`
- `400 ERR.BUSINESS.refund.exceeds_remaining`
- `402 ERR.BUSINESS.refund.not_captured`
- `403 ERR.AUTHZ.scope`
- `409 ERR.CONFLICT.idempotency`

### Agent decision
`POST /v1/refunds/{refund_id}/decision { "decision":"approve|deny", "note":"..." }`

### Submit refund (system)
`POST /v1/refunds/{refund_id}/submit` → enqueues provider job; returns `202`.

### Read/list
`GET /v1/refunds/{refund_id}`  
`GET /v1/orders/{order_id}/refunds`

### Webhooks (provider)
`POST /webhooks/payments` with `refund.succeeded|refund.failed` (signed + timestamp).

---

## 6) Idempotency & exactly-once

- **Create**: store request hash + response; on replay with same key/identical hash → return stored response; with different hash → `409`.
- **Submit**: generate **provider idempotency key** per provider contract; store `provider_refund_id` once; prevent duplicate posts.
- **Webhooks**: dedupe by `(provider,event_id)`; if already applied → **no-op**.

---

## 7) Provider adapters

- One adapter per PSP with uniform interface:
  - `submit_refund(order_ref, amount_minor, currency, key) -> { provider_refund_id, state }`
  - `verify_webhook(headers, body) -> { event_id, type, provider_refund_id, status }`
- Map PSP-specific states to our **state machine**.
- Handle **clock skew** and **signature** validation.

---

## 8) Reconciliation

- Daily job joins **ledger entries** with **provider refunds** by `provider_refund_id`.  
- Emit `refund_mismatch_rate_pct`.  
- Export CSV of mismatches with reasons (missing webhook, amount mismatch, currency mismatch).  
- Alert when mismatch exceeds threshold.

---

## 9) Security, privacy, abuse

- RBAC: only agents with scope can approve/deny; dual-control for **goodwill > threshold**.  
- Rate-limit refund create & decision endpoints.  
- No PAN or sensitive PII in logs; attachment storage **encrypted**; signed URLs with short TTL.  
- Prevent **over-refund** by transactional check against captured minus prior refunds.

---

## 10) Observability (signals)

**Logs (msgid)**
- `MSG.refund.requested`
- `MSG.refund.approved`
- `MSG.refund.submitted`
- `MSG.refund.completed`
- `MSG.refund.failed`

**Error codes**
- `ERR.BUSINESS.refund.not_captured`
- `ERR.BUSINESS.refund.exceeds_remaining`
- `ERR.CONFLICT.idempotency`
- `ERR.DEPENDENCY.timeout`

**Metrics**
- `refund_create_latency_ms` (histogram)
- `refund_outcome_total{state}`
- `refund_mismatch_rate_pct`
- `webhook_replay_dedup_total`

**Traces**
- Spans: `policy.evaluate`, `ledger.post`, `provider.submit`, `webhook.process`.
- Attach exemplars to histograms.

---

## 11) Failure modes & mitigations

- **Provider timeout/5xx** → mark `provider_pending`; retry with backoff + jitter; surface ETA to user.  
- **Webhook lost** → poll status via adapter; reconcile job detects gap.  
- **Idempotency key collision** → `409` with prior response; never overwrite.  
- **Ledger post failure** → outbox ensures retry; state blocked from `completed` until ledger write succeeds.  
- **Partial provider outage** → circuit breaker opens; queue submissions; show “processing” state.

---

## 12) Performance & capacity

- Budgets: create p95 **≤ 250 ms**; status read p95 **≤ 150 ms**.  
- Expected throughput aligned with order volume; queue sized for burst 3×.  
- Storage: attachments limited by type/size/count; virus scan pipeline.

---

## 13) UX & a11y hooks

- Message IDs for each state; short actionable copy.  
- **ARIA live** updates on status changes; focus returns to actionable control.  
- Emails/notifications localized; timelines vary by rail (card vs bank).  
- Dark mode/RTL verified; long text expansion ×1.3.

---

## 14) Testing strategy

**Unit**: amounts, remaining logic, state transitions, idempotency hashing.  
**Contract**: OpenAPI schemas; reject **unknown fields**.  
**Integration**: provider simulators (succeeded/failed/timeout); signature checks.  
**E2E**: MAE scenarios from the brief with evidence bundle.  
**Chaos**: provider timeouts and webhook delays.  
**Telemetry assertions**: msgid/err.code in logs; metrics changes; traces present.

---

## 15) Open questions

- Dual-control trigger thresholds by currency—static or FX-adjusted?  
- Allow **refund to different instrument** (out of scope v1 but design migration path)?  
- SLA visualization to customer—show **expected date** by rail?

---

## 16) API schema stub (OpenAPI fragment)

```yaml
post:
  /v1/orders/{order_id}/refunds:
    headers:
      Idempotency-Key: string
      X-Correlation-Id: string
    body:
      type: object
      required: [amount_minor, currency, reason]
      properties:
        amount_minor: { type: integer, minimum: 1 }
        currency: { type: string, pattern: "^[A-Z]{3}$" }
        reason: { $ref: "#/components/schemas/RefundReason" }
        evidence: { type: array, items: { $ref: "#/components/schemas/Attachment" } }
    responses:
      "202": { $ref: "#/components/schemas/Refund" }
      "400": { $ref: "#/components/schemas/Error" }
      "402": { $ref: "#/components/schemas/Error" }
      "403": { $ref: "#/components/schemas/Error" }
      "409": { $ref: "#/components/schemas/Error" }
```

---

## 17) Links

- Brief → `./brief.md`  
- API Coverage → `../../60-checklists/api-coverage.md`  
- Performance → `../../60-checklists/performance-review.md`  
- Security → `../../60-checklists/security-review.md`  
- SRE (SLOs & breakers) → `../../57-cross-discipline-bridges/for-sres.md`  
- Traceability → `../../65-review-gates-metrics-traceability/traceability.md`
