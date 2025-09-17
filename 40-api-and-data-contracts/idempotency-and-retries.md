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


# Idempotency & Retries (Contract + Playbook)

> Networks fail. Users double‑click. Webhooks replay.  
> This playbook defines **contracts** for making unsafe operations **idempotent** and for applying **retries with backoff + jitter**—so we get *exactly‑once effects* in an at‑least‑once world.

---

## Scope

- **Client → Server APIs** (POST/PUT/PATCH/DELETE)  
- **Webhooks/Events** (producer → consumer)  
- **Jobs/Workers** (outbox/inbox processing)  
- Money‑moving and resource creation where duplicates are costly.

---

## Core definitions

- **Idempotent operation**: same request (by **key**) produces the **same effect** and **same response**, no matter how many times it’s received.  
- **Idempotency key**: client‑supplied unique token for a *logical* operation (e.g., `"refund:charge_123:amount_1000:uuid"`).  
- **Dedup store**: persistence that maps `{idempotency_key → outcome}` with TTL; must be **atomic** with the effect.  
- **Retry**: automatic re‑attempt on **transient** failures (timeouts, 429, 5xx), using **exponential backoff + jitter**.

> **Create + Reverse**: If you make `POST /payments` idempotent, do the **same** for `POST /refunds`. Symmetry prevents costly inconsistencies.

---

## HTTP contract (client → server)

| Aspect | Contract |
|---|---|
| Header | `Idempotency-Key: <opaque>` (required for unsafe POST/DELETE reverse ops) |
| Scope | Key is **scoped** to `{route + auth principal}` by default |
| Lifetime | Server caches outcome for **24–72h** (configurable) |
| Response | On replay: **same HTTP code, body, and side‑effects**; include `Idempotency-Status: replayed` header |
| Conflicts | If the **same key** arrives with a **different payload hash** → `409 CONFLICT idempotency.payload_mismatch` |
| Storage | Persist `{key, payload_hash, status, result_ref, expires_at}` atomically with the operation |
| Safe verbs | GET/HEAD/OPTIONS are already idempotent by RFC—no key needed |
| Conditional | For updates, prefer **PUT + If-Match (ETag)** to ensure optimistic concurrency |

**Payload hash**: cryptographic digest (e.g., SHA‑256 of canonical JSON). Store to detect mismatched replays.

---

## Retry policy (client & server)

**When to retry**

- HTTP **429**, **5xx**, network timeouts, connection resets → **retryable**  
- HTTP **4xx** other than 429 (**validation/auth/policy**) → **do not retry**

**Schedule (example)**

| Attempt | Delay (ms) | Notes |
|---:|---:|---|
| 1 | 0 | initial |
| 2 | 100–200 | `base=100` + **full jitter** |
| 3 | 200–400 | double base + jitter |
| 4 | 400–800 |  |
| 5 | 800–1600 | cap at **max_backoff=2s** |
| stop | — | total ≤ **10s** (tunable) |

Use **Retry‑After** if provided by the server.

**Headers**

- Request: `Idempotency-Key`, optionally `X-Request-Timeout` (client budget)  
- Response: `Idempotency-Status: replayed|stored`, `Retry-After`, diagnostic `error_code`

---

## Webhooks & event consumers

At‑least‑once delivery means **dedupe** on the consumer.

| Aspect | Contract |
|---|---|
| Event id | Include **unique id** and **source**; e.g., `event_id`, `source_service` |
| Idempotency | Consumer **must** dedupe using `{event_id}` or a derived **dedupe_key** |
| Storage | Inbox table: `{event_id PK, received_at, processed_at, result, payload_hash}` |
| Ordering | Do **not** assume order; model **state** to handle out‑of‑order |
| Retries | Use backoff on transient consumer errors; DLQ after N attempts |
| Visibility | Log `idempotency_key/event_id`, `from_state→to_state`, and `correlation_id` |

---

## Design patterns

### A) Create with idempotency (server‑side)

1. On request: compute `payload_hash`; **insert** `{key, hash, status=pending}` if absent (atomic).  
2. If key exists with **same hash** and **final result** → **return stored response** (`Idempotency-Status: replayed`).  
3. If key exists with **different hash** → **409 CONFLICT**.  
4. Execute effect; persist outcome; update idempotency row with `status=complete`, `result_ref`.  
5. Return response.

### B) Reverse ops (refunds, voids)

- Same as above; ensure **exactly‑once effect** even with retries.  
- If underlying provider is at‑least‑once, dedupe by a **provider request id** as well.

### C) Outbox → Inbox (sagas)

- Producer writes **domain change** + **outbox row** in one transaction.  
- A delivery job sends events; retries until acknowledged.  
- Consumer writes **inbox row** (dedupe) before handling; retries handler; side‑effects **after** inbox write.

---

## Worked example — Refund API

**Route**: `POST /refunds`  
**Key rule**: **required** for all money‑moving POSTs.

**Request**

```
POST /refunds
Idempotency-Key: refund:ch_9ab…:1000:6f6c…
{
  "charge_id": "ch_9ab…",
  "amount": 1000
}
```

**Server behavior**

- First attempt → creates refund `rf_123`, returns `201`.  
- Replay (same key) → returns the **same refund `rf_123`** with `Idempotency-Status: replayed`.  
- Replay with **different amount** under same key → `409 idempotency.payload_mismatch`.

**Tests (select)**

1. **Same key twice** → **same** refund id; **one** ledger entry.  
2. **Different key** same payload → **new** refund (when policy allows).  
3. Gateway timeout then retry (same key) → **one** payout.  
4. Duplicate webhook `refund.succeeded` → consumer dedupes; **one** ledger entry.

**Oracles**

- DB: **one** row in refunds + **one** ledger entry.  
- Logs: `{idempotency_key, outcome=replayed|created}`.  
- Metrics: `idempotent.replays`, `idempotent.conflicts`.  
- Traces: span attributes with `idempotency_key`, `attempt`.

---

## Error taxonomy (transient vs terminal)

Map errors to **retry** behavior and **UX copy**.

| Code | Category | Retry? | UX/Caller hint |
|---|---|---:|---|
| `VALIDATION.*` | Client input invalid | ❌ | Fix input |
| `AUTH.*` | Auth/authz failed | ❌ | Re‑auth or permission |
| `CONFLICT.idempotency.payload_mismatch` | Key reused with different payload | ❌ | Generate new key |
| `RATE_LIMIT.exceeded` | 429 | ✅ (respect Retry‑After) | Wait & retry |
| `DEPENDENCY.timeout` | 504/timeout | ✅ | Retry with backoff |
| `DEPENDENCY.unavailable` | 503 | ✅ | Retry |
| `TRANSIENT.*` | Unknown transient | ✅ | Retry |
| `PROVIDER.duplicate` | Provider replay | ❌ (already applied) | Surface **existing result** |

---

## Storage schema (reference)

```sql
-- Idempotency store
CREATE TABLE idempotency_keys (
  key TEXT PRIMARY KEY,
  payload_hash TEXT NOT NULL,
  status TEXT NOT NULL CHECK (status IN ('pending','complete','failed')),
  result_ref TEXT,            -- e.g., refund id or serialized response
  http_code INT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  expires_at TIMESTAMPTZ NOT NULL,
  UNIQUE (key)
);

-- Inbox (consumer dedupe)
CREATE TABLE inbox_events (
  event_id TEXT PRIMARY KEY,
  source TEXT NOT NULL,
  payload_hash TEXT NOT NULL,
  received_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  processed_at TIMESTAMPTZ,
  result TEXT
);
```

---

## Pseudocode (server)

```python
def handle_post_refund(req):
    key = req.headers["Idempotency-Key"]
    phash = sha256(canonical_json(req.body))
    row = idempotency.get(key)
    if row:
        if row.payload_hash != phash:
            return 409, error("CONFLICT.idempotency.payload_mismatch")
        if row.status == "complete":
            return row.http_code, row.result_ref, {"Idempotency-Status": "replayed"}
        # else pending → wait or 409 depending on policy
    else:
        idempotency.insert_atomic(key, phash, status="pending", expires_at=now()+ttl)

    try:
        refund = create_refund(req.body)  # may call provider with retries
        idempotency.update(key, status="complete", result_ref=refund, http_code=201)
        return 201, refund, {"Idempotency-Status": "stored"}
    except TransientError:
        # do not mark failed; client will retry with same key
        raise
    except Exception as e:
        idempotency.update(key, status="failed", http_code=500)
        raise
```

---

## Observability contract

- **Logs**: `idempotency_key`, `payload_hash`, `status`, `attempt`, `from_state→to_state`, `error_code`.  
- **Headers**: `Idempotency-Status`.  
- **Metrics**: counters `idempotent.created|replayed|conflict`, histograms for retry delays.  
- **Traces**: tag spans with `idempotency_key` and `attempt`.

---

## Anti‑patterns

- Idempotency only on **create** but not on **reverse** (refund/void).  
- Key stored only **in memory** (lost on crash).  
- No **payload hash** → accidental key reuse creates silent mismatches.  
- Returning different **HTTP codes** on replay.  
- Retrying **non‑retryable** errors (validation/auth).  
- Webhook consumers without an **inbox/dedupe** table.  
- Treating retries without **jitter** (herd stampedes).  
- Using **offset pagination** during writes without stability guarantees (see pagination doc).

---

## Review checklist (quick gate)

- [ ] `Idempotency-Key` required on unsafe POSTs (incl. **refunds/voids**)  
- [ ] **Payload hash** stored; **409** on mismatched replay  
- [ ] **Same response** (code/body) on replay + `Idempotency-Status` header  
- [ ] Retry only on **429/5xx/timeouts**; **exponential backoff + jitter**; honor **Retry‑After**  
- [ ] **Outbox/Inboxes** used; consumers dedupe by `event_id`  
- [ ] Observability: logs, metrics, traces include **idempotency key** and **attempt**  
- [ ] Storage TTL & cleanup tasks documented  
- [ ] Symmetry: **create and reverse** operations both idempotent

---

## CSV seeds

**Client retry log**

```csv
attempt,delay_ms,status,code,notes
1,0,timeout,,initial timeout
2,137,5xx,502,proxy error
3,281,200,201,success
```

**Replay outcomes**

```csv
key,payload_hash,status,http_code,result_id,notes
k1,abc,complete,201,rf_123,created
k1,abc,complete,201,rf_123,replayed
k1,xyz,conflict,409,,payload mismatch
```

**Consumer inbox**

```csv
event_id,source,first_seen,processed,result
ev_001,payments,2025-09-15T12:00:00Z,2025-09-15T12:00:03Z,ok
ev_001,payments,2025-09-15T12:00:05Z,2025-09-15T12:00:05Z,replayed
```

---

## Links

- Boundary & Equivalence: `../20-techniques/boundary-and-equivalence.md`  
- State Models: `../20-techniques/state-models.md`  
- Pagination & Filtering: `./pagination-and-filtering.md`  
- Error Taxonomy → UX: `./error-taxonomy.md`  
- Cross‑feature interactions: `../30-scenario-patterns/cross-feature-interactions.md`

---

<p align="center">
  <sub>
    Built with <a href="https://treeifyai.com">Treeify</a> — AI test case design copilot.<br>
    <em>Structured, traceable, high-coverage test design — faster.</em>
  </sub>
</p>
