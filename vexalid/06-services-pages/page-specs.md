# 06 — Services Pages — Page Specifications

Lines marked **[existing]** come from the current vexalid.com and should be kept.
`TODO-COPY` marks placeholder text. **Every regulatory term needs practitioner review** (D-607).

---

## `/services` — overview

**Job:** show that Vexalid's three service lines are one coherent practice, then route to the right
one.

| # | Section | Component | Contents |
|---|---|---|---|
| 1 | Hero | `Hero` (compact) | Eyebrow: "Services". H1: *"Systems you can prove are working"* — *TODO-COPY*. Subhead naming the audience: regulated organisations, principally pharmaceutical manufacturing. CTA `Talk to us`. |
| 2 | What we solve | `Section tone="subtle"` | **[existing]** *"Rigour for compliance. Intelligence for growth."* Technology should advance an organisation without creating risk; validation discipline plus modern engineering. |
| 3 | The three services | `FeatureGrid` (3 cols) | **[existing]** numbering 01/02/03 retained. Each card: number, name, tagline, three capability bullets, link. |
| 4 | How we work | `Steps` (4) | **[existing]** Discover → Design → Deliver → Improve, compressed, → `/approach`. |
| 5 | Who we work with | `Section` | Regulated organisations — the roles and situations. Links to `/industries/pharmaceutical`. |
| 6 | Cross-track | `Section tone="subtle"` | **"We build our own software too."** Short: the HRM and StratumOne are built to the same validation discipline, which is why the service work and the products reinforce each other. Links to `/products`. (D-605) |
| 7 | CTA | `CtaBand` | → `/contact` |

---

## `/services/[service]` — the template (D-601)

One template, three pages, driven by `content/services/<slug>.mdx`.

| # | Section | Source | Contents |
|---|---|---|---|
| 1 | Breadcrumbs | route | Home / Services / \<Service\> |
| 2 | Hero | `number`, `name`, `tagline` | Eyebrow shows the number (01/02/03). H1 benefit-led. Subhead. CTA `Talk to us`; secondary `See our approach`. |
| 3 | Who this is for | `forWhom[]` | Two or three role + situation pairs: *"A QA manager preparing for an inspection with undocumented legacy systems."* **The section that makes a reader feel recognised** (D-604). |
| 4 | What we do | `capabilities[]` | 3–6 items as a `FeatureGrid`. The substance of the service. |
| 5 | **What you receive** | `deliverables[]` | An explicit list of artefacts, not benefits. **The decisive section for a consulting buyer** (D-602) — it is the difference between a practice and a website. |
| 6 | How it runs | `process[]` or `/approach` | Service-specific steps if defined; otherwise a compressed Discover → Design → Deliver → Improve with a link onward. |
| 7 | Deep dive | MDX body | Free-form: a worked example, a diagram, an explanation of method. |
| 8 | FAQ | `faqs[]` | Objection handling (D-608). Emits FAQPage JSON-LD. |
| 9 | Related | `relatedProducts[]` + sibling services | One product, one or two sibling services (D-605). |
| 10 | CTA | — | `CtaBand` → `/contact`, with the scoping line from D-606. |

---

## `/services/computerized-system-validation` — the flagship (D-603)

The longest page on the site, and the first to write.

**Hero.** Eyebrow `01`. H1 — benefit-led, *TODO-COPY*, e.g. *"Prove your systems do what you say they
do"*. `<title>`: `Computerized System Validation Services | Vexalid`.

**Who this is for** — *TODO-COPY, from T-601:*
- A QA manager whose legacy systems have no current validation documentation.
- An IT lead deploying a new MES, LIMS or ERP into a GMP environment.
- A site preparing for an inspection with gaps it already knows about.

**What we do** — from the live site, expanded:
- Validation planning
- Risk assessment and testing
- GMP-ready documentation

Expand each into what it actually involves. **Practitioner review required** (D-607).

**What you receive** — the decisive section (D-602). Blocked on OQ-602; must be real. The shape:
validation plan · risk assessment · test protocols and executed evidence · traceability matrix ·
deviation handling · summary report · handover pack.

**How it runs.** Where a validation engagement differs from the generic four steps — say so here.

**Deep dive.** A validation lifecycle diagram (SVG, theme-aware) and a worked example at the level of
detail a QA manager recognises as real.

**FAQ** — *TODO-COPY, from T-601:*
- How long does a typical engagement take?
- How much of our team's time does this need?
- Do you work on-site or remotely? (OQ-604)
- What if we're already mid-way through an implementation?
- What happens if a regulator raises a finding afterwards?
- Which frameworks do you work to? (OQ-603)

**Related:** → Custom Software · → StratumOne *(if OQ-003 confirms the data-integrity reading, this
is the most natural cross-link on the entire site)* · → `/industries/pharmaceutical`.

**Language warnings for this page specifically**
- Never "we guarantee audit success".
- Never "we make you compliant" — compliance is the client's, achieved with support.
- "Validated" describes an installation for its intended use, never software in the abstract.
- Name frameworks only where genuinely practised (OQ-603).

---

## `/services/ai-automation` (D-609)

**Hero.** Eyebrow `02`. H1 — *TODO-COPY*, something concrete like *"Take the repetition out of
regulated operations"*. Not "AI transformation".

**Who this is for**
- An operations lead whose team re-keys the same data between systems.
- A manager with no single view of what is happening across sites.

**What we do** — from the live site:
- Workflow automation
- AI-assisted operations
- Data-driven dashboards

**The restraint rule.** In regulated industry, automation that cannot be explained is a liability.
This page should say plainly: automation is scoped, documented, and reviewable; where AI is used, its
role is assistive and its output is checkable. **That restraint is the differentiator** — anyone can
claim AI; very few can claim AI a QA function will accept.

**What you receive.** Process map · automation specification · the built automation with
documentation · a handover and support plan.

**FAQ.** Does this replace staff? What happens when it gets something wrong? How is it validated?
Does it run on our infrastructure? — the last two matter most to this audience.

**Related:** → Custom Software · → HRM (reporting and workflow).

**Open risk:** if this service has no delivered work behind it (OQ-605), the page must describe an
offer rather than imply a track record — or be held back until it does.

---

## `/services/custom-software`

**Hero.** Eyebrow `03`. H1 — *TODO-COPY*, e.g. *"Software built to be validated, not retrofitted"*.

**Who this is for**
- A team whose process does not fit any off-the-shelf product.
- An organisation that needs an integration nobody sells.
- A regulated site that has been told its chosen tool cannot be validated.

**What we do** — from the live site:
- Business applications
- System integrations
- **Validation-ready design**

The third item is the differentiator and should lead, not trail: most software firms build first and
discover the validation burden later.

**What you receive.** Requirements specification · architecture and design documentation · the
software · test evidence · a validation-ready documentation pack · source and deployment handover.

**The proof section — unique to this page.** The HRM and StratumOne are Vexalid's own custom software,
built to this standard. This is the strongest cross-track link on the site: *"We don't just say we
build validation-ready software — here are two systems we built that way."* Links to both products.

**FAQ.** Who owns the code and the IP? What happens if we want to take it in-house? Can you work with
our existing team? Do you support it afterwards? — ownership is the first question every custom
software buyer has, and most vendor sites avoid it.

**Related:** → both products · → CSV.

---

## Imagery

| Page | Asset |
|---|---|
| `/services` | Three service marks, consistent with the design system's icon treatment |
| CSV | Validation lifecycle diagram — SVG, theme-aware, legible at 360px |
| AI Automation | Before/after process flow diagram |
| Custom Software | Architecture or delivery diagram; optionally a real product screenshot |

No stock photography of laboratories, scientists or people in meetings. Diagrams that show real
method are the imagery this audience responds to — and they are cheaper to produce and easier to keep
accurate than photography.
