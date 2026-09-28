# 06 — Leave Management — API Design

Conventions as in [01 api-design.md](../01-roles-permissions-auth/api-design.md). Every endpoint is
wrapped in `protectedRoute` and composes `employeeScopeFilter`.

## Helpers this feature provides

```ts
// src/lib/leave/cost.ts
computeLeaveCost(employeeId, start, end, startFraction, endFraction): Promise<LeaveCost>
type LeaveCost = {
  days: Decimal;
  breakdown: Array<{ date: string; working: boolean; days: number;
                     reason: "WORK_WEEK" | "WEEKEND" | "HOLIDAY" | "SHIFT";
                     holidayName?: string }>;
};
// Uses 03's batched workingDaysBetween (NFR-02). The breakdown is what the UI shows the
// employee before they submit — it is the difference between "7 days" and "3 days, because
// the weekend and Eid don't count".
```

```ts
// src/lib/leave/balance.ts
getBalance(employeeId, leaveTypeId, leaveYear?): Promise<Balance>
type Balance = { entitled: Decimal; used: Decimal; pending: Decimal; available: Decimal };
// Reads the snapshot (NFR-01). Four numbers, never one — collapsing them is what causes
// disputes (FR-B-04).

getLedger(employeeId, leaveTypeId, leaveYear?): Promise<LedgerEntry[]>
// The explanation behind the number.
```

```ts
// src/lib/leave/policy.ts
resolvePolicy(employeeId, leaveTypeId, date): Promise<LeavePolicy | null>
// Most-specific assignment wins (data-model.md). null = the type is unavailable to them,
// which is configuration, not an error.

availableLeaveTypes(employeeId, date): Promise<LeaveType[]>
```

Feature 04 reads `LeaveRequestDay` directly; feature 07 reads it and the ledger. Neither goes
through these helpers, and neither writes.

## Leave types and policies

| Method | Path | Permission |
|---|---|---|
| GET / POST | `/api/leave/types` | `leave.policy.read` / `.write` |
| GET / PATCH / DELETE | `/api/leave/types/:id` | |
| GET / POST | `/api/leave/policies` | `leave.policy.read` / `.write` |
| GET / PATCH | `/api/leave/policies/:id` | |
| GET / POST / DELETE | `/api/leave/policies/:id/assignments` | `leave.policy.write` |
| GET | `/api/leave/policies/resolve` | `leave.policy.read` |

`DELETE /api/leave/types/:id` → **409** `TYPE_IN_USE` naming the request and ledger counts, with
deactivation offered instead (FR-T-06).

`PATCH` on a policy's `entitlementDays` returns **409** `ENTITLEMENT_CHANGE_CONFIRM` unless the body
carries `"applyTo": "FUTURE_ONLY" | "ADJUST_CURRENT_YEAR"` (FR-T-08). The response carries the
affected employees and the per-person difference, so the confirmation can state it:

```json
{ "error": { "code": "ENTITLEMENT_CHANGE_CONFIRM",
             "message": "This changes entitlement for 34 employees.",
             "affected": 34, "deltaDays": 2,
             "options": { "FUTURE_ONLY": "Applies from next leave year.",
                          "ADJUST_CURRENT_YEAR": "Writes an adjustment of +2 days to 34 balances now." } } }
```

`resolve?employeeId=&leaveTypeId=&date=` exists for the UI and for support: it returns the policy
that applies and **why** — which assignment matched and at what specificity. Policy resolution is
the thing HR will most often disbelieve.

## Balances

| Method | Path | Permission | Notes |
|---|---|---|---|
| GET | `/api/leave/balances` | `leave.balance.read` | Scoped; all types for one or many employees |
| GET | `/api/leave/balances/:employeeId/:typeId/ledger` | `leave.balance.read` | The explanation |
| POST | `/api/leave/balances/adjust` | `leave.balance.adjust` | Writes an `ADJUSTMENT` entry |
| POST | `/api/leave/balances/rebuild` | `leave.run_accrual` | Rebuild snapshots from the ledger |

```json
{ "employeeId": 12, "leaveYear": "2026-01-01",
  "balances": [ { "leaveTypeId": 1, "leaveType": "Annual leave",
                  "entitled": 20, "used": 6.5, "pending": 2, "available": 11.5,
                  "expiring": { "days": 3, "on": "2027-03-31" } } ] }
```

`expiring` is surfaced on the balance itself rather than left to be discovered in the ledger — it is
the single most useful thing to tell someone about their leave, and the reason they take it before
it lapses.

### `POST /api/leave/balances/adjust`

```json
{ "employeeId": 12, "leaveTypeId": 1, "days": 2,
  "reason": "Goodwill for working the 2025 year-end shutdown" }
```

A reason is mandatory (FR-B-01). The entry is permanent and attributed. **422** `NEGATIVE_RESULT` if
the adjustment would take the balance below the policy's tolerance.

`rebuild` recomputes snapshots from the ledger and **reports** discrepancies found rather than
silently fixing them (FR-B-08) — a drifted snapshot means a bug, and quietly correcting it hides
the bug.

## Requests

| Method | Path | Permission | Notes |
|---|---|---|---|
| POST | `/api/leave/requests/cost` | `leave.request` | Preview the cost — writes nothing |
| GET | `/api/leave/requests` | `leave.read` | Scoped; filters by status, type, date, employee |
| POST | `/api/leave/requests` | `leave.request` | Submit |
| GET | `/api/leave/requests/:id` | `leave.read` | |
| POST | `/api/leave/requests/:id/withdraw` | requester | While `PENDING` |
| POST | `/api/leave/requests/:id/cancel` | requester or `leave.override` | After approval |
| POST | `/api/leave/requests/:id/approve` | `leave.approve` | |
| POST | `/api/leave/requests/:id/reject` | `leave.approve` | Reason required |
| POST | `/api/leave/requests/for-employee` | `leave.request_for_others` | HR records on behalf |

There is no `PATCH` on a request. An approved request is not edited (D-07); the UI offers cancel and
rebook.

### `POST /api/leave/requests/cost`

The endpoint behind US-02, called as the employee picks dates:

```json
{ "employeeId": 12, "leaveTypeId": 1, "startDate": "2026-10-01", "endDate": "2026-10-07",
  "startFraction": "FULL", "endFraction": "SECOND_HALF" }
```

```json
{ "days": 4.5,
  "breakdown": [ { "date": "2026-10-01", "working": true, "days": 1, "reason": "WORK_WEEK" },
                 { "date": "2026-10-03", "working": false, "days": 0, "reason": "WEEKEND" },
                 { "date": "2026-10-06", "working": false, "days": 0, "reason": "HOLIDAY",
                   "holidayName": "National Day" } ],
  "balanceBefore": 11.5, "balanceAfter": 7,
  "warnings": [ { "code": "TEAM_CLASH", "message": "2 colleagues are already off on 1–2 October.",
                  "employees": ["Imran Qadir", "Sara Ahmed"] } ] }
```

Warnings, not errors — the clash check never blocks (FR-C-03, OQ-611).

### `POST /api/leave/requests`

**201**. In one transaction (NFR-04): validate, write the request, write `LeaveRequestDay` rows, write
the `RESERVATION` ledger entry, resolve and store the approval chain, update the snapshot, and call
`notify("leave.request_pending", …)` on the caller's transaction (05 FR-D-02).

The balance check happens **inside** the transaction, against the ledger, not from a value read
before it. Two concurrent 2-day requests against a 2-day balance must not both succeed (acceptance
criterion 2).

Error codes, each specific because each has a different fix: `INSUFFICIENT_BALANCE` (with available
and requested), `OVERLAPPING_REQUEST` (naming the other request), `NOTICE_PERIOD` (with the earliest
allowed date), `MAX_CONSECUTIVE_EXCEEDED`, `TYPE_UNAVAILABLE` (not in policy, or still in probation),
`OUTSIDE_EMPLOYMENT`, `PERIOD_LOCKED` (payroll, FR-R-08), `ZERO_WORKING_DAYS` — the last being a
range entirely of weekends and holidays, which is a mistake rather than a free holiday.

Auto-approved types (FR-A-08) skip the chain and apply their effects immediately, returning
`"status": "APPROVED"`.

### `POST /api/leave/requests/:id/approve`

```json
{ "note": "Fine — Ravi is covering.", "confirmCostChange": false }
```

Recomputes the cost (FR-R-03). If it differs from submission, **409** `COST_CHANGED` with both
figures and the reason, until `confirmCostChange: true`:

```json
{ "error": { "code": "COST_CHANGED",
             "message": "This request now costs 3 days instead of 4.",
             "was": 4, "now": 3,
             "because": "6 October became a public holiday after this was submitted." } }
```

On approval: the step is decided; if it was the last, the request becomes `APPROVED`, the
`RESERVATION` is released and `TAKEN` written, `LeaveRequestDay` rows are marked approved, the
affected dates are marked dirty through 04's `markDirty` (FR-X-01), the requester is notified, and
any sibling approver's notification is superseded (FR-A-10).

**403** if the actor is not the current step's approver, not a delegate for them, and lacks
`leave.override`. **422** `SELF_APPROVAL` if they are the requester (FR-A-04).

### `POST /api/leave/requests/:id/cancel`

```json
{ "reason": "Trip cancelled", "refund": "FUTURE_ONLY" }
```

`refund` is `FUTURE_ONLY` (default) or `ALL` (FR-R-10). Writes `REFUND` for the refunded days, marks
those dates dirty, and notifies. **409** `PERIOD_LOCKED` for days inside a finalised payroll period,
naming the period and the adjustment route — the same shape as 04's lock error, deliberately, so the
two features teach the same lesson once.

## Approvals and delegation

| Method | Path | Permission |
|---|---|---|
| GET | `/api/leave/approvals/pending` | `leave.approve` |
| GET | `/api/leave/delegations` | `leave.delegate` |
| POST | `/api/leave/delegations` | `leave.delegate` |
| DELETE | `/api/leave/delegations/:id` | `leave.delegate` |

`approvals/pending` returns what is waiting on the actor — **including** requests routed to them by
delegation, each labelled with whose queue it is. It carries the context the approver needs inline
(US-09), so the decision does not require three more requests:

```json
{ "data": [ { "requestId": 88, "employee": { "id": 12, "name": "Ayesha Khan" },
              "leaveType": "Annual leave", "dates": "1–7 Oct", "days": 4.5,
              "reason": "Family trip", "requestedAt": "2026-09-14T10:02:00Z",
              "waitingDays": 1, "escalatesOn": "2026-09-17",
              "balanceAfter": 7, "recentLeaveDays90": 3,
              "clash": { "count": 2, "employees": ["Imran Qadir", "Sara Ahmed"] },
              "onBehalfOf": null } ] }
```

`POST /api/leave/delegations` validates that the delegate is not the delegator, that the range is
not in the past, and warns — does not block — when the delegate is themselves on approved leave for
part of it. That last check exists because it is exactly what happens in a small team.

## Calendar

### `GET /api/leave/calendar` — `leave.read`

`?from=&to=&departmentId=&includePending=true`. Scoped. Reads `LeaveRequestDay`, joined with 03's
holidays and the work week (FR-C-04), in a bounded number of queries (NFR-03).

```json
{ "days": [ { "employeeId": 12, "date": "2026-10-01", "fraction": "FULL",
              "status": "APPROVED", "leaveType": "Annual leave", "colour": "#4A7" } ],
  "holidays": [ { "date": "2026-10-06", "name": "National Day" } ],
  "nonWorkingDays": ["2026-10-03", "2026-10-04"] }
```

Confidential types return `"leaveType": null` with `"confidential": true` for anyone but HR and the
employee (FR-T-07, FR-C-05). The redaction happens server-side; the client never receives a name it
is expected not to show.

## The engines

| Method | Path | Permission |
|---|---|---|
| POST | `/api/leave/engine/accrual/preview` | `leave.run_accrual` |
| POST | `/api/leave/engine/accrual` | `leave.run_accrual` |
| POST | `/api/leave/engine/carry-over/preview` | `leave.run_accrual` |
| POST | `/api/leave/engine/carry-over` | `leave.run_accrual` |
| POST | `/api/leave/engine/settlement/:employeeId` | `leave.run_accrual` |
| GET | `/api/leave/engine/runs` | `leave.run_accrual` |

Every engine has a preview that writes nothing and returns the exact entries it would write
(FR-Y-05) — the same pattern as 04's recompute preview, for the same reason: an operation that
silently rewrites 200 balances is one people will refuse to run.

```json
{ "type": "CARRY_OVER", "periodKey": "2026", "employeeCount": 187,
  "entries": [ { "employeeId": 12, "leaveType": "Annual leave",
                 "remaining": 7, "cap": 5, "carriedOver": 5, "forfeited": 2,
                 "expiresOn": "2027-03-31" } ],
  "totals": { "carriedOver": 612, "forfeited": 88 } }
```

`forfeited` in the totals is deliberately prominent. Carry-over runs are the moment a company
discovers it is about to take 88 days off its staff, and that should be visible before the click,
not afterwards.

Running for a `periodKey` already run returns **200** with `"alreadyRun": true` and the original
run's details — not an error, because the honest answer to "did this happen?" is yes (D-08).

`settlement` computes a leaver's final position — encashment or forfeit per policy (FR-Y-08, OQ-609)
— and returns it for HR to confirm. It never applies automatically on termination, because the
figure usually needs a human to look at it.

## Scheduled jobs

| Job | Frequency | Does |
|---|---|---|
| Accrual | daily | Runs any accrual period that has become due; idempotent per period |
| Carry-over expiry | daily | Writes `CARRY_OVER_EXPIRY` for entries whose `expiresOn` has passed |
| Escalation | hourly | Escalates requests pending past the threshold (FR-A-07) |
| Expiry warning | weekly | Notifies employees with leave expiring within the warning window |
| Snapshot consistency | nightly | Compares snapshots with the ledger and reports drift (FR-B-08) |
| Delegation activation | daily | Nothing to write — delegations resolve by date — but reports delegations starting today so nobody is surprised |

Year-end carry-over is deliberately **not** scheduled. It is a decision with a preview, run by a
person (D-08, FR-Y-05).

## Open questions

| ID | Question |
|---|---|
| OQ-606 | The `approvalSteps` JSON shape depends on how conditional the chain needs to be. A fixed single step needs no JSON at all; per-type or per-length routing needs most of a rules engine. |
| OQ-614 | Should `requests/cost` be callable for another employee (a manager planning cover)? Scoped read would allow it; it also exposes their balance. Proposed: cost yes, balance no. |
| OQ-615 | Should cancellation of approved leave in the past require approval, rather than being unilateral? Currently the requester can cancel; for past days that is effectively editing history. |
| OQ-616 | Whether the settlement figure on termination should block the termination in feature 02 until acknowledged. Tempting, and a cross-feature coupling worth being deliberate about. |
