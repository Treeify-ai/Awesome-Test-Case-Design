<!-- ===== Treeify Header ===== -->
<p align="center">
  <!-- Optional: swap in your logo paths -->
  <!-- <img src="assets/logo-light.svg" alt="Treeify" height="72"> -->
</p>

<h3 align="center">Treeify — AI test case design copilot</h3>
<p align="center">
  <em>Structured, traceable, high-coverage test design — faster.</em><br>
  <a href="https://treeifyai.com">treeifyai.com</a>
</p>

<p align="center">
  <a href="https://treeifyai.com">
    <img alt="Try Treeify Free" src="https://img.shields.io/badge/Try%20Treeify%20Free-treeifyai.com-brightgreen?style=for-the-badge">
  </a>
</p>


# Scenarios (MAE)

> End‑to‑end scenarios for **Refund Workflow v1**, written in **Main / Alternative / Exception** form with explicit **oracles** and **evidence** to attach.

See also:  
- Brief → `./brief.md`  
- Design Notes → `./design-notes.md`

---

## Conventions

- IDs: `RF-<MAIN|ALT|EXC>-NNN`.  
- Use **Given/When/Then**.  
- Attach evidence: **HAR**, **JSON**, **screenshots** (light/dark, LTR/RTL), **logs** (`msgid`), **metrics** snapshots, **trace** links, **ledger↔provider CSV**.  
- Timezone: UTC for timestamps; amounts in **minor units**.

---

## Fixtures & test data

- Orders:  
  - **O100** — captured **$100.00** (single capture).  
  - **O010** — captured **$10.00**.  
  - **O000** — **not captured** (auth voided).  
- Users: **U1** (normal), **U2** (requires manual review for goodwill).  
- Currencies: `USD` baseline.  
- Provider simulator modes: `succeeded`, `failed`, `timeout`.  
- Reasons taxonomy from the Brief.

---

## MAIN

### RF-MAIN-001 — Full refund on captured payment
**Given** order **O100** captured `$100.00` for **U1**  
**When** `POST /v1/orders/O100/refunds { "amount_minor": 10000, "currency": "USD", "reason": "not_received" }` with `Idempotency-Key: idem:rf1`  
**Then**
- API returns `202` with `state="approved"` and `refund_id`  
- Background submit transitions to `provider_pending` then `completed`  
- Remaining refundable becomes `0`  
**Oracles**
- Logs: `MSG.refund.requested`, `MSG.refund.approved`, `MSG.refund.submitted`, `MSG.refund.completed`  
- Event: `refund.completed` with amounts & currency  
- Metrics: `refund_outcome_total{state="completed"}` incremented  
- Ledger: `REFUND_PENDING` then `REFUND_SETTLED` present and matched to provider `refund_id`  
**Evidence**: HAR (create/read), state timeline JSON, provider webhook sample, ledger vs provider CSV row, trace link.

---

### RF-MAIN-002 — Two partial refunds totaling less than captured
**Given** order **O100** captured `$100.00`  
**When** refund `$30.00` then `$20.00`  
**Then**
- Both complete; remaining refundable `$50.00`  
- No over‑refund; ledger parity holds  
**Oracles**
- Remaining computation matches `100 - (30+20)`  
- Provider has two distinct `refund_id`s; ledger has two matched entries  
**Evidence**: two request/response pairs, reconciliation CSV, metrics snapshot per state.

---

### RF-MAIN-003 — Idempotent create (replay same key)
**Given** order **O010** captured `$10.00`  
**When** send identical `POST` twice with same `Idempotency-Key: idem:replay1`  
**Then**
- Second response equals first and includes header `Idempotency-Status: replayed`  
- Only one refund is created  
**Oracles**: idempotency store readback; event count = 1  
**Evidence**: both HTTP responses, idempotency store audit.

---

### RF-MAIN-004 — Cancel before submission
**Given** order **O010** and a refund in `approved` state  
**When** `POST /v1/refunds/{refund_id}/cancel` (if allowed by policy)  
**Then** state becomes `canceled`; no provider call was made  
**Oracles**: absence of `provider_refund_id`; logs show `MSG.refund.canceled`  
**Evidence**: state readback, log snippet.

---

## ALTERNATIVE

### RF-ALT-001 — Manual review path (goodwill)
**Given** user **U2** triggers policy for manual review for **$10.00** goodwill  
**When** create refund request  
**Then** state `requested`; agent decision approves → `approved` → normal processing  
**Oracles**: audit trail with agent id/note; dual‑control if above threshold  
**Evidence**: agent console screenshot, audit log JSON, events.

---

### RF-ALT-002 — Multi‑attempt provider submission with success
**Given** provider simulator `timeout` for first attempt then `succeeded`  
**When** system retries with backoff  
**Then** single provider refund created; final state `completed`  
**Oracles**: `provider_attempts` incremented; inbox dedupe count stable (no dup apply)  
**Evidence**: job logs, webhook payloads, trace chain across attempts.

---

### RF-ALT-003 — Duplicate webhook delivery (replay)
**Given** provider emits `refund.succeeded` twice  
**When** both arrive  
**Then** first transitions `provider_pending → completed`; second is a **no‑op**  
**Oracles**: inbox table shows one processed, one deduped; `webhook_replay_dedup_total` increments  
**Evidence**: inbox rows, metric snapshot.

---

### RF-ALT-004 — Refund with attachments
**Given** user provides image evidence  
**When** create refund  
**Then** attachments stored with signed URLs; metadata recorded; PII policy enforced  
**Oracles**: virus scan passed; content type and size within limits  
**Evidence**: attachment metadata JSON, storage log, scan result.

---

### RF-ALT-005 — Goodwill threshold requires dual‑control
**Given** amount over configured threshold  
**When** first approver approves  
**Then** state waits for second approver; only after second approval can submit  
**Oracles**: audit trail with two distinct approvers; RBAC respected  
**Evidence**: audit log, UI screenshots.

---

### RF-ALT-006 — Mixed prior refunds (remaining check)
**Given** order **O100** has one completed `$70.00` refund  
**When** request new `$40.00` refund  
**Then** rejected as exceeds remaining  
**Oracles**: remaining calculation uses **completed + approved**  
**Evidence**: prior refunds list, error JSON.

---

## EXCEPTION

### RF-EXC-001 — Not captured
**Given** order **O000** (not captured)  
**When** request any refund  
**Then** `402 ERR.BUSINESS.refund.not_captured`  
**Oracles**: no ledger entries; no provider call  
**Evidence**: error JSON, logs.

---

### RF-EXC-002 — Exceeds remaining
**Given** order **O010** captured `$10.00` and `$5.00` already refunded  
**When** request `$6.00`  
**Then** `400 ERR.BUSINESS.refund.exceeds_remaining`  
**Oracles**: remaining calculation visible in response or logs  
**Evidence**: error JSON, prior refunds readback.

---

### RF-EXC-003 — Idempotency conflict (same key, different payload)
**When** replay same `Idempotency-Key` with **different** amount  
**Then** `409 ERR.CONFLICT.idempotency`; original response not overwritten  
**Oracles**: idempotency store shows original hash  
**Evidence**: both responses; store audit.

---

### RF-EXC-004 — Provider failure (hard)
**Given** provider simulator returns `failed`  
**When** submit  
**Then** final state `failed`; user notified; remaining refundable unchanged  
**Oracles**: logs `MSG.refund.failed`; metric increments `{state="failed"}`; ledger does **not** post SETTLED  
**Evidence**: webhook payload, ledger snapshot, notification sample.

---

### RF-EXC-005 — Provider timeout with no webhook
**Given** provider simulator `timeout` and no webhook delivered  
**When** retries exhausted  
**Then** state remains `provider_pending`; reconciliation job later resolves  
**Oracles**: alert on pending age; runbook linked; eventual poll fixes state  
**Evidence**: queue metrics, alert screenshot, later poll response.

---

### RF-EXC-006 — Attachment rejected (virus/size)
**When** create refund with disallowed file  
**Then** `400` validation or `ERR.BUSINESS.attachment.rejected`; no refund created  
**Oracles**: scan logs show reason; audit trail records attempt  
**Evidence**: error JSON, scan log excerpt.

---

### RF-EXC-007 — Authorization blocked
**When** low‑privilege agent attempts goodwill refund above threshold  
**Then** `403 ERR.AUTHZ.scope`  
**Oracles**: RBAC policy evaluation log; no state change  
**Evidence**: error JSON, policy log.

---

### RF-EXC-008 — Currency mismatch
**When** create refund with currency != order currency  
**Then** `400 ERR.VALIDATION.currency` (v1 requires original tender currency)  
**Oracles**: error taxonomy code present; copy mapped to message id  
**Evidence**: error JSON.

---

## Cross‑checks (post‑scenario)

- Events: `refund.created/approved/submitted/completed/failed` parity with state history.  
- Metrics: `refund_outcome_total{state}` tallies match counts; `refund_mismatch_rate_pct` ≤ threshold.  
- Logs: contain `msgid`, `err.code`, `correlation_id`; no PII.  
- Traces: present for slowest p99 creates and submissions.  
- Reconciliation: daily CSV shows **0** mismatches for these fixtures.

---

## CSV seeds (scenario register)

```csv
id,type,title
RF-MAIN-001,MAIN,Full refund captured
RF-MAIN-002,MAIN,Two partial refunds
RF-MAIN-003,MAIN,Idempotent create replay
RF-MAIN-004,MAIN,Cancel before submit
RF-ALT-001,ALT,Manual review (goodwill)
RF-ALT-002,ALT,Provider retries success
RF-ALT-003,ALT,Duplicate webhook replay
RF-ALT-004,ALT,Refund with attachments
RF-ALT-005,ALT,Dual-control threshold
RF-ALT-006,ALT,Remaining check with prior refund
RF-EXC-001,EXC,Not captured
RF-EXC-002,EXC,Exceeds remaining
RF-EXC-003,EXC,Idempotency conflict
RF-EXC-004,EXC,Provider hard failure
RF-EXC-005,EXC,Provider timeout no webhook
RF-EXC-006,EXC,Attachment rejected
RF-EXC-007,EXC,Authorization blocked
RF-EXC-008,EXC,Currency mismatch
```

---

## Evidence bundle (attach to PR)

- HARs for create/read; webhook samples.  
- State timeline JSON per refund.  
- Screenshots (user + agent flows; light/dark, LTR/RTL).  
- Logs (`MSG.refund.*` and `ERR.*`).  
- Traces (create + submit + webhook processing).  
- Metrics: outcomes, create latency, pending age.  
- Reconciliation CSV: provider vs ledger parity results.

```
case_id,scenario_id,status,owner,evidence
API-Refund-Full-001,RF-MAIN-001,pass,QA,link://traces/abc
UI-Refund-Manual-001,RF-ALT-001,pass,QA,link://screens/manrev
```

---

<p align="center">
  <sub>
    Built with <a href="https://treeifyai.com">Treeify</a> — AI test case design copilot.<br>
    <em>Structured, traceable, high-coverage test design — faster.</em>
  </sub>
</p>
