# 06 — Leave Management — Data Model

Conventions as in features 01–05. Leave days are `@db.Date`; fractions are `Decimal(5,2)`.

> **Revised 2026-09-18 — multi-tenant (OQ-301).** [MULTI-TENANCY.md](../MULTI-TENANCY.md) governs.
> - Every model gains `tenantId`. `LeaveType.name`/`code` and `LeavePolicy` become per tenant, as
>   does the ledger's idempotency key `[employee, type, periodKey, kind]`.
> - **Accrual, carry-over and expiry run per tenant**, isolating failure — one tenant's accrual
>   failing must not stop the other thirty-nine (D-T-06.1) — and using **that tenant's leave year
>   and timezone**.
> - The example leave types become **per-tenant provisioning options**, offered during setup rather
>   than seeded at migration time (D-T-07).
> - Confirmed types (OQ-601): **Annual, Casual, Medical**. Entitlement days are still outstanding
>   (OQ-601b), so policies cannot yet be seeded.
> - Multi-step configurable approval chains (OQ-606) make `LeavePolicy.approvalSteps` a real
>   structure rather than the placeholder it was drafted as.
> - The ledger becomes **append-only at the database level** (OQ-613 answered yes): revoke UPDATE
>   and DELETE on `leave_ledger_entries` for the application role.

## Entity overview

```
LeaveType ──*── LeavePolicy ──*── LeavePolicyAssignment ──→ (employment type / department / grade / employee)
    │                │
    │                └──────────────┐
    ▼                               ▼
LeaveLedgerEntry ←──────────── LeaveRequest ──*── LeaveRequestDay
    │  (the truth)                   │              (what 04 and 07 read)
    │                                └──*── LeaveApprovalStep
    ▼
LeaveBalanceSnapshot  (cache, rebuilt from the ledger, never written directly)

LeaveDelegation        (who covers whose approvals, and when)
LeaveEngineRun         (accrual / carry-over / expiry runs, for traceability)
```

`LeaveRequestDay` is the integration surface. Features 04 and 07 read it; neither re-derives a date
range, and neither touches the ledger.

## Enums

```prisma
enum LeaveUnit {
  DAY        // D-03. HOUR deliberately absent until OQ-603 is answered.
}

enum AccrualMethod {
  ANNUAL_GRANT      // whole entitlement at the start of the leave year
  MONTHLY_ACCRUAL   // 1/12 each month
  PER_PERIOD        // a fixed amount each accrual period
  NONE              // no automatic entitlement; balance moves only by adjustment
}

enum LeaveYearBasis {
  CALENDAR
  FISCAL            // from 03's company.fiscalYearStart
  ANNIVERSARY       // per-employee, from hire date — materially more work (OQ-602)
}

enum LeaveRequestStatus {
  DRAFT
  PENDING
  APPROVED
  REJECTED
  CANCELLED
  WITHDRAWN
}

enum LeaveLedgerKind {
  GRANT
  ACCRUAL
  CARRY_OVER
  CARRY_OVER_EXPIRY
  RESERVATION
  RESERVATION_RELEASE
  TAKEN
  REFUND
  ADJUSTMENT
  ENCASHMENT
  FORFEIT_ON_EXIT
}

enum ApprovalStepStatus {
  PENDING
  APPROVED
  REJECTED
  SKIPPED        // superseded by a rejection earlier in the chain, or by an override
}

enum DayFraction {
  FULL
  FIRST_HALF
  SECOND_HALF
}

enum EngineRunType {
  ACCRUAL
  CARRY_OVER
  EXPIRY
  SETTLEMENT
}
```

## Types and policies

```prisma
model LeaveType {
  id             Int      @id @default(autoincrement())
  name           String   @unique
  code           String   @unique
  colour         String?

  isPaid         Boolean  @default(true) @map("is_paid")
  deductsBalance Boolean  @default(true) @map("deducts_balance")   // FR-T-02
  requiresApproval Boolean @default(true) @map("requires_approval")
  allowsHalfDay  Boolean  @default(true) @map("allows_half_day")
  allowsAttachment Boolean @default(true) @map("allows_attachment")
  // Whether 04 should treat the day as present for attendance purposes.
  countsAsPresent Boolean @default(true) @map("counts_as_present")
  // Hide the type's name from colleagues' calendars (FR-T-07).
  isConfidential Boolean  @default(false) @map("is_confidential")

  isActive       Boolean  @default(true) @map("is_active")
  createdAt      DateTime @default(now()) @map("created_at")
  updatedAt      DateTime @updatedAt @map("updated_at")

  policies       LeavePolicy[]
  requests       LeaveRequest[]
  ledger         LeaveLedgerEntry[]
  snapshots      LeaveBalanceSnapshot[]

  @@map("leave_types")
}

model LeavePolicy {
  id              Int            @id @default(autoincrement())
  leaveTypeId     Int            @map("leave_type_id")
  name            String

  entitlementDays Decimal        @map("entitlement_days") @db.Decimal(5, 2)
  accrualMethod   AccrualMethod  @default(ANNUAL_GRANT) @map("accrual_method")
  yearBasis       LeaveYearBasis @default(CALENDAR) @map("year_basis")

  carryOverAllowed Boolean       @default(false) @map("carry_over_allowed")
  carryOverCapDays Decimal?      @map("carry_over_cap_days") @db.Decimal(5, 2)
  carryOverExpiryMonthDay String? @map("carry_over_expiry_month_day")   // "03-31"

  proRateOnJoin   Boolean        @default(true) @map("pro_rate_on_join")
  proRateOnExit   Boolean        @default(true) @map("pro_rate_on_exit")
  roundingRule    String         @default("NEAREST_HALF") @map("rounding_rule")

  // Restrictions
  availableAfterProbation Boolean @default(false) @map("available_after_probation")
  minNoticeDays   Int?           @map("min_notice_days")
  maxConsecutiveDays Int?        @map("max_consecutive_days")
  negativeBalanceLimitDays Decimal? @map("negative_balance_limit_days") @db.Decimal(5, 2)
  requiresAttachmentAfterDays Int? @map("requires_attachment_after_days")

  // Approval (OQ-606). Null = single step to the employee's manager.
  approvalSteps   Json?          @map("approval_steps")
  escalationDays  Int?           @map("escalation_days")

  effectiveFrom   DateTime       @map("effective_from") @db.Date
  effectiveTo     DateTime?      @map("effective_to") @db.Date
  isActive        Boolean        @default(true) @map("is_active")
  createdAt       DateTime       @default(now()) @map("created_at")
  updatedAt       DateTime       @updatedAt @map("updated_at")

  leaveType       LeaveType      @relation(fields: [leaveTypeId], references: [id])
  assignments     LeavePolicyAssignment[]

  @@index([leaveTypeId, effectiveFrom])
  @@map("leave_policies")
}

model LeavePolicyAssignment {
  id             Int             @id @default(autoincrement())
  policyId       Int             @map("policy_id")

  // Exactly one targeting dimension is set. Specificity order, most specific first:
  // employee > position grade > department > employment type > company-wide (all null).
  employeeId     Int?            @map("employee_id")
  departmentId   Int?            @map("department_id")
  employmentType EmploymentType? @map("employment_type")
  positionGrade  String?         @map("position_grade")

  effectiveFrom  DateTime        @map("effective_from") @db.Date
  effectiveTo    DateTime?       @map("effective_to") @db.Date
  createdAt      DateTime        @default(now()) @map("created_at")

  policy         LeavePolicy     @relation(fields: [policyId], references: [id], onDelete: Cascade)

  @@index([policyId])
  @@index([employeeId])
  @@index([departmentId])
  @@map("leave_policy_assignments")
}
```

**Policy resolution** for (employee, type, date): collect assignments in force on that date whose
target matches the employee, and take the most specific (FR-T-04). Ties are impossible by
construction, since each level is more specific than the last. No match → the type is unavailable to
that employee, which is a legitimate configuration, not an error.

`approvalSteps` is JSON rather than a table because a step is a small rule — `{ "approver":
"MANAGER" }`, `{ "approver": "PERMISSION", "key": "leave.approve", "afterDays": 5 }` — and modelling
two or three of those as rows buys nothing. The **resolved** chain for an actual request *is* a
table, because it is a record of what happened (D-06).

## The ledger

```prisma
model LeaveLedgerEntry {
  id           Int             @id @default(autoincrement())
  employeeId   Int             @map("employee_id")
  leaveTypeId  Int             @map("leave_type_id")
  // The leave year this entry belongs to, as its start date (FR-B-07).
  leaveYear    DateTime        @map("leave_year") @db.Date

  kind         LeaveLedgerKind
  // Signed: grants and refunds positive, reservations and taken negative.
  days         Decimal         @db.Decimal(6, 2)

  // Everything needed to explain the entry without re-deriving it (FR-B-06).
  description  String
  effectiveOn  DateTime        @map("effective_on") @db.Date
  expiresOn    DateTime?       @map("expires_on") @db.Date      // CARRY_OVER entries

  leaveRequestId Int?          @map("leave_request_id")
  engineRunId    Int?          @map("engine_run_id")
  // A compensating entry points at what it corrects (FR-B-05).
  correctsEntryId Int?         @map("corrects_entry_id")
  // Makes engine runs idempotent: unique per (employee, type, period, kind) — D-08, FR-Y-01.
  periodKey    String?         @map("period_key")

  createdById  Int?            @map("created_by_id")
  createdAt    DateTime        @default(now()) @map("created_at")

  employee     Employee        @relation(fields: [employeeId], references: [id], onDelete: Cascade)
  leaveType    LeaveType       @relation(fields: [leaveTypeId], references: [id])
  request      LeaveRequest?   @relation(fields: [leaveRequestId], references: [id])
  corrects     LeaveLedgerEntry? @relation("Correction", fields: [correctsEntryId], references: [id])
  correctedBy  LeaveLedgerEntry[] @relation("Correction")
  engineRun    LeaveEngineRun? @relation(fields: [engineRunId], references: [id])

  @@unique([employeeId, leaveTypeId, periodKey, kind])
  @@index([employeeId, leaveTypeId, leaveYear])
  @@index([leaveRequestId])
  @@index([expiresOn])
  @@map("leave_ledger_entries")
}

model LeaveBalanceSnapshot {
  employeeId   Int       @map("employee_id")
  leaveTypeId  Int       @map("leave_type_id")
  leaveYear    DateTime  @map("leave_year") @db.Date

  entitledDays Decimal   @default(0) @map("entitled_days") @db.Decimal(6, 2)
  usedDays     Decimal   @default(0) @map("used_days") @db.Decimal(6, 2)
  pendingDays  Decimal   @default(0) @map("pending_days") @db.Decimal(6, 2)
  availableDays Decimal  @default(0) @map("available_days") @db.Decimal(6, 2)

  // Cache coherence: the last entry folded in, and when (FR-B-08).
  lastEntryId  Int?      @map("last_entry_id")
  rebuiltAt    DateTime  @default(now()) @map("rebuilt_at")

  employee     Employee  @relation(fields: [employeeId], references: [id], onDelete: Cascade)
  leaveType    LeaveType @relation(fields: [leaveTypeId], references: [id])

  @@id([employeeId, leaveTypeId, leaveYear])
  @@map("leave_balance_snapshots")
}
```

**The unique constraint is the whole idempotency mechanism.** `[employeeId, leaveTypeId, periodKey,
kind]` with `periodKey` set to `"2026"` for an annual grant or `"2026-03"` for a monthly accrual
means a second run inserts nothing — enforced by the database, not by the engine remembering
(FR-Y-01, D-08). Entries with a null `periodKey` (reservations, takings, adjustments) are unaffected,
since Postgres treats nulls as distinct in unique constraints.

**The snapshot is a cache with a coherence marker.** `lastEntryId` lets the consistency check
(FR-B-08) detect drift cheaply: if the highest entry id for that key exceeds it, the snapshot is
stale. Drift is reported, not silently corrected, because silent correction hides the bug that
caused it.

## Requests and approval

```prisma
model LeaveRequest {
  id             Int                @id @default(autoincrement())
  employeeId     Int                @map("employee_id")
  leaveTypeId    Int                @map("leave_type_id")

  startDate      DateTime           @map("start_date") @db.Date
  endDate        DateTime           @map("end_date") @db.Date
  startFraction  DayFraction        @default(FULL) @map("start_fraction")
  endFraction    DayFraction        @default(FULL) @map("end_fraction")

  // Computed at submission, recomputed at approval (FR-R-03).
  dayCount       Decimal            @map("day_count") @db.Decimal(5, 2)
  dayCountAtApproval Decimal?       @map("day_count_at_approval") @db.Decimal(5, 2)

  status         LeaveRequestStatus @default(PENDING)
  reason         String?
  attachmentDocumentId Int?         @map("attachment_document_id")

  // Set when HR records leave for someone else (FR-R-12).
  createdForOther Boolean           @default(false) @map("created_for_other")
  isBackdated    Boolean            @default(false) @map("is_backdated")

  requestedById  Int?               @map("requested_by_id")
  requestedAt    DateTime           @default(now()) @map("requested_at")
  decidedAt      DateTime?          @map("decided_at")
  decisionNote   String?            @map("decision_note")
  cancelledAt    DateTime?          @map("cancelled_at")
  cancelledById  Int?               @map("cancelled_by_id")
  cancellationReason String?        @map("cancellation_reason")

  employee       Employee           @relation(fields: [employeeId], references: [id], onDelete: Cascade)
  leaveType      LeaveType          @relation(fields: [leaveTypeId], references: [id])
  days           LeaveRequestDay[]
  steps          LeaveApprovalStep[]
  ledgerEntries  LeaveLedgerEntry[]

  @@index([employeeId, startDate])
  @@index([status, requestedAt])
  @@index([startDate, endDate])
  @@map("leave_requests")
}

model LeaveRequestDay {
  id         Int          @id @default(autoincrement())
  requestId  Int          @map("request_id")
  employeeId Int          @map("employee_id")     // denormalised: 04 and 07 query by employee+date
  leaveTypeId Int         @map("leave_type_id")
  date       DateTime     @db.Date
  fraction   DayFraction  @default(FULL)
  days       Decimal      @db.Decimal(3, 2)       // 1.00 or 0.50
  isPaid     Boolean      @default(true) @map("is_paid")

  request    LeaveRequest @relation(fields: [requestId], references: [id], onDelete: Cascade)

  // One approved leave day per employee per date — FR-R-07, enforced by a partial index
  // on approved requests only (added in the migration; Prisma cannot express it).
  @@index([employeeId, date])
  @@index([date])
  @@map("leave_request_days")
}

model LeaveApprovalStep {
  id           Int                @id @default(autoincrement())
  requestId    Int                @map("request_id")
  stepOrder    Int                @map("step_order")

  approverUserId Int?             @map("approver_user_id")
  // Set when a delegate acted in the approver's place (FR-A-06).
  delegatedFromUserId Int?        @map("delegated_from_user_id")
  isOverride   Boolean            @default(false) @map("is_override")

  status       ApprovalStepStatus @default(PENDING)
  decidedById  Int?               @map("decided_by_id")
  decidedAt    DateTime?          @map("decided_at")
  note         String?
  escalatedAt  DateTime?          @map("escalated_at")

  request      LeaveRequest       @relation(fields: [requestId], references: [id], onDelete: Cascade)

  @@unique([requestId, stepOrder])
  @@index([approverUserId, status])
  @@map("leave_approval_steps")
}

model LeaveDelegation {
  id            Int      @id @default(autoincrement())
  fromUserId    Int      @map("from_user_id")
  toUserId      Int      @map("to_user_id")
  startDate     DateTime @map("start_date") @db.Date
  endDate       DateTime @map("end_date") @db.Date
  note          String?
  createdAt     DateTime @default(now()) @map("created_at")

  @@index([fromUserId, startDate, endDate])
  @@map("leave_delegations")
}

model LeaveEngineRun {
  id           Int           @id @default(autoincrement())
  type         EngineRunType
  periodKey    String        @map("period_key")
  leaveYear    DateTime      @map("leave_year") @db.Date

  employeeCount Int          @default(0) @map("employee_count")
  entryCount    Int          @default(0) @map("entry_count")
  isPreview     Boolean      @default(false) @map("is_preview")

  startedAt    DateTime      @default(now()) @map("started_at")
  completedAt  DateTime?     @map("completed_at")
  failedAt     DateTime?     @map("failed_at")
  error        String?
  runById      Int?          @map("run_by_id")

  entries      LeaveLedgerEntry[]

  @@index([type, periodKey])
  @@map("leave_engine_runs")
}
```

## Design notes

**Why `LeaveRequestDay` exists at all.** Without it, "was employee 12 on leave on 3 March" requires
scanning requests and re-deriving their working days — in 04's engine, on every day computed, for
every employee. One row per leave day turns that into an indexed lookup, and it is the same row
payroll reads. It is written once at approval and is never the source of truth for the balance.

**Why the request stores both `dayCount` and `dayCountAtApproval`.** FR-R-03 needs to show the
approver that the cost changed. Keeping both makes the discrepancy a fact rather than a
reconstruction, and it survives in the record afterwards.

**Why reservations are ledger entries rather than a `pending` column.** A pending column is another
mutable counter, and it drifts exactly as the balance would. As entries, a reservation and its
release are visible in the history: "reserved 3 days on 2 Sep, released when rejected on 4 Sep" is
the answer to a question people actually ask.

**Why approval steps are stored rather than resolved on demand.** D-06. A reorganisation between
submission and decision must not redirect a request someone is already looking at, and the record
must be able to say who the approver *was*.

**Why delegation is its own table, not a field on the user.** It is date-ranged, it can be set by
the approver or by HR, and there can be several over time. A field would answer "who covers now" and
nothing else.

## Volume

| Table | Rows/year at 200 employees | Notes |
|---|---|---|
| `leave_ledger_entries` | ~3 000 | Grants + reservations + takings + adjustments |
| `leave_requests` | ~1 200 | ~6 per employee |
| `leave_request_days` | ~4 000 | The table 04 and 07 read on the hot path |
| `leave_balance_snapshots` | ~600 | employees × types × year |
| `leave_approval_steps` | ~1 400 | |

Small by comparison with attendance. The performance risk here is not volume but query shape — a
balance computed by summing the ledger on every page load would be fine at this size and wrong by
the time anyone noticed (NFR-01).

## Seed and migration notes

Migration `leave_management`:

1. Create the tables above.
2. Add the **partial unique index** enforcing FR-R-07, which Prisma cannot express:
   ```sql
   CREATE UNIQUE INDEX leave_request_days_one_approved_per_day
   ON leave_request_days (employee_id, date)
   WHERE request_id IN (SELECT id FROM leave_requests WHERE status = 'APPROVED');
   ```
   A subquery is not allowed in an index predicate, so in practice this becomes a denormalised
   `is_approved` boolean on `LeaveRequestDay`, maintained with the request's status, with the index
   `WHERE is_approved`. Noted because the naive version does not compile and the workaround affects
   the model.
3. Add the nullable `leaveRequestId` FK to `AttendanceDay` (declared in 04's model, wired here).
4. Seed permission keys and role grants.
5. Seed 05's notification types: `leave.request_pending`, `leave.request_decided`,
   `leave.request_escalated`, `leave.delegation_started`, `leave.balance_expiring`.
6. Seed **example** leave types and policies — annual, sick, unpaid — clearly labelled as examples
   to edit, following 04's precedent with the example shift. Do not invent entitlement figures that
   look authoritative (OQ-601).
7. Settings added to 03's catalogue: `leave.defaultEscalationDays` (3),
   `leave.balanceExpiryWarningDays` (30), `leave.allowBackdating` (true),
   `leave.clashWarningThreshold` (2).
8. **Run the bulk attendance recompute** (FR-X-03) as a release step, over the period since
   attendance began, and record the run.

## Open questions

| ID | Question |
|---|---|
| OQ-602 | `LeaveYearBasis.ANNIVERSARY` is modelled but the engine for it is not planned in detail. Per-employee year boundaries mean the accrual and carry-over runs iterate employees rather than a single period, and reporting gets considerably harder. |
| OQ-603 | If leave must be measured in hours, `LeaveUnit` gains `HOUR` and every `Decimal` days field becomes ambiguous. Cheaper to decide now than to migrate a ledger. |
| OQ-610 | Comp-off would add `EARNED` and a link from an attendance overtime record to a ledger entry. The ledger already accommodates it. |
| OQ-613 | Should the ledger be append-only at the database level (a trigger or a revoked UPDATE/DELETE grant), rather than only by convention? It is the one table where a stray update is most damaging. |
