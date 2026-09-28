# 03 — Admin & Settings — Requirements

## Actors

| Actor | What they do here |
|---|---|
| Super admin | Everything, including the change-controlled actions: timezone, fiscal year, device registration and key rotation. |
| HR admin | Company profile, holiday calendar, work week, and the settings their own modules use. Reads device health but does not register devices. |
| IT / office manager | In many companies the person who physically installs the terminal. Holds `device.write` without holding HR permissions. |
| Manager, Employee | Read-only consumers, indirectly: they see holidays on calendars and dates rendered in the company format. They have no settings screens. |
| Device | Authenticates with its serial number and comm key. Not a user; holds no permissions. |

## User stories

### Company and localisation

- **US-01** — As an admin, I enter our company name, address, registration numbers, and logo, so
  documents and payslips carry our identity.
- **US-02** — As an admin, I set our timezone, date format, and currency once, and every screen and
  export uses them, so I never see an American date format on an invoice.
- **US-03** — As an admin, I set our fiscal year start, so payroll periods and annual reports align
  with our accounting year rather than the calendar.

### Work week and holidays

- **US-04** — As an admin, I define which days are normally working days and the standard daily
  hours, so the system knows what "a normal day" means before any shift is configured.
- **US-05** — As an admin, I add the year's public holidays, so nobody is marked absent on a day the
  office is shut.
- **US-06** — As an admin, I import next year's holidays in one go rather than adding twenty dates
  by hand.
- **US-07** — As an admin, I mark a day as a half-day holiday, so a shortened working day is not
  treated as a late departure.
- **US-08** — As an admin, I add a holiday that repeats annually on the same date, so I do not
  re-enter New Year's Day every year.
- **US-09** — As anyone, I see the coming holidays on a calendar, so I can plan.
- **US-10** — As an admin, I am warned when a holiday I am adding falls on a date that already has
  approved leave or recorded attendance, so I understand what my change affects.

### Devices

- **US-11** — As an admin, I register a terminal by its serial number and get a comm key to enter
  into the device, so it can start pushing punches.
- **US-12** — As an admin, I see at a glance whether every terminal is online, when each was last
  heard from, and how many records it has sent today.
- **US-13** — As an admin, I am alerted when a terminal stops reporting, so I find out on the day,
  not at month-end.
- **US-14** — As an admin, I rotate a device's comm key or deactivate it entirely, so a stolen or
  replaced terminal cannot inject data.
- **US-15** — As an admin, I see a log of a device's connections and errors, so I can tell "the
  network is down" from "the device is rejecting our key".
- **US-16** — As an admin, I give a device a friendly name and a location, so "SN 4823901" becomes
  "Main gate".

### Settings

- **US-17** — As an admin, I find every configurable setting in one place, grouped sensibly, each
  with an explanation of what changing it does.
- **US-18** — As an admin, I see who last changed a setting and when.
- **US-19** — As an admin, I am stopped and made to confirm before changing a setting that alters
  historical data or has system-wide effects.

## Functional requirements

### Company profile — FR-C

| ID | Requirement |
|---|---|
| FR-C-01 | Exactly one company record exists. It is created by the seed and cannot be deleted or duplicated (D-02). |
| FR-C-02 | Fields: legal name (required), trading name, logo, address, country, registration number, tax identifier, contact email and phone, website. |
| FR-C-03 | The logo is stored like an employee document — on the volume, metadata in Postgres — and is served through an endpoint, not a static path. Maximum 2 MB, PNG/JPEG/SVG. SVG is sanitised before storage, since SVG can carry script. |
| FR-C-04 | Fiscal year start is a month and day, defaulting to 1 January. |
| FR-C-05 | Changes to the company profile are audited. |

### Localisation — FR-L

| ID | Requirement |
|---|---|
| FR-L-01 | The company timezone is an IANA identifier (`Asia/Karachi`), never a fixed UTC offset — offsets do not carry DST rules. |
| FR-L-02 | All timestamps are stored in UTC (D-03). Conversion to company time happens when a punch is assigned to a calendar day and when a value is displayed. |
| FR-L-03 | Changing the timezone requires a distinct confirmation naming the consequence, is restricted to super admin, and is audited with its own action. It does not rewrite stored data — historical punches keep their UTC instants and are simply re-bucketed into days on read (D-04). |
| FR-L-04 | Date format, time format (12/24h), first day of week, currency code, and decimal/thousands separators are configurable and used consistently across UI, exports, and generated documents. |
| FR-L-05 | Currency is a display and payroll setting; no exchange-rate handling exists in this version. |

### Work week and holidays — FR-W

| ID | Requirement |
|---|---|
| FR-W-01 | The work week defines, for each of the seven days, whether it is a working day by default and the standard hours for it. |
| FR-W-02 | Standard daily hours and standard weekly hours are stored; weekly is derived from the daily values but may be overridden, because some contracts state a weekly figure that does not divide evenly. |
| FR-W-03 | Shifts in feature 04 override the work week per employee. The work week is the fallback for anyone without a shift. |
| FR-W-04 | A holiday has a date, a name, a calendar, a type (`PUBLIC`, `COMPANY`, `OPTIONAL`), and a duration (`FULL_DAY` or `HALF_DAY`). |
| FR-W-05 | A holiday may be marked as recurring annually on the same calendar date. Recurring entries are **materialised into dated rows for each year**, not evaluated on the fly, so a one-off change to a single year's date does not require an exception mechanism. |
| FR-W-06 | Holidays that move year to year (lunar calendars, "first Monday of May") are entered per year. The system does not attempt to compute them (see OQ-309). |
| FR-W-07 | Two holidays may not share a date within one calendar. |
| FR-W-08 | Adding, moving, or deleting a holiday shows what it affects: existing approved leave on that date, and recorded attendance. The change is permitted, but never silent (US-10). |
| FR-W-09 | `isWorkingDay(date, employeeId?)` returns whether the date is expected to be worked, and for how many hours, taking into account: the employee's shift (04) if any, otherwise the work week; minus holidays on the employee's calendar. It is the only implementation of this question in the system (D-05). |
| FR-W-10 | Multiple named holiday calendars are supported. One is the company default; an employee's calendar is resolved from their department, falling back to the default (D-06). |
| FR-W-11 | Holiday import accepts CSV and validates the whole file before writing, following the same pattern as feature 02's employee import. |
| FR-W-12 | The calendar must be populated at least one year ahead; the UI warns when fewer than 60 days of future holidays exist. |

### Devices — FR-D

| ID | Requirement |
|---|---|
| FR-D-01 | A device is registered with a serial number (unique, immutable), a friendly name, an optional location, and a model. |
| FR-D-02 | Registration generates a comm key: at least 32 bytes of entropy, displayed **once**, stored only as a hash (D-08). It cannot be retrieved afterwards — only rotated. |
| FR-D-03 | Every request on `/iclock/*` is authenticated: the serial number must match an active registered device, and the presented key must match the stored hash. Failures are rejected and logged as device events, never as silent 200s. |
| FR-D-04 | An unregistered serial number is rejected. It is recorded as an `UNKNOWN_DEVICE` event with the serial and source IP, and surfaced in the UI as "a device tried to connect" — so a genuine new terminal is easy to register and a rogue one is visible (D-07). |
| FR-D-05 | Rotating a key invalidates the old one immediately. The UI states plainly that the device will stop reporting until the new key is entered into it. |
| FR-D-06 | Deactivating a device rejects its traffic but keeps its history and its attendance rows. Devices are never deleted while attendance references them. |
| FR-D-07 | `lastSeenAt` is updated on every authenticated request. `lastPushAt` is updated only when records are actually received, since the two answer different questions ("is it reachable" vs "is it sending data"). |
| FR-D-08 | Health is derived: `ONLINE` if seen within the threshold (default 15 minutes), `STALE` up to 24 hours, `OFFLINE` beyond, `NEVER_CONNECTED` if never seen, `DISABLED` if deactivated. The threshold is a setting. |
| FR-D-09 | A device transitioning to `OFFLINE` raises a notification (feature 05). The transition is detected by a scheduled check, since a device that stops talking generates no event by definition. |
| FR-D-10 | Device events are recorded for: connection, authentication failure, unknown device, records received (with a count), command issued and acknowledged, and errors returned by the device. |
| FR-D-11 | Device event rows are capped by retention (default 90 days) — they accumulate faster than any other table in the system. |
| FR-D-12 | The device's own clock drift is recorded when the protocol exposes it. A terminal whose clock is wrong produces punches at the wrong time, and this is otherwise near-impossible to diagnose after the fact. |
| FR-D-13 | The comm key never appears in logs, API responses, exports, or error messages (NFR-03 of feature 01 applies). |
| FR-D-14 | A "test connection" action shows the device's recent events and current health. It cannot actively probe the device: in push mode the device initiates, so the honest answer is "here is what we have heard from it". |

### Settings registry — FR-S

| ID | Requirement |
|---|---|
| FR-S-01 | Each setting is declared in code with: key, module, type (`string`, `int`, `decimal`, `boolean`, `enum`, `json`), default, validator, label, description, and whether it is change-controlled. |
| FR-S-02 | Settings are read through a typed accessor. A key not in the catalogue is a compile-time error, not a runtime undefined. |
| FR-S-03 | A setting with no stored row returns its declared default. The seed does not need to write every default row. |
| FR-S-04 | Writes are validated against the declared type and validator before storage. |
| FR-S-05 | Change-controlled settings (timezone, fiscal year start, currency, working-day defaults) require a confirmation step that states the consequence, and are audited with a distinct action. |
| FR-S-06 | Every setting change is audited with before and after values. Settings marked secret (SMTP password, API credentials) record that they changed, never the value. |
| FR-S-07 | Settings are cached in the application process and invalidated on write, since they are read on nearly every request. In a multi-instance deployment this cache would need to be shared or short-TTL — noted with OQ-109. |
| FR-S-08 | The settings UI is generated from the catalogue, so a module adding a setting gets a UI for free and cannot forget to expose it. |

## Settings catalogue (initial)

| Key | Type | Default | Module | Change-controlled |
|---|---|---|---|---|
| `company.timezone` | string (IANA) | `UTC` | 03 | ✅ |
| `company.dateFormat` | enum | `DD/MM/YYYY` | 03 | |
| `company.timeFormat` | enum | `24h` | 03 | |
| `company.firstDayOfWeek` | enum | `MONDAY` | 03 | |
| `company.currency` | string (ISO 4217) | `USD` | 03 | ✅ |
| `company.fiscalYearStart` | string (MM-DD) | `01-01` | 03 | ✅ |
| `workweek.standardDailyHours` | decimal | `8.0` | 03 | ✅ |
| `workweek.standardWeeklyHours` | decimal | `40.0` | 03 | ✅ |
| `holiday.defaultCalendarId` | int | seeded | 03 | |
| `device.staleMinutes` | int | `15` | 03 | |
| `device.offlineHours` | int | `24` | 03 | |
| `device.eventRetentionDays` | int | `90` | 03 | |
| `device.rejectUnknownSerials` | boolean | `true` | 03 | ✅ |
| `attendance.*`, `leave.*`, `payroll.*`, `notification.*` | — | — | 04–07 | declared by those features |

## Non-functional requirements

| ID | Requirement |
|---|---|
| NFR-01 | Settings reads must not hit the database on every request — cached in process, invalidated on write (FR-S-07). |
| NFR-02 | `isWorkingDay` is called in loops (a payroll run asks it for every employee for every day of a month). It must answer from cached holiday and work-week data, and support a batched "working days between X and Y" form. Called naively it is 6 000 queries per payroll run. |
| NFR-03 | Device authentication adds one indexed lookup by serial number plus one hash comparison. It must not become a heavyweight path — a terminal polls frequently. |
| NFR-04 | The device endpoint must respond quickly and in the exact format the terminal expects. A slow or malformed response can make the device retry, duplicate, or drop records. |
| NFR-05 | Timezone conversion uses the platform's IANA database. No hand-rolled offset arithmetic anywhere in the codebase. |
| NFR-06 | The holiday calendar must remain correct across DST transitions in timezones that observe them: a holiday is a *date*, not a 24-hour span from midnight UTC. |
| NFR-07 | Device event writes must never block or fail a device request. Logging is best-effort; ingestion is not. |

## Permission keys added by this feature

| Key | Meaning | Default holders |
|---|---|---|
| `settings.read` | View settings and company profile | HR admin (ALL) |
| `settings.write` | Change ordinary settings and company profile | HR admin (ALL) |
| `settings.change_controlled` | Change timezone, currency, fiscal year, working-day defaults | Super admin only |
| `holiday.read` | View holidays and calendars | Everyone (ALL) — employees need to see holidays |
| `holiday.write` | Add, edit, delete holidays; manage calendars | HR admin (ALL) |
| `device.read` | View devices and their health | HR admin (ALL) |
| `device.write` | Register, edit, deactivate devices; rotate keys | Super admin; grantable to IT |

## Acceptance criteria (feature-level)

1. A fresh install seeds exactly one company record, one default holiday calendar, and a Mon–Fri
   work week, and the settings screen is usable before anything else is configured.
2. Setting the timezone to `Asia/Karachi` makes a punch stored as `2026-09-15T19:30:00Z` appear on
   16 September, and attendance for the 16th includes it.
3. Changing the timezone shows a confirmation naming the effect on historical data, is refused to a
   non-super-admin, and produces a distinct audit entry.
4. `isWorkingDay` returns false for a Sunday, false for a public holiday on a Wednesday, and
   `{ working: true, hours: 4 }` for a half-day holiday — for an employee with no shift.
5. A registered device presenting the correct key is accepted; the same device with a rotated key is
   rejected within one request; an unregistered serial is rejected and appears in the UI as an
   unknown-device attempt with its IP.
6. A comm key is displayed exactly once and cannot be retrieved afterwards through any endpoint,
   export, or log.
7. A device not seen for 20 minutes shows `STALE`; at 25 hours it shows `OFFLINE` and has raised a
   notification.
8. Adding a holiday on a date with three approved leave requests warns, naming the count, and
   proceeds only on confirmation.
9. Importing 20 holidays from CSV with one duplicate date writes nothing and names the offending row.
10. A setting changed by HR appears in the audit log with before and after values; the SMTP password
    appears as changed, without either value.
11. Deleting a device that has attendance rows is refused; deactivating it is offered instead.
12. The existing `devices` rows created during testing survive the migration, with a null key hash
    that forces registration to be completed before they are accepted.

## Open questions

Carried from the README: OQ-301 (multi-company), OQ-302 (timezone), OQ-303 (multiple calendars),
OQ-304 (varying work weeks), OQ-305 (half-days), OQ-306 (device count/locations), OQ-307 (per-device
comm key support — **blocked on the manual**), OQ-308 (offline-day attendance). Added here:

| ID | Question |
|---|---|
| OQ-309 | Are any of the company's holidays on a lunar or computed calendar (Eid, Easter, Chinese New Year)? FR-W-06 says those are entered per year by hand; if there are many, a computed-date mechanism or an imported feed becomes worth building. |
| OQ-310 | Should the system expose a read-only holiday feed (iCal) for employees to subscribe to in their own calendars? Small build, disproportionately liked. |
| OQ-311 | Who, organisationally, holds `device.write` — HR, or an IT person who should not see employee data? This determines whether device screens must be reachable without any HR permission. |
