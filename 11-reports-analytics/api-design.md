# 11 — Reports & Analytics — API Design

Conventions as in [01 api-design.md](../01-roles-permissions-auth/api-design.md).

## The report definition — the whole contract

Everything in this feature follows from this shape. A report is a declaration, not a query
(D-04, FR-C-01):

```ts
// src/lib/reports/definitions/attendance-overtime.ts
defineReport({
  key: "attendance.overtime",
  module: "04",
  title: "Overtime by department",
  description: "Approved overtime hours, by department, for a period.",

  // Required in addition to report.read (FR-P-01). report.read alone grants nothing.
  permission: "attendance.read",
  scope: "INHERIT",            // uses 04's employeeScopeFilter for the actor
  sensitive: false,            // no suppression (FR-S-01)
  companyWideRequires: null,   // or "report.company_wide" (OQ-1107)

  parameters: {
    from: dateParam({ required: true }),
    to: dateParam({ required: true, maxRangeDays: 400 }),
    departmentId: departmentParam({ includeDescendants: true }),
    employmentType: enumParam(EmploymentType),
  },

  columns: [
    { key: "department", label: "Department", type: "text" },
    { key: "employeeCount", label: "Employees", type: "integer" },
    { key: "overtimeHours", label: "Overtime hours", type: "decimal", aggregate: "sum" },
  ],

  // The data source is a CALL INTO FEATURE 04 (D-01, FR-C-02). No SQL is written here.
  source: async (params, ctx) =>
    attendance.overtimeByDepartment(params, ctx),

  historicalStructure: true,   // resolve departments as at each date (FR-H-01, D-06)
  supports: { chart: "bar", export: true, schedule: true,
              drillThrough: "attendance.byEmployee", compare: true },
});
```

`source` being a function rather than a query string is the single most important line in this
feature. It is what stops overtime hours in a report drifting from overtime hours in feature 04's
screens (D-01) — the failure that quietly discredits reporting systems.

A definition whose `source` has no implementation in the owning feature **does not ship**. The
correct response is a request to that feature, not a query written here.

## Catalogue

| Method | Path | Permission |
|---|---|---|
| GET | `/api/reports` | `report.read` |
| GET | `/api/reports/:key` | `report.read` + the report's own |

`GET /api/reports` returns only reports the actor can actually run (FR-C-03), grouped by module:

```json
{ "groups": [
    { "module": "04", "label": "Attendance",
      "reports": [ { "key": "attendance.overtime", "title": "Overtime by department",
                     "description": "Approved overtime hours, by department, for a period.",
                     "supports": { "chart": true, "export": true, "schedule": true } } ] } ] }
```

A report the actor lacks permission for is **absent**, not listed-and-disabled. A disabled row
advertises what exists and who has it — the same reasoning as feature 01's hidden navigation.

`GET /api/reports/:key` returns the parameter declaration so the filter UI is generated rather than
hand-built (FR-C-06) — the fifth generated-from-catalogue UI in the plan.

## Running

### `POST /api/reports/:key/run`

```json
{ "parameters": { "from": "2026-01-01", "to": "2026-09-30", "departmentId": 3 } }
```

The pipeline, in order, and the order matters:

1. Check `report.read` **and** the declared permission (FR-P-01).
2. Validate parameters against the declaration; reject unknown keys outright — a parameter not in
   the declaration is never passed through (FR-X-01).
3. Resolve the actor's scope through the owning feature's filter (FR-P-02).
4. Call `source(params, ctx)` — the owning feature applies scope **in its query** (NFR-03).
5. Apply suppression if the report is sensitive (FR-S-02 … FR-S-05).
6. Write a `ReportRun` record and return with provenance.

```json
{ "runId": 4412,
  "asOf": "2026-09-15T11:02:14Z",
  "parameters": { "from": "2026-01-01", "to": "2026-09-30", "departmentId": 3 },
  "sources": ["04", "02"],
  "scopeNote": "Your team (14 people)",
  "columns": [ … ],
  "rows": [ { "department": "Finance", "employeeCount": 14, "overtimeHours": 212.5 } ],
  "totals": { "employeeCount": 14, "overtimeHours": 212.5 },
  "rowCount": 1,
  "suppressed": { "cellCount": 0 },
  "durationMs": 340 }
```

`asOf`, `parameters`, `sources`, and `scopeNote` travel with every result and every export (D-05,
FR-R-03, FR-E-02). `scopeNote` is what stops a manager mistaking their team's figure for the
company's — the most common misreading of a scoped report.

A run exceeding the background threshold returns **202** with a run id, and the client polls
`GET /api/reports/runs/:id` (FR-R-05).

### Suppression in the response

When suppression applies, the values are **absent**, not nulled — and the response says how many and
why (FR-S-03):

```json
{ "rows": [ { "department": "Finance", "employeeCount": 14, "averageSalary": 84200 },
            { "department": "Legal", "employeeCount": 3, "averageSalary": null,
              "suppressed": ["averageSalary"] },
            { "department": "Executive", "employeeCount": 4, "averageSalary": null,
              "suppressed": ["averageSalary"] } ],
  "totals": { "employeeCount": 21, "averageSalary": null, "suppressed": ["averageSalary"] },
  "suppressed": { "cellCount": 3,
                  "threshold": 5,
                  "message": "Some figures are hidden because they cover fewer than 5 people." } }
```

Note the third row and the total. Legal alone is below the threshold; suppressing only Legal would
let its average be derived from the total and the other rows, so the next-smallest group and the
total are suppressed too (FR-S-04, `data-model.md` steps 3–4). **A response that suppresses exactly
one group is a bug**, and is worth an explicit test.

### Drill-through

`POST /api/reports/:key/drill` with a row's grouping values returns the underlying rows — **only if
the actor could have listed them directly** (FR-P-06). Where they could not, `supports.drillThrough`
is absent from the report declaration for that actor, so the UI never offers a control that would
then refuse.

### Comparison

`POST /api/reports/:key/run` with `"compareWith": { "from": …, "to": … }` returns both periods and
the difference per row (FR-R-04). Suppression is applied to both periods independently and to the
difference — a difference between two suppressed values is itself disclosive.

## Exports

| Method | Path | Permission |
|---|---|---|
| POST | `/api/reports/:key/export` | `report.export` + the report's own |
| GET | `/api/reports/exports/:runId/download` | the run's owner |

Exports stream (NFR-05) and carry the provenance block as header rows:

```
# Overtime by department
# Run 15 Sep 2026 11:02 UTC by hr@company.com
# Period 1 Jan 2026 – 30 Sep 2026 · Department: Finance (incl. sub-departments)
# Scope: your team (14 people) · Sources: Attendance, Employees
# Some figures are hidden because they cover fewer than 5 people.
Department,Employees,Overtime hours
Finance,14,212.5
```

A CSV on someone's desktop in six months must still say what it is, when it was made, and what was
withheld. Exports over `report.exportRowLimit` are refused with the count and a suggestion to narrow
the filters, rather than producing a file that will not open.

Every export writes an audit entry with filters and row count (D-10, FR-E-03).

## Saved views and schedules

| Method | Path | Permission |
|---|---|---|
| GET / POST | `/api/reports/views` | `report.read` |
| DELETE | `/api/reports/views/:id` | owner |
| GET / POST | `/api/reports/schedules` | `report.schedule` |
| PATCH / DELETE | `/api/reports/schedules/:id` | owner |

Creating a schedule validates that **every recipient** currently holds the report's permission, and
warns about any who do not rather than silently dropping them.

At delivery (FR-E-05): recipients are re-checked; those who have lost access are skipped and the
owner is told who and why. A schedule whose owner is suspended is disabled with a reason (FR-E-06).

Notifications carry a link only (D-08):

> **Payroll cost by department — September** is ready.
> *(Open the report)*

No figures, no attachment, no summary line with a total in it. The link requires a session, so
access control survives forwarding — which an emailed CSV does not.

## Dashboards

`GET /api/dashboards/:key` returns the tile declarations the actor may see; each tile is fetched by
its own `run` call in parallel (FR-D-05), so a slow tile does not hold the page.

Tiles the actor cannot see are omitted server-side (FR-D-02) — the client is never given a list it
must filter, since a filtered-out tile title still tells the reader what exists.

## What this API refuses

Stated explicitly, because these are the requests that will arrive (FR-X):

| Request | Response |
|---|---|
| An arbitrary SQL or query-DSL endpoint | Not built (D-04). |
| A `columns` or `groupBy` parameter chosen by the caller | Rejected — columns come from the declaration. |
| A report with a `permission` of only `report.read` | Cannot be declared; the field is required and validated at startup. |
| Database credentials for Excel or Power BI | Not supported (OQ-1105). Every permission rule in features 01–10 lives in application code. |
| A per-individual score, rank, or prediction | Not built (D-09, FR-X-02). |

The startup validation is worth noting: definitions are checked when the application boots — every
report has a permission, every sensitive report declares its measures, every `source` resolves.
A misdeclared report fails the boot rather than quietly shipping with no permission check.

## Scheduled jobs

| Job | Frequency | Does |
|---|---|---|
| Scheduled reports | hourly | Runs due schedules, re-checks recipients, notifies with links |
| Schedule hygiene | daily | Disables schedules whose owner is suspended; reports recipients who lost access |
| ~~Run retention~~ | — | **Removed 2026-09-18 (OQ-1002): no automatic deletion.** `ReportRun` rows are retained indefinitely; `report.runRetentionDays` becomes a reporting threshold rather than a deletion trigger |

## Open questions

| ID | Question |
|---|---|
| OQ-1101 | The six starting reports are a proposal. Each one costs real work; the machinery does not. |
| OQ-1105 | External BI access. Refusing it is easy now and politically harder after someone has been promised it. |
| OQ-1106 | PDF export needs a layout pass and 03's branding; CSV needs neither. |
| OQ-1111 | Should a scoped report's `scopeNote` appear in exports as well as the UI? Proposed yes — a manager's CSV emailed onward is otherwise indistinguishable from a company-wide one. |
