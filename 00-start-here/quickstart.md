# Quickstart

Welcome! This page gets you productive **today**—with time‑boxed tracks, role shortcuts, and a minimal repo setup. Pick one track, follow the steps, and ship a small improvement you can share.

---

## Time‑boxed tracks

### ⏱️ 10 minutes — “One technique, one feature”
**Goal:** apply a single technique on a real feature and produce 6–12 checkable cases.

1. Open **[`20-techniques/`](../20-techniques/)** and pick **one** (e.g., _Boundary & Equivalence_).
2. Choose a small, real target (e.g., “discount code” field).
3. Follow the playbook steps and write **explicit expected results**.
4. Run the **mini checklist** at the end of the playbook.
5. Save as `playground/<feature>/<technique>.md` (or add to your team notes).

**Deliverable:** a short Markdown file with the chosen technique, value sets, and expected results.

---

### ⌛ 60 minutes — “Scenario → Cases”
**Goal:** model flows, add edges, and produce a first set of cases with quality gates.

1. Map **MAE flows** (Main / Alternative / Exception) using [`30-scenario-patterns/main-alt-exception.md`](../30-scenario-patterns/main-alt-exception.md).
2. Add **edge conditions** from one technique (boundary/equivalence or decision tables).
3. If an API is involved, check **idempotency & retries** in [`40-api-and-data-contracts/idempotency-and-retries.md`](../40-api-and-data-contracts/idempotency-and-retries.md).
4. Turn scenarios into **test cases with explicit expected results**.
5. Run **coverage gates**:
   - Functional: [`60-checklists/functional-coverage.md`](../60-checklists/functional-coverage.md)
   - API (if applicable): [`60-checklists/api-coverage.md`](../60-checklists/api-coverage.md)

**Deliverable:** `scenarios.md` + `cases.md` under `playground/<feature>/`.

---

### 📆 2 hours/week — “4‑week improvement loop”
**Goal:** build repeatable muscle memory and measurable coverage gains.

- **Week 1 — Fundamentals:**  
  Read [`10-fundamentals/coverage-thinking.md`](../10-fundamentals/coverage-thinking.md) and apply to a feature.
- **Week 2 — Non‑functional angles:**  
  Add perf/resiliency/security using [`50-non-functional/`](../50-non-functional/).
- **Week 3 — API contracts:**  
  Validate schemas, errors, and idempotency in [`40-api-and-data-contracts/`](../40-api-and-data-contracts/).
- **Week 4 — Review & metrics:**  
  Use [`65-review-gates-metrics-traceability/`](../65-review-gates-metrics-traceability/) to compare coverage before/after.

**Deliverable:** a short “before/after” note under `05-field-notes/before-after-coverage.md` (or a new file using `_templates/field-note.md`).

---

## Role shortcuts

Pick the lane that matches your work today:

### 👤 Solo QA / SDET
- Start: `20-techniques/` → `30-scenario-patterns/`  
- Gate with: `60-checklists/functional-coverage.md`  
- Share a **field note** using `_templates/field-note.md`.

### 👥 Team Lead / QA Manager
- Define a **taxonomy** + naming rules in `65-review-gates-metrics-traceability/traceability.md`.  
- Add **review gates** in `65-review-gates-metrics-traceability/review-gates.md`.  
- Seed a **mini‑project** under `70-mini-projects/` and ask the team to contribute.

### 🧰 Vendor / Tooling partner
- Read `80-tools-and-integrations/mapping-to-test-managers.md`.  
- Propose an export mapping or integration idea.  
- See `90-community/partnership.md` for co‑marketing + credits.

### 📋 PM / Product
- Use `57-cross-discipline-bridges/for-pms.md` to write **testable acceptance criteria**.  
- Link PRD items → scenarios → cases (traceability starter in `65-*/traceability.md`).

### 👨‍💻 Developer
- Use `57-cross-discipline-bridges/for-developers.md` to define **observability contracts** (logs/metrics) that make results verifiable.  
- Add **error taxonomy → UX messages** with QA via `40-*/error-taxonomy.md`.

### 🛠️ SRE
- Translate SLOs into **synthetic checks** seeded by scenarios (`57-*/for-sres.md`).  
- Capture **failure modes** and retries/backoff in `40-*/idempotency-and-retries.md`.

### 🔐 Security / Compliance
- Map **threats → tests** with `57-*/for-security-compliance.md`.  
- Capture **evidence** for controls in `65-*/traceability.md`.

### 🎨 Design / UX
- Design **error & empty states** up front (`57-*/for-design-ux.md`).  
- Ensure **a11y acceptance** via `50-non-functional/accessibility-a11y.md`.

---

## Minimal repo setup

> You only need Git + a Markdown editor. The items below are optional helpers.

### 1) Clone
```bash
git clone https://github.com/Treeify-ai/Awesome-Test-Case-Design.git
cd Awesome-Test-Case-Design
```

### 2) Optional: local lint & link checks
- **Markdown lint (Node)**
  ```bash
  npm install -g markdownlint-cli2
  markdownlint-cli2 "**/*.md"
  ```
- **Link check (Docker)**
  ```bash
  docker run --rm -v "$PWD":/work ghcr.io/lycheeverse/lychee lychee --no-progress --verbose "./**/*.md"
  ```

> CI runs these automatically via `.github/workflows/markdown-lint.yml` and `link-check.yml`.

### 3) Use templates
Copy an appropriate template from `_templates/` and fill it:
- Technique → `_templates/technique.md`
- Scenario pattern → `_templates/scenario-pattern.md`
- Mini‑project → `_templates/mini-project.md`
- Checklist → `_templates/checklist.md`
- Field note / Case study / Postmortem → corresponding templates

Place new content in the matching folder (see the repo root README’s ToC).

### 4) Commit & PR
```bash
git checkout -b feat/<short-topic>
git add .
git commit -m "feat: add boundary examples for checkout discount"
git push origin feat/<short-topic>
```
Open a PR and complete the **PR template**. Label suggestions: `technique`, `scenario`, `checklist`, `mini-project`, `api`, `non-functional`, `beginner|intermediate|advanced`.

---

## First success checklist

- [ ] I applied **one** technique to a real feature.
- [ ] My cases include **explicit expected results**.
- [ ] I ran at least one **coverage checklist**.
- [ ] I wrote a **short field note** with outcomes or lessons.
- [ ] I shared it via PR (or a gist) to help the next person.

---

## Optional: Try Treeify (light touch)

Exploring AI‑augmented test design? [Treeify](https://treeifyai.com/?utm_source=github&utm_medium=awesome_repo&utm_campaign=readme_hero) can help you turn long PRDs into structured **test objects → scenarios → cases**, exportable to your tools—while you stay in control.

- Contact: **contact@treeifyai.com** (mention “Awesome repo” for early‑user perks).
