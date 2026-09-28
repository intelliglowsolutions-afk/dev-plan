# 01 — Foundation & Tech Stack — Project Structure

Repository root: `C:\Dev\vexalid-web` (see OQ-101 — rename from `Vexalidweb Project` to remove the
space before creating the project).

## Directory tree

```
vexalid-web/
├── app/                              # Next.js App Router — routes only
│   ├── layout.tsx                    # Root layout: fonts, <head>, skip link, analytics
│   ├── page.tsx                      # Home
│   ├── globals.css                   # Tailwind v4 @theme tokens + base layer  (segment 02)
│   ├── not-found.tsx                 # 404
│   ├── error.tsx                     # Client error boundary
│   ├── sitemap.ts                    # Generated from the content adapter     (segment 11)
│   ├── robots.ts
│   ├── rss.xml/route.ts
│   ├── opengraph-image.tsx           # Default OG image
│   ├── (marketing)/                  # Route group: shared header + footer chrome
│   │   ├── layout.tsx
│   │   ├── product/
│   │   │   ├── page.tsx              # Product overview
│   │   │   └── [module]/page.tsx     # attendance, leave, payroll, …          (segment 05)
│   │   ├── pricing/page.tsx
│   │   ├── resources/
│   │   │   ├── page.tsx              # Resources hub
│   │   │   ├── blog/
│   │   │   │   ├── page.tsx
│   │   │   │   └── [slug]/page.tsx
│   │   │   └── guides/[slug]/page.tsx                                          (segment 09)
│   │   ├── about/page.tsx
│   │   ├── contact/page.tsx
│   │   ├── careers/page.tsx
│   │   ├── legal/[slug]/page.tsx     # privacy, terms, dpa, cookies
│   │   └── thank-you/[type]/page.tsx
│   ├── demo/page.tsx                 # Outside (marketing): minimal chrome, focused conversion
│   └── api/
│       ├── newsletter/confirm/route.ts   # Double opt-in confirmation link
│       └── download/[token]/route.ts     # Gated guide delivery                (segment 10)
│
├── components/
│   ├── ui/                           # Primitives — no business meaning       (segment 02)
│   │   ├── button.tsx  input.tsx  textarea.tsx  select.tsx  checkbox.tsx
│   │   ├── card.tsx    badge.tsx   dialog.tsx   accordion.tsx  tabs.tsx
│   │   └── container.tsx  section.tsx  prose.tsx
│   ├── layout/
│   │   ├── site-header.tsx  site-nav.tsx  mobile-nav.tsx  site-footer.tsx
│   │   └── skip-link.tsx    breadcrumbs.tsx
│   ├── sections/                     # Composable page blocks                 (segment 05)
│   │   ├── hero.tsx  feature-grid.tsx  module-showcase.tsx  logo-wall.tsx
│   │   ├── testimonial.tsx  faq.tsx  stat-band.tsx  cta-band.tsx
│   │   └── comparison-table.tsx  pricing-table.tsx
│   ├── forms/                        # Form UIs + their client state          (segment 10)
│   │   ├── demo-request-form.tsx  newsletter-form.tsx  contact-form.tsx
│   │   ├── gated-download-form.tsx  form-field.tsx  form-status.tsx
│   │   └── turnstile.tsx
│   ├── content/                      # MDX renderers and MDX-available components (segment 03)
│   │   ├── mdx-components.tsx  mdx-image.tsx  callout.tsx  code-block.tsx
│   │   └── post-card.tsx  author-byline.tsx  table-of-contents.tsx
│   └── seo/
│       └── json-ld.tsx                                                        (segment 11)
│
├── content/                          # AUTHORED, not code. Committed to git.  (segment 03)
│   ├── blog/<yyyy>-<mm>-<slug>.mdx
│   ├── guides/<slug>.mdx             # Gated long-form
│   ├── modules/<slug>.mdx            # Per-HRM-module marketing copy
│   ├── legal/<slug>.mdx
│   ├── careers/<slug>.mdx
│   ├── authors/<slug>.json
│   └── site/                         # Global copy: nav labels, footer, CTAs
│       ├── navigation.ts
│       └── cta.ts
│
├── lib/
│   ├── env.ts                        # Zod-parsed environment, fail fast
│   ├── content/                      # THE ADAPTER — the only 1 that knows content is files
│   │   ├── index.ts                  # getPost, listPosts, getGuide, getModule, …
│   │   ├── schemas.ts                # Zod frontmatter schemas
│   │   ├── mdx.ts                    # compile/serialise, plugin config
│   │   └── types.ts                  # Post, Guide, Module, Author — CMS-agnostic shapes
│   ├── schemas/                      # Zod form schemas, shared client + server (segment 10)
│   ├── actions/                      # Server Actions                          (segment 10)
│   │   ├── submit-demo-request.ts  subscribe-newsletter.ts  submit-contact.ts
│   ├── email/                        # Provider wrapper + templates            (segment 10)
│   │   ├── client.ts  templates/
│   ├── db.ts                         # Prisma singleton
│   ├── analytics.ts                  # Typed Plausible event helpers           (segment 11)
│   ├── seo.ts                        # Metadata + JSON-LD builders             (segment 11)
│   ├── rate-limit.ts
│   └── utils/                        # cn(), slugify(), formatDate(), readingTime()
│
├── prisma/
│   ├── schema.prisma                                                          (segment 10)
│   └── migrations/
│
├── public/
│   ├── images/  logos/  og/  fonts/  favicon.ico  site.webmanifest
│   └── downloads/                    # Gated PDFs — served only via the token route, never linked
│
├── tests/
│   ├── unit/        # Vitest
│   └── e2e/         # Playwright                                              (segment 13)
│
├── .github/workflows/ci.yml                                                    (segment 12)
├── docker/  Dockerfile  docker-compose.yml  nginx.conf                         (segment 12)
├── .env.example  .nvmrc  .gitignore  .prettierrc
├── eslint.config.mjs  next.config.ts  postcss.config.mjs  tsconfig.json
├── AGENTS.md                         # Next-version warnings + repo conventions
└── README.md                         # → links to C:\Dev\dev-plan\vexalid\
```

## Path aliases

```jsonc
// tsconfig.json → compilerOptions.paths
{
  "@/*":            ["./*"],
  "@/components/*": ["./components/*"],
  "@/lib/*":        ["./lib/*"],
  "@/content/*":    ["./content/*"]
}
```

Only `@/`-prefixed imports. No relative imports that climb more than one level (`../../` is a lint
error) — this keeps files movable.

## Naming conventions

| Thing | Convention | Example |
|---|---|---|
| Component files | kebab-case | `demo-request-form.tsx` |
| Components | PascalCase, named exports, one per file | `export function DemoRequestForm()` |
| Hooks | `use-` prefix, kebab file | `use-media-query.ts` |
| Server Actions | verb-first, `'use server'` at the top of the file | `submit-demo-request.ts` |
| Zod schemas | `<Thing>Schema` + inferred `<Thing>Input` type | `DemoRequestSchema` |
| Content files | kebab-case slug; blog posts prefixed `YYYY-MM-` | `2026-09-hrm-buyers-guide.mdx` |
| CSS tokens | `--vx-<category>-<name>` | `--vx-color-accent-600` |
| Analytics events | snake_case, past tense | `demo_request_submitted` |
| Route segments | lowercase, hyphenated, never abbreviated | `/resources/case-studies` |

## Component rules

1. **Server Components by default.** `'use client'` only for interactivity, and as deep in the tree
   as possible. A `'use client'` on a page or layout is a review-blocking mistake.
2. **`components/ui/` knows nothing about Vexalid.** No product copy, no business logic. If a
   component mentions "demo request" it belongs in `components/forms/`.
3. **`components/sections/` are content-driven.** Props in, markup out — no data fetching inside a
   section component; the page fetches and passes down.
4. **Nothing outside `lib/content/` may read from `content/`.** This is what makes the CMS swap
   cheap. Enforced by an ESLint `no-restricted-imports` rule.
5. **No `any`.** No `@ts-expect-error` without a comment naming the issue.

## Environment variables

`lib/env.ts` parses these at boot with Zod and throws on a missing required value, so a
misconfigured container fails loudly at start rather than silently at the first form submission.

| Var | Required | Scope | Purpose |
|---|---|---|---|
| `NODE_ENV` | yes | both | — |
| `NEXT_PUBLIC_SITE_URL` | yes | client | Canonical URLs, OG tags, sitemap. e.g. `https://vexalid.com` |
| `NEXT_PUBLIC_APP_URL` | yes | client | The HRM product: `https://app.vexalid.com` |
| `DATABASE_URL` | yes | server | `vexalid_web` Postgres, dedicated role |
| `EMAIL_PROVIDER_API_KEY` | yes | server | Resend/Postmark |
| `EMAIL_FROM` | yes | server | e.g. `Vexalid <hello@vexalid.com>` |
| `SALES_NOTIFICATION_EMAIL` | yes | server | Where demo requests are announced |
| `LISTMONK_URL` / `LISTMONK_API_USER` / `LISTMONK_API_TOKEN` | yes | server | Newsletter list |
| `NEXT_PUBLIC_PLAUSIBLE_DOMAIN` | yes | client | Analytics site id |
| `NEXT_PUBLIC_PLAUSIBLE_HOST` | yes | client | Self-hosted Plausible origin |
| `TURNSTILE_SITE_KEY` / `TURNSTILE_SECRET_KEY` | yes | client / server | Anti-spam |
| `DOWNLOAD_TOKEN_SECRET` | yes | server | HMAC for gated-download links |
| `SENTRY_DSN` | no | both | Error monitoring |
| `REVALIDATE_SECRET` | no | server | Reserved for a future CMS webhook |

`.env.example` lists every key with a placeholder and a one-line comment. `.env*` is gitignored
except `.env.example`.

## Git conventions

- `main` is always deployable; production deploys from `main` only.
- Short-lived branches: `feat/…`, `fix/…`, `content/…`, `chore/…`.
- Conventional commits (`feat:`, `fix:`, `content:`, `chore:`, `docs:`).
- Content-only changes use `content:` so they are easy to filter and can skip the E2E suite in CI.
- **Committed:** `content/`, `prisma/migrations/`, `pnpm-lock.yaml`, `.env.example`, `public/` assets.
- **Not committed:** `.env*` (except example), `.next/`, `node_modules/`, `public/downloads/*` if any
  PDF exceeds ~10 MB (use Git LFS or ship it as a build artefact instead).

## npm scripts

| Script | Command |
|---|---|
| `dev` | `next dev` |
| `build` | `prisma generate && next build` |
| `start` | `next start` |
| `lint` | `eslint` |
| `format:check` | `prettier --check .` |
| `typecheck` | `tsc --noEmit` |
| `test` | `vitest run` |
| `test:e2e` | `playwright test` |
| `db:migrate` | `prisma migrate dev` |
| `db:deploy` | `prisma migrate deploy` |
| `check` | `pnpm typecheck && pnpm lint && pnpm test` — the pre-push gate |
