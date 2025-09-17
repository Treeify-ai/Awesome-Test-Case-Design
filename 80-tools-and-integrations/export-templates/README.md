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


# Export Templates (CSV/JSON) — add your mappings here

> Drop-in **CSV/JSON** stubs for pushing repo artifacts into your test manager of choice (TestRail, Jira Xray/Zephyr, Azure DevOps).  
> Keep **repo as source of truth**; tools mirror **IDs, titles, links, and short steps**.

---

## How to use this folder

1) **Copy** a template that matches your tool.  
2) **Fill** the rows with **short** fields (IDs, titles, phrases).  
3) **Link** back to the full markdown case in the repo (`repo_link`).  
4) **Sync** using your integration script (example pseudo below).  
5) **Round-trip** results: export **runs.csv / results.json** back into this folder.

> Tables/CSVs must remain **compact** (no long sentences). Prose stays in repo docs.

---

## Folder layout (suggested)

- `templates/` — untouched examples
  - `templates/testrail-cases.csv`
  - `templates/xray-tests.csv`
  - `templates/xray-results.json`
  - `templates/zephyr-cases.csv`
  - `templates/ado-cases.json`
  - `templates/runs.csv`
- `your-org/` — your customized exports
  - `your-org/sprint-42/testrail-cases.csv`
  - `your-org/sprint-42/xray-results.json`

---

## Minimal field dictionary (keywords only)

| Field | Meaning |
|---|---|
| `req_id` | Requirement ID (`REQ-*`) |
| `scenario_id` | Scenario ID (`SCN-*`) |
| `case_id` | Case ID (`CASE-*`) |
| `title` | Short title |
| `type` | API/UI/Perf/Sec/Compat/Data |
| `priority` | P0/P1/P2/P3 |
| `references` | Comma: `REQ-*,SCN-*` |
| `repo_link` | URL to markdown case |
| `steps_short` | 1–5 short lines |
| `expected_short` | 1–3 short lines |

---

## CSV templates (copy→edit)

### requirements.csv

```csv
req_id,title,owner,repo_link
REQ-Checkout-101,"Place card order",PM,https://your.repo/65-review-gates-metrics-traceability/traceability.md#REQ-Checkout-101
```

### scenarios.csv

```csv
scenario_id,req_id,type,title,owner
SCN-101-MAIN-01,REQ-Checkout-101,MAIN,"Frictionless checkout",QA
```

### cases.csv

```csv
case_id,scenario_id,type,title,priority,owner,component,labels,references,repo_link,steps_short,expected_short
API-Apply-Valid-001,SCN-101-MAIN-01,API,"Apply SAVE15",P1,QA,"Checkout","discount,api","REQ-Checkout-101,SCN-101-MAIN-01",https://your.repo/70-mini-projects/checkout-discount-code/cases.md,"POST /discounts/apply; capture JSON","200 OK; totals updated"
```

### runs.csv (results back from tools)

```csv
run_id,case_id,status,duration_s,env,defects,evidence
R2025-09-16-001,API-Apply-Valid-001,passed,4,prod-like,JIRA-123,"link://traces/abc; link://har/123"
```

---

## TestRail (import cases)

**CSV**

```csv
Title,Section,Template,Type,Priority,References,Custom case_id,Custom scenario_id,Custom req_id,Custom repo_link,Preconditions,Steps,Expected
"Apply SAVE15","Checkout/Discount","Test Case (Steps)","Functional","High","REQ-Checkout-101,SCN-101-MAIN-01","API-Apply-Valid-001","SCN-101-MAIN-01","REQ-Checkout-101","https://your.repo/.../cases.md","Cart A ready","POST /discounts/apply","200 OK; totals updated"
```

---

## Jira Xray (import tests)

**CSV**

```csv
Summary,Issue Type,Test Type,Labels,Components,Priority,Requirement Keys,Manual Test Steps,case_id,scenario_id,req_id,repo_link
"Apply SAVE15","Test","Manual","discount,api","Checkout","High","REQ-Checkout-101;SCN-101-MAIN-01","Step: POST /discounts/apply; Expected: 200 OK","API-Apply-Valid-001","SCN-101-MAIN-01","REQ-Checkout-101","https://your.repo/.../cases.md"
```

**Results JSON (execution)**

```json
{
  "testExecution": { "fields": { "summary": "Sprint 42 Execution" } },
  "tests": [
    {
      "testKey": "TEST-123",
      "status": "PASS",
      "comment": "Trace link: link://traces/abc",
      "evidence": [
        { "filename": "trace.txt", "url": "link://traces/abc" },
        { "filename": "har.har", "url": "link://har/123" }
      ]
    }
  ]
}
```

---

## Jira Zephyr Scale (import cases)

**CSV**

```csv
Project Key,Name,Folder,Status,Priority,Type,Labels,Components,Description,Test Script (Steps),Expected Result,Links
PROJ,"Apply SAVE15","Checkout/Discount","Approved","High","Functional","discount,api","Checkout","API case","POST /discounts/apply","200 OK; totals updated","REQ-Checkout-101;SCN-101-MAIN-01"
```

---

## Azure DevOps (ADO) Test Plans

**Work Item JSON (create case)**

```json
[
  {"op":"add","path":"/fields/System.Title","value":"Apply SAVE15"},
  {"op":"add","path":"/fields/System.Tags","value":"discount; api; CASE:API-Apply-Valid-001"},
  {"op":"add","path":"/fields/Microsoft.VSTS.TCM.Steps","value":"<steps id=\"0\" last=\"2\"><step id=\"1\"><parameterizedString isformatted=\"true\">POST /discounts/apply</parameterizedString><parameterizedString isformatted=\"true\">200 OK</parameterizedString></step></steps>"},
  {"op":"add","path":"/fields/Custom.RepoLink","value":"https://your.repo/.../cases.md"}
]
```

---

## Validation (pre-flight checks)

**CLI sketch (Python-like)**

```python
import csv, sys, urllib.parse as u

def short(s, n): return len(s or "") <= n
def url_ok(href): return bool(u.urlparse(href).scheme)

with open("cases.csv") as f:
    for row in csv.DictReader(f):
        assert row["case_id"].startswith(("API-","UI-","SEC-","PERF-","CASE-"))
        assert short(row["title"], 80)
        assert short(row["steps_short"], 140)
        assert short(row["expected_short"], 100)
        assert url_ok(row["repo_link"])
print("OK")
```

**Checklist**

- [ ] IDs unique (`REQ/SCN/CASE`).  
- [ ] `references` refer to known IDs.  
- [ ] `repo_link` reachable.  
- [ ] No long sentences in CSV cells.  
- [ ] Titles concise; steps/expected short.

---

## Tips & pitfalls

- Use **labels** like `CASE:API-Apply-Valid-001` for easy queries.  
- Don’t paste full steps into tools—link back to the repo page.  
- Keep import CSVs **UTF‑8** and **comma** separated.  
- Map priorities: `P0→Critical`, `P1→High`, `P2→Medium`, `P3→Low`.  
- Store **results** as CSV/JSON next to imports for audit.

---

## Links

- Mapping guide → `../mapping-to-test-managers.md`  
- Traceability → `../../65-review-gates-metrics-traceability/traceability.md`  
- Cases (examples) → `../../70-mini-projects/checkout-discount-code/cases.md`

---

<p align="center">
  <sub>
    Built with <a href="https://treeifyai.com">Treeify</a> — AI test case design copilot.<br>
    <em>Structured, traceable, high-coverage test design — faster.</em>
  </sub>
</p>
