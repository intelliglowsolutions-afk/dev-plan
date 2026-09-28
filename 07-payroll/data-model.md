# 07 — Payroll — Data Model

Conventions as in features 01–06. **Every monetary field is `Decimal(14,2)`. There are no floats
anywhere in this feature** (D-05, NFR-03).

> **Revised 2026-09-18 — multi-tenant (OQ-301).** [MULTI-TENANCY.md](../MULTI-TENANCY.md) governs.
> - Every model gains `tenantId`. `PayComponent.code`, `SalaryStructure.name` and `PayPeriod`'s
>   date range become unique per tenant; `PeriodLock` is looked up by `[tenantId, employeeId, date]`.
> - **Pay calendars, runs and locks are per tenant.** One tenant finalising September must not lock
>   another tenant's attendance — the lock helper's tenant scoping is load-bearing, not cosmetic.
> - Example components become per-tenant provisioning options (D-T-07). **Nothing that looks like a
>   tax table is seeded** — OQ-702 confirmed tax is configurable, and a plausible wrong rate is
>   worse than an empty one.
> - Payslip PDFs are now also emailed (OQ-709) — see FR-L-05 in `requirements.md` for the required
>   controls.
> - `PayrollExport` and `PayslipAccessLog` rows are retained, not swept (OQ-1002).
> - **OQ-701 is still outstanding**: without the real components, nothing here can be seeded or
>   tested, and the formula grammar is unvalidated against real formulas.

## Entity overview

```
PayComponent ──*── SalaryStructureComponent ──*── SalaryStructure
     │                                                  │
     ├── BracketTable ──*── BracketRow                  │
     │                                                  ▼
     │                         EmployeeCompensation (date-ranged) ──→ Employee
     │                         EmployeeBankAccount             ──→ Employee
     ▼
PayPeriod ──*── PayRun ──*── Payslip ──*── PayslipLine
                   │              │
                   │              └── inputSnapshot (JSON: everything it was computed from)
                   ├──*── PayRunException
                   ├──*── PayrollAdjustment  (applied to a future run)
                   └──*── PeriodLock         (read by 04 and 06)
                   └──*── PayrollExport
```

The arrow that matters most is the one that **is not there**: a finalised `Payslip` has no live
dependency on `PayComponent`, `EmployeeCompensation`, `AttendanceDay`, or anything else. It carries
its own inputs (D-03). The FKs that do exist are for navigation, not for rendering.

## Enums

```prisma
enum ComponentType {
  EARNING
  DEDUCTION
}

enum CalculationMethod {
  FIXED               // from the employee's compensation record
  FORMULA
  PERCENTAGE          // of another component or a named base
  BRACKET             // bracket-table lookup
  ATTENDANCE_DERIVED  // from 04
  LEAVE_DERIVED       // from 06
  MANUAL              // entered per employee per run
}

enum RoundingRule {
  NONE
  NEAREST_UNIT
  NEAREST_HUNDREDTH
  UP
  DOWN
}

enum BracketBasis {
  MARGINAL
  CUMULATIVE
}

enum PayRunStatus {
  DRAFT
  CALCULATING
  CALCULATED
  APPROVED
  FINALISED
  PAID
  CANCELLED
}

enum PayRunType {
  REGULAR
  OFF_CYCLE          // bonus, final settlement
}

enum ExceptionSeverity {
  BLOCKING           // the run cannot be approved
  WARNING            // acknowledgeable with a note
}

enum ExceptionKind {
  NO_COMPENSATION
  NO_BANK_DETAILS
  NEGATIVE_NET
  UNRESOLVED_ATTENDANCE     // 04 UNKNOWN days
  PENDING_CORRECTION
  PENDING_LEAVE
  VARIANCE
  FORMULA_ERROR
  MISSING_MANUAL_INPUT
  RECONCILIATION_MISMATCH
}

enum AdjustmentKind {
  CORRECTION
  ARREARS
  CUTOFF_SETTLEMENT
  BONUS
  RECOVERY
}
```

## Components and formulas

```prisma
model PayComponent {
  id               Int               @id @default(autoincrement())
  code             String            @unique          // referenced by formulas: BASIC, HRA
  name             String
  type             ComponentType
  method           CalculationMethod

  // Evaluated low to high. A component may only reference lower orders (FR-C-05).
  calculationOrder Int               @map("calculation_order")

  formula          String?                             // restricted grammar (D-02)
  percentageOf     String?           @map("percentage_of")   // a component code or "GROSS"
  percentageValue  Decimal?          @map("percentage_value") @db.Decimal(7, 4)
  bracketTableId   Int?              @map("bracket_table_id")

  roundingRule     RoundingRule      @default(NEAREST_UNIT) @map("rounding_rule")
  isTaxable        Boolean           @default(true) @map("is_taxable")
  showOnPayslip    Boolean           @default(true) @map("show_on_payslip")
  glCode           String?           @map("gl_code")

  // Surfaced in the UI so a stale statutory table is visible, not assumed current (FR-C-09).
  ratesReviewedOn  DateTime?         @map("rates_reviewed_on") @db.Date

  isActive         Boolean           @default(true) @map("is_active")
  createdAt        DateTime          @default(now()) @map("created_at")
  updatedAt        DateTime          @updatedAt @map("updated_at")

  bracketTable     BracketTable?     @relation(fields: [bracketTableId], references: [id])
  structures       SalaryStructureComponent[]

  @@index([calculationOrder])
  @@map("pay_components")
}

model BracketTable {
  id          Int          @id @default(autoincrement())
  name        String       @unique
  basis       BracketBasis @default(MARGINAL)
  description String?
  reviewedOn  DateTime?    @map("reviewed_on") @db.Date
  createdAt   DateTime     @default(now()) @map("created_at")
  updatedAt   DateTime     @updatedAt @map("updated_at")

  rows        BracketRow[]
  components  PayComponent[]

  @@map("bracket_tables")
}

model BracketRow {
  id          Int          @id @default(autoincrement())
  tableId     Int          @map("table_id")
  rowOrder    Int          @map("row_order")
  upperBound  Decimal?     @map("upper_bound") @db.Decimal(14, 2)   // null = no ceiling
  rate        Decimal      @db.Decimal(7, 4)
  fixedAmount Decimal      @default(0) @map("fixed_amount") @db.Decimal(14, 2)

  table       BracketTable @relation(fields: [tableId], references: [id], onDelete: Cascade)

  @@unique([tableId, rowOrder])
  @@map("bracket_rows")
}

model SalaryStructure {
  id          Int      @id @default(autoincrement())
  name        String   @unique
  description String?
  isActive    Boolean  @default(true) @map("is_active")
  createdAt   DateTime @default(now()) @map("created_at")
  updatedAt   DateTime @updatedAt @map("updated_at")

  components  SalaryStructureComponent[]
  compensations EmployeeCompensation[]

  @@map("salary_structures")
}

model SalaryStructureComponent {
  structureId  Int             @map("structure_id")
  componentId  Int             @map("component_id")
  defaultValue Decimal?        @map("default_value") @db.Decimal(14, 2)
  isRequired   Boolean         @default(false) @map("is_required")

  structure    SalaryStructure @relation(fields: [structureId], references: [id], onDelete: Cascade)
  component    PayComponent    @relation(fields: [componentId], references: [id], onDelete: Restrict)

  @@id([structureId, componentId])
  @@map("salary_structure_components")
}
```

**Why `formula` is a string and not structured JSON.** It is authored and read as an expression —
`BASIC * 0.4` — and parsed into an AST at validation and at evaluation. Storing an AST would make it
unreadable in the database and no safer, since the parser is the thing enforcing the grammar either
way (D-02).

## Employee compensation

```prisma
model EmployeeCompensation {
  id            Int              @id @default(autoincrement())
  employeeId    Int              @map("employee_id")
  structureId   Int?             @map("structure_id")

  baseSalary    Decimal          @map("base_salary") @db.Decimal(14, 2)
  // Per-component fixed values for FIXED components: { "HRA": 12000, "TRANSPORT": 3000 }
  componentValues Json?          @map("component_values")

  effectiveFrom DateTime         @map("effective_from") @db.Date
  effectiveTo   DateTime?        @map("effective_to") @db.Date
  reason        String?                                  // "Annual increment", "Promotion"
  isBackdated   Boolean          @default(false) @map("is_backdated")

  createdById   Int?             @map("created_by_id")
  createdAt     DateTime         @default(now()) @map("created_at")

  employee      Employee         @relation(fields: [employeeId], references: [id], onDelete: Cascade)
  structure     SalaryStructure? @relation(fields: [structureId], references: [id])

  @@index([employeeId, effectiveFrom])
  @@map("employee_compensation")
}

model EmployeeBankAccount {
  id            Int      @id @default(autoincrement())
  employeeId    Int      @map("employee_id")
  accountName   String   @map("account_name")
  accountNumber String   @map("account_number")
  iban          String?
  bankName      String   @map("bank_name")
  branch        String?
  isActive      Boolean  @default(true) @map("is_active")

  createdById   Int?     @map("created_by_id")
  createdAt     DateTime @default(now()) @map("created_at")
  updatedAt     DateTime @updatedAt @map("updated_at")

  employee      Employee @relation(fields: [employeeId], references: [id], onDelete: Cascade)

  @@index([employeeId, isActive])
  @@map("employee_bank_accounts")
}
```

Bank accounts are versioned by `isActive` rather than deleted, and a change notifies the employee
(FR-S-06). A silently changed bank account is the classic payroll fraud, and the notification is the
cheapest control against it.

## Periods, runs, and locks

```prisma
model PayPeriod {
  id          Int       @id @default(autoincrement())
  name        String    @unique                    // "September 2026"
  startDate   DateTime  @map("start_date") @db.Date
  endDate     DateTime  @map("end_date") @db.Date
  cutOffDate  DateTime  @map("cut_off_date") @db.Date
  payDate     DateTime  @map("pay_date") @db.Date
  fiscalYear  String    @map("fiscal_year")
  createdAt   DateTime  @default(now()) @map("created_at")

  runs        PayRun[]

  @@unique([startDate, endDate])
  @@index([payDate])
  @@map("pay_periods")
}

model PayRun {
  id            Int          @id @default(autoincrement())
  periodId      Int          @map("period_id")
  type          PayRunType   @default(REGULAR)
  status        PayRunStatus @default(DRAFT)
  name          String?                            // off-cycle runs need one

  employeeCount Int          @default(0) @map("employee_count")
  totalGross    Decimal      @default(0) @map("total_gross") @db.Decimal(16, 2)
  totalDeductions Decimal    @default(0) @map("total_deductions") @db.Decimal(16, 2)
  totalNet      Decimal      @default(0) @map("total_net") @db.Decimal(16, 2)
  exceptionCount Int         @default(0) @map("exception_count")

  calculatedAt  DateTime?    @map("calculated_at")
  calculatedById Int?        @map("calculated_by_id")
  approvedAt    DateTime?    @map("approved_at")
  approvedById  Int?         @map("approved_by_id")
  finalisedAt   DateTime?    @map("finalised_at")
  finalisedById Int?         @map("finalised_by_id")
  publishedAt   DateTime?    @map("published_at")
  paidAt        DateTime?    @map("paid_at")

  reopenedAt    DateTime?    @map("reopened_at")
  reopenedById  Int?         @map("reopened_by_id")
  reopenReason  String?      @map("reopen_reason")

  createdAt     DateTime     @default(now()) @map("created_at")

  period        PayPeriod    @relation(fields: [periodId], references: [id])
  payslips      Payslip[]
  exceptions    PayRunException[]
  locks         PeriodLock[]
  exports       PayrollExport[]

  @@index([periodId, status])
  @@map("pay_runs")
}

model PeriodLock {
  id         Int      @id @default(autoincrement())
  payRunId   Int      @map("pay_run_id")
  employeeId Int      @map("employee_id")
  startDate  DateTime @map("start_date") @db.Date
  endDate    DateTime @map("end_date") @db.Date
  releasedAt DateTime? @map("released_at")
  releasedById Int?   @map("released_by_id")
  releaseReason String? @map("release_reason")
  createdAt  DateTime @default(now()) @map("created_at")

  payRun     PayRun   @relation(fields: [payRunId], references: [id], onDelete: Cascade)

  // The query 04 and 06 run on every correction and leave request (FR-K-02).
  @@index([employeeId, startDate, endDate])
  @@map("period_locks")
}
```

**Why the lock is per employee, not per period.** An employee onboarded after finalisation must be
able to have their history entered for that period (FR-K-05), and an off-cycle run for three people
should not freeze the other 197. Per-employee rows cost more space and answer the question correctly.

## Payslips

```prisma
model Payslip {
  id            Int       @id @default(autoincrement())
  payRunId      Int       @map("pay_run_id")
  employeeId    Int       @map("employee_id")
  periodId      Int       @map("period_id")

  // Identity as at the period, snapshotted — the payslip must render with no other table (NFR-08).
  employeeCode  String    @map("employee_code")
  employeeName  String    @map("employee_name")
  departmentName String?  @map("department_name")
  positionTitle String?   @map("position_title")

  grossPay      Decimal   @map("gross_pay") @db.Decimal(14, 2)
  totalDeductions Decimal @map("total_deductions") @db.Decimal(14, 2)
  netPay        Decimal   @map("net_pay") @db.Decimal(14, 2)
  currency      String

  // Everything the calculation consumed: compensation, component definitions and formulas,
  // attendance and leave figures, period metadata, manual inputs (D-03, FR-X-01).
  inputSnapshot Json      @map("input_snapshot")

  bankAccountMasked String? @map("bank_account_masked")
  isSuperseded  Boolean   @default(false) @map("is_superseded")
  supersededByPayslipId Int? @map("superseded_by_payslip_id")

  publishedAt   DateTime? @map("published_at")
  createdAt     DateTime  @default(now()) @map("created_at")

  payRun        PayRun    @relation(fields: [payRunId], references: [id], onDelete: Cascade)
  employee      Employee  @relation(fields: [employeeId], references: [id])
  lines         PayslipLine[]
  accessLogs    PayslipAccessLog[]

  @@unique([payRunId, employeeId])
  @@index([employeeId, periodId])
  @@map("payslips")
}

model PayslipLine {
  id             Int           @id @default(autoincrement())
  payslipId      Int           @map("payslip_id")

  // Snapshotted, not FK-rendered: the component may be renamed or deactivated later.
  componentCode  String        @map("component_code")
  componentName  String        @map("component_name")
  type           ComponentType
  displayOrder   Int           @map("display_order")

  amount         Decimal       @db.Decimal(14, 2)
  // How this number came about: formula, substituted inputs, intermediate, rounding (D-08).
  calculationTrail Json?       @map("calculation_trail")

  adjustmentId   Int?          @map("adjustment_id")   // set when the line is an adjustment
  note           String?                                // shown to the employee (FR-A-02)

  payslip        Payslip       @relation(fields: [payslipId], references: [id], onDelete: Cascade)

  @@index([payslipId, displayOrder])
  @@map("payslip_lines")
}

model PayslipAccessLog {
  id         Int      @id @default(autoincrement())
  payslipId  Int      @map("payslip_id")
  userId     Int      @map("user_id")
  action     String                                 // "VIEW", "DOWNLOAD_PDF"
  ipAddress  String?  @map("ip_address")
  createdAt  DateTime @default(now()) @map("created_at")

  payslip    Payslip  @relation(fields: [payslipId], references: [id], onDelete: Cascade)

  @@index([payslipId, createdAt])
  @@index([userId, createdAt])
  @@map("payslip_access_logs")
}
```

**Why there is a separate access log rather than reusing 01's audit table.** "Who looked at whose
pay" is asked differently from "who changed what", it is queried by payslip and by user rather than
by entity, and it is written on reads — which would otherwise flood the audit log with entries that
drown the changes it exists to record (D-06, FR-L-06).

## Exceptions, adjustments, exports

```prisma
model PayRunException {
  id           Int               @id @default(autoincrement())
  payRunId     Int               @map("pay_run_id")
  employeeId   Int               @map("employee_id")
  kind         ExceptionKind
  severity     ExceptionSeverity
  message      String
  detail       Json?                                  // e.g. variance figures, the formula error

  acknowledgedById Int?          @map("acknowledged_by_id")
  acknowledgedAt   DateTime?     @map("acknowledged_at")
  acknowledgeNote  String?       @map("acknowledge_note")
  resolvedAt   DateTime?         @map("resolved_at")

  payRun       PayRun            @relation(fields: [payRunId], references: [id], onDelete: Cascade)

  @@index([payRunId, severity])
  @@index([employeeId])
  @@map("pay_run_exceptions")
}

model PayrollAdjustment {
  id            Int            @id @default(autoincrement())
  employeeId    Int            @map("employee_id")
  kind          AdjustmentKind
  componentCode String?        @map("component_code")
  amount        Decimal        @db.Decimal(14, 2)
  reason        String                                 // shown to the employee (FR-A-02)

  // What it corrects, and where it will be applied.
  relatesToPeriodId Int?       @map("relates_to_period_id")
  appliedInRunId    Int?       @map("applied_in_run_id")
  appliedAt         DateTime?  @map("applied_at")

  isProposed    Boolean        @default(false) @map("is_proposed")   // arrears await review
  createdById   Int?           @map("created_by_id")
  createdAt     DateTime       @default(now()) @map("created_at")

  employee      Employee       @relation(fields: [employeeId], references: [id], onDelete: Cascade)
  appliedInRun  PayRun?        @relation(fields: [appliedInRunId], references: [id])

  @@index([employeeId, appliedAt])
  @@index([appliedInRunId])
  @@map("payroll_adjustments")
}

model PayrollExport {
  id         Int      @id @default(autoincrement())
  payRunId   Int      @map("pay_run_id")
  kind       String                                   // "BANK", "ACCOUNTING"
  format     String
  rowCount   Int      @map("row_count")
  totalAmount Decimal @map("total_amount") @db.Decimal(16, 2)
  checksum   String                                   // of the generated file (FR-E-03)
  exportedById Int?   @map("exported_by_id")
  exportedAt DateTime @default(now()) @map("exported_at")

  payRun     PayRun   @relation(fields: [payRunId], references: [id])

  @@index([payRunId, kind])
  @@map("payroll_exports")
}
```

## Design notes

**The `inputSnapshot` is the feature.** It holds the employee's compensation as it stood, each
component's definition and formula text, the attendance and leave figures, the period's working-day
count, and any manual inputs. It is the difference between a payslip that can be explained in two
years and one that can only be re-derived from data that has since changed (D-03).

Its size matters: roughly 2–5 KB per payslip, so ~1 MB a year for 200 employees. That is a trivial
cost for permanent explicability, and it is the right trade every time.

**Why `PayslipLine` stores the component code and name rather than an FK.** A component renamed from
"Transport" to "Conveyance" must not silently rewrite last year's payslips. The FK to
`PayComponent` is deliberately absent.

**Why exceptions are rows, not a computed list.** They are acknowledged, annotated, and resolved by
people over the course of a run, and that state has to persist across recalculations (FR-R-06).

**Why adjustments carry `isProposed`.** Backdated compensation generates arrears automatically
(FR-A-03), and automatically paying a computed difference without review is precisely the behaviour
that produces a six-figure mistake. Proposed adjustments wait for a human.

## Volume

| Table | Rows/year at 200 employees | Notes |
|---|---|---|
| `payslips` | 2 400 | 12 periods × 200 |
| `payslip_lines` | ~24 000 | ~10 components each |
| `payslip_access_logs` | ~10 000 | Written on every view |
| `pay_run_exceptions` | hundreds | |
| `employee_compensation` | ~250 | One per raise |
| `period_locks` | 2 400 | One per employee per finalised run |

Small. Payroll's risk is never volume; it is correctness and reproducibility.

## Migration and seed notes

Migration `payroll`:

1. Create the tables above.
2. Seed the permission keys and role grants from `requirements.md` — note that `MANAGER` receives
   none.
3. Seed 05's notification types: `payroll.payslip_published`, `payroll.run_needs_approval`,
   `payroll.bank_details_changed`, `payroll.adjustment_proposed`. **None of them may carry an
   amount** (05 D-06, FR-L-05) — the denylist in 05's template validation must include every
   monetary variable.
4. Seed **example** components and one structure, clearly labelled as examples, following the
   precedent set by 04's example shift and 06's example leave types. Do **not** seed anything that
   looks like a tax table; a plausible-looking wrong tax rate is worse than an empty one (OQ-702).
5. Generate pay periods for the current and next fiscal year from the configured frequency.
6. Settings added to 03's catalogue: `payroll.frequency`, `payroll.cutOffDay`, `payroll.payDay`,
   `payroll.varianceThresholdPercent` (20), `payroll.varianceThresholdAmount`,
   `payroll.partialMonthBasis` (`WORKING_DAYS`), `payroll.incompleteAttendanceAssumption`
   (`ASSUME_PRESENT`), `payroll.overtimeMultiplier`, `payroll.bankFileFormat`.
7. Wire `isPeriodLocked` into 04 and 06, replacing the stubs those features shipped with. Both were
   specified to call it; until this migration they had nothing to call.

## Open questions

| ID | Question |
|---|---|
| OQ-701 | The real components. Everything above is shape; the content decides whether the formula language is sufficient. |
| OQ-702 | Who owns keeping a configured tax table current. The model surfaces `ratesReviewedOn`; it cannot make anyone look at it. |
| OQ-706 | Loans and advances would add a balance and a schedule — `EmployeeLoan` with `LoanInstalment` — rather than a recurring deduction. Worth knowing before components are configured to fake it. |
| OQ-713 | Should `inputSnapshot` be compressed or moved out of the row? At 5 KB × 2 400/year it does not matter yet, and would matter at 5 000 employees. |
| OQ-714 | Whether `payslip_access_logs` should record an employee viewing their *own* payslip. It doubles the volume and answers a question nobody asks. Proposed: log others' access only, and say so. |
