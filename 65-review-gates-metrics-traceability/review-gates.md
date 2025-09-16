# Review Gates

> Ship faster by **reducing ambiguity** and **raising evidence**.  
> These gates define **when** a change moves forward, **what** artifacts must exist, and **how** metrics tie back to requirements.

---

## Why gates?

- Catch gaps early (spec → design → code → ship).  
- Create **traceable evidence** for decisions.  
- Make quality **visible**: contracts, tests, a11y, perf, security, privacy, i18n.  
- Standardize **what “ready” means** across teams.

---

## Gate taxonomy (lightweight)

**G0 — Intake**  
- Problem statement, value, owners, scope/out-of-scope.

**G1 — Spec Ready (DoR)**  
- Contracts sketched (I/O, states, errors).  
- MAE acceptance drafted with oracles.  
- NFR budgets defined (p95/p99, retries, a11y, privacy, i18n, compatibility).  
- Metrics & instrumentation defined.

**G2 — Design Ready**  
- MAE screens: Empty/Loading/Error designed.  
- Focus order map; microcopy with message IDs.  
- Responsive & RTL notes; token usage; reduced-motion variants.

**G3 — Impl Ready**  
- Test data/fixtures; flags; environments; runbooks.  
- Observability wiring plan (logs/metrics/traces).  
- Security & privacy checklist planned.

**G4 — Pre-merge**  
- Contracts validated by tests.  
- Functional + API + a11y + perf quick passes green.  
- Security, compatibility, i18n checks green.  
- Evidence attached to PR.

**G5 — Ship / Launch**  
- Canary plan + guardrails; rollback.  
- Docs updated; on-call briefed; dashboards live.

**G6 — Post-launch**  
- Error budgets healthy; KPIs move as expected.  
- Postmortem if thresholds breached; action items tracked.

> Keep gates **fast** and **automatable**. Prefer short, high-signal checks.

---

## Evidence ledger (per gate)

- **G1**: one-pager, contracts draft, MAE acceptance, metrics plan.  
- **G2**: annotated screens, focus map, locale/RTL samples.  
- **G3**: test data list, flags/config, runbooks links.  
- **G4**: test reports (functional/API/a11y), perf snapshot, security scans, logs/traces.  
- **G5**: canary dashboard, alert rules, rollback doc.  
- **G6**: KPI dashboard, incident notes, learnings.

---

## Automation (CI/CD gates)

**Recommended jobs**

- Schema/contract diff → block incompatible changes.  
- Unit/integration/e2e → MAE smoke.  
- API contract tests → request/response shapes, message IDs.  
- A11y (axe/pa11y) → threshold of 0 critical.  
- i18n pseudo-localization + screenshots → no clipping.  
- Perf micro-bench / route p95 guardrails.  
- Security: SAST, SCA/SBOM, secret scan; CSP/headers snapshot.  
- Lint for logs without PII; check for correlation_id/msgid.  
- Coverage **by risk**: core modules ≥ target.

---

## Decision policy (ship/hold)

- **Ship** when all **G4** checks pass and **guardrails** are defined for G5.  
- **Hold** when:  
  - Contracts unclear; MAE missing.  
  - Critical a11y/security/perf failures.  
  - No rollback or on-call plan.

---

## Traceability

Link **Requirement → Acceptance → Test → Signal → Owner**.

**Minimal fields**

- `req_id`  
- `accept_id`  
- `test_id`  
- `signal` (metric/log/trace/msgid)  
- `owner`

Use the CSV seed below and keep it in-repo.

---

## Dashboards (standard set)

- **User journeys**: success rate, p95/p99, error rate.  
- **Perf**: route histograms; exemplars (trace links).  
- **A11y/i18n**: violations count; missing keys; pseudo-loc fail count.  
- **Security**: auth failures, rate limits, CSP reports, webhook signatures.  
- **Reliability**: error budgets, burn rates, queue age.  
- **Business**: north-star + guardrails.

---

## PR template (gate-aligned)

```
Feature: <name>    Req: <req_id>   Owners: <PM/Eng/QA/Design/SRE>

G1 — Spec Ready
[ ] Contracts (I/O, states, errors, message IDs)
[ ] MAE acceptance + oracles
[ ] NFR budgets (perf, resiliency, security, privacy, a11y, i18n, compatibility)
[ ] Metrics & instrumentation plan

G2 — Design Ready
[ ] MAE screens (Empty/Loading/Error)
[ ] Focus order map, message IDs
[ ] Responsive + RTL + reduced-motion notes

G3 — Impl Ready
[ ] Test data & flags
[ ] Observability plan (logs/metrics/traces)
[ ] Runbooks/rollout outline

G4 — Pre-merge
[ ] Functional/API tests green
[ ] A11y quick pass
[ ] Perf snapshot
[ ] Security checks
[ ] i18n pseudo-loc screenshots
[ ] Evidence attached (links)

G5 — Ship
[ ] Canary plan + guardrails
[ ] Rollback runbook
[ ] Dashboards live; alerts wired

G6 — Post-launch
[ ] KPIs + error budgets ok
[ ] Incidents & learnings logged
```

---

## Gate config (YAML template)

```yaml
gates:
  - id: G1
    name: Spec Ready
    required:
      - contracts
      - mae_acceptance
      - nfr_budgets
      - metrics_plan
  - id: G2
    name: Design Ready
    required:
      - mae_screens
      - focus_map
      - i18n_notes
  - id: G3
    name: Impl Ready
    required:
      - test_data
      - observability_plan
      - runbooks
  - id: G4
    name: Pre-merge
    required:
      - tests_functional
      - tests_api
      - checks_a11y
      - checks_perf
      - checks_security
      - checks_i18n
      - evidence_links
  - id: G5
    name: Ship
    required:
      - canary_plan
      - rollback
      - dashboards_alerts
  - id: G6
    name: Post-launch
    required:
      - kpi_check
      - error_budget_check
      - learnings_logged
```

---

## CSV seeds

**Gate registry**

```csv
gate_id,name,owner
G0,Intake,PM
G1,Spec Ready,PM
G2,Design Ready,Design
G3,Impl Ready,Eng
G4,Pre-merge,QA
G5,Ship,Eng+SRE
G6,Post-launch,PM+SRE
```

**Evidence register**

```csv
gate_id,artifact,type
G1,one_pager,doc
G1,contracts_draft,doc
G2,annotated_screens,images
G2,focus_map,images
G3,test_data,csv
G3,runbook,doc
G4,test_reports,pdf
G4,perf_snapshot,images
G4,security_scans,zip
G4,logs_traces,links
G5,canary_dashboard,link
G5,rollback_doc,doc
G6,kpi_dashboard,link
G6,postmortem,doc
```

**Traceability matrix**

```csv
req_id,accept_id,test_id,signal,owner
REQ-101,ACC-101,API-Create-001,metric:orders_create_p95,Checkout
REQ-102,ACC-201,UI-Error-RTL-001,log:MSG.validation.email,Web
REQ-103,ACC-301,Perf-Spike-002,trace:checkout_slow_exemplar,SRE
```

**Guardrails (canary)**

```csv
metric,threshold,action
error_rate_pct,> 1%,rollback
latency_p95_ms,> +20 vs base,hold
unsafe_rate_pct,> 0,rollback
kpi_uplift_pct,< 0,hold
```

---

## Review checklist (quick gate)

- [ ] Gates defined and understood by owners  
- [ ] Evidence ledger populated per gate  
- [ ] CI/CD automation mapped to gates  
- [ ] Dashboards exist and are linked in PRs  
- [ ] Traceability CSV updated  
- [ ] Guardrails + rollback documented

---

## Links

- Functional Coverage → `../60-checklists/functional-coverage.md`  
- API Coverage → `../60-checklists/api-coverage.md`  
- Performance → `../60-checklists/performance-review.md`  
- Security → `../60-checklists/security-review.md`  
- Compatibility → `../60-checklists/compatibility-review.md`  
- Accessibility → `../60-checklists/accessibility-review.md`  
- For PMs → `../57-cross-discipline-bridges/for-pms.md`  
- For SREs → `../57-cross-discipline-bridges/for-sres.md`
