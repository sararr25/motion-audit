# Phase 2 — Measuring the live page

The point of this phase: **replace guesses with numbers, and let the browser tell you what
it cannot composite.** A reading of the CSS can say "this might shift the layout". A trace
says "this shifts it by 0.1088, on `#smooth-wrapper`, at 1,299ms". Only the second one
lets you rank.

## Choosing a target

In order of preference:

1. **The deployed URL.** Best evidence: it is what visitors actually get, including the
   real CDN, real images and the production build.
2. **A dev server that is already runnable.** Only if dependencies are installed. Never
   run an install to make this phase possible — that mutates the project. A dev build also
   has different timing from production; say so in the report.
3. **Neither** → mark the phase `unverified`, run the `static` variant, and put a banner
   at the top of the report saying nothing was measured.

Find the deployed URL in `metadataBase`, `README`, the deploy config, or ask.

## What to collect, per page

Use whichever browser tooling is available (a performance-trace tool, or
`PerformanceObserver` in the page). A **cold load with cache ignored and a reload** is the
measurement that matters; attaching an observer to an already-loaded page returns nothing
useful, because the load-phase entries are gone.

Collect:

| Metric | Why it matters here | Good |
| --- | --- | --- |
| **CLS** + per-shift culprits (element, start time, score) | The one number that catches "the layout jumped after JS ran" | ≤ 0.1 |
| **LCP** + breakdown (TTFB / render delay) | A big render delay usually means motion setup or render-blocking CSS is in the way | ≤ 2.5s |
| **INP** | Whether interaction feedback is actually prompt | ≤ 200ms |
| **Non-composited animations** + the browser's stated reason | Authoritative. Overrides any judgement made from reading CSS | none |
| **Forced reflow** | Layout thrash, often from a motion library measuring mid-frame | none |
| **Render-blocking requests** | Upstream of every motion timing | minimal |

Then, per finding you intend to report:

- **A screenshot of the element**, for the report. A non-expert recognises a picture of the
  thing far faster than a selector.
- Where a fix is visual and cheap to demonstrate, a **before/after** pair.

## Required checks beyond the trace

1. **Reduced motion.** Emulate `prefers-reduced-motion: reduce`, reload, and observe what
   actually changes. Do not infer this from grepping for the media query: the common bug is
   a query that exists and does the wrong thing (see AUDIT.md 6a).
2. **Touch / mobile width.** Emulate a phone viewport and reload, so load-time device
   gates re-run. Check whether hover states stick after a tap.
3. **Cascade confirmation** (AUDIT.md category 9). Read matched rules in document order
   off the **shipped** stylesheet, not just the source file. Listing the rules that match a
   selector, in order, with their media conditions, settles precedence questions that
   source reading leaves ambiguous.
4. **Client-side navigation.** Navigate between pages the way a visitor does (clicking, not
   reloading) and confirm the motion still initialises. Motion that works on hard load and
   dies after navigation is invisible to every static check.

## Interpreting what you get

- **An opacity flash is not a layout shift.** If a finding predicted "a flash" and CLS is
  0.00, the flash is compositor-only: real, worth mentioning, but not a High. Say which it is.
- **A non-composited animation reported by the browser is a confirmed finding**, with its
  own reason attached, even if the CSS looked acceptable. Quote the reason.
- **One element dominating CLS** is a gift: it turns a vague "the page is janky" into a
  single fix with a measured payoff. Lead the report with it.
- **No measurable problem is a real result.** If CLS is 0.00 and LCP is fast, say the site
  is healthy and keep the remaining findings honestly ranked as polish. Do not inflate
  them to justify the audit.
