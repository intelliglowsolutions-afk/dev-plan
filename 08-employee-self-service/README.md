# 08 — Employee Self-Service Portal

**Priority:** Must-have · **Build order:** 8 of 11 · **Status:** planned (Step 3)

## Purpose

One place where an employee can see their own information and do the handful of things they need to
do: check a leave balance, request time off, look at a payslip, correct a missed punch, update a
phone number.

## What makes this feature different

Every other feature in this system is used by HR professionals: trained people, at a desk, daily,
who know what "shift window" and "pro-rated entitlement" mean. This one is used by **everyone**, on
a phone, a few times a month, by people who have never been shown how and will not read
instructions.

That inverts most of the design assumptions made so far:

| Elsewhere | Here |
|---|---|
| Dense tables, many columns | One thing per screen |
| Precise HR vocabulary | Plain language |
| Desktop-first | Mobile-first |
| Filters and configuration | Sensible defaults, no configuration |
| Training is assumed | Nothing is explained twice; it works or it doesn't |
| Errors are diagnostic | Errors say what to do next |

It is also the feature most likely to be judged. HR tolerates a rough edge in a tool they chose;
employees form an opinion of "the HR system" from this portal alone.

## The central constraint

**This feature adds almost no domain logic.** Every action it offers already exists in features 02,
04, 06, and 07, scoped to `SELF`. Requesting leave here calls the same endpoint an HR admin calls;
the balance shown is the same balance; the correction request is the same record.

The one genuinely new mechanism is **profile change requests** (D-03), because employees cannot be
allowed to edit their own employee record directly, and there is currently no way for them to ask.

That constraint is deliberate and load-bearing. The moment this feature starts computing its own
figures, there are two answers to "how much leave do I have" and one of them is wrong.

## Scope

**In scope**

- A **dashboard**: what needs the employee's attention, and the few facts they check most.
- **My profile** — view, and request changes to what they may not edit directly.
- **My attendance** — days, punches, and correction requests (the flow specified in 04).
- **My leave** — balances, requests, history, team calendar (the flows specified in 06).
- **My payslips** — published payslips only (07).
- **My documents** — their own documents from 02, and uploading what HR asks for.
- **Company directory** and org chart (02), which is the most-used feature in most portals and the
  one nobody plans for.
- **Notifications** and preferences (05).
- The **employee shell**: navigation, mobile layout, and the plain-language layer over everything
  above.

**Out of scope (owned elsewhere)**

- All domain logic and every calculation (D-01). This feature calls; it does not compute.
- Anything about other employees beyond the directory: no team attendance, no team leave balances,
  no approvals. A manager's approval screens live in their own features and are reached through the
  admin shell (see "Managers", below).
- Onboarding checklists for new hires → [10 Recruitment & Onboarding](../10-recruitment-onboarding/README.md),
  which will surface tasks here once it exists.

## Files

| File | Contents |
|---|---|
| [requirements.md](./requirements.md) | User stories, functional and non-functional requirements, permission keys |
| [data-model.md](./data-model.md) | The one new model, and what this feature reads |
| [api-design.md](./api-design.md) | The `/api/me/*` façade, the dashboard endpoint, change requests |
| [ui-ux.md](./ui-ux.md) | The employee shell, mobile-first screens, plain language, states and copy |

## Key decisions taken in this plan

| # | Decision | Rationale |
|---|---|---|
| D-01 | **No new domain logic.** Every screen reads or calls an existing feature's endpoint at `SELF` scope. If a screen needs something that does not exist, it is built in the owning feature, not here. | Two implementations of "my leave balance" means two answers, and the employee's is the one that generates a complaint. |
| D-02 | **Mobile-first**, built and reviewed at 375 px before any wider layout. | This is the only surface in the system used primarily on phones. Designing it desktop-first and shrinking produces exactly the portal everyone hates. |
| D-03 | Employees **request** profile changes rather than editing directly, except for a small configurable set of fields they own outright. | The employee record feeds payroll and statutory reporting. But making an employee email HR to fix their own phone number is how the data goes stale — so the split is per field, and configurable. |
| D-04 | **Same application, same login, same session** as the admin side. The landing page and navigation differ by permission. | Feature 01 OQ-107 assumed this. A separate portal application doubles the auth surface, the deployment, and the bug count. |
| D-05 | **Plain language, always.** No HR vocabulary, no internal status names, no feature jargon. `MISSING_PUNCH` is "You didn't sign out". | The reader has no training and no glossary. Every untranslated internal term is a support call. |
| D-06 | **Show nothing the employee cannot act on or understand.** No pending-computation notices, no device names, no internal states. | 04's daily grid tells HR that 12 days await computation because HR can do something about it. Telling an employee that is noise that reads as breakage. |
| D-07 | **Progressive disclosure over completeness.** Show the answer; put the derivation one tap away. | "11.5 days left" is the answer. The ledger that proves it belongs behind "How is this worked out?", not on the card. |
| D-08 | **The portal is useless to an employee without a user account** — and possibly without email. That gap is surfaced as a number to HR, not hidden. | 05 OQ-502. A portal that reaches 60% of the workforce is a decision, not an accident, and HR should see the other 40%. |
| D-09 | Sections **degrade gracefully** when their feature is not configured: no leave types yet, no payslips yet, no shift assigned. Each shows an explanation, not an empty box or an error. | This portal will be opened on day one, when almost nothing is configured. |

## Managers

A manager is an employee who also approves things. Two options were considered:

1. **Build a manager area inside the portal** — team attendance, team leave, approvals.
2. **Keep approvals in the features that own them**, reached through the admin shell, and let the
   portal be purely personal.

**v1 takes option 2**, with one exception: pending approvals appear on the portal dashboard as a
prompt with a link ("3 leave requests need you"), because a manager who only ever opens the portal
would otherwise never see them. The approval screens themselves stay in features 06 and 04, where
they are specified, and they are already built to work at phone width.

Option 1 becomes worth revisiting if managers turn out never to open the admin shell at all
(OQ-806).

## Dependencies

- **Depends on:** everything before it — 01 (session, permissions), 02 (profile, documents,
  directory), 03 (formats, timezone), 04 (attendance, corrections), 05 (notifications), 06 (leave),
  07 (payslips).
- **Depended on by:** 10, which will place onboarding tasks here.
- **Blocked by nothing**, but genuinely useful only once 06 and 07 exist — leave and payslips are
  why employees open a portal at all. Building it earlier would produce a shell with two screens.

## Open questions

| ID | Question | Why it matters | Proposed default |
|---|---|---|---|
| OQ-801 | **Which fields may an employee change themselves**, and which require HR approval? | D-03. Personal phone and emergency contacts are uncontroversial; address feeds payroll and statutory reporting; name and bank details are fraud vectors. | Self-service: personal phone, personal email, emergency contacts, photo. Request required: name, address, marital status, national ID. Never: employee code, job, salary, bank details. |
| OQ-802 | **Do employees have smartphones and network access at work?** Is there a shared kiosk instead? | D-02 assumes phones. A factory floor with no phones allowed needs a kiosk mode — a shared device, short sessions, no sensitive data on screen — which is a different build. | Personal phones, on their own time. |
| OQ-803 | Should employees be able to see **colleagues' leave** (the team calendar from 06)? | Useful for planning, and it reveals absence patterns. 06 already models confidential leave types. | Yes, for their own department, showing dates but not leave types. |
| OQ-804 | Should the portal show **attendance figures an employee might dispute** — lateness counts, absence totals — or only the raw record? | Transparency reduces disputes; it also creates anxiety and, over time, self-comparison. Raised as 04 OQ-420. | Show the record and the current month's totals; no trends, no rankings, no comparisons with colleagues. |
| OQ-805 | Is a **second language** needed here specifically? This is the surface where it matters most — HR can work in English; the whole workforce may not. | 01 OQ-115 and 05 OQ-506 both defer to this. This portal is the strongest argument for i18n in the whole system. | Ask. If the answer is yes anywhere, it is yes here first. |
| OQ-806 | Do managers actually use the admin shell, or will they only ever open the portal? | Decides whether the manager area is deferred or built. | Deferred, with dashboard prompts as the bridge. |
| OQ-807 | Should employees be able to **upload documents** (a sick note, a certificate) rather than only view them? | 06 already allows an attachment on a leave request, so half of this exists. A general upload needs a review queue so uploads are not ignored. | Upload allowed against a specific request only, not free-form. |
| OQ-808 | Is there a **company announcements** need — a noticeboard? | Every portal grows one. It is a small feature and a large scope question, and it belongs with notifications rather than here. | Not in v1. |
