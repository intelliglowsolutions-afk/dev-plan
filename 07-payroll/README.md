# 07 — Payroll

**Priority:** Must-have · **Build order:** 7 of 11 · **Status:** planned (Step 3)

## Purpose

Turn employment facts — a salary, hours worked, overtime, unpaid leave, an allowance, a deduction —
into an amount of money owed to each person for a period, and produce the payslip that explains it.

Payroll is the feature with the lowest tolerance for error in the system. Attendance being wrong is
an argument; payroll being wrong is someone's rent. And unlike every other feature, its output is
consumed outside the system — by banks, by accountants, by tax authorities, by the employee's
mortgage application two years from now.

## The central problem

A payslip issued in March must still be explicable in December, after the employee's salary has
changed twice, their department has been reorganised, the allowance policy has been rewritten, and
the tax rates have moved.

So payroll **snapshots everything at the moment of calculation**: the salary, the components, the
formulas, the attendance figures, the leave, the rates. A finalised payslip does not reference live
data; it contains its own inputs. Recalculating it is possible only while the run is a draft, and
after finalisation a mistake is corrected by an adjustment in a later run — never by an edit.

This is the same discipline as feature 04 (punches vs computed days) and feature 06 (the ledger),
taken one step further: here even the *rules* are snapshotted, because a payslip that changes when a
formula is edited is not a record of anything.

## Scope

**In scope**

- **Pay components**: earnings and deductions, fixed or computed, with a formula language and an
  explicit calculation order.
- **Salary structures**: named sets of components, assigned to employees or groups.
- **Employee compensation**: base salary and component values, date-ranged like feature 02's job
  assignments.
- **Bank details** for payment (moved here from 02 by that feature's D-05).
- **Pay calendar**: periods, pay dates, cut-off dates.
- **Pay runs**: the lifecycle from draft through calculation, review, approval, finalisation, and
  payment marking.
- **Payslips**: generation, publication to employees, PDF, and reissue.
- **Adjustments and arrears**: corrections in a later period, back-pay, off-cycle runs.
- **The period lock** that features 04 and 06 already defer to.
- **Bank payment file** and accounting export.

**Out of scope (owned elsewhere)**

- **Statutory tax and contribution rules.** By the Step 1 decision, payroll is generic and
  configurable: tax is a component with a formula or a bracket table that the company configures.
  The system ships no country's rules and makes no claim to be compliant with any.
- Hours, overtime, absence → [04](../04-attendance-tracking/README.md). Payroll reads
  `attendance_days`; it never recomputes attendance.
- Unpaid leave and encashment → [06](../06-leave-management/README.md). Payroll reads
  `LeaveRequestDay` and the ledger.
- Job, grade, employment type, hire and termination dates → [02](../02-employee-management/README.md).
- Actually moving money. This feature produces a payment file; a human uploads it to the bank.
- Accounting entries. This feature exports; the accounting system posts.

## Files

| File | Contents |
|---|---|
| [requirements.md](./requirements.md) | User stories, functional and non-functional requirements, the run lifecycle, permission keys |
| [data-model.md](./data-model.md) | Prisma models, the snapshot design, period locks, seed and migration notes |
| [api-design.md](./api-design.md) | Components, compensation, runs, payslips, exports, the calculation engine |
| [ui-ux.md](./ui-ux.md) | The run wizard, exception review, payslip, compensation screens |

## Key decisions taken in this plan

| # | Decision | Rationale |
|---|---|---|
| D-01 | **No statutory rules are built in.** Tax, social insurance, and every other deduction are configurable components. | The Step 1 decision. Encoding one country's rules makes the product wrong everywhere else and wrong at home the next time the law changes. |
| D-02 | Formulas are a **restricted expression language** over declared variables — arithmetic, comparisons, `min`/`max`/`round`, bracket-table lookups. Not arbitrary code. | A formula field that can execute code is a remote-code-execution hole with an HR admin's credentials in front of it. The restriction is also what makes formulas testable and explainable. |
| D-03 | **Everything is snapshotted at calculation**: inputs, component definitions, formulas, and rates. A finalised payslip references nothing live. | Reproducibility. A payslip is evidence, and evidence that changes when its source data is edited is not evidence. |
| D-04 | A pay run has an explicit **lifecycle** — `DRAFT` → `CALCULATED` → `APPROVED` → `FINALISED` → `PAID` — and finalisation **locks the period** against attendance and leave changes. | 04 and 06 already refuse edits to locked periods and name this feature as the route for corrections. This is where that promise is kept. |
| D-05 | **Money is `Decimal`, never a float.** Rounding is declared per component and applied at declared points, with a reconciliation check that the lines sum to the total. | Floating-point money produces payslips that are off by a cent and unexplainable. The reconciliation check is what catches a rounding rule applied twice. |
| D-06 | Salary is the **most sensitive data in the system**. It has its own permission set, never appears in email (05 D-06), and payslip access is logged separately from ordinary audit. | An HR admin who can see everyone's pay is a deliberate grant, not a side effect of holding `employee.read`. Feature 02's D-05 split the record for exactly this. |
| D-07 | After finalisation, corrections are **adjustments in a later run**, never edits to the finalised one. | Mirrors 04's corrections and 06's compensating entries. A reissued payslip with different numbers and the same period is how disputes become unresolvable. |
| D-08 | Calculation is **pure and deterministic**, and every payslip line carries a **calculation trail** showing the formula, the inputs, and the result. | Same requirement as 04's reason trail, and for the same reason: "why is this number what it is" must be answerable without a developer. |
| D-09 | Employee compensation is **date-ranged history**, not a mutable field. | "What was she earning in March" must stay answerable after April's raise. Same pattern as 02's job assignments. |
| D-10 | A pay run is **calculated per employee independently** and failures are isolated: one employee's error does not stop the run, it marks that employee as an exception for review. | A run that aborts on employee 34 of 200 is unusable at the exact moment it matters most. |

## The period lock

Features 04 and 06 both already refuse changes to a locked period and both point here. The contract
this feature owes them:

| Question | Answer |
|---|---|
| What locks? | Finalising a pay run locks every date in its period, for the employees in it. |
| What is blocked? | Attendance corrections and recomputes (04 FR-X-09, FR-R-09), leave requests, cancellations, and amendments touching those dates (06 FR-R-08). |
| Who can unlock? | Super admin only, deliberately, with a reason, and it is audited distinctly. Reopening a finalised run is a serious act. |
| What is the alternative? | An adjustment in the next run — which is what the blocked operations' error messages tell the user to do. |

Both features' lock errors already name the period and suggest the adjustment route. This feature
supplies the lock table and the adjustment mechanism that makes those messages true.

## Dependencies

- **Depends on:** 01 (permissions, audit), 02 (employees, employment periods, grade), 03 (currency,
  fiscal year, working days, job runner), 04 (attendance days, overtime), 06 (unpaid leave,
  encashment), 05 (notifying employees that a payslip is available — never the figures themselves).
- **Depended on by:** 08 (employees view their own payslips), 11 (payroll reporting).
- This feature reads from more of the system than any other and writes to none of it. Like feature
  06, that constraint is what makes it buildable.

## Open questions

| ID | Question | Why it matters | Proposed default |
|---|---|---|---|
| OQ-701 | **What are the actual pay components?** Basic, house rent, transport, medical, provident fund, income tax, loan deduction — what exists, and how is each calculated? | Everything in this feature is mechanism. Without the real components, nothing can be seeded or tested, and the formula language cannot be validated against real needs. | Ask before building. The plan ships clearly-labelled examples only. |
| OQ-702 | **Is there statutory tax to deduct, and who supplies the rules?** D-01 says the system does not know them — someone must configure them, and someone must own keeping them current. | A tax component configured once and never updated is worse than none: it produces confidently wrong deductions. | Configured by the company, with a visible "rates last reviewed" date on the component. |
| OQ-703 | **Pay frequency and period** — monthly, fortnightly, weekly? What is the cut-off date relative to the pay date? | Determines the pay calendar and how much of the period's attendance exists when the run happens. A monthly run on the 25th for a period ending the 30th must handle five days that have not happened yet. | Monthly, cut-off on the 25th, paid on the last working day. |
| OQ-704 | How is a **partial month** paid — a joiner, a leaver, or someone unpaid for part of it? Calendar days, working days, or a fixed divisor (e.g. /30)? | This single choice changes every pro-rated payslip and is the most common source of payroll disputes. | Working days in the period. |
| OQ-705 | **Does attendance actually affect pay**, and how? Deduction per absent day, per late arrival, paid overtime? | 04 produces the figures; whether they cost money is a policy decision, and a strict one damages trust quickly if it is wrong. | Unpaid leave and unauthorised absence deduct; lateness does not; approved overtime is paid at a configured multiplier. |
| OQ-706 | Are there **loans, advances, or recoverable payments** deducted over several periods? | A whole sub-feature: a balance, a schedule, and early settlement. Not modelled in v1 beyond a recurring deduction. | Recurring deduction only; flag if a real loan ledger is needed. |
| OQ-707 | Who **approves** a pay run, and is approval separate from finalisation? | Segregation of duties: the person who runs payroll usually should not be the one who approves it. | Separate permissions; the same person may hold both, but the two acts are recorded separately. |
| OQ-708 | What **bank file format** does the company's bank require? | Every bank differs, and a wrong format fails silently at the bank rather than in the system. | A generic CSV, with the real format added once known. |
| OQ-709 | Are payslips **emailed as PDFs** or only made available in the portal? | 05 D-06 forbids sensitive figures in email. A PDF attachment is the standard request and the standard way payslips end up in personal inboxes. | Portal only; email notifies that a payslip is ready and links to it. |
| OQ-710 | Is there a **13th month / bonus** cycle, and is it part of a normal run or an off-cycle one? | Affects the pay calendar and whether off-cycle runs are needed in v1. | Off-cycle run support is built; the specific bonus policy is configuration. |
| OQ-711 | Does payroll need to produce **accounting entries** (cost centre, GL codes), or is a summary export enough? | Full GL mapping is a significant addition and is usually wanted eventually. | Summary export by department in v1; component-level GL codes are modelled but optional. |
| OQ-712 | **Currency** — single, as feature 03 assumes? Anyone paid in another currency? | 03 D-02 and this feature both assume one. Multi-currency payroll is a different product. | Single currency. |
