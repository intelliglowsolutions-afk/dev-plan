# 09 — Performance Management

**Priority:** Nice-to-have · **Build order:** 9 of 11 · **Status:** planned (Step 3)

## Purpose

Record what people are expected to achieve, how they are progressing, and what was concluded at
review time — and keep that record honest, private, and fair.

## What makes this feature different

Everything planned so far records **facts**: a punch happened, a day was a holiday, a balance is
11.5, a payslip totals 105,600. Disputes are about whether the fact was recorded correctly.

This feature records **opinions**. A rating is a judgement, a goal's "progress" is an estimate, and
feedback is one person's view of another. That changes the design problems entirely:

| Elsewhere | Here |
|---|---|
| Correctness is verifiable | Correctness is not a meaningful concept; fairness is |
| Data is shown as soon as it exists | Data is deliberately withheld until it is ready to be shared |
| Audit answers "who changed this" | Audit also answers "who read this" |
| More data is better | More data is often worse |

And one further difference: this is the feature most likely to be **misused**. An HR system with
attendance data, leave data, and a rating field is one small step from automatically scoring people
on punctuality. The plan below refuses that step explicitly (D-04), because it is much easier to
refuse now than after someone has asked for it.

## Scope

**In scope**

- **Goals**: objectives for a person or a team, optionally measurable, with target dates, owners,
  and progress updates.
- **Check-ins**: lightweight periodic progress notes against goals.
- **Review cycles**: a named period with participants, stages, and deadlines.
- **Review forms**: configurable templates — sections, questions, rating scales — completed as a
  self-review, a manager review, and optionally peer reviews.
- **Ratings**: configurable scales, with no built-in methodology.
- **Feedback**: continuous, attributable notes between colleagues, and requested feedback.
- **Sharing and acknowledgement**: the explicit step where a manager's review becomes visible to the
  employee, and the employee records that they have seen it.

**Out of scope**

- **Any automatic link between attendance/leave data and performance** (D-04). This is a
  deliberate omission, not an oversight.
- **Any automatic link to pay.** Feature 07 reads nothing from here. A rating may inform a manual
  compensation decision made by a human in feature 07's screens; the system draws no line between
  them (OQ-905).
- Disciplinary process, grievances, and improvement plans as a formal workflow (OQ-907).
- Calibration sessions and forced distributions — modelled for later, not built (D-10).
- Succession planning, competency frameworks, skills matrices.
- Recruitment → [10](../10-recruitment-onboarding/README.md).

## Files

| File | Contents |
|---|---|
| [requirements.md](./requirements.md) | User stories, functional and non-functional requirements, visibility rules, permission keys |
| [data-model.md](./data-model.md) | Prisma models, visibility states, seed and migration notes |
| [api-design.md](./api-design.md) | Cycles, goals, reviews, feedback, and the sharing lifecycle |
| [ui-ux.md](./ui-ux.md) | Goal screens, the review form, manager dashboard, states and copy |

## Key decisions taken in this plan

| # | Decision | Rationale |
|---|---|---|
| D-01 | **No built-in methodology.** Cycles, forms, questions, and rating scales are all configuration. The system has no opinion about OKRs, SMART goals, nine-box grids, or five-point scales. | Same stance as payroll (Step 1) and leave. Companies' performance processes differ more than their payroll does, and a system that assumes one will be fought rather than used. |
| D-02 | **Everything has an explicit visibility state.** Nothing a manager writes is visible to the employee until it is deliberately shared. Nothing an employee writes in a self-review is visible to the manager until submitted. | The single most important rule in the feature. A manager who believes their draft notes might be visible writes nothing useful; an employee who sees a half-formed rating reads it as final. |
| D-03 | **Goals are jointly owned.** Both the employee and their manager can create and update them, and every change is attributed and kept. | A goal only one party can edit is either imposed or unmanaged. Attribution is what keeps joint editing honest. |
| D-04 | **No automatic scoring from attendance, leave, or payroll data. No integration exists to make it possible.** | Punctuality is not performance, and a system that quietly converts one into the other is both wrong and corrosive. Refusing structurally — no read path, not merely no feature — is what makes the refusal durable. |
| D-05 | **Feedback is attributable by default.** Anonymity is available only inside a structured multi-reviewer cycle, and only where a minimum number of responses prevents identification. | Anonymous free-text feedback in a small company is neither anonymous nor kind. The threshold is what makes the anonymity real rather than nominal. |
| D-06 | **A completed review is immutable** once acknowledged, like a finalised payslip. Corrections are appended, not edited. | The review is the record of what was said at the time. If it can be rewritten afterwards, it evidences nothing. |
| D-07 | **Ratings store both the scale value and a snapshot of its label and definition.** | A "3" means nothing in two years if the scale has been rewritten. Same snapshot discipline as feature 07. |
| D-08 | **Access is narrow and read access is logged.** There is no broad `performance.read`; visibility is the employee, their manager, HR, and — optionally — the skip-level manager. | This is the second-most sensitive data in the system after pay, and unlike pay it is qualitative and easily misread out of context. |
| D-09 | **Reviews follow the person, not the reporting line.** A change of manager does not hide history from the employee or from HR, and does not automatically expose it to the new manager (OQ-903). | A new manager inheriting three years of another manager's opinions, unasked, is a common and damaging default. |
| D-10 | **Calibration is modelled but not built.** The data model leaves room for a calibration stage; v1 has none. | Calibration is where performance systems become political, and it should not be built speculatively. |

## Dependencies

- **Depends on:** 01 (permissions, audit), 02 (employees, reporting lines — which determine who
  reviews whom), 03 (dates, job runner), 05 (notifications: cycle opened, review due,
  review shared), 08 (the employee-facing screens live in the portal shell).
- **Depended on by:** 11 (reporting, with the same visibility constraints).
- **Deliberately not connected to:** 04, 06, 07 (D-04).
- **Nice-to-have**, so it may be deferred entirely without affecting anything else. Nothing in
  features 01–08 or 10–11 depends on it.

## Open questions

| ID | Question | Why it matters | Proposed default |
|---|---|---|---|
| OQ-901 | **Does the company actually run performance reviews, and how?** Annual? Twice a year? Continuous check-ins with no formal cycle? | The whole feature's shape. Building a formal annual cycle for a company that does quarterly conversations produces an unused module. | Ask. Nothing here should be built on a guess. |
| OQ-902 | **What rating scale**, if any? Some companies deliberately have none. | D-01 makes it configurable, but "no rating at all" is a mode the UI must handle, not an empty scale. | Configurable, with a no-rating option supported. |
| OQ-903 | When someone changes manager, should the **new manager see past reviews**? | D-09 says not automatically. The alternative — full history on arrival — is common and has real costs. | New manager sees goals and the current cycle; past reviews require HR to grant access. |
| OQ-904 | Are **peer reviews** (360°) wanted, and if so are they anonymous? | D-05 constrains anonymity; whether peers are involved at all changes the cycle model materially. | Not in v1; the model supports adding them. |
| OQ-905 | Does a rating **feed a pay decision**, and should the system make that link visible? | The link exists in reality in most companies. Making it explicit is honest and makes ratings higher-stakes; leaving it implicit is the status quo. | No system link. A human decides in feature 07. |
| OQ-906 | Who sees an employee's goals — just them and their manager, or the team? | Shared goals improve coordination and remove privacy. | Employee and manager; team visibility is a per-goal option. |
| OQ-907 | Is a formal **performance improvement plan** process needed? | It is a legally-sensitive workflow with its own states, deadlines, and evidence requirements. Doing it badly is worse than not doing it. | Out of scope; record it as a goal with notes if needed. |
| OQ-908 | Retention: how long are reviews kept, and what happens to them when someone leaves? | Interacts with 02 OQ-203. Performance records about a former employee are exactly the kind of data that should not be kept forever by default. | Kept while employed plus two years; then reviewed. |
| OQ-909 | Should employees be able to see **who has read their review**? | Symmetry with D-08's read logging. Unusual, and arguably fair. | Not in v1; the log exists and could surface later. |
