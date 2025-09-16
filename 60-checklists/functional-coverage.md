# Functional Coverage Checklist

> “Did we cover the right things?”  
> Use this checklist to turn a feature into **testable slices** and ensure **no major gaps** before ship.

---

## TL;DR

- Cover **MAE**: **Main**, **Alternative**, **Exception** for each flow.  
- Verify **contracts** (inputs/outputs/states/errors) exist and are testable.  
- Prove **observability**: logs, metrics, traces, and user-visible evidence.  
- Tie to **non‑functional** gates: perf, resiliency, security, privacy, a11y, i18n, compatibility.  
- Use **risk** to prioritize depth: high‑impact + high‑change areas first.

---

## How to use this

- Copy the **Per‑feature checklist** into your PR/issue.  
- Link evidence (screenshots, HAR, logs, csv) right in the checkboxes.  
- For deeper guidance, jump to:  
  - Flows/States → `../10-fundamentals/flows-and-states.md`  
  - Risk‑based Prioritization → `../10-fundamentals/risk-based-prioritization.md`  
  - Error Taxonomy → `../40-api-and-data-contracts/error-taxonomy.md`  
  - i18n/l10n → `../55-domain-playbooks/i18n-l10n.md`  
  - Accessibility → `../50-non-functional/accessibility-a11y.md`  
  - Mobile → `../55-domain-playbooks/mobile-first-flows.md`  
  - Messaging → `../55-domain-playbooks/messaging-and-notifications.md`  
  - Payments → `../5-domain-playbooks/payments-and-checkout.md`

---

## Preconditions (before testing starts)

- [ ] **Contracts** drafted: request/response shapes, states, message IDs, error codes.  
- [ ] **Oracles** named: where to observe success/failure (UI state, API field, log, metric, trace).  
- [ ] **Test data** ready: fixtures, seed users, feature flags.  
- [ ] **Environments** stable: URLs, keys, mock/sandbox dependencies.  
- [ ] **Acceptance evidence** list agreed (what to capture).

---

## 1) Flows & states (MAE)

- [ ] **Main** flow passes end‑to‑end with evidence.  
- [ ] **Alternative** paths covered (e.g., 3DS challenge, empty list, retry).  
- [ ] **Exception** paths simulate controlled failure (timeouts, invalid input, denied permission).  
- [ ] All flows are **idempotent** where applicable (replays don’t double‑apply).  
- [ ] **State transitions** verified (created → active → archived, etc.).  
- [ ] **Empty / Loading / Error** states designed and exercised.

See `../30-scenario-patterns/main-alt-exception.md`.

---

## 2) Roles & permissions

- [ ] Role matrix applied (viewer, editor, admin, partner).  
- [ ] **Tenant isolation** enforced; no IDOR (cross‑tenant access).  
- [ ] Privileged actions **audited**.  
- [ ] Errors map to **ERR.AUTHZ.\*** and are user‑friendly.

See `../30-scenario-patterns/roles-and-permissions.md`.

---

## 3) Inputs & validation

- [ ] Field types, ranges, units enforced; helpful messages.  
- [ ] Optional/required behavior explicit.  
- [ ] Copy/paste, leading/trailing spaces, emoji/Unicode handled.  
- [ ] **Bulk/CSV** paths validate headers and rows (see Contracts & Schemas).  
- [ ] **Client + server** validation consistent.

---

## 4) Data lifecycle

- [ ] **CRUD**: create/read/update/delete covered.  
- [ ] **Idempotency** on unsafe POST/PUT/PATCH/DELETE.  
- [ ] Derived stores updated (search, cache, analytics).  
- [ ] **Soft/Hard/Anonymize** delete semantics clear.  
- [ ] Background jobs and retries proven.

See `../30-scenario-patterns/data-lifecycle.md`.

---

## 5) Errors & recoveries

- [ ] Error codes map to taxonomy; **message IDs** logged/emitted.  
- [ ] Retries with backoff+jitter on 429/5xx/timeouts; **deadlines** enforced.  
- [ ] User input preserved on failure; **Retry** CTA works.  
- [ ] **Policy** errors (access/region) explained with next step.

See `../40-api-and-data-contracts/error-taxonomy.md` and `../40-api-and-data-contracts/idempotency-and-retries.md`.

---

## 6) Integrations & webhooks

- [ ] **HMAC** verified; replay window; **inbox** dedupe.  
- [ ] Outbound retries with backoff; **DLQ**/replay path.  
- [ ] Contract versions match; pagination/filtering stable.  
- [ ] Evidence: webhook trail, signatures, correlation IDs.

See `../55-domain-playbooks/b2b-integrations.md`.

---

## 7) Feature flags & config

- [ ] Flags default **safe** on failure.  
- [ ] **Kill switch** verified.  
- [ ] Config schema validated; bad config rollback.  
- [ ] Experiment variants tracked (if applicable).

---

## 8) i18n / localization

- [ ] Locale negotiation works; **fallback** chain correct.  
- [ ] **Pluralization** & units format via ICU/CLDR.  
- [ ] **RTL** mirrored; numbers stay LTR where needed.  
- [ ] Text expansion ×1.3 survives; no clipping.  
- [ ] Dates/times show in user time zone.

See `../55-domain-playbooks/i18n-l10n.md`.

---

## 9) Accessibility (quick pass)

- [ ] Keyboard reachability; **focus visible**; logical order.  
- [ ] Labels & `aria-describedby` on form errors.  
- [ ] Headings/landmarks present; **Skip to content**.  
- [ ] Contrast meets WCAG; motion respects reduced‑motion.  
- [ ] Media has alt/captions.

See `../50-non-functional/accessibility-a11y.md`.

---

## 10) Mobile & responsive

- [ ] Breakpoints behave; thumb reach; touch targets ≥ 44×44 px.  
- [ ] **Offline** cache + queued writes where promised.  
- [ ] Deep links/universal links tested; return paths stable.  
- [ ] WebView quirks handled; 3DS/SSO return flows.

See `../55-domain-playbooks/mobile-first-flows.md`.

---

## 11) Non‑functional tie‑ins (sanity)

- [ ] Perf budgets: p95/p99 met for key routes.  
- [ ] Resiliency: retries, breakers, timeouts present.  
- [ ] Security: headers, authn/z, secrets hygiene; no PII in logs.  
- [ ] Privacy: consent, retention, DSR paths honored.  
- [ ] Compatibility: devices/browsers/locales/networks **T0** pass.

Links: `../50-non-functional/*`, `../57-cross-discipline-bridges/for-security-compliance.md`.

---

## 12) Observability & evidence

- [ ] JSON logs with **msgid**, **err.code**, **correlation_id**.  
- [ ] Metrics: RED/USE wired; dashboards show p95/p99 + error rates.  
- [ ] Traces: incoming+outgoing spans; exemplars linked to histograms.  
- [ ] Evidence attached: screenshots, HAR, csv, logs, trace links, webhook trails.  
- [ ] **Rollout plan** documented: flags, canary %, rollback criteria.

See `../57-cross-discipline-bridges/for-developers.md`, `../57-cross-discipline-bridges/for-sres.md`.

---

## Per‑feature checklist (copy/paste)

```
Feature: <name>
Owners: <PM, Eng, QA/Lead, Design, SRE>

[ ] Contracts drafted (I/O, states, errors, message IDs)
[ ] Oracles named (where to observe)
[ ] Test data ready (fixtures/users/flags)

Flows & States (MAE)
[ ] Main passes end‑to‑end with evidence
[ ] Alt path(s) covered (list)
[ ] Exception path(s) simulated (list)
[ ] Idempotency verified where applicable
[ ] Empty/Loading/Error states exercised

Roles & Permissions
[ ] Role matrix ok; tenant isolation enforced
[ ] Admin/privileged actions audited

Inputs & Validation
[ ] Types/ranges/units enforced; messages helpful
[ ] Bulk import/export paths validated

Data Lifecycle
[ ] CRUD covered; derived stores consistent
[ ] Delete semantics (soft/hard/anonymize) proven

Errors & Recoveries
[ ] Error codes map to taxonomy; message IDs emitted
[ ] Retries/backoff/deadlines work; inputs preserved

Integrations & Webhooks
[ ] HMAC & replay window verified; inbox dedupe
[ ] Pagination/filtering stable; evidence attached

Flags & Config
[ ] Safe defaults; kill switch; config validation

i18n & A11y
[ ] Locales/plurals/RTL; text expansion
[ ] Keyboard/focus/labels/contrast; reduced motion

Mobile & Responsive
[ ] Breakpoints; touch targets; deep links; offline (if promised)

Non‑functional sanity
[ ] Perf p95/p99 within budgets
[ ] Resiliency policies enforced
[ ] Security/privacy basics pass
[ ] Compatibility T0 matrix pass

Observability & Evidence
[ ] Logs/metrics/traces wired
[ ] Evidence artifacts attached
[ ] Rollout plan + rollback criteria set
```

---

## CSV seeds

**Coverage matrix (example)**

```csv
case_id,area,type,status,owner,evidence
Login-001,Flows,Main,pass,QA,link://trace/abc
Login-101,Errors,Exception,pass,QA,link://screenshot/err
Import-001,Data,Main,pass,QA,link://csv/report
Webhook-102,Integration,Exception,pass,QA,link://logs/webhook
```

**Risk register (lightweight)**

```csv
risk_id,area,impact,likelihood,notes,mitigation
R001,Payments,high,med,"3DS return path",extra alt/exception tests
R002,Import,med,high,"large files edge",stress + partial failure tests
```

**Perf budgets**

```csv
route,p95_ms,p99_ms
POST /otp/verify,300,800
POST /checkout,400,1000
GET /items,200,600
```

---

## Common gaps (watch for these)

- Missing **Exception** cases (timeouts, dependency brownouts).  
- **Empty/Loading** states not specified.  
- No **evidence** links in PRs.  
- **Idempotency** not exercised; double‑clicks create duplicates.  
- **Tenant isolation** not tested; IDOR leaks.  
- **Locale/RTL** and **a11y** skipped on “just a small feature”.

---

## Sign‑off

- [ ] All required checks complete.  
- [ ] Evidence linked.  
- [ ] Owners signed.  
- [ ] Release notes updated.  
- [ ] Rollout plan approved.
