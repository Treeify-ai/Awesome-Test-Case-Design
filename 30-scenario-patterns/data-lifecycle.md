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


# Data Lifecycle (Pattern)

> Data flows through **creation → use → evolution → archival → deletion (or anonymization)**.  
> This pattern turns lifecycle requirements into **clear scenarios**, **states**, and **evidence** so you can test correctness, privacy, and recoverability—not just CRUD.

---

## What & Why

- Lifecycle spans: **Ingest/Create → Validate → Active/Serving → Updated/Versioned → Archived/Cold → Deleted/Anonymized → (optional) Restored**.  
- Cross-cutting constraints: **retention**, **legal hold**, **privacy (PII/PHI/PCI)**, **audit**, **backups**, **derived data** (indexes, caches, analytics).  
- Failures often come from **edges between stages** (e.g., soft-delete vs search index, anonymized user lingering in logs, schema migration breaking replay).

Use this pattern to design **MAE scenarios** and **state models** for lifecycle-heavy features: profiles, orders, payments/refunds, logs/metrics, analytics events, documents/media.

---

## Quick workflow (8 steps)

1. **Identify data classes** (PII, financial, operational, analytics) and **owners/tenants**.  
2. Define **states** and **transitions** (see table below).  
3. Specify **retention** & **legal hold** rules (per class), plus **deletion semantics** (soft/delete/anonymize).  
4. Map **derived stores** (search index, cache, data lake, analytics warehouse) and **synchronization** rules.  
5. Define **oracles**: audit events, DB flags, S3 object state, index presence, metrics, traces.  
6. Write MAE scenarios (create, update/version, archive, delete/anonymize, restore, replay).  
7. Add **job behaviors**: retries/idempotency for archival and deletion workers.  
8. Gate with checklists: Privacy/Compliance, API, Resilience, Observability.

---

## Lifecycle states & transitions (reference)

| From         | Event/Trigger                 | Guard/Policy                         | Action/Side-effects                                     | To              | Observables (oracle)                               |
|--------------|-------------------------------|--------------------------------------|---------------------------------------------------------|-----------------|-----------------------------------------------------|
| `draft`      | `create`                      | schema valid                         | persist row; emit `created`                             | `active`        | DB row exists; audit `created`; trace span          |
| `active`     | `update`                      | allowed fields; ETag ok              | write delta; version++; emit `updated`                  | `active`        | DB updated; audit `updated`; ETag changes           |
| `active`     | `archive`                     | policy: inactive ≥ N days            | move/copy to cold store; mark `archived_at`             | `archived`      | storage object; audit `archived`; metric increment  |
| `archived`   | `restore`                     | not on legal_hold                    | copy back; clear `archived_at`                          | `active`        | DB restored; audit `restored`                       |
| `active`     | `soft_delete`                 | allow_soft_delete                     | set `soft_deleted=true`; remove from primary indexes    | `soft_deleted`  | flag set; search index drop; audit `soft_deleted`   |
| `soft_deleted`| `restore`                    | within restore window                | clear flag; re-index                                    | `active`        | audit `restored`                                    |
| `active|soft_deleted|archived` | `hard_delete` | retention met & not legal_hold       | delete primary; tombstone; purge derived copies         | `deleted`       | no row; tombstone logged; purge evidence            |
| `active|soft_deleted|archived` | `anonymize`   | privacy request; class supports anon | replace PII with tokens; keep referential integrity     | `anonymized`    | PII gone; audit `anonymized`; hash/salt recorded    |
| `*`          | `legal_hold_on/off`           | authorized                           | set/clear `legal_hold` → blocks delete/anonymize        | same            | audit `legal_hold`                                  |
| `*`          | `replay/backfill`             | idempotent keys available            | rebuild derived views (index/warehouse)                 | same            | job logs; counts match; checksums                   |

> **Choose one**: *hard delete* (remove) vs *anonymize* (replace PII, keep links). Regulatory regimes vary; make behavior explicit.

---

## MAE scenarios (copy/adapt)

### S-001 **Main** — Create → Update → Archive
- **Preconditions**: schema valid; retention policy exists  
- **Steps**: create record → update some fields → archive after N days  
- **Expected**: active → active (version+1) → archived; cold object present; search index shows “archived=false”  
- **Oracles**: DB row + `version`; S3 object; audit `created/updated/archived`; metric `archive.count`

### S-002 **Alt** — Restore from archive
- **Preconditions**: archived record; no legal hold  
- **Steps**: `POST /{id}:restore`  
- **Expected**: data returns to primary; index re-created  
- **Oracles**: DB restored; audit `restored`; search index count increases

### S-003 **Alt** — Soft delete with restore window
- **Steps**: `DELETE` (soft) → verify hidden from search/list → restore  
- **Expected**: `soft_deleted=true` hides it; restore flips flag and re-indexes  
- **Oracles**: flag flip; 204 on delete; 200 on restore; audit both events

### S-101 **Exception/Policy** — Hard delete blocked by legal hold
- **Preconditions**: `legal_hold=true`  
- **Steps**: `DELETE?force=true`  
- **Expected**: `423 LOCKED` (or `403 POLICY.legal_hold`), no purge  
- **Oracles**: audit `delete_blocked`; logs show `legal_hold=true`

### S-102 **Exception/Privacy** — Data subject erasure (anonymize)
- **Preconditions**: user requests **Right to Erasure**; retention for financial rows not met → **anonymize** instead of hard delete  
- **Steps**: run anonymizer worker  
- **Expected**: PII replaced with tokens; foreign keys intact; analytics events re-attributed to `anon_user_x`  
- **Oracles**: DB diff shows PII null/tokenized; search index has anonymized values; audit `anonymized`

### S-103 **Exception/Derived** — Search index out of sync
- **Steps**: soft delete → index still returns result  
- **Expected**: repair job reconciles; index drop confirmed  
- **Oracles**: mismatch detected metric; repair log; final parity

### S-104 **Exception/Resilience** — Archive job retry/idempotent
- **Preconditions**: object store flaps; first upload times out  
- **Steps**: job retries with same object key  
- **Expected**: **one** object; no duplicates; status recorded once  
- **Oracles**: idempotency key reuse; bucket listing; audit single `archived`

### S-105 **Exception/Backfill** — Replay with schema evolution
- **Preconditions**: events v1→v2; migration provides adapter  
- **Steps**: replay events for last 7 days  
- **Expected**: consistent derived tables; checksums match; no duplicate rows  
- **Oracles**: job run ID; counts per partition; checksum; sample records pass schema

---

## Deletion semantics (decide & test)

- **Soft delete**: keep row; set flag; remove from **default** lists/search; enable restore.  
- **Hard delete**: physically remove row and **all** derived copies; write **tombstone** (audit).  
- **Anonymize**: replace **PII** with **pseudonyms/hashes**; keep referential integrity (orders, invoices).  
- **Cascade rules**: child rows may be **deleted**, **anonymized**, or **re-parented** (e.g., to `anon_user`). Test each.

**Pitfalls**  
- Orphaned child rows; dangling cache/index entries; analytics still carrying PII; backups retaining PII beyond policy.

---

## Retention & legal hold

- **Retention clock**: from **create** or **last activity**—be explicit.  
- **Tiering**: hot vs warm vs cold storage; costs & access latencies (affect tests).  
- **Legal hold**: blocks delete/anonymize/archive transitions; must be **audited** and **reversible** by authorized roles only.  
- **Grace windows**: restore allowed up to N days after soft delete.

---

## Derived data mapping (don’t forget these)

- **Search index** (Elasticsearch/OpenSearch): add/remove on create/update/delete; soft-delete should **drop** index doc unless policy says “hidden”.  
- **Cache** (Redis): invalidate on update/delete/restore.  
- **Analytics/Warehouse** (BigQuery/Snowflake): late-arriving corrections; GDPR delete signals; **data subject erasure** fan-out.  
- **Data lake / object store**: archive object lifecycle rules; checksum verification.

---

## Oracles & Evidence

- **DB**: flags (`soft_deleted`, `archived_at`, `legal_hold`, `version`), FK integrity, row counts by state.  
- **Storage**: object presence (bucket/key), ETag/MD5, versioning, retention lock.  
- **Index/cache**: document count; absence/presence; TTL; invalidation logs.  
- **Audit**: `{actor, action, target, reason, legal_hold, correlation_id}` for **create/update/archive/delete/anonymize/restore**.  
- **Metrics**: `archive.count`, `delete.count`, `anonymize.count`, `repair.mismatch`, `backfill.duration`.  
- **Traces**: spans for lifecycle jobs with attributes (state_from/to, object_key, idempotency_key).

---

## Anti-patterns

- Deleting primary row but **not** derived copies (index/cache/warehouse).  
- **No tombstones** → later re-create with same ID hides the fact it previously existed.  
- **PII in logs** or exports; forgetting **backup** purges.  
- **Mixed semantics** (sometimes 404, sometimes 410) for deleted items.  
- Anonymization that **breaks FKs** or analytics attribution without a plan.  
- Replay/backfill that **duplicates** data (no idempotency keys).  
- Unclear retention start (clock resets unpredictably).

---

## Review checklist (quick gate)

- [ ] States & transitions documented (incl. **restore**, **legal hold**)  
- [ ] **Delete policy** chosen: soft vs hard vs anonymize; cascade rules clear  
- [ ] **Retention** clock & windows defined; **legal hold** blocks enforced  
- [ ] **Derived stores** updated consistently (index/cache/warehouse/lake)  
- [ ] **Jobs** are **idempotent** with retries/backoff  
- [ ] **Oracles** defined: audit, DB flags, storage/index evidence, metrics, traces  
- [ ] **Privacy**: PII removed/masked in logs/exports; backups strategy aligned  
- [ ] **HTTP semantics** consistent (404/410/409/423 etc.) and copy mapped to taxonomy

---

## CSV seeds

**State audit**

```csv
id,action,from,to,actor,legal_hold,evidence
PM-001,create,,active,system,false,audit:created+db_row
PM-002,archive,active,archived,job,false,bucket:key:v1+audit
PM-003,restore,archived,active,admin,false,db_row+audit_restored
PM-004,soft_delete,active,soft_deleted,editor,false,flag=true+index_drop
PM-005,hard_delete,soft_deleted,deleted,admin,false,row_absent+purged_index
PM-006,anonymize,active,anonymized,privacy_job,false,pii->token+audit
PM-007,delete_blocked,active,active,admin,true,resp423+audit_hold
```

**Retention test plan**

```csv
record_id,class,created_at,last_activity,retention_days,legal_hold,delete_eligible_expected
u-001,profile,2025-01-01,2025-03-01,180,false,false
o-101,order,2025-01-10,2025-01-20,365,false,false
log-9,access_log,2025-02-01,2025-02-01,30,false,true
```

**Derived parity check**

```csv
date,total_db,idx_docs,cache_keys,warehouse_rows,parity_ok
2025-09-01,1000,1000,120,995,false
2025-09-02,1250,1250,135,1248,false
2025-09-03,1500,1500,140,1500,true
```

---

## Templates

**Scenario card**

```
## S-<nnn> <Type> — <short name>
Preconditions: <state, class, holds, clocks>
Trigger: <api/job/user>
Expected: <new state + side-effects>
Oracles: <db flags, storage/index, audit, metrics, traces, http code>
Recovery (Exception): <repair/backfill/restore>
```

**Transition table**

```
| From | Event | Guard | Action | To | Observables |
|------|-------|-------|--------|----|-------------|
```

---

## Links

- Error taxonomy & HTTP mapping: `../40-api-and-data-contracts/error-taxonomy.md`  
- Idempotency & retries for jobs: `../40-api-and-data-contracts/idempotency-and-retries.md`  
- Pagination/filtering & stable sorts for lists: `../40-api-and-data-contracts/pagination-and-filtering.md`  
- Privacy & Compliance: `../50-non-functional/privacy-and-compliance.md`  
- Observability hooks (evidence capture): `../80-tools-and-integrations/observability-hooks.md`  
- Traceability (Req ↔ Scenario ↔ Case ↔ Evidence): `../65-review-gates-metrics-traceability/traceability.md`

---

<p align="center">
  <sub>
    Built with <a href="https://treeifyai.com">Treeify</a> — AI test case design copilot.<br>
    <em>Structured, traceable, high-coverage test design — faster.</em>
  </sub>
</p>
