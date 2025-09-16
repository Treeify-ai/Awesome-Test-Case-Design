# Glossary (Quick, Practical)

> Short, practitioner-friendly definitions you can use during design reviews and when writing cases.  
> Tip: skim this once; link back when terms pop up in playbooks or checklists.

---

## Notation & Abbreviations
- **MAE** — Main / Alternative / Exception (scenario families).
- **MECE** — Mutually Exclusive, Collectively Exhaustive (no overlap, no gaps).
- **SUT** — System Under Test.
- **I/O** — Input / Output.
- **RACI** — Responsible, Accountable, Consulted, Informed (ownership).

---

## Core Test-Design Concepts
- **Coverage thinking** — Structuring the space of behaviors so tests are *representative* and *sufficient* (not every permutation).
- **Equivalence classes** — Group inputs that should behave the same; test one per group + boundaries.
- **Boundary value analysis** — Hit “just in / just out” values around each limit (e.g., −1, 0, 1).
- **Decision table** — Tabular rules: conditions × actions. Ensures no conflicting or missing rules.
- **State model** — Valid states + transitions + guards; tests verify allowed/blocked moves and side-effects.
- **Pairwise testing** — Cover all pairs across factors; reduces explosion while catching many interaction bugs.
- **CRUD grid** — For each resource: Create/Read/Update/Delete × roles × constraints.
- **Oracle (test oracle)** — Mechanism to decide pass/fail (calc, invariant, golden file, service contract).
- **Traceability** — Links across Requirement ↔ Scenario ↔ Case ↔ Evidence; often a matrix (RTM).
- **Risk-based prioritization** — Test higher business/technical risk first (impact × likelihood).

---

## Scenario & Workflow
- **Main/Alt/Exception flows (MAE)** — Happy path, common variations, and error/edge paths.
- **Invariants** — Conditions that must always hold (e.g., sum of line items = order total).
- **Idempotent action** — Safe to repeat: same key/request ⇒ same outcome (e.g., retry POST with idempotency key).
- **Race condition** — Outcome depends on timing/order of concurrent events.
- **Idempotent consumer** — Consumer deduplicates replays, guaranteeing single-effect processing.

---

## API & Data Contracts
- **Contract test** — Verify provider/consumer agree on schema, types, required fields, and error shapes.
- **Schema validation** — Enforce structure (e.g., JSON Schema/OpenAPI) before deeper checks.
- **Pagination (offset vs cursor)** — Offset is page index; cursor is opaque pointer for stable traversal.
- **Consistency guarantees** — Strong / Read-after-write / Eventual; dictates assertion timing.
- **Idempotency key** — Client-generated token to dedupe retried operations.
- **Error taxonomy** — Canonical set of errors (auth, validation, rate-limit, transient) mapped to UX messages.
- **HTTP semantics** — Safe (GET), idempotent (PUT/DELETE), unsafe (POST) per RFC intent.
- **ETag / If-Match / If-None-Match** — Conditional requests for concurrency control & caching.
- **Pagination invariants** — Sorting stability or cursor monotonicity to avoid skips/dupes across pages.

---

## Non-Functional (Perf, Resilience, Compatibility, A11y)
- **p95 / p99** — 95th/99th percentile latency; budgets set expectations beyond average.
- **TTFB / TTI** — Time to First Byte / Time to Interactive (web UX timing signals).
- **SLO / SLI / SLA** — Objective / Indicator / Agreement for reliability commitments.
- **Budget** — Agreed threshold (e.g., p95 ≤ 500 ms) used as an acceptance criterion.
- **Retry w/ backoff** — Reattempt with increasing delay; add jitter to avoid thundering herds.
- **Circuit breaker** — Stop calling a failing dependency until it recovers to protect the system.
- **Graceful degradation** — Reduced functionality with clear UX when dependencies fail.
- **Compatibility matrix** — Supported OS/Browser/Device/App versions and features.
- **Internationalization (i18n) / Localization (l10n)** — Build for multiple locales / adapt strings, formats, rules.

---

## Observability & Evidence
- **Structured logs** — Key-value logs parsable by machines (include correlation/trace IDs).
- **Metrics** — Numeric time-series (counters, gauges, histograms) for SLOs and budgets.
- **Tracing** — End-to-end spans for a request; proves where time/errors occur.
- **Correlation ID** — ID that ties logs/metrics/traces for a single workflow.
- **Evidence artifact** — Saved proof: log snippet, screenshot, export, API transcript validating the expected result.

---

## Security & Privacy
- **Authentication vs Authorization** — Who you are vs what you may do.
- **Least privilege** — Grant minimal permissions needed for a task.
- **Input validation & encoding** — Validate on trust boundary; encode at sink (HTML/SQL/OS).
- **Secret handling** — No secrets in logs; use vaults/kms; rotate regularly.
- **PII / Data minimization** — Store the least personal data necessary; mask in lower environments.
- **Audit trail** — Tamper-evident record of who did what, when, and why.

---

## Accessibility (WCAG)
- **POUR** — Perceivable, Operable, Understandable, Robust (WCAG principles).
- **Keyboard accessibility** — Full operation without a mouse (focus order, visible focus).
- **ARIA semantics** — Roles/states to aid assistive tech (use sparingly and correctly).
- **Color contrast** — Sufficient luminance difference for text/UI elements.
- **Alt text & labels** — Programmatic names for images, inputs, controls.

---

## AI-Augmented Test Design (Optional)
- **Grounding (explicit-only)** — Use *only* provided requirements/artifacts as sources; no outside guessing.
- **Schema discipline** — Outputs adhere to a specified JSON/CSV/Markdown schema.
- **Deduplication** — Avoid semantic or structural duplicates in generated scenarios/cases.
- **Human-in-the-loop** — Reviewer notes → regenerate loop to refine outputs.
- **Hallucination** — Fabricated content not present in sources; must be prevented by guardrails.

---

## Review Gates & Quality Signals
- **Gate** — A checklist or automated criterion that must pass before merging/releasing.
- **Defect leakage** — Escaped defects per release; track ↓ as design quality ↑.
- **Case effectiveness** — Share of defects caught by designed tests vs ad-hoc discovery.
- **Coverage diff** — Change in scenario/case coverage between versions (what’s new or removed).

---

## Tiny Examples (copy-pasteable)

- **Boundary set (inclusive 0–100):** `-1, 0, 1, 99, 100, 101`
- **Idempotency test:** Send POST `/payments` with `Idempotency-Key: K1` twice → **same** result; with `K2` → new payment.
- **Retry/backoff:** `100ms, 200ms, 400ms (+ jitter)`; abort after N attempts or on non-retryable errors.
- **Error taxonomy → UX:** `VALIDATION_ERROR.amount.invalid` ⇒ “Enter a whole number between 0 and 10,000.”

---

## See Also
- `10-fundamentals/*` for deeper theory
- `20-techniques/*` for playbooks
- `60-checklists/*` for gates
- `65-review-gates-metrics-traceability/*` for metrics & traceability
