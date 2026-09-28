# 02 — Employee Management — API Design

Conventions, error shape, pagination, and status codes are as defined in
[01 api-design.md](../01-roles-permissions-auth/api-design.md). Every endpoint below is wrapped in
`protectedRoute(permission, handler)`, composes `employeeScopeFilter` into its query, and writes its
audit entry inside the same transaction as the change.

## The two helpers this feature adds

Everything else in the system will use these, so they matter more than the endpoints.

```ts
// src/lib/employees/scope.ts
resolveScopedEmployeeIds(ctx, scope): Promise<number[] | "ALL">
// The recursive CTE from data-model.md, cached per request (NFR-02).
// ALL        -> "ALL" (caller adds no filter)
// DEPARTMENT -> transitive reports + self  (+ department subtree if OQ-201 says so)
// SELF       -> [ctx.user.employeeId], or [] if the account has no employee link
// This is what feature 01's employeeScopeFilter delegates to once this feature exists.
```

```ts
// src/lib/employees/select.ts
selectEmployeeFields(ctx): Prisma.EmployeeSelect
// Builds the select clause from the actor's permissions. Without employee.read_sensitive the
// sensitive columns are never fetched — so they cannot be leaked by a forgotten serializer,
// an error dump, or a debug log (FR-E-07, NFR-04).
```

The rule for every list endpoint in this feature and in features 04–11: **scope is a filter applied
in the database query**, never a check applied to results after fetching. Fetching then filtering
leaks row counts through pagination totals.

## Employees

| Method | Path | Permission | Notes |
|---|---|---|---|
| GET | `/api/employees` | `employee.read` | List. Scoped. |
| POST | `/api/employees` | `employee.write` | Create. |
| GET | `/api/employees/:id` | `employee.read` | Scoped; 404 if out of scope (FR-Z-07). |
| PATCH | `/api/employees/:id` | `employee.write` | Field updates only — not department, position, manager, or status. |
| DELETE | `/api/employees/:id` | super admin | The 24-hour mistake window only (FR-E-06). |
| GET | `/api/employees/:id/history` | `employee.read` | Assignments and employment periods, newest first. |
| GET | `/api/employees/:id/reports` | `employee.read` | Direct reports. `?transitive=true` for the whole chain. |
| GET | `/api/employees/lookup` | `employee.read` | Lightweight `{id, code, name}` for pickers. Scoped. |

### `GET /api/employees`

Query: `q` (name, code, or work email), `departmentId` (`&includeSubDepartments=true`),
`positionId`, `managerId`, `status` (repeatable), `employmentType`, `hiredFrom`, `hiredTo`,
`page`, `pageSize`, `sort` (`name`, `-hireDate`, `code`).

```json
{ "data": [ { "id": 12, "employeeCode": "1042", "firstName": "Ayesha", "lastName": "Khan",
              "workEmail": "ayesha@company.com", "status": "ACTIVE",
              "department": { "id": 3, "name": "Finance" },
              "position": { "id": 8, "title": "Accountant" },
              "manager": { "id": 5, "name": "Ravi Menon" },
              "hireDate": "2024-02-01", "photoUrl": "/api/employees/12/photo" } ],
  "page": 1, "pageSize": 25, "total": 187 }
```

`status` defaults to the five employed statuses; terminated employees appear only with
`?status=TERMINATED` or `?includeTerminated=true`. A list that silently includes leavers is the kind
of thing that gets noticed at payroll time.

### `POST /api/employees`

```json
{ "employeeCode": "1043", "firstName": "Sara", "lastName": "Ahmed",
  "hireDate": "2026-10-01", "departmentId": 3, "positionId": 8, "managerId": 5,
  "employmentType": "FULL_TIME", "fte": 1.0, "workEmail": "sara@company.com",
  "status": "PROBATION" }
```

**201**. In one transaction: the employee row, the opening `EmploymentPeriod`, the opening
`EmployeeAssignment`, and the audit entry. Only code, first name, and last name are required
(FR-E-01).

**409** `CODE_TAKEN` — including when the code belongs to a *terminated* employee, since codes are
never reused (D-08). The message says so explicitly, because the obvious next thought is "then why
can't I use it".

**422** `INVALID_CODE` if the code fails the device-PIN rules (OQ-204).

### `PATCH /api/employees/:id`

Accepts identity, contact, and personal fields. Rejects `employeeCode` (**422** `CODE_IMMUTABLE`),
and rejects `departmentId`, `positionId`, `managerId`, `employmentType`, `fte`, and `status` with
**422** `USE_ASSIGNMENT_ENDPOINT` / `USE_LIFECYCLE_ENDPOINT`. Those changes carry an effective date
and history, and letting them through a generic PATCH is exactly how the history table ends up
lying.

Sensitive fields in the body without `employee.read_sensitive` → **403**.

## Assignments, lifecycle, and the org

| Method | Path | Permission | Notes |
|---|---|---|---|
| POST | `/api/employees/:id/assignments` | `employee.write` | Transfer / promote / change manager. |
| PATCH | `/api/employees/:id/assignments/:aid` | `employee.edit_history` | Correct or backdate. |
| POST | `/api/employees/:id/confirm` | `employee.lifecycle` | End probation. |
| POST | `/api/employees/:id/terminate` | `employee.lifecycle` | |
| POST | `/api/employees/:id/rehire` | `employee.lifecycle` | Opens a new employment period. |
| POST | `/api/employees/:id/status` | `employee.lifecycle` | `ON_LEAVE`, `SUSPENDED`, back to `ACTIVE`. |

### `POST /api/employees/:id/assignments`

```json
{ "effectiveFrom": "2026-10-01", "departmentId": 4, "positionId": 11,
  "managerId": 7, "reason": "Promotion to Senior Accountant" }
```

Omitted fields carry forward from the current assignment. In one transaction: close the open
assignment at `effectiveFrom - 1 day`, insert the new one, and — only if `effectiveFrom <= today` —
update the denormalised columns on `Employee`. A future date leaves the profile untouched until the
nightly job applies it (FR-H-03).

**422** `MANAGER_CYCLE` if the new manager reports to this employee, directly or transitively.
**422** `SELF_MANAGER`. **422** `OVERLAPPING_ASSIGNMENT` if the date falls inside a closed period —
that needs the history endpoint, not this one. **422** `BEFORE_HIRE_DATE`.

Audit action is specific: `employee.transferred`, `employee.promoted`, or `employee.manager_changed`
depending on what actually changed, because "employee.updated" is useless when someone asks why
approvals started going to a different person in April.

### `POST /api/employees/:id/terminate`

```json
{ "lastWorkingDay": "2026-11-30", "reason": "RESIGNATION", "note": "Moving abroad",
  "noticeDays": 30, "eligibleForRehire": true, "reassignReportsTo": 7 }
```

**200**. Closes the employment period and the open assignment, sets status `TERMINATED` and
`terminatedAt`. If the employee has reports, `reassignReportsTo` is **required** unless
`confirmOrphanReports: true` is passed (FR-O-08) — otherwise **422** `HAS_REPORTS`, listing them.

The response includes `"linkedUserId": 44` when a login account exists, so the UI can offer to
suspend it as a separate, deliberate step (FR-H-08). This endpoint never suspends it silently.

A future-dated termination is allowed and takes effect on its date; the employee stays `NOTICE`
until then.

### `POST /api/employees/:id/rehire`

```json
{ "startDate": "2027-03-01", "departmentId": 3, "positionId": 8, "managerId": 5 }
```

**200** — new `EmploymentPeriod`, new opening assignment, status back to `PROBATION` or `ACTIVE`.
**409** `NOT_ELIGIBLE_FOR_REHIRE` if the last period says so — overridable by a super admin with
`"override": true`, which is audited distinctly.

## Departments

| Method | Path | Permission |
|---|---|---|
| GET | `/api/departments` | `department.read` |
| GET | `/api/departments/tree` | `department.read` |
| POST | `/api/departments` | `department.write` |
| GET | `/api/departments/:id` | `department.read` |
| PATCH | `/api/departments/:id` | `department.write` |
| DELETE | `/api/departments/:id` | `department.write` |

`GET /api/departments/tree` returns the nested structure with an employee count per node
(`directCount` and `totalCount` including descendants) — one query, not one per node.

`PATCH` with a new `parentId` validates the tree: **422** `DEPARTMENT_CYCLE`, **422** `MAX_DEPTH`
(FR-O-02, FR-O-03). `DELETE` returns **409** `DEPARTMENT_IN_USE` with the employee and child counts,
and points at deactivation instead (FR-O-04).

## Positions

`GET`, `POST`, `GET/:id`, `PATCH`, `DELETE` at `/api/positions` under `position.read` / `.write`,
mirroring departments. `GET` filters by `departmentId`, `isActive`, and `q`. Delete is refused with
**409** `POSITION_IN_USE` while any employee or historical assignment references it.

## Org chart

### `GET /api/org-chart` — `department.read`

Query: `rootEmployeeId` (default: the employees with no manager), `depth` (default 2, max 4).

```json
{ "nodes": [ { "id": 5, "name": "Ravi Menon", "positionTitle": "Finance Manager",
               "departmentName": "Finance", "photoUrl": "/api/employees/5/photo",
               "directReportCount": 4, "managerId": 1 } ],
  "truncatedAt": [12] }
```

Flat node list with `managerId`, not a nested tree: the client already needs an index by id for
selection and highlighting, and a nested payload forces it to walk the tree to build one.
`truncatedAt` names the nodes that have children beyond the requested depth, so the UI knows where to
show an expander (NFR-03).

The org chart is **not** scope-filtered — it returns names, positions, and reporting lines
company-wide, deliberately (see `requirements.md`, permission keys). It contains no contact details
and no sensitive fields.

## Documents

| Method | Path | Permission | Notes |
|---|---|---|---|
| GET | `/api/employees/:id/documents` | `employee.document.read` | Metadata only. Excludes soft-deleted. |
| POST | `/api/employees/:id/documents` | `employee.document.write` | `multipart/form-data`. |
| GET | `/api/employees/:id/documents/:docId` | `employee.document.read` | Streams the file. |
| DELETE | `/api/employees/:id/documents/:docId` | `employee.document.write` | Soft delete. |
| GET | `/api/employees/:id/photo` | `employee.read` | The photo, or a 404 the UI renders as initials. |
| GET | `/api/documents/expiring` | `employee.document.read` | Across all in-scope employees, 60 days ahead. |

Upload fields: `file`, `type`, `title`, `issuedOn?`, `expiresOn?`, `note?`.

The upload handler: enforces the 10 MB cap **while streaming**, not after buffering; sniffs the
content type from the magic bytes and rejects a mismatch with the extension (**422**
`UNSUPPORTED_FILE_TYPE`); generates a UUID storage key; computes the SHA-256; writes the file, then
the row. If the row write fails, the orphaned file is removed — and if that cleanup fails, the
orphan is logged for the nightly sweeper rather than left silently.

Download re-checks permission and scope on every request, sets
`Content-Disposition: attachment; filename="<original>"` and `X-Content-Type-Options: nosniff`, and
writes an audit entry (FR-D-04). Files are never exposed through a static path — there is no
`/uploads/*` route, by design.

The photo endpoint is the one exception to `employee.document.read`: photos appear in lists and the
org chart, so they follow `employee.read`. Worth stating explicitly, since it is the only document
type a manager can see without the document permission.

## Import and export

| Method | Path | Permission |
|---|---|---|
| GET | `/api/employees/import/template` | `employee.import` |
| POST | `/api/employees/import/validate` | `employee.import` |
| POST | `/api/employees/import` | `employee.import` |
| GET | `/api/employees/export` | `employee.export` |

### `POST /api/employees/import/validate`

`multipart/form-data` with the CSV. Writes nothing. Returns the dry run:

```json
{ "totalRows": 240, "willCreate": 231, "willUpdate": 6,
  "errors": [ { "row": 14, "column": "employeeCode", "value": "10 42",
                "message": "Employee code must be numeric." },
              { "row": 88, "column": "managerCode", "value": "9999",
                "message": "No employee with this code, and it is not elsewhere in the file." } ],
  "unknownDepartments": ["Logistics"], "unknownPositions": [] }
```

### `POST /api/employees/import`

Same payload plus `{ "createMissingDepartments": true, "importValidRowsOnly": false }`.
Re-validates server-side (never trusts the earlier dry run), then runs the whole file in one
transaction: pass one creates and updates employees, pass two resolves managers (FR-I-05). Returns
the same shape plus `created`, `updated`, `skipped`.

With `importValidRowsOnly: false` — the default — a single error writes nothing (FR-I-02).

One audit entry for the run: filename, checksum, row counts, options, operator (FR-I-06). Not one
entry per row; 240 entries would bury the day's real activity.

### `GET /api/employees/export`

Same filters as the list endpoint. Returns CSV, scope-filtered, with sensitive columns present only
if the actor holds `employee.read_sensitive` (FR-I-07). Audited, including the filter used and the
row count — an export is a bulk disclosure and deserves a record.

## Open questions

| ID | Question |
|---|---|
| OQ-201 | Whether `resolveScopedEmployeeIds` unions the department subtree. It is one function; the decision is cheap to implement and expensive to get wrong. |
| OQ-210 | Should `GET /api/employees` support a `fields` parameter for partial responses? Useful for pickers, but it complicates the permission-driven select. Defaulting to a dedicated `/lookup` endpoint instead. |
| OQ-211 | Async import for large files. At 1 000 rows a synchronous request is fine (NFR-06); at 10 000 it needs a job and a progress endpoint. Deferred until someone has that many. |
| OQ-212 | Whether `GET /api/documents/expiring` belongs here or in feature 05, which will send the reminders. Proposed: the query lives here, the scheduling and delivery in 05. |
