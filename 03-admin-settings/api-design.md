# 03 — Admin & Settings — API Design

Conventions, error shape, pagination, and status codes as in
[01 api-design.md](../01-roles-permissions-auth/api-design.md). Every endpoint is wrapped in
`protectedRoute`, except the device authentication path, which is the system's one deliberate
exception and is specified in full below.

## The helpers this feature provides

These are used far more than the endpoints — three later features depend on them.

```ts
// src/lib/settings/index.ts
getSetting<K extends SettingKey>(key: K): Promise<SettingValue<K>>
// Typed by the code-declared catalogue (FR-S-02). An unknown key does not compile.
// Reads from the in-process cache; falls back to the declared default when no row exists.

setSetting(key, value, ctx): Promise<void>
// Validates against the declared type and validator, writes, audits, invalidates the cache.
// Refuses a change-controlled key without `settings.change_controlled`.
```

```ts
// src/lib/calendar/working-days.ts
isWorkingDay(date: CalendarDate, employeeId?: number): Promise<WorkingDayInfo>
type WorkingDayInfo = {
  isWorking: boolean;
  hours: number;                 // 0 on a non-working day, halved on a half-day holiday
  reason: "SHIFT" | "WORK_WEEK" | "HOLIDAY" | "WEEKEND";
  holiday?: { name: string; type: HolidayType; duration: HolidayDuration };
};

workingDaysBetween(from, to, employeeId?): Promise<WorkingDayInfo[]>
// The batched form. Payroll and leave MUST use this rather than looping isWorkingDay (NFR-02).
```

```ts
// src/lib/calendar/timezone.ts
companyToday(): Promise<CalendarDate>
toCompanyDate(instant: Date): Promise<CalendarDate>   // which calendar day a UTC instant falls on
startOfCompanyDay(date: CalendarDate): Promise<Date>  // the UTC instant a company day begins
```

The timezone helpers exist so that no module ever writes `new Date().toISOString().slice(0, 10)`.
That expression is the single most likely source of off-by-one-day bugs in a system like this, and
it is correct only when the server happens to run in the company's timezone (NFR-05).

## Company profile

| Method | Path | Permission |
|---|---|---|
| GET | `/api/company` | `settings.read` |
| PATCH | `/api/company` | `settings.write` |
| POST | `/api/company/logo` | `settings.write` |
| GET | `/api/company/logo` | public |
| DELETE | `/api/company/logo` | `settings.write` |

There is no POST or DELETE for the company itself (D-02). `GET /api/company/logo` is public because
the login screen renders it before anyone is authenticated; it is the only public endpoint this
feature adds, and it exposes nothing but the logo.

Logo upload follows feature 02's document rules: content sniffing, 2 MB cap, UUID storage key. SVG
uploads are **sanitised** before storage — stripped of `<script>`, event handlers, and external
references — or rejected if sanitisation fails (FR-C-03). An SVG is a document that executes.

## Settings

| Method | Path | Permission |
|---|---|---|
| GET | `/api/settings` | `settings.read` |
| PATCH | `/api/settings` | `settings.write` |
| GET | `/api/settings/public` | public |

### `GET /api/settings`

Returns the catalogue joined with current values, grouped by module — so the UI is generated rather
than hand-built (FR-S-08):

```json
{ "groups": [
    { "module": "03", "label": "Company",
      "settings": [
        { "key": "company.timezone", "type": "STRING", "value": "Asia/Karachi",
          "default": "UTC", "label": "Timezone",
          "description": "Decides which calendar day a punch belongs to. Changing it re-buckets historical attendance.",
          "isChangeControlled": true, "isSecret": false,
          "updatedAt": "2026-09-12T10:04:00Z", "updatedBy": "admin@company.com" } ] } ] }
```

Secret settings return `"value": null` with `"isSet": true` (FR-S-06, FR-D-13).

### `PATCH /api/settings`

```json
{ "changes": { "device.staleMinutes": 20, "company.dateFormat": "DD/MM/YYYY" },
  "confirmChangeControlled": false }
```

All changes apply in one transaction, each validated and audited individually. A change-controlled
key without `settings.change_controlled` → **403** `CHANGE_CONTROLLED`. With the permission but
without `confirmChangeControlled: true` → **409** `CONFIRMATION_REQUIRED`, and the response carries
the consequence text for the UI to display:

```json
{ "error": { "code": "CONFIRMATION_REQUIRED",
             "message": "This change affects historical data.",
             "consequences": [
               "Attendance for 2 480 past punches will be re-assigned to calendar days in the new timezone.",
               "Reports run before today may not match reports run after." ] } }
```

Making the server produce the consequence text, rather than the client, means the warning cannot
drift out of step with what the change actually does.

`GET /api/settings/public` returns the handful of keys needed before authentication — date format,
timezone, first day of week, company name — and nothing else. It is an explicit allowlist in code,
not a filter over the catalogue, so a new setting cannot become public by accident.

## Work week

| Method | Path | Permission |
|---|---|---|
| GET | `/api/work-week` | `settings.read` |
| PUT | `/api/work-week` | `settings.change_controlled` |

`PUT` replaces all seven days at once:

```json
{ "days": [ { "dayOfWeek": 1, "isWorkingDay": true, "standardHours": 8 },
            { "dayOfWeek": 0, "isWorkingDay": false, "standardHours": 0 } ] }
```

Change-controlled, because it changes what "absent" means for every past and future day. The
response reports the derived weekly hours so a mismatch with `workweek.standardWeeklyHours` is
visible immediately.

## Holidays and calendars

| Method | Path | Permission | Notes |
|---|---|---|---|
| GET | `/api/holiday-calendars` | `holiday.read` | With holiday counts per year |
| POST | `/api/holiday-calendars` | `holiday.write` | |
| PATCH | `/api/holiday-calendars/:id` | `holiday.write` | Including setting the default |
| DELETE | `/api/holiday-calendars/:id` | `holiday.write` | **409** if in use or default |
| GET | `/api/holidays` | `holiday.read` | `?calendarId=&year=&from=&to=` |
| POST | `/api/holidays` | `holiday.write` | |
| PATCH | `/api/holidays/:id` | `holiday.write` | |
| DELETE | `/api/holidays/:id` | `holiday.write` | |
| POST | `/api/holidays/import` | `holiday.write` | CSV, validate-then-write |
| POST | `/api/holidays/generate-recurring` | `holiday.write` | Materialise next year (FR-W-05) |
| GET | `/api/working-days` | `holiday.read` | The helper, over HTTP |

### `POST /api/holidays` and the impact check

```json
{ "calendarId": 1, "date": "2026-12-26", "name": "Boxing Day",
  "type": "PUBLIC", "duration": "FULL_DAY", "isRecurring": true }
```

Before writing, the endpoint counts what the date already carries (FR-W-08). If anything is
affected, it returns **409** `IMPACT_CONFIRMATION_REQUIRED` unless `"confirmImpact": true`:

```json
{ "error": { "code": "IMPACT_CONFIRMATION_REQUIRED",
             "message": "This date already has activity.",
             "impact": { "approvedLeaveRequests": 3, "attendanceRecords": 0,
                         "affectedEmployees": 3 } } }
```

Note what this does **not** do: it does not retroactively refund the three leave days. Whether a
holiday declared after leave was approved gives the day back is a leave policy question, and feature
06 owns it. This endpoint's job is to make sure the admin knows, and to leave a record that they
were told. The response names feature 06's screen for the follow-up.

**409** `DUPLICATE_DATE` if the calendar already has that date (FR-W-07).

`isRecurring: true` creates this year's row and marks it with a `recurringKey`;
`POST /api/holidays/generate-recurring` with `{ "year": 2027 }` creates the following year's rows,
skipping dates that already exist so re-running is safe.

### `GET /api/working-days`

`?from=2026-10-01&to=2026-10-31&employeeId=12` — the HTTP face of `workingDaysBetween`, used by the
leave request UI to show how many days a request will actually consume before it is submitted.

```json
{ "days": [ { "date": "2026-10-01", "isWorking": true, "hours": 8, "reason": "WORK_WEEK" },
            { "date": "2026-10-04", "isWorking": false, "hours": 0, "reason": "WEEKEND" },
            { "date": "2026-10-06", "isWorking": false, "hours": 0, "reason": "HOLIDAY",
              "holiday": { "name": "National Day", "type": "PUBLIC", "duration": "FULL_DAY" } } ],
  "workingDayCount": 21, "workingHours": 168 }
```

Scoped: `employeeId` outside the actor's scope → **404**.

## Devices

| Method | Path | Permission | Notes |
|---|---|---|---|
| GET | `/api/devices` | `device.read` | With derived health |
| POST | `/api/devices` | `device.write` | Registration — returns the key once |
| GET | `/api/devices/:id` | `device.read` | |
| PATCH | `/api/devices/:id` | `device.write` | Name, location, model. Not the serial. |
| POST | `/api/devices/:id/rotate-key` | `device.write` | Returns the new key once |
| POST | `/api/devices/:id/status` | `device.write` | Activate / deactivate |
| DELETE | `/api/devices/:id` | `device.write` | Only if no attendance rows |
| GET | `/api/devices/:id/events` | `device.read` | Paginated, filterable by type |
| GET | `/api/devices/unknown-attempts` | `device.read` | Unregistered serials seen recently |
| GET | `/api/devices/:id/commands` | `device.read` | |
| POST | `/api/devices/:id/commands` | `device.write` | Queue a command (payload defined by 04) |

### `POST /api/devices`

```json
{ "serialNumber": "4823901", "name": "Main gate", "location": "Building A", "model": "SenseFace 2A" }
```

**201** — and this is the only response in the entire system that contains a secret:

```json
{ "device": { "id": 2, "serialNumber": "4823901", "name": "Main gate", "health": "NEVER_CONNECTED" },
  "commKey": "k9f2…", 
  "warning": "This key is shown once. Enter it in the device's Cloud Server settings now. If it is lost, rotate the key and enter the new one." }
```

The key is generated server-side (≥32 bytes), hashed, and the plaintext is discarded after the
response is written. It is not logged, not cached, and not recoverable (FR-D-02, FR-D-13). A client
that fails to display it has lost it — which is why the UI treats this response as a modal that must
be dismissed deliberately (see `ui-ux.md`).

**409** `SERIAL_TAKEN`, including when the existing device is deactivated — the message says so,
since the natural next step is to reactivate rather than re-register.

### `POST /api/devices/:id/rotate-key`

Same response shape. The old hash is replaced immediately: the device stops being accepted from the
moment this returns (FR-D-05). Audited, and a `KEY_ROTATED` device event is written so the gap in
data has a visible cause.

### `GET /api/devices`

```json
{ "data": [ { "id": 1, "serialNumber": "4823901", "name": "Main gate", "location": "Building A",
              "status": "ACTIVE", "health": "ONLINE",
              "lastSeenAt": "2026-09-15T09:31:04Z", "lastPushAt": "2026-09-15T09:28:11Z",
              "recordsToday": 148, "commKeyLast4": "a91c",
              "clockDriftSeconds": -3 } ] }
```

`health` is derived per request from `lastSeenAt` and the threshold settings (FR-D-08). `recordsToday`
counts attendance rows for the company's current day, not the last 24 hours — an admin asking "has
the gate reported today" means today.

### `GET /api/devices/unknown-attempts`

The unknown-serial events from the last 7 days, grouped by serial with a count, first- and last-seen
time, and source IP. Each row offers *Register this device*, pre-filling the serial — the common
case is a genuine new terminal, and this turns a security log into the setup path (D-07).

## Device authentication — the one unauthenticated surface

`/iclock/*` (rewritten to `/api/device/iclock/[[...path]]` by `next.config.ts`) is exempt from
session auth (feature 01, FR-A-13). This feature supplies what replaces it. The order matters and is
specified so that feature 04 can build ingestion on top without re-deciding any of it:

> **Revised 2026-09-18.** Step 4 is gone: the SenseFace 2A has no comm key (OQ-307). The serial
> number is now the device's only identifier **and** the tenant selector
> ([MULTI-TENANCY.md](../MULTI-TENANCY.md) D-T-05), which makes this sequence weaker than planned and
> makes OQ-318 — what protects the endpoint instead — a prerequisite for building ingestion.

1. **Extract the serial number** from `?SN=`. Absent → `400`, log an `UNKNOWN_DEVICE` event with the
   IP.
2. **Look up the device** by serial (globally unique, indexed). Not found → reject, log
   `UNKNOWN_DEVICE` with serial and IP. The response is the terminal's expected error form, not a
   JSON 404 — a confused device retries endlessly and floods the log.
3. **Resolve the tenant** from the device row, and set `app.tenant_id` for the transaction
   (MULTI-TENANCY.md, "Query enforcement"). **Every subsequent step, including all of feature 04's
   ingestion, runs inside that tenant context.** A punch that cannot be attributed to a tenant is
   never written.
4. **Check status.** `DISABLED` → reject, log `AUTH_FAILED`.
5. ~~Check the comm key.~~ **Withdrawn — there is none.** Whatever OQ-318 settles on (LAN-only
   binding, HTTPS with a per-device IP allowlist, or a fronting proxy) is enforced *before* the
   request reaches the application, not here.
6. **Accept.** Update `lastSeenAt`. Write a `CONNECTED` event **only if** the previous contact was
   longer ago than the stale threshold — routine polls write no event (`data-model.md`, mitigation 1).
7. **Hand off to feature 04** for body parsing, within the tenant context from step 3. `lastPushAt`,
   `RECORDS_RECEIVED`, and the attendance rows are 04's business.

**What this sequence can and cannot do on its own.** It establishes *which registered device and
tenant* a push claims to be, refuses unregistered serials, and disables a stolen terminal. It cannot
prove the sender *is* that device.

**That is now resolved at the architecture level, not here** — see
**[DEVICE-INGESTION-SECURITY.md](../DEVICE-INGESTION-SECURITY.md)** (03 D-08b). The sequence above is
the **LAN-only** front door. In a hosted deployment the device talks to a per-site **collector**,
and the platform's real ingestion endpoint is:

### `POST /api/ingest/batch` — the collector endpoint

Authenticated with the collector's credential (`Authorization: Bearer …`), which is what resolves
the tenant — **not** the serial number. Body is a batch of punches across one or more devices.

1. **Authenticate the collector.** Unknown, revoked, or mismatched credential → `401`, logged. The
   credential is compared against `Collector.credentialHash`; a rotated credential stops working
   immediately, exactly as 03 D-08 originally intended for devices.
2. **Resolve the tenant from the collector**, and set `app.tenant_id`.
3. For each device serial in the batch: it must belong to **this collector's tenant**. A serial from
   another tenant is rejected and alerted on — it is either a misconfigured collector or an attack,
   and both need a human.
4. **IP pinning and plausibility checks** (Layer 3) applied per punch; anything failing them is
   accepted as `QUARANTINED` rather than rejected or silently trusted.
5. Hand off to feature 04's parser within the tenant context.

Idempotency is unchanged — the batch may be re-sent after a network failure and the unique
constraint on `(deviceId, userPin, punchTime)` absorbs it (04 FR-I-04). This is precisely why that
constraint was specified before anyone knew a collector would exist.

### `POST /api/ingest/heartbeat`

Collector reports version, buffered count, and per-device last-seen. This is what lets 03 tell
*"the collector is down"* from *"the device is silent"* — two different problems with two different
fixes, which the original design could not distinguish.

Rejections return the device's expected plain-text error form, never a session redirect and never an
HTML error page. Event logging is best-effort and must never fail the request (NFR-07): a logging
failure loses a log line, while a failed request loses attendance data.

Rate limiting is per serial number rather than per IP, since several terminals may sit behind one
NAT. An unregistered serial hammering the endpoint is throttled hard and its events are collapsed
into a count rather than one row per attempt.

## Scheduled jobs

| Job | Frequency | Does |
|---|---|---|
| Device health check | every 5 min | Finds devices whose `lastSeenAt` crossed the offline threshold since the last run and raises a notification (FR-D-09). Transition-based, so it alerts once, not every five minutes. |
| Device event retention | nightly | **Revised 2026-09-18 (OQ-1002).** Device events are machine-generated operational logs, not personal data, so this sweep is the one deletion the no-deletion rule should arguably still permit — **confirm**. If it must go too, mitigation 1 in `data-model.md` (log connections only on transition) becomes load-bearing rather than merely prudent: without either, one terminal writes over a million rows a year. |
| Recurring holiday generation | yearly (and on demand) | Materialises next year's recurring holidays (FR-W-05). |
| Holiday coverage warning | weekly | Raises a notification when fewer than 60 days of future holidays exist (FR-W-12). |
| Command expiry | hourly | Marks `PENDING` commands past `expiresAt` as `EXPIRED`. |

Until feature 05 exists, these jobs log instead of notifying. They share the runner with feature 01's
session-pruning job; the runner itself is a small piece of infrastructure this feature is the first
to genuinely need, so it is built here.

## Open questions

| ID | Question |
|---|---|
| OQ-307 | The comm-key parameter name and mechanism in the SenseFace push protocol — step 4 above cannot be implemented without it. **Blocked on OQ-000.** |
| OQ-312 | Whether firmware version, device timezone, and clock drift are available in the protocol at all. **Blocked on OQ-000.** |
| OQ-314 | Should `GET /api/settings/public` exist, or should the login screen render server-side and avoid it? Server-side rendering is cleaner but the shell needs the values on the client anyway. |
| OQ-315 | Where the scheduled-job runner lives: an in-process timer in the Next.js container, a separate container, or system cron hitting an authenticated endpoint. In-process is simplest and breaks quietly if the app is ever scaled to two instances (each would run every job). |
