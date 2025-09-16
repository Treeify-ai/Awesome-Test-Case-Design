# Postmortems & Anti‑patterns

> Blameless, fast, and practical. Use this page to document **what failed**, **why**, and **how we changed our test design** so it never happens again.  
> Focus on artifacts and repeatable **test‑design updates** (scenarios, cases, gates), not individual blame.

---

## How to run a 30‑minute micro‑postmortem

**Goal:** capture enough signal to update scenarios, cases, and gates within the **same day**.

1. **Assemble evidence pack** (10 min)
   - Timeline of events (UTC+0), who/what, timestamps
   - Logs (structured), traces, metrics screenshots
   - Inputs/requests, outputs/responses, user message(s)
   - Rollback/build IDs, feature flags, config diffs

2. **Root cause & contributing factors** (10 min)
   - Technique: **5‑Whys** for depth, **Fishbone** for breadth (people/process/tech/data)
   - Categorize: Functional · API/Data Contract · Non‑functional · Security · i18n · Deployment · Observability

3. **Update test design** (10 min)
   - Add/modify **scenarios** (MAE) and **cases** with explicit expected results
   - Add/adjust **checklists/gates** (performance, security, API)
   - Link evidence → tests (traceability), and open a PR referencing this note

> Keep it **blameless**. Stick to **facts & artifacts**. Prefer **small, immediate** design changes over long essays.

---

## Template (copy/paste)

```
# Postmortem: <concise title>

**Date/time (UTC+0):**  
**Impact:** <users/apps affected, duration, severity, metrics>  
**Category:** Functional | API/Data Contract | Non-functional | Security | i18n | Deployment | Observability

## Context
Short description + what changed (deploy ID/flag/config).

## Timeline (UTC+0)
- t0 — <event>
- t+X — <event>

## Evidence Pack
- Logs: <paths or paste sanitized snippet>
- Traces: <span names / IDs>
- Metrics: <charts or numbers>
- Requests/Responses: <sanitized samples>

## Root Cause
Primary cause written as a single, testable statement.

## Contributing Factors
- <factor 1>
- <factor 2>

## Fix
- Code/config: <summary>
- Test design: <new scenarios/cases>
- Gates/checklists: <new or updated checks>

## Verification
How we confirmed the fix (tests, monitors, synthetic checks).

## Lessons & Anti-patterns
- <lesson>
- <anti-pattern>

## Updates to Repo
- Scenarios: <links under 30-*/ or 70-*/>
- Cases: <links>
- Checklists/Gates: <links under 60-*/ or 65-*/>
- Traceability: <link to matrix>

> Owner: <name or role> · Review: <peers> · Labels: field-note, postmortem, <domain tags>
```

---

## Guidance & Do/Don’t

**Do**
- Redact PII/secrets; keep examples **sanitized**.
- Pin everything to **timestamps** and **artifacts** (logs/metrics/traces/requests).
- Convert findings into **concrete tests & gates** same‑day.
- Use **relative repo links** so readers can jump to new content.

**Don’t**
- Hunt for blame; focus on **systems and tests**.
- Ship fixes without **verification evidence**.
- Write “lessons” that don’t change a checklist, scenario, or gate.

---

## Mapping findings → repo updates

- **Functional gaps** → `30-scenario-patterns/*` · `20-techniques/*` · `60-checklists/functional-coverage.md`
- **API/Data Contract** → `40-api-and-data-contracts/*` · `60-checklists/api-coverage.md`
- **Performance/Resilience** → `50-non-functional/*` · `60-checklists/performance-review.md`
- **AuthZ/Roles** → `30-scenario-patterns/roles-and-permissions.md`
- **i18n/Encoding** → `55-domain-playbooks/i18n-l10n.md` · `20-techniques/crud-grids.md`
- **Traceability** → `65-review-gates-metrics-traceability/*`

---

## Example 1 — Double refund due to missing idempotency

**Category:** API/Data Contract · Money‑moving  
**Impact:** 37 duplicate refunds over 4h; manual reconciliation required

### Context
Retries were added for refund API timeouts. Payment create used idempotency keys; **refunds did not**.

### Timeline
- 08:13 — spike in gateway timeouts (external)
- 08:14 — client retries POST `/refunds` without key → duplicate records
- 08:27 — support reports double payout
- 08:42 — flag refund path; begin rollback; start reconciliation

### Evidence Pack
- Logs show identical payloads with different request IDs
- DB: duplicate refunds referencing same charge ID
- Metrics: spike in refund count vs charge count

### Root Cause
**POST `/refunds` lacked an idempotency key contract; client retries created duplicates.**

### Contributing Factors
- Assumed “only charges need idempotency”
- No **contract test** for “same key ⇒ same result”

### Fix
- Require `Idempotency-Key` header on refunds; persist request hash
- Retry policy: `100ms, 200ms, 400ms (+ jitter)`, stop on non‑retryables  
- **Tests added**  
  - `40-api-and-data-contracts/idempotency-and-retries.md` examples → cases under `70-mini-projects/refund-workflow/cases.md`  
  - Contract test: same key returns **same refund**; new key creates **new refund**

### Verification
- Replayed traffic against staging with synthetic timeouts; no duplicates
- Audit logs show dedupe events

### Lessons & Anti‑patterns
- Anti‑pattern: idempotency only for “create” but not “reverse” operations
- Lesson: **money‑moving = both directions need idempotency + backoff**

### Updates to Repo
- Scenarios/Cases: `70-mini-projects/refund-workflow/*`
- Checklist: `60-checklists/api-coverage.md` (“Idempotency for unsafe ops incl. refunds”)
- Gate: add synthetic retry test in CI

---

## Example 2 — Missing/duplicated items from pagination

**Category:** Functional · Consistency  
**Impact:** Admin list view intermittently missed new items under load

### Context
Offset‑based pagination sorted by `created_at DESC` without a tiebreaker; rapid inserts reordered items.

### Timeline
- 14:05 — import job inserts 5k rows in 2m
- 14:08 — users report “disappearing items”
- 14:22 — reproduced locally with parallel inserts

### Evidence Pack
- SQL explains re‑ordering around identical `created_at` values
- Screen recordings of duplicates/skips while paginating

### Root Cause
**Unstable sort (no unique tiebreaker) allowed items to shift between pages.**

### Contributing Factors
- Offset pagination under high write rate
- No test that mutated data while paging

### Fix
- Switch to **cursor pagination** with `(created_at, id)` composite sort  
- **Tests added**  
  - Insert‑while‑paging scenario under `30-scenario-patterns/cross-feature-interactions.md`  
  - Assertions for **no duplicates, no skips** in `20-techniques/state-models.md` example

### Verification
- Synthetic writer during paging test → stable results
- Monitors show consistent page sizes

### Lessons & Anti‑patterns
- Anti‑pattern: sorting solely on non‑unique column
- Lesson: always add a **tiebreaker** and test with concurrent writes

### Updates to Repo
- Pattern note: `05-field-notes/patterns-that-worked.md#pattern-6`
- Checklist: add “Stable sort or cursor with tiebreaker” to `60-checklists/functional-coverage.md`

---

## Example 3 — Unicode normalization crash on search

**Category:** i18n/Encoding  
**Impact:** 502s for queries containing decomposed characters; user search failures (JP + emoji)

### Context
Backend normalized user input to NFC; index stored strings in **mixed** NFC/NFD; equality and LIKE behaved inconsistently.

### Timeline
- 03:12 — alert on 5xx for search endpoint
- 03:20 — found correlation with iOS input methods
- 04:05 — identified mixed normalization in import pipeline

### Evidence Pack
- Request samples with decomposed glyphs (NFD)
- DB samples: `é` stored as `e + ́` in some rows
- Trace spans show error path during collation compare

### Root Cause
**Normalization mismatch (NFC vs NFD) between query path and stored data** caused collation errors and misses.

### Contributing Factors
- No data‑variation tests for normalization
- Mixed DB collation across tables

### Fix
- Normalize to **NFC at ingest**; migrate affected rows
- Enforce DB collation consistently
- **Tests added**  
  - Data variation grid (charset/length/normalization) under `20-techniques/crud-grids.md`  
  - Domain playbook updates under `55-domain-playbooks/i18n-l10n.md`

### Verification
- Replay queries with NFD/NFC → same results
- Incident dashboards trend back to baseline

### Lessons & Anti‑patterns
- Anti‑pattern: testing only ASCII/UTF‑8 “happy” paths
- Lesson: include **normalization** cases in the grid by default

### Updates to Repo
- Checklist: “Normalization (NFC/NFD) covered” in `60-checklists/compatibility-review.md`
- Case pack added to `70-mini-projects/checkout-discount-code/cases.md` (remark field)

---

## Anonymization & privacy

- Replace user IDs/emails with neutral tokens (e.g., `user_12345`).  
- Mask amounts, account numbers, and secrets.  
- Remove external customer names unless you have explicit permission.  
- If in doubt, **generalize** the detail and keep the test‑design change exact.

---

## Metrics to track over time

- **Defect leakage** per release  
- **Coverage diff** (new/removed scenarios & cases)  
- **Time to contain** (t0 → mitigation)  
- **Mean time to test‑design update** (incident → PR merged)  
- **Recurrence rate** (same root cause category)

> Aim for: shorter time to contain, faster test‑design updates, and a falling recurrence rate per category.
