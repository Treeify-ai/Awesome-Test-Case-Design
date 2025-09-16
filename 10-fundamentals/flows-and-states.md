# Flows & State

> Features aren’t just inputs and outputs—they’re **journeys through states**. Modeling **flows** (MAE: Main/Alt/Exception) and **state machines** gives you the scaffolding to write fewer tests with better coverage and clearer oracles.

---

## 1) Why flows & state?

- **Flow thinking** (MAE) captures **how** users/systems move through a task.
- **State modeling** captures **what can be true** at any moment and what **must not** be possible.
- Together they surface **invariants**, **guards**, and **observable outcomes**, which make tests **reliable and non-flaky**.

> If you can name the **current state** and the **event** that transitions it, you can design a precise, checkable test.

---

## 2) Quick workflow

1. **Name the goal** and **actors** (User, System, Scheduler, External Service).
2. Draft **MAE flows** (Main / Alt / Exception) as bullet steps.
3. Extract a **state list** from the flows (Nouns, not verbs).
4. Define **transitions**: `FROM --(event/trigger [guard])--> TO  {actions}`.
5. Write **invariants** (“must always hold”) and **terminal states**.
6. Add **observability**: log keys, metrics, traces for each transition.
7. Pick **edge cases**: retries, timeouts, cancellations, concurrency.

---

## 3) MAE flow skeleton

```
Main:     User applies discount code → API verify → Applied badge → Total updated
Alt:      Member applies after address step → Verify still valid → Applied
Exception: Expired code → Show message → Keep input → No total change
Exception: Not combinable → Offer remove conflict item
```

Keep each step an **observable action** (UI change, API call, log/metric).

---

## 4) State modeling: the core table

Represent transitions in a **single source of truth** table. Tests can be generated from it.

| From        | Event/Trigger            | Guard/Condition                    | Action/Side-effects                       | To          | Observables (oracle)                          |
|-------------|---------------------------|------------------------------------|-------------------------------------------|-------------|-----------------------------------------------|
| `idle`      | `apply(code)`             | `len 1..16 && not expired`         | mark `pending`, call `/verify`            | `pending`   | log: `event=apply`, `code_len`, trace `verify`|
| `pending`   | `verify.ok`               |                                    | set `applied=true`, update totals         | `applied`   | resp 200; UI badge; total delta               |
| `pending`   | `verify.fail(expired)`    |                                    | keep input; show message                   | `idle`      | message `VALIDATION.code.expired`             |
| `pending`   | `timeout` + `retry()`     | `attempts<N`                        | backoff & retry                           | `pending`   | metric `retry_count` inc; same correlation id |
| `pending`   | `cancel()`                |                                    | abort verify                              | `idle`      | log `cancelled=true`                           |

> Add a **tiebreaker** where ordering matters (e.g., `(created_at, id)` for lists) and include it in your observables.

---

## 5) Draw it (ASCII is fine)

```
 idle
  | apply(code) [valid]
  v
 pending -- verify.ok --> applied
   |  \
   |   \-- verify.fail(expired) --> idle
   |
   \-- timeout -> retry (<= N) -> pending
```

You don’t need UML perfection; you need a model **everyone understands**.

---

## 6) Invariants (make them testable)

- Totals never go **negative**.
- An **applied** code implies a **successful verify** in logs with the same `correlation_id`.
- **Expired** code ⇒ **no total change** and a visible message.
- `retry_count ≤ N`; **same idempotency key** ⇒ **same outcome**.

Turn each invariant into an **assertion** or a **monitor**.

---

## 7) Idempotency, retries, and time

State machines shine when dealing with **time** and **repeats**:

- **Idempotency**: The same request **in the same state** and with the **same key** yields the **same result**.  
- **Retries**: Only retry on **retryable** outcomes (timeouts, 429, 5xx) and **back off** (add jitter).
- **Timeouts**: Decide the state change on timeout (`stay pending` vs `fail fast`).
- **Deadlines**: Permit or block transitions after a cutoff.

Add these as lines in the transition table with an **oracle** (log key/metric).

---

## 8) Concurrency & cancellation

- **Concurrent apply**: Two quick applies should **dedupe** to the latest code or reject the second.
- **Cancel mid-verify**: UI cancel should roll back to `idle` with **no total change**.
- **Race with cart updates**: State transitions should reference a **version** (`cart_version`) to avoid lost updates.

Design a **negative + recovery** path for each concurrency risk.

---

## 9) Worked example — Refund workflow (money-moving)

States: `requested` → `processing` → `succeeded | failed | cancelled`

| From         | Event/Trigger        | Guard                             | Action                                    | To          | Observables                                     |
|--------------|----------------------|-----------------------------------|-------------------------------------------|-------------|-------------------------------------------------|
| requested    | submit(refund, k)    | `valid && amount≤charge && auth`  | create record; enqueue; log `key=k`       | processing  | resp 202; log `status=processing`               |
| processing   | gateway.ok           |                                   | persist txn id                             | succeeded   | event `refund.succeeded`; log `txn_id`          |
| processing   | gateway.fail         | retryable?                        | backoff + retry (max N)                    | processing  | metric `retry_count`                            |
| processing   | timeout              | attempts<N                        | retry with **same key k**                  | processing  | same `idempotency_key=k`                        |
| processing   | cancel()             | not yet terminal                  | mark cancelled                             | cancelled   | event `refund.cancelled`                        |
| processing   | duplicate submit(k)  |                                   | dedupe on `k`                              | processing  | resp 200 w/ existing result (idempotency)       |
| processing   | gateway.final_fail   |                                   | record error                               | failed      | error code; no duplicate payout                 |

**Invariants**
- **Never** two payouts for the same charge+amount.
- Same `idempotency_key` returns **same refund** regardless of retries.
- Terminal states are **absorbing** (no outgoing transitions except view).

**Test ideas (select 8–12)**
1. Submit valid refund → `processing` → `succeeded`.
2. Timeout then retry (same key) → one payout, success.
3. Duplicate submit with same key → returns existing result.
4. Retryable fails N times → backoff observed; still ≤ N attempts.
5. Final fail → no payout; error code surfaced.
6. Cancel mid-processing → `cancelled`; no payout.
7. Idempotent verification after success → returns success again.

Link to: `../40-api-and-data-contracts/idempotency-and-retries.md` and `../70-mini-projects/refund-workflow/*`.

---

## 10) Observability contract (make transitions visible)

For each transition, specify:

- **Log keys**: `event`, `from_state`, `to_state`, `correlation_id`, `idempotency_key`, `error_code`
- **Metrics**: counters for success/fail/retry; histograms for latency
- **Traces**: span names per step, with attributes for keys above
- **Evidence**: where to fetch (log query, dashboard, trace link)

> Put these into your **PRD/acceptance criteria** so tests can assert them.

---

## 11) Anti-patterns (avoid these)

- **Verb-based states** (`verifying`, `verifies`) instead of noun-based (`pending`).
- **Hidden transitions** via side-effects not captured in the model.
- **No guards** on transitions (e.g., allow cancel after terminal).
- **Unobservable** transitions (no logs/metrics/traces to prove them).
- **Ambiguous error handling** (sometimes fail fast, sometimes retry).

---

## 12) Review checklist (quick gate)

- [ ] MAE flows enumerated (at least one exception & one recovery)
- [ ] States are **nouns**, transitions have clear **events/guards/actions**
- [ ] **Invariants** listed and testable
- [ ] **Idempotency/retry** behavior defined where relevant
- [ ] **Cancellation & concurrency** cases included
- [ ] **Observability contract**: logs, metrics, traces
- [ ] Terminal states are **absorbing**
- [ ] Links to cases and checklists are provided

---

## 13) Starter table template

Copy this into your feature folder and fill it.

```md
| From | Event | Guard | Action | To | Observables |
|------|-------|-------|--------|----|-------------|
|      |       |       |        |    |             |
```

**Next:**  
- Build MAE flows in `../30-scenario-patterns/main-alt-exception.md`  
- Add edge inputs with `../20-techniques/boundary-and-equivalence.md`  
- Define contracts in `../40-api-and-data-contracts/*`  
- Gate with `../60-checklists/*`
