# 04 — Attendance Tracking — API Design

Conventions as in [01 api-design.md](../01-roles-permissions-auth/api-design.md). Every endpoint
except the device path is wrapped in `protectedRoute` and composes `employeeScopeFilter`.

> **Protocol caveat.** Everything in "Ingestion" below is specified against the general shape of the
> ZKTeco ADMS/iClock push protocol. The exact paths, parameter names, record field order, and code
> meanings **are not confirmed** — the SenseFace 2A manual is still missing (OQ-000). Each
> assumption is marked ⚠. They are isolated behind one adapter module so that correcting them is a
> contained change, not a rewrite. Build this half last (README, "The elephant").

## Ingestion

The device hits `/iclock/*`, rewritten by `next.config.ts` to
`/api/device/iclock/[[...path]]/route.ts` — the file that exists today as a placeholder.

Feature 03 owns steps 1–5 of the request (serial lookup, status, key check, `lastSeenAt`,
transition event). This feature owns what happens after the handoff.

### `GET /iclock/cdata` ⚠ — the handshake

The terminal's first call on connecting. It expects a plain-text configuration block telling it what
to do: how often to poll, what to push, the server's time, and the transfer encoding.

```
GET /iclock/cdata?SN=4823901&options=all&pushver=2.4.1
→ 200 text/plain

GET OPTION FROM: 4823901
Stamp=9999
OpStamp=9999
ErrorDelay=30
Delay=30
TransTimes=00:00;14:00
TransInterval=1
TransFlag=TransData AttLog OpLog
Realtime=1
Encrypt=0
```

⚠ Field names, which of these the SenseFace 2A honours, and whether `pushver` changes the dialect
are all unconfirmed. The response is generated from settings so it is tunable without a redeploy.

This exchange is also where the **server time** is communicated, which is how the device's clock is
kept honest. If the manual confirms the device reports its own time here, the difference is recorded
as clock drift (03 FR-D-12).

### `POST /iclock/cdata` ⚠ — the punch push

Body is newline-delimited, tab-separated records. The `table` query parameter says what they are;
`ATTLOG` is attendance.

```
POST /iclock/cdata?SN=4823901&table=ATTLOG&Stamp=9999
Content-Type: text/plain

1042	2026-09-15 09:14:22	0	1	0	0
1043	2026-09-15 09:15:01	0	15	0	0
```

⚠ Assumed field order: `pin`, `time`, `state`, `verify`, `workcode`, `reserved`. Timestamps assumed
device-local, no timezone marker.

Processing, in order:

1. Split into lines. A line that will not parse is stored raw with `flag: UNPARSEABLE` and does not
   stop the batch (FR-I-03).
2. Convert each timestamp from company-local to UTC (03's helpers; FR-I-06).
3. Resolve PIN → employee; no match increments `unmatched_pins` (FR-I-07, D-07).
4. Flag duplicates within the de-dup window, and punches in the future (FR-I-11).
5. **One batch insert**, `ON CONFLICT DO NOTHING` against the unique constraint (FR-I-04, NFR-02).
6. Mark affected (employee, date) pairs dirty — one upsert per distinct pair, not per punch
   (FR-I-09, D-10).
7. Update `lastPushAt` and write a `RECORDS_RECEIVED` event with the count (03).
8. **Then** acknowledge (FR-I-05).

Response: `OK: <count>` ⚠ — the exact acknowledgement the terminal expects is protocol-specific, and
getting it wrong makes the device either retry forever or assume success and discard records. This
is the single most important line in the ingestion path to verify against the manual.

The whole handler must be fast (NFR-01): parse, insert, mark dirty, respond. No classification, no
per-punch queries, no notifications.

### `GET /iclock/getrequest` ⚠ — the command channel

The device polls for commands. Returns the oldest `PENDING` command for that device (03's
`DeviceCommand`) as plain text, or `OK` when the queue is empty. Issuing the command marks it `SENT`.

### `POST /iclock/devicecmd` ⚠ — command acknowledgement

The device reports a command's result; the matching row moves to `ACKNOWLEDGED` or `FAILED` with the
response text.

### Everything else under `/iclock/*`

Logged as an unrecognised device request with the path and returns the benign acknowledgement — a
device that gets an error for an endpoint it considers routine can stop pushing attendance
altogether. Unrecognised paths are surfaced in 03's device event log, which is how the missing parts
of the protocol will actually be discovered if the manual stays lost.

## Attendance queries

| Method | Path | Permission | Notes |
|---|---|---|---|
| GET | `/api/attendance/daily` | `attendance.read` | One date, many employees — the main grid |
| GET | `/api/attendance/employee/:id` | `attendance.read` | One employee, a date range |
| GET | `/api/attendance/days/:id` | `attendance.read` | One day, with its full explanation |
| GET | `/api/attendance/summary` | `attendance.read` | Aggregates for a range |
| GET | `/api/attendance/punches` | `attendance.read` | Raw punches, for diagnosis |
| GET | `/api/attendance/export` | `attendance.export` | CSV |

### `GET /api/attendance/daily?date=2026-09-15`

Filters: `departmentId`, `shiftId`, `status` (repeatable), `q`. Scoped.

```json
{ "date": "2026-09-15",
  "summary": { "expected": 187, "present": 171, "late": 9, "absent": 4,
               "onLeave": 8, "missingPunch": 3, "unknown": 0, "notScheduled": 12 },
  "data": [ { "employeeId": 12, "name": "Ayesha Khan", "employeeCode": "1042",
              "department": "Finance", "shift": "General shift",
              "status": "LATE", "firstInAt": "2026-09-15T04:14:22Z", "lastOutAt": null,
              "workedHours": 0, "lateMinutes": 14, "overtimeHours": 0,
              "isCorrected": false, "hasPendingCorrection": false } ],
  "page": 1, "pageSize": 50, "total": 199 }
```

The `summary` block is computed in one grouped query, not by counting the page — it must describe
the whole filtered set, and it is the part HR actually looks at.

Read straight from `attendance_days` (NFR-05). Nothing is computed on read, ever. If today's rows do
not exist yet, the response says so via `"pendingComputation": 14` rather than silently returning
fewer rows — a grid that quietly omits people is worse than one that says it is behind.

### `GET /api/attendance/days/:id` — the explanation

The endpoint behind US-07, "why was I marked absent".

```json
{ "id": 90124, "employeeId": 12, "date": "2026-09-15", "status": "LATE",
  "shift": { "id": 1, "name": "General shift", "source": "ASSIGNMENT",
             "start": "2026-09-15T04:00:00Z", "end": "2026-09-15T12:00:00Z",
             "expectedHours": 8, "lateGraceMinutes": 10 },
  "punches": [ { "id": 55012, "time": "2026-09-15T04:14:22Z", "device": "Main gate",
                 "state": 0, "flag": "OK", "usedAs": "FIRST_IN" } ],
  "computed": { "workedHours": 7.6, "lateMinutes": 14, "overtimeHours": 0 },
  "corrections": [],
  "reasonTrail": [ { "rule": "SHIFT_RESOLVED", "value": "ASSIGNMENT:1" },
                   { "rule": "WORKING_DAY", "value": true },
                   { "rule": "PAIRED_FIRST_LAST", "value": "1 session" },
                   { "rule": "LATE_GRACE_EXCEEDED", "value": "14 > 10" },
                   { "rule": "STATUS", "value": "LATE" } ] }
```

`reasonTrail` renders as a plain-language list in the UI. It is stored compactly — rule codes and
values, never prose (NFR-07, OQ-413) — and translated for display.

### `GET /api/attendance/punches`

`?employeeId=&deviceId=&from=&to=&flag=&userPin=`. The diagnostic view: raw, unpaired, exactly as
received. Scoped. This is what gets looked at when someone disputes a day, so it shows everything
including duplicates and out-of-window punches, with their flags.

## Unmatched PINs

| Method | Path | Permission |
|---|---|---|
| GET | `/api/attendance/unmatched-pins` | `attendance.manage_unmatched` |
| POST | `/api/attendance/unmatched-pins/:pin/assign` | `attendance.manage_unmatched` |
| POST | `/api/attendance/unmatched-pins/:pin/dismiss` | `attendance.manage_unmatched` |

`assign` takes `{ "employeeId": 12 }`, sets `employeeId` on every punch with that PIN, removes the
unmatched row, and marks every affected (employee, date) dirty — which can be weeks of days at once,
so the response returns the count and the date range rather than pretending it was instant.

**409** `CODE_MISMATCH` if the chosen employee's `employeeCode` differs from the PIN: the right fix
is almost always to correct the employee's code, not to paper over it here. The message says so.

## Corrections

| Method | Path | Permission | Notes |
|---|---|---|---|
| GET | `/api/attendance/corrections` | `attendance.read` | Scoped; filter by status |
| POST | `/api/attendance/corrections` | `attendance.request_correction` | Request for own day |
| POST | `/api/attendance/days/:id/correct` | `attendance.write` | HR direct, self-approved |
| POST | `/api/attendance/corrections/:id/approve` | `attendance.approve_correction` | |
| POST | `/api/attendance/corrections/:id/reject` | `attendance.approve_correction` | Reason required |
| POST | `/api/attendance/corrections/:id/withdraw` | requester | While `PENDING` |
| POST | `/api/attendance/corrections/:id/reverse` | `attendance.write` | Creates a reversal |

### `POST /api/attendance/days/:id/correct`

```json
{ "setLastOutAt": "2026-09-15T12:05:00Z", "setStatus": "PRESENT",
  "reason": "Forgot to scan out; confirmed by supervisor" }
```

**200** with the updated day. Applies immediately as `HR_DIRECT`/`APPROVED`, marks the day dirty so
worked hours and overtime are recomputed **with** the correction applied (FR-X-03), and audits.

**409** `PERIOD_LOCKED` when payroll has finalised that period (FR-X-09):

```json
{ "error": { "code": "PERIOD_LOCKED",
             "message": "September 2026 payroll was finalised on 30 Sep. This day cannot be changed.",
             "period": "2026-09", "route": "Create a payroll adjustment instead." } }
```

Naming the route matters: the user has a real problem, and a bare refusal sends them to look for a
workaround.

**422** `NOTHING_SET` if no field is set — a correction that changes nothing is a mistake, usually a
reason typed with no value.

### `POST /api/attendance/corrections` — employee request

```json
{ "date": "2026-09-15", "setLastOutAt": "2026-09-15T12:05:00Z",
  "reason": "I forgot to scan out", "attachmentDocumentId": null }
```

Creates a `PENDING` correction, sets `hasPendingCorrection` on the day (FR-X-06), and notifies the
approver. The day itself is unchanged until approval. An employee may only request for their own
days — enforced by scope, not by trusting the body's employee id.

**409** `DUPLICATE_REQUEST` if a pending request already exists for that day.

### Bulk gap resolution

`POST /api/attendance/corrections/bulk` — `attendance.write`

```json
{ "date": "2026-09-14", "employeeIds": [12, 13, 14], "setStatus": "PRESENT",
  "reason": "Main gate terminal lost power 06:00–18:00", "deviceGapId": 7 }
```

One `bulkRef` groups the batch so it reads as one event, one audit entry names the count, and the
gap is marked resolved. Deliberately the only bulk-correction path (FR-X-12): a general bulk editor
over attendance would be used for things that should be corrections with reasons.

## Recomputation

| Method | Path | Permission |
|---|---|---|
| POST | `/api/attendance/recompute/preview` | `attendance.recompute` |
| POST | `/api/attendance/recompute` | `attendance.recompute` |
| GET | `/api/attendance/recompute/status` | `attendance.recompute` |

### `POST /api/attendance/recompute/preview`

```json
{ "from": "2026-09-01", "to": "2026-09-15", "employeeIds": [12, 13], "departmentId": null }
```

Computes without writing and returns only what would change (FR-R-07):

```json
{ "daysExamined": 30, "daysChanged": 4, "lockedPeriodsSkipped": [],
  "changes": [ { "employeeId": 12, "date": "2026-09-03",
                 "from": { "status": "ABSENT", "workedHours": 0 },
                 "to": { "status": "ON_LEAVE", "workedHours": 0 },
                 "trigger": "leave approved 2026-09-14" } ] }
```

This is what makes recompute safe to run. A recompute that silently rewrites a month is the kind of
operation people stop using; one that shows four changed days first is one they trust.

### `POST /api/attendance/recompute`

Same body plus `"confirm": true`. Marks the range dirty and lets the worker drain it, rather than
computing inline — so a large range cannot time out a request. Returns the queued count.

**409** `PERIOD_LOCKED` listing the locked periods (FR-R-09) — never a silent skip.

### `GET /api/attendance/recompute/status`

Queue depth, oldest entry age, failures with their errors, and whether the worker has run recently
(FR-R-10). A stuck queue means attendance has silently stopped updating, and this endpoint is what
the admin dashboard and any monitoring will watch.

## Device gaps

| Method | Path | Permission |
|---|---|---|
| GET | `/api/attendance/gaps` | `attendance.read` |
| POST | `/api/attendance/gaps/:id/resolve` | `attendance.write` |

A gap lists its device, range, affected employee-days, and the days currently `UNKNOWN`. Resolution
is either a bulk correction (above), or an explicit "leave as unknown" with a note — recorded, so the
gap stops nagging without pretending it was fixed.

## Shifts

Per [shifts.md](./shifts.md).

| Method | Path | Permission |
|---|---|---|
| GET / POST | `/api/shifts` | `shift.read` / `shift.write` |
| GET / PATCH / DELETE | `/api/shifts/:id` | |
| GET / POST | `/api/shift-patterns` | `shift.read` / `shift.write` |
| GET | `/api/shift-assignments` | `shift.read` |
| POST | `/api/shift-assignments` | `shift.assign` |
| POST | `/api/shift-assignments/bulk` | `shift.assign` |
| POST | `/api/shift-overrides` | `shift.assign` |
| POST | `/api/shift-overrides/swap` | `shift.assign` |
| GET | `/api/roster` | `shift.read` |

`PATCH /api/shifts/:id` returns **409** `SHIFT_IN_USE_CONFIRM` when computed days reference the
shift, with the affected count and date range, unless the body carries
`"apply": "FUTURE_ONLY" | "RECOMPUTE_HISTORY"` (FR-S-11). `FUTURE_ONLY` copies the shift, closes
current assignments, and opens new ones against the copy — which is why the endpoint, not the client,
performs it.

`DELETE` → **409** `SHIFT_IN_USE` naming assignments, overrides, and computed days (FR-S-12).

`GET /api/roster?from=&to=&departmentId=` returns resolved cells with their source, in a bounded
number of queries (FR-S-15):

```json
{ "employees": [ { "id": 12, "name": "Ayesha Khan" } ],
  "days": [ { "employeeId": 12, "date": "2026-09-15", "shiftId": 1,
              "shiftName": "General shift", "source": "ASSIGNMENT",
              "isRestDay": false, "isHoliday": false } ] }
```

## Scheduled jobs

| Job | Frequency | Does |
|---|---|---|
| Recompute worker | every minute | Drains `dirty_days` oldest-first, with backoff on failure (FR-R-04, FR-R-11) |
| Day opener | nightly, after midnight company time | Marks today dirty for every employed employee, so days exist even with no punches — this is what creates `ABSENT` and `WEEKLY_OFF` rows |
| Gap detector | every 15 min | Materialises `device_gaps` from 03's device health (FR-G-01) |
| Unmatched-PIN alert | hourly | Notifies on unmatched PINs older than the threshold (FR-U-04) |
| Gap alert | hourly | Notifies on unresolved gaps past the threshold (FR-G-05) |
| Pending-correction reminder | daily | Nudges approvers |

The **day opener** deserves emphasis: without it, an absent employee generates no punch, so nothing
marks their day dirty, so no row is ever written, and they silently do not appear in the absence
report. It is the least obvious job here and the one whose failure is hardest to notice — its last
run time belongs on the same status endpoint as the queue.

## Open questions

| ID | Question |
|---|---|
| OQ-000 | The SenseFace manual. Every ⚠ above depends on it. |
| OQ-415 | Does the terminal support a "fetch records since stamp" pull, as a recovery path when a push is missed? If so it belongs here as a manual *re-sync* action, and it is the answer to most device gaps. |
| OQ-416 | Should the day opener create rows for future dates (so a roster shows expected shifts), or only up to today? Proposed: today only; the roster resolves future days on demand. |
| OQ-417 | Whether `recompute` should be allowed over an unbounded range. Proposed: capped at one year per request, to keep the queue from being flooded by a stray click. |
