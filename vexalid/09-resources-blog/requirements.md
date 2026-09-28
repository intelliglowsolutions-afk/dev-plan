# 09 — Resources & Blog — Requirements

## `/resources` — hub

| Section | Contents |
|---|---|
| Hero | H1 "Resources for HR teams". One line on what is here. |
| Guides | 2–3 `Card`s for the guides — given top placement because they are the higher-value conversion (path 3). |
| Latest writing | The 3 most recent posts, plus "See all posts" → `/resources/blog`. |
| Newsletter | The footer newsletter form, repeated inline with a hub-specific heading. |

Skipped entirely in nav until there are ≥ 3 posts **and** ≥ 1 guide (D-910).

## `/resources/blog` — index

| Element | Behaviour |
|---|---|
| Featured post | The single post with `featured: true` — wide card, hero image, excerpt. Build fails if more than one is set. |
| Category filter | Six pills + "All". Updates `?category=` (shallow routing, scroll preserved), so filtered views are linkable and shareable. `aria-pressed` on the active pill. |
| Post grid | 3 columns desktop / 2 tablet / 1 mobile. Each card: image or a token-coloured fallback, category badge, title, excerpt (2 lines, clamped), date, reading time. **The whole card is one link** — no nested interactive elements. |
| Pagination | 12 per page, `/resources/blog?page=2`. Real links, so pages are crawlable; no infinite scroll (which hides content from crawlers and breaks the back button). |
| Empty state | "No posts in this category yet" + a reset link. Never a blank region. |
| Newsletter | Inline after the first 6 cards, styled as a card so it sits in the grid rhythm rather than interrupting it. |

## `/resources/blog/[slug]` — post template

Anatomy, top to bottom:

| # | Element | Notes |
|---|---|---|
| 1 | Breadcrumbs | Home / Resources / Blog / <title> |
| 2 | Category badge | Links to the filtered index |
| 3 | `h1` | The post title, full width, narrow measure |
| 4 | Byline row | Author avatar + name, published date, reading time. `updatedAt` shown as "Updated <date>" when present — signals freshness to both readers and search engines |
| 5 | Hero image | Optional. `priority` loaded, real dimensions, required `alt` |
| 6 | Table of contents | Desktop: sticky in the left margin with scroll-spy. Mobile: a collapsed `<details>` above the body. Hidden for posts with < 3 `h2`s (D-906) |
| 7 | Body | MDX in `Prose`, `--vx-container-narrow` (~68ch). Allowlisted components only (segment 03) |
| 8 | Inline CTA | One `Cta`, 9d after the section that delivers the main value — not in the first screen (D-904) |
| 9 | Share row | Copy link, LinkedIn, X, email — plain anchors (D-909) |
| 10 | Author card | Avatar, name, role, short bio |
| 11 | Related posts | 3 cards (T-911) |
| 12 | Newsletter | Full-width band |
| 13 | `CtaBand` | → `/demo` |

**Reading experience**
- Measure capped at 68ch. Long-form at full container width is unreadable and readers leave.
- Paragraph spacing is generous; `h2` gets roughly twice the space above it as below.
- Code blocks: highlighted at build time (Shiki), horizontally scrollable, with a copy button.
- Tables wrapped in `overflow-x: auto` with a visible scroll affordance on mobile.
- Blockquotes are visually distinct from `Callout`s — one is the author quoting, the other is the
  author interrupting.
- Images: full measure width, `radius-lg`, optional caption in `text-sm text-muted`.

## `/resources/guides/[slug]` — guide template

### Ungated

Renders like a post but longer, with a persistent "Download PDF" button in the sticky ToC area.

### Gated (`gated: true`)

| Element | Notes |
|---|---|
| Hero | Title, subtitle, page count, an illustrated PDF cover thumbnail |
| What you'll learn | The `summary[]` bullets from frontmatter — 3–6 items. This is what earns the email address |
| Excerpt | The MDX body up to the `<!--more-->` marker. **Split server-side.** The remainder is never sent to the client — a gate that ships the whole document in the page source is not a gate |
| Capture form | Card, beside or beneath the excerpt: email + first name + consent checkbox (segment 10) |
| Trust line | "We'll email you the PDF. No spam, unsubscribe any time." |
| After submit | Redirect to `/thank-you/guide`; the email carries a single-use, expiring download link |

The gated page is **fully indexable** — the excerpt and the `summary` bullets are real content that
ranks. Only the PDF is withheld.

## Categories (D-903)

| Slug | Label | Covers |
|---|---|---|
| `hr-operations` | HR Operations | Process, policy, day-to-day running |
| `payroll` | Payroll | Payroll process, payslips, configuration |
| `compliance` | Compliance | Records, audit, data protection |
| `attendance` | Attendance & Time | Shifts, scheduling, clock-in |
| `product` | Product | Releases, how-tos, feature deep dives |
| `company` | Company | Announcements — used sparingly |

Adding a category requires editing the Zod enum, which is deliberate friction.

## Editorial plan

### Launch set (3 posts minimum, D-910)

**The blog must serve both tracks.** Three posts is the minimum (D-910), and with two audiences the
right split at launch is **two services-track posts and two products-track posts** — four, not three.
A blog that only talks about HR software tells a pharma QA visitor that the consulting side is an
afterthought.

| # | Working title | Track | Category | Intent | Links to |
|---|---|---|---|---|---|
| 1 | "What a validation plan actually contains" | S | `validation` | Demonstrates practitioner knowledge in the first paragraph. **The highest-value post on the list** — it is the one a QA manager forwards to a colleague | `/services/computerized-system-validation` |
| 2 | "Why your systems' clocks matter more than you think" | S | `compliance` | Time integrity across GxP systems — reframes a problem most readers have not articulated | `/products/stratumone` ⚠️ *assumes OQ-003 reading (a)* |
| 3 | "What to record about every employee (and what not to)" | P | `hr-operations` | Useful regardless of vendor; introduces employee records naturally | `/products/hrm/employee-records` |
| 4 | "Designing a leave policy that survives contact with reality" | P | `hr-operations` | A real recurring pain with genuine search intent | `/products/hrm/leave-management` |

Post 1 is the one to write first and to write best. It is the cheapest possible demonstration that
Vexalid has done this work — and unlike a service page, a reader does not discount it as marketing.

### Gated guides — one per track

**Services track (write this one first): "A practical validation planning checklist."** What to
document, in what order, what a risk assessment needs to cover, what evidence an inspector expects to
see. Built from real engagement material, so it costs little to produce and is very hard for a
competitor to fake.

This is almost certainly **the best-converting asset on the site** (06 OQ-606). Its audience is small
but every download is a qualified lead, and the act of downloading it signals exactly the situation
Vexalid sells into. A newsletter subscriber is worth a little; someone who downloads a validation
checklist is worth a phone call.

**Products track: "The HR system buyer's guide."** How to evaluate an HRM, what to ask vendors, what
to check in a demo, migration questions, a comparison worksheet. Useful to someone evaluating *any*
HRM — which is exactly the person Vexalid wants to talk to.

Both earn the email honestly. A guide that is really a brochure gets downloaded once and
unsubscribed from immediately.

### Cadence (OQ-901)

Two posts a month, sustained. Every post gets a written brief before drafting: target query, reader's
problem, the argument, the one module it links to, and the CTA. Posts without a brief drift into
generic filler.

### Post brief template

```
Title:
Target query:
Reader's problem:
What they can do differently after reading:
Argument (3–5 h2s):
Links to module:
Inline CTA placement:
Category / tags:
Author:
Reviewed by (subject-matter, required — OQ-902):
```

## RSS (`/rss.xml`, D-907)

- RSS 2.0, full content in `content:encoded`, generated at build from `listPosts()`.
- Channel: title, description, `NEXT_PUBLIC_SITE_URL`, language, `lastBuildDate`.
- Items: title, absolute link, description (excerpt), `pubDate`, `guid` (permalink), category, author.
- Excludes drafts. Absolute URLs everywhere — relative URLs break in every feed reader.
- `<link rel="alternate" type="application/rss+xml">` in the root layout head.

## Performance

Content pages are where a marketing site usually gets slow. Budgets:

| Metric | Budget |
|---|---|
| Post page JS | < 90 KB gzipped |
| LCP (hero image or `h1`) | < 1.8s on 4G |
| CLS | < 0.05 — every image carries explicit dimensions |
| Syntax highlighting | 0 KB client-side (build-time Shiki) |
| ToC scroll-spy | The only client JS on a post, `IntersectionObserver`-based, < 2 KB |

Enforced by Lighthouse CI in segment 13.
