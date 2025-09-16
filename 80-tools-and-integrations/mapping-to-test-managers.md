# Mapping to Test Managers (CSV/JSON templates)

> Connect this repo’s **REQ → SCN → CASE → Evidence** to popular test managers (TestRail, Jira Xray/Zephyr, Azure DevOps).  
> Use the **IDs in-repo as the source of truth**; push metadata + links to tools, and pull results back for traceability.

---

## Objectives

- Keep **stable IDs**: `REQ-*`, `SCN-*`, `CASE-*`.  
- Minimize duplication: store **steps & acceptance** in-repo, push **references & status** to tools.  
- Enable **round‑trip**: CI can update run results in tools and attach **evidence links**.

---

## What gets synced

- **Outbound → tools**  
  - Requirements (optional): title/link.  
  - Test cases: title, objective, type, priority, references (`REQ/SCN`), link to repo case file.  
  - Steps (short) or a link to full steps.  
- **Inbound ← tools**  
  - Test run results: status, defects, duration, environment, evidence URLs.  
  - Comments minimal; keep long notes in repo PRs.

---

## Core fields (canonical)

- `req_id`, `scenario_id`, `case_id`  
- `title`, `type` (API/UI/Perf/Sec/Compat/Data)  
- `priority` (P0..P3)  
- `references` (comma: `REQ-*,SCN-*`)  
- `owner`, `component`, `labels`  
- `repo_link` (URL to md)  
- `steps_short` (plain 1–5 lines)  
- `expected_short` (plain 1–3 lines)

> Full steps live in `70-mini-projects/.../cases.md` or relevant doc.

---

## Templates (CSV seeds)

### requirements.csv

```csv
req_id,title,owner,repo_link
REQ-Checkout-101,"Place card order",PM,https://example.com/repo/65.../traceability.md#REQ-Checkout-101
```

### scenarios.csv

```csv
scenario_id,req_id,type,title,owner
SCN-101-MAIN-01,REQ-Checkout-101,MAIN,"Frictionless",QA
```

### cases.csv

```csv
case_id,scenario_id,type,title,priority,owner,component,labels,references,repo_link,steps_short,expected_short
API-Apply-Valid-001,SCN-101-MAIN-01,API,"Apply SAVE15",P1,QA,"Checkout","discount,api","REQ-Checkout-101,SCN-101-MAIN-01",https://example.com/.../cases.md,"POST /discounts/apply; capture JSON","200 OK; totals updated"
```

### runs.csv (results back from tools)

```csv
run_id,case_id,status,duration_s,env,defects,evidence
R2025-09-16-001,API-Apply-Valid-001,passed,4,prod-like,JIRA-123,"link://traces/abc; link://har/123"
```

---

## TestRail mapping

**Import (Cases)**  
- CSV columns → TestRail fields:
  - `Title` → **Title**  
  - `Section` → **Section** (folder)  
  - `Template` → **Test Case (Steps)**  
  - `Type` → custom or built-in (Regression, Functional)  
  - `Priority` → P0..P3 map  
  - `References` → `REQ-*,SCN-*`  
  - `Custom.repo_link` → URL  
  - `Custom.case_id` / `Custom.scenario_id` / `Custom.req_id`  
  - `Custom.component`, `Custom.labels`  
  - `Preconditions` → short context  
  - `Steps` / `Expected` → **short** (full in repo)

**CSV (TestRail import)**

```csv
Title,Section,Template,Type,Priority,References,Custom case_id,Custom scenario_id,Custom req_id,Custom repo_link,Preconditions,Steps,Expected
"Apply SAVE15","Checkout/Discount","Test Case (Steps)","Functional","High","REQ-Checkout-101,SCN-101-MAIN-01","API-Apply-Valid-001","SCN-101-MAIN-01","REQ-Checkout-101","https://example.com/.../cases.md","Cart A ready","POST /discounts/apply","200 OK; totals updated"
```

**Runs (via API JSON)**

```json
{
  "run": {"name": "Sprint 42 — Checkout"},
  "results": [
    {
      "case_id": "API-Apply-Valid-001",
      "status": "passed",
      "elapsed": "4s",
      "comment": "Trace link: link://traces/abc",
      "custom_evidence": ["link://traces/abc","link://har/123"]
    }
  ]
}
```

---

## Jira + Xray mapping

**Issue types**  
- `Test` (Xray) for cases.  
- `Requirement` (Story/Epic) linked via **"Tests"** relation.  
- `Precondition` (optional).

**CSV (Xray Cloud import minimal)**

```csv
Summary,Issue Type,Test Type,Labels,Components,Priority,Requirement Keys,Manual Test Steps,case_id,scenario_id,req_id,repo_link
"Apply SAVE15","Test","Manual","discount,api","Checkout","High","REQ-Checkout-101;SCN-101-MAIN-01","Step: POST /discounts/apply; Expected: 200 OK","API-Apply-Valid-001","SCN-101-MAIN-01","REQ-Checkout-101","https://example.com/.../cases.md"
```

**Xray JSON (automated test with evidence links)**

```json
{
  "testExecution": { "fields": { "summary": "Sprint 42 Execution" } },
  "tests": [
    {
      "testKey": "TEST-123",
      "start": "2025-09-16T03:00:00Z",
      "finish": "2025-09-16T03:00:04Z",
      "comment": "Trace link: link://traces/abc",
      "status": "PASS",
      "evidence": [
        { "filename": "trace.txt", "url": "link://traces/abc" },
        { "filename": "har.har", "url": "link://har/123" }
      ]
    }
  ]
}
```

**Links**  
- `Requirement Keys` ← `REQ-*` (use Story/Epic keys if already in Jira).  
- Add `labels`: `case_id:API-Apply-Valid-001` for easy queries.

---

## Jira + Zephyr Scale mapping

**CSV (Zephyr Scale)**

```csv
Project Key,Name,Folder,Status,Priority,Type,Labels,Components,Description,Test Script (Steps),Expected Result,Links
PROJ,"Apply SAVE15","Checkout/Discount","Approved","High","Functional","discount,api","Checkout","API case","POST /discounts/apply","200 OK; totals updated","REQ-Checkout-101;SCN-101-MAIN-01"
```

**Execution (API JSON)**

```json
{
  "testCycle": { "name": "Sprint 42" },
  "executions": [
    {
      "testCaseKey": "PROJ-T1",
      "status": "Pass",
      "actualEndDate": "2025-09-16T03:00:04Z",
      "comment": "link://traces/abc"
    }
  ]
}
```

---

## Azure DevOps (ADO) Test Plans mapping

**Create Test Case (Work Item JSON)**

```json
[
  {
    "op": "add",
    "path": "/fields/System.Title",
    "value": "Apply SAVE15"
  },
  { "op": "add", "path": "/fields/System.Tags", "value": "discount; api; CASE:API-Apply-Valid-001" },
  { "op": "add", "path": "/fields/Microsoft.VSTS.TCM.Steps", "value": "<steps id=\"0\" last=\"2\"><step id=\"1\"><parameterizedString isformatted=\"true\">POST /discounts/apply</parameterizedString><parameterizedString isformatted=\"true\">200 OK</parameterizedString></step></steps>" },
  { "op": "add", "path": "/fields/Custom.RepoLink", "value": "https://example.com/.../cases.md" }
]
```

**Test Run Result (API JSON)**

```json
{
  "name": "Sprint 42 — Checkout",
  "results": [
    {
      "testCase": { "id": 12345 },
      "outcome": "Passed",
      "durationInMs": 4000,
      "comment": "Trace link: link://traces/abc"
    }
  ]
}
```

---

## Crosswalk (keywords only)

| Repo field | TestRail | Xray | Zephyr Scale | ADO |
|---|---|---|---|---|
| case_id | Custom field | Label/custom | Label/custom | Tag/custom |
| scenario_id | Custom field | Requirement Key/label | Link/label | Tag/custom |
| req_id | References | Requirement Key | Links | Link |
| repo_link | Custom URL | Custom field | Description link | Custom.RepoLink |
| steps_short | Steps | Manual Steps | Test Script | Steps |

> Long prose belongs in repo docs; tools store **short** fields + links.

---

## Round‑trip strategy

- **Source of truth**: repo.  
- **One‑way** push of metadata; **two‑way** for results.  
- Use `case_id` as **external key** in tools (custom field or label).  
- CI uploads **evidence** (traces, HAR, screenshots) and sets status.  
- Tools link back to repo `cases.md` for full steps.

---

## CI sync (pseudocode)

```python
for case in load_csv("cases.csv"):
    ensure_tool_case(case)  # create or update by case_id
for result in load_csv("runs.csv"):
    post_run_result(result)  # attach evidence links; set status
```

---

## Naming & conventions

- Sections/folders mirror repo paths (e.g., `70-mini-projects/checkout-discount-code`).  
- Priority map: `P0=Critical`, `P1=High`, `P2=Medium`, `P3=Low`.  
- Labels: `domain:payments`, `type:api`, `component:checkout`.

---

## Validation checks (pre‑sync)

- Unique `case_id` / `scenario_id` / `req_id`.  
- `references` contains known IDs only.  
- `repo_link` reachable (HEAD 200).  
- Table fields stay **short** (titles, phrases).  
- Steps/expected length within tool limits.

---

## Example: from repo → TestRail

1) Export `cases.csv` with short fields.  
2) Import to TestRail into `Checkout/Discount`.  
3) Add **custom fields** once: `case_id`, `scenario_id`, `req_id`, `repo_link`.  
4) CI posts results JSON with evidence link bundle.  
5) Reconcile by `case_id` nightly.

---

## Example: from repo → Jira Xray

1) Create CSV with columns above.  
2) Import tests; link to **Story/Epic** using `Requirement Keys`.  
3) Add label `CASE:<id>` for JQL queries.  
4) Use Xray **Test Execution** API to post results from CI.

---

## Pitfalls & tips

- Don’t paste full steps into tools; use a **link**.  
- Don’t let tools rename your case IDs.  
- Avoid per‑run screenshots in Jira comments; attach as **evidence** object or external link.  
- Keep **short** titles; long text breaks imports.  
- Version your imports (e.g., `Sprint 42`) and archive old runs.

---

## Links

- Traceability → `../65-review-gates-metrics-traceability/traceability.md`  
- API Coverage → `../60-checklists/api-coverage.md`  
- Review Gates → `../65-review-gates-metrics-traceability/review-gates.md`

---
