# 08 — Employee Self-Service — Requirements

## Actors

| Actor | What they do here |
|---|---|
| Employee | Everything. The only actor this feature is designed for. |
| Manager | Uses it as an employee. Sees a prompt when approvals are waiting, which links out to features 04 and 06 (README, "Managers"). |
| HR admin | Does not use the portal, but configures which fields are self-editable (OQ-801) and processes the change requests it produces. |
| New hire | Arrives here on their first day, before they know anything about the company or the system. The portal's first-run experience is designed for them. |

## User stories

### Orientation

- **US-01** — As an employee, I sign in and immediately see anything that needs me, without hunting.
- **US-02** — As an employee, I see the few facts I check most — leave left, last payslip, whether
  today's attendance looks right — on one screen.
- **US-03** — As a new joiner, I understand what this system is for within a few seconds of my first
  visit.

### My information

- **US-04** — As an employee, I see my own details: job, department, manager, start date, contact
  information.
- **US-05** — As an employee, I correct my own phone number without asking anyone.
- **US-06** — As an employee, I ask HR to change something I cannot change myself, and I can see that
  they have received it and what they decided.
- **US-07** — As an employee, I keep my emergency contacts up to date.
- **US-08** — As an employee, I see the documents HR holds for me, and I am told when one is about
  to expire.

### My time

- **US-09** — As an employee, I see my attendance for this month: which days I was in, when, and
  anything that looks wrong.
- **US-10** — As an employee, I understand why a day is marked as it is, without needing to ask.
- **US-11** — As an employee, I tell HR when a day is wrong — a missed sign-out, a day I was here
  and the system says I wasn't — and I can see what happened to my request.

### My leave

- **US-12** — As an employee, I see how much leave I have left and when any of it expires.
- **US-13** — As an employee, I request leave and know before I submit what it will cost me and what
  I will have left.
- **US-14** — As an employee, I see where my request has got to and who has it.
- **US-15** — As an employee, I cancel leave I no longer need.
- **US-16** — As an employee, I see when my colleagues are off, so I do not ask for the same week.

### My pay

- **US-17** — As an employee, I see my payslips and download one as a PDF.
- **US-18** — As an employee, I understand what each line on my payslip means and why this month
  differs from last.

### Staying informed

- **US-19** — As an employee, I see my notifications and what needs my attention.
- **US-20** — As an employee, I choose which emails I get.
- **US-21** — As an employee, I look someone up in the company directory and see where they sit in
  the organisation.

## Functional requirements

### The shell — FR-S

| ID | Requirement |
|---|---|
| FR-S-01 | The portal is part of the same application and uses the same session as the admin side (D-04). |
| FR-S-02 | After sign-in, a user is routed by permission: someone with only self-service permissions lands on the portal dashboard; an admin lands on their dashboard; both can reach the portal. |
| FR-S-03 | A user holding admin permissions can switch between the admin shell and their own portal, and the current context is always visible — nobody should wonder whether they are looking at their own leave or someone else's. |
| FR-S-04 | Navigation is limited to what the employee can use: Home, My time, My leave, My pay, My profile, Directory. Sections whose feature is unavailable to them are absent, not disabled (01's rule). |
| FR-S-05 | Every screen is built and reviewed at 375 px first (D-02). |
| FR-S-06 | The portal never exposes internal state names, feature vocabulary, entity ids, or processing details (D-05, D-06). |
| FR-S-07 | Any screen whose underlying feature is not yet configured shows a plain explanation rather than an empty state or an error (D-09). |

### Dashboard — FR-D

| ID | Requirement |
|---|---|
| FR-D-01 | The dashboard is assembled from one request, not one per section (NFR-01). |
| FR-D-02 | It shows, in priority order: things requiring action, then things worth knowing, then quick actions. |
| FR-D-03 | Action items include: a pending profile change decision, a rejected correction or leave request, a document expiring, a new payslip, unread notifications and — for managers — approvals waiting (README, "Managers"). |
| FR-D-04 | Facts shown: leave balance for the primary type, today's or the most recent attendance, the latest published payslip's period, and the next public holiday. |
| FR-D-05 | Quick actions: request leave, report an attendance problem, view payslip. |
| FR-D-06 | An employee with nothing pending sees a calm, complete screen — not an apologetic empty state. "Nothing needs your attention" is a good outcome. |
| FR-D-07 | Sections the employee has no permission for are omitted from the payload entirely, not returned empty. |

### My profile and change requests — FR-P

| ID | Requirement |
|---|---|
| FR-P-01 | An employee sees their own record: name, code, job, department, manager, start date, contact details, and their own sensitive fields. Feature 02's `employee.read_sensitive` restriction does not apply to a person's own data. |
| FR-P-02 | Fields configured as self-editable (OQ-801) are edited directly and take effect immediately, audited as the employee's own change. |
| FR-P-03 | All other fields are changed by **request**: the employee submits the proposed value with an optional note; HR approves or rejects with a reason. |
| FR-P-04 | An approved change request is applied by the system to the employee record, attributed to the approving HR user with the employee named as the requester. |
| FR-P-05 | A field with a pending request is shown as pending and cannot have a second request raised. |
| FR-P-06 | The employee sees the status and history of their requests, including rejections and their reasons. |
| FR-P-07 | Change requests notify HR on submission and the employee on decision (05). |
| FR-P-08 | Emergency contacts are fully self-managed: add, edit, remove, no approval. Nobody benefits from HR mediating them. |
| FR-P-09 | The employee may update their photo where configured, subject to feature 02's document rules. |
| FR-P-10 | Bank details are **never** editable or requestable here (OQ-801). Changing them is an in-person or otherwise verified process, because it is the system's most attractive fraud target (07 FR-S-06). |

### My attendance — FR-A

| ID | Requirement |
|---|---|
| FR-A-01 | The employee sees their own attendance: a month view with each day's status, in and out times, and hours. |
| FR-A-02 | Statuses are shown in plain language: "Present", "Late", "You didn't sign out", "No record", "On leave", "Public holiday", "Day off" (D-05). |
| FR-A-03 | A day opens to show what was expected and what was recorded, in the same plain language. Feature 04's reason trail is translated, not rendered raw. |
| FR-A-04 | Where a day is `UNKNOWN` because of a device outage (04 D-08), the employee is told the terminal was not working and that this is not counted against them. This is the single most important translation in the feature. |
| FR-A-05 | The employee can request a correction for any of their days, subject to 04's period lock, using 04's existing flow. |
| FR-A-06 | Pending and decided correction requests are visible with their status and any rejection reason. |
| FR-A-07 | This month's totals — days present, days absent, late arrivals, overtime hours — are shown. No trends, no history beyond the current and previous month, no comparison with anyone (OQ-804). |

### My leave — FR-L

| ID | Requirement |
|---|---|
| FR-L-01 | Balances per type, showing available prominently and entitled/used/pending beneath (06 FR-B-04). |
| FR-L-02 | Expiring leave is surfaced on the balance with its date (06). |
| FR-L-03 | "How is this worked out?" opens the ledger as a dated, plain-language list (D-07). |
| FR-L-04 | Requesting leave uses 06's flow, including the live cost breakdown before submission — the most valuable interaction in the portal. |
| FR-L-05 | Request history shows status, dates, days, approver, and any rejection reason. |
| FR-L-06 | Pending requests can be withdrawn; approved future leave can be cancelled, subject to 06's rules and the period lock. |
| FR-L-07 | The team calendar is available subject to OQ-803, showing dates without leave types. |

### My pay — FR-Y

| ID | Requirement |
|---|---|
| FR-Y-01 | Published payslips only, newest first, with period and net pay (07 FR-L-05). |
| FR-Y-02 | A payslip renders its lines with the same plain-language explanations used elsewhere, and a PDF download. |
| FR-Y-03 | Month-on-month comparison is offered, highlighting what changed and by how much. |
| FR-Y-04 | Every payslip view and download is logged (07 FR-L-06), subject to 07 OQ-714. |
| FR-Y-05 | An employee with no payslips yet is told when to expect the first one, not shown an empty list. |
| FR-Y-06 | No pay figure is ever shown outside this section — not on the dashboard, not in a notification (07 D-06, 05 D-06). |

### Documents and directory — FR-X

| ID | Requirement |
|---|---|
| FR-X-01 | The employee sees their own documents with type, date, and expiry, and can download them (02's rules apply). |
| FR-X-02 | Expiring documents are flagged with what is needed. |
| FR-X-03 | Upload is permitted only against a specific request from HR, not as free-form storage (OQ-807). |
| FR-X-04 | The directory lists colleagues with name, position, department, work contact details, and photo. Never personal contact details, never sensitive fields. |
| FR-X-05 | The org chart is available from the directory, using 02's endpoint, with the phone-friendly indented list as the default at narrow widths (02's ui-ux). |
| FR-X-06 | Directory search works on name, position, and department. |

## Non-functional requirements

| ID | Requirement |
|---|---|
| NFR-01 | The dashboard is one request and renders in under a second on a mid-range phone over 4G. |
| NFR-02 | Every screen is usable at 375 px with no horizontal scrolling, and at 200% zoom. |
| NFR-03 | Interactive targets are at least 44×44 px on touch, with adequate spacing. |
| NFR-04 | The portal must work on the browsers employees actually have, including two-year-old Android devices. No dependency on the newest CSS or JS features without a fallback. |
| NFR-05 | Initial JavaScript payload is kept small: this is the one part of the system where the user is on a phone, possibly on mobile data, possibly on a plan they pay for. |
| NFR-06 | All figures — leave, attendance, pay — are fetched from the owning feature's endpoints. No number is computed in this feature's code (D-01). |
| NFR-07 | No sensitive data is cached in `localStorage` or exposed in URLs. |
| NFR-08 | Accessibility: keyboard operable, screen-reader labelled, 4.5:1 contrast, `prefers-reduced-motion` honoured. Some of the workforce will need this and will not ask. |

## Permission keys

This feature introduces **two**, and reuses everything else at `SELF` scope — which is the clearest
evidence that D-01 holds:

| Key | Meaning | Default holders |
|---|---|---|
| `profile.change_request` | Request a change to your own record | Everyone: SELF |
| `profile.change_request.decide` | Approve or reject change requests | HR: ALL |

Reused: `employee.read` (SELF), `employee.document.read` (SELF), `department.read`,
`attendance.read` (SELF), `attendance.request_correction`, `leave.read` (SELF), `leave.request`,
`leave.balance.read` (SELF), `payroll.read_own`, `notification.read`.

## Acceptance criteria (feature-level)

1. An employee with only self-service permissions signs in and lands on the portal, with no admin
   navigation visible anywhere.
2. The dashboard renders from a single request and shows nothing the employee lacks permission for.
3. An employee edits their personal phone number and it takes effect immediately; an attempt to edit
   their address raises a request instead.
4. A submitted change request notifies HR, appears with its status to the employee, and applies to
   the employee record on approval.
5. Bank details cannot be edited or requested anywhere in the portal.
6. A day marked `UNKNOWN` by a device outage reads as "The terminal wasn't working — this isn't
   counted against you", with no device name or internal status shown.
7. Requesting leave shows the cost breakdown before submission and creates the same record an HR
   admin's request would.
8. A rejected leave request shows its reason to the employee.
9. No pay figure appears on the dashboard or in any notification.
10. An employee with no payslips, no leave types configured, and no shift assigned sees three
    explanations rather than three empty boxes or errors.
11. Every screen passes a 375 px review with no horizontal scroll, and is operable by keyboard.
12. A manager sees "3 requests need you" on the dashboard, linking to feature 06's approval screen.
13. No number displayed in the portal is computed by this feature's own code.
