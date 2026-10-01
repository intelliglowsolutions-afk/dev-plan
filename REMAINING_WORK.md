# HRM System — remaining work

Written 2026-10-01, after Session 40. All eleven features are built, tested and pushed
(`hrm-system` at `47a1a5e`). This is everything still to do, drawn from the "remainders" rows and
open questions in `IMPLEMENTATION_LOG.md`. Where the two disagree, the log is the authority.

**Legend** — **You**: needs your decision, eyes or credentials. **Blocked**: build work that cannot
start until a question is answered. **Ready**: build work that can start now.

---

## 1. Decisions only you can make

These change what already exists. Until they are answered, the parts named run on placeholders.

| # | Question | What is waiting on it |
|---|---|---|
| OQ-319 | How the SenseFace terminals connect and authenticate | All of device ingestion (section 3). Attendance has no real punches until this is settled |
| OQ-701 | The real pay components and how each is calculated | Payroll is seeded with labelled examples only |
| OQ-601b | The real leave types and entitlements | Leave policies are examples only |
| OQ-150 | The suppression threshold for reports (default 5) | At 5, every pay figure is hidden in a company this small |
| OQ-151 | The rule for a day's pay (÷ working days, ÷ 30, ÷ 26?) | The `leave.liability` report |
| OQ-157 | Who may release interview feedback when one interviewer never submits | Today one silent interviewer hides everyone's feedback, from HR too |
| OQ-154 | Is the recruitment half wanted (OQ-1001), and the public form (OQ-1003)? | If "onboarding only", a "start onboarding for this employee" button is needed |
| OQ-1002 | Retention period for unsuccessful candidates — needs a qualified legal answer | A placeholder of 6 months is configured |
| OQ-901 / 902 | Does the company run formal reviews, and with ratings? | Performance was built as the full configurable mechanism |
| OQ-802 / 805 | Own phones or a shared kiosk; a second language | The portal assumes own phones, English only |
| OQ-502 | Do all employees have email addresses? | Notifications are email and in-app only |
| OQ-503 / 504 | Which SMTP service, and the "from" address | Real email delivery (section 5) |
| OQ-708 | The bank's file format | The bank export uses a generic layout |

**To confirm rather than decide** — defaults I chose and logged: OQ-141…149 (portal and performance),
OQ-152…153 (reports), OQ-155…160 (hiring and onboarding). Each is one row in the log's open-items
table; a "yes" closes it.

---

## 2. Verification owed — **You**

Nothing has been looked at in a browser. Logic is covered by 339 unit tests, 310 integration tests,
the database suite and an HTTP probe per feature; layout, focus and print are not.

In the order I would do it:

1. Employee portal at 375px wide (feature 08, acceptance 11) — home, time, leave, pay, profile.
2. The public job page and application form, at phone width.
3. A payslip: on screen, on a phone, and printed.
4. The performance review form at 375px.
5. The interview scorecard form, the pipeline board with many cards, the hire steps.
6. Sign-in, forced password change, the admin shell and navigation.
7. Everything else, feature by feature: employees and the import wizard, org chart, settings and
   holidays, roster and attendance grid, corrections, notifications and the template editor, leave
   request and calendar, pay run workspace, reports viewer, onboarding checklist editor.

For each: keyboard-only use, visible focus, dark mode if used, and nothing scrolling sideways.

---

## 3. Blocked build work

| Work | Blocked on |
|---|---|
| **Device ingestion** — the `/iclock` endpoint, the collector, quarantine of bad events, unmatched-PIN maintenance, the unknown-device panel, offline-terminal alerts | OQ-319 |
| Seeding real pay components and validating the formula language against them | OQ-701 |
| Seeding real leave policies | OQ-601b |
| `leave.liability` report, and the daily-rate function in payroll it needs | OQ-151 |
| Payslips as PDF and by email | OQ-136 |
| Salary structures by employee group | OQ-137 |
| Mid-period pay split; cut-off settlement | OQ-138, OQ-139 |
| The bank's own export format | OQ-708 |
| Photos in the portal and directory | OQ-142 |
| Document upload from the portal | OQ-807 |
| Peer and skip-level reviews | OQ-903 / 904 |
| Per-review access grants for a new manager | OQ-146 |
| Early release of interview feedback | OQ-157 |
| "Start onboarding" without a hire | OQ-154 |
| Delivery and bounce webhooks | OQ-503 |
| Kiosk mode; a second language | OQ-802; OQ-805 |

---

## 4. Ready build work

Grouped by feature. None of this needs an answer first.

**01 Auth and roles**
- ~~`/forgot-password` screen~~ — done 2026-10-01 (Session 41)
- Revoking a single session
- A user edit form

**02 Employees**
- History correction endpoint
- Org chart: pan, zoom, export; drag to re-parent

**03 Settings**
- SVG logos
- Twelve-month holiday grid
- Applying the configured date format across the app

**04 Attendance**
- Nothing ready beyond ingestion (section 3)

**05 Notifications**
- Inline actions in the notification list
- Snooze
- A password-changed notice on the reset-link path

**06 Leave**
- Attachments on a request (for example a sick note)
- Approval routing by leave type or length
- Minimum-staffing warning
- Compensatory time off; hours-based leave
- Fiscal-year views for HR

**07 Payroll**
- Reminder jobs
- Drag re-ordering of components

**08 Portal**
- Installable app (PWA)
- A headcount for HR of employees with no login

**09 Performance**
- Anonymous feedback aggregation
- Draft answer history
- Side-by-side self and manager review
- Drag re-ordering in the form editor

**10 Recruitment and onboarding**
- Editors for pipeline stages and scorecard forms (today: seeded examples only)
- A Reschedule button (the API exists)
- Confirmation message after a retention deletion (today the page just shows "Nothing is due")
- Copy the CV into the employee's documents on hire
- Send the login invite as part of the hire
- A list endpoint for offers
- Drag on the pipeline board
- References, background checks, e-signature, a candidate status page

**11 Reports**
- Drill-through from a figure to its rows
- Background runs and paging for large results
- PDF export
- Line and funnel charts; a pipeline-conversion report
- The backlog reports: device uptime, approval turnaround, review completion
- Fiscal-year periods
- A saved-views page

---

## 5. Going live

None of this has been started. The app runs only in a local development container.

- Production build and image; choice of host
- Production database: roles, migrations, first tenant and first super admin
- Secrets and environment configuration
- Real SMTP, with a reply-to that reaches a person
- The scheduled job runner in production (reminders, escalations, grants, digests)
- Document and CV storage on a volume that is backed up
- Database backups, and a tested restore
- HTTPS and the public hostname (the job page and terminals need one)
- Rate limiting that survives more than one app process (today it is in memory)
- Monitoring and error logging
- Loading real data: employees (CSV import exists), opening leave balances, current salaries
- A trial pay run alongside the existing process before relying on it

---

## 6. Housekeeping

- Remove `dev-stale-f10` and `dev-stale-f11` from the app container's `.next` volume.
- The dev-cache fault (new routes under a dynamic segment return a 404 page) has now happened
  four times. Worth finding the cause rather than moving the cache aside again.
- New tables without a foreign key into the existing set must be added by hand to the truncate list
  in `prisma/fixtures/canonical.sql`, or test data survives fixture reloads.
- Git is now installed on the host; the log's toolchain note still describes running it in a container.
- The two source documents the plan cites (`HRM_SYSTEM_PLANNING_INSTRUCTIONS.md`,
  `HRM_SYSTEM_DEPLOYMENT.md`) are not under `C:\Dev` (OQ-006).

---

## Suggested order

1. Push through section 2 (browser pass) — it will produce its own fix list.
2. Answer OQ-319, OQ-701 and OQ-601b; they unblock the three largest pieces.
3. Build device ingestion, then seed real pay and leave configuration.
4. `/forgot-password` and the hiring editors — small, and people will hit them early.
5. Section 5, ending with a trial pay run.
6. Everything else in section 4 as it is asked for.
