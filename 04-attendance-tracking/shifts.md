# 04 — Attendance Tracking — Shifts & Work Schedules

Split file, as agreed in Step 1. This document owns everything about *what an employee was supposed
to do on a given day*. The main feature files own what they actually did.

## Why this is its own document

The shift is the yardstick. Every judgement the attendance engine makes — late, early, absent,
overtime, half day — is a comparison against a shift. If the shift for a date is wrong, the day is
wrong, and no amount of correct punch data saves it.

It is also the part most likely to grow: rotations, rosters, and swaps are each a feature in their
own right. Keeping them separate makes it obvious what is in v1 and what is deferred.

## Concepts, smallest to largest

| Concept | What it is |
|---|---|
| **Shift** | A named pattern for one day: start, end, breaks, grace, overtime rules. "Day shift, 09:00–17:00." |
| **Shift pattern** | A repeating sequence of shifts over N days, with rest days. "4 on days, 2 off, 4 on nights, 2 off." |
| **Assignment** | Which shift or pattern an employee follows, over a date range. |
| **Roster** | The resolved, per-employee, per-date answer — what the assignment produces for a given day. |
| **Override** | A one-off change for one employee on one date, without touching the assignment. |

The roster is **derived, not stored** in v1 (see D-S-04). Asking "what shift did Sara have on 3
March" is a function of her assignments and overrides, resolved on demand.

## Key decisions

| # | Decision | Rationale |
|---|---|---|
| D-S-01 | A shift stores **times of day**, not timestamps — `09:00`, `17:00` — interpreted in the company timezone on the date being computed. | A shift is "nine in the morning", which survives DST transitions. Storing an instant does not. |
| D-S-02 | A shift that ends before it starts (`22:00`–`06:00`) **crosses midnight**, and this is inferred, not configured. | One less field to get wrong. The engine needs the fact, not the admin's opinion of it. |
| D-S-03 | Assignments are **date-ranged and non-overlapping** per employee, exactly like feature 02's employment assignments. | Same reasoning: "which shift was she on in March" must stay answerable after she moves to another shift in April. |
| D-S-04 | The roster is **resolved on demand** from assignments, patterns, and overrides — not materialised into a row per employee per day. | 200 employees × 365 days is 73 000 rows a year that must be regenerated whenever an assignment changes. Resolution is cheap and always current. Revisit only if rostering becomes a planning tool where people edit future cells directly (OQ-S-03). |
| D-S-05 | An employee with **no shift assignment** falls back to feature 03's company work week, with `standardDailyHours` and no defined start time. | Office staff who simply work days should not need a shift assigned. A day with no start time cannot be "late" — see the classification rules. |
| D-S-06 | **Breaks are a fixed unpaid deduction per shift** in v1, not measured from punches. | OQ-403: if people do not scan for lunch, measuring is impossible; assuming is honest and predictable. Measured breaks are modelled as a future option, not built. |
| D-S-07 | **Grace is per shift**, not global. | A factory shift and an office shift rarely share a grace period, and a single global value guarantees an argument. |
| D-S-08 | **Overtime thresholds are per shift**, and overtime is *detected* here but *approved* elsewhere (OQ-405). | Detection is arithmetic; approval is policy. |

## Shift definition

A shift carries:

**Identity** — name, code, colour (for the roster and calendar views), active flag.

**Times** — start time, end time, and therefore whether it crosses midnight (D-S-02). A shift may
also be marked **flexible**: a required number of hours with no fixed start, for which lateness is
undefined and only total hours matter.

**Breaks** — total unpaid break minutes, and optionally a nominal window for display ("13:00–14:00").
Only the total affects computation in v1 (D-S-06).

**Grace** —
- *Late grace*: minutes after start that are not counted as late.
- *Early-departure grace*: minutes before end that are not counted as leaving early.
- *Minimum hours for a full day*, below which the day is a half day.
- *Minimum hours for a half day*, below which the day is absent despite punches. Someone who
  appears for twenty minutes has not worked half a day, and without this floor the classification
  says they have.

**Overtime** —
- Whether overtime is tracked at all for this shift.
- Minutes beyond the shift end before overtime starts accruing (a threshold, so that five minutes of
  tidying up is not overtime).
- Whether pre-approval is required (OQ-405).
- A daily cap, beyond which overtime is flagged for review rather than silently accrued — a
  twenty-hour overtime day is a data error far more often than it is a heroic effort.

**Day window** — how far either side of the shift a punch is still considered part of it. This is
what implements D-05 (shift-anchored days) and is explained below, because it is the subtlest thing
in the feature.

## The shift window, and which day a punch belongs to

The problem, concretely: a night shift runs 22:00–06:00. An employee scans in at 21:52 on 3 March
and out at 06:04 on 4 March. Both punches belong to **the 3 March shift**. A calendar-day bucketing
puts them on different days and produces two broken records: a day with no exit and a day with no
entry.

The rule:

> Each shift instance for date D opens a window from `start(D) − windowBefore` to
> `end(D) + windowAfter`, where `end` already accounts for crossing midnight. A punch belongs to the
> shift instance whose window contains it. Defaults: `windowBefore` 4 hours, `windowAfter` 4 hours.

Consequences that must be handled and are easy to miss:

- **Windows of consecutive days can overlap.** A punch at 06:30 could fall in the tail of the night
  shift that began yesterday and in the head of the day shift beginning today. Resolution rule:
  the punch belongs to the shift instance whose **start is closest and not in the future relative to
  the punch**, with an open session (an in-punch without an out-punch) taking precedence — someone
  who scanned in at 21:52 and has not scanned out is still on that shift.
- **A punch outside every window** belongs to no shift instance. It is stored (D-02), attached to the
  calendar day for visibility, and flagged `OUT_OF_WINDOW` rather than discarded. Someone coming in
  on their day off produces these, and it is real information.
- **Changing a shift's window retroactively changes which day past punches belong to.** Window
  changes are therefore treated like a shift edit: they mark affected days dirty and trigger a
  recompute (see `requirements.md`, FR-R).
- For employees on the work-week fallback (D-S-05), the window is the calendar day in company time,
  since there is no shift to anchor to. This is the one case where calendar-day bucketing is
  correct, and it is correct only because those employees never work nights by definition — if they
  do, they need a shift.

## Shift patterns and rotation

A pattern is an ordered list of entries, each being *a shift* or *a rest day*, plus a cycle length
and an anchor date from which the cycle counts.

```
Pattern "4-2 rotating"     cycle: 12 days   anchor: 2026-01-05
  day 1–4   Day shift
  day 5–6   Rest
  day 7–10  Night shift
  day 11–12 Rest
```

Resolution for a date: `index = (date − anchor) mod cycleLength`, then read the entry. Two employees
on the same pattern with different anchor dates are offset from each other, which is how rotating
crews are actually staffed — so **the anchor lives on the assignment, not on the pattern**. That
single placement decision is what makes one pattern serve four crews instead of needing four
near-identical patterns.

Patterns are optional. A simple assignment to a fixed shift, with the shift applying on the work
week's working days, covers most companies and is the default (OQ-S-01).

## Assignment

An assignment links an employee to either a shift or a pattern, over a date range:

- `effectiveFrom` (required), `effectiveTo` (null = current).
- Exactly one of `shiftId` or `patternId`; with a pattern, also a `cycleAnchorDate`.
- Non-overlapping per employee (D-S-03). Creating one that overlaps closes the previous one the day
  before, exactly as feature 02 does for job assignments — the same helper should be reused.
- Future-dated assignments are allowed and resolve from their date.
- Bulk assignment is required: "put these 40 people on the night shift from 1 October" must be one
  action, not forty. Anything else guarantees it is done in a spreadsheet instead.

## Overrides

A one-off, per employee, per date: a different shift, or a declared rest day, with a reason.
Overrides win over assignments and patterns. They exist because the alternative — splitting an
assignment into three to cover one swapped Saturday — is how shift history becomes unreadable.

Shift **swaps** between two employees are modelled as two overrides created together, with a shared
reference so the pair is visible as one event. The swap *request and approval* workflow is deferred
(OQ-S-02); v1 has HR creating the overrides.

## Resolution order

For (employee, date), the shift is the first of:

1. An **override** for that employee and date.
2. The **assignment** covering that date:
   - a pattern → resolve the cycle index from the assignment's anchor;
   - a fixed shift → the shift, but **only on days the work week marks as working**, unless the
     assignment says the shift applies every day.
3. The **work-week fallback** (D-S-05): working day with `standardDailyHours` and no start time.

Then, independently, feature 03's holiday calendar can turn the resolved day into a non-working day.
A holiday beats a shift; an employee who works anyway produces punches on a `HOLIDAY` day (D-09).

This order is the single source of truth and is exposed as one function:

```ts
resolveShift(employeeId, date): Promise<ResolvedShift>
type ResolvedShift = {
  source: "OVERRIDE" | "PATTERN" | "ASSIGNMENT" | "WORK_WEEK";
  shift: ShiftSnapshot | null;      // null = rest day / weekly off
  isWorkingDay: boolean;
  expectedHours: number;
  windowStart: Date; windowEnd: Date;   // UTC instants, shift-anchored
};
```

`ShiftSnapshot` is a **copy of the shift's rules as they were relevant to that date**, not a live
reference — which matters for the next section.

## Editing a shift that is already in use

Changing a shift's start time changes history: every past day computed against it would recompute
differently. Three options were considered:

1. **Versioned shifts** — edits create a new version; past days keep the old one. Correct, and
   expensive: every shift reference becomes a version reference.
2. **Immutable shifts** — edits are forbidden; create a new shift and reassign. Simple, and
   miserable to use.
3. **Mutable shifts with explicit recompute** — edits are allowed, the affected date range is
   surfaced, and the admin chooses whether to recompute history or only apply going forward.

**v1 takes option 3.** The UI states plainly what the edit affects ("2 480 computed days between 1
Jan and today use this shift"), and offers *Apply to future only* — implemented by closing the
current assignment and opening a new one against a copied shift — or *Recompute history*. Option 1
remains the upgrade path if shift edits turn out to be frequent (OQ-S-04).

This is the sort of decision that looks like an implementation detail and is in fact the difference
between a system HR trusts and one they keep a parallel spreadsheet for.

## Requirements

| ID | Requirement |
|---|---|
| FR-S-01 | A shift has a unique name and code, times, breaks, grace values, overtime rules, and window sizes. |
| FR-S-02 | A shift whose end time is before its start time is treated as crossing midnight (D-S-02). |
| FR-S-03 | A flexible shift has required hours and no fixed start; lateness and early departure are not computed for it. |
| FR-S-04 | Minimum-hours thresholds classify a short day as half day or absent, in that order. |
| FR-S-05 | Shift patterns define a cycle of shifts and rest days; the cycle anchor is on the assignment, not the pattern. |
| FR-S-06 | An employee has at most one assignment in force on any date; assignments never overlap. |
| FR-S-07 | Assignments may be created in bulk for a set of employees, a department, or a filtered list. |
| FR-S-08 | Overrides apply to one employee and one date and win over everything except holidays. |
| FR-S-09 | A swap creates two linked overrides in one transaction; either both apply or neither does. |
| FR-S-10 | `resolveShift` is the only implementation of the resolution order, and is used by the computation engine, the roster UI, and feature 06 (which needs to know whether a leave day was a working day for that person). |
| FR-S-11 | Editing a shift surfaces the number of computed days affected and requires a choice between applying forward and recomputing history. |
| FR-S-12 | Deleting a shift is refused while any assignment, override, or computed day references it; it may be deactivated. |
| FR-S-13 | Shift changes — create, edit, assign, override — are all audited, with the affected employee count recorded on bulk operations. |
| FR-S-14 | The roster view shows, for a date range and a group of employees, the resolved shift per day, including the source of the resolution (override, pattern, assignment, fallback). |
| FR-S-15 | Resolving a roster for 200 employees over a month must be a bounded number of queries — assignments, patterns, and overrides loaded once, then resolved in memory. Not one query per cell. |

## Permission keys

| Key | Meaning | Default holders |
|---|---|---|
| `shift.read` | View shifts, patterns, assignments, and the roster | HR: ALL · Manager: DEPARTMENT · Employee: SELF |
| `shift.write` | Create and edit shifts and patterns | HR: ALL |
| `shift.assign` | Assign employees to shifts; create overrides and swaps | HR: ALL · Manager: DEPARTMENT (see OQ-S-05) |

## Acceptance criteria

1. A night shift 22:00–06:00 assigned on 3 March collects a punch at 21:52 on 3 March and one at
   06:04 on 4 March into the 3 March record, with worked hours of 8:12.
2. The same employee's 4 March record is not created as a separate day with a stray exit punch.
3. A punch at 06:30, with an open session from the night shift, attaches to the night shift; with no
   open session, it attaches to the day shift starting at 07:00.
4. An employee on a 4-2 pattern with an anchor of 5 Jan resolves to a rest day on the correct dates,
   and a second employee on the same pattern with a different anchor is correctly offset.
5. An employee with no assignment on a Tuesday resolves to the work-week fallback with 8 expected
   hours and no start time, and is never classified late.
6. A holiday on a night-shift date makes the day a holiday; punches that day still record and the
   day is flagged as worked on a holiday.
7. Editing a shift's start time shows the count of affected computed days and does not silently
   recompute them.
8. Assigning 40 employees to a shift is one operation and produces one audit entry naming the count.
9. Resolving a 200-employee, 31-day roster issues a bounded number of queries, verified by a test
   that counts them.
10. Deleting a shift in use is refused with a message naming what references it.

## Open questions

| ID | Question |
|---|---|
| OQ-S-01 | Does the company actually use rotating patterns, or is every employee on one fixed shift? If the latter, patterns can be deferred entirely and the build shrinks noticeably. |
| OQ-S-02 | Should employees be able to *request* a shift swap, with manager approval, or is HR entering overrides sufficient for v1? |
| OQ-S-03 | Will anyone want to plan future rosters by editing cells in a grid (a rostering tool), rather than assigning shifts? That would reverse D-S-04 and is a significantly larger feature. |
| OQ-S-04 | How often do shift definitions change? Frequent edits would justify versioned shifts (option 1 above) over the v1 approach. |
| OQ-S-05 | Should managers be able to assign shifts and create overrides for their own team, or is that HR-only? Affects the default grant for `shift.assign`. |
| OQ-S-06 | Are there minimum rest requirements between shifts (e.g. 11 hours) that the system should warn about when assigning? Common in regulated industries; pure validation, but it needs the rule stated. |
| OQ-403 | Whether breaks are scanned or assumed — D-S-06 assumes the latter. If they are scanned, multi-session pairing and measured breaks both become necessary. |
