# Phase 5 — The report

**The report is the deliverable.** It is written for the person who owns the site, who may
not be a developer. If they cannot tell what is wrong, how bad it is, and what to do, the
audit failed regardless of how good the findings are.

## Where it goes

A self-contained **single HTML file**, plus a `shots/` folder for screenshots.

- Default location: a directory **outside the repository** — the scratchpad, or a path the
  user names. Never create a folder inside their project unless they ask.
- Name it `motion-audit-<site>-<YYYY-MM-DD>.html`.
- Self-contained: inline CSS and JS, no build step, no network dependency, opens by
  double-click and still works offline. Reference screenshots relatively.
- Deliver it to the user (send the file, or publish it as a private artifact if they would
  rather have a link). Also print a 5-line summary in the terminal.

Build it from `assets/report-template.html`: fill the `REPORT_DATA` object with the
findings. The template renders everything else.

## The three rules that make it readable

### 1. Every finding leads with the symptom, in plain language

Before any selector, any property, any number, say **what a visitor would notice**. No
jargon in this sentence — no "cubic-bezier", no "composited", no "CLS".

| Bad | Good |
| --- | --- |
| "`transition: all 0.2s ease` on `.algo-btn`" | "This button animates every property at once, so its shadow and border drag behind the movement. It looks slightly cheap." |
| "`.button:active` loses the cascade" | "Pressing a button with a mouse shows no reaction at all. It works on phones but not on a computer, so a desktop visitor can't tell the click registered." |
| "CLS 0.11 from `#smooth-wrapper`" | "About 1.3 seconds after the page opens, the whole featured section jumps and resizes. If someone is already reading or reaching for a link, it moves under them." |

### 2. Severity in human terms, and effort alongside it

Never ship bare HIGH/MEDIUM/LOW.

| Level | Label to show | Means |
| --- | --- | --- |
| 1 | **Breaks the feel** | A visitor notices something is wrong, or an interaction gives no feedback |
| 2 | **Looks unpolished** | Nothing is broken; it reads as less carefully made |
| 3 | **Polish** | Only a designer would spot it |
| — | **Already right** | Checked and correct — worth saying, it builds trust in the rest |

Pair every finding with an honest effort estimate: **5 min**, **30 min**, **a few hours**.
That is what lets someone non-technical triage without asking anyone.

### 3. Every finding says how to see it yourself

One line: open which page, do what, watch what. "Open the home page, hard-refresh, and
watch the 'featured' strip about a second in — it resizes." A reader who can reproduce a
problem trusts the report and can confirm the fix.

## Structure of the page

1. **Header** — site, date, what was audited, and whether the page was measured or the
   audit was static-only (if static, a prominent banner saying so).
2. **Verdict band** — a one-paragraph plain-language verdict, then the measured numbers as
   three cards with traffic-light colour **and a plain caption under each**:
   - CLS → "How much the page jumps around while loading"
   - LCP → "How long until the main content appears"
   - INP → "How quickly the page answers a tap or click"
   Show the good/bad threshold next to the value, so the number means something.
3. **"If you only fix three things"** — the top three by impact ÷ effort, each one line,
   each linking to its finding. Many readers will read only this.
4. **Counts by severity** — "2 break the feel, 6 look unpolished, 9 are polish".
5. **Findings**, grouped by page or area, ordered by leverage. Each card:
   - the plain-language symptom (rule 1)
   - severity label + effort badge + a `measured` / `confirmed` / `unverified` tag
   - the screenshot, if there is one
   - "How to see it yourself" (rule 3)
   - **collapsed** technical detail: `file:line`, the current code, the exact replacement
     code, and which category and rule it comes from
6. **What's already right** — a short list. Honest, and it tells the reader which parts
   they do not need to touch.
7. **Easings and durations inventory** — a table of every distinct value and the files it
   appears in, with near-duplicates grouped. The most reusable output of the audit.
8. **Method and limits** — what was measured, on what, what was excluded (mirrored trees,
   design comps), and what remains unverified. Never let an unverified claim read as fact.

## Presentation requirements

- Works at phone width; no horizontal scroll.
- Light and dark mode via `prefers-color-scheme`, with an explicit background on `body`.
- Readable when printed to PDF (findings expanded in print, filters hidden).
- Filters for severity, effort and page; they must degrade to "everything visible" if JS
  fails.
- Colour is never the only signal — every traffic light also has a word.
- No chart library and no external fonts. Inline SVG or CSS for any visual.
