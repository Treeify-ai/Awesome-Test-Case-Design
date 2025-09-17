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


# State Models (Playbook)

> When flows get tricky (retries, cancellations, async jobs, money movement), **state models** keep you honest.  
> Model **states** (nouns), **events** (triggers), **guards** (conditions), and **actions** (effects) to design **precise, non-flaky** tests.

---

## What & Why

- **What**: A state machine describes where the system **can be**, how it **moves**, and what must be **observable** at each step.  
- **Why**: Reduces ambiguity, catches concurrency bugs, and provides **clear oracles** (logs/metrics/traces) so tests fail for the right reasons.

**Use for**: payments/refunds, idempotent POSTs, background processing, multi-step wizards, token refresh, fulfillment pipelines.

---

## Steps (10-step recipe)

1. **Scope boundary & actors** — SUT vs. dependencies; User/System/Scheduler/3rd-party.  
2. **List states (nouns)** — include **initial** and **terminal**; avoid verb-ish names.  
3. **Events** — user actions, webhooks, timeouts, cron ticks, retries, cancels.  
4. **Guards** — conditions required for the transition (`attempts < N`, `authz=Y`, `balance ≥ amount`).  
5. **Actions / side-effects** — DB writes, emitted events, API calls, UI updates.  
6. **Observables (oracles)** — log keys, metrics, response codes, trace spans.  
7. **Invariants** — truths that must always hold (no double payout; totals never negative).  
8. **Time** — deadlines, timeouts, backoff + jitter, idempotency windows.  
9. **Concurrency** — dedupe same key, version counters, cancellation rules.  
10. **Table + diagram** — single source of truth; generate tests from it.

Pair with:  
- MAE flows → `../30-scenario-patterns/main-alt-exception.md`  
- Idempotency & retries → `../40-api-and-data-contracts/idempotency-and-retries.md`

---

## Notation (used in tables)

| Column | Meaning |
|---|---|
| **From** | Current state |
| **Event/Trigger** | What happens to attempt a move |
| **Guard** | Condition that must be true |
| **Action** | Side-effects (DB/event/API) |
| **To** | Next state |
| **Observables** | Evidence your test should assert |

**Terminal** states are **absorbing** (no outgoing transitions).

---

## Worked Example A — Email Verification

**States**: `unverified → code_sent → verified | expired`  
**Events**: `request_code`, `submit_code`, `timeout`, `resend`  
**Invariants**: A code can verify **at most once**; after expiry, a new code is required.

| From        | Event           | Guard                     | Action                                   | To         | Observables                               |
|-------------|------------------|---------------------------|------------------------------------------|------------|-------------------------------------------|
| unverified  | request_code     | rate_limit ok             | generate code; send email; log `code_id` | code_sent  | event `email.code.sent`; metric increment |
| code_sent   | submit_code      | `code==latest && !expired`| mark verified; consume code              | verified   | resp 200; log `verified=true`             |
| code_sent   | submit_code      | `expired`                 | reject                                   | code_sent  | resp 400 `VALIDATION.code.expired`        |
| code_sent   | timeout          | `now>expiry`              | mark expired                             | expired    | log `expired=true`                        |
| code_sent   | resend           | rate_limit ok             | generate new code; invalidate old        | code_sent  | event `email.code.resent`; `code_id` new  |

**Tests (pick ~7–9)**
- Request code → submit valid within TTL → **verified**.  
- Submit **expired** code → **error** `VALIDATION.code.expired`.  
- Resend then submit **old** code → **reject**; new code works.  
- TTL passes (timeout) → **expired**, then new request works.

**Oracles**
- Logs with `code_id` and `verified=true`  
- Response message IDs  
- Metric: `email.code.sent` count

---

## Worked Example B — Order Fulfillment (with cancel & retry)

**States**: `created → picking → packed → shipped | cancelled | failed`  
**Events**: `start_pick`, `pack_ok`, `ship_ok`, `cancel`, `dep_fail`, `timeout`, `retry`  
**Invariants**:  
- Only **one** terminal state per order.  
- `cancel` after `shipped` is **not allowed** (policy).  
- Retries (**max N**) only on `dep_fail` or `timeout`.  
- Stock must decrement **exactly once**.

| From     | Event       | Guard                        | Action                                   | To        | Observables                                 |
|----------|-------------|------------------------------|------------------------------------------|-----------|---------------------------------------------|
| created  | start_pick  | stock ≥ qty                  | reserve stock                             | picking   | log `stock_reserved=true`                    |
| picking  | pack_ok     |                              | pack                                      | packed    | event `order.packed`                         |
| packed   | ship_ok     | carrier available            | ship; decrement stock                     | shipped   | event `order.shipped`; metric `ship_count`   |
| picking  | cancel      | not terminal                 | release reservation                       | cancelled | event `order.cancelled`; stock restored      |
| packed   | cancel      | policy: allow? (Y/N)         | if Y: restock; else: reject               | cancelled | or error `POLICY.cancel.not_allowed`         |
| picking  | dep_fail    | attempts < N                 | backoff + retry                           | picking   | metric `retry_count`                         |
| picking  | timeout     | attempts < N                 | retry                                     | picking   | same `correlation_id`; `retry_count++`       |
| picking  | dep_fail    | attempts ≥ N                 | mark failed                               | failed    | error code; no ship event                    |

**Concurrency**
- Duplicate `start_pick` → **dedupe** by order id (no double reservation).  
- Multiple `ship_ok` webhooks → **idempotent**: only first decrements stock; next are ignored.

**Tests (pick 10–12)**
1. Happy path `created → picking → packed → shipped`.  
2. Cancel during `picking` → `cancelled`, stock restored.  
3. Cancel after `shipped` → policy error.  
4. Retry on `dep_fail` then success; attempts ≤ N.  
5. Exceed retries → `failed`.  
6. Duplicate `ship_ok` → still one decrement (idempotent).  
7. Timeout + retry preserves same correlation id.

**Oracles**
- Stock delta exactly **once**  
- Events emitted (`order.*`)  
- Metrics for retries  
- Logs for `from_state`, `to_state`, `correlation_id`

---

## ASCII diagrams (good enough)

```
unverified --request_code--> code_sent --submit(ok)--> verified
        \--(rate_limit)X                                 ^
           \--resend-------------------------------------|
           \--timeout--> expired ------------------------/
```

```
created -> picking -> packed -> shipped
    |        |   \--cancel--> cancelled
    |        |--dep_fail/timeout (retry<=N)--> picking
    |                           \--(>N)--> failed
    \--invalid start_pick (no stock) X
```

---

## Invariants you should assert

- **Idempotency**: same webhook/request key does **not** duplicate effects.  
- **Single terminal**: once terminal, **no further transitions**.  
- **Conservation**: totals/stock counters **balance** (no negative inventory, no double payouts).  
- **Retry limits**: attempts never exceed **N**; backoff grows (± jitter).  
- **Traceability**: each change has a `correlation_id` and `from→to` record.

---

## Observability contract (put this in acceptance criteria)

- **Logs**: `event`, `from_state`, `to_state`, `attempt`, `correlation_id`, `idempotency_key`, `error_code`.  
- **Metrics**: counters for `*_succeeded`, `*_failed`, `retry_count`; histograms for latencies.  
- **Traces**: named spans per transition (`verify`, `pack`, `ship`) with attributes above.

---

## Anti-patterns

- **Verb-based states** (`verifying`) rather than nouns (`pending`).  
- **Hidden transitions** via side-effects not in the model.  
- **Ambiguous guards** (sometimes allow cancel, sometimes not).  
- **Unobservable** transitions (can’t assert outcomes).  
- **Over-modeling**: too many micro-states that don’t change behavior.

---

## Review checklist (quick gate)

- [ ] States are **nouns** with clear **initial/terminal**  
- [ ] Transitions specify **event + guard + action**  
- [ ] **Invariants** listed and testable  
- [ ] **Retries/backoff/idempotency** defined where relevant  
- [ ] **Cancellation & concurrency** covered  
- [ ] **Oracles** (logs/metrics/traces) explicit for each transition  
- [ ] Terminal states are **absorbing**  
- [ ] Diagram + table present; tests selected (8–15)

---

## CSV seeds

**Email verification transitions**

```csv
from,event,guard,action,to,observables
unverified,request_code,rate_limit_ok,gen+send+log_code,code_sent,email.code.sent
code_sent,submit_code,latest&&!expired,mark_verified,verified,resp200+log_verified
code_sent,submit_code,expired,reject_same_state,code_sent,resp400 VALIDATION.code.expired
code_sent,timeout,now>expiry,mark_expired,expired,log expired
code_sent,resend,rate_limit_ok,gen_new_invalidate_old,code_sent,email.code.resent
```

**Order fulfillment transitions**

```csv
from,event,guard,action,to,observables
created,start_pick,stock>=qty,reserve_stock,picking,log stock_reserved
picking,pack_ok,,pack,packed,order.packed
packed,ship_ok,carrier_ok,ship+decrement,shipped,order.shipped+metric ship_count
picking,cancel,not_terminal,release_reservation,cancelled,order.cancelled
picking,dep_fail,attempts<N,retry,picking,metric retry_count++
picking,timeout,attempts<N,retry,picking,trace correlation_id
picking,dep_fail,attempts>=N,mark_failed,failed,error code
```

---

## Contributor template

```
# State Model — <Topic>

## States
Initial: <state>  
Terminal: <state1>, <state2>

## Transitions
| From | Event | Guard | Action | To | Observables |
|------|-------|-------|--------|----|-------------|
|      |       |       |        |    |             |

## Invariants
- <...>

## Tests (8–15)
- <...>

## Observability Contract
Logs: <keys>  
Metrics: <list>  
Traces: <spans>

## Links
- MAE flows: `../30-scenario-patterns/main-alt-exception.md`
- Idempotency & retries: `../40-api-and-data-contracts/idempotency-and-retries.md`
- Checklists: `../60-checklists/*`
```

---

<p align="center">
  <sub>
    Built with <a href="https://treeifyai.com">Treeify</a> — AI test case design copilot.<br>
    <em>Structured, traceable, high-coverage test design — faster.</em>
  </sub>
</p>
