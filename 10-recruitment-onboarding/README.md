# 10 — Recruitment & Onboarding

**Priority:** Nice-to-have · **Build order:** 10 of 11 · **Status:** planned (Step 3)

## Purpose

Get from "we need someone" to "they are an employee with a login, a shift, and a desk" — and keep
the record of how that happened.

## Two halves and one hinge

This is really two features that share a seam:

- **Recruitment** deals with **people outside the company**: candidates, applications, interviews,
  offers. Its subject has no employment relationship and no account.
- **Onboarding** deals with **a new employee before and just after their first day**: paperwork,
  equipment, access, introductions.

The hinge between them is the **hire**: the moment a candidate becomes an employee record in
feature 02. That single transition is where most of this feature's risk lives, and it is specified
carefully (D-05).

## What makes this feature different

Every other feature in this system holds data about people who work here, with an employment
relationship that justifies holding it. This one holds data about **people who do not work here and
mostly never will** — their CVs, their contact details, sometimes their salary expectations and
interview assessments.

That inverts the retention question. Elsewhere the default is "keep it, someone may need it"
(feature 02 D-08: employees are never deleted). Here the default must be the opposite: **candidate
data is deleted on a schedule unless there is a reason to keep it** (D-02).

A rejected candidate's file, kept indefinitely for no reason, is the largest quiet liability this
system could accumulate — hundreds of people's personal data, held by a company they have no
relationship with, that nobody remembers is there.

## Scope

**In scope**

- **Requisitions**: a request to hire, with approval before a role is advertised.
- **Job postings**: the public-facing description of a requisition.
- **Applications and candidates**: intake by HR entry and by a public application form.
- **Pipeline**: configurable stages, with candidates moving through them.
- **Interviews**: scheduling, interviewers, and structured scorecards.
- **Offers**: recording what was offered, and its outcome.
- **Hire**: converting an accepted candidate into an employee (feature 02).
- **Onboarding**: a template of tasks instantiated per hire, with owners and dates relative to the
  start date.
- **Consent and retention**: what candidates were told, and automatic deletion when the period
  expires.

**Out of scope**

- **Automated screening, scoring, or ranking of candidates** (D-07). Structured human assessment,
  yes. A machine deciding who is worth interviewing, no.
- Job-board integrations (LinkedIn, Indeed), CV parsing, and any AI-assisted shortlisting (OQ-1005).
- Background checks and reference-checking workflows (OQ-1006).
- Electronic signature of contracts (OQ-1007).
- A full careers website. The plan includes a single public application form, not a marketing site.
- The employee record itself → [02](../02-employee-management/README.md). This feature calls 02's
  creation path; it never writes employee data directly.
- Compensation for the hire → [07](../07-payroll/README.md), set after the employee exists.

## Files

| File | Contents |
|---|---|
| [requirements.md](./requirements.md) | User stories, functional and non-functional requirements, the hire transition, permission keys |
| [data-model.md](./data-model.md) | Prisma models, retention, seed and migration notes |
| [api-design.md](./api-design.md) | Requisitions, pipeline, interviews, the public form, the hire conversion |
| [ui-ux.md](./ui-ux.md) | Pipeline board, candidate profile, scorecards, onboarding checklist |

## Key decisions taken in this plan

| # | Decision | Rationale |
|---|---|---|
| D-01 | **A candidate is not an employee.** Separate model, separate table, no `Employee` row until hire. | They have no employment relationship, no employee code, and no place in the org chart. Creating employee rows for applicants pollutes every count, report, and scope query in the system. |
| D-02 | ~~**Candidate data has a retention period and is deleted automatically when it expires**~~ **REVISED 2026-09-18 (OQ-1002): no automatic deletion anywhere in the system.** Retention is tracked and **reported**; deletion is an HR-initiated action. The purge job becomes a due-for-review report, and erasure on request stays available. | The original concern stands and is now a **deliberate, accepted liability**: candidate data accumulates indefinitely unless someone acts. The consent notice must not promise automatic deletion it will not perform — its wording changes from "we'll delete it after 6 months" to what the company will actually do. Worth revisiting with a data-protection adviser. |
| D-03 | **Pipeline stages are configurable** per requisition, from a template. | Hiring a developer and hiring a cleaner are not the same process, and a fixed five-stage pipeline gets worked around within a month. |
| D-04 | **Interview scorecards are hidden from other interviewers until submitted**, exactly as feature 09 hides review drafts. | Otherwise the first submitted scorecard anchors everyone else's. This is the single most valuable thing structured interviewing does, and it is lost entirely if feedback is visible early. |
| D-05 | **Hiring is a deliberate conversion with a preview**, and it calls feature 02's employee-creation path rather than writing employee data itself. | Two ways to create an employee means two sets of validation and one of them will be wrong. The preview exists because the conversion is hard to undo. |
| D-06 | **Onboarding tasks come from a template**, instantiated per hire with owners and dates **relative to the start date**. | "Two days before they start" is the useful unit, not a fixed date that is wrong for every subsequent hire. |
| D-07 | **No automated screening, scoring, or ranking.** Scorecards are structured human judgement; the system computes no candidate ranking and auto-rejects nobody. | Same stance as feature 09 D-04, for the same reason, with the additional point that automated screening carries discrimination risk the company would own without being able to explain it. |
| D-08 | **The public application form is untrusted input** and is treated as hostile: unauthenticated, rate-limited, sanitised, file-type-checked, size-capped, and never able to reach any authenticated surface. | It is the only endpoint in the entire system that accepts data from the open internet. Feature 03's device endpoint is at least authenticated by a comm key; this one is not. |
| D-09 | **Rejection records an internal reason and sends a separate, plainer message.** The two are never the same text. | The internal note ("weak on the technical round, poor culture fit") is not what should be sent, and a system that offers one field for both will eventually send it. |
| D-10 | **Onboarding runs against the employee record**, created at offer acceptance with a future start date — not against the candidate. | Feature 02 already supports future hire dates and a `PROBATION` status. Pre-start tasks belong to the person who is about to join, and this avoids a parallel identity for them. |

## The hire transition

The riskiest operation in the feature, so it is stated once, here:

1. An offer is recorded as accepted.
2. HR opens the **hire preview**: the employee record that will be created, with the candidate's
   details mapped to employee fields, gaps flagged.
3. HR supplies what only they can — employee code (validated against the device PIN rules, 02
   FR-E-02), department, position, manager, employment type, start date.
4. On confirmation, in one transaction: create the employee via feature 02's path, link the
   candidate to it, mark the application `HIRED`, instantiate the onboarding tasks (D-06), and
   optionally start feature 01's user invite.
5. The candidate record is **retained**, linked, and exempted from the retention purge (D-02) —
   the hiring record of a current employee is part of their employment history.

What deliberately does **not** happen automatically: no compensation is set (07), no shift is
assigned (04), no leave policy is applied (06). Each of those is a decision, and each has its own
screen. The onboarding checklist carries them as tasks with links, so they are visible as
outstanding rather than silently missing.

## Dependencies

- **Depends on:** 01 (permissions, audit, user invite), 02 (employee creation, departments,
  positions, documents), 03 (settings, job runner for the retention purge), 05 (notifications to
  candidates, interviewers, and task owners), 08 (new hires see their onboarding tasks in the
  portal).
- **Depended on by:** nothing. Like feature 09, this can be deferred entirely.
- **Shares a pattern with 09:** hidden-until-submitted assessment (D-04), and the same refusal to
  automate judgement (D-07).

## Open questions

| ID | Question | Why it matters | Proposed default |
|---|---|---|---|
| OQ-1001 | **Does the company hire often enough for this to be worth building?** A company hiring four people a year runs recruitment in a spreadsheet and a shared inbox, and is right to. | The honest question for a nice-to-have. Onboarding checklists are useful at any volume; the pipeline is not. | Ask. Consider building onboarding only. |
| OQ-1002 | **What retention period applies to unsuccessful candidates**, and does the company have a legal obligation here? | D-02. This is a question for whoever advises the company on data protection — it is jurisdiction-specific and this plan does not guess. | 6 months from the final decision, with consent-to-retain as an opt-in extension. **Needs confirming with someone qualified.** |
| OQ-1003 | Is there a **public application form**, or do candidates arrive by email and get entered by HR? | D-08. A public endpoint is a real security surface; not having one removes it entirely. | Public form, built defensively. |
| OQ-1004 | Who **approves a requisition** before a role is advertised — and is approval needed at all? | Determines whether requisitions need an approval chain like feature 06's leave requests. | Single approval by a named approver; skippable by configuration. |
| OQ-1005 | Any interest in **CV parsing or AI-assisted shortlisting**? | D-07 says no. Worth confirming that the answer is a considered no rather than an assumed one, since it will be asked for. | No. |
| OQ-1006 | Are **background or reference checks** part of the process, and should the system track them? | They involve third parties and sensitive results, which is a meaningful extension. | Tracked as a pipeline stage with a note, not as a structured workflow. |
| OQ-1007 | Should offer letters and contracts be **signed electronically** through the system? | A significant addition (document generation, signature capture, legal validity) and usually the first thing asked for after the basics work. | Offer recorded in the system; the document is handled outside it. |
| OQ-1008 | Do **interviewers need accounts**? External interviewers, or managers without system access, cannot fill in a scorecard. | Changes the scorecard flow considerably — a tokenised link for someone with no account is a different security model. | Interviewers are employees with accounts. |
| OQ-1009 | Should candidates get a **status page** to see where their application stands? | Considerably better candidate experience; also exposes internal stage names and timing. | Not in v1; status is communicated by email. |
| OQ-1010 | How much of the onboarding checklist involves **people outside HR** (IT, facilities)? Do they have accounts and will they use the system? | D-06 assigns task owners. Tasks owned by someone who never logs in are a checklist that lies. | Assign to HR and the manager; note where IT tasks are tracked elsewhere. |
