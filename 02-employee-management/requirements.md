# 02 — Employee Management — Requirements

## Actors

| Actor | What they do here |
|---|---|
| HR admin | Creates and maintains every employee record, the department tree, positions, and documents. Runs the initial import. |
| Manager | Reads their own team's records — contact details, job, reporting line. No sensitive personal data, no records outside their scope. |
| Employee | Reads their own record and requests corrections. The employee-facing UI is feature 08; the permission (`SELF` scope) is defined here. |
| Super admin | Everything HR admin does, plus the structural changes HR is not trusted with (deleting a department, editing history). |

## User stories

### Employee records

- **US-01** — As HR, I add a new employee with their identity, contact details, job, manager, and
  start date, so they exist in the system before their first day.
- **US-02** — As HR, I search and filter employees by name, code, department, position, status, and
  manager, so I can find a person in a list of hundreds.
- **US-03** — As HR, I open an employee's profile and see everything about them in one place, with
  the sensitive parts visible only if I am allowed to see them.
- **US-04** — As HR, I edit an employee's details, and the change is audited with before and after
  values.
- **US-05** — As a manager, I see my team members' profiles — contact details, position, reporting
  line — without seeing their national ID or home address.
- **US-06** — As HR, I upload an employee's contract and ID copies, and see at a glance which
  documents are missing or expiring.
- **US-07** — As HR, I record emergency contacts, so someone can be reached if there is an incident.

### Org structure

- **US-08** — As HR, I create departments, nest them under a parent, and name a department head, so
  the structure matches the company.
- **US-09** — As HR, I create positions (job titles), optionally tied to a department.
- **US-10** — As HR, I set each employee's manager, so approvals route to the right person and
  managers see the right team.
- **US-11** — As anyone with access, I view an org chart and navigate up and down the reporting
  lines, so I can understand the structure without asking.
- **US-12** — As HR, I move a whole department under a different parent, or rename it, without
  touching the employees in it.

### Lifecycle

- **US-13** — As HR, I transfer an employee to another department, promote them to another position,
  or change their manager, with an effective date — and last year's org structure still reads
  correctly afterwards.
- **US-14** — As HR, I confirm an employee at the end of probation.
- **US-15** — As HR, I terminate an employee with a last working day and a reason, so their access,
  attendance expectations, and payroll all stop at the right date.
- **US-16** — As HR, I rehire a former employee onto their existing record, so their history is
  continuous.
- **US-17** — As HR, I see an employee's full history — every department, position, and manager they
  have had, with dates.

### Getting started

- **US-18** — As HR, I import our existing workforce from a CSV, see exactly which rows failed and
  why, and fix and re-run without creating duplicates.
- **US-19** — As HR, I export the employee list to CSV for use outside the system.

## Functional requirements

### Employee record — FR-E

| ID | Requirement |
|---|---|
| FR-E-01 | An employee has: `employeeCode` (unique, never reused), first name, last name, and a status. Everything else is optional at creation, so HR can get a person into the system quickly and complete the record later. |
| FR-E-02 | `employeeCode` must be valid as a user ID on the SenseFace terminal. **Confirmed from the manual 2026-09-18 (OQ-204): 1–14 characters, alphanumeric** — *not* numeric up to 9 digits, which the plan previously assumed. It is set at creation and is **immutable** thereafter: attendance rows reference it as raw text, and the device itself refuses to change an ID after registration. Unique **per tenant** (MULTI-TENANCY.md D-T-09). |
| FR-E-03 | Work email is unique across employees where present. Personal email is not required to be unique (family members may share one). |
| FR-E-04 | Statuses: `PROBATION`, `ACTIVE`, `ON_LEAVE`, `SUSPENDED`, `NOTICE`, `TERMINATED`. Only the first five are "currently employed" for the purposes of attendance, leave, and payroll. |
| FR-E-05 | Status is **derived from the lifecycle, not typed in freehand**: it changes through the confirm / suspend / terminate / rehire actions, each of which records a date and a reason. |
| FR-E-06 | Employees are never hard-deleted (D-08). A record created by mistake may be deleted only by a super admin, only within 24 hours of creation, and only if no attendance, leave, or payroll rows reference it. |
| FR-E-07 | Sensitive fields — national ID, date of birth, home address, personal phone, bank details, marital status — are readable only with `employee.read_sensitive`, and are omitted from API responses (not nulled and not masked client-side) when the actor lacks it. |
| FR-E-08 | Each employee may have any number of emergency contacts, with a name, relationship, and at least one phone number. |
| FR-E-09 | A photograph may be uploaded; it is treated as a document (FR-D) and displayed on the profile and org chart. |
| FR-E-10 | Changing an employee's work email, code, or status is audited individually with a named action, not lumped under a generic "updated". |

### Org structure — FR-O

| ID | Requirement |
|---|---|
| FR-O-01 | A department has a unique name, an optional parent department, an optional head (an employee), and an active flag. |
| FR-O-02 | The department tree may not contain a cycle. Re-parenting validates this and is rejected with a clear message. |
| FR-O-03 | Tree depth is capped (default 6 levels) to keep the org chart and scope queries bounded. |
| FR-O-04 | A department with employees or sub-departments cannot be deleted; it can be **deactivated**, which hides it from selection lists but leaves history intact. |
| FR-O-05 | A position has a title, an optional department, an optional grade/level, and an active flag. Positions are not unique per employee — many people may hold the same position. |
| FR-O-06 | An employee's `managerId` points to another employee. Self-management is rejected, and so is any change that creates a cycle in the reporting chain. |
| FR-O-07 | Reporting-chain depth is capped (default 10) so that transitive-reports resolution terminates and cannot be made pathological by a bad edit. |
| FR-O-08 | Terminating a manager who still has reports requires reassigning those reports first, or explicitly confirming that they will be left without a manager. Leaving reports dangling silently would break approval routing in feature 06. |
| FR-O-09 | The org chart renders from the reporting chain, showing each node's name, photo, position, and direct-report count, and can be rooted at any employee. |

### Employment history — FR-H

| ID | Requirement |
|---|---|
| FR-H-01 | Every change to department, position, manager, employment type, or FTE writes an `EmployeeAssignment` row with an effective date. The previous row is closed on the day before. |
| FR-H-02 | Assignment periods for one employee do not overlap and do not gap while employed. |
| FR-H-03 | A future-dated change is allowed: it is stored and takes effect on its date. A nightly job applies pending changes to the employee's denormalised current values. |
| FR-H-04 | Every module that needs "which department/manager was this on date X" reads the assignment history, never the denormalised current fields. |
| FR-H-05 | Backdating a change is allowed with `employee.edit_history`; it is audited distinctly, because it silently changes answers other modules already gave. |
| FR-H-06 | Employment periods (hire → termination, and again on rehire) are recorded separately from assignments, so tenure and leave accrual can be computed across breaks in service (OQ-205). |
| FR-H-07 | Termination records last working day, reason, reason category (resignation, dismissal, redundancy, end of contract, retirement), notice period, and whether the employee is eligible for rehire. |
| FR-H-08 | On termination, the linked user account (feature 01) is **not** suspended automatically — it is offered as a prompted next step, because access sometimes must outlast the last working day, and sometimes must end before it. |

### Documents — FR-D

| ID | Requirement |
|---|---|
| FR-D-01 | A document has a type (from a seeded list), a file, an optional issue and expiry date, an uploader, and an upload timestamp. |
| FR-D-02 | Files are stored on a mounted volume, named by a generated UUID, never by the original filename. Metadata, including the original filename, lives in Postgres (D-07). |
| FR-D-03 | Allowed types: PDF, JPEG, PNG, and common Office formats. The type is validated by content sniffing, not by the file extension alone. Maximum 10 MB per file. |
| FR-D-04 | Documents are never served as static files. Every download goes through an endpoint that re-checks permission and scope, and is audited. |
| FR-D-05 | Documents are readable only with `employee.document.read`, which is narrower than `employee.read` — a manager may see their team without seeing their contracts. |
| FR-D-06 | Deleting a document is a soft delete; the file is retained and the entry is hidden, because "HR deleted the contract" is exactly the kind of event an audit log exists for. |
| FR-D-07 | Documents with an expiry date surface on the profile and in a "expiring soon" list, 60 days ahead. Feature 05 turns that into a notification. |
| FR-D-08 | Uploads are scanned for executable content and rejected; the upload path never makes the volume web-executable. |

### Import and export — FR-I

| ID | Requirement |
|---|---|
| FR-I-01 | CSV import accepts a documented column set, with a downloadable template. |
| FR-I-02 | Import is validated in full **before** anything is written, and reports every failing row with its row number and reason. Nothing is written if the file has errors, unless the operator explicitly chooses "import valid rows only". |
| FR-I-03 | Import matches existing employees by `employeeCode` and updates rather than duplicating. A dry-run mode shows what would be created versus updated. |
| FR-I-04 | Departments, positions, and managers referenced by name in the CSV are resolved to existing records; unknown values are reported, and optionally auto-created if the operator ticks that box. |
| FR-I-05 | Managers referenced by `employeeCode` may appear later in the file than their reports; resolution is a second pass. |
| FR-I-06 | The whole import runs in one transaction and is audited as a single entry recording the file name, row counts, and the operator. |
| FR-I-07 | Export respects the actor's scope and sensitive-field permission: a manager exporting gets their team, without sensitive columns. |

## Non-functional requirements

| ID | Requirement |
|---|---|
| NFR-01 | The employee list must stay responsive at 5 000 employees: server-side pagination, filtering, and sorting; no client-side "load everything then filter". |
| NFR-02 | Transitive-reports resolution (used by every `DEPARTMENT`-scoped query in every module) must be a single recursive CTE, cached per request. It is on the hot path of the entire system. |
| NFR-03 | The org chart loads one level at a time, expanding on demand. Rendering 5 000 nodes at once is not a requirement. |
| NFR-04 | Sensitive fields must not appear in server logs, error messages, or exports the actor is not entitled to. |
| NFR-05 | Document storage must survive container restarts — the volume is mounted, not written into the image layer. Backups must cover it; a database-only backup silently loses every contract. |
| NFR-06 | Import of 1 000 rows completes in under 60 seconds. |
| NFR-07 | Dates that are dates, not moments — hire date, last working day, date of birth — are stored as `@db.Date` and are not shifted by timezone conversion. A birthday that moves by a day when the server timezone changes is a defect. |

## Permission keys added by this feature

| Key | Meaning | Default scopes (see 01's role grid) |
|---|---|---|
| `employee.read` | View employee records, excluding sensitive fields | HR: ALL · Manager: DEPARTMENT · Employee: SELF |
| `employee.write` | Create and edit employee records | HR: ALL |
| `employee.read_sensitive` | View national ID, DOB, home address, personal contact | HR: ALL |
| `employee.lifecycle` | Confirm, suspend, terminate, rehire | HR: ALL |
| `employee.edit_history` | Backdate or correct assignment history | Super admin only |
| `employee.document.read` | View and download documents | HR: ALL · Employee: SELF |
| `employee.document.write` | Upload and remove documents | HR: ALL |
| `employee.import` | Run CSV import | HR: ALL |
| `employee.export` | Export employee data | HR: ALL |
| `department.read` | View departments and the org chart | HR: ALL · Manager: ALL · Employee: ALL |
| `department.write` | Create, edit, deactivate departments |  HR: ALL |
| `position.read` | View positions | as `department.read` |
| `position.write` | Create, edit, deactivate positions | HR: ALL |

`department.read` is granted at `ALL` even to employees: an org chart that shows only your own team
is not an org chart. It exposes names, positions, and reporting lines only — never contact details
or anything sensitive.

## Acceptance criteria (feature-level)

1. HR can create an employee with only a code, a first and last name, and a start date, and the
   record saves.
2. A manager listing employees sees exactly their own transitive reports plus themselves — proven by
   a test with a three-level reporting chain across two departments.
3. The same manager's API response for a team member contains no `nationalId`, `dateOfBirth`, or
   `homeAddress` **keys at all** — not keys set to null.
4. Setting employee A's manager to employee B, where B already reports to A, is rejected with a
   cycle error and no write occurs.
5. Transferring an employee on 1 April leaves a query for "department on 15 March" returning the old
   department, and "today" returning the new one.
6. A future-dated promotion does not change the profile until its effective date, then does.
7. Terminating a manager with three reports is refused until the reports are reassigned or the
   warning is explicitly confirmed.
8. A CSV with one bad row imports nothing by default and names the row number and the reason.
9. Re-running the same valid CSV twice creates employees once and updates them the second time.
10. Uploading a 12 MB file is rejected; uploading a renamed `.exe` as `.pdf` is rejected by content
    sniffing.
11. Downloading a document as a user without `employee.document.read` returns 403, and the attempt
    is audited.
12. The existing `employees` rows created during device testing survive the migration from free-text
    `department`/`position` to foreign keys, with their values preserved.
