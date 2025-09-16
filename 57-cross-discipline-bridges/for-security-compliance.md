# For Security/Compliance — Threat → Test Mapping

> Security & privacy are **requirements**, not vibes.  
> This bridge shows how to turn threats and obligations into **concrete tests**, **evidence**, and **ship gates** that dev, QA, and GRC can all use.

---

## TL;DR (defaults we recommend)

- Maintain a living **Threat Register** with **STRIDE** + **OWASP** tags and a **test per threat**.  
- Define **policy-as-code** gates (authz, secrets, headers, encryption) and assert them in CI.  
- Build **abuse cases** alongside user stories; treat them as **Exception** flows in MAE.  
- Standardize **message IDs** for security errors and log **no PII/secrets**.  
- Capture **evidence** on every PR: scans, headers, CSP reports, authz matrix, trace IDs.  
- Tie controls to **frameworks** (SOC 2, ISO 27001, PCI, GDPR) with clear acceptance.

---

## What to test (security-in-one-page)

1. **Identity & session** — authn, MFA, session flags, logout, rotation.  
2. **Authorization** — RBAC/ABAC, tenant isolation, **IDOR**.  
3. **Input & output** — XSS, SQLi/NoSQLi, SSRF, template injection, file upload.  
4. **Headers & TLS** — HSTS, CSP, frame-ancestors, referrer, TLS config.  
5. **Secrets & config** — storage, rotation, env exposure, third-party keys.  
6. **Rate limiting & abuse** — brute force, enumeration, spam, webhook storms.  
7. **Data protection** — at-rest, in-transit, key mgmt, backups, deletion.  
8. **Privacy** — consent, purpose limitation, DSRs, tracking controls.  
9. **SaaS posture** — S3/Blob perms, least privilege, audit, logging redaction.  
10. **Supply chain** — SCA, signatures, integrity, provenance.

---

## Threat → Test Map (starter)

| Threat tag | Attack surface | Test oracle |
|---|---|---|
| **IDOR** | object `/:id` | forbid cross-tenant access; 403 + `ERR.AUTHZ.idor` |
| **CSRF** | state-changing POST | token present + origin checks |
| **XSS** | templating, rich text | no script execute; CSP report 0 |
| **SSRF** | URL fetchers | egress denylist; metadata IP blocked |
| **SQLi/NoSQLi** | query builders | no error leakage; paramized queries |
| **Auth bypass** | flags, cookies | `HttpOnly; Secure; SameSite` + rotation |
| **Brute force** | login/OTP | 429 + cooldown; no timing oracle |
| **Enumeration** | signup/reset | generic errors; same response time |
| **File upload** | media/docs | content-type enforce; AV scan; no path traversal |
| **Webhook replay** | inbound hooks | HMAC timestamp window; dedupe inbox |

> Keep tables short; keep details below.

---

## Identity, session, and auth

- Cookies: `HttpOnly; Secure; SameSite=Lax|Strict`; rotation on privilege change.  
- Session fixation: new session on login; invalidate on logout.  
- MFA/OTP: throttle; code replay invalid; codes never logged.  
- OAuth/OIDC/SAML: validate issuer/audience/nonce; clock skew.  
- Device trust: bind tokens to device fingerprints (policy-based).

**Acceptance signals**  
- Headers present; cookie flags; `ERR.AUTHN.*` consistent; traces show authn spans.

---

## Authorization & tenant isolation

- RBAC matrix per route/resource; ABAC checks for tenant/org/project.  
- IDOR checks: path/query/body IDs must match caller’s scope.  
- Stored procedures / service layer enforce checks (not just UI).  
- Admin boundaries: scoped admin vs global; audit privileged actions.

**Tests**  
- Cross-tenant GET/PUT/DELETE → 403 with `ERR.AUTHZ.scope`.  
- List endpoints filter to scope; no count leaks.

---

## Input, output & templating

- Server-side encoding; no HTML injection; rich-text sanitized.  
- SQL/NoSQL with parameterization; no string concatenation.  
- SSRF defenses: block loopback/metadata IP ranges; DNS pinning where needed.  
- File upload: size/type/extension whitelist; store outside web root; AV scan; image processing sandbox.

**Evidence**  
- Security test harness logs; CSP violation report counts; ZAP/Burp scan diff.

---

## Headers, TLS & CSP (web)

- **HSTS** (include subdomains, preload if ready).  
- **CSP** default `script-src 'self'` + hashed/nonce; `frame-ancestors 'none'` unless embedding needed.  
- **Referrer-Policy** `strict-origin-when-cross-origin`.  
- **X-Content-Type-Options** `nosniff`; **X-Frame-Options** `DENY` (or CSP frame-ancestors).  
- TLS: modern ciphers; redirect HTTP→HTTPS; OCSP stapling.

**Acceptance**  
- `/security/headers` endpoint snapshot stored per release.

---

## Rate limiting, bot & abuse controls

- Per-IP/Per-user/Per-tenant buckets; **token bucket** with burst.  
- **Enumeration** defenses: same error body/time for exists vs not.  
- **CAPTCHA** where justified; never block critical recovery routes for legit users.  
- Webhooks: per-partner limits; backoff; dead-letter.

---

## Secrets, keys & config

- Secrets in **KMS/Secrets Manager**; never in repo or logs.  
- Rotation schedule; break-glass policy; audit reads.  
- Outbound creds (SMTP, PSP, S3) scoped minimal.  
- Config schema with validation; feature flags have **safe defaults**.

**Scans**  
- Secrets scanning in CI; dependency SCA for vulns; container baseline scan.

---

## Data protection & privacy

- **At-rest**: DB/disk/S3 encrypted; key rotation plan.  
- **In-transit**: TLS 1.2+; mTLS for sensitive internal paths.  
- **Backups**: encrypted, tested restores; retention matched to policy.  
- **Deletion**: hard/soft/anonymize semantics; purge **derived** copies.  
- **Consent**: stored with scope/version; enforced in analytics/marketing.  
- **DSRs**: export/erasure processes idempotent; evidence recorded.

Link: `../50-non-functional/privacy-and-compliance.md`.

---

## Cloud/SaaS posture (quick list)

- Buckets/Containers private by default; public-only with review.  
- IAM least-privilege; access keys rotated; no wildcards on critical roles.  
- Network: egress control for SSRF; security groups minimal.  
- Audit: admin actions logged; alerts on risky changes.

---

## Supply chain security

- **SCA**: SBOM per build; block critical CVEs with policy.  
- **Provenance**: signed artifacts/images; attestation in registry.  
- **Pin** dependency versions; **verify** checksums for remote assets.  
- **3rd-party JS/SDKs** gated by consent and CSP allowlist.

---

## Compliance mapping (starter)

| Control area | Examples |
|---|---|
| SOC 2 | change mgmt, access, logging, incident, backups |
| ISO 27001 | risk treatment, asset mgmt, cryptography, ops security |
| PCI DSS (if payments) | PAN isolation, SAQ A/A-EP, segmentation |
| GDPR/PDPA etc. | DSRs, consent, transfer mechanisms, records |

Keep a **matrix** linking threats → controls → tests → evidence.

---

## MAE Scenarios (copy/adapt)

### SEC-001 **Main** — IDOR blocked
- **Steps**: user A requests user B’s `/orders/{id}`  
- **Expected**: `403 ERR.AUTHZ.scope`  
- **Oracles**: log shows `msgid=MSG.authz.denied`, no data leaked, trace span tagged

### SEC-002 **Alt** — CSRF token & origin
- **Steps**: submit POST from third-party origin  
- **Expected**: 403 with `ERR.CSRF.failed`  
- **Oracles**: header checks; CSRF cookie + token mismatch recorded

### SEC-003 **Alt** — OTP brute-force throttle
- **Steps**: 6 invalid codes in 1 min  
- **Expected**: `429 RATE_LIMIT` and cooldown  
- **Oracles**: limiter metrics; no email/SMS sent

### SEC-101 **Exception** — CSP/XSS trap
- **Steps**: try `<img src=x onerror=alert(1)>` in comment  
- **Expected**: no execution; CSP report captured  
- **Oracles**: 0 script exec; CSP endpoint shows violation

### SEC-102 **Exception** — SSRF egress block
- **Steps**: upload URL `http://169.254.169.254/latest/meta-data`  
- **Expected**: blocked with `ERR.POLICY.egress`  
- **Oracles**: egress logs deny; alert fired

### SEC-201 **Cross-feature** — Privacy consent enforcement
- **Steps**: revoke analytics consent → reload  
- **Expected**: analytics disabled server-side; only necessary cookies  
- **Oracles**: tag manager state; event headers show `consent=false`

---

## Review checklist (quick gate)

- [ ] Threat Register updated; each threat has at least one **test** and **owner**  
- [ ] **Authn**: MFA, session flags, rotation, logout invalidation  
- [ ] **Authz**: RBAC/ABAC matrix; IDOR tests; admin audit  
- [ ] **Input/Output**: sanitizer, parameterization, upload sandbox  
- [ ] **Headers/TLS**: HSTS, CSP, frame-ancestors, referrer, TLS modern  
- [ ] **Rate limits**: per IP/user/tenant; enumeration defenses  
- [ ] **Secrets**: in KMS; rotation; CI secret scanning  
- [ ] **Data protection**: encryption, backups restore tested, deletion semantics  
- [ ] **Privacy**: consent capture/enforcement; DSR runbook  
- [ ] **Cloud posture**: least privilege, bucket privacy, audit enabled  
- [ ] **Supply chain**: SBOM, signed artifacts, SCA gates  
- [ ] **Evidence** attached to PR (headers snapshot, scan reports, logs, CSP reports)

---

## CSV seeds

**Threat register (snippet)**

```csv
id,tag,asset,owner,severity,test_id
T001,IDOR,Orders API,Auth,high,SEC-001
T002,XSS,Comments UI,Web,high,SEC-101
T003,SSRF,URL fetcher,Platform,high,SEC-102
T004,BruteForce,OTP,Identity,med,SEC-003
T005,CSRF,Checkout Web,Web,high,SEC-002
```

**Security headers baseline**

```csv
header,value
Strict-Transport-Security,max-age=31536000; includeSubDomains
Content-Security-Policy,default-src 'self'; script-src 'self'
Referrer-Policy,strict-origin-when-cross-origin
X-Content-Type-Options,nosniff
X-Frame-Options,DENY
```

**Authz matrix (snippet)**

```csv
route,role,allowed
GET /orders/:id,customer,true-if owner
GET /orders/:id,staff,true-if tenant_staff
DELETE /orders/:id,customer,false
```

**Rate limits**

```csv
key,limit,window_s,burst
ip_login,30,60,10
user_otp,5,60,5
partner_webhook,300,60,100
```

**Secrets inventory**

```csv
name,store,rotation_days,owner
psp_api_key,secrets-manager,90,Payments
smtp_password,secrets-manager,90,Messaging
s3_backup_key,kms-enc,180,Platform
```

---

## Templates

**Abuse case spec**

```
Abuser: <role/bot>
Goal: <what they try to do>
Vector: <endpoint, param, flow>
Signals: <what we monitor>
Mitigation: <rate limit, captcha, denylist, locks>
Test: <steps + oracles + expected error IDs>
```

**Security test plan (per feature)**

```
Area: <authn/authz/xss/ssrf/...>
Assets: <routes, components, data>
Threats: <tags>
Tests: <MAE + fuzz + scans>
Evidence: <headers snapshot, CSP reports, logs, traces>
Sign-off: <owners>
```

**CSP report (example JSON)**

```json
{
  "csp-report": {
    "document-uri": "https://app.example.com/orders",
    "violated-directive": "script-src",
    "blocked-uri": "https://evil.example",
    "original-policy": "default-src 'self'; script-src 'self'"
  }
}
```

---

## Links

- Error Taxonomy: `../40-api-and-data-contracts/error-taxonomy.md`  
- Contracts & Schemas: `../40-api-and-data-contracts/contracts-and-schemas.md`  
- Security Essentials (headers, CSP, secrets): `../50-non-functional/security-essentials.md`  
- Privacy & Compliance (consent, DSRs): `../50-non-functional/privacy-and-compliance.md`  
- Resiliency & Timeouts (brownouts, retries): `../50-non-functional/resiliency-and-timeouts.md`  
- Messaging (security alerts without tracking): `../55-domain-playbooks/messaging-and-notifications.md`
