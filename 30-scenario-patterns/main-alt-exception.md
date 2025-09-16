# Main / Alternative / Exception Flows (Pattern)

> MAE turns a big feature into a **small set of journeys** you can reason about.  
> **Main** covers the happy path. **Alternative** captures legitimate variations. **Exception** captures errors, conflicts, and recoveries.

This pattern is the fastest way to move from PRD text to **checkable scenarios** that map cleanly to cases, states, and contracts.

---

## What & Why

- **Main** — The **intended** end-to-end path when everything is valid and available.
- **Alternative** — Realistic variations that still **succeed** (late login, delayed apply, retry within budget, different role).
- **Exception** — Paths that **fail** or **degrade** gracefully (validation errors, policy conflicts, dependency timeouts), ideally with a **recovery** route.

**Why MAE?**  
- Forces clarity on **preconditions**, **triggers**, and **observable results**.  
- Prevents **happy-path bias**.  
- Gives you a minimal set of **scenario shells** to layer techniques (boundary/pairwise) onto.

---

## Quick workflow (7 steps)

1. **Scope** a thin slice (one user goal / API endpoint).  
2. List **preconditions** (role, state, flags, data).  
3. Write **Main** in ≤6 observable steps (UI state, API call, event).  
4. Add **2–3 Alternatives** that still end in success.  
5. Add **3–5 Exceptions** by category: **Validation**, **Policy/Conflict**, **Transient/Dependency**, **AuthZ**.  
6. For each Exception, write a **Recovery** (retry, correct input, fallback).  
7. Add **oracles** (message IDs, response codes, DB/event deltas, logs/metrics/traces).

Deliverables: `scenarios.md` with MAE entries, linked cases in `cases.md`.

---

## Scenario card template (copy/paste)

```
## S-<nnn> <Type: Main|Alt|Exception> — <short name>

**Goal**: <user/system outcome in 1 line>  
**Preconditions**: <role, state, flags, data>  
**Trigger**: <user action / API call / event>  
**Steps**:  
1) <observable step>  
2) ...  
**Expected result**: <what changes / persists / is visible>  
**Oracles (evidence)**: <response code/body, message IDs, logs, metrics, traces, DB deltas>  
**Recovery** (Exception only): <fix path, retry window, fallback>
```

> Keep each step **observable**. “Call verify” is observable if it leaves an API trace or log with a correlation ID.

---

## Example — Discount Code (Checkout)

**Feature slice**: “Apply discount code” on web checkout  
**Actors**: Guest or Member  
**Contracts**: `/discount/verify` (idempotent), error taxonomy → UX messages

### S-001 **Main** — Valid code applied
- **Preconditions**: cart has items; code `SAVE10` valid, not expired; role: Guest or Member
- **Trigger**: user enters code, presses **Apply**
- **Steps**:  
  1) UI sends `POST /discount/verify` with `code=SAVE10` and `Idempotency-Key=k`  
  2) API returns `200` with `applied=true`, discount amount  
  3) UI shows **Applied** badge; **Total** updates
- **Expected**: code persists on cart; totals recomputed
- **Oracles**: resp `200`; UI badge; DB/cart delta; log `{event:verify.ok, code_len:6, key:k}`

### S-002 **Alt** — Apply after address step (late apply)
- **Preconditions**: Member filled address; code is still valid
- **Trigger**: presses **Apply** after address
- **Steps**: same as S-001
- **Expected**: still applies (late stage)
- **Oracles**: resp `200`; trace `verify` span at step N

### S-003 **Alt** — Idempotent re-apply (duplicate click)
- **Preconditions**: S-001 done; same page
- **Trigger**: clicks Apply twice (same `Idempotency-Key=k`)
- **Expected**: **same** result; no duplicate discount rows
- **Oracles**: logs show **same key**; resp same discount

### S-101 **Exception/Validation** — Above-max length
- **Preconditions**: none
- **Trigger**: user inputs `"A"*17` and applies
- **Expected**: reject with `VALIDATION.code.length.exceeds`; input preserved; no total change
- **Recovery**: user edits to 16 chars → S-001
- **Oracles**: resp `400` with message ID; UI message visible; log `{error_code:VALIDATION.code.length.exceeds}`

### S-102 **Exception/Validation** — Forbidden char
- **Trigger**: `"SAVE!0"`
- **Expected**: `VALIDATION.code.charset`; input preserved; no total change
- **Recovery**: change to allowed charset → S-001

### S-103 **Exception/Policy** — Not combinable with gift card
- **Preconditions**: cart includes gift card; code is percent-off non-combinable
- **Expected**: `CONFLICT.code.not_combinable`; **offer remove** gift card
- **Recovery**: remove gift card → S-001
- **Oracles**: resp `409`; UI hint; audit log for decision table rule `R3`

### S-104 **Exception/Transient** — Timeout with retry
- **Preconditions**: gateway latency spike
- **Steps**:  
  1) Verify call times out  
  2) Client retries with backoff & **same key**
- **Expected**: eventually applies; no duplicate apply
- **Oracles**: metric `retry_count`; same `Idempotency-Key` across attempts; one cart delta

> Layer **Boundary** and **Decision Table** edges onto these scenarios rather than inventing new scenarios for every input variant.

---

## Example — Password Reset (complementary domain)

### S-201 **Main** — Reset via emailed code
- **Preconditions**: account exists; rate limit ok
- **Trigger**: user requests reset; then submits code within TTL
- **Expected**: session requires new password; old sessions invalidated
- **Oracles**: event `email.code.sent`; resp `200` on submit; log `verified=true`

### S-202 **Alt** — Resend then use latest
- **Expected**: old code rejected; latest works
- **Oracles**: reject with `VALIDATION.code.mismatch`; logs show new `code_id`

### S-301 **Exception/Validation** — Expired code
- **Expected**: message `VALIDATION.code.expired`
- **Recovery**: request new code → S-201

### S-302 **Exception/Policy** — Locked account
- **Expected**: `POLICY.account.locked`; suggest support path
- **Recovery**: unlock via support → retry S-201

---

## How many scenarios?

Aim for **1 Main + 2–3 Alt + 3–5 Exception** per feature slice.  
That’s usually **6–10 scenarios**. Each scenario can produce **1–3 cases** (e.g., boundaries or role variants), keeping your total case count **manageable**.

---

## Turning scenarios into cases

Use a thin mapping like:

| Scenario | Case example(s) |
|---|---|
| S-001 Main | `C-001`: Member + typical code; `C-002`: Guest + min length |
| S-103 Exception/Conflict | `C-010`: % code + gift card; `C-011`: fixed code + gift card (should allow if policy) |
| S-104 Exception/Transient | `C-020`: 1 timeout then success; `C-021`: N timeouts → user feedback |

Name cases `C-<nnn>` and link them to scenarios for **traceability**.

---

## Oracles & Evidence

- **Functional** — API response code/body; DB/event deltas; cart totals; session state.
- **UX** — message **IDs**, visible badges, disabled/enabled controls, focus behavior.
- **API Contract** — error taxonomy codes; idempotency result for retries.
- **Non-functional** — p95 latency for hot steps; retry counters; degraded modes.
- **Evidence** — logs with `scenario_id`, `correlation_id`, `error_code`; traces per step; metrics per path.

---

## Anti-patterns

- Only writing **Main**; no Exception with **recovery**.
- Steps that are **not observable** (e.g., “system validates” without any output).
- **Too many** scenarios from minor formatting variants (use one scenario; vary input inside cases).
- Mixing **policy** and **validation** categories so messages clash.
- No **role/state** preconditions; scenarios become ambiguous.

---

## Review checklist (quick gate)

- [ ] Main path written in ≤6 observable steps  
- [ ] ≥ 2 Alternative flows that still succeed  
- [ ] ≥ 3 Exception flows with **clear category** and a **recovery**  
- [ ] Preconditions/Triggers/Expected/Oracles present for each scenario  
- [ ] Message IDs mapped via **error taxonomy** (where applicable)  
- [ ] Links to **Boundary**, **Decision Table**, or **State Model** where relevant  
- [ ] Traceability: Scenario ↔ Cases present in PR

---

## CSV seeds

```csv
id,type,title,preconditions,trigger,expected,oracles,recovery
S-001,Main,Valid code applied,"cart>0","Apply","applied badge; total updated","200; log verify.ok; cart delta",
S-002,Alt,Apply after address,"member; address filled","Apply","applied","200; trace verify",""
S-003,Alt,Duplicate click idempotent,"applied once","Apply x2","same outcome; no duplicate","same key; one cart delta",""
S-101,Exception,Length overflow,"","Apply A*17","reject VALIDATION.code.length.exceeds; keep input","400; message id","edit to <=16"
S-102,Exception,Forbidden char,"","Apply SAVE!0","reject VALIDATION.code.charset","400; message id","remove !"
S-103,Exception,Not combinable,"gift card in cart","Apply","reject CONFLICT.code.not_combinable; offer remove","409; ui hint","remove gift card"
S-104,Exception,Timeout then retry,"latency spike","Apply","eventual apply; single discount","metrics retry_count; same key","auto retry <=N"
```

---

## Links

- Boundary & Equivalence: `../20-techniques/boundary-and-equivalence.md`  
- Decision Tables (rules like combinability): `../20-techniques/decision-tables.md`  
- State Models (idempotency/retries): `../20-techniques/state-models.md`  
- Error Taxonomy → UX: `../40-api-and-data-contracts/error-taxonomy.md`  
- Checklists: `../60-checklists/*`
