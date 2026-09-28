# 07 — Payroll — Requirements

## Actors

| Actor | What they do here |
|---|---|
| Payroll officer | Runs the cycle: opens the run, reviews exceptions, fixes inputs, recalculates, submits for approval. |
| Payroll approver | Reviews the totals and variances, approves or sends back. Usually not the same person as the officer (OQ-707). |
| HR admin | Maintains components, structures, and employee compensation. May or may not hold payroll run permissions. |
| Super admin | Reopens finalised runs, unlocks periods. Both are exceptional acts. |
| Employee | Sees their own payslips, and nothing else. |
| Manager | Sees **nothing** here by default. A manager's `DEPARTMENT` scope does not extend to pay (D-06). |
| The calculation engine | Produces every number on every payslip. Specified as carefully as any user. |

## User stories

### Setting up

- **US-01** — As HR, I define the components that make up pay — basic, allowances, deductions — and
  how each is calculated.
- **US-02** — As HR, I test a formula against sample values before anyone is paid by it.
- **US-03** — As HR, I group components into a salary structure and assign it to employees or groups.
- **US-04** — As HR, I set an employee's salary, with an effective date, and last month's payslip
  still shows last month's figure.
- **US-05** — As HR, I record an employee's bank details for payment.

### Running payroll

- **US-06** — As a payroll officer, I open a run for the period and see who is in it and why.
- **US-07** — As a payroll officer, I calculate the run and get a list of **exceptions** — people
  whose pay could not be computed, or looks wrong — rather than a silent result.
- **US-08** — As a payroll officer, I compare this run against last month per employee, so an
  unexpected change is visible before anyone is paid.
- **US-09** — As a payroll officer, I fix an input — an attendance correction, a missing salary —
  and recalculate without starting over.
- **US-10** — As an approver, I see the totals, the headcount, the variance, and the exceptions on
  one screen, and approve or send it back with a reason.
- **US-11** — As a payroll officer, I finalise the run, which locks the period so nobody can change
  the attendance and leave it was based on.
- **US-12** — As a payroll officer, I export the bank payment file and the accounting summary.
- **US-13** — As a payroll officer, I mark the run paid once the bank confirms.

### Payslips and corrections

- **US-14** — As an employee, I see my payslip, understand each line, and download it as a PDF.
- **US-15** — As an employee, I compare this month with last month.
- **US-16** — As a payroll officer, I correct a mistake found after finalisation, as an adjustment
  in the next run, and the employee can see what it was for.
- **US-17** — As a payroll officer, I run an off-cycle payment — a bonus, a final settlement —
  without disturbing the monthly cycle.
- **US-18** — As HR, I see an employee's pay history over time.

## Functional requirements

### Components and formulas — FR-C

| ID | Requirement |
|---|---|
| FR-C-01 | A component has: code, name, type (`EARNING` or `DEDUCTION`), calculation method, calculation order, rounding rule, taxability flag, whether it appears on the payslip, and an optional GL code. |
| FR-C-02 | Calculation methods: `FIXED` (from the employee's compensation), `FORMULA`, `PERCENTAGE` (of another component or of a named base), `BRACKET` (a lookup table), `ATTENDANCE_DERIVED` (from 04's figures), `LEAVE_DERIVED` (from 06), `MANUAL` (entered per run per employee). |
| FR-C-03 | Formulas may reference: other components by code, the employee's compensation values, attendance and leave figures for the period, period metadata (working days, calendar days), and constants. Every available variable is declared, and an unknown reference fails validation at save time. |
| FR-C-04 | The formula language supports arithmetic, parentheses, comparison and conditional expressions, `min`, `max`, `round`, `floor`, `ceil`, and bracket lookups — and nothing else (D-02). No loops, no function definitions, no property access, no host calls. |
| FR-C-05 | Calculation order is explicit and validated: a component may only reference components with a lower order. Circular references are rejected at save time, not discovered at run time. |
| FR-C-06 | A bracket table has ordered rows with an upper bound, a rate, and a fixed amount, and a declared basis (cumulative or marginal). This exists because most statutory deductions are shaped this way, without the system knowing any particular jurisdiction's numbers (D-01). |
| FR-C-07 | A component can be tested against sample inputs before use, showing the result and the intermediate values (US-02). |
| FR-C-08 | Editing a component does **not** change any finalised payslip (D-03). It applies to future calculations and to draft runs on recalculation. |
| FR-C-09 | Components carry a `ratesReviewedOn` date, surfaced in the UI, so a stale tax table is visible rather than assumed current (OQ-702). |
| FR-C-10 | A component cannot be deleted once any payslip references it; it is deactivated. |

### Structures and compensation — FR-S

| ID | Requirement |
|---|---|
| FR-S-01 | A salary structure is an ordered set of components with optional per-structure defaults. |
| FR-S-02 | Structures are assigned to employees directly, or by group (employment type, department, grade) with the same most-specific-wins resolution as feature 06's leave policies. |
| FR-S-03 | An employee's compensation is date-ranged: base salary, per-component fixed amounts, and the structure in force, each with an effective date (D-09). |
| FR-S-04 | A compensation change is a new dated record, never an edit. Backdated changes are allowed with `payroll.compensation.write` and produce arrears in the next run rather than altering finalised payslips (FR-A-03). |
| FR-S-05 | An employee with no compensation record in force for a period is an **exception**, not a zero. Paying someone nothing silently is the worst possible failure mode here. |
| FR-S-06 | Bank details: account name, number/IBAN, bank, branch, and an active flag. An employee may have one active account. Changes are audited and notify the employee (05) — a changed bank account is a classic fraud vector. |
| FR-S-07 | Compensation and bank details are visible only with `payroll.compensation.read`, which no default role holds except HR admin and super admin (D-06). |

### The pay calendar — FR-P

| ID | Requirement |
|---|---|
| FR-P-01 | A pay calendar defines periods: start date, end date, cut-off date, and pay date, generated from a frequency (OQ-703). |
| FR-P-02 | Periods are generated ahead and are editable before use. A period with a finalised run is immutable. |
| FR-P-03 | A period's attendance may be incomplete at cut-off (the period has not ended). The run records which days were actual and which were assumed, and the difference is settled in the next period (FR-A-04). |
| FR-P-04 | Off-cycle runs reference a period but are marked as such and do not consume the regular run slot (US-17). |

### Pay runs — FR-R

| ID | Requirement |
|---|---|
| FR-R-01 | A run has a lifecycle: `DRAFT` → `CALCULATED` → `APPROVED` → `FINALISED` → `PAID`, plus `CANCELLED` from any state before finalisation (D-04). Each transition is recorded with actor and time. |
| FR-R-02 | Opening a run selects its population: employed during the period, with compensation in force, excluding those already paid for it in another run. The selection is shown and can be adjusted before calculation. |
| FR-R-03 | Calculation processes employees independently. A failure marks that employee as an exception with the error and continues (D-10). |
| FR-R-04 | Exception categories, each with its own resolution path: no compensation, no bank details, negative net pay, unresolved attendance (`UNKNOWN` days from 04), pending attendance corrections, pending leave requests in the period, variance beyond threshold, formula error, missing input for a `MANUAL` component. |
| FR-R-05 | A run cannot be approved while blocking exceptions are unresolved. Non-blocking exceptions (a large variance that is legitimate) can be acknowledged with a note. |
| FR-R-06 | Recalculation of a `DRAFT` or `CALCULATED` run discards and recomputes, preserving manual inputs and acknowledgements. |
| FR-R-07 | **Variance analysis** compares each employee's net pay with their previous period and flags differences beyond a configured percentage or absolute amount (US-08). This is the single most effective check against a bad configuration reaching a bank. |
| FR-R-08 | Approval is a distinct act from finalisation, and both are audited with the actor. Where the same person holds both permissions, they are still two deliberate steps (OQ-707). |
| FR-R-09 | Finalisation writes the period lock, publishes nothing yet, and makes the payslips immutable. |
| FR-R-10 | Payslip publication is separate from finalisation, so payslips can be released on the pay date rather than when the run completes. |
| FR-R-11 | Reopening a finalised run requires super admin, a reason, and an audit entry; it releases the lock and returns the run to `CALCULATED`. Payslips already published are marked superseded, not deleted. |
| FR-R-12 | A run records totals: gross, each component's total, net, headcount, and the count of exceptions at approval. These are what the approver approved and are kept as part of the run. |

### Calculation — FR-X

| ID | Requirement |
|---|---|
| FR-X-01 | For each employee, the engine gathers inputs: compensation in force, the structure's components, attendance figures from 04, leave from 06, period metadata from 03, and any manual inputs — and **snapshots them all** onto the payslip (D-03). |
| FR-X-02 | Attendance inputs: paid days, unpaid days, absent days, approved overtime hours, worked-on-holiday hours. Read from `attendance_days`; never recomputed here. |
| FR-X-03 | Leave inputs: paid leave days, unpaid leave days, and encashment days from 06's ledger. |
| FR-X-04 | Where attendance for the period is incomplete at cut-off (FR-P-03), the remaining days are treated per configuration — assumed present is the default — and the assumption is recorded on the payslip line's trail, then settled next period. |
| FR-X-05 | Components are evaluated in order; each produces a payslip line with its value, the formula used, the inputs substituted, and the intermediate result (D-08). |
| FR-X-06 | Rounding is applied per the component's rule at the point declared, and the payslip's totals are reconciled against the sum of its lines. A discrepancy fails that employee as an exception rather than shipping a payslip that does not add up (D-05). |
| FR-X-07 | Net pay below zero is an exception, never a payslip. |
| FR-X-08 | Calculation is deterministic: the same inputs produce the same payslip, byte for byte. No `now()` inside the engine; the run's date is an input. |
| FR-X-09 | The engine must be testable without a database: pure functions taking a snapshot and returning lines (same requirement as 04 NFR-09). |

### Payslips — FR-L

| ID | Requirement |
|---|---|
| FR-L-01 | A payslip belongs to one employee, one run, one period, and is immutable once its run is finalised. |
| FR-L-02 | It shows: employee and job details as at the period, earnings lines, deduction lines, gross, total deductions, net, the payment account (masked), period dates, and year-to-date figures where configured. |
| FR-L-03 | Year-to-date figures are computed from finalised payslips in the same fiscal year (03), not from live data. |
| FR-L-04 | PDF generation is deterministic and reproducible from the stored payslip; the PDF is generated on demand, not stored, unless OQ-709 changes. |
| FR-L-05 | **Revised 2026-09-18 (OQ-709): payslips are delivered both in the portal and as an emailed PDF.** This overrides 05 D-06 for this one case. Required controls, because the PDF leaves the system's access control entirely — forwarded, archived on third-party servers, opened on unlocked phones: (a) the **email body still carries no figures** — it says a payslip is attached and links to the portal; (b) the **PDF is password-protected**, with the scheme confirmed at build time (employee code plus date of birth is the common choice, and is weak — a per-employee secret is better if one exists); (c) delivery is **audited** as a disclosure, per payslip; (d) a bounce or suppression (05 FR-D-06) must **not** silently drop a payslip — it raises an alert, because "I never got my payslip" is a payroll dispute, not a notification failure. |
| FR-L-06 | An employee sees only their own payslips. Access is logged separately from ordinary audit (D-06), so "who looked at whose pay" is answerable. |
| FR-L-07 | A superseded payslip (from a reopened run) remains visible to the employee with a clear marker and a link to its replacement. Hiding it would look like concealment. |
| FR-L-08 | Payslips are retained indefinitely, subject to feature 02's retention question (OQ-203). |

### Adjustments and arrears — FR-A

| ID | Requirement |
|---|---|
| FR-A-01 | An adjustment is a line applied to a future run, referencing what it corrects: a period, a component, an employee, an amount, and a reason (D-07). |
| FR-A-02 | Adjustments appear on the payslip as their own lines with their reason visible to the employee. A silent correction is indistinguishable from an error. |
| FR-A-03 | Backdated compensation changes generate arrears automatically: the difference between what was paid and what would have been paid, per affected period, proposed for review rather than applied silently. |
| FR-A-04 | Cut-off settlements (FR-X-04) are a system-generated adjustment in the following period, labelled as such. |
| FR-A-05 | Adjustments require `payroll.adjust` and are audited with before/after context. |

### Locks — FR-K

| ID | Requirement |
|---|---|
| FR-K-01 | Finalising a run writes a period lock covering its dates and its employees. |
| FR-K-02 | The lock is queryable by 04 and 06 through one shared helper: `isPeriodLocked(employeeId, date)`. |
| FR-K-03 | The lock's error response includes the period, the run, the finalisation date, and the adjustment route — which is what 04 and 06 already promise their users (04 FR-X-09, 06 FR-R-08). |
| FR-K-04 | Unlocking requires super admin and a reason, is audited distinctly, and is only possible by reopening the run (FR-R-11). |
| FR-K-05 | An employee added to the system after a period was locked is not retrospectively locked for that period, so a late-onboarded employee's history can still be entered. |

### Exports — FR-E

| ID | Requirement |
|---|---|
| FR-E-01 | The bank payment file contains payee name, account, amount, and reference, in the configured format (OQ-708), for employees with valid bank details in a finalised run. |
| FR-E-02 | Employees without bank details are excluded and **listed**, never silently omitted from a payment file. |
| FR-E-03 | Every export is recorded: who, when, which run, how many rows, the total amount, and a checksum of the file. Re-exporting is allowed and recorded separately. |
| FR-E-04 | The accounting export summarises by component and department (OQ-711). |
| FR-E-05 | Exports are permission-gated (`payroll.export_bank`) and audited as disclosures of sensitive data. |

## Non-functional requirements

| ID | Requirement |
|---|---|
| NFR-01 | A run for 200 employees calculates in under two minutes, loading inputs in batches — attendance, leave, and compensation are fetched per run, not per employee (03 NFR-02, 04 NFR-03). |
| NFR-02 | Calculation is resumable: interrupting a run leaves completed employees calculated and the rest pending. |
| NFR-03 | All monetary values are `Decimal(14,2)` in storage and arbitrary-precision decimal in computation. Floats are prohibited anywhere in the calculation path (D-05). |
| NFR-04 | Formula evaluation is sandboxed with no access to the host environment, and bounded in time and expression depth (D-02). |
| NFR-05 | Salary data must not appear in application logs, error messages, exports the actor is not entitled to, or any notification body (D-06). |
| NFR-06 | Payslip PDF generation for 200 employees must be possible as a background batch, not only one at a time. |
| NFR-07 | The engine is unit-testable with fixture snapshots and no database (FR-X-09). |
| NFR-08 | A payslip's stored snapshot must be sufficient to re-render it with no other table available. |

## Permission keys added by this feature

| Key | Meaning | Default holders |
|---|---|---|
| `payroll.read` | View runs and payslips for others | HR admin |
| `payroll.read_own` | View your own payslips | Everyone: SELF |
| `payroll.component.read` / `.write` | Components, structures, bracket tables | HR admin |
| `payroll.compensation.read` / `.write` | Salaries and bank details | HR admin |
| `payroll.run` | Open, calculate, recalculate runs | Payroll officer |
| `payroll.approve` | Approve a calculated run | Payroll approver |
| `payroll.finalise` | Finalise and lock | Payroll officer |
| `payroll.publish` | Release payslips to employees | Payroll officer |
| `payroll.adjust` | Create adjustments | Payroll officer |
| `payroll.export_bank` | Generate the payment file | Payroll officer |
| `payroll.reopen` | Reopen a finalised run, unlock a period | Super admin |

Note that `MANAGER` holds none of these. Feature 01's role grid gives managers `DEPARTMENT` scope
over attendance and leave and nothing over pay; that is deliberate and should survive review.

## Acceptance criteria (feature-level)

1. A component referencing another component of higher calculation order is rejected at save time.
2. A formula containing anything outside the permitted grammar is rejected, and no host function is
   reachable from a formula under test.
3. A payslip finalised in March renders identically in December after the employee's salary, the
   component's formula, and their department have all changed.
4. An employee with no compensation record appears as an exception; the run cannot be approved until
   it is resolved or they are removed from the run.
5. An employee with `UNKNOWN` attendance days (04's device gap) is flagged as an exception rather
   than paid on incomplete data.
6. Variance analysis flags an employee whose net pay changed by more than the threshold, and the
   reason is visible on the exception.
7. A run cannot be approved with blocking exceptions outstanding.
8. Approval and finalisation are two distinct recorded acts, even when performed by one person.
9. Finalising locks the period: an attendance correction for a date in it is refused with the
   period, run, and adjustment route named — matching what 04's error message promises.
10. Reopening a finalised run requires super admin and a reason, releases the lock, and marks
    already-published payslips superseded rather than deleting them.
11. A backdated salary increase produces a proposed arrears adjustment for review, and does not alter
    the finalised payslips it relates to.
12. The bank file excludes employees without bank details and lists them separately.
13. Payslip totals reconcile with the sum of lines for every employee in the run; a discrepancy is an
    exception, not a shipped payslip.
14. A notification that a payslip is available contains no figures.
15. The calculation engine's tests run without a database, from fixture snapshots.
