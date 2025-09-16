# Cases — Discount Code v1

> Executable test cases mapped to MAE scenarios.  
> Each case includes **preconditions**, **steps**, **expected**, and **signals/evidence** to attach.

See also:  
- Brief → `./brief.md`  
- Design Notes → `./design-notes.md`  
- Scenarios (MAE) → `./scenarios.md`

---

## How to run (quick)

1) Prepare **Cart A** (`$100.00`, all eligible) and **Cart B** (`$3.00`).  
2) Have fixture codes: `SAVE15`, `LESS500`, `SHIPFREE`, `NEWUSR`.  
3) Set headers: `Authorization`, `Idempotency-Key`, `X-Correlation-Id`.  
4) Capture artifacts: **HAR**, **JSON**, **screenshots**, **logs**, **traces**, **metrics**.

---

## Case registry (short)

| Case ID | Scenario | Type | Status | Owner |
|---|---|---|---|---|
| API-Apply-Valid-001 | CD-MAIN-001 | API | planned | QA |
| UI-Apply-Valid-001 | CD-MAIN-001 | UI | planned | QA |
| API-Remove-001 | CD-MAIN-002 | API | planned | QA |
| UI-Remove-001 | CD-MAIN-002 | UI | planned | QA |
| API-Persist-001 | CD-MAIN-003 | API | planned | QA |
| API-Order-Redemption-001 | CD-MAIN-004 | API | planned | QA |
| API-FixedCap-001 | CD-ALT-001 | API | planned | QA |
| UI-FreeShip-001 | CD-ALT-002 | UI | planned | QA |
| API-Reprice-MinDrop-001 | CD-ALT-003 | API | planned | QA |
| API-Idem-Reapply-001 | CD-ALT-004 | API | planned | QA |
| API-Race-TwoTabs-001 | CD-ALT-005 | API | planned | QA |
| API-PerUserLimit-001 | CD-ALT-006 | API | planned | QA |
| API-CurrencyMismatch-001 | CD-ALT-007 | API | planned | QA |
| ADM-Override-001 | CD-ALT-008 | Admin | planned | QA |
| API-BadFormat-001 | CD-EXC-001 | API | planned | QA |
| API-Expired-001 | CD-EXC-002 | API | planned | QA |
| API-RateLimit-001 | CD-EXC-003 | API | planned | QA |
| CHA-PromoBrownout-001 | CD-EXC-004 | Chaos | planned | SRE |
| CHA-TaxTimeout-001 | CD-EXC-005 | Chaos | planned | SRE |
| API-Idem-Conflict-001 | CD-EXC-006 | API | planned | QA |
| UI-Offline-001 | CD-EXC-007 | UI | planned | QA |

> Keep prose out of tables; details below.

---

## Case details

### API-Apply-Valid-001 — Apply `SAVE15` (percent)
**Pre**: Cart A exists, user U1, headers set.  
**Steps**
1. `POST /v1/checkout/{cart}/discounts/apply` body `{ "code": "SAVE15" }` + `Idempotency-Key: idem:valid1`  
2. Record response JSON, headers, timing.  
3. Fetch `/v1/checkout/{cart}` to confirm snapshot.
**Expected**
- `200` with `applied_code.code = "SAVE15"`; discount equals 15% (half-even).  
- Totals recomputed; currency correct.  
- Header `Idempotency-Status` absent.
**Signals/Evidence**
- Logs: `MSG.discount.apply.succeeded`.  
- Metric: `discount_attempts_total{result="applied"}` increments.  
- Trace spans: `promo.validate`, `pricing.compute`.  
- Artifacts: HAR, JSON, trace link, screenshot.

---

### UI-Apply-Valid-001 — UI apply flow
**Pre**: Web desktop, Cart A loaded, `en`.  
**Steps**
1. Enter `SAVE15`, click **Apply**.  
2. Wait for totals update.  
**Expected**
- New total shows; “Discount applied” toast/message.  
- Focus returns to input; screen reader announces update.  
**Signals/Evidence**
- Screenshot (light/dark), SR short video, HAR.

---

### API-Remove-001 — Remove code
**Pre**: Cart A has `SAVE15` applied.  
**Steps**
1. `DELETE /v1/checkout/{cart}/discounts/apply`.  
2. GET cart.
**Expected**
- `200`; `applied_code` absent; totals revert.  
**Signals/Evidence**
- Log `MSG.discount.removed`; event `discount.removed`.  
- HAR pair; before/after JSON.

---

### UI-Remove-001 — UI remove flow
**Pre**: UI shows `SAVE15` applied.  
**Steps**
1. Click **Remove**.  
2. Confirm totals revert.
**Expected**
- Return to original price; focus safe.  
**Evidence**: screenshot/gif.

---

### API-Persist-001 — Refresh + login
**Pre**: Guest U1 applied `SAVE15`.  
**Steps**
1. Refresh page; GET cart.  
2. Login; link cart; GET cart.  
**Expected**
- Applied code persists; no duplicate events.  
**Evidence**: JSON before/after; event counts.

---

### API-Order-Redemption-001 — Place order
**Pre**: Cart A with `SAVE15`; payment succeeds.  
**Steps**
1. `POST /v1/orders` (create from cart).  
2. Query redemption endpoint/admin.  
**Expected**
- `discount_redemptions` row created; unique `redemption_id`.  
**Evidence**: readback JSON/screenshot; log `MSG.redemption.created`.

---

### API-FixedCap-001 — Fixed amount capped
**Pre**: Cart B `$3.00`.  
**Steps**: Apply `LESS500`.  
**Expected**
- Discount = `$3.00`; total not below zero; rounding to minor units.  
**Evidence**: response JSON; note of rounding.

---

### UI-FreeShip-001 — Free shipping eligibility
**Pre**: Cart A, `standard` eligible, `express` not.  
**Steps**
1. Apply `SHIPFREE`.  
2. Switch to `express`; then back to `standard`.  
**Expected**
- `standard` cost `0`; `express` shows fee again.  
**Evidence**: two JSON snapshots; UI video.

---

### API-Reprice-MinDrop-001 — Eligibility lost on change
**Pre**: Cart A with `SAVE15`.  
**Steps**
1. Remove item to drop below min subtotal.  
2. GET cart.  
**Expected**
- Code removed or ignored; message available.  
**Evidence**: JSON before/after; log `ERR.BUSINESS.code.ineligible`.

---

### API-Idem-Reapply-001 — Re-apply with same key
**Pre**: None.  
**Steps**
1. Send two **identical** apply requests with `Idempotency-Key: idem:re1`.  
**Expected**
- First `200`; second `200` + header `Idempotency-Status: replayed`.  
**Evidence**: both responses; event counts unchanged.

---

### API-Race-TwoTabs-001 — Race with different keys
**Pre**: Two concurrent clients.  
**Steps**
1. Send two apply requests simultaneously with **different** keys.  
**Expected**
- Single applied state; stable total; no dup events.  
**Evidence**: timing logs; final cart JSON.

---

### API-PerUserLimit-001 — Per-user limit
**Pre**: Code `NEWUSR` limit `1`.  
**Steps**
1. Apply on Cart A → expect success.  
2. Apply on new Cart C same user → expect failure.  
**Expected**
- Second `400` with `ERR.BUSINESS.code.ineligible`.  
**Evidence**: error JSON; metrics show `ineligible` result.

---

### API-CurrencyMismatch-001 — Fixed code currency mismatch
**Pre**: Order currency `EUR`.  
**Steps**: Apply `LESS500` (`USD`).  
**Expected**
- `400` in v1; taxonomy code for ineligible.  
**Evidence**: error JSON; log snippet.

---

### ADM-Override-001 — Agent override
**Pre**: Admin access; ticket id available.  
**Steps**
1. Set override flag; apply code.  
2. Read audit log.  
**Expected**
- Discount applied; audit shows agent id + reason.  
**Evidence**: admin screenshot; audit log JSON.

---

### API-BadFormat-001 — Invalid code format
**Steps**: Apply `@@bad!!`.  
**Expected**
- `400 ERR.VALIDATION.code.format`; no state change.  
**Evidence**: error JSON; UI screenshot (optional).

---

### API-Expired-001 — Not started/Expired window
**Steps**: Apply fixture outside valid window.  
**Expected**
- `400 ERR.BUSINESS.code.ineligible`.  
**Evidence**: error JSON; logs `apply.failed`.

---

### API-RateLimit-001 — Brute-force attempt
**Steps**
1. Send 10 invalid codes in 60 s.  
**Expected**
- `429` on later requests; `Retry-After` present.  
**Evidence**: limiter metrics; error JSON.

---

### CHA-PromoBrownout-001 — Promo dependency brownout
**Pre**: Chaos tool ready.  
**Steps**
1. Inject 5xx + latency on promo RPC.  
2. Apply valid code.  
**Expected**
- Breaker opens; UI banner; checkout allowed without discount.  
**Evidence**: breaker metrics; synthetic journey remains green.

---

### CHA-TaxTimeout-001 — Tax timeout
**Steps**
1. Inject timeout on tax RPC.  
2. Apply valid code.  
**Expected**
- Estimated tax used; order marked for recompute.  
**Evidence**: trace with timeout; job log for recompute.

---

### API-Idem-Conflict-001 — Idempotency conflict
**Steps**
1. Apply with `Idempotency-Key: idem:x` and `{ code: "SAVE15" }`.  
2. Reuse same key with `{ code: "LESS500" }`.  
**Expected**
- `409 ERR.CONFLICT.idempotency`; original response preserved.  
**Evidence**: both responses; idempotency store proof.

---

### UI-Offline-001 — Offline / captive portal
**Pre**: Throttle network.  
**Steps**
1. Go offline; hit **Apply**.  
2. Reconnect; hit **Retry**.  
**Expected**
- Non-blocking banner; state preserved; retry succeeds.  
**Evidence**: devtools recording; screenshot.

---

## Negative & edges (add as needed)

- Unicode normalization on codes; whitespace trim.  
- Large carts; many lines; performance budgets.  
- Multi-tenant isolation; can’t apply other tenant’s private codes.  
- Retry after 429 honors backoff; no burst on recovery.

---

## Artifacts to attach

- HAR for **apply/remove**.  
- Response JSONs.  
- Screenshots (light/dark, LTR/RTL).  
- Log snippets (`msgid`, `err.code`, `correlation_id`).  
- Trace links (p99 exemplar).  
- Metrics screenshots (latency p95/p99; attempts result).  
- CSV result register.

**CSV template**

```csv
case_id,scenario_id,status,owner,evidence
API-Apply-Valid-001,CD-MAIN-001,pass,QA,link://traces/abc
UI-Remove-001,CD-MAIN-002,pass,QA,link://har/remove
```
