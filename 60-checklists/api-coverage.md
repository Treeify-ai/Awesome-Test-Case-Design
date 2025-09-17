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


# API Coverage Checklist

> “Spec is truth; tests are proof.”  
> Use this checklist to ensure every API surface is **contracted, observable, and resilient**—with test cases that prove behavior across happy, alt, and failure paths.

---

## TL;DR

- Cover **HTTP semantics**: methods, status codes, headers, caching, idempotency.  
- Verify **contracts** with schemas, example payloads, and **message IDs** for errors.  
- Exercise **pagination/filtering/sorting**, **rate limits**, **retries/timeouts**, and **webhooks**.  
- Include **security, privacy, and tenancy** gates.  
- Capture **evidence**: logs, traces, metrics, and reproducible `curl`/Postman collections.

---

## Preconditions (before testing starts)

- [ ] **Spec** available (OpenAPI/GraphQL schema/Protobuf) and versioned.  
- [ ] **Environments** and credentials ready; test tenant/users seeded.  
- [ ] **Data contracts** defined for inputs/outputs (types, ranges, required, enums, PII flags).  
- [ ] **Error taxonomy** and **message IDs** mapped.  
- [ ] **Observability**: log/metric/trace fields and correlation IDs defined.

Links:  
- Contracts & Schemas → `../40-api-and-data-contracts/contracts-and-schemas.md`  
- Error Taxonomy → `../40-api-and-data-contracts/error-taxonomy.md`  
- Idempotency & Retries → `../40-api-and-data-contracts/idempotency-and-retries.md`  
- Pagination & Filtering → `../40-api-and-data-contracts/pagination-and-filtering.md`  
- Security Essentials → `../50-non-functional/security-essentials.md`

---

## 1) Per-endpoint coverage

For each route (e.g., `POST /orders`, `GET /orders/:id`, `GET /orders`):

- [ ] **Method semantics** correct (`GET` is safe/idempotent; `POST`/`PATCH` unsafe).  
- [ ] **Status codes**: `2xx` success variants; `4xx` validation/policy; `5xx` dependency/unknown.  
- [ ] **Headers**: `Content-Type`, `Accept`, `ETag`, `If-None-Match`, `Idempotency-Key` (unsafe), `X-Correlation-Id`.  
- [ ] **Request validation**: type, range, enum, required/optional, unknown fields rejected.  
- [ ] **Response shape**: matches schema; fields typed; **nullability** correct; extra fields disallowed.  
- [ ] **Message IDs** included on errors; **error codes** stable.  
- [ ] **Caching**: `Cache-Control`, `ETag`/conditional GET where applicable.  
- [ ] **Latency** within budget (p95/p99); payload sizes within limits.  
- [ ] **Observability**: logs with `msgid`, metrics RED, traces with db/cache spans.

**Template (checklist to paste under each route)**

```
Route: <METHOD PATH>
Scope: <tenants/locales/features>

[ ] HTTP method & status codes
[ ] Request schema validation (incl. unknown fields)
[ ] Response schema validation (nullability, enums)
[ ] Headers (Accept, Content-Type, caching, correlation)
[ ] Authn/Authz (role/tenant matrix)
[ ] Idempotency (unsafe operations)
[ ] Pagination/filtering/sorting (for list)
[ ] Rate limits & 429 handling
[ ] Timeouts/retries/deadlines
[ ] Evidence captured (logs, traces, metrics, curl)
```

---

## 2) Idempotency & concurrency

- [ ] **Idempotency-Key** required for unsafe `POST`/`PATCH`/`DELETE`.  
- [ ] Replays with **same key** return **same response** (`Idempotency-Status: replayed`).  
- [ ] **Duplicate suppression** under concurrent requests (DB constraints/unique keys).  
- [ ] **Conditional requests** for updates: `If-Match` with `ETag` to prevent lost updates.

---

## 3) Pagination, filtering, sorting

- [ ] **Cursor-based** pagination with stable sort + tiebreaker.  
- [ ] `next_cursor`/`prev_cursor` behavior correct; empty list edge cases.  
- [ ] Filters validated (types, ranges, enums); unknown filters rejected.  
- [ ] **Count**/`total` optional but consistent; documented costs.  
- [ ] Sorting whitelisted; invalid sort rejected.

See: `../40-api-and-data-contracts/pagination-and-filtering.md`.

---

## 4) Errors & retries

- [ ] **Taxonomy**: errors carry `code` and **message ID**; HTTP codes aligned.  
- [ ] **Retryable** vs **non-retryable** clearly separated; `Retry-After` on 429/503.  
- [ ] **Deadlines** enforced server-side to avoid long tail.  
- [ ] **Partial failures** return per-item statuses with stable IDs.  
- [ ] **Problem details** style supported (JSON object with `type`, `title`, `detail`, `code`).

**Example error body**

```json
{
  "code": "ERR.VALIDATION.email",
  "message_id": "validation.email.invalid",
  "title": "Invalid email",
  "detail": "Must be a valid address",
  "request_id": "req_7YkP0v"
}
```

---

## 5) Security, privacy, tenancy

- [ ] **Authn** present (OAuth2/JWT/API key); tokens validated (aud/iss/exp/nonce).  
- [ ] **Authz** matrix enforced; **IDOR** protections on path/body/query IDs.  
- [ ] **Tenant isolation** proven; list endpoints scoped correctly.  
- [ ] **Headers**: `CORS` rules minimal; security headers where applicable.  
- [ ] **PII minimization**: payloads exclude secrets; hashing where needed; no PII in logs.  
- [ ] **Rate limits** per IP/user/tenant; safe defaults.

Links: `../57-cross-discipline-bridges/for-security-compliance.md`, `../50-non-functional/privacy-and-compliance.md`.

---

## 6) Webhooks & outbound calls

- [ ] Webhooks signed (HMAC + timestamp), **replay window** enforced.  
- [ ] **Inbox dedupe** table processes at-least-once deliveries; **idempotent** handlers.  
- [ ] **Retry policy** with backoff + jitter; **dead-letter** and **replay** endpoints.  
- [ ] **Ordering** not assumed; handlers reconstruct via `occurred_at` and IDs.  
- [ ] Evidence: delivery logs, signature verify, dedupe hits, replay proof.

See: `../55-domain-playbooks/b2b-integrations.md` and `../40-api-and-data-contracts/idempotency-and-retries.md`.

---

## 7) Long-running jobs

- [ ] **202 Accepted** with **operation id** and `Location` to poll status.  
- [ ] **Status endpoint** exposes `state` (`pending/running/succeeded/failed/canceled`), **progress**, **result/error**.  
- [ ] **Idempotent** resubmits; **cancellation** supported.  
- [ ] Evidence: job logs with control totals; final `MSG.job.completed`.

**Example**

```
POST /imports -> 202 Location: /operations/op_123
GET  /operations/op_123 -> { "state": "running", "progress": 0.42 }
```

---

## 8) Versioning & deprecation

- [ ] Versioned API (`/v1` or media types); **backwards-compatible** changes only in minors.  
- [ ] **Changelog** and deprecation windows communicated; **error on sunset** after date.  
- [ ] Test old and new versions side-by-side for critical routes.

---

## 9) Performance & limits

- [ ] Route **p95/p99** latency within budgets; payload sizes ≤ caps.  
- [ ] **Throughput** under expected RPS; soak test for connection reuse.  
- [ ] **N+1** queries avoided; DB indices present.  
- [ ] **Compression** (gzip/br) on large JSON; **streaming** for massive responses where appropriate.  
- [ ] **Cache** hints honored (ETag/Last-Modified).

Links: `../50-non-functional/performance-p95-p99.md`.

---

## 10) Observability & evidence

- [ ] **JSON logs** with `msgid`, `code`, `correlation_id`/`trace_id`.  
- [ ] **Metrics** (RED): request counts, error rate by code, duration histograms.  
- [ ] **Traces** across client/server/DB/cache; exemplars linked to latency histograms.  
- [ ] **Runbooks** linked for top alerts; dashboards bookmarked.  
- [ ] Evidence attached: `curl`/Postman collection, HAR, logs, traces, metrics screenshots.

Links: `../57-cross-discipline-bridges/for-developers.md`, `../57-cross-discipline-bridges/for-sres.md`.

---

## Per-route test skeletons

**Create (unsafe) — idempotent POST**

```bash
curl -s -X POST https://api.example.com/v1/orders \
 -H "Authorization: Bearer <token>" \
 -H "Idempotency-Key: idem:123" \
 -H "Content-Type: application/json" \
 -d '{"cart_id":"c_123","amount_minor":1099,"currency":"USD"}'
```

**List with cursor pagination**

```bash
curl -s "https://api.example.com/v1/orders?limit=50&cursor=<c>&status=paid&sort=-created_at"
```

**Conditional GET with ETag**

```bash
# First
ETAG=$(curl -sI https://api.example.com/v1/orders/ord_123 | grep ETag | cut -d' ' -f2)
# Second (should be 304)
curl -s -H "If-None-Match: $ETAG" https://api.example.com/v1/orders/ord_123 -o /dev/null -w "%{http_code}\n"
```

**429 handling (Retry-After)**

```bash
curl -i https://api.example.com/v1/orders | grep -i Retry-After
```

---

## Negative & edge tests (don’t skip)

- [ ] Unknown fields; wrong types; overlong strings; invalid enums.  
- [ ] **Boundary** values (0, max, empty arrays).  
- [ ] **Concurrency**: two POSTs with same idempotency key vs different keys.  
- [ ] **Clock skew** for signatures/timestamps.  
- [ ] **Out-of-order** webhook deliveries; duplications; delayed events.  
- [ ] **Network** failures: timeouts, DNS errors, TLS issues.

---

## GraphQL / gRPC notes

**GraphQL**

- [ ] Schema types & non-nullability; input validation; complexity limits.  
- [ ] Pagination via **connections + cursors**.  
- [ ] Partial errors in `errors[]` vs data; avoid leaking internals.  
- [ ] Persisted queries / allowlist.

**gRPC**

- [ ] Proto versioning; **deadline** propagation; **idempotency** via request ids.  
- [ ] Status codes mapped to errors; streaming backpressure.

---

## CSV seeds

**Route inventory**

```csv
method,path,auth,notes
POST,/v1/orders,oauth2,idempotent
GET,/v1/orders/:id,oauth2,etag
GET,/v1/orders,oauth2,cursor pagination, filters
POST,/v1/refunds,oauth2,idempotent
POST,/v1/webhooks/events,hmac,ingest
```

**Error taxonomy mapping (snippet)**

```csv
http,code,message_id
400,ERR.VALIDATION.email,validation.email.invalid
403,ERR.AUTHZ.scope,authz.scope.denied
404,ERR.RESOURCE.not_found,resource.not_found
409,ERR.CONFLICT.idempotency,idempotency.payload_mismatch
429,ERR.RATE.limit,rate.limit
503,ERR.DEPENDENCY.unavailable,dependency.unavailable
```

**Perf budgets**

```csv
route,p95_ms,p99_ms,max_payload_kb
POST /v1/orders,400,1000,32
GET  /v1/orders/:id,200,600,24
GET  /v1/orders,300,800,64
```

**Webhook contract (canonical)**

```csv
field,required,notes
event_id,yes,unique
type,yes,e.g., order.paid
occurred_at,yes,UTC ISO-8601 Z
signature,yes,HMAC header
data.id,yes,resource id
```

---

## Sign-off (per service)

- [ ] Routes covered with MAE + negative tests.  
- [ ] Schemas validated; **unknown fields rejected**.  
- [ ] Idempotency, pagination, rate limits verified.  
- [ ] Security/privacy/tenancy gates pass.  
- [ ] Webhooks/contracts exercised with dedupe/replay.  
- [ ] Perf budgets met; evidence attached.  
- [ ] Dashboards + runbooks linked; alerts green.

---

<p align="center">
  <sub>
    Built with <a href="https://treeifyai.com">Treeify</a> — AI test case design copilot.<br>
    <em>Structured, traceable, high-coverage test design — faster.</em>
  </sub>
</p>
