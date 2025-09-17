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

# Compatibility Matrix

> One product, many environments. This playbook shows how to **pick the right matrix**, **keep it small but meaningful**, and **prove compatibility** across devices, browsers, OSes, locales, networks, and integrations.

---

## TL;DR (defaults we recommend)

- Start with a **Tiered matrix**: **T0 Critical**, **T1 Broad**, **T2 Best-effort**.  
- Use **pairwise** across dimensions (Device × Browser × Locale × Network). Elevate **t=3** for risky triples (e.g., *Device × Wallet × Locale*).  
- Include **Accessibility** and **Dark mode** in T0.  
- Always test **latest − N** OS/app versions (pick N by user base/risk).  
- Shape **networks** (offline/2G/3G/4G/Wi‑Fi, captive portal, IPv6).  
- Capture **evidence**: screenshots, DOM snapshots, a11y reports, logs/traces with device & browser tags.

---

## Dimensions (pick what matters)

| Dimension | Examples (short) |
|---|---|
| Device | iPhone 14/15, iPad, Pixel 7/8, Samsung S22/S23, Windows laptop, Mac |
| OS | iOS latest−1, Android latest−1, Windows 11, macOS latest−1 |
| Browser | Safari, Chrome, Edge, Firefox (stable) |
| Engine | WebKit, Blink, Gecko |
| Screen | 320w, 390w, 768w, 1024w, 1440w; DPR 1×/2×/3× |
| Input | Touch, Mouse, Trackpad, Keyboard-only |
| Locale | en-US, fr-FR, de-DE, ja-JP, zh-CN, ar-SA (RTL) |
| Time zone | UTC, Asia/SG, US/PT, DST edges |
| Currency | USD, EUR, JPY, CNY |
| Network | Offline, 2G/3G, 4G, Wi‑Fi; high RTT; packet loss |
| IPv | IPv4, IPv6, Dual-stack |
| A11y | VoiceOver, TalkBack, NVDA, JAWS; contrast; focus |
| Theme | Light, Dark, High-contrast |
| App ver | latest, latest−1, latest−2 (mobile/desktop apps) |
| Integrations | OAuth/OIDC/SAML, Google/Apple sign-in, Wallets, Payment gateways |
| Rendering | WebView (iOS/Android), in-app browsers |

> Keep tables short; move prose to body. Do not list every device—choose **representatives**.

---

## Tiered coverage (example)

**T0 Critical** — block ship if broken  
- iPhone (latest iOS) Safari; Pixel (latest Android) Chrome  
- Windows 11 Chrome & Edge; macOS Safari  
- One **RTL** locale; one **CJK** locale  
- **Dark mode**, **Keyboard-only**, **Screen reader** quick pass  
- **4G** and **Offline** basic flows

**T1 Broad** — major variations  
- iPad Safari, Samsung Android skin, Firefox stable  
- 768w tablet breakpoints, 1440w desktop  
- Captive portal login, **IPv6**, slow 3G

**T2 Best-effort** — long tail  
- Older device models (latest−2), secondary locales, niche clients (email/PDF)

---

## Build your matrix (recipe)

1. **List dimensions** that affect behavior (above).  
2. Choose **representatives** per dimension (2–4 each).  
3. Apply **pairwise** to get ~**8–20** combos; add **seed** scenarios for high-risk triples.  
4. Add **a11y + theme** toggles to **every** T0 run.  
5. Add **network shaping** to at least 1 run per platform.  
6. Encode **oracles** and **evidence capture** (below).  
7. Review **risk deltas** each quarter; rotate long-tail items.

---

## Oracles & evidence

- **Visual**: screenshots per breakpoint; pixel-diffs for key screens.  
- **DOM**: semantic landmarks, ARIA roles, focus order.  
- **A11y**: axe/pa11y rulesets; color contrast reports.  
- **Network**: HAR files; timing waterfalls; retry/backoff traces.  
- **Storage**: cookie/localStorage/sessionStorage behavior, quota, eviction.  
- **Mobile**: deep links, push permission dialogs, safe area insets, WebView quirks.  
- **Logs/Traces**: device, OS, browser, locale, theme, network tags.

---

## Web specifics (checklist)

- Layout: **Grid/Flex** behavior; min/max content widths; sticky headers/footers.  
- Inputs: `input[type=date|number|email]`; virtual keyboards; autofill.  
- Media: images `srcset`/`sizes`, `<picture>`, **DPR scaling**; video autoplay policies.  
- JS: ES features (Intl API, ResizeObserver); module/nomodule fallbacks.  
- Storage: SameSite cookies; ITP; 3rd‑party cookie blocking.  
- Scrolling: momentum scroll, `overflow: hidden` traps, scroll restoration.  
- Printing/PDF: print stylesheets; PDF export fonts/emoji rendering.

---

## Mobile specifics (checklist)

- **iOS**: Safe area, notch, dynamic type, backgrounding, push prompts, universal links.  
- **Android**: Back gesture, overlay permissions, OEM WebView versions, deep links, Doze/App Standby.  
- **WebView**: JS bridges, cookie partitioning, storage quotas, file input.  
- **Apps**: in‑app update flows; app version **latest−N** compatibility with backend APIs.

---

## Internationalization (i18n/l10n)

- **Text expansion** (×1.3) and **CJK** line breaking.  
- **RTL**: directionality, icons mirroring, caret movement, text input.  
- **Numbers/dates**: decimal/grouping, calendars (Gregorian only vs others), 24h/12h time.  
- **Address formats** and name order.  
- **Pluralization** rules.  
- **Unicode**: NFC/NFD, emoji, bidi controls.

---

## Network & offline

- **Profiles**: offline, high latency (300–600 ms RTT), loss (1–5%), throttled bandwidth.  
- **Captive portal**: detect and recover.  
- **Retry policy**: backoff + jitter; idempotent POSTs.  
- **Offline UX**: caching, queueing, conflict resolution on reconnect.  
- **IPv6**: DNS64/NAT64, Happy Eyeballs.

---

## Accessibility (a11y)

- **Keyboard-only**: visible focus, logical order, escape traps.  
- **Screen readers**: VoiceOver, TalkBack, NVDA/JAWS quick sweep.  
- **Color/contrast**: WCAG AA/AAA where applicable.  
- **Motion**: `prefers-reduced-motion`; avoid parallax nausea.  
- **Forms**: labels, errors linked via `aria-describedby`; message IDs.

---

## Integrations matrix (pick reps)

| Area | Representatives (short) |
|---|---|
| Sign‑in | Google OIDC, Apple, SAML IdP |
| Payments | Card (3DS), PayPal, Apple/Google Pay |
| Maps | Google Maps, Mapbox |
| Email clients | Gmail web, Outlook desktop, iOS Mail |
| PDF viewers | Chrome, Acrobat Reader |
| Storage | S3‑compatible, GCS |

---

## Prioritization (how to choose)

- **User share** (recent analytics).  
- **Revenue/impact** (checkout paths).  
- **Regulatory** (a11y, payments).  
- **Change risk** (new layout engine, new app feature).  
- **Support cost** (top support cases).

---

## Execution strategy

- **Device lab** (real devices) + **cloud device farms**.  
- **Emulators/simulators** for breadth; verify T0 on **real hardware**.  
- **Network shaping**: OS tools or proxies.  
- **Automation**: smoke on every PR; deeper suites nightly/weekly.  
- **Visual testing**: per breakpoint; approve diffs.  
- **A11y**: rule‑based checks + manual spot checks.

---

## Anti‑patterns

- Bloated matrices with **hundreds** of low‑value combos.  
- Only testing **desktop Chrome**.  
- Ignoring **WebView/in‑app browsers**.  
- No **RTL/CJK** coverage.  
- No **network shaping**; only Wi‑Fi.  
- Treating **a11y** as optional.

---

## Review checklist (quick gate)

- [ ] T0/T1/T2 defined; **representatives** selected per dimension  
- [ ] Pairwise set produced; **triples** added for risky interactions  
- [ ] **A11y** & **Dark mode** included in T0  
- [ ] **Network shaping** added to each platform set  
- [ ] **Evidence capture** plan: screenshots, DOM, a11y, HAR, traces  
- [ ] **Internationalization**: RTL, CJK, numeric/date formats  
- [ ] **Mobile**: deep links, safe area, push prompts, WebView quirks  
- [ ] **Integrations**: at least one per category exercised  
- [ ] Automation cadence: PR smoke, nightly breadth, weekly long tail  
- [ ] Rotation plan for T2 long tail

---

## CSV seeds

**Devices**

```csv
id,device,os,notes
D01,iPhone 15,iOS latest,"T0"
D02,Pixel 8,Android latest,"T0"
D03,Galaxy S23,Android latest-1,"T1 OEM"
D04,iPad (11"),iPadOS latest,"T1"
D05,Windows 11 Laptop,Windows 11,"T0 Chrome/Edge"
D06,MacBook Air,macOS latest,"T0 Safari"
```

**Browsers**

```csv
id,browser,engine,channel
B01,Chrome,Blink,Stable
B02,Safari,WebKit,Stable
B03,Edge,Blink,Stable
B04,Firefox,Gecko,Stable
```

**Locales**

```csv
id,locale,script,rtl
L01,en-US,Latin,false
L02,fr-FR,Latin,false
L03,zh-CN,Han,false
L04,ja-JP,Kana/Han,false
L05,ar-SA,Arabic,true
```

**Networks**

```csv
id,profile,rtt_ms,down_mbps,up_mbps,loss
N01,Offline,inf,0,0,100%
N02,3G,300,1.5,0.75,1%
N03,4G,80,10,5,0.1%
N04,Wi‑Fi,20,100,50,0%
```

**A11y**

```csv
id,tool,platform,notes
A01,VoiceOver,iOS,"T0"
A02,TalkBack,Android,"T0"
A03,NVDA,Windows,"T1"
A04,JAWS,Windows,"T2"
```

**Runs (pairwise sample)**

```csv
run,device,browser,locale,network,theme,a11y
R01,D01,B02,L01,N03,Dark,VoiceOver
R02,D02,B01,L05,N02,Light,TalkBack
R03,D05,B01,L03,N03,Dark,Keyboard
R04,D06,B02,L02,N04,Light,NVDA
R05,D03,B01,L01,N03,Dark,None
R06,D04,B02,L04,N04,Light,None
```

---

## Templates

**Matrix row (short)**

```
Device × Browser × Locale × Network (+ Theme/A11y)
```

**Run record**

```csv
timestamp,env,commit,run_id,notes
2025-09-16,staging-3,abc123,CM-2025-09-16-01,"T0 sweep"
```

**Bug report fields**

```
env: device/os/browser/locale/network/theme/a11y
steps: short
expected: short
actual: short
evidence: screenshots + HAR + a11y report + console logs
```

---

## Links

- Pairwise: `../20-techniques/pairwise.md`  
- State Models (flows): `../20-techniques/state-models.md`  
- i18n playbook: `../55-domain-playbooks/i18n-l10n.md`  
- Performance p95/p99: `./performance-p95-p99.md`  
- Security: `./security-essentials.md`  
- Resiliency & Timeouts: `./resiliency-and-timeouts.md`

---

<p align="center">
  <sub>
    Built with <a href="https://treeifyai.com">Treeify</a> — AI test case design copilot.<br>
    <em>Structured, traceable, high-coverage test design — faster.</em>
  </sub>
</p>
