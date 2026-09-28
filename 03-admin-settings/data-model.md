# 03 — Admin & Settings — Data Model

Conventions as in features 01 and 02. Calendar dates use `@db.Date`; everything else is UTC
`DateTime`.

## Entity overview

```
Company (singleton)
Setting          (key → typed value; catalogue declared in code)
WorkWeekDay      (7 rows: is this weekday worked, and for how many hours)

HolidayCalendar ──*── Holiday
      │
      └──*── Department (feature 02, via calendarId)  → employee's calendar resolves through here

Device ──*── DeviceEvent
   │
   └──*── DeviceCommand
   └──*── Attendance (feature 04, existing relation)
```

## Enums

```prisma
enum HolidayType {
  PUBLIC     // statutory
  COMPANY    // company-declared closure
  OPTIONAL   // employee may choose to take it; feature 06 decides what that costs
}

enum HolidayDuration {
  FULL_DAY
  HALF_DAY
}

enum SettingType {
  STRING
  INT
  DECIMAL
  BOOLEAN
  ENUM
  JSON
}

enum DeviceStatus {
  ACTIVE
  DISABLED
}

enum DeviceEventType {
  CONNECTED           // authenticated request received
  AUTH_FAILED         // known serial, wrong key
  UNKNOWN_DEVICE      // unregistered serial (D-07)
  RECORDS_RECEIVED    // with a count
  COMMAND_ISSUED
  COMMAND_ACKED
  DEVICE_ERROR        // the terminal reported a problem
  KEY_ROTATED
  CLOCK_DRIFT         // reported device time differs from server time
}

enum DeviceCommandStatus {
  PENDING
  SENT
  ACKNOWLEDGED
  FAILED
  EXPIRED
}
```

`DeviceStatus` is deliberately only `ACTIVE`/`DISABLED`. Health (`ONLINE`/`STALE`/`OFFLINE`) is
**derived from `lastSeenAt`, not stored** — a stored status would need a job to keep it true and
would be wrong between runs.

## Models

> **Revised 2026-09-18 (OQ-301, OQ-307).** Two changes run through everything below:
>
> 1. **Multi-tenant.** Every model here gains `tenantId`, and `Company` is no longer a singleton —
>    it is one tenant's profile. Timezone moves to the `Tenant` model. See
>    [MULTI-TENANCY.md](../MULTI-TENANCY.md), which governs where it conflicts with this file.
> 2. **No device comm key exists.** The manual confirms the SenseFace 2A has no such field, so
>    `commKeyHash`, `commKeySetAt` and `commKeyLast4` below are **withdrawn** — shown struck through
>    rather than deleted, so the reasoning survives. Replacement protection is OQ-318.

```prisma
model Company {
  // Was a singleton (id @default(1)) under the withdrawn D-02. Now one row per tenant.
  id                 Int      @id @default(autoincrement())
  tenantId           Int      @unique @map("tenant_id")   // one profile per tenant
  legalName          String   @map("legal_name")
  tradingName        String?  @map("trading_name")

  registrationNumber String?  @map("registration_number")
  taxIdentifier      String?  @map("tax_identifier")

  addressLine1       String?  @map("address_line1")
  addressLine2       String?  @map("address_line2")
  city               String?
  postalCode         String?  @map("postal_code")
  country            String?

  contactEmail       String?  @map("contact_email")
  contactPhone       String?  @map("contact_phone")
  website            String?

  // Stored like an employee document: file on the volume, metadata here (FR-C-03).
  logoStorageKey     String?  @map("logo_storage_key")
  logoMimeType       String?  @map("logo_mime_type")

  createdAt          DateTime @default(now()) @map("created_at")
  updatedAt          DateTime @updatedAt @map("updated_at")

  @@map("company")
}

model Setting {
  key          String      @id                       // "company.timezone"
  module       String                                 // "03", "04" — the owning feature
  type         SettingType
  value        String                                 // serialised; the catalogue says how to parse
  isSecret     Boolean     @default(false) @map("is_secret")

  updatedById  Int?        @map("updated_by_id")
  updatedAt    DateTime    @updatedAt @map("updated_at")

  @@index([module])
  @@map("settings")
}

model WorkWeekDay {
  // 0 = Sunday … 6 = Saturday, matching JS getDay()
  dayOfWeek     Int      @id @map("day_of_week")
  isWorkingDay  Boolean  @default(true) @map("is_working_day")
  standardHours Decimal  @default(8.0) @map("standard_hours") @db.Decimal(4, 2)
  updatedAt     DateTime @updatedAt @map("updated_at")

  @@map("work_week_days")
}

model Location {
  // Added 2026-09-18 (OQ-206: multiple locations). Lives in feature 03 rather than 02 because
  // it is tenant infrastructure — an address, a holiday calendar, and the devices standing in
  // it — in the same way HolidayCalendar lives here while Department (02) references it.
  id           Int              @id @default(autoincrement())
  tenantId     Int              @map("tenant_id")
  name         String
  code         String?

  addressLine1 String?          @map("address_line1")
  addressLine2 String?          @map("address_line2")
  city         String?
  postalCode   String?          @map("postal_code")
  country      String?

  // Public holidays are geographic, so a location may carry its own calendar (see the
  // resolution order below). Null = inherit the tenant's default.
  calendarId   Int?             @map("calendar_id")

  // Reserved, UNUSED in v1. OQ-302 settled one timezone per tenant, so day boundaries come
  // from Tenant.timezone. A tenant whose locations span timezones needs OQ-324 answered
  // first — it changes attendance day bucketing per location, not just display.
  timezone     String?

  isActive     Boolean          @default(true) @map("is_active")
  createdAt    DateTime         @default(now()) @map("created_at")
  updatedAt    DateTime         @updatedAt @map("updated_at")

  calendar     HolidayCalendar? @relation(fields: [calendarId], references: [id])
  devices      Device[]
  employees    Employee[]

  @@unique([tenantId, name])
  @@index([tenantId, isActive])
  @@map("locations")
}

model HolidayCalendar {
  id          Int          @id @default(autoincrement())
  name        String       @unique
  description String?
  isDefault   Boolean      @default(false) @map("is_default")
  isActive    Boolean      @default(true) @map("is_active")
  createdAt   DateTime     @default(now()) @map("created_at")
  updatedAt   DateTime     @updatedAt @map("updated_at")

  holidays    Holiday[]
  departments Department[]    // feature 02 gains a nullable calendarId

  @@map("holiday_calendars")
}

model Holiday {
  id          Int             @id @default(autoincrement())
  calendarId  Int             @map("calendar_id")
  date        DateTime        @db.Date
  name        String
  type        HolidayType     @default(PUBLIC)
  duration    HolidayDuration @default(FULL_DAY)
  note        String?

  // Recurring entries are materialised per year (FR-W-05). This marks the ones generated
  // from a recurring rule, so regenerating next year's can tell them from hand-entered dates.
  isRecurring   Boolean       @default(false) @map("is_recurring")
  recurringKey  String?       @map("recurring_key")   // stable id across years, e.g. "new-year"

  createdById Int?            @map("created_by_id")
  createdAt   DateTime        @default(now()) @map("created_at")
  updatedAt   DateTime        @updatedAt @map("updated_at")

  calendar    HolidayCalendar @relation(fields: [calendarId], references: [id], onDelete: Restrict)

  @@unique([calendarId, date])            // FR-W-07
  @@index([date])
  @@map("holidays")
}

model Device {
  id              Int          @id @default(autoincrement())
  serialNumber    String       @unique @map("serial_number")   // immutable after registration
  name            String?
  model           String?                                       // "SenseFace 2A"
  // Was free text; now an FK (OQ-206). A device stands in exactly one location, and that is
  // what lets 04 tell "the Lahore gate is down" from "all devices are down".
  locationId      Int?         @map("location_id")
  ipAddress       String?      @map("ip_address")               // last seen from
  // WITHDRAWN 2026-09-18 (OQ-312, confirmed against the manual): the protocol surface does not
  // expose firmware version, device timezone, or clock drift. Speculative columns removed.
  //   firmwareVersion String? @map("firmware_version")
  //   deviceTimezone  String? @map("device_timezone")

  // WITHDRAWN 2026-09-18 (OQ-307). The SenseFace 2A has no comm-key field, so there is no
  // secret to hash. Do not implement these; do not leave them nullable-and-unused either.
  //   commKeyHash     String?   @map("comm_key_hash")
  //   commKeySetAt    DateTime? @map("comm_key_set_at")
  //   commKeyLast4    String?   @map("comm_key_last4")
  //
  // The serial number is now BOTH the device identifier and the tenant selector
  // (MULTI-TENANCY.md D-T-05), so it stays GLOBALLY unique — not composite with tenantId.
  tenantId        Int          @map("tenant_id")

  status          DeviceStatus @default(ACTIVE)
  lastSeenAt      DateTime?    @map("last_seen_at")   // any accepted request
  lastPushAt      DateTime?    @map("last_push_at")   // records actually received
  //   lastClockDriftSeconds — withdrawn with the two fields above (OQ-312)

  registeredById  Int?         @map("registered_by_id")
  createdAt       DateTime     @default(now()) @map("created_at")
  updatedAt       DateTime     @updatedAt @map("updated_at")

  events          DeviceEvent[]
  commands        DeviceCommand[]
  attendance      Attendance[]                                   // existing relation

  @@index([status])
  @@map("devices")
}

model DeviceEvent {
  id           Int             @id @default(autoincrement())
  // Nullable: an UNKNOWN_DEVICE event has no device row by definition (FR-D-04).
  deviceId     Int?            @map("device_id")
  serialNumber String?         @map("serial_number")   // always recorded, even when unmatched

  type         DeviceEventType
  message      String?
  recordCount  Int?            @map("record_count")
  ipAddress    String?         @map("ip_address")
  createdAt    DateTime        @default(now()) @map("created_at")

  device       Device?         @relation(fields: [deviceId], references: [id], onDelete: Cascade)

  @@index([deviceId, createdAt])
  @@index([type, createdAt])
  @@index([createdAt])          // for the retention sweep (FR-D-11)
  @@map("device_events")
}

model DeviceCommand {
  id           Int                 @id @default(autoincrement())
  deviceId     Int                 @map("device_id")
  command      String                                  // raw ADMS command text
  payload      Json?
  status       DeviceCommandStatus @default(PENDING)

  issuedById   Int?                @map("issued_by_id")
  issuedAt     DateTime            @default(now()) @map("issued_at")
  sentAt       DateTime?           @map("sent_at")
  acknowledgedAt DateTime?         @map("acknowledged_at")
  expiresAt    DateTime?           @map("expires_at")
  responseText String?             @map("response_text")

  device       Device              @relation(fields: [deviceId], references: [id], onDelete: Cascade)

  @@index([deviceId, status])
  @@map("device_commands")
}
```

## Change to a feature 02 model

`Department` gains a nullable calendar link, which is how an employee's holiday calendar is resolved
(D-06, FR-W-10):

```prisma
model Department {
  // ... existing fields
  calendarId Int?             @map("calendar_id")
  calendar   HolidayCalendar? @relation(fields: [calendarId], references: [id])
}
```

Resolution order for an employee's calendar, **revised 2026-09-18 for locations (OQ-206)**:

1. **The employee's location's calendar** — public holidays are geographic before they are
   organisational. A Lahore office and a Dubai office observe different days regardless of which
   department anyone sits in.
2. Their **department's** calendar.
3. Walking **up the department tree** — setting one calendar on "Manufacturing" should cover its
   sub-departments without setting it on each.
4. The tenant's **default** calendar.

Location first is the part worth arguing about. The alternative — department first — means a
department spanning two countries gets one country's public holidays, which is wrong in a way nobody
notices until someone is marked absent on their national holiday.

## Design notes

**Why settings are rows with a code-declared catalogue, not a JSON blob on `Company`.** A blob gives
no per-key audit (FR-S-06), no per-key secrecy flag, and no way for feature 07 to add a setting
without migrating a shared column. Rows with a typed accessor give all three, at the cost of one
cached query.

**Why `Setting.value` is a `String` and not `Json`.** The catalogue in code already knows the type
and does the parsing and validation. A `Json` column would allow a value whose shape contradicts its
declared type, and Postgres cannot check the declared type either way. A string keeps one parsing
path instead of two.

**Why recurring holidays are materialised.** Evaluating a rule at read time means `isWorkingDay` —
the hottest helper in the system (NFR-02) — carries rule evaluation, and it makes "move Christmas
observance to the 27th this year only" require an exception table. Materialising costs one job per
year and makes every read a plain indexed lookup.

**Why `commKeyLast4` exists.** Admins need to confirm that the key in the device matches the one the
system has, and "the key ending a91c" is the only way to do that without exposing the key. Four
characters of a 32-byte secret is not a meaningful disclosure.

**Why `DeviceEvent.serialNumber` is stored even when `deviceId` is set.** The unknown-device case has
no device row, and keeping the column populated in both cases means one query answers "everything
ever seen from this serial", including events from before it was registered.

**Why `Device.lastSeenAt` and `lastPushAt` are separate.** A terminal polling `getrequest` every
30 seconds but pushing nothing is reachable and broken. One column cannot express that, and the
difference is precisely what an admin needs when attendance data goes missing.

**Why `DeviceCommand` is here but driven by feature 04.** The queue is device infrastructure; what to
put in it is attendance logic. Defining the table now avoids 04 inventing a second one.

## Expected volume

| Table | Rows | Notes |
|---|---|---|
| `company`, `work_week_days` | 1, 7 | |
| `settings` | ~40 | Read constantly, written rarely — hence the cache (FR-S-07) |
| `holidays` | ~20/year/calendar | |
| `devices` | 1–5 | |
| `device_events` | **~3 000/device/day** | A terminal polling every 30s produces 2 880 `CONNECTED` events daily. See below. |
| `device_commands` | tens | |

`device_events` is the fastest-growing table in the system by an order of magnitude, and this is the
one thing in this feature most likely to cause a production problem. Two mitigations, both required:

1. **Do not log every poll as an event.** Routine authenticated polls update `lastSeenAt` and write
   no event. `CONNECTED` is logged only on a *transition* — the first contact after a gap longer than
   the stale threshold. Failures, unknown devices, and record receipts are always logged.
2. **Retention sweep** (FR-D-11) deletes events older than `device.eventRetentionDays`, run by the
   same nightly job that prunes expired sessions.

Without mitigation 1, a single terminal generates over a million rows a year to record nothing of
interest.

## Working-day resolution

The data behind `isWorkingDay(date, employeeId?)` (D-05, FR-W-09):

1. If an employee is given and feature 04 has a shift covering that date → the shift decides whether
   the day is worked and for how many hours.
2. Otherwise → `WorkWeekDay` for that weekday.
3. Then subtract holidays: resolve the employee's calendar (department → ancestors → default), look
   up `Holiday` by `(calendarId, date)`. `FULL_DAY` → not a working day. `HALF_DAY` → working, hours
   halved. `OPTIONAL` → still a working day here; feature 06 decides what taking it costs.

The batched form, needed by payroll (NFR-02), loads the work week (7 rows), the relevant holidays
(one query for the date range and calendar), and the employee's shifts once, then answers in memory
for every employee-day in the range.

## Seed data

1. **Company** — row 1, with a placeholder legal name and the country from an env var if present.
   Created unconditionally so no screen has to handle "no company yet".
2. **Work week** — 7 rows: Mon–Fri working at 8 hours, Sat–Sun not. Adjusted at install; the default
   is a guess and the UI should say so.
3. **Default holiday calendar** — one row, `isDefault: true`, named "Company holidays". No holidays
   seeded: guessing a country's public holidays wrongly is worse than an empty calendar.
4. **Settings** — nothing written. Defaults come from the catalogue (FR-S-03).
5. **Permissions and role grants** — the seven keys from `requirements.md`.
6. **Devices** — none. Registration is deliberate (D-07).

## Migration notes

One migration, `admin_settings`:

1. Create `company`, `settings`, `work_week_days`, `holiday_calendars`, `holidays`,
   `device_events`, `device_commands`.
2. Add `calendar_id` to `departments` (nullable).
3. Extend `devices` with the new columns, all nullable.
4. **Existing device rows** (created during push testing) keep their serial numbers and get a null
   `comm_key_hash`. Because authentication requires a key match (FR-D-03), those devices are refused
   until an admin completes registration by generating a key. That is the correct behaviour — a
   device that was never authenticated should not become authenticated by a migration — but it means
   **push testing stops working the moment this migration runs**, until keys are issued. Worth
   stating in the release notes; it will otherwise be reported as a bug.
5. Seed, as above.

`.env` / `.env.example`: `DEVICE_COMM_KEY` is superseded by per-device keys (D-08) and should be
removed, with a comment pointing here so nobody re-adds it. If OQ-307 reveals the terminal supports
only one shared key, this decision reverses and the key returns as a setting — not as an env var, so
it can at least be rotated without a redeploy.

## Open questions

| ID | Question |
|---|---|
| OQ-301 | Multi-company. `Company` as a singleton with a fixed id is the cheapest thing to change *now* and among the most expensive later — every table above would need a `companyId`. |
| OQ-303 | If one calendar is enough, `HolidayCalendar` is over-modelled — but removing it later is easy, whereas adding it after four modules call `isWorkingDay` is not. |
| OQ-307 | Per-device vs shared comm key — **blocked on the missing SenseFace manual (OQ-000)**. Decides whether `Device.commKeyHash` is per row or a single setting. |
| OQ-312 | Does the SenseFace 2A report its firmware version, timezone, and clock in the push protocol? `firmwareVersion`, `deviceTimezone`, and `lastClockDriftSeconds` are speculative until the manual confirms them. Blocked with OQ-000. |
| OQ-313 | Should `device_events` be a Postgres table at all, or a log stream? A table is right at this volume with mitigation 1 applied; it is wrong at fifty devices. |
