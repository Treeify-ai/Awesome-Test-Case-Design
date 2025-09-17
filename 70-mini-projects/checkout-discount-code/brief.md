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

# Checkout Discount Code — Brief

> Add a **discount/promo code** capability to checkout that is **predictable**, **auditable**, and **safe** under load and abuse.  
> This brief is a single source of truth for scope, contracts, rules, and testable acceptance.

---

## Problem → Value

**Problem**: Users cannot redeem marketing offers at checkout; agents must issue manual refunds; inconsistent pricing leads to churn.  
**Value**: Increase conversion and average order value, reduce support burden, enable targeted promotions.

---

## Outcomes (measurable)

- Conversion uplift on discount-eligible carts **≥ +1.5%** absolute.  
- Refunds due to pricing mistakes **≤ 0.1%** of orders.  
- Discount apply latency p95 **≤ 250 ms**; verify p95 **≤ 300 ms**.  
- Abuse (invalid/expired/overused attempts) blocked with 429 **≥ 99%**.

**Guardrails**:  
- Gross margin per order **not below floor**.  
- Refund rate **≤ 0.5%**; chargebacks **no increase**.

---

## Users & Roles

- **Shopper (guest/auth)** — applies/removes code.  
- **Agent** — views code effects, may override (policy bound).  
- **Partner/Promo Ops** — creates codes, sets rules/limits.  
- **System** — validates, applies, logs, prevents abuse.

---

## Scope

**In**  
- Apply/remove **one code** per order (v1).  
- Discount types: **percentage**, **fixed amount**, **free shipping**.  
- Constraints: **min subtotal**, **customer eligibility**, **product/category allowlist/blocklist**, **start/end**, **usage limits** (per code/per user).  
- **Stacking** disabled (v1).  
- **Multi-currency**: discount computed in order currency.

**Out (v1)**  
- Multi-code stacking, tiered/BOGO, bundle pricing.  
- Loyalty points, store credit, subscription proration.

---

## Business Rules (deterministic)

- **Order of operations**:  
  1) Item price × quantity → line subtotal  
  2) Item-level discounts (none in v1)  
  3) **Order-level code** applies to **eligible lines**  
  4) Shipping  
  5) Tax computed **after** discount where jurisdiction allows (config flag)

- **Rounding**: round to **minor units** (e.g., cents) per line, **half-even**.  
- **Caps**: discount ≤ eligible subtotal; not below zero.  
- **Eligibility caches**: 60 s TTL per code+cart hash.

**Examples**  
- Percentage: `discount = round_half_even(eligible_subtotal × rate)`  
- Fixed: `discount = min(fixed_amount, eligible_subtotal)`  
- Free shipping: `shipping_total = 0` for eligible methods only.

---

## Contracts (API)

### Apply code

`POST /v1/checkout/{cart_id}/discounts/apply`  
Headers: `Idempotency-Key`, `X-Correlation-Id`  
Body:
```json
{ "code": "SAVE15" }
```
Success `200`:
```json
{
  "cart_id": "c_123",
  "applied_code": {
    "code": "SAVE15",
    "type": "percent",
    "rate_pct": 15,
    "reason": "promo.public",
    "constraints": {
      "min_subtotal_minor": 5000,
      "starts_at": "2025-09-01T00:00:00Z",
      "ends_at": "2025-10-01T00:00:00Z",
      "usage_limit_total": 100000,
      "usage_limit_per_user": 3
    }
  },
  "pricing": {
    "items": [/* line totals after discount */],
    "subtotal_minor": 7900,
    "discount_minor": 1185,
    "shipping_minor": 900,
    "tax_minor": 540,
    "total_minor": 8155,
    "currency": "USD"
  }
}
```
Errors `4xx` (body uses taxonomy):  
- `ERR.VALIDATION.code.format` — not alphanumeric or wrong length  
- `ERR.BUSINESS.code.ineligible` — constraints failed  
- `ERR.RATE.limit` — brute-force attempts  
- `ERR.CONFLICT.idempotency` — mismatched replay

### Remove code

`DELETE /v1/checkout/{cart_id}/discounts/apply` → pricing recomputed.

### Pricing preview (idempotent)

`POST /v1/checkout/{cart_id}/pricing/preview` accepts optional `"code"` to simulate without persisting.

---

## Telemetry & Observability

- **Logs**: `MSG.discount.apply.requested`, `MSG.discount.apply.succeeded`, `MSG.discount.apply.failed` with `err.code`.  
- **Metrics**:  
  - `discount_apply_latency_ms` (histogram)  
  - `discount_apply_error_total{code}`  
  - `discount_eligibility_cache_hit_ratio`  
  - `discount_attempts_total{result=applied|invalid|expired|ineligible|rate_limited}`  
- **Traces**: span around validation, pricing, dependency calls (promo service, catalog, tax).  
- **Events**: `discount.applied`, `discount.removed` with `order_id` or `cart_id`.

---

## Security, Abuse & Privacy

- **Enumeration defense**: constant-time compare; generic error for invalid vs expired; rate-limit **per IP + device + account**.  
- **Brute force**: token bucket, cooldown after N failures; captcha **only** on high risk.  
- **Fraud signals**: velocity per card/account/device; high-risk → require auth or block.  
- **Privacy**: no PII in logs; hash user ids; consent respected for analytics.

---

## i18n & UX

- Copy mapped to **message IDs**:  
  - `discount.apply.success.title` → “Discount applied”  
  - `discount.apply.invalid.title` → “Code can’t be used”  
  - Secondary text suggests next actions (remove/try another).  
- **Empty/Loading/Error** and **focus** flows designed (see Design/UX bridge).  
- **RTL** layouts verified; long-string expansion ×1.3.

---

## Performance & Resiliency

- p95 budgets: apply ≤ **250 ms**, preview ≤ **200 ms** at 100 RPS/cart.  
- Backoff+jitter on promo service and tax/billing; **deadline < client timeout**.  
- **Circuit breaker**: if promo service brownouts, allow checkout **without** discount and surface banner.

---

## Data Model (simplified)

```yaml
DiscountCode:
  code: string           # case-insensitive
  type: enum[percent,fixed,free_shipping]
  rate_pct: number?      # 1..100
  amount_minor: integer?
  currency: string?      # for fixed
  constraints:
    min_subtotal_minor: integer?
    user_allowlist: string[]?
    product_allowlist: string[]?
    product_blocklist: string[]?
    starts_at: datetime
    ends_at: datetime
    usage_limit_total: integer
    usage_limit_per_user: integer
  status: enum[active,paused,expired]
```

---

## Acceptance (MAE) — testable & evidence

### MAIN — Valid code, percent
- **Given** eligible cart subtotal `≥` min  
- **When** user applies `SAVE15`  
- **Then** discount applied; totals recomputed; event/log emitted  
- **Evidence**: `200` body, metrics p95 snapshot, `MSG.discount.apply.succeeded`, trace id

### ALT — Fixed amount with currency & cap
- **Given** eligible lines sum `< fixed_amount`  
- **Then** discount limited to subtotal; total never < 0  
- **Evidence**: response totals; rounding verified to minor units

### ALT — Free shipping on eligible method
- **Given** shipping method `standard` eligible  
- **Then** shipping cost `0`; switching method not eligible restores fee

### EXCEPTION — Expired/Not started
- **Then** `400` with `ERR.BUSINESS.code.ineligible`; no pricing change  
- **Evidence**: log shows `apply.failed` + taxonomy code; no event `discount.applied`

### EXCEPTION — Rate limit / brute force
- **Then** `429` with `Retry-After`; attempts metric increments; no email/SMS sent  
- **Evidence**: limiter metrics; absence of apply success

### EXCEPTION — Promo service brownout
- **Then** breaker opens; UI shows banner; checkout allowed sans discount  
- **Evidence**: breaker metrics; journey SLI within budget

---

## Edge Cases

- Case-insensitive codes; trim whitespace; normalize Unicode.  
- Usage limit race → transactional decrement with **unique redemption id**.  
- Multi-currency fixed amounts: convert at **locked rate** when issuing the code; store both.  
- Tax after discount toggle per jurisdiction.  
- Gift cards and discounts order: gift card applies **after** discount (configurable).

---

## Synthetic & Data

- **Synthetic** journey per region: apply valid, invalid, expired.  
- **Fixtures**:  
  - Code `SAVE15` (15%); window wide; limits high  
  - Code `LESS500` (fixed 500 minor units)  
  - Code `SHIPFREE` (free shipping standard)  
  - Code `NEWUSR` (per-user = 1)

---

## Rollout & Risk

- Feature flag: `discount_code.v1` (server & client).  
- Canary to **10%**, monitor apply error rate & p95; rollback if guardrails breach.  
- Known risks: promo dependency latency, tax recalculation errors, refund surge.  
- Mitigations: cache, circuit breaker, precise rounding tests.

---

## Evidence to attach in PR

- Screenshots (apply/remove success & error; light/dark; LTR/RTL).  
- HAR of apply/remove; waterfals with timings.  
- Logs with `msgid` & `err.code` samples.  
- Trace links (slowest p99 apply).  
- Metrics screenshots (latency, attempts breakdown).  
- CSV test runs & reconciliation totals.

---

## Checklists

### Spec Ready (G1)
- [ ] Contracts outlined; message IDs  
- [ ] MAE acceptance written  
- [ ] NFR budgets set; telemetry plan

### Pre-merge (G4)
- [ ] Functional/API tests green  
- [ ] Perf p95/p99 snapshots  
- [ ] A11y/RTL quick pass  
- [ ] Security/abuse checks  
- [ ] Evidence attached

---

## CSV Seeds

**Promo codes (fixtures)**

```csv
code,type,rate_pct,amount_minor,currency,min_subtotal_minor,starts_at,ends_at,usage_limit_total,usage_limit_per_user
SAVE15,percent,15,,,5000,2025-09-01T00:00:00Z,2025-10-01T00:00:00Z,100000,3
LESS500,fixed,,500,USD,0,2025-09-01T00:00:00Z,2025-12-31T00:00:00Z,50000,10
SHIPFREE,free_shipping,,,,0,2025-09-01T00:00:00Z,2025-12-31T00:00:00Z,200000,5
NEWUSR,percent,10,,,0,2025-09-01T00:00:00Z,2026-01-01T00:00:00Z,1000000,1
```

**SLOs**

```csv
slo_id,description,objective,window,owner
discount_apply_latency_p95,Apply endpoint p95,<=250ms,7d,Checkout
discount_apply_error_rate,Error rate,<=1%,7d,Checkout
discount_invalid_block_rate,Invalid attempts blocked,>=99%,7d,Security
```

**Message IDs**

```csv
id,text
discount.apply.success.title,"Discount applied"
discount.apply.error.generic.title,"Code can’t be used"
discount.apply.error.expired.body,"This code has expired"
discount.apply.error.ineligible.body,"Your cart doesn’t meet the requirements"
```

---

## Templates

**Decision notes (one-pager)**

```
Decision: Tax after discount = true (US/EU), false (BR)
Reason: align with jurisdictions + revenue policy
Impacts: pricing service, invoice, reporting
Evidence: tax calc tests, jurisdiction matrix
```

**Test case skeleton**

```
CASE: API-Apply-Percent-Valid-001
Pre: cart subtotal 100.00 USD, items eligible
Steps: POST /discounts/apply code=SAVE15
Expected: discount 15.00, total 85.00, p95 < 250ms
Signals: log MSG.discount.apply.succeeded, metric discount_apply_latency_ms, trace id
Evidence: HAR, response JSON, screenshot, trace link
```

---

## Links

- API Coverage → `../../60-checklists/api-coverage.md`  
- Performance → `../../60-checklists/performance-review.md`  
- Security/Abuse → `../../60-checklists/security-review.md`  
- Design/UX → `../../57-cross-discipline-bridges/for-design-ux.md`  
- Metrics → `../../65-review-gates-metrics-traceability/metrics.md`  
- Traceability → `../../65-review-gates-metrics-traceability/traceability.md`

---

<p align="center">
  <sub>
    Built with <a href="https://treeifyai.com">Treeify</a> — AI test case design copilot.<br>
    <em>Structured, traceable, high-coverage test design — faster.</em>
  </sub>
</p>
