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


# Compatibility Review Checklist

> “Works on my machine” is not a strategy.  
> Use this checklist to define **tiers**, pick a **minimal passing matrix**, and capture **evidence** that your feature works across devices, browsers, locales, input modes, and networks.

---

## TL;DR

- Define **tiered support** (T0/T1/T2) for devices, browsers, OSes, locales, and networks.  
- Test **MAE** for compatibility: Main (T0), Alt (T1 variations), Exception (degraded/unsupported).  
- Cover **input modes** (keyboard, mouse, touch, SR), **text scaling**, **RTL**, **dark mode**, and **offline/captive portals**.  
- Capture **evidence**: screenshots, videos, HARs, device/UA strings, and OS/browser versions in every PR.

Links:  
- Compatibility Matrix (master list) → `../50-non-functional/compatibility-matrix.md`  
- Accessibility → `../50-non-functional/accessibility-a11y.md`  
- i18n/l10n → `../55-domain-playbooks/i18n-l10n.md`  
- Mobile-first → `../55-domain-playbooks/mobile-first-flows.md`  
- Performance p95/p99 → `../50-non-functional/performance-p95-p99.md`

---

## Preconditions

- [ ] **Tier policy** agreed (T0 = must pass; T1 = best effort; T2 = community-supported).  
- [ ] **Matrix defined**: devices, browsers, OS versions, locales, networks, assistive tech.  
- [ ] **Test data** and accounts seeded (multi-tenant, roles, multiple locales).  
- [ ] **Tools** chosen: real devices + emulators/simulators; browserstack/saucelabs (or lab).  
- [ ] **Instrumentation** ready: logs include device/UA/version; traces capture platform.

---

## 1) Tiers & scope

**T0 (must pass)**  
- Latest **Chrome** (desktop), **Safari** (iOS), **Chrome** (Android), **Firefox ESR** (desktop)  
- **iOS current-1**, **Android current-2** major versions  
- **Windows 11**, **macOS current-1**  
- Locales: **en**, **fr**, **ar (RTL)**, **ja**, **zh-Hans**  
- Networks: **Wi‑Fi**, **4G**, **captive portal** entry

**T1 (strong coverage)**  
- Edge (Chromium), Safari (macOS), Android OEM WebView latest-1  
- Additional locales of business relevance  
- 3G throttling; high-DPI screens; reduced-motion

**T2 (opportunistic)**  
- Older Android WebView -2; niche browsers  
- Tablet portrait/landscape edge cases

> Adjust to your product. Keep T0 small and defensible.

---

## 2) Surfaces to exercise

- **Web (desktop & mobile)**: layouts, pointers, scroll/virtualization, media, clipboard.  
- **Native mobile (iOS/Android)**: startup, deep links, permissions, background/foreground, text scaling.  
- **WebView/in-app browsers**: SSO/3DS flows, cookie/storage partitioning.  
- **Email/SMS surfaces**: links, previews, deep link parameters.  
- **APIs/SDKs**: client lib versions across platforms.

---

## 3) Input & interaction modes

- Keyboard (tab, shift-tab, arrow keys, escape), **focus visible**.  
- Mouse/trackpad (hover, drag, context).  
- **Touch** (tap, long press, pull-to-refresh), hit targets ≥ 44×44 px.  
- **Screen readers**: **NVDA/JAWS** (Windows), **VoiceOver** (iOS/macOS), **TalkBack** (Android).  
- OS text scaling / **Dynamic Type** ×1.3.  
- **RTL** layout mirroring; numbers remain LTR where appropriate.  
- **Dark mode** tokens and contrast.

---

## 4) Networks & offline

- Offline (airplane mode) — cache behavior, queued writes, conflict resolution.  
- **Captive portal** — detect and prompt; don’t loop retries.  
- Throttled 3G/4G — progress indicators, timeouts, retries/backoff.  
- **Proxy/VPN** — SSL/TLS and redirects behave.  
- Push notifications — token lifecycle, quiet hours (if applicable).

---

## 5) Evidence to collect (attach to PR)

- **Screenshots/videos** per tier for key screens (light/dark, LTR/RTL).  
- **Device/OS/Browser** versions and **UA strings**.  
- **HAR** files or waterfalls for slow screens.  
- **Logs/traces** with correlation IDs showing platform tags.  
- **Accessibility report** (axe/pa11y) for web; **screen reader** notes.  
- **Network profiles** used (e.g., 3G profile, offline, captive portal).

---

## 6) Web — per-browser checklist

```
Browser: <Chrome|Edge|Safari|Firefox>
OS: <Windows|macOS|iOS|Android>
Viewport: <mobile|tablet|desktop>

[ ] Layout stable across breakpoints; no overflow/clip
[ ] Focus order logical; keyboard-only operable
[ ] Hover/focus/active states differentiate; pointer + touch OK
[ ] Copy/paste and composition (IME) works (ja/zh)
[ ] File upload/download flows OK
[ ] Clipboard & drag-drop (if used) behave
[ ] Media (audio/video) controls accessible
[ ] Storage (cookies/localStorage/indexedDB) available; quota respected
[ ] Third-party/cross-site cookie policies handled (WebView/ITP)
[ ] Dark mode parity; contrast ≥ WCAG
[ ] RTL mirrored; numbers LTR where appropriate
[ ] Performance within budget; long tasks minimal
[ ] Errors map to message IDs; no unhandled exceptions in console
```

---

## 7) Native mobile — iOS/Android checklist

```
Platform: <iOS|Android>  Version: <x.y>  Device: <model>

[ ] Cold/warm start within budget (p95)
[ ] Deep links/universal/app links route to exact state
[ ] Back gesture parity; safe areas respected (notches/home)
[ ] Dynamic Type / Font size ×1.3; no clipping
[ ] Dark mode parity; icons and illustrations adapt
[ ] Permissions denied: graceful fallback; Settings path works
[ ] Offline cache; queued writes replay; conflicts handled
[ ] WebView flows (SSO/3DS): return path correct; no duplicate action
[ ] Push notifications: privacy text; quiet hours honored
[ ] Sensors (camera/location) behave with deny/revoke
[ ] Logs/traces include device, os_version, locale, timezone
```

---

## 8) Localization & scripts

- **Pluralization** and grammar via ICU; no string concatenation.  
- Long text expansion ×1.3 survives; **CJK** line breaks OK.  
- **RTL** mirrored; bidi isolation for mixed content.  
- Locale negotiation consistent (URL/cookie/`Accept-Language`).  
- Numbers, dates, currency format per locale; time zones respected.

---

## 9) Visuals & rendering

- Retina/high-DPI assets; **SVG** where possible.  
- Fonts subsetted; fallback stacks include target scripts.  
- Icons mirrored in RTL; avoid text baked into images.  
- Animations honor **prefers-reduced-motion**.

---

## 10) Degradation & unsupported states

- Show **browser unsupported** banner only when necessary; provide **next steps**.  
- **Brownouts**: disable heavy features on old devices; communicate clearly.  
- Fallback to **server rendering** when JS disabled (where feasible).  
- Track **unsupported** usage with telemetry (no PII).

---

## MAE Scenarios (compatibility)

### COMP-001 **Main** — T0 web browsers pass
- **Expected**: critical journeys clean on Chrome/Firefox/Safari; evidence captured.  
- **Oracles**: screenshots (LTR/RTL, dark), axe report, HAR for slow page.

### COMP-002 **Alt** — iOS Dynamic Type ×1.3 + dark mode
- **Expected**: no clipping/overlap; focus order sane; icons mirrored.  
- **Oracles**: device screenshots; a11y tree inspection.

### COMP-003 **Alt** — Android WebView latest-1 with 3DS
- **Expected**: return path to app; single charge; idempotency honored.  
- **Oracles**: PSP events; app logs; replay proof.

### COMP-101 **Exception** — Captive portal network
- **Expected**: portal detection; user prompted; retries limited; clear messaging.  
- **Oracles**: network logs; retry/backoff metrics.

### COMP-102 **Exception** — RTL + screen reader
- **Expected**: proper mirroring; numbers LTR; SR announces labels and errors.  
- **Oracles**: SR transcript; contrast snapshot.

---

## Review checklist (quick gate)

- [ ] T0/T1/T2 tiers defined and documented  
- [ ] Matrix covers devices/browsers/OS/locales/networks/assistive tech  
- [ ] Input modes tested (keyboard/mouse/touch/SR)  
- [ ] Text scaling, dark mode, RTL verified  
- [ ] Offline/captive portal/network throttling tested  
- [ ] WebView/SSO/3DS edge cases handled  
- [ ] Evidence attached (screens, videos, HARs, logs)  
- [ ] Perf budgets not violated on target devices  
- [ ] Any unsupported combos have clear messaging + telemetry

---

## CSV seeds

**Browser matrix (example)**

```csv
tier,browser,version,os,viewport
T0,Chrome,latest,Windows 11,desktop
T0,Firefox ESR,latest,Windows 11,desktop
T0,Safari,latest,iOS,mobile
T0,Chrome,latest,Android,mobile
T1,Edge,latest,Windows 11,desktop
T1,Safari,latest,macOS,desktop
T1,WebView,latest-1,Android,mobile
```

**Device matrix**

```csv
tier,platform,model,os_version
T0,iOS,iPhone 14,17.x
T0,Android,Pixel 7,14.x
T1,Android,Galaxy A54,14.x
T1,iOS,iPad (mid-tier),17.x
```

**Locale/script matrix**

```csv
tier,locale,dir,notes
T0,en,ltr,baseline
T0,ar,rtl,mirroring
T0,ja,ltr,IME & CJK
T1,fr,ltr,plurals
T1,zh-Hans,ltr,CJK line breaks
```

**Assistive tech**

```csv
tier,os,screen_reader,version
T0,Windows,NVDA,latest
T1,Windows,JAWS,latest
T0,iOS,VoiceOver,latest
T0,Android,TalkBack,latest
```

**Network profiles**

```csv
tier,profile,down_mbps,up_mbps,rt_ms,loss_pct
T0,Wi-Fi,50,10,20,0
T1,4G,9,3,60,0.5
T1,3G,1.6,0.5,150,1
T1,CaptivePortal,0,0,999,0
Offline,Airplane,0,0,0,0
```

---

## Templates

**Matrix entry (PR snippet)**

```
Device/Browser:
- Pixel 7 / Android 14 / Chrome 126 (mobile)
- iPhone 14 / iOS 17 / Safari 17 (mobile)
- Windows 11 / Chrome 126 (desktop)

Locales:
- en, ar (RTL), ja

Networks:
- Wi-Fi, 4G throttled, Captive Portal

Evidence:
- Screens (light/dark, RTL)
- HAR + waterfalls
- axe report
- Trace links (p95)
```

**Bug reproduction (compatibility)**

```
Env: <staging url>
Device/OS/Browser: <model, os, version>
Locale/Theme/Text size: <en|ar, light|dark, x1.3>
Network: <Wi-Fi|4G|CaptivePortal>
Steps: <1…n>
Expected: <short>
Actual: <short>
Artifacts: <screens, HAR, logs, trace id>
```

---

## Sign-off

- [ ] T0 surfaces pass with evidence  
- [ ] T1 spot-checks pass (or tracked)  
- [ ] Known unsupported combos documented with UX fallback  
- [ ] Matrix stored and linked; owners assigned  
- [ ] Post-merge synthetic checks cover top journeys per tier

---

<p align="center">
  <sub>
    Built with <a href="https://treeifyai.com">Treeify</a> — AI test case design copilot.<br>
    <em>Structured, traceable, high-coverage test design — faster.</em>
  </sub>
</p>
