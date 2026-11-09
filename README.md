# motion-audit

A Claude Code skill that audits the animation and motion of a website and
produces a **visual HTML report a non-developer can read**.

It is built on [Emil Kowalski's animation philosophy](https://emilkowal.ski/) —
his rules are the substance of the audit, and
[emilkowalski/skills](https://github.com/emilkowalski/skills) (MIT) is the
starting point this grew out of. Credit for the taste belongs there.

This skill adds four things a code-only audit cannot do.

---

## Why

I ran Emil's `improve-animations` skill against two real production sites and
wrote down what it got right, what it got wrong, and what it could not see at
all. Four gaps kept showing up. This skill is the answer to those four.

### 1. It measures the live page

The original reads code and never loads the site. So it can say "this *might*
shift the layout" but never "it shifts by 0.11, on this element, 1.3 seconds
in". Direction without magnitude is not prioritisation.

`motion-audit` loads the page, traces it, and attaches a real number to every
performance finding: CLS and its culprits, LCP and its breakdown, INP,
non-composited animations **with the browser's own stated reason**, forced
reflow. It also emulates `prefers-reduced-motion` and a touch device and
re-checks, instead of grepping for a media query and assuming.

On the two sites tested this changed the conclusions. One animation the
code-only audit cleared as "fine" was reported by the browser as
non-composited. A predicted "flash" turned out to be opacity-only, with CLS
0.00 — real, but not the emergency it looked like on paper.

### 2. It loads a rule pack for the stack you actually use

The original's rule catalogue speaks CSS and Framer Motion. Both sites I
tested were GSAP, and every single high-value finding came from GSAP-shaped
problems the catalogue has no rule for.

`packs/` holds one rule pack per stack, loaded after recon detects it:

- **`packs/gsap.md`** — `.from()` on server-rendered content, a layout-changing
  class added after paint, ScrollSmoother and ScrollTrigger lifecycle,
  `once: true` against pre-pin geometry, `clearProps` leaks, SplitText and
  TextPlugin per-frame cost, ambient loop budgets, React integration
  (the "mounted once with empty deps" bug that kills motion after a
  client-side navigation).
- **`packs/motion-react.md`** — hardware acceleration and the shorthand props,
  springs over durations, `AnimatePresence` exits, reduced motion, variants.
- **`packs/css.md`** — Tailwind specifics, `@starting-style`, duplicate
  keyframe names, `scroll-behavior`, and modern CSS worth reaching for.

### 3. It checks whether a rule actually wins

The category most audits skip, and the one that found the best bug in testing.

A rule that exists but loses is worse than a missing rule, because a code
search ticks the box and nobody looks again:

```css
.button:active { transform: scale(.98); }            /* declared first  */
@media (hover: hover) and (pointer: fine) {
  .button:hover { transform: translateY(-2px); }     /* declared later → WINS */
}
/* Result: on a mouse, the press state never appears. Touch-only feedback. */
```

Same blind spot hides JS animation libraries writing inline transforms that
permanently disable every CSS `:hover { transform }` on that element.
`AUDIT.md` category 9 covers declaration order, specificity, `!important`,
inline styles, and two systems fighting over one property.

### 4. The report is written for the person who owns the site

If they cannot tell what is wrong, how bad it is, and what to do, the audit
failed regardless of how good the findings are. Three rules make that work,
and the HTML is the least important of them:

1. **Every finding leads with the symptom, in plain language.** Not
   "`.button:active` loses the cascade" but "pressing a button with a mouse
   shows no reaction at all; it works on phones but not on a computer".
2. **Severity in human terms, with effort beside it.** *Breaks the feel* /
   *Looks unpolished* / *Polish*, paired with *5 min* / *30 min* /
   *a few hours*, so someone non-technical can triage without asking anyone.
3. **Every finding says how to see it yourself.** Open which page, do what,
   watch what.

The output is one self-contained HTML file: a plain-language verdict, the
measured numbers with captions explaining what each one means, an
"if you only fix three things" list, findings grouped and filterable, a
**"what's already right"** section, a full easing and duration inventory, and
an honest method-and-limits section. Technical detail sits behind a toggle.
It opens by double-click, works offline, prints to PDF, and has a dark mode.

---

## Install

```bash
npx skills@latest add sararr25/motion-audit
```

Or copy `skills/motion-audit/` into `~/.claude/skills/`.

## Use

```
/motion-audit
/motion-audit https://your-site.com
/motion-audit quick
/motion-audit deep
/motion-audit accessibility
/motion-audit static          # no runnable target; the report says so, loudly
```

It is **read-only on your source code**. It never writes into your repository,
never installs anything, and defaults its output outside the project.

## What it will not do

- It does not fix anything. It audits and reports.
- It cannot judge feel from code alone. Where a spring's bounce or a
  crossfade genuinely needs eyes, it says so and puts a feel-check in the
  report rather than guessing.
- Every finding is tagged `measured`, `confirmed` or `unverified`, and an
  unverified claim is never presented as fact. If the page could not be
  loaded, the report carries a banner saying nothing was measured.

## Files

| File | What it holds |
| --- | --- |
| `SKILL.md` | The workflow: recon → measure → audit → vet → report |
| `AUDIT.md` | Nine categories, each with the smell **and the cure** |
| `RUNTIME.md` | How to measure, what to collect, how to read it |
| `REPORT.md` | The output contract and the plain-language rules |
| `packs/*.md` | Stack-specific rules (GSAP, Motion/Framer, CSS) |
| `assets/report-template.html` | The report, with a worked example inside |

## Credit

The animation judgement comes from **Emil Kowalski** —
[emilkowal.ski](https://emilkowal.ski/) and
[emilkowalski/skills](https://github.com/emilkowalski/skills), MIT.
Easing values, duration budgets, the frequency-first idea and the
"delete the animation first" instinct are all his. If you want the original,
go there — it is excellent, and it is the reason this exists.

## License

MIT
