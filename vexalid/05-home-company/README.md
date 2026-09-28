# 05 — Home & Company Pages

**Priority:** Must-have · **Build order:** 5 of 13 · **Status:** planned (revised for the two-track site)

## Purpose

Build the pages that belong to neither track exclusively: the home page (which routes between them),
`/about`, `/approach`, the two conversion pages, thank-you pages, legal pages and error pages. Plus
the copy rules and the claim register that govern **every** page on the site.

## Scope

**In scope**

- `/` — the two-track home page (G-03).
- `/about` — who Vexalid is.
- `/approach` — Discover → Design → Deliver → Improve, carried over from the live site.
- `/contact` and `/demo` — the two conversion pages (form logic in segment 10).
- `/thank-you/[type]`, `/legal/[slug]`, 404, 500.
- Voice, copy rules, and the **claim register** covering services and products.

**Out of scope (owned elsewhere)**

- Service pages → [06 Services Pages](../06-services-pages/README.md).
- Product pages → [07 Products Pages](../07-products-pages/README.md).
- `/industries/pharmaceutical` → [08 Industries](../08-industries/README.md).
- Blog and guides → [09 Resources & Blog](../09-resources-blog/README.md).
- Form logic → [10 Lead Capture](../10-lead-capture/README.md).
- Metadata and JSON-LD → [11 SEO](../11-seo-analytics/README.md).

## Files

| File | Contents |
|---|---|
| [page-specs.md](./page-specs.md) | Section-by-section spec for each page here |
| [copy-guidelines.md](./copy-guidelines.md) | Voice, the claim register, and what may be claimed |

## Key decisions taken in this plan

| # | Decision | Rationale |
|---|---|---|
| D-501 | **Every page is a stack of `Section` components.** Bespoke layout markup in a page file signals a missing section component. | Consistent vertical rhythm is most of what makes a site look professional, and it only survives if it is structural. |
| D-502 | **The home page routes; it does not sell.** Its job is to establish identity in one screen, then hand off to the right track. | Trying to sell CSV *and* HR software above the fold sells neither. |
| D-503 | **No fabricated social proof** — no invented testimonials, placeholder client logos or made-up statistics, not even temporarily. Sections whose real content does not exist are **omitted**, not faked. | For a company whose product is compliance and evidence, a fake client logo is not a minor sin. |
| D-504 | **Every claim maps to the claim register with a build status.** Unbuilt capabilities are labelled `Coming soon` and described in future tense. | An HR buyer who books a demo on the strength of a feature that does not exist is a lost deal. For a compliance firm it is also a credibility problem in the one area it sells. |
| D-505 | **One primary CTA per page, matching that page's track.** | Two equal-weight CTAs convert worse than one. |
| D-506 | **The `h1` states a benefit; the `<title>` carries the search term.** | They serve different readers — the person who arrived, and the person scanning search results. |
| D-507 | **Above the fold on every page: what it is, who it's for, and the CTA.** | Mobile shows ~600px. It has to work. |
| D-508 | **Placeholder copy is marked `TODO-COPY` inline and fails a production build.** | Prevents lorem-grade copy shipping by accident. |
| D-509 | **`/pricing` explains the model honestly without numbers** (G-09). | Evasion loses the visit; a straight explanation of what drives cost keeps it. |
| D-510 | **Legal pages ship only with real, reviewed text.** No placeholder privacy policy, ever. | A wrong privacy policy is a legal exposure, not a copy bug. Blocks launch. |
| D-511 | **`/approach` keeps the existing site's four steps and its language** (Discover → Design → Deliver → Improve; *"A dependable path forward"*, *"From uncertainty to assurance"*). | It already exists, it already reads well, and continuity costs nothing. Rewriting working copy for novelty is a waste. |
| D-512 | **The existing hero line — *"Validation · Automation · Innovation"* / *"Confidence built into every system"* — is retained on the home page.** | It is the company's established positioning, it appears on live material, and it happens to cover both tracks. Replace it only for a reason, not by default. |

## Page inventory

| Page | Route | Sections | Phase |
|---|---|---|---|
| Home | `/` | 9 | 2 |
| About | `/about` | 5 | 2 |
| Approach | `/approach` | 5 | 2 |
| Contact (services enquiry) | `/contact` | 3 | 2 |
| Demo (product request) | `/demo` | 3 | 2 |
| Pricing | `/pricing` | 5 | 2 |
| Thank you (×4) | `/thank-you/[type]` | 2 (template) | 2–3 |
| Legal (×4) | `/legal/[slug]` | prose template | 2 |
| 404 / 500 | — | 1 | 2 |

## Dependencies

- **Depends on:** 02 (section components), 03 (legal content), 04 (final routes and the CTA-switching
  rule), and 06/07 for the tracks the home page routes into.
- **Blocked on:** OQ-011 (legal text — blocks launch), OQ-001 (HRM name), OQ-005 (legal entity),
  OQ-012 (who writes and reviews copy).
- **Depended on by:** 06, 07, 08 (all inherit the copy rules and the claim register), 11, 13.

## Tasks

| ID | Task | Done when |
|---|---|---|
| T-501 | Build the claim register in [copy-guidelines.md](./copy-guidelines.md): read each `dev-plan/0X-*/README.md`, record what each module does and its real status, and record what may be claimed for each service. | Every row filled and agreed with the user |
| T-502 | Build `/` per [page-specs.md](./page-specs.md). | Passes the 5-second test: a stranger can say what Vexalid does **and** which track they belong in |
| T-503 | Build `/approach` (D-511). | Four steps, faithful to the existing copy |
| T-504 | Build `/about`. | Nothing unverifiable |
| T-505 | Build `/contact` — services enquiry, reduced chrome, cross-link to `/demo`. | — |
| T-506 | Build `/demo` — product request, reduced chrome, cross-link to `/contact`. | — |
| T-507 | Build `/pricing` per D-509. | — |
| T-508 | Build `/thank-you/[type]` template, four variants, `noindex`. | — |
| T-509 | Build `/legal/[slug]` prose template with the `lastUpdated` banner. | Renders; real text pending OQ-011 |
| T-510 | Build 404 and 500 offering both tracks. | — |
| T-511 | Implement the context-aware header CTA (04 navigation.md). | Switches by route segment, server-side, no layout shift |
| T-512 | CI check failing a production build on any `TODO-COPY`. | Verified by a deliberate failing commit |
| T-513 | Read every page aloud at 360px and 1440px. | No horizontal scroll, no orphaned headings |

## Open questions

| ID | Question | Why it matters | Proposed default |
|---|---|---|---|
| OQ-501 | Does the home page lead with the **company** ("compliance engineering") or the **outcome** ("confidence in your systems")? | Sets the first sentence a stranger reads. | Keep the existing *"Confidence built into every system"* (D-512) and let the two-track split do the explaining |
| OQ-502 | For `/about`: is there a real founding story, named team, or years-in-operation figure? | An about page of platitudes actively reduces trust — and this is a due-diligence path for pharma buyers. | Write short and factual; omit anything unverifiable rather than inventing it |
| OQ-503 | Should the home page show the products at all to a first-time services visitor, or does it dilute the consulting pitch? | The core tension of a two-track site. | Show both (G-03). A services buyer discovering you ship your own validated software is a *strength*, not a distraction |
| OQ-504 | Is there a stated response-time commitment for enquiries and demo requests? | A concrete promise materially raises form completion — and must be kept. | State it only if it will be honoured |
| OQ-505 | British or American English? | Consistency across ~36 pages. | British, matching the HRM plan's existing spelling. Confirm |
| OQ-506 | Support model for product customers — email only, or more? | Affects `/contact` and `/pricing`. | State plainly whatever is true; never imply 24/7 support |
