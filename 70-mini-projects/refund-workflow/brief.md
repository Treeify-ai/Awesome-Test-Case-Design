# Refund Workflow — Brief

> Build a **predictable, auditable, customer-friendly** refund workflow that prevents abuse, keeps accounting clean, and provides clear signals for support & finance.

---

## Problem → Value

**Problem**: Refund handling is manual and inconsistent. Agents lack guidance, customers wait, accounting reconciliation is error‑prone, and fraud risk is high.  
**Value**: Faster resolutions, fewer chargebacks, lower support load, cleaner books, and improved trust (NPS/CSAT).

---

## Outcomes (measurable)

- **Resolution SLA**: 80% of refund requests **auto‑decisioned** within **2 minutes**; 95% completed within **T+5 days** (depending on rail).  
- **Accuracy**: Refund mismatch rate (gateway vs ledger) **≤ 0.05%**.  
- **Abuse**: Duplicate/over‑refund prevention effectiveness **≥ 99.9%**.  
- **Experience**: CSAT ≥ **4.5/5** for refund interactions.  
- **Ops**: Manual review rate **≤ 10%** after 30 days.

**Guardrails**  
- No negative order balances.  
- No refunds when payment authorization voided/never captured (unless policy allows).  
- Ledger and provider totals remain in **1:1 parity** (daily).

---

## Users & Roles

- **Customer** — requests refund (full/partial) with reason & evidence.  
- **Agent** — approves/denies/adjusts; communicates result.  
- **Finance** — reconciles payouts vs refunds; closes periods.  
- **Risk** — policies, limits, fraud rules.  
- **System** — creates operations, calls PSP, posts ledger entries, emits events.

---

## Scope

**In**  
- Refund on **captured** payments (card/PayNow/Bank transfer).  
- **Full** and **partial** refunds; multiple partials up to paid total.  
- **Reasons** taxonomy (quality, not received, duplicate, pricing error, goodwill).  
- **Evidence** attachments (images/docs).  
- Agent notes + customer messaging (email/notifications).  
- **Idempotent** refund creation; **exactly‑once** provider calls.  
- State machine with **retries** and **dead‑letter**.

**Out (v1)**  
- Returns/RMA logistics workflow (linkable, not included).  
- Chargeback dispute management.  
- Multi‑currency conversion refunds (refund in original tender/currency only).  
- Split‑tender refunds.

---

## Business Rules (deterministic)

- A refund **cannot exceed** the **remaining refundable** amount = `captured_total - sum(approved/completed refunds)`.  
- If capture is **pending/failed**, the system returns `ERR.BUSINESS.refund.not_captured`.  
- **Cooling‑off windows** and **reason‑based limits** (configurable).  
- **Goodwill** refunds require elevated role or dual‑control over threshold.  
- **Ledger** posts at **creation** (pending liability) and **settlement** (cash movement).  
- **Reconciliation**: match PSP refund IDs to ledger entries daily.

---

## State Machine

`requested → approved → submitting → provider_pending → completed | failed | canceled`  
**Notes**  
- `failed` may be retried if error is retryable (5xx/timeout).  
- `canceled` only before provider submission.  
- Provider async webhooks can move `provider_pending → completed/failed`.

---

## Contracts (API)

### Create refund request (idempotent)
`POST /v1/orders/{order_id}/refunds`  
Headers: `Idempotency-Key`, `X-Correlation-Id`  
Body:
```json
{
  "amount_minor": 2500,
  "currency": "USD",
  "reason": "not_received",
  "evidence": [{"type":"image","uri":"link://..."}]
}
```
Success `202 Accepted`:
```json
{
  "refund_id": "rf_123",
  "state": "approved",
  "remaining_refundable_minor": 5000,
  "message_id": "refund.request.accepted"
}
```
Errors:
- `400 ERR.VALIDATION.amount.range`
- `400 ERR.BUSINESS.refund.exceeds_remaining`
- `409 ERR.CONFLICT.idempotency`
- `402 ERR.BUSINESS.refund.not_captured`
- `403 ERR.AUTHZ.scope`

### Approve/deny (agent)
`POST /v1/refunds/{refund_id}/decision { "decision": "approve|deny", "note": "..." }`

### Submit to provider (system/async)
`POST /v1/refunds/{refund_id}/submit` → transitions to `submitting/provider_pending`.

### Read/list
`GET /v1/orders/{order_id}/refunds`  
`GET /v1/refunds/{refund_id}`

### Webhooks (provider→platform)
`POST /webhooks/payments` with event types: `refund.succeeded`, `refund.failed`.

---

## Idempotency & retries

- **Create** requires `Idempotency-Key`; replay returns same body + `Idempotency-Status: replayed`.  
- **Provider submission** guarded by **idempotent keys** per provider; store provider `refund_id`.  
- Retry policy: **exponential backoff + jitter**, **deadline < client timeout**.  
- **Outbox** pattern for provider calls; **inbox** dedupe for webhooks (exactly‑once).

---

## Telemetry & Observability

**Logs (msgid)**:  
- `MSG.refund.requested`, `MSG.refund.approved`, `MSG.refund.submitted`, `MSG.refund.completed`, `MSG.refund.failed`.

**Metrics**:  
- `refund_create_latency_ms` (histogram)  
- `refund_outcome_total{state}`  
- `refund_mismatch_rate_pct` (ledger vs provider)  
- `refund_auto_decision_rate_pct`  
- `webhook_replay_dedup_total`  

**Traces**: spans for `policy.evaluate`, `ledger.post`, `provider.submit`.

**Events**: `refund.created`, `refund.approved`, `refund.completed`, `refund.failed` (include amounts, currency, reason code, **no PII**).

---

## Security, Abuse & Privacy

- Prevent **over‑refund**/**duplicate** via transactional checks and unique constraints.  
- Role‑based access; dual‑control for high‑value refunds; audit trail (who/when/why).  
- No sensitive PII in logs; redact card data; encrypt attachments at rest.  
- Rate‑limit refund creation & decision endpoints; suspicious velocity flagged.

---

## i18n & UX

- Statuses have **message IDs**; short actionable copy.  
- Customer flow shows: **requested → processing → refunded** with expected timelines by payment rail.  
- **A11y**: focus management on status change; SR announcements; alt text for attachments.  
- **RTL** and long‑string expansion ×1.3 verified.

**Sample message IDs**  
- `refund.request.accepted` — “We’re processing your refund.”  
- `refund.completed` — “Your refund is complete.”  
- `refund.failed` — “We couldn’t complete your refund.”

---

## Performance & Resiliency

- p95 budgets: `refund create ≤ 250 ms`, `status read ≤ 150 ms`.  
- Provider calls isolated behind **circuit breakers**; queue with DLQ for retries.  
- Outage mode: accept requests, **defer submission**, notify users, keep SLA dashboard.

---

## Data Model (simplified)

```yaml
Refund:
  refund_id: string
  order_id: string
  user_id: string
  amount_minor: integer
  currency: string
  reason: enum[not_received,quality,duplicate,pricing_error,goodwill,other]
  evidence: array[Attachment]
  state: enum[requested,approved,submitting,provider_pending,completed,failed,canceled]
  provider_refund_id: string?
  ledger_entry_id: string?
  created_at: datetime
  updated_at: datetime
```

**Ledger entries**  
- `REFUND_PENDING` (liability) at approval.  
- `REFUND_SETTLED` (cash movement) at completion.

---

## Acceptance (MAE) — testable & evidence

### MAIN — Full refund on captured payment
- **Given** order captured `$100.00`  
- **When** request `$100.00` refund  
- **Then** auto‑approved → submitted → completed; remaining refundable `0`  
- **Evidence**: API `202`, state transitions, provider `refund_id`, ledger entries

### ALT — Partial refund then second partial
- **Given** captured `$100.00`  
- **When** refund `$30.00`, then `$20.00`  
- **Then** remaining refundable `$50.00`; both complete; no over‑refund  
- **Evidence**: readbacks; ledger parity

### ALT — Manual review (goodwill)
- **Given** policy requires agent approval  
- **Then** agent approves with note; audit trail recorded

### EXCEPTION — Exceeds remaining
- **Then** `400 ERR.BUSINESS.refund.exceeds_remaining`  
- **Evidence**: error JSON; log `MSG.refund.requested` with denial reason

### EXCEPTION — Not captured
- **Then** `402 ERR.BUSINESS.refund.not_captured`

### EXCEPTION — Provider timeout
- **Then** `provider_pending` with retries; eventual webhook → completed/failed; no duplicate submissions

### EXCEPTION — Idempotency conflict
- **When** reuse key with different payload  
- **Then** `409 ERR.CONFLICT.idempotency`

---

## Edge Cases

- Mixed tax/discount orders: refund **pro‑rata** by line amount (documented rule).  
- Multiple tenders **out of scope** (reject with message).  
- Refund after **void** or **chargeback** → policy deny.  
- Currency rounding to minor units; no negative totals.  
- Attachment limits: type/size/count; virus scan.

---

## Synthetic & Data

- Synthetic journey per region: create → status poll → complete.  
- Fixtures: captured orders at `$10/$100/$9999`; orders with zero remaining; orders not captured.  
- Provider simulator with `succeeded/failed/timeout` modes and **signature** headers.

---

## Rollout & Risk

- Feature flags: `refunds.v1`, `refunds.goodwill.dual_control`.  
- Canary: 10% → 50% → 100%; guardrails on mismatch rate and error rate.  
- Known risks: provider flakiness, ledger drift, agent override misuse.  
- Mitigations: outbox/inbox, reconciliation job, audit dashboards, dual‑control.

---

## Evidence to attach in PR

- Sequence/state diagram, OpenAPI snippet.  
- Screenshots (user & agent flows; light/dark, LTR/RTL).  
- HAR for create/read; provider webhook samples.  
- Logs (`MSG.refund.*`), traces, metrics screenshots.  
- Ledger vs provider **reconciliation CSV**.

---

## Checklists

### Spec Ready (G1)
- [ ] Contracts & state machine  
- [ ] Policy rules defined (auto vs manual)  
- [ ] NFR budgets & telemetry plan  
- [ ] Reconciliation design

### Pre‑merge (G4)
- [ ] Functional/API tests green  
- [ ] Chaos (provider timeout) results attached  
- [ ] Security/abuse/privacy checks  
- [ ] Evidence bundle attached

---

## CSV Seeds

**Reasons taxonomy**

```csv
reason,description
not_received,Item did not arrive
quality,Quality issue
duplicate,Duplicate charge/order
pricing_error,Incorrect pricing
goodwill,Goodwill/retention
other,Other
```

**SLOs**

```csv
slo_id,description,objective,window,owner
refund_auto_decision_rate,Auto decision percentage,>=80%,14d,Payments
refund_resolution_t95,T95 time to complete,<=5d,30d,Payments
refund_mismatch_rate,Ledger vs provider mismatch,<=0.05%,7d,Finance
```

**Message IDs**

```csv
id,text
refund.request.accepted,"We’re processing your refund."
refund.completed,"Your refund is complete."
refund.failed,"We couldn’t complete your refund."
refund.exceeds_remaining,"This refund exceeds the available amount."
refund.not_captured,"We can’t refund this payment yet."
```

---

## Templates

**Test case skeleton**

```
CASE: API-Refund-Full-001
Pre: order captured 100.00 USD
Steps: POST /refunds 100.00 -> poll status
Expected: completed, remaining 0
Signals: MSG.refund.*, trace id, refund_outcome_total{state="completed"}
Evidence: HAR, JSON, webhook, ledger CSV
```

**Decision note**

```
Decision: Require dual-control for goodwill > $200
Reason: mitigate insider risk
Evidence: incident review; risk policy
```

---

## Links

- API Coverage → `../../60-checklists/api-coverage.md`  
- Performance → `../../60-checklists/performance-review.md`  
- Security → `../../60-checklists/security-review.md`  
- Payments & Checkout → `../../5-domain-playbooks/payments-and-checkout.md`  
- Traceability → `../../65-review-gates-metrics-traceability/traceability.md`
