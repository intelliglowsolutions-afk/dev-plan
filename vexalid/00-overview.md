# Vexalid Website — Planning Overview

> Planning docs for **vexalid.com** — the company website for Vexalid, covering both its **services**
> (computerized system validation, AI automation, custom software) and its **own products** (the HRM
> system, and StratumOne).
>
> The HRM product itself is planned in [`../`](../00-overview.md) (the `dev-plan/` root) and built in
> `C:\Dev\hrm-system`. This site markets it — alongside everything else Vexalid sells.

## What Vexalid is

A Sri Lanka–based compliance engineering company that does two things:

**Services** — validation and software work for **regulated organisations**, principally
pharmaceutical manufacturing. Current positioning: *"Validation · Automation · Innovation"* /
*"Confidence built into every system."*

**Products** — its own software, built with the same validation discipline:
- **HRM system** — the full HR suite planned in `dev-plan/01`–`11`
- **StratumOne** — a centralized clock system ⚠️ *(scope unconfirmed — see OQ-003)*

The distinctive thing about Vexalid is that these two halves reinforce each other: a company that
validates GMP systems for a living builds its own software validation-ready. That is the positioning
the whole site is built on.

## What the site must do

1. Establish Vexalid as credible to a **pharma QA / IT manager** evaluating a validation partner.
2. Establish the **products** as real, supported software — not consultancy side-projects.
3. Convert both audiences: a **services enquiry** and a **product demo request** are different asks
   with different urgency, and need separate paths.
4. Capture lower-intent researchers via **newsletter and gated guides**.
5. Rank for both service terms (CSV, GMP validation) and product terms (HR software).

## Current site — what it replaces

`vexalid.com` is live today as a **single-page site**. Every nav item (`Expertise`, `Approach`,
`Services`, `Let's talk`) is an anchor to a section on that one page. It carries:

- Hero: *"Validation · Automation · Innovation"* / *"Confidence built into every system."*
- *"What we solve"* — *"Rigour for compliance. Intelligence for growth."*
- Three numbered services: 01 Computerized System Validation · 02 AI Automation · 03 Custom Software
- Four principles: quality first · intelligent automation · tailored solutions · continuous improvement
- Four-step process: Discover → Design → Deliver → Improve
- Contact: +94 71 955 7557 · contact@vexalid.com · 301/2, Mihidu Mawatha, Makola North, Sri Lanka

**Nothing about the products appears on it.** That is the gap this project closes.

Because the existing URLs are anchors (`#services`, `#approach`) rather than pages, the redirect risk
is low — but the anchors themselves should be preserved as redirects to the new pages
([04 Site Structure](./04-site-structure-navigation/README.md) D-410).

## Confirmed decisions

| # | Decision | Source |
|---|---|---|
| G-01 | **Everything on one domain: `vexalid.com`.** Services at `/services/*`, products at `/products/*`. No product subdomains, no separate product domains. | User |
| G-02 | **Full rebuild** of vexalid.com — company, three service lines, both products, resources, lead capture. The existing one-pager is replaced. | User |
| G-03 | **Two-track home page.** The hero states the compliance-engineering identity, then splits cleanly into Services and Products so each visitor self-selects. | User |
| G-04 | **The HRM leads with regulated manufacturers** — GMP, audit trails, validation-ready. Broader market reachable but not the headline. | User |
| G-05 | Separate Next.js application and repo from `hrm-system`; code in `C:\Dev\vexalid-web`. | User |
| G-06 | Content authored as **MDX in-repo** behind a content adapter, so a CMS can replace it later without touching pages. | User |
| G-07 | **Self-hosted on the user's own VPS.** No Vercel, no serverless assumptions. | User |
| G-08 | v1 conversion paths: **services enquiry**, **product demo request**, **newsletter / gated content**. No self-serve trial signup. | User |
| G-09 | Public pricing with plan tiers is **deferred** past v1. | User |
| G-10 | Same core stack as `hrm-system` (Next.js 16 + React 19 + TypeScript + Tailwind v4 + Prisma + Postgres). | [01 Foundation](./01-foundation-tech-stack/README.md) |

## Segments (build order)

| # | Segment | Priority | Ships in |
|---|---|---|---|
| 01 | [Foundation & Tech Stack](./01-foundation-tech-stack/README.md) | Must-have | v1 |
| 02 | [Design System](./02-design-system/README.md) | Must-have | v1 |
| 03 | [Content Layer](./03-content-layer/README.md) | Must-have | v1 |
| 04 | [Site Structure & Navigation](./04-site-structure-navigation/README.md) | Must-have | v1 |
| 05 | [Home & Company Pages](./05-home-company/README.md) | Must-have | v1 |
| 06 | [Services Pages](./06-services-pages/README.md) — CSV, AI automation, custom software | Must-have | v1 |
| 07 | [Products Pages](./07-products-pages/README.md) — HRM + modules, StratumOne | Must-have | v1 |
| 08 | [Industries](./08-industries/README.md) — the regulated-manufacturing wedge | Should-have | v1 |
| 09 | [Resources & Blog](./09-resources-blog/README.md) | Should-have | v1.1 |
| 10 | [Lead Capture](./10-lead-capture/README.md) | Must-have | v1 |
| 11 | [SEO & Analytics](./11-seo-analytics/README.md) | Must-have | v1 |
| 12 | [Infrastructure & Deployment](./12-infrastructure-deployment/README.md) | Must-have | v1 |
| 13 | [Launch & QA](./13-launch-qa/README.md) | Must-have | v1 |

01→04 are sequential foundations. 05, 06, 07, 08 then run in parallel. 10, 11, 12 proceed alongside;
12 should be stood up in week one (a staging URL on the VPS), not at the end. 13 is a gate.

## Recommended phasing

**Phase 1 — Foundation (01–04).** Repo, Docker image, staging on the VPS behind basic auth, design
tokens carried over from the existing brand, component primitives, content adapter, final sitemap
and navigation. Nothing public.

**Phase 2 — Parity + products (05, 06, 07, 10, 11, 12, 13).** Everything the current one-pager says,
now as real pages, **plus** the two product sections and both lead-capture paths. This is the
minimum publishable replacement — it must not be a downgrade from what is live today.

**Phase 3 — The wedge (08) + resources (09).** The regulated-manufacturing industry page, blog,
guides, gated content.

**Phase 4 — Post-launch.** Pricing tiers, case studies, careers, a `/security` page, additional
industry pages, Payload CMS. Each is an explicit re-plan.

**Rule: do not take the current site down until Phase 2 is complete.** A live one-pager that converts
is worth more than a half-finished multi-page site.

## Cross-cutting concerns

- **Two audiences, one site.** A pharma QA manager evaluating a validation partner and an HR manager
  evaluating software are different people with different vocabularies. The navigation must let each
  find their track within one click, and no page may try to serve both at once.
- **Two conversion endpoints.** `/contact` (services — consultative, "let's talk") and `/demo`
  (products — specific, "show me"). Mixing them loses both.
- **The brand already exists.** The live site has a visual identity and a voice
  (*"Precision in the details. Clarity in the outcome."*). Segment 02 extends it rather than
  inventing one — but the current design has not been visually reviewed yet (OQ-002).
- **Claim discipline is higher here than on a normal marketing site.** Vexalid sells compliance. A
  site that overstates what its products do undermines the one thing it is selling. Every product
  claim is registered and status-tracked in [05](./05-home-company/copy-guidelines.md) D-504.
- **Performance budget.** Lighthouse ≥ 95, LCP < 1.8s on 4G, enforced as a build gate (segment 13).
- **Privacy.** Self-hosted cookieless analytics, so no consent banner is required. Any ad pixel or
  Google Analytics reverses that (segment 11 D-1101).
- **Accessibility.** WCAG 2.2 AA. Pharma procurement screens for it.
- **Sri Lanka based, serving regulated industries.** Whether the target market is domestic, regional
  or international changes keyword strategy, currency, and legal text (OQ-004).

## Consolidated open questions

| ID | Question | Blocks | Proposed default |
|---|---|---|---|
| OQ-001 | **What is the HRM product actually called?** "Vexalid HRM", or its own name like StratumOne? | 04, 05, 07 — every URL, nav label and page title | **Blocking.** Placeholder `Vexalid HRM` at `/products/hrm` throughout. StratumOne having its own name implies the HRM should too |
| OQ-002 | Brand assets: logo files, colour values, typefaces from the live site. Is the new site a continuation or a redesign? | 02 — token values | Continuation. Extract tokens from the live site; needs a visual review or the asset files |
| OQ-003 | **What is StratumOne, precisely?** "Centralized clock system" could mean (a) a synchronised time source for GMP data integrity, or (b) employee time-clock consolidation. | 07 — the whole page | **Blocking.** Reading (a) — NTP/time integrity for regulated systems — fits the CSV service line and is assumed throughout, clearly flagged |
| OQ-004 | Target market: Sri Lanka, South Asia, or international? | 11 keyword strategy, 05 copy, legal | Sri Lanka + regional, English only |
| OQ-005 | Is the legal entity name "Vexalid" alone, or a longer registered name? | 05 legal pages, 11 Organization schema | Confirm before legal pages ship |
| OQ-006 | Are there **real pharma clients** who would appear as a named reference or case study? | 06, 08 — the strongest possible proof | Assume none nameable; no fabricated logos or testimonials (D-503) |
| OQ-007 | Any certifications, accreditations or standards Vexalid holds (ISO, GAMP 5 practitioner, etc.)? | 06 CSV page credibility | Claim nothing uncertified; describe methodology factually |
| OQ-008 | Is the HRM actually built and deployable today, or still in development? | 07 — decides `available` vs `Coming soon` | Assume **in development**; the plan's default is honest labelling |
| OQ-009 | VPS specs, OS, existing services, shared with the HRM product or not? | 12 — **blocking** | Ubuntu 24.04, 2 vCPU / 4 GB, Docker; marketing site isolated from the product |
| OQ-010 | Transactional email provider, and SPF/DKIM/DMARC on `vexalid.com`? | 10, 12 — **blocking for launch** | A managed provider, **not** raw VPS SMTP |
| OQ-011 | Who supplies privacy policy, terms and DPA text? | **Launch** | Legal review required; no placeholder legal text ships |
| OQ-012 | Who writes marketing copy, and who reviews technical claims about validation and GMP? | 05, 06, 08 | A named subject-matter reviewer is required for every compliance claim |

## Files in this plan

```
dev-plan/vexalid/
├── 00-overview.md                     ← you are here
├── IMPLEMENTATION_LOG.md
├── 01-foundation-tech-stack/          README, stack-decisions, project-structure
├── 02-design-system/                  README, brand-tokens, components
├── 03-content-layer/                  README, content-model
├── 04-site-structure-navigation/      README, sitemap, navigation
├── 05-home-company/                   README, page-specs, copy-guidelines
├── 06-services-pages/                 README, page-specs
├── 07-products-pages/                 README, page-specs
├── 08-industries/                     README
├── 09-resources-blog/                 README, requirements
├── 10-lead-capture/                   README, requirements, data-model, api-design
├── 11-seo-analytics/                  README, requirements
├── 12-infrastructure-deployment/      README, vps-runbook
└── 13-launch-qa/                      README, checklist
```
