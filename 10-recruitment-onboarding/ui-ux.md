# 10 — Recruitment & Onboarding — UI / UX

Builds on the shell and principles in [01 ui-ux.md](../01-roles-permissions-auth/ui-ux.md).
The new hire's screens live in the portal shell from [08](../08-employee-self-service/ui-ux.md).

> **Design grounding.** Written against the workspace `ui-ux-pro-max` guideline set per
> `C:\Dev\CLAUDE.md`. Guidelines cited inline as *[UX-nn]* and listed at the end.

## Principles specific to this feature

1. **The public form is the only screen a stranger sees.** It is the company's face to people who
   do not work there, half of whom will be rejected. It must be fast, accessible, honest about what
   happens to their data, and possible to complete on a phone with one hand.
2. **Nobody falls through the cracks.** The pipeline's job is to make "we forgot about her for three
   weeks" visible. Stalled candidates are surfaced everywhere they appear.
3. **Assessment before consensus.** Scorecards are hidden until everyone has submitted, and the UI
   never hints at what others thought or who is holding things up *[D-04]*.
4. **Deletion is a feature, shown as one.** Retention and purges are visible, explained, and
   previewable — not a silent background job people discover when data is gone.

## Screen map

```
Public (no account)
  /jobs/:slug                      One job posting + apply form

Admin shell
  /recruitment                     Requisitions and open roles       requisition.read
  /recruitment/requisitions/:id    One requisition
  /recruitment/postings/:id/pipeline   The pipeline board            candidate.read
  /recruitment/candidates/:id      Candidate profile                 candidate.read
  /recruitment/interviews/mine     My interviews and scorecards      scorecard.submit
  /recruitment/scorecards/:id      The scorecard form                scorecard.submit
  /recruitment/applications/:id/hire   Hire flow                     hire
  /recruitment/retention           What is due for deletion          retention
  /onboarding                      Board of upcoming starters        onboarding.read
  /onboarding/employees/:id        One person's checklist            onboarding.read
  /admin/onboarding/templates      Templates                         onboarding.write

Portal
  /portal/onboarding               My tasks before and after I start onboarding.read (SELF)
```

## `/jobs/:slug` — the public posting and form

One column, mobile-first, no navigation chrome beyond the company name and logo.

**The posting**: title, location, employment type, summary, description. Salary if the posting says
to show it — and if it does not, no empty "Salary: —" row, which reads worse than silence.

**The form**, beneath it on the same page rather than behind a button. An extra click before a form
is a measurable drop-off, and there is nothing here worth hiding:

- Name, email, phone, location, cover note (optional), CV upload.
- Every field has a **visible label** *[UX-54, UX-43]*, correct `inputmode` and `autocomplete`
  *[UX-63]*, and is marked required where it is *[UX-59]*.
- The CV field states what it accepts and the size limit **before** a file is chosen, not after a
  rejected upload. Upload shows progress and a stable layout *[UX-78, UX-19]*.
- Validation on blur, inline, beside the field, announced *[UX-55, UX-44]*, each error carrying its
  fix *[UX-80]*.

**The data notice** sits above the submit button, not behind a link:

> We'll keep your application for **6 months** and then delete it. Only our hiring team can see it.
> ☐ You can also keep my details on file for future roles *(optional)*

The keep-on-file box is unticked and separate from submitting *[FR-R-03]*. Bundling consent into the
submit action is both a dark pattern and, in many jurisdictions, not consent at all.

**No CAPTCHA that blocks assistive technology** *[D-08, UX-1]*. The honeypot and timing checks in
`api-design.md` are invisible to everyone, including screen-reader users. If a visible challenge ever
becomes necessary, it needs an accessible alternative before it ships.

**After submitting**, the form is replaced by a confirmation — not a toast, since the page is the
whole experience: *"Thank you. We've received your application and will be in touch."* Nothing about
whether they have applied before *[NFR-03]*.

Accessibility here is not optional in the way it sometimes quietly is elsewhere: applicants include
people with disabilities, and an inaccessible application form is an accessible way to be sued.
Keyboard operable throughout, 4.5:1 contrast *[UX-36]*, 16 px body text *[UX-67]*, targets ≥ 44 px
*[UX-22, UX-66]*, tested at 320 px *[UX-65]*.

## `/recruitment/postings/:id/pipeline` — the board

Columns are stages; cards are candidates. The familiar shape, with the familiar accessibility trap.

**Dragging must never be the only way to move a candidate** *[UX-103]*. Every card has a *Move to*
menu listing the stages, operable by keyboard, and the drag is an enhancement on top. This is the
single most commonly failed guideline in software of this kind.

Card contents, deliberately minimal: name, days in stage, next interview, and a marker when
scorecards are outstanding. Not the CV, not a photo, not a score — because there is no score
*[D-07]*.

**Stalled candidates** carry an icon and the word, never colour alone *[UX-37]*: "18 days in
Interview". A column header shows its count and how many are stalled. This is principle 2 made
visible, and it is the board's main reason to exist.

Bulk selection via checkboxes with an action bar *[UX-91]*; bulk reject confirms with the count
spelled out *[UX-35]*: "Reject 6 candidates? They'll each get the message below."

Below tablet width the board becomes a stage-filtered list. A five-column drag board on a phone is
not worth attempting, and the *Move to* menu already makes the list fully capable.

## `/recruitment/candidates/:id` — the profile

Header: name, contact, source, and — prominently if present — **previous applications**: "Applied
twice before. Rejected at Interview, March 2024." That single line is the institutional memory the
system exists to provide *[FR-C-02]*.

Tabs: **Application** (stage history as a timeline with durations), **Documents** (CV, permission
checked), **Interviews** (with scorecards, subject to the visibility rule), **Notes**, **Offer**.

The stage timeline shows how long each stage took. Nothing else makes a slow process as obvious as
seeing "Screening: 19 days" written down.

**Rejecting** opens a dialog with the two fields visibly separated *[D-09]*:

```
Why are we rejecting?          (internal — never sent)
  Reason   [ Skills mismatch ▾ ]
  Notes    [ Strong communicator, but no experience with…      ]

What should we send them?      (the candidate will read this)
  [ Thank you for your time. We've decided to move forward… ]
  ☑ Send this message
```

The two labels and their parenthetical qualifiers are the design. A single "reason" field is how an
internal note reaches a candidate.

## `/recruitment/scorecards/:id` — the scorecard

The screen where D-04 is either upheld or quietly undone.

A banner at the top, always:

> 🔒 **Only you can see this.** Other interviewers' feedback appears once everyone has submitted.

Then the criteria, each with a rating scale showing **definitions, not just labels** (same reasoning
as feature 09), a comment field, and finally an overall recommendation and free notes.

- Autosave with a visible "Saved 14:02" *[FR-I-06's sibling in 09; NFR-01 there]*.
- Submission confirms and states its finality *[UX-35]*: "Submit your feedback on Imran? You can't
  edit it afterwards — you can add a comment later."
- Before everyone has submitted, the page says how many are outstanding and **not who**: "Waiting on
  2 more." Naming them creates exactly the pressure structured interviewing is meant to remove.
- Once all are in, the same screen shows every scorecard side by side, which is when the hiring
  conversation should happen.

## The hire flow

Three steps with a progress indicator *[UX-81]*: **Confirm details → Fill the gaps → Review**.

The final review renders the API's preview as three plainly-labelled lists:

> **We'll create an employee record for Sara Ahmed**, starting 1 November.
>
> **We'll also:**
> - Create 11 onboarding tasks, first due 30 October
> - Keep her candidate record permanently, as part of her employment history
>
> **We won't:**
> - Set her salary — do that in Payroll after hiring
> - Assign a shift — do that in Attendance
> - Assign a leave policy — do that in Leave
>
> *These three appear as onboarding tasks so they don't get missed.*

The "we won't" list is the most useful thing on the screen *[FR-H-06]*. Those omissions are
deliberate, and stating them is what stops a new hire reaching their first payday with no
compensation record.

Errors surfaced from feature 02 — a duplicate employee code — appear inline on the field that caused
them *[UX-55]*, with 02's own wording, since it explains the rule ("codes are never reused") better
than a generic message would.

## Onboarding

**`/onboarding`** — a board of people starting soon, one card each: name, start date, days until,
tasks done / total, and overdue count. Sorted by start date. Overdue is text plus icon *[UX-37]*.

**One person's checklist** — tasks grouped by owner (HR, manager, IT, the employee), each with due
date and status. Completing is one tap; *Not applicable* requires a reason, and the field is
required because a blank one is indistinguishable from a skipped task *[FR-N-04]*.

Tasks with a `linkPath` carry a *Go to* button — "Set salary → Payroll" — so the checklist is a
launcher rather than a reminder to go and find something.

**`/portal/onboarding`** — the new hire's view, and their first experience of the system. Warm, short,
mobile-first: what to bring, what to read, what to sign. Their tasks only; never a hint of the
internal ones about their equipment or their contract.

**Templates** — an ordered list of task definitions, reorderable by drag **and** explicit move
controls *[UX-103]*, each with owner role and a day offset shown in words: "2 days before start",
"On their first day", "1 week after". The offset is the thing people get wrong, and rendering it as
a phrase rather than `-2` is most of the fix.

## `/recruitment/retention` — deletion, shown

An unusual screen: a privacy action presented to its operator rather than buried in a job.

```
Due for deletion on 22 September          142 candidates

  Accountant (closed Mar 2026)            38
  Warehouse Operative (closed Apr 2026)   104

  Applications between 4 Jan and 2 Apr 2026

What happens: their names, contact details, CVs and notes are deleted.
We keep the role, the stage they reached, and the dates — with nothing
identifying them — so hiring statistics still work.

[ Preview file ]  [ Delete now ]
```

Counts and postings, **never the candidates' names** *[FR-R-08]* — a preview of a privacy action
should not itself be a list of personal data. Deleting confirms with the count typed back, the
same type-to-confirm reserved elsewhere for genuinely irreversible acts *[UX-35, 03 ui-ux]*.

## States, accessibility, responsive

- Six states everywhere; plus **stalled** on pipeline surfaces and **waiting on others** on
  scorecards.
- Drag is never the only interaction, on the board or in template ordering *[UX-103]*.
- Colour never carries meaning alone *[UX-37]*.
- The public form is the accessibility high-water mark: keyboard, screen reader, 320 px, no
  inaccessible challenge, real labels *[UX-54, UX-43, UX-63, UX-65, UX-22]*.
- Focus visible and never obscured by sticky bars *[UX-28, UX-100]*; reduced motion honoured
  *[UX-9]*.
- Realistic sample data in design and testing, never lorem ipsum — a pipeline board full of
  placeholder names hides how it reads with real ones *[UX-87]*.

## Copy reference

| Situation | Text |
|---|---|
| Public data notice | We'll keep your application for {n} months and then delete it. Only our hiring team can see it. |
| Keep on file | You can also keep my details on file for future roles *(optional)* |
| Application received | Thank you. We've received your application and will be in touch. |
| Previous applications | Applied {n} times before. {outcome} at {stage}, {date}. |
| Stalled | {n} days in {stage}. |
| Rejection, internal field | Why are we rejecting? *(internal — never sent)* |
| Rejection, message field | What should we send them? *(the candidate will read this)* |
| Bulk reject | Reject {n} candidates? They'll each get the message below. |
| Scorecard privacy | Only you can see this. Other interviewers' feedback appears once everyone has submitted. |
| Waiting on others | Waiting on {n} more. |
| Scorecard submit | Submit your feedback on {name}? You can't edit it afterwards — you can add a comment later. |
| Hire, will also | Keep her candidate record permanently, as part of her employment history. |
| Hire, won't | Set her salary — do that in Payroll after hiring. |
| Hire, reassurance | These three appear as onboarding tasks so they don't get missed. |
| Task not applicable | Why doesn't this apply? |
| Retention explanation | We keep the role, the stage they reached, and the dates — with nothing identifying them — so hiring statistics still work. |
| Template offset | 2 days before start / On their first day / 1 week after |

## Guidelines applied

From `.claude/skills/ui-ux-pro-max/data/ux-guidelines.csv`:

| Ref | Guideline | Where |
|---|---|---|
| UX-103 | Dragging needs a single-pointer alternative | Pipeline board, template ordering |
| UX-54 / UX-43 / UX-59 | Visible labels, never placeholder-only, required marked | Public form, all forms |
| UX-63 | `inputmode` and autocomplete | Public form |
| UX-55 / UX-44 / UX-80 | Inline announced errors with a fix | Public form, hire flow |
| UX-78 / UX-19 | Stable loading, reserved space | CV upload, board |
| UX-35 | Confirm irreversible actions | Bulk reject, scorecard submit, purge |
| UX-37 | Never colour alone | Stalled, overdue, statuses |
| UX-91 | Bulk actions with checkbox and action bar | Pipeline |
| UX-81 | Progress for multi-step | Hire flow |
| UX-22 / UX-66 / UX-65 / UX-67 / UX-36 | Touch size, mobile-first, readable text, contrast | Public form especially |
| UX-28 / UX-100 / UX-9 | Focus, obscuring, reduced motion | Throughout |
| UX-87 | Realistic sample content | Design and test data |

## Open questions

| ID | Question |
|---|---|
| OQ-1003 | Without a public form, this feature's most demanding screen disappears — along with its only unauthenticated write surface. |
| OQ-1009 | A candidate status page would be a second public surface and a marked improvement in how rejection feels. It also commits the company to keeping it current, which is harder than it sounds. |
| OQ-1014 | Whether the CV is required. Requiring a document excludes people applying from a phone with nothing to hand. |
| OQ-1015 | Should the pipeline board show interviewer names on cards? Useful for coordination; it also makes "who is holding this up" visible, which cuts against principle 3. |
