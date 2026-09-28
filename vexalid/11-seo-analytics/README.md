# 11 — SEO & Analytics

**Priority:** Must-have · **Build order:** 11 of 13 · **Status:** planned

## Purpose

Make the site findable and make its performance measurable — without adding a single third-party
tracking script or triggering a cookie-consent obligation.

## Scope

**In scope**

- Metadata strategy: titles, descriptions, canonicals, Open Graph, Twitter cards.
- Structured data (JSON-LD): Organization, SoftwareApplication, Article, FAQPage, BreadcrumbList.
- `sitemap.xml`, `robots.txt`, RSS discovery.
- OG image generation.
- Keyword and page-intent mapping.
- Technical SEO: canonicals, crawl budget, internal linking signals, Core Web Vitals.
- Analytics: self-hosted Plausible, event taxonomy, the conversion funnel, the reporting cadence.
- Search Console and Bing Webmaster setup.

**Out of scope (owned elsewhere)**

- Page copy and headings → [05 Core Pages](../05-home-company/README.md).
- URL structure → [04 Site Structure](../04-site-structure-navigation/README.md).
- Performance *budgets and enforcement* → [13 Launch & QA](../13-launch-qa/README.md). This segment
  states why the vitals matter for ranking; segment 13 enforces them.
- Hosting the Plausible instance → [12 Infrastructure](../12-infrastructure-deployment/README.md).

## Files

| File | Contents |
|---|---|
| [requirements.md](./requirements.md) | Metadata rules, JSON-LD shapes, keyword map, analytics taxonomy |

## Key decisions taken in this plan

| # | Decision | Rationale |
|---|---|---|
| D-1101 | **Self-hosted Plausible only. No Google Analytics, no ad pixels, no third-party tags.** | Plausible is cookieless and first-party, so **no cookie-consent banner is legally required**. That removes a UI surface, a conversion tax and a compliance risk in one decision. Adding GA or any pixel later makes a consent banner mandatory and re-opens this segment. |
| D-1102 | **Every page's metadata comes from Next's `generateMetadata`**, built by a shared `buildMetadata()` helper. No hand-written `<head>` tags. | One 11 to get canonicals, OG and titles right; impossible for a new page to silently ship without them. |
| D-1103 | **`<title>` carries the search term; the `h1` carries the benefit** (D-506). | They serve different readers — the SERP scanner and the person who arrived. Forcing them to match makes one of them worse. |
| D-1104 | **A missing or out-of-bounds title/description fails the build** (segment 03 D-303). | SEO debt is invisible until it is expensive. |
| D-1105 | **OG images are generated at build time** from a template using `next/og`, with a per-page title. | 25+ hand-made social images will not be maintained. A generated one always matches the page. |
| D-1106 | **Self-referencing canonicals on every page**, absolute, from `NEXT_PUBLIC_SITE_URL`. | Prevents duplicate-content splits from UTM parameters, filter query strings and casing. |
| D-1107 | **JSON-LD must describe what is actually on the page.** FAQ schema only where the FAQ is visible in the DOM. | Mismatched structured data earns a manual penalty, not a rich result. |
| D-1108 | **No keyword stuffing, no doorway pages, no AI-generated bulk content.** Depth over volume. | The site's ranking asset is a small number of genuinely useful pages. Thin pages dilute the whole domain. |
| D-1109 | **Conversion events fire from the thank-you pages**, not from form submission. | Only completed conversions count. Optimising against inflated numbers is worse than not measuring. |
| D-1110 | **No personal data is ever sent to analytics.** No emails, names or company names in event properties. | Would undo the cookieless/no-consent position in a single line of code. |
| D-1111 | **Core Web Vitals are treated as a ranking input, not just a nicety.** | They are a real (if modest) ranking factor and a large conversion factor. Both point the same way. |

## Dependencies

- **Depends on:** 04 (final URLs), 05 and 06 (pages to describe), 09 (Plausible hosted),
  OQ-001 (domain).
- **Depended on by:** 10 (the launch checklist verifies this segment's output).

## Tasks

| ID | Task | Done when |
|---|---|---|
| T-1101 | `lib/seo.ts` — `buildMetadata()` covering title, description, canonical, OG, Twitter, robots. | Every page uses it; none hand-writes head tags |
| T-1102 | `components/seo/json-ld.tsx` + builders per type. | Google Rich Results Test passes for each |
| T-1103 | Organization + WebSite JSON-LD in the root layout. | — |
| T-1104 | SoftwareApplication JSON-LD on `/products/*` and module pages. | Accurate — no invented `aggregateRating`, which is a policy violation without real reviews |
| T-1105 | Article JSON-LD on blog posts; FAQPage where FAQs are visible; BreadcrumbList where breadcrumbs render. | — |
| T-1106 | `app/sitemap.ts` from `listAllContentUrls()`. Excludes drafts, `noindex`, thank-you and `/api/*`. | Valid XML; correct `lastModified` |
| T-1107 | `app/robots.ts`. Disallows `/api/`, `/thank-you/`, `/downloads/`, `/design`. Points at the sitemap. | — |
| T-1108 | `app/opengraph-image.tsx` template + per-route overrides. | Renders correctly in the LinkedIn, X and Slack preview debuggers |
| T-1109 | Deploy self-hosted Plausible; add the script; verify pageviews. | Real traffic visible |
| T-1110 | `lib/analytics.ts` — typed event helpers per the taxonomy. | An unknown event name is a type error |
| T-1111 | Wire conversion events on the thank-you pages (D-1109). | Each shows in Plausible goals |
| T-1112 | Verify the domain in Google Search Console and Bing Webmaster Tools; submit the sitemap. | Both verified, sitemap accepted |
| T-1113 | Keyword → page map in [requirements.md](./requirements.md). | Every v1 page has one primary intent, with no two pages competing for the same term |
| T-1114 | `.well-known/security.txt` and a favicon/manifest set. | — |
| T-1115 | Pre-launch crawl (Screaming Frog free tier or `lychee`). | No broken links, no redirect chains, no missing metadata, no duplicate titles |

## Open questions

| ID | Question | Why it matters | Proposed default |
|---|---|---|---|
| OQ-1101 | Which market is being targeted, and is `hreflang` needed? (= OQ-003) | Keyword research is worthless without a market; search volume and competition differ enormously by country. | English, single market; revisit on the user's answer |
| OQ-1102 | Is there budget for paid search at launch? | Ad pixels would force a consent banner (D-1101) and re-open this segment. | No paid acquisition in v1. If it starts, plan the consent banner in the same sprint |
| OQ-1103 | Does a Google Business Profile or a LinkedIn company page exist? | Organization JSON-LD `sameAs` links, and brand-search presence. | Create the LinkedIn page before launch — it is free, and it is where B2B prospects verify a vendor is real |
| OQ-1104 | Should a `/security` page be published? | Enterprise HR buyers search for it by name during evaluation. | Only once there are real practices or certifications to state (OQ-009). Phase 4 |
| OQ-1105 | Does the site need a competitor-comparison page for terms like "BambooHR alternative"? | High commercial intent, but easy to get legally and reputationally wrong. | Phase 4, with care, factual only, and never disparaging |
