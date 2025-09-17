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


# Domain: Messaging & Notifications

> Right message, right channel, right time. This playbook makes messaging testable across **email, SMS, push, in‑app, and webhooks**—covering contracts, preferences, deliverability, retries, and evidence.

---

## TL;DR (defaults we recommend)

- Separate **transactional** and **marketing** by **category** and **sender identity** (domains/IPs).  
- Model a **Message** with explicit states and IDs; every send is **idempotent** by a **dedupe key**.  
- Enforce **preferences & consent** before enqueue; support **quiet hours** by timezone.  
- Use **provider webhooks** with **HMAC signatures**; consumers **dedupe** by `event_id`.  
- Implement **retries with backoff+jitter** for transient failures; **failover** to a secondary provider under policy.  
- Use **SPF/DKIM/DMARC** for email; **opt‑out/unsubscribe** must be one‑click and honored everywhere.  
- Avoid PII in URLs; support **link tracking opt‑out** and **private click/open** modes.  
- Capture **evidence**: message ids, provider ids, headers, webhook trails, and user‑visible artifacts.

---

## Scope

- Channels: **Email**, **SMS**, **Push** (APNs/FCM), **In‑app**, **Webhooks** (to partners).  
- Lifecycle: **Compose → Personalize → Preference check → Schedule → Enqueue → Submit → Provider → Delivery callbacks → User actions**.  
- Domains: onboarding, 2FA/MFA, receipts, alerts, marketing, digests, system incidents.

---

## Channels at a glance

| Channel | Strengths | Hazards |
|---|---|---|
| Email | rich, durable, cross‑device | spam filters, bounces |
| SMS | urgent, high open | cost, length limits, country rules |
| Push | instant, app‑linked | device tokens expire |
| In‑app | contextual, free | needs active session |
| Webhook | partner automation | retry & signature, order |

*Keep tables short; details live in sections below.*

---

## Message model & states

Each send gets a stable `message_id` (internal) and, once submitted, a `provider_message_id`.

```
draft → queued → submitted → accepted → delivered → (opened | clicked | unsubscribed)
                                  ↘ bounced | dropped | spam | failed
```

- **Idempotency**: dedupe key like `user:{id}:template:{version}:context_hash:{sha256}`; same key ⇒ same outcome.  
- **Suppression**: global blocks for hard bounces/complaints; per‑category opt‑outs.

---

## Contracts

### Send API (internal)

```
POST /messages/send
Idempotency-Key: msg:{uuid}
{
  "to": "user_123",
  "channel": "email|sms|push|inapp",
  "template_id": "receipt.v3",
  "locale": "en-US",
  "context": { "order_id": "ord_123", "total": 1099, "currency": "USD" },
  "category": "transactional|marketing",
  "schedule_at": "2025-09-16T13:00:00Z"  // optional
}
```

- Validates **preferences/consent** and **quiet hours**.  
- Returns `202` with `message_id`.  
- **Replay** with same key returns same `message_id` and status (`Idempotency-Status: replayed`).

### Provider webhooks (ingest)

- POST from provider with `event_id`, `type`, `occurred_at`, `provider_message_id`, `message_id` (if supported), payload.  
- **HMAC** signature header; reject if invalid.  
- Consumer **inbox** table dedupes by `event_id`.

**Event types (canonical)**  
`accepted`, `delivered`, `opened`, `clicked`, `bounced`, `complained`, `dropped`, `deferred`, `unsubscribed`.

Map vendor‑specific names to these canonical types.

---

## Preferences, consent, and quiet hours

- **Categories**: `security`, `product_updates`, `receipts`, `marketing`, etc.  
- **Granularity**: global mute, channel‑specific, category‑specific.  
- **Quiet hours**: local time window (e.g., 22:00–07:00); define **override** for critical (`security`, `fraud`).  
- **Audit**: store who/what/when/version for consent and changes.

**Tests**  
- Marketing send during quiet hours → deferred to window.  
- Transactional security alert ignores quiet hours and sends.  
- Preference toggle updates across **email + push + SMS**.

---

## Templates & localization

- Versioned templates with **message IDs** for strings; variables typed & validated.  
- **Localization**: fallback chain (e.g., `zh-SG → zh → en`).  
- **A/B**: template versioning includes experiment id; results captured (opens/clicks/conversions).  
- **In‑app**: render via rich text/blocks with an **accessible** structure and **dark mode** tokens.

---

## Scheduling, batching, digests

- **Schedule** absolute UTC or **relative** (e.g., +2h after signup).  
- **Batch** large campaigns; enforce **rate limits** per provider and per tenant.  
- **Digest** to reduce noise: accumulate events and send summaries at intervals; ensure **per‑user idempotent** rollups.

---

## Retries, backoff, failover

- Transient fails (`5xx`, timeouts, 429) → exponential backoff + full jitter; cap by **deadline** (e.g., ≤ 2h for email, minutes for SMS).  
- **Do not** retry permanent fails (bad address, blocklist).  
- **Failover**: policy‑driven (primary → secondary), with **contractual** differences handled (headers, footers, unsubscribe).

---

## Deliverability (email)

- **SPF/DKIM/DMARC** aligned; subdomain separation (`txn.example.com`, `mktg.example.com`).  
- **Reputation**: warm up IPs/domains; dedicated pools for heavy senders.  
- **Content**: clear From/Subject, no link obfuscation for security emails; short URLs on **custom domains**.  
- **Feedback loops**: act on complaints; add to suppression list.  
- **Unsubscribe**: `List-Unsubscribe` header and visible footer; one‑click; honor within **< 24 h**.

---

## SMS specifics

- Short content; include brand; handle country Sender IDs (alpha vs short code).  
- **TTL** per message; retries are limited by carrier rules; capture **carrier status**.  
- Opt‑out via keywords (e.g., `STOP` → global channel opt‑out).

---

## Push specifics

- Handle **token expiry** & **unregistered** responses; clean tokens.  
- iOS **apns-push-type** correct; high‑priority only for time‑sensitive alerts.  
- Avoid sensitive PII in notification body; deep link into app.

---

## In‑app inbox

- Store messages per user with `read_at`, `pinned`, `expires_at`.  
- Badge counts are derived; **recompute** server‑side.  
- Respect **privacy**: only store necessary context; link out to full data.

---

## Privacy & security

- No PII in query strings or tracking params; consider **link signing** with short TTL.  
- Respect **Do Not Track** / tracking preference; provide **tracking‑free** links.  
- Encrypt secrets; rotate provider API keys; audit send operations.

---

## Observability & evidence

- **IDs**: `message_id`, `provider_message_id`, `event_id`, `campaign_id`.  
- **Metrics** by channel/category: send rate, accepted, delivered, open/click, bounce, complaint, unsubscribe.  
- **Traces**: end‑to‑end from enqueue to provider callbacks.  
- **Artifacts**: rendered template snapshots, headers, webhook logs.

---

## MAE Scenarios

### S-001 **Main** — Receipt email with fallback push
- **Steps**: enqueue `receipt.v3` → email delivered  
- **Expected**: email delivered within SLA; open tracked (if allowed)  
- **Alt**: if email hard‑bounces, send push (if opted‑in)  
- **Oracles**: provider events; suppression updated; push token used

### S-002 **Alt** — Quiet hours defer marketing
- **Steps**: marketing campaign at 23:00 user local  
- **Expected**: deferred until 07:00 unless user opted into night delivery  
- **Oracles**: scheduled job evidence; actual send time

### S-101 **Exception** — Webhook replay
- **Trigger**: provider retries `delivered` twice  
- **Expected**: single state transition; idempotent consumer  
- **Oracles**: inbox table shows one processed, one replayed

### S-102 **Exception** — Unsubscribe honored
- **Steps**: user unsubscribes from marketing → attempt send  
- **Expected**: blocked; audit entry; preference visible in UI  
- **Oracles**: 4xx with `POLICY.unsubscribe` when calling send API

### S-103 **Exception** — Provider outage failover
- **Steps**: primary 5xx spikes → breaker opens → secondary used  
- **Expected**: delivery continues within relaxed SLA; later automatic recovery  
- **Oracles**: breaker metrics; provider share shifts

### S-201 **Cross‑feature** — Security alert respects privacy
- **Expected**: no tracking links or pixels; copy avoids phishing cues; DKIM/SPF aligned  
- **Oracles**: headers, content checks, no tracking params

---

## Review checklist (quick gate)

- [ ] Message model with **idempotency** & **suppression** lists  
- [ ] Preferences & **consent** enforced; **quiet hours** by timezone  
- [ ] Templates versioned & localized; variables validated; A/B tracked  
- [ ] Provider **webhooks**: HMAC verified; consumer **dedupe**; canonical event mapping  
- [ ] Retries with **backoff+jitter**; **failover** policy defined  
- [ ] Email deliverability: SPF/DKIM/DMARC, unsubscribe headers/footers  
- [ ] SMS rules: opt‑out keywords, Sender ID country rules  
- [ ] Push tokens lifecycle; sensitive data avoided in payloads  
- [ ] Privacy: no PII in URLs; tracking opt‑out supported  
- [ ] Evidence: rendered snapshots, headers, webhook trail, metrics & traces

---

## CSV seeds

**Template registry**

```csv
template_id,version,category,locale_fallback,notes
welcome.v2,2,transactional,en-US,"supports zh-*"
receipt.v3,3,transactional,en-US,"amount,currency,order_id"
weekly-digest.v1,1,marketing,en-US,"digest window 7d"
security-alert.v1,1,security,en-US,"no tracking"
```

**Preference matrix (sample)**

```csv
user_id,email_txn,email_mktg,sms_txn,sms_mktg,push,inapp,quiet_hours
u_101,true,false,true,false,true,true,22:00-07:00
u_202,true,true,false,false,false,true,23:00-08:00
```

**Throttle & retry**

```csv
channel,max_rps,max_retries,backoff_cap_s
email,200,5,600
sms,50,3,180
push,500,3,60
webhook,100,5,300
```

**Webhook events (canonicalized)**

```csv
event_id,type,message_id,provider_message_id,occurred_at
e_001,accepted,m_001,p_aaa,2025-09-16T12:00:01Z
e_002,delivered,m_001,p_aaa,2025-09-16T12:00:05Z
e_003,opened,m_001,p_aaa,2025-09-16T12:03:10Z
e_004,bounced,m_002,p_bbb,2025-09-16T12:04:00Z
```

**A/B results (snippet)**

```csv
template_id,variant,sent,open_rate,click_rate,conversion_rate
receipt.v3,A,10000,55%,8%,3.1%
receipt.v3,B,10050,57%,8.2%,3.3%
```

---

## Templates

**Message spec**

```
Template: <id>@v<version>
Category: transactional|marketing|security
Channels: <email|sms|push|inapp>
Inputs: <typed fields>
Localization: <supported locales + fallback>
Links: <tracked|untracked> + domain rules
Unsubscribe: <mechanism or N/A>
Evidence: <rendered snapshot + headers>
```

**Test plan (feature)**

```
Given: <trigger/event>
When: <compose → preference check → send>
Then: <delivery + user action + evidence>
Edge: <quiet hours, unsubscribes, outage failover, webhook replays>
```

---

## Links

- Idempotency & Retries: `../40-api-and-data-contracts/idempotency-and-retries.md`  
- Error Taxonomy: `../40-api-and-data-contracts/error-taxonomy.md`  
- Contracts & Schemas: `../40-api-and-data-contracts/contracts-and-schemas.md`  
- Privacy & Compliance (consent): `../50-non-functional/privacy-and-compliance.md`  
- Resiliency & Timeouts: `../50-non-functional/resiliency-and-timeouts.md`  
- Accessibility & i18n: `../50-non-functional/accessibility-a11y.md`, `../55-domain-playbooks/i18n-l10n.md`

---

<p align="center">
  <sub>
    Built with <a href="https://treeifyai.com">Treeify</a> — AI test case design copilot.<br>
    <em>Structured, traceable, high-coverage test design — faster.</em>
  </sub>
</p>
