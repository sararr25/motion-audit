# Motion Audit Playbook

Nine categories, what to look for, and **the exact value to write in the fix**. Never
approximate a value that appears here — copy it. Rule substance after Emil Kowalski
([emilkowal.ski](https://emilkowal.ski/)); categories 6b and 9 and the cures are additions.

Every category below gives both the **smell** (what to hunt) and the **cure** (what to
write instead). A finding without a cure is not finished.

---

## 1. Purpose & frequency

Every animation must answer "why does this animate?" — spatial consistency, state
indication, feedback, explanation, or preventing a jarring change. "It looks cool" on a
frequently-seen element is not a purpose.

**The frequency ladder depends on the product type from recon. Use the right column.**

| How often the user sees it | `app` | `marketing` | `docs` |
| --- | --- | --- | --- |
| Keyboard shortcut / command palette / 100+ a day | **No animation, ever** | n/a — these rarely exist | **No animation** |
| Tens of times a day (list hover, nav, toggles) | Remove or drastically reduce | Reduce; keep it under 200ms | Remove |
| Several times a session (menus, accordions, modals) | Standard animation | Standard animation | Minimal |
| Once per visit (hero, scroll reveals, section entrances) | Reduce — users return often | **Standard; this is the budget** | Minimal |
| Rare / first-time (onboarding, success, celebration) | Delight allowed | Delight allowed | Rarely |

On a `marketing` site, **do not claim an element is hit "100+ times a day"** — a visitor
sees the hero once. The high-frequency surfaces on a marketing site are: the header nav,
the mobile menu toggle, hover states on cards and buttons, and an accordion. That is all.

- **Smell**: motion on keyboard-initiated actions; a command palette with an open/close
  transition (Raycast has none — correct); decorative loops on elements the user passes
  constantly; a hover effect that re-triggers heavy motion on text the user is reading;
  multiple infinite loops stacked in one viewport.
- **Cure, in order**: delete it → reduce it (shorter, smaller, fewer properties) → gate it
  (first visit only, or `once: true`) → keep it.
- **Infinite ambient loops**: cap at **two per viewport**, pause them when off-screen
  (`IntersectionObserver`, or ScrollTrigger `onToggle`), and never run them behind text the
  user is expected to read.

## 2. Easing & duration

Decision order for easing:

- Entering or exiting → **`ease-out`**
- Moving / morphing on screen → **`ease-in-out`**
- Hover / colour change → **`ease`** is acceptable; a token is better
- Constant motion (marquee, progress) → **`linear`**
- Scroll-scrubbed motion → **`linear` / `none`** (the scroll drives it; this is correct, never a finding)
- Ambient infinite yoyo → **`ease-in-out` / `sine.inOut`** (correct, never a finding)
- Default → **`ease-out`**

**`ease-in` on UI is always a finding.** It starts slow, delaying the moment the user is
watching. Built-in CSS easings are weak for deliberate motion.

**Cure — the canonical token set.** Define once, reference everywhere:

```css
:root {
  --ease-out:    cubic-bezier(0.23, 1, 0.32, 1);     /* UI enter/exit */
  --ease-in-out: cubic-bezier(0.77, 0, 0.175, 1);    /* on-screen movement */
  --ease-drawer: cubic-bezier(0.32, 0.72, 0, 1);     /* iOS-like drawer */
  --dur-press:   140ms;
  --dur-tooltip: 160ms;
  --dur-menu:    200ms;
  --dur-modal:   300ms;
}
```

Duration budgets — **UI stays under 300ms**:

| Element | Duration |
| --- | --- |
| Button press feedback | 100–160ms |
| Tooltips, small popovers | 125–200ms |
| Dropdowns, selects, menus | 150–250ms |
| Modals, drawers | 200–500ms |
| Accordion / disclosure | 200–300ms |
| Marketing entrance, scroll reveal | 400–900ms (exempt from the UI budget) |

- **Smell**: `ease-in` anywhere on UI; bare `ease`/`linear` on an entrance; UI over 300ms;
  a tooltip delay *and* an animation on every tooltip in a toolbar (after the first they
  should be instant); ten hand-typed durations 20ms apart.
- **Cure**: swap to the token above matching the element's row. Consolidate near-duplicates
  into one token rather than preserving the spread.

## 3. Physicality & origin

- **Never `scale(0)`** — nothing appears from nothing. Use `scale(0.9–0.97)` + `opacity: 0`.
- **Popovers / dropdowns / tooltips scale from their trigger**, not the centre:
  `transform-origin: var(--transform-origin)` (Base UI/Radix expose this).
  **Modals are exempt** — centred is correct. Do not report them.
- **Press feedback** on every pressable element:

```css
.button { transition: transform var(--dur-press) var(--ease-out); }
.button:active { transform: scale(0.97); }   /* keep 0.95–0.98 */
```

  A link whose only feedback is colour is acceptable. A button-shaped or card-shaped
  target with no `:active` state is a finding. **Then check category 9** — a press rule
  that loses the cascade is worse than none, because it looks handled.

- **Smell**: `scale(0)`; pure-fade entrances with no initial transform; `transform-origin:
  center` (or none) on a trigger-anchored element; pressables with no `:active`.
- **Cure**: the snippets above. For an entrance, pair opacity with a `y`/`scale` offset,
  and prefer `translateY(100%)` (self-relative) or `clip-path: inset()` over pixel offsets.

## 4. Interruptibility

CSS **transitions** retarget from the current state; **keyframes** restart from zero.
Anything triggered rapidly or reversible mid-motion must use transitions or springs.

- Entry without JS: `@starting-style`.
- Gesture-driven motion uses **springs** — they carry velocity when interrupted.
  Apple-style default: `{ type: "spring", duration: 0.5, bounce: 0.2 }`. Keep bounce
  0.1–0.3; save visible bounce for drag-to-dismiss and playful moments.
- **Asymmetric timing**: the deliberate phase (press, hold, destructive confirm) is
  slower; the system's response snaps. Symmetric press-and-release is a finding.
- **Exit animations**: a panel that animates in and is then unmounted instantly is a
  finding. Either animate both or neither.

- **Smell**: `@keyframes` on toasts, toggles, menus, accordions; a conditionally-rendered
  panel with an enter keyframe and no exit; fixed-duration tweens in a gesture handler;
  repeated triggers stacking tweens with no `overwrite`/kill; drags dismissed on distance
  alone (use velocity: `Math.abs(distance)/elapsedMs > ~0.11`); hard stops at drag bounds
  instead of rising friction.
- **Cure**: keyframes → transitions; add `overwrite: "auto"` (or kill the previous tween)
  to anything re-triggerable; give every animated-in panel a matching exit.

## 5. Performance

- **Animate `transform` and `opacity` only.** `width`/`height`/`margin`/`padding`/
  `top`/`left` trigger layout, paint and composite.
- **`transition: all`** animates unintended properties off the GPU — always a finding.
- **Trust the browser over the rulebook.** If Phase 2 reported an animation as
  non-composited, that is a confirmed finding with the browser's own reason, even if the
  property looked fine in CSS. Common culprits the rulebook misses: `stroke-dashoffset`,
  `clip-path`, `filter`, and anything on an element with an invalid compositing state.
- **`clip-path` on large containers** repaints big subtrees every frame. Acceptable for a
  one-shot reveal on a small element; a finding on a full-width section.
- Don't drive child transforms from a CSS variable on the parent — it recalcs every child.
- Keep transition-time `filter: blur()` under 20px. Static blur is cheaper but still
  costs on a scrolling page; flag large static blurs only alongside a measured problem.
- CSS and WAAPI beat rAF-based JS under load. Use CSS for predetermined motion; JS and
  springs for dynamic, interruptible, gesture-driven motion.
- **Re-implementing a CSS animation in JS is a finding** — if a keyframe already does it,
  the JS version costs main-thread time for nothing.

- **Cure**: name the properties; move to `transform`/`opacity`; for SVG draws prefer
  `clip-path`/opacity or accept the cost knowingly; replace a JS ambient loop with the
  CSS keyframe it duplicates.

## 6a. Accessibility — reduced motion

Reduced motion means **fewer and gentler**, not zero. Keep what aids comprehension,
remove position changes.

**The blanket reset is an anti-pattern. Do not write this:**

```css
/* WRONG — kills every transition on the site, including focus rings */
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: .01ms !important;
    transition-duration: .01ms !important;
  }
}
```

**Write this instead:**

```css
@media (prefers-reduced-motion: reduce) {
  html { scroll-behavior: auto; }

  /* Drop movement, keep a short fade so state changes stay legible */
  .entrance, .reveal, [data-animate] {
    animation: none;
    opacity: 1;
    transform: none;
  }
  .entrance { transition: opacity 150ms ease; }

  /* Kill ambient loops outright */
  .float, .halo, .orb, .marquee { animation: none; }

  /* KEEP colour, border, background and focus-ring transitions as they are. */
}
```

In JS, branch the values rather than skipping setup entirely:

```js
// GSAP
const media = gsap.matchMedia();
media.add("(prefers-reduced-motion: reduce)", () => {
  gsap.set(targets, { clearProps: "all" });
  gsap.from(targets, { autoAlpha: 0, duration: 0.2 });  // fade, no movement
});
media.add("(prefers-reduced-motion: no-preference)", () => { /* full motion */ });
```

- **Smell**: movement with no reduced-motion handling at all; the blanket `*` reset; a
  JS gate that early-returns and leaves zero feedback; a gate that covers the JS timeline
  but not the CSS hovers on the same page; an explicit `behavior: "smooth"` in JS
  overriding the CSS `scroll-behavior: auto`.
- **Verify by emulation in Phase 2**, not by grepping for the media query.

## 6b. Accessibility — the rest of motion a11y

The reduced-motion rule is not the whole of accessible motion.

- **Focus visibility**: a blanket reduced-motion reset also flattens focus-ring
  transitions. Any element with a motion hover needs an equally visible `:focus-visible`.
- **Hover gating**: touch fires a false hover on tap and it sticks.

```css
@media (hover: hover) and (pointer: fine) {
  .card:hover { transform: translateY(-4px); }
}
```

- **False affordance**: a hover lift on a non-interactive element (an `<article>` with no
  link or handler) promises a click that does not exist. Either make it interactive or
  remove the lift.
- **Keyboard parity for gestures**: anything draggable or swipeable needs a keyboard path
  (arrow keys) and a visible focus state.
- **Native elements that snap**: `<details>`/`<summary>` jumps its own height and shoves
  the content below it. Animating only the inner content does not fix the jump.
- **Target size**: a pressable needs ~44px minimum; a press animation on a 20px target
  is not the problem to solve first.

## 7. Cohesion & tokens

- Motion matches the product's personality — playful can be bouncier, a dashboard stays
  crisp. One bouncy component in a crisp app is a finding; bounce in a playful brand is not.
- Curves and durations live as **shared tokens**. Five hand-typed cubic-beziers that
  almost match is a consolidation finding.
- **CSS and JS must share one scale.** If the stylesheet has
  `--ease-out-quart` and the JS uses `power3.out`, that is two vocabularies for one intent.
  Pick one source of truth and derive the other.
- Group entrances get a **30–80ms stagger**. Above ~100ms it reads as slow and sequential.
  Stagger is decorative — it must never block interaction.
- A jarring crossfade that double-exposes two states can be masked with
  `filter: blur(2px)` during the transition.

**Always output the inventory**: every distinct easing and every distinct duration, with
the files each appears in. It makes near-duplicates visible and it is the most reusable
artifact the audit produces.

## 8. Missed opportunities

Additive, not corrective. Report at most a handful, grounded in seams actually observed.

- State changes that teleport (a content swap, a success message appearing and shoving the
  layout) where a brief transition would prevent a jarring change.
- Spatially-connected UI (a menu growing from its toggle) with no motion explaining where
  it came from.
- Rare, high-emotion moments (first run, form success, celebration) rendered with none of
  the delight budget they are allowed.

## 9. Cascade & precedence — does the rule actually win?

**The category most audits skip.** A rule that exists but loses is worse than a missing
rule, because a code search ticks the box and nobody looks again.

Check, for every motion rule a finding depends on:

1. **Declaration order at equal specificity** — the later rule wins. The classic failure:

```css
.button:active { transform: scale(.98); }            /* declared first  */
@media (hover: hover) and (pointer: fine) {
  .button:hover { transform: translateY(-2px); }     /* declared later → WINS */
}
/* Result: on a mouse, the press state never appears. Touch-only feedback. */
```

   **Cure**: raise the press state's precedence and make it co-exist with hover:

```css
@media (hover: hover) and (pointer: fine) {
  .button:hover  { transform: translateY(-2px); }
  .button:active { transform: translateY(0) scale(.97); }  /* after hover */
}
```

2. **Inline styles beat the stylesheet.** A JS animation library writing `transform` inline
   (GSAP without `clearProps`, any `.set()`/`.from()` left in place) permanently disables
   every CSS `:hover { transform }` on that element. Check every element touched by both.
3. **`!important` in a hover rule** overriding a JS-driven transform — the element jumps
   when hovered mid-animation.
4. **Two systems on one property.** A CSS `transition: transform` on an element whose
   transform is written every frame by a scroll scrub means each write starts a new
   transition: the result lags and jitters. One owner per property.
5. **Specificity inversion** — a token-based rule outranked by a later, more specific
   hand-written one, so the token is never used.
6. **Media-query shadowing** — a rule inside `(hover: hover)` or a breakpoint silently
   replacing the base rule on some devices only.

**How to check properly**: read declaration order in the source, and confirm against the
**shipped** stylesheet by listing matched rules in document order in the browser. Do not
reason about specificity alone; order decides ties and ties are the common case.
