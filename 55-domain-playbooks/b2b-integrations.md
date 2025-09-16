# Domain: B2B Integrations

> Two companies, two stacks, one contract.  
> This playbook turns partner integrations into **predictable, testable** work across **APIs, webhooks, files (SFTP/CSV), EDI**, and hybrid flows—covering onboarding, contracts, security, SLAs, and evidence.

---

## TL;DR (defaults we recommend)

- Pick a **primary mode** (API, Webhook, or File) and treat others as **bridges**, not random extras.  
- Standardize **contracts**: versioned schemas, **idempotency keys**, **message IDs**, **pagination**, **retry/backoff**, **error taxonomy**.  
- **Authenticate** with **OAuth2 client credentials** or **mTLS** for APIs; **SSH keys + IP allowlist** for SFTP; **HMAC** for webhooks.  
- Build an **inbox** (dedupe table) for **at-least-once** events/files; support **replay** windows.  
- Define **SLAs** by flow (latency for APIs, schedule windows for files) and **prove** with metrics/dashboards per partner.  
- Ship a **Partner Certification** checklist: happy-path + negatives + replays + cutover/backfill.  
- Keep **observability** first-class: correlation IDs, partner IDs, control totals, evidence archives.

---

## Scope (what we cover)

- Modes: **REST/GraphQL/SOAP APIs**, **Webhooks**, **Files** (CSV/TSV/JSON), **SFTP**/**PGP**, **EDI X12/EDIFACT**, **message buses** (optional).  
- Flows: **orders**, **catalog**, **inventory**, **shipment/ASN**, **invoices**, **payments**, **usage**, **entitlements**, **user/SSO**.  
- Actors: **Provider** (we offer API), **Consumer** (we call theirs), **Trading partner** (both sides).

---

## Onboarding (partner playbook)

1. **Business alignment**: objectives, flows, volumes, SLAs, go-live criteria.  
2. **Contract choice**: API vs File vs Hybrid; agree on **source of truth** and **direction** (push/pull).  
3. **Security exchange**: keys/certs, IPs, webhook secrets, SFTP creds via secure channel.  
4. **Environments**: sandbox URLs, sample data, **mocks** for non-existent systems.  
5. **Mappings**: fields, enums, currencies, units, time zones; define **defaults**.  
6. **Test plan**: MAE scenarios, negative cases, **replay** and **dup** handling.  
7. **Certification**: pass criteria + artifacts (logs, files, screenshots, reports).  
8. **Cutover**: backfill, dual-run window, rollback plan, support contacts, paging.

---

## Contracts & Schemas (must-haves)

- **IDs**: stable `external_id` for partner entities; our `internal_id` stays private.  
- **Versioning**: `/v1` or media types; **deprecate** with window; changelog maintained.  
- **Idempotency**: keys on all **unsafe POSTs** and **file batches**; **replay returns same outcome**.  
- **Pagination/Filtering**: **cursor** based; stable sort with tiebreakers; documented filters.  
- **Errors**: return **error codes + message IDs**; provide **mapping doc** from partner to ours.  
- **Time**: all timestamps in **UTC ISO-8601 Z**; include **time zone** when input is local.  
- **Money/Units**: minor units + `currency`; SI units + `unit`.  
- **PII flags** per field; retention/consent requirements.

See: `../40-api-and-data-contracts/contracts-and-schemas.md`, `../40-api-and-data-contracts/error-taxonomy.md`.

---

## Authentication & Authorization

| Context | Recommended |
|---|---|
| API → our system | **OAuth2 client_credentials** with scopes; **mTLS** optional; IP allowlist optional |
| Our system → partner API | OAuth2 or **mTLS**; rotate secrets; scope tokens |
| Webhooks from us | **HMAC-SHA256** signature with timestamp; replay window 5–10 min |
| SFTP | **SSH key** auth; **PGP** at-rest (optional but recommended) |
| EDI VAN/AS2 | Signed & encrypted; MDN receipts; control numbers/ACKs |

**Tests**: expired tokens (401), wrong scope (403), invalid HMAC (401), unapproved IP (403).

---

## Real-time (APIs & Webhooks)

### REST/GraphQL/SOAP

- Respect **timeouts** and **retries w/ jitter** on `429/5xx/timeouts`; cap by **deadline**.  
- **Backpressure**: rate limits with `429` + `Retry-After`.  
- **Bulk** endpoints for large syncs; partial failure reporting.

### Webhooks (push)

- **At-least-once** delivery: consumer **inbox** dedupes by `event_id + hash(payload)`; **idempotent** handlers.  
- **Signature** header: `X-Signature: t=<ts>, v1=<HMAC>`; expire after window.  
- **Retry policy**: exponential backoff with cap; **dead-letter** after N tries; **/replay** endpoint for ops.  
- **Ordering**: **not guaranteed**; include `occurred_at` and related IDs to reconstruct.

**Tests**: out-of-order events; duplicate deliveries; signature failures; long delays; replay.

---

## Batch (SFTP/CSV/JSON) & Manifests

- **Structure**: `inbox/`, `outbox/`, `archive/`, `error/` directories per partner.  
- **File naming**: `partner_flow_YYYYMMDDThhmmssZ_seq.ext` + **monotonic** sequence.  
- **Atomicity**: upload to temp name → **rename** on complete.  
- **Integrity**: sidecar `.sha256`; optional `.pgp` encryption; **manifest** with **control totals** (record count, sums).  
- **Partial** processing: reject whole file on structural error; report **line-level** errors in an **error file**.  
- **Acknowledgments**: drop `ACK` file or API callback.

**Tests**: truncated upload (no rename); wrong checksum; duplicate file (same seq); missing manifest; control-totals mismatch.

---

## EDI (X12/EDIFACT) quick primer

- Common docs: **850** (PO), **855** (PO Ack), **856** (ASN), **810** (Invoice), **846** (Inventory).  
- **Interchange/Group/Transaction** control numbers; **997** functional ACKs.  
- **Mapping** to internal schema; manage per-partner quirks (qualifiers, code lists).  
- **Dedup**: control numbers + partner id; store 90–180 days.

**Tests**: bad segment terminator; missing mandatory segments; duplicate control number; delayed 997 handling.

---

## Schedules, SLAs, and Windows

- **APIs**: latency SLOs (e.g., p95 ≤ 300 ms, p99 ≤ 800 ms), uptime, rate limits.  
- **Files**: delivery windows (e.g., 02:00–03:00 UTC daily), lateness thresholds, **retries** before paging.  
- **Replays**: retention window (e.g., 30 days) and **/replay** mechanics (event id, file name).  
- **Brownouts**: degrade behavior (cache, queue) when partner slow; clear error signals to ops.

---

## Observability & Evidence

- **IDs**: `partner_id`, `integration_id`, `correlation_id`, `event_id`, `file_seq`.  
- **Metrics**: success/failure by flow & partner; latency; retry count; replay count; backlog sizes.  
- **Logs**: structured; **no PII**; include mapping outcomes and rule ids.  
- **Traces**: cross-system with correlation id.  
- **Archives**: immutable copies of **sent/received** payloads (redacted where needed), manifests, checksums, ACKs.

Dashboards per partner; **alerts** on SLA breaches, retries, backlog growth, control total mismatch.

---

## Failure handling & resilience

- **Idempotent upserts** everywhere; dedupe tables (events, files, EDI control).  
- **Backoff + jitter**; **circuit breakers** for flaky partners; **bulkheads** per dependency.  
- **Dead-letter queues** with ops UI to inspect & replay.  
- **Poison messages** detection; quarantine & notify.  
- **Compensation** flows when two-way sync diverges.

---

## Privacy, Compliance & Security

- **Data minimization** per flow; mark PII fields; redact in logs.  
- **Cross-border** location promises honored (region pinning).  
- **Processor agreements/DPAs** tracked; **vendor register** maintained.  
- **Key rotation** schedule for SSH/PGP/tokens; secret scanning in CI.  
- **Access**: least privilege on SFTP buckets, queues, topics; audit admin actions.

---

## Cutover, Backfill & Dual-run

- **Backfill**: one-time export/import; track **high-water mark** (timestamp, id).  
- **Dual-run**: compare aggregates and samples; reconcile diffs; freeze changes if needed.  
- **Rollback**: switch to previous path; clear queued deltas; communicate timelines.

---

## MAE Scenarios (copy/adapt)

### B-001 **Main** — Order export (API pull)
- **Steps**: partner pulls `/v1/orders?changed_since=<ts>&cursor=<c>`  
- **Expected**: stable cursor; no dupes/misses under concurrent writes  
- **Oracles**: control query; item count equals delta; traces show pagination invariants

### B-002 **Alt** — Webhook order.created + replay
- **Steps**: receive `order.created`; process; later replay same `event_id`  
- **Expected**: exactly-once outcome; second delivery marked **replayed**  
- **Oracles**: inbox table; idempotent handler logs

### B-003 **Alt** — SFTP inventory file with manifest
- **Steps**: upload `inventory_20250916T0200Z_001.csv` + `.sha256` + `manifest.json`  
- **Expected**: process after atomic rename; control totals match; ACK dropped in `outbox/`  
- **Oracles**: archive copy; control sums; ACK file

### B-101 **Exception** — Duplicate file sequence
- **Steps**: resend `_001.csv`  
- **Expected**: ignored with audit; optional `409` via API  
- **Oracles**: dedupe table hit; error report

### B-102 **Exception** — Webhook signature fail
- **Steps**: tampered body  
- **Expected**: `401` reject; no processing; alert generated  
- **Oracles**: signature verify logs; alert fired

### B-103 **Exception** — Partner API slowness
- **Steps**: p95 +200 ms; occasional 503  
- **Expected**: backoff + jitter; circuit opens; brownout path (queue)  
- **Oracles**: breaker metrics; backlog within limits

### B-104 **Exception** — EDI duplicate control number
- **Steps**: receive same **ISA/GS/ST** numbers  
- **Expected**: dedup; 997/MDN behavior correct; notify partner  
- **Oracles**: control table; ACK status

### B-201 **Cross-feature** — Tax/Inventory/Price sync
- **Expected**: consistent order totals; no stale price on shipment; inventory decremented once  
- **Oracles**: reconciliation report; event chain shows idempotency

---

## Review checklist (quick gate)

- [ ] Contract chosen (API/Webhook/File/EDI) with **source of truth** & versioning  
- [ ] **Auth** configured (OAuth2/mTLS/HMAC/SSH); keys rotated; IP allowlist as needed  
- [ ] **Idempotency** + **dedupe** tables; replay endpoints/windows  
- [ ] **Pagination** & **filtering** contracts; invariants documented  
- [ ] **Error taxonomy** mapped; decline/validation/policy codes aligned  
- [ ] **SLA/SLOs** set per flow; dashboards & alerts per partner  
- [ ] **Batch**: atomic upload, checksum, manifest, control totals, ACKs  
- [ ] **EDI**: control numbers, 997/MDN, partner profile stored  
- [ ] **Privacy**: PII fields flagged; logs redacted; region pinning respected  
- [ ] **Cutover/backfill** plan with dual-run and rollback  
- [ ] **Certification** plan: MAE scenarios + evidence artifacts

---

## CSV seeds

**Partner registry**

```csv
partner_id,name,mode,security,contact,sla
p_001,Acme Retail,API+Webhook,oauth2+hmac,ops@acme.com,"p95<=300ms, uptime 99.9%"
p_002,Globex,File(SFTP),ssh+pgp,integrations@globex.example,"file@02:00Z daily"
p_003,Initech,EDI,as2+mdn,edi@initech.example,"ack<=15m, replay 30d"
```

**Webhook secret registry**

```csv
partner_id,endpoint,secret,window_s
p_001,https://acme.example/webhooks/orders,***,600
```

**File manifest (example JSON)**

```json
{
  "file": "inventory_20250916T0200Z_001.csv",
  "records": 12543,
  "sums": { "qty": 347991 },
  "generated_at": "2025-09-16T02:00:03Z",
  "sha256": "b2f0…",
  "schema_version": 1
}
```

**Retry policy**

```csv
flow,max_attempts,backoff_ms_cap,jitter
webhook.delivery,8,300000,full
api.call,4,60000,full
file.transfer,5,600000,full
```

**Field mapping (snippet)**

```csv
ours,partner,type,notes
order_id,OrderNumber,string,stable external id
total_amount,TotalGross,integer,minor units
currency,CurrencyCode,string,ISO-4217
updated_at,LastModified,datetime,UTC ISO-8601
```

---

## Templates

**Integration spec header**

```
Partner: <name>  Partner ID: <id>
Flows: <orders|catalog|inventory|invoice|shipment|…>
Mode: <API|Webhook|File|EDI|Hybrid>  Source of truth: <who>
Auth: <oauth2|mtls|hmac|ssh|as2>
Endpoints/Dirs: <urls|sftp paths>
Schemas: <links to JSON Schema/OpenAPI/CSV spec/EDI map>
SLAs: <latency|windows|ack>
Observability: <metrics|dashboards|alerts>
Runbooks: <links>  Contacts: <ops, escalation>
```

**Webhook signature verify (pseudo)**

```python
def verify(sig_header, timestamp, body, secret, window_s=600):
    if abs(now() - timestamp) > window_s: raise Expired()
    expected = hmac_sha256(f"{timestamp}.{body}", secret)
    if not constant_time_eq(expected, sig_header.v1): raise Invalid()
```

**SFTP atomic upload**

```
upload tmp -> verify size/hash -> rename to final -> drop manifest -> poller moves to /archive on success
```

---

## Links

- Contracts & Schemas: `../40-api-and-data-contracts/contracts-and-schemas.md`  
- Pagination & Filtering: `../40-api-and-data-contracts/pagination-and-filtering.md`  
- Error Taxonomy: `../40-api-and-data-contracts/error-taxonomy.md`  
- Idempotency & Retries: `../40-api-and-data-contracts/idempotency-and-retries.md`  
- Resiliency & Timeouts: `../50-non-functional/resiliency-and-timeouts.md`  
- Privacy & Compliance: `../50-non-functional/privacy-and-compliance.md`  
- Compatibility Matrix: `../50-non-functional/compatibility-matrix.md`
+