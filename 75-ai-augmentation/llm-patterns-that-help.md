# LLM Patterns That Help

> Ship AI features that are **predictable**, **auditable**, and **useful**—without hand-wavy magic.  
> These patterns turn LLMs into reliable components you can spec, test, and monitor.

---

## TL;DR

- **Structure first**: make the model emit **strict JSON** that validates against a schema.  
- **Plan → Act → Critique**: decompose the task, call tools, then run a short **critic** pass.  
- **Sample smart, not wild**: use **self‑consistency** (n=3–7) with a deterministic **aggregator**.  
- **Ground answers**: Retrieval‑Augmented Generation (**RAG**) with citations and source tags.  
- **Ask, don’t guess**: if inputs are ambiguous, ask **one clarifying question** and stop.  
- **Version & observe**: tag **prompt_version**, **schema_version**, **model_version**; emit **msgid**.

---

## When to use which pattern (rules of thumb)

- **Classification / extraction** → Structured output + small few‑shot + self‑consistency.  
- **Summarization / rewrite** → Targeted instructions + length/style constraints + citations.  
- **Question answering** → RAG (retrieve, rerank, cite) + refusal when sources are absent.  
- **Planning / multi‑step** → ReAct (reason+act) or Plan‑then‑Execute with a brief **critic** pass.  
- **Data generation** → Schema + property‑based constraints + dedup + seeded randomness.

---

## 1) Structured Output (always)

**Why**: Parseability, tests, and contracts.  
**How**: JSON Schema / Pydantic; `additionalProperties: false`; refuse unknowns.

**Prompt footer (enforce)**

> Return **only** minified JSON that matches the schema.  
> If invalid, return `{"version":"1.0","msgid":"MSG.schema.invalid"}`.

---

## 2) Clarify‑then‑Answer (CTA) state machine

**Idea**: Don’t guess missing input.  
**Contract**: Two states: `clarify_needed` (one question) → `answer_ready`.  
**Signals**: `msgid` of `MSG.clarify.required` vs `MSG.ok`.  
**Test**: MAE cases for ambiguous inputs.

---

## 3) ReAct & Critic‑then‑Actor

- **ReAct**: model alternates **reasoning** and **tool calls** with short, explicit notes.  
- **Critic pass**: a small second call checks **policy**, **schema**, and **logic**; it **doesn’t** rewrite unless issues are found (then return a refusal or a minimal fix).

**Critic checklist**

- Schema valid?  
- Claims grounded (if RAG)?  
- Safety policy respected?  
- Duplicates removed?  
- Message IDs present?

---

## 4) Self‑Consistency (n‑best + aggregator)

**Why**: Reduces variance without raising temperature too high.  
**How**: Sample the model **k=3–7** times at `temperature≈0.7`, then aggregate **deterministically**.

**Aggregator (classification pseudocode)**

```python
from collections import Counter

def aggregate(samples):
    # each sample is {"label": "...", "evidence": [...]}
    labels = [s["label"] for s in samples]
    winner, _ = Counter(labels).most_common(1)[0]
    # keep evidence for the winning label only
    ev = next(s["evidence"] for s in samples if s["label"] == winner)
    return {"label": winner, "evidence": ev, "msgid": "MSG.ok", "version": "1.0"}
```

**Tests**: deterministic result given the same set; tie‑break by stable order.

---

## 5) Decomposition (Tree‑/Graph‑of‑Thoughts, safely)

**Goal**: Break down tasks into smaller sub‑prompts; **return only final outputs**.  
**Guardrail**: Never leak long internal scratchpads to end users; store them as artifacts if needed for debugging.

**Pattern**  
- Step 1: list sub‑tasks (short bullet limits).  
- Step 2: execute each with schema outputs.  
- Step 3: merge/dedup; run critic.

---

## 6) Retrieval‑Augmented Generation (RAG) that actually helps

- **Chunking**: semantic chunks with overlap; store **source_id**, **page**, **span**.  
- **Retrieval**: BM25 + dense hybrid; **rerank** top‑k (cross‑encoder) for precision.  
- **Citations**: include `[source_id:#, page:]` next to each claim; refuse if no source supports it.  
- **Cache**: key by `(query_hash, index_version)`.  
- **Tests**: golden Q&A with exact‑match sources; regression on **citation coverage**.

---

## 7) Constrained decoding

- **JSON‑only** decoding via schema/grammar.  
- **Regex** constraints for small formats (dates, IDs).  
- **Stop sequences** to prevent run‑on generation.

**Why**: eliminates a class of parsing failures; makes evals cheap.

---

## 8) Few‑shot, but surgical

- Keep **3–5** high‑quality shots; each illustrates **one rule**.  
- Tag shots with **msgid** so evals can assert on outcomes.  
- Avoid bloated “mega‑shots”; use **short, focused** exemplars.

---

## 9) Safety & policy patterns

- **Refusal templates** with `msgid` and minimal text.  
- **Injection resistance**: boundary markers around user text; ignore contrary instructions inside them.  
- **Redaction**: remove secrets by regex before prompting.  
- **Domain off‑ramps**: clear handoff (e.g., “contact support”) for high‑risk asks.

---

## 10) Versioning & Observability

- Emit: `prompt_version`, `schema_version`, `model_version`, `msgid`, `latency_ms`.  
- Logs link to **traces** of retrieval/tool calls.  
- Dashboards: `schema_invalid_rate_pct`, `refusal_rate_pct`, `clarification_rate_pct`, `dedup_ratio_pct`.

---

## 11) Data generation patterns (for tests)

- **Property‑based**: generate inputs under constraints, assert invariants (e.g., totals ≥ 0).  
- **Boundary cases**: near limits (length, numeric, date).  
- **Pairwise** for parameter grids; dedup by canonical form.  
- **Seeded randomness** to reproduce failures.

---

## 12) Idempotency & traceability

- **Idempotency keys** for tool calls; never repeat side‑effects.  
- Map **REQ → SCN → CASE → Evidence** using IDs so model outputs are auditable.

---

## MAE scenarios (for these patterns)

### PAT-MAIN-001 — Structured classification
- **Given** clean input  
- **Then** valid JSON; correct label; `MSG.ok` present

### PAT-ALT-001 — Ambiguous input
- **Then** model returns one **clarifying question** and stops (`MSG.clarify.required`)

### PAT-ALT-002 — Self‑consistency majority
- **Then** aggregator returns stable result from k samples

### PAT-EXC-001 — No sources in RAG
- **Then** refusal with `MSG.evidence.missing` (or similar); no hallucinated claims

### PAT-EXC-002 — Schema violation
- **Then** minimal invalid JSON; log `ERR.SCHEMA.invalid`

---

## Review checklist

- [ ] Output uses **strict schema**; `additionalProperties: false`.  
- [ ] **Clarify‑then‑Answer** implemented for ambiguous inputs.  
- [ ] **Self‑consistency** enabled where accuracy matters; aggregator deterministic.  
- [ ] **RAG** uses hybrid retrieve + rerank; citations attached; empty results → refusal.  
- [ ] **Constrained decoding** or grammar used for JSON.  
- [ ] **Few‑shot** exemplars short and labeled with `msgid`.  
- [ ] **Safety**: injection resistance, refusal templates, redaction.  
- [ ] **Versioning**: prompt/schema/model versions emitted.  
- [ ] **Observability**: metrics & traces wired; dashboards exist.  
- [ ] **Evals**: golden sets + adversarial tests pass in CI.

---

## Templates

**Eval harness (YAML)**

```yaml
task: classification
schema_version: "1.0"
prompt_version: "2025-09-16"
k: 5  # self-consistency samples
metrics:
  - accuracy
  - schema_invalid_rate_pct
  - clarification_rate_pct
  - refusal_rate_pct
goldens: tests/goldens/classify_email.jsonl
shots: prompts/shots/classify_minimal.json
```

**Aggregator contract (JSON)**

```json
{
  "type": "object",
  "required": ["label", "evidence", "version", "msgid"],
  "additionalProperties": false,
  "properties": {
    "label": { "type": "string" },
    "evidence": { "type": "array", "items": { "type": "string" } },
    "version": { "type": "string", "enum": ["1.0"] },
    "msgid": { "type": "string", "pattern": "^MSG\\." }
  }
}
```

**Refusal JSON**

```json
{"version":"1.0","msgid":"MSG.evidence.missing"}
```

---

## CSV seeds

**Golden set (example)**

```csv
id,input,label
1,"Please reset my password",account_support
2,"Refund card ending 4242",billing
3,"When is the release?",product_info
```

**Metrics registry**

```csv
metric,unit,owner
schema_invalid_rate_pct,pct,AI
refusal_rate_pct,pct,AI
clarification_rate_pct,pct,AI
self_consistency_agreement_pct,pct,AI
inference_latency_ms,ms,AI
```

---

## Links

- Prompt Guardrails → `./prompt-guardrails.md`  
- Traceability → `../65-review-gates-metrics-traceability/traceability.md`  
- API Error Taxonomy → `../40-api-and-data-contracts/error-taxonomy.md`  
- Performance (p95/p99) → `../50-non-functional/performance-p95-p99.md`
