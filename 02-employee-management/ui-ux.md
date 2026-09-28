# 02 — Employee Management — UI / UX

Builds on the shell, navigation gating, and copy principles in
[01 ui-ux.md](../01-roles-permissions-auth/ui-ux.md). This is the HR-facing UI; the employee's own
view of their record is feature 08.

## Principles specific to this feature

1. **A manager and an HR admin see the same screens with different amounts of data.** There is no
   separate "manager view" to build and maintain — the profile simply renders fewer panels. Sections
   the actor cannot see are absent, not greyed out (a greyed-out *National ID* row still tells you
   there is one).
2. **Structural changes are dated events, not edits.** Transfer, promote, terminate are their own
   flows with an effective date, never a dropdown on the edit form. The UI must make that feel
   natural rather than bureaucratic, or HR will ask for the dropdown.
3. **The profile is the hub.** Attendance, leave, and payroll (features 04/06/07) will add tabs to
   it. The tab structure is designed now so those land cleanly instead of being bolted on.

## Screen map

```
/employees                        List                            employee.read
/employees/new                    Create                          employee.write
/employees/:id                    Profile — tabbed                employee.read
/employees/:id/edit               Edit details                    employee.write
/employees/import                 CSV import wizard               employee.import
/org-chart                        Org chart                       department.read
/admin/departments                Department tree                 department.read
/admin/positions                  Positions                       position.read
```

## `/employees` — the list

The screen HR lives in. A dense table, 25 rows per page:

| Column | Notes |
|---|---|
| Photo + name | 28 px avatar, initials fallback. Preferred name shown, legal name in the tooltip |
| Code | Monospaced — it is the device PIN and gets read aloud and typed |
| Department | |
| Position | |
| Manager | Links to that profile |
| Status | Badge with text, never colour alone |
| Hire date | Absolute; relative tenure in the tooltip |

Toolbar: search (name, code, email — debounced, minimum 2 characters), and filters for department
(a tree picker with an *include sub-departments* toggle), position, status, and manager. Filters are
reflected in the URL so a view can be shared. An active-filter row sits above the table with
removable chips and a *Clear all*.

Terminated employees are **excluded by default**, and the filter chip row says so explicitly —
"Showing employed only" with a one-click *Include terminated*. A silent exclusion is worse than a
visible one.

Actions: **Add employee**, **Import**, **Export** — each rendered only with its permission. Export
downloads the current filter's result and warns when the row count is large.

Row click opens the profile. No inline editing: every change here has dating or auditing
consequences.

**Empty states.** No employees at all: "No employees yet. Add them one at a time, or import your
existing list from a spreadsheet." with both buttons — the day-one screen, and the moment HR decides
whether this system is going to be painful. No results for a filter: "No employees match these
filters", with *Clear filters*. A manager whose team is empty: "You don't have any direct reports
yet." — not "no employees", which would read as a broken system.

## `/employees/new`

One page, not a wizard — HR creating a batch of new joiners should not click *Next* four times per
person. Grouped sections, with only the first expanded:

1. **Identity** (required): code, first name, last name. The code field shows the format rule and
   validates as you type; a duplicate is caught on blur, not on submit, and says so plainly —
   including when the code belongs to a terminated employee: "Code 1042 belonged to Imran Sheikh
   (left Mar 2025). Codes are never reused."
2. **Job**: department, position, manager, employment type, FTE, hire date, status (Probation or
   Active).
3. **Contact**: work email and phone.
4. **Personal** (only with `employee.read_sensitive`): DOB, gender, national ID, address.
5. **Emergency contacts**: repeatable rows.

Save creates and lands on the new profile, with a toast offering the two most likely next steps:
*Upload documents* and *Create a login account* (the latter deep-links into feature 01's invite flow
with the employee pre-linked). **Save and add another** keeps the department, position, and manager
values, which are usually the same across a batch of joiners.

## `/employees/:id` — the profile

**Header:** photo, preferred name (legal name beneath), position and department, status badge, code,
manager (linked), tenure ("2 years, 7 months"). Action buttons, permission-gated and grouped so that
the routine ones are prominent and the consequential ones are one click further away:
**Edit** · **Documents** · a *More* menu holding *Transfer or promote*, *Confirm probation*,
*Change status*, *Terminate*.

**Tabs:**

- **Overview** — contact details, personal details (only with `employee.read_sensitive`), emergency
  contacts, direct reports as a small card list.
- **Job & history** — current assignment, then the full timeline: every department, position, and
  manager change with dates and reasons, and the employment periods. Rendered as a vertical
  timeline, newest first, with backdated entries marked. This is the tab that answers "why is this
  person's approval going to the wrong manager".
- **Documents** — grouped by type, with expiry dates; expiring-soon in amber, expired in red.
  Upload via drag-and-drop or a button.
- **Attendance** — placeholder in this feature; filled by 04.
- **Leave** — placeholder; filled by 06.
- **Payroll** — placeholder; filled by 07, behind its own permission.

A manager viewing a team member sees Overview (without the personal section), Job & history, and
later Attendance and Leave. No Documents tab, no Payroll tab.

For a terminated employee the whole profile renders read-only behind a banner: "Left the company on
30 Nov 2026 — Resignation. Eligible for rehire." with a *Rehire* action for those permitted.

## Lifecycle flows

Each is a focused dialog, not a page. All four share a shape: what is changing, from what to what,
on which date, and what else it will affect.

### Transfer or promote

Fields: effective date (default today), new department, new position, new manager, employment type,
FTE, reason. Unchanged fields show their current value greyed with "unchanged".

A live summary sentence updates as fields change: "From 1 Oct 2026, Sara Ahmed moves from *Finance*
to *Operations* and reports to *Ravi Menon* instead of *Imran Qadir*."

A future date shows: "This takes effect on 1 Oct 2026. Until then the profile shows the current
job." A past date warns, and requires `employee.edit_history`: "Backdating changes answers this
system has already given — reports and approvals routed before today will not be recalculated."

Cycle errors are caught in the picker, not at submit: the manager picker excludes the employee and
everyone who reports to them, so the invalid choice cannot be made. The server still rejects it
(FR-O-06) — the picker is convenience, not enforcement.

### Terminate

The most consequential action in the feature, so it is the most deliberate. Fields: last working
day, reason category, note, notice days, eligible for rehire.

If the employee has reports, the dialog shows them by name with a required **reassign to** picker,
and refuses to proceed without it — or an explicit tick of "leave them without a manager for now",
which states the consequence: "Leave approvals for these 3 people will have nowhere to route."

The confirmation step lists what will happen and what will not:

> Sara Ahmed will be marked terminated on 30 Nov 2026.
> - Attendance will stop being expected from 1 Dec.
> - 3 direct reports will move to Ravi Menon.
> - Her login account stays active — suspend it separately if needed. *(Suspend now)*
> - Her records, attendance, and payslips are kept.

That last group of lines exists because the two most common post-termination support questions are
"why can they still log in" and "where did their payslips go".

### Confirm probation / change status

Small dialogs: effective date and a note. *Change status* offers On leave, Suspended, or back to
Active, each with a short explanation of what it means for attendance and leave — because the
difference between "on leave" and "suspended" is invisible otherwise.

## `/employees/import` — the CSV wizard

The one place a wizard is right, because the steps are genuinely sequential.

1. **Download the template**, with the column list and rules stated on-page, not only in the file.
2. **Upload** — drag and drop, with a file name and row count shown before proceeding.
3. **Preview** — the dry run. A summary line ("231 to create, 6 to update, 3 rows with errors"),
   then the errors table with row number, column, offending value, and a plain-language reason.
   Unknown departments and positions are listed with a *Create these automatically* checkbox.
   Errors are downloadable as an annotated CSV, so HR can fix the file rather than transcribing.
4. **Import** — with the default "don't import anything if there are errors" stated as a sentence,
   and the alternative available as a deliberate choice.
5. **Result** — counts, a link to the filtered list of everyone just created, and the errors again
   if any rows were skipped.

The import must never look like it silently half-worked. If it wrote nothing, the result screen says
so in the first line.

## `/org-chart`

Zoom and pan over a node tree, rooted at the top of the company or at any employee. Each node: photo,
name, position, department, direct-report count. Collapsed nodes with children show an expander with
the count.

Controls: search (jumps to and highlights a person), *root at this person*, depth selector, zoom,
*fit to screen*, and export to PNG/PDF. Clicking a node opens the profile; a breadcrumb along the top
shows the chain from the root to the selected person.

Load two levels initially, expand on demand (NFR-03). Employees with no manager render as multiple
roots side by side, with a note — usually that is the CEO alone, and anything else is a data problem
worth surfacing: "3 employees have no manager set."

Below tablet width the chart falls back to an indented list view. A pan-and-zoom canvas on a phone
is a worse way to read a hierarchy than an outline.

## `/admin/departments`

A tree with drag-to-reparent (with a confirmation naming both departments, and keyboard equivalents
via a *Move to* menu — drag must not be the only way). Each node shows name, head, direct and total
employee counts. Inline *Add sub-department*.

Editing opens a side panel: name, code, parent, head, description, active. Deactivating explains the
effect: "*Logistics* will no longer be selectable for new employees. The 12 people currently in it
keep their department, and history is unchanged."

Delete is offered only for an empty department with no history; otherwise the control is disabled
with the reason and a count that links to the filtered employee list.

## `/admin/positions`

A plain table: title, department, grade, employee count, active. Filter by department. Same
deactivate-not-delete pattern. Creating a position that duplicates an existing title in the same
department is caught on blur.

## States, accessibility, responsive

All list and detail screens need the six states from feature 01: loading (skeleton rows), empty,
filtered-empty, error, permission-denied, and saving.

- Status badges carry text; expiry states carry an icon as well as a colour.
- The org chart needs a keyboard-navigable alternative — the indented list view is it, offered as a
  toggle at every width, not only on mobile.
- Photos have `alt` text of the person's name; initials avatars are `aria-hidden` with the name in
  adjacent text rather than encoded in the avatar.
- The timeline on Job & history is a `<ol>`, so it reads as an ordered sequence.
- Tables: the employee list falls back to stacked cards below 768 px, keeping name, code, department,
  and status.

## Copy reference

| Situation | Text |
|---|---|
| Code reused | Code {code} belonged to {name} (left {date}). Codes are never reused. |
| Code format | Employee codes must match the attendance terminal: numbers only, up to 9 digits. |
| Immutable code | The employee code cannot be changed — attendance records are matched by it. |
| Future assignment | This takes effect on {date}. Until then the profile shows the current job. |
| Backdating | Backdating changes answers this system has already given — reports and approvals routed before today will not be recalculated. |
| Terminate with reports | {n} people report to {name}. Choose a new manager for them, or continue and leave them unassigned. |
| Orphan reports warning | Leave approvals for these {n} people will have nowhere to route. |
| Login account note | Their login account stays active — suspend it separately if needed. |
| Department deactivated | {name} will no longer be selectable for new employees. The {n} people currently in it keep their department, and history is unchanged. |
| Department in use | {n} employees and {m} sub-departments are in this department. Deactivate it instead. |
| Import, nothing written | Nothing was imported. Fix the {n} rows listed below and upload the file again. |
| No manager set | {n} employees have no manager set. |
| Manager's empty team | You don't have any direct reports yet. |

## Open questions

| ID | Question |
|---|---|
| OQ-213 | Does HR want to edit an employee's photo separately from documents? Treating it as a document is tidy in the model but may feel indirect in the UI. |
| OQ-214 | Should the profile show a "completeness" indicator (missing DOB, no contract on file)? Useful for chasing paperwork; risks nagging. |
| OQ-202 | Which personal fields to show at all — the Personal section's contents depend entirely on this. |
| OQ-206 | If locations exist, the list needs a location filter and the org chart a location tint. |
