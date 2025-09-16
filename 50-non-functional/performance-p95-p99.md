# Performance (p95/p99)

> Users feel the **tail**, not the average.  
> This playbook shows how to **design, run, and judge** performance work using **p95/p99 latency**, throughput, and error budgets—without boiling the ocean.

---

## TL;DR (defaults we recommend)

- Define **SLIs** first (what you measure), then **SLOs** (targets).  
- Use **open‑loop** load (constant arrival rate) for tail latency; **closed‑loop** for capacity checks.  
- Track **p50/p95/p99**, **error rate**, and **throughput** per endpoint/feature.  
- Publish **budgets**: *p95 ≤ 300 ms, p99 ≤ 800 ms* for interactive APIs; adjust for your domain.  
- Always log **correlation_id**, **attempt**, **error_code**, and **retry** data; separate 2xx and non‑2xx percentiles.  
- Prove fixes with **A/B before/after** runs and **time‑series percentiles** (not just single aggregates).

---

## 1) Concepts (SLI/SLO/SLA & percentiles)

- **SLI** (Service Level Indicator): a measurement, e.g., *“Checkout API end‑to‑end latency.”*  
- **SLO** (Objective): a target over a window, e.g., *“p95 ≤ 300 ms, p99 ≤ 800 ms over 28 days.”*  
- **SLA** (Agreement): business contract; usually a subset of SLOs with penalties—**don’t** test SLAs directly.

**Percentiles**
- **p95**: 95% of requests are **at or below** this latency. **p99** exposes **long tails** (locks, GC, cold caches).  
- Prefer **HDR histograms** / base‑2 buckets; avoid averaging percentiles (math doesn’t support it).  
- Report percentiles per **status class** (2xx vs 4xx/5xx) and per **attempt** (first vs retries).

---

## 2) What to measure (SLIs)

| Layer | SLI | Why it matters |
|---|---|---|
| **User** | End‑to‑end interaction time | Actual experience; budgets per page/workflow |
| **API** | Request latency (TTFB & total) | Contracts for services, retriable vs terminal |
| **Job** | Time‑to‑completion, queue wait | Backlogs, SLOs for async work |
| **Infra** | CPU, memory, GC, disk IOPS, network | Root‑cause tails; saturation |
| **Dependency** | Downstream call latency/error | Hot spots; budget splits |

Augment with **Rate‑Errors‑Duration (RED)** and **Utilization‑Saturation‑Errors (USE)**.

---

## 3) Budgets (set them explicitly)

**Interactive APIs (CRUD/eligibility/verify)**  
- p50 ≤ 100 ms, **p95 ≤ 300 ms**, **p99 ≤ 800 ms**, error rate ≤ 0.1% (2xx success), over 7–28 days.

**Background jobs (per item)**  
- p95 ≤ 2 s, p99 ≤ 5 s; queue wait p95 ≤ 500 ms.

**Web UI** (ballpark; adapt to product)  
- Action → paint **≤ 200 ms** feels instant; **≤ 1 s** keeps flow; beyond that, show progress.

**Budget splits** (example, server‑side 300 ms p95)
- Auth + routing: 20 ms  
- **Business compute**: 120 ms  
- Dependencies (sum): 120 ms (no single dep > 80 ms)  
- Serialization + network: 40 ms

> Keep **p99 ≤ 2.5 × p95** as a healthy rule; bigger gaps signal tail issues.

---

## 4) Workload modeling (make it realistic)

- **Traffic mix**: % per endpoint/use case; include **hot paths**.  
- **Data mix**: payload sizes (p10/p50/p90), item counts, **i18n** strings, real cardinalities.  
- **Think‑time** (UI), **diurnal** cycles, spikes (deploys, campaigns).  
- **Cache** behavior: warm/cold ratios; TTL expirations; skew (hot keys).  
- **Read/write** mix and transaction sizes.  
- **Security**: authn/authz enabled; TLS on; real token scopes.

---

## 5) Test types

- **Baseline smoke** — verify environment; steady 1–5 RPS.  
- **Ramp** — step up to target RPS; observe p95/p99 slope.  
- **Steady‑state** — 15–60 min at target; judge SLOs.  
- **Spike** — 2–5× burst for 1–5 min; ensure graceful shed.  
- **Soak** — 2–24 h to surface leaks, GC stalls, clock drift.  
- **Stress / Breakpoint** — push until p95 or error rate violates SLO; record **knee**.  
- **Faulted** — inject latency/500s from dependencies; verify **retries/backoff** and **timeouts**.

---

## 6) Open‑loop vs Closed‑loop load

- **Open‑loop** (constant arrival rate): sends requests on schedule **regardless of response** → better at revealing **tail** and queueing.  
- **Closed‑loop** (concurrency controlled): each client waits for response → can mask tails due to backpressure.  
- Use **open‑loop** to validate **p95/p99**, then **closed‑loop** to find **max sustainable RPS**.

**Little’s Law**: `L = λ × W`  
- Given arrival rate `λ` and average latency `W`, expect concurrency `L`. Use it to right‑size workers/pools.

---

## 7) Oracles & evidence

- **Latency histograms** (p50/p95/p99) **per endpoint** and **per status**.  
- **Time‑series** percentiles (1–5 min windows) to see regressions.  
- **Trace spans** with `correlation_id`, `attempt`, `idempotency_key`, downstream timings.  
- **Metrics**: CPU, GC time, heap, thread pool queue length, connection pool saturation, queue depth.  
- **Logs**: structured `route`, `params_hash`, `error_code`, `retry_count`, `timeout=true/false`.

---

## 8) Selecting test cases (examples)

**Eligibility API (`POST /discount/verify`)**
1. Typical payload, cache **warm** → ensure **p95 ≤ 300 ms**.  
2. Cold cache (first hit after TTL) → **p99 ≤ 800 ms**; only first request misses.  
3. Dependency adds 200 ms at p95 → endpoint stays within budget via fallback or tightened compute.  
4. Retry budget: **≤ 1** retry at most; total still under p99.  
5. Error rate under load ≤ 0.1%; no surge of **429/5xx**.

**Pagination under writes** (see contracts doc)
- p95 list latency ≤ 200 ms at 200 RPS while concurrent inserts occur; **no dupes/skips**.

**Refunds**
- With provider timeouts, client retries (same key). End‑to‑end p99 ≤ 2 s; **exactly once** ledger entry.

---

## 9) Fix patterns (what usually works)

- **N+1** → add joins/batch endpoints; use **request coalescing**.  
- **Hot lock / contention** → shard, reduce critical section, use **copy‑on‑write** snapshots.  
- **GC stalls** → tune heap, reduce allocations, pool objects/buffers.  
- **Cold start** → warm pools, pre‑JIT, lazy‑load only on background.  
- **Chatty deps** → widen payload (fewer round trips), cache results, add **circuit breakers**.  
- **Large payloads** → gzip/br, paginate, project only needed fields.  
- **Indexes** → verify query plans; add covering indexes; avoid wildcard sorts without tiebreakers.  
- **Thread/conn pools** → size using **Little’s Law**; add backpressure and timeouts.  
- **Retries** → exponential backoff + jitter; honor **Retry‑After**.

---

## 10) Acceptance gates

A run **passes** if:

- For each **SLO’d endpoint**:
  - p50/p95/p99 within budgets over the steady‑state window.  
  - Error rate ≤ budget; no burst of 5xx.  
  - Retries within allowed limits; **no retry storms**.  
- System metrics show **no saturation** (CPU < 75% p95, queue depth bounded).  
- Logs/traces show **no long‑tail outliers** tied to repeats (locks, GC, cold I/O).  
- Under **fault**, graceful degradation keeps user‑visible p95 within “brownout” budget and **functional correctness** holds.

---

## 11) Anti‑patterns

- Reporting a single run’s **average** latency only.  
- Mixing **2xx + 4xx/5xx** into percentiles.  
- Ignoring **warmup** (JIT, caches) and **autoscaling** ramp.  
- Using **closed‑loop only** → hides tails.  
- Running on tiny datasets that fit entirely in memory when prod does not.  
- No **time‑series** (p95 spikes hidden by long windows).  
- Comparing runs with **different traffic mixes** or payload sizes.  
- Treating **client‑side timeouts** as success (requests cancelled in client aren’t counted).

---

## 12) Review checklist (quick gate)

- [ ] SLIs defined; SLOs with **p50/p95/p99** and error budgets set per endpoint/job  
- [ ] Workload model covers **mix, data sizes, cache states, diurnal/spikes**  
- [ ] Test plan includes **baseline, ramp, steady, spike, soak, stress, faulted**  
- [ ] Open‑loop for tails; closed‑loop for capacity; **Little’s Law** applied  
- [ ] Evidence: **histograms + time‑series** percentiles, traces, infra metrics  
- [ ] Fault injection covers **dependency latency/5xx**; retries/backoff honored  
- [ ] Acceptance gates agreed and automated in CI/CD  
- [ ] Before/after comparisons for regressions with the **same** mix & settings

---

## 13) CSV seeds

**Load plan**

```csv
phase,duration_s,arrival_rps,concurrency,notes
baseline,60,5,10,env check
ramp,300,"25->200",auto,step 25 rps every 60s
steady,1800,200,auto,measure p95/p99
spike,120,500,auto,burst traffic
soak,14400,100,auto,leak/GC check
faulted,600,200,auto,tax api +200ms p95 + 1% 5xx
```

**Budgets per endpoint**

```csv
endpoint,p50_ms,p95_ms,p99_ms,error_budget
POST /discount/verify,100,300,800,0.1%
GET /orders,80,200,500,0.1%
POST /refunds,150,400,2000,0.05%
```

**Results sample**

```csv
endpoint,window_start,p50_ms,p95_ms,p99_ms,rps,error_rate
POST /discount/verify,2025-09-16T12:00:00Z,95,270,620,210,0.03%
POST /discount/verify,2025-09-16T12:05:00Z,110,340,920,220,0.07%
```

---

## 14) Templates

**Test plan header**

```
# Perf Plan — <Feature/Endpoint>
Owner: <team>  Date: <YYYY-MM-DD>

SLIs: <list>
SLOs: p50 <= _, p95 <= _, p99 <= _, error rate <= _

Workload:
- Traffic mix: <…>
- Data mix: <sizes, distributions>
- Cache states: warm/cold %, TTLs

Phases: baseline → ramp → steady → spike → soak → stress → faulted

Acceptance gates:
- <bullets>

Evidence:
- Histograms (p50/p95/p99)
- Time-series charts
- Traces (top outliers)
- Infra metrics (CPU/GC/IO)
```

**Run record**

```csv
timestamp,commit,env,seed,notes
2025-09-16,abc123,staging-3,42,"post index fix"
```

---

## Links

- Idempotency & Retries: `../40-api-and-data-contracts/idempotency-and-retries.md`  
- Pagination & Filtering (list invariants): `../40-api-and-data-contracts/pagination-and-filtering.md`  
- Error Taxonomy (separate 2xx and 5xx): `../40-api-and-data-contracts/error-taxonomy.md`  
- State Models (timeouts/retries): `../20-techniques/state-models.md`  
- Cross-feature Interactions (promo × tax × gift card): `../30-scenario-patterns/cross-feature-interactions.md`  
- Observability hooks (evidence capture): `../80-tools-and-integrations/observability-hooks.md`
