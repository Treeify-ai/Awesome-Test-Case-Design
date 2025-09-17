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


# Roles & Permissions (Pattern)

> Authorization bugs are some of the **most expensive** to fix and the **easiest** to miss.  
> This pattern turns role/permission requirements into **clear scenarios**, a **matrix**, and a small set of **allow/deny** tests that close common gaps (IDOR, cross-tenant leaks, admin bypass, indirect routes).

---

## What & Why

- **RBAC** (Role-Based Access Control): capabilities determined by role (Viewer/Editor/Admin).  
- **ABAC** (Attribute-Based): decisions use attributes like `owner_id`, `tenant_id`, `region`, `labels`.  
- **Horizontal vs Vertical** authZ:
  - **Horizontal**: access your **own** resource vs **someone else’s** (IDOR).  
  - **Vertical**: capability based on privilege level (Viewer vs Admin).  
- **Least privilege**: minimum rights needed; **break-glass** admin with justification.  
- **Tenancy**: isolate by `tenant_id/org_id/project_id`; decide **404 vs 403** policy for cross-tenant access.

Why this pattern: convert requirements into an **Operations × Roles × Context** matrix, then into **MAE scenarios** with explicit **oracles** and **audit** evidence.

---

## Quick workflow (7 steps)

1. **List resources & operations** (C/R/U/D, list/search, export, import, bulk, share/assign).  
2. Define **roles** (RBAC) and **attributes** (ABAC) that affect decisions (owner, tenant, status).  
3. Decide **deny semantics**: `403` vs `404` for cross-tenant; consistent message IDs.  
4. Fill a **CRUD × Roles** matrix, then add **ownership/tenancy/state** notes.  
5. Add **indirect routes** (exports, bulk ops, webhooks, API tokens, public links).  
6. Write **MAE scenarios**: 1 Main, 2–3 Alternative, 3–5 Exception/negative with recovery.  
7. Add **oracles**: HTTP code + message ID, UI state, **audit log** event, and privacy-safe error bodies.

---

## Baseline matrix (example: `discount_code`)

**Roles**: Viewer, Editor, Admin  
**Context attributes**: `tenant_id`, `owner_id`, `state ∈ {draft, active, archived, soft_deleted}`

| Operation                 | Viewer            | Editor                 | Admin                  | Notes |
|--------------------------|-------------------|------------------------|------------------------|------|
| Create                    | ❌                | ✅ own tenant          | ✅ any tenant          | Idempotent create (key required) |
| Read by id (own tenant)   | ✅ (own)          | ✅ (own)               | ✅ (any)               | Mask secrets on all roles |
| Read by id (cross tenant) | ❌ → 404/403      | ❌ → 404/403           | ✅                     | Pick a policy and stick to it |
| List/Search               | ✅ (own)          | ✅ (own)               | ✅ (any)               | Cursor pagination; stable sort |
| Update (PATCH)            | ❌                | ✅ own tenant, allowed fields | ✅ allowed fields    | Immutable fields enforced |
| Delete (soft)             | ❌                | ✅ own tenant          | ✅ any tenant          | `soft_deleted=true`; 204 |
| Restore                   | ❌                | ✅ own tenant          | ✅ any tenant          | via `:restore` endpoint |
| Hard delete               | ❌                | ❌                     | ✅                    | Audit reason required |
| Export CSV                | ❌                | ✅ own tenant          | ✅ any tenant          | PII redaction; filename tags |
| Import CSV                | ❌                | ✅ own tenant          | ✅ any tenant          | Partial failure report (207) |
| Share/Assign              | ❌                | ✅ within tenant       | ✅ cross-tenant (policy) | Log grantee |

**Deny semantics examples**
- Role denied → `403 AUTHZ.role.denied`  
- Cross-tenant → **choose** `404 AUTHZ.scope.tenant` (leak-safe) **or** `403 AUTHZ.scope.tenant` (explicit)

Map codes to copy in `../40-api-and-data-contracts/error-taxonomy.md`.

---

## Scenarios (MAE) — Roles & Permissions

> Keep each step **observable** (response codes, UI states, audit events).

### S-001 **Main** — Editor updates name in own tenant
- **Preconditions**: role=Editor; `tenant_id=A`; resource belongs to `A`; not soft-deleted
- **Trigger**: `PATCH /codes/{id}` with `name="Spring Sale"`
- **Expected**: `200 OK`; field updated; ETag changes; audit `{actor, action:update, target:id, before/after}`
- **Oracles**: resp `200`; DB delta; audit entry; trace span `update.code`

### S-002 **Alt** — Admin restores soft-deleted resource
- **Preconditions**: role=Admin; resource `soft_deleted=true`
- **Trigger**: `POST /codes/{id}:restore`
- **Expected**: `200 OK`; `soft_deleted=false`; audit `restore`
- **Oracles**: resp `200`; DB delta; audit `restore`

### S-003 **Alt** — Editor export within tenant
- **Preconditions**: role=Editor; tenant=A
- **Trigger**: `GET /codes:export?tenant=A`
- **Expected**: `200 OK` file; no PII columns; filename `codes_A_YYYYMMDD.csv`
- **Oracles**: resp schema; sampled file; audit `export` w/ `tenant_id=A`

### S-101 **Exception/Vertical** — Viewer attempts update
- **Trigger**: `PATCH /codes/{id}` by Viewer
- **Expected**: `403 AUTHZ.role.denied`; **no DB change**
- **Recovery**: escalate request to Editor/Admin with approval
- **Oracles**: resp `403` + message ID; audit **denied** event with `actor_role=Viewer`

### S-102 **Exception/Horizontal (IDOR)** — Editor reads other tenant’s resource
- **Preconditions**: Editor in tenant=A; resource belongs to tenant=B
- **Trigger**: `GET /codes/{id_of_B}`
- **Expected**: `404 AUTHZ.scope.tenant` (or `403` per policy); no leakage (no foreign IDs in body)
- **Recovery**: request cross-tenant grant (if policy allows)
- **Oracles**: resp code; empty body; audit **denied** with `target_tenant=B`

### S-103 **Exception/State** — Update soft-deleted resource (non-admin)
- **Preconditions**: Editor; resource `soft_deleted=true`
- **Trigger**: `PATCH /codes/{id}`
- **Expected**: `409 CONFLICT.soft_deleted`; suggest `:restore`
- **Oracles**: resp `409`; audit **blocked** with reason

### S-104 **Exception/Indirect** — Share outside tenant by Editor
- **Trigger**: `POST /codes/{id}:share {grantee_tenant:B}`
- **Expected**: `403 AUTHZ.scope.tenant` unless Admin; audit **attempt**
- **Oracles**: resp `403`; audit entry captured

### S-105 **Exception/Token Scope** — API token missing scope
- **Preconditions**: service token without `codes:write`
- **Trigger**: `PATCH /codes/{id}`
- **Expected**: `403 AUTHZ.scope.token`; www-authenticate hint
- **Oracles**: resp `403`; logs with `token_scopes`; audit **denied**

---

## Negative checklist (must-have denies)

- [ ] Viewer cannot **create/update/delete/export/import**  
- [ ] Editor cannot access **cross-tenant** resources (404/403 policy)  
- [ ] Editor cannot **hard delete**  
- [ ] Editor cannot update **immutable fields**  
- [ ] Soft-deleted resources: only **restore** allowed (non-admin)  
- [ ] API tokens respect **scope/claims** (`aud`, `exp`, `sub`, `tenant`)  
- [ ] **Indirect** paths also denied (exports, bulk, webhooks, signed links)  
- [ ] **UI** mirrors backend decisions (buttons hidden/disabled with reason)

---

## Oracles & Evidence

- **HTTP**: status (`200/201/204/207/403/404/409/412`), **message IDs**.  
- **Audit**: `actor_id`, `role`, `token_sub`, `tenant_id`, `action`, `target`, `result`, `reason`, `correlation_id`.  
- **DB**: ownership/tenant columns; field deltas; `soft_deleted` flags.  
- **Logs**: structured keys for `authz_result`, `rule_id`, `scope`, `owner_match`, `tenant_match`.  
- **Traces/Metrics**: spans tagged with role/tenant; counters for `authz.denied`.

---

## Anti-patterns

- **Front-end only** enforcement (disabled buttons) without backend checks.  
- Inconsistent **deny** (`403` sometimes, `404` other times) for the same policy.  
- **Leaky errors**: returning foreign IDs or hinting existence across tenants.  
- No **audit** on denies or admin actions.  
- **Superuser** tokens not time-bound or justification-free.  
- Ignoring **indirect** routes (exports/bulk/webhooks/signed URLs).

---

## Review checklist (quick gate)

- [ ] CRUD × Roles matrix complete with **allow/deny** and notes  
- [ ] **Ownership/tenancy** rules explicit; `404/403` policy chosen and consistent  
- [ ] **Immutable fields** and state-specific rules covered  
- [ ] **Indirect routes** (export/import/bulk/webhook/signed link) tested  
- [ ] **Token scopes/claims** enforced; expiration and audience checked  
- [ ] **Audit** events for both allow and deny  
- [ ] **Error taxonomy** message IDs linked to copy  
- [ ] UI reflects backend decisions (no “phantom” actions)

---

## CSV seeds

**Matrix-driven tests**

```csv
id,operation,role,tenant_scope,state,expected,code,notes
T-001,create,Viewer,own,*,DENY,AUTHZ.role.denied,
T-002,create,Editor,own,*,ALLOW,OK,"idempotent create"
T-003,read_by_id,Editor,cross,*,DENY,AUTHZ.scope.tenant,"404 or 403 per policy"
T-004,update,Editor,own,soft_deleted,DENY,CONFLICT.soft_deleted,"suggest restore"
T-005,delete_hard,Editor,own,*,DENY,AUTHZ.role.denied,
T-006,delete_hard,Admin,own,*,ALLOW,OK,"audit reason required"
T-007,export_csv,Editor,own,*,ALLOW,OK,"PII redaction"
T-008,share,Editor,cross,*,DENY,AUTHZ.scope.tenant,
T-009,update,Token(codes:read),own,*,DENY,AUTHZ.scope.token,"missing write scope"
```

**Ownership/IDOR**

```csv
id,actor_role,actor_tenant,resource_tenant,expected,code
IDOR-01,Editor,A,B,DENY,AUTHZ.scope.tenant
IDOR-02,Viewer,A,A,DENY,AUTHZ.role.denied
IDOR-03,Admin,A,B,ALLOW,OK
```

---

## Templates

**CRUD × Roles**

```
| Operation | Viewer | Editor | Admin | Notes |
|---|:---:|:---:|:---:|---|
| Create |  |  |  |  |
...
```

**Scenario card**

```
## S-<nnn> <Type> — <short name>
Preconditions: <role, tenant, state>
Trigger: <call/action>
Expected: <code + message + state change>
Oracles: <HTTP + audit + DB/log/trace>
Recovery (if Exception): <...>
```

---

## Links

- CRUD & Data Variation: `../20-techniques/crud-grids.md`  
- Error Taxonomy → UX: `../40-api-and-data-contracts/error-taxonomy.md`  
- Idempotency & Retries: `../40-api-and-data-contracts/idempotency-and-retries.md`  
- Privacy & Compliance: `../50-non-functional/privacy-and-compliance.md`  
- Review gates & metrics: `../65-review-gates-metrics-traceability/*`

---

<p align="center">
  <sub>
    Built with <a href="https://treeifyai.com">Treeify</a> — AI test case design copilot.<br>
    <em>Structured, traceable, high-coverage test design — faster.</em>
  </sub>
</p>
