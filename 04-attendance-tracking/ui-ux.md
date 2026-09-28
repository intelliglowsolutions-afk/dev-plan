# 04 — Attendance Tracking — UI / UX

Builds on the shell and principles in [01 ui-ux.md](../01-roles-permissions-auth/ui-ux.md).

## Principles specific to this feature

1. **Every judgement must be explainable in one click.** The system tells someone they were late.
   That claim has to survive being questioned, and HR must be able to answer without understanding
   shift windows or pairing. The day detail exists for exactly this.
2. **Never present a guess as a fact.** `UNKNOWN` looks different from `ABSENT`, a corrected value
   looks different from a computed one, and a pending correction is visible on the day.
3. **The absent list is an accusation.** Four names on a screen saying "absent" leads to four
   conversations. It must be right, and where it might not be — a device outage, a pending
   correction, a leave request not yet approved — the screen must say so before HR acts on it.
4. **HR's day starts on one screen.** The daily grid answers "who is not here" in five seconds, or
   it will be replaced by a WhatsApp group.

## Screen map

```
/attendance                        Daily grid — the landing screen        attendance.read
/attendance/employee/:id           One employee, one month                attendance.read
/attendance/day/:id                One day, fully explained               attendance.read
/attendance/punches                Raw punch log (diagnostic)             attendance.read
/attendance/corrections            Corrections queue                      attendance.read
/attendance/unmatched              Unmatched PINs                         attendance.manage_unmatched
/attendance/gaps                   Device outages                         attendance.read
/attendance/recompute              Recompute with preview                 attendance.recompute
/shifts                            Shifts and patterns                    shift.read
/shifts/assignments                Assignments                            shift.read
/roster                            Roster grid                            shift.read
```

## `/attendance` — the daily grid

**Top: the summary strip.** Large counts, each a filter: Present · Late · Absent · On leave ·
Missing punch · Unknown · Not scheduled. This is the five-second answer.

Above it, when relevant, a **warning band** — these are the point of the screen, not decoration:

> ⚠ **Main gate has been offline since 06:04.** 148 people have no punches today. Their days are
> marked *Unknown*, not absent. *(View device)*

> ⚠ **12 days are waiting to be computed.** Figures below may be incomplete. *(Details)*

> ⚠ **3 unmatched PINs** have punched today — someone's code may be wrong. *(Review)*

**The table:** employee (photo, name, code), department, shift, in, out, worked, late, overtime,
status, and a flag column for corrections and pending requests.

Status is a badge with text. `UNKNOWN` renders visually distinct from `ABSENT` — grey and
outlined, with the word *Unknown* and a tooltip: "No device was reporting during this shift. This is
not an absence." If HR takes one thing from this screen, it should be that distinction.

Rows with corrections carry a small marker; hovering shows who corrected it and why. Rows with a
pending request are marked so nobody chases a day that is already being disputed.

**Controls:** date picker (defaulting to today, with *yesterday* and *this week* shortcuts),
department, shift, and status filters, and a text search. Managers see their team; the filter row
says so — "Showing your team (14 people)" — rather than leaving them to wonder why the count is low.

**Live-ish:** the grid refreshes on an interval during working hours. A "last updated 30 seconds ago"
line, with manual refresh. Not a websocket in v1; the data is minutes-fresh by nature since the
device pushes periodically, and pretending to be real-time would misrepresent it.

**Empty states:** before the day opener has run, "Today's attendance has not been prepared yet" with
the expected time — never an empty table that reads as "nobody came to work".

## `/attendance/employee/:id` — one person, one month

A calendar month grid, one cell per day: status colour, in/out times, hours. Weekends and holidays
shaded from the work week and holiday calendar, so a blank weekend never looks like missing data.

Beneath it, the same month as a table for detail and export, plus a summary bar: days present, days
absent, total hours, overtime, late count, average in-time. That last one is the number managers
actually ask for.

Clicking a day opens the day detail. A *Request correction* button on each day for the employee
themselves (feature 08 renders the same component).

Month navigation, and a shift row showing which shift applied across the month — a mid-month shift
change is otherwise invisible and explains half the anomalies on the screen.

## `/attendance/day/:id` — the explanation

The most important screen in the feature, and the one most likely to be skipped as "just a detail
page". It answers "why does it say this".

**Verdict** — status, worked hours, late minutes, overtime, prominently.

**Expected** — the shift, where it came from ("From her assignment to *General shift*, effective 1
Feb"), its start and end for this date, grace, and expected hours.

**What happened** — a timeline of the punches: time, device, how each was used (first in, last out,
ignored as duplicate, outside the window). Ignored punches are shown greyed **with the reason** —
"ignored: 20 seconds after the previous punch" — because an unexplained missing punch is exactly
what makes people distrust the system.

**How it was decided** — the reason trail in plain language:

> - Shift *General shift* applied (from her assignment)
> - 15 September is a working day
> - One session paired: 09:14 → 17:03
> - Arrived 14 minutes after 09:00; grace is 10 minutes → **Late**

**Corrections** — any applied, with before, after, who, when, and why. And the actions: *Correct
this day* (HR) or *Request a correction* (employee).

If the day is `UNKNOWN`, this screen leads with the device outage and its times, not with the
employee.

## `/attendance/corrections` — the queue

A work list, newest first: employee, date, what is being changed (computed → requested), reason,
requester, age. Filters by status, department, date range.

Each row expands to show the day's detail inline — an approver should not have to open another
screen to judge a request, or they will approve without looking.

Approve and reject are per row; reject requires a reason and says it will be shown to the requester.
Bulk approve exists for the common "twelve people affected by the same outage" case, and requires
selecting rows deliberately — no "approve all" button.

Pending requests older than a few days are highlighted: a correction queue that silently grows is
a payroll problem waiting to happen.

## `/attendance/unmatched` — unmatched PINs

A short list, and an important one: PIN, punch count, first and last seen, device, and a *Assign to
employee* picker plus *Not a person*.

The panel explains itself, because this screen's whole purpose is to be understood by someone who
has never thought about PINs:

> These PINs were scanned at a terminal but match no employee code. Usually this means an employee's
> code was typed differently in the system than on the device — their attendance is being recorded
> but not counted. Assigning the PIN attaches all past punches and recomputes those days.

Assigning shows what will happen first: "This will attach 47 punches from 12 Aug to today and
recompute 23 days for Ayesha Khan."

When the chosen employee's code does not match the PIN, the dialog says so and suggests fixing the
employee's code instead — the correct fix in nearly every case.

## `/attendance/gaps` — device outages

One card per gap: device, start, end (or "ongoing"), duration, affected employee-days, and how many
days are currently `UNKNOWN`.

Resolution offers three honest options, worded as they are:

- **Mark everyone present** — for a known outage during normal working hours. Requires a reason,
  creates one bulk correction, and states the count.
- **Ask employees to confirm** — notifies the affected employees to request corrections.
- **Leave as unknown** — with a note. Recorded, so the gap stops nagging without being falsified.

The first option carries a caution: "This marks 148 people present for 14 September without punch
data. Only do this if you know the office was open and staffed."

## `/attendance/recompute`

Deliberately plain and a little slow. Pick a date range, optionally employees or a department, and
press **Preview changes** — which is the primary button. *Apply* is disabled until a preview has
run.

The preview is a table of would-change days: employee, date, from → to, and the trigger. A summary
line reads "30 days examined, 4 would change". When nothing would change, it says exactly that,
which is reassuring rather than disappointing.

Locked payroll periods appear as an explicit list — "September 2026 is locked and was skipped" —
never as a silently smaller result.

Applying queues the work and shows queue progress, with a note that changes appear as the worker
drains the queue.

## `/shifts` and `/roster`

**`/shifts`** — a list: name, times (with a "crosses midnight" marker), break, grace, overtime,
assigned employee count. The editor is a form with a **live preview**: "09:00–17:00, 1 hour break →
7 hours expected. Arriving after 09:10 is late. Leaving before 16:50 is early."  Numbers like
`fullDayMinHours` are meaningless to most users until shown as consequences.

Editing a shift already in use shows the impact panel before saving (`shifts.md` FR-S-11), with the
two options as full sentences rather than radio labels:

> This shift is used by 34 people and 2 480 computed days since 1 January.
> - **Apply to future only** — past attendance keeps the old times. (Recommended)
> - **Recompute history** — past days are recalculated with the new times. Attendance reports for
>   previous months may change.

**`/roster`** — employees down the side, dates across the top, a coloured cell per shift with rest
days blank and holidays marked. Cells show their source on hover; overrides are outlined so a
one-off change is visible at a glance.

Actions: bulk-assign a shift to selected employees over a date range, create an override, create a
swap. Selection is by checkbox with a "select all filtered" that names the count — bulk shift
assignment is the operation most likely to be done to the wrong forty people.

Below tablet width the roster becomes a per-employee list; a 31-column grid on a phone is not worth
attempting.

## States, accessibility, responsive

- Six states everywhere; plus the attendance-specific one: **computation pending**, which must never
  be rendered as zero or absent.
- Status is never colour alone — every badge carries its word, and `UNKNOWN` differs from `ABSENT`
  in shape as well as colour.
- The month calendar has a table equivalent, always available, not only on mobile.
- Times are shown in company time with the timezone named in the header, since a reader's device may
  be elsewhere.
- The daily grid is the highest-traffic screen: it must be keyboard-navigable and fast, with the
  date picker reachable without a mouse.

## Copy reference

| Situation | Text |
|---|---|
| Unknown status tooltip | No device was reporting during this shift. This is not an absence. |
| Device offline band | {Device} has been offline since {time}. {n} people have no punches today. Their days are marked Unknown, not absent. |
| Pending computation | {n} days are waiting to be computed. Figures below may be incomplete. |
| Unmatched PIN explainer | These PINs were scanned at a terminal but match no employee code. Their attendance is being recorded but not counted. |
| Assign PIN impact | This will attach {n} punches from {date} to today and recompute {m} days for {name}. |
| Code mismatch | This employee's code is {code}, not {pin}. If the device is right, change their employee code instead. |
| Missing punch | Scanned in at {time} but never out. No hours are counted until this is corrected. |
| Ignored punch | Ignored: {n} seconds after the previous punch. |
| Bulk present caution | This marks {n} people present for {date} without punch data. Only do this if you know the office was open and staffed. |
| Period locked | {Month} payroll was finalised on {date}. This day cannot be changed. Create a payroll adjustment instead. |
| Shift edit impact | This shift is used by {n} people and {m} computed days since {date}. |
| Recompute, no changes | 30 days examined. Nothing would change. |
| Manager scope | Showing your team ({n} people). |

## Open questions

| ID | Question |
|---|---|
| OQ-418 | Should the daily grid auto-refresh, and how often? Too frequent is noisy and costly; too rare and HR keeps reloading. Proposed: 60 seconds during configured working hours, manual otherwise. |
| OQ-419 | Does HR want a live "who is in the building right now" view, distinct from today's attendance? Different question, different screen — and genuinely useful for fire-safety roll calls. |
| OQ-420 | Should employees see their own lateness trend over months? Motivating for some, corrosive for others; worth asking rather than assuming. Relates to feature 08. |
| OQ-406 | Missing-punch handling drives several screens here. If the answer changes to "assume shift end", the day detail and corrections queue both simplify — and become less honest. |
