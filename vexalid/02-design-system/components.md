# 02 — Design System — Components

Every component listed here lives in `components/ui/` or `components/layout/`, is a Server Component
unless marked **client**, and references only semantic tokens (D-202).

## Primitives — `components/ui/`

### `Button`

| Prop | Values | Default |
|---|---|---|
| `variant` | `primary` \| `secondary` \| `ghost` \| `link` | `primary` |
| `size` | `sm` (36px) \| `md` (44px) \| `lg` (52px) | `md` |
| `asChild` | boolean — renders as a `Link` via Radix `Slot` | `false` |
| `loading` | boolean — shows a spinner, sets `aria-busy`, keeps width stable | `false` |

- `primary`: accent fill, `accent-text` label. The page's one main action.
- `secondary`: `border-strong` outline, transparent fill.
- `ghost`: no border; nav and toolbar use only.
- `link`: underlined accent text, inline in prose.
- **Every size is ≥ 44px tall on touch** — `sm` is desktop-only, enforced in review.
- States: rest, hover, `:focus-visible` (2px `--vx-color-focus` ring, 2px offset), active, disabled
  (`opacity: 0.5`, `cursor: not-allowed`, **still focusable** for screen-reader discoverability),
  loading.
- A `<button>` that navigates is a bug. Use `asChild` with `next/link`.

### `Input` / `Textarea`

- 44px min height; 16px font size (**below 16px, iOS zooms the page on focus** — a real mobile bug,
  not a preference).
- Always paired with a visible `<label>`. placeholder-as-label is prohibited.
- Error state: `danger` border + an error message wired via `aria-describedby`, and
  `aria-invalid="true"`. Colour alone never signals the error.
- Help text also via `aria-describedby`, rendered below the field.

### `Select` — **client**

Radix `Select`. Native `<select>` is acceptable and preferred where no custom rendering is needed —
it is more reliable on mobile.

### `Checkbox` — **client**

Radix `Checkbox`. Label is clickable. 24px box with a 44px hit area. Required for the newsletter
consent checkbox (segment 10).

### `Card`

`surface` background, `border`, `radius-lg`, `shadow-sm`. Optional `interactive` prop adds hover
lift (`shadow-md`, 2px translate) and requires the whole card be wrapped in one link — never nest
multiple interactive elements inside a clickable card.

### `Badge`

`variant`: `neutral` | `accent` | `success` | `warning`. Used for "New", post categories, and the
`Coming soon` marker on unbuilt module pages (segment 05).

### `Container`

Wraps content to `--vx-container-max` (or `narrow`), applies `--vx-container-gutter` as side padding.
**Vertical padding is set with `padding-block`, never a `padding` shorthand** — a shorthand silently
zeroes the side gutter, which is the single most common cause of edge-to-edge text on phones.

### `Section`

Semantic `<section>` with `--vx-section-y` block padding, optional `tone` prop
(`default` | `subtle` | `accent` | `inverse`) mapping to background tokens. Optional `id` for
anchor links. Every page is a stack of `Section`s.

### `Prose`

Wraps MDX output. `@tailwindcss/typography` with every colour, size and spacing mapped to tokens.
Max width `--vx-container-narrow`. Overrides needed: heading scale, link colour and underline
offset, `code` styling, blockquote, table (wrapped in `overflow-x: auto`), image `radius-lg`,
`hr` colour, list marker colour.

### `Dialog`, `Accordion`, `Tabs`, `Tooltip`, `DropdownMenu` — all **client**

Radix, styled with tokens. Notes:
- `Accordion` drives the FAQ section; each item must render its answer in the DOM (collapsed, not
  absent) so search engines index it and the FAQ JSON-LD stays truthful.
- `Tooltip` is never the only 2 information appears — it is unreachable on touch.
- `Dialog` traps focus, restores focus on close, closes on Escape. Radix handles this; do not
  reimplement.

## Layout — `components/layout/`

### `SiteHeader` + `SiteNav`

- Sticky, `surface` background with a bottom border that appears only after scroll (subtle, not a
  shadow).
- Height 64px mobile / 72px desktop.
- Left: logo → `/`. Centre: nav. Right: `Log in` (ghost, → `NEXT_PUBLIC_APP_URL`) + `Book a demo` (primary).
- Desktop nav uses Radix `NavigationMenu` for the Product mega-menu (segment 04 defines its contents).
- **Two CTAs maximum in the header.** A third dilutes both.

### `MobileNav` — **client**

- Below `--vx-bp-md`. Full-screen sheet, not a cramped dropdown.
- Trigger is a 44px button labelled `aria-label="Open menu"` with `aria-expanded`.
- Focus trapped while open; body scroll locked; Escape closes; focus returns to the trigger.
- Product/Resources groups are accordions inside the sheet.
- Both CTAs pinned at the bottom of the sheet, full width, thumb-reachable.

### `SiteFooter`

Four columns on desktop, stacked on mobile: Product · Resources · Company · Legal. Plus the
newsletter form (segment 10), social links, copyright with the legal entity name (OQ-001), and
optionally the theme toggle (OQ-204).

### `SkipLink`

First focusable element in the DOM. Visually hidden until focused, then pinned top-left. Targets
`#main-content`, which every page layout must carry on its `<main>`. Non-negotiable WCAG item.

### `Breadcrumbs`

On `/products/hrm/[module]`, `/resources/**`, `/legal/*`. Renders `BreadcrumbList` JSON-LD (segment 11).
Omitted on the home page and `/demo`.

## Marketing sections — `components/sections/`

Specified here as a contract; composed into pages by segment 05.

| Component | Purpose | Key props |
|---|---|---|
| `Hero` | Page-opening statement | `eyebrow`, `heading`, `subheading`, `primaryCta`, `secondaryCta`, `media` |
| `FeatureGrid` | 2–4 column benefit cards | `items[]` of `{ icon, title, body, href? }`, `columns` |
| `ModuleShowcase` | Alternating text/visual rows, one per HRM module | `modules[]`, `reversed` |
| `StatBand` | 3–4 proof numbers on an accent band | `stats[]` of `{ value, label }` |
| `LogoWall` | Customer logos | `logos[]` — **omitted entirely until real customers exist** (OQ-005) |
| `Testimonial` | Quote + attribution + photo | `quote`, `author`, `role`, `company` — real only |
| `Faq` | Accordion Q&A | `items[]` — emits FAQPage JSON-LD |
| `ComparisonTable` | Vexalid vs spreadsheets/legacy HR | `rows[]` — factual comparisons only, no named competitors |
| `CtaBand` | Closing conversion block | `heading`, `body`, `cta` — appears at the foot of every marketing page |
| `PricingTable` | Plan tiers | Deferred to Phase 4 (OQ-004) |

**Rules for section components**
1. No data fetching inside a section. Pages fetch; sections render.
2. No fixed heights. Content length varies; sections grow.
3. Each accepts an optional `className` merged via `cn()` — but never for colour overrides.
4. Every section renders correctly with its optional props absent.

## Accessibility requirements (all components)

| Requirement | Applies to |
|---|---|
| Visible `:focus-visible` ring, never removed | Every interactive element |
| Touch targets ≥ 44×44px | Every interactive element on mobile |
| Semantic HTML first; ARIA only to fill genuine gaps | All |
| Colour is never the sole carrier of meaning | Status, errors, badges, charts |
| Images: meaningful `alt`, or `alt=""` when decorative | All images |
| One `<h1>` per page; heading levels never skip | All pages |
| Keyboard-operable end to end, in a sensible tab order | All |
| Contrast AA in both themes | All |
| Live regions (`role="status"`) announce async results | Form submissions |
| `<html lang>` set | Root layout |

Verified per-route by `@axe-core/playwright` in segment 13; zero serious or critical violations is
a release gate, not a target.
