# 06 — Leave Management

**Priority:** Must-have · **Build order:** 6 of 11 · **Status:** planned (Step 3)

## Purpose

Track what leave each person is entitled to, what they have used, what they have left, and route
requests to whoever decides. Then make sure attendance and payroll agree with the answer.

Leave is where an HRM earns or loses trust fastest. Attendance can be argued about; a leave balance
is a number the employee believes they own, and if the system says 11.5 and they think it is 13,
they stop believing everything else it says too.

## The central problem

A balance is not a field. It is the result of a history: an annual grant, a carried-over remainder
that expires in March, three days taken, one request pending, a half-day that fell on a public
holiday and was refunded. Store the balance as a number and every one of those events is a chance to
corrupt it silently.

So this feature uses a **ledger**. Every movement is an immutable entry — granted, accrued, taken,
cancelled, expired, adjusted — and the balance is their sum. The same split as feature 04's punches
and computed days, for the same reason: keep the evidence, derive the answer.

## Scope

**In scope**

- **Leave types**: annual, sick, unpaid, maternity, bereavement, and whatever else the company has.
  Fully configurable; no built-in statutory rules (consistent with the payroll decision from Step 1).
- **Policies**: entitlement, accrual method, carry-over and its expiry, probation rules,
  pro-rating for mid-year joiners and leavers, negative-balance tolerance, notice requirements.
- **The ledger** and the balances derived from it.
- **Requests**: date ranges, half-days, attachments, the working-day count, and what they cost.
- **Approval**: routing, multi-step chains, delegation when an approver is away, cancellation and
  amendment after approval.
- **Team calendar** and clash detection.
- **Year-end**: accrual runs, carry-over, expiry — all explicit and previewable.
- The handshake with attendance (a day becomes `ON_LEAVE`) and with payroll (unpaid leave, and
  encashment if it exists).

**Out of scope (owned elsewhere)**

- Whether a date is a working day → [03](../03-admin-settings/README.md)'s `isWorkingDay`. This
  feature never counts calendar days and never re-implements the holiday logic.
- Which shift someone was on → [04](../04-attendance-tracking/README.md)'s `resolveShift`. A leave
  day's value depends on what they would otherwise have worked.
- Writing attendance records → 04. This feature marks days dirty; 04's engine reclassifies them.
  Nothing here writes to `attendance_days` (04 FR-R-12).
- Paying or deducting → [07](../07-payroll/README.md). This feature says a day was unpaid leave;
  payroll decides what that costs.
- Telling people → [05](../05-notifications/README.md). This feature calls `notify()`.
- The employee's own leave screens → [08](../08-employee-self-service/README.md), which renders the
  components specified here.

## Files

| File | Contents |
|---|---|
| [requirements.md](./requirements.md) | User stories, functional and non-functional requirements, policy mechanics, permission keys |
| [data-model.md](./data-model.md) | Prisma models, the ledger, balance derivation, seed and migration notes |
| [api-design.md](./api-design.md) | Requests, approvals, balances, the accrual and year-end engines |
| [ui-ux.md](./ui-ux.md) | Request flow, approvals queue, team calendar, admin policy screens |

## Key decisions taken in this plan

| # | Decision | Rationale |
|---|---|---|
| D-01 | **Balances are a ledger, never a stored counter.** Every movement is an immutable entry; the balance is their sum, with a cached snapshot for speed. | A mutable counter has no history, cannot explain itself, and drifts. When an employee disputes a balance, the answer must be a list of dated entries, not "the system says 11.5". |
| D-02 | **Leave types and policies are fully configurable**, with no statutory rules built in. | Matches the Step 1 decision on payroll. Country rules change and differ; encoding them makes the product wrong somewhere. |
| D-03 | Leave is measured in **days with half-day granularity**, not hours. | Everyone talks about leave in days. Hours are needed only where shift lengths vary within one person's week — see OQ-603, which is the case that would reverse this. |
| D-04 | The **cost of a request is computed from working days** via 03's helper, per employee, at the moment of submission — and **re-computed at approval**. | A request spanning a weekend and a public holiday costs three days, not seven. Re-computing at approval catches a holiday declared in between. |
| D-05 | Submitting a request **reserves** the days against the balance immediately. | Otherwise someone with 2 days left submits three separate 2-day requests and all three pass validation. The reservation is a pending ledger entry, released on rejection or cancellation. |
| D-06 | The **approval chain is resolved and stored at submission**, not looked up at each decision. | A reorganisation mid-approval must not silently redirect a request that someone is already looking at. |
| D-07 | An approved request is **not edited**. Changes are a cancellation plus a new request, or an explicit amendment that writes its own ledger entries. | Editing an approved request breaks the ledger's correspondence with reality and makes payroll's view of the month unstable. |
| D-08 | **Accrual and year-end are explicit, idempotent, previewable jobs**, keyed by period. Running twice grants nothing twice. | A double-run that silently doubles everyone's entitlement is discovered months later, by which point people have spent it. |
| D-09 | **Leave never writes to attendance.** It marks days dirty and lets 04's engine reclassify. | One writer per table (04 FR-R-12). Two features writing attendance is how a recompute and a leave approval start fighting. |
| D-10 | **A holiday declared over already-approved leave does not silently refund it.** The impact is surfaced (03 FR-W-08) and the refund is a deliberate, audited action. | 03's holiday screen already promises this. Automatic refunds would rewrite balances people have planned around. |

## What this feature owes feature 04

Feature 04 was built first, so `ON_LEAVE` classification had nothing to read. Two obligations fall
due here (04 OQ-410):

1. **A bulk recompute** over the period since attendance went live, so days that computed as
   `ABSENT` become `ON_LEAVE` where leave exists. This is part of this feature's release, not a
   follow-up.
2. **The dirty-day trigger**: approving, cancelling, or amending leave marks the affected
   (employee, date) pairs dirty through 04's shared `markDirty` helper (04 FR-R-03).

Both are listed in `requirements.md` as requirements, not notes, so they cannot be quietly skipped.

## Dependencies

- **Depends on:** 01 (permissions, audit), 02 (employees, managers for routing, employment periods
  for pro-rating), 03 (`isWorkingDay`, company timezone, job runner, settings), 04 (`resolveShift`,
  `markDirty`), 05 (`notify`).
- **Depended on by:** 07 (unpaid leave and encashment feed pay), 08 (self-service), 11 (reports).
- This is the most connected feature in the system: it reads from four and writes to none of them
  directly. That constraint is deliberate and is what keeps it buildable.

## Open questions

| ID | Question | Why it matters | Proposed default |
|---|---|---|---|
| OQ-601 | **What leave types does the company have, and what is the entitlement for each?** | Everything else is mechanism. Without this the policies cannot be seeded and the feature cannot be tested against reality. | Ask before building; the plan ships with examples clearly labelled as examples. |
| OQ-602 | **Accrual method**: a lump sum granted annually, monthly accrual, or accrual per hours worked? And on what year — calendar, fiscal, or employment anniversary? | Changes the accrual engine and the year-end run. Anniversary-based accrual means every employee has a different year boundary, which is materially more work. | Annual grant on the calendar year, pro-rated for joiners. |
| OQ-603 | Do shift lengths vary enough that leave must be measured in **hours** rather than days? | Reverses D-03. Someone on a 12-hour shift taking "a day" is not the same as someone on 8. | Days, with half-days. Revisit if OQ-401 (night shifts) comes back yes with varying lengths. |
| OQ-604 | **Carry-over**: allowed? capped? expiring when? | The most common source of balance disputes. Every variant is easy to build and impossible to guess. | Carry-over allowed, capped at 5 days, expiring 31 March. |
| OQ-605 | Can balances go **negative** (leave taken in advance), and up to what limit? | Affects validation and payroll recovery on termination. | No negative balances in v1. |
| OQ-606 | **Approval chain**: manager only, or manager then HR? Does it vary by leave type or by length (a day vs three weeks)? | Determines whether the chain model needs to be conditional, which is a real step up in complexity. | Single-step, the employee's manager, with HR able to override. |
| OQ-607 | What happens when the approver is **absent**? Auto-escalate after N days, delegate, or fall to their manager? | Leave requests stall during exactly the period when approvers are themselves on leave. Nothing else in the system has this circularity. | Explicit delegation, plus escalation to the approver's manager after 3 working days. |
| OQ-608 | Does **sick leave require a document** after N days, and does the system enforce it? | Cheap to support, contentious to enforce. | Attachment optional, prompted after 2 consecutive days. |
| OQ-609 | Is unused leave **encashed** on termination or at year-end? | Payroll integration; also affects whether the ledger needs an encashment entry type. | Encashment on termination only; the ledger supports it either way. |
| OQ-610 | Is there **comp-off / time in lieu** for overtime or holiday working? Feature 04 records both. | A natural pairing, and a whole sub-feature: earning leave rather than being paid. | Not in v1; the ledger's entry types leave room. |
| OQ-611 | **Minimum staffing** — should the system block or merely warn when too many of a team are off at once? | Blocking needs a rule per team and will be wrong sometimes; warning is nearly free. | Warn, never block. |
| OQ-612 | How far in advance must leave be requested, and can it be **backdated**? | Backdated leave is how a mis-classified absence gets fixed, so forbidding it entirely creates a dead end. | Backdating allowed with a reason, restricted by the same payroll-period lock as attendance (04 OQ-408). |
