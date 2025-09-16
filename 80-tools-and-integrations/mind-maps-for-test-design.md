# Mind Maps for Test Design (tool-agnostic)

> Mind maps help teams **see the space**: features, flows, risks, data, environments.  
> This guide makes mind maps actionable—bridging sticky-notes to **scenarios**, **cases**, and **evidence** in this repo.

---

## TL;DR

- Use mind maps to **discover** and **decompose** before you write cases.  
- Keep branches **orthogonal**: **Flows**, **Data**, **Rules**, **Risks**, **Environments**, **Non‑functional**.  
- Tag leaves with **stable IDs** (`SCN-*`, `CASE-*`, `RISK-*`) and **owners**.  
- Export nodes to **CSV/JSON** and sync with your scenario/case registers.  
- Always map MAE (**Main / Alternative / Exception**) into the tree.

---

## Why mind maps for testing?

- Make hidden assumptions visible; align **PM × Eng × QA × Design × SRE**.  
- Encourage **breadth first** thinking (coverage) before depth (steps).  
- Create a shared language for **risk-based prioritization**.  
- Faster onboarding: one glance shows **what matters** and **what’s missing**.

---

## Core structure (recommended)

Root = **Feature/Capability**  
1. **Flows** — user journeys, system sequences, state transitions  
2. **Data** — input classes, boundaries, equivalence partitions  
3. **Rules** — business constraints, eligibility, calculations  
4. **Risks** — failure modes, abuse, security/privacy  
5. **Environments** — browsers/devices/OS/locales/tenants  
6. **Non‑functional** — performance, resiliency, a11y, compatibility  
7. **Integrations** — upstream/downstream, webhooks, idempotency  
8. **Observability** — logs, metrics, traces (message IDs, SLOs)  
9. **Acceptance (MAE)** — main/alt/exception scenarios → **SCN-***  
10. **Evidence** — artifacts to collect (HAR, screenshots, traces)

> Keep labels short. Detailed prose belongs in the corresponding `*/cases.md` or `*/scenarios.md`.

---

## Notation & tagging

- `[…]` for **IDs**: `[SCN-101-MAIN-01]`, `[CASE-UI-3DS-Return-001]`, `[RISK.DOS]`.  
- `{owner:Team}`, `{prio:P0..P3}`, `{gate:G1..G6}` as key:value tags.  
- Use **verbs** for flow nodes (“Apply code”, “Remove”, “Retry”).  
- Use **nouns/sets** for data nodes (“Code format”, “Currency”, “Subtotal buckets”).

---

## Example (Mermaid mindmap)

```mermaid
mindmap
  root((Checkout — Discount Code))
    Flows
      Apply code [SCN-101-MAIN-01]{owner:Checkout}{prio:P1}
      Remove code [SCN-101-MAIN-02]{owner:Checkout}{prio:P2}
      Place order w/ code [SCN-101-MAIN-03]{owner:Payments}{prio:P0}
      Retry after 429 [SCN-101-EXC-03]{owner:Security}{prio:P1}
    Data
      Code format
        ASCII A–Z0–9
        Length 3–32
        Unicode normalize
      Constraints
        Min subtotal buckets
        Allow/Block lists
        Currency (fixed)
    Rules
      Rounding half-even
      Caps ≤ eligible subtotal
      Tax after discount (flag)
    Risks
      Abuse/Enumeration [RISK.ABUSE]
      PSP Timeout [RISK.DEPENDENCY]
      Idempotency conflict [RISK.CONFLICT]
    Environments
      Browsers: Chrome/Safari/FF
      Mobile Web (iOS/Android)
      Locales: en, ar (RTL)
    Non-functional
      Perf p95≤250ms
      A11y focus & SR
      Compat matrix snapshot
    Integrations
      Promo Service
      Tax
      Catalog
      Webhooks
    Observability
      Logs MSG.discount.*
      Metrics latency/attempts
      Traces promo.validate
    Acceptance (MAE)
      Main
        Valid percent [SCN-101-MAIN-01]
      Alternative
        Free shipping [SCN-101-ALT-02]
        Fixed cap [SCN-101-ALT-01]
      Exception
        Rate limit [SCN-101-EXC-03]
        Expired/Not started [SCN-101-EXC-02]
    Evidence
      HAR apply/remove
      Screens (light/dark, LTR/RTL)
      Metric snapshots p95/p99
      Trace exemplar
```

> Use Mermaid when your platform renders it. Otherwise, keep the outline in Markdown and export to CSV.

---

## Example (FreeMind/XMind compatible outline)

```
Checkout — Discount Code
  Flows
    Apply code [SCN-101-MAIN-01] {prio:P1}
    Remove code [SCN-101-MAIN-02] {prio:P2}
    Place order [SCN-101-MAIN-03] {prio:P0}
    Retry after 429 [SCN-101-EXC-03]
  Data
    Code format -> ASCII A–Z0–9; length 3–32; normalize
    Constraints -> min subtotal; allow/block; currency
  Rules -> rounding half-even; cap ≤ eligible; tax flag
  Risks -> Abuse/Enumeration [RISK.ABUSE]; PSP Timeout [RISK.DEPENDENCY]
  Environments -> Chrome/Safari/FF; iOS/Android; en/ar
  Non-functional -> perf p95≤250; a11y focus; compat
  Integrations -> Promo; Tax; Catalog; Webhooks
  Observability -> MSG.discount.*; latency; traces
  Acceptance -> MAE (Main/Alt/Exc)
  Evidence -> HAR; Screens; Metrics; Trace
```

---

## From mind map → scenarios & cases (workflow)

1) **Draft** the mind map with cross‑discipline group (30–45 min).  
2) Tag **MAE scenarios** as `[SCN-*]` under Acceptance/Flows.  
3) For each `[SCN-*]`, add **1–n** `[CASE-*]` in a **Case Backlog** branch or in your `*/cases.md`.  
4) Attach **signals** under Observability (message IDs, metrics, traces).  
5) Export to **CSV** and commit alongside `scenarios.csv` / `cases.csv`.  
6) CI validates IDs and references (see `65-review-gates-…/traceability.md`).

---

## Export formats

**CSV (nodes → scenarios)**

```csv
scenario_id,req_id,type,title,owner,priority,tags
SCN-101-MAIN-01,REQ-Checkout-101,MAIN,"Apply code",Checkout,P1,"flows,discount"
SCN-101-ALT-01,REQ-Checkout-101,ALT,"Fixed cap",Checkout,P2,"data,calc"
SCN-101-EXC-03,REQ-Checkout-101,EXC,"Rate limit",Security,P1,"risk,abuse"
```

**CSV (nodes → cases)**

```csv
case_id,scenario_id,type,title,owner,repo_link
API-Apply-Valid-001,SCN-101-MAIN-01,API,"Apply SAVE15",QA,../../70-mini-projects/checkout-discount-code/cases.md
UI-FreeShip-001,SCN-101-ALT-02,UI,"Free shipping eligibility",QA,../../70-mini-projects/checkout-discount-code/cases.md
```

**JSON (generic mind map export)**

```json
{
  "root": "Checkout — Discount Code",
  "nodes": [
    { "path": ["Flows","Apply code"], "id": "SCN-101-MAIN-01", "owner": "Checkout", "prio": "P1" },
    { "path": ["Risks","PSP Timeout"], "id": "RISK.DEPENDENCY" }
  ]
}
```

---

## Workshop script (60 minutes)

- **0–5**: Frame outcomes, constraints, risks.  
- **5–15**: Map **Flows** (Main path first).  
- **15–25**: Add **Alt/Exception** branches; call out **idempotency** and **retries**.  
- **25–35**: Map **Data** partitions & **Rules**; capture boundaries.  
- **35–45**: Add **Risks**, **Environments**, **Non‑functional**.  
- **45–55**: Tag **MAE** → `SCN-*`; assign owners & priorities.  
- **55–60**: Decide **next actions**: which `[SCN-*]` → `[CASE-*]` this sprint.

---

## Patterns that work

- **One root = one capability**; big products → multiple maps.  
- **MAE baked in**: each main flow must have at least one alt & one exception.  
- **Risk colors** (if your tool supports): P0 red, P1 amber, etc.  
- **Short nouns/verbs**; details link to repo markdown.  
- **Link** nodes to **observability** early—design tests with signals.

---

## Anti-patterns

- Turning the map into a **dumping ground** of paragraphs.  
- Mixing **UI widgets** with **behavior**; prefer behavior.  
- Duplicating the same scenario under multiple branches (use **tags**).  
- No owners/priorities; maps get stale.

---

## Automation (convert map → registers)

**Pseudocode**

```python
def walk(node, path):
    items = []
    cur = path + [node["title"]]
    if "id" in node and node["id"].startswith("SCN-"):
        items.append(scn_row_from(node))
    for child in node.get("children", []):
        items += walk(child, cur)
    return items

# Write scenarios.csv and cases.csv; validate IDs via CI.
```

---

## Review checklist

- [ ] Branches cover **Flows**, **Data**, **Rules**, **Risks**, **Environments**, **Non‑functional**, **Observability**.  
- [ ] Each **Main** flow has at least **one Alt** and **one Exception**.  
- [ ] Leaves have **IDs**, **owners**, **priorities**.  
- [ ] Observability (logs/metrics/traces) present for critical paths.  
- [ ] Exports committed (`mindmap.json/csv`) and validated by CI.  
- [ ] Links from nodes to **repo pages** exist for details.

---

## Links

- Coverage Thinking → `../10-fundamentals/coverage-thinking.md`  
- Flows & States → `../10-fundamentals/flows-and-states.md`  
- Risk-based Prioritization → `../10-fundamentals/risk-based-prioritization.md`  
- Boundary & Equivalence → `../20-techniques/boundary-and-equivalence.md`  
- Traceability → `../65-review-gates-metrics-traceability/traceability.md`  
- Mapping to Test Managers → `./mapping-to-test-managers.md`

---
