# 06 — Services Pages

**Priority:** Must-have · **Build order:** 6 of 13 · **Status:** planned

## Purpose

Build the services track: the `/services` overview and three service pages. These carry the business
Vexalid runs on today — validation work for regulated organisations — and they are what the current
one-page site says in compressed form. Turning three paragraphs into three real pages is the single
biggest content gain of this project.

## Scope

**In scope**

- `/services` — the overview.
- `/services/computerized-system-validation` — the flagship page.
- `/services/ai-automation`.
- `/services/custom-software`.
- The service page template and the content schema behind it.
- Copy structure for a consulting sale, which differs materially from a product sale.

**Out of scope (owned elsewhere)**

- `/approach` — the method itself → [05 Home & Company](../05-home-company/README.md) D-511.
- `/industries/pharmaceutical` → [08 Industries](../08-industries/README.md).
- The enquiry form → [10 Lead Capture](../10-lead-capture/README.md).
- Copy rules and the claim register → [05](../05-home-company/copy-guidelines.md) — **this segment
  inherits them, and the regulatory-terminology rule applies here more than anywhere.**

## Files

| File | Contents |
|---|---|
| [page-specs.md](./page-specs.md) | Section-by-section spec for the overview and all three service pages |

## How a services page differs from a product page

This is worth stating explicitly, because building them the same way is the obvious mistake:

| | Product page | Service page |
|---|---|---|
| The reader wants | To see the thing | To believe you can do the thing |
| Proof is | Screenshots, feature lists | Method, deliverables, credentials, prior work |
| The ask | "Show me" — low commitment | "Let's talk" — higher commitment, longer cycle |
| Decisive section | Capabilities grid | **What you receive**, and **how you work** |
| Fatal weakness | Overpromising features | Vagueness — a service described abstractly reads as "we'll figure it out" |

A pharma QA manager evaluating a validation partner is asking one question: *do these people know
what a validation deliverable actually looks like?* The pages must answer it concretely.

## Key decisions taken in this plan

| # | Decision | Rationale |
|---|---|---|
| D-601 | **One template for all three service pages**, driven by `content/services/*.mdx`. | Three hand-built pages drift. One template cannot. |
| D-602 | **Every service page names its deliverables explicitly.** Not "we help with validation" but "you receive a validation plan, a risk assessment, executed IQ/OQ/PQ protocols with evidence, a traceability matrix, and a summary report." | Concreteness is the entire persuasion mechanism for a consulting sale. It is also how a reader distinguishes a real practice from a website. |
| D-603 | **The CSV page is the flagship** and gets more depth than the other two: the longest page on the site, and the one to write first. | It is the current revenue, the hardest to fake, and the highest-intent search term Vexalid can realistically own. |
| D-604 | **Each service page carries a "who this is for" section** naming the role and the situation. | A page that could be about anyone convinces no one. It also lets the wrong visitor leave quickly, which is a feature. |
| D-605 | **Every service page links to at least one product**, and explains why that is relevant. | The cross-track link is the positioning (04 navigation.md). A services visitor learning that Vexalid ships its own validated software is proof of capability, not a distraction. |
| D-606 | **No pricing on service pages.** One line: scoped per engagement, → `/contact`. | Consulting pricing on a web page is either meaningless or wrong. But the question must still be acknowledged, not ignored. |
| D-607 | **Regulatory terminology is reviewed by a practitioner before publication** (OQ-012, 05 copy rule 8). | Misusing "validated", "Part 11" or "GAMP" in front of a QA audience is disqualifying in a way no amount of good design recovers. |
| D-608 | **An FAQ on every service page**, answering the objections a buyer actually raises. | For consulting these are the real conversion blockers: how long, how much involvement from us, what if we fail an audit, do you work remotely. |
| D-609 | **`/services/ai-automation` must not overclaim.** It is workflow automation and dashboards with AI assistance — describe that, not "AI transformation". | The audience is regulated industry, where unexplainable automation is a liability, not a selling point. Restraint here is a competitive advantage. |

## Content schema

Service pages are driven by `content/services/<slug>.mdx` — schema defined in
[03 Content Layer](../03-content-layer/content-model.md). Key fields:

| Field | Purpose |
|---|---|
| `name`, `number` | "Computerized System Validation", `"01"` — keeps the existing site's numbering |
| `tagline` | One line, used in nav, cards and the mega-menu |
| `forWhom[]` | Role + situation pairs (D-604) |
| `capabilities[]` | What the service covers |
| `deliverables[]` | **What the client receives** (D-602) — the decisive field |
| `process[]` | Optional service-specific steps, otherwise falls back to `/approach` |
| `faqs[]` | Objection handling (D-608) |
| `relatedProducts[]` | The cross-track link (D-605) |
| `reviewedBy`, `reviewedAt` | Who signed off the regulatory language, and when (D-607) |

`reviewedBy` is not decorative: a production build **fails** if a service page has no reviewer
recorded. That is the mechanism that stops unreviewed compliance language reaching the site.

## Dependencies

- **Depends on:** 02 (sections), 03 (service content schema), 04 (routes), 05 (copy rules, `/approach`).
- **Blocked on:** OQ-006 (nameable clients), OQ-007 (certifications), OQ-012 (practitioner reviewer —
  **blocking for the CSV page**).
- **Depended on by:** 08 (the industry page draws on these), 11 (keyword map).

## Tasks

| ID | Task | Done when |
|---|---|---|
| T-601 | Interview whoever runs these engagements: real deliverables, real timelines, real objections. | Notes captured; this is the raw material for all three pages |
| T-602 | Write `content/services/*.mdx` for all three. | Frontmatter validates; `reviewedBy` set |
| T-603 | Build the `/services/[service]` template + `generateStaticParams`. | All three pages render |
| T-604 | Build `/services` overview. | — |
| T-605 | Write the CSV page in full depth (D-603). | Practitioner-reviewed |
| T-606 | Write the AI Automation page per D-609. | No overclaiming |
| T-607 | Write the Custom Software page, using the HRM and StratumOne as the proof. | Cross-links to both products |
| T-608 | Build the CI check failing a build when `reviewedBy` is missing. | Verified by a deliberate failing commit |
| T-609 | FAQ content for each page from T-601's objection list. | Emits FAQPage JSON-LD |
| T-610 | Diagrams: the validation lifecycle, the automation flow. SVG, theme-aware. | — |

## Open questions

| ID | Question | Why it matters | Proposed default |
|---|---|---|---|
| OQ-601 | Who is the practitioner reviewer for regulatory language? | **Blocking for the CSV page** (D-607). | Must be named before the page is written, not after |
| OQ-602 | What does a typical CSV engagement actually produce, and over what timeline? | D-602 makes this the decisive content. Without it the page is generic. | Blocked on T-601 — cannot be invented |
| OQ-603 | Which regulatory frameworks does Vexalid actually work to — GAMP 5, 21 CFR Part 11, EU Annex 11, PIC/S? | Naming them correctly is a credibility signal; naming them wrongly is fatal. | Name only what is genuinely practised |
| OQ-604 | Does Vexalid work on-site, remotely, or both — and in which countries? | A pharma buyer needs to know before enquiring. | State plainly on the CSV page |
| OQ-605 | Is "AI Automation" a real delivered service with prior work, or an emerging offer? | Changes how confidently the page can be written, and whether it should launch at all. | If it has no delivered work behind it, describe it as an offer rather than a track record |
| OQ-606 | Should the services track have its own lead magnet — e.g. a validation planning checklist? | Higher-intent than a general newsletter, and cheap to produce from existing engagement material. | Strong Phase 3 candidate; likely the best-converting asset on the site |
