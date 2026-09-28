# 07 — Products Pages — Page Specifications

`TODO-COPY` marks placeholder text. All claims subject to the register in
[05 copy-guidelines.md](../05-home-company/copy-guidelines.md).

---

## `/products` — overview

**Job:** answer the unasked question — *why does a validation consultancy sell software?* — and then
route to the two products as peers (D-706, D-707).

| # | Section | Component | Contents |
|---|---|---|---|
| 1 | Hero | `Hero` (compact) | Eyebrow: "Products". H1: *TODO-COPY*, e.g. *"Software built by people who validate systems for a living"*. Subhead. CTA `Book a demo`; secondary `Talk to us`. |
| 2 | **Why we build software** | `Section` | The D-706 answer, in three or four sentences. Roughly: *we spent years validating other people's systems, saw the same failures repeatedly — no audit trail, no access control, assumptions baked in that no site could configure around — and built our own to the standard we ask of everyone else.* **If this paragraph is weak, the products read as side projects.** Links to `/services/custom-software`. |
| 3 | The two products | `FeatureGrid` (2 cols, equal weight) | **HRM** and **StratumOne**, given identical visual treatment (D-707). Each: name, tagline, three bullets, link. |
| 4 | Built to the same standard | `FeatureGrid` (4 cols) | The properties both products share: role-based access, audit trail, configurability, self-hosting available (D-709). Sourced from `dev-plan/01-roles-permissions-auth` and `dev-plan/03-admin-settings`. |
| 5 | Validation-ready — what that means | `Section tone="subtle"` | The D-703 distinction, explained plainly: *software is not validated; an installation is validated for its intended use. What we ship is designed so that validation is straightforward rather than a fight — documented behaviour, controlled change, evidence you can hand an auditor.* **Being precise here is itself the sales argument.** Links to the CSV page. |
| 6 | FAQ | `Faq` | Can we buy one without the other? Can we self-host? Do you help us validate it? What happens to our data if we leave? |
| 7 | CTA | `CtaBand` | → `/demo` |

---

## `/products/hrm` — HRM overview ⚠️ *name pending OQ-001*

**Job:** present the HR suite, led by the regulated-manufacturer angle (D-702), then route to modules.

| # | Section | Contents |
|---|---|---|
| 1 | Breadcrumbs | Home / Products / HRM |
| 2 | Hero | H1 benefit-led — *TODO-COPY*, e.g. *"HR for sites that have to prove what happened"*. Subhead naming the buyer: HR and operations leads at regulated manufacturers. `Coming soon` badge if OQ-704 requires it. CTA `Book a demo`. |
| 3 | **Why this HRM** | The differentiator, stated early: append-only audit trail, role-based access with scopes, configurable policies rather than baked-in country assumptions, self-hosting available. Every one of these is a procurement question at a GMP site and an afterthought in most HR products. Links to `/industries/pharmaceutical`. |
| 4 | Lifecycle | Hire → onboard → work → pay → develop → leave, each stage naming its module. `Coming soon` badges where applicable (D-704). |
| 5 | Module grid | All eight modules from the adapter: icon, name, tagline, status, link. |
| 6 | Payroll, stated honestly | A short block on the configurable-payroll position: **no built-in country rules** — you configure your own rules, deductions and contribution bands. This is simultaneously the differentiator and the limitation, and stating both is more persuasive than stating either. |
| 7 | Deployment | Cloud or self-hosted, with the trade-offs (D-709). |
| 8 | FAQ | Implementation time, data migration, employee limits, where data lives, export, validation support. |
| 9 | Cross-track | Short: Vexalid also validates systems for a living, → CSV page (D-705). |
| 10 | CTA | `CtaBand` → `/demo` |

---

## `/products/hrm/[module]` — module template (D-701)

Driven entirely by `content/products/hrm/modules/<slug>.mdx`.

| # | Section | Source | Contents |
|---|---|---|---|
| 1 | Breadcrumbs | route | Home / Products / HRM / \<Module\> |
| 2 | Hero | `name`, `tagline`, `status` | H1 benefit-led. `Coming soon` badge when `status !== 'available'`, with the honest line from D-504. CTA `See it with your own data` — or `Talk to us about the roadmap` when unbuilt. |
| 3 | Capabilities | `capabilities[]` | 3–8 items as a `FeatureGrid`. **The section an evaluator scans.** Future tense where unbuilt. |
| 4 | Deep dive | MDX body | Walkthrough, diagram or worked scenario, in `Prose`. |
| 5 | FAQ | `faqs[]` | Module-specific. FAQPage JSON-LD. |
| 6 | Related modules | `relatedModules[]` | 2–3 sibling cards. |
| 7 | CTA | — | `CtaBand` → `/demo` |

**Per-module emphasis**, from the HRM plan:

| Module | Lead with | Regulated angle | Plan reference |
|---|---|---|---|
| Employee Records | One record per person; documents, structure, history | Training records and qualifications are GMP-relevant — worth naming | `dev-plan/02-employee-management` |
| Attendance | Device clock-in, shifts, schedules | Who was on shift, provably — a real traceability question | `dev-plan/04-attendance-tracking` |
| Leave Management | Policy configuration, correct balances, approval chains | Approval chains with an audit trail | `dev-plan/06-leave-management` |
| Payroll | **Configurable, not country-locked** | Change control over payroll rules | `dev-plan/07-payroll` |
| Self-Service | Employees answer their own questions | Access scoped to self, enforced server-side | `dev-plan/08-employee-self-service` |
| Performance | Reviews and goals that get completed | Competency tracking | `dev-plan/09-performance-management` |
| Recruitment | Pipeline through to a created employee record | No re-keying, so no transcription errors | `dev-plan/10-recruitment-onboarding` |
| Reports | The questions a board asks | Evidence without an export | `dev-plan/11-reports-analytics` |

The regulated angle is a **column, not a whole page** — each module page is still primarily about the
HR function. Overdoing the compliance framing would make the product unsellable to anyone outside
pharma, which is not the intent of G-04.

---

## `/products/stratumone` ⚠️ **blocked on OQ-003**

Two specifications, one per reading. **Do not write this page until the reading is confirmed.**

### If reading (a) — time integrity for regulated systems

This is the stronger position, and the page should be structured as a **service-adjacent product**.

| # | Section | Contents |
|---|---|---|
| 1 | Hero | H1: *TODO-COPY*, e.g. *"One clock. Every system. Provably."* Subhead: a centralised, traceable time source so audit trails across your systems actually reconcile. CTA `Book a demo`. |
| 2 | The problem | Why this exists: audit trails from an MES, a LIMS and an ERP are only reconcilable if the clocks agree. Drift is invisible until an investigation needs a timeline — and then it is a finding. **This section does most of the selling**, because many buyers have not framed the problem this way. |
| 3 | What it does | Capabilities — centralised synchronisation, traceability to a reference source, drift monitoring, alerting, evidence and reporting. *Pending confirmation of actual scope.* |
| 4 | Why it matters for GxP | The data-integrity argument, in the language of the audience. **Practitioner review required.** |
| 5 | How it deploys | On-premises, self-hosted, network topology, what it touches (D-709). Regulated IT will ask before anything else. |
| 6 | Cross-track | The tightest link on the site: the people who built this validate systems for a living → CSV page. |
| 7 | FAQ | What hardware, what it integrates with, how it is validated, what evidence it produces. |
| 8 | CTA | `CtaBand` → `/demo` |

### If reading (b) — centralised employee time-clock management

| # | Section | Contents |
|---|---|---|
| 1 | Hero | Multi-site attendance terminal consolidation. |
| 2 | The problem | Attendance devices across sites, no single view, manual reconciliation. |
| 3 | What it does | Device management, centralised punch collection, sync, reporting. |
| 4 | **Boundary section — mandatory** (OQ-703) | Explicitly: what StratumOne does that the HRM's attendance module does not, and when a buyer needs one, the other, or both. **Without this section the two products appear to compete and both lose credibility.** |
| 5 | Deployment | — |
| 6 | Cross-track | → HRM attendance module, → Custom Software. |
| 7 | CTA | `CtaBand` → `/demo` |

---

## Imagery

| Page | Asset | Status |
|---|---|---|
| `/products` | Two product marks, equal weight (D-707) | To design |
| `/products/hrm` | UI composition — abstract until the interface is presentable (D-708) | Blocked on OQ-002/OQ-008 |
| Each module page | One screenshot or diagram | Blocked on OQ-002 |
| `/products/stratumone` | Topology or architecture diagram — likely more useful than a UI shot under reading (a) | Blocked on OQ-003 |

Diagrams in SVG, theme-aware via `currentColor` and tokens. Real `alt` throughout. No stock
photography.
