# 09 — Performance Management — Requirements

## Actors

| Actor | What they do here |
|---|---|
| Employee | Sets and updates their own goals, writes a self-review, gives and receives feedback, reads and acknowledges their review. |
| Manager | Agrees goals with their reports, checks in, writes reviews, shares them, holds the conversation. |
| Skip-level manager | Optionally reads their reports' reports' reviews (OQ-903). Not a default. |
| HR admin | Configures cycles, templates, and scales; runs the cycle; sees completion, not content, by default (FR-V-06). |
| Super admin | Grants exceptional access, handles disputes, and is the only actor who can unlock an acknowledged review. |

## User stories

### Goals

- **US-01** — As an employee, I see what I am expected to achieve this period.
- **US-02** — As an employee or manager, I create a goal, with a target date and how it will be
  measured.
- **US-03** — As an employee, I update my progress and add a note, so my manager is not surprised at
  review time.
- **US-04** — As a manager, I see all my team's goals and their progress in one place.
- **US-05** — As either party, I see a goal's history — what changed, when, and who changed it.

### Reviews

- **US-06** — As HR, I open a review cycle, choose who is in it, and set the deadlines.
- **US-07** — As an employee, I complete my self-review, saving as I go, and submit it when ready.
- **US-08** — As a manager, I write my review of each report, with my draft private until I share it.
- **US-09** — As a manager, I see my team's completion status and what is outstanding.
- **US-10** — As a manager, I share the review with the employee when we are ready to discuss it.
- **US-11** — As an employee, I read my review and acknowledge it, optionally adding my own comment.
- **US-12** — As an employee, I can see my past reviews.

### Feedback

- **US-13** — As anyone, I give a colleague feedback outside the review cycle, attributed to me.
- **US-14** — As anyone, I request feedback about myself from named colleagues.
- **US-15** — As a manager, I see the feedback my report has received when writing their review.

### Administration

- **US-16** — As HR, I configure the review form: sections, questions, and whether each carries a
  rating.
- **US-17** — As HR, I configure rating scales — or configure that there is no rating at all.
- **US-18** — As HR, I track cycle progress across the company without reading the content.

## Functional requirements

### Visibility — FR-V

**This section governs every other section.** Where any other requirement conflicts with it, this
one wins.

| ID | Requirement |
|---|---|
| FR-V-01 | Every review, review answer, and goal comment has an explicit visibility state (D-02). Nothing is visible outside its author until a deliberate transition. |
| FR-V-02 | A manager's review is `DRAFT` (author only), then `SUBMITTED` (author and HR), then `SHARED` (the employee too). There is no state in which an employee sees a draft. |
| FR-V-03 | A self-review is `DRAFT` (employee only), then `SUBMITTED` (manager and HR too). A manager cannot see a partial self-review, and cannot be shown that one exists in draft. |
| FR-V-04 | A manager's review cannot be shared before the employee's self-review is submitted, unless the deadline has passed or HR overrides — otherwise the self-review is written in response to the rating. |
| FR-V-05 | Once `SHARED`, a review cannot be edited (D-06). An addition is a dated, attributed appendix, visible to both parties. |
| FR-V-06 | **HR sees completion, not content, by default.** Reading review content requires `performance.read_content`, which is a separate grant and is logged. |
| FR-V-07 | Every read of review content is logged: who, what, when (D-08). This is a read log like feature 07's payslip access log, not an audit entry. |
| FR-V-08 | A manager sees their direct reports' content. A skip-level manager sees it only if the company enables that (OQ-903), and never retrospectively for reviews written before they became the manager (D-09). |
| FR-V-09 | On a manager change, the previous manager loses access to content at the end of the current cycle; the new manager gains access to goals and the current cycle only. Past reviews require HR to grant access explicitly. |
| FR-V-10 | An employee can always see everything about themselves that has been shared, and their own drafts. |
| FR-V-11 | Peer feedback given inside an anonymous multi-reviewer cycle is shown only in aggregate, and only when at least the configured minimum number of responses exists (D-05). Below that threshold it is not shown at all — not even to HR. |

### Goals — FR-G

| ID | Requirement |
|---|---|
| FR-G-01 | A goal has: title, description, owner (an employee), optional cycle, start and target dates, status, progress, and an optional measure (target value, unit, current value). |
| FR-G-02 | Goals may be created by the owner or their manager, and edited by either (D-03). Every change records who and when. |
| FR-G-03 | Statuses: `DRAFT`, `ACTIVE`, `ACHIEVED`, `PARTIALLY_ACHIEVED`, `NOT_ACHIEVED`, `CANCELLED`. A cancelled goal keeps its history and its reason. |
| FR-G-04 | Progress is a percentage or a measured value, updated by a **check-in**: a dated note with an optional progress value, attributed to its author. |
| FR-G-05 | Check-ins are visible to the owner and their manager (FR-V) and are never silently edited — a correction is a new check-in. |
| FR-G-06 | A goal may be linked to a parent goal, so a team objective can have individual contributions beneath it. One level of nesting is enough for v1; the model allows more. |
| FR-G-07 | Goals may optionally be visible to the wider team (OQ-906), per goal, off by default. |
| FR-G-08 | Goals outlive cycles: a goal not tied to a cycle continues until closed. |
| FR-G-09 | Overdue goals are surfaced to the owner and manager, not to HR or anyone else. |

### Cycles — FR-C

| ID | Requirement |
|---|---|
| FR-C-01 | A cycle has a name, a period, a review template, and dated stages: self-review opens/closes, manager review closes, sharing closes, acknowledgement closes. |
| FR-C-02 | Participants are chosen by rule — everyone, a department, an employment type — with explicit inclusion and exclusion, and are resolved and **stored** when the cycle opens, not evaluated live. |
| FR-C-03 | Employees hired after the cycle opens, or on probation, may be excluded by rule; the reason is recorded so "why am I not in this" is answerable. |
| FR-C-04 | Each participant generates review instances: one self-review, one manager review, and any peer reviews. |
| FR-C-05 | A participant whose manager is unset is an **exception** at cycle open, listed for HR, not silently skipped. |
| FR-C-06 | Cycle stages drive notifications (05): opening, reminders before each deadline, overdue notices, and "your review has been shared". |
| FR-C-07 | Deadlines do not enforce themselves: a passed deadline marks a review overdue and permits HR to act, but never auto-submits or auto-acknowledges on someone's behalf. |
| FR-C-08 | A cycle can be closed; closing locks all instances, and incomplete ones are recorded as incomplete rather than deleted. |
| FR-C-09 | An employee who leaves mid-cycle is removed from the active list and their instances are retained per retention policy (OQ-908). |

### Review forms — FR-F

| ID | Requirement |
|---|---|
| FR-F-01 | A template has ordered sections; each section has ordered questions. |
| FR-F-02 | Question types: free text, rating (against a named scale), rating plus comment, single choice, multiple choice, and goal review (auto-populated from the participant's goals). |
| FR-F-03 | Each question declares which reviewer types answer it — self, manager, peer — so one template serves the whole cycle without duplication. |
| FR-F-04 | A question may be required; a review cannot be submitted with required questions unanswered. |
| FR-F-05 | Templates are versioned by snapshot: a review instance stores the template as it was when the instance was created, so editing a template mid-cycle does not change reviews in progress (D-07's discipline applied to forms). |
| FR-F-06 | A rating scale has ordered levels with a value, label, and definition. Scales are configurable, and a template may contain no rating questions at all (OQ-902). |
| FR-F-07 | A stored answer keeps the rating's value **and** a snapshot of its label and definition (D-07). |
| FR-F-08 | An overall rating, if the template has one, is a question like any other. The system does **not** compute one by averaging (see FR-X-02). |

### Review instances — FR-R

| ID | Requirement |
|---|---|
| FR-R-01 | An instance has a subject, a reviewer, a type, a state, a template snapshot, answers, and timestamps for each transition. |
| FR-R-02 | Drafts save automatically and often; losing a half-written review is unacceptable and is the most likely cause of the feature being abandoned. |
| FR-R-03 | Submission validates required questions and moves the instance to `SUBMITTED`. |
| FR-R-04 | Sharing (manager instances only) moves it to `SHARED` and notifies the employee. It is a deliberate act, separate from submission (FR-V-02). |
| FR-R-05 | The employee acknowledges a shared review, optionally with a comment, moving it to `ACKNOWLEDGED`. Acknowledgement means "I have seen this", and the UI must say so — not "I agree". |
| FR-R-06 | An employee may decline to acknowledge, with a comment. This is recorded as `ACKNOWLEDGED_WITH_DISAGREEMENT`, never as nothing. A system that only records agreement is not recording anything. |
| FR-R-07 | After acknowledgement the instance is immutable (D-06); either party may append a dated comment. |
| FR-R-08 | Unlocking an acknowledged review requires super admin and a reason, and is audited. |
| FR-R-09 | A review's history — each transition, with actor and time — is visible to both parties. |

### Feedback — FR-B

| ID | Requirement |
|---|---|
| FR-B-01 | Feedback has an author, a subject, a body, an optional cycle link, and a visibility setting: to the subject only, to the subject and their manager, or to the manager only. |
| FR-B-02 | Feedback is attributable by default (D-05). The author is always recorded even where display is anonymised. |
| FR-B-03 | Feedback requests: an employee asks named colleagues for feedback; requests are tracked and can be declined without a reason. |
| FR-B-04 | A manager writing a review sees feedback about that employee that is visible to them (FR-B-01). |
| FR-B-05 | Feedback cannot be edited after the subject has seen it; a correction is a new entry. |
| FR-B-06 | Manager-only feedback about an employee is visible to that employee **on request through HR**, and the UI tells authors this before they write it. Feedback the subject can never see is a rumour with a database row. |

### Explicit non-requirements — FR-X

Stated as requirements because "we did not build it" and "we decided not to build it" are different
things, and only the second survives a feature request.

| ID | Requirement |
|---|---|
| FR-X-01 | **The system must not read attendance, leave, or payroll data for any performance purpose.** No such read path exists (D-04). A future request to "factor in punctuality" is a change to this requirement, not a small addition. |
| FR-X-02 | **The system must not compute an overall rating** by averaging question ratings, nor rank employees against one another, nor produce a distribution to be enforced. |
| FR-X-03 | **The system must not notify a manager that an employee is writing a self-review**, has saved a draft, or how long they spent on it. |
| FR-X-04 | **No engagement metrics about people** — logins, time in review, word counts — are collected or surfaced. |

## Non-functional requirements

| ID | Requirement |
|---|---|
| NFR-01 | Autosave must be reliable and visible: a saved draft states when it was saved (FR-R-02). |
| NFR-02 | A manager's team review dashboard for 15 reports loads in one request. |
| NFR-03 | Review content is never included in notifications, exports the actor is not entitled to, or application logs (05 D-06's reasoning applies equally). |
| NFR-04 | Visibility rules are enforced server-side in the query, not by filtering after fetch — the same rule as feature 02's scope filtering. |
| NFR-05 | Read logging must not slow reads perceptibly; it is written asynchronously and its failure never fails the read. |
| NFR-06 | Free-text answers may be long. Storage and rendering must handle several thousand words without truncating silently. |
| NFR-07 | The portal (08) hosts the employee-facing screens; they must work at 375 px like everything else there. |

## Permission keys added by this feature

| Key | Meaning | Default holders |
|---|---|---|
| `performance.goal.read` | See goals | Employee: SELF · Manager: DEPARTMENT |
| `performance.goal.write` | Create and update goals | Employee: SELF · Manager: DEPARTMENT |
| `performance.review.participate` | Complete reviews assigned to you | Everyone: SELF |
| `performance.review.manage_team` | Write, share reviews for your reports | Manager: DEPARTMENT |
| `performance.cycle.read` | See cycle progress and completion | HR: ALL |
| `performance.cycle.write` | Configure cycles, templates, scales | HR: ALL |
| `performance.read_content` | Read review content for others | **Nobody by default** — granted deliberately (FR-V-06) |
| `performance.unlock` | Unlock an acknowledged review | Super admin |
| `performance.feedback.give` | Give and request feedback | Everyone: SELF |

`performance.read_content` holding no default — not even for HR admin — is the clearest expression
of D-08 and should survive review.

## Acceptance criteria (feature-level)

1. A manager's draft review is invisible to the employee through every endpoint, verified by a test
   that requests it directly as the employee.
2. A manager cannot see that a self-review draft exists before it is submitted.
3. A manager's review cannot be shared before the self-review is submitted, unless the deadline has
   passed or HR overrides — and the override is recorded.
4. An HR admin without `performance.read_content` sees completion percentages and no answers.
5. Every read of review content produces a read-log entry naming the reader.
6. An acknowledged review cannot be edited; an appended comment is dated and attributed.
7. Declining to acknowledge is recorded with its comment, not discarded.
8. Editing a template mid-cycle does not change reviews already in progress.
9. A rating stored in 2026 still renders its label and definition in 2029 after the scale is
   rewritten.
10. Anonymous peer feedback is withheld entirely below the minimum-response threshold, including
    from HR.
11. A manager change removes the old manager's content access at cycle end and does not grant the
    new manager past reviews.
12. There is no code path, endpoint, or report in which attendance or leave data influences a
    performance figure (FR-X-01).
13. Autosave preserves a draft across a browser crash, and the UI states when it last saved.
