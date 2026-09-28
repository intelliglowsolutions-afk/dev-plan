# 02 — Employee Management

**Priority:** Must-have · **Build order:** 2 of 11 · **Status:** planned (Step 3)

## Purpose

Hold the record of every person the company employs: who they are, how to reach them, what job they
do, who they report to, and what has changed over time. Nine of the eleven features reference an
employee; this one owns that record.

It also supplies the **reporting lines** that make `DEPARTMENT` scope work. Until this feature
lands, feature 01's scope resolution deliberately returns an empty set (FR-Z-06), so managers can
see nothing until the org structure exists.

## Scope

**In scope**

- Employee records: identity, contact details, personal details, emergency contacts.
- The org structure: departments (a tree), positions/job titles, and the reporting line for each
  employee.
- **Employment history** — every department, position, manager, and employment-type change kept as a
  dated record, not overwritten.
- The employment lifecycle: hire, confirm from probation, transfer, promote, terminate, rehire.
- Employee documents: contracts, ID copies, certificates — upload, list, download, expiry tracking.
- The link to the attendance terminal: `employeeCode` is the PIN enrolled on the SenseFace device.
- Bulk import from CSV, for getting an existing workforce into the system on day one.
- Org chart view.

**Out of scope (owned elsewhere)**

- The login account and permissions → [01 Roles, Permissions & Auth](../01-roles-permissions-auth/README.md).
  An employee may have no account at all.
- Company profile, holiday calendar, work week, device registration → [03 Admin & Settings](../03-admin-settings/README.md).
- Shift assignment and attendance records → [04 Attendance Tracking](../04-attendance-tracking/README.md).
  This feature supplies the employee; attendance supplies the punches.
- Leave balances and entitlement → [06 Leave Management](../06-leave-management/README.md).
- **Salary figures, pay structure, and bank details used for payment** → [07 Payroll](../07-payroll/README.md).
  This feature stores the *job* (position, grade, employment type); payroll stores what that job
  pays. See D-05 — this split is deliberate and worth reading before building either.
- The employee's own view of their record → [08 Employee Self-Service](../08-employee-self-service/README.md).
  This feature's UI is the HR-facing one.
- Candidate records before hire → [10 Recruitment & Onboarding](../10-recruitment-onboarding/README.md),
  which ends by creating an employee here.

## Files

| File | Contents |
|---|---|
| [requirements.md](./requirements.md) | User stories, functional and non-functional requirements, the permission keys this feature adds |
| [data-model.md](./data-model.md) | Prisma models, the migration away from the current free-text `department`/`position` columns, seed and import notes |
| [api-design.md](./api-design.md) | Route handlers, payloads, scope filtering, the org-chart and import endpoints |
| [ui-ux.md](./ui-ux.md) | Screens, the employee profile, org chart, lifecycle flows, states and copy |

## Key decisions taken in this plan

| # | Decision | Rationale |
|---|---|---|
| D-01 | `Department` and `Position` become **models with foreign keys**, replacing the free-text `department`/`position` strings on the existing `Employee` model. | Free text cannot be filtered reliably, cannot carry a manager or a parent department, and cannot be renamed in one place. "Sales", "sales", and "Sales Dept" are three departments today. |
| D-02 | Departments form a **tree** (self-referencing parent). Positions are a **flat list** scoped to an optional department. | Companies nest departments; job titles do not nest. |
| D-03 | The reporting line is a **direct link on the employee** (`managerId`), not derived from the department head. | The two differ often enough — a person seconded to another team, a department head who does not line-manage everyone in it — that deriving it produces wrong answers in exactly the cases that matter for approvals. |
| D-04 | Every change of department, position, manager, or employment type is written to an **`EmployeeAssignment`** history row with a date range. The employee's current values are a denormalised convenience, not the source of truth. | Payroll and reports must be able to ask "which department was this person in last March", and an approval routed in April must stay explicable in December. |
| D-05 | This feature stores **job structure** (position, grade, employment type, FTE). Feature 07 stores **compensation** (amounts, allowances, bank account for payment). | Salary is the most sensitive data in the system and needs its own permission boundary. Keeping the amounts out of the employee record means `employee.read` never leaks pay. |
| D-06 | Sensitive personal fields — national ID, date of birth, bank details, home address — sit behind a **separate permission** (`employee.read_sensitive`), not behind plain `employee.read`. | A manager needs to see their team's contact details and job history. They do not need everyone's passport number. |
| D-07 | Documents are stored as **files on a mounted volume**, with metadata in Postgres. Not as `bytea`. | Contracts and scans are megabytes. In-row storage bloats the database, breaks backups, and makes every employee query slower. |
| D-08 | Employees are **never deleted**, only terminated. `employeeCode` is never reused. | Attendance punches, payslips, and audit entries all point here. A rehire gets the same employee record reactivated, not a new one. |
| D-09 | `employeeCode` is the **device PIN** and is therefore constrained by what the SenseFace terminal accepts. | The existing schema already comments this. Getting it wrong means attendance silently fails to resolve to a person. See OQ-204. |

## Dependencies

- **Depends on:** 01 — every endpoint here is permission-checked and audited; the sensitive-field
  split (D-06) and `employeeScopeFilter` come from there.
- **Depended on by:** 04, 06, 07, 08, 09, 10, 11 — all key off `Employee`. 01's `DEPARTMENT` scope
  becomes functional only once this feature supplies `managerId`.
- **Touches existing code:** the `Employee` model in `prisma/schema.prisma` already exists with
  three attendance-related fields and two free-text columns. This feature migrates it (see
  `data-model.md`), so the migration must preserve any rows already seeded during device testing.

## Open questions

| ID | Question | Why it matters | Proposed default |
|---|---|---|---|
| OQ-201 | Is `DEPARTMENT` scope resolved by the **manager chain** (the actor's reports, transitively) or by **department membership** (everyone in the actor's department and its sub-departments)? | Feature 01 assumed the manager chain. A department head who is not everyone's line manager would then see less than expected. The two answers give materially different access. | Manager chain, as 01 assumed, **plus** department membership where the actor is that department's head. Needs confirmation — this is the single most consequential question in this feature. |
| OQ-202 | Which personal fields does the company actually need to hold? National ID, marital status, religion, nationality, photograph, medical notes? | Collecting personal data that is never used is a liability, and some of these are protected categories in many jurisdictions. | A minimal set (see `data-model.md`); anything beyond it should be justified per field. |
| OQ-203 | Is there a legal or contractual retention period for terminated employees' records, after which they must be deleted or anonymised? | D-08 says never delete. A retention obligation would override that and requires an anonymisation path. | Retain indefinitely until told otherwise; flagged as needing a real answer before go-live. |
| OQ-204 | What exactly does the SenseFace 2A accept as a user PIN — numeric only? maximum length? | `employeeCode` must be enrollable on the terminal, or attendance never resolves to a person (D-09). Blocked on the same missing manual as OQ-000. | Numeric, up to 9 digits, until the manual says otherwise. |
| OQ-205 | Do employees have more than one contract/employment period (rehires), and should each be a separate record or a new date range on the same employee? | Affects tenure calculations, leave accrual, and payroll history. | One employee record, multiple `EmploymentPeriod` ranges. |
| OQ-206 | Are there multiple work locations / branches, and does an employee belong to one? | Attendance devices are per-location, and reports are usually wanted per branch. If yes, it is cheaper to add now than later. | Assume a single location in v1; the model leaves room. |
| OQ-207 | Custom fields — does HR need to add their own fields to the employee record without a developer? | A generic EAV table is a large build and slows every query. | No custom fields in v1. |
| OQ-208 | Document types and whether any are mandatory before an employee can be marked active. | Drives onboarding checklists in feature 10. | A seeded list of types, none mandatory in v1. |
