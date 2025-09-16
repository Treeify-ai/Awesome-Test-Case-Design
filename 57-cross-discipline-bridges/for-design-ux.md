# For Design/UX — Error States, Empty States, A11y

> Design is how it **fails gracefully**.  
> This bridge helps Designers ship UI that **degrades well**, is **accessible by default**, and comes with **testable specs** that engineering can implement and QA can verify.

---

## TL;DR (defaults we recommend)

- **Design for MAE**: show **Main**, **Alternative**, and **Exception** for each flow—with copy, visuals, and recoveries.  
- **A11y first**: keyboard reachability, visible focus, headings/landmarks, contrast, labels, and screen reader notes.  
- **Empty → Loading → Error**: specify all three; prefer **skeletons** over spinners; show next action in each.  
- **Responsive**: mobile first (one-hand reach), tablet, desktop; specify breakpoints and behavior, not just sizes.  
- **Tokens, not pixels**: color/typography/spacing componentized; dark mode parity.  
- **Microcopy**: actionable, specific, and paired with **message IDs** to align with error taxonomy.  
- **Evidence**: attach annotated screenshots for MAE screens and record focus order for a11y.

---

## What to spec (and how to make it testable)

1. **States** for each screen: Normal, **Empty**, **Loading**, **Error**, **Partial** (some data), **Disabled**.  
2. **Controls**: buttons/links/toggles with enabled/hover/focus/active/disabled.  
3. **Focus order** and **keyboard shortcuts** (if any).  
4. **Copy** with **message IDs** (not just visible text).  
5. **Content limits**: truncation, wrapping, clamping, and overflow handling.  
6. **Responsive behavior** at named breakpoints.  
7. **Motion** and **reduced-motion** alternatives.  
8. **Evidence list** per flow (what to capture in QA).

---

## Error states (practical patterns)

**Principles**

- **Name the problem**, **say what happened**, **what to do next**.  
- Use **short titles** + **body** + **action**. Map to **message IDs** (see Error Taxonomy).

**Error types & defaults**

- **Validation**: inline near field; summary at top; focus returns to first error.  
- **Policy/Permissions**: explain lack of access; offer “Request access” or contact.  
- **Network/Timeout**: offer **Retry**; keep user input; auto-retry at most once.  
- **Dependency down**: degrade (brownout) with reduced features; show status.  
- **Not found**: show safe fallback (e.g., list) and a “Try again” CTA.

**Acceptance oracles**

- Error is **programmatically associated** with control; screen reader **announces** it.  
- Focus moves to the first error; **visible** outline.  
- Retry keeps state and is **idempotent**.

---

## Empty states (make them useful)

**Types**

- **First-time**: teach; short how-to; link to import/sample data.  
- **Cleared/No results**: suggest filters, example queries, recent items.  
- **After action**: confirm and show next best step.

**Design rules**

- Keep **illustrations optional** and content-light; ensure dark mode contrast.  
- Show **primary CTA** and at most one **secondary**.

**Acceptance oracles**

- Empty states appear only when **datasets are truly empty**; no flashes during loading.  
- CTA is reachable via **keyboard** and announced properly.

---

## Loading states (reduce anxiety)

- Prefer **skeletons** for lists/cards; reserve spinners for short, blocking actions.  
- Use **progress** where known (uploads, imports).  
- Respect **`prefers-reduced-motion`**; offer a static placeholder.  
- Never block **navigation** unless required; allow **Cancel** for long operations.

**Acceptance oracles**

- Skeleton stops within expected budget; when data arrives, content replaces without layout jump.  
- Reduced motion: no shimmering animations.

---

## Microcopy & message IDs

- Write **actionable** headlines (“Can’t connect to payments”) and **specific** remedies (“Check your network or retry”).  
- Pair each visible string with a **message ID** to keep contracts stable across locales.  
- Avoid blamey tones; keep error text **scannable**.

**Template**

```
message_id: checkout.payment.timeout
title: "Payment is taking too long"
body: "We didn't hear back from the bank. You can retry or use another method."
primary_cta: "Retry"
secondary_cta: "Choose another method"
```

---

## Accessibility (must-haves for every screen)

- **Keyboard**: Tab order follows reading order; all controls operable; **focus visible**.  
- **Labels**: form inputs have labels and descriptions; errors linked with `aria-describedby`.  
- **Structure**: logical headings; `main/nav/header/footer` landmarks; **Skip to content**.  
- **Contrast**: text ≥ **4.5:1** (normal), **3:1** (large); interactive components ≥ **3:1**.  
- **Motion**: honor reduced motion; avoid auto-animating layout.  
- **Media**: alt text for images (or `""` if decorative); captions/transcripts.  
- **RTL**: mirror layout and icons; isolate mixed-direction text.

See detailed checklist: `../50-non-functional/accessibility-a11y.md`.

---

## Responsive & mobile-first

- Breakpoints named (e.g., `xs, sm, md, lg`) with intent (“1 column list at `xs`; 2 columns at `md`”).  
- **Thumb reach**: primary actions near bottom on mobile; **≥ 44×44 px** touch targets.  
- Test **Dynamic Type** / OS font scaling; ensure layout survives **×1.3** text.  
- Handle **orientation** changes; safe areas (notches, home indicator).

Link: `../55-domain-playbooks/mobile-first-flows.md`.

---

## Motion, feedback, and haptics

- Use motion to **confirm** actions and **orient**; not for flair.  
- Keep durations **≤ 200–300 ms**; set **easing** tokens; provide **reduced-motion** variants.  
- Haptics for irreversible actions; never vibrate for errors repeatedly.

---

## Forms that help (and don’t fight)

- Inputs grouped with **fieldset/legend**; inline hints; examples; character counters.  
- **Required** fields marked in text and programmatically.  
- **Input masks** that don’t block pasting; show format as the user types.  
- **Save** affordances (auto-save or explicit); show **last saved**.

---

## Evidence package (what to attach with designs)

- **Annotated screenshots**: MAE states with IDs and expected behaviors.  
- **Focus order map**: numbers next to tabbable items.  
- **Contrast spot-checks** for key tokens.  
- **Responsive storyboard** across breakpoints.  
- **Reduced-motion** variants.  
- **Localization samples**: long German strings, CJK, RTL screenshots.  
- **Empty/loading/error** triptych for lists, forms, details.

---

## MAE Scenarios (example)

### UX-001 **Main** — Search results found
- **Expected**: results list with total count, filters accessible; initial focus on query.  
- **Oracles**: heading levels correct; landmarks present; list virtualized without trapping focus.

### UX-002 **Alt** — No results (empty)
- **Expected**: “No results” with suggestions and a “Clear filters” CTA.  
- **Oracles**: screen reader announces “No results”; CTA tabbable.

### UX-101 **Exception** — Network timeout
- **Expected**: non-blocking banner with “Retry”; preserves inputs and filters.  
- **Oracles**: retry idempotent; banner dismissible; message ID logged.

### UX-102 **Exception** — Form validation
- **Expected**: first invalid field focused; error text linked; summary at top.  
- **Oracles**: `aria-describedby` points to error; keyboard loop intact.

---

## Review checklist (quick gate)

- [ ] MAE states designed with copy, visuals, and actions  
- [ ] Empty/Loading/Error present for lists, forms, and details  
- [ ] A11y: keyboard, focus, structure, labels, contrast, motion  
- [ ] Responsive: breakpoints + behaviors; thumb reach; touch target sizes  
- [ ] Microcopy paired with **message IDs**; no color-only messaging  
- [ ] RTL & long-text samples; truncation rules documented  
- [ ] Dark mode tokens defined; contrasts rechecked  
- [ ] Evidence kit attached (annotated shots, focus map, reduced-motion, locale set)

---

## CSV seeds

**Message IDs (snippet)**

```csv
id,type,text
search.empty.title,title,"No results"
search.empty.body,body,"Try different keywords or clear filters."
error.timeout.title,title,"Connection is slow"
error.timeout.body,body,"We didn't hear back. Retry?"
form.error.required,inline,"This field is required"
```

**Breakpoints**

```csv
name,min_width_px,columns
xs,0,1
sm,480,1
md,768,2
lg,1024,3
xl,1280,4
```

**A11y focus order (example)**

```csv
screen,tab_index,control
search,1,Search input
search,2,Submit button
search,3,Filter button
search,4,Results list
```

---

## Templates

**Design spec (per screen)**

```
Screen: <name>
States: Normal | Empty | Loading | Error | Disabled | Partial
Actions: <primary/secondary and when visible>
Copy: <titles, body, hints> + message IDs
A11y: focus order, labels, headings, landmarks, reduced motion
Responsive: breakpoints + behavior notes
Evidence: screenshots (light/dark, LTR/RTL), focus map, contrast checks
```

**Component spec (control)**

```
Component: <Button/Toast/Form Field/Modal/...>
States: default, hover, focus, active, disabled
Tokens: color, typography, spacing, border radius, elevation
A11y: role, keyboard behavior, aria attributes
Usage: when to use / when not to use
```

---

## Links

- Accessibility: `../50-non-functional/accessibility-a11y.md`  
- Mobile-first flows: `../55-domain-playbooks/mobile-first-flows.md`  
- Error taxonomy & message IDs: `../40-api-and-data-contracts/error-taxonomy.md`  
- Contracts & Schemas (mark message IDs in responses): `../40-api-and-data-contracts/contracts-and-schemas.md`  
- Messaging & Notifications (empty/error communications): `../55-domain-playbooks/messaging-and-notifications.md`
