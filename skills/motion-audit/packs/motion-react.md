# Rule pack — Motion / Framer Motion (React)

Load when `motion` or `framer-motion` is a dependency.

## M1. Hardware acceleration and the shorthand props

- `x`, `y`, `scale` shorthands were **not** hardware-accelerated before **motion 14.2**:
  they ran on the main thread and dropped frames under load. On older versions use the
  full transform string: `animate={{ transform: "translateX(100px)" }}`.
- Check the installed version in the lockfile before reporting this. On 14.2+ the
  shorthands are accelerated on HTML elements and the rule does not apply.
- `layout` / `layoutId` animations run on the main thread in **every** version. They are
  the right tool for a shared-element transition and the wrong tool on a busy page.

## M2. Springs over durations

- Gesture-driven and interruptible motion should be a spring — it carries velocity when
  caught mid-flight. Default: `{ type: "spring", duration: 0.5, bounce: 0.2 }`.
- A fixed `duration` on a drag or swipe handler is a finding.
- Keep `bounce` 0.1–0.3. Visible bounce belongs to drag-to-dismiss and playful moments.

## M3. Exit animations

- `AnimatePresence` is required for an exit; a conditionally-rendered component without it
  vanishes instantly while animating in politely. See AUDIT.md category 4.
- `mode="wait"` vs `"popLayout"` changes whether the outgoing element still occupies space
  — the wrong one causes a layout jump during the swap.

## M4. Reduced motion

- `useReducedMotion()` is the hook. **Branch the values**, do not skip the animation:
  keep an opacity fade, drop the movement. Returning `null` early is the "zero, not
  gentler" anti-pattern from AUDIT.md 6a.

## M5. Variants and stagger

- `staggerChildren` 0.03–0.08. Above ~0.1 it reads sequential.
- Variants must not be redefined inline per render — a new object each render defeats
  memoisation and can restart animations.

## M6. Scroll

- `useScroll` + `useTransform` is the scrubbed path; the output must be linear in the
  scroll input (no extra easing on top).
- `whileInView` with `viewport={{ once: true }}` is the reveal path. Without `once` it
  re-fires every pass — judge that under category 1.

## M7. Inline style ownership

Motion writes `style` inline, so the same inline-beats-stylesheet rule as GSAP applies:
a CSS `:hover { transform }` on a `motion.div` that animates transform will not win.
See AUDIT.md category 9.
