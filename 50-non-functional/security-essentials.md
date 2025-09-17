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


# Security Essentials

> Ship fast **and** safe. This playbook gives a pragmatic baseline for **security-by-default** and **testable controls** across web, API, mobile, and jobs.

---

## TL;DR (defaults we recommend)

- **Authentication**: short-lived access tokens, refresh flow, **MFA** optional → required for sensitive ops.  
- **Authorization**: **deny-by-default**, explicit **RBAC/ABAC** rules, tenancy isolation; choose **404 vs 403** policy and be consistent.  
- **Input → Output**: validate on **boundaries**, encode on **output**; prefer allow-lists; use **message IDs** (no leaking internals).  
- **Secrets**: env/KMS-managed; keys **rotate**, never in Git; **SBOM** + dependency scanning.  
- **Transport/Storage**: TLS1.2+; `SameSite=Lax|Strict` cookies; at-rest enc via KMS; per-tenant data scoping.  
- **Web defenses**: CSP, HSTS, `X-Frame-Options`, **CSRF token** for state-changing requests, strong CORS contract.  
- **Abuse**: rate limits, CAPTCHA after threshold, idempotency for money-moving.  
- **Observability**: structured **audit logs** + **security events**; no PII in logs; redaction everywhere.

Use this page as a **gate** for features touching auth, money, PII, uploads, or cross-tenant data.

---

## Threat Modeling (quick & useful)

- **Model**: STRIDE for security (Spoofing, Tampering, Repudiation, Information disclosure, Denial of service, Elevation of privilege).  
- **Surface**: routes, roles, data classes (PII/PCI/PHI), dependencies, storage, jobs, webhooks.  
- **Abuser stories**: *“As a malicious user, I try to read another tenant’s invoice via predictable IDs.”*  
- **Decide controls**: authn, authz, input/output, storage, network, monitoring, incident hooks.

**Template**

```
Asset: <data/feature> | Tenant: <id>
Trust boundaries: <client ↔ API, API ↔ DB, API ↔ third-party>
Threats: <STRIDE bullets>
Controls: <headers, tokens, roles, validations>
Evidence: <logs, metrics, traces, audit, alerts>
```

---

## Authentication (authn)

- **Tokens**: short-lived **access** (5–15 min) + refresh (days).  
- **MFA**: time-based OTP or WebAuthn for privileged actions (refunds, exports).  
- **Sessions (cookies)**: `HttpOnly`, `Secure`, `SameSite=Lax` (or `Strict` for sensitive apps).  
- **Mobile**: use OS keychain/secure storage; device binding where possible.

**Tests**
- Expired/invalid token → `401 AUTH.invalid_credentials`.  
- Refresh flow: expired access token + valid refresh → **new** access token; audit `token.rotated`.  
- MFA enforced for sensitive endpoint → `403 POLICY.mfa.required`.

---

## Authorization (authz)

- **RBAC/ABAC**: explicit matrix (see `../30-scenario-patterns/roles-and-permissions.md`).  
- **Tenancy**: scope by `tenant_id`; decide **leak-safe 404** or explicit 403.  
- **IDOR**: never trust client IDs for ownership; verify on server.

**Tests**
- Viewer cannot **create/update/delete/export/import** → `403 AUTHZ.role.denied`.  
- Cross-tenant read blocked → `404 AUTHZ.scope.tenant` (or 403 per policy).  
- Soft-deleted update (non-admin) → `409 CONFLICT.soft_deleted`.

---

## Input Validation & Output Encoding

- **Validate** near the boundary; reject with **taxonomy** codes (see `../40-api-and-data-contracts/error-taxonomy.md`).  
- **Encode** on output (HTML/JS/URL).  
- **Allow-list** patterns: e.g., promo code `[A-Z0-9-]{1,16}`.

**Common classes**
- **XSS** (reflected/stored/DOM): escape HTML; use CSP; sanitize rich text via allow-list (e.g., DOMPurify).  
- **SQL/NoSQL/ORM injection**: parameterized queries; no string concatenation.  
- **Command/Template injection**: avoid `exec`; sandbox templates.  
- **Path traversal**: normalize paths; restrict to **chroot**/prefix.  
- **Deserialization**: avoid unsafe formats; use signed tokens; whitelist types.  
- **SSRF**: block internal IP ranges; validate URLs; metadata endpoints denied.

**Tests**
- Render user-provided strings in UI with `<script>` payload → must be **escaped**; CSP blocks inline.  
- Upload path `../../etc/passwd` → **reject**.  
- URL fields pointing to `169.254.169.254` → **blocked** (`VALIDATION.url.private_range`).

---

## Web Security Headers (baseline)

| Header | Recommended value | Why |
|---|---|---|
| **Content-Security-Policy** | at least `default-src 'self'; frame-ancestors 'none'` (+ script/style nonces) | XSS mitigation |
| **Strict-Transport-Security** | `max-age=31536000; includeSubDomains; preload` | Force HTTPS |
| **X-Frame-Options** | `DENY` (or CSP `frame-ancestors`) | Clickjacking |
| **X-Content-Type-Options** | `nosniff` | MIME sniffing |
| **Referrer-Policy** | `strict-origin-when-cross-origin` | Minimize leakage |
| **Permissions-Policy** | restrict sensors/embeds | Reduce attack surface |
| **Cross-Origin-Opener-Policy** | `same-origin` | XS-Leaks |
| **Cross-Origin-Resource-Policy** | `same-site` | Resource isolation |

**Tests**
- Verify header presence & values on protected routes.  
- CSP report-only rollout: confirm **no violations** in reports after hard-enforce.

---

## CSRF & CORS (contracts)

**CSRF**  
- For cookie-auth apps, state-changing requests require **CSRF token** (double-submit or same-origin).  
- Set cookies with `SameSite=Lax|Strict`.  
- Do **not** rely on CORS to prevent CSRF.

**CORS**  
- Explicit **allow-list** of origins.  
- `Access-Control-Allow-Credentials: true` only when needed; never with `*`.  
- Preflight handles methods/headers you actually use.

**Tests**
- Cross-site POST without CSRF token → **blocked**.  
- Preflight for `Authorization` header from allowed origin → **200** with correct headers; other origins → **403**.

---

## File Uploads & Downloads

- Validate **MIME** (from server-side sniff), **extension**, **size**; reject **polyglots**.  
- Run **AV scan**/image re-encode; strip metadata; throttle large/ZIP files (zip bombs).  
- Store outside web root; randomize filenames; signed URLs for access; short-lived.

**Tests**
- Image with embedded script (SVG with `<script>`) → sanitized or rejected.  
- Oversized upload (> limit) → `413` with clear error code.  
- ZIP bomb detection triggers rejection & audit.

---

## Secrets & Supply Chain

- Secrets in **KMS/SM**; fetched at runtime; rotate periodically; **least-privilege** IAM.  
- **Never** commit secrets; enable **secret scanning** in CI (pre-receive hooks).  
- **Dependencies**: lockfiles, **pin** versions; enable SCA (Software Composition Analysis), **SBOM** generation; verify signatures for critical deps.  
- **Builds**: reproducible, signed artifacts; container base images kept small & patched.

**Tests**
- CI fails on detected secret patterns.  
- Dependency with known CVE → build fails or quarantined with risk note.  
- SBOM present in artifacts.

---

## Transport & Storage

- **TLS**: v1.2+; strong ciphers; redirect HTTP→HTTPS; optional pinning in mobile.  
- **At rest**: encrypt with KMS-managed keys; per-tenant key option for high-sensitivity data.  
- **Backups**: encrypted; restore tested; retention/lifecycle compliant.

**Tests**
- TLS scanner reports **A** grade.  
- Backup **restore drill** produces working dataset; audit logs captured.

---

## Logging, Auditing, Privacy

- **Structured logs**: no secrets/PII; mask tokens, emails, phone numbers.  
- **Audit trail**: `actor`, `action`, `target`, `result`, `reason`, `tenant_id`, `correlation_id`.  
- **Privacy**: limit collection; delete/anonymize per policy; GDPR/CCPA requests wired (see `../30-scenario-patterns/data-lifecycle.md`).

**Tests**
- Redaction unit tests (regex/format aware).  
- Audit exists for sensitive actions (refunds, role changes, exports).

---

## Abuse Prevention

- **Rate limiting**: token bucket by **IP + user + route**; burst + sustained limits.  
- **Bot controls**: device fingerprints; CAPTCHA after threshold; login throttling; password reset throttling.  
- **Enumeration**: uniform errors for unknown emails/usernames; **no timing leak**.

**Tests**
- 100 login attempts from same IP → throttle; audit `rate_limit`.  
- Password reset for unknown email returns generic message, no timing leak beyond threshold.

---

## Dependency, Container & Infra Security

- **SAST/DAST/IAST** in CI; PR annotations.  
- **Container**: non-root user; read-only fs; health probes; seccomp/apparmor profiles.  
- **IaC**: scan Terraform/K8s manifests for dangerous settings; least privilege.  
- **Prod access**: SSO, short-lived creds, just-in-time elevation.

**Tests**
- CI fails on container running as root.  
- IaC scan flags open security groups; PR must address or waive with risk.

---

## Security MAE Scenarios (examples)

### S-001 **Main** — Login with MFA
- **Preconditions**: user has MFA enrolled  
- **Steps**: submit credentials → OTP → session established  
- **Expected**: `Set-Cookie` secure+httponly+samesite; audit `login.success`  
- **Oracles**: 200; headers; audit log; no PII in logs

### S-101 **Exception** — IDOR on resource read
- **Trigger**: user from tenant A requests `GET /orders/{id_of_B}`  
- **Expected**: `404 AUTHZ.scope.tenant`; no leakage  
- **Oracles**: response body minimal; audit denied

### S-102 **Exception** — CSRF defense
- **Trigger**: cross-site POST without CSRF token  
- **Expected**: 403/419; cookie `SameSite` prevents send or server rejects  
- **Oracles**: logs show `csrf=false`, origin mismatch

### S-103 **Exception** — XSS attempt
- **Trigger**: render `<img src=x onerror=alert(1)>`  
- **Expected**: escaped; CSP blocks inline; report-only → enforce  
- **Oracles**: CSP report, no JS execution

---

## Review Checklist (quick gate)

- [ ] Authn: token lifetimes, refresh, MFA for sensitive ops  
- [ ] Authz: **deny-by-default**, tenant isolation, IDOR tests  
- [ ] Validation/encoding: boundaries + output escaping; SSRF/path traversal defenses  
- [ ] Web headers: CSP/HSTS/XFO/XCTO/Referrer/Permissions/COOP/CORP present  
- [ ] CSRF: tokens & `SameSite` cookies; CORS allow-list exact  
- [ ] Uploads: MIME sniff + AV/scan + size limits; safe storage & signed URLs  
- [ ] Secrets: KMS-managed; scans enabled; **no secrets in Git**  
- [ ] Dependencies: SCA, SBOM, signatures; base images patched; non-root containers  
- [ ] Transport/storage: TLS1.2+, at-rest enc; backup restore tested  
- [ ] Observability: redaction; **audit** for sensitive actions  
- [ ] Abuse: rate limits; CAPTCHA threshold; enumeration-safe messages  
- [ ] CI/CD: SAST/DAST/IaC scans gate PRs; waivers recorded with risk  
- [ ] Incident hooks: alerts for authz_denied spikes, CSP violations, rate_limit

---

## CSV Seeds

**Header checks**

```csv
route,header,expected
/api/*,Content-Security-Policy,default-src 'self'
/,Strict-Transport-Security,max-age>=31536000; includeSubDomains
/,X-Frame-Options,DENY
/api/*,X-Content-Type-Options,nosniff
```

**Authz matrix (deny cases)**

```csv
operation,role,tenant_scope,expected,code
update,Viewer,own,DENY,AUTHZ.role.denied
read_by_id,Editor,cross,DENY,AUTHZ.scope.tenant
hard_delete,Editor,own,DENY,AUTHZ.role.denied
export_csv,Viewer,own,DENY,AUTHZ.role.denied
```

**Abuse cases**

```csv
case,action,rate,window,expected
login-throttle,POST /login,>10/min,5m,429 then CAPTCHA
pwd-reset-throttle,POST /password/reset,>5/min,10m,429 generic
```

---

## Templates

**Security Test Plan (feature)**

```
# Security Plan — <Feature>
Threats: <STRIDE bullets>
Controls: <authn/authz/headers/validation>
Abuse: <rate limits, CAPTCHA, enumeration>
Evidence: <logs/audit/metrics/headers>
Test set: <MAE scenarios + negative cases>
```

**CSP checklist**

```
default-src 'self'
script-src 'self' 'nonce-<dynamic>' https://trusted.cdn
style-src 'self' 'nonce-<dynamic>'
img-src 'self' data:
frame-ancestors 'none'
report-uri https://csp.example.com/report
```

---

## Links

- Roles & Permissions: `../30-scenario-patterns/roles-and-permissions.md`  
- Data Lifecycle (privacy & deletion): `../30-scenario-patterns/data-lifecycle.md`  
- Error Taxonomy (message IDs): `../40-api-and-data-contracts/error-taxonomy.md`  
- Idempotency & Retries: `../40-api-and-data-contracts/idempotency-and-retries.md`  
- Resiliency & Timeouts: `./resiliency-and-timeouts.md`  
- Performance (p95/p99): `./performance-p95-p99.md`

---

<p align="center">
  <sub>
    Built with <a href="https://treeifyai.com">Treeify</a> — AI test case design copilot.<br>
    <em>Structured, traceable, high-coverage test design — faster.</em>
  </sub>
</p>
