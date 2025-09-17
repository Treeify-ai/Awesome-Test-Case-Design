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


# Cross-feature Interactions (Pattern)

> Most production bugs live **between** features: promo vs gift card, pagination vs imports, cache vs DB, flags vs eligibility.  
> This pattern helps you map **interaction edges**, select a **minimal scenario set**, and define **evidence** so integration bugs don’t slip through.

---

## What & Why

- Features rarely operate in isolation. Interactions introduce **ordering, consistency, and policy** constraints that single-feature tests miss.
- Treat interactions as **first-class scenarios** with their own oracles (events, logs, metrics) and invariants.

Use this for: checkout stacks (promo × tax × shipping × gift card), list views (pagination × writes), account flows (auth × 2FA × lockout), content (search index × soft delete), data lifecycle (archive × export), feature flags (rollout × eligibility), and multi-service sagas.

---

## Quick workflow (8 steps)

1. **List features** in the slice and draw a **dependency map** (who reads/writes whom, sync vs async).  
2. Identify **edges** that change behavior (promo ↔ gift card conflict; cache ↔ DB read-after-write; pagination ↔ concurrent inserts).  
3. For each edge, write **invariants** (e.g., “single discount applied”, “no duplicate/skip across pages”).  
4. Decide **consistency model**: strong, read-after-write, eventual + compensation.  
5. Capture **contracts**: API schemas, event shapes + idempotency keys, outbox/inbox behavior.  
6. Design **MAE scenarios** that exercise the interactions (keep to 6–10).  
7. Add **faults** (latency, timeouts, out-of-order events) to 1–2 scenarios.  
8. Define **oracles & evidence**: cross-service correlation IDs, logs with rule IDs, metrics counters.

---

## Interaction Map (template)

| Node (feature/service) | Reads from | Writes to | Events emitted | Notes |
|---|---|---|---|---|
| Checkout UI | Promotions API, Tax API, Cart | Cart | `ui.apply.clicked` | sends idempotent verify |
| Promotions | Cart, Catalog | Cart | `promo.applied`, `promo.rejected` | conflict rules vs gift card |
| Gift Card | Balance | Cart | `giftcard.applied` | single redeem per cart |
| Tax | Catalog, Address | Cart | `tax.recalculated` | depends on address |
| Cart | - | Cart DB, Search Index | `cart.updated` | totals = sum(items) − discounts |

---

## Contracts (event/APIs) — example

| Producer → Consumer | Name | Schema (extract) | Idempotency | Versioning |
|---|---|---|---|---|
| Promotions → Cart | `promo.applied` | `{cart_id, code, amount, rule_id, correlation_id}` | `correlation_id` | `v1` |
| Promotions → Cart | `promo.rejected` | `{cart_id, code, error_code, correlation_id}` | `correlation_id` | `v1` |
| GiftCard → Cart | `giftcard.applied` | `{cart_id, giftcard_id, amount}` | `dedupe_key=cart_id+giftcard_id` | `v1` |
| Cart (API) | `POST /discount/verify` | `{code}` → `{applied, amount}` | `Idempotency-Key` | OpenAPI v1 |

---

## MAE Scenarios — Checkout: Promo × Gift Card × Tax

### S-001 **Main** — Promo applied before tax calculation
- **Preconditions**: valid percent code; no gift card; address present
- **Steps**: apply promo → recalc tax → update total
- **Expected**: one discount line; tax on discounted subtotal
- **Oracles**: `promo.applied` event; `tax.recalculated`; cart total matches formula

### S-002 **Alt** — Gift card first, then promo (combinable)
- **Preconditions**: combinable fixed code; gift card with balance
- **Steps**: apply gift card → apply promo
- **Expected**: both apply; cap gift card to new subtotal; no negative totals
- **Oracles**: cart lines (giftcard then promo), invariant `total ≥ 0`

### S-003 **Alt** — Idempotent promo re-apply
- **Steps**: click apply twice with same key
- **Expected**: same outcome; no duplicate lines
- **Oracles**: logs show same `Idempotency-Key`; one discount row

### S-101 **Exception/Policy** — Non-combinable percent with gift card
- **Preconditions**: percent code marked non-combinable; gift card applied
- **Steps**: apply promo
- **Expected**: `CONFLICT.code.not_combinable`; offer remove gift card
- **Oracles**: decision table rule ID in logs; `promo.rejected` with code

### S-102 **Exception/Ordering** — Promo after address change
- **Preconditions**: address change triggers tax recalculation
- **Steps**: apply promo → change address (tax zone)
- **Expected**: totals recomputed correctly; promo amount based on new taxable subtotal
- **Oracles**: `tax.recalculated` follows promo; final total matches formula

### S-103 **Exception/Resilience** — Tax API timeout with retry
- **Steps**: promo ok → tax call times out → client retries within budget
- **Expected**: eventual success; no duplicate discount; backoff observed
- **Oracles**: metric `retry_count`; traces with same correlation ID

---

## MAE Scenarios — List View: Pagination × Concurrent Inserts

### S-201 **Main** — Stable cursor pagination under writes
- **Preconditions**: sort `(created_at DESC, id DESC)`; cursor API
- **Steps**: page 1 → simulated inserts → page 2 → page 3
- **Expected**: **no duplicates, no skips**
- **Oracles**: response cursors monotonic; item IDs unique across pages

### S-202 **Alt** — Offset pagination with low write rate
- **Expected**: acceptable (document caveat)
- **Oracles**: no skips/dupes in a quiet window

### S-203 **Exception** — Offset pagination under heavy inserts
- **Expected**: duplicates or skips detected; recommend cursor; gate fails
- **Oracles**: test detects drift; report includes conflicting timestamps

> Pair this with `../40-api-and-data-contracts/pagination-and-filtering.md` (stable sorts + invariants).

---

## Selecting a minimal set

- Use **pairwise** across features (`Promo × Gift Card × Tax × Address`) to pick **~8–12 combinations**.  
- Elevate **triples (t=3)** for known risky triples (e.g., `Device × Wallet × Locale`).  
- Layer **Boundary** inputs into 1–2 rows (e.g., promo code max length).

---

## Invariants (assert across services)

- **Totals balance**: `total = items + shipping + tax − discounts − giftcard`, never negative.  
- **Single discount application** per code per cart.  
- **No duplicate/skip** across paginated pages.  
- **Idempotency** across retries/webhooks (same key ⇒ same outcome).  
- **Eventual consistency bound**: cart reflects promo within **T ≤ 2s** after `promo.applied`.

Represent invariants as **assertions** or **monitors** (metric thresholds).

---

## Fault injection (lightweight)

Add one fault to at least one scenario:

- **Latency**: inject 95th percentile delay on Tax API.  
- **Timeout**: first call fails, second succeeds.  
- **Out-of-order events**: deliver `giftcard.applied` after `promo.applied`.  
- **Cache stale**: add/evict delay; verify final consistency bound.  
- **Network split**: Promotions writes succeed; Cart index lags; recovery job reconciles.

---

## Oracles & Evidence

- **Events**: `promo.applied/rejected`, `tax.recalculated`, `cart.updated` with **correlation_id**.  
- **API responses**: codes & bodies; **message IDs** for conflicts/policy.  
- **Logs**: `rule_id`, `from_state→to_state`, `idempotency_key`.  
- **Metrics**: retries, time to consistency, pagination parity errors.  
- **Traces**: spans across services stitched by correlation ID.

---

## Anti-patterns

- Testing features **in isolation** and assuming composition works.  
- No **stable sort/tiebreaker** on lists while writing concurrently.  
- Ignoring **non-combinable** policy interactions.  
- Missing **idempotency** on cross-feature operations (e.g., refund + ledger).  
- No **cross-service correlation**; debugging is guesswork.  
- Treating eventual consistency as “eventually never” (no bounds/monitors).

---

## Review checklist (quick gate)

- [ ] Interaction map done (reads × writes × events)  
- [ ] Contracts captured (schemas, idempotency, versions)  
- [ ] 6–10 MAE scenarios cover main, alt, and exception + **1 fault injection**  
- [ ] Invariants stated and testable across services  
- [ ] Pagination invariants if lists are involved  
- [ ] Oracles span events, logs, metrics, traces with correlation IDs  
- [ ] Pairwise/triple selection documented; boundaries layered in  
- [ ] Clear **consistency model** (read-after-write vs eventual + bounds)

---

## CSV seeds

**Interaction pairs (for coverage planning)**

```csv
pair,why_matter,invariant
Promo × GiftCard,conflict rules,single discount or helpful conflict
Promo × Tax,order/amount correctness,tax on discounted subtotal
GiftCard × Tax,cap after discounts,total>=0
Pagination × Inserts,ordering stability,no duplicate/skip
Cache × DB,read-after-write,staleness<=2s
Flags × Eligibility,rollout gates,flag implies capability
```

**Event contract list**

```csv
producer,event,keys,idempotency,version
Promotions,promo.applied,cart_id,correlation_id,v1
Promotions,promo.rejected,cart_id,correlation_id,v1
GiftCard,giftcard.applied,cart_id+giftcard_id,dedupe_key,v1
Cart,cart.updated,cart_id,correlation_id,v1
Tax,tax.recalculated,cart_id,correlation_id,v1
```

**Pagination parity run**

```csv
page,ids,count,dupes,skips
1,"[100,99,98,97,96,95,94,93,92,91]",10,0,0
2,"[90,89,88,87,86,85,84,83,82,81]",10,0,0
3,"[80,79,78,77,76,75,74,73,72,71]",10,0,0
```

---

## Templates

**Interaction map**

```
| Node | Reads | Writes | Events | Notes |
|------|-------|--------|--------|-------|
```

**Scenario card (interaction)**

```
## S-<nnn> <Type> — <interaction name>
Preconditions: <features+states>
Trigger: <user/api/event>
Steps: <observable sequence across features>
Expected: <joint outcome + invariants>
Oracles: <events+logs+metrics+traces across services>
Fault (optional): <latency/timeout/ordering/cache>
```

---

## Links

- Main/Alt/Exception flows: `./main-alt-exception.md`  
- Roles & Permissions: `./roles-and-permissions.md`  
- Boundary & Equivalence: `../20-techniques/boundary-and-equivalence.md`  
- Decision Tables: `../20-techniques/decision-tables.md`  
- State Models: `../20-techniques/state-models.md`  
- Pagination & Filtering: `../40-api-and-data-contracts/pagination-and-filtering.md`  
- Idempotency & Retries: `../40-api-and-data-contracts/idempotency-and-retries.md`

---

<p align="center">
  <sub>
    Built with <a href="https://treeifyai.com">Treeify</a> — AI test case design copilot.<br>
    <em>Structured, traceable, high-coverage test design — faster.</em>
  </sub>
</p>
