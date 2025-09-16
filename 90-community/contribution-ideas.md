# Contribution Ideas

> Help grow **Awesome Test Case Design v2** with real-world patterns, runnable examples, and evidence-driven practices.  
> This page curates **high‑impact ideas** and shows how to turn them into great PRs.

---

## What makes a great PR here

- **Evidence‑driven**: scenarios/cases include **oracles** and **signals** (logs/metrics/traces).  
- **Runnable or reusable**: checklists, fixtures, or CSV/JSON templates others can copy.  
- **Short prose, rich artifacts**: keep explanations crisp; attach examples.  
- **Consistent IDs**: map to `REQ ↔ SCN ↔ CASE ↔ Evidence` where relevant.  
- **Accessible**: a11y and i18n considered when adding UI examples.  

See: `65-review-gates-metrics-traceability/traceability.md`, `80-tools-and-integrations/observability-hooks.md`.

---

## Contribution lanes (pick one to start)

- **Docs & Fundamentals**  
  - Clarify concepts in `10-fundamentals/*` with small diagrams or examples.  
  - Add a 1‑page primer for newcomers in `00-start-here/`.

- **Patterns & Field Notes**  
  - Expand `05-field-notes/*` with “pattern that worked” + sample evidence.  
  - Add before/after diffs that improved coverage.

- **Mini‑Projects**  
  - Propose a realistic flow under `70-mini-projects/` with **MAE → cases**.  
  - Include fixtures, contract stubs, and an evidence bundle checklist.

- **Checklists & Gates**  
  - Improve `60-checklists/*` with tight, non‑redundant items.  
  - Add review gates that are easy to enforce in CI.

- **API & Contracts**  
  - Extend `40-api-and-data-contracts/*` with examples and error taxonomies.  
  - Add schema snippets that tools can validate.

- **Non‑functional**  
  - Add budgets and test ideas in `50-non-functional/*` (perf, resiliency, security, a11y, compat).

- **AI Augmentation**  
  - Add eval goldens or guardrail patterns in `75-ai-augmentation/*`.  
  - Contribute a tiny aggregator or validator snippet.

- **Tools & Integrations**  
  - Export templates and sync scripts in `80-tools-and-integrations/*`.  
  - Mind‑map → CSV helpers or path→ID mappers.

- **Teaching Kits**  
  - Slides/worksheets in `85-teaching-kits/*` for workshops or onboarding.  
  - Add a 15‑minute micro‑exercise.

- **Translations**  
  - Localize key docs (glossary, quickstart) with RTL checks.

---

## Beginner‑friendly ideas (good first PRs)

- Add 3–5 **glossary** entries with crisp definitions: `00-start-here/glossary.md`.  
- Write one **MAE scenario** plus a matching **case** for an existing mini‑project.  
- Tighten one checklist with **duplicates removed** and stronger verbs.  
- Add a **CSV template** under `80-tools-and-integrations/export-templates/`.  
- Contribute a **small mind‑map** (Mermaid or outline) for a feature and export a short `mindmap.csv`.

---

## Intermediate ideas

- Add **boundary/equivalence** examples for a tricky input domain (dates, money, Unicode).  
- Expand `30-scenario-patterns/*` with a new pattern and a 4‑example gallery.  
- Create a **non‑functional probe** (perf or resiliency) and document signals to assert.  
- Add **error taxonomy** examples that map to UI copy + HTTP responses.

---

## Advanced ideas

- New **domain playbook** in `55-domain-playbooks/*` (e.g., search relevance, identity & auth, content moderation).  
- Full **mini‑project** (brief → design‑notes → scenarios → cases → evidence bundle).  
- **Observability hooks** implementation snippet (pino/structlog/otel) with dashboards.  
- **AI eval harness**: goldens + aggregator + metrics registry tied to guardrails.

---

## Idea backlog (grab‑and‑go)

- Payments: **3‑D Secure** return path MAE + evidence.  
- Subscriptions: **proration** scenarios with tax edge cases.  
- Messaging: **webhook retries** and dedupe proofs.  
- B2B: **OAuth client rotation** failure modes.  
- Mobile: **offline queue** → replay semantics.  
- i18n: **bi‑di** and **narrow no‑break space** in price formatting.  
- A11y: **focus traps** and **SR live region** announcements.  
- Security: **rate‑limit tests** with Retry‑After assertions.  
- Perf: **cold vs warm cache** comparisons with p95/p99.

---

## How to propose

1. Open an issue titled: `Proposal: <short feature/pattern name>`.  
2. Include: goal, scope, acceptance, files you’ll touch, and evidence you’ll attach.  
3. Tag with labels: `good-first-issue`, `mini-project`, `docs`, `checklist`, `ai`, `nfr`.  
4. Link to similar pages to avoid overlap.

**Issue stub**

```
Goal: <value/outcome>
Scope: <docs | mini-project | checklist | integration>
Files: <relative paths>
Acceptance: <bulleted, testable>
Evidence: <what you’ll attach>
```

---

## PR checklist (quick)

- [ ] Short title & description.  
- [ ] IDs consistent (`REQ/SCN/CASE`) where applicable.  
- [ ] Oracles & signals named; evidence bundle list included.  
- [ ] No long sentences in tables.  
- [ ] Links are relative and valid.  
- [ ] Lint: headings start with `#`, fenced blocks closed.

---

## Evidence bundle (attach to PR)

- Screenshots or recordings (light/dark, LTR/RTL when relevant).  
- HAR or cURL + JSON responses.  
- Log snippets with `msgid` / `err.code` / `correlation_id`.  
- Metric screenshots (p95/p99, counters).  
- Trace links for slow/failed exemplars.  
- CSVs: seeds, registers, or results.

---

## Community & partnership

- Looking to **co‑develop** patterns or mini‑projects?  
  - Open an issue with `[Partnership]` in the title.  
  - Share your domain context and constraints.  
  - We’ll scope a small, high‑value addition first.  
- If you’re a **KOL/educator**, propose a **teaching kit** or a 15/30/60‑min workshop plan.

---

## Maintainers’ review guide (what we look for)

- Clear **value** for readers, not just opinions.  
- **Reproducible** steps and **verifiable** outcomes.  
- Scope fits the repo’s structure; filenames match conventions.  
- Content is **tool‑agnostic** unless the folder says otherwise.  
- Respect privacy & security; no proprietary data.

---

## Thanks

Small, **focused** PRs merge fastest. Big efforts welcomed—open a proposal first.
