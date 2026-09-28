# 07 — Payroll — UI / UX

Builds on the shell and principles in [01 ui-ux.md](../01-roles-permissions-auth/ui-ux.md).

> **Design grounding.** Written against the workspace `ui-ux-pro-max` guideline set per
> `C:\Dev\CLAUDE.md`. Guidelines cited inline as *[UX-nn]* and listed at the end. Dense-admin
> direction, consistent with 01–06.

## Principles specific to this feature

1. **Make it hard to pay the wrong amount, easy to see why.** Every screen in the run flow is
   arranged around review: exceptions before totals, variance before approval, exclusions beside
   inclusions. The UI's job is to slow down the one action that should be slow.
2. **Numbers must be readable as numbers.** Right-aligned, tabular figures, consistent decimal
   places, thousands separators from company settings, currency stated once per surface rather than
   repeated on every cell. Negative amounts are shown with a minus and a distinct treatment, never
   by colour alone *[UX-37]*.
3. **Every figure can be traced.** A payslip line opens its calculation trail; an exception names
   where it is fixed; a variance names its likely cause. Nobody should have to ask a developer why
   a number is what it is.
4. **Sensitive by default.** Salary figures are not shown in passing. No pay data appears on an
   employee profile, a search result, a hover card, or an export the actor is not entitled to
   (D-06).

## Screen map

```
/payroll                        Dashboard — periods, run status         payroll.read
/payroll/runs/:id               The run workspace (tabbed)              payroll.read
/payroll/runs/:id/exceptions      · exceptions                          payroll.read
/payroll/runs/:id/variance        · variance                            payroll.read
/payroll/runs/:id/payslips        · payslips                            payroll.read
/payroll/payslips/:id           One payslip                             payroll.read / own
/payroll/compensation/:empId    Salary history and bank details         payroll.compensation.read
/admin/payroll/components       Components, formulas, bracket tables    payroll.component.read
/admin/payroll/structures       Salary structures                       payroll.component.read
/admin/payroll/periods          Pay calendar                            payroll.read
/me/payslips                    My payslips                             payroll.read_own
```

Nothing in this feature appears in a manager's navigation. `MANAGER` holds no payroll permission,
and the absence should be verified in review rather than assumed *[01 D-05]*.

## `/payroll` — the dashboard

Current period first: dates, cut-off, pay date, and the run's state as a **progress track**, since
the lifecycle is the mental model the whole feature runs on *[UX-81]*:

```
Draft ──● Calculated ──○ Approved ──○ Finalised ──○ Paid
         3 blocking exceptions
```

Each state shows what is needed to leave it and who can do it. A run sitting in `CALCULATED` for
three days with an approver who has not looked is the most common way payroll is late, so the track
names the person and how long it has waited.

Below: previous periods with their totals and status, and a **cut-off countdown** — "Cut-off in 4
days. 12 attendance corrections are still pending." — which links into feature 04. Payroll's quality
is mostly decided before the run starts, and this is where that is made visible.

## The run workspace

One page, four tabs: **Overview · Exceptions · Variance · Payslips**. The tab order is the order of
work, and the numbers on each tab say whether it needs attention.

Actions live in a persistent header bar, showing only the transition available now, with the others
visibly absent rather than disabled-and-mysterious. Every action that changes state confirms, and
the confirmation states the consequence in numbers *[UX-35]*.

### Overview

Headcount, gross, deductions, net, and a component-by-component total table. Beside each total, the
change from the previous period as an amount and a percentage.

**Population** is here too, with its exclusions listed (FR-R-02): "187 included · 4 excluded" with
each exclusion's reason. "Why wasn't X paid" is answered before it is asked.

### Exceptions — the working screen

Grouped by kind, blocking first. Each row: employee, what is wrong, and **where to fix it**.

```
BLOCKING · Unresolved attendance (3)
  Ayesha Khan      3 days have no attendance data (device offline 14–16 Sep)   → Device gaps
  Bilal Aslam      No active bank account                                      → Bank details
  Nadia Rauf       Net pay is negative (−2,400)                                → Payslip

WARNING · Variance (12)                                              [Acknowledge selected]
```

Every row's link opens the place the problem is solved — mostly in features 04 and 06. This is what
makes the exception list a workflow rather than a complaint.

Warnings can be acknowledged with a note, in bulk via a checkbox column and an action bar
*[UX-91]* — but blocking exceptions have no bulk dismissal, deliberately. Multi-select exists for
the case that is genuinely repetitive and is withheld from the case that should be considered one at
a time.

Recalculating preserves acknowledgements, and the screen says so before the officer worries about
losing twenty minutes of review.

### Variance

A table sorted by absolute change: employee, previous net, current net, change, percentage, and the
likely cause. Rows beyond threshold are marked with an icon and text *[UX-37]*.

This is the last line of defence against a misconfigured component reaching a bank, so it is dense
and scannable rather than pretty: tabular numerals, right-aligned figures, one row per person, no
cards.

Wide on narrow screens by nature — the table gets a horizontal scroll container rather than
collapsing, because the comparison only works with the columns side by side *[UX-71]*.

### Payslips

A list with employee, gross, deductions, net, and status. Click through to the payslip. Batch PDF
generation runs in the background with progress and a stable layout while it works *[UX-78]*.

## The payslip

Designed to be read by someone who is not an accountant, and to be printed.

**Header** — employee, code, department, position, period, pay date. All from the snapshot, so it
reflects the period, not today (D-03).

**Earnings** and **Deductions** as two blocks, each line showing name and amount, right-aligned,
with gross, total deductions, and **net pay** given clear hierarchy. Net pay is the number the reader
came for.

**Each line expands to its calculation trail**:

> **House rent allowance — 18,182**
> Basic salary ÷ working days × paid days = 50,000 ÷ 22 × 20 = 45,454.55
> × 40% = 18,181.82
> Rounded to nearest unit = **18,182**

Written as arithmetic a person can check by hand. This single affordance removes most payroll
disputes, because the answer to "why is this less than last month" becomes visible rather than
requiring HR to reconstruct it.

**Adjustment lines** carry their reason inline — "Arrears for August (backdated increase)" — because
an unexplained correction reads as an error (FR-A-02).

**Attendance summary** for the period — working days, paid days, unpaid days, overtime hours — since
these are the inputs people question most.

A **superseded** payslip carries a prominent banner with a link to its replacement, never a silent
substitution (FR-L-07).

## `/me/payslips` — the employee's view

A simple list by period with net pay and a download. Opening one shows the same payslip component in
a lighter shell (feature 08 hosts this).

A month-to-month comparison is offered — previous and current side by side with differences
highlighted — because it is the question every employee actually has, and answering it in the
product prevents a conversation with HR.

Only published payslips appear. Nothing here hints that an unpublished one exists.

## `/payroll/compensation/:empId`

Behind `payroll.compensation.read`, which no default role except HR admin and super admin holds.

Current salary and structure at the top; the dated history below as a timeline — effective date,
base, structure, reason, who recorded it. Changes are additions to the timeline, never edits.

**Adding a change** shows its consequence before saving:

> From 1 August 2026, base salary becomes 60,000 (was 52,000).
> ⚠ This date is inside two finalised periods. Arrears of **16,000** will be proposed for review and
> paid in the next run. The August and September payslips will not change.

That last sentence is the one that prevents a support ticket.

**Bank details** in a separate panel, account number masked except the last four, with a change
history. Changing it warns that the employee will be notified — the notification is a control, and
the person changing it should know it exists.

## `/admin/payroll/components` — components and formulas

**List** — code, name, type, method, order, rounding, active, and `ratesReviewedOn` with a stale
marker where it is over a year old *[UX-37: icon plus text]*.

**Editor** — the fields, and beneath them a **test panel** that is the point of the screen:

```
Formula   BASIC / WORKING_DAYS * PAID_DAYS

Test with   BASIC 50,000   WORKING_DAYS 22   PAID_DAYS 20
→ 45,454.55  →  × 40% = 18,181.82  →  rounded 18,182
```

Available identifiers are listed as insertable chips, wrapping rather than clipping, with a `+n`
disclosure if they overflow *[UX-115, UX-116]*. Validation runs on blur with the error beside the
field and the available names listed *[UX-55, UX-80]*:

> `HRA_OLD` is not a component or a variable.
> Available: BASIC, HRA, TRANSPORT, WORKING_DAYS, PAID_DAYS, UNPAID_DAYS, OVERTIME_HOURS

**Calculation order** is shown as an ordered list of all components, so the consequence of an order
number is visible rather than inferred. Reordering is possible by drag **and** by explicit *Move up
/ Move down* controls — dragging is never the only way *[UX-103]*.

Editing a component in use shows the same kind of impact panel features 03, 04, and 06 use: "Used by
3 structures and 187 employees. Finalised payslips are not affected." The second sentence is the
reassurance that lets someone make the edit at all.

**Bracket tables** get a plain row editor with upper bound, rate, and fixed amount, plus a preview
that runs a sample value through the table and shows the bracket it landed in.

## States, accessibility, responsive

- Six states everywhere, plus **calculating** — a progress state with a count, `aria-busy`, and a
  stable layout, since a run takes a minute or two *[UX-78, UX-81]*.
- Monetary values use tabular figures and consistent decimals; currency appears in the column header
  or a caption rather than on every row.
- Negative and reduced amounts carry a sign and a shape, not only colour *[UX-37]*.
- All state-changing actions confirm with the consequence stated numerically *[UX-35]*.
- Bulk acknowledgement uses a checkbox column with an action bar; blocking items are excluded from
  it *[UX-91]*.
- Wide comparison tables scroll horizontally within their container rather than collapsing
  *[UX-71]*; the payslip itself is fully responsive and readable on a phone, because that is where
  employees open it.
- Errors are inline, field-associated, announced, and carry a recovery path *[UX-55, UX-44, UX-80]*.
- Focus is never obscured by the sticky run action bar; it is offset with `scroll-padding`
  *[UX-100]*.

## Copy reference

| Situation | Text |
|---|---|
| Cut-off countdown | Cut-off in {n} days. {m} attendance corrections are still pending. |
| Population exclusion | {name} — {reason} |
| Blocking exception | This run can't be approved until {n} blocking exceptions are resolved. |
| Unresolved attendance | {n} days have no attendance data ({device} offline {dates}). |
| Negative net | Net pay is negative ({amount}). Check deductions for this employee. |
| Variance cause | {n} unpaid leave days this period ({m} last period). |
| Recalculate reassurance | Your acknowledgements and manual entries are kept. |
| Approve confirm | Approve {n} employees, {total} net? This records your approval against these totals. |
| Finalise confirm | Finalising locks {period} for {n} employees. Attendance and leave for those dates can't be changed afterwards — corrections become adjustments in the next run. |
| Reopen warning | Reopening releases the lock on {period}. Published payslips will be marked superseded. |
| Backdated pay change | This date is inside {n} finalised periods. Arrears of {amount} will be proposed for review. The {periods} payslips will not change. |
| Bank change notice | {name} will be notified that their bank details changed. |
| Bank export exclusions | {n} employees were excluded — no active bank account: {names}. |
| Stale rates | Rates last reviewed {date}. Check they are still current. |
| Superseded payslip | This payslip was replaced on {date}. *(View the current one)* |
| Unknown formula name | `{name}` is not a component or a variable. Available: {list}. |

## Guidelines applied

From `.claude/skills/ui-ux-pro-max/data/ux-guidelines.csv`:

| Ref | Guideline | Where it shows up here |
|---|---|---|
| UX-81 | Progress indicators for multi-step processes | The run lifecycle track |
| UX-78 | Loading matched to the wait, `aria-busy`, stable layout | Calculation and batch PDF |
| UX-35 | Confirm irreversible actions | Approve, finalise, reopen, bank change |
| UX-37 | Never colour alone | Negative amounts, exception severity, stale rates |
| UX-55 / UX-44 / UX-80 | Inline announced errors with a recovery path | Formula editor, exceptions |
| UX-91 | Bulk actions via checkbox column and action bar | Acknowledging warnings |
| UX-71 | Wide tables scroll rather than break | Variance and component totals |
| UX-115 / UX-116 | Chip collections wrap or disclose; labels stay whole | Available-identifier chips |
| UX-103 | Dragging is never the only way | Calculation-order reordering |
| UX-19 | Reserve space for async content | Totals and calculation progress |
| UX-100 | Focus never obscured by sticky UI | The run action bar |

## Open questions

| ID | Question |
|---|---|
| OQ-717 | Should the payslip's calculation trail be visible to employees, or only to payroll? Showing it prevents disputes and also exposes the formula structure, including how allowances are derived. Proposed: show it — the alternative is HR explaining the same arithmetic by hand. |
| OQ-709 | Payslip delivery. The UI assumes portal-only with an email notification carrying no figures. A PDF attachment would change this screen and conflict with 05 D-06. |
| OQ-718 | Does the approver need a sampling view — "show me 10 random payslips in full" — in addition to variance? Common in payroll practice, and cheap to add. |
| OQ-701 | Every screen here renders whatever components exist. Until the real ones are known, the layout is validated but the content is not. |
