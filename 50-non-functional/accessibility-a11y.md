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


# Accessibility (a11y)

> Accessibility is **product quality**. This playbook gives practical, testable guidance to make flows usable with **keyboard**, **screen readers**, **high contrast**, **reduced motion**, and **assistive tech**—and to keep them accessible as the product evolves.

---

## TL;DR (defaults we recommend)

- **Keyboard first**: everything reachable and operable with keyboard; visible **focus** always.  
- **Semantics**: native HTML elements first; ARIA only to **fix** gaps.  
- **Color**: text contrast **≥ 4.5:1** (normal) / **3:1** (large); state also indicated by **non-color**.  
- **Motion**: respect `prefers-reduced-motion`; avoid parallax/auto-anim for core tasks.  
- **Structure**: one **H1**, logical headings, landmarks (`header/main/nav/footer`), **Skip to content**.  
- **Forms**: each input has a **label**; errors are programmatically associated and announced.  
- **Media**: images need **alt**; decorative use empty alt; captions for video; transcripts for audio.  
- **Testing**: run **axe/pa11y** + manual keyboard + at least one screen reader per platform.

---

## 1) Semantics & Landmarks

- Use native elements: `<button>`, `<a>`, `<label>`, `<fieldset>`, `<table>`, `<dialog>`.  
- Landmark regions: `header`, `nav`, `main`, `aside`, `footer`.  
- **Headings**: H1–H6 in order; no skipping for visual size only.  
- **Skip link**: visible on focus, jumps to `#main`.

**Oracles**  
- DOM has correct landmarks and heading hierarchy.  
- Screen reader rotor/landmark nav shows expected regions.

---

## 2) Keyboard & Focus

- **Tab** order follows visual/logical order.  
- **Focus indicator** visible with sufficient contrast; not removed.  
- **Roving tabindex** for menus, tabs, carousels.  
- Trap focus **inside** modals; restore focus on close.  
- Keyboard shortcuts do **not** steal common OS keys; provide a list and a toggle.

**Tests**  
1. Navigate entire page with **Tab/Shift+Tab/Enter/Space/Arrows/Escape**.  
2. Open modal → focus is inside; `Esc` closes; focus returns to trigger.  
3. Hover-only menus: ensure keyboard alternative (Arrow keys) exists.

---

## 3) Forms & Validation

- Inputs have **labels** (or `aria-label`) and **descriptions** (`aria-describedby`).  
- Group related options with `<fieldset><legend>`.  
- Required fields marked in text (not just color) and via `aria-required="true"` or validation attributes.  
- Errors: **inline** (near input) + **summary** at top; programmatically associated; **screen-reader announced**.  
- Helpful **message IDs** (see error taxonomy) instead of free text; suggest fixes.

**Tests**  
- Submit empty form → focus jumps to first error; error announced.  
- Live validation (`aria-live="polite"`) announces changes without stealing focus.

---

## 4) Color, Contrast, and Non-Color Cues

- Contrast ratios: **4.5:1** (normal), **3:1** (large ≥ 18pt or 14pt bold), **3:1** for UI components/icons.  
- States (error/success/selected) also indicated with **icon/shape/text**, not just color.  
- Underlines for links in body text; avoid color-only differentiation.

**Tools**: contrast checkers in devtools; design tokens with contrast baked in.

---

## 5) Motion, Animation, and Auto-play

- Respect `prefers-reduced-motion: reduce`; disable or simplify transitions.  
- Avoid auto-animating essential layout; never auto-play audio; videos default **muted** with controls.  
- Provide **Pause/Stop/Hide** for moving content.

**Tests**  
- With reduced motion enabled, no large parallax or autoplaying animations occur.

---

## 6) Media & Images

- **Alt text** for informative images (describe **purpose**, not pixels).  
- Decorative images: `alt=""`.  
- Icons: use **`aria-hidden="true"`** if purely decorative; otherwise provide **accessible name**.  
- Videos: **captions**; audio-only: **transcripts**.  
- Charts: provide a **data table** or concise **text summary**; include **aria-describedby** pointing to it.

---

## 7) Components (patterns that work)

- **Buttons**: `<button>` (not `<div>`); disabled state announced (`disabled` attribute).  
- **Links**: meaningful text; avoid “Click here”.  
- **Tabs**: `role="tablist"`, `role="tab"`, `aria-selected`, `aria-controls`; Arrow keys change tabs.  
- **Accordion**: `<button aria-expanded>` + content with `id` referenced by `aria-controls`.  
- **Menus**: menubar/menuitem semantics, typeahead, Escape closes.  
- **Dialog/Modal**: `<dialog>` or `role="dialog"` + labelled by title; focus trap; `aria-modal="true"`.  
- **Toasts**: `role="status"` (`polite`) for non-critical; `role="alert"` (`assertive`) for critical.

---

## 8) Internationalization (i18n/l10n)

- Set **`lang`** on `<html>` and switch on locale change.  
- Support RTL (`dir="rtl"`); mirror icons/arrows; test caret movement and selection.  
- Date/time/number/currency formatting via i18n libs; avoid hard-coded formats.  
- Text expansion **×1.3** won’t break layout.

---

## 9) Accessibility for Mobile & Touch

- **Target sizes**: minimum **44×44 px** touch targets.  
- **Spacing**: avoid crowding; prevent accidental taps.  
- **Dynamic Type** (iOS) / **Font size** (Android): respect OS text scaling.  
- **Safe areas**: notches, home indicator; avoid covered controls.  
- **Gestures**: always provide alternatives; no gesture-only actions.

---

## 10) Live Regions & Async Updates

- Use `aria-live="polite"` for non-critical updates; `assertive` sparingly.  
- Announce **loading** states; avoid infinite spinners with no context.  
- For background tasks, provide **progress** and **completion** announcements.

---

## 11) Automation + Manual Testing

**Automation**  
- Run **axe**/**pa11y**/**eslint-plugin-jsx-a11y** in CI.  
- Lint color tokens for contrast; unit test accessible names.

**Manual**  
- **Keyboard pass** on core flows.  
- Screen reader sweep:  
  - **macOS/iOS**: VoiceOver  
  - **Windows**: NVDA (and optionally JAWS)  
  - **Android**: TalkBack

Record **evidence**: videos/gifs, transcripts of announcements, devtools snapshots.

---

## 12) MAE Scenarios (examples)

### S-001 **Main** — Keyboard-only checkout
- **Preconditions**: keyboard-only; reduced motion off  
- **Steps**: Tab through cart → address → payment → place order  
- **Expected**: focus visible; order completes without mouse  
- **Oracles**: video capture; focus ring visible; no dead ends

### S-101 **Exception** — Modal trap broken
- **Trigger**: open modal, press `Tab` past last control  
- **Expected**: focus loops inside; `Esc` closes  
- **Oracles**: devtools focus outline; automated check flags if focus escapes

### S-102 **Exception** — Error not announced
- **Trigger**: submit empty required field  
- **Expected**: error described via `aria-describedby`; `role="alert"` or live region announces  
- **Oracles**: screen reader output includes error text; focus moves to field

### S-103 **Exception** — Insufficient contrast
- **Trigger**: dark mode badge  
- **Expected**: contrast ≥ 4.5:1  
- **Oracles**: contrast tool reading; themed token meets ratio

---

## 13) Acceptance Gates

A feature **passes** when:

- Keyboard-only users can complete core tasks; **no keyboard traps**.  
- Screen readers announce **structure, labels, state, and errors** correctly.  
- Contrast ratios meet targets in **light & dark** themes.  
- Motion respects user preference; no autoplay audio; moving blocks are **pausable**.  
- Forms have labels/descriptions; errors are **programmatically associated** and **announced**.  
- Evidence captured and linked in PR (axe report + manual notes).

---

## 14) Anti-patterns

- Building interactive controls with `<div>`/`<span>` + click handlers.  
- Removing focus outlines without a high-contrast replacement.  
- Color-only error/success indications.  
- Placeholder text used as label.  
- Using `role="application"` on the whole app (traps users).  
- `aria-*` without the **required** keyboard semantics.  
- Modals without focus trap and labelling.

---

## 15) Review Checklist (quick gate)

- [ ] Landmarks + heading structure; **Skip to content**  
- [ ] Keyboard: operable, visible focus, no traps; modals restore focus  
- [ ] Forms: labels, descriptions, required markers, error association & announcements  
- [ ] Contrast: text, icons, controls meet ratios (light/dark)  
- [ ] Motion: honors reduced motion; no autoplay audio  
- [ ] Media: alt text; captions/transcripts; charts described  
- [ ] Components: tabs/accordion/menu/dialog follow patterns; toasts use proper roles  
- [ ] i18n: `lang` set; RTL works; text expansion ok  
- [ ] Mobile: touch target sizes; safe areas; OS text scaling  
- [ ] Live regions: used appropriately; no spammy announcements  
- [ ] Testing: axe/pa11y in CI; manual screen reader sweep captured  
- [ ] Evidence attached to PR

---

## 16) CSV Seeds

**Contrast checks**

```csv
component,theme,foreground,background,ratio,pass
Button,Light,#1E293B,#FFFFFF,5.2,true
Badge,Dark,#94A3B8,#0B1220,3.2,false
```

**Keyboard map**

```csv
component,action,keys
Modal,Close,Escape
Tabs,Next,ArrowRight
Tabs,Previous,ArrowLeft
Menu,Open,Enter/Space
Menu,Close,Escape
```

**ARIA wiring**

```csv
control,attribute,target
Input,label[for],#first-name
Input,aria-describedby,#first-name-hint #first-name-error
Dialog,aria-labelledby,#dialog-title
Tab,aria-controls,#tabpanel-1
```

---

## 17) Templates

**A11y test plan**

```
Feature: <name>
Flows: <list of tasks>
Environments: desktop (NVDA), mobile (VoiceOver)
Assistive tech: <tools>
Evidence: <links to captures + reports>
Issues: <tracked tickets>
```

**Screen reader notes (snippet)**

```
VoiceOver:
- Landmarks: Header, Navigation, Main, Footer
- Headings: Level 1 "Checkout", Level 2 "Shipping Address" …
- Form control "Email" (edit text) required; error "Enter a valid email" announced
```

---

## Links

- Error taxonomy & message IDs: `../40-api-and-data-contracts/error-taxonomy.md`  
- Compatibility matrix (devices, locales, networks): `./compatibility-matrix.md`  
- Performance (interaction budgets): `./performance-p95-p99.md`  
- Security essentials (headers, CSP): `./security-essentials.md`

---

<p align="center">
  <sub>
    Built with <a href="https://treeifyai.com">Treeify</a> — AI test case design copilot.<br>
    <em>Structured, traceable, high-coverage test design — faster.</em>
  </sub>
</p>
