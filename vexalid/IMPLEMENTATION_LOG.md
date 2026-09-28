# Vexalid Website — Implementation Log

Single log of all planning and build work on **vexalid.com** — the company website covering Vexalid's
services (CSV, AI automation, custom software) and its own products (the HRM system, StratumOne).
**Every working session must append an entry before it ends** — newest at the top of "Session entries".

The HRM product itself has its own log at [`../IMPLEMENTATION_LOG.md`](../IMPLEMENTATION_LOG.md).

Source documents:
- [`00-overview.md`](./00-overview.md) — scope, phasing, cross-cutting concerns
- Segment folders `01/`–`13/`
- The live vexalid.com — the current one-page site this project replaces
- HRM product plan at [`../00-overview.md`](../00-overview.md) — the authority for what may be claimed
  on product pages (segment 05 D-504)

---

## Status board

### Planning

| # | Segment | README | Detail docs | Status |
|---|---|---|---|---|
| — | Overview | ✅ | — | ✅ Revised 2026-09-15 |
| 01 | Foundation & Tech Stack | ✅ | ✅ stack-decisions, project-structure | ✅ Done |
| 02 | Design System | ✅ | ✅ brand-tokens, components | ✅ Revised — brand now extends the live site |
| 03 | Content Layer | ✅ | ✅ content-model | ✅ Revised — services/products/industries collections added |
| 04 | Site Structure & Navigation | ✅ | ✅ sitemap, navigation | ✅ Rewritten for two tracks |
| 05 | Home & Company Pages | ✅ | ✅ page-specs, copy-guidelines | ✅ Rewritten |
| 06 | Services Pages | ✅ | ✅ page-specs | ✅ New |
| 07 | Products Pages | ✅ | ✅ page-specs | ✅ New |
| 08 | Industries | ✅ | — | ✅ New |
| 09 | Resources & Blog | ✅ | ✅ requirements | ✅ Revised — editorial plan covers both tracks |
| 10 | Lead Capture | ✅ | ✅ requirements, data-model, api-design | ✅ Revised — second enquiry form added |
| 11 | SEO & Analytics | ✅ | ✅ requirements | ✅ Revised — two-track keyword map |
| 12 | Infrastructure & Deployment | ✅ | ✅ vps-runbook | ✅ Done (unaffected by the restructure) |
| 13 | Launch & QA | ✅ | ✅ checklist | ✅ Done |

### Build

_Not started. No application code yet._

| Phase | Contents | Status |
|---|---|---|
| 1 — Foundation | 01–04: repo, Docker, staging, tokens, components, content adapter, routes | ⬜ Not started |
| 2 — Parity + products | 05, 06, 07, 10, 11, 12, 13: home, services, products, both forms, SEO, deploy | ⬜ Not started |
| 3 — Wedge + resources | 08, 09: pharmaceutical page, blog, guides | ⬜ Not started |
| 4 — Post-launch | Pricing tiers, case studies, careers, `/security`, more industries, Payload CMS | ⬜ Not started |

**Rule: the current one-pager stays live until Phase 2 is complete.**

---

## Blocking open questions

| ID | Question | Blocks | Owner |
|---|---|---|---|
| OQ-001 | **What is the HRM product actually called?** StratumOne has a name; the HRM needs one. | Every URL, nav label and title on the products track (04, 05, 07) | User |
| OQ-003 | **What is StratumOne, precisely?** Time synchronisation for GxP data integrity, or employee time-clock consolidation? | The whole StratumOne page; the HRM/StratumOne boundary; one blog post; one cross-link | User |
| OQ-002 | Brand assets from the live site — logo files, colour values, typefaces. Continuation or redesign? | Segment 02 token values | User |
| OQ-012 | **Who reviews the regulatory language?** (GMP, Part 11, GAMP, "validated") | CSV page, industries page — **blocking both** | User |
| OQ-009 | VPS specs, OS, existing services, shared with the HRM or not? | Segment 12, all of it | User |
| OQ-010 | Email provider, and SPF/DKIM/DMARC on vexalid.com | Segment 10 production email; launch | User |
| OQ-011 | Privacy policy, terms and DPA text | Launch | User / legal |
| OQ-008 | Is the HRM deployable today, in pilot, or in development? | `available` vs `Coming soon` across nine pages | User |

## Decisions awaiting the user

| ID | Question | Recommendation |
|---|---|---|
| OQ-004 | Target market — Sri Lanka, regional, or international? | Materially changes the SEO strategy. If domestic-only, organic search will not fill the services pipeline and the plan should say so |
| OQ-006 | Any nameable clients, or anonymous engagement descriptions? | Even *"a mid-size manufacturer preparing for inspection"* beats any amount of capability description |
| OQ-007 | Certifications or accreditations held? | Claim nothing uncertified |
| OQ-505 | British or American English? | British, matching the HRM plan |
| OQ-605 | Is AI Automation a delivered service with prior work, or an emerging offer? | If no delivered work, describe it as an offer — or hold the page back |
| OQ-1003 | Calendar booking (self-hosted Cal.com) instead of the demo form? | Form in v1; strong Phase 4 candidate given the VPS |

---

## Session entries

### 2026-09-15 (2) — Restructured: single-product site → company site with two tracks

**Trigger.** The user corrected a foundational assumption: Vexalid is not an HRM vendor. It is a
consulting and services company with **its own products**, of which the HRM is one; StratumOne, a
centralized clock system, is another. `vexalid.lk` does not resolve; the live site is `vexalid.com`.

**What the live site turned out to be.** A single-page site — every nav item (`Expertise`,
`Approach`, `Services`, `Let's talk`) is an anchor on one page. It carries the positioning
*"Validation · Automation · Innovation"* / *"Confidence built into every system"*, three numbered
service lines (Computerized System Validation, AI Automation, Custom Software), four principles,
a four-step process, and real contact details in Sri Lanka. **No mention of the products at all.**

**Decisions taken with the user**
- **G-01** Everything on one domain. `/services/*` and `/products/*`. No subdomains.
- **G-02** Full rebuild; the one-pager is replaced.
- **G-03** Two-track home page: identity first, then a clean split.
- **G-04** The HRM leads with **regulated manufacturers** — GMP, audit trails, validation-ready.

**Structural changes**
- **10 segments → 13.** Split the old "Core Pages" into 05 Home & Company, and added
  06 Services Pages, 07 Products Pages, 08 Industries. Segments 06–10 renumbered to 09–13;
  all decision, task and open-question IDs renumbered to match.
- **Sitemap: 1 route → 36.** Full rewrite of segment 04.
- **Two conversion endpoints**, not one: `/contact` (services enquiry) and `/demo` (product demo),
  with separate forms, separate tables and separate follow-up (10 D-1014).
- **Context-aware header CTA** — `Talk to us` by default, `Book a demo` on `/products/**`, decided
  server-side from the route segment (04 navigation.md).
- **New content collections:** `Service`, `Product`, `Industry`; HRM modules moved under
  `content/products/hrm/modules/` and their route from `/product/[module]` to `/products/hrm/[module]`.
- **Two-track keyword map** (11), with an explicit warning that services terms are low-volume and may
  not be a viable acquisition channel in a small domestic market.
- **New build-enforced field: `reviewedBy`** on service and industry pages. A page carrying
  regulatory language with no recorded practitioner reviewer **fails the build** (06 D-607).

**The positioning the whole site now rests on**

> The people who validate your systems also build software — and hold it to the same standard.

This is stated on the home page, proved on `/services/custom-software` (the products *are* the
portfolio), and argued in full on `/industries/pharmaceutical`. Cross-track links are load-bearing,
not decorative: without them the site is two brochures sharing a domain. `cross_track_click` was
added as an analytics event specifically to measure whether it works.

**Claim discipline tightened.** Vexalid sells compliance, so overstating what its software does
damages the service line as well as the product. The claim register now covers services and products,
and three rules were added that matter more than any design decision in this plan:
- Never "validated" of software — only "validation-ready", with the distinction explained.
- Never "21 CFR Part 11 compliant" of a product; compliance is a property of a validated installation.
- Never "payroll for <country>" — `dev-plan/07` is explicit that payroll has no built-in country rules.

**Two blocking unknowns remain, and both are cheap to answer:** the HRM's product name (OQ-001) and
StratumOne's actual scope (OQ-003). Segment 07 documents both possible readings of "centralized clock
system" and assumes the GxP time-integrity one, flagged throughout.

**Housekeeping.** The bulk renumbering script introduced two encoding/replacement faults (UTF-8 read
as ANSI; a case-insensitive replace corrupting the word "place" and stripping `@@` from Prisma
attributes). All three were detected and repaired in-session; a verification pass over all 28 files
came back clean.

**Next**
1. Answer OQ-001 and OQ-003 — they block segment 07 entirely.
2. Name the regulatory reviewer (OQ-012) — blocks the CSV and industries pages.
3. Supply brand assets or confirm a redesign (OQ-002).
4. Confirm the target market (OQ-004) — it decides whether the services SEO strategy is worth funding.
5. Then: rename the project folder, `pnpm create next-app`, and stand up staging in week one.

**Open `TODO-COPY` items**
_None yet — no pages written._

---

### 2026-09-15 (1) — Planning: initial website plan created

Created `dev-plan/vexalid/` with an overview, this log, and ten segment folders mirroring the HRM
plan's convention. Wrote 22 planning documents covering tech stack, design system, content layer,
site structure, pages, blog, lead capture, SEO, infrastructure and launch QA.

**Decisions from that session that still stand**
- **G-05** Separate Next.js app and repo from `hrm-system`; code in `C:\Dev\vexalid-web`
  (rename from `Vexalidweb Project` — the space breaks tooling).
- **G-06** Content as MDX in-repo behind an adapter, with self-hosted Payload CMS as a Phase 4
  migration. *The user's reply ("we have our own VPS, no need to put to git") conflated content
  management with hosting; clarified in-session — the VPS is where the site runs, git is where the
  code lives and what CI deploys from. Self-hosting makes version control more important, not less.*
- **G-07** Self-hosted on the user's VPS. No Vercel, no serverless assumptions.
- **G-10** Next.js 16 + React 19 + TypeScript + Tailwind v4 + Prisma + Postgres, mirroring the HRM
  stack so knowledge transfers.
- Static rendering by default (01 D-104); leads in a separate `vexalid_web` database with **no
  grants on the HRM database** (01 D-107); self-hosted cookieless Plausible, removing the
  cookie-consent banner entirely (11 D-1101); transactional email through a managed provider, not
  VPS SMTP (01 D-108); Docker + compose behind host nginx, CI builds and the VPS only pulls
  (12 D-1201, D-1205).

**Observation on the HRM plan, still open:** `dev-plan/IMPLEMENTATION_LOG.md` records features 01–03
as planned, but only `01-roles-permissions-auth/` has documents on disk — `02-employee-management/`
and `03-admin-settings/` are empty. Worth checking whether that work was lost. It matters here
because the claim register needs each module's real scope; with only feature 01 documented, every
module defaults to `status: planned`.
