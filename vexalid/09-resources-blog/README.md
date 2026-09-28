# 09 — Resources & Blog

**Priority:** Should-have · **Build order:** 9 of 13 · **Status:** planned

## Purpose

Build the content surface that drives conversion path 3 (the researcher): a resources hub, a blog,
and gated long-form guides. This is the site's organic acquisition engine — the part that brings in
visitors who have never heard of Vexalid.

## Scope

**In scope**

- `/resources` hub, `/resources/blog` index with category filtering, `/resources/blog/[slug]` post
  template, `/resources/guides/[slug]` guide template.
- Post template features: table of contents, reading time, byline, related posts, inline CTAs, share links.
- RSS feed.
- The editorial plan: categories, launch posts, cadence, and the brief for each piece.
- How content converts without becoming an advertisement.

**Out of scope (owned elsewhere)**

- Content schemas and the adapter → [03 Content Layer](../03-content-layer/README.md).
- The gating mechanism — email capture, token, PDF delivery → [10 Lead Capture](../10-lead-capture/README.md).
  This segment owns the gate's *page*; segment 10 owns its *logic*.
- Article/JSON-LD metadata → [11 SEO](../11-seo-analytics/README.md).

## Files

| File | Contents |
|---|---|
| [requirements.md](./requirements.md) | Page-by-page requirements, post template anatomy, editorial plan |

## Key decisions taken in this plan

| # | Decision | Rationale |
|---|---|---|
| D-901 | **All blog posts are ungated.** Only 1–2 substantial guides are gated (OQ-305). | Gating a blog post destroys the organic traffic it was written to earn. Gate the artefact, never the article. |
| D-902 | **Write for the reader's problem, not the product's features.** A post about building an onboarding checklist is genuinely useful with or without Vexalid. | Content that is a disguised advertisement neither ranks nor converts. The product link earns its 9 by being relevant, not by being inserted. |
| D-903 | **Six fixed categories**, defined in the Zod enum (segment 03), not free-form tags for navigation. | Free-form categories multiply into 30 near-duplicates within a year. Tags exist but are display-only, never a navigable index in v1. |
| D-904 | **One inline `Cta` per post, 9d after the point of maximum usefulness** — never in the first screen. | An interruption before value is delivered is an ad. After it, it is a reasonable next step. |
| D-905 | **Category pages are not separate routes in v1** — `/resources/blog` filters client-side via a query param. | Six thin category pages are an SEO liability until each has 5+ posts. Promote to real routes when they do. |
| D-906 | **Reading time and table of contents are computed**, not authored (D-310). | — |
| D-907 | **RSS at `/rss.xml`, full content, not excerpts.** | Costs nothing, and the audience most likely to use RSS (technical HR ops, other builders) is worth having. |
| D-908 | **No comments section.** | Moderation cost, spam surface, and a GDPR obligation, for near-zero benefit on a B2B blog. |
| D-909 | **Share links are plain anchors** (`mailto:`, LinkedIn, X intent URLs) — **no third-party share widgets**. | Widgets are tracking scripts. Plain links carry no consent obligation and no performance cost. |
| D-910 | **Do not launch the blog with one post.** Three at minimum, on launch day. | A blog with one post signals abandonment. Better to launch the site without `/resources` in nav and add it when there are three pieces. |

## Dependencies

- **Depends on:** 03 (adapter, MDX pipeline), 02 (`Prose`, `Card`), 07 (gate logic for guides).
- **Depended on by:** 08 (RSS, Article JSON-LD).
- **Phase:** 3. The site can launch without this; it should not stay launched without it for long.

## Tasks

| ID | Task | Done when |
|---|---|---|
| T-901 | Build `/resources` hub. | Links to blog and guides; shows the latest 3 posts |
| T-902 | Build `/resources/blog` index: featured post, grid, category filter, pagination at 12/page. | Empty and single-post states both look deliberate |
| T-903 | Build `/resources/blog/[slug]` per [requirements.md](./requirements.md). | ToC, byline, reading time, related posts, inline CTA all working |
| T-904 | Build `/resources/guides/[slug]` with gated and ungated variants. | Gated variant does **not** ship the body to the client (segment 03 rule) |
| T-905 | Build `/rss.xml`. | Validates against the W3C feed validator |
| T-906 | Sticky table of contents with scroll-spy on desktop; collapsed `<details>` on mobile. | Keyboard-operable; hidden entirely for posts with < 3 `h2`s |
| T-907 | Share links (D-909). | No third-party script in the network panel |
| T-908 | Write the three launch posts per the editorial plan. | Real, useful, human-reviewed |
| T-909 | Write the first gated guide + its PDF. | PDF exists at `pdfPath`, matches the web version |
| T-910 | Author bios and avatars (OQ-304). | At least `vexalid-team` exists |
| T-911 | Related-posts logic: same category first, then shared tags, then recency. | Never shows the current post; always returns 3 once 4 posts exist |

## Open questions

| ID | Question | Why it matters | Proposed default |
|---|---|---|---|
| OQ-901 | Who writes the posts, and at what cadence can it be sustained? | A blog that stops after four posts is worse for credibility than no blog. | Two posts a month, sustained, beats eight then silence. Do not launch the blog until that cadence is committed |
| OQ-902 | Is AI-assisted drafting acceptable, and will a human subject-matter expert review before publishing? | Unreviewed generated content ranks poorly and can be factually wrong about HR and payroll — a domain where errors are costly. | Drafting assistance is fine; **human review before publish is mandatory**, especially for anything touching payroll, leave entitlement or employment law |
| OQ-903 | Which market's HR practices do posts assume? Leave entitlements, payroll rules and employment law are jurisdiction-specific. | A post assuming the wrong jurisdiction is actively misleading. | Write jurisdiction-neutral operational content ("how to design a leave policy") and avoid statutory specifics until OQ-003 settles the market |
| OQ-904 | Should guides exist as PDFs at all, or as web pages with the email capture in front? | A PDF is a better lead magnet; a web page ranks. | Both: the guide is a real indexable web page, with the PDF as the download reward |
| OQ-905 | Newsletter cadence and content — post digest, or original writing? | Determines whether the subscription is worth having. | Monthly digest of new posts plus one original note. A list that is never emailed is a liability, not an asset |
