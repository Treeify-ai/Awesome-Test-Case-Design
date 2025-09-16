# Human-in-the-loop — review notes → regenerate cycles

> Use people **where they add the most signal**, and make every review **train the system**.  
> This page standardizes reviewer rubrics, queues, feedback schemas, and regenerate loops so AI features improve **predictably**.

---

## TL;DR

- Gate AI outputs with **risk tiers** and **confidence thresholds**.  
- Reviews create **structured feedback** (machine-actionable), not freeform prose.  
- Regeneration cycles are **bounded** (N tries) with **escalation** and **ownership**.  
- Evidence is attached on every hop: **schema validation**, **citations/grounding**, **policy**.  
- Close the loop: turn reviews into **goldens**, update **evals**, and track **drift**.

---

## When to put a human in the loop

- **Policy-sensitive**: security, privacy, legal, compliance, abuse.  
- **High-risk impacts**: money movement, medical, safety.  
- **Low confidence**: model `confidence < threshold` or `schema_invalid`.  
- **Missing evidence**: no source supports the claim (RAG empty).  
- **Long-tail**: novel intents, unseen languages/locales, domain edge-cases.

**Default thresholds (edit to fit):**  
- Auto-approve if `confidence ≥ 0.85` AND `schema_valid` AND `no_policy_flags`.  
- Route to review if `0.5 ≤ confidence < 0.85` OR `needs_citation`.  
- Auto-refuse/regenerate if `confidence < 0.5` OR `schema_invalid` OR `policy_flagged`.

---

## Workflow (states & signals)

`intake → draft → critic → review → regenerate? → approve → publish | refuse | escalate`

- **intake**: create task with payload, context, and **trace_id**.  
- **draft**: first model output (schema-constrained).  
- **critic**: automated checks (schema, safety, citations).  
- **review**: human rubric applies; feedback captured as **structured deltas**.  
- **regenerate**: model retries using feedback; max **N cycles** (default: 2).  
- **approve/publish**: changes applied; evidence bundled.  
- **refuse**: deterministic refusal with message id.  
- **escalate**: to subject-matter expert when unresolved.

**Queue priorities:** P0 (safety/security) → P1 (customer-visible) → P2 (backlog).

---

## Reviewer rubric (short, high-signal)

**Binary gates**  
- Schema valid?  
- Policy clean? (no PII/leakage, allowed domain)  
- Grounded? (citations support claims)  
- Duplicates removed?  
- Clear & complete?

**Scaled fields** (0..3)  
- Task relevance  
- Factuality  
- Usefulness

**Evidence attachments**  
- Validation error (if any), citation list, diff vs input, screenshots/logs.

> Keep comments short and **actionable**. Long notes belong in a linked doc/postmortem.

---

## Feedback schema (machine-actionable)

```json
{
  "version": "1.0",
  "decision": "approve|regenerate|refuse|escalate",
  "reasons": ["SCHEMA_INVALID","POLICY_BREACH","GROUNDING_MISSING","LOW_CONFIDENCE","DUPLICATE","AMBIGUOUS"],
  "edits": [
    { "op": "replace", "path": "/title", "value": "Short, plain title" },
    { "op": "delete", "path": "/items/2" }
  ],
  "hints": ["add_citations", "tighten_claims", "dedup_items"],
  "msgid": "MSG.review.feedback",
  "evidence": ["link://trace/abc123","link://screens/shot1"]
}
```

**Error taxonomy seeds**  
- `SCHEMA_INVALID` — JSON fails validation  
- `POLICY_BREACH` — safety/privacy/compliance rule violated  
- `GROUNDING_MISSING` — claim lacks citation/support  
- `LOW_CONFIDENCE` — model uncertainty below threshold  
- `DUPLICATE` — outputs repeat meaningfully identical items  
- `AMBIGUOUS` — user intent unclear; ask 1 question

---

## Regenerate loop (bounded)

- **N cycles** max (default 2). On last failure → **escalate** or **refuse**.  
- Each cycle **diffs** output vs previous, applies reviewer **edits/hints**, re-runs **critic**.  
- **Idempotency**: attach `attempt_id` and preserve **prior best** result.  
- **Cold-path**: if critic finds only schema issues, **auto-regenerate** without waiting for human.  

**Regeneration prompt (sketch)**

```
SYSTEM: Apply ONLY the structured feedback below. Do not invent content.
INPUT: <current_output_json/>
FEEDBACK: <feedback_json/>
OUTPUT: Return minified JSON that validates the schema. Keep IDs stable; dedup.
```

---

## Observability

**Logs (msgid)**  
- `MSG.hitl.queue.accepted`  
- `MSG.hitl.review.feedback`  
- `MSG.hitl.regenerate.requested`  
- `MSG.hitl.regenerate.succeeded|failed`  
- `MSG.hitl.approved|refused|escalated`

**Metrics**  
- `hitl_queue_depth`  
- `hitl_time_to_decision_ms` (p50/p95)  
- `hitl_cycles_per_item` (avg)  
- `hitl_approval_rate_pct`  
- `hitl_escalation_rate_pct`  
- `hitl_schema_invalid_rate_pct`  
- `hitl_grounding_missing_rate_pct`  
- **Quality**: inter-rater agreement (kappa), post-ship edit rate

**Traces**  
- Spans: `critic.validate`, `review.capture`, `regenerate.call`, with **exemplars** for slow items.

---

## Roles, permissions, audit

- **Reviewer**: apply rubric; cannot change prompts/schemas.  
- **Editor**: can update goldens, tweak shots.  
- **Owner**: adjusts thresholds, N cycles, escalation policy.  
- **Auditor**: read-only access to logs, evidence, decisions.

Audit log fields: `who`, `when`, `decision`, `reasons`, `diff_hash`, `trace_id`.

---

## Data & privacy

- Redact secrets (tokens, account numbers) before display.  
- Remove PII from training/evals unless consented and necessary.  
- Keep reviewer freeform notes **out of prompts**; use structured feedback only.  
- Retention: purge raw inputs after **T days** unless required for audit.

---

## Integrating with this repo

- Map **REQ → SCN → CASE → Evidence** (`65-review-gates-metrics-traceability`).  
- Gate in CI at **G4**: require **schema-valid** outputs and **no policy flags** for auto-merge; otherwise route to HITL queue.  
- Evaluation drifts feed back to **goldens** in `tests/goldens/…` with PR review.

---

## MAE scenarios (HITL)

### HITL-MAIN-001 — Low-confidence, single pass
- **Given** model `confidence=0.7`, schema valid  
- **Then** route to review → approve with zero edits

### HITL-ALT-001 — Missing citations
- **Given** output claims facts with `sources=[]`  
- **Then** reviewer requests regenerate with `GROUNDING_MISSING` → model adds citations → approve

### HITL-ALT-002 — Dedup fix
- **Given** near-duplicate items  
- **Then** feedback `DUPLICATE` + `edits` delete one → regenerate passes

### HITL-EXC-001 — Schema invalid
- **Given** invalid JSON  
- **Then** auto-regenerate once; if still invalid → escalate

### HITL-EXC-002 — Policy breach
- **Given** PII leaked in output  
- **Then** refuse with `POLICY_BREACH` and notify owner

---

## Checklists

**Setup**  
- [ ] Risk tiers and thresholds defined  
- [ ] Feedback schema & error taxonomy stable  
- [ ] Critic validators (schema/safety/citations) in CI  
- [ ] Queues with priorities & SLAs  
- [ ] Reviewer training & calibration session held

**Per-PR**  
- [ ] Evidence bundle attached (inputs, outputs, feedback, cycles)  
- [ ] Metrics dashboard links (queue depth, time to decision)  
- [ ] Goldens updated when appropriate  
- [ ] Trace IDs included for slow/failed cycles

---

## Templates

**Reviewer form JSON**

```json
{
  "decision": "regenerate",
  "reasons": ["GROUNDING_MISSING"],
  "hints": ["add_citations"],
  "edits": [],
  "msgid": "MSG.review.feedback",
  "notes": "Keep claims narrow; cite source 12, p.3"
}
```

**Escalation note**

```
Case: <id>
Reason: POLICY_BREACH (PII)
What’s needed: SME review for redaction policy
Links: <trace/dashboards/evidence>
```

**Queue item (YAML)**

```yaml
id: case_123
priority: P1
state: review
confidence: 0.62
attempts: 1
trace_id: tr_abc
sla_deadline: 2025-09-17T09:00:00Z
```

---

## CSV seeds

**Queue register**

```csv
id,priority,state,attempts,confidence,owner
case_001,P1,review,0,0.68,AI
case_002,P0,review,1,0.42,Security
case_003,P2,approve,0,0.90,Docs
```

**Reviewer roster**

```csv
name,role,time_zone
Alice,Reviewer,UTC+08
Ben,Owner,UTC+08
Chin,Auditor,UTC+08
```

**Error taxonomy**

```csv
code,meaning
SCHEMA_INVALID,JSON failed validation
POLICY_BREACH,Safety or privacy issue
GROUNDING_MISSING,Citations/evidence absent
LOW_CONFIDENCE,Model score below threshold
DUPLICATE,Duplicate content
AMBIGUOUS,Needs clarification
```

---

## Common pitfalls

- Freeform reviewer notes that models can’t consume.  
- Unlimited regenerate loops; costs rise, quality stalls.  
- Mixing policy and taste—keep the rubric objective.  
- Not updating goldens; regressions repeat.  
- Missing audit trail; hard to explain decisions later.

---

## Sign-off

- [ ] Thresholds tuned; queue SLAs met p95.  
- [ ] Regenerate cycles bounded; escalation path exercised.  
- [ ] Reviewers calibrated (kappa ≥ 0.7).  
- [ ] Dashboards & traces wired; owners on-call.  
- [ ] Goldens/evals updated from review learnings.
