# Scenarios (MAE)

> Concrete, end‑to‑end scenarios for **Discount Code v1**.  
> Each scenario uses **MAE** (Main / Alternative / Exception), has explicit **oracles**, and lists **evidence** to attach.

See also:  
- Brief → `./brief.md`  
- Design Notes → `./design-notes.md`

---

## Conventions

- IDs: `CD-<MAIN|ALT|EXC>-NNN`.  
- Given/When/Then style; short, verifiable steps.  
- Evidence: **HAR**, **response JSON**, **screenshots** (light/dark, LTR/RTL), **logs** (`msgid`), **metrics** snapshots, **trace** links.  
- Use fixtures from the Brief (`SAVE15`, `LESS500`, `SHIPFREE`, `NEWUSR`).

---

## Test data (fixtures)

- Cart A: subtotal **$100.00**, all items eligible, shipping `standard`.  
- Cart B: subtotal **$3.00**, below min for some codes.  
- User U1 (first‑time), U2 (repeat).  
- Locales: `en`, `ar` (RTL).  
- Devices: desktop web (Chrome), mobile web (Safari iOS).

---

## MAIN

### CD-MAIN-001 — Apply valid percent code
**Given** Cart A, user U1  
**When** `POST /v1/checkout/c_A/discounts/apply { "code": "SAVE15" }` with `Idempotency-Key: idem:001`  
**Then**
- `200` with `applied_code.code = "SAVE15"`  
- `pricing.discount_minor` = **15%** of eligible subtotal (rounded half‑even)  
- Logs include `MSG.discount.apply.succeeded`  
- Metric increments `discount_attempts_total{result="applied"}`  
- Trace shows `promo.validate` and `pricing.compute` spans

**Evidence**: HAR, JSON, metrics screenshot, trace link, UI screenshot showing new total.

---

### CD-MAIN-002 — Remove code
**Given** Cart A has `SAVE15` applied  
**When** `DELETE /v1/checkout/c_A/discounts/apply`  
**Then**
- `200`; totals recomputed without discount  
- Event `discount.removed` emitted  
- UI shows prior price; focus returns to input

**Evidence**: HAR pair (before/after), screenshot, log `MSG.discount.removed`.

---

### CD-MAIN-003 — Persist across refresh and login boundary
**Given** Guest user U1 applied `SAVE15` on Cart A  
**When** refresh page; then login same user (link cart)  
**Then**
- `applied_code` persists server‑side; totals consistent  
- No duplicate apply events; idempotency status not replayed

**Evidence**: JSON before/after, event log counts, UI screenshots.

---

### CD-MAIN-004 — Place order with discount
**Given** Cart A with `SAVE15`  
**When** place order successfully  
**Then**
- Row in `discount_redemptions` with **unique** `redemption_id`  
- Event `order.created` includes discount snapshot  
- Receipt shows discounted totals

**Evidence**: DB/admin view screenshot (or API read), receipt PDF/HTML, log `MSG.redemption.created`.

---

## ALTERNATIVE

### CD-ALT-001 — Fixed amount with cap
**Given** Cart B subtotal **$3.00**, code `LESS500` ($5.00 off)  
**When** apply  
**Then**
- Discount limited to **$3.00**; total not negative  
- Rounding to minor units verified

**Evidence**: JSON totals; rounding note.

---

### CD-ALT-002 — Free shipping on eligible method
**Given** Cart A, shipping `standard` (eligible), code `SHIPFREE`  
**When** apply; then switch shipping to `express` (not eligible)  
**Then**
- `standard` cost becomes `0`; switching to `express` restores fee  
- No item price changes

**Evidence**: two JSON snapshots; UI switch video/gif.

---

### CD-ALT-003 — Cart changes invalidate eligibility
**Given** Cart A with `SAVE15` applied  
**When** remove eligible item so subtotal drops below min  
**Then**
- Reprice drops discount; message shows “No longer meets requirements”  
- `MSG.discount.apply.failed` with `ERR.BUSINESS.code.ineligible`

**Evidence**: before/after JSON; log snippet; UI banner screenshot.

---

### CD-ALT-004 — Idempotent re‑apply (same key)
**Given** Cart A; prepare `Idempotency-Key: idem:004`  
**When** send **same** `POST .../apply` twice concurrently  
**Then**
- First `200`; second `200` with `Idempotency-Status: replayed`  
- No duplicate events or double discounts

**Evidence**: both responses; event counts.

---

### CD-ALT-005 — Two tabs race (different keys)
**Given** Two browser tabs on Cart A  
**When** both apply `SAVE15` simultaneously with **different** keys  
**Then**
- Single applied state; totals consistent; no duplicate redemptions later

**Evidence**: timing diagram, logs, final JSON.

---

### CD-ALT-006 — Per‑user limit enforced
**Given** Code `NEWUSR` limit `1` per user  
**When** U1 applies → succeeds; apply again on new cart  
**Then**
- Second attempt fails with `ERR.BUSINESS.code.ineligible` (limit)  
- Attempts metric shows `ineligible`

**Evidence**: JSON, logs, metrics.

---

### CD-ALT-007 — Fixed code currency mismatch
**Given** Order currency `EUR`, code `LESS500` in `USD`  
**When** apply  
**Then**
- Rejected (`ERR.BUSINESS.code.ineligible`) in v1  
**Evidence**: error body; logs.

---

### CD-ALT-008 — Agent override (policy)
**Given** U1 fails eligibility by small margin; agent applies override in admin  
**When** override flag set with reason  
**Then**
- Discount applied; audit log records agent id + ticket  
**Evidence**: admin screenshot; audit log.

---

## EXCEPTION

### CD-EXC-001 — Invalid format
**When** apply code `@@bad!!`  
**Then**
- `400 ERR.VALIDATION.code.format`; input preserved; focus on field

**Evidence**: error JSON; UI screenshot; SR transcript (optional).

---

### CD-EXC-002 — Not started / expired
**When** apply `SAVE15` outside window  
**Then**
- `400 ERR.BUSINESS.code.ineligible`; generic UI copy  
- No state change to cart

**Evidence**: error JSON; logs `apply.failed`.

---

### CD-EXC-003 — Rate limit (brute force)
**When** 10 invalid codes within 1 minute  
**Then**
- `429` with `Retry-After` set; limiter metrics increase; no sms/email fired  
- UI shows “Too many attempts”

**Evidence**: limiter metrics, error JSON, UI screenshot.

---

### CD-EXC-004 — Promo service brownout
**Given** promo returns slow 5xx  
**When** apply valid code  
**Then**
- Circuit breaker opens; UI banner “Discount temporarily unavailable”; checkout allowed without discount  
- Journey SLI remains within budget

**Evidence**: breaker metrics, logs, synthetic check result.

---

### CD-EXC-005 — Tax service delay
**When** apply code but tax RPC times out  
**Then**
- Use estimate; mark order for recompute; user can proceed  
- Error logged `ERR.DEPENDENCY.timeout`; retry path available

**Evidence**: logs, trace showing timeout, follow-up recompute job log.

---

### CD-EXC-006 — Idempotency conflict
**When** reuse **same** `Idempotency-Key` but **different** payload/cart hash  
**Then**
- `409 ERR.CONFLICT.idempotency`; previous response not overwritten

**Evidence**: both responses, server idempotency store audit.

---

### CD-EXC-007 — Offline / captive portal
**When** network offline or captive portal during apply  
**Then**
- UI shows non-blocking banner; local state preserved; retry CTA  
- No backend event emitted

**Evidence**: devtools network log, UI screenshot.

---

## Cross‑checks (post‑scenario)

- Verify **events**: `discount.applied` and `discount.removed` parity with UI.  
- Verify **metrics** breakdown by result: `applied | invalid | expired | ineligible | rate_limited`.  
- Verify **logs** include `msgid`, `err.code`, `correlation_id`.  
- Verify **traces** exist for slowest 1% (p99 apply).

---

## CSV seeds (scenario register)

```csv
id,type,title
CD-MAIN-001,MAIN,Apply valid percent
CD-MAIN-002,MAIN,Remove code
CD-MAIN-003,MAIN,Persist across refresh/login
CD-MAIN-004,MAIN,Place order with discount
CD-ALT-001,ALT,Fixed amount with cap
CD-ALT-002,ALT,Free shipping eligibility
CD-ALT-003,ALT,Cart changes invalidate eligibility
CD-ALT-004,ALT,Idempotent re-apply
CD-ALT-005,ALT,Two tabs race
CD-ALT-006,ALT,Per-user limit
CD-ALT-007,ALT,Currency mismatch
CD-ALT-008,ALT,Agent override
CD-EXC-001,EXC,Invalid format
CD-EXC-002,EXC,Not started/Expired
CD-EXC-003,EXC,Rate limit
CD-EXC-004,EXC,Promo brownout
CD-EXC-005,EXC,Tax delay
CD-EXC-006,EXC,Idempotency conflict
CD-EXC-007,EXC,Offline/Captive portal
```

---

## Evidence bundle (attach to PR)

- **Screenshots**: apply/remove success & errors (light/dark, LTR/RTL).  
- **HAR**: `apply` and `remove`, with timings.  
- **Logs**: `MSG.discount.apply.*`, `ERR.*` samples.  
- **Metrics**: latency p95/p99, attempts breakdown.  
- **Traces**: at least one p99 exemplar link.  
- **CSV**: scenario run register with status & owners.

```
case_id,scenario_id,status,owner,evidence
API-Apply-Valid-001,CD-MAIN-001,pass,QA,link://traces/abc
UI-Remove-001,CD-MAIN-002,pass,QA,link://har/remove
```
