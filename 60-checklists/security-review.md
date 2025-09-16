# Security Review Checklist

> Security is a **ship gate**, not an afterthought.  
> Use this checklist to translate threats into **tests**, ensure controls are **observable**, and attach **evidence** that auditors and engineers both trust.

---

## TL;DR

- Maintain a **Threat Register** (STRIDE + OWASP) with at least **one test per threat**.  
- Verify **authn/authz**, **tenant isolation**, **headers/TLS/CSP**, **secrets hygiene**, **rate-limits/abuse**, **privacy/DSRs**, and **supply chain**.  
- Capture **evidence** on every PR: headers snapshot, logs with `msgid` + `err.code`, CSP reports, scans, and runbook links.

Links:  
- Threat→Test mapping → `../57-cross-discipline-bridges/for-security-compliance.md`  
- Error taxonomy → `../40-api-and-data-contracts/error-taxonomy.md`  
- Contracts & Schemas → `../40-api-and-data-contracts/contracts-and-schemas.md`  
- Idempotency & Retries → `../40-api-and-data-contracts/idempotency-and-retries.md`  
- Security Essentials → `../50-non-functional/security-essentials.md`  
- Privacy & Compliance → `../50-non-functional/privacy-and-compliance.md`

---

## Preconditions (before review)

- [ ] **Threat Register** updated for the change (assets, tags, owners, tests).  
- [ ] **Scope** defined (routes, data classes, roles, tenants, locales).  
- [ ] **Contracts** drafted (I/O, states, **message IDs**, error codes).  
- [ ] **Environments** ready (sandbox keys, test tenants, webhook secrets).  
- [ ] **Observability** wired: JSON logs, metrics, traces; **no PII** in logs.  
- [ ] **Evidence plan** agreed (what artifacts to attach).

---

## 1) Identity & session

- [ ] Authn method defined (OTP/MFA/SSO); throttles in place.  
- [ ] Cookies: `HttpOnly; Secure; SameSite` set; rotation on privilege change.  
- [ ] Session fixation prevented; logout invalidates server state.  
- [ ] Token validation (iss/aud/exp/nbf/nonce); clock skew tolerated.  
- [ ] No secrets or codes logged; telemetry uses hashes only.

**Evidence**: header snapshot, authn logs with `MSG.login.*`, traces with authn spans.

---

## 2) Authorization & tenancy

- [ ] RBAC/ABAC matrix applied; least privilege.  
- [ ] **IDOR** checks on path/body/query IDs; list endpoints scoped by tenant.  
- [ ] Admin routes gated; privileged actions audited.  
- [ ] Policy errors map to `ERR.AUTHZ.*`; messages safe and non-revealing.

**Evidence**: failing cross-tenant tests (403), audit logs, role matrix doc.

---

## 3) Input, output & templating

- [ ] Server-side validation; rejects unknown fields.  
- [ ] Parameterized SQL/NoSQL; ORMs not used with string concatenation.  
- [ ] Output encoding; rich text sanitized; no HTML injection.  
- [ ] File upload constraints: size/type/extension; AV scan; sandboxed transforms.  
- [ ] **SSRF** protections: egress denylist; block metadata IP ranges; DNS pinning if applicable.

**Evidence**: ZAP/Burp report diffs, sanitizer unit tests, SSRF deny logs.

---

## 4) Headers, TLS & CSP (web)

- [ ] **HSTS** enabled (consider preload).  
- [ ] **CSP** with `default-src 'self'` and nonce/sha for scripts; `frame-ancestors` set.  
- [ ] **Referrer-Policy** `strict-origin-when-cross-origin`.  
- [ ] **X-Content-Type-Options** `nosniff`; **X-Frame-Options** `DENY` (or CSP).  
- [ ] TLS modern ciphers; HTTP→HTTPS redirect; OCSP stapling where supported.

**Evidence**: `/security/headers` endpoint snapshot; CSP violation report counts.

---

## 5) Secrets & config

- [ ] Secrets in **KMS/Secrets Manager**; never in repo or logs.  
- [ ] Rotation schedule and break-glass policy documented.  
- [ ] Outbound provider keys scoped minimal; IP allowlists where possible.  
- [ ] Config schema validated; feature flags **fail safe**.

**Evidence**: secret scans in CI, rotation schedule, config schema tests.

---

## 6) Rate limiting & abuse

- [ ] Token buckets per IP/user/tenant for sensitive routes.  
- [ ] Enumeration defenses: same response/time for exists vs not.  
- [ ] CAPTCHA only when justified; not on critical recovery flows.  
- [ ] Webhooks have per-partner limits; backoff, DLQ, and replay.

**Evidence**: limiter metrics, abuse test logs, webhook rate-limit proofs.

---

## 7) Webhooks & integrations

- [ ] Inbound webhooks signed (HMAC + timestamp); replay window enforced.  
- [ ] Consumer **inbox** dedupe table; idempotent handlers.  
- [ ] Outbound retries/backoff with deadlines; circuit breakers.  
- [ ] No secrets in query strings; partner endpoints stored securely.

**Evidence**: signature verification logs, dedupe hits, replay runs.

---

## 8) Data protection & privacy

- [ ] Data classified; PII flagged in schemas; **minimization** applied.  
- [ ] At-rest encryption (DB/disk/object); key rotation plan.  
- [ ] In-transit TLS 1.2+; mTLS where needed.  
- [ ] **Deletion semantics**: soft/hard/anonymize; derived data purged.  
- [ ] **Consent** enforced (marketing/analytics); tracking off when revoked.  
- [ ] **DSRs** (export/erase) implemented and **idempotent**.

**Evidence**: deletion job logs, consent flags in events, DSR runbook.

---

## 9) Cloud/SaaS posture

- [ ] Buckets/containers **private-by-default**; least-privilege IAM roles.  
- [ ] No wildcard `*` on critical roles; access keys rotated.  
- [ ] Network egress control; security groups minimal.  
- [ ] Admin actions audited; alerts on risky changes.

**Evidence**: IaC lint results, cloud config diff, access review.

---

## 10) Supply chain & artifacts

- [ ] SBOM produced; SCA gates block **critical** CVEs.  
- [ ] Build provenance signed; images/artifacts attested.  
- [ ] Dependencies pinned/verified checksums; 3rd-party JS/SDKs allowlisted via CSP.  
- [ ] Container baselines scanned; minimal base images.

**Evidence**: SBOM, SCA report, signature/attestation links.

---

## 11) Logging & observability (security signals)

- [ ] JSON logs with `msgid` and `error.code`; **no PII**; correlation/trace IDs present.  
- [ ] Security events recorded: authn failures, authz denials, rate limits, CSP violations.  
- [ ] Alerts exist for notable spikes (auth failures, 5xx, webhook signature failures).  
- [ ] Runbooks linked from alerts.

**Evidence**: log queries, alert screenshots, runbook links.

---

## 12) Mobile & desktop app specifics (if applicable)

- [ ] Secrets in Keychain/Keystore; **no plaintext** storage.  
- [ ] Screenshots protected for sensitive views (`FLAG_SECURE` / protected views).  
- [ ] Deep link validation; universal/app links pinned.  
- [ ] No PII in notifications or pasteboard.

**Evidence**: platform config, static analysis output, screenshots of settings.

---

## MAE Scenarios (security)

### SEC-001 **Main** — Authz blocks cross-tenant access
- **Expected**: 403 with `ERR.AUTHZ.scope`; no data leakage.  
- **Oracles**: logs show `MSG.authz.denied` with correlation_id; trace span tagged.

### SEC-002 **Alt** — CSRF token + origin validated
- **Expected**: cross-origin POST rejected; token required on same-site.  
- **Oracles**: headers verified; CSRF cookie/token pairing logged.

### SEC-101 **Exception** — Webhook replay
- **Expected**: duplicate `event_id` ignored; single state transition.  
- **Oracles**: inbox dedupe hit; `Idempotency-Status: replayed`.

### SEC-102 **Exception** — SSRF attempt
- **Expected**: call blocked by egress rules; `ERR.POLICY.egress`.  
- **Oracles**: deny logs; alert fired.

### SEC-201 **Cross-feature** — Consent revocation enforced
- **Expected**: analytics disabled; only necessary cookies; messaging switches to transactional-only.  
- **Oracles**: event headers show `consent=false`; messaging blocked by policy.

---

## Review checklist (quick gate)

- [ ] Threat register tests exist with owners  
- [ ] Identity/session secure; cookies flagged; MFA/throttles in place  
- [ ] Authz/tenant isolation enforced; admin audited  
- [ ] Input/output sanitized; uploads safe; SSRF blocked  
- [ ] Headers/TLS/CSP correct; `/security/headers` snapshot saved  
- [ ] Secrets in manager; rotation policy; config validation; flags fail safe  
- [ ] Rate limits & abuse controls verified  
- [ ] Webhooks signed; dedupe/replay; outbound retries with deadlines  
- [ ] Data protection: encryption, deletion, consent, DSR runbooks  
- [ ] Cloud posture least-privilege; audits enabled  
- [ ] Supply chain: SBOM, SCA gates, signed artifacts  
- [ ] Logs/metrics/traces wired; security alerts + runbooks  
- [ ] Evidence artifacts attached to PR

---

## CSV seeds

**Threat register (snippet)**

```csv
id,tag,asset,owner,test_id
T001,IDOR,Orders API,Auth,SEC-001
T002,CSRF,Checkout Web,Web,SEC-002
T003,SSRF,URL Fetcher,Platform,SEC-102
T004,WebhookReplay,Webhook Ingest,Integrations,SEC-101
T005,Consent,Analytics,Privacy,SEC-201
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

**Security test plan (per feature)**

```
Area: <authn/authz/xss/ssrf/...>
Assets: <routes, components, data>
Threats: <tags>
Tests: <MAE + fuzz + scans>
Evidence: <headers snapshot, CSP reports, logs, traces>
Sign-off: <owners>
```

**DSR runbook (erase/export)**

```
Data subject: <user/tenant>
Scope: <systems + datasets>
Find: <queries/tools>
Export: <format + fields>
Erase: <hard/soft/anonymize + derived purge>
Audit: <evidence + approver>
SLA: <days>
```

**Incident report (security)**

```
Summary: <what/when/who affected>
Impact: <data + scope>
Timeline: <detailed events>
Root causes: <technical + organizational>
Controls: <what failed/what worked>
Actions: <fixes + owners + dates>
Evidence: <logs, traces, headers, scans>
```

---

## Sign-off

- [ ] All required security checks pass.  
- [ ] Evidence attached and archived.  
- [ ] Owners signed (Security + Feature).  
- [ ] Post-merge monitoring and alerts verified.
