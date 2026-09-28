# 05 — Home & Company Pages — Page Specifications

Copy here is **structural placeholder** (D-508). Strings marked `TODO-COPY` must be replaced before
production. Lines quoted from the existing vexalid.com are marked **[existing]** and should be kept
unless there is a reason to change them (D-511, D-512).

---

## `/` — Home

**Job:** establish what Vexalid is in one screen, then route the visitor into the correct track
(D-502). It does not try to close either sale.

| # | Section | Component | Contents |
|---|---|---|---|
| 1 | Hero | `Hero` | Eyebrow: **[existing]** *"Validation · Automation · Innovation"*. H1: **[existing]** *"Confidence built into every system."* Subhead, two lines: what Vexalid does and for whom — regulated organisations, validation, automation, dependable software. **Two CTAs of equal weight, side by side:** `Explore services` → `/services` and `See our products` → `/products`. This is the one page where two equal CTAs are correct — they are a routing choice, not competing asks. |
| 2 | The split | `TwoTrackBand` *(new component)* | Two large cards, side by side on desktop, stacked on mobile. **Left — Services:** "We validate and build systems for regulated organisations", three service names, → `/services`. **Right — Products:** "We build our own software to the same standard", both product names, → `/products`. **This is the most important section on the site.** Everything below it is reinforcement. |
| 3 | What we solve | `Section tone="subtle"` | **[existing]** Heading *"What we solve"*, subhead *"Rigour for compliance. Intelligence for growth."* Two or three short paragraphs: technology should advance an organisation without creating risk; validation discipline plus modern engineering. |
| 4 | Services summary | `FeatureGrid` (3 cols) | The three service lines with their existing numbering (01/02/03), one sentence each, each linking to its page. |
| 5 | Products summary | `ModuleShowcase` (2 rows) | HRM and StratumOne. Each: name, one-line value, three bullets, link. HRM's row leads with the regulated-manufacturer angle (G-04). |
| 6 | Why both | `Section` | The argument that ties the site together: *"The people who validate your systems also build software — and hold it to the same standard."* Links to `/industries/pharmaceutical`. **This paragraph is the site's central claim.** If it is weak, the two-track structure reads as an unfocused company rather than a coherent one. |
| 7 | Principles | `FeatureGrid` (4 cols) | **[existing]** Quality first · Intelligent automation · Tailored solutions · Continuous improvement. Each with one supporting sentence. |
| 8 | Approach | `Steps` (4) | **[existing]** Discover → Design → Deliver → Improve, compressed, → `/approach`. |
| 9 | Closing CTA | `CtaBand` | Both CTAs again, `Talk to us` weighted slightly ahead of `Book a demo` — a visitor who read the whole page and did not pick a track is more likely a services enquiry. |

**Deliberately absent** (D-503): client logo wall, testimonials, statistics band. All return when
real content exists.

**Above the fold at 360px:** logo, eyebrow, H1, subhead, both CTAs.
**LCP element:** the hero media. Preloaded, `priority`, and budgeted explicitly.

---

## `/approach` (D-511)

**Job:** for a consulting buyer, *how you work* is a primary evaluation criterion. This page is the
evidence that CSV at Vexalid is a method, not an improvisation.

| # | Section | Contents |
|---|---|---|
| 1 | Hero | **[existing]** H1 *"A dependable path forward"*, subhead *"From uncertainty to assurance."* CTA `Talk to us`. |
| 2 | The four steps | **[existing]** Discover → Design → Deliver → Improve, as a vertical sequence with generous space. Each step: what happens, what the client does, what they receive at the end of it. **The deliverable per step is what a QA manager is reading for.** |
| 3 | What you get | `FeatureGrid` — the artefacts an engagement produces: validation plan, risk assessment, test protocols and evidence, traceability, final report. *TODO-COPY — must be reviewed by whoever actually runs these engagements (OQ-012).* |
| 4 | Engagement models | Fixed-scope vs retained vs embedded, and roughly when each fits (OQ-405). Answers a question every consulting buyer has and most sites duck. |
| 5 | Closing CTA | `CtaBand` → `/contact` |

---

## `/about`

**Job:** due diligence (path 6). A prospect mid-evaluation checking Vexalid is real and competent.

| # | Section | Contents |
|---|---|---|
| 1 | Hero | H1: *"Why we built Vexalid"* or similar. No stock photography. |
| 2 | Story | Two or three short paragraphs: the problem observed in regulated industry, the decision to combine validation with engineering, the principle followed. **Must be true** (OQ-502) — omit rather than invent. |
| 3 | Principles | **[existing]** The four principles, expanded — each grounded in something Vexalid actually does. |
| 4 | Team | Real names and photos, or **the section is omitted entirely** (D-503). For a consultancy the team *is* the product, so this section is worth more here than on a typical SaaS site — but silhouettes are worse than nothing. |
| 5 | Contact + CTA | Real address, phone, email — the same details already public on the live site. A physical address is a disproportionately strong trust signal for a compliance firm. `CtaBand` → `/contact`. |

---

## `/contact` — services enquiry (D-402)

Reduced chrome: logo only, no mega-menus, minimal footer, no newsletter.

| # | Section | Contents |
|---|---|---|
| 1 | Two-column | **Left (55%):** H1 *"Let's talk about your systems"*. Three bullets on what a first conversation covers. "What happens next" in three steps. Response-time commitment (OQ-504). Real phone, email and address. **Right (45%):** the enquiry form in a raised `Card` (fields in segment 10). |
| 2 | Reassurance | One quiet row: no obligation · your details are never shared · NDA available on request. The NDA line matters to pharma buyers discussing internal systems. |
| 3 | Cross-link + footer | Small line: *"Looking at our software instead? [Book a demo](/demo)"*. Minimal footer. |

**Mobile order:** H1 → one-line value → **form** → bullets → contact details.

---

## `/demo` — product demo request (D-402)

Same reduced chrome, different tone.

| # | Section | Contents |
|---|---|---|
| 1 | Two-column | **Left (60%):** H1 *"See it with your own data"*. What the demo covers, as three bullets. What happens next. Response-time commitment. **Right (40%):** the demo form. |
| 2 | Reassurance | No credit card · no obligation · your data is never shared. |
| 3 | Cross-link + footer | *"Need validation services? [Talk to us](/contact)"*. Minimal footer. |

**Mobile order:** H1 → one-line value → **form** → bullets. Someone who tapped "Book a demo" has
already decided; the form goes above the supporting copy.

---

## `/pricing` (D-509)

Covers the **products** only. Services are quoted per engagement and that is stated plainly.

| # | Section | Contents |
|---|---|---|
| 1 | Hero | H1: *"Pricing that scales with your organisation"*. Subhead stating plainly that pricing is quoted per organisation, and why. |
| 2 | What drives the price | Three cards: **What drives it** (employees, modules, deployment model) · **What's always included** (updates, support, data export, no per-admin fees) · **What costs extra** (implementation, migration, custom integrations, validation support). The honesty is the persuasion. |
| 3 | Deployment options | Cloud (hosted by Vexalid) vs self-hosted on the customer's own infrastructure, with trade-offs. **Genuinely differentiating for regulated buyers**, where data residency and validated-environment control are live procurement concerns. |
| 4 | Services pricing | One short block: *"Validation and custom software are scoped per engagement. [Talk to us](/contact)."* Answers the question rather than leaving a services visitor stranded on a product pricing page. |
| 5 | FAQ + CTA | Minimum term, billing, price on headcount change, trial availability, what happens to data on cancellation. `CtaBand` → `/demo`. |

When G-09 is revisited, section 2 is replaced by a `PricingTable` and the rest stands.

---

## `/thank-you/[type]` — template

`noindex, nofollow`. Fires the matching analytics conversion event (segment 11).

| Type | Heading | Body | Next step |
|---|---|---|---|
| `enquiry` | "Thanks — we've received your enquiry" | Who will reply and when; what they will ask. | → `/approach` |
| `demo` | "Thanks — we've got your request" | What happens next, when, from whom. | → a guide, or `/products/hrm` |
| `newsletter` | "Almost there — confirm your email" | Explains double opt-in; check spam; offer a resend. | → `/resources/blog` |
| `guide` | "Your guide is on its way" | Which email, to which address; a fallback download link. | → `/demo` |

Real pages at real URLs, not client-side state swaps — measurable, linkable, and they survive a refresh.

---

## `/legal/[slug]` — template

| Element | Contents |
|---|---|
| Header | H1 = document title; beneath it *"Last updated: <date>"* from `lastUpdated`. |
| Body | MDX in `Prose`, `--vx-container-narrow`. |
| ToC | Sticky on desktop for documents with more than four `h2`s. |
| Footer | Data-protection contact line on the privacy and DPA pages. |

**Blocked by OQ-011.** Real reviewed text only (D-510). A DPA in particular will be read closely by
pharma and HR buyers — it is a sales document as much as a legal one.

---

## 404 / 500

Specified in [../04-site-structure-navigation/navigation.md](../04-site-structure-navigation/navigation.md).
The 404 must offer **both tracks** — a visitor arriving on a broken link could belong to either.

---

## Imagery requirements

| Page | Asset | Status |
|---|---|---|
| Home hero | Abstract composition conveying precision and systems — **not** stock photos of people in labs or meetings | To design |
| Home § 2 | Two track cards need distinct visual treatment without becoming decorative | To design |
| `/approach` | Four-step process diagram, SVG, theme-aware | To design |
| `/about` | Team photos, or section omitted | Blocked on OQ-502 |
| All pages | OG image, generated from a template | Segment 11 |

Rules: SVG for diagrams, theme-aware via `currentColor` and tokens — never two exported PNGs. AVIF +
WebP with a JPEG fallback for photography. Real `alt` on everything. Nothing above the fold is
lazy-loaded; nothing below it is eager.
