# 03 — Content Layer

**Priority:** Must-have · **Build order:** 3 of 13 · **Status:** planned

## Purpose

Decide where marketing copy, blog posts, guides, legal text and module descriptions live, how they
are validated, and how pages read them — in a way that lets a CMS replace the file system later
without rewriting a single page component.

## Scope

**In scope**

- The MDX-in-repo authoring model and its directory layout.
- Frontmatter schemas, validated by Zod at build time.
- The content adapter in `lib/content/` — the sole interface between content and pages.
- MDX component mapping: which React components are available inside content.
- Remark/rehype plugin pipeline: heading anchors, table of contents, syntax highlighting, external links.
- Image handling for content.
- The CMS migration path (Phase 4).
- The content authoring workflow for a non-developer.

**Out of scope (owned elsewhere)**

- The actual marketing copy → [05 Core Pages](../05-home-company/README.md) and [9 Resources](../09-resources-blog/README.md).
- Prose *styling* → [02 Design System](../02-design-system/README.md) supplies `Prose`.
- Gating logic for guides (email-for-download) → [10 Lead Capture](../10-lead-capture/README.md).
  This segment only marks a guide as `gated: true`.
- Sitemap and RSS generation → [11 SEO](../11-seo-analytics/README.md) consumes the adapter.

## Files

| File | Contents |
|---|---|
| [content-model.md](./content-model.md) | Every content type, its frontmatter schema, the adapter API, the MDX pipeline |

## Why MDX in-repo, not a CMS (v1)

The user has a VPS and could self-host Payload CMS today. The reasons not to, yet:

1. **Volume doesn't justify it.** v1 is roughly 20 pages and zero blog backlog. A CMS is a database,
   an admin authentication surface, a media store, a backup policy and an upgrade treadmill — all
   maintained *before* there is content to justify it.
2. **A public CMS admin is attack surface** on the same VPS family as an HR product. Files have none.
3. **Review and rollback are better with files.** Content changes get a diff, a reviewer and a
   `git revert`. Recovering a bad CMS edit means restoring a database.
4. **A typo fix does not need a CMS** — it needs a one-line commit.
5. **The hedge is cheap.** The adapter (D-302) means adding Payload later is confined to
   `lib/content/`, not to every page.

**When to revisit:** when a non-developer needs to publish more than ~2 posts a month without
waiting on an engineer, or when a marketing hire arrives. At that point, Payload CMS + Postgres on
the same VPS, behind the same adapter. That is a planned Phase 4 item, not a failure of this plan.

**Note on the VPS question:** the VPS is where the site *runs*; git is where the code *lives* and is
what CI deploys *from*. Self-hosting does not remove the need for version control — it makes it more
important, because there is no platform-managed rollback to fall back on.

## Key decisions taken in this plan

| # | Decision | Rationale |
|---|---|---|
| D-301 | Content is **MDX files in `content/`, committed to git**. | See above. |
| D-302 | **All content access goes through `lib/content/`.** No page, component or route may import from `content/` directly; enforced by an ESLint `no-restricted-imports` rule. | This one rule is what makes the CMS swap a contained change rather than a rewrite. |
| D-303 | Frontmatter is **validated with Zod at build time**, and a schema violation **fails the build**. | A missing `description` is a silently degraded search result. Better to break the build than ship it. |
| D-304 | Adapter return types (`Post`, `Guide`, `Module`, `Author`) are **CMS-agnostic** — no field named after a file path or an MDX quirk. | The types are the contract a CMS would later have to satisfy. |
| D-305 | MDX has a **small, explicit component allowlist** (`Callout`, `Figure`, `Cta`, `Stat`, `Video`). Arbitrary JSX and imports inside content are not permitted. | Content that can import anything is code, and stops being portable to a CMS. |
| D-306 | Content images live in `public/images/content/<collection>/<slug>/` and are referenced by a plain path, rendered through a `MdxImage` wrapper over `next/image`. | Authors write a path; the wrapper supplies dimensions, lazy loading and formats. |
| D-307 | Every content type carries `draft: boolean`. Drafts render on staging, are **excluded from production builds**, the sitemap and RSS. | Lets work-in-progress be reviewed on a real URL without leaking. |
| D-308 | Global UI copy (nav labels, footer links, CTA text) lives in **typed TS files** in `content/site/`, not MDX. | It is structured data, not prose; TypeScript catches a missing nav href, MDX would not. |
| D-309 | Legal pages (`privacy`, `terms`, `dpa`, `cookies`) are MDX with a mandatory `lastUpdated` date rendered on the page. | A privacy policy with no visible effective date is a compliance smell. |
| D-310 | Reading time is **computed**, not authored. Dates are ISO `YYYY-MM-DD` and formatted at render. | Authors should not be able to get derived data wrong. |

## Dependencies

- **Depends on:** [01 Foundation](../01-foundation-tech-stack/README.md) (MDX packages, Zod),
  [02 Design System](../02-design-system/README.md) (`Prose`, `Callout`).
- **Depended on by:** 05 (module pages, legal), 06 (blog, guides), 08 (sitemap, RSS, metadata),
  07 (gated guide metadata).

## Tasks

| ID | Task | Done when |
|---|---|---|
| T-301 | Install `next-mdx-remote`, `gray-matter`, `remark-gfm`, `rehype-slug`, `rehype-autolink-headings`, `rehype-pretty-code`, `remark-reading-time`. | — |
| T-302 | Write `lib/content/schemas.ts` — Zod schema per content type per [content-model.md](./content-model.md). | Invalid frontmatter throws with the file path in the message |
| T-303 | Write `lib/content/types.ts` — CMS-agnostic result types. | — |
| T-304 | Write `lib/content/index.ts` — the adapter API; cache reads per build with `React.cache`. | Content read once per build, not once per page |
| T-305 | Write `lib/content/mdx.ts` — plugin pipeline and `mdx-components.tsx` allowlist. | A sample post renders with anchors, GFM tables and highlighted code |
| T-306 | ESLint rule forbidding `content/` imports outside `lib/content/`. | Violating import fails `pnpm lint` |
| T-307 | Draft filtering keyed on `NODE_ENV` / an explicit `SHOW_DRAFTS` flag. | Draft post 404s in a production build, renders on staging |
| T-308 | `content/site/navigation.ts` and `cta.ts`, typed. | Header and footer read from them |
| T-309 | Seed content: 2 blog posts, 1 guide, 8 module files, 4 legal pages — placeholder copy clearly marked `TODO-COPY`. | Every route renders with real structure |
| T-310 | `CONTENT.md` in the app repo: how to add a post, frontmatter reference, image rules, the publish flow. | A non-developer could follow it with a git client |
| T-311 | Unit tests for the adapter: listing order, draft filtering, slug resolution, schema failure. | `pnpm test` green |

## Open questions

| ID | Question | Why it matters | Proposed default |
|---|---|---|---|
| OQ-301 | Who authors content after launch, and are they comfortable with git? | Decides how soon the Phase 4 CMS is needed. | Assume a developer-assisted flow in v1; revisit at the first marketing hire |
| OQ-302 | Is a second language needed (OQ-003)? | i18n changes the content directory shape (`content/<locale>/…`) and every adapter signature. | English only; adapter signatures accept an optional `locale` argument that is currently unused but reserves the shape |
| OQ-303 | Who supplies the legal text for privacy, terms and DPA? | Cannot be written by Claude or shipped as placeholder — a wrong privacy policy is a legal exposure, not a copy bug. | **Blocking for launch.** Legal review required; pages ship only when real text exists |
| OQ-304 | Will blog posts have multiple named authors with bios, or publish under "Vexalid Team"? | Affects `Author` type, byline UI and Article JSON-LD. | Support named authors from day one; default to a `vexalid-team` author record |
| OQ-305 | Are any guides genuinely gated, or is gating theatre that costs traffic? | Gating trades reach for leads. | Gate 1–2 substantial guides; keep all blog posts ungated |
