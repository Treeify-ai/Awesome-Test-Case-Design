# Pagination & Filtering (Contract + Playbook)

> Listings seem simple—until you page while data is changing.  
> This playbook defines **stable, testable contracts** for pagination and filtering so you avoid **duplicates, skips, leaks, and surprises**.

---

## TL;DR (defaults we recommend)

- Prefer **cursor (keyset) pagination** with a **stable sort** and **tiebreaker** (`(created_at DESC, id DESC)`).
- Return **opaque cursors** (`next_cursor`, `prev_cursor`), not numeric offsets.
- Enforce **max page size** (e.g., ≤ 100); validate and clamp `page_size`.
- Lists should **exclude soft-deleted** items by default; enable explicit `include_deleted=true` when needed.
- Filters are **validated**; unknown filters or invalid values return **400** with **message IDs** (error taxonomy).
- Document **invariants** and provide testable evidence (ids, sort keys, cursors).

---

## API Shapes

### Cursor (recommended)

```
GET /orders?sort=created_at.desc,id.desc&page_size=50&cursor=<opaque>
200 OK
{
  "data": [ ... items ... ],
  "page_size": 50,
  "next_cursor": "eyJjcmVhdGVkX2F0IjoiMjAyNS0wOS0xNVQxMjozNTozMFoiLCJpZCI6Ijk5OSJ9",
  "prev_cursor": "…",
  "count": null,            // optional & expensive, see below
  "links": { "next": "...", "prev": "..." } // (RFC5988-style optional)
}
```

**Cursor content** should encode the **last sort keys** (e.g., `created_at`, `id`) and the direction. Keep it **opaque** to callers.

### Offset (legacy/support)

```
GET /orders?sort=created_at.desc,id.desc&limit=50&offset=100
```

Allow only when write rates are **low** and you can accept occasional **drift**. Clearly document caveats. Prefer **snapshot reads** if DB supports it.

---

## Sorting Contract

- Define a **primary** key (e.g., `created_at DESC`) and a **tiebreaker** (e.g., `id DESC`) to make ordering **total** and **stable**.
- Expose allowed fields and directions: `sort=field1.[asc|desc],field2.[asc|desc]`.
- Reject unknown fields or illegal combos with **400** (`VALIDATION.sort.field`).

**Invariants to test**
1. **Monotonic cursors**: following `next` always moves forward; `prev` moves back to the same set.
2. **No duplicates across pages**.
3. **No skips** when moving sequentially without intervening writes (or within a snapshot window).
4. **Stable under writes** (cursor): New inserts **do not** cause reordering of items already paged.

---

## Filtering Contract

### Filter syntax

- Simple equality: `status=active`  
- Range: `created_at.gte=2025-09-01T00:00:00Z`, `created_at.lt=2025-09-16T00:00:00Z`  
- Collections: `status.in=active,cancelled`  
- Text: `q=shipping+label` (document behavior: **full-text vs prefix**)

**Rules**
- Filters combine with **AND** by default; provide an explicit `.or=` if needed (or a simple `logic=or` for limited fields).
- **Timezone**: interpret timestamps as **UTC**; suffix `Z`. If client sends naive dates, **reject** with `VALIDATION.datetime.timezone_required`.
- **Inclusivity**: `_gte` and `_lte` are inclusive; `_gt` and `_lt` exclusive. State this clearly.

**Validation**
- Unknown filter keys → `400 VALIDATION.filter.unknown_key`.  
- Wrong type/value → `400 VALIDATION.filter.value_invalid`.  
- Disallowed combinations (e.g., `deleted=true` without role) → `403 AUTHZ.filter.denied`.

---

## Soft Delete & Visibility

- Default: `soft_deleted=false` (hidden).  
- Allow `include_deleted=true` for admin/reporting.  
- For **hard deleted**, prefer `404` on read and ensure list filters **cannot** retrieve them.

---

## Total Count semantics

- `count` may be **null** or omitted for performance. If provided, define if it is:
  - **Exact** snapshot (transactional or materialized view), or
  - **Approximate** (e.g., track in a counter table / search engine).
- Expose `count_strategy: exact|approximate` if you return it.

---

## HTTP & Caching

- Support **ETag** on list responses (hash of ids+params) for **client caching** and **conditional GET**.
- Use **Cache-Control** with short TTLs for hot lists if safe.
- Consider **`Prefer: snapshot=true`** to pin a snapshot (DB-specific).

---

## Examples (SQL-ish)

### Cursor query (DESC)

```sql
SELECT * FROM orders
WHERE (created_at, id) <= (:cursor_created_at, :cursor_id) -- for DESC
AND   status IN (:status_in)
ORDER BY created_at DESC, id DESC
LIMIT :page_size + 1; -- fetch one extra to decide has_more
```

### Offset query (DESC, caveat)

```sql
SELECT * FROM orders
WHERE status IN (:status_in)
ORDER BY created_at DESC, id DESC
OFFSET :offset LIMIT :limit;
-- Under concurrent inserts, items can shift between pages → duplicates/skips
```

---

## Test Design (what to assert)

### A) Functional Paging Invariants

- **No duplicates** across pages.
- **No skips** when paged sequentially without writes.
- **Monotonicity**: sort keys never increase (for DESC) as you advance pages.
- **Prev/Next correctness**: round-trip `page1 → next → prev` returns identical items.

### B) Concurrency under writes

Simulate **inserts/updates/deletes** while paging:

- For **cursor**:
  - Inserts **older** than the last item on the page → **do not** affect already seen items.
  - Inserts **newer** than the first page appear on **earlier pages** you haven’t yet fetched; acceptable.
- For **offset**:
  - Expect **duplicates/skips** under heavy writes; your gate should **fail** and recommend cursor.

### C) Filters & Boundaries

- Date boundaries (`gte/lt`) include/exclude correctly at the **exact timestamp**.
- Combining filters narrows results (AND semantics).
- Invalid filters return **400** with **message IDs**.
- `include_deleted` hides or shows as specified.

### D) Page Size Constraints

- Clamp `page_size` to max; rejecting absurd values (e.g., >1000) with `400 VALIDATION.page_size.max`.
- `page_size=0` → `400 VALIDATION.page_size.min` (or treat as 1).

---

## Worked Example — Orders API

**Routes**

```
GET /orders?status.in=active,cancelled&created_at.gte=2025-09-01T00:00:00Z&sort=created_at.desc,id.desc&page_size=25
GET /orders?cursor=<opaque>&page_size=25
```

**Responses (extract)**

```json
{
  "data": [
    {"id":"1010","created_at":"2025-09-15T12:34:30Z","status":"active"},
    {"id":"1009","created_at":"2025-09-15T12:33:59Z","status":"cancelled"}
  ],
  "page_size": 25,
  "next_cursor": "eyJjcmVhdGVkX2F0IjoiMjAyNS0wOS0xNVQxMjozMzo1OVoiLCJpZCI6IjEwMDgifQ=="
}
```

**Invariants**

- For any contiguous page sequence P1..Pn, `set(IDs(P1..Pn))` has **no duplicates**.
- For **cursor**, if you **insert** rows with `created_at` **after** P1 fetched, they **don’t** appear in P2..Pn of that run.

---

## Observability (evidence)

- **Response**: include `sort`, `page_size`, and `cursor_meta` (decoded server-side for debugging if needed).  
- **Logs**: `{route, params_hash, first_id, last_id, next_cursor_present}`.  
- **Metrics**: `list.requests`, `list.page_size_hist`, `list.overflow`, `list.offset_warning`.  
- **Traces**: span attributes: `sort_fields`, `cursor_keys`, `filters_applied`.

---

## Anti-patterns

- Sorting on a **non-unique** key without a tiebreaker.  
- Switching sort fields **between pages** (e.g., client changes `sort` mid-run).  
- Exposing **raw offsets** for high-churn lists.  
- **Mutable filters** in cursor (cursor must encode sort keys, not entire query if unsafe).  
- Returning **different shapes** for the same endpoint (schema drift).  
- Not documenting **UTC** and **inclusivity** for date filters.

---

## Review Checklist (quick gate)

- [ ] **Cursor pagination** offered; offset caveats documented  
- [ ] **Stable sort** with a **tiebreaker** defined and enforced  
- [ ] `page_size` validated & clamped; sensible defaults and maximums  
- [ ] Filters normalized (types, domains) and **validated** with message IDs  
- [ ] **UTC** timestamps and **gte/gt/lte/lt** semantics documented  
- [ ] Soft-deleted items hidden by default; explicit `include_deleted=true` support (policy-based)  
- [ ] Invariants tested: **no dupes**, **no skips**, **monotonic cursors**, **prev/next round-trip**  
- [ ] Concurrency tests simulate **inserts while paging** (cursor passes; offset fails)  
- [ ] Observability: logs/metrics/traces include paging/filter metadata  
- [ ] Total count semantics (exact vs approximate) clear if returned

---

## CSV Seeds

**Paging parity run**

```csv
page,ids,count,dupes,skips
1,"[200,199,198,197,196]",5,0,0
2,"[195,194,193,192,191]",5,0,0
3,"[190,189,188,187,186]",5,0,0
```

**Filter boundary checks**

```csv
case,created_at_gte,created_at_lt,expected_contains,expected_excludes
B-001,2025-09-01T00:00:00Z,2025-09-02T00:00:00Z,ids on 09-01 exact midnight,ids after 09-02 00:00:00Z
B-002,2025-09-15T12:33:59Z,2025-09-15T12:34:00Z,1009,1010
```

**Page size policy**

```csv
input_page_size,clamped_to,expected_code
1000,100,200
0,1,400
-5,1,400
```

---

## OpenAPI (snippet)

```yaml
paths:
  /orders:
    get:
      parameters:
        - in: query
          name: sort
          schema: { type: string, example: "created_at.desc,id.desc" }
        - in: query
          name: page_size
          schema: { type: integer, minimum: 1, maximum: 100, default: 25 }
        - in: query
          name: cursor
          schema: { type: string, description: "Opaque; from previous response" }
        - in: query
          name: created_at.gte
          schema: { type: string, format: date-time }
        - in: query
          name: created_at.lt
          schema: { type: string, format: date-time }
        - in: query
          name: status.in
          schema: { type: string, example: "active,cancelled" }
      responses:
        "200":
          description: Paged list of orders
        "400":
          description: Invalid sort/filter/page params
```

---

## Links

- State models & pagination invariants: `../20-techniques/state-models.md`  
- Cross-feature interactions (paging under writes): `../30-scenario-patterns/cross-feature-interactions.md`  
- Idempotency & Retries: `./idempotency-and-retries.md`  
- Error taxonomy: `./error-taxonomy.md`
