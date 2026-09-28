# 07 — Products Pages

**Priority:** Must-have · **Build order:** 7 of 13 · **Status:** planned

## Purpose

Build the products track: the `/products` overview, the HRM product with its eight module pages, and
StratumOne. This is the half of the site that does not exist at all today.

## Scope

**In scope**

- `/products` — the overview, and the argument for why a validation firm ships its own software.
- `/products/hrm` — the HRM overview ⚠️ *name pending OQ-001*.
- `/products/hrm/[module]` — eight module pages from one template.
- `/products/stratumone` ⚠️ *scope pending OQ-003*.
- The product and module content schemas.

**Out of scope (owned elsewhere)**

- `/pricing` → [05 Home & Company](../05-home-company/README.md).
- `/demo` and its form → [05](../05-home-company/README.md) and [10 Lead Capture](../10-lead-capture/README.md).
- `/industries/pharmaceutical` → [08 Industries](../08-industries/README.md).
- Copy rules and the claim register → [05](../05-home-company/copy-guidelines.md). **The product
  claim rules are the strictest on the site and apply in full here.**

## Files

| File | Contents |
|---|---|
| [page-specs.md](./page-specs.md) | Section-by-section spec for the overview, HRM, the module template, and StratumOne |

## Two blocking unknowns

This segment cannot be completed until two questions are answered, and both are cheap to answer:

| | Question | Effect |
|---|---|---|
| **OQ-001** | What is the HRM called? | Every URL, nav label, page title and OG image on this track. `/products/hrm` is a placeholder. StratumOne having a real name strongly implies the HRM should have one too — "Vexalid HRM" next to "StratumOne" reads as one product being taken seriously and the other not. |
| **OQ-003** | What is StratumOne? | The entire page. "Centralized clock system" admits two readings, and they are different products for different buyers. |

### The StratumOne ambiguity, stated plainly

**Reading (a) — a synchronised time source for regulated systems.** Centralised NTP, traceable to a
reference, with monitoring and evidence that every GxP system on a site shares one clock. This is a
genuine data-integrity requirement: audit trails across systems are only reconcilable if the
timestamps agree, and inspectors do look at it. If this is what StratumOne is, it is a **natural
extension of the CSV service line**, sold to the same buyer, and the cross-track link between
`/services/computerized-system-validation` and `/products/stratumone` becomes the tightest
argument on the site.

**Reading (b) — centralised employee time-clock management.** Consolidating attendance terminals
across sites into one system. If this is what it is, StratumOne overlaps the HRM's attendance module
and the site must explain the boundary clearly — otherwise the two products appear to compete with
each other, which confuses buyers and undermines both.

**This plan assumes reading (a)** throughout, flagged wherever it matters. If (b) is correct, the
page specification and the product positioning both need revising, and the HRM/StratumOne boundary
becomes the most important thing this segment has to get right.

## Key decisions taken in this plan

| # | Decision | Rationale |
|---|---|---|
| D-701 | **One template for all eight HRM module pages**, driven by `content/products/hrm/modules/*.mdx`. | Eight near-identical hand-built pages drift within a month. |
| D-702 | **The HRM leads with regulated manufacturers** (G-04): audit trail, access control, configurability, self-hosting. The general HR feature set follows. | It is the only angle no general-purpose HRM competitor can copy, and it is backed by a service line that proves it. |
| D-703 | **"Validation-ready", never "validated"** (05 copy rules). Every mention explains the distinction. | Software is not validated; an installation is validated for its intended use. Getting this wrong in front of a QA audience is disqualifying — and Vexalid sells validation. |
| D-704 | **Every unbuilt module carries a visible `Coming soon` badge and future-tense copy** (D-504). | See the claim register. Overpromising here damages the services credibility too. |
| D-705 | **Every product page links to at least one service page.** | The cross-track link is the positioning: the vendor is also the validation firm. |
| D-706 | **`/products` opens by answering "why does a consultancy sell software?"** | An unasked question that every visitor has. Unanswered, the products read as side projects. Answered well, they read as proof the consultancy knows what it is doing. |
| D-707 | **The two products are presented as peers**, even though the HRM has far more surface area. | Visual hierarchy implying StratumOne is an afterthought will make buyers treat it as one. |
| D-708 | **No product screenshot ships until the UI is presentable** (OQ-002/OQ-008). Abstract UI illustrations until then. | A screenshot of an unfinished interface does more damage than no screenshot. |
| D-709 | **Self-hosting is given prominence** on both product pages. | For regulated buyers, data residency and control of the validated environment are procurement criteria, not preferences. It is also true of Vexalid's own deployment model. |

## Content schema

| Collection | Path | Drives |
|---|---|---|
| Products | `content/products/<slug>.mdx` | `/products/<slug>` |
| HRM modules | `content/products/hrm/modules/<slug>.mdx` | `/products/hrm/<module>` |

Module frontmatter carries `status` and `planRef` — the honesty mechanism (D-704). `planRef` points
at the HRM plan folder that defines the module; `status` drives the badge and suppresses
"available today" phrasing. Schemas in
[03 Content Layer](../03-content-layer/content-model.md).

## Dependencies

- **Depends on:** 02, 03 (schemas), 04 (routes), 05 (claim register), 06 (the service pages each
  product cross-links to).
- **Blocked on:** **OQ-001** (name), **OQ-003** (StratumOne scope), OQ-008 (real build status),
  OQ-002 (screenshots).
- **Depended on by:** 08, 11, 13.

## Tasks

| ID | Task | Done when |
|---|---|---|
| T-701 | Resolve OQ-001 and OQ-003 with the user. | **Blocking.** Answers recorded here |
| T-702 | Confirm the real build status of every HRM module against `dev-plan/` and the codebase (OQ-008). | Claim register statuses agreed (05 T-501) |
| T-703 | Write `content/products/hrm/modules/*.mdx` — all eight. | Frontmatter validates; statuses accurate |
| T-704 | Build `/products/hrm/[module]` template + `generateStaticParams`. | All eight render |
| T-705 | Build `/products/hrm` overview. | Leads with the regulated angle (D-702) |
| T-706 | Build `/products` overview, opening with D-706. | — |
| T-707 | Write and build `/products/stratumone`. | **Blocked on OQ-003** |
| T-708 | Abstract UI illustrations for both products (D-708). | Theme-aware, legible at 360px |
| T-709 | Cross-track links from both products into the services track (D-705). | Present in body copy, not only the footer |
| T-710 | Module FAQs. | Emits FAQPage JSON-LD |

## Open questions

| ID | Question | Why it matters | Proposed default |
|---|---|---|---|
| OQ-701 | The HRM's product name and URL. (= OQ-001) | **Blocking.** Every URL on this track. | `/products/hrm`, "Vexalid HRM", revisited on answer |
| OQ-702 | StratumOne's actual scope. (= OQ-003) | **Blocking.** The whole page. | Reading (a) — time integrity for regulated systems |
| OQ-703 | If reading (b) is correct, where is the boundary between StratumOne and the HRM's attendance module? | Two products that appear to overlap confuse buyers and devalue both. | Must be answered explicitly on both pages, not left implicit |
| OQ-704 | Is the HRM deployable today, in pilot, or in development? (= OQ-008) | Decides `available` vs `Coming soon` across eight pages. | Assume in development; label honestly |
| OQ-705 | Is there a pilot customer — even unnamed — whose usage can be described in general terms? | *"In production at a pharmaceutical manufacturer"* is worth more than any feature list. | Only with consent; never fabricated |
| OQ-706 | Does the HRM genuinely support self-hosting today? | D-709 gives it prominence; it must be true. | Confirm before writing |
| OQ-707 | Do the two products share a platform, a login, or anything technical? | If yes, that is a real selling point worth a section. If no, do not imply it. | Assume independent unless told otherwise |
