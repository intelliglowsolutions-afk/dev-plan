# 11 — SEO & Analytics — Requirements

## Metadata

### `buildMetadata()` — `lib/seo.ts` (D-1102)

```ts
type SeoInput = {
  title: string                 // 30–60 chars, before the site suffix
  description: string           // 70–160 chars
  path: string                  // '/products/hrm/attendance'
  ogImage?: string
  type?: 'website' | 'article'
  publishedTime?: string
  modifiedTime?: string
  authors?: string[]
  noindex?: boolean
}
```

Produces:

| Tag | Rule |
|---|---|
| `<title>` | `{title} | Vexalid` — except the home page, which is `Vexalid — {positioning}` |
| `<meta name="description">` | Verbatim from input |
| `<link rel="canonical">` | Absolute, `NEXT_PUBLIC_SITE_URL + path`, lowercase, no trailing slash, no query (D-1106) |
| `og:title` `og:description` `og:url` `og:site_name` `og:type` `og:locale` | — |
| `og:image` | 1200×630, absolute URL, with `og:image:alt` |
| `twitter:card` | `summary_large_image` |
| `robots` | `index,follow` unless `noindex` |
| `article:*` | Only when `type === 'article'` |

Title bounds are enforced by the Zod content schemas (segment 03 D-303), so an over-long title is a
build failure rather than a truncated search result discovered months later.

### Title and description map

| Page | `<title>` | Description intent |
|---|---|---|
| `/` | `Vexalid — Validation, Automation, Innovation` | Who Vexalid is; brand terms only (see the keyword map) |
| `/services` | `Services | Vexalid` | The three service lines in one sentence |
| `/services/computerized-system-validation` | `Computerized System Validation Services | Vexalid` | Risk-based validation, GMP documentation |
| `/services/ai-automation` | `Workflow & AI Automation Services | Vexalid` | Automation regulated operations can accept |
| `/services/custom-software` | `Custom Software Development | Vexalid` | Validation-ready applications and integrations |
| `/approach` | `How We Work | Vexalid` | Discover, Design, Deliver, Improve |
| `/industries/pharmaceutical` | `Software & Validation for Pharmaceutical Manufacturers | Vexalid` | The wedge page |
| `/products` | `Our Products | Vexalid` | Both products, and why we build them |
| `/products/hrm` | `HR Management System | Vexalid` | The suite; regulated-manufacturer angle |
| `/products/hrm/employee-records` | `Employee Records & HR Database Software | Vexalid` | One record per person |
| `/products/hrm/attendance` | `Attendance Tracking & Time Management Software | Vexalid` | Clock-in, shifts, schedules |
| `/products/hrm/leave-management` | `Leave Management Software | Vexalid` | Policies, balances, approvals |
| `/products/hrm/payroll` | `Configurable Payroll Software | Vexalid` | **Configurable, not country-locked** |
| `/products/hrm/employee-self-service` | `Employee Self-Service Portal | Vexalid` | Staff serve themselves |
| `/products/hrm/performance` | `Performance Management Software | Vexalid` | Reviews and goals |
| `/products/hrm/recruitment` | `Recruitment & Onboarding Software | Vexalid` | Pipeline to employee record |
| `/products/hrm/reports` | `HR Reports & Analytics | Vexalid` | The questions a board asks |
| `/products/stratumone` | `StratumOne | Vexalid` | ⚠️ pending OQ-003 |
| `/pricing` | `Pricing | Vexalid` | How pricing works; how to get a quote |
| `/contact` | `Talk to Us | Vexalid` | Services enquiry; should rank — `noindex` **not** applied |
| `/demo` | `Book a Demo | Vexalid` | Product demo; should rank |
| `/about` | `About Vexalid` | Who we are and why |
| `/resources/blog` | `Blog | Vexalid` | — |
| `/legal/*` | `<Doc name> | Vexalid` | `noindex` **not** applied — legal pages are part of due diligence |
| `/thank-you/*` | — | `noindex, nofollow` |

## Keyword → page map — TWO TRACKS (T-1113, D-1108)

The site now targets two unrelated query families. They do not compete with each other, which is
fortunate — but each needs its own strategy, and the services family is the harder and more valuable
of the two.

### Track S — services

| Page | Primary intent | Secondary | Notes |
|---|---|---|---|
| `/services/computerized-system-validation` | "computerized system validation", "CSV services pharma" | "GMP validation services", "software validation pharma" | **Low volume, extremely high intent.** A handful of visits a month from the right people is worth more than thousands of HR visits. Optimise for precision, never for volume |
| `/services/ai-automation` | "workflow automation pharma", "process automation regulated" | "AI automation consulting" | Avoid generic "AI" terms — unwinnable and the traffic is unqualified |
| `/services/custom-software` | "custom software development <market>", "validation-ready software" | "bespoke business applications" | Broad and competitive; the regulated qualifier is the wedge |
| `/industries/pharmaceutical` | "pharmaceutical software validation", "GxP compliance services" | "data integrity pharma" | The highest-intent commercial page on the site |
| `/approach` | — | Brand and evaluation queries | Ranks for little; converts well. Written for humans mid-evaluation |

**Reality check on the services track:** these terms have low search volume in most markets, and in a
small domestic market (OQ-803) they may have almost none. If that is the case, the services pages
still earn their place — as the material a referred prospect reads before calling — but **organic
search will not be the channel that fills the pipeline**, and nobody should plan as though it will.
Confirm the market before investing heavily here.

### Track P — products

| Page | Primary intent | Secondary | Supporting content |
|---|---|---|---|
| `/products/hrm` | "HR management system", "HRM software" | "HR software for manufacturers" | Home + module pages |
| `/products/hrm/attendance` | "attendance tracking software", "employee time tracking" | "shift scheduling software" | Blog post on attendance data |
| `/products/hrm/leave-management` | "leave management system" | "annual leave software" | Blog post on leave policy |
| `/products/hrm/payroll` | "payroll software", "configurable payroll system" | — | — |
| `/products/hrm/employee-records` | "employee database software", "HR records system" | — | Blog post on employee records |
| `/products/stratumone` | Depends entirely on OQ-003: reading (a) → "time synchronisation GxP", "NTP data integrity pharma" (very low volume, very high intent); reading (b) → "centralised time clock system" (higher volume, more competitive) | — | — |

**Method:** each page owns one function term. Each blog post supports one page with an informational
query and links to it. Depth on few topics, not thin pages on many (D-1108).

### The one place the tracks could collide

`/` competes with itself if the home page tries to rank for both "HR software" and "validation
services". It should rank for **brand terms only** — "Vexalid" — and route everything else. Do not
optimise the home page for a service or product term; that is what the track pages are for.

### Rejected outright

Location pages, competitor-comparison pages at launch, programmatically generated page sets, and any
attempt to rank for generic "AI" terms.

### Governing rule

One primary intent per page. Two pages must never target the same term — they compete with each
other, and the weaker one drags the stronger down. With 36 routes across two tracks this is easy to
violate by accident, so the map above is the authority: a new page needs a primary intent that no
existing page holds, or it does not get built.

## Structured data (D-1107)

### Root layout — every page

**Organization**
```jsonc
{
  "@type": "Organization",
  "name": "Vexalid",
  "url": "https://vexalid.com",
  "logo": "https://vexalid.com/logos/vexalid.png",
  "description": "...",
  "sameAs": ["<LinkedIn>", "<X>"],           // only real, maintained accounts (OQ-1103)
  "telephone": "+94719557557",
  "email": "contact@vexalid.com",
  "address": {
    "@type": "PostalAddress",
    "streetAddress": "301/2, Mihidu Mawatha",
    "addressLocality": "Makola North",
    "addressCountry": "LK"
  },
  "contactPoint": [
    { "@type": "ContactPoint", "contactType": "sales", "email": "contact@vexalid.com" }
  ]
}
```

The address and phone are already public on the live site, so publishing them as structured data
costs nothing and gains a real trust and local-search signal. For a consultancy selling to regulated
industry, a verifiable physical presence is worth more than it would be for a pure SaaS vendor.

**WebSite** — with `publisher` pointing at the Organization. **No `SearchAction`** — there is no site
search (OQ-403), and claiming one that does not exist is exactly the mismatch D-1107 forbids.

### `/products` and module pages — SoftwareApplication

```jsonc
{
  "@type": "SoftwareApplication",
  "name": "Vexalid",
  "applicationCategory": "BusinessApplication",
  "applicationSubCategory": "Human Resource Management",
  "operatingSystem": "Web",
  "offers": { "@type": "Offer", "priceCurrency": "...", "price": "0", "description": "Contact for pricing" }
}
```

**No `aggregateRating`.** Fabricating one is a structured-data policy violation and a manual-action
risk. Add it only with real, verifiable reviews.

### Blog posts — Article

`headline` (≤ 110 chars), `description`, `image`, `datePublished`, `dateModified`, `author`
(Person or Organization), `publisher`, `mainEntityOfPage`.

### FAQ sections — FAQPage

Only where the FAQ is **visible in the rendered DOM** (D-1107). The accordion renders answers
collapsed, not absent, precisely so this stays truthful (segment 02).

### Breadcrumbs — BreadcrumbList

Wherever breadcrumbs render. Improves how the URL displays in search results.

## `sitemap.xml` (T-1106)

Generated from `listAllContentUrls()`.

| Rule | Detail |
|---|---|
| Included | Every indexable page — static routes, module pages, blog posts, guides, legal |
| Excluded | Drafts, `noindex` pages, `/thank-you/*`, `/api/*`, `/design`, `/404`, `/500` |
| `lastModified` | Content pages: `updatedAt ?? publishedAt`. Static pages: build date |
| `changeFrequency` | Home `weekly`; product `monthly`; blog posts `yearly`; legal `yearly` |
| `priority` | Per the table in [../04-site-structure-navigation/sitemap.md](../04-site-structure-navigation/sitemap.md) |
| URLs | Absolute, canonical form, lowercase, no trailing slash |

## `robots.txt` (T-1107)

```
User-agent: *
Allow: /
Disallow: /api/
Disallow: /thank-you/
Disallow: /downloads/
Disallow: /design

Sitemap: https://vexalid.com/sitemap.xml
```

**Staging must serve `Disallow: /` for every agent.** A staging site indexed alongside production is
a duplicate-content problem that takes months to unwind — the most common self-inflicted SEO wound
there is. Segment 12 additionally puts staging behind basic auth (belt and braces).

No AI-crawler blocks by default. If the user wants `GPTBot`, `CCBot` etc. excluded, that is a
business decision, not a technical default — flag it before launch.

## OG images (D-1105, T-1108)

- Default: `app/opengraph-image.tsx` via `next/og` — brand background, wordmark, page title, a
  category label. Generated at build.
- Blog posts and guides: the same template with the post title and category.
- Per-page override via the `ogImage` frontmatter field for anything that warrants a bespoke image.
- 1200×630, under 300 KB, **always with `og:image:alt`**.
- Verified in the LinkedIn Post Inspector, X Card Validator and Slack's unfurler before launch —
  LinkedIn in particular caches aggressively, so a wrong image on launch day is visible for weeks.

## Analytics — Plausible (D-1101)

### Setup

- Self-hosted on the VPS at `analytics.vexalid.com` (segment 12).
- Script added in the root layout with `defer`; ~1 KB, no cookies, no consent banner (D-1101).
- Proxied through the site's own domain (`/js/script.js` → the Plausible instance) so ad-blockers
  that block by hostname do not silently erase a third of the data.
- Outbound-link and file-download tracking enabled.

### Event taxonomy (T-1110, D-1110)

| Event | Fires on | Properties |
|---|---|---|
| `services_enquiry_submitted` | `/thank-you/enquiry` | `interest`, `stage`, `sourcePath` |
| `demo_request_submitted` | `/thank-you/demo` | `product`, `companySize`, `sourcePath` |
| `newsletter_subscribed` | `/thank-you/newsletter` | `source` |
| `newsletter_confirmed` | confirm redirect | `source` |
| `guide_download_requested` | `/thank-you/guide` | `guideSlug` |
| `enquiry_cta_clicked` | Any `Talk to us` | `location` (header / hero / cta-band / service) |
| `demo_cta_clicked` | Any `Book a demo` | `location` |
| `service_page_viewed` | `/services/[service]` | `service` |
| `product_page_viewed` | `/products/[product]` | `product` |
| `module_page_viewed` | `/products/hrm/[module]` | `module` |
| `cross_track_click` | Any link from a services page to a product page, or the reverse | `from`, `to` |
| `pricing_viewed` | `/pricing` | — |
| `app_login_clicked` | Header `Log in` | — |

**`cross_track_click` is the one unusual event here**, and the most interesting. It measures whether
the two-track structure actually works — whether services visitors discover the products and vice
versa. If it stays near zero after three months, the cross-links are too weak or in the wrong places,
and the positioning argument (04 navigation.md) is not reaching anyone.

**Never as a property:** email, name, company name, IP, or anything else identifying (D-1110).
`companySize` is a bucket, not an identifier, and is the one exception worth making.

### Goals and funnel

| Funnel | Steps |
|---|---|
| Services (paths 1 & 2) | Landing → `/services` or a service page → `/approach` → `enquiry_cta_clicked` → `/contact` → `services_enquiry_submitted` |
| Products (paths 3 & 4) | Landing → `/products` or a module page → `demo_cta_clicked` → `/demo` → `demo_request_submitted` |
| Cross-track | Any page → `cross_track_click` → the other track → a conversion. **The funnel that tells you whether the two-track structure is working** |
| Content (path 3) | Blog post → `newsletter_subscribed` or `guide_download_requested` → `newsletter_confirmed` |

### Reporting cadence

**Weekly:** sessions, top entry pages, demo requests, newsletter signups, and the `EmailLog` failure
rate (segment 10). The last of these is the alarm — zero demo requests for a week is far more often a
broken form than a quiet market.

**Monthly:** organic vs direct vs referral, Search Console impressions and average position per
target keyword, top content by conversions (not by pageviews — a post with 5,000 views and no
conversions is entertainment), and Core Web Vitals field data.

## Technical SEO checklist (T-1115)

- [ ] One `h1` per page; heading levels never skip
- [ ] Self-referencing canonical on every page, absolute
- [ ] No duplicate titles or descriptions anywhere
- [ ] No redirect chains — one hop maximum
- [ ] No 404s from internal links
- [ ] No orphan pages (segment 04 T-410)
- [ ] `<html lang>` set
- [ ] Every image has `alt`; every decorative image has `alt=""`
- [ ] Core Web Vitals green on mobile (D-1111; enforced in segment 13)
- [ ] HTTPS everywhere, HSTS set, no mixed content
- [ ] `www` → apex (or the reverse) resolved with a 308, consistently
- [ ] Sitemap submitted and accepted in Search Console and Bing
- [ ] Staging is `Disallow: /` **and** behind basic auth
- [ ] Structured data passes the Rich Results Test with no errors
- [ ] No `noindex` accidentally shipped to production — check the built HTML, not the source
