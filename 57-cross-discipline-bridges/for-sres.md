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


# For SREs — Failure Modes, SLOs, Synthetic Checks

> Reliability is a feature.  
> This guide helps SREs design **defensible SLOs**, enumerate **failure modes**, wire **synthetic checks**, and run **incidents** with evidence and fast recovery.

---

## TL;DR (defaults we recommend)

- Anchor on **user-journey SLOs** (checkout, login, search), backed by **component SLIs**.  
- Adopt **brownout first**: degrade non-essential features before failing.  
- Timeouts **<** retries **<** client deadlines, with backoff + jitter and **circuit breakers**.  
- Keep **error budgets** visible; throttle launches when budget is **burning too fast**.  
- **Synthetic journeys** in each region + **deep checks** for critical dependencies.  
- Drill **regional failover**, **data restore** (RTO/RPO), and **runbook** exercises quarterly.  
- Capture **artifacts**: dashboards, logs, traces, packet captures when relevant.

---

## SLI/SLO patterns

**User-journey SLIs (preferred)**

- `checkout_success_rate = successful_checkouts / checkout_attempts`  
- `otp_verify_latency_p95_ms` at **/otp/verify**  
- `message_delivery_rate = delivered / accepted (rolling 1h)`  
- `search_time_to_first_result_p95_ms`

**Component SLIs (supporting)**

- HTTP `availability = 1 - 5xx_rate`  
- Queue `age_p95`  
- DB `error_rate`, `latency_p95`  
- Cache `hit_ratio`

**SLOs (examples)**

- Checkout success **≥ 99.5%** (monthly).  
- OTP verify p95 **≤ 300 ms**; p99 **≤ 800 ms** (weekly).  
- API availability **≥ 99.9%** (monthly).

**Error budget policy**

- Burn rate alerts at **14×** (fast) and **6×** (medium) over 1h/6h windows.  
- Freeze risky launches when weekly budget **< 25%** remaining.

---

## Failure modes (catalog)

| Mode | Detection | Mitigation |
|---|---|---|
| Dependency **brownout** (slow 5xx/429) | latency p95↑, queue age↑ | breaker open, fallback cache, brownout UI |
| **Regional outage** | synthetic fail in region | failover DNS/traffic, read-only mode |
| **Thundering herd** | spikes on retries | jitter, max attempts, token bucket |
| **Hot shard** | p99 DB table/index | rebalance, shard split, cache |
| **Noisy neighbor** | CPU steal, IO wait | quotas, limiters, isolate pool |
| **Memory leak** | restart count, RSS↑ | restart window, heap dump analysis |
| **Clock skew** | auth/signature fails | time sync alerts, NTP fallback |
| **Config blast radius** | error surge post change | staged rollout, canary, fast revert |
| **Webhook storm/replay** | dup event ids | inbox dedupe, rate limits |
| **DNS/PKI failure** | TLS errors, NXDOMAIN | stale DNS cache, secondary CA |

*Keep tables short; details live in runbooks.*

---

## Resiliency design (practical defaults)

- **Backoff + jitter** for 429/5xx/timeouts; cap total **deadline** per call.  
- **Circuit breakers** on high failure rate or long latency; half-open probes.  
- **Bulkheads**: separate pools for critical vs background traffic.  
- **Load shedding**: return `503` for non-critical when saturated; protect core routes.  
- **Brownout** toggles: disable avatars, recommendations, heavy queries under stress.  
- **Idempotency** on unsafe POSTs to avoid double effects on retries.

See: `../50-non-functional/resiliency-and-timeouts.md`, `../40-api-and-data-contracts/idempotency-and-retries.md`.

---

## Synthetic monitoring

**Journeys** (per region):

- Home → Login OTP → Search → Add to cart → Checkout (stub payment)  
- Health endpoints are **not** enough—exercise real dependencies (DB, cache, queue, PSP sandbox).

**Deep checks**:

- **DB**: `SELECT 1` + lightweight app query.  
- **Queue**: publish → consume loopback.  
- **Webhooks**: signed callback to a **canary endpoint**.  
- **DNS/TLS**: expiry, chain, OCSP stapling.

**Design rules**

- Use **tenant/test accounts** with **idempotent** operations.  
- Tag metrics with `synthetic=true`, `region`, `journey`.  
- Store **evidence**: rendered screenshots, HARs, response bodies (redacted).

---

## Capacity & scaling

- **Headroom**: plan for **2×** normal daily peak; **4×** burst for flash sales.  
- Autoscaling on **concurrency** or **queue depth**, not CPU alone.  
- Control **cold starts**; keep warm pools for latency-sensitive paths.  
- Do **load tests** with production-like data, then set SLO budgets accordingly.

---

## Disaster recovery (RTO/RPO)

- **RTO** (restore time) and **RPO** (data loss window) per system.  
- Test **point-in-time restore** for DB; verify **binlog** replay window.  
- Cross-region **replication** lag monitored; failover drills with **read-only** and **promotion** runbooks.  
- Backups **encrypted**, **tested**, **labeled** with checksums.

---

## Incident response (lite playbook)

- **Declare** early; page **roles**: Incident Commander, Ops, Comms, Scribe.  
- **Stabilize** with brownouts, breakers, and rate limits.  
- **Diagnose** using **dashboards + logs + traces**; capture **timeline**.  
- **Mitigate** (rollback, failover, config fix).  
- **Communicate** status every 15–30 min to stakeholders.  
- **Close**: record **impact**, **MTTR**, **root cause(s)**, **learn** (avoid blame).  
- **Follow-up**: action items with owners and dates; link PRs and dashboards.

---

## Observability you must have

- **RED** metrics per route; **USE** metrics per resource.  
- **Traces** with **exemplars** tied to latency histograms.  
- **Structured logs** with `correlation_id`, `msgid`, `err.code`.  
- Black-box **synthetics** + white-box **component** probes.  
- **Budget** dashboards: burn rates, availability, latency p95/p99, backlog sizes.  
- **Alert routing**: severity, ownership, runbook links; paging rules per tier.

See: `./for-developers.md` for log/metrics/trace contracts.

---

## Guardrails & change management

- **Feature flags** with kill switches and **safe defaults**.  
- **Progressive delivery**: canary %, region-by-region, automatic rollback on guardrail breach.  
- **Config schema** with validation; **review + diff** for changes.  
- **Secret rotation**: planned jobs with alarms on failure.

---

## MAE Scenarios (copy/adapt)

### SRE-001 **Main** — Checkout synthetic passes
- **Steps**: run hourly journey in all regions  
- **Expected**: p95 ≤ 2s; success ≥ 99.5%  
- **Oracles**: synthetic dashboard; HAR + screenshots stored

### SRE-002 **Alt** — Dependency brownout (payments)
- **Steps**: PSP sandbox returns 5xx + slow  
- **Expected**: breaker opens; brownout to “pay later”; error rate < budget  
- **Oracles**: breaker metrics; route-level error rate; user-journey SLI still ok

### SRE-101 **Exception** — Regional failure drill
- **Steps**: block egress in region `ap-sg`  
- **Expected**: failover to `ap-jp` in ≤ 5 min; RPO ≤ 60s  
- **Oracles**: traffic split, replication lag, error budget burn

### SRE-102 **Exception** — Config misdeploy
- **Steps**: push bad DB pool size  
- **Expected**: canary detects; auto-rollback; alert with runbook link  
- **Oracles**: deploy logs; canary diff; rollback event

### SRE-201 **Cross-feature** — Quiet hours vs paging
- **Expected**: customer-facing alerts respect quiet hours; **on-call** ignores quiet hours  
- **Oracles**: alert routes; escalation policy test

---

## Review checklist (quick gate)

- [ ] User-journey **SLIs/SLOs** defined; component SLIs mapped  
- [ ] **Error budget** policy with multi-window burn-rate alerts  
- [ ] **Timeouts/retries/breakers** consistent across services  
- [ ] **Brownout** controls and **load shedding** implemented  
- [ ] **Synthetic checks** for journeys and deep dependencies  
- [ ] **Capacity** model + load-test results; autoscaling rules  
- [ ] **Backups** verified; **RTO/RPO** documented and tested  
- [ ] **Failover** drill records; runbooks current and linked  
- [ ] **Observability** dashboards + paging rules + runbooks wired

---

## CSV seeds

**SLO registry**

```csv
slo_id,description,objective,window,owner
checkout_success,Checkout success rate,>=99.5%,30d,Payments SRE
api_availability,HTTP 5xx inverted,>=99.9%,30d,Platform SRE
otp_verify_latency_p95,p95 ms for /otp/verify,<=300ms,7d,Identity SRE
```

**Burn-rate alerts (multi-window)**

```csv
slo_id,window,threshold_x
checkout_success,5m,14
checkout_success,1h,6
api_availability,30m,14
api_availability,6h,6
```

**Synthetic registry**

```csv
journey,region,frequency,deadline_ms
checkout_full,ap-sg,5m,3000
login_otp,us-east,5m,1500
search_basic,eu-west,5m,2000
```

**Failover drill log**

```csv
date,region,duration_min,success,notes
2025-09-10,ap-sg,18,true,"rds promotion 3m; cache warm 5m"
```

---

## Queries & rules (snippets)

**PromQL — burn rate**

```promql
# Fast burn (5m) for availability SLO
(1 - sum(rate(http_requests_total{status=~"5.."}[5m])) 
    / sum(rate(http_requests_total[5m]))) / (1 - 0.999)
```

**PromQL — latency p95**

```promql
histogram_quantile(0.95, sum(rate(request_duration_ms_bucket{route="/otp/verify"}[5m])) by (le, region))
```

**SLO spec (YAML)**

```yaml
slo:
  id: checkout_success
  objective: ">=99.5%"
  window: 30d
sli:
  numerator: metric: checkout_success_count{region="$region"}
  denominator: metric: checkout_attempt_count{region="$region"}
alerts:
  - burn_rate: 14x
    windows: [5m, 30m]
  - burn_rate: 6x
    windows: [1h, 6h]
runbook: "link://runbooks/checkout"
```

---

## Templates

**Runbook skeleton**

```
Service: <name>
Owner: <team>  Pager: <rotation>
Dashboards: <links>
SLOs: <list>
Dependencies: <list + health links>
Guardrails: <flags, breakers, rate limits>
Symptoms: <common alerts + meanings>
Checks: <commands/queries to run>
Mitigations: <brownouts, rollbacks, failovers>
Comms: <channels + cadence + templates>
Post-incident: <data to collect + forms>
```

**Chaos experiment**

```
Name: <dep> brownout
Hypothesis: breaker opens within 60s and error budget burn < 1%
Method: inject latency + 5xx for 5m
Metrics: route error rate, p95, queue age
Abort: p95 > target by 2× for 2m
```

---

## Links

- Resiliency & Timeouts: `../50-non-functional/resiliency-and-timeouts.md`  
- Performance p95/p99: `../50-non-functional/performance-p95-p99.md`  
- Compatibility Matrix (networks/locales): `../50-non-functional/compatibility-matrix.md`  
- Observability & Logs (devs): `./for-developers.md`  
- PM Acceptance (evidence and SLIs): `./for-pms.md`  
- Messaging playbook (quiet hours & policies): `../55-domain-playbooks/messaging-and-notifications.md`

---

<p align="center">
  <sub>
    Built with <a href="https://treeifyai.com">Treeify</a> — AI test case design copilot.<br>
    <em>Structured, traceable, high-coverage test design — faster.</em>
  </sub>
</p>
