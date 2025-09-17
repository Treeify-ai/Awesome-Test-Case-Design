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


# Style Guide — tone, examples, citations

> Keep content **useful, testable, and skimmable**.  
> This guide standardizes tone, structure, examples, and how to cite sources across the repo.

---

## 1) Voice & tone

- **Helpful, concise, practical.** Teach patterns; avoid opinionated rants.
- **Evidence-first.** Prefer examples, oracles, and signals over long prose.
- **Neutral, non-salesy.** We may mention Treeify where relevant, but *useful > promotional*.
- **Inclusive & plain language.** Short sentences. Explain acronyms on first use.
- **Consistency.** Use the *same* terms across docs (see `00-start-here/glossary.md`).

**Do**

- “Assert `MSG.refund.completed` is present and `refund_outcome_total{state="completed"}` increments.”
- “Use **MAE**: Main, Alternative, Exception.”

**Don’t**

- “Our amazing, cutting-edge solution will revolutionize testing!!!”
- “Obviously the system just works.”

---

## 2) Structure & formatting

- Use **ATX headings**: `# H1`, `## H2`, `### H3`.
- Headings in **Sentence case** (“Risk-based prioritization”, not “RISK-BASED PRIORITIZATION”).
- One idea per paragraph; prefer lists for steps.
- **Tables**: keywords/phrases/numbers only—**no long sentences**. (Prose belongs in body.)
- Code fences with language tags: ```` ```json ```` / ```` ```yaml ```` / ```` ```python ````.
- Keep lines ≤ ~100 chars where reasonable.
- Use **relative links** within the repo; keep them stable.

**Folder & file names**

- Lowercase, **kebab-case**: `refund-workflow`, `risk-based-prioritization.md`.
- Ordered sections use numeric prefixes already (e.g., `10-fundamentals/`).

---

## 3) Patterns for examples

**Good example anatomy**

- A short **context** line (what and why).
- A minimal **example** (code, payload, or diagram).
- A clear **oracle** (what proves it worked).
- Optional **evidence** to attach (logs/metrics/traces/screenshots).

**Before → After (clarity)**

**Before**

> The refund should work normally unless there are problems with the provider, in which case you might see errors.

**After**

- **Given** `O100` captured `$100.00`  
- **When** `POST /v1/orders/O100/refunds { "amount_minor": 10000 }`  
- **Then** `state="approved"` → `completed`; metric `refund_outcome_total{state="completed"}` +1  
- **Else** (provider timeout) → `provider_pending`; retry; see `MSG.refund.submitted`

---

## 4) MAE, scenarios, and cases

- Use **MAE** consistently:
  - **MAIN** — expected happy path
  - **ALT** — valid variations
  - **EXC** — controlled failures / edges
- **Scenario**: user/system intent and outcome.
- **Case**: concrete preconditions, steps, expected, and **signals**.

**Scenario (short)**

```
RF-MAIN-001 — Full refund captured
Given O100 captured
When create refund 100.00
Then completed; remaining 0
Signals: MSG.refund.*, refund_outcome_total, trace link
```

**Case (short)**

```
API-Refund-Full-001
Pre: O100 captured 100.00
Steps: POST /refunds -> poll
Expected: completed; remaining 0
Evidence: JSON, logs, metric, trace
```

---

## 5) Citations & linking

**Internal (within repo)**

- Use **relative links** to files: `../../70-mini-projects/.../cases.md`.
- Link specific sections with headings anchors.

**External sources**

- Link to **primary sources** (standards, official docs) when quoting definitions or budgets.
- Format: *Title* — Site/Org (YYYY-MM-DD). Example:  
  `NIST SP 800-63B — NIST (2020-03-02)`
- Include **absolute dates** (ISO 8601) for time-sensitive claims.
- Avoid quoting more than a few lines; paraphrase and link.

**Images & diagrams**

- Use descriptive alt text. Include **source and license** when not original.
- For Mermaid diagrams, provide a **text fallback** (bullets) if possible.

---

## 6) Numbers, units, currency, dates

- **Dates**: ISO 8601 (`2025-09-16`). Include timezone when relevant (e.g., `UTC`, `UTC+08`).
- **Currency**: specify **minor units** in APIs and exact currency (`USD`, `EUR`).  
  - Rounding: state rule (e.g., half-even).
- **Performance**: p95/p99 in **milliseconds** unless stated.
- **Thresholds**: express oracles with exact comparators (e.g., `≤ 250 ms`).

---

## 7) Accessibility & i18n notes

- Text alternatives for images/diagrams.
- Keep color assumptions out of meaning.
- For UI examples, mention screen reader behavior where relevant.
- If contributing translations, keep **IDs and code** in English; localize prose.

---

## 8) Promotion & partnership mentions

- Limit to a **single** short call-out near the top of README and relevant pages.
- Keep tone **measured**; link to `./PARTNERSHIP.md`.
- No banners, popups, or repetitive CTAs.

**Allowed snippet (short)**

> **Partner with Treeify** — Co-build useful testing content and co-market responsibly. See **[PARTNERSHIP.md](../PARTNERSHIP.md)**.

---

## 9) Checklists & tables (brevity)

- Tables store **labels** and **numbers**, not essays.
- Keep column names short; use tooltips or footnotes sparingly.
- Example (OK):

| Case ID | Type | Priority |
|---|---|---|
| API-Apply-Valid-001 | API | P1 |

---

## 10) Code & data snippets

- Keep samples **minimal** and **runnable** (curl, JSON, YAML).
- Redact secrets; never commit real tokens.
- Prefer **pseudocode** where language/stack is not important.

---

## 11) Review checklist (for PRs)

- [ ] Tone: helpful, concise; no salesy language.  
- [ ] MAE/Scenario/Case structures used correctly.  
- [ ] Tables contain only keywords/phrases/numbers (no long sentences).  
- [ ] Oracles and signals named for each case/example.  
- [ ] Links: relative in-repo; external sources include title + date.  
- [ ] Code fences labeled; JSON/YAML/CSV valid.  
- [ ] Images/diagrams have alt text and license/source if external.  
- [ ] File/heading names follow case & naming rules.

---

## 12) Templates (copy-paste)

**Example citation block**

```
Further reading:
- NIST SP 800-63B — NIST (2020-03-02)
- WAI-ARIA Authoring Practices — W3C (2024-01-15)
```

**Evidence bundle list**

```
Attach: HAR, response JSON, log snippet (msgid/err.code), metric screenshot (p95/p99 or counter delta), trace link.
```

**Scenario + Case starter**

```
SCN-ID — <Short title>
Given …
When …
Then …
Signals: <msgid>, <metric>, <trace>

CASE-ID
Pre:
Steps:
Expected:
Evidence:
```

---

## 13) Common pitfalls

- Wall-of-text intros; prefer **purpose → example → oracle**.
- Vague expectations (“should work”). State **exact** fields or signals.
- Mixing tool-specific settings into generic docs.
- Long sentences inside tables (don’t).
- Overuse of adverbs and hype words.

---

## 14) Linting (lightweight)

- Headings start with `#`, not underlines.
- No trailing whitespace; newline at EOF.
- Validate JSON/YAML blocks before commit.
- Keep Mermaid fenced blocks closed; add text fallback.

---

## 15) Changelog & attribution

- Summarize changes in `CHANGELOG.md` with PR links.
- Credit contributors (handles) for notable additions.
- Respect the repo license (**CC BY-NC 4.0**) when copying/deriving content.

---

<p align="center">
  <sub>
    Built with <a href="https://treeifyai.com">Treeify</a> — AI test case design copilot.<br>
    <em>Structured, traceable, high-coverage test design — faster.</em>
  </sub>
</p>
