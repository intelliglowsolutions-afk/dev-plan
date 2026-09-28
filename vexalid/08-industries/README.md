# 08 — Industries

**Priority:** Should-have · **Build order:** 8 of 13 · **Status:** planned · **Phase:** 3

## Purpose

Build `/industries/pharmaceutical` — the one page where Vexalid's services and products are presented
together, to the audience they were both built for. It is the page that makes the two-track structure
read as a coherent company rather than two businesses sharing a domain.

## Scope

**In scope**

- `/industries/pharmaceutical` — content, structure, and the argument it makes.
- The industry page template, built so a second industry costs a content file rather than a rebuild.
- The criteria for adding further industry pages.

**Out of scope (owned elsewhere)**

- Service pages → [06 Services Pages](../06-services-pages/README.md).
- Product pages → [07 Products Pages](../07-products-pages/README.md).
- Copy rules and the claim register → [05](../05-home-company/copy-guidelines.md).
- Additional industries (medical devices, food & beverage) → Phase 4.

## Why this page exists

Every other page on the site commits to one track (D-403). This one deliberately does not, because
the argument it makes requires both halves:

> A pharmaceutical manufacturer has systems that must be validated, processes that should be
> automated, and HR records that are themselves GxP-relevant — training, qualifications, who was on
> shift. Vexalid is one supplier that can do all three, and holds its own software to the standard it
> audits other people's against.

No competitor holds that position. A generic HRM vendor cannot validate anything. A validation
consultancy does not ship software. Stated once, clearly, on one page, this is the strongest asset
the site has.

It is also the hardest page to write, which is why it is Phase 3 rather than Phase 2 (04 OQ-403) —
written properly later beats written thinly at launch.

## Key decisions taken in this plan

| # | Decision | Rationale |
|---|---|---|
| D-801 | **One industry page at launch, not several.** Pharmaceutical only. | Three thin industry pages are an SEO liability and a credibility problem. One page with real substance is an asset. Add the second only when the first earns its keep. |
| D-802 | **This is the only page that links into both tracks** (04 D-404). | It is the page where the connection is the argument. Everywhere else, split attention costs conversion. |
| D-803 | **It must be written by, or reviewed by, someone who has worked in or with regulated manufacturing** (OQ-012). | A pharma buyer can tell within two paragraphs whether the author has been inside a GMP site. Nothing else on the site is as easy to get wrong in a way the audience detects instantly. |
| D-804 | **Built as a template** (`/industries/[slug]`), even with one instance. | The second industry page should cost a content file. Building it as a one-off guarantees it will be copy-pasted later. |
| D-805 | **The primary CTA is `Talk to us`**, not `Book a demo`. | A visitor arriving on an industry page is more often evaluating a partner than a product. The demo route stays available as a secondary action. |
| D-806 | **No regulatory advice.** The page describes what Vexalid does; it does not tell readers what the regulations require of them. | Publishing regulatory interpretation creates liability and invites correction. Describing your own practice does neither. |
| D-807 | **Terminology precision is absolute here.** GMP, GxP, data integrity, ALCOA+, Part 11, Annex 11, GAMP 5 — used correctly or not at all. | This page's entire job is demonstrating that Vexalid belongs in this industry. One loose usage undoes it. |

## Page specification — `/industries/pharmaceutical`

| # | Section | Contents |
|---|---|---|
| 1 | Breadcrumbs | Home / Industries / Pharmaceutical |
| 2 | Hero | Eyebrow: "Pharmaceutical". H1 — *TODO-COPY*, e.g. *"Built for sites that have to prove it"*. Subhead naming the audience: QA, IT and operations at pharmaceutical manufacturers. CTA `Talk to us` (D-805). |
| 3 | **What makes this different** | Three or four paragraphs on why software and systems in a GMP environment carry obligations most vendors have never had to meet: documented behaviour, controlled change, evidence that survives an inspection. Written from experience (D-803). No advice on what the regulations require (D-806). |
| 4 | Where we help | Three columns mapping to the three service lines: validating the systems you already run · automating processes without creating unexplainable behaviour · building software designed to be validated. Each links to its service page. |
| 5 | Our own software | The HRM and StratumOne, framed for this audience: HR records that are GxP-relevant (training, qualifications, shift attendance), and — under reading (a) of OQ-003 — time integrity across systems. Links to both products. |
| 6 | Validation-ready, precisely | The D-703 distinction again, in this audience's own terms. Repeating it here is correct, not redundant: this is the page where the reader most needs it, and where getting it right earns the most credit. |
| 7 | How we work | A compressed Discover → Design → Deliver → Improve, framed around a regulated engagement. → `/approach`. |
| 8 | FAQ | Do you work on-site? Which frameworks? Can you work alongside our existing validation team? Can your software be self-hosted in our environment? What happens at inspection? |
| 9 | CTA | `CtaBand` → `/contact`, with a secondary `Book a demo`. |

## Dependencies

- **Depends on:** 06 (service pages to link into), 07 (product pages), 05 (copy rules and the
  claim register), 04 (routes).
- **Blocked on:** OQ-012 (the reviewer — **blocking**), OQ-003 (StratumOne's framing for section 5),
  OQ-006 (whether any client work can be referenced even in general terms).
- **Depended on by:** 11 (this page targets the highest-intent commercial terms on the site).

## Tasks

| ID | Task | Done when |
|---|---|---|
| T-801 | Confirm the author/reviewer with regulated-manufacturing experience (D-803). | **Blocking.** Named |
| T-802 | Build the `/industries/[slug]` template (D-804). | Renders from `content/industries/*.mdx` |
| T-803 | Write `content/industries/pharmaceutical.mdx`. | Practitioner-reviewed; `reviewedBy` set |
| T-804 | Cross-track links into all three services and both products (D-802). | In body copy, not just the footer |
| T-805 | Terminology audit against D-807. | Every regulatory term checked, one by one |
| T-806 | FAQ content from real buyer objections. | FAQPage JSON-LD |
| T-807 | Add `Industries` to the header nav and the Services mega-menu panel. | 04 T-403 updated |

## Open questions

| ID | Question | Why it matters | Proposed default |
|---|---|---|---|
| OQ-801 | Who writes or reviews this page? (= OQ-012) | **Blocking** (D-803). The page cannot be written credibly without them. | Must be named before drafting begins |
| OQ-802 | Is pharmaceutical manufacturing genuinely the main market, or are there others (medical devices, food, cosmetics, clinical labs)? | Decides whether "pharmaceutical" is the right page or too narrow. | Pharmaceutical, per the live site's GMP language. Confirm |
| OQ-803 | Is the market Sri Lankan pharmaceutical manufacturing, or export/international? (= OQ-004) | A domestic market is small and relationship-driven — which would make this page less important than direct outreach, and would change how the whole site should be judged. | Confirm before investing heavily in this page |
| OQ-804 | Can any prior engagement be described, even anonymously ("a mid-size manufacturer preparing for inspection")? | An anonymous but real example is worth more than any amount of capability description. | Only with consent; never fabricated |
| OQ-805 | Should a second industry page follow soon, or is one enough? | D-801 says one until it proves itself. | One. Revisit after two quarters of search data |
