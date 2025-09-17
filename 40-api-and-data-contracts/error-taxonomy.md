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


# Error Taxonomy (Contracts + Message IDs)

> Clear, consistent errors turn bugs into fast fixes and UX into trust.  
> This taxonomy standardizes **error codes**, **HTTP mapping**, **message IDs**, and **oracles** across API, UI, logs, metrics, and events.

---

## Goals

- **Consistency** — same condition ⇒ same **code** and **HTTP** everywhere.
- **Observability** — structured **error_code** for logs/metrics/traces.
- **UX ready** — stable **message IDs** for localization; no free‑text coupling.
- **Actionability** — classify **retryability** and **owner** (caller vs system).

---

## Namespaces (codes)

Use `UPPER.SNAKE` namespaces with dot segments: `FAMILY.subfamily.reason[.detail]`.

| Family        | Meaning (caller action)                             | Example codes                                       |
|---------------|-----------------------------------------------------|-----------------------------------------------------|
| `VALIDATION`  | Bad input; caller must **fix**                      | `VALIDATION.code.length.exceeds`, `VALIDATION.date.range` |
| `AUTH`        | Authentication failed                               | `AUTH.invalid_credentials`, `AUTH.token.expired`    |
| `AUTHZ`       | Permission/tenant scope denied                      | `AUTHZ.role.denied`, `AUTHZ.scope.tenant`           |
| `POLICY`      | Business rule blocks action                         | `POLICY.account.locked`, `POLICY.legal_hold`        |
| `CONFLICT`    | Resource conflict / state precondition fails        | `CONFLICT.code.not_combinable`, `CONFLICT.etag.mismatch` |
| `NOT_FOUND`   | Resource does not exist or is hidden by policy      | `NOT_FOUND.order`                                   |
| `GONE`        | Resource previously existed but is permanently gone | `GONE.order`                                        |
| `RATE_LIMIT`  | Too many requests                                   | `RATE_LIMIT.exceeded`                               |
| `DEPENDENCY`  | Downstream/system dependency issue                   | `DEPENDENCY.timeout`, `DEPENDENCY.unavailable`      |
| `TRANSIENT`   | Temporary internal issue                            | `TRANSIENT.error`                                   |
| `INTERNAL`    | Unhandled server fault                              | `INTERNAL.unexpected`                               |

> Choose **one** family per error. Don’t mix validation + policy in one response.

---

## HTTP mapping (canonical)

| Family       | Typical HTTP | Retry? | Caller Fix? | Notes |
|--------------|--------------|:------:|:-----------:|------|
| VALIDATION   | 400          |   ❌   |     ✅      | Field/location details encouraged |
| AUTH         | 401          |   ❌   |     ✅      | `WWW-Authenticate` hints |
| AUTHZ        | 403 (or 404 for tenant leak safety) | ❌ | ✅ (request rights) | Be consistent per policy |
| POLICY       | 403 / 409    |   ❌   |     ➖      | Explain next step (support/approval) |
| CONFLICT     | 409          |   ❌   |     ✅      | ETag mismatch, idempotency payload mismatch |
| NOT_FOUND    | 404          |   ❌   |     ➖      | May be policy-masked |
| GONE         | 410          |   ❌   |     ➖      | Historical pointer optional |
| RATE_LIMIT   | 429          |   ✅   |     ➖      | Respect `Retry-After` |
| DEPENDENCY   | 502/503/504  |   ✅   |     ➖      | Backoff + jitter |
| TRANSIENT    | 500          |   ✅   |     ➖      | Limited retries |
| INTERNAL     | 500          |   ❌   |     ➖      | Page on-call; redact details |

---

## Standard error shape (API)

Return a **single shape** everywhere.

```json
{
  "error": {
    "code": "VALIDATION.code.length.exceeds",
    "message_id": "error.validation.code.length.exceeds",
    "http": 400,
    "retryable": false,
    "correlation_id": "3f8c…",
    "details": {
      "fields": {
        "code": {
          "reason": "length",
          "max": 16,
          "actual": 17
        }
      }
    },
    "docs": "https://docs.example.com/errors#validation-code-length-exceeds"
  }
}
```

### Field errors
- Use `details.fields.<field>` with `reason`, **constraints** (e.g., `min/max/regex`), and **actual**.
- Return **multiple** field errors for forms:  
  `{ "errors": [ {error}, {error} ] }` **or** `{ "error": { "code": "...", "details": { "fields": { ... }}}}` — pick one and standardize.

### Message IDs
- `message_id` is the **stable key** the UI uses to render localized copy.  
- Backend **may** send a human `message` for logs, but UI should **prefer message_id** + client dictionary.

---

## UX copy & i18n (contract)

**Client dictionaries** (example)

```json
{
  "error.validation.code.length.exceeds": {
    "en-US": "Enter a code of at most {max} characters.",
    "fr-FR": "Saisissez un code de {max} caractères maximum."
  },
  "error.conflict.code.not_combinable": {
    "en-US": "This code can’t be combined with gift cards.",
    "fr-FR": "Ce code ne peut pas être combiné avec des cartes-cadeaux."
  }
}
```

Guidelines:
- Use **placeholders** (`{max}`, `{min}`), not string concatenation.
- Avoid leaking PII or internal IDs in messages.
- Keep **tone** consistent and actionable; provide **next-step** hints where useful.

---

## Logging, metrics, traces (oracles)

- **Logs** (structured): `{error_code, message_id, correlation_id, route, actor_role, tenant_id, http, retryable}`  
- **Metrics**: counters per `error_code` family; `*_rate_limit`, `*_authz_denied`, `idempotency_conflict`.  
- **Traces**: tag spans with `error_code`, `message_id`, **and** inputs like `rule_id`.

Make these **review gates**: every negative path must emit the code and be traceable.

---

## Canonical examples

### A) Validation — Discount code length

**Request**: `POST /discount/verify` with `"A"*17`  
**Response**: 400

```json
{
  "error": {
    "code": "VALIDATION.code.length.exceeds",
    "message_id": "error.validation.code.length.exceeds",
    "http": 400,
    "retryable": false,
    "details": { "fields": { "code": { "max": 16, "actual": 17 } } }
  }
}
```

**UI behavior**: show localized copy near the input; keep value.

---

### B) Conflict — Not combinable with gift card

**Response**: 409

```json
{
  "error": {
    "code": "CONFLICT.code.not_combinable",
    "message_id": "error.conflict.code.not_combinable",
    "http": 409,
    "retryable": false
  }
}
```

**UI**: present **remove gift card** call to action.

---

### C) Idempotency payload mismatch

**Response**: 409

```json
{
  "error": {
    "code": "CONFLICT.idempotency.payload_mismatch",
    "message_id": "error.conflict.idempotency.payload_mismatch",
    "http": 409,
    "retryable": false
  }
}
```

**Caller**: must **generate a new key** with the new payload.

---

### D) Rate limit

**Response**: 429 with `Retry-After: 2`

```json
{
  "error": {
    "code": "RATE_LIMIT.exceeded",
    "message_id": "error.rate_limit.exceeded",
    "http": 429,
    "retryable": true
  }
}
```

Client: back off per `Retry-After` and retry.

---

### E) Dependency timeout (retryable)

**Response**: 504/503

```json
{
  "error": {
    "code": "DEPENDENCY.timeout",
    "message_id": "error.dependency.timeout",
    "http": 504,
    "retryable": true
  }
}
```

---

## Mapping to scenarios & tests

- **MAE Exception** scenarios must cite **message IDs** in their **oracles**.  
- **Boundary** cases assert specific `VALIDATION.*` codes.  
- **Decision tables** should log `rule_id` and return a precise `POLICY.*` or `CONFLICT.*`.

**Gate**: A PR adding a new negative path **must**:  
1) Pick a **code** from this taxonomy (or propose one),  
2) Add **client dictionary** entries,  
3) Add **observability** (log+metric+trace),  
4) Add **tests** that assert the code.

---

## CSV seeds

**Code registry**

```csv
code,http,retryable,owner,notes
VALIDATION.code.length.exceeds,400,false,caller,max=16
VALIDATION.code.charset,400,false,caller,allowed=[A-Z0-9-]
CONFLICT.code.not_combinable,409,false,caller,offer_remove
CONFLICT.idempotency.payload_mismatch,409,false,caller,generate_new_key
AUTH.invalid_credentials,401,false,caller,reauth
AUTHZ.role.denied,403,false,caller,need higher role
AUTHZ.scope.tenant,404,false,caller,leak_safe
RATE_LIMIT.exceeded,429,true,system,respect Retry-After
DEPENDENCY.timeout,504,true,system,backoff+jitter
INTERNAL.unexpected,500,false,system,page_oncall
```

**Form field example**

```csv
field,code,reason,limit
code,VALIDATION.code.length.exceeds,length.max,16
code,VALIDATION.code.charset,charset,[A-Z0-9-]
```

---

## Templates

**Error response**

```json
{
  "error": {
    "code": "<FAMILY.reason.detail>",
    "message_id": "error.<family>.<reason>.<detail>",
    "http": <status>,
    "retryable": <true|false>,
    "correlation_id": "<uuid>",
    "details": { "fields": { "<field>": { "reason": "<...>" } } }
  }
}
```

**Client dictionary entry**

```json
"error.<family>.<reason>.<detail>": {
  "en-US": "…",
  "fr-FR": "…",
  "zh-CN": "…"
}
```

---

## Anti-patterns

- Returning **free‑text** only; no stable `message_id`.  
- Overloading **HTTP 400** for everything negative.  
- Mixing **validation** and **policy** errors in one response.  
- Inconsistent **404 vs 403** for cross-tenant.  
- Leaking **PII** or internal implementation detail in messages.  
- No **retry** hints for 429/5xx.  
- Codes that **change names** between services.

---

## Review checklist (quick gate)

- [ ] Code chosen from taxonomy (or added to registry)  
- [ ] HTTP status correct and consistent with family  
- [ ] `message_id` present; client dictionaries updated  
- [ ] `retryable` flag set correctly; **Retry-After** honored for 429  
- [ ] **Field details** included for validation errors  
- [ ] Logs/metrics/traces include `error_code` & `message_id`  
- [ ] Tests assert **code** (not free text) in oracles  
- [ ] 404/403 policy consistent for tenancy and documented

---

## Links

- Boundary & Equivalence: `../20-techniques/boundary-and-equivalence.md`  
- Decision Tables (rules → codes): `../20-techniques/decision-tables.md`  
- Idempotency & Retries: `./idempotency-and-retries.md`  
- Pagination & Filtering (validation): `./pagination-and-filtering.md`  
- Roles & Permissions: `../30-scenario-patterns/roles-and-permissions.md`

---

<p align="center">
  <sub>
    Built with <a href="https://treeifyai.com">Treeify</a> — AI test case design copilot.<br>
    <em>Structured, traceable, high-coverage test design — faster.</em>
  </sub>
</p>
