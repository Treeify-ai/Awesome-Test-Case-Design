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


# Metrics & Quality Signals

> What you can’t see, you can’t ship with confidence.  
> This guide standardizes **SLIs/SLOs, KPIs, guardrails, and diagnostics** across web, mobile, backend, data/ML, and security—so every feature has **measurable quality** and **traceable evidence**.

---

## TL;DR (defaults we recommend)

- Separate **business KPIs**, **user-facing SLIs**, and **engineering diagnostics**.  
- Define **SLOs** for top journeys; manage error budgets with **multi-window burn rate** alerts.  
- Emit **RED** (Rate–Errors–Duration) and **USE** (Utilization–Saturation–Errors) by default.  
- Attach **exemplars** (trace IDs) to latency histograms for fast drill-down.  
- Tag everything with a **minimal label set**: `service`, `route`, `method`, `region`, `env`, `version`, `tenant_tier`, `synthetic`.  
- Own metrics: each time series has an **owner** and **runbook**.

---

## Vocabulary

- **KPI**: product or business outcome (conversion, revenue, retention).  
- **SLI**: user-observable quality signal (availability, latency, correctness).  
- **SLO**: target for an SLI (e.g., `p95 ≤ 300 ms` weekly).  
- **Error budget**: `1 - SLO` over a window (e.g., `0.1%` for 99.9%).  
- **Guardrail**: threshold that triggers rollback/hold (e.g., unsafe content rate `> 0`).  
- **Diagnostic**: supporting signals (queue depth, cache hit ratio).

---

## Naming & units

- Metric name: `area_subject_verb_unit` (snake_case).  
- Units in the name when unclear: `_ms`, `_pct`, `_count`, `_bytes`.  
- Types: **counter** (monotonic), **gauge** (instant), **histogram** (latency/sizes).

**Examples**  
- `request_duration_ms` (histogram)  
- `http_requests_total` (counter)  
- `queue_depth` (gauge)  
- `unsafe_content_rate_pct` (gauge)

---

## Labels (keep cardinality in check)

Required: `service`, `route`, `method`, `region`, `env`, `version`.  
Optional (budgeted): `tenant_tier`, `synthetic`, `device`, `locale`, `feature_flag`.

> Avoid unbounded labels (`user_id`, raw URLs, query strings).

---

## Standard signals by layer

### Backend / APIs

- **SLIs**: availability `1 - 5xx_rate`, `request_duration_ms` p50/p95/p99, `errors_total{code}`.  
- **Diagnostics**: DB latency, cache hit ratio, queue age, worker saturation.

### Web Front-end

- **SLIs**: Core Web Vitals (**LCP**, **INP**, **CLS**) p75; SPA route `ttv_ms` (time-to-view).  
- **Diagnostics**: JS error rate, long tasks count, bundle size, failed requests.

### Mobile (iOS/Android)

- **SLIs**: cold start p95, warm start p95, **TTI** p95, crash-free users %, ANR rate.  
- **Diagnostics**: network payload before first interaction, dropped frames.

### Data / ML

- **SLIs**: prediction latency p95, win rate vs baseline, unsafe rate %, label delay.  
- **Diagnostics**: drift (PSI/KS), skew checks, feature freshness, cost/request.

### Security / Privacy

- **SLIs**: auth success rate, webhook signature failure rate, CSP violation rate, DSR SLA.  
- **Diagnostics**: rate-limit hits, abuse blocks, consent=false event rate.

---

## Error budgets & burn rate

**Budget** for an availability SLO of `99.9%` over 30d is `0.1%` unavailability.

**Fast/slow burn alerts (examples)**  
- **14×** over **5m/30m** windows (page immediately).  
- **6×** over **1h/6h** windows (investigate).

**PromQL (availability burn rate)**

```promql
# availability = 1 - error_rate
# error_rate over window W
sum(rate(http_requests_total{status=~"5.."}[5m])) 
  / sum(rate(http_requests_total[5m]))
```

---

## Exemplars & traceability

- Attach **exemplars** to histograms with `trace_id` for top p99 samples.  
- Include `correlation_id` in logs and headers (`X-Correlation-Id`).  
- Dashboard drill links: latency → exemplar trace → DB/cache spans.

---

## Ownership & runbooks

Each metric/SLO must have: **owner**, **dashboard**, **alert**, **runbook**, **on-call rotation**.

---

## Dashboards (minimum set)

- **User journeys**: success rate, p95/p99 latency, error rate.  
- **Perf**: route histograms with exemplars; top offenders.  
- **Reliability**: budgets & burn rates; saturation; queue age.  
- **Security**: auth failures, rate limits, CSP, webhook signatures.  
- **Data/ML**: drift, unsafe %, label delay, cost.  
- **Business**: KPI funnel + guardrails.

---

## Review checklist (quick gate)

- [ ] SLIs & SLOs defined for top journeys.  
- [ ] Metrics have owners, dashboards, and runbooks.  
- [ ] Labels within budget; exemplars enabled.  
- [ ] Burn-rate alerts wired with multi-window policy.  
- [ ] KPIs have guardrails; rollback rules documented.  
- [ ] Evidence: links to dashboards + sample traces attached to PR.

---

## Queries (snippets)

**Latency p95 (PromQL)**

```promql
histogram_quantile(0.95, sum(rate(request_duration_ms_bucket{route="/checkout"}[5m])) by (le, region))
```

**Error rate by code**

```promql
sum(rate(http_requests_total{status=~"5.."}[5m])) by (code)
  / sum(rate(http_requests_total[5m]))
```

**Core Web Vitals (BigQuery-like pseudocode)**

```sql
SELECT
  APPROX_QUANTILES(lcp_ms, 100)[OFFSET(75)] AS lcp_p75_ms,
  APPROX_QUANTILES(inp_ms, 100)[OFFSET(75)] AS inp_p75_ms,
  APPROX_QUANTILES(cls, 100)[OFFSET(75)] AS cls_p75
FROM web_vitals
WHERE env = 'prod' AND date BETWEEN @start AND @end;
```

**ML drift (PSI pseudocode)**

```sql
SELECT feature, psi(train_hist, prod_hist) AS psi
FROM feature_stats
WHERE window = '24h';
```

---

## Metric definition (YAML template)

```yaml
metric: request_duration_ms
type: histogram
unit: ms
labels: [service, route, method, region, env, version]
buckets_ms: [25, 50, 100, 200, 400, 800, 1600]
owner: "Checkout SRE"
slo:
  objective: "p95 <= 400 ms"
  window: "7d"
alerts:
  - type: burn_rate
    windows: ["5m", "30m"]
    multiple_of_budget: 14
  - type: threshold
    expr: "p99 > 1000"
runbook: "link://runbooks/checkout"
dashboard: "link://dashboards/checkout"
```

---

## SLO spec (YAML template)

```yaml
slo:
  id: checkout_success
  sli:
    numerator: metric: checkout_success_count{region="$region"}
    denominator: metric: checkout_attempt_count{region="$region"}
  objective: ">= 99.5%"
  window: "30d"
  owners: ["Payments", "SRE"]
alerts:
  fast_burn:
    multiple_of_budget: 14
    windows: ["5m", "30m"]
  slow_burn:
    multiple_of_budget: 6
    windows: ["1h", "6h"]
guardrails:
  error_rate_pct: "< 1%"
  latency_p95_ms: "<= +20 vs baseline"
```

---

## Event dictionary (analytics/telemetry entry)

```yaml
event: checkout.completed
required_props:
  - order_id
  - amount_minor
  - currency
  - payment_method
pii_policy: hashes_only
consent_required: true
owner: "Growth"
```

---

## CSV seeds

**Metric registry**

```csv
metric,type,unit,owner,dashboard
request_duration_ms,histogram,ms,API,SRE
http_requests_total,counter,count,API,SRE
web_vitals_lcp_ms,histogram,ms,Web,Web
mobile_cold_start_ms,histogram,ms,Mobile,Mobile
auth_success_rate_pct,gauge,pct,Security,Security
ml_prediction_latency_ms,histogram,ms,ML,ML
```

**SLO register**

```csv
slo_id,description,objective,window,owner
checkout_success,Checkout success rate,>=99.5%,30d,Payments
api_availability,HTTP availability,>=99.9%,30d,Platform
otp_verify_latency_p95,p95 for /otp/verify,<=300ms,7d,Identity
web_lcp_p75,LCP p75 (mobile),<=2500ms,14d,Web
ml_unsafe_rate,Unsafe content rate,==0,7d,ML
```

**Alert rules**

```csv
slo_id,window,multiple_of_budget,severity
api_availability,5m,14,P1
api_availability,1h,6,P2
checkout_success,30m,14,P1
```

**Ownership**

```csv
metric,owner,runbook
request_duration_ms,Platform SRE,link://runbooks/api
web_vitals_lcp_ms,Web Perf,link://runbooks/web
ml_prediction_latency_ms,ML Ops,link://runbooks/ml
```

**KPI guardrails**

```csv
kpi,guardrail,action
conversion_rate_pct,drop > 1% vs baseline,hold
refund_rate_pct,> 0.5%,rollback
ticket_volume_pct,> +10% week over week,investigate
```

---

## Putting it together (example slice)

**Journey**: Checkout (web + API + PSP)  
**SLIs**: success rate, API p95, LCP p75  
**SLOs**: `>= 99.5%`, `p95 <= 400 ms`, `LCP p75 <= 2500 ms`  
**Guardrails**: refund rate `<= 0.5%`, unsafe content `= 0`  
**Diagnostics**: DB/cache latency, queue age  
**Evidence**: dashboard links + top 3 exemplar traces + PSP logs

---

## Links

- Review Gates → `./review-gates.md`  
- Performance → `../60-checklists/performance-review.md`  
- SRE playbook → `../57-cross-discipline-bridges/for-sres.md`  
- Dev observability → `../57-cross-discipline-bridges/for-developers.md`  
- Security/Compliance → `../57-cross-discipline-bridges/for-security-compliance.md`

---

<p align="center">
  <sub>
    Built with <a href="https://treeifyai.com">Treeify</a> — AI test case design copilot.<br>
    <em>Structured, traceable, high-coverage test design — faster.</em>
  </sub>
</p>
