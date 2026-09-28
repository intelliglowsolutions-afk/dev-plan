# 04 — Site Structure & Navigation — Sitemap

Legend — **Track**: S = services, P = products, C = company/shared.
**Phase**: 2 = launch surface, 3 = wedge + resources, 4 = post-launch.

## Full route table

| # | URL | Track | Page | Job | Primary CTA | Phase | Priority |
|---|---|---|---|---|---|---|---|
| 1 | `/` | C | Home | Establish what Vexalid is, then split cleanly into two tracks (G-03) | Both, side by side | 2 | 1.0 |
| 2 | `/services` | S | Services overview | The three service lines, and the method behind them | Talk to us | 2 | 0.9 |
| 3 | `/services/computerized-system-validation` | S | CSV | **The flagship service page.** Risk-based validation, GMP documentation | Talk to us | 2 | 0.9 |
| 4 | `/services/ai-automation` | S | AI Automation | Workflow automation, AI-assisted ops, dashboards | Talk to us | 2 | 0.8 |
| 5 | `/services/custom-software` | S | Custom Software | Business applications, integrations, validation-ready design | Talk to us | 2 | 0.8 |
| 6 | `/approach` | C | Approach | Discover → Design → Deliver → Improve. The trust page for a services buyer | Talk to us | 2 | 0.7 |
| 7 | `/products` | P | Products overview | Both products, and why a validation firm builds software | Book a demo | 2 | 0.9 |
| 8 | `/products/hrm` | P | HRM overview ⚠️ *name pending OQ-001* | The HR suite, led by the regulated-manufacturer angle (G-04) | Book a demo | 2 | 0.9 |
| 9 | `/products/hrm/employee-records` | P | Module | Employee data, org structure, documents | Book a demo | 2 | 0.7 |
| 10 | `/products/hrm/attendance` | P | Module | Clock-in, shifts, schedules | Book a demo | 2 | 0.7 |
| 11 | `/products/hrm/leave-management` | P | Module | Leave types, balances, approvals | Book a demo | 2 | 0.7 |
| 12 | `/products/hrm/payroll` | P | Module | Configurable payroll runs, payslips | Book a demo | 2 | 0.7 |
| 13 | `/products/hrm/employee-self-service` | P | Module | The employee's own portal | Book a demo | 3 | 0.6 |
| 14 | `/products/hrm/performance` | P | Module | Reviews, goals | Book a demo | 3 | 0.6 |
| 15 | `/products/hrm/recruitment` | P | Module | Hiring pipeline, onboarding | Book a demo | 3 | 0.6 |
| 16 | `/products/hrm/reports` | P | Module | HR analytics and reporting | Book a demo | 3 | 0.6 |
| 17 | `/products/stratumone` | P | StratumOne ⚠️ *scope pending OQ-003* | Centralized clock system | Book a demo | 2 | 0.8 |
| 18 | `/industries/pharmaceutical` | C | Industry | **The wedge.** Where services and products meet: GMP, data integrity, validation | Talk to us | 3 | 0.8 |
| 19 | `/pricing` | P | Pricing | Explain the model without numbers; remove the "how much?" exit | Book a demo | 2 | 0.6 |
| 20 | `/contact` | S | Contact | **Services enquiry** — the consultative conversion page | Send enquiry | 2 | 0.8 |
| 21 | `/demo` | P | Demo | **Product demo request** — the specific conversion page | Book my demo | 2 | 0.8 |
| 22 | `/about` | C | About | Who Vexalid is; due-diligence path | Talk to us | 2 | 0.6 |
| 23 | `/resources` | C | Resources hub | Route to blog and guides | Subscribe | 3 | 0.5 |
| 24 | `/resources/blog` | C | Blog index | List posts, filter by category | Subscribe | 3 | 0.6 |
| 25 | `/resources/blog/[slug]` | C | Blog post | Organic entry point | Inline CTA | 3 | 0.5 |
| 26 | `/resources/guides/[slug]` | C | Guide | Gated long-form lead magnet | Get the guide | 3 | 0.5 |
| 27 | `/legal/privacy` | C | Privacy policy | Compliance + due diligence | — | 2 | 0.3 |
| 28 | `/legal/terms` | C | Terms of service | Compliance | — | 2 | 0.3 |
| 29 | `/legal/cookies` | C | Cookie notice | Short — analytics is cookieless | — | 2 | 0.3 |
| 30 | `/legal/dpa` | C | Data processing addendum | HR and pharma buyers ask early | — | 2 | 0.3 |
| 31 | `/thank-you/enquiry` | S | Post-services-enquiry | Confirm, set expectations | Read the approach | 2 | — (noindex) |
| 32 | `/thank-you/demo` | P | Post-demo-request | Confirm, set expectations | Read a guide | 2 | — (noindex) |
| 33 | `/thank-you/newsletter` | C | Post-subscribe | Confirm double opt-in sent | — | 3 | — (noindex) |
| 34 | `/thank-you/guide` | C | Post-download | Confirm the email | Book a demo | 3 | — (noindex) |
| 35 | `/404` | C | Not found | Recover the visit — **offers both tracks** | Popular links | 2 | — |
| 36 | `/500` | C | Error | Fail gracefully | Contact | 2 | — |

**36 routes at launch**, against the current site's one. Twelve of them (the eight modules, blog
posts, guides) are template-driven, so the hand-built page count is closer to 20.

## Machine routes

| URL | Purpose | Segment |
|---|---|---|
| `/sitemap.xml` | From the content adapter; excludes drafts, `noindex`, thank-you pages | 11 |
| `/robots.txt` | Disallows `/thank-you/`, `/api/`, `/downloads/`, `/design` | 11 |
| `/rss.xml` | Blog feed | 09 |
| `/api/newsletter/confirm` | Double opt-in confirmation target | 10 |
| `/api/download/[token]` | Gated PDF delivery, HMAC, single-use, expiring | 10 |
| `/opengraph-image` | Default OG image | 11 |
| `/design` | Component gallery — staging only | 02 |

## Deferred to Phase 4

| URL | Why deferred |
|---|---|
| `/industries/medical-devices`, `/industries/food-beverage` | One industry page done properly beats three thin ones. Add when `/industries/pharmaceutical` proves itself |
| `/case-studies`, `/case-studies/[slug]` | Needs a consenting named client (OQ-006) — and it would be the single strongest asset on the site |
| `/pricing` with plan tiers | G-09 |
| `/careers`, `/careers/[slug]` | Only with real openings |
| `/security` | Only with real practices or certifications to state (OQ-007) |
| `/services/[service]/[sub-service]` | Only if a service line grows enough to need sub-pages |
| `/integrations` | Needs a real integration catalogue |
| `/team` | Only with real photos and names (D-503) |

## URL conventions (D-401, D-407)

| Rule | Example |
|---|---|
| Lowercase, hyphen-separated | `/services/computerized-system-validation` |
| No trailing slash | `/about` not `/about/` |
| **Max 3 path segments** | `/products/hrm/attendance` sits exactly at the limit |
| Nouns, not verbs | `/pricing` not `/see-pricing` |
| Spell service names out | `/services/computerized-system-validation`, not `/services/csv` — "csv" reads as a spreadsheet format to both humans and search engines |
| Track prefix is explicit | `/services/*` and `/products/*` make the two tracks legible in the URL itself |
| No dates in blog URLs | `/resources/blog/<slug>` |

**On the module depth:** `/products/hrm/attendance` is three segments, which is the stated ceiling.
The alternative — `/hrm/attendance` at the root — reads better but breaks the `/products/*` grouping
that makes the two-track structure obvious. If OQ-001 gives the HRM a distinct product name, revisit:
a named product could justify a root-level path (like `/stratumone`), which would also free a segment
of depth.

## Redirect policy (D-410)

The existing one-pager's anchors must not simply land on a home page that no longer has those
sections. Map them:

| Old | New |
|---|---|
| `/#services` | `/services` |
| `/#expertise` | `/services` |
| `/#approach` | `/approach` |
| `/#contact` | `/contact` |

Fragments are never sent to the server, so a server redirect cannot see them. Handle these with a
small client-side redirect in the root layout reading `window.location.hash` on first load — one of
the few places client-side routing is the correct tool. Any inbound link, bookmark or shared URL
carrying an old anchor then lands on the right page instead of silently doing nothing.

General policy:

| Situation | Status |
|---|---|
| Slug changes after publication | 308 permanent |
| Page removed, clear successor | 308 permanent |
| Page removed, no successor | 410 Gone — never a blanket redirect to home |
| Casing / trailing-slash normalisation | 308 permanent |
| Temporary campaign URL | 307 temporary |

Every entry carries `// added YYYY-MM-DD: reason`. Redirects are not deleted without checking server
logs for live traffic.

## Depth and orphan rules (T-410)

- Every page reachable from home in **≤ 3 clicks**.
- Every page has **≥ 2 inbound internal links** excluding the footer.
- Every service page links to `/services`, `/approach`, at least one other service, and `/contact`.
- Every module page links to `/products/hrm`, two sibling modules, and `/demo`.
- `/industries/pharmaceutical` links into **both tracks** — it is the only page that deliberately
  does, because it is where the two halves of the business meet.
- Each product page links to at least one service page, and each service page to at least one
  product. That cross-link *is* the positioning: it is how a visitor discovers that the software
  vendor is also the validation firm.
