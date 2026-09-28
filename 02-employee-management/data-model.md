# 02 — Employee Management — Data Model

Same conventions as feature 01: `camelCase` fields mapped to `snake_case`, plural `@@map` table
names, `Int` autoincrement keys, UTC timestamps. Calendar dates use `@db.Date` (NFR-07).

## Entity overview

```
Department ──self──┐
     │ head        │ parent
     ▼             │
  Employee ──self──┘ manager
     ├──*── EmployeeAssignment  (department / position / manager / type, dated)
     ├──*── EmploymentPeriod    (hire → termination, one per spell of service)
     ├──*── EmployeeDocument
     ├──*── EmergencyContact
     ├──0..1─ User              (feature 01)
     └──*── Attendance          (existing)
  Position ──*── EmployeeAssignment
```

The employee's `departmentId`, `positionId`, and `managerId` are **denormalised current values**.
The truth is `EmployeeAssignment` (D-04). Anything asking a question with a date in it must read the
history.

## Changes to the existing `Employee` model

The scaffold's model has five fields this feature reshapes. The migration is the riskiest part of
the build, so it is specified in full at the end of this document.

| Existing | Becomes |
|---|---|
| `department String?` | `departmentId Int?` → `Department` |
| `position String?` | `positionId Int?` → `Position` |
| `status EmploymentStatus` (ACTIVE / ON_LEAVE / TERMINATED) | extended enum (FR-E-04) |
| `hireDate DateTime? @db.Date` | kept; also the start of the first `EmploymentPeriod` |
| `employeeCode`, `firstName`, `lastName`, `email` | kept; `email` is renamed `workEmail` for clarity |

## Enums

```prisma
enum EmploymentStatus {
  PROBATION
  ACTIVE
  ON_LEAVE      // long leave: maternity, sabbatical. Not a day off.
  SUSPENDED     // disciplinary; still employed
  NOTICE        // resigned or given notice; still working
  TERMINATED
}

enum EmploymentType {
  FULL_TIME
  PART_TIME
  CONTRACT
  INTERN
  CONSULTANT
}

enum TerminationReason {
  RESIGNATION
  DISMISSAL
  REDUNDANCY
  END_OF_CONTRACT
  RETIREMENT
  DEATH
  OTHER
}

enum Gender {
  MALE
  FEMALE
  OTHER
  UNDISCLOSED
}

enum DocumentType {
  CONTRACT
  ID_CARD
  PASSPORT
  VISA_PERMIT
  CERTIFICATE
  CV
  PHOTO
  OFFER_LETTER
  TERMINATION_LETTER
  OTHER
}
```

`ON_LEAVE` and `SUSPENDED` both mean "employed but not expected at work", and features 04 and 06
must treat them as such. That distinction is why the enum is not a boolean.

## Models

> **Revised 2026-09-18.** Multi-tenant (OQ-301): every model below gains `tenantId`, and the unique
> constraints marked in [MULTI-TENANCY.md](../MULTI-TENANCY.md) D-T-09 become composite with it.
> `employeeCode`'s format is now confirmed from the manual (OQ-204).

```prisma
model Employee {
  id             Int              @id @default(autoincrement())
  tenantId       Int              @map("tenant_id")
  // The user ID enrolled on the SenseFace terminal: 1–14 alphanumeric characters, confirmed
  // from the manual (OQ-204). Immutable after creation (FR-E-02). Unique per tenant.
  employeeCode   String           @map("employee_code")

  firstName      String           @map("first_name")
  lastName       String           @map("last_name")
  middleName     String?          @map("middle_name")
  preferredName  String?          @map("preferred_name")

  workEmail      String?          @unique @map("work_email")
  workPhone      String?          @map("work_phone")

  // --- sensitive: requires employee.read_sensitive (FR-E-07) ---
  personalEmail  String?          @map("personal_email")
  personalPhone  String?          @map("personal_phone")
  dateOfBirth    DateTime?        @map("date_of_birth") @db.Date
  gender         Gender?
  nationalId     String?          @map("national_id")
  addressLine1   String?          @map("address_line1")
  addressLine2   String?          @map("address_line2")
  city           String?
  postalCode     String?          @map("postal_code")
  country        String?
  // ------------------------------------------------------------

  status         EmploymentStatus @default(PROBATION)
  hireDate       DateTime?        @map("hire_date") @db.Date
  confirmedAt    DateTime?        @map("confirmed_at") @db.Date  // end of probation
  terminatedAt   DateTime?        @map("terminated_at") @db.Date // last working day

  // Denormalised current assignment — derived from EmployeeAssignment (D-04).
  departmentId   Int?             @map("department_id")
  positionId     Int?             @map("position_id")
  managerId      Int?             @map("manager_id")
  // Added 2026-09-18 (OQ-206). Owned by feature 03; drives holiday-calendar resolution and
  // gives reports a location dimension. Dated in EmployeeAssignment like every other
  // structural fact, so "which site was she at in March" stays answerable (D-04).
  locationId     Int?             @map("location_id")
  employmentType EmploymentType?  @map("employment_type")
  fte            Decimal?         @db.Decimal(4, 3)   // 1.000 = full time

  photoDocumentId Int?            @unique @map("photo_document_id")

  createdById    Int?             @map("created_by_id")
  createdAt      DateTime         @default(now()) @map("created_at")
  updatedAt      DateTime         @updatedAt @map("updated_at")

  department     Department?      @relation("EmployeeDepartment", fields: [departmentId], references: [id])
  position       Position?        @relation(fields: [positionId], references: [id])
  manager        Employee?        @relation("ReportsTo", fields: [managerId], references: [id])
  reports        Employee[]       @relation("ReportsTo")
  photo          EmployeeDocument? @relation("EmployeePhoto", fields: [photoDocumentId], references: [id])

  headOf         Department[]     @relation("DepartmentHead")
  assignments    EmployeeAssignment[]
  periods        EmploymentPeriod[]
  documents      EmployeeDocument[] @relation("EmployeeDocuments")
  emergencyContacts EmergencyContact[]

  user           User?            // feature 01
  attendance     Attendance[]     // existing

  @@unique([tenantId, employeeCode])          // per tenant (MULTI-TENANCY.md D-T-09)
  @@index([tenantId, status])
  @@index([tenantId, departmentId])
  @@index([tenantId, managerId])
  @@index([tenantId, lastName, firstName])
  @@map("employees")
}

model Department {
  id          Int          @id @default(autoincrement())
  name        String       @unique
  code        String?      @unique          // short code for reports and CSV import
  parentId    Int?         @map("parent_id")
  headId      Int?         @map("head_id")
  description String?
  isActive    Boolean      @default(true) @map("is_active")
  createdAt   DateTime     @default(now()) @map("created_at")
  updatedAt   DateTime     @updatedAt @map("updated_at")

  parent      Department?  @relation("DepartmentTree", fields: [parentId], references: [id])
  children    Department[] @relation("DepartmentTree")
  head        Employee?    @relation("DepartmentHead", fields: [headId], references: [id])

  employees   Employee[]   @relation("EmployeeDepartment")
  positions   Position[]
  assignments EmployeeAssignment[]

  @@index([parentId])
  @@map("departments")
}

model Position {
  id           Int          @id @default(autoincrement())
  title        String
  code         String?      @unique
  departmentId Int?         @map("department_id")
  grade        String?                        // "L3", "Senior" — free text; payroll may key off it
  description  String?
  isActive     Boolean      @default(true) @map("is_active")
  createdAt    DateTime     @default(now()) @map("created_at")
  updatedAt    DateTime     @updatedAt @map("updated_at")

  department   Department?  @relation(fields: [departmentId], references: [id])
  employees    Employee[]
  assignments  EmployeeAssignment[]

  @@unique([title, departmentId])
  @@map("positions")
}

model EmployeeAssignment {
  id             Int             @id @default(autoincrement())
  employeeId     Int             @map("employee_id")

  departmentId   Int?            @map("department_id")
  positionId     Int?            @map("position_id")
  managerId      Int?            @map("manager_id")
  locationId     Int?            @map("location_id")     // OQ-206 — dated like the rest
  employmentType EmploymentType? @map("employment_type")
  fte            Decimal?        @db.Decimal(4, 3)

  effectiveFrom  DateTime        @map("effective_from") @db.Date
  effectiveTo    DateTime?       @map("effective_to") @db.Date   // null = current
  reason         String?                                          // "Promotion", "Restructure"
  isBackdated    Boolean         @default(false) @map("is_backdated")

  createdById    Int?            @map("created_by_id")
  createdAt      DateTime        @default(now()) @map("created_at")

  employee       Employee        @relation(fields: [employeeId], references: [id], onDelete: Cascade)
  department     Department?     @relation(fields: [departmentId], references: [id])
  position       Position?       @relation(fields: [positionId], references: [id])

  @@index([employeeId, effectiveFrom])
  @@index([departmentId, effectiveFrom])
  @@map("employee_assignments")
}

model EmploymentPeriod {
  id           Int                @id @default(autoincrement())
  employeeId   Int                @map("employee_id")
  startDate    DateTime           @map("start_date") @db.Date
  endDate      DateTime?          @map("end_date") @db.Date

  terminationReason   TerminationReason? @map("termination_reason")
  terminationNote     String?            @map("termination_note")
  noticeDays          Int?               @map("notice_days")
  eligibleForRehire   Boolean?           @map("eligible_for_rehire")

  createdById  Int?               @map("created_by_id")
  createdAt    DateTime           @default(now()) @map("created_at")

  employee     Employee           @relation(fields: [employeeId], references: [id], onDelete: Cascade)

  @@index([employeeId, startDate])
  @@map("employment_periods")
}

model EmployeeDocument {
  id               Int          @id @default(autoincrement())
  employeeId       Int          @map("employee_id")
  type             DocumentType
  title            String
  // UUID filename on the mounted volume — never the user's filename (FR-D-02).
  storageKey       String       @unique @map("storage_key")
  originalFilename String       @map("original_filename")
  mimeType         String       @map("mime_type")
  sizeBytes        Int          @map("size_bytes")
  checksumSha256   String       @map("checksum_sha256")

  issuedOn         DateTime?    @map("issued_on") @db.Date
  expiresOn        DateTime?    @map("expires_on") @db.Date
  note             String?

  uploadedById     Int?         @map("uploaded_by_id")
  uploadedAt       DateTime     @default(now()) @map("uploaded_at")
  deletedAt        DateTime?    @map("deleted_at")       // soft delete (FR-D-06)
  deletedById      Int?         @map("deleted_by_id")

  employee         Employee     @relation("EmployeeDocuments", fields: [employeeId], references: [id], onDelete: Cascade)
  photoOf          Employee?    @relation("EmployeePhoto")

  @@index([employeeId, type])
  @@index([expiresOn])
  @@map("employee_documents")
}

model EmergencyContact {
  id           Int      @id @default(autoincrement())
  employeeId   Int      @map("employee_id")
  name         String
  relationship String?
  phone        String
  altPhone     String?  @map("alt_phone")
  email        String?
  isPrimary    Boolean  @default(false) @map("is_primary")
  createdAt    DateTime @default(now()) @map("created_at")
  updatedAt    DateTime @updatedAt @map("updated_at")

  employee     Employee @relation(fields: [employeeId], references: [id], onDelete: Cascade)

  @@index([employeeId])
  @@map("emergency_contacts")
}
```

## Design notes

**Why the current assignment is duplicated on `Employee`.** Every list screen and every join in the
system wants "this person's department, now". Resolving that through a date-ranged history table on
every query is a join per row and an index that cannot be used well. The denormalised columns are
written by exactly one code path — the assignment service — and a nightly job reconciles them
against the history, so drift is detectable rather than silent.

**Why `EmploymentPeriod` is separate from `EmployeeAssignment`.** They answer different questions.
"Was this person employed on 3 March" is a period question, used by payroll and leave accrual.
"Which department were they in on 3 March" is an assignment question, used by reports and approval
routing. Conflating them makes a rehire look like a transfer.

**Why sensitive fields are columns on `Employee` and not a separate table.** A separate `EmployeePii`
table would enforce the boundary at the schema level, which is tempting. But it doubles the writes
for every employee edit and makes the import path considerably more complex, and the boundary still
has to be enforced in code anyway (a query can join anything). The chosen approach is a single
`selectEmployeeFields(ctx)` helper that builds the Prisma `select` from the actor's permissions, so
unauthorised fields are never fetched — see `api-design.md`. Revisit if a stricter boundary is
required, e.g. for a compliance audit.

**Why `fte` is `Decimal(4,3)` and not a float.** 0.8 FTE feeds payroll. Floats and money do not mix.

**Why `Position` is unique on `[title, departmentId]` rather than on title alone.** "Manager" exists
in every department and means something different in each.

**Why documents carry a checksum.** It detects a corrupted or swapped file on the volume, and it
makes deduplication possible later. It costs one hash at upload time.

## Transitive reports — the hot query

Used by `employeeScopeFilter` (feature 01) on nearly every request in the system (NFR-02):

```sql
WITH RECURSIVE chain AS (
  SELECT id FROM employees WHERE id = $1
  UNION ALL
  SELECT e.id FROM employees e JOIN chain c ON e.manager_id = c.id
)
SELECT id FROM chain;
```

Depth is bounded by FR-O-07 (max 10), and cycles are prevented at write time by FR-O-06 — but the
query should still carry a depth guard, because a cycle introduced by a direct database edit would
otherwise hang every request in the system rather than one.

If OQ-201 is answered "manager chain **plus** department headship", the filter becomes the union of
the above and:

```sql
WITH RECURSIVE subtree AS (
  SELECT id FROM departments WHERE head_id = $1
  UNION ALL
  SELECT d.id FROM departments d JOIN subtree s ON d.parent_id = s.id
)
SELECT id FROM employees WHERE department_id IN (SELECT id FROM subtree);
```

## Seed data

Idempotent, upserted by natural key:

1. **Permissions** — the 13 keys from `requirements.md`, added to feature 01's catalogue.
2. **Role grants** — `employee.read`/`department.read`/`position.read` to HR (ALL), Manager
   (DEPARTMENT), Employee (SELF); the write and sensitive keys to HR only; `employee.edit_history`
   to nobody (super admin holds it implicitly).
3. **Document types** — the enum needs no seed, but a `document_type_config` of display names and
   which types expect an expiry date is worth seeding once OQ-208 is settled.
4. **No sample departments or employees.** A demo seed is a separate, explicitly-run script; it must
   never run against a real database.

## Migration notes

One migration, `employee_management`, in this order. Steps 3–5 are data migration and must be in the
same transaction as the schema change so a failure leaves nothing half-done.

1. Create `departments`, `positions`, `employee_assignments`, `employment_periods`,
   `employee_documents`, `emergency_contacts`.
2. Add the new columns to `employees` (nullable), and extend `EmploymentStatus` with `PROBATION`,
   `SUSPENDED`, `NOTICE`. Postgres allows adding enum values but **not** removing them — the three
   existing values stay, which is what we want.
3. **Backfill departments and positions from the free-text columns:** `INSERT INTO departments
   (name) SELECT DISTINCT trim(department) FROM employees WHERE department IS NOT NULL AND
   trim(department) <> ''`, same for positions. Names differing only in case or whitespace collapse
   to one row — this is the point of the migration, but it means the operator should review the
   result (acceptance criterion 12).
4. Set `employees.department_id` / `position_id` by matching on the trimmed, case-insensitive name.
5. Create one `EmploymentPeriod` per employee from `hire_date` (null hire dates produce a period
   with a null start, flagged for HR to complete), and one open `EmployeeAssignment` per employee
   from the new FK values with `effective_from = coalesce(hire_date, created_at::date)`.
6. Drop `employees.department` and `employees.position`. **Take a backup first** — this is the only
   irreversible step, and step 3's collapsing means the original strings cannot be reconstructed.
7. Rename `employees.email` → `work_email`.

Rollback: steps 1–5 are additive and reversible. After step 6 the only route back is the backup.
Consider shipping steps 1–5 and 6–7 as two migrations one release apart if this database already
holds real data. It currently holds only device-test rows, so a single migration is acceptable now —
but that is true only until the system goes live.

## Open questions

| ID | Question |
|---|---|
| OQ-201 | Manager chain vs department subtree for `DEPARTMENT` scope — the recursive query above changes shape depending on the answer. |
| OQ-202 | Which of the sensitive columns are actually needed. Unused PII columns are pure liability; deleting a column later is easy, deleting the data that accumulated in it is not. |
| OQ-203 | Retention for terminated employees. If a deletion obligation exists, an anonymisation path is needed: keep the attendance and payroll rows, blank the identity. |
| OQ-204 | `employeeCode` format constraints from the SenseFace terminal (blocked with OQ-000). |
| OQ-206 | Work locations / branches. If added later, `locationId` goes on both `Employee` and `EmployeeAssignment`, which means revisiting the history table — cheaper to decide now. |
| OQ-209 | Where the document volume lives in `docker-compose.yml`, and whether the backup story covers it (NFR-05). Needs a decision before the first real upload. |
