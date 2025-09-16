# Observability Hooks — logs/metrics for assertions

# Observability Hooks — logs/metrics for assertions

> Make quality **observable**.  
> This page standardizes **what to emit** (logs/metrics/traces/events), **where to emit**, and **how tests assert** on those signals so every case can attach durable **evidence**.

---

## Goals

- Stable, **parseable** signals → easy assertions in CI.  
- **MsgIDs** + **err.codes** tie to taxonomy.  
- **Correlation IDs** link client ↔ API ↔ services ↔ jobs.  
- **Exemplars** connect p95/p99 metrics to **trace IDs**.  
- Privacy-first: **no PII** in logs; IDs hashed; retention sane.

---

## What to emit (minimum contract)

**Structured logs** (JSON)
- `timestamp`, `level`, `msgid`, `err.code?`, `service`, `route`, `correlation_id`, `user_hash?`, `tenant_id?`, `attrs{}`

**Metrics**
- **Counters**: events/errors (`*_total`)  
- **Histograms**: latencies (`*_latency_ms`)  
- **Gauges**: queue depth, cache ratio  
- **Ratios**: computed in dashboards, not emitted

**Traces**
- Spans for critical steps; include attributes that match log fields (`msgid`, `route`, `err.code`).  
- Attach **exemplars** from histograms to trace ids.

**Events**
- Domain events (`discount.applied`, `refund.completed`) with minimal payload + IDs.

---

## Hooks by layer

- **Client/UI**
  - Page/view load timings, API call timing, **a11y** error events.  
  - Include `correlation_id` header on outbound calls.  
- **API Gateway**
  - Request/response logs with `msgid`, `route`, `status`, latency.  
  - Rate-limit outcomes (`429`) with reason code.  
- **Service**
  - Business msgids (apply/approve/etc).  
  - Error taxonomy codes on failures.  
  - Dependency timings (PSP, promo, tax) as spans + metrics.  
- **Data/Jobs**
  - Reconcile outcomes, DLQ depth, retry counts.  
- **Infra**
  - Circuit breaker state, cache hit ratio, pool saturation.

---

## Naming & conventions

**MsgIDs** (`MSG.*`) and **Err codes** (`ERR.*`) reuse: `../40-api-and-data-contracts/error-taxonomy.md`.  
**Metric names**: `noun_action_metric_unit` + labels.

Examples:
- `discount_apply_latency_ms{route="/v1/checkout/…",region="sg"}`  
- `discount_attempts_total{result="applied|invalid|expired|ineligible|rate_limited"}`  
- `refund_outcome_total{state="completed|failed|provider_pending"}`  
- `refund_create_latency_ms{provider="sim",region="sg"}`  
- `webhook_replay_dedup_total{provider="psp"}`  
- `cache_hit_ratio{cache="promo_meta"}`

**Low-cardinality labels only** (env, region, route, result code). Avoid user ids.

---

## Log contract (JSON examples)

**Success (discount applied)**

```json
{
  "timestamp":"2025-09-16T02:15:33.101Z",
  "level":"INFO",
  "msgid":"MSG.discount.apply.succeeded",
  "service":"checkout",
  "route":"/v1/checkout/{cart_id}/discounts/apply",
  "correlation_id":"c-7b2d",
  "user_hash":"u:7f2e",
  "attrs":{"code":"SAVE15","currency":"USD","discount_minor":1185}
}
```

**Failure (refund exceeds)**

```json
{
  "timestamp":"2025-09-16T03:01:22.005Z",
  "level":"WARN",
  "msgid":"MSG.refund.requested",
  "err":{"code":"ERR.BUSINESS.refund.exceeds_remaining"},
  "service":"payments",
  "route":"/v1/orders/{order_id}/refunds",
  "correlation_id":"c-91aa",
  "attrs":{"order_id":"O100","amount_minor":600,"remaining_minor":500}
}
```

---

## Trace attributes (OpenTelemetry)

- `service.name`, `http.route`, `http.status_code`, `enduser.id_hash`, `tenant.id`.  
- Business: `msgid`, `err.code`, `code`, `order_id`, `redemption_id`.  
- For dependency spans: `peer.service` (promo, tax, PSP), `db.system`, `net.peer.name`.  
- **Links**: attach `correlation_id` as span attribute; include in logs.

---

## Histograms & exemplars

Emit histograms for **endpoints** and **critical RPCs**.  
Attach exemplar with `trace_id` for p95/p99 samples.

**Prometheus (OpenMetrics) sketch**

```
# HELP discount_apply_latency_ms Apply endpoint latency in ms
# TYPE discount_apply_latency_ms histogram
discount_apply_latency_ms_bucket{le="50",route="/v1/..."} 1200
...
discount_apply_latency_ms_sum{route="/v1/..."} 345000
discount_apply_latency_ms_count{route="/v1/..."} 1500
# EXEMPLAR trace_id="abcd1234" 245
```

---

## Assertions in tests (how to use hooks)

**API test pseudocode**

```python
res = post("/v1/checkout/c_A/discounts/apply", json={"code":"SAVE15"}, headers=hdrs)
assert res.status_code == 200
logs = search_logs(correlation_id=hdrs["X-Correlation-Id"], msgid="MSG.discount.apply.succeeded")
assert logs and logs[0]["attrs"]["code"] == "SAVE15"
lat = read_metric("discount_apply_latency_ms", labels={"route":"/v1/checkout/..."}).p95
assert lat <= 250  # ms budget
trace = get_exemplar_trace("discount_apply_latency_ms", sample="p99")
attach(trace.link)
```

**Refund hard-failure**

```python
res = submit_refund(order="O100", amount=100)
assert last_metric("refund_outcome_total", {"state":"failed"}).delta == 1
log = last_log(msgid="MSG.refund.failed", corr=cid)
assert log["err"]["code"] == "ERR.DEPENDENCY.timeout"
```

> Evidence bundle: response JSON, log snippet, metric snapshot, **trace link**.

---

## Dashboards (minimum viable)

- **API latencies**: p50/p95/p99 by route; error rate by taxonomy.  
- **Attempts breakdown**: `discount_attempts_total` by `result`.  
- **Refund funnel**: requested → approved → provider_pending → completed/failed.  
- **Breaker & cache**: breaker open ratio; cache hit ratio.  
- **SLOs**: burn rates for apply p95 and refund completion T95.

Include **drill-through** to traces via exemplars.

---

## Privacy & safety

- No PII in logs; hash durable IDs (`user_hash`).  
- Redact tokens (`Authorization`, `Set-Cookie`) and card data.  
- Keep **message text** out of logs; use `msgid` + attributes.  
- Retention: short for DEBUG; longer for AUDIT streams.  
- Access: restrict **raw logs**; dashboards safe for wider audience.

---

## Sampling

- **Always-on** for counters and audits.  
- **Adaptive** trace sampling: baseline 1–5%, but **keep** error traces and slow spans.  
- **Tail-based** keep rules: `err.code`, `latency>p95`, `route in {/v1/checkout/...,/v1/refunds}`.

---

## Hook checklists

**Emitters (dev)**  
- [ ] Structured JSON logs with `msgid`, `err.code`, `correlation_id`.  
- [ ] Histograms for critical endpoints; counters for outcomes.  
- [ ] Traces for dependencies; attributes aligned with logs.  
- [ ] Exemplars wired for latency histograms.  
- [ ] PII redaction tested.

**Test writers (QA)**  
- [ ] Each **CASE** asserts on at least one **metric** and one **log**.  
- [ ] Attach **trace links** for slowest sample.  
- [ ] Use **message IDs** as oracles instead of free text.  
- [ ] Evidence bundle attached to PR.

**SRE/Observability**  
- [ ] Dashboards built; SLOs defined with budgets.  
- [ ] Alerts based on **rates** and **burn**, not single spikes.  
- [ ] Run synthetic journeys for MAIN/EXC scenarios.

---

## Synthetic journeys (continuous)

Create synthetic checks that execute **MAIN** and one **EXC** path per feature.

- Discount: valid apply; invalid/expired apply → expect `400` + `MSG.discount.apply.failed`.  
- Refund: full refund happy path; provider timeout path.

Emit `synthetic_journey_outcome_total{journey="discount_main"}`.

---

## CSV seeds

**Signal registry**

```csv
id,type,name,labels,owner,notes
SIG-DISC-APPLY-LAT,histogram,discount_apply_latency_ms,route|region,Checkout,p95<=250ms
SIG-DISC-ATTEMPTS,counter,discount_attempts_total,result|region,Checkout,by result
SIG-REF-OUTCOME,counter,refund_outcome_total,state|provider,Payments,completed/failed/pending
SIG-REF-CREATE-LAT,histogram,refund_create_latency_ms,provider|region,Payments,p95<=250ms
SIG-BREAKER,gauge,breaker_open_ratio,service|peer,Platform,<5%
SIG-CACHE-RATIO,gauge,cache_hit_ratio,cache,Platform,>=0.9
```

**MsgID registry**

```csv
msgid,meaning
MSG.discount.apply.requested,Apply attempt
MSG.discount.apply.succeeded,Apply success
MSG.discount.apply.failed,Apply failed
MSG.discount.removed,Code removed
MSG.refund.requested,Refund requested
MSG.refund.approved,Refund approved
MSG.refund.submitted,Refund submitted
MSG.refund.completed,Refund completed
MSG.refund.failed,Refund failed
```

---

## Wiring examples

**Node (pino + otel)**

```js
logger.info({ msgid: "MSG.discount.apply.succeeded", correlation_id, attrs: { code, currency, discount_minor }});
histogram.observe(latencyMs, { route, region });
span.setAttributes({ "msgid": "MSG.discount.apply.succeeded", "http.route": route });
```

**Python (structlog + prometheus_client + otel)**

```python
log = logger.bind(msgid="MSG.refund.completed", correlation_id=cid, attrs={"refund_id": rid, "amount_minor": amt})
log.info("refund")
refund_outcome_total.labels(state="completed", provider=prov).inc()
refund_create_latency_ms.labels(provider=prov, region="sg").observe(lat_ms)
span.set_attribute("msgid", "MSG.refund.completed")
```

---

## Integrations with this repo

- Traceability map: `../65-review-gates-metrics-traceability/traceability.md`  
- Metrics & Quality Signals: `../65-review-gates-metrics-traceability/metrics.md`  
- Performance budgets: `../50-non-functional/performance-p95-p99.md`  
- For Developers (log contracts): `../57-cross-discipline-bridges/for-developers.md`  
- For SREs (SLOs & synthetics): `../57-cross-discipline-bridges/for-sres.md`

---
