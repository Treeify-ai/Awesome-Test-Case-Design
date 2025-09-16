# Test Cases (with Expected Results)

> Executable cases for **Refund Workflow v1**, mapped to **MAE** scenarios with clear **oracles** and **evidence** to attach.

See also:  
- Brief → `./brief.md`  
- Design Notes → `./design-notes.md`  
- Scenarios (MAE) → `./scenarios.md`

---

## How to run (quick)

1. Fixtures: `O100` ($100 captured), `O010` ($10 captured), `O000` (not captured).  
2. Headers: `Authorization`, `Idempotency-Key`, `X-Correlation-Id`.  
3. Capture artifacts: **HAR**, **JSON**, **logs** (`msgid`), **traces**, **metrics**, **recon CSV**.  
4. Use provider simulator modes where noted: `succeeded | failed | timeout`.

---

## Case registry (short)

| Case ID | Scenario | Type | Status | Owner |
|---|---|---|---|---|
| API-Refund-Full-001 | RF-MAIN-001 | API | planned | QA |
| UI-Refund-Full-001 | RF-MAIN-001 | UI | planned | QA |
| API-Refund-Partial-TwoSteps-001 | RF-MAIN-002 | API | planned | QA |
| API-Refund-Idem-Create-001 | RF-MAIN-003 | API | planned | QA |
| API-Refund-Cancel-001 | RF-MAIN-004 | API | planned | QA |
| UI-Refund-Manual-Review-001 | RF-ALT-001 | UI | planned | QA |
| CHA-Provider-Retry-001 | RF-ALT-002 | Chaos | planned | SRE |
| API-Webhook-Dedupe-001 | RF-ALT-003 | API | planned | QA |
| API-Refund-Attachment-001 | RF-ALT-004 | API | planned | QA |
| ADM-Refund-DualControl-001 | RF-ALT-005 | Admin | planned | QA |
| API-Refund-Remaining-Check-001 | RF-ALT-006 | API | planned | QA |
| API-Refund-NotCaptured-001 | RF-EXC-001 | API | planned | QA |
| API-Refund-ExceedsRemaining-001 | RF-EXC-002 | API | planned | QA |
| API-Refund-Idem-Conflict-001 | RF-EXC-003 | API | planned | QA |
| CHA-Provider-HardFail-001 | RF-EXC-004 | Chaos | planned | SRE |
| CHA-Provider-Timeout-NoWebhook-001 | RF-EXC-005 | Chaos | planned | SRE |
| API-Attachment-Rejected-001 | RF-EXC-006 | API | planned | QA |
| API-Authz-Blocked-001 | RF-EXC-007 | API | planned | QA |
| API-Currency-Mismatch-001 | RF-EXC-008 | API | planned | QA |

> Keep prose out of the table; see detailed steps below.

---

## Detailed cases

### API-Refund-Full-001 — Full refund succeeds
**Maps**: RF-MAIN-001  
**Pre**: Order `O100` captured `$100.00`.  
**Steps**
1) `POST /v1/orders/O100/refunds` body `{ "amount_minor": 10000, "currency": "USD", "reason": "not_received" }` + `Idempotency-Key: idem:rf1`  
2) Poll `GET /v1/refunds/{refund_id}` until terminal.  
**Expected**
- `202` with `state="approved"`; later `completed`.  
- Remaining refundable `0`.  
**Signals/Evidence**
- Logs: `MSG.refund.requested/approved/submitted/completed`.  
- Metric: `refund_outcome_total{state="completed"}` +1.  
- Trace: create → submit → webhook.  
- Recon CSV row: provider `refund_id` ↔ ledger `REFUND_SETTLED`.

---

### UI-Refund-Full-001 — User sees refunded status
**Maps**: RF-MAIN-001  
**Pre**: `O100` refund created.  
**Steps**
1) Open order detail → Refund timeline.  
2) Verify status transitions and copy (message IDs).  
**Expected**
- Timeline shows `processing → refunded`.  
- A11y: SR announces status; focus management correct.  
**Evidence**: screenshots (light/dark; LTR/RTL), SR short video.

---

### API-Refund-Partial-TwoSteps-001 — Two partial refunds
**Maps**: RF-MAIN-002  
**Pre**: `O100` captured `$100.00`.  
**Steps**
1) Create refund `$30.00`.  
2) After completion, create refund `$20.00`.  
3) Read `remaining_refundable_minor`.  
**Expected**
- Both complete; remaining `$50.00`.  
**Signals/Evidence**
- Two distinct `provider_refund_id`s.  
- Ledger shows two `REFUND_SETTLED` entries; parity holds.

---

### API-Refund-Idem-Create-001 — Idempotent create replay
**Maps**: RF-MAIN-003  
**Pre**: `O010`.  
**Steps**
1) Send same `POST` twice with `Idempotency-Key: idem:replay1`.  
**Expected**
- Second response equals first; header `Idempotency-Status: replayed`.  
**Evidence**: both HTTP responses; idempotency store readback.

---

### API-Refund-Cancel-001 — Cancel before submit
**Maps**: RF-MAIN-004  
**Pre**: Refund in `approved`.  
**Steps**
1) `POST /v1/refunds/{refund_id}/cancel`.  
**Expected**
- State `canceled`; no provider call.  
**Evidence**: state readback; absence of `provider_refund_id`; log `MSG.refund.canceled`.

---

### UI-Refund-Manual-Review-001 — Agent approves manual review
**Maps**: RF-ALT-001  
**Pre**: User triggers manual review policy.  
**Steps**
1) Agent console: review → approve with note.  
**Expected**
- State `requested → approved`; audit trail records agent id + note.  
**Evidence**: console screenshots; audit JSON.

---

### CHA-Provider-Retry-001 — Retry after timeout then succeed
**Maps**: RF-ALT-002  
**Pre**: Simulator mode `timeout` then `succeeded`.  
**Steps**
1) Create refund; observe `provider_pending`.  
2) Wait retry; verify final `completed`.  
**Expected**
- Single provider refund; `provider_attempts` ≥ 2.  
**Evidence**: job logs; webhook payloads; trace chain.

---

### API-Webhook-Dedupe-001 — Duplicate webhook is no-op
**Maps**: RF-ALT-003  
**Steps**
1) Deliver `refund.succeeded` twice (same `event_id`).  
**Expected**
- First transitions to `completed`; second deduped.  
**Evidence**: inbox rows; metric `webhook_replay_dedup_total` +1.

---

### API-Refund-Attachment-001 — Evidence attachments stored
**Maps**: RF-ALT-004  
**Steps**
1) Create refund with image attachment.  
2) GET refund; fetch attachment metadata.  
**Expected**
- Signed URLs; types/sizes within limits; virus scan pass.  
**Evidence**: metadata JSON; scan log.

---

### ADM-Refund-DualControl-001 — Dual-approval required
**Maps**: RF-ALT-005  
**Steps**
1) Approver A approves;  
2) Approver B approves.  
**Expected**
- Only after second approval can submit; audit lists two approvers.  
**Evidence**: audit log; UI screenshots.

---

### API-Refund-Remaining-Check-001 — Remaining guard
**Maps**: RF-ALT-006  
**Pre**: Prior `$70.00` refund completed on `O100`.  
**Steps**
1) Request `$40.00`.  
**Expected**
- `400 ERR.BUSINESS.refund.exceeds_remaining`.  
**Evidence**: error JSON; prior refund list.

---

### API-Refund-NotCaptured-001 — Not captured
**Maps**: RF-EXC-001  
**Steps**
1) Create refund on `O000`.  
**Expected**
- `402 ERR.BUSINESS.refund.not_captured`; no provider/ledger writes.  
**Evidence**: error JSON; absence checks.

---

### API-Refund-ExceedsRemaining-001 — Exceeds remaining
**Maps**: RF-EXC-002  
**Pre**: `O010` with `$5.00` already refunded.  
**Steps**
1) Request `$6.00`.  
**Expected**
- `400 ERR.BUSINESS.refund.exceeds_remaining`.  
**Evidence**: error JSON; remaining calc in logs.

---

### API-Refund-Idem-Conflict-001 — Idempotency conflict
**Maps**: RF-EXC-003  
**Steps**
1) Create with `Idempotency-Key: idem:x` and amount `$1.00`.  
2) Reuse same key with amount `$2.00`.  
**Expected**
- `409 ERR.CONFLICT.idempotency`; original response preserved.  
**Evidence**: both responses; idempotency store audit.

---

### CHA-Provider-HardFail-001 — Hard failure
**Maps**: RF-EXC-004  
**Pre**: Simulator `failed`.  
**Steps**
1) Submit refund.  
**Expected**
- Final state `failed`; user notified; remaining unchanged; no `REFUND_SETTLED`.  
**Evidence**: webhook payload, ledger snapshot, notification.

---

### CHA-Provider-Timeout-NoWebhook-001 — Timeout with no webhook
**Maps**: RF-EXC-005  
**Pre**: Simulator `timeout` and no webhook.  
**Steps**
1) Create refund; exhaust retries.  
**Expected**
- State `provider_pending`; alert raised on age; reconciliation later fixes.  
**Evidence**: queue metrics; alert screenshot; post-reconcile update.

---

### API-Attachment-Rejected-001 — Attachment rejected
**Maps**: RF-EXC-006  
**Steps**
1) Create refund with disallowed file.  
**Expected**
- `400` validation or `ERR.BUSINESS.attachment.rejected`.  
**Evidence**: error JSON; scan log excerpt.

---

### API-Authz-Blocked-001 — Authorization blocked
**Maps**: RF-EXC-007  
**Steps**
1) Low-privilege agent attempts goodwill refund above threshold.  
**Expected**
- `403 ERR.AUTHZ.scope`; no state change.  
**Evidence**: error JSON; policy evaluation log.

---

### API-Currency-Mismatch-001 — Currency mismatch
**Maps**: RF-EXC-008  
**Steps**
1) Create refund with currency != order currency.  
**Expected**
- `400 ERR.VALIDATION.currency`.  
**Evidence**: error JSON.

---

## Artifacts bundle (per PR)

- HAR: create/read; webhook sample(s).  
- JSON: responses and state timelines.  
- Logs: `MSG.refund.*`, `ERR.*` with `correlation_id`.  
- Traces: create + submit + webhook processing.  
- Metrics: `refund_outcome_total` breakdown; `refund_create_latency_ms`.  
- Reconciliation CSV: provider vs ledger parity.

---

## CSV seeds

**Case results (template)**

```csv
case_id,scenario_id,status,owner,evidence
API-Refund-Full-001,RF-MAIN-001,pass,QA,link://traces/abc
UI-Refund-Manual-Review-001,RF-ALT-001,pass,QA,link://screens/review
CHA-Provider-Retry-001,RF-ALT-002,pass,SRE,link://logs/retry
```

**Recon parity (example row)**

```csv
provider_refund_id,ledger_entry_id,amount_minor,currency,status,matched
prf_123,led_987,10000,USD,completed,true
```
