# 01 — Foundation & Tech Stack — Stack Decisions

Each decision records what was chosen, what was rejected, and the cost of being wrong. Revisit a
decision only by amending it here with a date and reason.

## 1. Framework

**Chosen: Next.js 16, App Router, self-hosted in standalone output mode.**

| Alternative | Why rejected |
|---|---|
| Astro | Fastest static output and excellent MDX story, but the site needs server-handled forms, gated downloads and rate limiting. Astro can do this via adapters, at the cost of a second framework in the organisation and zero transfer from `hrm-system`. |
| Plain Vite + React SPA | No SSG/SSR means no meaningful SEO without pre-rendering bolted on. Non-starter for a site whose main job is organic search. |
| WordPress | Gives non-technical editing immediately, but the maintenance and security surface of a public PHP CMS on a VPS is a permanent tax, and the design would fight the theme system. Also splits the stack. |
| Hugo / Eleventy | Excellent static builds, no React, no server actions, and the team's skills are in React. |

**Cost of being wrong:** low. Marketing pages are mostly content and components; a migration would
be a rewrite of layout code, not of business logic.

**Configuration notes**
- `output: 'standalone'` in `next.config.ts` — produces a minimal self-contained server directory,
  which is what the Docker image copies. Without this the image carries the whole `node_modules`.
- No `next/image` remote patterns unless a CMS arrives; all images are local and optimised at build.
- `images.formats: ['image/avif', 'image/webp']`.
- Read `node_modules/next/dist/docs/` before writing App Router code. This Next major has breaking
  changes relative to model training data — the same warning `hrm-system/AGENTS.md` carries.

## 2. Rendering strategy

**Chosen: static by default, dynamic only where unavoidable.**

| Route class | Rendering | Note |
|---|---|---|
| Home, product, module, company, legal pages | SSG | `export const dynamic = 'force-static'` |
| Blog index, post pages, guide pages | SSG with `generateStaticParams` from the content adapter | Rebuild on content change |
| `/demo`, `/contact`, newsletter forms | Static page, **Server Action** handles POST | Page itself still cacheable |
| Gated guide download | Route handler, per-request | Validates the email token before streaming the PDF |
| `/sitemap.xml`, `/rss.xml`, `/robots.txt` | Built at build time | Next's metadata routes |
| Nothing | ISR / `revalidate` | Rejected in v1 — content changes arrive with a deploy, so time-based revalidation buys nothing but cache-coherence bugs |

**Cost of being wrong:** low, and reversible per route.

## 3. Styling

**Chosen: Tailwind CSS v4, CSS-first configuration, design tokens in `@theme`.**

| Alternative | Why rejected |
|---|---|
| CSS Modules | Perfectly viable, but no shared token vocabulary without extra plumbing, and slower to build marketing layouts in. |
| styled-components / Emotion | Runtime CSS-in-JS conflicts with React Server Components and adds hydration cost for zero benefit on a static site. |
| Vanilla CSS with custom properties only | Would work; loses the layout velocity and the enforced spacing scale that Tailwind's constraints provide. |

**Notes**
- Tailwind v4 has no `tailwind.config.js` by default. Tokens live in `app/globals.css` under `@theme`,
  which makes them real CSS custom properties readable outside Tailwind too. Segment 02 owns their values.
- `@tailwindcss/typography` for MDX prose (`.prose`) — required for blog and guide bodies.
- No arbitrary-value soup in components: if a value is used three times it becomes a token.

## 4. Component primitives

**Chosen: Radix UI primitives, unstyled, wrapped in own components.**

Taken from Radix, only where accessibility is genuinely hard: `Dialog`, `DropdownMenu`, `Accordion`,
`Tabs`, `Tooltip`, `NavigationMenu`, `VisuallyHidden`.

Hand-written, because they are trivial and the design is the point: `Button`, `Input`, `Textarea`,
`Select`, `Checkbox`, `Badge`, `Card`, `Container`, `Section`, `Prose`.

| Alternative | Why rejected |
|---|---|
| shadcn/ui wholesale | Copies in a large surface of opinionated styling that then has to be un-styled to match the brand. Cherry-pick patterns from it by hand instead. |
| MUI / Chakra / Mantine | Heavy, opinionated, and their visual identity leaks through everything. Wrong tool for a brand site. |
| Headless UI | Fine, but a smaller primitive set than Radix and less momentum. |

## 5. Content

**Chosen: MDX files in `content/`, parsed with `gray-matter` + `next-mdx-remote/rsc`, frontmatter validated by Zod, all of it behind `lib/content/`.**

Full reasoning and the CMS migration path are in
[03 Content Layer](../03-content-layer/README.md). Summary of why not a CMS in v1:

- v1 is roughly 20 pages and zero blog backlog. A CMS is a database, an admin auth surface, a backup
  policy and an upgrade treadmill to maintain *before* there is content to justify it.
- MDX gives content review through pull requests and rollback through deploys — both things a CMS makes harder.
- The adapter (D-302) means Payload can be swapped in behind the same function signatures later.

**Cost of being wrong:** moderate, and explicitly hedged. If marketing needs self-serve editing
sooner than expected, the work is confined to `lib/content/` plus a Payload instance on the VPS.

## 6. Forms and validation

**Chosen: React Server Actions + Zod, with progressive enhancement.**

- One Zod schema per form in `lib/schemas/`, imported by both the client (inline validation) and
  the Server Action (authoritative validation). Never trust the client copy.
- Forms use a real `<form action={serverAction}>` so they submit without JavaScript.
- Anti-spam: hidden honeypot field + a minimum time-to-submit check + Cloudflare Turnstile.
  Turnstile over reCAPTCHA because it is privacy-preserving and does not require a consent banner.
- Rate limiting per IP in Postgres (no Redis in v1 — see segment 10 D-1004).

| Alternative | Why rejected |
|---|---|
| React Hook Form + API route | More code and a second validation path. Server Actions with Zod cover it. |
| Formspree / Netlify Forms / Tally embed | Prospect PII leaves the user's infrastructure, and the form's look is constrained by someone else's widget. |

## 7. Database

**Chosen: PostgreSQL 17, Prisma 6, a dedicated `vexalid_web` database with its own role.**

- Stores: demo requests, newsletter subscribers, gated-content leads, form rate-limit buckets,
  and an outbound email log. Nothing else.
- **Hard rule:** the marketing site's database role has no grants on the HRM database. A compromise
  of a public marketing site must not reach employee or payroll data.
- Migrations are checked in and run as an explicit deploy step (`prisma migrate deploy`), never
  automatically on container start.

| Alternative | Why rejected |
|---|---|
| SQLite file | Tempting for this volume, but backups, concurrent writes during a rolling deploy, and eventual CRM sync all get easier with the Postgres the VPS already runs. |
| Sharing the HRM database | Violates the isolation rule above. Non-negotiable. |
| Airtable / Google Sheets as the store | Prospect PII in a third-party sheet, plus API rate limits on a public endpoint. |

## 8. Email

**Chosen: a transactional email provider (Resend or Postmark) for outbound; self-hosted Listmonk for the newsletter list.**

- Transactional (demo-request notification to sales, auto-reply to the prospect, gated-content
  download link) goes through the provider. A fresh VPS IP has no sending reputation; these messages
  must arrive.
- Postmark if deliverability reporting matters most; Resend if developer ergonomics do. Either is
  fine — wrap it in `lib/email/` so the provider is one file.
- The newsletter list itself lives in **Listmonk on the VPS**: free, self-hosted, owns the
  subscriber data, handles double opt-in and unsubscribe compliance. Listmonk sends *through* the
  same transactional provider's SMTP relay, so reputation stays managed.
- Every send is logged to an `EmailLog` row before dispatch so a provider outage is diagnosable.

**Blocked on OQ-008:** the sending domain needs SPF, DKIM and a DMARC policy before any mail is sent
in production.

## 9. Analytics and monitoring

**Chosen: self-hosted Plausible; GlitchTip or Sentry for errors; nginx logs for the rest.**

- Plausible is cookieless and first-party, so **no cookie-consent banner is legally required** —
  which removes an entire UI surface, a conversion-rate tax, and a compliance risk. This is the main
  reason for the choice, not ideology.
- Custom events needed at launch: `demo_request_submitted`, `newsletter_subscribed`,
  `guide_downloaded`, `pricing_cta_clicked`, `app_login_clicked`.
- **If Google Analytics or any ad pixel is ever added, a consent banner becomes mandatory** and
  segment 11 must be re-planned. Flagged as D-1101.

## 10. Testing

| Layer | Tool | What it covers |
|---|---|---|
| Unit | Vitest | Content adapter, Zod schemas, slug and date helpers, email templating |
| Component | Vitest + Testing Library | Form validation states, nav behaviour |
| E2E smoke | Playwright | Every route returns 200; demo form happy path and validation path; newsletter double opt-in |
| Accessibility | `@axe-core/playwright` on every route | Zero serious/critical violations |
| Performance | Lighthouse CI against the built site | Budgets from segment 13; fails the build |
| Link integrity | `linkinator` or `lychee` over the built output | No broken internal links, no broken outbound links |

## 11. Package manager, Node, and lockfile

- **pnpm 9**, `packageManager` field pinned so CI and the VPS cannot drift.
- **Node 22 LTS**, pinned in `.nvmrc`, `engines`, and the Dockerfile base image.
- `pnpm-lock.yaml` committed; CI installs with `--frozen-lockfile`.
- Dependency updates via Renovate or Dependabot, grouped weekly, with the E2E suite as the gate.

## 12. What is deliberately absent

Listed so these are recognised as decisions rather than oversights.

| Absent | Why |
|---|---|
| Redis | No session state, no distributed cache needed. Rate limiting fits in Postgres at this volume. Add it when there is a measured reason. |
| A CMS | v1 content volume does not justify it. Phase 4, behind the adapter. |
| Internationalisation | Pending OQ-003. Route structure stays i18n-compatible (no locale segment now, but no locale-hostile assumptions either). |
| A design-token build pipeline (Style Dictionary) | Two consumers (Tailwind, hand-written CSS) both read CSS custom properties directly. A pipeline would be ceremony. |
| Storybook | ~20 components, one consuming app. The cost of maintaining a second render environment exceeds the benefit. Revisit if the component count triples. |
| A monorepo shared with `hrm-system` | D-102. Independent deploys are the point. |
| Server-side A/B testing | Requires dynamic rendering on every page, which contradicts D-104. Revisit post-launch with real traffic. |
