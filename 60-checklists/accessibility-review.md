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


# Accessibility Review Checklist

> Accessible by default. Measurable at review.  
> Use this checklist to verify **WCAG 2.2 AA** coverage, catch regressions early, and attach **evidence** that design, code, and QA can all trust.

---

## TL;DR

- Treat a11y like **functional acceptance**: write **MAE** (Main/Alt/Exception) cases and oracles.  
- Cover **keyboard**, **focus**, **semantics/landmarks**, **labels**, **error messaging**, **contrast**, **motion**, **media**, **touch**, **RTL**, and **text scaling**.  
- Test with **axe/pa11y** and at least **one screen reader** (NVDA/VoiceOver), plus **reduced-motion** and **high-contrast** modes.  
- Capture **evidence**: annotated screenshots, focus-order map, contrast checks, SR transcripts, and tool reports.

Links:  
- A11y fundamentals → `../50-non-functional/accessibility-a11y.md`  
- Design/UX guidance → `../57-cross-discipline-bridges/for-design-ux.md`  
- Mobile-first flows → `../55-domain-playbooks/mobile-first-flows.md`  
- i18n/RTL → `../55-domain-playbooks/i18n-l10n.md`  
- Compatibility matrix → `../50-non-functional/compatibility-matrix.md`

---

## Preconditions (before testing)

- [ ] Designs include **Empty/Loading/Error** states and a **focus order** map.  
- [ ] Components use **semantic HTML** or mapped roles; ARIA only to enhance, not replace semantics.  
- [ ] **Message IDs** paired with error copy; error taxonomy defined.  
- [ ] Contrast tokens provided for light/dark; motion tokens include reduced-motion variants.  
- [ ] Test environment accessible without production data; feature flags documented.

---

## 1) Keyboard & focus

- [ ] All interactive elements are reachable by **Tab/Shift+Tab**; no keyboard traps.  
- [ ] **Visible focus** indicator on every interactive element (not color-only).  
- [ ] **Logical order** follows reading order; skip links present.  
- [ ] **Esc** closes modals/popovers; focus returns to the invoking control.  
- [ ] Focus **not stolen** on content updates except where announced and expected.  
- [ ] Arrow-key patterns honored (menus, radio-groups, tabs, lists, grids).

**Evidence**: focus-order map screenshot with indices; keyboard walkthrough video.

---

## 2) Semantics, landmarks, and structure

- [ ] One `<h1>` per page; headings in order (`h2`, `h3`...).  
- [ ] Landmarks used: `header`, `nav`, `main`, `aside`, `footer`.  
- [ ] **Role**/name/state/value exposed correctly (native where possible).  
- [ ] Decorative images `alt=""`; meaningful images have descriptive `alt`.  
- [ ] Dynamic content uses **live regions** only when needed; polite vs assertive chosen thoughtfully.

**Evidence**: Accessibility tree snapshot; axe report excerpt.

---

## 3) Forms, labels, and errors

- [ ] Every input has a **programmatic label** and **described-by** help text as needed.  
- [ ] Required fields marked in text and programmatically (not color-only).  
- [ ] Validation errors:  
  - Announced by SRs, linked via `aria-describedby`.  
  - Focus moves to first invalid control; summary at top lists all errors.  
  - Copy references **message IDs** and is actionable.  
- [ ] Inputs support paste and IME composition; masks don’t block entry.

**Evidence**: SR transcript navigating to errors; screenshots of inline + summary.

---

## 4) Color, contrast, and motion

- [ ] Text contrast: **4.5:1** normal, **3:1** large.  
- [ ] Non-text UI components contrast: **≥ 3:1**.  
- [ ] No color-only meaning; add icons/text/aria.  
- [ ] Respects `prefers-reduced-motion`; provide non-animated equivalents.  
- [ ] Animations ≤ 200–300ms, no seizure-triggering patterns.

**Evidence**: token contrast table snapshot; reduced-motion recording.

---

## 5) Components with special rules

**Modals/Dialogs**  
- [ ] Focus trapped inside; background inert; labeled by title.  
- [ ] Close with Esc and a visible button; return focus on close.

**Menus/Combobox**  
- [ ] Arrow navigation; Home/End; type-ahead; `aria-expanded`/`aria-controls`.  
- [ ] Options are elements with correct roles; active option announced.

**Tabs**  
- [ ] Arrow navigation within tablist; selected tab has `aria-controls` and `aria-selected`.

**Tables/Data grids**  
- [ ] Header cells as `<th>` with scope; sortable columns have state announced.  
- [ ] Virtualized lists/grids preserve semantics and focus order.

**Toasts/Alerts**  
- [ ] `role="status"` (polite) or `role="alert"` (assertive) used correctly.

**Evidence**: component-by-component checklist; SR transcript for at least one.

---

## 6) Media, files, and graphics

- [ ] Video has **captions**; audio has **transcripts**.  
- [ ] Autoplay off by default; if autoplay, **muted** + controls + pause.  
- [ ] PDFs are tagged and accessible; offer HTML alternative when feasible.  
- [ ] Canvas/SVG content has accessible name/desc and keyboard affordances if interactive.

**Evidence**: caption track screenshot; PDF tag tree sample.

---

## 7) Mobile specifics

- [ ] Touch targets **≥ 44×44 px**; adequate spacing.  
- [ ] **Dynamic Type** / OS font scaling ×1.3 works without clipping.  
- [ ] Screen reader (VoiceOver/TalkBack) order matches visual order; rotor actions named.  
- [ ] System colors respected in dark mode; high-contrast modes checked.  
- [ ] Native controls used when possible; custom controls expose roles/states.

**Evidence**: device screenshots (light/dark, font ×1.3), SR short video.

---

## 8) Localization, RTL, and numeric direction

- [ ] RTL mirrored layouts; icons that imply direction are flipped.  
- [ ] Numbers remain **LTR** within RTL contexts when appropriate.  
- [ ] Pluralization via ICU MessageFormat; dates, numbers, currency localized.  
- [ ] Long strings don’t clip; CJK line breaking OK.

**Evidence**: screenshots of `en`, `ar`, `ja` (or target locales).

---

## 9) Performance & resilience interaction

- [ ] Skeletons instead of spinners; progress for known durations.  
- [ ] Errors non-blocking where possible; retry keeps focus and inputs.  
- [ ] Loading states don’t cause **layout shifts** that confuse keyboard/SR users.

**Evidence**: Lighthouse/INP/CLS snapshot; retry flow recording.

---

## MAE Scenarios (accessibility)

### A11Y-001 **Main** — Form submission with one invalid field
- **Expected**: first invalid field focused; error announced; summary lists all.  
- **Oracles**: SR transcript; `aria-describedby` links; focus ring visible.

### A11Y-002 **Alt** — Dialog interaction via keyboard only
- **Expected**: open → interact → close; focus trapped and restored; Esc works.  
- **Oracles**: keyboard recording; SR labels read correctly.

### A11Y-101 **Exception** — Reduced motion on animated list
- **Expected**: no shimmer/slide; static skeleton; performance unchanged.  
- **Oracles**: media query active; no CSS transitions applied.

### A11Y-102 **Exception** — RTL with data entry
- **Expected**: layout mirrored; numeric inputs LTR; validation messages read once.  
- **Oracles**: screenshots; SR transcript shows correct order.

---

## Review checklist (quick gate)

- [ ] Keyboard reachable; focus visible; logical order + skip links  
- [ ] Landmarks & headings; accessible names/roles/states  
- [ ] Forms: labels, required indicators, helpful errors linked and announced  
- [ ] Contrast passes; no color-only meaning; reduced-motion respected  
- [ ] Modals/menus/tabs/tables follow patterns; toasts/alerts announced correctly  
- [ ] Media accessible (captions/transcripts); PDFs tagged or HTML alt provided  
- [ ] Mobile: touch targets, Dynamic Type, SR ordering, dark/high-contrast modes  
- [ ] RTL and localization behave; numbers/dates/currency localized  
- [ ] Evidence attached: focus map, SR transcript, axe/pa11y report, contrast snapshot, videos

---

## CSV seeds

**Focus order (example)**

```csv
screen,tab_index,control
Checkout,1,Email input
Checkout,2,Card number input
Checkout,3,Expiry input
Checkout,4,CVC input
Checkout,5,Place order (button)
```

**Contrast tokens (sample)**

```csv
token,foreground,background,ratio
text.primary,#1F2937,#FFFFFF,12.7
text.muted,#6B7280,#FFFFFF,5.4
button.primary.text,#FFFFFF,#2563EB,8.2
button.secondary.text,#111827,#E5E7EB,7.1
```

**SR test set**

```csv
screen,reader,platform,flow
Form error,NVDA,Windows,"tab through -> submit invalid -> errors"
Dialog,VoiceOver,macOS,"open -> navigate -> close"
Mobile list,TalkBack,Android,"swipe next -> announce counts"
```

**Axe/pa11y runs**

```csv
page,tool,violations
/account,axe,0
/checkout,axe,2
/search,pa11y,1
```

---

## Templates

**A11y spec (per screen)**

```
Screen: <name>
States: Normal | Empty | Loading | Error | Disabled | Partial
Focus order: <1..n> (map attached)
Keyboard: <keys + patterns>
Labels: <inputs + described-by ids>
Announcements: <live regions, alerts, toasts>
Contrast: <tokens used + ratios>
Motion: <animations + reduced-motion variant>
Localization: <locales + RTL notes>
Evidence: <screenshots, SR transcript, axe report>
```

**SR transcript (short)**

```
Reader: <NVDA|JAWS|VoiceOver|TalkBack>
Platform: <Windows|macOS|iOS|Android>
Flow: <what you’re testing>

Narration:
- "Heading level 1, Checkout"
- "Edit text, Email, required"
- "Error, Email: Please enter a valid address"
- "Button, Place order"
```

---

## Sign-off

- [ ] All a11y checks pass for target tiers (see matrix).  
- [ ] Evidence attached and archived with the PR.  
- [ ] Owners signed (Design/UX + Eng + QA).  
- [ ] Post-merge monitoring includes basic a11y synthetics (axe on key pages).

---

<p align="center">
  <sub>
    Built with <a href="https://treeifyai.com">Treeify</a> — AI test case design copilot.<br>
    <em>Structured, traceable, high-coverage test design — faster.</em>
  </sub>
</p>
