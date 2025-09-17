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


# Resiliency & Timeouts

> Systems fail in **unexpected** ways: slow downstreams, partial outages, retry storms, partitions.  
> This playbook defines **contracts, patterns, and tests** for resilient behavior—**timeouts, retries, circuit breakers, load shedding, backpressure, and graceful degradation**—so the product bends without breaking.

---

## TL;DR (defaults we recommend)

- Set an **end-to-end deadline** for each user request (e.g., **2s** for interactive APIs); **propagate** it downstream.
- Use **exponential backoff + full jitter** on **retryable** failures only (429/5xx/timeouts).  
- **Per-attempt timeout** at each hop: `attempt_timeout = min(deadline_remaining − safety_margin, hop_max)`.  
- Add **circuit breakers** (fail fast) + **bulkheads** (isolation) for risky dependencies.  
- Implement **load shedding** once the system nears saturation; better **fast 503** than slow 5xx storms.  
- Prefer **idempotent** write paths; require `Idempotency-Key` for unsafe POSTs.  
- Design **graceful degradation** and **fallbacks** (cached values, lighter features) with explicit **oracles**.

---

## 1) Concepts that matter

- **Deadline** vs **timeout**: a **deadline** is the total remaining time for the **whole** operation; a **timeout** is the max wait for **one** hop.  
- **Backpressure**: tell callers you’re saturated; do not buffer unbounded.  
- **Hedging**: send a **duplicate** request after a delay to cut long tails (reads only; beware write semantics).  
- **Partial availability**: show a subset (e.g., cached price) while a dependent feature is degraded.  
- **Fail-fast**: return clear errors **quickly** when a dependency is known unhealthy (open circuit).

---

## 2) Timeout & retry budgets (how to calculate)

Let **B** be the **end-to-end deadline** (e.g., **2s**). With **N** serial downstream calls and **R** max retries total:

```
attempt_timeout = min(deadline_remaining - safety_margin, hop_max)
total_time_with_retries ≤ B
```

**Rule of thumb**
- Interactive endpoint B = **2s** → distribute roughly: app **400–600 ms**, deps combined **≤ 1200 ms**, network+serialize **≤ 200 ms**.  
- Per dependency at p95: **≤ 80–120 ms**.  
- Safety margin **≥ 100 ms** to avoid breaching the client deadline.

**Per-hop example**

| Hop | Max attempts | Per-attempt timeout | Notes |
|---:|---:|---:|---|
| Auth service | 1 | 100 ms | no retry |
| Catalog | 2 | 150 ms | retryable |
| Tax API | 2 | 200 ms | retryable |
| Total budget | — | — | fits under 2s including app compute |

---

## 3) Policies (what to retry & what to fail)

**Retryable**: `429`, `5xx`, network **timeouts/resets**, **dependency timeout** from error taxonomy.  
**Do not retry**: `400`/`401`/`403`/`404`, **validation/policy/authz** errors, `CONFLICT.*` (including idempotency payload mismatch).

**Backoff schedule (example)**

| Attempt | Delay (ms) |
|---:|---:|
| 1 | 0 |
| 2 | 100–200 (full jitter) |
| 3 | 200–400 |
| 4 | 400–800 (cap at 2s) |

Honor **Retry-After** for 429.

---

## 4) Circuit breakers, bulkheads, load shedding

- **Circuit breaker**: tracks error/latency rates per dependency. States: **closed → open → half-open**.  
  - Open on **p95 > threshold** or **error rate > X%** for Y seconds.  
  - Half-open: allow **K probe** requests; close on success.
- **Bulkheads**: isolate resource pools (thread/conn) **per dependency**; one slow service shouldn’t starve others.
- **Load shedding**: enforce **max in-flight** or **queue size**; reject with **503** when above thresholds.

**Oracles**  
- Logs with `breaker_state`, `shed=true`, `inflight`, `queue_depth`.  
- Metrics: `breaker.open.count`, `shed.requests`, dependency p95.

---

## 5) Graceful degradation & fallbacks

Design **explicit** degraded modes:
- **Cache fallback**: serve **stale** value with banner `data_age`.  
- **Feature flag off**: hide non-essential panels; keep core flow.  
- **Approximate computation**: skip expensive step; return estimate with `approximate=true`.  
- **Queue & notify**: accept request, enqueue work, notify when ready (idempotent).

**Contract**: surface a **signal** in response (header/field) so clients can adapt and tests can assert.

---

## 6) Deadlines & propagation (contract)

- Clients send `X-Request-Timeout: <ms>` or `X-Request-Deadline: <RFC3339>`; servers **subtract** compute time and forward the **remaining** (never increase).  
- On **deadline exceeded**, return **504** (or **408** at front door) with error code `DEPENDENCY.timeout` (see taxonomy).  
- Include **`X-Deadline-Remaining`** in responses (optional; great for debugging).

**Pseudo**

```python
deadline = now() + client_budget
while step in plan:
    remaining = deadline - now() - safety_margin
    t = clamp(remaining, 0, hop_max)
    if t <= 0: return 504, error("DEPENDENCY.timeout")
    call(step, timeout=t)
```

---

## 7) Hedging (advanced tail reducer)

- Use **only for idempotent reads**. Start a **hedged** duplicate if the first attempt exceeds the **p95** of historical latency.  
- Cancel the loser; keep one response.  
- Rate-limit hedges to avoid cost explosions.

**Oracles**: metric `hedge.count`, log `hedge=true`, trace `hedge_delay_ms`.

---

## 8) Tests (design & evidence)

### A) Dependency latency injection
- Inject **+200 ms** at p95 for Tax API; assert:
  - Endpoint still meets **p99** budget,
  - **Retries** occur with jitter,
  - **Breaker** opens if sustained,
  - **No retry storm** (bounded attempts per request).

### B) Outage (5xx burst)
- Return 5xx for N seconds; expect:
  - **Open breaker**, fast **503/Fail-fast**, **backoff** in clients,
  - **Partial availability** path or **fallback** kicks in,
  - **Recovery**: half-open probes then close; latencies normalize.

### C) Load shedding
- Overwhelm with RPS 3–5× target; expect:
  - **503** with `Retry-After` rather than long tail 5xx,
  - Queue depth capped, **no OOM** or thread starvation.

### D) Deadline propagation
- Pass a **2s** client deadline; confirm each hop honors **remaining** time; no hop exceeds its `attempt_timeout`.  
- When exceeded, **504** with `message_id` for timeout.

### E) Idempotent retries
- For writes with `Idempotency-Key`, force a timeout on first attempt and success on retry; expect **exactly-once** effect.

**Evidence across all**
- Logs: `{correlation_id, attempt, retry_delay_ms, breaker_state, deadline_remaining}`  
- Metrics: `retry.count`, `timeout.count`, `shed.count`, dependency `p95/p99`.  
- Traces: spans per hop with `timeout_ms`, `attempt`, `breaker_state`.

---

## 9) Runbooks (operational)

- **Golden signals** per dependency: `rps`, `p95/p99`, `error_rate`, `breaker_state`, `shed_rate`.  
- **Dashboards**: request percentiles split by **status class**; retry count; deadline remaining distribution.  
- **Alerts**:  
  - `p99 > SLO × 1.5 for 5m`,  
  - `breaker open > 5% of requests for 10m`,  
  - `shed_rate > 1%`.  
- **Levers**: raise/lower `max_inflight`, breaker thresholds, disable hedging, toggle degradation flags.

---

## 10) Anti-patterns

- **No jitter** in retries → synchronized storms.  
- **Nested retries** at multiple layers → multiplicative explosions.  
- **Unlimited queues** → OOM and death spirals.  
- **Waiting for timeouts** when a breaker could **fail fast**.  
- **Hedging writes** (duplicates) or hedging without budgets.  
- **Ignoring client deadlines**; each hop uses its own fixed timeout.  
- **Swallowing errors** (200 OK with empty body) instead of clear codes.

---

## 11) Review checklist (quick gate)

- [ ] End-to-end **deadline** defined per endpoint; **propagated** downstream  
- [ ] Retry policy: **which codes**, **how many attempts**, **backoff + jitter**, **max backoff**  
- [ ] **Per-hop timeouts** computed from deadline; **safety margin** applied  
- [ ] Circuit breakers configured (thresholds, **half-open** probes); bulkheads per dependency  
- [ ] **Load shedding** enabled with reasonable queue limits  
- [ ] **Graceful degradation** paths documented & testable signals in responses  
- [ ] **Idempotency** required for unsafe POSTs; replay returns **same result**  
- [ ] Fault-injection tests cover latency, 5xx, outage, and recovery  
- [ ] Observability: logs/metrics/traces capture **attempts, breaker state, deadlines**  
- [ ] Runbooks & alerts defined; rollback/kill switches present

---

## 12) CSV seeds

**Dependency budget**

```csv
dep,role,max_attempts,per_attempt_timeout_ms,breaker_threshold_p95_ms,notes
auth,critical,1,100,150,no retry
catalog,important,2,150,250,backoff+jitter
tax,important,2,200,300,hedge_reads=true
search,optional,1,150,250,graceful_degrade=cache
```

**Fault plan**

```csv
case,dep,inject,window_s,expected
LAT-200,tax,+200ms@p95,600,"retry<=1, breaker maybe open"
OUTAGE-5xx,catalog,5xx,120,"open breaker, fast 503, recovery to closed"
OVERLOAD,*,rps x4,180,"shed 503<=2%, no OOM"
DEADLINE,*,deadline=2s,60,"hop timeouts<=remaining, 504 on exceed"
```

**Runbook thresholds**

```csv
metric,threshold,window,action
http.p99_ms,>800,5m,open incident; flip feature flag
breaker.open_share,>0.05,10m,investigate dependency
shed.rate,>0.02,5m,scale out; lower max_inflight
retry.count_per_req,>2,5m,inspect timeouts; tune budgets
```

---

## 13) Templates

**Resiliency plan header**

```
# Resiliency Plan — <Feature/Endpoint>
Owner: <team>  Date: <YYYY-MM-DD>

End-to-end deadline: <ms>
Retry policy: <codes> attempts=<n> backoff=<strategy>
Breakers: <deps + thresholds>
Bulkheads: <pool sizes>
Load shedding: <limits>
Degradation: <fallbacks + signals>
Observability: <logs/metrics/traces>
Fault injection: <latency/5xx/outage/overload>
Acceptance gates: <criteria>
```

**Code stub (client)**

```python
def call_with_deadline(url, deadline_ms, max_attempts=3):
    deadline = now_ms() + deadline_ms
    for attempt in range(1, max_attempts+1):
        remaining = deadline - now_ms() - 100  # safety
        if remaining <= 0:
            raise Timeout("deadline exceeded")
        t = min(remaining, 500)  # hop cap
        try:
            return http_get(url, timeout=t)
        except Retryable as e:
            if attempt == max_attempts: raise
            sleep(jitter_backoff(attempt, base=0.1, cap=2.0))
```

---

## Links

- Error taxonomy (retryable vs terminal): `../40-api-and-data-contracts/error-taxonomy.md`  
- Idempotency & Retries (contracts): `../40-api-and-data-contracts/idempotency-and-retries.md`  
- Performance p95/p99: `./performance-p95-p99.md`  
- Cross-feature interactions & degradation: `../30-scenario-patterns/cross-feature-interactions.md`  
- Observability hooks (evidence capture): `../80-tools-and-integrations/observability-hooks.md`

---

<p align="center">
  <sub>
    Built with <a href="https://treeifyai.com">Treeify</a> — AI test case design copilot.<br>
    <em>Structured, traceable, high-coverage test design — faster.</em>
  </sub>
</p>
