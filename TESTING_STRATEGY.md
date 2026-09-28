# Testing Strategy

**Written 2026-09-18**, closing the largest gap named in `DEFINITION_OF_DONE.md` Part 5.
Read before writing code, not after.

---

## What makes *this* system hard to test

Generic advice would not help here. Six specific properties drive everything below:

| Property | Why it is hard | Where it bites |
|---|---|---|
| **Time is the domain** | Timezones, shift windows crossing midnight, DST, "which calendar day does this instant belong to" | 03, 04, 06, 07 |
| **Money must be reproducible** | A payslip issued in March must render identically in December after salaries and formulas have changed | 07 |
| **State is derived and recomputable** | The same inputs must produce byte-identical output, every time, or recompute is unsafe | 04, 06, 07 |
| **Isolation fails open** | One forgotten `WHERE tenant_id` returns another company's payroll, and nothing visibly breaks | all |
| **The hardware is not available** | Ingestion depends on a device protocol we cannot exercise on demand | 03, 04 |
| **Permissions are a large matrix** | ~90 keys × 3 scopes × dozens of endpoints. Nobody tests that by hand | 01, and every feature |

Four of those six fail **silently**. That is what the strategy is shaped around.

---

## The shape: four layers, unequal value

Not a pyramid drawn from habit — a distribution matched to where this system's risk actually sits.

```
      ▲  E2E (few)            the three flows that must never break
     ███ Integration (many)    permissions, tenancy, transactions — needs a real DB
    █████ Contract (focused)   the ~14 cross-feature helpers
   ███████ Pure unit (most)    the two engines, plus every date and money calculation
```

### Layer 1 — Pure unit tests: the engines

**The highest-value tests in the project**, and both features already require them:
04 NFR-09 and 07 FR-X-09 say the engines must be testable with no database, no device, and no clock.

- **04's classification engine** takes punches, a shift, holidays and leave; returns a classified
  day. A test is an array of timestamps in, a status out.
- **07's calculation engine** takes an input snapshot; returns payslip lines.

If either engine can only be tested through HTTP and a database, that is a design defect, not a
testing inconvenience — and it should be fixed in the code rather than worked around in the tests.

**Cases that must exist**, because they are where the plan already predicted trouble:

| Engine | Cases |
|---|---|
| 04 | Night shift 22:00–06:00 with punches either side of midnight · a punch at 06:30 with and without an open session · two punches 20 seconds apart · in with no out · zero punches on a working day · zero punches during a device gap · half-day holiday · leave plus a half day worked · overtime above the daily cap · a day before hire and after termination |
| 07 | Pro-rated joiner and leaver · unpaid leave deduction · overtime at the multiplier · a bracket-table lookup at each boundary **and exactly on each boundary** · rounding applied once · negative net · a formula referencing a component of higher calculation order (must fail at validation, not at run) |

### Layer 2 — Contract tests: the cross-feature helpers

The ~14 helpers in `00-overview.md` §5 are where features meet, so they are where drift happens.
Each gets a test suite that is **owned by the provider and consumed by the caller**:

`requirePermission` · `employeeScopeFilter` / `resolveScopedEmployeeIds` · `selectEmployeeFields` ·
`getSetting` · `isWorkingDay` / `workingDaysBetween` · `companyToday` / `toCompanyDate` ·
`resolveShift` · `markDirty` · `notify` / `supersede` · `computeLeaveCost` · `getBalance` ·
`isPeriodLocked` · `reviewVisibilityFilter`

Two deserve special attention:

- **`isWorkingDay`** is called by four features. Its test suite is the single source of truth for
  what a working day is; if 06 and 07 disagree about a holiday, this is where it is caught.
- **`isPeriodLocked`** ships as a stub in 04 and 06 and is replaced by 07. The stub and the real
  implementation must pass **the same contract tests**, or the replacement silently changes
  behaviour in two features that were built against the stub.

### Layer 3 — Integration tests: what needs a real database

Postgres-backed, because these properties do not exist without it:

- **RLS actually works.** A query without the tenant predicate returns zero rows. This cannot be
  tested against a mock, and it is the enforcing layer for the system's most dangerous failure.
- **Transactions roll back together.** An audit entry and its change; a leave reservation and its
  request; `notify()` inside a caller's transaction that aborts.
- **Unique constraints hold** — including the composite ones and the deliberate global exceptions.
- **Concurrency**: two leave requests against a 2-day balance submitted simultaneously; two
  notification workers claiming the same delivery; two payroll runs on one period.
- **Scope queries**: the recursive CTEs for transitive reports and department subtrees.

### Layer 4 — End-to-end: three flows only

E2E tests are slow and brittle; buy only what is worth the price:

1. **Punch → attendance day → payslip line.** The system's spine. If this works, the core is real.
2. **Leave request → approval → attendance reclassification → payroll effect.** The most
   cross-feature path in the system.
3. **Sign in → forced password change → permission-gated navigation.** The gate everything sits
   behind.

Everything else is covered more cheaply at a lower layer.

---

## Test data: two tenants, always

**Every test runs with a second tenant present.** Not a special isolation suite — always.

```
Tenant "Acme"    the tenant under test. Known shape, known people.
Tenant "Globex"  a mirror-image tenant that no test ever asserts on.
```

Globex exists so that a leak has somewhere to leak *from*. A test asserting "the manager sees 3
employees" passes whether or not tenancy works, if Acme is the only tenant in the database. With
Globex present and holding employees with the **same employee codes, the same department names, and
overlapping dates**, the same assertion becomes meaningful.

The deliberate collisions matter: identical `employeeCode` values across tenants prove the composite
uniqueness is right, and would catch a forgotten tenant predicate in a way that distinct data never
would.

### The canonical fixture

One seeded shape, used everywhere, documented in one file:

- 3 departments, one nested; 2 locations; a 3-level reporting chain.
- ~12 employees covering: a manager with reports, a manager with none, an employee with no user
  account, a terminated employee, a future-dated joiner, a rehire with two employment periods.
- One fixed shift, one night shift, one rotating pattern, one employee on the work-week fallback.
- A holiday calendar per location, including a half-day and one holiday falling on a weekend.
- Leave: a balance with carry-over expiring, a pending request, an approved request spanning a
  weekend and a holiday.
- Payroll: one finalised period (locked), one open.

**Factories for variation, fixtures for the baseline.** Tests that need something unusual build it;
tests that need "a normal company" reuse the fixture. A test file that constructs twelve employees
inline is a test file nobody will maintain.

---

## Controlling time

The single most common source of flaky tests in a system like this.

1. **No `new Date()` in business logic — ever.** The clock is injected. 04's engine already forbids
   it (FR-C-20), and the rule extends to every feature.
2. **Tests run at a fixed instant.** A "golden date" is chosen deliberately: a Wednesday, mid-month,
   mid-year, not adjacent to a DST transition — so that ordinary tests are boring.
3. **The awkward dates get their own tests**, named for what they are: month end, year end, 29
   February, the DST spring-forward and fall-back instants in the tenant's timezone, a leave year
   boundary, a payroll cut-off.
4. **The server's timezone is set to something inconvenient in CI** — `UTC-8` or similar, never the
   tenant's. A test suite that passes only because the server happens to run in the company's
   timezone is testing nothing, and this is precisely the bug 03's helpers exist to prevent.

---

## The permission matrix, generated not hand-written

~90 permission keys against dozens of endpoints is not a hand-written suite.

Because permissions are a **code-declared catalogue** (01 D-02), the tests can enumerate it:

- **Every endpoint declares a permission** — a test walks the route registry and fails on any that
  does not. This is the fail-closed guarantee, asserted rather than assumed.
- **Table-driven access tests**: for each endpoint, a row per role stating expected outcome
  (200 / 403 / 404). Adding an endpoint without adding rows fails the suite.
- **Scope assertions** are separate and per feature, because "sees 3 of 12" is domain-specific.
- **404-not-403 where existence is sensitive** is asserted explicitly — it is a deliberate decision
  (01 FR-Z-07, 09, 10) and the kind that gets "tidied" into a 403 by someone being helpful.

The same generation trick applies to the other code-declared catalogues: settings, notification
types, self-service fields, report definitions. Each gets a test that the catalogue is internally
consistent — no unknown variables in templates, no report without a permission, no setting without a
validator.

---

## Properties worth asserting, not just examples

A handful of invariants are better expressed as properties than as cases:

| Property | Statement |
|---|---|
| **Recompute is idempotent** | Computing a day twice produces an identical `resultHash` and writes nothing the second time (04 FR-R-05) |
| **Corrections survive recompute** | Apply a correction, recompute, assert the correction still applies (04 FR-X-03) |
| **Ingestion is idempotent** | Replaying a batch inserts nothing new and still acknowledges (04 FR-I-04) |
| **Accrual is idempotent** | Running a period twice writes one set of entries (06 FR-Y-01) |
| **The ledger sums to the balance** | For any employee and type, snapshot equals the sum of entries (06 FR-B-08) |
| **Payslips reconcile** | Lines sum to the total, for every payslip in a run (07 FR-X-06) |
| **Payslips are frozen** | Render a finalised payslip, mutate the component and the employee's salary, re-render, assert identical (07 D-03) |
| **Suppression resists differencing** | No suppressed value is derivable from the total and the visible groups (11 FR-S-04) |

The last one deserves a real test rather than a comment: it is the requirement most likely to be
implemented as "hide the small group" and shipped.

---

## Testing the device protocol without the device

Ingestion is the one place where the real dependency cannot be exercised on demand.

1. **A device simulator** — a small script that speaks the iClock protocol as the manual describes:
   handshake, push, poll for commands. It is the test double *and* the tool for exercising a real
   deployment before a terminal is installed.
2. **Recorded traffic.** The first time a real terminal connects, capture the exchange verbatim and
   commit it as a fixture. That recording is worth more than any amount of inference from the
   manual, and the ⚠ assumptions in 04's `api-design.md` should be confirmed against it and the
   markers removed.
3. **Collector contract tests** on both sides of `/api/ingest/batch` — the collector's forwarder and
   the platform's receiver tested against the same fixtures, including the failure paths: credential
   revoked, serial from another tenant, buffer replay after an outage.
4. **Quarantine paths** tested explicitly: unpinned IP, implausible timestamp, unknown collector.

---

## What not to spend effort on

Stated so the effort goes where it matters:

- **UI snapshot tests.** They break on every legitimate change and catch almost nothing. The six
  UI states are worth testing as behaviour (does the empty state render when the list is empty),
  not as pixels.
- **Line-coverage targets.** A percentage is not a goal. **The real target is already written:
  every numbered acceptance criterion in every feature's `requirements.md` is an automated test.**
  There are roughly 140 of them, they were written to be executable, and they are a far better
  measure of done than 80% of lines.
- **Mocking the database in integration tests.** The properties being tested — RLS, constraints,
  transactions, concurrency — exist only in Postgres. A mocked one tests the mock.

---

## Environments and safety

| Environment | Notes |
|---|---|
| **Local** | Docker Compose with Postgres. **RLS enabled, exactly as in production** — a local database without it lets a missing predicate pass every local test |
| **CI** | Ephemeral Postgres per run; server timezone deliberately not the tenant's; migrations run forwards from empty **and** against a restored snapshot |
| **Staging** | Anonymised data only |

**Three hard safety rules**, each guarding against a mistake this plan can already foresee:

1. **`notification.deliveryMode` is `LOCAL_OUTBOX` everywhere but production**, and it is an
   explicit setting, never inferred from `NODE_ENV` (05 FR-D-08). A container pointed at a copy of
   production data with the wrong mode emails the entire company.
2. **No production data in any other environment**, and no restore of production into staging
   without anonymisation. This database holds salaries, national IDs and performance reviews for
   multiple tenants.
3. **Tests never touch a real SMTP server, a real bank file destination, or a real device.**

---

## Where to start

The order matters, because early choices constrain later ones:

1. **The canonical fixture and the two-tenant harness.** Everything else builds on them, and
   retrofitting Globex later means revisiting every assertion.
2. **The injected clock**, before any date logic exists.
3. **04's and 07's engine suites**, alongside the engines themselves — they are the tests that
   justify the engines' pure design.
4. **The generated permission and catalogue-consistency suites**, as soon as the catalogues exist.
5. **RLS isolation tests**, with the first tenant-owned table.
6. Contract suites as each helper lands; E2E last, and only the three flows.
