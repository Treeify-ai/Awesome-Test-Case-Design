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

# Decision Tables (Playbook)

> Turn **messy business rules** into a **complete, consistent, and testable** set of cases.  
> Decision tables map **conditions** (inputs/facts) to **actions** (outcomes/effects), making gaps and conflicts obvious.

---

## What & Why

- **What**: a tabular model `conditions × actions` that enumerates **rules** (columns) defining what to do for each combination of condition values.
- **Why**: complex policies (pricing, eligibility, routing, access) often hide **missing rules** or **conflicting actions**. A decision table forces **completeness** and **consistency**, and generates a **minimal yet sufficient** set of tests.

**Best for**: combinable discounts, shipping eligibility, KYC/AML checks, feature flags/rollouts, access control (AuthZ), UI state rules, retry policies.

---

## Steps (8-step recipe)

1. **List conditions** as **boolean or enumerations**; define exact meaning of each value.  
2. **Normalize** ambiguous terms (e.g., “VIP” → concrete thresholds; “large order” → `subtotal ≥ 100`).  
3. **List actions** with **observable effects** (accept/reject, message ID, flag set, price adjustment).  
4. **Draft rules** (columns) covering **distinct behavior**; don’t explode into full cartesian product yet.  
5. Add a **priority / resolution strategy** (first-match wins, all-match apply, or weighted).  
6. **Check completeness** (no gaps) and **consistency** (no conflicting actions in one rule).  
7. **Minimize**: merge or eliminate redundant rules; aim for a **small representative set**.  
8. **Generate tests**: 1+ test per rule; add **boundary inputs** for conditions that use ranges.

---

## Notation

- Conditions: `Y` = true, `N` = false, `–` = don’t-care (any), or enumerated values (`EU/US/ROW`).  
- Actions: explicit outcomes (e.g., `ACCEPT`, `REJECT(code=...)`, `APPLY 10%`, `DISABLE button`).  
- Priority: top-to-bottom evaluation order if using **first-match wins**.

---

## Worked Example A — Discount Code Combinability

**Policy (normalized)**  
- Code length: `1..16` and charset `[A-Z0-9-]` (see Boundary playbook).  
- Code **may** be **combinable** (`combinable=Y|N`).  
- **Gift card** promotion cannot be combined with **percentage**-off codes.  
- Code requires **min spend** (e.g., `subtotal ≥ 50`).  
- If multiple conflicts, **show the most actionable** message to the user.

**Conditions**

| ID | Condition (fact)                      | Domain              |
|----|--------------------------------------|---------------------|
| C1 | Input valid (length/charset)         | Y/N                 |
| C2 | Code expired                          | Y/N                 |
| C3 | Combinable                            | Y/N                 |
| C4 | Cart has gift card                    | Y/N                 |
| C5 | Meets min spend (subtotal ≥ 50)       | Y/N                 |
| C6 | Code type                              | `percent`/`fixed`   |

**Actions (observable)**

| A# | Action                                         |
|----|-----------------------------------------------|
| A1 | `ACCEPT` & apply discount                      |
| A2 | `REJECT code=VALIDATION.code.expired`          |
| A3 | `REJECT code=VALIDATION.code.length/charset`   |
| A4 | `REJECT code=CONFLICT.code.not_combinable`     |
| A5 | `REJECT code=VALIDATION.code.min_spend`        |
| A6 | Offer **remove conflicting item** (UI hint)    |

**Rules (first-match wins)**

| Rule | C1 Valid | C2 Expired | C3 Combinable | C4 GiftCard | C5 MinSpend | C6 Type  | Actions                         | Note |
|-----:|:--------:|:----------:|:-------------:|:-----------:|:-----------:|:--------:|----------------------------------|------|
| R1   | N        | –          | –             | –           | –           | –        | A3                               | Invalid input takes precedence |
| R2   | Y        | Y          | –             | –           | –           | –        | A2                               | Expired beats other checks     |
| R3   | Y        | N          | N             | Y           | –           | percent  | A4 + A6                          | Not combinable w/ gift card    |
| R4   | Y        | N          | Y             | –           | N           | –        | A5                               | Min spend not met              |
| R5   | Y        | N          | –             | –           | Y           | –        | A1                               | Accept                         |

**Generated Tests (extract)**

- R1 — `"SAVE 10"` (inner space) → **REJECT** `VALIDATION.code.charset`; input preserved.  
- R2 — `"SAVE10"` expired → **REJECT** `VALIDATION.code.expired`.  
- R3 — `"SAVE10"` non-combinable percent + gift card → **REJECT** `CONFLICT.code.not_combinable`; UI offers removal.  
- R4 — `"SAVE10"` combinable but subtotal `< 50` → **REJECT** `VALIDATION.code.min_spend`.  
- R5 — Happy path (any type) with min spend met → **ACCEPT**; badge, total update.

> Pair this with **Boundary & Equivalence** for code format and **Error Taxonomy → UX** for message IDs:  
> `../20-techniques/boundary-and-equivalence.md`, `../40-api-and-data-contracts/error-taxonomy.md`.

---

## Worked Example B — Shipping Method Eligibility

**Policy (normalized)**  
- **Free shipping** for `subtotal ≥ 100` (domestic only).  
- **No shipping** to embargoed regions.  
- **Heavy items** (`weight > 20kg`) require **Freight** (no standard/free).  
- **Weekend dispatch** disabled for **Cold Chain** items.

**Conditions**

| ID | Condition                         | Domain      |
|----|-----------------------------------|-------------|
| C1 | Region allowed                    | Y/N         |
| C2 | Domestic                          | Y/N         |
| C3 | Subtotal ≥ 100                    | Y/N         |
| C4 | Weight > 20kg                     | Y/N         |
| C5 | Cold Chain                        | Y/N         |
| C6 | Today is weekend                  | Y/N         |

**Actions**

| A# | Action                                   |
|----|------------------------------------------|
| A1 | Offer **Free** shipping                  |
| A2 | Offer **Standard** shipping              |
| A3 | Offer **Freight** shipping only          |
| A4 | **Reject all** (no shipping)             |
| A5 | Hide **Weekend Dispatch** option         |

**Rules (priority: safety > offers)**

| Rule | C1 Region | C2 Domestic | C3 Subtot≥100 | C4 Heavy | C5 Cold | C6 Weekend | Actions      | Note                             |
|-----:|:---------:|:-----------:|:-------------:|:--------:|:-------:|:----------:|--------------|----------------------------------|
| R1   | N         | –           | –             | –        | –       | –          | A4           | Embargo → no shipping            |
| R2   | Y         | –           | –             | Y        | –       | –          | A3           | Heavy → Freight only             |
| R3   | Y         | Y           | Y             | N        | –       | –          | A1 + A2      | Domestic free + standard         |
| R4   | Y         | –           | –             | –        | Y       | Y          | A2 + A5      | Cold + weekend → hide weekend    |
| R5   | Y         | –           | –             | N        | –       | –          | A2           | Default standard                  |

**Tests (extract)**
- R1 — Region embargoed → **Reject all** (A4) with message code `POLICY.region.embargoed`.
- R2 — Heavy item → **Freight only**.  
- R3 — Domestic + ≥100 → **Free + Standard** visible.  
- R4 — Cold-chain on weekend → **Standard only**, weekend option hidden.  
- R5 — Else → **Standard** available.

---

## Completeness & Consistency Checks

- **Completeness**: every reachable combination of condition values hits **at least one** rule (no gaps).  
- **Consistency**: no rule column assigns **conflicting actions** (e.g., `ACCEPT` & `REJECT`).  
- **Priority**: for **first-match wins**, order the rules so that **safety** and **validation** rules come first.  
- **Default rule**: include an explicit **fallback** (e.g., “Else → Standard shipping”) to avoid silent gaps.  
- **MECE**: make condition domains **mutually exclusive** by adding tie-breakers (e.g., `created_at, id` in sorting rules).

---

## From Table → Minimal Tests

Aim for **1 test per rule** + **boundaries** on numeric/date conditions.

**MC/DC-inspired selection** (practical version)
- For each **condition**, include a pair of tests that **change only that condition** and **flip the action** where possible.  
- This yields a small set that still **demonstrates independence** of conditions.

**Example (Shipping)**
- Start with R3 (Free+Standard).  
- Flip `C3` (subtotal < 100) keeping others same → expect Standard only (R5).  
- Flip `C4` to Heavy → Freight (R2).  
- Flip `C1` to embargoed → Reject (R1).  
- Flip `C5 & C6` → hide weekend (R4).

---

## Oracles & Evidence

- **Functional**: API/DB mutations (shipping option list; discount applied), response codes, feature flags.
- **UX**: presence/absence of options; message IDs; button disabled states.
- **API Contract**: error taxonomy codes, schema invariants for returned options.
- **Non-functional**: p95 latency for eligibility computation; consistent pagination if options are listed.
- **Evidence**: structured logs (`rule_id`, `conditions` summary), metrics per rule hit, traces per evaluation.

> Add a log like `{rule_id:"R3", conditions:{domestic:true, subtotal_ge_100:true}}` to make tests and prod issues diagnosable.

---

## Anti-patterns

- Implicit **priority** (ambiguous evaluation order).  
- **Overlapping** rules that both fire with conflicting actions.  
- **Wordy conditions** that hide real checks (“VIP user” vs “total_spend ≥ 1000 in 90d”).  
- No **default** rule.  
- Actions not **observable** (can’t assert outcome).  
- Ignoring **boundary** values for range-based conditions.

---

## Review Checklist (quick gate)

- [ ] Conditions are **normalized**; domains defined; booleans or enums  
- [ ] Actions are **observable** (API/DB/UX)  
- [ ] Rules have explicit **priority** and **no conflicts**  
- [ ] **Completeness**: all meaningful combos covered (with default rule)  
- [ ] **Boundaries** added for numeric/date conditions  
- [ ] Tests: **1+ per rule**; MC/DC-style flips for key conditions  
- [ ] Evidence: logs with `rule_id` and condition snapshot; metrics per rule

---

## CSV Seeds

**Discount combinability**

```csv
rule,input_valid,expired,combinable,gift_card,min_spend,code_type,expected,oracle
R1,N,-,-,-,-,-,REJECT VALIDATION.code.charset,resp400+message_id
R2,Y,Y,-,-,-,-,REJECT VALIDATION.code.expired,resp400+message_id
R3,Y,N,N,Y,-,percent,REJECT CONFLICT.code.not_combinable+offer_remove,resp409+ui_hint
R4,Y,N,Y,-,N,-,REJECT VALIDATION.code.min_spend,resp400+message_id
R5,Y,N,-,-,Y,-,ACCEPT,resp200+badge+total_change
```

**Shipping eligibility**

```csv
rule,region_allowed,domestic,subtotal_ge_100,heavy,cold_chain,weekend,expected
R1,N,-,-,-,-,-,REJECT_ALL
R2,Y,-,-,Y,-,-,FREIGHT_ONLY
R3,Y,Y,Y,N,-,-,FREE_AND_STANDARD
R4,Y,-,-,N,Y,Y,STANDARD_HIDE_WEEKEND
R5,Y,-,-,N,-,-,STANDARD
```

---

## Contributor Template (copy/paste)

```
# Decision Table — <Topic>

## Conditions
- C1: <name> — <domain>
- C2: <name> — <domain>
...

## Actions
- A1: <observable>
- A2: ...

## Rules (priority: first-match wins | all-apply)
| Rule | C1 | C2 | ... | Actions | Note |
|-----:|:--:|:--:|-----|---------|------|
| R1   |    |    |     |         |      |
...

## Tests
- R1 — <input> → <expected> (oracle: <evidence>)
...

## Completeness & Consistency
- Gaps: <...>
- Conflicts: <...>
- Default rule: <...>

## Links
- Boundary & Equivalence: `../20-techniques/boundary-and-equivalence.md`
- Error taxonomy: `../40-api-and-data-contracts/error-taxonomy.md`
```

**See also**  
- Boundary & Equivalence: `./boundary-and-equivalence.md`  
- State Models (for lifecycles): `./state-models.md`  
- Error taxonomy → UX: `../40-api-and-data-contracts/error-taxonomy.md`  
- Checklists: `../60-checklists/functional-coverage.md`, `../60-checklists/api-coverage.md`

---

<p align="center">
  <sub>
    Built with <a href="https://treeifyai.com">Treeify</a> — AI test case design copilot.<br>
    <em>Structured, traceable, high-coverage test design — faster.</em>
  </sub>
</p>
