# Design Notes

> “Predictable pricing, zero surprises.”  
> These notes capture architecture, contracts, failure modes, and test hooks for **Discount Code v1**. Pair with the **Brief** for acceptance and metrics.

---

## 1) Goals & Non‑Goals

**Goals**
- Deterministic pricing with **one** order-level code per cart.
- Strong **observability** and **abuse defenses**.
- **Idempotent** apply/remove with safe retries.
- Production-ready under burst traffic (flash sales).

**Non‑Goals (v1)**
- Multi-code stacking, BOGO/bundles, tiered comps.
- Loyalty/points, gift card logic changes.
- Subscription proration.

---

## 2) Architecture (logical)

```
[Web/Mobile UI]
     |
     v
[Checkout API] --(RPC)--> [Pricing Service]
     |                         |
     |                         +--(RPC)--> [Promo Service]
     |                         +--(RPC)--> [Catalog/Inventory]
     |                         +--(RPC)--> [Tax Service]
     |
     +--(Events)--> [Analytics/Event Bus]
     +--(DB)-----> [Orders + Redemptions]
     +--(Cache)--> [Redis: code meta, eligibility, cart price]
```

**Notes**
- Checkout API orchestrates. Pricing is stateless; promo maintains code metadata.
- Redis caches **code metadata** (TTL 5–15 min) and **eligibility** (TTL 60 s per `code+cart_hash`).
- DB is source of truth for **redemptions** and **order totals**.

---

## 3) Core flows (sequence)

### 3.1 Apply code

```
Client -> Checkout API: POST /c/{cart}/discounts/apply { code }
Checkout API:
  - validate request + rate-limit
  - fetch code meta (cache -> promo)
  - build eligibility context (user, items, totals)
  - call Pricing: price(cart, code, ctx)
Pricing:
  - evaluate constraints + eligible lines
  - compute discount, rounding (half-even), caps
  - return priced breakdown
Checkout API:
  - persist cart state (applied_code snapshot)
  - emit events + logs
  - respond with priced totals
```

### 3.2 Remove code

```
Client -> Checkout API: DELETE /c/{cart}/discounts/apply
API:
  - clear applied_code snapshot
  - reprice without code
  - respond + emit events
```

### 3.3 Commit order (with code)

```
API -> Payments: authorize(…)
API -> Persist order + redemption row (unique redemption_id)
API -> Emit order.created + discount.applied
```

---

## 4) Data model (persistence)

```sql
-- promos (owned by Promo Service; replicated read)
CREATE TABLE discount_codes (
  code TEXT PRIMARY KEY,                 -- case-insensitive, normalize
  type TEXT CHECK (type IN ('percent','fixed','free_shipping')),
  rate_pct NUMERIC,                      -- 1..100
  amount_minor INTEGER,
  currency TEXT,                         -- for fixed
  starts_at TIMESTAMPTZ,
  ends_at TIMESTAMPTZ,
  min_subtotal_minor INTEGER DEFAULT 0,
  usage_limit_total INTEGER,
  usage_limit_per_user INTEGER,
  status TEXT CHECK (status IN ('active','paused','expired')),
  created_at TIMESTAMPTZ,
  updated_at TIMESTAMPTZ
);

-- per-order redemption
CREATE TABLE discount_redemptions (
  redemption_id TEXT PRIMARY KEY,        -- unique, idempotency target
  order_id TEXT UNIQUE,                  -- one code per order v1
  code TEXT REFERENCES discount_codes(code),
  user_id TEXT,                          -- hashed in logs, plain in DB
  amount_minor INTEGER NOT NULL,
  currency TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL
);
CREATE INDEX ON discount_redemptions (code, user_id);

-- counters (optional if using atomic upsert elsewhere)
CREATE TABLE discount_usage_counters (
  code TEXT PRIMARY KEY,
  total_used INTEGER NOT NULL DEFAULT 0,
  updated_at TIMESTAMPTZ NOT NULL
);
```

**Normalization**
- Store **snapshots** of code parameters on the order for audit (rate, fixed amount, constraints at apply-time).

---

## 5) Validation & eligibility (pipeline)

1. **Normalize**: uppercase, trim, Unicode NFC.
2. **Basic format**: `[A-Z0-9]{3,32}`.
3. **Temporal**: `starts_at/ends_at` with clock skew tolerance (±2 min).
4. **Status**: `active`.
5. **Usage limits**: total and per-user; checked atomically at commit time.
6. **Cart constraints**: min subtotal, product/category allow/block lists.
7. **Shipping constraint** (free shipping): method must be eligible.

Return **reason codes** (non-PII) for ineligibility; map to taxonomy.

---

## 6) Pricing rules

- Order-level code applies to **eligible lines** only.  
- **Percent**: `discount = round_half_even(eligible_subtotal * rate_pct/100)`  
- **Fixed**: `discount = min(fixed_amount, eligible_subtotal)`  
- **Free shipping**: set shipping price to `0` for allowed methods.  
- **Rounding**: per currency minor unit; ties **to even** (banker’s).  
- **Caps**: not below zero; no negative totals.

---

## 7) Idempotency & concurrency

**Apply / Remove**
- Require `Idempotency-Key` on apply.  
- Fingerprint payload + cart state; store response for replay.  
- Respond with `Idempotency-Status: replayed` when matched.

**Commit**
- Create `redemption_id` once; DB **unique** constraint guarantees **exactly-once**.  
- If retry races, the second writer gets `409 ERR.CONFLICT.idempotency`.

**Usage limits**
- Enforced on **commit**, not on apply, to avoid reservation deadlocks.  
- Opportunistic warning on apply if close to limit.

---

## 8) Caching strategy

- **Code metadata**: Redis key `code:meta:{code}`; TTL 5–15 min; stampede protection (single-flight).  
- **Eligibility**: `elig:{code}:{cart_hash}` TTL 60 s to curb abuse.  
- **Negative cache** for unknown/expired codes (TTL 1–5 min).  
- **Invalidation**: on promo changes, publish `promo.invalidate(code)`; subscribers purge cache.

---

## 9) Consistency model

- Pricing is **deterministic** given `(cart, code, ctx)`.  
- **Read-your-writes** for cart state via same store/partition.  
- Eventual consistency between Promo and Checkout caches is acceptable; final truth at **commit**.

---

## 10) Security, privacy, abuse

- **Enumeration defense**: constant-time compare; generic errors for invalid vs expired (optional more detail in body with **reason codes** guarded).  
- **Rate limiting**: per IP + device + user; token bucket + cooldown.  
- **Velocity** rules: attempts/min; distinct codes tried.  
- **Device binding** (cookie/local storage) for guests to dampen rotation.  
- **Logs**: hash user ids; **no** raw emails or PAN; include `msgid` and `err.code`.  
- **Staff overrides** audited (agent id, reason, ticket).  
- **Privacy**: consent flags honored for analytics events; no marketing payload on transactional endpoints.

---

## 11) Observability (signals)

**Logs (message IDs)**
- `MSG.discount.apply.requested`
- `MSG.discount.apply.succeeded`
- `MSG.discount.apply.failed`
- `MSG.discount.removed`
- `MSG.redemption.created`

**Error codes**
- `ERR.VALIDATION.code.format`
- `ERR.BUSINESS.code.ineligible`
- `ERR.RATE.limit`
- `ERR.CONFLICT.idempotency`
- `ERR.DEPENDENCY.timeout`

**Metrics**
- `discount_apply_latency_ms` (histogram)  
- `discount_attempts_total{result}`  
- `discount_apply_error_total{code}`  
- `discount_eligibility_cache_hit_ratio`  
- `redemption_created_total`

**Traces**
- Spans: `promo.validate`, `pricing.compute`, `tax.calculate`  
- Exemplars attached to latency histograms.

Dashboards: route p95/p99, attempts breakdown, error codes, cache hit ratio.

---

## 12) Failure modes & mitigations

- **Promo slow/5xx** → open breaker; allow checkout without discount; banner shown.  
- **Tax slow/5xx** → estimate with last-known rate; mark order for **recalc**.  
- **Catalog mismatch** → ignore non-eligible lines; log `ERR.BUSINESS.code.ineligible`.  
- **Cache outage** → fall back to direct RPC; raise latency budget by 20% temporarily.  
- **Clock skew** → accept within ±2 min; log skew metric.

---

## 13) UX & a11y hooks

- UI states: **Empty**, **Applying**, **Applied**, **Invalid**, **Expired**, **Rate-limited**.  
- Keep user input; show **Retry** on transient errors.  
- Announce errors via `aria-live="polite"`; focus back to code input.  
- Message IDs bound to copy for localization and contract stability.

---

## 14) i18n & currency

- Codes are **ASCII** only v1; reject non-ASCII to avoid homoglyph abuse.  
- Currency handling:  
  - **Percent**: currency-agnostic.  
  - **Fixed**: code currency must equal order currency (v1).  
- VAT/GST toggle: tax **after discount** where law allows (configured).

---

## 15) Performance & capacity

- Budgets: apply p95 ≤ **250 ms**, preview ≤ **200 ms**.  
- Expected RPS: baseline 100, burst 500+.  
- Redis hit ratio target ≥ **0.9** for code meta.  
- Prewarm top codes before campaigns.

---

## 16) Rollout plan

- Flags: `discount_code.v1` (server & client).  
- Canary 10% → 50% → 100%; guardrails on error rate and p95.  
- Kill switch: disable promo dependency, keep checkout flowing.  
- Backfill: none (v1); future migration when stacking introduced.

---

## 17) Testing strategy

**Unit**
- Rounding property tests; boundary of min subtotal; free shipping eligibility.

**Contract**
- OpenAPI schema validation; rejects **unknown fields**.

**Integration**
- Pricing with catalog/tax stubs; promo cache hit/miss.

**E2E**
- MAE scenarios from the Brief; evidence attached.

**Load**
- Spike 3×; coordinated-omission-correcting tool; capture p95/p99, error rate.

**Chaos**
- Inject promo brownout; verify breaker + banner; SLO holds.

**Telemetry assertions**
- Logs contain `msgid` + `err.code`; metrics increment; traces present.

---

## 18) Open questions (track to closure)

- Do we allow **agent-only** codes with scope to org/tenant?  
- Fixed amount in **different currency** than order (convert at issuance vs apply)?  
- Free shipping with **multi-shipment** carts—apply per shipment or whole order?

---

## 19) Appendix — Apply endpoint schema (stub)

```yaml
post:
  /v1/checkout/{cart_id}/discounts/apply:
    headers:
      Idempotency-Key: string
      X-Correlation-Id: string
    body:
      type: object
      required: [code]
      properties:
        code: { type: string, pattern: "^[A-Z0-9]{3,32}$" }
    responses:
      "200":
        $ref: "#/components/schemas/PricingResponse"
      "400":
        $ref: "#/components/schemas/Error"
      "409":
        $ref: "#/components/schemas/Error"
      "429":
        $ref: "#/components/schemas/Error"
```

---

## 20) Links

- Brief → `./brief.md`  
- API Coverage → `../../60-checklists/api-coverage.md`  
- Performance → `../../60-checklists/performance-review.md`  
- Security → `../../60-checklists/security-review.md`  
- Observability → `../../57-cross-discipline-bridges/for-developers.md`  
- SRE (SLOs & breakers) → `../../57-cross-discipline-bridges/for-sres.md`
