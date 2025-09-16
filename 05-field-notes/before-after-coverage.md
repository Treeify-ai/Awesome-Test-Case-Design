# Before / After Coverage

> A lightweight way to **show impact**. Capture a snapshot **before** you apply a technique or pattern, then an **after** snapshot once you’ve updated scenarios/cases/gates. Keep it to one page so reviewers and stakeholders can scan it in < 3 minutes.

---

## How to use this page

1. Pick a **feature** (e.g., checkout discount code) and note the **baseline** commit/branch.
2. Run the relevant **coverage checklists** in `60-checklists/*` and record results.
3. Apply **one or two techniques** (e.g., Boundary & Equivalence + MAE flows).
4. Update **scenarios/cases** and re-run gates.
5. Fill in the **After** snapshot and summarize **outcomes** (defects found, gates passing, time saved).

Link this note from your PR description so reviewers see the improvement.

---

## Snapshot Template

> Copy below into a new file under `05-field-notes/before-after-coverage-<feature>.md` and fill the blanks.

```
# Before / After — <Feature/Flow>

**Owner:** <name/role>  
**When:** <YYYY-MM-DD>  
**Baseline:** <commit/branch/tag>  
**After:** <commit/branch/tag>  
**Scope:** <area, endpoints, UI screens, roles>

## Before (Baseline)
- **Scenarios:** <#> (Main: <#>, Alt: <#>, Exception: <#>)
- **Cases:** <#>
- **Checklists/Gates:**  
  - Functional: <X>/<Total> pass  
  - API: <X>/<Total> pass  
  - Non-functional (perf/security/compat): <brief summary>
- **Gaps observed:** <bullets>

## Change Applied
- Techniques/Patterns: <e.g., Boundary & Equivalence, Error Taxonomy → UX>
- Files updated: <links to 20-*/30-*/40-*/60-*/70-*/>
- Notes: <short rationale, assumptions>

## After
- **Scenarios:** <#> (Main: <#>, Alt: <#>, Exception: <#>)
- **Cases:** <#>
- **Checklists/Gates:**  
  - Functional: <X>/<Total> pass  
  - API: <X>/<Total> pass  
  - Non-functional: <brief summary>
- **Evidence:** <links to artifacts: logs, API transcripts, screenshots>

## Outcomes
- Defects found/avoided: <# + short description>  
- Time saved in review/regression: <estimate>  
- Remaining risks / Follow-ups: <bullets>
```

---

## Example — Checkout Discount Code

**Owner:** QA (Solo)  
**When:** 2025-09-15  
**Baseline:** `feature/discount-code @ 7a1b2c3`  
**After:** `feature/discount-code @ 9d4e5f6`  
**Scope:** Web checkout (apply code), API `/discount/verify`, roles: Guest, Signed-in

### Before (Baseline)
- **Scenarios:** 4 (Main: 1, Alt: 1, Exception: 2)  
- **Cases:** 11  
- **Checklists/Gates:**  
  - Functional: **7/12** pass (missed: max length, charset mix, expired code)  
  - API: **4/9** pass (no idempotency check on verify; missing error taxonomy → UX mapping)  
  - Non-functional: perf budget **undefined**
- **Gaps observed:**  
  - No **boundary** tests for code length (min/max/overflow)  
  - No **i18n** (emoji/Chinese/accents) input variation  
  - Error codes not mapped to **user messages**; duplicates in support tickets

### Change Applied
- Techniques/Patterns: **Boundary & Equivalence**, **MAE flows**, **Error Taxonomy → UX**  
- Files updated:  
  - `20-techniques/boundary-and-equivalence.md` (worked example added)  
  - `30-scenario-patterns/main-alt-exception.md` (exception cases expanded)  
  - `40-api-and-data-contracts/error-taxonomy.md` (codes + messages)  
  - `60-checklists/functional-coverage.md` (length/charset items)  
  - `70-mini-projects/checkout-discount-code/*` (scenarios & cases)
- Notes: Added ±1 around length bound, mixed-charset values, and explicit expected results per error.

### After
- **Scenarios:** 7 (Main: 1, Alt: 2, Exception: 4)  
- **Cases:** 20  
- **Checklists/Gates:**  
  - Functional: **12/12** pass  
  - API: **8/9** pass (remaining: pagination consistency on list codes)  
  - Non-functional: set **p95 ≤ 500ms** for verify; micro-check added in CI
- **Evidence:**  
  - API transcript for `VALIDATION.code.length.exceeds` → “Enter a code of at most 16 characters.”  
  - Logs with `correlation_id` and `error_code` per failed attempt  
  - CI job link for perf check

### Outcomes
- **3 pre-release defects** found (overflow, unsupported charset, wrong UX copy)  
- **Review time −15 min/PR** (cases shared and reusable)  
- **Support tickets for “code not working” down ~25%** in the next release  
- Follow-ups: add **cursor pagination** for admin list; extend **role** matrix to Staff/Admin

---

## Minimal CSV for tracking (optional)

You can paste this into a sheet or export it for dashboards.

```csv
date,feature,baseline,after,scenarios_before,cases_before,scenarios_after,cases_after,functional_pass,api_pass,notes
2025-09-15,checkout-discount,7a1b2c3,9d4e5f6,4,11,7,20,12/12,8/9,"added boundary & equivalence, error taxonomy→UX; set p95≤500ms"
```

---

## Tips

- Keep “Before/After” **diffs small** and focused (one or two techniques).  
- Prefer **observable expected results** (messages, codes, UI state) over internal assertions.  
- Link **artifacts** (logs/metrics/traces) for faster review.  
- Add a single **metric** (defects found, gate passes, time saved) to prove value.

If your note helps others, consider converting it into a longer **field note** or **case study** using `_templates/field-note.md` or `_templates/case-study.md`.
