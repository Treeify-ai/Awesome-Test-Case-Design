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


# Pairwise / Combinatorics (Playbook)

> When factors multiply (device × browser × role × payment × locale …), pairwise testing gives you **broad interaction coverage** with **minimal cases**.  
> Cover **every pair of factor values at least once** (t=2). Use higher strength (**t=3**) when pairs aren’t enough.

---

## What & Why

- **Factors** are independent dimensions (e.g., *Device*, *Browser*, *Role*). **Levels** are their values (e.g., *iOS*, *Android*).  
- **Pairwise** (t=2) builds a small set where **every pair** of levels across different factors appears **at least once**.  
- **Benefits**: huge reduction vs full cartesian product while catching many interaction bugs (mis-wired flags, default collisions, missing feature guards).  
- **Not a silver bullet**: use **State Models** for order/sequences and **Boundary & Equivalence** for numeric/format constraints.

Use when: UI matrices, eligibility/feature flags, API parameter combos, configuration surfaces.  
Avoid as the **only** technique when: stateful flows, temporal ordering, concurrency, or numerical edge correctness dominates.

---

## Key Concepts

- **Strength (t)**: pairwise is t=2. For tricky domains, move to **t=3** (all triples).  
- **Constraints**: invalid combos you must **exclude** (e.g., *Safari* on *Android*).  
- **Seeds/Must-Haves**: specific cases you **require** (e.g., “Member × Refund × zh-CN”).  
- **Weights/Priorities**: some pairs matter more (e.g., *Payment × Device*).

---

## Steps (8-step recipe)

1. **List factors** and **levels**. Normalize ambiguous values (e.g., “Mobile” → **iOS**, **Android**).  
2. Add **constraints** (invalid pairs/tuples) as precise rules.  
3. Choose **strength**: default **t=2**, bump to **t=3** for safety-critical or known multi-way bugs.  
4. Add **seed cases** you must include.  
5. **Generate** a minimal set (tool-agnostic; greedy works fine for small sets).  
6. Add **negative/invalid** tests separately (don’t mix them into the generator).  
7. Layer **boundary/format** edges within chosen cases (e.g., pick *max length* input for one row).  
8. Produce a **coverage matrix** (pairs → case id) for review.

---

## Worked Example A — Checkout Matrix (UI)

**Factors & levels**

| Factor  | Levels                          |
|---------|----------------------------------|
| Device  | iOS, Android, Desktop           |
| Browser | Safari, Chrome, Firefox         |
| Role    | Guest, Member                   |
| Payment | Card, PayPal                    |
| Locale  | en-US, fr-FR                    |

**Constraints**

- `NOT (Device=Android AND Browser=Safari)` (no Safari on Android)

**Generated pairwise set (sample, 12 cases)**

> One of many minimal solutions; any that covers all pairs + respects constraints is fine.

| Case | Device  | Browser | Role   | Payment | Locale |
|-----:|---------|---------|--------|---------|--------|
| C01  | iOS     | Safari  | Guest  | Card    | en-US  |
| C02  | iOS     | Chrome  | Member | PayPal  | fr-FR  |
| C03  | iOS     | Firefox | Guest  | PayPal  | fr-FR  |
| C04  | Android | Chrome  | Guest  | Card    | fr-FR  |
| C05  | Android | Firefox | Member | Card    | en-US  |
| C06  | Desktop | Safari  | Member | Card    | fr-FR  |
| C07  | Desktop | Chrome  | Guest  | PayPal  | en-US  |
| C08  | Desktop | Firefox | Member | PayPal  | fr-FR  |
| C09  | iOS     | Safari  | Member | PayPal  | en-US  |
| C10  | Android | Chrome  | Member | PayPal  | en-US  |
| C11  | Desktop | Chrome  | Member | Card    | fr-FR  |
| C12  | iOS     | Firefox | Member | Card    | en-US  |

**Notes**
- Every **pair** (e.g., *Android × Member*, *PayPal × fr-FR*, *Safari × Desktop*) appears somewhere.  
- We included both **Guest** and **Member** with all payments/locales across devices/browsers at least once.  
- Add **boundary content** inside chosen rows when needed (e.g., in C09, apply a **max-length** discount code from Boundary playbook).

**Prioritize pairs**
- Weight *(Payment × Device)* and *(Browser × Payment)* if wallet integrations are finicky. Ensure those pairs appear **multiple times** or in **seed cases** with richer assertions.

---

## Worked Example B — API Query Parameters

**Factors & levels**

| Factor     | Levels                      |
|------------|-----------------------------|
| sort_by    | created_at, price           |
| order      | asc, desc                   |
| page_size  | 10, 50                      |
| filter     | none, in_stock              |
| include    | none, relations             |

**Constraints**
- When `include=relations`, `page_size` must be ≤ 50 (OK here).
- None other; all combos valid.

**Pairwise set (8 cases)**

| Case | sort_by    | order | page_size | filter    | include    |
|-----:|------------|-------|-----------|-----------|------------|
| A01  | created_at | asc   | 10        | none      | none       |
| A02  | created_at | desc  | 50        | in_stock  | relations  |
| A03  | price      | asc   | 50        | none      | relations  |
| A04  | price      | desc  | 10        | in_stock  | none       |
| A05  | created_at | asc   | 50        | in_stock  | none       |
| A06  | price      | desc  | 50        | none      | none       |
| A07  | created_at | desc  | 10        | none      | relations  |
| A08  | price      | asc   | 10        | in_stock  | none       |

**Add invariants & oracles**
- **Stable sort**: if equal `created_at`, tie-break with `id` → no duplicate/skip across pages.  
- **Schema**: response matches contract; `include=relations` enriches fields.  
- **Perf**: `page_size=50` must meet p95 budget.  
- **Evidence**: logs include `params`, trace spans include `sort_by`, `order`, `page_size`.

---

## When to use **t=3** (triples)

- Complex feature flags (A × B × C gating).  
- Layout/compat issues where **three-way** interactions break (e.g., *Device × Browser × Locale*).  
- Serializations where **two** parameters are fine, but **three** trigger size/latency limits.

Tip: apply **t=3** to a smaller subset of factors (e.g., just *Device × Browser × Payment*), not the whole set.

---

## Oracles & Evidence

- **Functional**: visible option set, applied feature flag, API response fields.  
- **UX**: presence/absence of controls; message IDs; enabled/disabled states.  
- **API Contract**: schema, pagination invariants, error taxonomy on invalid mixes.  
- **Non-functional**: p95 latency on representative rows; payload sizes within budget.  
- **Evidence**: structured log with `{case_id, factors...}` (or derived hash), metrics per failure type, trace attributes for each factor.

---

## Anti-patterns

- Using pairwise for **numeric boundaries** → use Boundary & Equivalence first.  
- Ignoring **constraints** → generator produces impossible combos.  
- Assuming pairwise catches **sequence/order** bugs → use State Models.  
- Exploding levels (10× levels per factor) → over-large set; **merge or re-bucket**.  
- Mixing **invalid** cases into the generator → generate **valid** set; test invalids separately.

---

## Review Checklist (quick gate)

- [ ] Factors and **normalized levels** listed  
- [ ] **Constraints** explicit and enforced  
- [ ] Strength chosen (**t=2** default; **t=3** where justified)  
- [ ] **Seeds** added for must-have risky paths  
- [ ] Generated set is **small** yet covers **all pairs**  
- [ ] Boundary/format edges **layered into** selected rows  
- [ ] Oracles & evidence **explicit**; logs include factor tags  
- [ ] Coverage matrix available (pairs → case id)

---

## CSV seeds

**UI matrix**

```csv
id,device,browser,role,payment,locale,notes
C01,iOS,Safari,Guest,Card,en-US,"seed: baseline"
C02,iOS,Chrome,Member,PayPal,fr-FR,"wallet path"
C03,iOS,Firefox,Guest,PayPal,fr-FR,"alt browser on iOS"
C04,Android,Chrome,Guest,Card,fr-FR,"android baseline"
C05,Android,Firefox,Member,Card,en-US,"role swap"
C06,Desktop,Safari,Member,Card,fr-FR,"desktop safari"
C07,Desktop,Chrome,Guest,PayPal,en-US,"wallet on desktop"
C08,Desktop,Firefox,Member,PayPal,fr-FR,"fallback browser"
C09,iOS,Safari,Member,PayPal,en-US,"max-length code"
C10,Android,Chrome,Member,PayPal,en-US,"risk pair weight"
C11,Desktop,Chrome,Member,Card,fr-FR,"seed: regression"
C12,iOS,Firefox,Member,Card,en-US,"rounding case"
```

**API params**

```csv
id,sort_by,order,page_size,filter,include,notes
A01,created_at,asc,10,none,none,"baseline"
A02,created_at,desc,50,in_stock,relations,"heavy payload"
A03,price,asc,50,none,relations,"perf guard"
A04,price,desc,10,in_stock,none,"filter"
A05,created_at,asc,50,in_stock,none,"page size edge"
A06,price,desc,50,none,none,"descending perf"
A07,created_at,desc,10,none,relations,"relations small page"
A08,price,asc,10,in_stock,none,"alt"
```

---

## Minimal greedy generator (reference only)

> You don’t need a perfect OA implementation. A simple greedy builder works for small sets.

```python
def pairs(factors):
    # factors: dict[str, list[str]]
    keys = list(factors.keys())
    req = set()
    for i in range(len(keys)):
        for j in range(i+1, len(keys)):
            a, b = keys[i], keys[j]
            for va in factors[a]:
                for vb in factors[b]:
                    req.add(((a, va), (b, vb)))
    return req

def covers(case, pair):
    return pair[0] in case and pair[1] in case

def greedy_pairwise(factors, constraints=lambda c: True, seeds=None):
    remaining = pairs(factors)
    cases = set()
    if seeds:
        for s in seeds:
            if constraints(s):
                cases.add(tuple(sorted(s.items())))
                remaining = {p for p in remaining if not covers(set(s.items()), p)}
    # naive greedy search
    while remaining:
        best = None; best_cov = set()
        for va in factors[list(factors.keys())[0]]:
            pass  # sketch only; use real tools in practice
        # In practice, use an existing tool or small script to pick the next case that covers most remaining pairs.
        break
    return cases
```

Use an existing library or tool if available; validate results with a **coverage matrix**.

---

## Links

- Boundary & Equivalence: `./boundary-and-equivalence.md`  
- State Models: `./state-models.md`  
- Scenario Patterns (MAE): `../30-scenario-patterns/main-alt-exception.md`  
- API contracts & pagination invariants: `../40-api-and-data-contracts/*`  
- Checklists: `../60-checklists/*`

---

<p align="center">
  <sub>
    Built with <a href="https://treeifyai.com">Treeify</a> — AI test case design copilot.<br>
    <em>Structured, traceable, high-coverage test design — faster.</em>
  </sub>
</p>
