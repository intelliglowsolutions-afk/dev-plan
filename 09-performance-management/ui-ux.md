# 09 — Performance Management — UI / UX

Builds on the shell and principles in [01 ui-ux.md](../01-roles-permissions-auth/ui-ux.md).
Employee-facing screens are hosted in the portal shell from
[08](../08-employee-self-service/ui-ux.md).

> **Design grounding.** Written against the workspace `ui-ux-pro-max` guideline set per
> `C:\Dev\CLAUDE.md`. Guidelines cited inline as *[UX-nn]* and listed at the end.

## Principles specific to this feature

1. **Visibility is the interface.** The most important thing any screen here communicates is *who
   can see this*. It appears on every writing surface, permanently, in plain words — not in a
   tooltip, not once at the top of a flow.
2. **Never lose someone's writing.** A manager loses forty minutes of careful thought if a draft
   disappears, and they will not write it twice. Autosave is a correctness requirement, not a
   convenience *[FR-R-02]*.
3. **A review is a conversation, not a form submission.** The interface should not make it feel like
   filing a tax return: one section at a time, room to write, no counters, no progress-gamification.
4. **Refuse to become a surveillance tool.** Nothing here shows who is writing what, when, or how
   long they took *[FR-X-03, FR-X-04]*. If a screen would enable that, it is not built.

## Screen map

```
Portal (employee)
  /portal/goals                 My goals
  /portal/goals/:id             One goal, with check-ins
  /portal/reviews               My reviews — to complete, and about me
  /portal/reviews/:id           The review form, or a shared review
  /portal/feedback              Feedback given and received

Admin shell (manager / HR)
  /performance/team             My team's goals and review status      manage_team
  /performance/reviews/:id      Writing a review                       manage_team
  /performance/cycles           Cycles                                 cycle.read
  /performance/cycles/:id       One cycle: progress, participants      cycle.read
  /admin/performance/templates  Review templates                       cycle.write
  /admin/performance/scales     Rating scales                          cycle.write
```

## The visibility banner

Used on every writing surface, above the fold, always present:

```
┌────────────────────────────────────────────────────┐
│ 🔒 Only you can see this until you share it.       │
│    Ayesha will see it when you choose to share.    │
└────────────────────────────────────────────────────┘
```

Variants, one per state — and each says who, not merely "private":

| Where | Text |
|---|---|
| Manager review, draft | Only you can see this until you share it. Ayesha will see it when you choose to share. |
| Manager review, submitted | Submitted. HR can see this. **Ayesha cannot see it yet** — share it when you're ready to talk. |
| Self-review, draft | Only you can see this. Your manager sees it when you submit. |
| Self-review, submitted | Submitted. Your manager and HR can see this. |
| Shared review | Shared with Ayesha on 15 September. |
| Feedback, manager-only | Visible to Ayesha's manager, not to Ayesha. She can ask HR to see it. |

The bold line in the second row exists because a manager who has submitted naturally assumes the
employee can now see it. The distinction between submitting and sharing is the feature's least
intuitive rule and needs stating where the mistake would happen *[D-02, FR-V-02]*.

## The review form

The screen this feature lives or dies by.

**One section at a time**, with a step indicator showing where you are *[UX-81]*. Not an
accordion of twelve questions, and not one scrolling page — a long review on a single page is
where people give up.

```
Goals and achievements            Section 2 of 5

How did Ayesha perform against her goals this year?
(Her goals are listed below with her check-ins.)

┌──────────────────────────────────────────────┐
│                                              │
│                                              │
└──────────────────────────────────────────────┘

                      Saved 10:42 · [Back] [Next]
```

- Every question has a **visible label** and its own help text *[UX-54, UX-43]* — never a
  placeholder standing in for a label.
- Required questions are marked as required *[UX-59]*, and the submit step names the ones
  outstanding with a link to each *[UX-55, UX-80]*, rather than a generic "please complete all
  fields".
- Text areas are generous and resizable, with line length bounded for readability *[UX-73]*. No
  character counters unless there is a real limit — a counter turns writing into a task.
- **Saved {time}** sits beside the navigation, updating quietly *[FR-R-02, NFR-01]*. On failure it
  becomes "Couldn't save — retrying", and it must never be silent about a failure: a manager who
  believes their work is saved and loses it will not use the system again.
- `GOAL_REVIEW` questions render the subject's goals inline with their check-ins, so the reviewer is
  not reconstructing the year from memory in another tab.
- Navigation between sections is free — forward, back, and by the step indicator. Nothing is gated.

**Rating questions** show the scale's levels with their **definitions**, not just labels *[D-07]*:

```
○ 1  Below expectations   Did not meet the agreed objectives.
○ 2  Approaching          Met some objectives, with gaps.
● 3  Meets expectations   Did what the role requires, well.
○ 4  Exceeds              Consistently beyond what the role requires.
○ 5  Outstanding          Rare — sustained impact beyond the role.
```

Definitions visible at the point of choosing is what makes ratings comparable between managers.
Hidden behind a tooltip, they are not read, and every manager invents their own scale.

**Submission** is a distinct confirming step naming what happens next *[UX-35]*:

> Submit your review of Ayesha?
> HR will see it. **Ayesha will not see it until you share it.** You can't edit after submitting.

## Sharing

A separate, deliberate action from a separate place — the team screen, or the completed review —
never a checkbox on the submit dialog *[FR-R-04]*.

Where the self-review is outstanding, the control is disabled with the reasoning, not a bare block
*[FR-V-04]*:

> Ayesha hasn't submitted her self-review yet. Sharing now means she writes it after seeing your
> rating. *Due 20 September.*

## Reading a review about you

Calm, readable, one page — this is read, not filled in. Sections with the manager's answers, the
employee's own self-review answers shown alongside where both exist, and the rating with its
definition.

**Acknowledgement** is at the end and its wording is the most carefully chosen in the feature
*[FR-R-05, FR-R-06]*:

> **Acknowledge this review**
> This records that you've seen and discussed it. It doesn't mean you agree.
>
> ○ I've seen this
> ○ I've seen this and I don't agree
>
> Anything you'd like to add? *(optional, visible to your manager and HR)*
>
> [ Acknowledge ]

Two options with equal visual weight. A single "I acknowledge" button records agreement nobody gave,
and disagreement that has nowhere to go is expressed in ways the system cannot help with.

After acknowledgement both parties can append dated comments; the review itself is fixed *[FR-R-07]*.

## `/performance/team` — the manager's screen

One table: report, goals on track / at risk / overdue, self-review status, my review status, shared,
acknowledged. Each cell links to the thing.

Statuses carry words, never colour alone *[UX-37]*. A manager with fifteen reports needs to see what
is outstanding in one glance, so this screen is dense — the one place in this feature where density
is right.

What it deliberately does **not** show: whether a report has *started* their self-review, how long
anyone spent, or any activity timestamp beyond the state transitions *[FR-X-03, FR-X-04]*.

## Goals

**My goals** as cards: title, progress, target date, status. Overdue ones first, marked with an icon
and text *[UX-37]*.

A goal opens to its description, measure, and a **check-in timeline** — each entry dated and
attributed, newest first. Adding a check-in is one field plus an optional progress update, and it is
the most frequent action in the feature, so it is one tap from the goal and one from the dashboard.

Editing shows who last changed what, since goals are jointly owned *[D-03]* and "my manager changed
my goal without telling me" is the failure mode.

The team view lists reports with their goals nested, filterable to overdue or at-risk.

## Cycles (HR)

**Cycle list** with status and progress. Creating one is a short form: name, period, template,
participants rule, and the four deadlines.

**Preview participants** before opening — the same preview-before-commit pattern as features 04, 06,
and 07 — showing included, excluded with reasons, and problems: "Bilal Aslam has no manager set. He
will get a self-review only." with a link to fix it *[FR-C-05]*.

**Progress** is completion only *[FR-V-06]*: four stage bars, a department breakdown, and overdue
counts. An HR admin without `performance.read_content` cannot click through to any answer, and the
screen does not imply they can — there is no disabled "view" affordance teasing content they cannot
reach.

## Templates and scales

Template editor: sections and questions, reorderable by drag **and** by explicit move controls
*[UX-103]*. Each question sets its type, whether it is required, which reviewer types answer it, and
its rating scale.

A **preview** renders the form as each reviewer type will see it — the only reliable way to catch a
question that makes sense for a manager and reads absurdly in the self-review.

Editing a template used by an open cycle shows the now-familiar impact panel: "This template is in
use by *2026 annual review*. Reviews already started keep the current questions." *[FR-F-05]* —
which is reassurance rather than a block.

Scale editor: ordered levels with label and definition, with a warning if any definition is empty,
because an undefined level is where rating drift starts.

## States, accessibility, responsive

- Six states everywhere, plus **saving / saved / save failed** on every writing surface.
- Autosave must not fight the user: no cursor jumps, no re-ordering, no interruption of typing.
  State transitions on controls cancel cleanly rather than depending on transition events for
  correctness *[UX-119]*.
- All inputs have visible labels; required fields are marked *[UX-54, UX-59, UX-43]*.
- Line length bounded for long text *[UX-73]*; body text ≥ 16 px.
- Every status carries text *[UX-37]*; focus visible and never obscured *[UX-28, UX-100]*.
- Reduced motion honoured *[UX-9]*.
- The review form works at 375 px — a manager will write one on a phone at least once, and an
  employee will read one on a phone every time.
- Sample content in design and testing is realistic, not lorem ipsum *[UX-87]*: this interface is
  about tone, and placeholder text hides whether the tone works.

## Copy reference

| Situation | Text |
|---|---|
| Manager draft | Only you can see this until you share it. {Name} will see it when you choose to share. |
| Manager submitted | Submitted. HR can see this. {Name} cannot see it yet — share it when you're ready to talk. |
| Self-review draft | Only you can see this. Your manager sees it when you submit. |
| Shared | Shared with {name} on {date}. |
| Submit confirm | HR will see it. {Name} will not see it until you share it. You can't edit after submitting. |
| Share blocked | {Name} hasn't submitted their self-review yet. Sharing now means they write it after seeing your rating. |
| Acknowledgement | This records that you've seen and discussed it. It doesn't mean you agree. |
| Acknowledgement options | I've seen this / I've seen this and I don't agree |
| Save state | Saved {time} · Couldn't save — retrying |
| Missing required | {n} questions still need an answer: {links}. |
| Manager-only feedback | Visible to {name}'s manager, not to {name}. They can ask HR to see it. |
| No manager | {Name} has no manager set. They will get a self-review only. |
| Template in use | This template is in use by {cycle}. Reviews already started keep the current questions. |
| Empty scale definition | This level has no definition. Managers will interpret it differently. |

## Guidelines applied

From `.claude/skills/ui-ux-pro-max/data/ux-guidelines.csv`:

| Ref | Guideline | Where |
|---|---|---|
| UX-81 | Progress indicators for multi-step processes | Section-by-section review form |
| UX-54 / UX-43 / UX-59 | Visible labels, never placeholder-only, required marked | Every form field |
| UX-55 / UX-80 | Inline errors that name the fix | Submit validation |
| UX-73 | Bounded line length | Long free-text answers |
| UX-35 | Confirm irreversible actions | Submit, share, acknowledge |
| UX-37 | Never colour alone | All statuses |
| UX-103 | Drag is never the only way | Template reordering |
| UX-119 | Cancellable state transitions | Autosave and control state |
| UX-28 / UX-100 | Visible focus, never obscured | Throughout |
| UX-9 | Reduced motion | Transitions |
| UX-87 | Realistic sample content | Design and test data |

## Open questions

| ID | Question |
|---|---|
| OQ-910 | Draft history. Autosave overwrites; a manager who deletes a paragraph cannot recover it. A per-answer append-only history would fix it cheaply and is worth deciding before the form is built. |
| OQ-902 | A no-rating configuration changes this UI materially — the form becomes prose-only and the team screen loses its most scannable column. Both modes need designing, not one with the other bolted on. |
| OQ-912 | Should the employee see their manager's review and their own self-review side by side, or sequentially? Side by side invites direct comparison, which is either the most useful thing here or the most uncomfortable. |
| OQ-901 | If there is no formal cycle, the review form and cycle screens are unbuilt and this feature is goals plus feedback — roughly a third of the work. |
