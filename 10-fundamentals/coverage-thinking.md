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


# Coverage Thinking

> The goal isn’t to test everything — it’s to **design a small, representative set** that provably covers the space that matters. Coverage thinking is the discipline of **decomposing behavior** and selecting **high‑leverage test cases** with explicit oracles and evidence.

---

## 1) Mental model

A simple way to reason about any feature:

1. **System boundary** — what’s inside vs. outside (dependencies, contracts).
2. **Actors & roles** — who triggers behavior (user types, services, schedulers).
3. **States & transitions** — where the system can be and how it moves.
4. **Inputs & constraints** — values, formats, ranges, timing, ordering.
5. **Invariants & outcomes** — what must always hold; what changes; what is observable.
6. **Risks** — impact × likelihood; where failure hurts most.

> If you can’t point to a **state**, **invariant**, or **observable outcome**, you probably can’t make a reliable test.

---

## 2) Decompose MECE (no overlaps, no gaps)

Use **MECE** to break the space into **independent dimensions**. Common cuts:

- **Flows:** Main / Alternative / Exception (MAE)
- **Inputs:** ranges, formats, encodings, nullability, ordering
- **Roles/Permissions:** who can/can’t
- **State:** preconditions, lifecycle stage, feature flags
- **Time:** now vs. past/future, deadlines, rate limits, retries
- **Data lifecycle:** create → use → archive → delete; retention; exports
- **Environment:** mobile/desktop, browser/OS, locale, network
- **Dependencies:** upstream/downstream APIs, queues, caches

Keep the decomposition **shallow but meaningful**. Two or three levels is plenty for most features.

---

## 3) The coverage lattice (what to combine)

Think of coverage as a **lattice** of dimensions. You **don’t** take the full cartesian product. You combine the **few** that create **distinct behavior**.

Typical combinations:
- **Flow × Input edge** (e.g., Main flow + boundary length)
- **Role × Operation** (CRUD × roles matrix)
- **State × API contract** (retry while in “processing” state)
- **Perf budget × Scenario** (p95 ≤ budget on the hot path)

Small example:

| Dimension       | Values                                      |
|-----------------|---------------------------------------------|
| Flow            | Main, Alt (apply later), Exception (expired)|
| Input (code)    | min−1, min, typical, max, max+1             |
| Role            | Guest, Member                               |

Pick **6–12** combinations that change behavior (e.g., “Guest × expired × max length” is valuable; “Member × typical × Main” may be redundant).

---

## 4) Choose techniques on purpose

- **Boundary & Equivalence** — numeric/length/format constraints, fast ROI.
- **Decision Tables** — business rules with condition × action mapping.
- **State Models** — lifecycles, idempotency, cancellation, retries.
- **Pairwise/Combinatorics** — many factors where pair coverage is enough.
- **CRUD Grids** — resources × roles × constraints (authZ surfaces).

Map your dimensions to techniques; don’t mix techniques blindly.

---

## 5) Oracles & observability (make results checkable)

A case is only useful if **pass/fail** can be decided reliably.

- **Functional oracle** — DB delta, response body/code, emitted event.
- **UX oracle** — message text/ID, UI state, disabled control.
- **API oracle** — schema match, error taxonomy code, idempotency result.
- **Non‑functional oracle** — p95 metric, retry counter, circuit opened.
- **Evidence** — structured logs, metrics, traces, screenshots, exports.

Add **observability contracts** early: which log keys/metrics/traces prove the outcome?

---

## 6) Risk‑based prioritization (when to stop)

Score scenarios by **Impact × Likelihood** (H/M/L). Hit **H/H** first, then H/M.

- **Impact examples:** money movement, privacy breach, data loss, fraud.
- **Likelihood examples:** complex logic, new code, flaky dependency, concurrency, i18n.

Stop when:
- All **H/H** and **H/M** scenarios have at least one **edge input** covered.
- Every **exception flow** has **one negative** and **one recovery** path.
- **Budgets** (perf/a11y) are measured on hot paths.
- You can explain **what’s intentionally out-of-scope** (with rationale).

---

## 7) Worked example — Discount code (mini)

**Context**: Web checkout “Apply code”. Constraints: `len 1..16`, charset `[A–Z0–9-]`, expires at timestamp, not combinable with gift cards. Roles: Guest, Member.

**Decompose**
- Flow: Main (valid), Alt (apply later), Exception (expired / invalid / too long / not combinable)
- Inputs: len bounds, charset mix, case sensitivity, leading/trailing spaces
- Roles: Guest vs Member (saved codes)
- State: cart total, existing discounts
- API: `/discount/verify` (idempotent verify), error taxonomy → UX

**Select techniques**
- Boundary & Equivalence (length, charset, trimming)
- Decision table (combinability rules)
- API contract + Idempotency
- MAE flows

**Sample cases (extract)**
1. **Main × min length (1)** → accepts, shows applied badge, total updated. *Oracle:* response 200 + `applied=true`; log `code_len=1`.
2. **Exception × length max+1 (17)** → rejects with `VALIDATION.code.length.exceeds`; input preserved; focus stays. *Oracle:* message ID, no total change.
3. **Exception × expired** → `VALIDATION.code.expired`; explain expiry. *Oracle:* error code + message.
4. **Exception × not combinable with gift card** → `CONFLICT.code.not_combinable`; offer remove option.
5. **Alt × apply after address** (Member) → still valid; idempotent verify (same key ⇒ same result).

**Perf budget**: `/discount/verify` p95 ≤ 500 ms in CI synthetic check.

**Traceability anchors**
- Req: “Apply discount code”  
- Scenarios: MAE list above  
- Cases: explicit expected results per item  
- Evidence: API transcript, log with `correlation_id`, CI perf job link

---

## 8) Edge‑inventory template

Keep a living list per feature; pull from it whenever you add cases.

```
# Edge Inventory — <Feature>

## Inputs
- Range/length: <min,max,±1>
- Format/charset: <ascii, unicode, emoji, punctuation, normalization>
- Null/empty/whitespace: <…>
- Ordering/timing: <…>

## Roles/Permissions
- Role matrix assumptions
- Negative cases (deny + UX)

## State
- Preconditions (flags, lifecycle)
- Concurrency (races, retries)
- Idempotency (same key ⇒ same result)

## Dependencies
- Upstream/downstream failure modes
- Timeouts, backoff, circuit breaker

## Non‑functional
- Perf budgets (p95/p99)
- A11y acceptance
- Compatibility matrix (OS/Browser/Device/Locale)
```

---

## 9) Common anti‑patterns

- **Case dump** without a model — many cases, thin coverage.
- **Happy‑path bias** — no exception/recovery scenarios.
- **Input tunnel vision** — ignore roles, state, or time.
- **No oracles** — assertions on internals only; flaky & unverifiable.
- **Unobservable outcomes** — no logs/metrics/traces to prove it.

---

## 10) Review checklist (quick gate)

- [ ] MECE decomposition exists (flows/inputs/roles/state/time)
- [ ] At least one **exception** and one **recovery** scenario
- [ ] Inputs include **±1 boundaries** where relevant
- [ ] **Error taxonomy → UX** mapped for negative paths
- [ ] **Idempotency** and **retries/backoff** addressed (if applicable)
- [ ] **Perf/a11y** budgets set on hot paths
- [ ] Oracles & evidence are **explicit**
- [ ] Traceability: Requirement ↔ Scenario ↔ Case ↔ Evidence

---

## 11) Starter CSV (coverage sheet)

Use this to seed a quick coverage table.

```csv
req,scenario,case_id,role,flow,input_edge,state,oracle,evidence,priority
Apply discount code,Expired code rejected,C-001,Guest,Exception,expired,,message_id=VALIDATION.code.expired,api_log+ui_screenshot,H
Apply discount code,Max length accepted,C-002,Member,Main,len=16,,applied=true; total_updated,api_transcript,H
Apply discount code,Overflow rejected,C-003,Guest,Exception,len=17,,message_id=VALIDATION.code.length.exceeds,api_log,M
Apply discount code,Not combinable conflict,C-004,Member,Exception,with_gift_card,has_gift_card,code=CONFLICT.code.not_combinable,api_log,M
Apply discount code,Idempotent verify,C-005,Member,Alt,,processing,idempotent_same_key,api_transcript,H
```

---

## 12) Where to next?

- Learn **Boundary & Equivalence**: `../20-techniques/boundary-and-equivalence.md`  
- Model **MAE flows**: `../30-scenario-patterns/main-alt-exception.md`  
- Set **API contracts & idempotency**: `../40-api-and-data-contracts/*`  
- Gate with **checklists**: `../60-checklists/*` and **review metrics** in `../65-review-gates-metrics-traceability/*`

---

<p align="center">
  <sub>
    Built with <a href="https://treeifyai.com">Treeify</a> — AI test case design copilot.<br>
    <em>Structured, traceable, high-coverage test design — faster.</em>
  </sub>
</p>
