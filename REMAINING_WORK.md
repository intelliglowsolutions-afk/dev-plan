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
| ~~OQ-319~~ | ~~Hosting~~ — answered: one hosted install, many companies | All of device ingestion (section 3). Attendance has no real punches until this is settled |
| OQ-701 | The real pay components and how each is calculated | Payroll is seeded with labelled examples only |
| OQ-601b | The real leave types and entitlements | Leave policies are examples only |
| ~~OQ-150~~ | ~~Suppression threshold~~ — answered: keep 5 | At 5, every pay figure is hidden in a company this small |
| ~~OQ-151~~ | ~~A day's pay~~ — answered: ÷ 30 | The `leave.liability` report |
| ~~OQ-157~~ | ~~Feedback release~~ — answered: HR, with a reason | Today one silent interviewer hides everyone's feedback, from HR too |
| ~~OQ-154~~ | ~~Hiring and public form~~ — answered: hiring yes, form no | If "onboarding only", a "start onboarding for this employee" button is needed |
| OQ-1002 | Retention period for unsuccessful candidates — needs a qualified legal answer | A placeholder of 6 months is configured |
| ~~OQ-901 / 902~~ | ~~Reviews~~ — answered: formal reviews, no ratings | Performance was built as the full configurable mechanism |
| OQ-802 / 805 | Own phones or a shared kiosk; a second language | The portal assumes own phones, English only |
| ~~OQ-502~~ | ~~Email addresses~~ — answered: office staff only | Notifications are email and in-app only |
| OQ-503 / 504 | An email provider (answered) — **which one**, and the "from" address | Real email delivery (section 5) |
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
| Seeding real pay components and validating the formula language against them | OQ-701 |
| Seeding real leave policies | OQ-601b |
| Payslips as PDF and by email | OQ-136 |
| Salary structures by employee group | OQ-137 |
| Mid-period pay split; cut-off settlement | OQ-138, OQ-139 |
| The bank's own export format | OQ-708 |
| Photos in the portal and directory | OQ-142 |
| Document upload from the portal | OQ-807 |
| Peer and skip-level reviews | OQ-903 / 904 |
| Per-review access grants for a new manager | OQ-146 |
| "Start onboarding" without a hire | OQ-154 |
| Delivery and bounce webhooks | OQ-503 |
| Kiosk mode; a second language | OQ-802; OQ-805 |

---

## 4. Ready build work

Grouped by feature. None of this needs an answer first.

**Unlocked by the answers of 2026-10-05**
- ~~Public form off by default~~, ~~early feedback release~~, ~~daily rate and the leave value report~~,
  ~~example review forms without ratings~~ — done 2026-10-05 (Session 60)
- ~~Device ingestion, hosted design~~ — done 2026-10-05 (Session 63): collector, forward endpoint, quarantine,
  held-punch screen, direct mode (off). **With the real terminal:** switch it to TA push, point it at a
  collector (collector/README.md). ~~An alert when a collector goes silent~~ — done (Session 64)

**01 Auth and roles**
- ~~`/forgot-password` screen~~ — done 2026-10-01 (Session 41)
- ~~Revoking a single session~~, ~~a user edit form~~ — done 2026-10-01 (Session 44). Feature 01
  has no remainders left.

**02 Employees**
- ~~History correction endpoint~~, ~~org chart pan, zoom and print-to-PDF~~, ~~drag to re-parent
  departments~~ — done 2026-10-02 (Session 45)
- ~~Org chart PNG export~~ — done 2026-10-05 (Session 57) with `html-to-image`, added with your
  approval. The image itself is checked in your browser pass
- ~~The daily job that applies scheduled job changes~~ — done 2026-10-02 (Session 53); the
  first-read path stays as a fallback

**03 Settings**
- ~~SVG logos~~, ~~twelve-month holiday grid~~ — done 2026-10-02 (Session 46)
- ~~Date and time format in the remaining places~~ — done 2026-10-05 (Session 56): stamps, clocks,
  notices and export headers follow the setting and company time. Dates in words and the data
  columns of exports stay as they are, on purpose

**04 Attendance**
- Nothing ready beyond ingestion (section 3)

**05 Notifications**
- ~~Inline actions in the notification list~~, ~~snooze~~, ~~a password-changed notice on the
  reset-link path~~, ~~a test-send policy~~ — done 2026-10-02 (Session 47)
- ~~Inline decisions for employee change requests~~ — done 2026-10-02 (Session 54), with the old
  and new value shown. Pay-run approval and arrears review deliberately stay on their own
  screens (OQ-171)
- Delivery and bounce webhooks — blocked on the choice of mail service (OQ-503)

**06 Leave**
- ~~Attachments on a request~~, ~~approval routing by length~~ (by type already existed, per
  policy), ~~minimum-staffing warning~~ — done 2026-10-02 (Session 48)
- Compensatory time off; hours-based leave — each is a sub-feature of its own and waits on a
  decision (OQ-610, OQ-603)
- ~~Fiscal-year views for HR~~ — done 2026-10-02 (Session 53): the balances table reads each
  type in its own leave year (it used the calendar year for everyone), and can open past years

**07 Payroll**
- ~~Reminder jobs~~ (run due, approval waiting, rates not reviewed), ~~re-ordering components~~
  (Move up / Move down) — done 2026-10-02 (Session 49)

**08 Portal**
- ~~Installable app~~ (home-screen install, nothing kept offline), ~~a count for HR of employees
  with no login~~ — done 2026-10-02 (Session 50)

**09 Performance**
- ~~Side-by-side self and manager review~~ — done 2026-10-02 (Session 50)
- ~~Re-ordering in the form editor~~ — was already there (Move up / Move down, Session 39)
- ~~Draft answer history~~ — decided against in the data model; not owed
- Anonymous feedback aggregation — **blocked** on peer reviews (OQ-904)

**10 Recruitment and onboarding**
- ~~Editors for pipeline stages and scorecard forms~~ — done 2026-10-01 (Session 42)
- ~~A Reschedule button~~, ~~confirmation after a retention deletion~~, ~~copy the CV into the
  employee's documents on hire~~ — done 2026-10-01 (Session 43)
- ~~Send the login invite as part of the hire~~, ~~a list of offers~~ (endpoint and page), ~~drag on
  the pipeline board~~ — done 2026-10-02 (Session 52)
- References, background checks, e-signature, a candidate status page — no requirement written
  for any of them yet; each needs a decision that it is wanted

**11 Reports**
- ~~Drill-through from a row to its records~~, ~~background runs~~, ~~paging~~ — done 2026-10-02
  (Session 55). **Every ready item in this section is now done or struck with a reason.**
- ~~A funnel chart and a pipeline-conversion report~~, ~~approval turnaround~~, ~~review completion~~,
  ~~fiscal-year periods~~, ~~print or save as PDF~~ — done 2026-10-02 (Session 51)
- Line charts — with the first report that is a series over time
- A branded, server-made PDF — waits on OQ-1106 and would need a package
- Device uptime — **blocked** on ingestion (04): nothing records when a terminal was reachable
- ~~A saved-views page~~ — done 2026-10-05 (Session 56), with stale views flagged

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
- ~~Review `npm audit`~~ — done 2026-10-05 (Session 58): Next, Vitest and Prisma's tooling updated.
  Left: one `braces` chain inside the lint tool, with no fix released — recheck before going live
- Loading real data: employees (CSV import exists), opening leave balances, current salaries
- A trial pay run alongside the existing process before relying on it

---

## 6. Housekeeping

- ~~Tests shared the app's database~~ — fixed 2026-10-05 (Session 61): tests use `hrm_test`;
  `scripts/dev-reset.ps1` resets the dev database with working sign-ins
- Remove `dev-stale-f10`, `dev-stale-f10b`, `dev-stale-f10c` and `dev-stale-f11` from the app
  container's `.next` volume (old dev caches moved aside; nothing uses them).
- ~~The dev-cache fault~~ — addressed in Session 43: `npm run dev` now empties the dev server's
  compile cache at every start. Watch that it holds.
- Docker Desktop's VM has 3.7 GB; the dev server reaches about 2 GB after a full probe. Giving
  Docker more memory would make the hangs less likely.
- New tables without a foreign key into the existing set must be added by hand to the truncate list
  in `prisma/fixtures/canonical.sql`, or test data survives fixture reloads.
- Git is now installed on the host; the log's toolchain note still describes running it in a container.
- ~~The two source documents~~ — found in `C:\Dev\zkt` (Session 62)

---

## Suggested order

1. Push through section 2 (browser pass) — it will produce its own fix list.
2. Answer OQ-319, OQ-701 and OQ-601b; they unblock the three largest pieces.
3. Build device ingestion, then seed real pay and leave configuration.
4. `/forgot-password` and the hiring editors — small, and people will hit them early.
5. Section 5, ending with a trial pay run.
6. Everything else in section 4 as it is asked for.
