# 04 — Attendance Tracking — Requirements

Shift definition, assignment, and resolution live in [shifts.md](./shifts.md). This document covers
ingestion, classification, corrections, and recomputation.

## Actors

| Actor | What they do here |
|---|---|
| HR admin | Watches the daily attendance, fixes what the machine got wrong, enters attendance for people the device missed, runs recomputes. |
| Manager | Sees their team's attendance, approves their correction requests. |
| Employee | Sees their own attendance and asks for a correction when it is wrong. Screens in feature 08; the flow is defined here. |
| Device | Pushes punches. Authenticated by feature 03. |
| The engine | The scheduled computation itself — it writes more rows than every human user combined, and its behaviour is specified as carefully as any user's. |

## User stories

### Ingestion

- **US-01** — As the system, I accept punches pushed by a registered terminal and store them exactly
  as sent, so no record is ever lost or altered.
- **US-02** — As the system, I ignore duplicate punches when a device resends after reconnecting, so
  a reconnection does not double someone's hours.
- **US-03** — As HR, I see punches whose PIN matches no employee, so a mistyped code is found in days
  rather than at month-end.
- **US-04** — As HR, I assign an unmatched PIN to the right employee, and their attendance for those
  days is recomputed with the punches included.

### Daily attendance

- **US-05** — As HR, I see today's attendance for everyone in one grid: who is in, who is late, who
  is missing, who is on leave.
- **US-06** — As HR, I open one employee's month and see each day's status, in and out times, hours,
  and overtime.
- **US-07** — As HR, I see *why* a day was classified as it was — which shift applied, which punches
  were used, which rule fired — so I can explain it to the employee asking.
- **US-08** — As a manager, I see the same for my team only.
- **US-09** — As HR, I export a month's attendance for payroll or for an external report.

### When it goes wrong

- **US-10** — As HR, I correct a day's in or out time, with a reason, and the original punches remain
  visible.
- **US-11** — As HR, I mark someone present for a day the device missed entirely.
- **US-12** — As an employee, I request a correction for a day I was marked absent or missing a
  punch, and see its approval status.
- **US-13** — As a manager, I approve or reject my team's correction requests, with a reason.
- **US-14** — As HR, I see at a glance which days are affected by a device outage, so I do not chase
  employees for something that was not their fault.
- **US-15** — As HR, I recompute a date range after fixing a shift, a holiday, or a leave record, and
  see exactly what changed.

## Functional requirements

### Ingestion — FR-I

| ID | Requirement |
|---|---|
| FR-I-01 | Ingestion runs only after feature 03's device authentication has succeeded. An unauthenticated request never reaches the parser. **Revised 2026-09-18:** in a hosted deployment that means an authenticated **collector batch** (03 D-08b, `POST /api/ingest/batch`), where the collector's credential — not the serial number — resolves the tenant. Direct device pushes are accepted only in LAN-only deployments. Same parser either way; see [DEVICE-INGESTION-SECURITY.md](../DEVICE-INGESTION-SECURITY.md). |
| FR-I-01b | A punch that fails source-IP pinning or a plausibility check is stored with flag `QUARANTINED`: it is **not** classified into attendance, **not** counted in any report, and is surfaced for approval. Approving it marks the affected days dirty and lets the engine reclassify normally. |
| FR-I-01c | A serial in a collector batch that does not belong to that collector's tenant is rejected and alerted on — it is either a misconfigured collector or an attack, and both need a person. |
| FR-I-02 | The request body is parsed into punch records: device, user PIN, punch timestamp, punch state, verify mode, plus the raw line. The raw line is always stored (D-02). |
| FR-I-03 | A malformed line is stored as a raw record flagged `UNPARSEABLE` and does not fail the request. One bad line must never cost the other 199 in the same push. |
| FR-I-04 | Ingestion is idempotent on `(deviceId, userPin, punchTime)` — the unique constraint already in the schema. A resent batch inserts nothing new and still acknowledges success. |
| FR-I-05 | The device receives its expected acknowledgement **after** the punches are committed, never before. An acknowledgement the device believes and a transaction that rolled back loses records permanently. |
| FR-I-06 | Punch timestamps arrive in the device's local time and are converted to UTC using the company timezone (03). If the device reports its own timezone and it differs, the discrepancy is recorded as clock drift (03 FR-D-12) and the company timezone is used. |
| FR-I-07 | A PIN is resolved to an employee by `employeeCode` (02 FR-E-02). No match → the punch is stored with a null employee and queued as unmatched (D-07). |
| FR-I-08 | Resolution uses the employee who holds that code **at the time of the punch**, not at the time of processing. Codes are never reused (02 D-08), so this is a safeguard rather than a live concern. |
| FR-I-09 | Ingestion marks affected days dirty (FR-R-01) and returns. It never computes inline (D-10). |
| FR-I-10 | A punch outside every shift window is stored, attached to its calendar day, and flagged `OUT_OF_WINDOW` (`shifts.md`). |
| FR-I-11 | Punches arriving for a future date are stored but flagged; they indicate a device clock problem, and classifying tomorrow from them would be wrong. |
| FR-I-12 | Ingestion never writes to `attendance_days` directly. Only the engine does. |

### Unmatched punches — FR-U

| ID | Requirement |
|---|---|
| FR-U-01 | Unmatched punches are listed with their PIN, device, first and last seen times, and a count. |
| FR-U-02 | HR can map a PIN to an employee. The mapping applies to all past unmatched punches with that PIN and to future ones, and marks the affected days dirty. |
| FR-U-03 | A PIN can be dismissed as "not a person" (a test punch, a visitor), which hides it from the list without deleting the punches. |
| FR-U-04 | Unmatched punches older than a threshold raise a notification, since an unmatched PIN is usually a real employee with no attendance record. |

### Classification — FR-C

The core. For each (employee, date), the engine produces one `attendance_day` row.

| ID | Requirement |
|---|---|
| FR-C-01 | The engine resolves the shift via `resolveShift` (`shifts.md` FR-S-10) and the working-day status via 03's `isWorkingDay`. It re-implements neither. |
| FR-C-02 | Punches within the shift window are collected and ordered. Punches from any device count equally unless OQ-402 says otherwise. |
| FR-C-03 | Default pairing is **first punch in, last punch out** (D-06). Punch-state codes from the device are recorded but not trusted for pairing, because they are wrong whenever the employee presses the wrong button or the device defaults. |
| FR-C-04 | A shift may opt into multi-session pairing, which pairs punches in sequence and sums the sessions. With an odd number of punches the last unpaired one is reported rather than silently dropped. |
| FR-C-05 | Two punches within a **de-duplication window** (default 60 seconds) count as one. People scan twice when the first beep is missed. |
| FR-C-06 | Worked hours = paired session time − the shift's unpaid break deduction (`shifts.md` D-S-06), floored at zero. |
| FR-C-07 | Late = first in > shift start + late grace. Early departure = last out < shift end − early grace. Neither is computed for flexible shifts or for the work-week fallback, which have no fixed start (D-S-05). |
| FR-C-08 | Classification, in priority order: `UNKNOWN` (device gap, FR-C-13) → `HOLIDAY` → `WEEKLY_OFF` → `ON_LEAVE` → `MISSING_PUNCH` → `ABSENT` → `HALF_DAY` → `LATE` / `EARLY_DEPARTURE` → `PRESENT`. The order is the specification: a day is classified by the first rule that applies. |
| FR-C-09 | `MISSING_PUNCH`: there is at least one punch but pairing cannot produce a complete session — typically an in with no out. Worked hours are **not** assumed (OQ-406); the day carries zero hours and requires a correction. |
| FR-C-10 | `ABSENT`: a working day, no punches, no approved leave, and no device gap. |
| FR-C-11 | `HALF_DAY`: worked hours below the shift's full-day minimum but at or above the half-day minimum. Below the half-day minimum the day is `ABSENT` despite punches, and the punches remain visible on it. |
| FR-C-12 | `ON_LEAVE` is set from approved leave for that date (feature 06). A half-day leave plus a half day worked produces `HALF_DAY` with a leave reference, not two conflicting rows. |
| FR-C-13 | `UNKNOWN`: every device that would normally cover this employee was offline for the whole shift window (03's device health). The day is **never** `ABSENT` in this case (D-08), is visibly flagged, and is excluded from absence counts in reports. |
| FR-C-14 | A day with punches on a `HOLIDAY` or `WEEKLY_OFF` keeps that classification and additionally records `workedOnNonWorkingDay` with the hours, for payroll (D-09). |
| FR-C-15 | Overtime = worked hours beyond the shift's expected hours, minus the overtime threshold, and only if the shift tracks overtime. It is recorded with status `PENDING_APPROVAL` where the shift requires approval (OQ-405). |
| FR-C-16 | Overtime beyond the shift's daily cap is recorded but flagged `REVIEW_REQUIRED` rather than accrued silently. |
| FR-C-17 | The engine records the **reason trail**: which shift applied and from where, which punches were used, which thresholds fired, and which rule produced the final status. This is what US-07 renders. |
| FR-C-18 | The engine writes no row for a date before the employee's hire date or after their last working day (02). Employment periods are respected, including gaps from a rehire. |
| FR-C-19 | Days are computed for employees in employment statuses that expect attendance. `TERMINATED` produces nothing; `ON_LEAVE` and `SUSPENDED` produce days classified accordingly rather than absent. |
| FR-C-20 | Computation is pure and deterministic (D-03): no `now()`, no randomness, no dependence on row order. Given the same punches, shift, holidays, and leave, it produces the identical row. |

### Corrections — FR-X

| ID | Requirement |
|---|---|
| FR-X-01 | A correction is a separate record referencing the day, never an edit of the computed row (D-04). |
| FR-X-02 | A correction may set: in time, out time, status, worked hours, overtime hours, or a note — any subset. Fields it does not set keep their computed values. |
| FR-X-03 | Recomputation re-applies corrections after computing, so a correction survives any number of recomputes. |
| FR-X-04 | Every correction records who requested it, who approved it, when, and why. A reason is mandatory. |
| FR-X-05 | HR with `attendance.write` may apply a correction directly; it is recorded as self-approved and audited (OQ-407). |
| FR-X-06 | An employee may **request** a correction for their own day. It is `PENDING` until approved, and the computed day is unchanged in the meantime — but visibly marked as having a pending request, so the same day is not chased twice. |
| FR-X-07 | Approval routes to the employee's manager, with HR able to approve or override (OQ-407). Approval applies the correction and marks the day for recompute. |
| FR-X-08 | Rejection requires a reason, which is shown to the requester. |
| FR-X-09 | A correction cannot be applied to a period locked by a finalised payroll run (OQ-408). The attempt returns a specific error naming the period and the route available (an adjustment in feature 07), rather than a generic refusal. |
| FR-X-10 | Corrections are visible on the day forever: the original computed values and the corrected values are both shown, with who changed them and why. |
| FR-X-11 | A correction may be withdrawn while pending, and reversed after approval. A reversal is a new record, not a deletion. |
| FR-X-12 | Bulk correction is supported for one specific case — marking a set of employees present for a device-outage day — and requires a reason. It is deliberately not a general bulk-edit tool. |

### Recomputation — FR-R

| ID | Requirement |
|---|---|
| FR-R-01 | A **dirty-day queue** records (employee, date) pairs needing computation. Writing to it is cheap and idempotent. |
| FR-R-02 | Days are marked dirty by: new punches, an unmatched PIN being mapped, a shift edit or assignment change, an override, a holiday change, a work-week change, a timezone change, an approved leave that covers the date, a correction, and an employment change (hire, terminate, rehire). |
| FR-R-03 | Each of those triggers lives with the feature that owns the change — 03 marks days dirty when a holiday moves; 06 when leave is approved — through one shared `markDirty(employeeIds, dateRange, reason)` helper. |
| FR-R-04 | A worker drains the queue on a schedule (default every minute) and on demand. It processes oldest first and is safe to run concurrently with ingestion. |
| FR-R-05 | Recomputing a day that has not changed produces an identical row and does **not** bump `updatedAt` or write an audit entry. Otherwise a nightly full recompute would produce a million meaningless audit rows. |
| FR-R-06 | When a recompute changes a day's status or hours, the change is recorded with the previous values and the trigger reason. |
| FR-R-07 | HR can trigger a recompute for a date range and a set of employees, see a **preview of what would change** before committing, and then apply it. |
| FR-R-08 | Recomputation never touches punches (D-02) and never discards corrections (FR-X-03). |
| FR-R-09 | A recompute of a locked payroll period is refused (FR-X-09), and the refusal names the locked periods rather than silently skipping them. |
| FR-R-10 | The queue is observable: depth, oldest entry, failures. A stuck queue means attendance silently stops updating, which is invisible otherwise. |
| FR-R-11 | A day that fails to compute is recorded with the error and retried with backoff; after repeated failure it is surfaced rather than retried forever. |
| FR-R-12 | The engine is the only writer of `attendance_days`. Every other path goes through corrections or the queue (FR-I-12). |

### Device gaps — FR-G

| ID | Requirement |
|---|---|
| FR-G-01 | A device gap is a period during which a device that normally reports produced nothing, derived from 03's device health and event history. **Revised 2026-09-18:** a healthy collector with a non-empty buffer means punches are *delayed*, not *missing* — the gap detector must not open a gap in that case, and 03's heartbeat is what tells it the difference. A collector buffering through a WAN outage should produce **no** `UNKNOWN` days once it drains, which is one of the main reasons the collector is worth building. |
| FR-G-02 | A day whose entire shift window falls inside a gap covering every relevant device is classified `UNKNOWN` (FR-C-13). |
| FR-G-03 | Gaps are listed for HR with their date range, device, and the number of employee-days affected. |
| FR-G-04 | HR can resolve a gap by bulk-marking the affected employees present (FR-X-12), by leaving the days unknown, or by requesting corrections from the employees. |
| FR-G-05 | Unresolved gaps older than a threshold raise a notification: an unresolved gap is unpaid or wrongly-paid time. |
| FR-G-06 | With more than one device, a gap on one device is only a gap for employees who punch at it — which the system cannot know for certain. v1 treats a gap as company-wide if **all** devices were down, and otherwise flags the days for review rather than classifying them `UNKNOWN` (OQ-402). |

## Non-functional requirements

| ID | Requirement |
|---|---|
| NFR-01 | Ingestion must acknowledge a device push within the terminal's timeout. Parsing and insertion only; everything else is deferred to the queue (D-10, 03 NFR-04). |
| NFR-02 | A push of 1 000 punch records inserts in one batch, not 1 000 statements. |
| NFR-03 | Computing one employee-day must not issue a query per lookup. Shift, holidays, leave, and punches are loaded per batch of days, then computed in memory (`shifts.md` FR-S-15, 03 NFR-02). |
| NFR-04 | A full recompute of 200 employees × 31 days (6 200 days) completes in under two minutes. |
| NFR-05 | The daily attendance grid for 200 employees loads in under a second — it is read from `attendance_days`, never computed on read. |
| NFR-06 | `attendance_punches` grows at roughly employees × punches/day. At 200 employees and 4 punches a day that is ~290 000 rows a year: indexed, not partitioned, but it must not be scanned for routine queries. |
| NFR-07 | The reason trail (FR-C-17) must be compact. It is written for every day of every employee; a verbose JSON blob per day is tens of megabytes a year for no benefit. |
| NFR-08 | All date arithmetic uses 03's timezone helpers. No module-local date parsing (03 NFR-05). |
| NFR-09 | The engine must be unit-testable without a device, a database, or a clock: pure functions taking punches, shift, holidays, and leave, returning a classification. This is the single most important testability requirement in the system. |

## Permission keys added by this feature

| Key | Meaning | Default holders |
|---|---|---|
| `attendance.read` | View attendance days and punches | HR: ALL · Manager: DEPARTMENT · Employee: SELF |
| `attendance.write` | Apply corrections directly, enter manual attendance | HR: ALL |
| `attendance.request_correction` | Request a correction for your own day | Everyone: SELF |
| `attendance.approve_correction` | Approve or reject correction requests | HR: ALL · Manager: DEPARTMENT |
| `attendance.recompute` | Trigger recomputation | HR: ALL |
| `attendance.export` | Export attendance data | HR: ALL · Manager: DEPARTMENT |
| `attendance.manage_unmatched` | Map or dismiss unmatched PINs | HR: ALL |

Shift permissions are in [shifts.md](./shifts.md).

## Acceptance criteria (feature-level)

1. A push of 50 punches stores 50 rows; re-pushing the same batch stores none and still returns the
   device's expected acknowledgement.
2. A punch with an unknown PIN is stored with a null employee and appears in the unmatched list
   within one refresh.
3. Mapping that PIN to an employee recomputes the affected days, and the punches appear on them.
4. An employee scanning in at 08:58 and out at 17:03 on a 09:00–17:00 shift is `PRESENT` with 8:05
   worked, less the break deduction.
5. The same employee in at 09:08 with a 10-minute grace is `PRESENT`, not `LATE`; at 09:11 they are
   `LATE` by 11 minutes.
6. Scanning in and never out yields `MISSING_PUNCH` with zero hours — never an assumed departure.
7. Two scans 20 seconds apart count as one punch.
8. A working day with no punches and no leave is `ABSENT`; the same day with approved leave is
   `ON_LEAVE`; the same day during a total device outage is `UNKNOWN` and is excluded from absence
   counts.
9. Recomputing an unchanged month changes no `updatedAt` and writes no audit entries.
10. Correcting a day, then recomputing it, preserves the correction and still shows the original
    computed values.
11. A correction to a month locked by a finalised payroll run is refused, naming the period.
12. HR's recompute preview lists the days that would change, with old and new values, before anything
    is written.
13. The classification engine's tests run without a database, feeding punch arrays and shift
    definitions directly into pure functions.
14. The existing `attendance` rows from device testing survive the rename to `attendance_punches`
    with their data intact.
