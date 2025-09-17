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


# Privacy & Compliance

> Privacy is a **product feature** and a **team habit**. This playbook turns regulations and policies into **practical, testable work** you can ship: data maps, retention rules, DSRs, consent, cross-border controls, and evidence.

> **Note**: This is guidance, not legal advice. Align with your counsel and your latest policy.

---

## TL;DR (defaults we recommend)

- **Data map first**: know **what** personal data you collect, **why**, **where it flows**, and **who** processes it.  
- **Minimize**: collect only what you need; set **retention** clocks; default to **opt-in** for optional data.  
- **Consent & preferences**: record **who/what/when/how**; enforce across products, not just the web.  
- **DSRs** (Data Subject Requests): **verify identity**, **scope**, run **find/fulfill/verify**, and **log** the outcome.  
- **Deletion semantics**: choose per class: **soft delete**, **hard delete**, or **anonymize**—and purge **derived** copies.  
- **Cross-border**: document **locations**, legal **transfer mechanism**, and processor **agreements**.  
- **Evidence** everywhere: audit logs, change logs, DPIA/PIA docs, redaction tests, and runbooks.

---

## 1) Data classes (taxonomy)

Classify once; reuse in code, schemas, and reviews.

| Class | Examples (short) |
|---|---|
| **PII** | name, email, phone, address, IP |
| **Sensitive PII** | national ID, SSN, passport, health, biometrics |
| **Financial** | card tokens, bank last4, invoice data |
| **Auth** | passwords (hashed), session IDs, refresh tokens |
| **Telemetry** | device, browser, IP, events |
| **Content** | user files, chat logs, images |
| **Derived** | embeddings, features, analytics aggregates |
| **Operational** | logs, metrics, traces with IDs |

**Rule**: default to **PII** unless proven otherwise; treat **IP** as personal data.

---

## 2) Legal bases & notices (policy-to-product)

- **Purpose**: state **why** you collect each field; bind to a **legal basis** (e.g., contract, consent, legitimate interest).  
- **Notice**: publish concise **privacy notices** at collection points (forms, SDKs, APIs).  
- **Changes**: version notices and keep **diff history**.

**Tests**  
- Each field in a schema maps to **purpose + basis**.  
- UI/API shows a **just-in-time** notice for optional data.

---

## 3) Consent & preferences

- **Granular**: marketing, analytics, personalization, 3rd‑party sharing.  
- **Record**: subject, scope, version of notice, **timestamp**, channel (web/app/API).  
- **Respect**: propagate to **SDKs**, **tags**, **emails**, **data lake**; prefer **server-side** tagging.  
- **Children/Teens**: age gates + parental consent when applicable.

**Tests**  
- Revoke analytics → SDK calls drop; tags disabled.  
- Marketing opt-out → email system blocks sends.

---

## 4) Data map & flows

- Inventory **systems** (apps, DBs, queues, lakes), **stores** (S3, BigQuery), **processors** (vendors) and **flows** (API/event).  
- Track **locations** (region), **replication**, **backups**, **encryption**, and **retention**.

**Artifacts**  
- **Data map** diagram (systems ↔ flows).  
- **Record of processing**: purpose, basis, recipients, retention, security controls.

---

## 5) Retention & deletion

- Define **clocks** per class (e.g., invoices ≥ X years; telemetry ≤ Y days).  
- Pick semantics per class: **soft** (restore window), **hard** (purge), **anonymize** (keep links).  
- Propagate to **derived** stores (index/cache/warehouse/lake) and **backups**.

**Tests**  
- Deletion removes **search index** docs and **cache** keys.  
- Anonymization keeps **referential integrity** (orders still link to `anon_user`).

> Pair with `../30-scenario-patterns/data-lifecycle.md` for jobs, flags, evidence, and recovery.

---

## 6) Data Subject Requests (DSRs)

Types commonly supported:
- **Access** (export), **Rectification**, **Erasure** (delete/anonymize), **Portability**, **Restriction**, **Objection**, **Marketing opt-out**.

**Workflow**
1. **Verify identity** (and authority).  
2. **Scope** (account, tenant, timeframe, identifiers).  
3. **Find** across **primary + derived** stores; include **backups policy**.  
4. **Fulfill** (export/delete/anonymize) **idempotently**.  
5. **Verify** evidence; log **who/when/how**.  
6. **Respond** within your **policy window**; escalate edge cases.

**Tests**  
- Erasure leaves analytics rows **de-identified**; no direct identifiers remain.  
- Export bundle contains **all fields** for the scope; checksum + format documented.

---

## 7) Cross-border transfers

- Document **data locations** and **processors**.  
- Use an approved **transfer mechanism** where required (e.g., SCCs/approved contracts).  
- Apply **regionalization** where promised (e.g., “EU data stays in EU”).  
- Keep a **vendor register** with **DPAs**, purpose, and data classes processed.

**Tests**  
- Requests from region `X` are served from allowed **regions** only (headers/logs).  
- Vendor list shows **current** contracts and **data classes**.

---

## 8) Privacy by design (practical controls)

- **Minimization**: drop unused fields; default fields **off**.  
- **Pseudonymization**: replace direct IDs with **surrogates** where possible.  
- **Encryption**: in transit (TLS1.2+), at rest (KMS); key rotation.  
- **Access**: **least privilege**; short-lived roles; audit on access to PII.  
- **Redaction**: logs/metrics/traces strip PII by **policy** (format-aware).  
- **Testing**: add **privacy gates** to PRs.

---

## 9) DPIA/PIA (impact assessments)

Run when adding **new high-risk processing**, sensitive classes, or large-scale profiling.

**DPIA contents (short)**
- Purpose, legal basis, data classes, subjects, volumes  
- Risks (collection, storage, access, transfer, sharing)  
- Controls (auth/authz, encryption, retention, DSRs, consent)  
- Residual risk & decision; review schedule

**Test**  
- DPIA exists for flagged features and is **linked in PR**.

---

## 10) Cookies, SDKs, and tracking

- Categorize cookies/SDKs: **strictly necessary**, **functional**, **analytics**, **marketing**.  
- **SameSite**: Lax/Strict for session; secure; short lifetimes.  
- **Consent** gates SDK initializations and event sends; honor **Do Not Track** where applicable.

**Tests**  
- Without consent: only **necessary** cookies set; analytics **off**.  
- With consent revoke: event volume drops; tags disabled.

---

## 11) Logging & redaction

- Ban PII/secrets in logs; emit **message IDs** rather than raw text (see error taxonomy).  
- Redact **email, phone, tokens, addresses, exact locations**; hash where needed.  
- Keep **sampling** for heavy endpoints; never sample **security** events.

**Tests**  
- Regex & format-aware **unit tests** for redaction filters.  
- Golden logs from e2e do **not** contain PII.

---

## 12) Incident response (privacy events)

- **Detect** (alerts, anomaly, reports) → **triage** → **contain** → **eradicate** → **recover** → **learn**.  
- Keep a **runbook**: roles, contacts, decision tree, comms templates, post-incident review.  
- Determine if it is a **notifiable incident** per your policy and law; follow your **notification** process.

**Tests**  
- Dry run: simulate an exposure; verify **timeline**, **approvals**, **communications** artifacts.

---

## 13) MAE scenarios (examples)

### S-001 **Main** — Consent capture & enforcement
- **Preconditions**: user toggles analytics consent **on**  
- **Steps**: banner accept → SDK initializes → events flow  
- **Expected**: consent log written; analytics events tagged with consent version  
- **Oracles**: DB consent row; event headers; tag manager state

### S-101 **Exception** — DSR erasure with derived stores
- **Preconditions**: user with orders, analytics, search index  
- **Steps**: verify → delete/anonymize → reconcile derived stores  
- **Expected**: primary rows removed or anonymized; index/cache/warehouse purged or de-identified  
- **Oracles**: row absence; index doc missing; warehouse de-identified

### S-102 **Exception** — Cross-border policy breach attempt
- **Steps**: request triggers transfer to disallowed region  
- **Expected**: blocked with policy error; audit entry  
- **Oracles**: response code; audit log; region headers

---

## 14) Review checklist (quick gate)

- [ ] **Data map** complete: systems, stores, processors, flows, regions  
- [ ] Each field maps to **purpose + legal basis**; notices versioned  
- [ ] **Consent** recorded, enforced, and revocable across channels  
- [ ] **Retention** clocks set; deletion/anonymization cascades to **derived** stores & backups policy documented  
- [ ] **DSR** runbook & SLAs; identity verification steps; evidence logged  
- [ ] **Cross-border** registers & transfer mechanism documented  
- [ ] **Privacy by design** controls in code (minimize, pseudo, encrypt, least privilege)  
- [ ] **DPIA/PIA** completed where required; linked in PRs  
- [ ] **Cookies/SDKs** gated by consent; storage settings sane  
- [ ] **Logging** redacts PII; message IDs used; sampling policy  
- [ ] **Incident** runbook & drills performed; notification workflow ready

---

## 15) CSV seeds

**Data inventory**

```csv
system,store,region,data_class,retention_days,processor
webapp,Postgres,SG,PII,365,self
events,BigQuery,US,Telemetry,90,vendor-analytics
storage,S3,EU,Content,730,cloud-provider
search,OpenSearch,SG,Derived,30,self
backups,S3-Glacier,SG,All,3650,cloud-provider
```

**Consent log**

```csv
subject_id,scope,version,channel,timestamp,granted
u_123,analytics,1.4,web,2025-09-16T12:00:01Z,true
u_123,marketing,1.0,email,2025-09-16T12:00:02Z,false
```

**DSR tracker**

```csv
request_id,subject_id,type,status,opened_at,closed_at,evidence
dsr_001,u_123,erasure,closed,2025-09-01T10:00:00Z,2025-09-02T14:00:00Z,link://artifact
dsr_002,u_456,access,open,2025-09-10T09:30:00Z,,link://artifact
```

**Retention policy**

```csv
data_class,delete,anonymize,soft_delete_window_days,notes
Telemetry,after 90d,,0,rollup before delete
PII,after 365d,if linked to invoices,30,see finance policy
Content,,after 730d,,tokenize owner id
```

---

## 16) Templates

**DPIA/PIA**

```
Feature: <name>
Owner: <team>  Version: <v>
Purpose & legal basis: <short>
Data subjects & classes: <short>
Locations & processors: <short>
Risks: <bullets>
Controls: <bullets>
Residual risk: <low/med/high>  Decision: <proceed/mitigate/block>
Review: <date cadence>
```

**DSR runbook**

```
Type: <access/rectify/erase/portability>
Verify: <steps>
Scope: <systems, timeframe, identifiers>
Fulfill: <export/delete/anonymize>
Evidence: <bundle, checksums, logs>
Response: <template + window>
```

**Data map (table skeleton)**

```csv
system,endpoint/event,fields,purpose,basis,processor,region,retention_days
```

---

## Links

- Data Lifecycle: `../30-scenario-patterns/data-lifecycle.md`  
- Error Taxonomy (message IDs for forms & APIs): `../40-api-and-data-contracts/error-taxonomy.md`  
- Contracts & Schemas (mark PII, nullability, units): `../40-api-and-data-contracts/contracts-and-schemas.md`  
- Security Essentials (headers, authn/z, secrets): `./security-essentials.md`  
- Resiliency & Timeouts (brownouts vs strict failures): `./resiliency-and-timeouts.md`  
- Performance p95/p99 (budgets for encryption/calls): `./performance-p95-p99.md`

---

<p align="center">
  <sub>
    Built with <a href="https://treeifyai.com">Treeify</a> — AI test case design copilot.<br>
    <em>Structured, traceable, high-coverage test design — faster.</em>
  </sub>
</p>
