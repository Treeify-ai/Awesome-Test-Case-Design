# Boundary & Equivalence (Playbook)

> Design a **small, representative** set of inputs that catches the majority of input-related failures with minimal cases.  
> You’ll partition inputs into **equivalence classes** and probe **boundaries** with “just in / just out” values—plus key cross-field constraints.

---

## What & Why

- **Equivalence classes:** groups of inputs that should produce the **same behavior** (choose one representative each).
- **Boundary values:** values **at, just below, just above** each limit, where bugs love to hide (off-by-one, overflow, encoding).
- **Outcome:** 8–12 high-yield cases with **explicit expected results**, oracles, and evidence.

When this shines: numeric ranges, string lengths/formats, dates/times, pagination limits, counts/quotas, file sizes, and any **“must be between / must match pattern”** requirement.

---

## Steps (7-step recipe)

1. **Extract constraints** from PRD/contracts: ranges, inclusivity, length, charset, regex, nullability, trimming, case sensitivity, rounding, unit.  
2. **Partition classes** (valid vs invalid) so they’re **MECE** (no overlaps/gaps).  
3. **Pick boundary set** for each dimension: *min−1, min, min+1, mid, max−1, max, max+1*.  
4. **Add “format/encoding” edges**: leading/trailing spaces, unicode/emoji, normalization (NFC/NFD), mixed case.  
5. **Cross-field constraints:** relate fields (e.g., `discount ≤ subtotal`, `start ≤ end`).  
6. **Oracles & evidence:** what proves the result (response code/body, DB delta, message ID, log keys, metrics).  
7. **Shrink to fit:** remove mid-domain duplicates; keep distinct-behavior reps only.

---

## Worked Example A — Amount field (0..10,000, integer)

**Constraint summary**  
- Domain: integers `0..10,000` (inclusive)  
- Rounding: **reject** non-integers  
- Unit: **cents** (displayed as major currency)  
- Null/empty: treat empty as **validation error**  

### Classes & Boundaries

| Class                         | Representative(s)         | Why it matters |
|------------------------------|---------------------------|----------------|
| Below min                    | `-1`                      | off-by-one underflow |
| Min                          | `0`                       | inclusive edge |
| Just above min               | `1`                       | edge correctness |
| Mid-domain                   | `1234`                    | sanity |
| Just below max               | `9,999`                   | edge correctness |
| Max                          | `10,000`                  | inclusive edge |
| Above max                    | `10,001`                  | overflow |
| Non-integer                  | `12.34`, `"100.0"`        | reject decimals/strings |
| Empty/null/whitespace        | `""`, `null`, `"  "`      | validation |
| Non-numeric                  | `"ABC"`                   | validation |

### Sample cases (extract)

1. **Accept 0** → store `0`, show “$0.00”; message none.  
   *Oracle:* resp 200 + DB amount `0`; UI shows `$0.00`; log `{amount:0}`.
2. **Reject −1** → error `VALIDATION.amount.min`; no DB write.  
   *Oracle:* resp 400 + message ID; no state change.
3. **Accept 10,000** → store 1000000 cents; display `$10,000.00`.  
4. **Reject 10,001** → `VALIDATION.amount.max`; no change.  
5. **Reject 12.34** → `VALIDATION.amount.type.integer`.  
6. **Reject empty/whitespace** → `VALIDATION.amount.required`.

> Add cross-field edges in [Worked Example C](#worked-example-c—cross-field-constraints).

---

## Worked Example B — Discount code (len 1..16; charset `[A–Z0–9-]`; case-insensitive; trims)

**Constraint summary**  
- Length: `1..16` inclusive  
- Allowed: uppercase letters, digits, hyphen (`[A-Z0-9-]`)  
- Case-insensitive match; **store uppercase**  
- Trim leading/trailing spaces; **reject** inner whitespace  
- Normalize to **NFC** (avoid mixed normalization)  

### Classes & Boundaries

| Dimension  | Class / Edge                                     | Representative(s)                  |
|-----------:|---------------------------------------------------|------------------------------------|
| Length     | below min                                         | `""` (empty)                       |
|            | min                                               | `"A"`                              |
|            | typical                                           | `"SAVE10"`                         |
|            | max                                               | `"A"*16`                           |
|            | above max                                         | `"A"*17`                           |
| Charset    | allowed                                           | `"A-Z0-9-"` mix (`"A1-B2"`)        |
|            | forbidden char                                    | `"SAVE!0"` (`!`)                   |
|            | emoji / non-latin                                 | `"SAVE🎉"` / `"减10"`               |
| Spacing    | leading/trailing (trim)                           | `" SAVE10 "`                       |
|            | inner space (reject)                              | `"SAVE 10"`                        |
| Case       | lowercase normalization                           | `"save10"` → accepts, stores `"SAVE10"` |
| Unicode    | normalization (NFC vs NFD)                        | `é` vs `é`                   |

### Sample cases (extract)

1. **Accept min (len=1)**: `"A"` → normalized to `"A"`.  
   *Oracle:* 200 + stored uppercase; UI badge; log `code_len=1`.
2. **Accept with trim**: `" SAVE10 "` → accepts `"SAVE10"`.  
3. **Reject inner space**: `"SAVE 10"` → `VALIDATION.code.charset`.  
4. **Reject above max (17)**: `"A"*17` → `VALIDATION.code.length.exceeds`.  
5. **Reject forbidden char**: `"SAVE!0"` → `VALIDATION.code.charset`.  
6. **Unicode NFC/NFD parity**: code matches identically after normalization; **either both accept or both reject** by policy.

> Pair with **error taxonomy → UX** mapping in `../40-api-and-data-contracts/error-taxonomy.md`.

---

## Worked Example C — Cross-field constraints

Many defects are **between fields**. Treat cross-field rules as **boundaries** too.

**Discount ≤ Subtotal**  
- Classes: `{discount == 0}`, `{0 < discount < subtotal}`, `{discount == subtotal}`, `{discount > subtotal}`  
- Boundaries per `subtotal=100`:
  - `discount = -1, 0, 1, 99, 100, 101`  
- Tests:
  1. **Equal to subtotal** → `CONFLICT.discount.exceeds_subtotal`; total stays 100; message suggests lowering discount.
  2. **Just below** (`99`) → accept; total `1`.
  3. **Just above** (`101`) → reject; no total change.

**Dates (start ≤ end)**  
- `start = end`, `start = end ± 1s`, `start > end`.  
- Timezones & DST edges if applicable.

---

## Generating a boundary set (quick helper)

You can sketch a tiny generator (pseudo):

```python
def boundaries(min_v, max_v, step=1):
    return [min_v- step, min_v, min_v+ step, (min_v+max_v)//2, max_v- step, max_v, max_v+ step]

# Example: 0..10000 ints → [-1, 0, 1, 5000, 9999, 10000, 10001]
```

> Don’t ship the generator as tests; **use it to propose** an initial set, then prune to cases that **change behavior**.

---

## Oracles & Evidence (make pass/fail obvious)

- **Functional:** DB delta (amount stored), event emitted, response schema & code.  
- **UX:** message **IDs** (not free text), visible state (badge, disabled button).  
- **API:** error taxonomy codes; idempotency (same key ⇒ same result).  
- **Non-functional:** p95 latency on verify path; retry count with backoff.  
- **Evidence:** structured logs (include `correlation_id`, `error_code`, `code_len`), metrics, traces, screenshots.

---

## Anti-patterns

- Using many **mid-domain** values; low yield.  
- Missing **±1** around min/max.  
- Treating “inclusive” edges as “exclusive” (or vice versa).  
- Ignoring **unit/rounding** rules (cents vs dollars).  
- Dropping **cross-field** constraints (discount vs subtotal, start vs end).  
- No **oracles** (asserting only internal state).  
- Forgetting **normalization/trim** for strings.

---

## Review checklist (quick gate)

- [ ] Constraints extracted (range, inclusivity, length, charset, trim, case, normalization, unit/rounding)  
- [ ] Valid/invalid **equivalence classes** are **MECE**  
- [ ] Boundaries include **min−1, min, min+1, max−1, max, max+1**  
- [ ] **Format/encoding** edges covered (spaces, unicode, NFC/NFD)  
- [ ] **Cross-field** constraints have boundary cases  
- [ ] **Expected results** and **oracles** are explicit  
- [ ] Evidence: logs/metrics/traces/snapshots referenced  
- [ ] Redundant mid-domain cases removed

---

## Tiny CSV seeds

**Amount 0..10,000**

```csv
id,input,klass,expected,oracle
A-001,-1,below_min,400 VALIDATION.amount.min,resp400+no_db_write
A-002,0,min,200 OK,resp200+db=0
A-003,1,just_above_min,200 OK,resp200+db=1
A-004,9999,just_below_max,200 OK,resp200+db=9999
A-005,10000,max,200 OK,resp200+db=10000
A-006,10001,above_max,400 VALIDATION.amount.max,resp400+no_db_write
A-007,12.34,non_integer,400 VALIDATION.amount.type.integer,resp400
A-008,"",empty,400 VALIDATION.amount.required,resp400
```

**Discount code 1..16 `[A-Z0-9-]`**

```csv
id,input,klass,expected,oracle
C-001,"A",min,accept,resp200+store=uppercase
C-002," SAVE10 ",trim,accept,resp200+store=SAVE10
C-003,"SAVE 10",inner_space,reject,400 VALIDATION.code.charset
C-004,"A"*17,above_max,reject,400 VALIDATION.code.length.exceeds
C-005,"SAVE!0",forbidden,reject,400 VALIDATION.code.charset
C-006,"save10",lowercase_norm,accept,resp200+store=SAVE10
```

---

## See also

- Scenarios (MAE): `../30-scenario-patterns/main-alt-exception.md`  
- Error taxonomy → UX: `../40-api-and-data-contracts/error-taxonomy.md`  
- Idempotency & retries: `../40-api-and-data-contracts/idempotency-and-retries.md`  
- Checklists: `../60-checklists/functional-coverage.md`, `../60-checklists/api-coverage.md`  
- Mini-project: `../70-mini-projects/checkout-discount-code/*`
