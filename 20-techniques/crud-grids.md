# CRUD Grids & Data Variation (Playbook)

> Many escaped bugs hide in **authorization**, **field-level rules**, **bulk ops**, and **exports/imports**—not just the simple Create/Read/Update/Delete.  
> A **CRUD grid** makes these rules explicit across **roles**, **states**, and **tenancy**, while a **data-variation grid** hardens inputs/encodings.

---

## What & Why

- **CRUD grid**: a matrix of **Operations × Roles** (and sometimes × States/Tenancy) that defines **allow/deny + expected evidence**.  
- **Data-variation grid**: a table of **input variations** (charset, lengths, normalization, locale) to catch real-world data issues.
- **Why**: AuthZ slips (e.g., export by Viewer), soft-delete loopholes, cross-tenant reads, and field-level edits are common production incidents.

**Use for**: admin panels, catalogs, user management, any RBAC/ABAC surface, multi-tenant APIs, reporting/exports, CSV imports.

---

## Steps (10-step recipe)

1. **Identify resource(s)** (e.g., `discount_code`, `product`, `user`) and their **states** (draft/active/archived/soft-deleted).  
2. **List roles** (Viewer/Editor/Admin, Service) and any **tenancy constraints** (tenant, org, project).  
3. **Enumerate operations**: **C**reate, **R**ead (by id, list, search), **U**pdate (full/partial), **D**elete (soft/hard/restore), **Export**, **Import**, **Bulk**, **Share/Assign**.  
4. Define **expected HTTP/UX outcomes** per cell (allow/deny), with **message IDs** and HTTP codes.  
5. Add **field-level rules** (immutable fields, computed fields, protected attributes).  
6. Add **concurrency**: ETag/`If-Match`, optimistic locking, idempotent create.  
7. Add **audit**: required events/fields for `who/what/when/why`, redaction rules.  
8. Layer a **data-variation grid** (charset/length/normalization) for text fields.  
9. Generate **positive + negative** tests from the grid (include **indirect routes**: bulk actions, exports, API tokens).  
10. Gate with checklists: Functional, API, Security/Privacy, Compatibility.

---

## Baseline matrix (example: `discount_code`)

**Roles**: Viewer, Editor, Admin  
**States**: draft, active, archived, soft_deleted  
**Tenancy**: scoped by `tenant_id` (cross-tenant must 404/403 per policy)

| Operation | Viewer | Editor | Admin | Notes / Expected |
|---|:---:|:---:|:---:|---|
| Create        | ❌ | ✅ | ✅ | `POST /codes` — **idempotency key** required; 201 |
| Read by id    | ✅ (own tenant) | ✅ (own tenant) | ✅ | 404 for cross-tenant; **mask secrets** |
| List/Search   | ✅ (own tenant) | ✅ (own tenant) | ✅ | Cursor pagination; stable sort `(created_at,id)` |
| Update (PATCH)| ❌ | ✅ (fields: name, expiry) | ✅ (all allowed) | **Immutable**: `tenant_id`, `created_by`, `type` |
| Delete (soft) | ❌ | ✅ | ✅ | `DELETE` → marks `soft_deleted=true`; 204 |
| Restore       | ❌ | ✅ | ✅ | `POST /codes/{id}:restore` |
| Hard delete   | ❌ | ❌ | ✅ | `DELETE?force=true` — admin only; audit reason required |
| Export CSV    | ❌ | ✅ (own tenant) | ✅ | File name tags; row limit; PII redaction |
| Import CSV    | ❌ | ✅ | ✅ | Schema checks; partial failure report |
| Bulk update   | ❌ | ✅ | ✅ | Atomic per item; overall 207 Multi-Status |
| Share/Assign  | ❌ | ✅ | ✅ | Only within tenant unless admin |

**Deny semantics**:  
- **Viewer** trying to update → `403 AUTHZ.role.denied` (message ID mapped in `../40-api-and-data-contracts/error-taxonomy.md`)  
- **Cross-tenant** read → `404` (tenant-leak-safe) or `403` per policy; be consistent.

---

## Field-level rules (example)

| Field         | Create | Update (Editor) | Update (Admin) | Notes |
|---------------|:-----:|:----------------:|:--------------:|------|
| `code`        | set   | ❌               | ❌             | Immutable after creation |
| `type`        | set   | ❌               | ❌             | Immutable (affects persistence & rules) |
| `name`        | set   | ✅               | ✅             | Trim; NFC normalize |
| `expiry_at`   | set   | ✅               | ✅             | Must be ≥ now; timezone-safe |
| `created_by`  | auto  | ❌               | ❌             | From auth token |
| `tenant_id`   | auto  | ❌               | ❌             | From auth/route |
| `soft_deleted`| auto  | auto            | auto           | Only via endpoints (delete/restore) |

---

## Concurrency & contracts

- **Create is idempotent**: require `Idempotency-Key`; **same key ⇒ same resource**. See `../40-api-and-data-contracts/idempotency-and-retries.md`.  
- **Update uses ETag**: require `If-Match: <etag>`; if mismatch → `412 Precondition Failed`.  
- **List is stable**: sort by `(created_at,id)`; assert **no duplicates/skips** across pages.  
- **Soft delete is absorbing** for non-admin update paths: PATCH/PUT on soft-deleted returns `409 CONFLICT.soft_deleted`.

---

## Data-variation grid (text inputs)

| Dimension     | Variations                       | Example            | Expectation |
|---------------|----------------------------------|--------------------|-------------|
| Charset       | ASCII, Latin-1, UTF-8 (emoji)    | `SALE`, `Café`, `🎉SAVE` | Accept/reject per policy; storage round-trips |
| Length        | min−1, min, typical, max, max+1  | 0, 1, 8, 16, 17    | Min/max policy enforced |
| Whitespace    | leading/trailing, inner          | `" SAVE"`, `"SA VE"` | Trim edges; reject inner |
| Case          | lower/mixed/upper                | `save10`           | Normalize/store uppercase |
| Normalization | NFC/NFD                           | `é` vs `e+́`       | Equal per normalization rule |
| Locale        | en-US, fr-FR, zh-CN              | messages/dates     | Localized messages if applicable |

---

## Test selection (positive + negative)

Pick **2–3 critical operations** and design **allow/deny** pairs per role × state × tenancy.

**Examples (extract)**

1. **Editor creates code (idempotent)**  
   - Request twice with same `Idempotency-Key` → **201 then 200** (same resource).  
   - Oracle: response ID stable; audit log `action=create`, `key=<k>`.

2. **Viewer attempts update**  
   - PATCH name → **403 AUTHZ.role.denied**; no DB delta; audit entry for attempt.

3. **Cross-tenant read**  
   - Viewer from tenant A requests code from tenant B → **404** (or 403).  
   - Oracle: no leakage in error body; audit includes `target_tenant_id` masked.

4. **ETag conflict**  
   - Get resource → PATCH with stale `If-Match` → **412**; suggest re-fetch.

5. **Soft delete & restore**  
   - DELETE (soft) → list excludes item by default; filter `include_deleted=true` shows it.  
   - Restore endpoint brings it back; **hard delete** only admin + reason.

6. **Export CSV privacy**  
   - Editor export → **no PII fields**; filename includes `tenant_id` stamp; size/row limits enforced.

7. **Bulk update 207**  
   - Submit mixed-good/bad items → success items applied; failures report detailed codes per row.

---

## Oracles & Evidence

- **API**: status codes (`201/200/403/404/409/412/207`), schema contracts (OpenAPI), message IDs.  
- **DB**: row/field deltas; soft-delete flag changes; `updated_at` version increments.  
- **Audit log**: `actor`, `action`, `target`, `before/after` (redacted), `reason`, `correlation_id`.  
- **Metrics**: counts for `authz_denied`, `idempotent_replay`, `etag_conflict`, `bulk_partial`.  
- **Traces**: spans for `create/update/delete/export/import` with attributes (role, tenant, op).

---

## Anti-patterns

- Only testing the **direct** path; ignoring **indirect routes** (bulk, exports, API tokens).  
- No **field-level** rules (immutable/computed fields silently editable).  
- Missing **tenancy** checks → cross-tenant data leaks.  
- **Soft delete** without **list filters** or restore path tests.  
- No **idempotency** for create; **no ETag** for update.  
- Export/import that bypasses **audit** or **privacy redaction**.

---

## Review checklist (quick gate)

- [ ] Matrix covers **roles × operations × states** with expected **allow/deny**  
- [ ] **Tenancy** behavior defined (own tenant vs cross-tenant)  
- [ ] **Field-level** immutability/computed rules specified  
- [ ] **Idempotent create** and **ETag/If-Match** for updates  
- [ ] **Soft delete/restore/hard delete** policies tested  
- [ ] **Indirect routes** (bulk/export/import) covered  
- [ ] **Data-variation grid** applied to text fields  
- [ ] **Audit/metrics/traces** oracles defined  
- [ ] Error taxonomy → message IDs linked

---

## CSV seeds

**AuthZ matrix**

```csv
operation,role,state,tenant_scope,expected,code
create,Viewer,*,own,403,AUTHZ.role.denied
create,Editor,*,own,201,OK
read_by_id,Viewer,*,cross,404,AUTHZ.scope.tenant
update,Editor,active,own,200,OK
update,Editor,soft_deleted,own,409,CONFLICT.soft_deleted
delete_soft,Editor,active,own,204,OK
delete_hard,Admin,*,own,204,OK
export_csv,Editor,*,own,200,OK
import_csv,Editor,*,own,207,MULTI_STATUS
```

**Field rules**

```csv
field,create,update_editor,update_admin,notes
code,set,deny,deny,immutable
type,set,deny,deny,immutable
name,set,allow,allow,trim+nfc
expiry_at,set,allow,allow,>= now
tenant_id,auto,deny,deny,from_auth
```

**Data variation**

```csv
field,klass,input,expected
name,min_len,"A",OK
name,max_len,"A"*64,OK
name,above_max,"A"*65,VALIDATION.name.length.exceeds
name,emoji,"Summer 🎉",OK
name,inner_space,"SAVE CODE",VALIDATION.name.charset
name,trim," SAVE ",OK_store_trimmed
name,nfd,"Cafe\u0301",OK_equal_to_"Café"
```

---

## Templates

**CRUD grid**

```
| Operation | Viewer | Editor | Admin | Notes |
|---|:---:|:---:|:---:|---|
| Create |  |  |  |  |
...
```

**Field rules**

```
| Field | Create | Update (Editor) | Update (Admin) | Notes |
|------|:-----:|:---------------:|:--------------:|------|
```

**Data variation**

```
| Dimension | Variations | Example | Expectation |
|-----------|------------|---------|-------------|
```

---

## Links

- Scenario pattern — Roles & Permissions: `../30-scenario-patterns/roles-and-permissions.md`  
- Error Taxonomy → UX: `../40-api-and-data-contracts/error-taxonomy.md`  
- Idempotency & Retries: `../40-api-and-data-contracts/idempotency-and-retries.md`  
- Pagination/Filtering: `../40-api-and-data-contracts/pagination-and-filtering.md`  
- Privacy & Compliance: `../50-non-functional/privacy-and-compliance.md`  
- Compatibility / i18n data grid: `../50-non-functional/compatibility-matrix.md`, `../55-domain-playbooks/i18n-l10n.md`  
- Review gates & metrics: `../65-review-gates-metrics-traceability/*`
