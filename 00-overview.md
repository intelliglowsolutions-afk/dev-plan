# HRM System — Planning Overview

> Step 4 of the planning process. Written once all eleven feature folders were complete, so that it
> describes what was actually planned rather than what was intended.
>
> **Read this first.** Each feature folder goes deep on one area; this document is the map, the
> shared vocabulary, and the set of patterns that recur across all of them.

> ## ⚠ Status: revision pending (2026-09-18)
>
> The open questions were answered on 2026-09-18 (see **[OPEN_QUESTIONS.md](./OPEN_QUESTIONS.md)**).
> Seven answers overturn decisions written into the feature files, and **those files have not yet
> been revised**. Until they are, read them with these corrections in mind:
>
> | Change | Effect |
> |---|---|
> | **Multi-tenant** (OQ-301) | The plan is written single-company throughout. Every data model needs a tenant discriminator and every query needs tenant scoping. **The largest outstanding change** |
> | **Multiple locations** (OQ-206) | `Location` becomes a model; employees, devices, calendars and reports gain a location dimension |
> | **No automatic deletion** (OQ-1002/105/203) | Retention is reported, never enforced. Feature 10's purge job becomes a report |
> | **NextAuth** (OQ-101) | Feature 01's auth endpoints are rewritten around Auth.js v5, keeping database sessions |
> | **No device comm key** (OQ-307) | The manual confirms the terminal cannot hold one. 03 D-08 is replaced by **D-08b: a per-site collector** holds the credential instead — see [DEVICE-INGESTION-SECURITY.md](./DEVICE-INGESTION-SECURITY.md). OQ-318 resolved; **OQ-319** (hosted vs per-company install) must be answered before ingestion is built |
> | **`employeeCode` is 1–14 alphanumeric** (OQ-204) | Not numeric ≤ 9 digits as assumed |
> | **Payslips also emailed as PDFs** (OQ-709) | Overrides "no sensitive data in email" for this one case |
> | **Multi-step configurable approvals** (OQ-606/407) | Leave and attendance corrections both need real chains |
> | **Recruitment built last** (OQ-1001) | Feature 10 moves after 11 in the build order |

---

## 1. System summary

A self-hosted HR management system for a single company, built around a **ZKTeco SenseFace 2A**
attendance terminal that pushes face-scan punches to the application over the local network.

Eleven features cover the employment lifecycle: who may use the system and what they may see;
employee records and the org structure; company configuration and devices; attendance and shifts;
notifications; leave; payroll; an employee self-service portal; performance; recruitment and
onboarding; and reporting.

**Stack** — Next.js 16 (App Router) · React 19 · TypeScript · Prisma 6 · PostgreSQL 16 · Tailwind 4 ·
Docker Compose (app, postgres, adminer). The device pushes to host port 8081, forwarded to the app.

**Scale it is designed for** — roughly 200 employees per tenant, one or two devices, **multiple
locations** (OQ-206), one currency. **Multi-tenant** (OQ-301): the system serves more than one
company, which is the single biggest constraint on every data model.
Every capacity decision in the plan is made against those numbers, and the points at which each
stops being adequate are recorded as open questions rather than guessed at.

**What it deliberately does not do** — no built-in tax or statutory rules (they are configurable,
per OQ-702); no multi-currency; no automated judgement about individuals (no performance scoring from attendance, no
candidate screening, no attrition prediction); no ad-hoc query builder; no direct external database
access.

---

## 2. Features and build order

| # | Feature | Priority | Depends on | Notes |
|---|---|---|---|---|
| 01 | [Roles, Permissions & Auth](./01-roles-permissions-auth/README.md) | Must | — | Every feature checks permissions and writes audit entries |
| 02 | [Employee Management](./02-employee-management/README.md) | Must | 01 | The core entity; supplies reporting lines for scope |
| 03 | [Admin & Settings](./03-admin-settings/README.md) | Must | 01, 02 | Timezone, holidays, work week, devices |
| 04 | [Attendance Tracking](./04-attendance-tracking/README.md) (+ [shifts](./04-attendance-tracking/shifts.md)) | Must | 01–03 | The largest feature |
| 05 | [Notifications](./05-notifications/README.md) | Must | 01–03 | Arrives owing a backlog from 01–04 |
| 06 | [Leave Management](./06-leave-management/README.md) | Must | 01–05 | Owes 04 a bulk recompute on release |
| 07 | [Payroll](./07-payroll/README.md) | Must | 01–06 | Owns the period lock 04 and 06 defer to |
| 08 | [Employee Self-Service](./08-employee-self-service/README.md) | Must | 01–07 | Adds almost no domain logic |
| 09 | [Performance Management](./09-performance-management/README.md) | Nice | 01, 02, 05, 08 | Deliberately not connected to 04/06/07 |
| 10 | [Recruitment & Onboarding](./10-recruitment-onboarding/README.md) | Nice | 01–03, 05, 08 | Ends by creating an employee in 02 |
| 11 | [Reports & Analytics](./11-reports-analytics/README.md) | Should | all | Reads everything, computes nothing |

The order is a dependency order, not a priority order. 09 and 10 are nice-to-have and can be dropped
entirely; 11 degrades cleanly and can be built early with whatever modules exist.

**Two features could be resequenced with care:** 05 (Notifications) could come earlier, since 01's
invite and password-reset emails do not work until it exists; and 06 before 04 would avoid the bulk
recompute 06 now owes. Neither is recommended — the dependencies run the other way — but both are
worth knowing.

---

## 3. Shared data model

The entities most features touch. Feature-owned detail lives in each folder's `data-model.md`.

```
                        ┌─────────────┐
                        │    User     │  01 — login account, roles, sessions
                        └──────┬──────┘
                          0..1 │ (optional link)
                        ┌──────┴──────┐
        ┌───────────────│  Employee   │───────────────┐   02 — the person
        │               └──┬───┬───┬──┘               │
        │                  │   │   │                  │
   ┌────┴─────┐   ┌────────┘   │   └────────┐    ┌────┴──────┐
   │Department│   │            │            │    │ Candidate │  10 (linked at hire)
   │ Position │   │            │            │    └───────────┘
   │(02, tree)│   │            │            │
   └──────────┘   │            │            │
                  │            │            │
        ┌─────────┴──┐  ┌──────┴──────┐  ┌──┴──────────┐
        │Attendance  │  │Leave ledger │  │  Payslip    │
        │Day / Punch │  │ + requests  │  │ + lines     │
        │   (04)     │  │    (06)     │  │    (07)     │
        └─────┬──────┘  └──────┬──────┘  └──────┬──────┘
              │                │                │
         ┌────┴────┐      reads│           reads│
         │ Device  │  03        └───────┬────────┘
         │ Shift   │  04                │
         └─────────┘              ┌─────┴──────┐
                                  │  Reports   │  11 — reads, never writes
                                  └────────────┘
```

**Key relationships and why they are shaped that way:**

- **`User` ⟷ `Employee` is optional in both directions** (01 D-01). The initial super admin has no
  employee record; factory staff who only punch at the terminal have no login.
- **`Employee` is never deleted** (02 D-08); `employeeCode` is never reused, because attendance rows
  match on it as raw text.
- **`Candidate` is a separate entity** (10 D-01) and is *deleted on a schedule* (10 D-02) — the
  inverse of every other rule in the system, and deliberate.
- **Departments form a tree; the reporting line is its own field** (02 D-03). They differ often
  enough that deriving one from the other gives wrong answers exactly where it matters.
- **Historical structure is kept** (02 D-04): every department, position, manager and employment-type
  change is a dated `EmployeeAssignment` row. Current values on `Employee` are a denormalised
  convenience. Feature 11 depends entirely on this (11 D-06).

---

## 4. The patterns

Nine patterns recur across features. They are listed here because consistency is most of what makes
a system this size learnable, and because a new feature should reach for these before inventing
something.

### 4.1 Code-declared catalogues

A fixed set of things is declared **in code**, with keys referenced from the database. Users
configure *which* and *how*, never *what exists*.

Used for: permissions (01), settings (03), notification types (05), self-service field policies
(08), report definitions (11).

Why: a key that nothing reads is dead weight, and a typo should fail at seed or compile time rather
than silently doing nothing. By the third occurrence it became the house answer to "user-referenced
identifiers into code-defined things".

### 4.2 Facts and derived opinions, kept separate

Immutable evidence in one table; the computed interpretation in another, fully recomputable.

- 04: `attendance_punches` (facts) → `attendance_days` (derived, recomputable)
- 06: `leave_ledger_entries` (facts) → balance snapshots (derived)
- 07: inputs (snapshotted) → payslip lines (derived while draft, frozen at finalisation)

Why: it makes "the shift was wrong, fix it and recompute last month" a routine operation instead of
a data-recovery exercise, and it means an employee disputing a number can be shown the entries that
produce it.

### 4.3 Snapshotting the rules, not just the data

A record keeps a copy of the rules that produced it, so it still explains itself after the rules
change.

Used in: 04 (shift snapshot on the day), 06 (approval chain stored at submission), 07 (payslip
`inputSnapshot` including formula text), 09 (review template and rating-label snapshots), 10
(scorecard criteria, onboarding task text, consent notice text).

Why: a payslip that changes when a formula is edited is not a record of anything.

### 4.4 Preview before commit

Any operation touching many rows previews exactly what it will change, and writes nothing until
confirmed.

Used in: 04 (recompute preview), 06 (accrual and carry-over previews), 07 (hire-population and
variance review), 10 (hire preview, retention purge preview), 11 (report runs are inherently
previews), plus 02's import dry run and 03's impact checks.

Why: an operation that silently rewrites 200 balances is one people refuse to run.

### 4.5 One writer per table

Each table has exactly one feature that writes it. Others request changes through a helper.

Most consequential instance: **leave never writes `attendance_days`** (06 D-09). It marks days dirty
and lets 04's engine reclassify. Two features writing attendance is how a recompute and a leave
approval start fighting.

### 4.6 Explicit visibility states

Content has a state that governs who can see it, and transitions are deliberate acts.

- 09: review `DRAFT` → `SUBMITTED` → `SHARED` — submitting is **not** sharing.
- 10: scorecards hidden from other interviewers until every one is submitted.
- 06: confidential leave types redacted server-side.

Why: a manager who thinks their draft might be visible writes nothing useful; the first submitted
scorecard anchors everyone else's.

### 4.7 Separate read logs for sensitive data

Ordinary changes go to 01's audit log. **Reads** of pay (07) and performance content (09) go to their
own tables.

Why: reads are frequent and would drown the change history that the audit log exists for; and "who
looked at whose salary" is asked differently from "who changed it".

### 4.8 Scope and suppression enforced in the query

Row visibility is a `WHERE` clause, never a filter applied to fetched results (02, 04, 06, 09, 11).
Fetching then filtering leaks row counts through pagination totals. Feature 11 adds suppression of
small-population aggregates, because scope alone does not stop inference.

### 4.9 Refusing automated judgement about individuals

Three features independently say no to the same class of feature:

- 09 D-04: no performance scoring derived from attendance or leave — **and no read path exists** to
  make it possible.
- 10 D-07: no automated candidate screening, scoring or ranking.
- 11 D-09: no scoring, ranking or prediction about individuals.

Each is recorded as an explicit non-requirement rather than an omission, so that "we decided not to"
survives the first feature request — where "we didn't get to it" would not.

---

## 5. Cross-feature contracts

The helpers features owe each other. These matter more than the endpoints: get them right and the
system stays coherent; work around them and it does not.

| Owner | Helper | Used by |
|---|---|---|
| 01 | `requireSession`, `requirePermission`, `protectedRoute` | every feature, every endpoint |
| 01 | `writeAudit(tx, entry)` — takes the transaction, so the audit lives or dies with the change | all |
| 01 → 02 | `employeeScopeFilter` → `resolveScopedEmployeeIds` (recursive CTE) | 02, 04, 06, 09, 11 |
| 02 | `selectEmployeeFields(ctx)` — permission-driven select, so sensitive columns are never fetched | 02, 08, 11 |
| 03 | `getSetting` / `setSetting` (typed, cached) | all |
| 03 | `isWorkingDay` / `workingDaysBetween` — **the only implementation of "is this a working day"** | 04, 06, 07, 11 |
| 03 | `companyToday`, `toCompanyDate`, `startOfCompanyDay` | 04, 06, 07, 11 |
| 04 | `resolveShift(employee, date)` | 04, 06 |
| 04 | `markDirty(employees, range, reason)` | 03, 06, and 04 itself |
| 05 | `notify(typeKey, ctx, tx)`, `supersede(entity)` | 01–04, 06, 07, 09, 10, 11 |
| 06 | `computeLeaveCost`, `getBalance`, `resolvePolicy` | 06, 07, 08 |
| 07 | `isPeriodLocked`, `lockedRangesFor` | **04 and 06 ship calling a stub until 07 exists** |
| 09 | `reviewVisibilityFilter` | 09 only, but it governs every query there |

Two of these are worth flagging for the build:

- **04 and 06 ship against an `isPeriodLocked` stub** that always returns false. Their error
  messages already promise a payroll adjustment route that does not exist until 07 lands.
- **05 arrives owing a backlog.** Features 01–04 each specified notifications that currently only
  write to a log. 01's invite and password-reset emails are the urgent ones: a real invite flow does
  not exist until 05 ships.

---

## 6. Cross-cutting concerns

### 6.1 Authentication and authorisation

Database-backed opaque sessions in an `httpOnly` cookie, not JWTs (01 D-04) — revocation must be
immediate. RBAC is roles → permissions, each grant carrying a **scope**: `ALL`, `DEPARTMENT`, or
`SELF` (01 D-03). Roughly 90 permission keys across the eleven features.

Scope is a data filter, not a yes/no gate (01 FR-Z-05). `DEPARTMENT` resolution is the plan's most
consequential unanswered question (OQ-201).

### 6.2 Time and dates

**Everything is stored UTC.** The company timezone is applied at exactly two edges: deciding which
calendar day a punch belongs to, and display (03 D-03). Changing it is change-controlled because it
re-buckets historical attendance (03 D-04).

Calendar dates — hire date, leave date, holiday — are `@db.Date` and must not be shifted by timezone
conversion. No module parses dates locally; all date arithmetic goes through 03's helpers.

### 6.3 The SenseFace dependency

The device pushes over ADMS/iClock to `/iclock/*`, rewritten to `/api/device/iclock/[[...path]]`.
It is the **only endpoint exempt from session authentication** (01 FR-A-13), authenticated instead by
a per-device comm key (03 D-08).

03 owns device identity, keys, and health; 04 owns parsing and interpretation. The boundary is drawn
precisely in 03's README.

**The manual was found on 2026-09-18** in `C:\Dev\Datasheets` (OQ-000 resolved). Three things it
settled:

- `employeeCode` / user ID is **1–14 characters, alphanumeric** — not numeric ≤ 9 digits as assumed
  (OQ-204). The device also refuses to change an ID after registration, which independently confirms
  02's immutability decision.
- **There is no comm-key field on the device** (OQ-307). Cloud Server Settings offers only domain
  name, server address, server port, and proxy; HTTPS is available and on by default, but it
  authenticates the *server to the device*, not the reverse. 03 D-08 is therefore replaced by
  **D-08b — a per-site collector** that speaks iClock on the LAN and forwards to the platform with a
  real, rotatable credential. That also makes the credential the tenant selector instead of the
  serial number, and its buffer removes WAN-outage attendance gaps.
  See **[DEVICE-INGESTION-SECURITY.md](./DEVICE-INGESTION-SECURITY.md)**.
- The protocol surface does not expose firmware version, device timezone, or clock drift (OQ-312).

The remaining ⚠ assumptions in 04's `api-design.md` — record format, field order, acknowledgement
string — still need confirming against the manual's ADMS section or observed device traffic. The
recommended sequence is unchanged: build 04's engine and shifts against synthetic punches first.

### 6.4 The one internet-facing surface

If feature 10's public application form is built, it is the only unauthenticated write endpoint
reachable from the open internet (10 D-08). It gets its own handler, its own rate limiter, and no
shared code path with authenticated endpoints. Declining to build it (OQ-1003) removes that surface
entirely.

### 6.5 Sensitive data

Three tiers, each with its own boundary:

1. **Pay** (07) — separate permission set, own read log, never in email or any notification body,
   never visible to managers by default.
2. **Performance content** (09) — `performance.read_content` granted to *nobody* by default,
   including HR admin; own read log.
3. **Personal data** (02) — national ID, DOB, address behind `employee.read_sensitive`, implemented
   as a permission-driven Prisma `select` so unauthorised fields are never fetched rather than
   fetched and stripped.

Candidate data (10) is the outlier: least justified to hold, and therefore deleted on a schedule.

### 6.6 The job runner

The plan defines roughly **35 scheduled jobs** across features 03–11. This makes the job runner
critical infrastructure rather than a detail — feature 03 builds it because it is the first to need
one, and where it lives is still open (OQ-315).

Several jobs fail *silently* in the sense that nobody notices for weeks:

| Job | Owner | What its failure looks like |
|---|---|---|
| **Day opener** | 04 | Absent employees never get a row, so they never appear in the absence report |
| Recompute worker | 04 | Attendance quietly stops updating |
| Notification worker | 05 | Approvals wait forever; nobody is told |
| Accrual | 06 | Leave balances stop growing |
| Retention purge | 10 | Candidate data accumulates past its promised deletion date |

Each of those features exposes a status endpoint reporting queue depth *and last successful run* —
a queue of zero with a stale worker is the failure that looks like health.

### 6.7 Degradation

Every feature is specified to work when the ones after it do not exist yet: 04 classifies `ON_LEAVE`
as a no-op until 06 arrives; 05's senders log until it ships; 08 shows explanations rather than empty
boxes for unconfigured modules; 11 omits reports whose source feature is absent.

---

## 7. Migration sequence

One migration per feature, in build order. The ones needing care:

| Migration | Risk |
|---|---|
| 02 `employee_management` | Drops the free-text `department`/`position` columns after backfilling FK models. **Irreversible** — case and whitespace variants collapse. Fine now; not after go-live |
| 03 `admin_settings` | Existing device rows get a null comm key, so **push testing stops working** until keys are issued. Correct behaviour; belongs in release notes |
| 04 `attendance_engine` | Renames `attendance` → `attendance_punches`. **Must be hand-written** — an auto-generated Prisma migration drops and recreates, losing the test rows |
| 05 `notifications` | Drops 01's temporary password-reset outbox in the same release. Two outboxes is how a reset email goes missing |
| 06 `leave_management` | The "one approved leave day per employee per date" index cannot be written as a partial index with a subquery; it needs a denormalised `is_approved` column |
| 07 `payroll` | Wires `isPeriodLocked` into 04 and 06, replacing their stubs |

Every feature seeds its permission keys, its role grants, and its notification types. Several seed
**clearly-labelled example** configuration — one shift (04), leave types (06), pay components (07), a
rating scale and template (09), pipeline and onboarding templates (10). None seeds anything that
looks authoritative: no tax table, no entitlement figures, no holidays.

---

## 8. Consolidated open questions

**175 open questions** are recorded across the eleven features. They are consolidated, tiered, and
given proposed defaults in **[OPEN_QUESTIONS.md](./OPEN_QUESTIONS.md)** — which is where to go next.
Twelve block work; the rest have defaults that silence accepts.

The ones that block or reshape work, rather than filling in a detail:

| ID | Question | Why it matters |
|---|---|---|
| **OQ-000** | The SenseFace 2A manual | Blocks 04's ingestion half and 03's comm-key mechanism. Everything else can proceed |
| **OQ-201** | Is `DEPARTMENT` scope the manager chain, the department subtree, or both? | On the hot path of every scoped query in every feature |
| **OQ-301** | Will this ever serve more than one company or legal entity? | Cheapest decision now, closest to a rewrite later |
| **OQ-601** | The real leave types and entitlements | Feature 06 is all mechanism without them |
| **OQ-701 / 702** | The real pay components; who keeps any tax table current | Feature 07 likewise; a stale tax table is worse than none |
| **OQ-802 / 805** | Portal on personal phones or a shared kiosk? A second language? | Either answer changes feature 08 materially |
| **OQ-1002** | Candidate retention period | Needs a qualified data-protection answer, not a product decision |
| **OQ-1105** | Will anyone want Excel/Power BI connected directly to the database? | Bypasses every permission rule; easy to refuse now |
| **OQ-101** | Auth.js vs hand-rolled sessions | Shapes every endpoint in feature 01 |
| **OQ-401 / S-01** | Night shifts? Rotating patterns? | Each materially changes feature 04's size |

---

## 9. Known gaps in the plan itself

Stated here rather than discovered later:

- **Features 01–04's `ui-ux.md` files predate the workspace design rules** in `C:\Dev\CLAUDE.md` and
  are not grounded in the installed UI/UX skills. 05–11 are. A retro-pass is outstanding.
- **No testing strategy document.** Individual features state their testability requirements — 04's
  and 07's engines must be unit-testable without a database — but there is no overall approach to
  test data, fixtures, or environments.
- **No deployment or operations plan** beyond the existing Docker Compose: no backup procedure
  (though 02 NFR-05 notes that a database-only backup silently loses every uploaded document), no
  monitoring, no upgrade path.
- **No estimates.** Nothing in this plan says how long any of it takes.
- **The `dev-plan/` folder committed inside `hrm-system` on 2026-09-14 is stale** (OQ-004); this
  copy at `C:\Dev\dev-plan` is authoritative.

---

## 10. Where to start

If building tomorrow, in order:

1. Answer OQ-201, OQ-301, and OQ-101 — all three shape code that everything else sits on.
2. Build 01 and 02 together; they are hard to separate in practice, and 02 is what makes 01's
   `DEPARTMENT` scope real.
3. Build 03 next, and get the timezone right before any attendance data exists.
4. Build 04's computation engine and shifts against synthetic punches, deferring ingestion until the
   manual arrives.
5. Build 05 early enough that 01's invite emails work before anyone needs to onboard a real user.

Then 06 → 07 in order, since 07 closes the period-lock loop both 04 and 06 leave open. 08 becomes
worthwhile once 06 and 07 exist. 09, 10, 11 are genuinely optional and can be sequenced by whichever
question the company most wants answered.
