# 03 — Admin & Settings — UI / UX

Builds on the shell and copy principles in [01 ui-ux.md](../01-roles-permissions-auth/ui-ux.md).

## Principles specific to this feature

1. **Every setting says what it does, in consequences, not in definitions.** "Timezone" with a
   dropdown is useless; "Decides which calendar day a punch belongs to" is the information the admin
   actually needs. The description comes from the catalogue, so it is written once and cannot drift
   from behaviour.
2. **Dangerous changes are slow on purpose.** Timezone, work week, and key rotation each take an
   extra deliberate step that names what breaks. Everything else saves immediately.
3. **Device screens are diagnostic tools, not CRUD forms.** The question an admin brings to this
   screen is almost always "why is attendance missing", and the screen should answer it without
   them knowing the protocol.
4. **This is the screen a new install lands on.** It must be usable when nothing is configured, and
   it should say what to do next.

## Screen map

```
/admin/settings                     Settings home — sectioned            settings.read
/admin/settings/company             Company profile                      settings.read
/admin/settings/localisation        Timezone, formats, currency          settings.read
/admin/settings/work-week           Working days and hours               settings.read
/admin/settings/advanced            Generated from the catalogue         settings.read
/admin/holidays                     Holiday calendar                     holiday.read
/admin/holidays/import              CSV import                           holiday.write
/admin/devices                      Device list and health               device.read
/admin/devices/new                  Register a device                    device.write
/admin/devices/:id                  Device detail and event log          device.read
```

`/admin/devices` must be reachable by someone holding only `device.read`/`device.write` and no HR
permissions (OQ-311) — the navigation cannot assume settings and devices travel together.

## `/admin/settings` — the home

A sectioned landing page rather than a long form: Company, Localisation, Work week, Holidays,
Devices, Notifications (feature 05), and Advanced. Each card carries its current headline value —
"Asia/Karachi · DD/MM/YYYY · PKR", "Mon–Fri, 8h/day", "2 devices, 1 offline" — so the page answers
"how is this system configured" without any clicking.

On a fresh install the cards show a setup state instead: **"Not configured yet"** with a short
explanation of why it matters and a direct link. An explicit ordered checklist sits at the top until
it is complete:

> **Finish setting up**
> 1. ✅ Company name
> 2. ⬜ Timezone — attendance dates depend on this. *Set it before recording attendance.*
> 3. ⬜ Working days
> 4. ⬜ This year's holidays
> 5. ⬜ Register your attendance terminal

Item 2's caution is deliberate: setting the timezone after attendance exists is a change-controlled
action with consequences, and the checklist is the cheapest place to prevent that.

## `/admin/settings/localisation`

The most consequential screen in the feature. Fields: timezone, date format, time format, first day
of week, currency, number format, with a live preview panel showing a sample date, time, and amount
as they will appear throughout the system.

**Timezone** is not an ordinary field. It is displayed with its current value and a *Change* button;
clicking it opens a dedicated dialog:

> **Change the company timezone?**
> Currently **Asia/Karachi (UTC+5)**. Changing to **Asia/Dubai (UTC+4)** will:
> - Re-assign 2 480 recorded punches to different calendar days. Some days will gain or lose
>   attendance records.
> - Change which day future punches are counted on, from tonight.
> - Reports run before and after this change will not match.
>
> This does not alter the punch times themselves, only which day they belong to.
>
> Type **Asia/Dubai** to confirm.

The consequence lines come from the server (`api-design.md`), so they reflect the real record count.
Type-to-confirm is used here and for key rotation only — reserving it for genuinely irreversible
actions keeps it meaningful.

Currency and fiscal year get a lighter confirmation: a dialog naming the effect on payroll periods
and payslips, without type-to-confirm.

## `/admin/settings/work-week`

A seven-row grid: day, a working/not toggle, and standard hours. A summary line beneath: "5 working
days, 40 hours per week", which turns red when it disagrees with the configured standard weekly
hours, with an explanation rather than a block — some contracts legitimately state a different
weekly figure (FR-W-02).

Saving is change-controlled: "This changes what counts as a working day for everyone without a
shift, for past and future dates. Attendance reports for previous months may change."

A note below the grid points at feature 04: "Employees on a shift follow their shift instead. This
is the default for everyone else."

## `/admin/holidays`

Two views over the same data, toggled: **Calendar** and **List**.

**Calendar** — a twelve-month grid for the selected year, holidays marked and colour-coded by type,
half-days shown with a half-fill. Weekends are shaded using the configured work week, so a holiday
falling on a non-working day is visibly redundant. Clicking a day adds a holiday; clicking an
existing one edits it.

**List** — a table: date, day of week, name, type, duration, calendar, recurring flag. Sortable,
filterable by type and calendar. This is the view HR uses when entering a year's worth at once.

Header controls: year selector, calendar selector (hidden entirely when only the default calendar
exists — OQ-303), **Add holiday**, **Import**, and **Generate recurring for {next year}**.

A persistent banner when coverage is thin (FR-W-12): "Only 14 days of holidays are set beyond today.
Add next year's holidays so leave and attendance stay accurate." — dismissible per session, not
permanently, since it re-earns its place every year.

### Adding a holiday with impact

The add/edit dialog is small: date, name, type, duration, calendar, repeat annually. Where it earns
its keep is the impact check. If the date already carries activity, a warning panel appears **before**
saving:

> **3 approved leave requests fall on this date.**
> Marking it a holiday does not automatically give those days back. Review them in
> Leave → Requests after saving. *(View them)*
> - Sara Ahmed · Annual leave · 26 Dec
> - …

The wording matters: the honest statement is that the system will not fix this for them, plus a route
to where they can. Silence here produces three people who believe they have a day of leave they no
longer have.

Deleting a holiday shows the mirror-image warning: attendance already recorded against it, and
employees who may now be retroactively marked absent.

## `/admin/holidays/import`

The same wizard shape as feature 02's employee import — template, upload, preview with per-row
errors, then write — and the same rule: **nothing is written if any row fails**, stated on the
screen rather than implied. Duplicate dates within the file, and dates that already exist in the
calendar, are shown as separate error categories, since the fixes differ.

## `/admin/devices`

Cards rather than a table — there are rarely more than a handful, and each one wants a status at a
glance:

```
┌─────────────────────────────────────────┐
│ ● Main gate                     ONLINE  │
│   SenseFace 2A · Building A             │
│   SN 4823901 · key ending a91c          │
│   Last seen 40 seconds ago              │
│   148 records today                     │
└─────────────────────────────────────────┘
```

Health is the dominant element: a green dot for online, amber for stale, red for offline, grey for
disabled, and an outline for never-connected. Every state carries a word as well as a colour.

Offline and stale cards lead with the problem and a plain-language reading of it:

> ⚠ **Offline — last seen 2 days ago (13 Sep, 18:04)**
> No attendance has been recorded from this device since then. Check that it is powered on and
> connected to the network. *(View event log)*

The second sentence is the one that matters to HR, who do not think in devices but in missing data.

An **unknown device attempts** panel appears above the list when unregistered serials have been seen
(FR-D-04): "A device with serial 7710233 tried to connect 14 times from 192.168.1.40, most recently
2 minutes ago." with *Register it* and *Ignore*. This is simultaneously the security surface and the
happy path for installing a new terminal.

## `/admin/devices/new` — registration, and the key

A short form: serial number (with a note on where to find it on the device), name, location, model.

On save, a **modal that cannot be dismissed by clicking outside**:

> **Comm key for Main gate**
>
> `k9f2-8Xq4-…-a91c`  *(Copy)*
>
> Enter this in the device: **Menu → Comm. → Cloud Server Setting**, together with the server
> address `http://192.168.1.10:8081`.
>
> **This key is shown once.** We store only a hash of it, so we cannot show it again. If it is lost,
> rotate the key and enter the new one in the device.
>
> ☐ I have copied the key
> *(Done — disabled until ticked)*

The server address is assembled from the configured host and the device push port, so the admin does
not have to work out what to type into the terminal. The checkbox is the point: a modal dismissed by
a stray click has cost the key.

*Rotate key* on the device detail page uses the same modal, preceded by a type-to-confirm warning:
"The device will stop recording attendance until you enter the new key into it."

## `/admin/devices/:id`

Header: name, serial, health, location, model, key fingerprint, registered by and when.
Actions: *Edit*, *Rotate key*, *Disable*, *Delete* (offered only when no attendance rows exist,
otherwise disabled with "This device has 4 812 attendance records. Disable it instead.").

**Health panel** — last seen, last push, records today / this week, and clock drift where available.
Drift gets its own line when it exceeds a minute, because it is invisible otherwise and it makes
every punch wrong: "Device clock is 4 minutes ahead of the server. Punch times will be recorded 4
minutes early."

**Event log** — a filterable, paginated table: time, type, message, record count, IP. Types are
shown as readable phrases ("Authentication failed", "Records received"), with colour only as
reinforcement. Retention is stated at the bottom of the list — "Events older than 90 days are
removed" — so an admin looking for last quarter is not left wondering.

A small chart of records received per day for the last 30 days sits above the log. It makes a gap
obvious in a way a table of events never does, and gaps are the thing being looked for.

**Commands** — the queue, with status and acknowledgement times. Read-only in this feature; feature
04 decides what can be queued.

## `/admin/settings/advanced`

Generated entirely from the settings catalogue (FR-S-08): grouped by module, each row rendering per
its declared type — toggle, number, select, text. Each shows its label, description, current value,
whether it differs from the default (with a *Reset to default*), and who changed it last.

Change-controlled rows carry a lock icon and require the confirmation flow. Secret rows show
"Set — last changed 12 Sep" and a *Replace* action, never a value.

A search box across keys, labels, and descriptions, because this page will eventually hold forty
settings and nobody remembers which module owns which.

## States, accessibility, responsive

- Settings screens save **on explicit save**, not on blur. A timezone that changes because focus
  moved is exactly the wrong behaviour.
- Unsaved changes warn on navigation; a sticky footer shows the count and Save/Discard.
- Every health state, holiday type, and event type carries text alongside its colour.
- The holiday calendar grid is navigable by keyboard, and each day cell's accessible name includes
  the date, weekday, and holiday name where present. The list view is the accessible equivalent of
  the grid and is always available, not a mobile fallback.
- The records-per-day chart has a table equivalent behind a toggle.
- Device cards stack on narrow screens; the holiday calendar drops to a single month at a time below
  768 px.

## Copy reference

| Situation | Text |
|---|---|
| Setup checklist, timezone | Attendance dates depend on this. Set it before recording attendance. |
| Timezone change | This does not alter the punch times themselves, only which day they belong to. |
| Work week change | This changes what counts as a working day for everyone without a shift, for past and future dates. |
| Shift note | Employees on a shift follow their shift instead. This is the default for everyone else. |
| Holiday impact | Marking this a holiday does not automatically give those days back. Review them in Leave → Requests after saving. |
| Thin coverage | Only {n} days of holidays are set beyond today. Add next year's holidays so leave and attendance stay accurate. |
| Key shown once | This key is shown once. We store only a hash of it, so we cannot show it again. |
| Key rotation | The device will stop recording attendance until you enter the new key into it. |
| Device offline | No attendance has been recorded from this device since then. Check that it is powered on and connected to the network. |
| Unknown device | A device with serial {sn} tried to connect {n} times from {ip}, most recently {time}. |
| Clock drift | Device clock is {n} minutes ahead of the server. Punch times will be recorded {n} minutes early. |
| Device delete blocked | This device has {n} attendance records. Disable it instead. |
| Serial taken (disabled device) | Serial {sn} is already registered to {name}, which is currently disabled. Reactivate it instead of registering it again. |
| Event retention | Events older than {n} days are removed. |

## Open questions

| ID | Question |
|---|---|
| OQ-303 | Whether the calendar selector appears at all. Hidden when there is one calendar; the moment there are two, several screens need a calendar column. |
| OQ-311 | Whether device screens must be reachable without HR permissions — changes the navigation structure, not just the gating. |
| OQ-316 | Should the setup checklist persist as a dismissible card after completion, or disappear? Disappearing is cleaner; persisting helps when configuration is revisited a year later by someone new. |
| OQ-317 | The device server address shown in the registration modal needs the externally-reachable host, which the app cannot reliably detect behind Docker. Probably a setting the admin confirms once. |
