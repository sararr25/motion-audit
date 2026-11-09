# Rule pack — plain CSS / Tailwind

Load when there is no JS motion library, or for the CSS half of a hybrid codebase.

## C1. Tailwind specifics

- `transition-all` is Tailwind's `transition: all` — always a finding. Use
  `transition-transform`, `transition-opacity`, `transition-colors`.
- A bare `transition-transform duration-300` with **no `ease-*` class** falls back to
  Tailwind's default `cubic-bezier(0.4, 0, 0.2, 1)` — a weak curve on deliberate motion.
  Add an `ease-*` utility or a custom one from the token set.
- `motion-safe:` and `motion-reduce:` variants exist and are almost never used. They are
  the cleanest fix for ungated animation utilities.
- Keyframes and `animation` tokens defined in `tailwind.config.js` have **no**
  reduced-motion handling and every consumer inherits that.
- **Check which config is actually live.** Multiple config files (`tailwind.config.js`,
  `.app.`, `.base.`, `.home.`) are a common mess; most are often unreferenced. Confirm via
  `postcss.config.js` and the build setup before attributing a token to a file.
- **`cdn.tailwindcss.com` in production is a separate, larger problem.** It is a CSS
  compiler running in the visitor's browser, documented by Tailwind as development-only.
  It is render-blocking and it produces CSS after parse, which is upstream of every motion
  timing you are auditing. Report it prominently even though it is not strictly a motion
  rule — nothing else you fix will matter as much.

## C2. Entry animation without JS

```css
.panel { opacity: 0; transform: translateY(8px); transition: opacity 200ms var(--ease-out), transform 200ms var(--ease-out); }
@starting-style { .panel:popover-open { opacity: 0; transform: translateY(8px); } }
```

Prefer `@starting-style` over a `data-mounted` attribute set in an effect.

## C3. Keyframe hygiene

- **Same-name keyframes defined twice with different values is a real collision.** A
  global `@keyframes spin` redefined in a page stylesheet silently changes every
  `animate-spin` on every page that imports it. Hunt duplicate `@keyframes` names across
  the whole shipped CSS and diff their bodies.
- A hover-triggered `animation:` restarts from zero on each hover and cuts dead on
  mouse-out. Hover is reversible — use a transition. If an infinite animation starts on
  hover, on touch it starts on tap and never stops.
- An `animation` and a `transition` on the same property on the same element compete.

## C4. Scroll behaviour

- `html { scroll-behavior: smooth }` applies to every in-page link and every
  `scrollIntoView`. It must be reset under reduced motion.
- An explicit `behavior: "smooth"` in JS **overrides** the CSS reset — reduced-motion users
  still get smooth scrolling. Branch it in JS.

## C5. Modern CSS worth reaching for

- `@starting-style` for entry without JS.
- `transition-behavior: allow-discrete` to animate `display`/`content-visibility`.
- `linear()` easing for a spring-like curve in pure CSS.
- `clip-path: inset()` and self-relative `translate` percentages for reveals, instead of
  hardcoded pixel offsets.
- `view-transition-name` for cross-document and SPA transitions — check it is behind a
  reduced-motion gate.
