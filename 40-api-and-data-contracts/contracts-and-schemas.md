# Contracts & Schemas (Design + Test Playbook)

> If **behavior** is the movie, **contracts** are the script.  
> Clear, versioned contracts (APIs and events) make systems testable, evolvable, and debuggable.

This page distills **practical, testable guidelines** for REST/HTTP, GraphQL, gRPC, and event payloads (JSON/Avro/Proto), with **checklists, examples, and review gates**.

---

## 1) Principles

- **Stable by default** — minimize breaking changes; prefer additive evolution.  
- **Explicit** — types, formats, units, ranges, nullability, defaults.  
- **Testable** — schemas available; examples executable; **oracles** defined (headers, ids, message IDs).  
- **Observable** — correlation IDs, rule IDs, version tags in logs/traces.  
- **Versioned** — per resource/route/event with a documented deprecation path.

---

## 2) Resource modeling (REST/HTTP)

- **Nouns for resources**; verbs in actions only when needed:  
  - CRUD: `/orders`, `/orders/{id}`  
  - Actions: `/orders/{id}:cancel`, `/refunds` (reverse op)
- **IDs**: use **ULID/UUID**; avoid sequential ids for public APIs.  
- **Timestamps**: **UTC ISO-8601** with `Z` (e.g., `2025-09-16T13:45:00Z`).  
- **Money**: `amount` in **minor units** (cents); include `currency` (ISO-4217).  
- **Booleans**: avoid tri-state unless required; otherwise model with enums.  
- **Null vs Absent**: `null` means **explicitly empty**; **absent** means **not provided** (no change on PATCH). Document both.

**Envelope (recommended)**

```json
{
  "id": "ord_123",
  "object": "order",
  "version": 1,
  "created_at": "2025-09-16T12:00:00Z",
  "updated_at": "2025-09-16T12:00:05Z",
  "data": { /* resource-specific fields */ },
  "meta": { "tenant_id": "t_abc" }
}
```

---

## 3) Standard headers & metadata

- **Idempotency**: `Idempotency-Key` for unsafe POST/DELETE (see `./idempotency-and-retries.md`).  
- **Caching**: `ETag` for GET; use `If-None-Match` / `If-Match`.  
- **Paging**: `next_cursor`/`prev_cursor` (see `./pagination-and-filtering.md`).  
- **Errors**: `error_code` + `message_id` in body (see `./error-taxonomy.md`).  
- **Correlation**: `X-Correlation-Id` echoed in responses & logs.  
- **Versioning**:  
  - URI: `/v1/orders` (coarse-grained), or  
  - **Media type**: `Accept: application/vnd.example.orders+json;version=1` (fine-grained).

---

## 4) Versioning policy

- **Additive** changes only in minor revisions: add fields (nullable/optional), new enum members (document default), new endpoints.  
- **Breaking** changes only in **major** versions with migration notes.  
- **Deprecation**: mark fields/endpoints with `Deprecation` response header and docs; offer overlap window (e.g., **90–180 days**).  
- **Compatibility matrix**

| Change type                                      | Backward safe? | Forward safe? | Notes |
|--------------------------------------------------|:--------------:|:-------------:|------|
| Add optional field                               | ✅ | ✅ | Provide default/nullable |
| Add required field to **requests**               | ❌ | ✅ | Breaks old clients |
| Add required field to **responses**              | ✅ | ❌ | Breaks strict consumers |
| Rename/remove field                               | ❌ | ❌ | Major version only |
| Add enum value                                   | ✅* | ❌ | *If clients ignore unknowns |
| Tighten validation (narrow range/regex)          | ❌ | ❌ | Usually breaking |
| Widen validation (expand range/regex)            | ✅ | ✅ | Safe |
| Change numeric type (int→decimal)                | ❌ | ❌ | Major only |
| Change semantics (currency unit, time zone)      | ❌ | ❌ | Major only |

---

## 5) JSON Schema & OpenAPI (REST)

**JSON Schema snippet (request)**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.example.com/orders/create.v1.json",
  "type": "object",
  "required": ["amount", "currency"],
  "properties": {
    "amount": { "type": "integer", "minimum": 0, "maximum": 1000000 },
    "currency": { "type": "string", "pattern": "^[A-Z]{3}$" },
    "note": { "type": "string", "maxLength": 256 }
  },
  "additionalProperties": false
}
```

**OpenAPI (YAML, response)**

```yaml
components:
  schemas:
    Order:
      type: object
      required: [id, amount, currency, created_at]
      properties:
        id: { type: string, example: "ord_123" }
        amount: { type: integer, example: 1099, description: "Minor units" }
        currency: { type: string, example: "USD" }
        created_at: { type: string, format: date-time }
        status: { type: string, enum: [created, paid, cancelled] }
```

**Contract tests (REST)**
- Validate **request/response** against schema.  
- Assert **headers**: `ETag`, `Idempotency-Status`, `X-Correlation-Id`.  
- Negative tests assert **error_code/message_id** (taxonomy).

---

## 6) GraphQL specifics

- **Non-null** (`!`) means **always present**; changing to nullable is breaking for strict clients.  
- Adding a **field** is generally **non-breaking**.  
- Avoid breaking renames; **deprecate** with `@deprecated(reason: "use ...")` and keep for the overlap window.  
- Version via **field-level deprecation** and/or **namespacing types** (no global `/v2`).  
- **Pagination**: prefer **Relay-style connections** with cursors and **stable sort**.  
- **Contract tests**: capture **persisted queries**; validate responses against the **GraphQL schema** and assert deprecations don’t regress.

---

## 7) gRPC / Protobuf specifics

- Prefer **proto3**; fields are optional by default (presence available in recent versions).  
- **Never** reuse or re-order field numbers; mark removed fields as **reserved**.  
- Additive: adding new fields is safe; unknowns are ignored.  
- Changing scalar types or field numbers is **breaking**.  
- Use **oneof** for union types; keep a **default** branch server-side.  
- Contract tests: exercise **back/forward** compatibility by mixing client/server versions in CI (matrix).

**Proto example**

```proto
syntax = "proto3";
package orders.v1;

message Order {
  string id = 1;
  int64 amount = 2;      // minor units
  string currency = 3;   // ISO-4217
  string created_at = 4; // ISO-8601 Z
  enum Status { CREATED = 0; PAID = 1; CANCELLED = 2; }
  Status status = 5;
  // reserved 6; // do not reuse
}
```

---

## 8) Event contracts (JSON/Avro/Proto)

- **Envelope**

```json
{
  "event": "order.created",
  "version": 1,
  "id": "evt_abc",
  "occurred_at": "2025-09-16T12:00:00Z",
  "correlation_id": "3f8c…",
  "producer": "orders",
  "data": { "id": "ord_123", "amount": 1099, "currency": "USD" },
  "schema_id": "schema://orders/order.created/v1"
}
```

- **Idempotency**: consumer dedupes by `event_id`/`schema_id` (see inbox in `idempotency-and-retries.md`).  
- **Schema registry**: each event **topic + name** maps to a **schema id** (Avro/JSON Schema/Proto).  
- **Evolution**: prefer **additive** fields; support **default values** for new fields in consumers.  
- **Ordering**: do **not** assume; model **state** and idempotent handling.

**Avro tip**: add fields with **default** values; keep field order but do not rely on it.

---

## 9) Consumer-Driven Contracts (CDC)

- Producers publish schemas with **examples**; consumers write CDC tests that **pin** what they rely on (Pact/Optic/etc.).  
- Run CDC in CI for **every PR** that touches an API/event.  
- Ensure **backward** compatibility by replaying consumer expectations against producer changes.

**CDC test oracles**
- Specific fields present with types and example values  
- Error envelopes contain `error_code/message_id`  
- Pagination fields (`next_cursor/page_size`) present where applicable

---

## 10) Examples & oracles (end-to-end)

### Refund creation (REST)

- **Request schema** verified (amount integer, currency string).  
- **Headers** asserted: `Idempotency-Key` required; on replay `Idempotency-Status: replayed`.  
- **Response schema** verified; `status=created|succeeded` enum enforced.  
- **Events**: `refund.succeeded` follows with schema `v1`; consumer **inbox** dedupes.

### Pagination list

- Schema for list response includes `data[]`, `next_cursor`, `page_size`.  
- Sort keys (`created_at`, `id`) documented and **monotonic** across pages (see pagination doc).  
- Filters type-checked (dates in UTC; enums validated).

---

## 11) Anti-patterns

- **Free-form JSON** (“object” with `additionalProperties: true`) without documentation.  
- **Implicit semantics** (unknown timezones; mixed units; stringified numbers).  
- **Enums as strings without registry** (client guesses meaning).  
- **No correlation id** → untraceable failures.  
- **Silent breaking changes** (remove/rename fields without version bump).  
- Relying on **offset pagination** during writes (see pagination).

---

## 12) Review checklist (quick gate)

- [ ] Resource routes and action endpoints clear and consistent  
- [ ] IDs, timestamps, money units, **null vs absent** semantics specified  
- [ ] Headers/contracts: **Idempotency, ETag, Correlation** included where relevant  
- [ ] **Versioning policy** documented; deprecation path defined  
- [ ] **Schemas** published (JSON Schema/OpenAPI/Proto/Avro) with examples  
- [ ] Pagination/filtering contracts align with dedicated doc  
- [ ] Error envelope uses **taxonomy + message IDs**  
- [ ] Event envelopes have **schema_id**; consumers dedupe (inbox)  
- [ ] CDC present in CI; back/forward compatibility tests in matrix  
- [ ] Observability: logs/metrics/traces include **version, correlation, rule_id**

---

## 13) CSV seeds

**Schema registry**

```csv
kind,name,version,id,location,notes
rest,orders.create,1,schema://orders/create.v1,https://schemas.example.com/orders/create.v1.json,json-schema
rest,refunds.create,1,schema://refunds/create.v1,https://schemas.example.com/refunds/create.v1.json,json-schema
event,order.created,1,schema://orders/order.created.v1,https://schemas.example.com/orders/order.created.v1.json,envelope+json
event,refund.succeeded,1,schema://refunds/refund.succeeded.v1,https://schemas.example.com/refunds/refund.succeeded.v1.avsc,avro
grpc,orders.v1.Order,1,schema://orders.v1/Order.proto,https://schemas.example.com/orders/v1/Order.proto,proto3
```

**Breaking-change tracker**

```csv
change_id,area,change,impact,mitigation,owner,status
CH-001,orders,v1->v2: rename status->state,breaking,"dual-write + compat layer",api-team,planned
CH-002,refunds,add field reason (optional),safe,,payments,shipped
```

---

## 14) Templates

**Schema doc header**

```
# <Resource/Event> — v<major.minor>

**Owner**: <team>  
**Status**: active | deprecated (EOL: <date>)  
**Location**: <schema URL>  
**Changelog**: <links>  
**Examples**: <link to fixtures>
```

**Contract test outline**

```
Given: <preconditions>
When: <call/event>
Then: <assert schema + headers + error envelope + invariants>
Evidence: <logs/metrics/traces with correlation_id + version>
```

---

## Links

- Idempotency & Retries: `./idempotency-and-retries.md`  
- Pagination & Filtering: `./pagination-and-filtering.md`  
- Error Taxonomy: `./error-taxonomy.md`  
- State Models: `../20-techniques/state-models.md`  
- Cross-feature Interactions: `../30-scenario-patterns/cross-feature-interactions.md`
