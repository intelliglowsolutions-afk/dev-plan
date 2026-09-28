# 04 — Site Structure & Navigation — Navigation

All structures are declared in `content/site/navigation.ts` as typed constants, so a broken `href` is
a TypeScript error rather than a runtime 404.

## Header

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│ [Vexalid]   Services ▾   Products ▾   Approach   About   │   Log in   [Talk to us] │
└────────────────────────────────────────────────────────────────────────────────────┘
```

| Slot | Item | Behaviour |
|---|---|---|
| Logo | Vexalid wordmark → `/` | `aria-label="Vexalid home"`; a heading rather than a link on `/` |
| 1 | **Services** ▾ | Mega-menu (below) |
| 2 | **Products** ▾ | Mega-menu (below) |
| 3 | **Approach** → `/approach` | Direct link |
| 4 | **About** → `/about` | Direct link |
| Right | **Log in** → the HRM app | `Button variant="ghost"`, subordinate (D-409) |
| Right | **Talk to us** → `/contact` | `Button variant="primary"` |

Four primary items, two dropdowns. Adding a fifth is a decision, not a default.

### The header CTA problem, and how it is solved

The site has two conversion actions but room for one header button. Options considered:

| Approach | Verdict |
|---|---|
| Two buttons — `Talk to us` + `Book a demo` | **Rejected.** Three right-hand controls with `Log in` is clutter, and two equal CTAs convert worse than one. |
| One generic CTA — `Get in touch` | **Rejected.** Generic CTAs underperform specific ones, and it throws away the routing information the visitor already gave us. |
| **Context-aware CTA** | **Chosen.** The header shows `Talk to us` → `/contact` by default and on every `/services/*`, `/approach`, `/about` and company page; it switches to `Book a demo` → `/demo` on `/products/**` and `/pricing`. |

The switch is a server-side decision from the current route segment — no client JavaScript, no layout
shift. On `/industries/pharmaceutical` (the both-tracks page, D-404) it stays `Talk to us`, since an
industry visitor is more often a services buyer.

**Behaviour**
- Sticky. `surface` background; bottom border fades in after ~8px of scroll.
- Current section gets `aria-current="page"` and a visible underline.
- Dropdowns open on click **and** hover-with-intent (~120ms delay, so a diagonal mouse path does not
  open both). Keyboard: Enter/Space opens, arrows move, Escape closes and restores focus.
- **Only one mega-menu open at a time** — opening Products closes Services. Radix `NavigationMenu`
  handles this; do not reimplement it.

## Services mega-menu

Two columns, ~640px.

**Column 1 — What we do**

| Item | Href | Description |
|---|---|---|
| Computerized System Validation | `/services/computerized-system-validation` | Risk-based validation, audit-ready documentation |
| AI Automation | `/services/ai-automation` | Workflow automation and data-driven operations |
| Custom Software | `/services/custom-software` | Purpose-built, validation-ready applications |

**Column 2 — Panel** (`tone="subtle"`)

- *"Rigour for compliance. Intelligence for growth."* — the existing site's own line
- Link: **How we work** → `/approach`
- Link: **Pharmaceutical** → `/industries/pharmaceutical`
- Secondary button: `Talk to us`

## Products mega-menu

Three columns, ~780px.

**Column 1 — HRM** ⚠️ *label pending OQ-001*

| Item | Href |
|---|---|
| Overview | `/products/hrm` |
| Employee Records | `/products/hrm/employee-records` |
| Attendance | `/products/hrm/attendance` |
| Leave Management | `/products/hrm/leave-management` |
| Payroll | `/products/hrm/payroll` |

**Column 2 — HRM, continued**

| Item | Href |
|---|---|
| Employee Self-Service | `/products/hrm/employee-self-service` |
| Performance | `/products/hrm/performance` |
| Recruitment & Onboarding | `/products/hrm/recruitment` |
| Reports & Analytics | `/products/hrm/reports` |

**Column 3 — StratumOne + panel**

| Item | Href |
|---|---|
| **StratumOne** | `/products/stratumone` — with its one-line description |
| All products | `/products` |
| Pricing | `/pricing` |
| Secondary button | `Book a demo` |

Each module item renders its `icon` and `tagline` from `content/products/hrm/modules/*.mdx`, so the
menu cannot drift from the pages. Modules with `status: 'planned'` show a small `Coming soon` badge —
the same honesty rule that governs the pages themselves (D-504).

**Note on balance:** nine HRM items against one StratumOne item makes the menu visually lopsided and
implies StratumOne is an afterthought. If StratumOne has real depth (pending OQ-003), give it two or
three child pages so the columns balance. If it genuinely is a single-page product, consider showing
only the HRM *overview* plus four headline modules here, with "All modules" linking onward — a
shorter menu that treats both products as peers.

## Mobile navigation (< 768px)

- Trigger: 44px button, `aria-label="Open menu"`, `aria-expanded`, `aria-controls`.
- Full-screen sheet. At 360px a nested dropdown is unusable.
- Order:
  1. **Services** — accordion → three service pages + Approach
  2. **Products** — accordion → HRM overview, eight modules, StratumOne, Pricing
  3. `Industries`
  4. `About`
  5. `Resources`
  6. Divider
  7. `Log in` — full-width secondary
  8. **`Talk to us`** and **`Book a demo`** — both full-width, side by side, **pinned at the bottom**
- On mobile, showing **both** CTAs is correct: there is room, the visitor has already opened a menu
  deliberately, and the thumb zone is the right place for them.
- Focus trapped, body scroll locked, Escape closes, focus returns to the trigger, closes on route change.

## Footer

Five columns on desktop, stacked accordions on mobile.

| Services | Products | Company | Resources | Legal |
|---|---|---|---|---|
| Computerized System Validation | HRM overview | About | Blog | Privacy Policy |
| AI Automation | Employee Records | Approach | Guides | Terms of Service |
| Custom Software | Attendance | Contact | | Cookie Notice |
| Pharmaceutical | Leave Management | Careers *(if live)* | | Data Processing Addendum |
| | Payroll | | | |
| | Self-Service | | | |
| | Performance | | | |
| | Recruitment | | | |
| | Reports | | | |
| | StratumOne | | | |
| | Pricing | | | |

**Below the columns**
- Newsletter form (segment 10).
- Logo and a one-line positioning statement.
- **Real contact details** — `+94 71 955 7557`, `contact@vexalid.com`,
  `301/2, Mihidu Mawatha, Makola North, Sri Lanka`. A physical address is a trust signal that matters
  disproportionately for a compliance consultancy, and it is already public on the current site.
- Social links — only accounts that exist and are maintained.
- `© 2026 <legal entity>. All rights reserved.` (OQ-005).

Footer links do not count toward the "≥ 2 inbound links" rule (sitemap.md) — body copy must link too.

## `/contact` and `/demo` chrome (D-402)

Both conversion pages use reduced chrome: logo only, no mega-menus, minimal footer, no newsletter
form. A second ask on a conversion page reduces completion of the first.

They differ in tone, deliberately:

| | `/contact` — services | `/demo` — products |
|---|---|---|
| Heading | "Let's talk about your systems" | "See it with your own data" |
| Framing | Consultative. What you're facing, what stage you're at | Specific. Which modules, how many employees |
| Trust elements | Approach summary, response-time promise, named contact | What the demo covers, no obligation |
| Alternate route | Small link: *"Looking at our software instead? Book a demo"* | Small link: *"Need validation services? Talk to us"* |

That cross-link matters: a visitor who lands on the wrong form should be redirected in one click, not
forced back through the nav.

## Internal linking strategy

| Page type | Must link to |
|---|---|
| Home | `/services`, `/products`, ≥ 2 service pages, ≥ 2 product pages, `/approach`, both CTAs |
| `/services` | All 3 service pages, `/approach`, `/contact`, **≥ 1 product page** |
| Service page | `/services`, `/approach`, ≥ 1 sibling service, `/contact`, ≥ 1 product where genuinely relevant |
| `/products` | Both products, `/pricing`, `/demo`, **≥ 1 service page** |
| `/products/hrm` | All 8 modules, `/pricing`, `/demo`, `/industries/pharmaceutical` |
| Module page | `/products/hrm`, 2 sibling modules, `/demo` |
| `/products/stratumone` | `/products`, `/demo`, ≥ 1 service page |
| `/industries/pharmaceutical` | **Both tracks** — CSV service and both products |
| `/approach` | All 3 service pages, `/contact` |
| Blog post | ≥ 1 service or product page where genuinely relevant, 2–3 related posts |
| 404 | Home, `/services`, `/products`, `/contact` |

**The cross-track links are load-bearing.** They are how a services visitor discovers there are
products, and how a product visitor discovers the vendor validates GMP systems for a living. Without
them the site is two brochures sharing a domain.

**Anchor text:** descriptive and varied. Never "click here"; never "learn more" as the only anchor on
a page. It should make sense read aloud out of context — an accessibility requirement (screen-reader
link lists) and how search engines read it.

## Breadcrumbs (T-409)

On `/services/*`, `/products/**`, `/industries/*`, `/resources/**`, `/legal/*`.
Hidden on `/`, `/contact`, `/demo`, `/thank-you/*`, `/about`, `/approach`.

```
Home  /  Services  /  Computerized System Validation
Home  /  Products  /  HRM  /  Attendance
```

- `<nav aria-label="Breadcrumb">` with an ordered list; current page unlinked, `aria-current="page"`.
- Emits `BreadcrumbList` JSON-LD (segment 11).
- Truncates the final item over ~40 characters on mobile, with the full text kept in the accessible name.

Breadcrumbs matter more here than on a single-product site: three-segment URLs under two tracks are
exactly the case where a visitor loses their place.

## Error pages (T-408)

**404** — "We couldn't find that page", then **both tracks offered**:
`Home` · `Services` · `Products` · `Blog` · `Contact us`. No oversized "404" graphic pushing the
useful links below the fold.

**500** — "Something went wrong on our end", a retry button, a contact link. No stack trace, no error
id shown to the visitor; the id goes to Sentry.

Both are statically rendered so they work when the database or an upstream service is down.
