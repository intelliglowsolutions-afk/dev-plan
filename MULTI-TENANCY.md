# Multi-Tenancy — cross-cutting specification

**Status:** written 2026-09-18 in response to OQ-301 (answered: yes, multi-tenant).
**Applies to:** all eleven features. Where a feature's `data-model.md` conflicts with this document,
this document wins until that file is revised.

---

## Why this is one document rather than eleven edits

Adding a `tenantId` column to every model is the easy part and not the risky part. The risky parts
are the same in every feature, and getting different answers to them in different features is how
a multi-tenant system leaks:

- Which **unique constraints** must become composite, and which must stay global.
- How the **unauthenticated device endpoint** knows which tenant a punch belongs to.
- Whether a **forgotten `WHERE tenant_id = …`** returns another company's payroll.
- How **scheduled jobs** behave when there are forty tenants and one of them fails.
- What a **platform operator** can see, versus a tenant's own super admin.
- How **aggregates and reports** avoid crossing tenants while already juggling scope and suppression.

Deciding these once, here, is the whole point of the document.

---

## D-T-01 — Isolation strategy: shared schema, `tenantId` column, enforced by Postgres RLS

Three options were considered:

| Option | Isolation | Cost |
|---|---|---|
| **Database per tenant** | Strongest — a query cannot cross tenants | Migrations × N, connection pool × N, painful at more than a handful of tenants |
| **Schema per tenant** | Strong | Prisma support is awkward; migrations still × N |
| **Shared schema + `tenant_id`** | Weakest by default — one forgotten predicate leaks everything | One migration, one pool, simple operationally |

**Chosen: shared schema with a `tenant_id` column on every tenant-owned table, plus PostgreSQL
Row-Level Security as a second line of defence.**

RLS is what makes the weak option acceptable. A policy of the form
`USING (tenant_id = current_setting('app.tenant_id')::int)` means a query that forgets its predicate
returns **nothing** rather than everything. The application sets `app.tenant_id` once per request,
inside the transaction, from the session.

This matters more here than in most systems: this database holds salaries, national IDs, and
performance reviews for multiple companies. A cross-tenant leak is not a bug report, it is an
incident with two customers in it.

**If the tenant count stays very small (2–3) and isolation is worth more than operational
simplicity, database-per-tenant is a legitimate reversal of this decision** — but it must be taken
before the first migration, not after.

## D-T-02 — Every tenant-owned table carries `tenantId`, including child tables

Denormalised onto children rather than reached through a parent join. `PayslipLine` carries
`tenantId` even though it could be derived through `Payslip`.

Why: RLS policies must be evaluatable per row without a join, and a composite index beginning with
`tenant_id` is what keeps queries fast once several tenants share a table.

**Not tenant-owned** (global): the code-declared catalogues (permissions, settings definitions,
notification types, report definitions, self-service field definitions), and the `Tenant` table
itself.

## D-T-03 — A user belongs to exactly one tenant

One `User` row, one tenant. Someone who works for two tenants has two accounts with two passwords.

Why: cross-tenant identity means every session carries a tenant selection, every permission check
becomes two-dimensional, and the login flow needs a tenant picker. That is a substantial feature and
it serves a rare case. Revisit only if it becomes common (**OQ-T-01**).

**`User.email` stays globally unique** so login needs no tenant hint. Everything else — employee
codes, department names, leave type codes, pay component codes — becomes unique **per tenant**.

## D-T-04 — Two levels of administrator

| Role | Scope | Sees |
|---|---|---|
| **Platform operator** | The installation | Tenant list, provisioning, health, job status. **Not** employee, payroll, or performance data |
| **Tenant super admin** | One tenant | Everything within their tenant, as feature 01 already describes |

Feature 01's `SUPER_ADMIN` becomes the **tenant** super admin. The platform operator is a new,
separate concept that deliberately does not inherit tenant data access.

A platform operator who needs to see inside a tenant to support it must be granted access by that
tenant, and the grant is time-boxed and audited. Support access that is silent and permanent is how
a hosting provider ends up reading a customer's payroll.

## D-T-05 — Tenant resolution: three paths, one result

Every request resolves to exactly one `tenantId` before any query runs.

| Entry point | How the tenant is resolved |
|---|---|
| **Authenticated request** | From the session's user. Never from a header, query parameter, or request body — a client-supplied tenant is an authorisation bypass waiting to happen |
| **Device push** (`/iclock/*`) | From the **serial number** → `Device` row → its tenant. This is the only reason the serial allowlist is load-bearing rather than merely tidy |
| **Public application form** (feature 10) | From the posting's `publicSlug` → `JobPosting` → its tenant |

All three are server-derived. There is no code path where the caller states which tenant they are.

The device case deserves emphasis: with no comm key available on the SenseFace 2A (OQ-307), the
serial number is simultaneously **the tenant selector and the only identifier**. An attacker who
guesses a registered serial can write attendance into that tenant. This raises the stakes on OQ-318
considerably and is an argument for per-tenant network isolation rather than a single shared
ingestion port.

## D-T-06 — Scheduled jobs run per tenant, and fail per tenant

The plan defines ~35 scheduled jobs. With N tenants, each becomes N runs.

Rules:

1. A job iterates tenants and **isolates failure**: one tenant's accrual failing must not stop the
   other thirty-nine. Same discipline as feature 07's per-employee payroll calculation.
2. Job status is reported **per tenant** — "accrual last ran successfully for 38 of 40 tenants" is
   the useful sentence, and the two failures must be nameable.
3. Jobs that write notifications set the tenant on every row.
4. Timezone-sensitive jobs (04's day opener, 05's digest, 06's accrual) run **per tenant's
   timezone**, not on one global clock. OQ-302 answered "one company, one timezone" — which now
   means *one timezone per tenant*, not one for the installation.

Point 4 is easy to miss and produces attendance bucketed into the wrong day for every tenant outside
the server's timezone.

## D-T-07 — Settings and configuration are per tenant; their definitions are global

Feature 03's settings catalogue stays code-declared and global; the **values** become per-tenant
rows. Same for notification templates (05), leave policies (06), pay components (07), shifts (04),
review templates (09), pipeline and onboarding templates (10).

A new tenant starts with **no configuration and the catalogue's defaults**, and is walked through
03's setup checklist. The example seeds — one shift, three leave types, example components — become
**per-tenant provisioning options**, offered during setup rather than written into the database at
migration time.

## D-T-08 — Reports and aggregates are tenant-scoped before anything else

Feature 11's pipeline gains a step zero: tenant scoping, applied before permission scoping, before
`employeeScopeFilter`, and before suppression.

Suppression thresholds (11 FR-S) are evaluated **within** a tenant. A tenant with 12 employees will
see most department-level aggregates suppressed, which is correct — and is a reason to revisit
OQ-1103's threshold of 5 now that tenant size varies.

There is no cross-tenant report. Platform-level metrics (tenants, active users, storage) are a
separate, operator-only surface that touches no employee data.

## D-T-09 — Uniqueness, rewritten

The single most error-prone part of this change. Every unique constraint in the plan must be
classified:

**Becomes composite with `tenant_id`:**

| Feature | Constraint |
|---|---|
| 02 | `employees.employee_code`, `employees.work_email`, `departments.name`, `departments.code`, `positions [title, department_id]` |
| 03 | `devices.serial_number` — **see the caveat below**, `holiday_calendars.name`, `holidays [calendar_id, date]` |
| 04 | `shifts.name`, `shifts.code`, `attendance_days [employee_id, date]` (already implies tenant via employee, but the index should lead with tenant) |
| 06 | `leave_types.name`, `leave_types.code`, the ledger's `[employee, type, period_key, kind]` |
| 07 | `pay_components.code`, `salary_structures.name`, `pay_periods [start, end]` |
| 09 | `rating_scales.name`, `review_templates.name`, `performance_cycles.name` |
| 10 | `job_postings.public_slug` — **must stay globally unique**, since it is resolved before a tenant is known |
| 11 | saved views, schedules |

**Stays globally unique:** `users.email` (D-T-03), `job_postings.public_slug` (D-T-05),
`sessions.token_hash`, `password_reset_tokens.token_hash`, all storage keys for uploaded files.

**`devices.serial_number` is the awkward one.** It is physically globally unique, and it resolves the
tenant (D-T-05) — so it must be **globally unique**, not composite. Two tenants cannot register the
same serial, and attempting to is a meaningful error: either a device is being moved between
tenants, or someone is trying to hijack another tenant's attendance feed. The error message must not
reveal which tenant currently holds it.

## D-T-10 — File storage is partitioned by tenant

Uploaded documents (02), candidate CVs (10), and company logos (03) are stored under a
per-tenant prefix on the volume: `/<tenantId>/<uuid>`. The download endpoint checks the tenant of
the row **and** that the storage key sits under that tenant's prefix — belt and braces, because a
mismatched key is either a bug or an attack and both should fail.

Backups and any future per-tenant export or deletion become tractable as a result.

---

## The `Tenant` model

```prisma
enum TenantStatus {
  ACTIVE
  SUSPENDED     // non-payment or abuse; data retained, access blocked
  ARCHIVED      // read-only
}

model Tenant {
  id            Int          @id @default(autoincrement())
  // Used for subdomain or path routing if ever needed; unique and immutable.
  slug          String       @unique
  name          String
  status        TenantStatus @default(ACTIVE)

  // OQ-302 now means one timezone PER TENANT (D-T-06.4).
  timezone      String       @default("UTC")
  locale        String       @default("en")
  currency      String       @default("USD")

  provisionedAt DateTime?    @map("provisioned_at")
  suspendedAt   DateTime?    @map("suspended_at")
  createdAt     DateTime     @default(now()) @map("created_at")
  updatedAt     DateTime     @updatedAt @map("updated_at")

  @@map("tenants")
}
```

Feature 03's `Company` becomes the tenant's **profile** — legal name, address, registration numbers,
logo — with a `tenantId` and no longer a singleton. The distinction is worth keeping: `Tenant` is
how the platform knows about a customer; `Company` is what that customer puts on a payslip.

---

## Query enforcement

Three layers, in order of how much they are trusted:

1. **Prisma middleware / extension** injects `tenantId` into every `where` and every `create` for
   tenant-owned models, from request-scoped context. Convenient, and the layer most likely to be
   bypassed by a raw query.
2. **PostgreSQL RLS** on every tenant-owned table. `app.tenant_id` is set inside the transaction
   that serves the request. This is the layer that actually holds, because it catches raw SQL, the
   recursive CTEs in 02 and 04, and any query written by a future developer who did not read this
   document.
3. **Tests** that assert cross-tenant invisibility per feature, not just per model.

The recursive CTEs deserve a specific note: feature 02's transitive-reports query and feature 04's
window resolution both walk relationships. Under RLS they are naturally confined. Without RLS, a
manager chain that somehow crossed tenants would silently widen scope — which is the worst failure
this document exists to prevent.

---

## Provisioning a new tenant

1. Create the `Tenant` row.
2. Create its `Company` profile.
3. Seed its role set from the global permission catalogue — the four system roles, per tenant.
4. Create its first tenant super admin and send the invite (feature 01/05).
5. Offer the example configuration (shift, leave types, pay components) as **optional** starting
   points (D-T-07).
6. Walk the setup checklist from 03's settings home — **timezone first**, before any attendance
   exists.

Provisioning is a platform-operator action and is audited at the platform level.

---

## What this changes, per feature

| Feature | Work |
|---|---|
| 01 | `tenantId` on users, roles, sessions, audit. Tenant-scoped role sets. Platform operator vs tenant super admin (D-T-04). Auth.js session must carry the tenant |
| 02 | `tenantId` throughout; composite uniqueness; scope CTE confined by RLS; storage prefix |
| 03 | **`Company` de-singletonised** (was D-02); settings become per-tenant values; devices resolve the tenant; timezone moves to `Tenant` |
| 04 | Device push resolves tenant by serial (D-T-05); day opener and recompute run per tenant per timezone; shift/attendance uniqueness |
| 05 | Notifications carry the tenant; digests run per tenant timezone; templates per tenant |
| 06 | Ledger, policies and accrual per tenant; accrual periods per tenant leave year |
| 07 | Pay components, periods, runs and payslips per tenant; period locks per tenant |
| 08 | Portal inherits the session's tenant; nothing else changes — it computes nothing (08 D-01) |
| 09 | Cycles, templates and scales per tenant |
| 10 | Public form resolves tenant by posting slug; candidates are tenant-owned |
| 11 | Tenant scoping as step zero; suppression evaluated within a tenant; no cross-tenant reports |

Feature 08 being nearly unaffected is a small vindication of its "no new domain logic" constraint.

---

## Open questions raised by this document

| ID | Question | Proposed |
|---|---|---|
| **OQ-T-01** | Can one person work for two tenants with a single login? | No — two accounts (D-T-03) |
| **OQ-T-02** | How are tenants routed — subdomain (`acme.hrm.example`), path (`/t/acme`), or purely from the session? | From the session; `slug` is reserved so subdomains remain possible |
| **OQ-T-03** | How many tenants are expected? Under ~5 would justify revisiting database-per-tenant (D-T-01) | Assume tens |
| **OQ-T-04** | Can a platform operator access tenant data for support, and under what control? | Only by tenant grant, time-boxed and audited (D-T-04) |
| **OQ-T-05** | Is tenant data export or deletion required — offboarding a customer? | Export yes, deletion on request. Interacts with "no automatic deletion" (OQ-1002) |
| **OQ-T-06** | Does the suppression threshold (11, OQ-1103) still make sense when tenants vary from 12 to 500 employees? | Make it per-tenant configurable |
| **OQ-318** | With the serial number now acting as tenant selector *and* sole device identifier, how is the push endpoint protected? | Per-tenant network isolation looks stronger than a shared port |
