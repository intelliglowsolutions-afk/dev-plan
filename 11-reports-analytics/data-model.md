# 11 — Reports & Analytics — Data Model

Like feature 08, this document is short — and for the same reason. If it grew, D-01 would have been
broken and this feature would be computing things it should be reading.

> **Revised 2026-09-18 — multi-tenant (OQ-301).** [MULTI-TENANCY.md](../MULTI-TENANCY.md) governs.
> - `SavedReportView`, `ReportSchedule` and `ReportRun` gain `tenantId`. Report *definitions* stay
>   global and code-declared (D-T-02).
> - **Tenant scoping is step zero** of the run pipeline — before permission scoping, before
>   `employeeScopeFilter`, before suppression (D-T-08). There is no cross-tenant report.
> - **Suppression is evaluated within a tenant**, which makes the fixed threshold of 5 questionable
>   now that tenants vary in size: a 12-person tenant would see nearly everything suppressed. Make
>   it per-tenant configurable (OQ-T-06, and it supersedes OQ-1103's single answer).
> - Scheduled reports run **per tenant**, and a recipient check crossing tenants is a bug, not an
>   edge case.
> - **The nightly run-retention sweep is removed** (OQ-1002); `report.runRetentionDays` becomes a
>   reporting threshold rather than a deletion trigger.
> - Reports gain a **location** dimension (OQ-206), and historical location comes from
>   `EmployeeAssignment` like every other structural fact (D-06).

## What this feature adds

Three models: saved views, schedules, and a run log. **No reporting tables, no aggregate tables, no
warehouse** (D-07).

## What this feature reads

Through the owning features' queries and helpers, never by writing its own SQL against their tables
(D-01, FR-C-02):

| Data | Owner | Read via |
|---|---|---|
| Employees, departments, positions, assignment history | 02 | employee queries + `EmployeeAssignment` for historical structure |
| Devices, holidays, work week | 03 | settings and `isWorkingDay` |
| `attendance_days`, corrections, device gaps | 04 | attendance queries — never `attendance_punches` (NFR-02) |
| Leave ledger, requests, balances | 06 | `getBalance`, leave queries |
| Finalised payslips and their snapshots | 07 | payslip queries |
| Cycle completion (not content) | 09 | `/cycles/:id/progress` equivalent |
| Applications, stage events, offers | 10 | recruitment queries |

The contract: a report definition names a **function**, not a table. If the function does not exist,
the report is not built until the owning feature provides one.

## Models

```prisma
enum ScheduleFrequency {
  DAILY
  WEEKLY
  MONTHLY
  QUARTERLY
}

enum ReportRunStatus {
  RUNNING
  COMPLETE
  FAILED
  SUPPRESSED      // completed, but every value was withheld (FR-R-06)
}

model SavedReportView {
  id          Int      @id @default(autoincrement())
  // Key from the code-declared catalogue (D-04). Not an FK — the catalogue is not a table.
  reportKey   String   @map("report_key")
  ownerUserId Int      @map("owner_user_id")
  name        String
  // Parameters only, never results (FR-E-07).
  parameters  Json
  isShared    Boolean  @default(false) @map("is_shared")

  createdAt   DateTime @default(now()) @map("created_at")
  updatedAt   DateTime @updatedAt @map("updated_at")

  owner       User     @relation(fields: [ownerUserId], references: [id], onDelete: Cascade)

  @@index([ownerUserId, reportKey])
  @@map("saved_report_views")
}

model ReportSchedule {
  id            Int               @id @default(autoincrement())
  reportKey     String            @map("report_key")
  ownerUserId   Int               @map("owner_user_id")
  name          String
  parameters    Json

  frequency     ScheduleFrequency
  dayOfMonth    Int?              @map("day_of_month")
  dayOfWeek     Int?              @map("day_of_week")
  hour          Int               @default(8)

  // User ids. Checked against permissions at delivery, not at creation (FR-E-05).
  recipientUserIds Int[]          @map("recipient_user_ids")

  isActive      Boolean           @default(true) @map("is_active")
  // Set when the owner is suspended (FR-E-06) or a recipient loses access.
  disabledAt    DateTime?         @map("disabled_at")
  disabledReason String?          @map("disabled_reason")

  lastRunAt     DateTime?         @map("last_run_at")
  nextRunAt     DateTime?         @map("next_run_at")

  createdAt     DateTime          @default(now()) @map("created_at")

  owner         User              @relation(fields: [ownerUserId], references: [id], onDelete: Cascade)

  @@index([isActive, nextRunAt])
  @@map("report_schedules")
}

model ReportRun {
  id          Int             @id @default(autoincrement())
  reportKey   String          @map("report_key")
  runByUserId Int?            @map("run_by_user_id")
  scheduleId  Int?            @map("schedule_id")

  parameters  Json
  // The provenance block shown in the UI and written into every export (FR-R-03, D-05).
  asOf        DateTime        @map("as_of")
  rowCount    Int?            @map("row_count")
  suppressedCellCount Int     @default(0) @map("suppressed_cell_count")
  sources     String[]                                  // ["02", "04"]

  status      ReportRunStatus @default(RUNNING)
  durationMs  Int?            @map("duration_ms")
  error       String?

  // Export details, when this run produced a file (FR-E-03, D-10).
  exportFormat String?        @map("export_format")
  exportRowCount Int?         @map("export_row_count")

  createdAt   DateTime        @default(now()) @map("created_at")

  @@index([reportKey, createdAt])
  @@index([runByUserId, createdAt])
  @@map("report_runs")
}
```

## Design notes

**Why `ReportRun` records every run, not only exports.** Two reasons. It makes slow reports visible
(`durationMs`) so NFR-01 can be checked against reality rather than hoped for, and
`suppressedCellCount` over time shows whether the suppression threshold is set sensibly — a report
that is 90% suppressed is not protecting anything, it is just broken (OQ-1103).

**Why `recipientUserIds` is an array rather than a join table.** Recipients are a short list on a
row that is written rarely and read on a schedule. A join table would be more correct and buy
nothing here. Permission checking happens at delivery regardless (FR-E-05), so the array is never
treated as authorisation.

**Why schedules are disabled rather than deleted when their owner is suspended.** The schedule is
evidence of what was being sent to whom. Deleting it on suspension would erase that at exactly the
moment someone might need to ask (FR-E-06).

**Why saved views store parameters and never results.** A stored result is a stale copy of data
whose permissions may since have changed — the classic way a report cache becomes a leak (FR-E-07).

**Why there is no materialised aggregate table.** D-07. Every such table is a second source of truth
that will eventually disagree with the first, plus an invalidation problem. At this system's volumes
it buys nothing. The place it becomes necessary is named in OQ-1104 rather than pre-built.

## Historical structure — the query shape

FR-H-01 is the requirement most likely to be implemented wrongly, so the shape is stated here.

**Wrong** — joins to the employee's *current* department, which silently rewrites history at every
restructure:

```sql
SELECT d.name, count(*) FROM attendance_days ad
JOIN employees e ON e.id = ad.employee_id
JOIN departments d ON d.id = e.department_id      -- today's department
WHERE ad.date BETWEEN $1 AND $2 GROUP BY d.name;
```

**Right** — joins to the assignment in force on each date (02 D-04):

```sql
SELECT d.name, count(*) FROM attendance_days ad
JOIN employee_assignments ea
  ON ea.employee_id = ad.employee_id
 AND ad.date >= ea.effective_from
 AND (ea.effective_to IS NULL OR ad.date <= ea.effective_to)
JOIN departments d ON d.id = ea.department_id
WHERE ad.date BETWEEN $1 AND $2 GROUP BY d.name;
```

This is why feature 02 kept assignment history rather than only current values, and why that
decision (02 D-04) is repaid here rather than in feature 02 itself.

The same pattern applies to position, manager, and employment type. Payroll reports need none of it:
finalised payslips already carry their own snapshots (FR-H-03, 07 D-03).

## Suppression — the implementation shape

FR-S-04 (resisting differencing) is subtle enough to warrant the algorithm:

1. Compute the aggregate with its group populations.
2. Mark groups below the threshold as suppressed.
3. **If exactly one group is suppressed**, its value is derivable from the total and the other
   groups — so suppress the next-smallest group as well.
4. **If the total is shown and all groups are suppressed but one**, suppress the total too.
5. Report `suppressedCellCount` so the effect is visible on the run record.

Steps 3 and 4 are the ones usually omitted, and their omission makes the whole mechanism decorative.

## Seed and migration notes

Migration `reporting`:

1. Create the three tables.
2. Seed permission keys: `report.read`, `report.export`, `report.schedule`, `report.company_wide`.
3. Seed 05's notification types: `report.scheduled_ready`, `report.schedule_disabled`,
   `report.run_failed`. Bodies carry **a link and never data** (D-08).
4. Settings in 03's catalogue: `report.suppressionThreshold` (5, **change-controlled** per FR-S-07),
   `report.backgroundThresholdSeconds` (5), `report.exportRowLimit` (50 000),
   `report.runRetentionDays` (180).
5. No report definitions are seeded — they live in code (D-04).

## Open questions

| ID | Question |
|---|---|
| OQ-1103 | The suppression threshold's real value. `ReportRun.suppressedCellCount` is there specifically to answer this empirically after a month of use. |
| OQ-1104 | When live querying stops being adequate. The answer changes `data-model.md` more than any other file here. |
| OQ-1109 | User-arranged dashboards would need a layout model per user. Deliberately absent from v1 (FR-D-04). |
| OQ-1110 | Should `ReportRun` retain the parameters of every run indefinitely? They can contain department and date filters, which is mild, but the table grows with use. 180 days proposed. |
