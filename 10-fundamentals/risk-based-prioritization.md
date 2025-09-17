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


# Risk‑Based Prioritization

> You can’t test everything. You **can** test the *right* things first.  
> Risk‑based prioritization helps you pick a **small, high‑leverage** set of scenarios and cases that cut the most risk for the least effort.

---

## 1) What “risk” means here

- **Impact** — What happens if this fails? (money, privacy, safety, compliance, reputation, operational cost)  
- **Likelihood** — How likely is failure? (complexity, novelty, churn, defect history, dependency reliability, concurrency, data variety)

> Prioritize **Impact × Likelihood**, not just feature size or visibility.

---

## 2) Scoring model (simple, practical)

Score each factor **1–5** (low→high). Weighting is optional; default is equal weights.

### Impact factors
- **Financial** (money movement, pricing, billing)
- **Privacy/Compliance** (PII, consent, retention)
- **Irrecoverable data** (loss/corruption)
- **User harm / safety / reputation**
- **Operational blast radius** (rollbacks, on‑call load)

### Likelihood factors
- **Complexity** (branches, statefulness, concurrency)
- **Novelty** (new code, new team, unfamiliar stack)
- **Churn** (recent changes touching the area)
- **Defect history** (hotspots)
- **Dependency fragility** (3rd‑party SLAs, rate limits)
- **Data variety** (i18n, encodings, large volumes)

**Risk score = Impact_avg × Likelihood_avg** (1–25)

| Score | Bucket | What to do |
|------:|:------:|------------|
| 16–25 | High   | Test now. Add gates. Add observability. |
| 9–15  | Medium | Cover key edges. Monitor closely. |
| 1–8   | Low    | Smoke only; defer advanced tests. |

> Heuristic override: **Money, privacy, and irrecoverable data are “High” by default** unless proven otherwise.

---

## 3) The triage board

Use this four‑cell planner to avoid analysis paralysis:

- **Now** (High) — design cases today; add checklist/gates.
- **Next** (Med) — backlog for this iteration; monitor.
- **Maybe** (Low) — park; re‑score if the design changes.
- **Never** — intentionally out‑of‑scope; write down why.

Keep the board in your PR description or planning doc.

---

## 4) From risk → test design

Map risk to **techniques** and **gates**:

| Risk driver | Design move | Example tests | Gate |
|---|---|---|---|
| Money movement | **Idempotency + retries** | same key ⇒ same result; refund cannot double‑pay | API checklist + synthetic retry |
| Privacy/PII | **Data handling & logs** | no PII in logs; consent flags respected | Security/privacy checklist |
| Irrecoverable data | **Backups & invariants** | restore path; invariant “sum totals match” | Data integrity gate |
| Concurrency | **State model + races** | double apply, cancel mid‑operation | Concurrency checklist |
| i18n/encoding | **Data variation grid** | NFC/NFD, emoji, RTL | Compatibility/i18n checklist |
| Fragile dependency | **Timeouts, backoff, circuit breaker** | degrade gracefully | Resilience checklist |
| Complex rules | **Decision tables** | conditions × actions complete & consistent | Rule coverage check |

---

## 5) Worked examples (fast)

### A) Refund workflow (POST `/refunds`)
- **Impact:** Financial 5, Irrecoverable 4, Reputation 4 → **4.3**  
- **Likelihood:** Novelty 3, Concurrency 4, Dependency 3 → **3.3**  
- **Risk ≈ 14.2 (High/Med)** → **Now**
- **Design:** state model + idempotency + retries; evidence via logs/metrics/traces.
- **Gates:** API coverage (idempotency for refunds), resilience (retry/backoff), audit trail.

### B) Search suggestions (typeahead)
- **Impact:** Mostly UX/latency → **2.2**  
- **Likelihood:** New code + external API → **3.0**  
- **Risk ≈ 6.6 (Low)** → **Maybe**
- **Design:** perf budget p95; fallback to cached list; minimal correctness checks.

### C) Marketing banner scheduler
- **Impact:** Low money/privacy; small reputational risk → **2.0**  
- **Likelihood:** Low complexity; few deps → **1.5**  
- **Risk ≈ 3.0 (Low)** → **Never/Maybe** with a smoke test.

---

## 6) Lightweight worksheet (copy/paste)

```
# Risk Card — <Feature/Scenario>

## Impact (1–5)
Financial: _  Privacy/Compliance: _  Irrecoverable data: _  Reputation/Safety: _  Operational: _
Impact_avg: _

## Likelihood (1–5)
Complexity: _  Novelty: _  Churn: _  Defect history: _  Dependency fragility: _  Data variety: _
Likelihood_avg: _

Risk = Impact_avg × Likelihood_avg = _  → Bucket: High | Medium | Low

## Design moves
Techniques: <boundary, state, decision table, pairwise, idempotency, variation grid, …>
Scenarios to write first: <…>
Gates/Checklists: <…>
Observability: <log keys, metrics, traces>
Out-of-scope (with rationale): <…>
Owner: <name>  Review: <peers>  Date: <YYYY-MM-DD>
```

---

## 7) CSV seed for backlog (import into a sheet)

```csv
id,item,type,impact_avg,likelihood_avg,risk,bucket,design_moves,gates,owner,status
R-001,Refund API,idempotent post,4.3,3.3,14.2,High,"state,idempotency,retry","api,resilience,audit",QA,Now
R-002,Search typeahead,ux perf,2.2,3.0,6.6,Low,"perf budget,fallback","perf",FE,Maybe
R-003,Discount code combinability,business rules,3.5,2.8,9.8,Medium,"decision table,boundary","functional,api",QA,Next
```

---

## 8) Stop rules (when are we “done enough”?)

- All **High** risk scenarios have:
  - ≥ 1 **exception** and ≥ 1 **recovery** path tested
  - ≥ 1 **edge input** (±1, invalid, conflicting)
  - **Oracles & evidence** are explicit (logs/metrics/traces)
- Medium risks have at least a **happy + edge** case.
- Low risks **smoke only** unless they escalate.
- **Perf/a11y budgets** measured on hot paths.
- You can state what’s **out‑of‑scope** and why.

---

## 9) Common pitfalls (and fixes)

- **Big Score Theater** — Excess spreadsheets. **Fix:** use the worksheet; time‑box to 10 minutes.
- **Visibility bias** — Loud UI gets tests; silent money path doesn’t. **Fix:** impact first.
- **One‑and‑done** — Scores rot. **Fix:** re‑score after major PRs/incidents.
- **No evidence** — “We tested it” without oracles. **Fix:** require observability keys.

---

## 10) Review checklist (quick gate)

- [ ] Impact factors assessed (money, privacy, irrecoverable data, reputation, ops)
- [ ] Likelihood factors assessed (complexity, novelty, churn, history, deps, data variety)
- [ ] Bucketed into **Now / Next / Maybe / Never** with rationale
- [ ] Design moves mapped to **techniques + gates + observability**
- [ ] **High** risks include **exception + recovery** scenarios
- [ ] Clear **stop rules** and **out‑of‑scope** notes

**Next:** apply the chosen techniques:  
- Boundary & Equivalence → `../20-techniques/boundary-and-equivalence.md`  
- MAE flows → `../30-scenario-patterns/main-alt-exception.md`  
- Idempotency & retries → `../40-api-and-data-contracts/idempotency-and-retries.md`  
- Gates & metrics → `../60-checklists/*` and `../65-review-gates-metrics-traceability/*`

---

<p align="center">
  <sub>
    Built with <a href="https://treeifyai.com">Treeify</a> — AI test case design copilot.<br>
    <em>Structured, traceable, high-coverage test design — faster.</em>
  </sub>
</p>
