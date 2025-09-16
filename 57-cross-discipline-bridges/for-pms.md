# For PMs — Testable Requirements & Acceptance Criteria

> Your spec is the first test.  
> This guide helps PMs write requirements that **engineers can build**, **test can verify**, and **stakeholders can sign off**—fast, unambiguous, and backed by evidence.

---

## TL;DR (defaults we recommend)

- Every requirement must be **observable** (what signal proves it works?) and **bounded** (how fast? how many? how often?).  
- Use **MAE** acceptance (Main / Alternative / Exception) to cover the happy path **and** the edges.  
- Define **SLIs/SLOs** for performance and reliability; add **budgets** (p95/p99).  
- Specify **contracts**: inputs, outputs, **error taxonomy**, and **states**.  
- Include **instrumentation** and **success metrics** at kickoff—not after launch.  
- Add **non‑functional** acceptance: accessibility, privacy, security, compatibility, i18n.  
- Agree on **evidence** artifacts before work starts (screens, logs, traces, csvs).

---

## What “testable” means (in practice)

A requirement is testable when a reviewer can answer **Yes/No** using **objective evidence** within minutes.

**Recipe**
1. **Observable** outcome (metric, event, response field, UI state).  
2. **Threshold** or **range** (e.g., “p95 ≤ 800 ms”).  
3. **Where to measure** (endpoint, screen, log, dashboard).  
4. **Who owns** the signal (team/channel).  
5. **Acceptance**: MAE cases + evidence list.

---

## Story skeleton (fill-in)

```
As a <role>,
I want <capability>,
so that <value>.

Scope:
- In: <entities, surfaces, tenants, locales>
- Out: <explicitly excluded>

Assumptions:
- <what must be true>

Dependencies:
- <services, data, partners, flags>

Contracts:
- Input: <shape + units + constraints>
- Output: <shape + fields + states>
- Errors: <message IDs from taxonomy>

Non‑functional (bounds):
- Perf: p95 ≤ <ms>, p99 ≤ <ms>, peak RPS <n>
- Resiliency: retries <n>, deadline <ms>, fallback <desc>
- Security: authn/z matrix, headers, PII handled
- Privacy: data classes, retention, consent
- Accessibility: keyboard, contrast, screen reader notes
- Compatibility: matrix tier (T0/T1), locales
- i18n: locales, RTL, text expansion ×1.3

Instrumentation:
- Events: <names + properties>
- Metrics: <SLIs/SLOs>
- Logs/Traces: <ids, correlation>

Acceptance (MAE):
- Main: <happy path>
- Alt: <variations>
- Exception: <errors/failures>

Evidence:
- <screenshots, HARs, csvs, log extracts, dashboard link>

Rollout/Guardrails:
- flags, canary %, rollback criteria, kill switch

Done when:
- All acceptance checks pass and evidence attached in PR.
```

---

## MAE acceptance (pattern)

- **Main** — primary, most common flow.  
- **Alt** — legitimate variation (e.g., 3DS challenge, empty list, secondary locale).  
- **Exception** — controlled failure (timeouts, invalid input, denied permission).

Tie each to **oracles** (exact signals we’ll check).

---

## Example: “Login with email + one-time code (OTP)”

**Scope**: Web + mobile web; en, fr, ar; guest users only. Out of scope: SSO, rate limits UI.  
**Dependencies**: Messaging service (email), Auth API.  
**Contracts**:  
- `POST /otp/request { email }` → `202` with `message_id`; error IDs: `VALIDATION.email`, `RATE_LIMIT`.  
- `POST /otp/verify { email, code }` → `200 { session_id }` or `400 CODE.invalid`.

**Non‑functional**  
- Perf: `request` p95 ≤ 400 ms; `verify` p95 ≤ 300 ms.  
- Security: throttle 5/min per IP+email; no code in logs; cookie `HttpOnly; Secure; SameSite=Lax`.  
- Privacy: store email as PII; retention 365d; consent banner before analytics.  
- Accessibility: keyboard-only; focus returns to error; labels and error announcements.  
- Compatibility: T0 matrix devices/browsers; RTL mirror.

**Instrumentation**  
- Events: `otp.requested`, `otp.delivered`, `otp.verified`, `login.success`, `login.fail`.  
- Metrics: verification success rate ≥ 85%, drop-off < 10% between request→verify.

**Acceptance (MAE)**  
- **Main**: Request → receive email → verify within 5 min → session created.  
  - **Evidence**: provider webhook `delivered`, server logs `otp.verified`, cookie set, screenshot.  
- **Alt**: Resend after 60 s throttle → latest code wins.  
  - **Evidence**: event trail shows `resend=true`, first code rejected.  
- **Exception**: 6th attempt in 1 min → `429 RATE_LIMIT` and no email sent.  
  - **Evidence**: response code + absence of provider event.

---

## Example: “CSV import of customers”

**Contracts**  
- Accepts `.csv` UTF‑8, header row required.  
- Schema: `email, name, country, created_at` (see [Contracts & Schemas](../40-api-and-data-contracts/contracts-and-schemas.md)).  
- Errors: per‑row with **message IDs**; idempotent by file hash.

**Acceptance**  
- **Main**: 10k valid rows → 100% imported in < 5 min.  
- **Alt**: Duplicate upload → `Idempotency-Status: replayed`.  
- **Exception**: 1 bad row → file accepted; row rejected with `VALIDATION.email`.

**Evidence**: import report CSV, counts match, dedupe table entry, metrics snapshot.

---

## Example: “Checkout — card with 3DS”

**Contracts**: Payment intent model; idempotency; webhooks (see [Payments](../5-domain-playbooks/payments-and-checkout.md)).  
**Acceptance**  
- **Main**: frictionless auth → capture → receipt.  
- **Alt**: challenge flow returns to app; no duplicate charge.  
- **Exception**: provider timeout → replay with same key; exactly-once capture.  
**Evidence**: PSP events, ledger entries, receipt email, correlation IDs.

---

## Writing crisp acceptance criteria

Prefer **imperatives + oracles**:

- ✅ “**Pressing ‘Place order’ returns 200 with `order_id`, creates a ledger entry, and sends `order.created` within 1s.**”  
- ❌ “Works quickly” / “Should handle errors gracefully”.

Add **numbers** and **IDs**. Point to **taxonomy codes** and **metrics**.

---

## Non‑functional requirements (NFRs) you must call out

- **Performance**: p95/p99 budgets, payload size caps.  
- **Resiliency**: retries/backoff, deadlines, circuit breakers, load shedding.  
- **Security**: authn/z, headers, secrets, audit.  
- **Privacy**: classes, retention, DSRs, consent.  
- **Accessibility**: keyboard, contrast, SR announcements.  
- **Compatibility**: devices/browsers/locales/networks ([Matrix](../50-non-functional/compatibility-matrix.md)).  
- **i18n**: locales, RTL, pluralization ([i18n/l10n](../55-domain-playbooks/i18n-l10n.md)).  
- **Observability**: logs (no PII), metrics, traces, **message IDs**.

Link to the playbooks for details.

---

## Success metrics & guardrails (define at kickoff)

- **North-star**: e.g., conversion uplift +X%, time‑to‑value −Y%.  
- **SLIs/SLOs**: uptime, latency, error rate, delivery rate, a11y violations = 0.  
- **Guardrails**: bounce/complaint rate ≤ threshold, unsafe content = 0, cost/request ≤ cap.  
- **Experiment**: A/B owner, sample size, duration, success thresholds.

---

## Evidence kit (attach to PR / change review)

- Screens/videos, **axe/pa11y** report, **HAR**/waterfalls.  
- Logs/traces with **correlation_id** and **message IDs**.  
- CSVs: import/export reports, reconciliation, drift.  
- Dashboard links: p95/p99, error rates, funnel conversion.  
- Webhook trails and dedupe/inbox snapshots.  
- Feature flag + rollout notes.

---

## Bad smells (rewrite these)

- “Support all browsers and devices.” → **Name your T0/T1/T2**.  
- “User sees error if something goes wrong.” → **Which error? Which code? Where?**  
- “Log in should be fast.” → **p95 ≤ 300 ms**, measure at **/otp/verify**.  
- “Should handle retries.” → **Backoff+jitter on 429/5xx**, **max attempts 3**, **deadline 2s**.  
- “Make it secure.” → **Authn** + **Authz** matrix, **CSP/HSTS**, **no PII in logs**.

---

## Rituals that keep you fast

- **Definition of Ready (DoR)** — a story is ready when:  
  - Scope/Out-of-scope listed, **contracts drafted**, NFRs filled, acceptance MAE written, instrumentation defined, dependencies aligned, evidence agreed.  
- **Definition of Done (DoD)** — done when:  
  - All MAE pass with evidence; a11y & compatibility quick passes; dashboards/alerts live; runbook/rollback documented.

---

## Checklists

### DoR (before sprint)

- [ ] Problem & value defined; users & roles named  
- [ ] Scope / Out-of-scope written  
- [ ] Contracts (I/O, states, errors) sketched  
- [ ] Non‑functional budgets set (perf, resiliency, security, privacy, a11y, i18n, compatibility)  
- [ ] Instrumentation & metrics defined; experiment plan if applicable  
- [ ] MAE acceptance criteria drafted with oracles  
- [ ] Evidence artifacts agreed  
- [ ] Dependencies & flags listed; rollout plan sketched

### DoD (acceptance)

- [ ] All MAE pass with evidence attached  
- [ ] Logs/traces/metrics show healthy SLIs/SLOs post‑merge  
- [ ] Docs updated (runbooks, API spec, user help)  
- [ ] Rollout notes + kill switch verified  
- [ ] Monitoring alerts configured and green

---

## CSV seeds

**Acceptance matrix (example)**

```csv
case,type,preconditions,steps,expected,evidence
Login-001,Main,guest,"request code -> verify","200 + session","webhook delivered + cookie set"
Login-101,Exception,throttle,"6 requests/min","429 RATE_LIMIT","no provider event"
Import-001,Main,file 10k rows,"upload -> wait","imported 100% < 5m","report.csv + counts"
Checkout-003,Alt,3DS challenge,"confirm -> challenge -> return","authorized -> captured","PSP events + receipt"
```

**SLI/SLO register (snippet)**

```csv
sli,definition,slo,owner
otp_verify_latency_p95_ms,p95 of POST /otp/verify,<=300,Identity
email_delivery_rate_pct,delivered/accepted (rolling 1h),>=98,Messaging
checkout_error_rate_pct,5xx/total (rolling 15m),<0.5,Payments
```

**Event dictionary (starter)**

```csv
event,required_props,notes
otp.requested,user_id,email_hash,do not store raw email
otp.verified,user_id,method,success=true/false
order.placed,order_id,amount_minor,currency
```

---

## Templates

**One‑pager (feature) — testable slice**

```
Feature: <name>
Value: <who benefits + outcome>
Scope: In <...> / Out <...>
Contracts: <I/O + states + errors>
NFRs: <perf, resiliency, security, privacy, a11y, compatibility, i18n>
Instrumentation: <events + metrics>
Acceptance (MAE): <bullets + oracles>
Evidence: <artifacts list>
Rollout: <flags, canary, guardrails, rollback>
Owners: <PM, Eng, QA/Lead, Design, Data>
```

**Gherkin (when helpful)**

```
Given a guest user with a valid email
When they request a one-time code
Then the system sends an email and returns 202 with message_id
And verifying the code within 5 minutes returns 200 with session_id
And the event "otp.verified" is emitted
```

---

## Links

- Error Taxonomy: `../40-api-and-data-contracts/error-taxonomy.md`  
- Contracts & Schemas: `../40-api-and-data-contracts/contracts-and-schemas.md`  
- Performance budgets: `../50-non-functional/performance-p95-p99.md`  
- Resiliency & Timeouts: `../50-non-functional/resiliency-and-timeouts.md`  
- Accessibility: `../50-non-functional/accessibility-a11y.md`  
- Compatibility Matrix: `../50-non-functional/compatibility-matrix.md`  
- Payments: `../5-domain-playbooks/payments-and-checkout.md`  
- Messaging: `../55-domain-playbooks/messaging-and-notifications.md`  
- i18n/l10n: `../55-domain-playbooks/i18n-l10n.md`  
- Data/ML: `../55-domain-playbooks/data-ml-features.md`
