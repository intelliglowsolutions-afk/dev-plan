# 07 — Payroll — API Design

Conventions as in [01 api-design.md](../01-roles-permissions-auth/api-design.md). Every endpoint is
wrapped in `protectedRoute`. Scope is narrower here than anywhere else: most endpoints require an
`ALL`-scoped payroll permission, and the employee-facing ones are `SELF` only. There is no
`DEPARTMENT` scope in this feature (D-06).

## The helper this feature owes the rest of the system

```ts
// src/lib/payroll/locks.ts
isPeriodLocked(employeeId: number, date: CalendarDate): Promise<LockInfo | null>
type LockInfo = {
  payRunId: number; periodName: string; finalisedAt: Date;
  startDate: string; endDate: string;
};
// Features 04 and 06 call this before every attendance correction, recompute, leave
// request, cancellation, and amendment (FR-K-02). Both shipped with a stub that always
// returned null; this feature's migration replaces it.

lockedRangesFor(employeeIds: number[], from, to): Promise<Map<number, LockInfo[]>>
// The batched form, for 04's recompute preview and 06's engine previews, which need to
// report "these periods were skipped" rather than failing per day.
```

Returning the run and the period — not just `true` — is what lets 04 and 06 produce the error
message they already promise: "September 2026 payroll was finalised on 30 Sep. Create a payroll
adjustment instead."

## Components, brackets, structures

| Method | Path | Permission |
|---|---|---|
| GET / POST | `/api/payroll/components` | `payroll.component.read` / `.write` |
| GET / PATCH | `/api/payroll/components/:id` | |
| POST | `/api/payroll/components/validate` | `payroll.component.write` |
| POST | `/api/payroll/components/test` | `payroll.component.write` |
| GET / POST | `/api/payroll/bracket-tables` | `payroll.component.read` / `.write` |
| PUT | `/api/payroll/bracket-tables/:id/rows` | `payroll.component.write` |
| GET / POST | `/api/payroll/structures` | `payroll.component.read` / `.write` |
| PUT | `/api/payroll/structures/:id/components` | `payroll.component.write` |

### `POST /api/payroll/components/validate`

Parses the formula and checks it without saving:

```json
{ "formula": "min(BASIC * 0.4, 20000)", "calculationOrder": 20 }
```

```json
{ "valid": true, "references": ["BASIC"], "usesVariables": [],
  "warnings": [ { "code": "UNROUNDED", "message": "No rounding rule set; NEAREST_UNIT is the default." } ] }
```

Failures are specific, because a formula error message is the entire debugging experience for
someone who is not a programmer:

```json
{ "valid": false,
  "errors": [ { "code": "FORWARD_REFERENCE", "message": "TAX is calculated after this component (order 90). A component can only use components calculated before it.", "at": "TAX" },
              { "code": "UNKNOWN_IDENTIFIER", "message": "'HRA_OLD' is not a component or a variable.", "at": "HRA_OLD",
                "available": ["BASIC", "HRA", "TRANSPORT", "WORKING_DAYS", "PAID_DAYS"] } ] }
```

The grammar is closed (D-02): identifiers, numbers, `+ - * / ( )`, comparisons, `? :`, `min`, `max`,
`round`, `floor`, `ceil`, `bracket(TABLE, value)`. Anything else — a property access, a call to
something undeclared, an assignment — is a parse error, not a runtime one. Evaluation is bounded in
depth and time (NFR-04).

### `POST /api/payroll/components/test`

```json
{ "componentId": 4, "inputs": { "BASIC": 50000, "WORKING_DAYS": 22, "PAID_DAYS": 20 } }
```

```json
{ "result": 18181.82, "rounded": 18182,
  "trail": [ { "step": "formula", "expression": "BASIC / WORKING_DAYS * PAID_DAYS",
               "substituted": "50000 / 22 * 20", "value": 45454.55 },
             { "step": "percentage", "expression": "× 40%", "value": 18181.82 },
             { "step": "rounding", "rule": "NEAREST_UNIT", "value": 18182 } ] }
```

This endpoint is US-02 and it is worth more than it looks: it is the only way a payroll officer can
be confident in a formula before 200 people are paid by it.

## Compensation and bank details

| Method | Path | Permission | Notes |
|---|---|---|---|
| GET | `/api/payroll/compensation/:employeeId` | `payroll.compensation.read` | Full dated history |
| POST | `/api/payroll/compensation` | `payroll.compensation.write` | New dated record |
| GET | `/api/payroll/compensation/:employeeId/current` | `payroll.compensation.read` | |
| GET / POST | `/api/payroll/bank-accounts/:employeeId` | `payroll.compensation.read` / `.write` | |

There is no `PATCH` on compensation. A change is a new dated record (D-09, FR-S-04).

`POST /api/payroll/compensation` with an `effectiveFrom` inside a finalised period returns **200**
with a proposed arrears adjustment rather than an error — the change is legitimate, and the system's
job is to compute the difference and hand it to a human (FR-A-03):

```json
{ "compensation": { "id": 412, "effectiveFrom": "2026-08-01", "baseSalary": 60000 },
  "arrears": { "isProposed": true, "periods": [ { "period": "August 2026", "difference": 8000 },
                                                { "period": "September 2026", "difference": 8000 } ],
               "total": 16000,
               "message": "This backdated change affects 2 finalised periods. Review the proposed arrears before the next run." } }
```

Creating or changing a bank account triggers `notify("payroll.bank_details_changed", …)` to the
employee — with no account details in the message (FR-S-06).

## Pay periods

| Method | Path | Permission |
|---|---|---|
| GET | `/api/payroll/periods` | `payroll.read` |
| POST | `/api/payroll/periods/generate` | `payroll.component.write` |
| PATCH | `/api/payroll/periods/:id` | `payroll.component.write` |

`PATCH` → **409** `PERIOD_HAS_FINALISED_RUN` (FR-P-02).

## Pay runs — the lifecycle

| Method | Path | Permission | Transition |
|---|---|---|---|
| POST | `/api/payroll/runs` | `payroll.run` | → `DRAFT` |
| GET | `/api/payroll/runs` | `payroll.read` | |
| GET | `/api/payroll/runs/:id` | `payroll.read` | |
| PATCH | `/api/payroll/runs/:id/population` | `payroll.run` | Adjust who is in it |
| POST | `/api/payroll/runs/:id/calculate` | `payroll.run` | → `CALCULATING` → `CALCULATED` |
| GET | `/api/payroll/runs/:id/exceptions` | `payroll.read` | |
| POST | `/api/payroll/runs/:id/exceptions/:eid/acknowledge` | `payroll.run` | |
| GET | `/api/payroll/runs/:id/variance` | `payroll.read` | |
| POST | `/api/payroll/runs/:id/approve` | `payroll.approve` | → `APPROVED` |
| POST | `/api/payroll/runs/:id/reject` | `payroll.approve` | → `DRAFT`, with a reason |
| POST | `/api/payroll/runs/:id/finalise` | `payroll.finalise` | → `FINALISED` + locks |
| POST | `/api/payroll/runs/:id/publish` | `payroll.publish` | Payslips visible |
| POST | `/api/payroll/runs/:id/mark-paid` | `payroll.finalise` | → `PAID` |
| POST | `/api/payroll/runs/:id/reopen` | `payroll.reopen` | → `CALCULATED`, releases locks |
| POST | `/api/payroll/runs/:id/cancel` | `payroll.run` | → `CANCELLED` |

Every transition validates the current state and returns **409** `INVALID_TRANSITION` naming the
current status. The lifecycle is the feature's main safety mechanism and is enforced server-side,
never by the UI hiding buttons.

### `POST /api/payroll/runs`

```json
{ "periodId": 9, "type": "REGULAR" }
```

**201** with the resolved population and why each person is in or out:

```json
{ "run": { "id": 14, "status": "DRAFT" },
  "population": { "included": 187,
                  "excluded": [ { "employeeId": 44, "name": "Imran Sheikh",
                                  "reason": "Terminated 12 Aug 2026" },
                                { "employeeId": 61, "name": "Nadia Rauf",
                                  "reason": "Already paid in off-cycle run #13" } ] } }
```

Showing the exclusions is as important as the inclusions. "Why wasn't X paid" is the question this
prevents.

### `POST /api/payroll/runs/:id/calculate`

Asynchronous — it processes employees independently and isolates failures (D-10, FR-R-03). Returns
immediately with a job reference; progress is polled from the run.

Per employee: gather inputs, snapshot them, evaluate components in order, reconcile the totals, and
write the payslip and its lines — or write an exception and continue. Manual inputs and
acknowledgements from a previous calculation are preserved (FR-R-06).

```json
{ "status": "CALCULATED", "employeeCount": 187,
  "totals": { "gross": 9840000, "deductions": 1480000, "net": 8360000 },
  "exceptions": { "blocking": 3, "warning": 12 },
  "durationSeconds": 41 }
```

### `GET /api/payroll/runs/:id/exceptions`

The screen the payroll officer actually works in:

```json
{ "data": [ { "id": 88, "employeeId": 12, "name": "Ayesha Khan",
              "kind": "UNRESOLVED_ATTENDANCE", "severity": "BLOCKING",
              "message": "3 days have no attendance data (device offline 14–16 Sep).",
              "detail": { "dates": ["2026-09-14","2026-09-15","2026-09-16"], "deviceGapId": 7 },
              "resolutionPath": "/attendance/gaps/7" } ] }
```

`resolutionPath` is what turns an exception list into a workflow: every exception knows where it is
fixed, and most of those places are in features 04 and 06.

### `GET /api/payroll/runs/:id/variance`

```json
{ "threshold": { "percent": 20, "amount": 5000 },
  "data": [ { "employeeId": 12, "name": "Ayesha Khan",
              "previousNet": 41200, "currentNet": 28400, "change": -12800, "changePercent": -31.1,
              "likelyCause": "6 unpaid leave days this period (0 last period)" } ] }
```

`likelyCause` is derived by comparing the two snapshots component by component and naming the
largest contributor. It is a hint, not a verdict, and it is the difference between a variance report
someone reads and one they scroll past.

### `POST /api/payroll/runs/:id/approve`

**409** `BLOCKING_EXCEPTIONS` if any remain, listing them (FR-R-05). Approving records the totals as
approved (FR-R-12) — so a later recalculation that changes them is visible as a discrepancy against
what was actually approved.

### `POST /api/payroll/runs/:id/finalise`

In one transaction: set status, write `PeriodLock` rows for every employee in the run, mark payslips
immutable. **409** `NOT_APPROVED` if the run has not been approved — the two steps are never
collapsed, even for one person holding both permissions (FR-R-08, OQ-707).

### `POST /api/payroll/runs/:id/reopen`

```json
{ "reason": "Overtime for the night shift was missing from the source data." }
```

Super admin only. Releases the locks, returns the run to `CALCULATED`, marks published payslips
superseded (FR-R-11), and audits distinctly. The response states what became editable again, because
the consequence is that 04 and 06 accept changes to that period once more.

## Payslips

| Method | Path | Permission | Notes |
|---|---|---|---|
| GET | `/api/payroll/payslips` | `payroll.read` | All; filters by run, period, employee |
| GET | `/api/payroll/payslips/:id` | `payroll.read` or own | Logged (FR-L-06) |
| GET | `/api/payroll/payslips/:id/pdf` | `payroll.read` or own | Logged |
| GET | `/api/me/payslips` | `payroll.read_own` | The employee's own, published only |
| POST | `/api/payroll/payslips/batch-pdf` | `payroll.read` | Background batch (NFR-06) |

A payslip response carries its lines with their trails, and the snapshot is available to those with
`payroll.read` — it is what answers "why is this number this". Employees see the lines and the
notes, not the raw snapshot.

`/api/me/payslips` returns only **published** payslips, never drafts. An employee seeing a figure
that later changes is worse than seeing it a day later.

## Adjustments

| Method | Path | Permission |
|---|---|---|
| GET / POST | `/api/payroll/adjustments` | `payroll.adjust` |
| POST | `/api/payroll/adjustments/:id/approve` | `payroll.adjust` |
| DELETE | `/api/payroll/adjustments/:id` | `payroll.adjust` |

Proposed arrears (FR-A-03) arrive as `isProposed: true` and are applied only once approved. Deleting
is possible only before application; an applied adjustment is corrected by another adjustment,
consistent with everything else in this feature (D-07).

## Exports

| Method | Path | Permission |
|---|---|---|
| POST | `/api/payroll/runs/:id/export/bank` | `payroll.export_bank` |
| POST | `/api/payroll/runs/:id/export/accounting` | `payroll.export_bank` |
| GET | `/api/payroll/runs/:id/exports` | `payroll.read` |

The bank export returns the file **and** the exclusions, in the same response, so the omission cannot
be missed (FR-E-02):

```json
{ "fileUrl": "/api/payroll/exports/55/download", "rowCount": 185, "totalAmount": 8360000,
  "checksum": "sha256:…",
  "excluded": [ { "employeeId": 77, "name": "Bilal Aslam", "reason": "No active bank account" } ] }
```

Only a `FINALISED` or `PAID` run can be exported. Every export is recorded with its checksum, and
re-exports are separate records (FR-E-03) — "which file did we actually send the bank" must be
answerable.

## Scheduled jobs

| Job | Frequency | Does |
|---|---|---|
| Period generation | monthly | Keeps the pay calendar populated a year ahead |
| Cut-off settlement | on run creation | Proposes adjustments for the previous period's assumed days (FR-X-04, FR-A-04) |
| Run reminder | daily | Notifies payroll when a period's cut-off is approaching and no run exists |
| Approval reminder | daily | Notifies the approver about a `CALCULATED` run awaiting them |
| Stale rates warning | monthly | Flags components whose `ratesReviewedOn` is older than a year (FR-C-09) |

There is no scheduled job that calculates, approves, finalises, or pays. Every step that moves money
is taken by a person (D-04).

## Open questions

| ID | Question |
|---|---|
| OQ-701 | Whether the closed grammar in `validate` covers the real components. If a needed calculation cannot be expressed, the answer is a new `CalculationMethod`, not an escape hatch into arbitrary code. |
| OQ-704 | `payroll.partialMonthBasis` changes every pro-rated figure. It is a setting so it can be changed, but changing it after a run invalidates comparisons — it may deserve to be change-controlled (03 FR-S-05). |
| OQ-715 | Should `calculate` be resumable across a restart, or is re-running from scratch acceptable? At 200 employees, re-running is fine; at 5 000 it is not. |
| OQ-716 | Should the variance report compare against the previous period or a rolling average? Previous-period comparison flags every legitimate one-off twice — once when it happens and once when it stops. |
