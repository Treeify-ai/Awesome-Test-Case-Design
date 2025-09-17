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


# Domain: Mobile-first Flows

> Mobile is not a smaller desktop—it’s **touch, sensors, backgrounding, spotty networks, and OS rules**.  
> This playbook turns those constraints into **design choices, test cases, and evidence** for apps, mobile web, and in-app browsers.

---

## TL;DR (defaults we recommend)

- Optimize for **one‑hand use** and **thumb reach**; critical actions within safe zones.  
- Respect **OS text scaling** (Dynamic Type / Font Size) and **dark mode**; layout must not break with ×1.3 text.  
- Handle **offline-first**: cache, queue writes, reconcile on reconnect; show **conflict resolution**.  
- Use **deep links** and **universal/app links** with **idempotent** targets; support **handoff** from web → app.  
- Keep **startup ≤ 1s p95**, **TTI ≤ 2s p95** on mid‑tier devices; avoid heavy cold starts.  
- Ask permissions **in context**; degrade gracefully when denied.  
- **Secure storage** for secrets; **no PII** in notifications or pasteboard.  
- Instrument **crash + ANR**, **cold/warm start**, **screen taps**, **network**; review per release.

---

## Core mobile flows (what to cover)

1. **Install/first run** → feature highlights, privacy notice, telemetry opt-in.  
2. **Sign in / SSO** → email+OTP, OAuth, device biometrics; session refresh.  
3. **Onboarding** → profile, preferences, data pulls while idle.  
4. **Permissions** → notifications, camera, location, photos; ask only when needed.  
5. **Navigation** → tabbed root, deep link entry, back gesture parity.  
6. **Content & sync** → list/detail, optimistic updates, offline queue.  
7. **Payments** (if any) → wallets, 3DS in WebView, app store policies.  
8. **Push & in-app messages** → preference/quiet hours, token lifecycle.  
9. **Uploads** → camera roll, camera capture, background upload, retries.  
10. **Settings & support** → diagnostics, logs export, contact support.  

---

## UX principles (mobile specifics)

- **Thumb reach**: primary CTA bottom; large tap targets (≥ 44×44 px).  
- **Gesture fallback**: every gesture has a button/visible control.  
- **Progressive disclosure**: small screens—stage complexity.  
- **Safe areas**: respect notches, home indicator; avoid edge swipes conflicts.  
- **Haptics & motion**: subtle; honor `reduce motion`.  
- **Text & locale**: dynamic type, truncation rules, RTL mirroring.

---

## Platform contracts

### iOS
- **Keychain** for secrets; **UserDefaults** for low‑risk prefs.  
- Background modes: fetch, processing, location, VoIP (policy).  
- **Universal Links** with associated domains; **Scene** restoration.  
- **App Extensions** (Share, Widgets) follow separate lifecycles.

### Android
- **EncryptedSharedPreferences** / **Keystore** for secrets.  
- Doze/App Standby affects background work; use **WorkManager**.  
- **App Links** with verified domains; **Back** gesture parity.  
- OEM WebViews vary by version; test OEM skins.

### WebView / In‑app browsers
- Cookie partitioning & third‑party cookie blocking; storage quotas.  
- **3DS** and SSO inside WebView can be fragile—prefer system browsers or SDKs.  
- Provide **return URLs** and **idempotent checkpoints**.

---

## Offline & sync (design rules)

- **Cache** lists and last views; show content immediately on open.  
- **Queue writes** with stable **client operation IDs**; replay idempotently.  
- **Conflict resolution**: server‑wins, client‑wins, or merge—**show the choice** and record audit.  
- Visible **sync state**: syncing, failed, conflicts; allow **retry** and **discard**.

---

## Networking & performance

- **Budget**: cold start ≤ 1s p95, first interaction ≤ 2s p95; background sync ≤ 5s p95.  
- **Network shaping**: target 3G/4G; handle captive portals; retry with backoff+jitter.  
- **Compression**: gzip/br; image sizes responsive; lazy load.  
- **Caching**: ETag/If‑None‑Match; client caches; cancel stale requests on navigation.  
- **Pagination**: cursor‑based; stable sort + tiebreakers.

---

## Security & privacy (mobile)

- Store tokens in **Keychain/Keystore**; never in plain prefs.  
- **Biometric unlock** as convenience gate; still enforce server authz.  
- Strip PII from logs; guard screenshots of sensitive screens (`FLAG_SECURE` on Android, `isProtectedDataAvailable` on iOS where relevant).  
- Clipboard: avoid auto‑copy of secrets; respect pasteboard privacy.  
- Notifications: omit sensitive content; allow **privacy‑mode** previews.

---

## Push, badges, and inbox

- Token lifecycle: register, rotate, revoke; handle `unregistered`.  
- **Quiet hours** by timezone; override only for security alerts.  
- In‑app inbox mirrors push; **source of truth** is server; idempotent read receipts.  
- Deep link from notification into **exact state**; fallback to safe screen.

---

## Payments on mobile

- Prefer **Apple/Google Pay**; for cards, use PSP **hosted fields** or native SDKs.  
- 3DS: app‑to‑browser handoff or SDK challenge; handle **return** reliably.  
- Handle **app store** in‑app purchase rules if selling digital goods.

---

## Diagnostics & observability

- **Crash** + **ANR** reports; symbolicated with version/commit.  
- Performance: cold/warm start times, TTI, screen transition times, payload sizes.  
- Network: error codes, retry counts, TLS handshake times, 3xx loops.  
- **UX analytics**: taps, scroll depth (privacy‑aware), consent‑gated.  
- **Debug screen**: env, feature flags, device info, push token, logs export.

---

## MAE Scenarios (copy/adapt)

### M-001 **Main** — First run → sign in → deep link
- **Steps**: install → open → accept privacy → email OTP → success → tap magic link  
- **Expected**: session stored securely; deep link opens correct tab/state  
- **Oracles**: keychain entry; analytics consent; deep link params parsed

### M-002 **Alt** — Offline posting with later sync
- **Steps**: go offline → create post → queue → reconnect  
- **Expected**: optimistic UI; server assigns id; no duplicate on retry  
- **Oracles**: client op IDs; idempotent server response; conflict UI absent

### M-003 **Alt** — Dynamic Type × RTL × Dark
- **Steps**: set large text, RTL, dark → navigate core flows  
- **Expected**: no clipping/overlap; focus order sane; icons mirrored  
- **Oracles**: screenshots; a11y tree; tokens applied

### M-101 **Exception** — Permission denied
- **Steps**: deny camera → try upload → see fallback  
- **Expected**: clear guidance; open settings; app continues  
- **Oracles**: error message id; no crash

### M-102 **Exception** — Captive portal
- **Steps**: join Wi‑Fi with portal → attempt API call  
- **Expected**: detect portal; prompt to sign in or switch network  
- **Oracles**: network status; retry once after portal resolution

### M-103 **Exception** — 3DS in WebView return path
- **Steps**: trigger 3DS challenge → external browser → return  
- **Expected**: resume exactly where left; no double charge  
- **Oracles**: idempotency key; state restored

### M-201 **Cross‑feature** — Push → deep link → offline
- **Steps**: receive push → tap → app opens while offline  
- **Expected**: show cached screen; banner for offline; queue actions  
- **Oracles**: cache hit; queued operations

---

## Acceptance gates

- **Startup budgets** met on mid‑tier devices (cold/warm).  
- Core flows pass **keyboard/VoiceOver/TalkBack** basics; target sizes met.  
- **Offline** create/update/delete queue works; conflicts handled.  
- **Deep links** & **universal/app links** verified across iOS/Android and from email/SMS/web.  
- **Permissions** handled with graceful fallback; settings path tested.  
- **Push** token lifecycle; quiet hours; deep link routing.  
- **Payments** flows robust across SDK/WebView; idempotency proven.  
- **Evidence** captured (videos, HAR, traces, logs export).

---

## Review checklist (quick gate)

- [ ] One‑hand reach & tap sizes; safe areas respected  
- [ ] Dynamic Type / Font size; dark mode; RTL  
- [ ] Offline cache + queued writes; conflict policy explicit  
- [ ] Deep linking (app/universal links) + web handoff  
- [ ] Permissions ask **in context**; denial fallback  
- [ ] Push tokens lifecycle; quiet hours; notification privacy  
- [ ] Payments: wallet/3DS; app store compliance  
- [ ] Diagnostics: crash/ANR, debug screen, logs export  
- [ ] Observability: performance metrics, retries, network errors  
- [ ] Evidence attached to PR (screens, videos, traces)

---

## CSV seeds

**Deep link registry**

```csv
pattern,route,requires_auth,notes
myapp://orders/{id},orders/detail,true,"resume state"
https://example.com/r/{code},referral,false,"onboard variant"
https://example.com/pay/{session},checkout,true,"3DS return"
```

**Permission prompts**

```csv
feature,permission,ask_point,fallback
Profile photo,Photos,On first upload,Use camera; placeholder avatar
Scanner,Camera,On first scan,Manual code entry
Location tips,Location,After feature discovery,Zip/postal input
Notifications,Push,After success action,In-app inbox only
```

**Offline queue**

```csv
op_id,endpoint,method,state,retries
op_001,/posts,POST,queued,0
op_002,/likes/123,DELETE,failed,2
op_003,/orders/999,PUT,sent,1
```

**Performance budgets**

```csv
metric,target
cold_start_p95_ms,1000
tti_p95_ms,2000
screen_transition_p95_ms,300
```

---

## Templates

**Mobile flow spec**

```
Name: <flow>
Surfaces: <iOS | Android | WebView | Mobile web>
Preconditions: <auth, network, locale, theme, text size>
Steps: <short list>
Expected: <short bullets>
Evidence: <screenshots, videos, traces, logs>
```

**Debug screen fields**

```
app_version, build, commit
device, os_version, locale, timezone
feature_flags
push_token (copy)
last_deep_link
logs_export
```

---

## Links

- Messaging & Notifications: `./messaging-and-notifications.md`  
- Payments & Checkout: `../5-domain-playbooks/payments-and-checkout.md`  
- Performance p95/p99: `../50-non-functional/performance-p95-p99.md`  
- Resiliency & Timeouts: `../50-non-functional/resiliency-and-timeouts.md`  
- Accessibility (mobile): `../50-non-functional/accessibility-a11y.md`  
- Compatibility Matrix (devices/locales/networks): `../50-non-functional/compatibility-matrix.md`

---

<p align="center">
  <sub>
    Built with <a href="https://treeifyai.com">Treeify</a> — AI test case design copilot.<br>
    <em>Structured, traceable, high-coverage test design — faster.</em>
  </sub>
</p>
