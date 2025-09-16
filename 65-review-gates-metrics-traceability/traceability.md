# Traceability (Req ↔ Scenario ↔ Case ↔ Evidence)

> If it matters, it must be **traceable**.  
> This guide turns requirements into **scenarios**, scenarios into **test cases**, and cases into **evidence** that ties back to **signals** (logs/metrics/traces). Minimal ceremony, maximal confidence.

---

## TL;DR (defaults we recommend)

- Use **MAE** structure per requirement: **Main / Alternative / Exception** scenarios.  
- Register every item with **stable IDs**: `REQ-*`, `SCN-*`, `CASE-*`, `SIG-*`.  
- Evidence is a **file or link** attached per case: screenshot/HAR/log/trace/CSV/report.  
- Connect to **signals**: `metric:*`, `log:MSG.*`, `trace:<id>`, `webhook:<event_id>`.  
- Keep a lightweight **CSV** in-repo; auto-check it in CI before merge.  
- Gate on **coverage**: all **acceptance** scenarios mapped to at least one **case** and one **signal**.

---

## Why traceability?

- Shrinks review time; clarifies **what “done” looks like**.  
- Surfaces **gaps** (e.g., missing Exception flow).  
- Makes audits simple; speeds postmortems with links to **evidence**.  
- Enables **automation** (bots can verify coverage).

---

## Minimal data model (fields)

- `req_id`: requirement identifier.  
- `req_version`: semver/date.  
- `scenario_id`: MAE slice under `req_id`.  
- `case_id`: executable test (manual or automated).  
- `signal`: metric/log/trace/webhook id pattern.  
- `evidence_uri`: link to artifact.  
- `owner`: team or person.  
- `gate`: lifecycle gate (`G1…G6`).  
- `status`: `planned | in_progress | passed | failed | waived`.  
- `notes`: short phrase; avoid paragraphs.

**YAML schema (reference)**

```yaml
trace_item:
  req_id: "REQ-101"
  req_version: "2025.09"
  scenario_id: "SCN-101-MAIN"
  case_id: "CASE-API-Create-001"
  signal: "metric:request_duration_ms{route='/v1/orders'}"
  evidence_uri: "link://traces/abc123"
  owner: "Checkout"
  gate: "G4"
  status: "passed"
  notes: "happy path"
```

---

## ID conventions

- Requirement: `REQ-<area>-<number>` (e.g., `REQ-Checkout-101`).  
- Scenario: `SCN-<req-number>-<MAIN|ALT|EXC>-<seq>` (e.g., `SCN-101-EXC-01`).  
- Case: `CASE-<type>-<short>-<seq>` (e.g., `CASE-UI-Form-Invalid-001`).  
- Signal: `metric:*`, `log:MSG.*`, `err:ERR.*`, `trace:*`, `webhook:*`.

---

## Building the map (incremental workflow)

**G1 — Spec Ready**  
- Create `REQ-*` and list **MAE** scenarios as `SCN-*`.  
- Attach acceptance oracles and **intended signals**.

**G2 — Design Ready**  
- Link UI states (Empty/Loading/Error), message IDs, focus map.

**G3 — Impl Ready**  
- Draft **cases** (`CASE-*`) with steps + expected signals; prep test data.

**G4 — Pre-merge**  
- Execute cases; attach **evidence URIs**; mark status.

**G5 — Ship**  
- Link canary **dashboards**; add **guardrail signals**.

**G6 — Post-launch**  
- Add incident/postmortem evidence; update notes.

---

## Scenario taxonomy (short list)

- **Main**: happy path, most frequent.  
- **Alt**: valid variations (3DS, empty list, resend OTP).  
- **Exception**: controlled failures (timeout, rate limit, authz denied).  
- **Cross-feature**: interactions across modules (pricing × inventory).  
- **Non-functional**: perf p95/p99, a11y, security, compatibility, privacy.

---

## Evidence types (canonical)

- UI: screenshots (light/dark, RTL), videos, focus maps.  
- Network: **HAR**, waterfalls.  
- Backend: logs (`msgid`), traces (`trace_id`), metrics screenshots.  
- Data: CSV exports, parity reports, drift charts.  
- Integrations: webhook trails, signature verify logs.  
- Docs: runbooks, header snapshots, scan reports.

> Prefer links that **don’t expire**. Keep a copy in-repo when possible.

---

## Signals (bind cases to oracles)

- `metric:request_duration_ms{route="/checkout"}`  
- `metric:ml_unsafe_rate_pct`  
- `log:MSG.checkout.placed`  
- `err:ERR.AUTHZ.scope`  
- `trace:abc123`  
- `webhook:event_id=evt_42`  
- `header:CSP frame-ancestors 'none'`

---

## Example: Checkout slice

**Requirement** `REQ-Checkout-101`  
- Value: users can place card orders; 3DS supported.  
- Budgets: `p95 <= 400ms`, error rate `< 1%`.  
- MAE scenarios:
  - `SCN-101-MAIN-01` Place order, frictionless 3DS.  
  - `SCN-101-ALT-01` 3DS challenge return path.  
  - `SCN-101-EXC-01` PSP timeout; idempotent retry.

**Traceability CSV (snippet)**

```csv
req_id,req_version,scenario_id,case_id,signal,evidence_uri,owner,gate,status,notes
REQ-Checkout-101,2025.09,SCN-101-MAIN-01,CASE-API-Order-Create-001,log:MSG.checkout.placed,link://traces/tx_abc,Payments,G4,passed,prod-like
REQ-Checkout-101,2025.09,SCN-101-MAIN-01,CASE-API-Order-Create-002,metric:request_duration_ms{route="/v1/orders"},link://dashboards/p95,Payments,G4,passed,p95
REQ-Checkout-101,2025.09,SCN-101-ALT-01,CASE-UI-3DS-Return-001,webhook:event_id,link://webhook/logs,Payments,G4,passed,exactly-once
REQ-Checkout-101,2025.09,SCN-101-EXC-01,CASE-API-PSP-Timeout-001,err:ERR.DEPENDENCY.timeout,link://logs/error,Payments,G4,passed,rollback
```

---

## CI automation (what the bot checks)

- Every `REQ-*` has ≥ 1 `SCN-*`.  
- Every `SCN-*` has ≥ 1 `CASE-*`.  
- Every `CASE-*` has ≥ 1 `signal` and `evidence_uri`.  
- No **unknown** signals (must match allowed prefixes).  
- No **waived** status without an **owner** + short `notes`.  
- **Diff**: if `req_version` changes, require re-run of `CASE-*`.

**Gate rule (YAML)**

```yaml
checks:
  require_mapping: true
  allowed_signal_prefixes: [metric, log, err, trace, webhook, header]
  forbid_status: [""]
  require_owner: true
  require_evidence_uri: true
```

---

## Queries (snippets)

**Find unmapped scenarios**

```sql
SELECT scenario_id
FROM scenarios
LEFT JOIN cases USING (scenario_id)
WHERE cases.case_id IS NULL;
```

**Coverage by gate**

```sql
SELECT gate, COUNT(*) AS cases, SUM(status='passed') AS passed
FROM traceability
GROUP BY gate;
```

**Drill from KPI to traces (concept)**

- KPI dip → `metric:*` → p95 exemplar → `trace:*` → DB span → PR link.

---

## Review checklist (quick gate)

- [ ] IDs stable; naming consistent.  
- [ ] Each acceptance **scenario** mapped to **case(s)**.  
- [ ] Each case has **signal** and **evidence**.  
- [ ] Owners present; statuses updated.  
- [ ] CSV validated in CI; no unknown signals.  
- [ ] Trace from requirement → evidence is click-thru.

---

## CSV seeds

**Registers**

```csv
# requirements.csv
req_id,version,owner,title
REQ-Checkout-101,2025.09,PM,"Place card order"

# scenarios.csv
scenario_id,req_id,type,title
SCN-101-MAIN-01,REQ-Checkout-101,MAIN,"Frictionless"
SCN-101-ALT-01,REQ-Checkout-101,ALT,"3DS challenge"
SCN-101-EXC-01,REQ-Checkout-101,EXC,"PSP timeout"

# cases.csv
case_id,scenario_id,type,owner
CASE-API-Order-Create-001,SCN-101-MAIN-01,API,QA
CASE-UI-3DS-Return-001,SCN-101-ALT-01,UI,QA
CASE-API-PSP-Timeout-001,SCN-101-EXC-01,API,QA

# traceability.csv
req_id,req_version,scenario_id,case_id,signal,evidence_uri,owner,gate,status,notes
```

---

## Templates

**Requirement one-pager (traceable)**

```
REQ: <id>  Version: <semver/date>
Value: <short>
Acceptance (MAE): <SCN-* list + oracles>
NFRs: <perf, resiliency, security, privacy, a11y, i18n, compatibility>
Signals: <metric/log/trace/webhook list>
Owners: <PM, Eng, QA, SRE, Design>
```

**Case skeleton**

```
CASE: <id>  Scenario: <SCN-id>  Type: <API|UI|Perf|Security|A11y|Compat|Data>
Pre: <fixtures, flags, accounts>
Steps: <1..n>
Expected: <short>
Signals: <metric/log/err/trace/webhook>
Evidence: <links>
```

**Evidence log (per PR)**

```
PR: <link>
Reqs: <REQ-*>
Cases run: <CASE-*, status>
Artifacts: <screens, HAR, logs, traces, csv>
```

---

## Common pitfalls

- Ambiguous IDs; renaming mid-flight.  
- Evidence not durable; links expire.  
- Missing **Exception** scenarios.  
- Signals too vague (no `msgid`/`err.code`).  
- CSV drifts from reality; no CI check.

---

## Sign-off

- [ ] Mapping files present and validated.  
- [ ] Coverage complete for MAE scenarios.  
- [ ] Evidence attached and durable.  
- [ ] Owners & gates current.  
- [ ] Dashboards link from **signals**.

---

## Links

- Review Gates → `./review-gates.md`  
- Metrics & Signals → `./metrics.md`  
- Functional Coverage → `../60-checklists/functional-coverage.md`  
- API Coverage → `../60-checklists/api-coverage.md`  
- For PMs (acceptance) → `../57-cross-discipline-bridges/for-pms.md`
