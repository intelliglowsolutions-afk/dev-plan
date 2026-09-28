# 03 — Admin & Settings

**Priority:** Must-have · **Build order:** 3 of 11 · **Status:** planned (Step 3)

## Purpose

Hold the facts about *this company* that every other module needs but none of them owns: who the
company is, what timezone it works in, which days are working days, which days are holidays, and
which attendance terminals are allowed to talk to the system.

Two of these are load-bearing far beyond their apparent size:

- **The company timezone** decides where one day ends and the next begins. Every punch, every late
  calculation, every leave day, and every payroll period depends on it. Get it wrong and attendance
  is quietly wrong by a few hours for everyone, every day.
- **The holiday calendar and work week** decide whether a given date is expected to be worked. That
  single question feeds attendance (absent or not), leave (does a leave day consume a balance), and
  payroll (is this overtime at a premium rate).

## Scope

**In scope**

- Company profile: legal name, trading name, logo, address, registration and tax identifiers,
  fiscal year start.
- Localisation: timezone, date and time format, first day of week, currency, number format.
- Work week: which days are working days by default, and the standard daily hours.
- Holiday calendar: dated holidays, full and half days, recurring entries, bulk import, and a
  year-ahead view.
- System settings registry: typed, code-declared settings that other modules read.
- **Device management**: registering SenseFace terminals, issuing and rotating their comm keys,
  monitoring their health, and viewing their connection history.

**Out of scope (owned elsewhere)**

- Parsing the punches the devices push → [04 Attendance Tracking](../04-attendance-tracking/README.md).
  This feature decides *which devices are allowed to connect and who they are*; 04 decides what their
  data means. The dividing line is the `/iclock/*` endpoint: 03 authenticates it, 04 interprets it.
- Shift definitions and per-employee schedules → 04. This feature only sets the company-wide default
  work week that shifts start from.
- SMTP credentials and delivery settings → [05 Notifications](../05-notifications/README.md), which
  owns sending. The *setting* lives in this feature's registry; the meaning lives in 05.
- Users, roles, and permissions → [01](../01-roles-permissions-auth/README.md). "Settings" here
  means company configuration, not access control.
- Leave types and policies → [06](../06-leave-management/README.md). Pay components → [07](../07-payroll/README.md).
  Both are configuration, but both belong with the module that gives them meaning.

## Files

| File | Contents |
|---|---|
| [requirements.md](./requirements.md) | User stories, functional and non-functional requirements, the settings catalogue, permission keys |
| [data-model.md](./data-model.md) | Prisma models, the `Device` migration, the settings registry, seed and migration notes |
| [api-design.md](./api-design.md) | Settings, holidays, and device endpoints; the working-day helper; device authentication |
| [ui-ux.md](./ui-ux.md) | The settings shell, holiday calendar, device monitoring, states and copy |

## Key decisions taken in this plan

| # | Decision | Rationale |
|---|---|---|
| D-01 | Settings are a **typed registry with a code-declared catalogue** — each key has a type, a default, a validator, and an owning module — stored as rows, not as a free-form JSON blob. | Same reasoning as feature 01's permission catalogue: a setting nothing reads is dead weight, and a typo in a key name should fail at seed time, not silently return undefined in payroll. |
| D-02 | ~~**One company.** A singleton row, not a tenants table.~~ **SUPERSEDED 2026-09-18 — the system is multi-tenant** (OQ-301 answered: yes). `Company` becomes a tenant table, not a singleton; every model in every feature gains a tenant discriminator, and every query, scope check, settings read, scheduled job and report becomes tenant-scoped. | The original reasoning stands and is why this was asked before building: multi-tenancy touches every query and permission check in all eleven features, and must be designed in rather than retrofitted. It now is. **This document's models and endpoints still read as single-company and need a revision pass** — tracked in `IMPLEMENTATION_LOG.md`. |
| D-03 | **All timestamps are stored UTC.** The company timezone is applied at the edges: when interpreting a device punch into a calendar day, and when displaying. | Storing local time makes every DST transition a data-corruption event and every comparison ambiguous. |
| D-04 | The company timezone is **change-controlled**: changing it is a distinct, heavily-warned action that is audited separately, not an ordinary settings edit. | It retroactively changes which day historical punches belong to. It is closer to a data migration than to a preference. |
| D-05 | A **`isWorkingDay(date, employee)` helper** in this feature is the single answer used by 04, 06, 07, and 11. No module re-implements it. | Three modules independently deciding what a holiday is guarantees three different answers, discovered at payroll time. |
| D-06 | Holidays live on **named calendars**; the company has a default one and employees inherit it. | Multiple locations or religions with different holidays are common. Modelling it now costs one nullable FK; retrofitting it means rewriting the helper in D-05 after four modules depend on it. |
| D-07 | Devices are an **explicit allowlist**. A terminal whose serial number is not registered is rejected and the attempt logged — never auto-created. | The device endpoint is the only unauthenticated-by-session surface in the system. Auto-registration means anyone who can reach port 8081 can inject attendance data. |
| D-08 | ~~Each device has its own comm key, stored **hashed**, shown **once** at generation, and rotatable.~~ **NOT IMPLEMENTABLE — withdrawn 2026-09-18.** The SenseFace 2A manual (OQ-000, now read) shows Cloud Server Settings offering only Enable Domain Name, Server Address, Server Port, and Enable Proxy Server. There is no comm-key field; "Comm Key" appears nowhere in either manual. | The device cannot present a shared secret, so the push endpoint cannot be authenticated this way. `Device.commKeyHash`, `commKeyLast4` and the shown-once flow should be **removed rather than left as security theatre**. The serial allowlist, unknown-device logging and per-device disable (D-07) all still apply. **Replaced 2026-09-18 by D-08b.** |
| D-08b | **Authentication moves off the device and onto a per-site collector.** A small service on the customer's LAN speaks iClock to the terminals, buffers, and forwards to the platform over HTTPS with a per-collector credential that the platform issues, hashes, rotates and revokes. Where a customer's IT prefers it, a site-to-site tunnel is an equal-strength alternative. In every deployment: source-IP pinning, a quarantine state for unexpected sources, rate limits, and alerting. | The device cannot hold a secret, so the architecture holds it instead. This also makes the **credential**, not the serial number, the tenant selector — closing the multi-tenancy hole — and the collector's buffer removes WAN-outage attendance gaps, which feature 04 would otherwise classify as `UNKNOWN`. Full reasoning and the model changes: **[DEVICE-INGESTION-SECURITY.md](../DEVICE-INGESTION-SECURITY.md)**. |
| D-09 | Device health is derived from a **heartbeat freshness threshold**, and a device that goes quiet is a surfaced alert, not a silent gap in the data. | A terminal that has been offline for three days is discovered at month-end otherwise — after the attendance data is already wrong. |

## Dependencies

- **Depends on:** 01 (permissions, audit) and 02 (employees, for department-based calendar assignment
  and for naming who registered a device).
- **Depended on by:** 04 (devices, working days, timezone), 06 (holidays in leave-day counting),
  07 (fiscal year, currency, working days for pro-rating), 05 (SMTP settings), 11 (currency, date
  formats, fiscal year).
- **Touches existing code:** the `Device` model in `prisma/schema.prisma` (four fields today) is
  extended here, and `.env`'s single `DEVICE_COMM_KEY` is superseded by per-device keys (D-08). The
  `/iclock/*` route keeps its shape; this feature adds the authentication in front of it.

## The device boundary, precisely

Because it spans two features and is the most security-sensitive path in the system, the split is
worth stating once, explicitly:

| Concern | Owner |
|---|---|
| Is this serial number registered and active? | 03 |
| Does the presented comm key match? | 03 |
| Recording that the device was seen, and its health | 03 |
| Rejecting and logging unknown or unauthorised devices | 03 |
| Parsing the ATTLOG body into punch records | 04 |
| Resolving a PIN to an employee | 04 |
| Deciding what a punch means (late, early, overtime) | 04 |
| Queuing commands back to the device | 03 stores them, 04 decides what to send |

## Open questions

| ID | Question | Why it matters | Proposed default |
|---|---|---|---|
| OQ-301 | Will this system ever serve more than one company, or one company with legally separate entities? | D-02 assumes not. Retrofitting multi-tenancy after eleven features is close to a rewrite; designing for it now costs perhaps 10% more work. This is the cheapest decision on the list to make now and the most expensive to defer. | Single company. |
| OQ-302 | What is the company's timezone, and has it ever changed? Do any employees work in a different one? | D-03/D-04. A second timezone would mean per-location day boundaries, which changes the attendance model materially. | One timezone, set at install. |
| OQ-303 | Do different groups of employees observe different holidays (by location, religion, or contract)? | D-06 models it; whether the UI needs to expose it in v1 depends on the answer. | One company calendar in v1, model ready for more. |
| OQ-304 | Does the work week vary by employee group (e.g. Mon–Fri for office, six days for factory)? | If yes, the default work week is nearly meaningless and shift patterns in 04 carry the real answer. | A single company default, overridable per shift in 04. |
| OQ-305 | Half-day holidays — do they exist, and what does "half" mean in hours? | Affects attendance expectations and leave deduction. | Supported in the model; half = half the standard daily hours. |
| OQ-306 | How many SenseFace terminals are there, and at how many physical locations? | Drives whether device management needs grouping and per-location filtering, and interacts with OQ-206 (locations). | One or two devices, one location. |
| OQ-307 | Should the comm key be per device (D-08) or does the SenseFace 2A only support a single shared key in its Cloud Server settings? | If the device firmware supports only one shared key, D-08 is not implementable as designed and needs rethinking. | Per device — but this is **blocked on the missing manual (OQ-000)** and must be verified before build. |
| OQ-308 | What happens to attendance for a day when a device was offline — is a gap treated as absence, or as unknown pending manual entry? | This is really a feature 04 question, but the health data that answers it is defined here. | Unknown, flagged, never auto-absent. |
