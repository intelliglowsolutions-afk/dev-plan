# 01 — Foundation & Tech Stack

**Priority:** Must-have · **Build order:** 1 of 13 · **Status:** planned

## Purpose

Choose the runtime, framework, libraries and repository layout for `vexalid.com`, and stand up a
working skeleton that every other segment builds inside. When this segment is done, a placeholder
home page renders from the real stack, in the real folder structure, deployable to the real VPS.

## Scope

**In scope**

- Framework, language, styling, package manager and Node version decisions, with rationale.
- Library selection for content, forms, validation, animation, icons, email and analytics clients.
- Repository layout: directory conventions, path aliases, naming rules.
- Configuration baseline: `tsconfig`, ESLint, Prettier, `next.config.ts`, PostCSS/Tailwind, env schema.
- Local development workflow and the `.env.example` contract.
- Git conventions: branch model, commit style, what is and is not committed.

**Out of scope (owned elsewhere)**

- Design tokens and component implementation → [02 Design System](../02-design-system/README.md).
- Content file formats and frontmatter schemas → [03 Content Layer](../03-content-layer/README.md).
- Docker images, nginx, TLS, CI/CD pipelines, the VPS itself → [12 Infrastructure](../12-infrastructure-deployment/README.md).
  This segment only ensures the app is *containerisable* and has no serverless-only dependencies.
- Test authoring; only the test *harness* choice lives here → [13 Launch & QA](../13-launch-qa/README.md).

## Files

| File | Contents |
|---|---|
| [stack-decisions.md](./stack-decisions.md) | Every technology choice, the alternatives rejected, and why |
| [project-structure.md](./project-structure.md) | Directory tree, path aliases, naming conventions, config files, env vars |

## The stack at a glance

| Layer | Choice | Version |
|---|---|---|
| Runtime | Node.js LTS | 22.x |
| Package manager | pnpm | 9.x |
| Framework | Next.js, App Router | 16.x |
| UI library | React | 19.x |
| Language | TypeScript, `strict` | 5.x |
| Styling | Tailwind CSS v4 (CSS-first config) | 4.x |
| Primitives | Radix UI (unstyled) | latest |
| Icons | `lucide-react` | latest |
| Content | MDX files + `gray-matter` + `next-mdx-remote` behind an adapter | — |
| Schema validation | Zod | 4.x |
| Forms | React Server Actions + Zod, progressive enhancement | — |
| Database (leads only) | PostgreSQL via Prisma | PG 17 / Prisma 6.x |
| Transactional email | Resend or Postmark SDK | — |
| Newsletter | Listmonk, self-hosted on the VPS | latest |
| Animation | CSS transitions first; `motion` only where genuinely needed | — |
| Analytics | Plausible, self-hosted | latest |
| Error monitoring | Sentry (SaaS free tier) or GlitchTip self-hosted | — |
| Unit tests | Vitest | latest |
| E2E / smoke | Playwright | latest |
| Lint / format | ESLint 9 (flat config) + Prettier | — |
| Deploy target | Docker container on the user's VPS behind nginx | — |

## Key decisions taken in this plan

| # | Decision | Rationale |
|---|---|---|
| D-101 | **Next.js 16 App Router**, not Astro, not a static site generator. | The site needs server-side form handling (demo requests write to Postgres and send email) and gated content. Astro would be marginally faster on static pages but would mean a second framework, second set of conventions, and no reuse from `hrm-system`. |
| D-102 | **Separate repository and separate deploy** from `hrm-system`. | A marketing copy fix must not trigger a redeploy of a payroll system. Different release cadence, different risk profile, different audience for downtime. |
| D-103 | **Mirror the HRM stack where there's no reason to differ** (Next, React, TS, Tailwind, Prisma). | Two apps, one mental model. Patterns, snippets and fixes transfer. |
| D-104 | **Static rendering by default.** Every marketing page is statically generated at build time; only form actions and gated downloads run per-request. | Sub-second TTFB on a modest VPS, trivial caching, and a static page cannot be taken down by a database outage. |
| D-105 | **Tailwind v4 with CSS-first config**, tokens as CSS custom properties in `@theme`. | v4's config-in-CSS means the design tokens from segment 02 have exactly one home, readable by both Tailwind utilities and hand-written CSS. |
| D-106 | **No component library** (no MUI, Chakra, shadcn wholesale). Radix primitives + own components on own tokens. | A marketing site's visual identity *is* the deliverable. Overriding someone else's design system costs more than writing ~20 components. Radix is taken only for the accessibility-hard parts: dialog, dropdown, accordion, tabs, tooltip. |
| D-107 | **Postgres + Prisma for leads**, not a SaaS form backend. | Self-hosting was a stated requirement; the data is prospect PII and belongs on infrastructure the user controls. Prisma is already the team's ORM. Separate database (`vexalid_web`) — never the HRM database. |
| D-108 | **Transactional email through a provider, not the VPS's own SMTP.** | A fresh VPS IP has no sending reputation; demo-request notifications landing in spam is a silent revenue leak. DKIM/SPF/DMARC on a managed sender is a one-time setup. |
| D-109 | **Self-hosted Plausible for analytics, not Google Analytics.** | Cookieless and first-party, so no consent banner is legally required and no third-party script blocks rendering. Also keeps prospect behaviour data on the user's own VPS. |
| D-110 | **pnpm, not npm.** | Faster installs and a strict node_modules layout that surfaces phantom dependencies before they reach the VPS build. |
| D-111 | **No `src/` directory**; app code at repo root in `app/`, `components/`, `lib/`, `content/`. | `hrm-system` uses `src/`, but this repo has a large `content/` tree that is authored, not compiled — keeping it a sibling of `app/` makes the authoring/code split obvious. Deliberate divergence, noted here so it isn't "fixed" later. |
| D-112 | **`content/` is committed to git.** | Content is versioned, reviewable and rolls back with the deploy. This is what replaces a CMS in v1 — see [03 Content Layer](../03-content-layer/README.md) D-301. |

## Dependencies

- **Depends on:** OQ-007 (VPS specifics) to confirm Docker and Node 22 are viable, and OQ-001 (domain).
  Neither blocks starting local development.
- **Depended on by:** every other segment.

## Tasks

| ID | Task | Done when |
|---|---|---|
| T-101 | `pnpm create next-app` in `C:\Dev\Vexalidweb Project`, TypeScript + Tailwind + App Router, no `src/`. | `pnpm dev` serves a page |
| T-102 | Read `node_modules/next/dist/docs/` before writing App Router code — this Next major has breaking changes from training data (see `hrm-system/AGENTS.md`). | Notes captured in the repo's own `AGENTS.md` |
| T-103 | `tsconfig.json`: `strict`, `noUncheckedIndexedAccess`, path aliases per [project-structure.md](./project-structure.md). | `pnpm typecheck` passes |
| T-104 | ESLint 9 flat config + Prettier + `pnpm lint`, `pnpm format:check` scripts. | Both pass on a clean tree |
| T-105 | Tailwind v4 `app/globals.css` with an empty `@theme` block ready for segment 02. | Utilities resolve |
| T-106 | `lib/env.ts` — Zod-parsed, fail-fast environment schema; `.env.example` committed. | App refuses to boot with a missing required var |
| T-107 | Prisma init against a local Postgres, `vexalid_web` schema, no models yet. | `pnpm prisma migrate dev` succeeds |
| T-108 | Git init, `.gitignore`, branch model (`main` = production, short-lived feature branches), conventional commits. | First commit on `main` |
| T-109 | Vitest + Playwright installed with one trivial passing test each. | `pnpm test`, `pnpm test:e2e` green |
| T-110 | `README.md` in the app repo: prerequisites, setup, scripts, and a link back to `C:\Dev\dev-plan\vexalid\`. | A new machine can get to `pnpm dev` from the README alone |

## Open questions

| ID | Question | Why it matters | Proposed default |
|---|---|---|---|
| OQ-101 | Is `C:\Dev\Vexalidweb Project` the final path? The space in the folder name breaks some CLI tooling and Docker bind mounts on Windows. | Tooling friction all the way through the project. | Rename to `C:\Dev\vexalid-web` before `pnpm create`. Recommend doing this now — it is free today and annoying later. |
| OQ-102 | Node 22 vs 24 on the VPS. | Must match local and container versions. | Node 22 LTS; pin in `.nvmrc`, `package.json#engines`, and the Dockerfile |
| OQ-103 | Is a separate Postgres instance available on the VPS, or must this share the HRM's? | D-107 requires database isolation for PII separation. | Same Postgres server, **separate database and separate role** with no grants on the HRM database |
| OQ-104 | `motion` (framer-motion) in the bundle at all? It is ~35 KB gzipped for effects CSS mostly covers. | Performance budget in segment 13. | Not installed in v1. Add only with a named page that needs it and a measured budget impact. |
| OQ-105 | Sentry SaaS (data leaves the VPS) vs self-hosted GlitchTip (one more service to run). | Consistency with the self-hosting stance. | GlitchTip on the VPS if resources allow (OQ-007); Sentry free tier otherwise, with PII scrubbing on |
