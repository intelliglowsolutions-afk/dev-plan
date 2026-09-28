# 02 — Design System

**Priority:** Must-have · **Build order:** 2 of 13 · **Status:** planned

## Purpose

Define the visual language of Vexalid — colour, type, space, motion, elevation — as a token set, and
implement the ~20 primitive components every page is assembled from. The goal is that segment 05 can
build a page by composing existing parts, never by inventing a new shade of blue.

## Scope

**In scope**

- Brand direction: what Vexalid should *feel* like to an HR buyer, and the visual decisions that follow.
- Design tokens: colour (light + dark), typography scale, spacing, radii, shadows, motion, breakpoints.
- Primitive components in `components/ui/`.
- Layout components: header, nav, mobile nav, footer, container, section.
- Iconography and illustration approach.
- Accessibility baseline: contrast, focus, motion preferences, target sizes.

**Out of scope (owned elsewhere)**

- Page-level composition and marketing copy → [05 Core Pages](../05-home-company/README.md).
- Form logic, validation and submission → [10 Lead Capture](../10-lead-capture/README.md).
  This segment supplies only the visual `Input`, `Button` and error/help styling.
- Tailwind/PostCSS setup → [01 Foundation](../01-foundation-tech-stack/README.md).
- MDX prose styling rules → [03 Content Layer](../03-content-layer/README.md) consumes `Prose` from here.

## Files

| File | Contents |
|---|---|
| [brand-tokens.md](./brand-tokens.md) | Brand direction, the full token set, light/dark values, contrast targets |
| [components.md](./components.md) | Every primitive and layout component: API, variants, states, a11y requirements |

## Brand direction

**A brand already exists.** vexalid.com is live with a visual identity and a voice:
*"Validation · Automation · Innovation"*, *"Confidence built into every system"*, and —
most tellingly — *"Precision in the details. Clarity in the outcome."*

That last line is effectively a design brief. This segment **extends the existing identity rather
than inventing one** (D-213). The token values below are a proposal until the real ones are extracted
from the live site (OQ-201); the token *names* and component APIs are final either way, which is the
point of D-202.

Vexalid sells **compliance and evidence** to regulated industry, and **software that handles pay,
leave and attendance** to HR teams. Both audiences need the same thing from the design: *credibility
and calm competence*. A pharma QA manager must believe these people are methodical; an HR manager
must believe this system will not lose a payroll run. That rules out both default aesthetics —
playful-startup (gradient blobs, cartoon illustrations) and enterprise-grey (dated, hard to read).

It also sets a harder constraint than a normal marketing site: **the design itself is evidence.** A
site with inconsistent spacing and three shades of the same blue undercuts a company whose selling
proposition is precision. Visible orderliness is not decoration here; it is an argument.

The target is **precise, warm, and quietly modern**:

- A restrained neutral base (near-black on off-white, not pure black on pure white) so dense
  information stays readable.
- **One** saturated accent, used sparingly and consistently for action. If everything is accented,
  nothing is.
- Generous whitespace and a strict 4px spacing rhythm — visible orderliness is the argument.
- Product UI screenshots and **method diagrams** as the primary imagery, not stock photography of
  people in meetings or laboratories. For the products, the UI is the proof; for the services, a
  diagram that shows real method is the proof. Both are cheaper to produce and easier to keep
  accurate than photography.
- Motion that is functional only: reveal, focus, state change. Nothing decorative that moves.

**The two-track requirement.** The design must make it obvious which half of the business a visitor
is in, without creating two visual identities. The mechanism: **one palette, one type scale, one
spacing rhythm — differentiated only by section `tone` and by the accent's role.** Services pages and
product pages should look like the same company, and like different rooms in it. Anything stronger
(two colour schemes, two header treatments) makes the site read as two businesses sharing a domain,
which is precisely the impression the whole plan is trying to avoid.

## Key decisions taken in this plan

| # | Decision | Rationale |
|---|---|---|
| D-201 | Tokens are **CSS custom properties** declared in Tailwind v4's `@theme` block in `app/globals.css`. | One source of truth readable by Tailwind utilities, hand-written CSS and inline styles alike. No build pipeline needed. |
| D-202 | Token names are **semantic, not literal**: `--vx-color-surface`, `--vx-color-accent`, not `--vx-blue-500`. A literal palette exists underneath and is never used directly in components. | Rebranding, or the dark theme, changes the mapping layer only. Components never need editing. |
| D-203 | **Dark mode ships in v1**, via `prefers-color-scheme` with an optional manual toggle. | It is nearly free if the token layer is semantic from day one and expensive to retrofit. It also signals product competence to a technical evaluator. |
| D-204 | **A single accent hue.** Secondary colours exist only as semantic status (success, warning, danger, info). | Disciplined colour reads as competence; a rainbow reads as a template. |
| D-205 | **Variable fonts, self-hosted** in `public/fonts/`, with `font-display: swap` and preloaded subsets. | No Google Fonts request — one fewer third-party origin, no GDPR question about IP addresses reaching Google, and no render-blocking external CSS. |
| D-206 | **Type scale is fluid** using `clamp()` between defined min/max, rather than per-breakpoint overrides. | Marketing headlines need to work at 360px and 1920px; fluid type removes a class of breakpoint bugs. |
| D-207 | **Spacing is a 4px base scale**, exposed as tokens. Arbitrary Tailwind values (`p-[13px]`) are a lint error. | The rhythm is what makes a layout look designed rather than assembled. |
| D-208 | **`prefers-reduced-motion` is honoured globally** in `globals.css`, not per-component. | One rule, no component can forget it. |
| D-209 | **Every interactive element has a visible focus ring** using `:focus-visible` and a dedicated token. Never `outline: none` without a replacement. | Keyboard operability is a hard WCAG requirement and a procurement checklist item. |
| D-210 | **Icons: `lucide-react`, 24px grid, 1.5px stroke, imported individually.** No icon fonts, no sprite sheets. | Tree-shakeable, consistent weight, and matches the "precise" brand direction. |
| D-211 | **No stock photography of people.** Imagery is product UI, abstract diagrams, or nothing. | Generic stock photos actively reduce trust in B2B software, and licensing them is a recurring cost. |
| D-212 | Contrast target is **WCAG AA (4.5:1 body, 3:1 large text and UI boundaries)**, verified in both themes. | AA is the procurement baseline. Tokens are chosen to pass, so pages cannot accidentally fail. |
| D-213 | **Extend the existing brand, don't replace it.** Extract colour, type and the logo from the live vexalid.com; change a value only for a stated reason. | The identity is already in use on live material and in whatever the company has sent to clients. Rewriting it for novelty costs continuity and buys nothing. |
| D-214 | **One visual system across both tracks**, differentiated only by `Section tone` and accent usage — never by separate palettes or header treatments. | Two visual identities would make the site read as two companies, undoing the positioning the whole plan rests on. |
| D-215 | **Diagrams are a first-class component type**, not an afterthought — theme-aware SVG using `currentColor` and tokens, legible at 360px. | The services track sells method, and method is best shown as a diagram. Two exported PNGs (one per theme) drift and break; one tokenised SVG does not. |

## Dependencies

- **Depends on:** [01 Foundation](../01-foundation-tech-stack/README.md) (Tailwind v4 + Radix installed).
  **Blocked on OQ-002** — if a logo or brand palette exists, token values change (the token *names*
  and component APIs do not, which is the point of D-202).
- **Depended on by:** 04, 05, 06, 07 — every visual segment.

## Tasks

| ID | Task | Done when |
|---|---|---|
| T-201 | Resolve OQ-002 with the user: existing logo/palette, or design from scratch. | Answer recorded in this file |
| T-202 | Write the full `@theme` token block in `app/globals.css` per [brand-tokens.md](./brand-tokens.md). | Tokens resolve as Tailwind utilities |
| T-203 | Dark-theme token overrides under `@media (prefers-color-scheme: dark)` and `[data-theme='dark']`. | Toggling either flips the whole site correctly |
| T-204 | Self-host font files, subset to Latin, preload the two faces used above the fold. | No external font request in the network panel |
| T-205 | Base layer: resets, `:focus-visible` ring, reduced-motion rule, selection colour, `scroll-behavior`. | — |
| T-206 | Build `components/ui/` primitives per [components.md](./components.md). | Each has all documented variants and states |
| T-207 | Build `components/layout/`: header, desktop nav, mobile nav, footer, container, section. | Header works at 360px through 1920px |
| T-208 | `Prose` component + `@tailwindcss/typography` overrides mapped to tokens. | An MDX body renders correctly |
| T-209 | Contrast-audit every token pair in both themes. | Documented pass table in [brand-tokens.md](./brand-tokens.md) |
| T-210 | A private `/design` route rendering every component in every state, both themes. | Visual review possible in one page; excluded from sitemap and `noindex` |

## Open questions

| ID | Question | Why it matters | Proposed default |
|---|---|---|---|
| OQ-201 | **Extract the real brand from the live site** — logo files (SVG preferred), exact colour values, the typefaces in use. Is the rebuild a continuation or a redesign? | **Blocks token values** (D-213). Component work can start regardless. | Continuation. Needs either the asset files or a visual review of vexalid.com — the site's text was read during planning, but its design has not been seen |
| OQ-202 | Typeface licensing — is a paid display face acceptable, or open-source only? | Affects the headline voice most. | Open-source: Inter Variable (UI/body) + Instrument Serif or Fraunces (headlines only) for a note of warmth |
| OQ-203 | Do product UI screenshots exist yet, and is the product visually finished enough to show? | D-211 makes screenshots the primary imagery. If the product is unfinished, the site needs abstract diagrams instead. | Assume no usable screenshots at launch; plan tasteful abstract UI illustrations with a screenshot swap later |
| OQ-204 | Manual dark-mode toggle, or follow the OS only? | A toggle needs a header control and localStorage. | Ship OS-following in v1; add the toggle in the footer if asked |
| OQ-205 | Is `/design` acceptable as a public-but-noindexed route, or must it be behind basic auth? | Exposes unshipped components. | Basic auth on staging, entirely absent from the production build |
