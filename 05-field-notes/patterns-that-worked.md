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

# Patterns That Worked

> Real, repeatable tactics from projects. Each pattern is a **short narrative** you can copy, adapt, and prove with metrics.  
> Format: **Context → Tactic → Steps → Evidence → Risks → Reuse checklist**.

Use these as living notes. If your team runs a variation, add a new card with your outcomes.

---

## How to contribute a pattern

1. Copy `_templates/field-note.md` into `05-field-notes/` and rename it meaningfully (e.g., `boundary-first-on-pricing.md`).  
2. Fill in **Context / Approach / Trade-offs / Outcome (metrics)**.  
3. Add a **Reuse checklist** and link any artifacts (sanitized).  
4. Open a PR and tag it with `field-note` and relevant domain tags (e.g., `api`, `payments`, `i18n`).

**Evidence ideas:** before/after defect rates, % coverage increase, time saved, incident reduction, user-facing metrics (refund accuracy, retried ops success).

---

## Pattern 1 — Boundary-first on high-risk fields

**Context**  
Long forms and price/quantity fields were leaking edge-case bugs (off-by-one, illegal chars).

**Tactic**  
Start design with **Boundary & Equivalence** for the 2–3 highest-risk fields *before* writing flows.

**Steps**  
1. Define domain ranges and encodings (e.g., `0–10,000`, integers only).  
2. Pick boundary set: `-1, 0, 1, 9,999, 10,000, 10,001`.  
3. Add **cross-field** constraint checks (e.g., `discount ≤ subtotal`).  
4. Turn into cases with **explicit expected results**.

**Evidence**  
- Defect leakage on pricing fields ↓ **42%** over two releases.  
- Review time per PR ↓ **~20 min** (pre-baked cases).

**Risks / anti-patterns**  
- Missing cross-field constraints; only testing single-field bounds.

**Reuse checklist**  
- [ ] Numeric & length limits captured  
- [ ] ±1 around each limit tested  
- [ ] Cross-field constraints included  
- [ ] Expected results observable

---

## Pattern 2 — Error taxonomy mapped to UX messages

**Context**  
Users saw inconsistent, vague error messages; support tickets spiked after releases.

**Tactic**  
Create a **canonical error taxonomy** and map each error to a **user-facing message**.

**Steps**  
1. Define taxonomy: `VALIDATION`, `AUTH`, `RATE_LIMIT`, `TRANSIENT`, `CONFLICT`, etc.  
2. Standardize error codes (e.g., `VALIDATION.amount.invalid`).  
3. Map each to UX text and recovery hint.  
4. Write tests asserting **code + message + hint**.

**Evidence**  
- Duplicate support tickets about “failed checkout” ↓ **30%**.  
- Tests caught 3 mismapped messages pre-release.

**Risks**  
- Drift between backend codes and frontend copy.

**Reuse checklist**  
- [ ] Error code catalog checked in  
- [ ] UX copy reviewed & versioned  
- [ ] Contract tests assert code + message

---

## Pattern 3 — Idempotency & retries for money-moving

**Context**  
Intermittent timeouts caused **double charges** on retries.

**Tactic**  
Enforce **idempotency keys** on payment/refund POSTs with **backoff + jitter** retries.

**Steps**  
1. Require `Idempotency-Key` for unsafe ops.  
2. Retry on `429/5xx/timeout` with `100ms, 200ms, 400ms (+ jitter)`.  
3. Assert **same key ⇒ same outcome**, **new key ⇒ new op**.  
4. Add consumer-side **dedupe** idempotency as a safety net.

**Evidence**  
- Duplicate-charge incidents dropped to **0** over 3 months.  
- Observability logs show matched keys across retries.

**Risks**  
- Forgetting to include idempotency on **refunds** as well as charges.

**Reuse checklist**  
- [ ] Keys required for POST  
- [ ] Retry policy documented & tested  
- [ ] Same-key contract tests included

---

## Pattern 4 — CRUD × Roles sweep for authorization edges

**Context**  
Role-based bugs kept escaping (e.g., “viewer” could export).

**Tactic**  
Run a **CRUD grid** across roles with a tiny **matrix of allowed/denied** cases.

**Steps**  
1. List resources and operations.  
2. For each role, mark **allow/deny** and rationale.  
3. Generate tests from the matrix (positive + negative).  
4. Assert **HTTP code + UX notice + audit trail**.

**Evidence**  
- AuthZ defects in production ↓ **60%**.  
- Faster onboarding of new features (copy an existing matrix).

**Risks**  
- Overlooking **indirect** routes (bulk actions, exports, API tokens).

**Reuse checklist**  
- [ ] Matrix covers CRUD + exports/imports  
- [ ] Negative tests per role  
- [ ] Audit log checks added

---

## Pattern 5 — Data variation grid for i18n & special chars

**Context**  
Intermittent failures on emoji/Chinese inputs; inconsistent normalization.

**Tactic**  
Use a **data grid** with variations for **charset, length, locale, normalization**.

**Steps**  
1. Build a table: ASCII / Latin-1 / UTF-8 (emoji), mixed scripts.  
2. Vary lengths: min / typical / max / overflow.  
3. Include normalization (NFC/NFD) and trimming cases.  
4. Assert storage, retrieval, and rendering invariants.

**Evidence**  
- i18n data-related incidents ↓ **70%**.  
- Found 2 DB collation misconfigs pre-release.

**Risks**  
- Only testing input; forgetting export/CSV & email templates.

**Reuse checklist**  
- [ ] Charset mix covered  
- [ ] Length & normalization  
- [ ] End-to-end rendering/export verified

---

## Pattern 6 — Cursor pagination with stability invariants

**Context**  
Users saw missing/duplicated items during rapid updates.

**Tactic**  
Adopt **cursor-based pagination** with **stable sort** and test invariants.

**Steps**  
1. Define sort keys and stability rule (e.g., `created_at DESC, id DESC`).  
2. Simulate inserts while paging.  
3. Assert **no duplicates, no skips** across pages.  
4. Verify **next/prev** cursor semantics.

**Evidence**  
- “Missing items” reports ↓ **95%** after enforcing cursor rules.  
- Found a race on secondary index before GA.

**Risks**  
- Sorting on non-unique keys without tiebreaker.

**Reuse checklist**  
- [ ] Stable (tie-broken) sort  
- [ ] Insert-while-paging test  
- [ ] Duplicate/skip assertions

---

## Pattern 7 — Observability contracts for verifiable tests

**Context**  
Tests flaked because outcomes weren’t externally observable.

**Tactic**  
Define **log/metric/tracing keys** as part of the requirement (observability contract).

**Steps**  
1. Choose **structured log keys** (e.g., `order_id`, `idempotency_key`, `status`).  
2. Emit **metric** for critical path (e.g., `refund.success`).  
3. Add **trace/span names** for each step.  
4. Tests assert evidence artifacts (log line or metric delta).

**Evidence**  
- Flaky checks stabilized; test runtime ↓ **18%** by removing sleeps.  
- Faster incident triage with correlation IDs.

**Risks**  
- Logging PII or secrets.

**Reuse checklist**  
- [ ] Structured logs (no secrets)  
- [ ] Metrics & traces defined  
- [ ] Evidence captured in tests

---

## Pattern 8 — p95 budgets as acceptance criteria

**Context**  
Perf regressions slipped in small PRs.

**Tactic**  
Set **p95 budgets** per scenario and gate with a lightweight check.

**Steps**  
1. Define budgets (e.g., `p95 ≤ 500 ms` for search).  
2. Seed a **synthetic** or micro-benchmark for the path.  
3. Fail the gate if p95 breaches by > X%.  
4. Track trend over time.

**Evidence**  
- p95 spikes caught pre-merge 4× in one quarter.  
- Smoother release cadence; fewer “hotfix Friday” events.

**Risks**  
- Measuring on noisy infra; use relative deltas or fixed rigs.

**Reuse checklist**  
- [ ] Budget documented  
- [ ] Synthetic or micro-bench in CI  
- [ ] Trend visualized

---

## Submit yours

- Keep stories short and **specific**.  
- Include at least one **metric**.  
- Offer a **reuse checklist**.  
- Sanitize data and remove confidential details.

Use `_templates/field-note.md` or `_templates/case-study.md`. Thank you for helping others ship better tests.

---

<p align="center">
  <sub>
    Built with <a href="https://treeifyai.com">Treeify</a> — AI test case design copilot.<br>
    <em>Structured, traceable, high-coverage test design — faster.</em>
  </sub>
</p>
