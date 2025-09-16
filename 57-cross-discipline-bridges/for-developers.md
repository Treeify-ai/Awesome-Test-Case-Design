# For Developers — Observability & Log Contracts

> If you didn’t **instrument** it, you didn’t build it.  
> This guide shows how to make features **observable by default** with **structured logs**, **metrics**, and **traces**—so bugs are findable, SLAs are provable, and tests have evidence.

---

## TL;DR (defaults we recommend)

- **Structured logs only** (JSON), never printf/plaintext.  
- Every request/job/event gets a **correlation_id** and (if tracing) **trace_id/span_id**.  
- Use **message IDs** (`MSG.*`) and **error codes** (`ERR.*`) from shared taxonomies.  
- **No PII/secrets** in logs; emit **hashes** or **tokens** only.  
- Emit **RED** (Rate‑Errors‑Duration) metrics for each route; **USE** (Utilization‑Saturation‑Errors) for infra.  
- **OpenTelemetry** everywhere: HTTP/RPC/DB/cache/queue instrumentation + baggage propagation.  
- Control **cardinality**: pre-aggregate high-card values; label budgets per metric.  
- Make **tests assert on signals**: logs, metrics, traces are part of acceptance.

---

## Log contract (shape & rules)

**Shape (JSON)** — minimal, stable, append-only evolution:

```json
{
  "ts": "2025-09-16T12:00:01.234Z",
  "level": "INFO",
  "msgid": "MSG.checkout.placed",
  "message": "Order placed",
  "service": "api",
  "env": "prod",
  "version": "git:abc123",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "correlation_id": "req_7YkP0v",
  "tenant_id": "t_123",
  "user_id_hash": "u_kYc…",
  "route": "POST /checkout",
  "status_code": 200,
  "duration_ms": 312,
  "attempt": 1,
  "error": null,
  "fields": { "order_id": "ord_123", "amount_minor": 1099, "currency": "USD" }
}
```

**Rules**

- **Levels**: `DEBUG` (dev only), `INFO` (state change), `WARN` (retriable), `ERROR` (failed), `FATAL` (crash).  
- **Message IDs** are **stable keys**, not free text.  
- **Error block** (when present):

```json
"error": {
  "code": "ERR.AUTHZ.role.denied",
  "class": "ForbiddenError",
  "message": "role lacks permission",   // short, non-PII
  "stack": "..."                        // only in non-prod or gated
}
```

- **Never** log: secrets, tokens, PAN, email, full phone, addresses, access tokens.  
- Hash or truncate: `user_id_hash`, `email_hash`, `phone_hash_last2`.

**Evolution**: add new fields under `fields` or new **optional** top-level fields; never repurpose names.

---

## Correlation & propagation

- Generate a **correlation_id** at **ingress** (API edge, job enqueue) and set response header `X-Correlation-Id`.  
- Propagate across services (HTTP/gRPC/queues) via headers:  
  - Tracing: `traceparent`/`tracestate` (W3C)  
  - Baggage (key/value context): `baggage`  
  - Fallback correlation: `X-Correlation-Id`

**Example (HTTP)**

```
Incoming:  traceparent, baggage, X-Correlation-Id
Outbound:  forward same; create child spans; copy correlation
```

---

## Metrics contract

Emit **RED** for every user‑visible operation (route, RPC, job):

- `requests_total{route,method,status}` — **counter**  
- `request_duration_ms{route,method}` — **histogram** (p50/p95/p99)  
- `errors_total{route,method,code}` — **counter**

Emit **USE** for resources:

- `cpu_utilization{service,host}` — **gauge**  
- `queue_depth{queue}` — **gauge**  
- `worker_saturation{pool}` — **gauge**

**Budgets** (example): p95 ≤ 400 ms for `POST /otp/verify`, p99 ≤ 800 ms.

**Cardinality guardrails**

- Keep label sets small (`route`, not full URL; `tenant_tier`, not tenant_id).  
- Use **exemplars** (trace_ids) to link histograms to traces.

---

## Tracing (OpenTelemetry)

Instrument:

- **Server**: HTTP/gRPC servers (`server span`)  
- **Client**: HTTP/gRPC clients (`client span`)  
- **DB/Cache**: SQL/NoSQL, Redis, etc.  
- **Queues**: publish/consume; link spans across enqueues → handlers.

**Span attributes (short)**

| Kind | Keys (short) |
|---|---|
| HTTP server | `http.route`, `http.method`, `http.status_code` |
| DB | `db.system`, `db.statement?` (redacted/summary), `db.sql.table` |
| Messaging | `messaging.system`, `messaging.operation`, `messaging.destination` |
| General | `enduser.id_hash`, `tenant.id`, `feature.flag` |

Add **events** on spans for checkpoints (e.g., `3ds.challenge.start`).

---

## Jobs, schedulers, and batch

- **Job IDs** and **attempt** count logged; **deadline** and **backoff** parameters included.  
- Record **control totals** (rows processed, inserted, failed).  
- Emit a final `MSG.job.completed` with outcome **SUCCESS|PARTIAL|FAILED** and counts.

**Batch import** must log:

- File name/hash, schema version  
- Accepted rows, rejected rows (with **message IDs**)  
- Idempotency (`Idempotency-Status: replayed`)

---

## PII redaction & validation

- Central **redaction filters**: tokens, emails, phones, PAN, secrets.  
- Unit tests for regex + **format‑aware** masking (e.g., keep last 2 digits).  
- CI gate: reject PRs adding raw PII to logs.

**Examples**

```json
"fields": { "email_hash": "sha256:…", "phone_hash_last2": "…-**34" }
```

---

## Logging do / don’t

**Do**

- Log **state changes** and **decisions** (why we chose path B).  
- Include **message IDs**, **correlation**, **tenant tier**, **feature flags**.  
- Emit a **single line** JSON per event (no multi-line).

**Don’t**

- Log request/response bodies with PII or secrets.  
- Create **exploding cardinality** (user_id, raw query strings).  
- Overuse `DEBUG` in prod; prefer **trace exemplars** + sampling.

---

## Sampling & cost

- Keep **logs** thin; send **metrics** and **traces** for detail.  
- Dynamic tracing **sampling**: baseline 1–5%, upsample on errors or during incidents.  
- Use **head sampling** for cost control; **tail sampling** for rare/slow traces if supported.

---

## Observability acceptance (tests you can automate)

**Smoke (per PR environment)**

- Hitting `POST /checkout` emits:  
  - 1 `server span` with `http.route=POST /checkout`  
  - Logs with `msgid=MSG.checkout.placed`, `trace_id`, `correlation_id`  
  - Metrics updated: `requests_total`, `request_duration_ms`

**Error path**

- Force `AUTHZ.role.denied` → 403:  
  - Log contains `error.code=ERR.AUTHZ.role.denied` (no PII)  
  - Trace shows **span status = ERROR** with event `authz.denied`  
  - Metrics increment `errors_total{code="AUTHZ.role.denied"}`

**Batch import**

- Upload invalid row → log `MSG.import.row_rejected` with `ERR.VALIDATION.email`  
- Import summary `MSG.job.completed` shows counts match.

---

## Queries (examples)

**Logs (pseudo)**

```sql
from logs
| where service = "api" and msgid = "MSG.checkout.placed"
| summarize p95 = percentile(duration_ms, 95) by route
```

**Metrics (PromQL)**

```promql
histogram_quantile(0.95, sum(rate(request_duration_ms_bucket{route="/checkout"}[5m])) by (le))
```

**Traces**

- Filter: `service="api" http.route="/checkout" duration>500ms has_error=true`  
- Inspect child spans for DB/PSP latency.

---

## Review checklist (quick gate)

- [ ] **JSON logs**: message IDs, correlation/trace IDs, env, version present  
- [ ] **No PII**: redaction filters applied; unit tests for masking  
- [ ] **Error taxonomy** codes emitted on failures; consistent mapping  
- [ ] **Metrics**: RED/USE in place; label cardinality budget respected  
- [ ] **Tracing**: OTel server+client+DB+queue spans; baggage propagated  
- [ ] **Jobs/Batch**: control totals; job outcome message; idempotency noted  
- [ ] **Headers**: `X-Correlation-Id` in responses; `traceparent` propagated  
- [ ] **Dashboards**: p95/p99, error rate, saturation, backlog; alerts wired  
- [ ] **Tests** assert on signals; artifacts linked in PR

---

## CSV seeds

**Message ID registry (snippet)**

```csv
msgid,level,notes
MSG.login.success,INFO,session created
MSG.login.failed,WARN,invalid creds or rate limit
MSG.checkout.placed,INFO,order placed
MSG.job.completed,INFO,outcome + counters
MSG.import.row_rejected,INFO,per-row validation fail
```

**Error taxonomy mapping (snippet)**

```csv
provider_code,err_code,http
ACCESS_DENIED,ERR.AUTHZ.role.denied,403
RATE_LIMIT,ERR.RATE.limit,429
INVALID_EMAIL,ERR.VALIDATION.email,400
TIMEOUT,ERR.DEPENDENCY.timeout,504
```

**Metric label budget**

```csv
metric,max_labels,notes
requests_total,500,"route,method,status only"
request_duration_ms,500,"route,method only"
errors_total,200,"route,code only"
```

**Routes with budgets**

```csv
route,p95_ms,p99_ms
POST /otp/verify,300,800
POST /checkout,400,1000
GET /orders/:id,200,600
```

---

## Templates

**Log contract (YAML)**

```yaml
name: api_log_v1
fields:
  ts: datetime_iso8601_z
  level: [DEBUG, INFO, WARN, ERROR, FATAL]
  msgid: string           # stable message key
  message: string         # short, non-PII
  service: string
  env: string
  version: string         # git sha or build id
  trace_id: string
  span_id: string
  correlation_id: string
  tenant_id: string?
  user_id_hash: string?
  route: string?
  status_code: int?
  duration_ms: int?
  attempt: int?
  error:
    code: string?
    class: string?
    message: string?
  fields: object          # extensible bag
evolution:
  - additive only
  - never repurpose names
```

**Metrics spec (YAML)**

```yaml
metric: request_duration_ms
type: histogram
labels: [route, method]
buckets_ms: [25, 50, 100, 200, 400, 800, 1600]
slo:
  p95_ms: 400
  p99_ms: 800
exemplars: true
```

**Tracing policy (YAML)**

```yaml
sampling:
  head: 0.05
  rules:
    - when: error=true
      sample: 1.0
    - when: latency_ms > 1000
      sample: 1.0
propagation:
  headers: [traceparent, tracestate, baggage, X-Correlation-Id]
instrumentation:
  http_server: true
  http_client: true
  db_clients: [postgres, redis]
  messaging: [sqs, kafka]
```

---

## Links

- Error Taxonomy: `../40-api-and-data-contracts/error-taxonomy.md`  
- Contracts & Schemas: `../40-api-and-data-contracts/contracts-and-schemas.md`  
- Idempotency & Retries: `../40-api-and-data-contracts/idempotency-and-retries.md`  
- Resiliency & Timeouts: `../50-non-functional/resiliency-and-timeouts.md`  
- Security Essentials (no PII in logs): `../50-non-functional/security-essentials.md`  
- Performance p95/p99: `../50-non-functional/performance-p95-p99.md`  
- For PMs (evidence in specs): `./for-pms.md`
