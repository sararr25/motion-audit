# Rule pack — GSAP

Load when `gsap` is a dependency. These are the failures a CSS/Framer-shaped playbook
cannot express, and on a GSAP site they are usually the highest-leverage findings.

**First, what is CORRECT in GSAP and must never be reported as a finding:**

- `power2.out`, `power3.out`, `power4.out`, `expo.out` — strong ease-outs. They satisfy
  the "responsive easing" rule. They are not the weak built-in CSS easings.
- `sine.inOut` on an infinite `yoyo: true` loop — correct for ambient motion.
- `ease: "none"` on a `scrollTrigger` with `scrub` — **required**. The scroll position
  drives the motion; any other ease double-eases it.
- `gsap.matchMedia()` with a `prefers-reduced-motion` condition — a valid, live
  reduced-motion mechanism. It re-runs when the preference changes, which a one-time JS
  `matchMedia().matches` check does not.
- `power2.inOut` on a draw or an on-screen move — acceptable.

Only flag an ease if it is genuinely wrong for the job: an `*.in` ease on an entrance, or
`none`/`linear` on an unscrubbed spatial move.

---

## G1. `.from()` / `.fromTo()` on server-rendered content → flash

The single most common GSAP defect in Next/SSR apps.

`gsap.from(el, { autoAlpha: 0, y: 40 })` means: the element's *current* state is the end
state. The server already rendered it visible, so the sequence the visitor sees is
**visible → suddenly hidden → animates back in**. The longer hydration takes, the longer
the content sits there before it jumps.

- **Hunt**: every `.from()`/`.fromTo()` whose target exists in the SSR HTML; the gap
  between first paint and when motion starts (a `setTimeout` fallback, a `load` listener,
  or a custom event makes it worse — a 450ms fallback means up to 450ms of visible content
  before the jump).
- **Distinguish, using the Phase 2 trace**:
  - transform/opacity only → an **opacity flash**. Ugly, but **CLS stays 0**. Medium.
  - anything that changes layout → a **real layout shift**. Check the measured CLS and
    name the element. High.
- **Cure**: declare the initial hidden state in CSS so the server renders it hidden, then
  let GSAP animate to visible with `.to()` / `.fromTo()`; or gate the content on a
  `data-motion-ready` attribute set in the same frame the timeline is built. Never leave a
  timed fallback as the only start trigger.

## G2. A class that changes layout, added after paint → CLS

`document.documentElement.classList.add("gsap-enhanced")` (or `has-gsap`, `js-ready`, …)
in an effect, with stylesheet rules like:

```css
.gsap-enhanced .section { height: clamp(38rem, 78svh, 49rem); overflow: hidden; }
.gsap-enhanced .section-header { position: absolute; }
```

The page renders in flow layout, then re-lays out when the class lands. That is CLS, and
it is usually the site's single biggest shift.

- **Hunt**: every rule keyed on a JS-added root class; whether it touches `height`,
  `position`, `inset`, `overflow`, `display` or `grid`.
- **Confirm with the trace** — the culprit element and its score. A wrapper
  (`#smooth-wrapper`, the pinned section) scoring >0.1 on its own is a High finding.
- **Cure**: reserve the space in the base CSS (set the pinned section's height for both
  states), or add the class before first paint (an inline script in `<head>`), or accept
  the enhanced layout as the default and let the no-JS case be the fallback.

## G3. ScrollSmoother

- Is it created inside a `matchMedia` context and `kill()`ed in the cleanup?
- `smoothTouch: 0` is correct — smoothing on touch fights native momentum.
- `ignoreMobileResize: true` is correct — avoids re-layout on address-bar resize.
- The wrapper's `overflow: hidden` is static CSS: if the smoother is killed (reduced
  motion, unmount) the wrapper is no longer a scroll container. Check the page still
  scrolls in that state.
- `effects: true` makes it parse data attributes across the document — cost grows with
  page size.
- **Smoothing is applied to the highest-frequency action on the page.** Judge it under
  category 1: adding lag to every wheel event needs a real reason.

## G4. ScrollTrigger lifecycle

- Triggers created **inside** a `gsap.matchMedia()` callback are reverted by
  `media.revert()`. Triggers created **outside** it — in an event handler, a `.then()`, a
  later effect — are **not**, and leak past unmount.
- `document.fonts.ready.then(() => ScrollTrigger.refresh())` is uncancellable; it fires
  after unmount. Low severity, but it is a leak.
- **`once: true` plus a pin**: a pin adds spacer height, which moves every later trigger's
  start. If `refresh()` only runs on `load`/fonts-ready, a `start: "top 82%"` can be
  computed against pre-pin geometry. Deep-link or reload mid-page and the element can stay
  stuck in its hidden `from` state. **Hunt this explicitly** — it is a content-invisible bug.
- Any `matchMedia` condition change (a resize across a breakpoint, toggling reduced
  motion) reverts and re-runs everything: entrances replay on rotate. Check the condition
  set is as narrow as it needs to be.

## G5. React integration

- **Mounted once in the layout with `[]` deps** → on client-side route changes the effect
  never re-runs, so the next page's entrances and triggers are never set up. Motion works
  on hard load and silently vanishes after navigation. **Verify in the browser** by
  navigating client-side; do not report it from code alone.
- `useLayoutEffect` runs synchronously before paint after hydration: it blocks that paint
  with every `querySelectorAll` and `.from()` write. It avoids the G1 flash but costs
  first-paint time. State the tradeoff; do not call it simply wrong.
- An effect whose deps include fetched content (`[content]`) re-runs when the fetch
  resolves, reverting and restarting a running entrance. That is a visible restart.
- Prefer `gsap.context()` / `useGSAP()` so cleanup is automatic.

## G6. `clearProps` and inline-transform leaks

GSAP writes `transform` inline. Without `clearProps`, the inline value **permanently
outranks** every CSS `:hover { transform }` on that element.

- **Hunt**: any element that is both a GSAP target and has a CSS hover/active transform.
  This is extremely common on cards in a staggered grid reveal — the entrance works, and
  the hover lift silently never does again.
- **Cure**: `clearProps: "transform,opacity,visibility"` on the tween's completion, or
  move the hover to a property GSAP does not own, or drive the hover in GSAP too.
- Cross-reference **AUDIT.md category 9** — this is the inline-style case.

## G7. Plugins that cost per frame

- **SplitText**: `type: "words"` avoids line measurement; `"lines"` re-measures and must be
  re-split on resize (`autoSplit`). If the split runs before a CMS/fetch rewrites the text,
  the stored nodes are detached and later tweens animate nothing.
- **TextPlugin**: rewrites the text node every tick → layout every frame. With
  `immediateRender` it also replaces the SSR text with the `from` string the moment the
  timeline is built, which is a visible content flash and a reflow.
- **DrawSVG**: animates `stroke-dashoffset`, which **cannot be composited** — the browser
  will report it as a non-composited animation. Fine for one small stroke; a finding when
  several run on a scrubbed timeline.
- **MotionPath / Draggable**: per-frame main-thread work. `Draggable` without
  `InertiaPlugin` carries no release velocity and stops dead at bounds — see AUDIT.md
  category 4. Set `edgeResistance` for rising friction.
- **Count the plugins against the page's needs.** Seven registered plugins on one
  portfolio homepage is a shipping-weight finding, not just a runtime one.

## G8. Ambient `repeat: -1, yoyo: true` loops

- Count them per viewport. GSAP's ticker runs them all on the main thread forever.
- None of them pause off-screen unless you say so. Add ScrollTrigger `onToggle` or an
  `IntersectionObserver`.
- GSAP's ticker does stop when the tab is hidden, so tab-switching is not the issue.
- **If a CSS `@keyframes` already does the same motion and a `.has-gsap` rule disables it
  so GSAP can redo it in JS — that is a pure regression.** Keep the CSS version.
