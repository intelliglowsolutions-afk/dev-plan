# 06 — Leave Management — Requirements

## Actors

| Actor | What they do here |
|---|---|
| Employee | Checks their balance, requests leave, cancels it, sees where a request has got to. |
| Manager | Approves or rejects their team's requests, sees who is off when, delegates while away. |
| HR admin | Configures types and policies, adjusts balances, overrides approvals, runs accrual and year-end, fixes mistakes. |
| Super admin | Balance adjustments that HR should not make unsupervised, and edits to locked periods. |
| The accrual engine | Grants, carries over, and expires entitlement on a schedule. Like 04's computation engine, it writes more than every human combined. |

## User stories

### Employee

- **US-01** — As an employee, I see how much leave I have left, of each type, and how that number was
  arrived at.
- **US-02** — As an employee, I request leave by picking dates, and I am told before I submit exactly
  how many days it will cost and what my balance will be afterwards.
- **US-03** — As an employee, I take a half day.
- **US-04** — As an employee, I attach a sick note.
- **US-05** — As an employee, I see who is considering my request and how long it has been waiting.
- **US-06** — As an employee, I cancel a request — before or after approval — and my balance returns.
- **US-07** — As an employee, I see my team's booked leave before I choose dates, so I do not request
  the week my colleague is already off.

### Manager

- **US-08** — As a manager, I am told when a request needs me, and I can approve or reject with a
  reason from one screen.
- **US-09** — As a manager, I see the requester's balance, their recent leave, and who else on the
  team is off on those dates, without leaving the approval screen.
- **US-10** — As a manager, I delegate my approvals while I am away, so requests do not stall.
- **US-11** — As a manager, I see a calendar of my team's leave, including pending requests, so I can
  plan.

### HR

- **US-12** — As HR, I define leave types and the policies that govern them.
- **US-13** — As HR, I assign different policies to different groups — a longer annual entitlement
  for senior staff, a different one for contractors.
- **US-14** — As HR, I adjust someone's balance with a reason, and the adjustment is visible forever.
- **US-15** — As HR, I run the year-end carry-over, see exactly what it will do to every balance
  before it does it, and then apply it.
- **US-16** — As HR, I see everyone's balances, filter by who has too much left, and chase them.
- **US-17** — As HR, I approve or reject on a manager's behalf when something is stuck.
- **US-18** — As HR, I record leave for someone who does not use the system.

## Functional requirements

### Leave types and policies — FR-T

| ID | Requirement |
|---|---|
| FR-T-01 | A leave type has a name, code, colour, and flags: paid or unpaid, deducts from a balance or not, requires approval or not, allows half days, allows attachments, counts toward attendance as present. |
| FR-T-02 | A type that does not deduct from a balance (unpaid leave, bereavement in some companies) still produces requests, approvals, and attendance classification — it simply writes no debit to the ledger. |
| FR-T-03 | A policy attaches to a type and defines: annual entitlement, accrual method, accrual frequency, carry-over allowance and cap, carry-over expiry, probation restriction, notice period, maximum consecutive days, negative-balance tolerance, and pro-rating rules. |
| FR-T-04 | Policies are assigned to employees by rule — employment type, department, position grade — with an explicit per-employee override. Every employee resolves to exactly one policy per type, or to none, which means the type is unavailable to them. |
| FR-T-05 | Policy resolution is date-aware: a policy change applies from its effective date and does not retroactively alter entitlement already granted. |
| FR-T-06 | Deactivating a type hides it from new requests and leaves all history intact. Types are never deleted while ledger entries reference them. |
| FR-T-07 | A type may restrict who can see it. Some types (unpaid disciplinary, medical) should not appear in a team calendar by name — the calendar shows "on leave" without the type (see FR-C-05). |
| FR-T-08 | Changing a policy's entitlement mid-year does not silently adjust balances. It surfaces the affected employees and the difference, and the adjustment is applied deliberately, as ledger entries with a reason. |

### The ledger and balances — FR-B

| ID | Requirement |
|---|---|
| FR-B-01 | Every change to a balance is an immutable ledger entry: employee, type, period, amount (signed), kind, source, reason, actor, timestamp (D-01). |
| FR-B-02 | Entry kinds: `GRANT`, `ACCRUAL`, `CARRY_OVER`, `CARRY_OVER_EXPIRY`, `RESERVATION`, `RESERVATION_RELEASE`, `TAKEN`, `REFUND`, `ADJUSTMENT`, `ENCASHMENT`, `FORFEIT_ON_EXIT`. |
| FR-B-03 | A balance is the sum of entries for (employee, type, leave year). It is never stored as a mutable field; a cached snapshot exists for query speed and is rebuilt from the ledger, never written to directly. |
| FR-B-04 | The balance is reported in three parts, because collapsing them is what causes disputes: **entitled**, **used**, **pending** (reserved but not yet taken), and **available** = entitled − used − pending. |
| FR-B-05 | Ledger entries are never deleted or edited. A mistake is corrected by a compensating entry that references the original. |
| FR-B-06 | Every entry carries enough context to explain itself in the UI: "Carried over from 2025, expires 31 March" is a property of the entry, not a guess made at render time. |
| FR-B-07 | Entries are tied to a **leave year** so that carry-over, expiry, and year-end reporting are unambiguous. The leave year's boundaries come from policy (OQ-602). |
| FR-B-08 | A cached balance that disagrees with its ledger is a defect. A scheduled consistency check recomputes snapshots and reports discrepancies rather than silently correcting them. |

### Requests — FR-R

| ID | Requirement |
|---|---|
| FR-R-01 | A request has: employee, type, start date, end date, half-day flags for the first and last day, reason, optional attachment, and a computed day count. |
| FR-R-02 | The day count is computed from **working days** for that employee — 03's `isWorkingDay`, which already accounts for the work week, holidays, and (via 04) their shift (D-04). Weekends and holidays inside a range cost nothing. |
| FR-R-03 | The cost is recomputed at approval. If it has changed — a holiday was declared in the interim — the approver is shown both numbers and must confirm (D-04). |
| FR-R-04 | Half days are supported at the start and end of a range, where the type allows (FR-T-01). A half day costs 0.5 regardless of shift length (D-03). |
| FR-R-05 | Submitting reserves the days: a `RESERVATION` ledger entry (D-05). Rejection, cancellation, or withdrawal writes `RESERVATION_RELEASE`. Approval converts the reservation to `TAKEN`. |
| FR-R-06 | Validation at submission: sufficient available balance (unless the policy tolerates negative), within maximum consecutive days, satisfies notice period, not overlapping another request of the employee's, dates within employment, and the type is available to them. |
| FR-R-07 | Overlapping requests are rejected outright. Two approved leaves on one day is a state nothing downstream can interpret. |
| FR-R-08 | Backdated requests are allowed with a reason (OQ-612), except into a payroll-locked period, which is refused with the same error and the same suggested route as attendance corrections (04 FR-X-09). |
| FR-R-09 | A request may be withdrawn by the requester while pending, and cancelled after approval — subject to the same period lock. Cancellation writes a `REFUND` entry and marks the attendance days dirty. |
| FR-R-10 | Cancelling leave that is partly in the past refunds only the future portion by default, with an explicit option to refund the whole thing. Someone who took Monday and Tuesday and cancels on Tuesday afternoon did take Monday. |
| FR-R-11 | An approved request is not edited (D-07). The UI offers "cancel and rebook", and an `AMENDMENT` path for HR that writes its own compensating entries. |
| FR-R-12 | HR may record leave on an employee's behalf (US-18); it is attributed to HR as the creator, with the employee as the subject, and is auto-approved with that fact recorded. |
| FR-R-13 | A request produces one `LeaveRequestDay` row per working day covered, each with its fraction. This is what attendance, payroll, and the calendar read — none of them re-derive the range. |

### Approval — FR-A

| ID | Requirement |
|---|---|
| FR-A-01 | The approval chain is resolved at submission from the policy and the employee's reporting line, and stored as ordered steps (D-06). |
| FR-A-02 | A step names an approver and, where relevant, a fallback. Steps are decided in order; a rejection at any step ends the request. |
| FR-A-03 | Rejection requires a reason, which is shown to the requester (consistent with 04 FR-X-08). |
| FR-A-04 | An approver cannot approve their own request. Where the chain resolves to the requester — a manager requesting leave — it escalates to their own manager. |
| FR-A-05 | HR with `leave.override` may approve or reject at any step, recorded as an override with a reason, not as a normal approval. |
| FR-A-06 | Delegation: an approver may nominate a delegate for a date range. Requests arriving in that window route to the delegate, and the record shows both names — "approved by X, delegated by Y" (OQ-607). |
| FR-A-07 | Escalation: a request pending beyond the configured threshold notifies the approver's manager and becomes approvable by them. It is never auto-approved. Silence is not consent. |
| FR-A-08 | Types configured as not requiring approval are auto-approved at submission, with the ledger and attendance effects applied immediately. |
| FR-A-09 | Approving notifies the requester; a decision on one step notifies the next approver (05). |
| FR-A-10 | When a request is decided, any notification waiting on another approver is superseded (05's `supersede`). |

### Calendar and clashes — FR-C

| ID | Requirement |
|---|---|
| FR-C-01 | A team calendar shows approved and pending leave by employee and date, for the actor's scope. |
| FR-C-02 | Pending leave is visually distinct from approved. A manager planning around leave that might not happen needs to know which is which. |
| FR-C-03 | Clash detection: when a request overlaps others in the same team, the count is shown to the requester before submitting and to the approver when deciding. It warns; it does not block (OQ-611). |
| FR-C-04 | The calendar shows holidays and weekends from 03, so a range that looks long is visibly mostly non-working. |
| FR-C-05 | Restricted types (FR-T-07) appear as "On leave" without the type name to anyone but HR and the employee. |
| FR-C-06 | The calendar is exportable, and readable by month, by week, and as a list. |

### Accrual and year-end — FR-Y

| ID | Requirement |
|---|---|
| FR-Y-01 | The accrual engine grants entitlement per policy on a schedule, writing `GRANT` or `ACCRUAL` entries keyed by (employee, type, period) — so a second run for the same period writes nothing (D-08). |
| FR-Y-02 | Pro-rating: an employee who joins or leaves mid-period receives entitlement proportional to their employment (02's employment periods), rounded per policy. |
| FR-Y-03 | Year-end carry-over computes each employee's remainder, applies the cap, writes `CARRY_OVER` into the new year, and schedules `CARRY_OVER_EXPIRY` for the expiry date. |
| FR-Y-04 | Expiry runs on its date and forfeits unused carried-over days, as an entry with a reason — not by deleting the original grant. |
| FR-Y-05 | Every engine run supports **preview**: the full list of employees and the entries that would be written, before anything is written (D-08, and the same pattern as 04's recompute preview). |
| FR-Y-06 | Runs are recorded: who ran it, when, over which period, how many employees, how many entries. A run can be traced from any entry it produced. |
| FR-Y-07 | A run that fails partway leaves no partial state; it is one transaction, or per-employee transactions with a resumable record of which employees completed. |
| FR-Y-08 | Termination triggers a settlement: remaining balance is encashed or forfeited per policy (OQ-609), as explicit entries, surfaced to HR rather than applied silently. |

### Integration — FR-X

| ID | Requirement |
|---|---|
| FR-X-01 | Approving, cancelling, or amending leave marks the affected (employee, date) pairs dirty via 04's `markDirty` (D-09, 04 FR-R-03). This feature never writes to `attendance_days`. |
| FR-X-02 | 04's classification reads `LeaveRequestDay` to produce `ON_LEAVE`, and a half-day leave plus a half day worked produces `HALF_DAY` with a leave reference (04 FR-C-12). |
| FR-X-03 | **A bulk recompute of attendance over the period since attendance went live is part of this feature's release** (04 OQ-410). Days that computed `ABSENT` for want of leave data must be corrected before anyone reports on them. |
| FR-X-04 | Payroll reads `LeaveRequestDay` and the ledger for unpaid-leave deductions and encashment. This feature exposes them; it does not compute money. |
| FR-X-05 | A holiday declared over approved leave does not auto-refund (D-10). 03 surfaces the impact; HR performs the refund as an explicit action which writes a `REFUND` entry. |
| FR-X-06 | Notifications for submission, decision, escalation, delegation, and expiry-approaching go through 05, with the type keys registered in its catalogue. |

## Non-functional requirements

| ID | Requirement |
|---|---|
| NFR-01 | A balance query must not scan the whole ledger. Snapshots are maintained per (employee, type, year) and rebuilt on write. |
| NFR-02 | The working-day cost of a request is computed with 03's **batched** helper, not a call per day (03 NFR-02). |
| NFR-03 | The team calendar for 200 employees over a month is a bounded number of queries, reading `LeaveRequestDay`, not deriving ranges. |
| NFR-04 | Reservation and approval must be atomic against concurrent requests: two requests submitted simultaneously against a balance of 2 must not both succeed. Enforced by a transaction and a balance check inside it, not by a read-then-write. |
| NFR-05 | Accrual for 200 employees completes in under a minute and is safe to interrupt. |
| NFR-06 | Attachments follow feature 02's document rules — volume storage, content sniffing, permission-checked download, never a static path. |
| NFR-07 | Leave reasons and medical attachments are sensitive. They are visible to the employee, their approvers, and HR — not to anyone with plain `leave.read`. |
| NFR-08 | All date handling uses 03's timezone helpers. A leave day is a date, not a 24-hour span. |

## Permission keys added by this feature

| Key | Meaning | Default holders |
|---|---|---|
| `leave.read` | View leave requests and the calendar | HR: ALL · Manager: DEPARTMENT · Employee: SELF |
| `leave.request` | Create a request for yourself | Everyone: SELF |
| `leave.request_for_others` | Record leave on someone's behalf | HR: ALL |
| `leave.approve` | Decide requests routed to you | HR: ALL · Manager: DEPARTMENT |
| `leave.override` | Decide out of chain, amend approved requests | HR: ALL |
| `leave.balance.read` | View balances | HR: ALL · Manager: DEPARTMENT · Employee: SELF |
| `leave.balance.adjust` | Write adjustment entries | HR: ALL |
| `leave.policy.read` / `.write` | View / configure types and policies | HR: ALL |
| `leave.run_accrual` | Run accrual, carry-over, expiry | HR: ALL |
| `leave.delegate` | Nominate a delegate for your approvals | Manager, HR: SELF |

## Acceptance criteria (feature-level)

1. A request from Thursday to the following Tuesday, with a public holiday on the Monday, costs
   3 days — and the UI says so before submission.
2. An employee with 2 days available who submits two 2-day requests has the second rejected for
   insufficient balance, including when both are submitted concurrently.
3. Submitting reserves; rejecting releases; the balance returns to its prior value exactly.
4. Approving converts the reservation to taken, and the affected attendance days reclassify to
   `ON_LEAVE` without anything writing to `attendance_days` directly.
5. Cancelling approved leave that is half in the past refunds only the future days by default.
6. A manager's own request escalates to their manager rather than to themselves.
7. A request pending past the threshold notifies the approver's manager and is never auto-approved.
8. With a delegate active, requests route to the delegate and the record names both people.
9. Running the annual grant twice for the same year writes entries once.
10. Year-end preview lists every employee's carry-over, the cap applied, and the expiry date, before
    anything is written.
11. A carried-over remainder is forfeited on its expiry date by an entry, with the original grant
    untouched.
12. A balance disputed by an employee can be explained as a dated list of entries that sums to the
    displayed number.
13. The bulk attendance recompute (FR-X-03) runs as part of this release and converts previously
    `ABSENT` days to `ON_LEAVE` where leave exists.
14. A restricted leave type shows as "On leave" on a colleague's calendar, with no type name.
