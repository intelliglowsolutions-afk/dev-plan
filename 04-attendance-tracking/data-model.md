# 04 — Attendance Tracking — Data Model

Conventions as in features 01–03. Calendar dates `@db.Date`; instants UTC.

> **Revised 2026-09-18.** Three changes; [MULTI-TENANCY.md](../MULTI-TENANCY.md) and
> [DEVICE-INGESTION-SECURITY.md](../DEVICE-INGESTION-SECURITY.md) govern.
> - **Multi-tenant:** every model here gains `tenantId`. `Shift.name`/`code` become unique per
>   tenant; `attendance_days`' unique key and all hot indexes lead with `tenantId`.
> - **Per-tenant timezone:** the day opener and the recompute worker run **per tenant, in that
>   tenant's timezone** (D-T-06.4). A single global clock buckets attendance into the wrong day for
>   every tenant outside the server's zone.
> - **Ingestion:** punches arrive via an authenticated collector batch, whose credential resolves
>   the tenant — the serial no longer does. New `PunchFlag.QUARANTINED` below.
> - `AttendancePunch` gains `collectorId` (nullable) and `sourceIpAddress`, for IP pinning.

## Entity overview

```
Device (03) ──*── AttendancePunch ──0..1── Employee (02)
                         │
                         │  (collected by the engine, never referenced by FK)
                         ▼
Employee ──*── AttendanceDay ──*── AttendanceCorrection
                   │
                   └── reasonTrail (compact JSON: what decided this day)

Shift ──*── ShiftPatternEntry ──*── ShiftPattern
  │                                      │
  └──────*── ShiftAssignment ────────────┘
  └──────*── ShiftOverride

DirtyDay            (the recompute queue)
UnmatchedPin        (aggregated view of unresolvable PINs)
DeviceGap           (derived from 03's device events, materialised)
```

`AttendanceDay` deliberately has **no foreign key to punches**. The punches that produced it are
found by the same window query the engine used, so a recompute cannot be fooled by a stale link, and
a punch arriving late is picked up automatically.

## Enums

```prisma
enum AttendanceStatus {
  PRESENT
  LATE
  EARLY_DEPARTURE
  HALF_DAY
  ABSENT
  ON_LEAVE
  HOLIDAY
  WEEKLY_OFF
  MISSING_PUNCH
  UNKNOWN          // device gap — never absent (D-08)
  NOT_EMPLOYED     // before hire or after termination; row exists only for range completeness
}

enum PunchFlag {
  OK
  DUPLICATE        // inside the de-dup window of another punch
  OUT_OF_WINDOW    // belongs to no shift instance
  FUTURE           // timestamp ahead of ingestion time — device clock problem
  UNPARSEABLE      // raw line could not be read
  // Added 2026-09-18 (DEVICE-INGESTION-SECURITY.md Layer 3.2): arrived from an unpinned IP,
  // an unknown collector, or otherwise failed a plausibility check. STORED but NOT classified
  // into attendance until an admin approves it. Same instinct as unmatched PINs (D-07) and
  // device gaps (D-08): keep the evidence, make a human decide the interpretation.
  QUARANTINED
}

enum OvertimeStatus {
  NONE
  PENDING_APPROVAL
  APPROVED
  REJECTED
  REVIEW_REQUIRED  // beyond the shift's daily cap
}

enum CorrectionStatus {
  PENDING
  APPROVED
  REJECTED
  WITHDRAWN
  REVERSED
}

enum CorrectionSource {
  HR_DIRECT        // applied by HR, self-approved
  EMPLOYEE_REQUEST
  BULK_GAP_RESOLUTION
  SYSTEM           // e.g. reversal of a prior correction
}

enum ShiftType {
  FIXED
  FLEXIBLE         // required hours, no fixed start
}

enum ShiftSource {
  OVERRIDE
  PATTERN
  ASSIGNMENT
  WORK_WEEK
}
```

## Punches

The existing `Attendance` model becomes this, renamed and extended (D-01):

```prisma
model AttendancePunch {
  id         Int       @id @default(autoincrement())
  deviceId   Int       @map("device_id")
  // Raw PIN as sent by the terminal; resolved to an employee when one matches (FR-I-07).
  userPin    String    @map("user_pin")
  employeeId Int?      @map("employee_id")
  // UTC instant. Converted from device-local time on ingestion (FR-I-06).
  punchTime  DateTime  @map("punch_time")
  // ZKTeco punch state (0 check-in, 1 check-out, …) and verify mode (face, card, …).
  // Recorded, but NOT trusted for pairing (FR-C-03).
  punchState Int?      @map("punch_state")
  verifyMode Int?      @map("verify_mode")
  flag       PunchFlag @default(OK)
  raw        String?
  createdAt  DateTime  @default(now()) @map("created_at")

  device     Device    @relation(fields: [deviceId], references: [id])
  employee   Employee? @relation(fields: [employeeId], references: [id])

  // Devices resend after reconnecting; this makes ingestion idempotent (FR-I-04).
  @@unique([deviceId, userPin, punchTime])
  @@index([employeeId, punchTime])
  @@index([punchTime])
  @@index([userPin])          // for the unmatched-PIN list
  @@map("attendance_punches")
}
```

Only `employeeId` is ever updated, and only when an unmatched PIN is mapped (FR-U-02). Everything
else is write-once (D-02).

## The computed day

```prisma
model AttendanceDay {
  id           Int              @id @default(autoincrement())
  employeeId   Int              @map("employee_id")
  date         DateTime         @db.Date          // the shift-anchored day (D-05)

  status       AttendanceStatus
  // Snapshot of what was expected, so the row explains itself without re-resolving.
  shiftId      Int?             @map("shift_id")
  shiftSource  ShiftSource?     @map("shift_source")
  expectedHours    Decimal?     @map("expected_hours") @db.Decimal(5, 2)
  shiftStart   DateTime?        @map("shift_start")    // UTC instant for this date
  shiftEnd     DateTime?        @map("shift_end")

  firstInAt    DateTime?        @map("first_in_at")
  lastOutAt    DateTime?        @map("last_out_at")
  punchCount   Int              @default(0) @map("punch_count")

  workedHours  Decimal          @default(0) @map("worked_hours") @db.Decimal(5, 2)
  breakHours   Decimal          @default(0) @map("break_hours") @db.Decimal(5, 2)
  lateMinutes      Int          @default(0) @map("late_minutes")
  earlyLeaveMinutes Int         @default(0) @map("early_leave_minutes")

  overtimeHours    Decimal      @default(0) @map("overtime_hours") @db.Decimal(5, 2)
  overtimeStatus   OvertimeStatus @default(NONE) @map("overtime_status")

  // Worked on a holiday or weekly off (D-09) — payroll usually rates this differently.
  workedOnNonWorkingDay Boolean @default(false) @map("worked_on_non_working_day")

  leaveRequestId Int?           @map("leave_request_id")   // feature 06, nullable FK added there
  deviceGapId    Int?           @map("device_gap_id")

  // Compact record of what decided this day (FR-C-17, NFR-07).
  reasonTrail  Json?            @map("reason_trail")

  isCorrected  Boolean          @default(false) @map("is_corrected")
  hasPendingCorrection Boolean  @default(false) @map("has_pending_correction")
  // Values before any correction overlay — so the day always shows both (FR-X-10).
  computedSnapshot Json?        @map("computed_snapshot")

  computedAt   DateTime         @default(now()) @map("computed_at")
  // Hash of the computed values; lets a recompute detect "nothing changed" cheaply (FR-R-05).
  resultHash   String           @map("result_hash")

  employee     Employee         @relation(fields: [employeeId], references: [id], onDelete: Cascade)
  shift        Shift?           @relation(fields: [shiftId], references: [id])
  corrections  AttendanceCorrection[]

  @@unique([employeeId, date])
  @@index([date, status])
  @@index([employeeId, date])
  @@index([status])
  @@map("attendance_days")
}
```

**`resultHash` is the mechanism behind FR-R-05.** The engine computes the row, hashes the
classification-relevant fields, and compares. Equal → no write at all: no `updatedAt`, no audit, no
change record. Without it, a nightly full recompute writes 6 200 identical rows a month and buries
every real change.

**`computedSnapshot` holds the pre-correction values.** Cheaper and simpler than reconstructing them
by replaying corrections in reverse, and it makes the UI's "computed / corrected" comparison a
single row read.

## Corrections

```prisma
model AttendanceCorrection {
  id           Int              @id @default(autoincrement())
  attendanceDayId Int           @map("attendance_day_id")
  employeeId   Int              @map("employee_id")      // denormalised for scoped queries
  date         DateTime         @db.Date

  source       CorrectionSource
  status       CorrectionStatus @default(PENDING)

  // Any subset may be set; unset fields keep their computed values (FR-X-02).
  setFirstInAt   DateTime?      @map("set_first_in_at")
  setLastOutAt   DateTime?      @map("set_last_out_at")
  setStatus      AttendanceStatus? @map("set_status")
  setWorkedHours Decimal?       @map("set_worked_hours") @db.Decimal(5, 2)
  setOvertimeHours Decimal?     @map("set_overtime_hours") @db.Decimal(5, 2)

  reason       String                                     // mandatory (FR-X-04)
  attachmentDocumentId Int?     @map("attachment_document_id")

  requestedById Int?            @map("requested_by_id")   // User
  requestedAt   DateTime        @default(now()) @map("requested_at")
  decidedById   Int?            @map("decided_by_id")
  decidedAt     DateTime?       @map("decided_at")
  decisionNote  String?         @map("decision_note")

  reversesId    Int?            @map("reverses_id")       // a reversal points at what it undoes
  bulkRef       String?         @map("bulk_ref")          // groups a gap-resolution batch

  day          AttendanceDay    @relation(fields: [attendanceDayId], references: [id], onDelete: Cascade)
  reverses     AttendanceCorrection? @relation("Reversal", fields: [reversesId], references: [id])
  reversedBy   AttendanceCorrection[] @relation("Reversal")

  @@index([employeeId, date])
  @@index([status, requestedAt])
  @@index([bulkRef])
  @@map("attendance_corrections")
}
```

Applying a correction is: set the fields on the day, set `isCorrected`, leave `computedSnapshot`
alone. Recompute recomputes into `computedSnapshot`, then re-applies every `APPROVED` correction in
`decidedAt` order (FR-X-03).

## Shifts

Models for [shifts.md](./shifts.md).

```prisma
model Shift {
  id            Int       @id @default(autoincrement())
  name          String    @unique
  code          String?   @unique
  type          ShiftType @default(FIXED)
  colour        String?                                    // roster display

  // Times of day in company timezone, stored as minutes from midnight (D-S-01).
  // endMinutes <= startMinutes means the shift crosses midnight (D-S-02).
  startMinutes  Int?      @map("start_minutes")
  endMinutes    Int?      @map("end_minutes")
  requiredHours Decimal?  @map("required_hours") @db.Decimal(5, 2)   // FLEXIBLE shifts

  breakMinutes  Int       @default(0) @map("break_minutes")

  lateGraceMinutes       Int @default(10) @map("late_grace_minutes")
  earlyLeaveGraceMinutes Int @default(10) @map("early_leave_grace_minutes")
  fullDayMinHours Decimal @default(6) @map("full_day_min_hours") @db.Decimal(5, 2)
  halfDayMinHours Decimal @default(3) @map("half_day_min_hours") @db.Decimal(5, 2)

  tracksOvertime      Boolean @default(false) @map("tracks_overtime")
  overtimeAfterMinutes Int    @default(30) @map("overtime_after_minutes")
  overtimeNeedsApproval Boolean @default(true) @map("overtime_needs_approval")
  overtimeDailyCapHours Decimal? @map("overtime_daily_cap_hours") @db.Decimal(5, 2)

  windowBeforeMinutes Int @default(240) @map("window_before_minutes")
  windowAfterMinutes  Int @default(240) @map("window_after_minutes")
  multiSession        Boolean @default(false) @map("multi_session")
  appliesEveryDay     Boolean @default(false) @map("applies_every_day")

  isActive      Boolean   @default(true) @map("is_active")
  createdAt     DateTime  @default(now()) @map("created_at")
  updatedAt     DateTime  @updatedAt @map("updated_at")

  assignments   ShiftAssignment[]
  overrides     ShiftOverride[]
  patternEntries ShiftPatternEntry[]
  days          AttendanceDay[]

  @@map("shifts")
}

model ShiftPattern {
  id          Int      @id @default(autoincrement())
  name        String   @unique
  cycleLength Int      @map("cycle_length")     // days
  description String?
  isActive    Boolean  @default(true) @map("is_active")
  createdAt   DateTime @default(now()) @map("created_at")
  updatedAt   DateTime @updatedAt @map("updated_at")

  entries     ShiftPatternEntry[]
  assignments ShiftAssignment[]

  @@map("shift_patterns")
}

model ShiftPatternEntry {
  id         Int          @id @default(autoincrement())
  patternId  Int          @map("pattern_id")
  dayIndex   Int          @map("day_index")     // 0-based within the cycle
  shiftId    Int?         @map("shift_id")      // null = rest day
  isRestDay  Boolean      @default(false) @map("is_rest_day")

  pattern    ShiftPattern @relation(fields: [patternId], references: [id], onDelete: Cascade)
  shift      Shift?       @relation(fields: [shiftId], references: [id])

  @@unique([patternId, dayIndex])
  @@map("shift_pattern_entries")
}

model ShiftAssignment {
  id           Int           @id @default(autoincrement())
  employeeId   Int           @map("employee_id")
  shiftId      Int?          @map("shift_id")
  patternId    Int?          @map("pattern_id")
  // The cycle anchor lives here, not on the pattern — that is what lets one pattern
  // serve several offset crews (shifts.md, "Shift patterns and rotation").
  cycleAnchorDate DateTime?  @map("cycle_anchor_date") @db.Date

  effectiveFrom DateTime     @map("effective_from") @db.Date
  effectiveTo   DateTime?    @map("effective_to") @db.Date
  note          String?

  createdById   Int?         @map("created_by_id")
  bulkRef       String?      @map("bulk_ref")
  createdAt     DateTime     @default(now()) @map("created_at")

  employee      Employee     @relation(fields: [employeeId], references: [id], onDelete: Cascade)
  shift         Shift?       @relation(fields: [shiftId], references: [id])
  pattern       ShiftPattern? @relation(fields: [patternId], references: [id])

  @@index([employeeId, effectiveFrom])
  @@map("shift_assignments")
}

model ShiftOverride {
  id         Int      @id @default(autoincrement())
  employeeId Int      @map("employee_id")
  date       DateTime @db.Date
  shiftId    Int?     @map("shift_id")          // null + isRestDay = declared day off
  isRestDay  Boolean  @default(false) @map("is_rest_day")
  reason     String?
  swapRef    String?  @map("swap_ref")          // pairs the two sides of a swap

  createdById Int?    @map("created_by_id")
  createdAt  DateTime @default(now()) @map("created_at")

  employee   Employee @relation(fields: [employeeId], references: [id], onDelete: Cascade)
  shift      Shift?   @relation(fields: [shiftId], references: [id])

  @@unique([employeeId, date])
  @@index([date])
  @@map("shift_overrides")
}
```

Exactly one of `shiftId` / `patternId` must be set on an assignment. Prisma cannot express that, so
it is a check constraint added in the migration plus validation in the service — both, because the
database is the last line and the service gives the good error message.

## The recompute queue and gaps

```prisma
model DirtyDay {
  employeeId Int      @map("employee_id")
  date       DateTime @db.Date
  reason     String                                  // "punch", "shift_change", "leave_approved"
  markedAt   DateTime @default(now()) @map("marked_at")
  attempts   Int      @default(0)
  lastError  String?  @map("last_error")
  nextTryAt  DateTime @default(now()) @map("next_try_at")

  @@id([employeeId, date])                           // idempotent marking (FR-R-01)
  @@index([nextTryAt])
  @@map("dirty_days")
}

model UnmatchedPin {
  userPin      String    @id @map("user_pin")
  punchCount   Int       @default(0) @map("punch_count")
  firstSeenAt  DateTime  @map("first_seen_at")
  lastSeenAt   DateTime  @map("last_seen_at")
  dismissedAt  DateTime? @map("dismissed_at")
  dismissedById Int?     @map("dismissed_by_id")
  note         String?

  @@map("unmatched_pins")
}

model DeviceGap {
  id          Int       @id @default(autoincrement())
  deviceId    Int?      @map("device_id")            // null = all devices
  startedAt   DateTime  @map("started_at")
  endedAt     DateTime? @map("ended_at")
  isCompanyWide Boolean @default(false) @map("is_company_wide")
  affectedDayCount Int  @default(0) @map("affected_day_count")
  resolvedAt  DateTime? @map("resolved_at")
  resolvedById Int?     @map("resolved_by_id")
  resolutionNote String? @map("resolution_note")

  @@index([startedAt])
  @@map("device_gaps")
}
```

`DirtyDay` uses a composite primary key rather than an autoincrement id, so marking the same day
dirty twice is a single upsert and the queue cannot accumulate duplicates — important, because
ingesting 200 punches for one employee-day marks it dirty 200 times.

`UnmatchedPin` is an aggregate maintained on ingestion, not a view over punches. The punches table
will hold hundreds of thousands of rows and the unmatched list must not scan it (NFR-06).

`DeviceGap` is materialised from 03's device events by the health job. Deriving it on read would
mean scanning `device_events` on every attendance query.

## Volume

| Table | Rows/year at 200 employees | Notes |
|---|---|---|
| `attendance_punches` | ~290 000 | 4 punches/day × 250 working days × 200 (NFR-06) |
| `attendance_days` | ~73 000 | every employee × every calendar day |
| `attendance_corrections` | hundreds | |
| `dirty_days` | transient — should be near zero at rest | A persistently non-empty queue is an alert (FR-R-10) |

`attendance_days` covering *every* calendar day, not only working days, is deliberate: a query for
"March" must return 31 rows per employee without the caller reasoning about which days should exist.
`WEEKLY_OFF` and `HOLIDAY` rows are cheap and make every downstream query simpler.

## Migration notes

Migration `attendance_engine`:

1. **Rename** `attendance` → `attendance_punches`, and the Prisma model `Attendance` →
   `AttendancePunch`. Data is preserved; the unique constraint and indexes carry over. The existing
   test rows survive intact (acceptance criterion 14). Prisma generates a rename only if the
   migration is written by hand — an auto-generated migration will drop and recreate. **Write this
   one manually and check the generated SQL.**
2. Add `flag` to punches, defaulting to `OK`.
3. Create the shift tables, `attendance_days`, `attendance_corrections`, `dirty_days`,
   `unmatched_pins`, `device_gaps`.
4. Add the check constraint on `shift_assignments` (exactly one of shift/pattern).
5. Backfill `unmatched_pins` from existing punches with a null `employee_id`.
6. Do **not** backfill `attendance_days`. Computing history requires shifts that do not exist yet.
   The first real computation happens after shifts are configured, via an explicit recompute over a
   chosen date range — which is also the first genuine test of the engine.
7. Seed: the permission keys from this file and `shifts.md`, their role grants, and one shift
   ("General shift, 09:00–17:00") as a starting point, marked in its description as an example to
   edit. Seeding a plausible default is kinder than an empty shift list, provided it is labelled.

Settings added to 03's catalogue: `attendance.dedupeSeconds` (60),
`attendance.recomputeIntervalSeconds` (60), `attendance.unmatchedPinAlertHours` (48),
`attendance.gapAlertHours` (24), `attendance.autoOvertimeApproval` (false).

## Open questions

| ID | Question |
|---|---|
| OQ-412 | Should `attendance_days` rows exist for non-working days at all? Keeping them costs ~40% more rows and makes every downstream query simpler. The plan keeps them; the alternative is worth naming. |
| OQ-413 | `reasonTrail` shape and size cap. It is written 73 000 times a year, so it must stay small (NFR-07) — proposed as a short array of rule codes with values, not prose. |
| OQ-414 | Whether `attendance_punches` needs partitioning by month. At 290 000 rows/year it does not; at ten times that it would. Decide before the system is three years old, not after. |
| OQ-402 | Multi-device pairing and per-device gaps (FR-G-06) — the model supports it, the v1 logic deliberately does not. |
