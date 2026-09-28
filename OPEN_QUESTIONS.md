# HRM System — Open Questions

> Step 5 of the planning process. Every open question raised across the eleven features, in one
> place, ordered by how much it costs to leave unanswered.

---

# ✅ ANSWERED — 2026-09-18

**The user answered every question.** Explicit answers are recorded below; everything not named
explicitly takes its proposed default, per the user's instruction "default to all the other".

Two items could not be defaulted and are still needed — see [Still needed](#still-needed).

## Answers that change the plan

These contradict a decision already written into a feature, so each needs a revision pass.
**Tracked in `IMPLEMENTATION_LOG.md` as the Session 16 rework list.**

| ID | Answer | What it overturns | Consequence |
|---|---|---|---|
| **OQ-301** | **Multi-tenant — yes** | 03 **D-02** ("one company, a singleton row, not a tenants table") | **The largest change in the plan.** Every table needs a tenant discriminator; every query, scope check, permission grant, settings read, job run and report must be tenant-scoped. Affects all 11 features' `data-model.md`. Costs roughly 10–15% more work overall — far less than retrofitting later, which is why this was Tier 1 |
| **OQ-1002 / 105 / 203 / 908** | **No automatic data deletion, anywhere** | 10 **D-02** (candidate purge job), plus retention sweeps in 01, 05, 10, 11 | Retention becomes **HR-initiated and manual**: the system reports what is due and a person decides. The purge job becomes a report. **This is a deliberate liability trade** — candidate data now accumulates indefinitely unless someone acts. Worth revisiting with a data-protection adviser |
| **OQ-101** | **NextAuth** | 01 OQ-101's proposed default (hand-rolled) | Feature 01's auth endpoints are rewritten around Auth.js v5. **01 D-04 survives** — Auth.js supports database sessions via an adapter, so immediate revocation is still available; JWT sessions must be avoided. Next.js 16 compatibility must be verified early (`AGENTS.md`) |
| **OQ-709** | **Payslips by portal *and* email PDF** | 07 FR-L-05 and 05 **D-06** ("no sensitive data in email bodies") | Emailed payslips leave the system's access control entirely — forwarded, archived on third-party servers, readable on an unlocked phone. Proceeding as asked, with two mitigations to confirm: password-protected PDFs (employee code or DOB), and the email body itself carrying no figures |
| **OQ-206** | **Multiple locations — yes** | 02 OQ-206's default (single location) | `Location` becomes a model; employees, devices, holiday calendars and reports gain a location dimension. Affects 02, 03, 04, 11 |
| **OQ-606 / 407** | **Multi-step, configurable approval chains** (leave *and* attendance corrections) | 06's proposed single-step default; 04's manager-approves default | 06's `approvalSteps` JSON becomes a real configurable chain rather than a placeholder; 04 gains the same mechanism. Both are a meaningful step up from the planned default |
| **OQ-803 / 508 / 909 / 910 / 809** | Employees **cannot** see colleagues' leave; managers **can** see notification read state; no "who read my review"; no draft history; rejected change-request values not kept | 06 FR-C-01, 08 OQ-803 (own department), 05 OQ-508 (no) | Team calendar becomes HR/manager-only. Read state is a small surveillance surface — flagged, not blocked |

## What the manual settled (OQ-000 resolved)

The SenseFace 2A manuals were found in `C:\Dev\Datasheets` and read on 2026-09-18 (text extracted
with a stdlib script — the PDFs are CID-encoded and no PDF tooling is installed).

| ID | Finding | Effect on the plan |
|---|---|---|
| **OQ-204** | *"The user ID may contain **1 to 14 digits by default, supporting both numbers and alphabetic characters**"*, and *"during the initial registration, you can modify your ID, but not after registration"* | The plan assumed **numeric, ≤ 9 digits**. `employeeCode` is actually **1–14 alphanumeric**. 02 FR-E-02's validation must change. The device enforcing immutability after registration independently confirms 02's decision to make the code immutable |
| **OQ-307** | **Cloud Server Settings contains only: Enable Domain Name, Server Address, Server Port, Enable Proxy Server.** There is **no comm-key field** — "Comm Key" appears nowhere in either manual | **Breaks 03 D-08** (per-device comm keys, hashed, shown once). The device cannot present a shared secret through this screen, so the push endpoint cannot be authenticated the way the plan assumed. See below |
| **OQ-312** | Confirmed: the protocol surface described does not expose firmware version, device timezone, or clock drift | Matches the user's answer ("312 – no"). Those speculative columns come out of 03's `Device` model |
| — | HTTPS can be enabled on the device | The one real transport protection available; should be used |

### The device-authentication problem

03 D-08 assumed each device would hold its own comm key. It cannot. What is actually available to
authenticate a push is: **the serial number** (which the device sends, and which is not a secret),
**the network path**, and **HTTPS**.

That leaves the ingestion endpoint meaningfully weaker than planned. Anyone who can reach port 8081
and knows or guesses a registered serial number can inject attendance records. Mitigations to decide
between — this is a **new open question, OQ-318**:

1. **Network isolation** — bind the device port to the LAN only, firewall it from everything else.
   Probably sufficient for a single-site deployment and costs nothing.
2. **HTTPS plus IP allowlist** per registered device, since terminals have static addresses.
3. **A reverse proxy in front** that adds authentication the device cannot (mTLS, an IP-based rule).

03's serial allowlist, unknown-device logging, and per-device disable all still work and still
matter — they are just no longer backed by a secret. `Device.commKeyHash` and the "shown once" flow
should be removed rather than left as security theatre.

## Answers that confirm the plan as written

`OQ-000` manual located · `OQ-201` union of both · `OQ-302` one timezone · `OQ-702` configurable tax
rules (already the design — bracket tables, no built-in rules) · `OQ-802` employees have access ·
`OQ-805` English only · `OQ-1105` no external BI · `OQ-401` night shifts yes · `OQ-S-01` fixed **and**
rotating, configurable · `OQ-403` fixed break deduction · `OQ-405` overtime approved · `OQ-406`
request + approval · `OQ-404` 10 min grace · `OQ-408` lock after approval, drafts editable, edits
logged · `OQ-602` yearly accrual · `OQ-604` 5 days, expiring 31 March · `OQ-607` delegation +
escalation · `OQ-609` encashment on termination · `OQ-603` days with half-days · `OQ-703/704/705/707`
defaults · `OQ-708` CSV · `OQ-901` annual reviews · `OQ-1003` both intake routes · `OQ-613` ledger
append-only at database level · `OQ-615` cancelling past leave needs approval · `OQ-214` completeness
indicator · `OQ-415` re-sync pull supported · `OQ-903` HR can grant history access · `OQ-102` MFA
yes · `OQ-106` multiple roles · `OQ-109` shared rate-limit storage · `OQ-312` protocol does not
expose firmware/timezone/drift · `OQ-402` one or two devices · `OQ-410` recompute after leave lands.

`OQ-108` is a small change: **no "remember this device"; automatic logout enforced.**

## Sequencing

**OQ-1001 — recruitment is wanted, but built last.** Feature 10 moves to the end of the build order,
after 11. Onboarding is the half that earns its place at any hiring volume.

## Still needed

Neither can be defaulted — there is no sensible default to fall back on.

| ID | What is still needed | Blocks |
|---|---|---|
| **OQ-701** | **The actual pay components and how each is calculated** — basic, allowances, deductions, and the tax structure. "Configurable tax rules" (OQ-702) settles the mechanism but not the content | Feature 07 cannot be seeded or tested. The formula language cannot be validated until real formulas exist |
| **OQ-601** | Leave types are settled (**Annual, Casual, Medical**) but **not their entitlements** — how many days each, per year, and whether they differ by grade or employment type | Feature 06's policies cannot be seeded |

One answer was ambiguous: **OQ-416** was given twice, as "default" and as "yes". The default is *no
future-dated attendance rows*; "yes" would create them. Taking the default — say if you meant
otherwise.

---

## How to use this (original, for reference)

Three tiers:

- **Tier 1 — Blocking (12).** Answer before building. Each one shapes code that other code sits on,
  or cannot be guessed responsibly.
- **Tier 2 — Shapes the build (23).** Answer before the feature they belong to. Wrong guesses here
  mean rework, not a rewrite.
- **Tier 3 — Defaults proposed (~140).** Each has a sensible default already written into the plan.
  Skim for anything that looks wrong; ignore the rest.

Two questions need someone outside this project — they are flagged **⚖ external**.

The fastest useful pass: answer Tier 1, skim Tier 2 for the features you intend to build first, and
ignore Tier 3 until something surprises you.

---

## Tier 1 — Blocking

| ID | Question | Why it blocks | Proposed default | Who answers |
|---|---|---|---|---|
| **OQ-000** | The ZKTeco SenseFace 2A manual — can it be obtained? | The push protocol's paths, record format, field order, and comm-key mechanism are unverified. Blocks 04's ingestion and 03's device auth. **Everything else can proceed** — build 04's engine against synthetic punches and ingestion last | Proceed without it; isolate protocol assumptions behind one adapter | You / the device vendor |
| **OQ-201** | Does `DEPARTMENT` scope mean *the actor's transitive reports*, *everyone in the department they head*, or the union? | On the hot path of every scoped query in every feature. A department head who is not everyone's line manager sees materially less under the first reading | The union of both | Company |
| **OQ-301** | Will this ever serve more than one company, or legally separate entities? | Retrofitting multi-tenancy after eleven features is close to a rewrite; designing for it now costs ~10% more. The cheapest decision on this list to make, and the most expensive to defer | Single company | Company |
| **OQ-101** | Auth.js (NextAuth) v5, or hand-rolled sessions? | Shapes every endpoint in feature 01. `.env` already carries `NEXTAUTH_*`, but Next.js 16 compatibility is unverified | Hand-rolled — ~200 lines, no framework-version risk, and the DB-session decision removes Auth.js's main draw | Technical |
| **OQ-302** | What is the company timezone, and does anyone work in a different one? | Decides which calendar day every punch belongs to. Getting it wrong makes attendance quietly wrong by hours, every day, for everyone. Must be set before any attendance data exists | One timezone, set at install | Company |
| **OQ-601** | What leave types exist, and what is the entitlement for each? | Feature 06 is entirely mechanism without them. Nothing can be seeded or tested against reality | — | Company |
| **OQ-701** | What are the actual pay components, and how is each calculated? | Same for feature 07, and it also validates whether the formula language is sufficient | — | Company |
| **OQ-702** | ⚖ Is there statutory tax to deduct, and **who owns keeping the rates current**? | The system ships no tax rules by design. A configured table nobody reviews produces confidently wrong deductions — worse than none | Configured by the company, with a visible "rates last reviewed" date | ⚖ Payroll/tax adviser |
| **OQ-802** | Do employees have smartphones and network access, or is there a shared kiosk? | A kiosk needs short sessions, PIN-gated pay, and nothing personal persisting on screen. A materially different build from the mobile-first portal planned | Personal phones | Company |
| **OQ-805** | Is a second language needed? | Feature 08 is the strongest case for i18n: HR can work in English, the whole workforce may not. Retrofitting after eleven features is expensive; the portal's copy is already written as a translatable table | English only | Company |
| **OQ-1002** | ⚖ What retention period applies to unsuccessful candidates? Is there a legal obligation? | Feature 10 deletes candidate data on a schedule. The period is jurisdiction-specific and this plan deliberately does not guess | 6 months from the final decision | ⚖ Data-protection adviser |
| **OQ-1105** | Will anyone want Excel or Power BI connected **directly to the database**? | This bypasses every permission rule in features 01–10. It is also the single most common request once reporting exists — easy to refuse now, politically hard after someone has been promised it | Refuse; offer exports and, later, a permission-respecting API | Company |

---

## Tier 2 — Shapes the build

Answer before building the feature each belongs to.

### Attendance and shifts (04)

| ID | Question | Proposed default |
|---|---|---|
| OQ-401 | Do you run night shifts, or any shift crossing midnight? | Yes — the shift-anchored day window is built in from the start |
| OQ-S-01 | Rotating shift patterns, or is everyone on one fixed shift? | Patterns supported; if not needed, the build shrinks noticeably |
| OQ-403 | Do people scan for breaks/lunch, or only start and end of day? | Start and end only; breaks are a fixed deduction |
| OQ-405 | Is overtime paid, and does it need pre-approval? | Detected and marked pending approval; payroll pays only approved |
| OQ-406 | What happens when someone forgets to scan out? | `MISSING_PUNCH`, zero hours, correction required — never an assumed departure time |
| OQ-404 | Grace periods, and is there a cumulative-lateness rule (3 lates = half day)? | 10 minutes each way; no cumulative rule in v1 |
| OQ-407 | Who approves attendance corrections — manager, HR, or both? | Manager approves; HR can override |
| OQ-408 | Does a finalised payroll run lock the period against corrections? | Yes; corrections afterwards become payroll adjustments |

### Leave (06)

| ID | Question | Proposed default |
|---|---|---|
| OQ-602 | Accrual method (annual grant / monthly / per hours) and leave year (calendar / fiscal / anniversary)? | Annual grant on the calendar year, pro-rated for joiners. **Anniversary-based is materially more work** |
| OQ-604 | Carry-over: allowed? capped? expiring when? | Allowed, capped at 5 days, expiring 31 March |
| OQ-606 | Approval chain: manager only, or manager then HR? Varies by type or length? | Single step to the manager, HR override. Conditional routing is a real step up in complexity |
| OQ-607 | What happens when the approver is themselves on leave? | Explicit delegation, plus escalation to their manager after 3 working days |
| OQ-609 | Is unused leave encashed on termination or at year-end? | On termination only |
| OQ-603 | Do shift lengths vary enough that leave must be measured in hours rather than days? | Days, with half-days |

### Payroll (07)

| ID | Question | Proposed default |
|---|---|---|
| OQ-703 | Pay frequency, period, and cut-off relative to pay date? | Monthly, cut-off on the 25th, paid on the last working day |
| OQ-704 | How is a partial month paid — calendar days, working days, or a fixed divisor? | Working days. **Changes every pro-rated payslip; the most common source of payroll disputes** |
| OQ-705 | Does attendance affect pay, and how? | Unpaid leave and unauthorised absence deduct; lateness does not; approved overtime paid at a multiplier |
| OQ-707 | Who approves a pay run, and is approval separate from finalisation? | Separate permissions; two recorded acts even when one person holds both |
| OQ-708 | What bank file format does your bank require? | Generic CSV until the real format is known — a wrong format fails at the bank, not in the system |
| OQ-709 | Payslips emailed as PDFs, or portal-only? | Portal-only; the email notification carries no figures |

### Other features

| ID | Question | Proposed default |
|---|---|---|
| OQ-502 | Do all employees have work email addresses? | Office staff do. **If a large share do not, notifications don't reach them and that changes the product** |
| OQ-901 | Do you run performance reviews — annual, twice-yearly, or continuous check-ins with no formal cycle? | If there is no cycle, feature 09 is goals plus feedback: about a third of the planned work |
| OQ-1001 | Do you hire often enough for the recruitment pipeline to be worth building? | Onboarding checklists are useful at any volume; a pipeline is not. **Consider building onboarding only** |
| OQ-1003 | Is there a public application form, or do candidates arrive by email? | Public form. Declining it removes the system's only internet-facing write surface |

---

## Tier 3 — Defaults proposed

Skim for anything that looks wrong. Silence accepts the default.

### 01 — Roles, Permissions & Auth

| ID | Question | Default |
|---|---|---|
| 102 | MFA for admin accounts in v1? | No; the model leaves room |
| 103 | Password policy — length, complexity, expiry, reuse | 12 chars minimum, no composition rules, no expiry, last 3 blocked |
| 104 | SSO (company Google/Microsoft accounts) now or later? | Password-only in v1 |
| 105 | Audit log retention | 24 months, then archive |
| 106 | May a user hold more than one role? | Yes — union of permissions, widest scope wins |
| 107 | Self-service users on the same login as admins? | Yes; landing page differs by permission |
| 108 | "Remember this device" for longer sessions? | No |
| 109 | Rate-limit storage if the app is ever scaled to two containers | In-process; revisit if scaled |
| 110 | Does the session endpoint return nav items, or does the client derive them? | Client derives from permissions |
| 111 | API versioning — `/api/` or `/api/v1/`? | Unversioned for now |
| 112 | After a password reset, sign the user in or send them to login? | Send to login |
| 113 | One dashboard with gated cards, or one per role? | Gated cards |
| 114 | Branding — logo and colours on the login screen before settings exist | Text wordmark until feature 03 |
| 115 | Language (see OQ-805) | English only |

### 02 — Employee Management

| ID | Question | Default |
|---|---|---|
| 202 | Which personal fields do you actually need (national ID, DOB, gender, marital status)? | A minimal set — **unused PII is pure liability** |
| 203 | Retention/deletion obligation for terminated employees' records | Retain indefinitely until told otherwise |
| 204 | SenseFace user-PIN format (numeric? max length?) | Numeric, up to 9 digits — blocked on OQ-000 |
| 205 | Rehires: one record with multiple employment periods, or separate records? | One record, multiple periods |
| 206 | Multiple work locations/branches? | Single location; cheaper to add now than later |
| 207 | Custom employee fields without a developer? | No |
| 208 | Document types, and any mandatory before an employee is active? | Seeded list, none mandatory |
| 209 | Where the document volume lives in Docker, and whether backups cover it | **Needs deciding before the first real upload — a DB-only backup loses every contract** |
| 210 | A `fields` parameter for partial employee responses? | No; a dedicated lookup endpoint instead |
| 211 | Async import for large files | Synchronous up to ~1 000 rows |
| 212 | Does the expiring-documents query belong to 02 or 05? | Query in 02, scheduling and delivery in 05 |
| 213 | Is the employee photo edited separately from documents? | Treated as a document |
| 214 | A profile "completeness" indicator? | No — risks nagging |

### 03 — Admin & Settings

| ID | Question | Default |
|---|---|---|
| 303 | Do different groups observe different holidays (location, religion, contract)? | One company calendar in v1; model supports more |
| 304 | Does the work week vary by group (office Mon–Fri vs factory six days)? | One company default, overridable per shift |
| 305 | Do half-day holidays exist, and what does "half" mean in hours? | Half the standard daily hours |
| 306 | How many terminals, at how many locations? | One or two, one location |
| 307 | Per-device comm key, or does the firmware only support one shared key? | Per device — blocked on OQ-000 |
| 308 | When a device was offline, is the gap absence or unknown? | **Unknown, flagged, never auto-absent** |
| 309 | Any lunar or computed-date holidays (Eid, Easter)? | Entered per year by hand |
| 310 | A read-only iCal holiday feed for employees? | Not in v1 |
| 311 | Who holds `device.write` — HR, or an IT person who shouldn't see employee data? | Grantable separately |
| 312 | Does the protocol expose firmware version, device timezone, clock drift? | Assumed yes — blocked on OQ-000 |
| 313 | Should device events be a table or a log stream? | Table, with transition-only logging and retention |
| 314 | A public settings endpoint for the login screen? | Yes, explicit allowlist |
| 315 | Where the scheduled-job runner lives (in-process, separate container, cron) | In-process; breaks quietly if scaled to two instances |
| 316 | Does the setup checklist persist after completion? | Disappears |
| 317 | How the device's server address is determined behind Docker | A setting the admin confirms once |

### 04 — Attendance & shifts

| ID | Question | Default |
|---|---|---|
| 402 | One device or several (entry/exit, multiple buildings)? | One or two; pairing ignores which |
| 409 | Different attendance rules for probation/trainees? | No |
| 410 | Recompute attendance history once leave exists? | Yes — part of feature 06's release |
| 411 | Record which device a punch came from on the daily record? | On the punch, not the day |
| 412 | Should `attendance_days` rows exist for non-working days? | Yes — ~40% more rows, simpler downstream queries |
| 413 | Reason-trail shape and size cap | Short rule codes with values, never prose |
| 414 | Will `attendance_punches` need partitioning? | Not at ~290k rows/year; decide before year three |
| 415 | Does the terminal support a "fetch records since" pull for recovery? | Unknown — would answer most device gaps |
| 416 | Create attendance rows for future dates? | Today only |
| 417 | Cap the recompute range per request? | One year |
| 418 | Auto-refresh the daily grid, and how often? | 60s during working hours |
| 419 | A live "who is in the building now" view? | Not in v1 — useful for fire roll-calls |
| 420 | Show employees their own lateness trend over months? | No |
| S-02 | Employee-requested shift swaps with approval? | HR enters overrides |
| S-03 | A rostering grid where future cells are edited directly? | No — significantly larger feature |
| S-04 | How often do shift definitions change? | Infrequently; mutable shifts with explicit recompute |
| S-05 | Can managers assign shifts for their own team? | HR only |
| S-06 | Minimum rest between shifts (e.g. 11 hours) to warn about? | No rule |

### 05 — Notifications

| ID | Question | Default |
|---|---|---|
| 501 | SMS or WhatsApp for anything? | Email and in-app only |
| 503 | SMTP: company server or a provider (SES/SendGrid/Postmark)? | Generic SMTP; a provider would add real bounce webhooks |
| 504 | The "from" address and a reply-to that reaches a human | `hr@company` as reply-to, not no-reply |
| 505 | Digest frequency and time | Daily at 08:00 |
| 506 | Templates in a second language (see OQ-805) | English only |
| 507 | Retention for in-app notifications and the delivery log | 12 months / 90 days |
| 508 | Can managers see whether their team read a notification? | No |
| 509 | Cap the size of notification context? | Yes, validated per type |
| 510 | Must `notify()` always be called inside a transaction? | Optional, documented |
| 511 | Snooze a notification ("remind me tomorrow")? | Not in v1 |
| 512 | Can any notification type be test-sent? | Yes, except those with realistic figures |
| 513 | Badge count: polling or push (SSE/websocket)? | Polling at 60s, matching the attendance grid |
| 514 | An "only things I can act on" filter? | Not in v1 |

### 06 — Leave

| ID | Question | Default |
|---|---|---|
| 605 | Can balances go negative (leave taken in advance)? | No |
| 608 | Does sick leave require a document after N days, and is it enforced? | Prompted after 2 days, not enforced |
| 610 | Comp-off / time in lieu for overtime or holiday working? | Not in v1; the ledger leaves room |
| 611 | Minimum staffing: block or warn when too many are off? | Warn, never block |
| 612 | How far back may leave be backdated? | Allowed with a reason, subject to the payroll lock |
| 613 | Make the ledger append-only at the database level (revoked UPDATE/DELETE)? | By convention; worth hardening |
| 614 | Can a manager preview leave cost for a team member? | Cost yes, balance no |
| 615 | Should cancelling past leave require approval? | No |
| 616 | Should an unsettled leave balance block a termination in feature 02? | No |
| 617 | Can employees see leave for people outside their department? | Own department only |
| 618 | Draft leave requests before submitting? | Withdraw-after-submit covers it |

### 07 — Payroll

| ID | Question | Default |
|---|---|---|
| 706 | Loans or advances recovered over several periods? | Recurring deduction only; a real loan ledger is a sub-feature |
| 710 | A 13th-month or bonus cycle? | Off-cycle runs supported; the policy is configuration |
| 711 | Accounting entries with GL codes, or a summary export? | Summary by department; GL codes modelled but optional |
| 712 | Anyone paid in another currency? | Single currency |
| 713 | Compress or externalise the payslip input snapshot? | No — ~1 MB/year at this size |
| 714 | Log an employee viewing their *own* payslip? | No — doubles volume, answers nothing |
| 715 | Must a pay run be resumable across a restart? | Re-running is fine at 200 employees |
| 716 | Variance compared with the previous period or a rolling average? | Previous period |
| 717 | Should employees see the payslip calculation trail? | Yes — the alternative is HR explaining the arithmetic by hand |
| 718 | A "show me 10 random payslips in full" sampling view for the approver? | Not in v1; common in payroll practice |

### 08 — Self-service

| ID | Question | Default |
|---|---|---|
| 801 | Which fields can employees change themselves vs request vs never? | Self: phone, personal email, emergency contacts, photo. Request: name, address, marital status, national ID. **Never: bank details** |
| 803 | Can employees see colleagues' leave? | Own department, dates without types |
| 804 | How much attendance detail — records only, or totals and trends? | Record plus this month's totals; no trends, no comparisons |
| 806 | Do managers use the admin shell, or only the portal? | Admin shell, with dashboard prompts as a bridge |
| 807 | Can employees upload documents freely? | Only against a specific request |
| 808 | A company announcements noticeboard? | Not in v1 |
| 809 | Keep a rejected change request's proposed value visible? | Yes |
| 810 | Cache the dashboard aggregate per user? | Not in v1 |
| 811 | `GET /api/directory` belongs to feature 02 — add it there? | Yes, retrospectively |
| 812 | Installable as a PWA (home-screen icon)? | Not in v1; markedly changes how often people open it |

### 09 — Performance

| ID | Question | Default |
|---|---|---|
| 902 | What rating scale, if any? | Configurable; **"no rating at all" is a supported mode, not an empty scale** |
| 903 | On a manager change, does the new manager see past reviews? | No — goals and the current cycle only; HR can grant more |
| 904 | Peer / 360° reviews, and anonymous? | Not in v1; anonymity only above a response threshold |
| 905 | Does a rating feed a pay decision, and should the system show that link? | No system link; a human decides in payroll |
| 906 | Who sees an employee's goals — them and their manager, or the team? | Employee and manager; team visibility per goal |
| 907 | A formal performance-improvement-plan workflow? | Out of scope — legally sensitive, worse done badly |
| 908 | Retention of reviews, especially for former employees | Employment plus two years, then reviewed |
| 909 | Can employees see who read their review? | Not in v1; the log exists |
| 910 | Keep draft history so a deleted paragraph can be recovered? | **Worth deciding before the review form is built** |
| 911 | Do reviews from a previous employment period show after a rehire? | Open |
| 912 | Show the manager's review and the self-review side by side? | Open |

### 10 — Recruitment & Onboarding

| ID | Question | Default |
|---|---|---|
| 1004 | Who approves a requisition, and is approval needed? | Single named approver, skippable by configuration |
| 1005 | CV parsing or AI-assisted shortlisting? | **No** — worth confirming this is a considered no, since it will be asked for |
| 1006 | Are background or reference checks tracked? | A pipeline stage with a note, not a workflow |
| 1007 | Electronic signature of offers and contracts? | Recorded in the system; the document handled outside it |
| 1008 | Do interviewers without accounts need to fill in scorecards? | No — a tokenised link would be a second unauthenticated write surface |
| 1009 | A candidate status page? | Not in v1 |
| 1010 | Are onboarding tasks owned by people outside HR (IT, facilities) who will actually log in? | Assign to HR and the manager |
| 1011 | Should anonymised applications keep the posting link, or aggregate to counts? | Keep rows for per-role statistics |
| 1012 | Purge free-text candidate notes earlier than the rest? | Worth considering — the most sensitive, least useful to keep |
| 1013 | Is a hire reversible beyond feature 02's 24-hour window? | No |
| 1014 | Is a CV required to apply? | Yes — but requiring one excludes phone applicants |
| 1015 | Show interviewer names on pipeline cards? | No — cuts against hidden-until-submitted |

### 11 — Reports

| ID | Question | Default |
|---|---|---|
| 1101 | Which reports do you actually need? | Start with six; the machinery is cheap, each report is not |
| 1102 | Any statutory or industry reporting with a fixed format? | None assumed — a required format is a hard requirement |
| 1103 | Suppression threshold and which measures are sensitive | 5; **in a 40-person company this suppresses most department reporting — needs sanity-checking** |
| 1104 | When does live querying stop scaling? | Revisit past 2M rows or 1 000 employees |
| 1106 | PDF export and company branding? | CSV first |
| 1107 | Who sees company-wide aggregates — HR only, or department heads? | HR and super admin |
| 1108 | A turnover/attrition report? | Aggregate only; never per-individual prediction |
| 1109 | User-arranged dashboard layouts? | No — per-role composition |
| 1110 | How long to keep report-run parameters? | 180 days |
| 1111 | Include the scope note in exports as well as on screen? | Yes |
| 1112 | Chart summaries generated from data, or written per report? | Open |

### Project and environment

| ID | Question | Status |
|---|---|---|
| OQ-003 | A GitHub token was pasted into chat on 2026-09-14 and needs revoking | **Open — your action** |
| OQ-004 | The stale `dev-plan/` copy committed inside `hrm-system` | Open — this copy is authoritative |
| OQ-005 | `git` is not on PATH on this machine; nothing can be committed or pushed from here | Open — your action |
| OQ-006 | The two source docs (`HRM_SYSTEM_PLANNING_INSTRUCTIONS.md`, `HRM_SYSTEM_DEPLOYMENT.md`) are missing from `C:\Dev` | Open |

---

## If you only answer eight things

In order of what they unlock:

1. **OQ-301** — one company, or more? *(Everything sits on this.)*
2. **OQ-201** — what does `DEPARTMENT` scope mean?
3. **OQ-302** — the company timezone.
4. **OQ-601** — leave types and entitlements.
5. **OQ-701** — pay components.
6. **OQ-802** — phones or a kiosk?
7. **OQ-805** — one language or two?
8. **OQ-1001** — do you hire often enough to need a recruitment pipeline, or just onboarding?

Answers 1–3 let features 01–04 be built. Answers 4–5 let 06–07 be built. Answers 6–8 decide how much
of 08–10 is worth building at all.

---

## Already resolved

| ID | Question | Resolution |
|---|---|---|
| OQ-001 | Feature list and build order | Confirmed 2026-09-11 — all 11 features |
| OQ-002 | Remote repository not connected | Resolved 2026-09-14 — `origin` → `intelliglowsolutions-afk/hrm-system` |
