# Domain: Payments & Checkout

> People pay, money moves, risks explode. This playbook turns payment complexity into **clear flows, contracts, and evidence** so you can ship confidently across cards, wallets, bank debits, and marketplaces.

---

## TL;DR (defaults we recommend)

- Use a **Payment Intent** style model: `create → confirm(auth) → capture → (refund|dispute)` with explicit **states**.  
- Require **Idempotency-Key** for **all** unsafe POSTs (`/payments`, `/captures`, `/refunds`, `/voids`).  
- Treat **webhooks/events** as **at-least-once**: dedupe by `event_id`, order-independent handlers.  
- Store **amounts in minor units**; respect currency **exponent** (0, 2, 3).  
- Support **3DS/SCA** (challenge & frictionless) and **wallets** (Apple/Google Pay) as first-class flows.  
- Keep a **double-entry ledger** for money moves; reconcile to provider **payout files** daily.  
- Build **dunning** + **retry** policy for recurring payments; never blind-retry declines without rules.  
- Privacy & PCI: **never touch PAN**; use **hosted fields**/tokens; pick the correct **SAQ** level with your PSP.

---

## Scope (what this covers)

- **Instruments**: cards (credit/debit), tokenized wallets (Apple/Google Pay), bank debits (ACH/SEPA/etc.), BNPL (conceptually).  
- **Flows**: one-time purchase, auth-then-capture, partial capture, refunds (full/partial), voids, disputes/chargebacks, recurring/subscriptions, marketplace split.  
- **Surfaces**: web, mobile/webview, server-to-server, webhooks.  
- **Actors**: buyer, merchant, platform/marketplace, PSP (gateway/acquirer), antifraud provider, tax/shipping services.

---

## Key concepts (quick glossary)

- **Authorization (auth)**: bank approves hold on funds; not yet moved. Expires after T (hours–days).  
- **Capture**: finalize the auth; funds move; fees applied.  
- **Void**: cancel an **uncaptured** auth.  
- **Refund**: return settled funds (full/partial); may be **asynchronous**.  
- **Dispute/Chargeback**: cardholder challenges; evidence submitted; outcome **won/lost** much later.  
- **SCA / 3DS**: strong customer auth; may require **challenge**.  
- **Tokenization**: store **payment tokens**, not PANs; network or PSP tokens.  
- **Payout/Settlement**: batched transfers from PSP to merchant bank, net of fees/chargebacks.  
- **Ledger**: internal **double-entry** accounting system mirroring provider events.

---

## States & transitions (reference)

```
created → requires_action? → authorized → captured → settled → (refunded | disputed)
                 ↘ failed
authorized → voided (if before capture) | expired (auth timeout)
captured → partially_refunded* → refunded
disputed → evidence_submitted → won | lost
```

Observables: PSP payment/charge id, status, **webhook events**, ledger entries, receipt emails, audit.

---

## Contracts (must-haves)

- **Idempotency**: keys required for `POST /payments`, `/captures`, `/refunds`, `/voids`. On replay, **same response/body**; mismatch → `409 CONFLICT.idempotency.payload_mismatch`.  
- **Webhooks**: include unique `event_id`, `type` (e.g., `payment.authorized`), `occurred_at`, `data` (ids, amounts), and `correlation_id`. Consumers **dedupe** to an **inbox** table.  
- **Amounts**: integers in **minor units**; include `currency` (ISO‑4217). Keep a **currency exponent** table.  
- **Errors**: return standardized **message IDs** (see error taxonomy). Map provider decline codes to your taxonomy.  
- **Metadata**: include `order_id`, `customer_id`, `cart_id` for reconciliation and support.

---

## Amounts, currency, rounding

- **Minor units**: store as int.  
- Currency **exponent**: 0 (JPY, KRW), 2 (USD, EUR), 3 (BHD, KWD).  
- **Rounding**: define consistent rule (e.g., **banker’s** or **half‑away‑from‑zero**).  
- **Presentment vs settlement**: FX differences; store both if provider offers multi‑currency.  
- Prevent **over‑refund** (sum of refunds ≤ captured).

---

## SCA/3DS & wallets

- **3DS** paths: frictionless (no challenge) vs **challenge** with redirect/app switch.  
- **Wallets**: Apple/Google Pay return **payment tokens**; treat as card‑on‑file with device auth.  
- **Retry** policy: after failed challenge, allow **retry** or **fallback** instrument; never loop silently.

**Tests to include**  
- 3DS challenge success/failure; challenge timeout; device/app return path.  
- Wallet authorization success; **merchant validation** where required; network tokenization quirks.

---

## Fraud & risk (just enough)

- **Signals**: AVS/CVV match, device fingerprint, velocity, IP reputation, email/phone risk, history.  
- **Rules**: allowlist/denylist, step‑up SCA, hold for manual review.  
- **Liability shift**: 3DS success changes chargeback liability (region‑dependent).  
- **KYC/AML** (for marketplaces): verify sellers; block payouts until compliant.

---

## Dunning & recurring

- **Schedule**: smart retries (time of day, issuer guidance).  
- **Payment method updates**: account updater/network tokens.  
- **Grace periods** & **webhooks** to update access/entitlements.  
- **Invoicing**: retry window, invoice states, email notifications.

---

## Marketplace / split payments (if applicable)

- **Split** capture to **connected accounts**; platform takes a **fee**.  
- Handle **reversals** (refunds across splits) and **negative balances**.  
- Payout schedules per sub‑merchant; compliance (KYC, onboarding).

---

## Reconciliation (don’t skip this)

Daily: match provider **balance/transactions** to your **ledger** by `payment_id`/`charge_id` and **date/amount**.

- **Inputs**: provider payout reports, fee reports, FX rates.  
- **Outputs**: reconciled totals, unmatched list, adjustments.  
- **Evidence**: csv artifacts, checksums, signed reports.

---

## MAE Scenarios (copy/adapt)

### S-001 **Main** — Card auth → capture → receipt
- **Steps**: create payment intent → confirm (auth) → capture → send receipt  
- **Expected**: states: `authorized → captured`; ledger balanced; receipt email sent  
- **Oracles**: PSP events `payment.authorized`, `payment.captured`; ledger entries; email log

### S-002 **Alt** — Partial capture (split shipments)
- **Steps**: capture 60% → later capture 40% (before auth expiry)  
- **Expected**: two capture events; sum equals auth; invoices reflect shipments  
- **Oracles**: captures total == original; ledger shows two entries

### S-003 **Alt** — 3DS challenge success
- **Steps**: confirm triggers 3DS challenge → complete challenge → auth  
- **Expected**: payment state `authorized`; challenge flow recorded  
- **Oracles**: 3DS auth result, PSP status, audit trail

### S-101 **Exception** — Duplicate payment submit (double-click)
- **Steps**: two `POST /payments` with same **Idempotency-Key**  
- **Expected**: one payment; second is **replayed** response  
- **Oracles**: idempotency store; logs show `replayed=true`

### S-102 **Exception** — Provider timeout with replay
- **Steps**: first confirm times out; client retries with **same key**  
- **Expected**: exactly‑once capture; no double charge  
- **Oracles**: one PSP charge; one ledger entry

### S-103 **Exception** — Decline → fallback card
- **Steps**: decline with `insufficient_funds` → select second card → success  
- **Expected**: first payment **failed** with mapped error; second succeeds  
- **Oracles**: error code taxonomy; event trail

### S-104 **Exception** — Refunds (partial + idempotent)
- **Steps**: refund part A → retry same refund key → refund part B → try over‑refund  
- **Expected**: replays stable; total ≤ captured; over‑refund → `409 CONFLICT.refund.amount_exceeds`  
- **Oracles**: refund totals; ledger; events `refund.succeeded`

### S-105 **Exception** — Void after auth
- **Steps**: void before capture window closes  
- **Expected**: payment → `voided`; no capture happened  
- **Oracles**: PSP event; ledger compensating entries

### S-106 **Exception** — Dispute lifecycle
- **Steps**: dispute created → evidence submitted → outcome  
- **Expected**: state transitions + deadlines; access possibly revoked  
- **Oracles**: dispute events; evidence artifacts; emails

### S-201 **Cross-feature** — Promo × Gift card × Tax × Payment
- **Expected**: total never negative; tax on discounted subtotal; single discount application  
- **Oracles**: pricing engine outputs; cart lines; payment amount == final order total

### S-301 **Reconciliation** — Daily payout match
- **Steps**: ingest payout report → match transactions → produce unmatched list  
- **Expected**: sums match after fees/FX; small diffs explained  
- **Oracles**: csv artifacts; checksums; ledger balances to zero per day

---

## Evidence & observability

- **IDs**: `payment_id`, `charge_id`, `refund_id`, `dispute_id`, `payout_id`.  
- **Correlation**: `X-Correlation-Id` across API, jobs, webhooks.  
- **Metrics**: auth rate, capture success, 3DS challenge rate/success, decline reason mix, refund volume, dispute rate/outcome.  
- **Logs**: structured, **no PAN/PII**; include `idempotency_key`, `rule_id`, `error_code`.  
- **Traces**: steps in checkout; downstream timings; retries/backoff.

---

## PCI & compliance (minimum stance)

- Do **not** handle PAN directly; use PSP **hosted fields** or **client SDK tokens**.  
- Choose correct **SAQ** (A / A‑EP) with your PSP.  
- Encrypt at rest; restrict access to PII; audit sensitive actions.  
- Support **SCA/3DS** in regulated regions; keep a **country matrix** for when SCA applies.

---

## Test data & environments

- Use **PSP sandbox** tokens; use provider‑supplied **test card numbers** & 3DS simulators.  
- Seed **decline scenarios** via provider toggles (insufficient funds, lost/stolen, do not honor).  
- Avoid real PII in fixtures; use realistic names, addresses, locales, and currencies.

---

## Review checklist (quick gate)

- [ ] Contracts: idempotency on payments/captures/refunds/voids; stable errors with message IDs  
- [ ] 3DS/SCA & wallets supported with success and failure paths  
- [ ] Auth‑then‑capture, **partial capture**, **void** before capture, and **refunds** (full/partial) covered  
- [ ] Webhooks: **dedupe** by `event_id`; out‑of‑order tolerant; retry with backoff  
- [ ] Ledger: **double‑entry** balanced; daily **reconciliation** with provider reports  
- [ ] Currency: minor units; exponent table; over‑refund prevented; rounding defined  
- [ ] Dunning & retries for recurring; updater/network tokens supported  
- [ ] Privacy/PCI: no PAN in logs; hosted fields/tokens; SAQ chosen  
- [ ] Observability: metrics, logs, traces with correlation & ids; receipts/audit sent
- [ ] Evidence artifacts attached (csv reports, screenshots, transcripts)

---

## CSV seeds

**Currency exponent table**

```csv
currency,exponent
USD,2
EUR,2
JPY,0
KRW,0
KWD,3
BHD,3
```

**Ledger example (double-entry)**

```csv
entry_id,account,debit,credit,ref
E1,AR (customer),0,1099,ord_123
E2,Sales,1099,0,ord_123
E3,PSP Fees,299,0,ch_abc
E4,Cash (bank),800,0,payout_555
```

**Reconciliation pointers**

```csv
date,psp_gross,psp_fees,psp_net,ledger_net,match
2025-09-15,100000,3000,97000,97000,true
```

**Decline mix (sandbox)**

```csv
reason,count
insufficient_funds,12
do_not_honor,5
incorrect_cvc,3
expired_card,2
```

---

## Templates

**Flow outline**

```
Given: cart total, currency, customer
When: create intent → confirm (3DS?) → capture or void → (refund?)
Then: states & events transition; ledger balances; receipt sent
Evidence: PSP events, ledger entries, emails, traces
```

**Webhook handler skeleton**

```python
def handle_event(evt):
    if inbox.seen(evt.id):
        return "replayed"
    inbox.store(evt.id, evt.type, evt.occurred_at, hash(evt.payload))
    route(evt)  # order paid / refund succeeded / dispute opened
    inbox.mark_processed(evt.id)
```

---

## Links

- Idempotency & Retries: `../40-api-and-data-contracts/idempotency-and-retries.md`  
- Error Taxonomy: `../40-api-and-data-contracts/error-taxonomy.md`  
- Contracts & Schemas: `../40-api-and-data-contracts/contracts-and-schemas.md`  
- Data Lifecycle (refunds/chargebacks/ledger): `../30-scenario-patterns/data-lifecycle.md`  
- Cross-feature Interactions (promo × tax × gift card × payment): `../30-scenario-patterns/cross-feature-interactions.md`  
- Performance/Resiliency (timeouts & p99): `../50-non-functional/performance-p95-p99.md`, `../50-non-functional/resiliency-and-timeouts.md`
