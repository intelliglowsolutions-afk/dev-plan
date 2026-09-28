# 04 — Site Structure & Navigation

**Priority:** Must-have · **Build order:** 4 of 13 · **Status:** planned (revised for the two-track site)

## Purpose

Define every URL, what each page is for, and how a visitor moves between them — for a site that now
has to serve **two different buyers** without confusing either. This segment decides the shape;
05, 06, 07 and 08 fill it in.

## Scope

**In scope**

- The complete sitemap, with each page's track, job and priority.
- The two-track information architecture and how it is expressed in navigation.
- URL conventions and the redirect policy, including the existing site's anchors.
- Header navigation, two mega-menus, mobile navigation, footer.
- Internal linking strategy and the conversion paths.
- Breadcrumbs, page hierarchy, error pages.

**Out of scope (owned elsewhere)**

- Visual implementation of header/footer → [02 Design System](../02-design-system/README.md).
- Page content → [05](../05-home-company/README.md), [06](../06-services-pages/README.md),
  [07](../07-products-pages/README.md), [08](../08-industries/README.md).
- `sitemap.xml` and canonicals → [11 SEO](../11-seo-analytics/README.md).

## Files

| File | Contents |
|---|---|
| [sitemap.md](./sitemap.md) | Every route, its track, purpose, phase, priority; URL and redirect policy |
| [navigation.md](./navigation.md) | Header, both mega-menus, mobile nav, footer, internal linking |

## The core structural problem

Vexalid sells **consulting services** to pharma QA/IT managers and **software products** to HR and
operations managers. These are different people, different buying cycles, different vocabularies, and
different conversion actions.

The failure mode for a company in this position is a site that averages the two — generic "solutions"
language that means nothing to either. The structure below avoids that by keeping the tracks
**visibly separate everywhere except the home page and one industry page**, where the connection
between them is the whole argument.

## Key decisions taken in this plan

| # | Decision | Rationale |
|---|---|---|
| D-401 | **Two top-level tracks: `/services/*` and `/products/*`.** Legible in the URL, the nav and the page design. | A visitor must know within one click which half of the business they are in. |
| D-402 | **Two separate conversion endpoints: `/contact` (services enquiry) and `/demo` (product demo).** | "Talk to us about validating our MES" and "show me the leave module" are different asks. One form serving both would fit neither, and the follow-up routing would be guesswork. |
| D-403 | **The home page is the only page that presents both tracks equally** (G-03). Every other page commits to one. | Split attention converts badly. The home page's job is to route; every other page's job is to convince. |
| D-404 | **`/industries/pharmaceutical` is the deliberate exception** — it links into both tracks. | It is the page that makes the argument the whole company rests on: the people who validate your systems also build your software. |
| D-405 | **Two mega-menus, not one.** `Services ▾` and `Products ▾`, each listing its own children. | A single "What we do" menu with eight unlike items is a wall of text nobody reads. |
| D-406 | **`/approach` is a real page**, carried over from the existing site's anchor section. | For a consulting buyer, *how you work* is a primary evaluation criterion, not a footnote. It is also proof that the CSV service is methodical rather than ad hoc. |
| D-407 | **Flat, shallow URLs**; trailing slashes off; lowercase; hyphens. Nothing deeper than three segments. | One canonical form per page. `/products/hrm/attendance` sits at the ceiling — see sitemap.md. |
| D-408 | **Every page ends with a `CtaBand` matching its own track.** No page is a dead end, and no page offers the wrong CTA. | The commonest conversion failure is a visitor reaching the bottom of a good page with nothing to do. |
| D-409 | **`Log in` links to the HRM app and is visually subordinate** to the track CTA. | Existing users navigate by habit; the header's persuasive slot belongs to prospects. |
| D-410 | **The existing site's anchors (`#services`, `#approach`, `#contact`) are redirected** to their new pages via a client-side hash handler. | Fragments never reach the server. Without this, every existing inbound link and bookmark silently lands nowhere useful. |
| D-411 | **Service URLs spell names out** — `/services/computerized-system-validation`, not `/services/csv`. | "CSV" means a spreadsheet format to most readers and to search engines. The industry abbreviation is fine in headings, wrong in a URL. |
| D-412 | **No locale segment in v1**, but no locale-hostile assumptions. | Pending OQ-004. |

## The six conversion paths

Every page belongs to at least one. A page serving none should not exist.

| # | Path | Who | Route | Goal |
|---|---|---|---|---|
| 1 | **Services evaluator** | Pharma QA / IT manager | Home → `/services` → `/services/computerized-system-validation` → `/approach` → `/contact` | Services enquiry |
| 2 | **Services searcher** | Same, arriving on organic search | `/services/<service>` → `/approach` → `/contact` | Services enquiry |
| 3 | **Product evaluator** | HR / ops manager | Home → `/products` → `/products/hrm` → a module → `/pricing` → `/demo` | Demo request |
| 4 | **Product searcher** | Same, on a module term | `/products/hrm/<module>` → `/products/hrm` → `/demo` | Demo request |
| 5 | **Researcher** | Either, early stage | Blog post → related post or guide → newsletter / gated download | Subscriber |
| 6 | **Due diligence** | Either, mid-evaluation | `/about`, `/approach`, `/legal/*`, `/industries/pharmaceutical` | Remove objections |

Path 1 is the current revenue. Path 3 is the growth bet. The site must not sacrifice one for the other.

## Dependencies

- **Depends on:** [03 Content Layer](../03-content-layer/README.md) for service, product and module
  slugs; OQ-001 (HRM name — affects URLs), OQ-003 (StratumOne scope), OQ-004 (locale).
- **Depended on by:** 05, 06, 07, 08, 11. The sitemap is the work breakdown for all four page segments.

## Tasks

| ID | Task | Done when |
|---|---|---|
| T-401 | Agree the final route list in [sitemap.md](./sitemap.md), including the HRM's name and URL. | Signed off |
| T-402 | App Router skeleton — every v1 route with a placeholder page. | Every URL returns 200 |
| T-403 | `content/site/navigation.ts` — typed header, both mega-menus, footer. | Header and footer render from data |
| T-404 | `SiteHeader` with two Radix `NavigationMenu` mega-menus. | Keyboard-operable; only one menu open at a time |
| T-405 | `MobileNav` sheet with both tracks as accordions and both CTAs pinned. | Tested at 360px |
| T-406 | `SiteFooter` — four columns plus the newsletter slot. | — |
| T-407 | `next.config.ts` redirects + the client-side hash handler for the old anchors (D-410). | `/#approach` lands on `/approach` |
| T-408 | `not-found.tsx` and `error.tsx` offering **both** tracks. | 404 is useful to either visitor |
| T-409 | `Breadcrumbs` on `/services/*`, `/products/**`, `/industries/*`, `/resources/**`, `/legal/*`. | Renders + JSON-LD |
| T-410 | Internal-link audit: ≤ 3 clicks from home, no orphans, cross-track links present. | Link checker clean |

## Open questions

| ID | Question | Why it matters | Proposed default |
|---|---|---|---|
| OQ-401 | Does the HRM get its own product name and root-level URL, or stay at `/products/hrm`? (= OQ-001) | Changes URLs, nav labels, titles. Cheap now, a redirect map later. | `/products/hrm` as placeholder; revisit the moment the name exists |
| OQ-402 | Is `/approach` the right label, or "How we work" / "Method"? | It is carried over from the live site, so it has continuity value. | Keep `Approach` — it is already the company's own word |
| OQ-403 | Should `/industries/pharmaceutical` exist at launch, or wait for Phase 3? | It is the strongest positioning page on the site, but also the hardest to write well. | Phase 3, written carefully, rather than Phase 2 written thinly |
| OQ-404 | Site-wide search in v1? | Real cost for a ~36-page site. | No. Revisit past ~40 blog posts |
| OQ-405 | Do services need a shared "engagement models" page (fixed-scope vs retained)? | A common consulting-buyer question. | Fold into `/approach` in v1; split out if it grows |
