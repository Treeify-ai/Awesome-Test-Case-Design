# Performance Review Checklist

> p95 feels like the product. p99 hurts your users.  
> Use this checklist to set **budgets**, run **repeatable tests**, and collect **evidence** that performance won’t regress after launch.

---

## TL;DR

- Define **budgets** per critical route: p95/p99 latency, throughput, payload caps, memory/CPU ceilings.  
- Test **MAE** for perf: Main (steady-state), Alt (spike/burst/cold), Exception (brownouts, timeouts, retries).  
- Protect against **coordinated omission**; measure at the **client edge**; warm-up, then steady window.  
- Track **Core Web Vitals** on web; **cold/warm start** and **TTI** on mobile.  
- Capture evidence (dashboards, traces, waterfalls, CSVs) and link it in the PR.

Links:  
- Perf budgets & theory → `../50-non-functional/performance-p95-p99.md`  
- Resiliency/timeouts → `../50-non-functional/resiliency-and-timeouts.md`  
- Mobile-first → `../55-domain-playbooks/mobile-first-flows.md`  
- Observability (signals) → `../57-cross-discipline-bridges/for-developers.md`, `../57-cross-discipline-bridges/for-sres.md`

---

## Preconditions (before testing)

- [ ] **Routes & screens** prioritized (top journeys + SLIs).  
- [ ] **Budgets** agreed (p95/p99, RPS, payload caps).  
- [ ] **Env parity**: production-like instance sizes, TLS, caches, WAF/CDN.  
- [ ] **Data realism**: prod-shaped datasets, indexes, skew, large users/tenants.  
- [ ] **Observability**: RED/USE metrics, tracing, logs with correlation IDs.  
- [ ] **Load rig** ready: tool chosen (k6/Locust/JMeter/Gatling), time sync OK.

---

## Metrics & definitions

- **Latency**: end-to-end at client edge; also record server time.  
- **Throughput**: requests/sec (RPS) or tasks/sec (TPS).  
- **Error rate**: 5xx + policy 4xx (if applicable).  
- **Resource**: CPU %, memory RSS, GC pauses, IO wait, queue depth.  
- **Front-end**: LCP, INP, CLS; TTFB; bundle size; # requests.  
- **Mobile**: cold start p95, warm start p95, TTI p95.

Budgets live next to routes/screens (see CSV seeds).

---

## Test types (cover at least these)

1. **Smoke** — tiny load, correctness + signals wired.  
2. **Baseline** — steady-state at expected RPS.  
3. **Spike/Burst** — 0 → 3× RPS in seconds; hold; recover.  
4. **Stress/Break** — ramp until SLO breach; find knee.  
5. **Soak/Endurance** — hours at baseline; look for leaks/rot.  
6. **Cold-path** — caches cold, empty DB cache, cold function start.  
7. **N+1 guard** — list/detail with growing dataset.  
8. **Concurrency** — same record contested updates (ETag/If-Match).  
9. **e2e Journey** — home → login → search → add → checkout (synthetic).

---

## Method (repeatable)

- **Warm-up**: 2–5 min to stabilize JIT/caches.  
- **Stable window**: ≥ 10 min for percentile stats.  
- **Time boxes**: keep tests short, frequent, and automated in CI.  
- **Coordinated omission**: use load tools that correct for it; avoid “max one in-flight”.  
- **Client location**: run from expected regions; include CDN/TLS.  
- **Sampling**: collect traces for slowest 1% (p99 exemplars).  
- **Artifacts**: export CSVs and screenshots; store with run id.

---

## Backend routes (per-route checklist)

```
Route: <METHOD PATH>
Budget: p95 ≤ <ms>, p99 ≤ <ms>, RPS ≥ <n>, payload ≤ <kB>

[ ] Cold vs warm latency recorded
[ ] 2xx/4xx/5xx split acceptable
[ ] Retries/backoff won’t exceed client deadlines
[ ] DB: no N+1; right indexes; cache hit ratio ≥ target
[ ] Downstream deps within budget (DB/cache/PSP/queue)
[ ] Response size within cap; compression on
[ ] Traces: slow children identified; top offenders listed
[ ] Evidence: metrics dashboard + top 5 slow traces + raw CSV
```

---

## Web Front-end (quick pass)

- [ ] **LCP** p75 ≤ 2.5s (mobile), ≤ 1.8s (desktop).  
- [ ] **INP** p75 ≤ 200ms; **CLS** p75 ≤ 0.1.  
- [ ] Bundle size within cap; code-split; defer non-critical.  
- [ ] Images responsive; modern formats; lazy-load below fold.  
- [ ] Fonts subsetted; `font-display: swap`.  
- [ ] Render-blocking minimized; preconnect/preload used sparingly.  
- [ ] Third-parties budgeted; async; self-hosted where possible.  
- [ ] Evidence: Lighthouse/CrUX/trace screenshots + network waterfall.

---

## Mobile apps (quick pass)

- [ ] **Cold start p95 ≤ 1s**, **TTI p95 ≤ 2s** on mid-tier.  
- [ ] Network budget: < 100KB before first interaction (as feasible).  
- [ ] **Dynamic Type** ×1.3 doesn’t reflow into jank.  
- [ ] **Offline** screen fast; queued writes replay on reconnect.  
- [ ] Evidence: startup traces, screen transition timings, payload sizes.

---

## Database & storage

- [ ] Hot queries bounded; use **covered indexes**; proper tiebreaker sort.  
- [ ] Heavy writes batched; **idempotent** upserts; contention measured.  
- [ ] Large scans paginated; `LIMIT` + cursor; no random `OFFSET` for big sets.  
- [ ] Cache layer hit ratio target; invalidation tested.  
- [ ] Evidence: `EXPLAIN` plans, slow logs, index usage.

---

## Caching & CDN

- [ ] Static assets **immutable** with long `max-age`; hashed filenames.  
- [ ] API **ETag**/`If-None-Match` for read-heavy routes.  
- [ ] Edge caching rules documented; purge tested.  
- [ ] Evidence: cache hit charts; 304 ratios.

---

## Resiliency interplay (perf under failure)

- [ ] Breakers open under brownouts; latency tail capped.  
- [ ] Retries with jitter don’t amplify load; deadlines protect servers.  
- [ ] Load shedding on non-critical routes when saturated.  
- [ ] Evidence: chaos test results; burn-rate charts stable.

---

## MAE Scenarios (performance)

### PERF-001 **Main** — Steady-state at target RPS
- **Expected**: p95/p99 within budget; error rate < threshold.  
- **Oracles**: RED metrics; top traces; CSV export.

### PERF-002 **Alt** — Spike to 3× RPS
- **Expected**: p95 may rise ≤ 1.5×; no error flood; recovers in ≤ 2 min.  
- **Oracles**: latency graph; queue depth; breaker closed after spike.

### PERF-101 **Exception** — Dependency brownout
- **Expected**: breaker opens; fallbacks; user-journey SLO honored.  
- **Oracles**: breaker metrics; latency tail clipped; synthetic journey green.

---

## Review checklist (quick gate)

- [ ] Budgets defined per route/screen; visible in docs  
- [ ] Tests cover spike/stress/soak/cold paths  
- [ ] Front-end CWV and mobile startup budgets met  
- [ ] DB/index/caching issues identified & fixed  
- [ ] Resiliency policies verified under load  
- [ ] Evidence attached (dashboards, traces, waterfalls, CSVs)  
- [ ] Regression guard in CI (perf thresholds) enabled

---

## CSV seeds

**Route budgets**

```csv
route,p95_ms,p99_ms,max_payload_kb,target_rps
POST /otp/verify,300,800,16,200
POST /checkout,400,1000,32,120
GET /items,200,600,64,500
```

**Front-end budgets**

```csv
surface,metric,target
web,LCP_p75_ms,2500
web,INP_p75_ms,200
web,CLS_p75,0.1
mobile,cold_start_p95_ms,1000
mobile,tti_p95_ms,2000
```

**Load stages (k6 example)**

```csv
stage,duration,rps
warmup,2m,50
baseline,10m,200
spike,2m,600
recovery,5m,200
```

**DB watchlist**

```csv
query,owner,goal
List orders (by tenant),Orders,<=50ms p95
Search items (prefix),Catalog,<=80ms p95
Top sellers (agg),Analytics,<=120ms p95
```

---

## Snippets & templates

**k6 skeleton**

```js
import http from 'k6/http';
import { check, sleep } from 'k6';
export let options = {
  thresholds: { http_req_duration: ['p(95)<400', 'p(99)<1000'] },
  stages: [
    { duration: '2m', target: 200 },
    { duration: '10m', target: 200 },
    { duration: '2m', target: 600 },
    { duration: '5m', target: 200 },
  ],
};
export default function () {
  const res = http.get(__ENV.BASE_URL + '/health');
  check(res, { 'status is 200': (r) => r.status === 200 });
  sleep(1);
}
```

**Perf review doc (per feature)**

```
Feature: <name>
Routes/Screens: <list>
Budgets: <p95/p99/RPS/payload>
Method: <tools, stages, regions>
Evidence: <dashboards, traces, waterfalls, csvs>
Risks: <known hot spots + mitigations>
Decision: <ship|hold>  Owners: <names>
```

---

## Common pitfalls

- Testing only “happy fast” while **cold-start** and **first-byte** are slow.  
- Ignoring **client-side** latency (TLS, DNS, CDN) and measuring server-only.  
- No correction for **coordinated omission**.  
- Using **tiny datasets** that hide N+1 and bad query plans.  
- Running from a single region; missing real-world latencies.  
- Skipping **evidence** or not storing raw CSV/trace links.

---

## Sign-off

- [ ] All target journeys meet budgets (p95/p99, RPS).  
- [ ] Front-end CWV and mobile startup within targets.  
- [ ] Resiliency verified under stress.  
- [ ] Evidence attached and archived with run id.  
- [ ] Regression gates added to CI and dashboards bookmarked.
