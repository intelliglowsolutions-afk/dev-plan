# 06 — Leave Management — UI / UX

Builds on the shell and principles in [01 ui-ux.md](../01-roles-permissions-auth/ui-ux.md).

> **Design grounding.** Written against the workspace `ui-ux-pro-max` guideline set per
> `C:\Dev\CLAUDE.md`, not from memory. Guidelines cited inline as *[UX-nn]* and listed at the end.
> Visual direction stays with the dense-admin default established in 01–05.

## Principles specific to this feature

1. **Tell them the cost before they commit.** The single most valuable thing this UI does is turn
   "1–7 October" into "4.5 days, because the weekend and National Day don't count, leaving you 7".
   Every screen that touches a date range shows its consequence.
2. **A balance must be able to explain itself.** The number is never presented alone; one click
   reaches the dated entries that sum to it. This is what turns a dispute into a conversation.
3. **Approvers decide in one place.** If judging a request requires opening the employee's profile,
   their balance, and the team calendar, it will be approved without any of them being looked at.
4. **Nothing irreversible happens quietly.** Cancelling leave, forfeiting carry-over, and changing
   entitlement all state their effect in numbers before they run.

## Screen map

```
/leave                        My leave — balances, requests, history     leave.read (SELF)
/leave/request                Request form                              leave.request
/leave/approvals              Approvals queue                           leave.approve
/leave/calendar               Team calendar                             leave.read
/leave/balances               Everyone's balances (HR)                  leave.balance.read
/leave/requests               All requests (HR)                         leave.read
/admin/leave/types            Types and policies                        leave.policy.read
/admin/leave/engine           Accrual, carry-over, year-end             leave.run_accrual
```

`/leave` is the employee-facing hub; feature 08 embeds the same components in the self-service
shell rather than duplicating them.

## `/leave` — my leave

**Balance cards**, one per type available to the employee. Each shows the four numbers separately,
because collapsing them is what causes disputes (FR-B-04):

```
Annual leave
11.5 days available
Entitled 20  ·  Used 6.5  ·  Pending 2
⚠ 3 days expire on 31 March 2027
```

The expiry line is the most actionable thing on the screen and earns its prominence. It carries an
icon and text, never colour alone *[UX-37]*.

Each card links to **How this was calculated** — the ledger as a dated list: "1 Jan 2026 · Annual
grant · +20", "14 Feb · Leave taken, 12–13 Feb · −2", "2 Sep · Reserved, pending approval · −2".
Nothing here is a mystery, and support questions end here rather than with HR.

Below: **My requests**, newest first, with status, dates, days, and where a pending one has got to
("With Ravi Menon since Monday · escalates Thursday"). Approved future leave can be cancelled;
pending can be withdrawn.

Empty state: "No leave requested yet. You have 20 days of annual leave." — the balance belongs in
the empty state, since that is the question behind opening the page.

## `/leave/request` — the request form

The most important form in the system. Layout: fields left, a **live cost panel** right (beneath, on
narrow screens).

**Fields** — type, dates, half-day options, reason, attachment. Every input has a real `<label>`
*[UX-43]*.

- **Type** first, because it changes everything after it: whether half days are allowed, whether an
  attachment is expected, whether approval is needed. Unavailable types are absent, not disabled —
  with one exception: a type withheld only by probation shows the date it becomes available, since
  that is information the employee needs rather than an absence to puzzle over.
- **Dates** — two date inputs *and* a calendar for range selection. The calendar is the pleasant
  path; the inputs are the accessible one. Drag-to-select a range is never the only way to choose
  dates *[UX-103]*, and day cells are at least 24×24 CSS px with adequate spacing *[UX-104]*.
- The calendar shades non-working days from the work week and marks holidays by name, so a range
  that looks long is visibly mostly free *[FR-C-04]*.
- **Half-day** controls appear only for the first and last day of the range, and only where the type
  allows them — progressive disclosure rather than four controls that are usually irrelevant.
- **Attachment** prompts itself after the configured number of consecutive days: "Sick leave over 2
  days usually needs a doctor's note." A prompt, not a block (OQ-608).

**The cost panel** updates as the dates change, in a fixed-height container so nothing below it
jumps as the numbers arrive *[UX-19]*:

```
4.5 days
  1 Oct  Thu   1      2 Oct  Fri   1
  3–4 Oct       Weekend
  6 Oct         National Day
  7 Oct  Tue   0.5    (afternoon)

Balance after: 7 days
⚠ 2 colleagues are already off on 1–2 October — Imran Qadir, Sara Ahmed
```

The per-day breakdown is what makes the total believable. The clash line is a warning, never a block
*[FR-C-03]*.

**Validation** is inline, beside the field, connected with `aria-describedby`, and announced
*[UX-55, UX-44]* — never only a summary at the top. And every error carries its recovery *[UX-80]*:

- Insufficient balance → "You have 11.5 days; this needs 12. Shorten the range by half a day, or ask
  HR about unpaid leave." with the alternative type linked.
- Notice period → "Annual leave needs 7 days' notice. The earliest you can start is 22 September."
  with a control to shift the range there.
- Overlapping request → names the other request and links to it.
- Zero working days → "This range is all weekends and holidays — no leave is needed."

Submit shows progress, then a success state naming who it went to *[UX-61, UX-34]*. Not a toast
alone: the record matters more than the flourish.

## `/leave/approvals` — the queue

A work list built so that the decision needs nothing else open *[principle 3]*. Each row expands in
place to the full context the API already returns:

```
Ayesha Khan · Annual leave · 1–7 Oct · 4.5 days            waiting 1 day
  "Family trip"
  Balance after: 7 of 20   ·   3 days taken in the last 90
  2 others off on 1–2 Oct: Imran Qadir, Sara Ahmed
  Escalates to Imran Qadir on Thursday
                                        [Reject]  [Approve]
```

- **Approve** is the primary action but not the easy accident: it sits right of Reject with adequate
  separation, and both are ≥ 24 px targets with spacing *[UX-104]*.
- **Reject** opens a small dialog requiring a reason, stating that the requester will see it
  *[FR-A-03]*.
- Rows routed by delegation are labelled — "In Ravi Menon's queue, delegated to you until 20 Sep" —
  so an approver always knows whose authority they are exercising *[FR-A-06]*.
- If the cost changed since submission, the row says so before the approver acts: "Now 3 days, was 4
  — 6 October became a public holiday." *[FR-R-03]*
- Bulk approve exists but requires explicit row selection; there is no "approve all" *[UX-35]*.

Requests waiting longer than the escalation threshold are marked with the date they escalate, not
merely coloured *[UX-37]*.

Empty state: "Nothing waiting on you." — which is a good outcome and should read like one.

## `/leave/calendar` — the team calendar

A month grid: employees down the side, days across. Cells are coloured by leave type and **carry the
type's initial or a pattern**, so colour is reinforcement, not the only channel *[UX-37]*. Pending
leave is visually distinct from approved — hatched rather than solid — with a legend *[FR-C-02]*.

Weekends and holidays shade the whole column, so a week that looks empty of leave is visibly a week
with two days nobody was expected anyway.

**Confidential types** render as "On leave" with no name, to everyone but HR and the employee. The
server never sends the name (FR-T-07) — the UI is not the thing keeping the secret.

Controls: month navigation, department filter, include-pending toggle, and view switch (month /
week / list). Filters live in the URL so a view can be shared *[UX-5]*.

The **list view is the accessible equivalent** of the grid and is available at every width, not only
on mobile — a dense grid is not readable by screen reader however carefully it is marked up. Below
tablet width the grid becomes the list by default.

Hovering a cell shows dates, type, and status; the same information is reachable by keyboard focus,
not hover alone *[UX-117 from 05]*.

## `/leave/balances` — HR's view

A table: employee, department, and a column per leave type showing available, with entitled/used on
hover and in the row's expansion. Filters for department, type, and the two queries HR actually runs:
**"more than N days left"** (chasing people to take leave) and **"expiring within 60 days"**.

Row actions: adjust balance (dialog, reason mandatory, showing before and after), view ledger,
record leave on their behalf.

Export respects scope and the same columns.

## `/admin/leave/types` — types and policies

**Types list** — name, paid, deducts balance, approval required, half days, confidential, active,
and the count of requests using it. Deactivation rather than deletion, with the same wording pattern
feature 03 uses for departments.

**Policy editor** — the fields are meaningless in isolation, so each is rendered with its
consequence, the way feature 04 renders shift rules:

> Entitlement **20** days · Annual grant on 1 January
> Carry-over **allowed**, capped at **5** days, expiring **31 March**
> → *An employee finishing the year with 7 unused days carries 5 into next year and loses 2. The 5
> must be used by 31 March.*

That generated sentence is the review mechanism. Most policy mistakes are invisible in the fields
and obvious in the sentence.

**Assignment** — who this policy applies to, by employment type, department, grade, or named
employee, with a live count: "Applies to 34 employees." and a link to see them. A *Check an
employee* control resolves the policy for one person and shows which assignment won, since policy
resolution is what HR will most often disbelieve.

Changing entitlement mid-year opens the confirmation the API demands (FR-T-08), stated in people and
days: "This affects 34 employees by +2 days each. Apply from next leave year, or write an adjustment
of +2 to all 34 balances now?"

## `/admin/leave/engine` — accrual and year-end

The screen where 200 balances change at once, so it is deliberately unhurried — the same shape as
feature 04's recompute screen.

Pick the run and the period, then **Preview** (the primary action). Apply stays disabled until a
preview has run.

The preview leads with totals, and `forfeited` is given equal weight to `carried over`:

```
Carry-over to 2027 · 187 employees
Carried over  612 days        Forfeited  88 days
```

Then the per-employee table: remaining, cap, carried, forfeited, expiry. Sortable by forfeited
descending, because that column is the one someone should look at before clicking.

Applying confirms with the totals restated *[UX-35]*, then shows the run record. A period already
run says so plainly — "Carry-over for 2027 was run on 2 Jan by admin@company.com. 612 days carried
over." — rather than erroring, because the honest answer to "did this happen?" is yes (D-08).

**Run history** below: type, period, who, when, counts, with a link from any run to the entries it
produced.

## States, accessibility, responsive

- Six states everywhere, plus **pending approval** as a first-class state on every request surface.
- Colour is never the only carrier of leave type, status, or pending-vs-approved *[UX-37]*.
- All errors are inline, field-associated, and announced *[UX-55, UX-44]*.
- Calendar range selection always has a non-drag, keyboard-operable path *[UX-103]*; day cells meet
  the minimum target size *[UX-104]*.
- Destructive and bulk actions confirm with their effect stated in numbers *[UX-35]*.
- Nothing asks for information the system already holds *[UX-106]* — the request form never asks who
  the approver is, or for dates already chosen in the calendar.
- Motion is limited to the cost panel's updates and calendar transitions, using transform and
  opacity, and is suppressed under `prefers-reduced-motion` *[UX-13, UX-9]*.
- Below 768 px: the cost panel moves beneath the fields and becomes sticky at the bottom of the
  viewport while dates are being chosen — the number must stay visible while it is changing.

## Copy reference

| Situation | Text |
|---|---|
| Balance expiring | {n} days expire on {date}. |
| Cost breakdown header | {n} days — weekends and holidays in this range don't count. |
| Zero working days | This range is all weekends and holidays — no leave is needed. |
| Insufficient balance | You have {available} days; this needs {needed}. Shorten the range, or ask HR about unpaid leave. |
| Notice period | {Type} needs {n} days' notice. The earliest you can start is {date}. |
| Overlap | You already have leave booked from {start} to {end}. |
| Clash warning | {n} colleagues are already off on {dates} — {names}. |
| Attachment prompt | Sick leave over {n} days usually needs a doctor's note. |
| Waiting | With {approver} since {day} · escalates {day}. |
| Delegated | In {name}'s queue, delegated to you until {date}. |
| Cost changed | Now {n} days, was {m} — {date} became a public holiday. |
| Cancel confirm | Cancelling returns {n} days to your balance. Days already taken ({m}) are not refunded. |
| Period locked | {Month} payroll was finalised on {date}. This leave can't be changed. Ask HR to make a payroll adjustment. |
| Policy sentence | An employee finishing the year with {n} unused days carries {cap} into next year and loses {lost}. |
| Entitlement change | This affects {n} employees by {delta} days each. |
| Already run | {Run} for {period} was run on {date} by {user}. {n} days carried over. |
| Nothing waiting | Nothing waiting on you. |

## Guidelines applied

From `.claude/skills/ui-ux-pro-max/data/ux-guidelines.csv`:

| Ref | Guideline | Where it shows up here |
|---|---|---|
| UX-103 | Dragging movements need a single-pointer alternative | Calendar range selection always has date inputs |
| UX-104 | Target size minimum 24 CSS px | Calendar day cells, approve/reject buttons |
| UX-37 | Never convey information by colour alone | Leave types, pending vs approved, expiry warnings |
| UX-55 / UX-44 | Inline errors beside the field, announced | Request form validation |
| UX-80 | Errors carry a recovery path | Every rejection reason offers a next step |
| UX-35 | Confirm destructive and irreversible actions | Cancel, forfeit, entitlement change |
| UX-61 / UX-34 | Submit feedback: loading then success | Request submission |
| UX-19 | Reserve space for async content | The live cost panel |
| UX-43 | Real labels on every input | Request form |
| UX-106 | Never ask for the same information twice | Approver and dates never re-entered |
| UX-5 | URLs reflect state | Calendar filters are shareable |
| UX-13 / UX-9 | Transform-based motion, reduced-motion honoured | Cost panel and calendar |

## Open questions

| ID | Question |
|---|---|
| OQ-617 | Should the team calendar show leave for people outside the viewer's department (names only, no type)? Useful for cross-team planning; a privacy decision rather than a UI one. |
| OQ-618 | Does the request form need a "check with my manager first" draft state, or is withdraw-after-submit enough? Drafts add a status; withdrawal already covers most of it. |
| OQ-611 | Whether clashes warn or block — the UI is built for warn. Blocking would need a staffing rule per team and a clear message about who to ask. |
| OQ-601 | The balance cards and policy sentences are only as good as the real types and entitlements. Until those are known, every screenshot of this feature is illustrative. |
