# 02 — Design System — Brand & Tokens

> Values here are a **proposal pending OQ-002**. Token *names* are final; if a brand palette arrives,
> only the values in the "primitive palette" and "semantic mapping" tables change.

## Token architecture

Three layers, each depending only on the one above:

1. **Primitive palette** — raw values. `--vx-slate-700`, `--vx-indigo-600`. Never used in a component.
2. **Semantic tokens** — role-named, mapped to primitives, redefined per theme.
   `--vx-color-surface`, `--vx-color-accent`. **This is the only layer components reference.**
3. **Component tokens** — occasional, for a component with genuinely local needs.
   `--vx-button-height-md`.

This is what makes D-203 (dark mode) and a future rebrand cheap: layer 2's mapping changes, layers
above it do not.

## Primitive palette (proposed)

**Neutrals — Slate.** Cool grey, reads as precise rather than warm-corporate.

| Token | Hex | Used for |
|---|---|---|
| `--vx-slate-50` | `#F8FAFC` | Light page background |
| `--vx-slate-100` | `#F1F5F9` | Light subtle surface |
| `--vx-slate-200` | `#E2E8F0` | Light borders |
| `--vx-slate-300` | `#CBD5E1` | Light strong borders, disabled text |
| `--vx-slate-400` | `#94A3B8` | placeholder |
| `--vx-slate-500` | `#64748B` | Muted text (light) |
| `--vx-slate-600` | `#475569` | Secondary text (light) |
| `--vx-slate-700` | `#334155` | Dark strong border |
| `--vx-slate-800` | `#1E293B` | Dark subtle surface |
| `--vx-slate-900` | `#0F172A` | Primary text (light) / dark surface |
| `--vx-slate-950` | `#020617` | Dark page background |

**Accent — Indigo.** One hue, used only for action and emphasis (D-204).

| Token | Hex | Note |
|---|---|---|
| `--vx-indigo-50` | `#EEF2FF` | Tinted backgrounds |
| `--vx-indigo-100` | `#E0E7FF` | |
| `--vx-indigo-400` | `#818CF8` | Accent on dark surfaces — passes AA on `slate-950` |
| `--vx-indigo-500` | `#6366F1` | Hover on dark |
| `--vx-indigo-600` | `#4F46E5` | **Primary action on light.** 7.0:1 on `slate-50` |
| `--vx-indigo-700` | `#4338CA` | Hover on light |
| `--vx-indigo-900` | `#312E81` | Deep accent surfaces |

**Status.** Semantic only; never decorative.

| Role | Light | Dark | Use |
|---|---|---|---|
| Success | `#047857` | `#34D399` | Form submitted, subscription confirmed |
| Warning | `#B45309` | `#FBBF24` | Non-blocking caution in content callouts |
| Danger | `#B91C1C` | `#F87171` | Validation errors, destructive copy |
| Info | `#0369A1` | `#38BDF8` | Content callouts, "note" blocks |

## Semantic mapping

```css
@theme {
  /* ---- Colour: light (default) ---- */
  --vx-color-bg:              var(--vx-slate-50);
  --vx-color-bg-subtle:       var(--vx-slate-100);
  --vx-color-surface:         #FFFFFF;
  --vx-color-surface-raised:  #FFFFFF;
  --vx-color-border:          var(--vx-slate-200);
  --vx-color-border-strong:   var(--vx-slate-300);

  --vx-color-text:            var(--vx-slate-900);
  --vx-color-text-secondary:  var(--vx-slate-600);
  --vx-color-text-muted:      var(--vx-slate-500);
  --vx-color-text-inverse:    var(--vx-slate-50);

  --vx-color-accent:          var(--vx-indigo-600);
  --vx-color-accent-hover:    var(--vx-indigo-700);
  --vx-color-accent-subtle:   var(--vx-indigo-50);
  --vx-color-accent-text:     #FFFFFF;

  --vx-color-focus:           var(--vx-indigo-600);

  --vx-color-success: …; --vx-color-warning: …;
  --vx-color-danger:  …; --vx-color-info:    …;
}
```

Dark theme redefines **only** these semantic tokens, under both
`@media (prefers-color-scheme: dark)` and `[data-theme='dark']` (so an eventual manual toggle wins
in both directions):

| Token | Dark value |
|---|---|
| `--vx-color-bg` | `--vx-slate-950` |
| `--vx-color-bg-subtle` | `--vx-slate-900` |
| `--vx-color-surface` | `--vx-slate-900` |
| `--vx-color-surface-raised` | `--vx-slate-800` |
| `--vx-color-border` | `--vx-slate-800` |
| `--vx-color-border-strong` | `--vx-slate-700` |
| `--vx-color-text` | `--vx-slate-50` |
| `--vx-color-text-secondary` | `--vx-slate-300` |
| `--vx-color-text-muted` | `--vx-slate-400` |
| `--vx-color-accent` | `--vx-indigo-400` |
| `--vx-color-accent-hover` | `--vx-indigo-500` |
| `--vx-color-accent-subtle` | `--vx-indigo-900` |
| `--vx-color-accent-text` | `--vx-slate-950` |

**Rule:** no colour may have its only definition inside a dark-mode block. Define in `:root`, then
override.

## Typography

**Faces (D-205, self-hosted, variable, pending OQ-202)**

| Token | Family | Use |
|---|---|---|
| `--vx-font-sans` | Inter Variable | All UI, body, and most headings |
| `--vx-font-display` | Instrument Serif | Hero H1 and section eyebrows only — a deliberate touch of warmth |
| `--vx-font-mono` | JetBrains Mono | Code blocks in blog posts |

Fallback stacks are mandatory: `Inter, -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif`.

**Fluid scale (D-206)** — `clamp(min, preferred, max)`, min at 360px, max at 1280px.

| Token | Min → Max | Line height | Use |
|---|---|---|---|
| `--vx-text-xs` | 12 → 12px | 1.5 | Labels, captions |
| `--vx-text-sm` | 14 → 14px | 1.55 | Secondary text, form help |
| `--vx-text-base` | 16 → 17px | 1.65 | Body |
| `--vx-text-lg` | 18 → 20px | 1.6 | Lead paragraph |
| `--vx-text-xl` | 20 → 24px | 1.45 | H4, card titles |
| `--vx-text-2xl` | 24 → 30px | 1.35 | H3 |
| `--vx-text-3xl` | 30 → 38px | 1.25 | H2 |
| `--vx-text-4xl` | 36 → 52px | 1.15 | Page H1 |
| `--vx-text-5xl` | 44 → 72px | 1.05 | Hero H1 |

**Rules**
- Body copy never below 16px. Never.
- Measure capped at `--vx-measure: 68ch` for prose, `48ch` for hero subheads.
- Headings use `text-wrap: balance`; paragraphs use `text-wrap: pretty`.
- Weights: 400 body, 500 UI labels, 600 headings. No 700+ — it reads shouty at display sizes.
- `letter-spacing: -0.02em` on `3xl` and above; `0` below.

## Spacing (D-207)

4px base. `--vx-space-1` = 4px through `--vx-space-32` = 128px, on the scale
`4 8 12 16 20 24 32 40 48 64 80 96 128`.

Section rhythm — the single biggest lever on whether the site looks designed:

| Token | Value | Use |
|---|---|---|
| `--vx-section-y` | `clamp(64px, 9vw, 128px)` | Vertical padding between page sections |
| `--vx-section-y-tight` | `clamp(40px, 6vw, 72px)` | Between closely related sections |
| `--vx-container-max` | `1200px` | Standard content width |
| `--vx-container-narrow` | `760px` | Prose (blog, legal, guides) |
| `--vx-container-gutter` | `clamp(16px, 5vw, 32px)` | **Minimum 16px side gutter at every width** |

## Radii, elevation, borders

| Token | Value | Use |
|---|---|---|
| `--vx-radius-sm` | 6px | Badges, small inputs |
| `--vx-radius-md` | 10px | Buttons, inputs |
| `--vx-radius-lg` | 16px | Cards |
| `--vx-radius-xl` | 24px | Feature panels, hero media |
| `--vx-radius-full` | 9999px | Pills, avatars |

Elevation is **restrained** — two shadow levels, not six. Layered, low-opacity, neutral-tinted:

| Token | Use |
|---|---|
| `--vx-shadow-sm` | Resting cards, dropdown triggers |
| `--vx-shadow-md` | Hovered cards, popovers, dialogs |

In dark mode, shadows are near-invisible — depth comes from `--vx-color-surface-raised` and borders
instead. Do not simply darken the shadow.

## Motion (D-208)

| Token | Value | Use |
|---|---|---|
| `--vx-ease` | `cubic-bezier(0.2, 0, 0, 1)` | Default — decelerating, feels responsive |
| `--vx-ease-in-out` | `cubic-bezier(0.4, 0, 0.2, 1)` | Two-way transitions |
| `--vx-duration-fast` | 120ms | Hover, focus, colour change |
| `--vx-duration-base` | 200ms | Dropdowns, accordions |
| `--vx-duration-slow` | 320ms | Dialogs, page-section reveals |

Global, non-negotiable:

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

Scroll-reveal animations use `IntersectionObserver` with a **once-only** trigger and are skipped
entirely under reduced motion — content must be visible without JavaScript, so reveals animate *in*
from a visible state, never from `opacity: 0` set in CSS alone.

## Breakpoints

| Token | Width | Target |
|---|---|---|
| `--vx-bp-sm` | 640px | Large phone |
| `--vx-bp-md` | 768px | Tablet — nav collapses below this |
| `--vx-bp-lg` | 1024px | Laptop |
| `--vx-bp-xl` | 1280px | Desktop |

Design and test at **360px** first. The layout must never scroll horizontally; only tables, code
blocks and diagrams may exceed the viewport, each inside its own `overflow-x: auto` container.

## Contrast audit (T-209)

Every pair below must be verified and recorded before segment 05 begins. Target AA: 4.5:1 body,
3:1 for large text and UI component boundaries.

| Foreground | Background | Light | Dark | Required |
|---|---|---|---|---|
| `text` | `bg` | TBD | TBD | 4.5:1 |
| `text-secondary` | `bg` | TBD | TBD | 4.5:1 |
| `text-muted` | `bg` | TBD | TBD | 4.5:1 |
| `accent` | `bg` | TBD | TBD | 4.5:1 (it is used as link text) |
| `accent-text` | `accent` | TBD | TBD | 4.5:1 (button label) |
| `border-strong` | `bg` | TBD | TBD | 3:1 (input boundary) |
| `focus` | `bg` | TBD | TBD | 3:1 |
| `danger` | `bg` | TBD | TBD | 4.5:1 |

`text-muted` is the one most likely to fail. If it does, darken it rather than accepting the
failure — "it's only placeholder text" is exactly the reasoning an accessibility audit rejects.

## Logo and favicon (pending OQ-201)

Needed regardless of who designs them:

- Wordmark, horizontal, SVG, light and dark variants.
- Mark/monogram alone, square, for favicon and social avatars.
- `favicon.ico` (32px), `icon.svg`, `apple-touch-icon.png` (180px), `site.webmanifest`.
- Default OG image at 1200×630 — generated from a template, not hand-made per page (segment 11).
- Minimum clear space and minimum size rules, documented here once the mark exists.
