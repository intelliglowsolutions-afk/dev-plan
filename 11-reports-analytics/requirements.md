# 11 — Reports & Analytics — Requirements

## Actors

| Actor | What they do here |
|---|---|
| HR admin | Runs most reports, exports for external use, schedules recurring ones. |
| Manager | Runs the same reports, scoped to their own team, and sees fewer of them. |
| Payroll officer | Payroll cost and variance reporting, gated by payroll permissions. |
| Super admin | Everything HR sees, plus company-wide aggregates where those are restricted (OQ-1107). |
| Employee | Sees no reports. Their own figures are in the portal (08). |

## User stories

- **US-01** — As HR, I find the report I need without knowing where its data comes from.
- **US-02** — As HR, I filter a report by period, department, and employment type, and see the result
  immediately.
- **US-03** — As HR, I export a report to CSV for someone outside the system.
- **US-04** — As a manager, I run the same reports for my own team and nothing wider.
- **US-05** — As HR, I save a set of filters I use monthly and re-run it in one click.
- **US-06** — As HR, I schedule a report to be prepared monthly, and am told when it is ready.
- **US-07** — As HR, I open a dashboard showing the handful of numbers I check regularly.
- **US-08** — As HR, I see a figure and can tell where it came from and when it was calculated.
- **US-09** — As HR, I click a number in an aggregate and see the records behind it, where I am
  permitted to.
- **US-10** — As a report reader, I am told when a figure has been suppressed, and why.
- **US-11** — As HR, I compare a period with the previous one without exporting both and using
  a spreadsheet.

## Functional requirements

### The catalogue — FR-C

| ID | Requirement |
|---|---|
| FR-C-01 | Every report is declared in code with: key, owning module, title, description, parameters, columns (with types and formats), the permission it requires, its scope behaviour, its sensitivity, and its data source (D-04). |
| FR-C-02 | A report's **data source is a call into the owning feature**, not SQL written here (D-01). Where the owning feature has no suitable query, the report is blocked until that feature provides one. |
| FR-C-03 | The catalogue is filtered per actor: a report whose permission the actor lacks does not appear. An empty category is not shown. |
| FR-C-04 | Reports whose source feature is not built are absent from the catalogue entirely, not shown as unavailable. |
| FR-C-05 | Each report declares whether it supports: charting, export, scheduling, drill-through, and period comparison. |
| FR-C-06 | Adding a report requires no migration and no UI work — the viewer is generated from the declaration, like 03's settings screen and 05's preferences screen. |

### Permissions and scope — FR-P

| ID | Requirement |
|---|---|
| FR-P-01 | Running a report requires `report.read` **and** the permission its declaration names — e.g. the payroll cost report also requires `payroll.read` (D-02). |
| FR-P-02 | Scope is applied in the query using the owning feature's scope filter. A manager running the attendance summary gets their transitive reports and nothing else. |
| FR-P-03 | There is no permission that widens a report beyond what the actor could see in the owning module. `report.read` alone grants nothing. |
| FR-P-04 | Reports over pay or performance data require their module's read permission, which by default almost nobody holds (07 D-06, 09 D-08). |
| FR-P-05 | Company-wide aggregates may be restricted separately from scoped ones (OQ-1107), declared per report. |
| FR-P-06 | Drill-through from an aggregate to its rows is permitted only where the actor could have listed those rows directly (US-09). Where they could not, the aggregate is shown and the drill-through is absent — not present and erroring. |

### Suppression — FR-S

The requirements that make D-03 real.

| ID | Requirement |
|---|---|
| FR-S-01 | Every report declares whether its measures are **sensitive** — pay, performance, and anything derived from them, at minimum. |
| FR-S-02 | An aggregate over a sensitive measure is suppressed when its population is below the configured minimum (default 5). |
| FR-S-03 | A suppressed cell shows a marker and an explanation, not a blank or a zero. A zero would be read as a fact. |
| FR-S-04 | Suppression must not be defeatable by **differencing**: if a total is shown and all but one group is suppressed, the remaining group is inferable. Where suppression applies, the report suppresses enough groups — including, where necessary, the total — that no suppressed value can be derived. |
| FR-S-05 | Suppression applies after filtering. A filter that narrows a population below the threshold triggers it, which is precisely the attack it exists to stop. |
| FR-S-06 | Row-level reports over sensitive data are not suppressed — they are permission-gated instead. Suppression is for aggregates; permissions are for rows. Conflating them protects neither. |
| FR-S-07 | The threshold is a setting, and changing it is change-controlled (03 FR-S-05). |

### Historical accuracy — FR-H

| ID | Requirement |
|---|---|
| FR-H-01 | A report over a past period uses the organisational structure **as it was** in that period (D-06): department, position, manager, and employment type from 02's assignment history. |
| FR-H-02 | Where a report deliberately uses current structure — "current headcount by today's departments" — its title and its declaration say so. Both are legitimate; only ambiguity is not. |
| FR-H-03 | Payroll reports read finalised payslips, whose snapshots already hold the historical values (07 D-03). They must not re-derive from current data. |
| FR-H-04 | A report over a period that includes a reorganisation must not double-count or drop people who moved. |
| FR-H-05 | Reports over attendance use `attendance_days` as computed, including any corrections applied (04 D-04). A recompute that changes history changes the report, which is correct and must be explicable via the as-of stamp (FR-R-03). |

### Running and results — FR-R

| ID | Requirement |
|---|---|
| FR-R-01 | Parameters are typed and validated: date ranges, department, employment type, status, and report-specific ones. Invalid combinations are rejected with a specific message. |
| FR-R-02 | Results paginate; aggregates return totals computed over the whole result, not the page. |
| FR-R-03 | Every result carries: the as-of timestamp, the filters applied, the row count, and the source modules (D-05). These appear in the UI and in every export. |
| FR-R-04 | Reports supporting period comparison return both periods and the difference, with the change expressed as both absolute and percentage. |
| FR-R-05 | A report that takes longer than a few seconds runs as a background job with progress, rather than blocking a request. |
| FR-R-06 | An empty result is distinguishable from a failed one, and from one entirely suppressed. Three different messages. |
| FR-R-07 | Currency, dates, and numbers format per 03's settings. |

### Dashboards — FR-D

| ID | Requirement |
|---|---|
| FR-D-01 | A dashboard is a composition of report tiles, defined in code per role. |
| FR-D-02 | Tiles the actor cannot see are omitted, and the layout closes up rather than leaving gaps. |
| FR-D-03 | Each tile links to the full report with the same filters applied. |
| FR-D-04 | Dashboards are read-only compositions in v1 — no user-arranged layouts (OQ-1109). |
| FR-D-05 | A dashboard must load in one request per tile at most, in parallel, with tiles rendering as they arrive rather than the page waiting for the slowest. |

### Exports and schedules — FR-E

| ID | Requirement |
|---|---|
| FR-E-01 | CSV export of any report the actor may run, respecting scope, suppression, and column-level permissions. |
| FR-E-02 | Exports include the provenance block (FR-R-03) as a header, so a spreadsheet on someone's desktop still says what it is and when it was made. |
| FR-E-03 | Every export is audited as a disclosure: who, what, filters, row count, when (D-10). |
| FR-E-04 | Scheduled reports produce a result on a schedule and notify the recipients with a **link** (D-08). No data in the message, no attachment. |
| FR-E-05 | A schedule's recipients are checked at delivery time: someone who has lost the permission stops receiving it, and the schedule's owner is told. |
| FR-E-06 | Schedules are owned by a user and disabled automatically when that user is suspended — an inherited schedule quietly emailing a departed employee's report is a real leak. |
| FR-E-07 | Saved views store parameters only, never results. |

### Explicit non-requirements — FR-X

| ID | Requirement |
|---|---|
| FR-X-01 | **No ad-hoc query builder** (D-04). |
| FR-X-02 | **No scoring, ranking, or prediction about individuals** (D-09) — no attrition risk, no performance prediction, no "employees to watch". |
| FR-X-03 | **No direct external database access** as a supported integration (OQ-1105). |
| FR-X-04 | **No report that reveals, through aggregation, something the actor could not see directly** (FR-S-04). |

## The initial catalogue

Six reports to start (OQ-1101), chosen because they answer questions people actually ask:

| Key | Title | Source | Permission | Sensitive |
|---|---|---|---|---|
| `headcount.summary` | Headcount and changes | 02 | `employee.read` | no |
| `attendance.summary` | Attendance and absence | 04 | `attendance.read` | no |
| `attendance.overtime` | Overtime by department | 04 | `attendance.read` | no |
| `leave.balances` | Leave taken and remaining | 06 | `leave.balance.read` | no |
| `leave.liability` | Leave liability | 06 + 07 | `payroll.read` | **yes** |
| `payroll.cost` | Payroll cost by department | 07 | `payroll.read` | **yes** |

The full list in the README is the backlog, not the build.

## Non-functional requirements

| ID | Requirement |
|---|---|
| NFR-01 | A report over a year of data for 200 employees returns in under 3 seconds, or runs as a background job (FR-R-05). |
| NFR-02 | Report queries must not scan `attendance_punches`; they read `attendance_days` (04 NFR-06). |
| NFR-03 | Suppression and scope are applied in the query. Fetching then filtering leaks totals and row counts through pagination. |
| NFR-04 | No report may issue a query per row. Historical structure (FR-H-01) is resolved with one join to assignment history, not per person. |
| NFR-05 | Exports stream rather than buffering the whole result in memory. |
| NFR-06 | Report definitions must be testable in isolation, with fixture data and a fixed as-of time. |
| NFR-07 | No personal data in application logs, including in query parameters. |

## Permission keys added by this feature

| Key | Meaning | Default holders |
|---|---|---|
| `report.read` | Use the reporting section at all | HR: ALL · Manager: DEPARTMENT |
| `report.export` | Export results | HR: ALL · Manager: DEPARTMENT |
| `report.schedule` | Create scheduled reports | HR: ALL |
| `report.company_wide` | See unscoped, company-wide aggregates | HR admin, super admin (OQ-1107) |

`report.read` grants **nothing on its own** (FR-P-03). Every report additionally requires its
module's permission, which is what keeps this feature from becoming a side door.

## Acceptance criteria (feature-level)

1. A manager running the attendance summary sees only their transitive reports, verified with a
   three-level chain across two departments.
2. An actor without `payroll.read` does not see the payroll cost report in the catalogue at all.
3. An average-salary aggregate for a department of 3 is suppressed with an explanation, not a blank.
4. Filtering a suppressed report until one group remains does not reveal the value by differencing:
   enough groups, including the total where needed, are suppressed.
5. Headcount by department for March uses March's departments, and differs from today's where a
   reorganisation happened.
6. The payroll cost report for a finalised period matches the sum of that run's payslips exactly.
7. Overtime hours in the attendance report match feature 04's own screens for the same filters.
8. Every result and every export carries as-of time, filters, row count, and sources.
9. An export is audited with the filters used and the row count.
10. A scheduled report to a user who has lost the required permission stops being delivered, and the
    schedule's owner is notified.
11. A schedule owned by a suspended user is disabled automatically.
12. An empty result, a fully suppressed result, and a failed run produce three distinct messages.
13. No endpoint accepts a user-supplied query, column list, or grouping outside its declaration.
