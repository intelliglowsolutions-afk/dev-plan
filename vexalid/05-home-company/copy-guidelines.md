# 05 — Copy Guidelines & Claim Register

These rules govern **every page on the site**, not just the ones in segment 05.

## Voice

The existing site already has a voice, and it is a good one: *"Precision in the details. Clarity in
the outcome."* Plain, specific, unhurried — the way a competent engineer explains something.

| Do | Don't |
|---|---|
| "Risk-based validation with audit-ready documentation" | "Best-in-class compliance solutions" |
| "Approve leave in two clicks" | "Revolutionise your HR workflow" |
| Name the thing: "validation plan", "IQ/OQ/PQ", "payslip", "approval chain" | Abstract nouns: "solutions", "capabilities", "synergies" |
| Second person: "your systems", "your team" | Third person: "organisations can" |
| Short sentences. One idea each. | Three subordinate clauses |
| Concrete numbers where they are real | Invented numbers ("reduce validation effort by 40%") |

**Specific rules**

1. **No superlatives without evidence.** "The best" is unverifiable and reads as filler.
2. **No AI claims beyond what is true.** "AI Automation" is a named service line; that does not make
   the HRM an AI product. Do not let the service's language leak onto the product pages.
3. **No urgency manufacturing.** No countdowns, no artificial scarcity. Both a validation engagement
   and an HRM purchase are months-long evaluations; pressure tactics damage credibility.
4. **Sentence case for headings.** Title Case reads dated and scans worse.
5. **Active voice.** "Managers approve requests", not "requests are approved by managers".
6. **Avoid "simply", "just", "easy".** Telling a QA manager that validation is easy is the fastest
   possible way to lose them.
7. **Every number has a source.** If it cannot be sourced, cut it.
8. **Regulatory terms are used precisely or not at all.** GMP, GxP, 21 CFR Part 11, EU Annex 11,
   GAMP 5, IQ/OQ/PQ, data integrity, ALCOA+ — each means something specific to the reader. Using one
   loosely marks Vexalid as an outsider to the exact audience it is trying to win. **Every use must
   be reviewed by someone who does this work** (OQ-012).
9. **One English variant throughout** — British proposed (OQ-505).

## Heading rules

- `h1`: benefit-led, ≤ 60 characters, one per page (D-506).
- `<title>`: carries the **search term** — "Computerized System Validation Services | Vexalid" —
  while the `h1` carries the benefit. Deliberately different.
- `h2`s should carry the argument on their own for a scanning reader.
- Never skip a heading level for visual reasons; use tokens to change size.

## CTA copy

Declared once in `content/site/cta.ts` so they stay consistent site-wide.

| Context | Label | Never |
|---|---|---|
| Services, site-wide | `Talk to us` | "Get a quote" — too transactional for a consulting sale |
| Services page | `Discuss your project` | "Click here" |
| Products, site-wide | `Book a demo` | "Get started" / "Try free" — **no self-serve signup exists** (G-08) |
| Product module page | `See it with your own data` | "Learn more" |
| Pricing | `Get a quote` | "Buy now" |
| Newsletter | `Subscribe` | "Join the revolution" |
| Gated guide | `Get the guide` | "Download now!!" |

**"Get started" and "Start free trial" are banned site-wide.** There is no self-serve flow to start.

---

## The claim register (D-504, T-501)

**Nothing may be claimed anywhere on the site that is not registered here with an accurate status.**

### Services

| Service | Page | May claim | Evidence | Reviewer |
|---|---|---|---|---|
| Computerized System Validation | `/services/computerized-system-validation` | Risk-based validation; validation planning; risk assessment and testing; GMP-ready documentation | Already claimed on the live site | **Required** (OQ-012) |
| AI Automation | `/services/ai-automation` | Workflow automation; AI-assisted operations; data-driven dashboards | Already claimed on the live site | Required |
| Custom Software | `/services/custom-software` | Business applications; system integrations; validation-ready design | Already claimed on the live site; the HRM and StratumOne are the proof | Required |

**Service claims requiring care**

| Claim | Rule |
|---|---|
| "GAMP 5 compliant" / "21 CFR Part 11 compliant" | **Only if accurate and reviewed.** These are specific frameworks. Prefer "aligned with GAMP 5 principles" if that is what is true — and only if a practitioner confirms it. |
| "We are certified in X" | Only with a certificate (OQ-007). |
| "Used by N pharmaceutical manufacturers" | Only with N real clients. |
| "We guarantee audit success" | **Never.** Nobody can guarantee a regulator's finding. |
| Years of experience, team size, project counts | Only if true and checkable. |
| Naming any client | Only with written consent (OQ-006). |

### Products

Status values match the `status` frontmatter: `available` · `in-development` · `planned`.

> **State of the underlying plan, as of 2026-09-15:** only
> `dev-plan/01-roles-permissions-auth/` has written planning documents. Folders 02–11 exist but are
> empty. Statuses below are therefore **provisional and default to `planned`** and must be confirmed
> against the real build state (T-501, OQ-008) before any product page ships.

| Module | Page | May claim | Status | Plan reference |
|---|---|---|---|---|
| Roles, permissions & auth | `/products/hrm` (cross-cutting) | Role-based access; permission scopes (all/department/self); **append-only audit log**; immediate revocation on role change | planned — **documented in detail** | `dev-plan/01-roles-permissions-auth` |
| Employee Management | `/products/hrm/employee-records` | One employee record, departments, reporting lines, documents | planned | `dev-plan/02-employee-management` |
| Admin & Settings | `/products/hrm` (cross-cutting) | Company profile, system configuration, device registration | planned | `dev-plan/03-admin-settings` |
| Attendance Tracking | `/products/hrm/attendance` | Device clock-in, shifts, schedules | planned | `dev-plan/04-attendance-tracking` |
| Notifications | `/products/hrm` (cross-cutting) | Email notifications on approvals and requests | planned | `dev-plan/05-notifications` |
| Leave Management | `/products/hrm/leave-management` | Leave types, balances, approval chains | planned | `dev-plan/06-leave-management` |
| Payroll | `/products/hrm/payroll` | **Generic and configurable — explicitly no built-in country rules.** A real differentiator *and* a real limitation; state both | planned | `dev-plan/07-payroll` |
| Self-Service | `/products/hrm/employee-self-service` | Employees view records, request leave, download payslips | planned | `dev-plan/08-employee-self-service` |
| Performance | `/products/hrm/performance` | Reviews, goals | planned (nice-to-have) | `dev-plan/09-performance-management` |
| Recruitment & Onboarding | `/products/hrm/recruitment` | Pipeline, onboarding | planned (nice-to-have) | `dev-plan/10-recruitment-onboarding` |
| Reports & Analytics | `/products/hrm/reports` | HR reporting and analytics | planned (should-have) | `dev-plan/11-reports-analytics` |
| **StratumOne** | `/products/stratumone` | ⚠️ **Unknown — OQ-003 is unanswered.** Nothing may be written for this page until the product's actual scope is confirmed | unknown | none |

**Product claims requiring care**

| Claim | Rule |
|---|---|
| "Payroll for Sri Lanka" (or any country) | **Never.** `dev-plan/07` is explicit that payroll is generic with no built-in country rules. Say: "configure your own rules, deductions and contribution bands". |
| "21 CFR Part 11 compliant HRM" | **Never, as stated.** Part 11 compliance is a property of a validated implementation, not of shipped software. Say what is true: "audit trail, access control and electronic records designed to support Part 11 validation" — and only after review. |
| "Validated" | **Never of the product itself.** Software is not validated; an *installation* is validated for its intended use. Say "validation-ready" or "designed to be validated", and explain the difference. **Getting this wrong in front of a QA audience is disqualifying.** |
| "Free trial" / "Get started free" | Not in v1 — no self-serve signup exists (G-08). |
| "Trusted by X companies" | Only with X real, consenting companies. |
| "Integrates with <system>" | Only for integrations that exist and are supported. |
| "Bank-level security" | Meaningless. Describe the actual controls. |
| Uptime percentages | Only with monitoring history to back them. |

### Handling `planned` and `in-development` modules

A page for an unbuilt module still earns its place — it captures search traffic and shows the
roadmap. It must:

1. Carry a visible `Coming soon` badge in the hero.
2. Use **future tense** for unbuilt capabilities: "will let managers approve in bulk", not "lets".
3. Include one honest line: *"This module is in development. We can walk you through the design and
   timeline."*
4. Change its CTA to `Talk to us about the roadmap` rather than `See it with your own data`.
5. Be excluded from any "what you get today" list.

This costs some conversion. It costs far less than a demo call where a pharma QA manager discovers
the audit trail does not exist yet — at which point the services credibility is damaged too.

---

## Placeholder copy protocol (D-508)

- Placeholders are real, plausible sentences — never lorem ipsum — so layout review is meaningful.
- Each is tagged inline: `{/* TODO-COPY: needs practitioner review of the IQ/OQ/PQ wording, OQ-012 */}`.
- CI greps the production build for `TODO-COPY` and fails (T-512).
- Open items are tracked in `IMPLEMENTATION_LOG.md`.

## Pre-publication checklist (per page)

- [ ] Every claim appears in the register above with a matching status
- [ ] Regulatory terminology reviewed by a practitioner (OQ-012)
- [ ] No fabricated proof — no invented logos, testimonials or statistics (D-503)
- [ ] One `h1`, benefit-led, ≤ 60 characters
- [ ] `<title>` ≤ 70 characters, carries the search term
- [ ] Meta description 70–160 characters
- [ ] Exactly one primary CTA, matching the page's track (D-505)
- [ ] Page ends with a track-appropriate `CtaBand` (D-408)
- [ ] Meets the internal-linking rule for its page type, **including the cross-track link**
- [ ] Every image has meaningful `alt`, or explicit `alt=""`
- [ ] Reads correctly at 360px with no horizontal scroll
- [ ] No `TODO-COPY` remaining
- [ ] Spell-checked in the chosen English variant
- [ ] Read aloud once — the test most copy fails
