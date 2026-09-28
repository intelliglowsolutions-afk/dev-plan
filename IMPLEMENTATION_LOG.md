# HRM System — Implementation Log

Single, unified log of all planning and build work on the HRM System. **Every working session must append an entry before it ends** — newest entry at the top of the "Session entries" section.

Source documents:
- `HRM_SYSTEM_PLANNING_INSTRUCTIONS.md` — planning process (steps 0–6)
- `HRM_SYSTEM_DEPLOYMENT.md` — stack, Docker setup, SenseFace push-mode notes
- ZKTeco SenseFace 2A User Manual (§10 Communication, §20 ZKBio CVAccess) — **not yet in the repo; see OQ-000**

---

## Status board

### Planning phase (per `HRM_SYSTEM_PLANNING_INSTRUCTIONS.md`)

| Step | Item | Status | Notes |
|---|---|---|---|
| 0 | Create `dev-plan/` | ✅ Done | 2026-09-11 |
| 1 | Draft feature list → user confirms → set build order | ✅ Done | Confirmed 2026-09-11: all 11 features |
| 2 | Create numbered feature subfolders + `00-overview.md` | ✅ Done | Overview is a skeleton until Step 4 |
| 3 | Five files per feature (README, requirements, data-model, api-design, ui-ux) | ✅ Done | All 11 features, 2026-09-15. 56 files (04 has a sixth, `shifts.md`) |
| 4 | Root overview (`00-overview.md`) | ✅ Done | 2026-09-15 — summary, shared data model, the nine recurring patterns, cross-feature contracts, cross-cutting concerns, migration sequence, known gaps |
| 5 | Final Open Questions pass, shared with user | ✅ Done | 2026-09-15 asked · **2026-09-18 answered.** All 175 resolved; 2 still needed (OQ-701, OQ-601 entitlements) |
| 5b | **Revise feature files for the 7 overturning answers** | ✅ Done | 2026-09-18. Two cross-cutting docs (`MULTI-TENANCY.md`, `DEVICE-INGESTION-SECURITY.md`) plus revision notes and targeted edits across all 11 features. Data-model *detail* (every `tenantId` column and composite index spelled out per model) is deferred to build time, governed by the cross-cutting docs |
| 6 | Definition of done checklist | ✅ Done | 2026-09-18 — `DEFINITION_OF_DONE.md`. **Planning is complete** |

### Per-feature planning progress

| # | Feature | Priority | README | requirements | data-model | api-design | ui-ux | Open Qs |
|---|---|---|---|---|---|---|---|---|
| 01 | Roles, Permissions & Auth | Must | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ OQ-101…115 |
| 02 | Employee Management | Must | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ OQ-201…214 |
| 03 | Admin & Settings | Must | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ OQ-301…317 |
| 04 | Attendance Tracking (+ shifts) | Must | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ OQ-401…420, OQ-S-01…06 |
| 05 | Notifications | Must | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ OQ-501…514 |
| 06 | Leave Management | Must | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ OQ-601…618 |
| 07 | Payroll (generic) | Must | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ OQ-701…718 |
| 08 | Employee Self-Service Portal | Must | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ OQ-801…812 |
| 09 | Performance Management | Nice | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ OQ-901…912 |
| 10 | Recruitment & Onboarding | Nice | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ OQ-1001…1015 |
| 11 | Reports & Analytics | Should | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ OQ-1101…1112 |

### Build phase

**Started 2026-09-18.**

| Item | Status |
|---|---|
| Foundation Prisma schema (tenancy + 01 + 02 + Location) | ✅ Written and **validated** |
| Migrations (enum values → foundation → RLS) | ✅ **Applied to a live database** |
| Database roles (`prisma/roles.sql`) | ✅ Created — `hrm_app` (RLS enforced), `hrm_auth` (sign-in only), `hrm_user` (owner/migrations) |
| Tenant isolation | ✅ **Verified end to end**: no context → 0 rows; per-tenant context → only that tenant; cross-tenant INSERT refused |
| Test harness — canonical fixture, two tenants, injected clock | ✅ Built |
| Database test suite (`npm run test:db`) | ✅ **25 assertions passing** |
| Unit test suite — Vitest (`npm test`) | ✅ **150 tests passing** |
| Integration suite — app code vs real DB (`npm run test:integration`) | ✅ **88 tests passing**, mutation-checked |
| Typecheck (`npx tsc --noEmit`) | ✅ **clean** (run `next typegen` first when routes change) |
| **Stack proven over HTTP** — `next dev`, Auth.js sign-in, forced change, revocation, scoping | ✅ 2026-09-28 |
| 01/02 **contract layer** — permission catalogue, authorization, scope, audit | ✅ Built and tested |
| 01 auth — Auth.js wiring, password hashing, lockout, sessions | ✅ Built and tested |
| Seed — permissions, system roles per tenant, first super admin | ✅ Built, idempotent, verified |
| `protectedRoute` wrapper + `GET /api/employees` | ✅ Built, typechecks |
| **Version control** | ✅ `hrm-system` 10 commits, **7 unpushed**; `dev-plan` a repo with no remote. **Pushing needs your credentials** |
| Sign-in and forced change-password screens (UI, skill-grounded) | ✅ 2026-09-28 |
| **Feature 01 API** — users, roles, permissions, audit log, invite/reset, audited sign-in | ✅ 2026-09-28 |
| **Feature 01 screens** — shell, users, invite, user detail, roles + matrix, audit log, reset/invite, /403 | ✅ 2026-09-28 (visual check in a browser still owed — see Session 30) |
| Feature 01 remainders — /forgot-password (needs 05), per-session revoke, user edit form | ⬜ Deferred, listed in Session 30 |
| Remaining 02 route handlers and screens | ⬜ Step 5 |

**Toolchain on this machine:** no Node, npm, or git — but **Docker works**, so the toolchain runs
in containers (`docker run --rm -v C:\Dev\hrm-system:/app node:20-alpine …`), and git runs as
`alpine/git` against the bind-mounted repo. The app runs with
`docker compose up -d --build app` (dev target, hot reload). **New files are not always picked up by
the container's watcher on the Windows bind mount** — if Turbopack reports "Module not found" for a
file that exists, `docker restart hrm-system-app-1`. The host's `node_modules/.bin/next` is a broken
macOS symlink: call `node node_modules/next/dist/bin/next …` instead.

**Planning completed 2026-09-18.** Prior to the above, build had not started. The gate before starting each feature is
Part 3 of `DEFINITION_OF_DONE.md`; three questions still block work (OQ-319, OQ-701, OQ-601b), and
the recommended first step is the testing-strategy document, which is the plan's largest remaining
gap.

---

## Decisions

| Date | Decision | Source |
|---|---|---|
| 2026-09-11 | All 11 features are in scope for this version; build order as in the table above. | User (Step 1) |
| 2026-09-11 | Shift/work-schedule management is part of 04 Attendance, as a split file (`shifts.md`). | User (Step 1) |
| 2026-09-11 | Payroll is generic and configurable (allowances/deductions); no built-in country statutory rules. | User (Step 1) |
| 2026-09-11 | Performance and Recruitment marked nice-to-have; Reports marked should-have. Rest must-have. | Proposed; user accepted the list as-is |
| 2026-09-15 | `C:\Dev\dev-plan` (standalone, outside the app repo) is the authoritative copy of the plan. The `dev-plan/` folder committed inside `hrm-system` on 2026-09-14 is stale. | User (Session 3) |
| 2026-09-18 | **The system is multi-tenant** (OQ-301). Supersedes 03 D-02. Every model gains a tenant discriminator; every query, scope check, settings read, job and report becomes tenant-scoped. | User |
| 2026-09-18 | **No automatic data deletion anywhere** (OQ-1002, 105, 203, 908). Retention is tracked and reported; deletion is HR-initiated. Revises 10 D-02 and removes the retention sweeps from 01, 05, 10, 11. | User |
| 2026-09-18 | **NextAuth (Auth.js v5)** for authentication (OQ-101), keeping database sessions so 01 D-04's immediate revocation survives. | User |
| 2026-09-18 | **Multiple locations** (OQ-206); **payslips delivered by portal and emailed PDF** (OQ-709, overriding 05 D-06 for this case); **multi-step configurable approval chains** for leave and attendance corrections (OQ-606, 407); **employees cannot see colleagues' leave** (OQ-803); **managers can see notification read state** (OQ-508); **recruitment built last** (OQ-1001). | User |
| 2026-09-18 | **03 D-08b — device ingestion is authenticated by a per-site collector**, not by the device. The collector speaks iClock on the LAN, buffers, and forwards over HTTPS with a platform-issued credential that is hashed, rotatable and revocable. The credential — not the serial number — resolves the tenant. Network tunnel offered as an equal-strength alternative; source-IP pinning, punch quarantine, rate limits and alerting apply in every deployment. Resolves OQ-318. | `DEVICE-INGESTION-SECURITY.md` |
| 2026-09-18 | **03 D-08 withdrawn — the SenseFace 2A has no comm-key field** (OQ-307, confirmed from the manual). Device pushes cannot be authenticated by a shared secret; replacement protection is OQ-318. `employeeCode` is 1–14 alphanumeric, not numeric ≤ 9 (OQ-204). | Manual, `C:\Dev\Datasheets` |
| 2026-09-15 | Feature 11 D-01…D-10: reports compute nothing (every figure comes from the owning feature's own query); every report inherits the actor's scope and the module's permission, with `report.read` granting nothing alone; aggregates over sensitive measures suppressed below a minimum population, including defeating differencing; reports declared in code, no ad-hoc query builder; every result carries as-of time, filters and sources; historical reports use historical org structure; query live with no pre-aggregation in v1; scheduled reports deliver a link, never data; no scoring, ranking or prediction about individuals; exports audited as disclosures. | Proposed in `11-reports-analytics/README.md`; needs user review |
| 2026-09-15 | Feature 10 D-01…D-10: a candidate is not an employee (separate model, linked at hire); **candidate data is deleted on a schedule unless hired or opted in** — the inverse of 02's never-delete rule; pipeline stages configurable per requisition; interview scorecards hidden from other interviewers until submitted; hiring is a previewed conversion calling 02's creation path; onboarding tasks from a template with offsets relative to start date; no automated screening, scoring or ranking; the public application form is treated as hostile input; rejection stores an internal reason and the sent message as separate fields; onboarding runs against the employee record created at offer acceptance with a future start date. | Proposed in `10-recruitment-onboarding/README.md`; needs user review |
| 2026-09-15 | Feature 09 D-01…D-10: no built-in performance methodology (cycles, forms, scales all configurable); every review and comment has an explicit visibility state and nothing is visible until deliberately shared; goals are jointly owned with attribution; **no automatic scoring from attendance/leave/payroll — and no read path exists to make it possible**; feedback attributable by default with anonymity only above a response threshold; acknowledged reviews are immutable; ratings snapshot their label and definition; access is narrow and content reads are logged; reviews follow the person, not the reporting line; calibration modelled but not built. | Proposed in `09-performance-management/README.md`; needs user review |
| 2026-09-15 | Feature 08 D-01…D-09: no new domain logic (every screen calls an existing feature's endpoint at SELF scope); mobile-first at 375px; employees request profile changes rather than editing, per configurable field policy; same app/login/session as admin with routing by permission; plain language with no internal vocabulary; show nothing the employee cannot act on; answer first with derivation one tap away; the portal is useless without a user account and that gap is surfaced to HR; sections degrade with explanations when their feature is unconfigured. | Proposed in `08-employee-self-service/README.md`; needs user review |
| 2026-09-15 | Feature 07 D-01…D-10: no statutory rules built in (tax is a configurable component); formulas are a restricted expression language, not code; everything is snapshotted at calculation so a finalised payslip references nothing live; explicit run lifecycle where finalisation locks the period; money is `Decimal` with declared rounding and a reconciliation check; salary has its own permission set and its own access log; post-finalisation corrections are adjustments in a later run; calculation is pure with a per-line trail; compensation is date-ranged history; per-employee calculation with isolated failures. | Proposed in `07-payroll/README.md`; needs user review |
| 2026-09-15 | Feature 06 D-01…D-10: balances are a ledger, never a stored counter; types and policies fully configurable with no statutory rules; leave measured in days with half-days; cost computed from working days at submission and re-checked at approval; submitting reserves the days; the approval chain is resolved and stored at submission; approved requests are never edited; accrual/year-end are explicit, idempotent, previewable runs; leave never writes to attendance (it marks days dirty); a holiday declared over approved leave does not auto-refund. | Proposed in `06-leave-management/README.md`; needs user review |
| 2026-09-15 | Feature 05 D-01…D-10: code-declared notification type catalogue; nothing sent inline (outbox + worker); in-app always created, email optional; recipients resolved by rule (relationship/permission), never stored address lists; every notification carries a caller-supplied idempotency key; no sensitive data in email bodies; mandatory types cannot be silenced; informational types digest by default; failed delivery is visible and bounces suppress; templates editable with a shipped default and reset. | Proposed in `05-notifications/README.md`; needs user review |
| 2026-09-15 | Workspace `C:\Dev\CLAUDE.md` took effect this session: UI/UX work must be grounded in the installed skills rather than written from memory. Applied to `05-notifications/ui-ux.md`, which now cites the specific `ui-ux-pro-max` guidelines it follows. Python is not installed, so the skill's `search.py` is unusable — its CSV data was queried directly with Grep, as the workspace file instructs. | `C:\Dev\CLAUDE.md` |
| 2026-09-15 | Feature 04 D-01…D-10: raw punches and computed days are separate tables; punches immutable and never deleted; the daily record is fully recomputable and idempotent; corrections are an overlay re-applied after every recompute; a punch belongs to a shift-anchored day (not a calendar day); default pairing is first-in/last-out; unmatched PINs are queued not dropped; **a device gap yields `UNKNOWN`, never `ABSENT`**; leave and holidays classify a day without deleting its punches; computation runs async off a dirty-day queue. | Proposed in `04-attendance-tracking/README.md`; needs user review |
| 2026-09-15 | Feature 04 shifts D-S-01…D-S-08: shifts store times-of-day not instants; midnight crossing is inferred; assignments are date-ranged and non-overlapping; the roster is resolved on demand, not materialised; no assignment falls back to the company work week; breaks are a fixed unpaid deduction in v1; grace and overtime thresholds are per shift; the pattern cycle anchor lives on the assignment so one pattern serves several offset crews. | Proposed in `04-attendance-tracking/shifts.md`; needs user review |
| 2026-09-15 | Feature 03 D-01…D-09: typed settings registry with a code-declared catalogue; single company (singleton row, no multi-tenancy); all timestamps UTC with the company timezone applied at the edges; timezone changes are change-controlled; one `isWorkingDay` helper owned here and used by 04/06/07/11; holidays live on named calendars resolved via department; devices are an explicit allowlist; per-device comm keys stored hashed and shown once; device health derived from heartbeat freshness. | Proposed in `03-admin-settings/README.md`; needs user review |
| 2026-09-15 | Feature 02 D-01…D-09: departments/positions become FK models (replacing the free-text columns), department tree + flat positions, reporting line is a direct `managerId` rather than derived from department headship, every structural change written to a dated `EmployeeAssignment` history, job structure here vs compensation in 07, sensitive PII behind `employee.read_sensitive`, documents on a mounted volume not in Postgres, employees never deleted and codes never reused, `employeeCode` constrained by the device PIN format. | Proposed in `02-employee-management/README.md`; needs user review |
| 2026-09-15 | Feature 01 D-01…D-07: `User` separate from `Employee`; permissions are a fixed seeded catalogue; scope (`ALL`/`DEPARTMENT`/`SELF`) lives on the role→permission grant; DB-backed opaque sessions rather than JWTs; server-side enforcement on every entry point; append-only audit written in the same transaction; four seeded system roles. | Proposed in `01-roles-permissions-auth/README.md`; needs user review |

---

## Open items / blockers tracked across sessions

> **The full, tiered list with proposed defaults now lives in [`OPEN_QUESTIONS.md`](./OPEN_QUESTIONS.md)**
> (Step 5, 2026-09-15). That document is the one to work from; this table remains as the
> session-by-session record of when each item was raised.

| ID | Item | Raised | Status |
|---|---|---|---|
| OQ-318 | With no comm key available on the device (OQ-307), how is the push endpoint protected? | 2026-09-18 | ✅ **Resolved 2026-09-18** — `DEVICE-INGESTION-SECURITY.md`: a per-site collector holds the credential (03 D-08b), with a network tunnel as an equal-strength alternative and IP pinning + quarantine in every deployment |
| OQ-319 | **New, and blocks ingestion:** is this one hosted installation serving many tenants, or one installation per company? It decides whether the collector is necessary or the LAN is already the trust boundary. | 2026-09-18 | Open — answer before building ingestion |
| OQ-320…323 | Collector details: API key vs mTLS · does the device's Server Address accept a path · who installs and updates the collector · buffer and alert thresholds. | 2026-09-18 | Open |
| OQ-701 | **Still needed.** The actual pay components and how each is calculated. No default is possible. | 2026-09-15 | Open — blocks feature 07 |
| OQ-601b | **Still needed.** Entitlement days for Annual / Casual / Medical, and whether they vary by grade or employment type. Types confirmed 2026-09-18. | 2026-09-18 | Open — blocks feature 06 seeding |
| OQ-000 | SenseFace 2A manual not found on the dev machine. Deployment doc's §8 summary is used for now; the manual is needed for the exact push payload shapes in Attendance `api-design.md`. | 2026-09-11 | Open |
| OQ-001 | Confirm feature list and build order (Step 1). | 2026-09-11 | ✅ Resolved 2026-09-11 |
| OQ-002 | Remote repository not yet connected: no git remote is set and `gh` is not installed. Needs the target repo URL and credentials set up by the user. | 2026-09-14 | ✅ Resolved 2026-09-14 — `gh` installed, user authenticated, `origin` → `intelliglowsolutions-afk/hrm-system` (private), `main` pushed |
| OQ-003 | A GitHub personal access token was pasted into the chat in plaintext on 2026-09-14 and must be revoked/rotated. Not used by Claude. | 2026-09-14 | Open — user action |
| OQ-004 | Two copies of the plan exist: `C:\Dev\dev-plan` (authoritative) and `hrm-system/dev-plan/` (committed 2026-09-14, now stale). The stale copy needs to be reconciled or removed. | 2026-09-15 | ✅ Resolved 2026-09-28 (Session 28) — deletion committed |
| OQ-005 | Work has moved to a Windows machine (`C:\Dev`). `git` is not on PATH in the session shell, so nothing can be committed or pushed from here; earlier sessions ran on macOS with `gh` set up. | 2026-09-15 | Committing solved (git in a container). **Pushing still needs the user's credentials** — nothing is on GitHub since 2026-09-14 |
| OQ-116 | **Argon2id vs scrypt.** 01 FR-A-02 says Argon2id; scrypt is implemented because Argon2 bindings are native modules and this builds on Alpine. One-file swap; existing hashes upgrade on next sign-in. | 2026-09-28 | Open — needs the security owner's call |
| OQ-117 | **Confirm OQ-201's consequence:** under the union answer a department head sees their own manager (Ravi sees Ayesha). Now proven over HTTP and pinned by integration tests. | 2026-09-28 | Open — confirm intended |
| OQ-119 | **Confirm the no-escalation rule** (added 2026-09-28, not in the spec): nobody may assign a role, edit a role's grants, or change/suspend/reset a user whose access exceeds their own. Without it HR_ADMIN (`user.assign_role` at ALL) can make colleagues SUPER_ADMIN and vice versa. Consequence: an HR admin cannot manage the owner's account. | 2026-09-28 | Open — confirm intended |
| OQ-120 | Invite links live 60 minutes, same as reset links (ui-ux.md). Realistic for a link handed over in person; too short once invites are emailed and read the next morning. Revisit with feature 05. | 2026-09-28 | Open — revisit with 05 |
| OQ-118 | **Device-event retention.** Does "no automatic deletion" (OQ-1002 et al.) extend to machine logs? Without a sweep or transition-only logging, one terminal writes >1M rows a year. | 2026-09-28 | Open — before feature 04 ingestion |
| OQ-006 | The two source documents the plan is built on (`HRM_SYSTEM_PLANNING_INSTRUCTIONS.md`, `HRM_SYSTEM_DEPLOYMENT.md`) are not present anywhere under `C:\Dev`. | 2026-09-15 | Open |
| OQ-101 | Auth library: Auth.js (NextAuth) v5 vs hand-rolled sessions. Plan assumes hand-rolled. | 2026-09-15 | Open — needs decision before build |
| OQ-102 | MFA for admin accounts in v1? Plan assumes no. | 2026-09-15 | Open |
| OQ-103 | Password policy: length, complexity, expiry, reuse history. Plan assumes 12 chars, no expiry, last 3 blocked. | 2026-09-15 | Open |
| OQ-104 | SSO (company Google/Microsoft accounts) now or later? Plan assumes password-only in v1. | 2026-09-15 | Open |
| OQ-105 | Audit log retention period. Plan assumes 24 months. | 2026-09-15 | Open |
| OQ-106 | May a user hold more than one role? Plan assumes yes, union of permissions, widest scope wins. | 2026-09-15 | Open |
| OQ-107 | Self-service (08) users on the same login as admins? Plan assumes yes. | 2026-09-15 | Open |
| OQ-1101 | Which reports does the company actually need? The machinery is cheap; each report is not. Plan starts with six and adds on request. | 2026-09-15 | Open |
| OQ-1103 | The suppression threshold (default 5) and which measures count as sensitive. In a 40-person company a threshold of 5 suppresses most department-level reporting — needs sanity-checking against real headcount. | 2026-09-15 | Open |
| OQ-1105 | **Will anyone want to connect Excel or Power BI directly to the database?** This bypasses every permission rule in features 01–10, and is the most common request once reporting exists. Easy to refuse now, politically hard after someone has been promised it. | 2026-09-15 | Open — worth settling early |
| OQ-1107 | Who may see company-wide aggregates — HR and super admin only, or department heads too? | 2026-09-15 | Open |
| OQ-1102, OQ-1104, OQ-1106, OQ-1108…1112 | Remaining feature-11 questions (statutory reporting formats, when live querying stops scaling, PDF export and branding, turnover reporting scope, user-arranged dashboards, run-parameter retention, scope note in exports, generated vs written chart summaries). See the feature files. | 2026-09-15 | Open |
| OQ-1002 | **What retention period applies to unsuccessful candidates, and is there a legal obligation?** This is a question for whoever advises the company on data protection — jurisdiction-specific, and the plan deliberately does not guess. A placeholder of 6 months is configured as a setting, not baked in. | 2026-09-15 | Open — needs a qualified answer, not a product decision |
| OQ-1001 | Does the company hire often enough for the recruitment half to be worth building? Onboarding checklists are useful at any volume; a pipeline is not. Consider building onboarding only. | 2026-09-15 | Open |
| OQ-1003 | Is there a public application form, or do candidates arrive by email? Without one, the system has **no internet-facing write surface at all** — a meaningful reduction in risk. | 2026-09-15 | Open |
| OQ-1004 | Who approves a requisition before a role is advertised, and is approval needed at all? | 2026-09-15 | Open |
| OQ-1005 | Any interest in CV parsing or AI-assisted shortlisting? Plan says no (D-07). Worth confirming this is a considered no rather than an assumed one, since it will be asked for. | 2026-09-15 | Open |
| OQ-1006…1015 | Remaining feature-10 questions (background/reference checks, e-signature, interviewers without accounts, candidate status page, non-HR task owners, anonymised application retention shape, candidate notes purged earlier, hire reversibility, CV required, interviewer names on cards). See the feature files. | 2026-09-15 | Open |
| OQ-901 | **Does the company actually run performance reviews, and how** — annual, twice-yearly, or continuous check-ins with no formal cycle? If there is no cycle, feature 09 is goals plus feedback, roughly a third of the planned work. | 2026-09-15 | Open — needs decision before build |
| OQ-902 | What rating scale, if any? "No rating at all" is a deliberate mode some companies choose and must be designed for, not an empty scale. | 2026-09-15 | Open |
| OQ-903 | On a manager change, should the new manager see past reviews? Plan says not automatically — goals and the current cycle only, with HR granting more. | 2026-09-15 | Open |
| OQ-905 | Does a performance rating feed a pay decision, and should the system make that link visible? Plan has no system link; a human decides in feature 07. | 2026-09-15 | Open |
| OQ-908 | Retention of performance records, especially for former employees — the clearest case in the system for a deletion policy rather than indefinite retention. Interacts with OQ-203. | 2026-09-15 | Open |
| OQ-904, OQ-906, OQ-907, OQ-909…912 | Remaining feature-09 questions (peer/360 reviews, goal visibility to team, formal improvement plans, showing who read your review, draft answer history, side-by-side self/manager review, rehire review history). See the feature files. | 2026-09-15 | Open |
| OQ-801 | Which profile fields may an employee change themselves vs request vs never? A question about who owns which facts, not a technical one. Plan seeds proposed defaults; bank details are absent from the catalogue entirely so no configuration can enable them. | 2026-09-15 | Open |
| OQ-802 | **Do employees have smartphones and network access, or is there a shared kiosk?** A kiosk needs short sessions, PIN-gated pay, and nothing personal persisting on screen — a materially different build from the mobile-first design planned. | 2026-09-15 | Open — needs decision before build |
| OQ-805 | **Second language.** Feature 08 is the strongest argument for i18n in the system: HR can work in English, the whole workforce may not. 01 OQ-115 and 05 OQ-506 both defer here. The portal's copy table is written as a table specifically so it can be translated. | 2026-09-15 | Open — needs decision before build |
| OQ-803, OQ-804, OQ-806…812 | Remaining feature-08 questions (colleagues' leave visibility, attendance detail/trends, manager area vs dashboard prompts, document upload, announcements, rejected-request values, dashboard caching, directory endpoint ownership, PWA). See the feature files. | 2026-09-15 | Open |
| OQ-701 | **What are the actual pay components, and how is each calculated?** Everything in feature 07 is mechanism; without the real components nothing can be seeded or tested, and the formula language cannot be validated against real needs. | 2026-09-15 | Open — needs decision before build |
| OQ-702 | Is there statutory tax to deduct, and **who owns keeping the configured rates current**? A stale tax table produces confidently wrong deductions — worse than none. | 2026-09-15 | Open |
| OQ-703 | Pay frequency, period, and cut-off date relative to pay date. A run before the period ends must handle days that have not happened yet. | 2026-09-15 | Open |
| OQ-704 | How a partial month is paid (calendar days / working days / fixed divisor). Changes every pro-rated payslip; the most common source of payroll disputes. | 2026-09-15 | Open |
| OQ-705 | Does attendance actually affect pay, and how? Plan assumes unpaid leave and unauthorised absence deduct, lateness does not, approved overtime is paid at a multiplier. | 2026-09-15 | Open |
| OQ-707 | Who approves a pay run, and is approval separate from finalisation? Plan keeps them as two recorded acts even when one person holds both permissions. | 2026-09-15 | Open |
| OQ-708 | What bank file format the company's bank requires. A wrong format fails at the bank, not in the system. | 2026-09-15 | Open |
| OQ-709 | Are payslips emailed as PDFs or portal-only? 05 D-06 forbids figures in email; plan is portal-only with a figure-free notification. | 2026-09-15 | Open |
| OQ-706, OQ-710…718 | Remaining feature-07 questions (loans/advances, bonus cycles, GL/accounting depth, multi-currency, snapshot storage, own-payslip access logging, resumable calculation, variance baseline, employee-visible calculation trail, approver sampling view). See the feature files. | 2026-09-15 | Open |
| OQ-601 | **What leave types does the company have, and what is the entitlement for each?** Everything else in feature 06 is mechanism; without this the policies cannot be seeded and nothing can be tested against reality. | 2026-09-15 | Open — needs decision before build |
| OQ-602 | Accrual method (annual grant / monthly / per hours worked) and the leave year (calendar / fiscal / employment anniversary). Anniversary-based accrual is materially more work. | 2026-09-15 | Open |
| OQ-604 | Carry-over: allowed? capped? expiring when? The most common source of balance disputes. | 2026-09-15 | Open |
| OQ-606 | Approval chain: manager only, or manager then HR? Does it vary by type or length? Conditional routing is a real step up in complexity. | 2026-09-15 | Open |
| OQ-607 | What happens when the approver is themselves absent — delegation, escalation, or both? Leave requests stall during exactly the period approvers are on leave. | 2026-09-15 | Open |
| OQ-609 | Is unused leave encashed on termination or at year-end? Feeds payroll. | 2026-09-15 | Open |
| OQ-603, OQ-605, OQ-608, OQ-610…618 | Remaining feature-06 questions (hours vs days, negative balances, sick-note enforcement, comp-off/time in lieu, minimum staffing block vs warn, backdating, ledger append-only at DB level, cost preview for others, cancelling past leave, settlement blocking termination, cross-department calendar visibility, draft requests). See the feature files. | 2026-09-15 | Open |
| OQ-502 | **Do all employees have work email addresses?** If a large share do not, the notification design needs a different answer for them (SMS, printed notices, supervisor relay) — and an HRM where half the workforce cannot be notified is a different product. | 2026-09-15 | Open — needs decision before build |
| OQ-501 | Is SMS/WhatsApp needed for anything? Plan is email + in-app only in v1; the model allows a new channel cheaply. | 2026-09-15 | Open |
| OQ-503 | What SMTP service — company mail server or a provider (SES/SendGrid/Postmark)? Decides whether real delivery/bounce webhooks exist or bounce handling is limited to what SMTP reports. | 2026-09-15 | Open |
| OQ-504 | The "from" address/display name, and a reply-to that reaches a human. Plan avoids no-reply for approval mail. | 2026-09-15 | Open |
| OQ-505…514 | Remaining feature-05 questions (digest timing, i18n of templates, retention, managers seeing team read state, `notify()` transaction requirement, snooze, test-send scope, context size cap, badge polling vs push, actionable-only filter). See the feature files. | 2026-09-15 | Open |
| OQ-401 | Does the company run night shifts or any shift crossing midnight? The plan builds the shift-anchored day window from the start; if there are none, the engine simplifies considerably, but adding it later touches every part of it. | 2026-09-15 | Open — needs decision before build |
| OQ-403 | Do employees scan for breaks/lunch, or only start and end of day? Decides first/last vs multi-session pairing, and measured vs assumed breaks. | 2026-09-15 | Open |
| OQ-404 | Grace periods (late / early), and whether repeated lateness has a tracked consequence (e.g. 3 lates = half day). Plan assumes 10 min each way, no cumulative rule. | 2026-09-15 | Open |
| OQ-405 | Is overtime paid, and does it need pre-approval? Plan detects it and marks it `PENDING_APPROVAL`; payroll pays only approved overtime. | 2026-09-15 | Open |
| OQ-406 | What happens when someone forgets to scan out? Plan says `MISSING_PUNCH` with zero hours and a required correction — never an assumed departure time. | 2026-09-15 | Open |
| OQ-407 | Who approves attendance corrections — line manager, HR, or both? | 2026-09-15 | Open |
| OQ-408 | How far back may attendance be corrected, and does a finalised payroll run lock the period? Plan assumes locked, with adjustments handled in feature 07. | 2026-09-15 | Open |
| OQ-410 | Feature 06 is built after 04, so `ON_LEAVE` has nothing to read initially and those days will compute as `ABSENT`. Plan makes a bulk recompute part of 06's release. | 2026-09-15 | Open — action lands in feature 06 |
| OQ-S-01 | Does the company use rotating shift patterns, or is everyone on one fixed shift? If fixed, patterns can be deferred and the build shrinks noticeably. | 2026-09-15 | Open |
| OQ-402, OQ-409, OQ-411…420, OQ-S-02…06 | Remaining feature-04 questions (multi-device pairing, probation rules, device on the day record, non-working-day rows, reason-trail shape, punch partitioning, protocol re-sync pull, future-dated rows, recompute range cap, grid refresh, live "who's in", employee lateness trends, swap requests, rostering grid, shift versioning, manager-assigned shifts, minimum rest between shifts). See the feature files. | 2026-09-15 | Open |
| OQ-301 | **Cheapest-now, most-expensive-later question in the plan.** Will the system ever serve more than one company or legal entity? Plan assumes no (singleton `Company`). Retrofitting multi-tenancy after eleven features is close to a rewrite. | 2026-09-15 | Open — needs decision before build |
| OQ-302 | The company's timezone, whether it has ever changed, and whether any employees work in a different one. Decides day boundaries for all attendance. | 2026-09-15 | Open |
| OQ-303 | Do different employee groups observe different holidays (location, religion, contract)? Model supports multiple calendars; UI exposure depends on the answer. | 2026-09-15 | Open |
| OQ-304 | Does the work week vary by employee group (office Mon–Fri vs factory six days)? | 2026-09-15 | Open |
| OQ-306 | How many SenseFace terminals, at how many locations? Interacts with OQ-206. | 2026-09-15 | Open |
| OQ-307 | Does the SenseFace 2A support a per-device comm key, or only one shared key? If shared, feature 03's D-08 is not implementable as designed. | 2026-09-15 | Open — blocked by OQ-000 |
| OQ-305, OQ-308…317 | Remaining feature-03 questions (half-day semantics, offline-day attendance, lunar/computed holidays, iCal feed, who holds `device.write`, protocol fields available, public settings endpoint, event storage at scale, job runner location, setup checklist, device server address). See the feature files. | 2026-09-15 | Open |
| OQ-201 | **Highest-impact open question so far.** Is `DEPARTMENT` scope the actor's manager chain, the department subtree they head, or the union? Feature 01 assumed the manager chain; the plan now proposes the union. Affects every scoped query in every module. | 2026-09-15 | Open — needs decision before build |
| OQ-202 | Which personal/PII fields the company actually needs to hold (national ID, DOB, address, gender, marital status…). Unused PII is pure liability. | 2026-09-15 | Open |
| OQ-203 | Retention/deletion obligation for terminated employees' records. Would override "never delete" and require an anonymisation path. | 2026-09-15 | Open |
| OQ-204 | SenseFace 2A user-PIN format (numeric? max length?) — constrains `employeeCode`. Blocked on the same missing manual as OQ-000. | 2026-09-15 | Open — blocked by OQ-000 |
| OQ-205 | Rehires: one employee record with multiple employment periods (assumed) vs separate records. | 2026-09-15 | Open |
| OQ-206 | Multiple work locations/branches? Cheaper to add now than after the history table exists. Plan assumes single location. | 2026-09-15 | Open |
| OQ-207…214 | Lower-impact feature-02 questions (custom fields, document types, doc volume + backup in docker-compose, `fields` param, async import, expiring-docs ownership, photo handling, profile completeness indicator). See the feature files. | 2026-09-15 | Open |
| OQ-108…115 | Lower-impact feature-01 questions (remember-device, rate-limit storage if scaled, nav derivation, API versioning, post-reset sign-in, dashboard shape, branding, i18n). See the feature files. | 2026-09-15 | Open |

---

## Session entries

### 2026-09-28 — Session 30: Feature 01 — API, screens, and the rules that fail silently

**Built — the API** (01 api-design.md), on domain services shared by route handlers and Server Actions
- `src/lib/users/service.ts`, `roles/service.ts`, `audit-log/service.ts`, `auth/password-reset.ts`,
  `auth/sign-in.ts`. Every mutation takes the tenant transaction, so RLS confines it and its audit
  entry commits or rolls back with it (FR-L-05).
- `/api/users` (+ `:id`, `roles`, `status`, `reset-password`, `sessions`) · `/api/roles` (+ `:id`,
  `permissions`) · `/api/permissions` · `/api/audit-logs` (+ `:id`, `export`) ·
  `POST /api/auth/password/reset`.

**Rules enforced in the services, each with a test**
- FR-U-06 no self role/status change · FR-U-08 last active super admin protected, **serialised
  with a per-tenant advisory lock** (two admins suspending each other at once would otherwise both
  pass the check) · FR-Z-08/09 system and in-use roles · suspension and admin resets end sessions
  in the same transaction.
- **New, not in the spec — the no-escalation rule (OQ-119).** HR_ADMIN holds `user.assign_role` at
  ALL, and FR-U-06 only blocks changing your *own* roles, so two HR admins could make each other
  SUPER_ADMIN. Now nobody may assign a role, edit grants, or act on a user beyond their own access.
  The UI shows those options disabled with the reason.
- Reset/invite tokens: 256-bit, SHA-256 at rest, 60 min, single-use and **claimed atomically**
  (two tabs redeeming one link: exactly one succeeds — tested).
- `sessions` has no tenant column and so no RLS: session reads resolve the user through the
  RLS-protected `users` table first. Tested: Acme cannot list a Globex user's sessions.

**Sign-in rebuilt around `attemptSignIn()`** (out of the NextAuth config, so it is testable)
- Audits `auth.login` / `login_failed` / `account_locked` / `logout` (FR-L-06 — the gap Session 29
  found). All writes in one tenant transaction as `hrm_app`; `hrm_auth` only finds the user.
- **Lockout counters re-read `FOR UPDATE`**: five concurrent wrong guesses now always count to five
  — tested; a read-modify-write lost-update is a classic way past a lockout.
- Same-cost dummy verify for unknown emails; per-IP throttle (10/15 min); only `ACTIVE` may sign in
  (it previously let `INVITED` through `authorize`); session rows now carry IP and user agent.
- Auth.js no longer logs every wrong password as an error with a stack.

**Built — the screens** (`ui-ux-pro-max` + minimalist tokens; `impeccable` skipped because its first
run downloads a binary, which `C:\Dev\CLAUDE.md` says to ask about first)
- App shell: nav derived server-side from permissions (correct on first paint; empty groups
  omitted); top bar with company name and user menu; drawer below 768 px; skip link.
- Users list (URL-held filters) · invite (roles beyond your access disabled, with the reason) ·
  user detail (**guard rails as disabled buttons with a visible reason**, not a hover tooltip;
  confirmations state the consequence) · roles list · **the permission matrix** (labelled radio
  group per permission, collapsible groups with "n of m", group setter, changed rows marked, sticky
  "N changes" footer, blast-radius confirmation, unsaved-changes warning, SUPER_ADMIN read-only) ·
  audit log (in-place before/after diff; CSV export that respects filters, **neutralises formula
  injection** and audits itself) · `/reset-password` for invites and resets (`no-referrer`, since
  the token is in the URL) · `/403` · a dashboard showing only what the person can open.
- `pageGuard` / `actionGuard` give Server Components and Server Actions `protectedRoute`'s
  guarantees: a declared permission, the forced-change block, the tenant transaction.
- New status tokens measured in both themes; minimalist-ui's yellow text failed AA and was darkened.

**Bugs found along the way**
- **A cross-tenant duplicate email surfaced as a raw 500.** Postgres omits the conflicting key from
  a unique-violation error when that row is hidden by RLS (it will not disclose another tenant's
  data), so Prisma reported no target. Handled explicitly, with the reason in `errors.ts`.
- **A string constant exported from a `"use client"` module** (my own, Session 29) reached Server
  Components as a client *reference*. Shared class strings now live in a plain module.
- Audit before/after were stored as JSON `null`, which `IS NULL` misses; now SQL `NULL`.
- Two effects that set state (cascading renders) and an impure `Date.now()` in render — caught by
  the React lint rules, fixed with the documented patterns.

**Verification**
- **HTTP smoke, 30/30** against the running app: every screen for the admin; the employee's view
  (403 JSON from the API, `/403` from pages, no Administration group); **acceptance criteria 3**
  (invite → link → set password → sign in as MANAGER), **5, 6** (suspension kills the next request),
  **8** (audit trail), **9**; cross-origin mutation refused over the wire.
- **150 unit · 88 integration · 25 database · tsc and lint clean.**
- Mutation check: weakening the super-admin floor fails the suite. A second mutation (disabling the
  escalation check) was **blocked by an auto-mode guard** as security-weakening and not retried; the
  rule is pinned by five tests that expect `ESCALATION_REFUSED`.

**Not verified — owed**
- **The admin screens have not been looked at in a browser.** Signing in there needs the admin
  password typed, and the guard that blocked reading it (Session 29) still applies. Their HTML,
  behaviour and permissions were verified over HTTP; layout, dark mode and the matrix's feel were
  not. A human visual pass is the next thing to do.

**Deviations and deferrals**
- `/forgot-password` not built: it needs feature 05 to deliver the link. Admins issue links instead.
- Per-session revoke (only "sign out everywhere"); a user edit form (the PATCH API exists); the
  spec's "send invite email" checkbox (nothing sends email yet); toasts replaced by inline
  `role="status"` messages (a toast vanishes before some users can read it).
- The forced change-password screen still stands alone rather than inside an inert shell.
- The "300/min per session" general rate limit is not implemented; login and reset limits are.
- Unknown-email sign-in failures are not audited: they belong to no tenant.

**Next**
- A visual pass over the new screens (you, or grant the browser sign-in).
- Step 5: feature 02 — employees CRUD, departments, positions, dated assignments, documents, CSV
  import, org chart.
- Decisions: OQ-119 (no-escalation), OQ-120 (invite lifetime), plus those carried from Session 29.

### 2026-09-28 — Session 29: Stack proven over HTTP; auth screens; integration test layer

**Step 1 — the stack, over real HTTP (previously never run)**
- `docker compose up -d --build app` → `next dev` (Next 16.3.4, Turbopack) against the live DB.
- Proven with scripted HTTP probes: unauthenticated → 401 · wrong password → no session cookie ·
  sign-in issues a JWT whose server-side `sessions` row stores only a 43-char SHA-256 hash · the
  forced password change returns **403 `PASSWORD_CHANGE_REQUIRED` even to SUPER_ADMIN** · SUPER_ADMIN
  holds 19/19 permissions from zero stored grants · ALL / DEPARTMENT / SELF scoping correct for
  admin, manager and employee · sensitive fields **absent** without `employee.read_sensitive` · no
  Globex rows ever · `includeTerminated` works · **deleting the session row kills a live cookie on
  the next request** (01 D-04 immediate revocation, the reason for the sid-in-JWT design) · sign-out
  deletes the row and a replayed cookie gets 401.

**The bug HTTP found that the SQL suite hid**
- Over HTTP, Ravi did **not** see Ayesha, contradicting Session 26. The app's CTE was right; the
  *data* was wrong: `test/db/06` re-pointed department heads inside a `DO` block, which autocommits,
  so **every `test:db` run permanently altered the canonical fixture**. Wrapped in
  `BEGIN … ROLLBACK`; the fixture now survives the suite. With canonical data Ravi sees Ayesha, as
  OQ-201's union predicts.
- The same investigation showed `test/db/06` tests a **copy** of the CTE, not `scope.ts` — its
  header claimed it would catch divergence; it could not. Corrected, and it motivated step 3.

**Step 2 — sign-in and change-password screens** (`ui-ux-pro-max` → `minimalist-ui`, per `C:\Dev\CLAUDE.md`)
- `/login`, `/change-password` (forced and voluntary), placeholder `/dashboard`, `/` → `/dashboard`.
  Server Actions + `useActionState`; copy exactly per 01 `ui-ux.md`.
- Skill grounding: Swiss-minimal for dense admin (design-system query); WCAG 2.2 accessible
  authentication (paste + password managers allowed, correct `autocomplete`); `role="alert"`;
  errors linked by `aria-describedby`; pending state on submit; Server Actions validate their own
  input. **Rejected from the skills:** the landing-page "Hero + CTA" pattern and GSAP motion
  (wrong for a form; no new animation library), and Inter (Geist is already loaded).
- **Two `minimalist-ui` values failed measured contrast** and were replaced: muted text `#787774`
  is 4.48:1 (→ `#6B6A67`, 5.2:1); `#EAEAEA` is 1.2:1 as an input edge (→ `#8F8E8A`, 3.3:1, WCAG
  1.4.11). Tokens now live in `globals.css`; the scaffold's Arial override is gone.
- **Measured, not assumed:** the show/hide toggle was 36px tall — fixed to 44px.
- Locked accounts get the spec's one non-generic message via a coded `CredentialsSignin` thrown
  from `authorize()`; verified in the browser.
- Password policy moved to a crypto-free `password-policy.ts`, shared by the live checklist and the
  server, so the UI cannot tick a rule the server rejects.
- Post-sign-in redirect accepts same-origin relative paths only (open-redirect guard, 25 tests).
- **Verified over HTTP, including a no-JavaScript form submission** (React's progressive-enhancement
  fields): 20 checks — wrong current password counts towards lockout; policy, mismatch and reuse
  errors land on the right fields; success clears the flag, revokes every old session, issues a new
  cookie, writes `auth.password_changed` with no password material, and the old password stops
  working.

**Deviations, deliberate — review if you disagree**
- **Change-password revokes ALL sessions and issues a fresh one**; the spec keeps the current one.
  The user stays signed in either way; rotation on a credential change is strictly stronger and
  needs no session id exposed to the browser.
- **The checklist does not disable the submit button** (reset-password spec says it should): a
  disabled button gives no reason and cannot be focused. Submitting shows the specific problem.
- `/forgot-password` is not built (needs feature 05); the login screen says "Ask an administrator to
  reset it" rather than linking to nothing.

**Step 3 — the integration test layer (did not exist)**
- `npm run test:integration`: reload fixture → run the **real** `prisma/seed.ts` for grants (the
  fixture deliberately has none) → Vitest on the compose network **as `hrm_app`**. 30 tests over
  `resolveScopedEmployeeIds`, `withTenant`/RLS through Prisma (including no context leaking across
  pooled connections), and the real `GET /api/employees` handler with only `getSession` mocked.
- **Mutation-checked:** deleting the department-head half of the CTE fails 5 tests (the SQL copy
  would fail none); removing sub-department recursion fails 1. That exercise **caught a weak test of
  my own** — built on Ayesha, whose reporting chain already covers everyone, it passed with half the
  CTE deleted. Rewritten around Sara.
- `loadGrants` moved to `src/lib/auth/grants.ts` so tests use the production path without NextAuth.
- The runner deletes its throwaway super admin afterwards; left in place, it made the seed skip
  creating the real admin — found when the admin probe failed after a run.

**Environment notes**
- `.env` (gitignored) now holds generated **local test** credentials: `SEED_ADMIN_PASSWORD` for
  `admin@acme.test`, and `FIXTURE_TEST_PASSWORD`, set on `ravi@` / `imran@acme.test` for probes.
  Any fixture reload (`test:db`, `test:integration`) wipes these users; re-run the seed.
- The dev DB is left seeded with `admin@acme.test` pending its forced password change, so the UI
  can be tried from the start.
- tsc needs `next typegen` first when routes change (`PageProps<"/login">` is generated).
- An auto-mode guard blocked me from reading the test password to type it into the browser, so the
  **change-password screen has not been visually checked** in a browser — its behaviour was
  verified over HTTP. The login screen was checked in the browser: dark, light, 375px, focus rings.

**Status: 138 unit · 30 integration · 25 database · typecheck and lint clean.** `hrm-system` has 8
commits; the last 5 (from `5d1baf6`) are not on GitHub — the remote stops at `f630325` (2026-09-14).

**Found, not fixed (for step 4)**
- Sign-in events are not audited: 01 api-design specifies `auth.login` / `auth.login_failed` /
  `auth.account_locked`; `authorize()` writes none.
- Audit `before`/`after` store JSON `null` rather than SQL `NULL` when there is no diff; queries
  using `IS NULL` would miss them.
- Auth.js logs every failed sign-in as `[auth][error] CredentialsSignin` with a stack — noise that
  will bury real errors; configure its `logger`.
- The app shell with inert links during a forced change (01 ui-ux) waits for the shell itself.

**Next**
- Step 4: finish feature 01 — shell, sign-in auditing, users/roles/audit endpoints and screens,
  forgot/reset once feature 05's outbox exists (or an interim admin-issued reset link).
- Your decisions: OQ-701, OQ-601b, OQ-319 (blocking) · OQ-116 Argon2id vs scrypt · OQ-117 confirm
  the OQ-201 consequence · OQ-118 device-event retention · **OQ-003 revoke the pasted token** · push
  both repos.

### 2026-09-28 — Session 28: Git under control, seed working, first protected endpoint

**Version control — the standing risk, closed**
- `git` is still not installed on this machine, so it runs in a container (`alpine/git`) against the bind-mounted repos. No host install needed.
- **`dev-plan` had never been a git repository at all** — 97 files describing eleven features existed only on one disk. It is now a repo with its own first commit. That was the larger of the two risks and was invisible because `hrm-system` *was* versioned.
- `hrm-system`: two commits added. Before committing, 49 files showed as modified; **42 of them were pure file-mode churn** from the macOS→Windows move, with zero content changes. `core.fileMode false` reduced it to the 7 real changes, which kept the diff reviewable instead of drowning the actual work.
- The stale `dev-plan/` copy inside `hrm-system` was already deleted on disk; the deletion is now recorded, which closes **OQ-004**.
- **Push still needs the user's credentials** and was not attempted. Commits are local, so the work is protected against accidental edits but not against losing the disk. I do not handle tokens — which is also why OQ-003 is still open.

**Seed**
- `prisma/seed.ts`, run with `tsx`. Verified idempotent: the second run created nothing.
- Orphaned permission keys are **reported, never deleted** — a typo in the catalogue must not silently revoke access that roles depend on.
- Existing grants are not overwritten, because HR may have edited three of the four system roles deliberately.
- **It refuses to invent a default admin password.** A seeded well-known credential is the most common way a system like this is compromised, so missing env vars are a hard failure. `mustChangePassword` is set, because the value handed to the seed has been in a shell history.

**Proved the auth core against real data**
- The seeded admin's password verifies; a wrong password is rejected; the hash is scrypt as intended.
- **SUPER_ADMIN has zero stored grant rows and 19 of 19 effective permissions** — the computed-not-stored design working exactly as specified, confirmed against a live database rather than only in unit tests.

**The inconsistency that verification exposed**
- `MANAGER.department.read` was `DEPARTMENT` in the database but `ALL` in the code catalogue. Cause: the test fixture seeded its own `role_permissions`, and since the seed only inserts **missing** grants (it must not undo deliberate admin edits), the fixture's wrong value won.
- Fixed by **removing grants from the fixture entirely**. There is now one source of truth — the code catalogue — and the database tests never needed grant semantics anyway; those are tested in `permissions.test.ts`. Two seeders for the same data was the actual defect, not the mismatched value.

**Built**
- `protectedRoute`: requires a permission **in its signature**, so a handler registered without one does not compile. "Fails closed" becomes a type error rather than a review item. It also hands the handler a transaction with `app.tenant_id` already set, so RLS is active and the global client cannot be reached by accident.
- `GET /api/employees`: the vertical slice — session → permission → tenant context → RLS → scope filter → permission-driven field selection. Scope is composed into the `WHERE` clause, so the pagination total cannot leak the real row count; sensitive columns are not *selected* without the permission rather than selected and stripped.

**Status: 113 unit tests, 25 database assertions, typecheck clean, 3 commits.**

**Honest gap**
- The endpoint has **not been called over HTTP**. It typechecks and its parts are tested, but `next dev` has not run, so the Auth.js integration is unproven end to end. That needs the app container up and is the next thing to do, together with the sign-in page — which is UI work and needs the skill grounding `C:\Dev\CLAUDE.md` requires.

### 2026-09-28 — Session 27: Auth.js wiring — and a library constraint that forced a design change

**The finding that shaped the session**
- **Auth.js v5's Credentials provider cannot use database sessions.** Confirmed by reading `node_modules/@auth/core/lib/actions/callback/index.js` (the `provider.type === "credentials"` branch, lines ~247–276): it always encodes a JWT cookie and **never calls `adapter.createSession`**, regardless of adapter or `session.strategy`. `lib/init.js:74` only chooses "database" as a default when an adapter is present; the credentials path ignores it.
- That collides head-on with **01 D-04** (database sessions, immediate revocation), which the OQ-101 answer was assumed to be compatible with. The risk was flagged when NextAuth was chosen; it is now confirmed rather than suspected.

**The resolution: the JWT carries only a session id**
- No adapter is passed, and `strategy: "jwt"` is explicit — with an adapter present Auth.js would default to the database strategy and then not use it for credentials, which is the worst of both.
- The JWT holds a `sid`. The authoritative record is a row in our own `sessions` table, re-read on **every** request along with the user's status and permissions (01 FR-A-06).
- So D-04's *behaviour* survives: revocation is a DELETE and is immediate; "sign out everywhere" and the live-session list remain possible, which a pure JWT design cannot offer at all. Only the token in the JWT is hashed at rest (SHA-256), preserving the original reasoning that a database leak should yield no usable sessions.

**Built**
- `src/lib/auth/password.ts` · `session-token.ts` · `lockout.ts` · `src/lib/db.ts` (two clients: `prisma` as `hrm_app` with RLS, `authPrisma` as `hrm_auth` for the sign-in path only) · `src/auth.ts` · the Auth.js route handler · `src/lib/auth/session.ts` (`getSession` → `SessionContext`, so nothing outside the auth layer knows which library is in use).

**A second deviation, flagged for a decision**
- **Password hashing uses scrypt from Node's `crypto`, not Argon2id** as 01 FR-A-02 specifies. Every Argon2 binding for Node is a native module, and this project builds in `node:20-alpine` where native builds are the least reliable part of the toolchain. scrypt is memory-hard and OWASP-accepted (second, after Argon2id and ahead of bcrypt), so it is defensible — but it is a deviation and the call belongs to whoever owns security. Swapping is a one-file change by design: `verify()` recognises hashes by algorithm prefix and `needsRehash()` already returns true for non-scrypt hashes, so existing passwords upgrade on next sign-in.

**Typechecking earned its place immediately**
- Vitest does not typecheck, so I ran `tsc --noEmit`. It found 7 errors, one of which was a **latent runtime bug**: `SessionRowLike` declared `expiresAt` while the Prisma model's column is `expires`. Because the call sites cast through `as never`, nothing complained — and `isSessionValid` would have called `.getTime()` on `undefined`, making **every session read throw**. Fixed, the casts removed, and the reason recorded in the file so it does not come back.
- Also fixed: `promisify(scrypt)` silently drops the options argument, which would have ignored the tuned N/r/p and maxmem and produced a hash far cheaper to attack than intended. Replaced with an explicit Promise wrapper.
- TypeScript itself was another broken macOS leftover (like Prisma in Session 25); reinstalled.

**Status: 113 unit tests passing, 25 database assertions passing, typecheck clean.**

**Next**
- The seed (permission catalogue + system roles per tenant), then the sign-in page and the first protected route handlers.
- Not yet exercised end to end: no user has actually signed in, because no seed has created one with a real password hash. That is the next thing worth proving.

### 2026-09-23 — Session 26: Features 01/02 — the contract layer, built and tested

**Read first**
- Per `AGENTS.md`, read the installed Next.js docs before writing app code. Route handlers are conventional Web `Request`/`Response` in `app/**/route.ts`, not cached by default for `GET`, and `params` arrives as a Promise — which matches the existing device route.

**Built — the cross-feature contract layer (`00-overview.md` §5)**
Started here rather than with endpoints because the overview is explicit that these helpers matter more: get them right and the system stays coherent; work around them and it does not.
- `src/lib/auth/permissions.ts` — the code-declared permission catalogue (19 keys for features 01/02), the scope precedence rule, and the seeded system-role grants.
- `src/lib/auth/authorize.ts` — `effectivePermissions`, `requirePermission`, `employeeScopeFilter`, `canSeeEmployee`. All pure.
- `src/lib/audit.ts` — `writeAudit(tx, entry)` with write-time redaction and changed-fields diffing.
- `src/lib/employees/scope.ts` — the recursive CTE for DEPARTMENT scope, plus cycle checks for reporting lines and the department tree.

**Design points worth keeping**
- **`requirePermission` returns the SCOPE, not a boolean.** That is what stops a caller treating authorization as a yes/no gate and forgetting to narrow the query (01 FR-Z-05). There is no "allowed: true" to misuse.
- **`EmployeeScopeFilter` is `"ALL" | { employeeIds: number[] }`.** An empty array and "no restriction" are different types, so they cannot be confused — which is exactly how 01 FR-Z-06 ("never fall back to everyone") gets violated in practice. Pinned by a test.
- **SUPER_ADMIN is computed from the catalogue, never stored.** A permission added by a future feature is held by the owner the moment it exists, rather than being silently missing until someone remembers to grant it.
- **`writeAudit` takes a transaction client and has no way to reach a global one**, so 01 FR-L-05 (audit lives or dies with the change) is enforced by the signature rather than by convention.
- **Redaction happens at write time**, not read time: a value never stored cannot leak later. Salary figures are deliberately *not* redacted — auditing them is a primary reason the log exists.

**Tests: 73 unit + 25 database, all passing**
- New database suite `06-scope-resolution.sql` exercises the real CTE against the fixture.

**A consequence of OQ-201 you should confirm**
- The union answer means **a department head sees everyone in their department, including people senior to them in the reporting line.** In the fixture, Ravi heads Operations and therefore sees Ayesha — his own manager. That follows directly from "union of both" and is asserted as intended behaviour, but it is the kind of thing worth agreeing explicitly before it surprises someone. If it is wrong, the fix is small now and awkward later.

**Next**
- Auth.js wiring: the Credentials provider, the Prisma adapter with database sessions (never JWT), the session callback carrying `tenantId`, and the custom adapter needed to hash session tokens rather than store them in plaintext.
- Then the seed (permissions + system roles per tenant), then route handlers.

### 2026-09-23 — Session 25: Vitest installed; unit layer complete. **43 tests green across both layers.**

**Done**
- Installed **Vitest 2.1.9** (approved this session) and repaired the Prisma packages, which were macOS leftovers whose CLI binary did not exist under Linux — `rm -rf node_modules/prisma` and reinstall fixed it. `prisma generate` now works, so the generated client exists for type resolution.
- Added `vitest.config.ts` with the `@/` → `src/` alias set explicitly rather than pulling in a tsconfig-paths plugin for a single alias.
- Added scripts: `npm test`, `npm run test:watch`, `npm run test:db`.
- Wrote the unit layer — **23 tests**, all passing, no database, no network, no real clock.
- Wrote `test/README.md` so the two-layer split, the two-tenant rule, and the role-per-test rule survive without someone reading the whole plan.

**Both layers now green**
- `npm test` → 23 passed · `npm run test:db` → 20 assertions passed.

**Decisions encoded in the tests rather than only in prose**
- **`TZ=America/Los_Angeles` in the Vitest config**, deliberately not the tenants' `Asia/Karachi`. A suite that passes only because the server shares the company's timezone is testing nothing, and this is the cheapest possible guard against that.
- **`set_config`'s third argument must be `true`** — asserted directly. With `false` the setting persists on a pooled connection and the next request inherits whichever tenant ran last. That is a cross-tenant leak no amount of correct application code would catch, so it gets its own test.
- **The tenant id is parameterised, not interpolated** — also asserted, including that the literal value does not appear in the SQL string.
- **`withTenant` gives its callback no way to change the context it runs under** — a test on the shape of the API, not just its behaviour.
- **Invalid tenant ids throw rather than proceeding.** Under RLS an empty context returns zero rows, which reads as "this tenant has no data" rather than "this is a bug" — so failing loudly is the correct behaviour and is pinned by `it.each([0, -1, 1.5, NaN])`.
- **`fixedClock` hands out copies.** `Date` is mutable; returning the same instance would let one caller's `setTime()` silently change what every other caller sees.
- **The golden date is asserted to be a Wednesday**, mid-month, mid-June. If someone later "tidies" it to a round number like 1 January, the test fails and explains why that matters.

**Next**
- Feature 01/02 application code. `AGENTS.md` requires reading `node_modules/next/dist/docs/` before writing any Next.js code — that comes first, since the installed Next.js 16 diverges from training data.
- Still outstanding from planning: **OQ-319** (hosted vs per-company, blocks ingestion), **OQ-701** (pay components), **OQ-601b** (leave entitlement days). None blocks features 01–03.

### 2026-09-23 — Session 24: Test harness built — 20 assertions, all passing

**Done**
- Built the harness `TESTING_STRATEGY.md` called for, in the order it specified: canonical fixture and two-tenant setup first, injected clock second.
- `src/lib/clock.ts` — `Clock` interface with `systemClock`, `fixedClock`, `mutableClock`, and the golden date (2026-06-17: a Wednesday, mid-month, mid-year, away from DST). Business logic never calls `new Date()`.
- `src/lib/tenant-context.ts` — `withTenant()`, the only place `app.tenant_id` is set, transaction-local.
- `prisma/fixtures/canonical.sql` — two tenants, **six employees each with deliberately colliding codes, department names and position titles**, a three-level reporting chain, a nested department, two locations, and the awkward cases (no user account, terminated, probation). Sara Ahmed moved Assembly → Finance in 2024, specifically so a naive historical query has something to get wrong.
- `test/db/*.sql` + `run.ps1` — five suites, 20 assertions, all passing.

**Decision: no new npm dependency**
- The Vitest install was declined, so the harness uses **psql and SQL instead of a JS test runner**. That turns out to suit the highest-value tests anyway — RLS, constraints, and recursive queries exist only in Postgres, and `TESTING_STRATEGY.md` already argued that mocking the database there only tests the mock.
- **Still outstanding:** the TypeScript unit layer for feature 04's classification engine and 07's payroll engine has no runner. Node 20 has no built-in TypeScript, so that layer needs either a dependency (Vitest, or `tsx` plus `node:test`) or a build step. **A decision is needed before those engines are written** — they are the tests the whole engine design exists to enable.

**Three bugs the suite caught, which is the point of writing it**
1. **RLS crashed on an empty tenant context.** `current_setting('app.tenant_id', true)` returns `NULL` when never set but `''` when cleared, and `''::int` raises `invalid input syntax for type integer`. So clearing a context produced a 500 rather than an empty result. Hardened with `NULLIF(..., '')` and shipped as migration `20260923000000_rls_empty_context`. Both behaviours are "safe"; only one is correct, and the error would have surfaced as a mysterious crash rather than a visible boundary.
2. **`overlaps` is a reserved SQL keyword** (the `OVERLAPS` operator), so it cannot be a PL/pgSQL variable name — it produced a bare `syntax error at or near ">"` pointing at the wrong line.
3. **The fixture was not re-runnable**: the global permission catalogue is deliberately not truncated, so a second run collided with itself. Now `ON CONFLICT DO NOTHING`, which is also how the real code-declared seed will behave.

**Also fixed**
- `run.ps1` cannot use `$ErrorActionPreference = 'Stop'`: psql writes `RAISE NOTICE` — our PASS lines — to stderr, and PowerShell 5.1 wraps native stderr in a `NativeCommandError`, so the first passing assertion aborted the run. Failure is detected from `$LASTEXITCODE`, which is what `ON_ERROR_STOP` actually sets.
- Docker Desktop was not running after the gap since Session 23; started it and the Postgres container came back healthy with its volume intact.

**What the suite now proves**
- Fails closed with no tenant context · each tenant sees only its own rows · cross-tenant INSERT refused by `WITH CHECK` · cross-tenant UPDATE affects nothing · isolation holds on departments, users, assignments and roles · `hrm_app` cannot bypass RLS · every `tenant_id` table has forced RLS · employee codes unique per tenant and reusable across tenants · emails globally unique · positions unique per department · **historical department resolution differs from the naive join** · no overlapping assignment periods · the reporting-chain CTE is correct at three levels, for a leaf, and **does not cross tenants** · terminated employees stay in the chain.

**Next**
- Decide the TypeScript test runner.
- Feature 01/02 application code, which needs `node_modules/next/dist/docs/` read first per `AGENTS.md`.

### 2026-09-18 — Session 23: Migrations applied; tenant isolation verified — and one serious bug caught

**Done**
- Brought up Postgres via compose, generated the schema delta, split it into three migrations, and **applied them all**: `20260917235900_employment_status_enum_values` → `20260918000000_foundation_tenancy_auth_employees` → `20260918000100_row_level_security`.
- Created `prisma/roles.sql` and the three database roles.
- **Verified tenant isolation against the live database** rather than assuming it: no tenant context returns **0 rows** (fails closed), each tenant's context returns only its own rows, a cross-tenant INSERT is refused by `WITH CHECK`, and a same-tenant INSERT still works.

**The bug worth the whole session**
- The first isolation test **failed**: every row was visible regardless of context, and the cross-tenant INSERT succeeded. The cause is that **`hrm_user` is a SUPERUSER** — the postgres image makes `POSTGRES_USER` one — and **superusers bypass RLS even with `FORCE ROW LEVEL SECURITY`**. Nothing overrides that.
- `docker-compose.yml` pointed the application at exactly that role. So as configured, **every tenant-isolation policy would have been silently inert** while `\d` continued to show them enabled. This is precisely the failure mode `MULTI-TENANCY.md` was written to prevent, and it would have shipped.
- `rls.sql` had already documented the *owner* bypass and prescribed FORCE for it. It did not know about the *superuser* bypass. Both are now documented in `roles.sql`, and the compose file and `.env.example` now use `hrm_app`, a non-superuser with `NOBYPASSRLS`.
- The lesson is the one `TESTING_STRATEGY.md` already argued for and which I nearly skipped: **RLS has to be exercised against a real database.** Reading the schema would have shown policies enabled and forced, and told me nothing.

**Three smaller obstacles, each leaving a trace in the repo**
- `prisma migrate dev` refuses to run non-interactively when it has warnings, so migrations were produced with `migrate diff` and applied with `migrate deploy` — which had the side benefit of forcing a review of the SQL before it ran.
- **A UTF-8 BOM** from PowerShell's `Set-Content -Encoding utf8` made Postgres reject the first migration outright (`syntax error at or near "﻿"`). Files are now written BOM-less via `UTF8Encoding($false)`.
- **Postgres refuses to use a new enum value in the transaction that adds it**, so the three new `EmploymentStatus` values needed their own earlier migration. Documented in that file so the reason survives.

**Recorded in the migration itself**
- The foundation migration is **safe only on an empty `employees` table**: it drops the free-text `department`/`position`/`email` columns and adds `tenant_id NOT NULL` with no default. This database was empty so neither bit, but a deployment with real data needs 02's backfill sequence first. That warning is now a header comment in the migration rather than a note in a planning file nobody will open at 2am.

**Environment**
- Exploratory test rows were removed; the database is empty and migrated.
- Postgres is left running (`hrm-system-postgres-1`). Stop with `docker compose -f C:\Dev\hrm-system\docker-compose.yml down`.

**Next**
- The test harness: canonical fixture, the two-tenant setup, injected clock — now with a verified isolation mechanism to build fixtures against.
- Application code still needs `node_modules/next/dist/docs/` read first, per `AGENTS.md`.

### 2026-09-18 — Session 22: Build starts — foundation schema and RLS

**Done**
- Established that this machine **can** build after all: no Node, npm or git, but Docker is installed and running, and the project already containerises Node. Tooling now runs as `docker run --rm -v C:\Dev\hrm-system:/app node:20-alpine …`. The repo's `node_modules` came from macOS and its Prisma binary does not work under Linux, so validation uses an **ephemeral global install** inside the container rather than modifying their `node_modules`.
- Wrote the **foundation `prisma/schema.prisma`**: `Tenant`, feature 01 (users, roles, permissions, grants, sessions, reset tokens, audit), feature 02 (employees, departments, positions, assignment history, employment periods, documents, emergency contacts), and `Location` + `HolidayCalendar` from feature 03, which 02 depends on. The scaffold's `Device`/`Attendance` are left untouched for features 03/04.
- **Validated it** — `prisma validate` passes.
- Wrote `prisma/rls.sql`: the Row-Level Security layer Prisma cannot express, which is the actual enforcing mechanism for tenant isolation.

**Two things the work surfaced that the plan had not**

1. **Auth.js stores session tokens in plaintext.** The plan's 01 D-04 specified a SHA-256 hash so that a database leak yields nothing usable. The Auth.js Prisma adapter's `Session.sessionToken` is the raw token. Recorded as a comment on the model. Closing it needs a custom adapter that hashes on write and lookup — worth doing, and a real cost of choosing Auth.js that was not visible when OQ-101 was answered.

2. **Sign-in cannot run under tenant RLS**, which is a genuine circularity: the query that establishes the tenant (look up a user by email; look up a session by token) would itself require the tenant to already be known. Resolved with **two database roles** — `hrm_app` with RLS enforced for all normal queries, and a narrowly-scoped `hrm_auth` with `BYPASSRLS` used *only* by the Auth.js adapter on the sign-in path, as separate connection strings. Safe while that confinement holds; dangerous the moment something else borrows that connection, so it is documented prominently in the file.

**Details worth keeping**
- `FORCE ROW LEVEL SECURITY` as well as `ENABLE`: without FORCE, the table owner bypasses every policy silently, so if migrations and the app both connect as the owner, ENABLE alone protects nothing.
- `WITH CHECK` as well as `USING`: USING filters reads, WITH CHECK constrains writes. Without it, code could insert a row carrying another tenant's id — the exact mistake the mechanism exists to catch.
- `set_config(..., true)` makes the tenant context **transaction-local**. With `false` it persists on a pooled connection and hands the next request whichever tenant ran last.
- The hand-maintained table list in `rls.sql` will eventually miss a table, so the file ends with the **CI query that actually keeps it honest**: any table with a `tenant_id` column and no forced policy, expected to return zero rows.

**Next**
- Bring up Postgres via compose and generate the schema migration, then apply `rls.sql` as its own migration.
- Then the test harness — canonical fixture, the two-tenant setup, injected clock — before any feature code, per `TESTING_STRATEGY.md`.
- Application code will need `node_modules/next/dist/docs/` read first, per `AGENTS.md`.

### 2026-09-18 — Session 21: Testing strategy — the last pre-build gap closed

**Done**
- Wrote `TESTING_STRATEGY.md`, closing gap 1 from `DEFINITION_OF_DONE.md` — the largest remaining gap and the one that needed closing before code rather than after.
- Shaped it around six properties that make *this* system hard to test — time is the domain, money must be reproducible, state is derived, isolation fails open, the hardware is unavailable, permissions are a large matrix — **four of which fail silently**. That is the same spine as the definition-of-done register, deliberately.

**The decisions worth recording**
- **Two tenants in every test, always** — not a separate isolation suite. "Acme" is under test; "Globex" exists solely so a leak has somewhere to leak from, with **deliberately colliding employee codes and department names**. A test asserting "the manager sees 3 employees" passes whether or not tenancy works if only one tenant exists; with Globex present, the same assertion becomes meaningful. This is the single most valuable structural choice in the document and is expensive to retrofit.
- **The permission matrix is generated, not hand-written.** Because permissions are a code-declared catalogue, a test can walk the route registry and fail on any endpoint without a declared permission — turning 01's fail-closed guarantee from an intention into an assertion. The same trick covers the other four catalogues.
- **The CI server's timezone is deliberately not the tenant's.** A suite that passes only because the server happens to run in the company's timezone is testing nothing, and that is precisely the bug 03's date helpers exist to prevent.
- **`isPeriodLocked`'s stub and its real implementation must pass the same contract tests.** 04 and 06 are built against the stub and 07 replaces it; without a shared contract, the replacement silently changes behaviour in two finished features.
- **The coverage target is not a percentage** — it is that every numbered acceptance criterion across the eleven `requirements.md` files (~140 of them) is an automated test. They were written to be executable, which is now the reason that mattered.
- Eight **properties** worth asserting rather than exampling — recompute idempotency, ledger-sums-to-balance, payslip reconciliation, payslips frozen after finalisation, and suppression resisting differencing. The last is the requirement most likely to be implemented as "hide the small group" and shipped.

**On the device**
- A **device simulator** doubles as the test double and as the tool for exercising a real deployment before a terminal is installed.
- When a real terminal first connects, **capture the exchange verbatim and commit it as a fixture**. That recording is worth more than any inference from the manual, and it is what finally removes the ⚠ markers from 04's `api-design.md`.

**Status**
- Planning complete; the pre-build gap closed. The recommended first code is now the **test harness** — canonical fixture, two-tenant setup, injected clock — because all three shape everything after them and none can be retrofitted cheaply.
- Still blocking: OQ-319, OQ-701, OQ-601b. Still open and unactioned: **OQ-003, the pasted GitHub token.**

### 2026-09-18 — Session 20: Step 6 — definition of done. **Planning complete.**

**Done**
- Wrote `DEFINITION_OF_DONE.md`, the last planning step. Steps 0–6 are now all complete.
- Organised it around **how this system actually fails** rather than as a generic checklist. A list saying "code reviewed, tests pass" would be true of any project and useful to none.

**The centre of the document is Part 2 — the silent-failure register**
- Fourteen entries, each a failure that nobody notices for weeks, paired with the check that catches it. Writing the eleven features surfaced these one at a time; collecting them in one place is the first time the pattern is visible.
- The worst of them: **04's day opener stopping**. Absent employees generate no punch, so nothing marks their day dirty, so no row is written — and attendance looks *better* than reality. Its check is not just a last-run timestamp but a daily assertion that every employed person has a row for yesterday.
- Second: **backups missing the document volume**. The database restores cleanly and every contract, CV and payslip PDF is gone. The check has to be a restore drill that opens a file, not a green tick on a backup job.
- Third: **cross-tenant leaks**, which may never be noticed from inside the system at all. Hence RLS as the enforcing layer rather than application code, and a CI check that every tenant-owned table has a policy.
- A recurring principle in the register: a consistency check should **report** drift, never silently correct it. Silent correction hides the bug that caused it.

**Part 3 is a hard gate**
- Which features may start, given what is answered. Three are outstanding and two block outright: **OQ-701** stops feature 07 and **OQ-319** stops 04's ingestion. OQ-601b blocks seeding feature 06 but not building it.
- The engine-first sequencing for 04, decided back in Session 6 when the manual was missing, still pays off: its blocker costs nothing yet.

**Part 4 — go-live**
- The line I would defend hardest: **run one full payroll cycle in parallel with the existing process and reconcile line by line before cutting over.** Everything else in this plan is recoverable; paying people wrongly is not.
- Handover is defined as someone other than the builder being able to register a device, correct an attendance day, approve leave, run payroll, and explain a payslip line to an employee.

**Part 5 — honest gaps**
- Recorded rather than buried: **no testing strategy** (the largest remaining gap, and worth closing before code starts), no deployment/operations plan, no estimates, 01–04's un-grounded UI specs, the stale `dev-plan/` copy, no `git` on this machine, and **OQ-003 — the pasted GitHub token still not confirmed revoked**.

**Status**
- **Planning is complete.** 56 feature files, a root overview, two cross-cutting specifications, an open-questions register, and this. What remains is not planning.

### 2026-09-18 — Session 19: Step 5b completed

**Done**
- **`Location` model added** (OQ-206), placed in feature 03 rather than 02 — it is tenant infrastructure (an address, a calendar, the devices standing in it), in the same way `HolidayCalendar` lives in 03 while 02's `Department` references it. `Employee` and `EmployeeAssignment` both gain `locationId`, so "which site was she at in March" stays answerable like every other structural fact.
- **Holiday resolution reordered: location first**, then department, then up the department tree, then the tenant default. Worth the argument — department-first means a department spanning two countries gets one country's public holidays, which nobody notices until someone is marked absent on their national holiday.
- **No-deletion sweep applied** (OQ-1002): 10's purge became a weekly *review* that deletes nothing, and the retention sweeps in 05 and 11 were removed outright. 03's device-event sweep is flagged for confirmation rather than removed — those are machine-generated operational logs, not personal data, and without either that sweep or the transition-only logging mitigation, one terminal writes over a million rows a year.
- **Auth.js recorded as 01 D-11** with the specifics that matter: database session strategy via the Prisma adapter (JWT strategy prohibited, or D-04's immediate revocation is lost), the Credentials provider holding the lockout and generic-failure logic, the session extended to carry `tenantId`, and password reset/invite staying outside Auth.js. Two risks flagged to close early — unverified Next.js 16 compatibility, and reconciling the adapter's session table with `data-model.md`'s `Session` rather than shipping both.
- **Emailed payslips recorded as an exception, not an erosion** (OQ-709): 05 D-06 still stands everywhere else, and 07 FR-L-05 now carries the four controls — no figures in the body, password-protected PDF, per-payslip disclosure audit, and a bounce raising an alert rather than silently dropping someone's payslip.
- **Tenancy revision notes added to all eight remaining data models**, each naming that feature's specific composite constraints, per-tenant jobs and provisioning changes rather than repeating boilerplate.

**Judgement call worth recording**
- I did **not** mechanically add a `tenantId` line to every model in every file. The cross-cutting document specifies the rule once, and each feature file now names its own exceptions and hot indexes. Spelling out ~150 identical column declarations across eleven files would have been volume rather than progress, and would have created eleven places for the rule to drift. The detail lands at build time, governed by `MULTI-TENANCY.md`.

**Two things that fell out of the work**
- **`JobPosting.publicSlug` must stay globally unique and unguessable** — it resolves the tenant for the unauthenticated public form before any tenant is known. Sequential slugs would let someone enumerate tenants.
- **Feature 11's suppression threshold of 5 no longer makes sense as a fixed value** now that tenants vary in size — a 12-person tenant would see nearly everything suppressed. It becomes per-tenant configurable (OQ-T-06 supersedes OQ-1103).

**Outstanding**
- **OQ-319** (hosted vs per-company install) — blocks ingestion.
- **OQ-701** (pay components) and **OQ-601b** (entitlement days) — block seeding 07 and 06.
- **OQ-324** (new): can one tenant's locations span timezones? `Location.timezone` is reserved but unused; day bucketing is tenant-level per OQ-302.
- Step 6, the definition-of-done checklist.
- Retro-pass of 01–04 `ui-ux.md` files.

### 2026-09-18 — Session 18: Step 5b — fixing device ingestion security (OQ-318)

**Done**
- Wrote **`DEVICE-INGESTION-SECURITY.md`**, resolving OQ-318 with an architectural fix rather than a mitigation list.
- Verified against the manual first: HTTPS is available and enabled by default, but it authenticates the **server to the device** — the wrong direction. It does not close the gap.
- Wired the fix through the features it touches: 03 (D-08b, `Collector` model, the `/api/ingest/batch` and `/heartbeat` endpoints), 04 (`QUARANTINED` punch flag, FR-I-01b/c, and the gap detector no longer opening gaps while a healthy collector is buffering), and the overview.

**The fix**
- **A per-site collector.** A small service on the customer's LAN speaks iClock to the terminals, buffers durably, and forwards to the platform over HTTPS with a per-collector credential the platform issues, hashes, rotates and revokes.
- The reasoning is that the device cannot hold a secret and never will, so the architecture holds it instead. Three consequences make this better than a workaround:
  1. **The credential, not the serial number, resolves the tenant** — which closes the multi-tenancy hole directly. A forged serial can now only affect a tenant whose credential you already stole.
  2. **The unauthenticatable hop is confined to the LAN**, where physical and network access is already the trust boundary.
  3. **It removes WAN-outage attendance gaps.** Feature 04 currently classifies those days `UNKNOWN`; a buffering collector drains when the link returns. That is worth building on its own merits, independent of security.
- A site-to-site tunnel (WireGuard/IPSec) is offered as an **equal-strength alternative** for customers whose IT prefers it — better isolation, no software to maintain, but no buffering.
- **Defence in depth in every deployment**, including LAN-only: source-IP pinning with trust-on-first-use, a **quarantine** state for punches from unexpected sources, per-serial rate limits, plausibility checks, and alerting on new/concurrent source IPs.
- Quarantine deliberately follows the plan's existing instincts rather than inventing a new one — unmatched PINs are queued not dropped (04 D-07), device gaps are `UNKNOWN` not `ABSENT` (04 D-08). Keep the evidence; make a human decide the interpretation.
- Idempotency needed no change: the `(deviceId, userPin, punchTime)` unique constraint specified in Session 6 absorbs collector re-sends after a network failure, which is a small vindication of having specified it before anyone knew a collector would exist.

**Raised**
- **OQ-319 — hosted multi-tenant installation, or one install per company?** This decides whether the collector is necessary at all or the LAN is already the trust boundary. It blocks ingestion and should be answered first; multi-tenancy implies hosted, but it needs confirming.
- OQ-320…323: API key vs mTLS, whether the device's Server Address accepts a path, who installs and updates the collector, and buffer/alert thresholds.

**Recommendation recorded**
- Do not ship internet-exposed, serial-only ingestion. If the collector is not ready, keep ingestion LAN-only — feature 04's engine and shifts can be built and tested against synthetic punches regardless, so this blocks nothing else.

**Outstanding in Step 5b**
- Unchanged from Session 17: `tenantId` through 01, 04–11; the `Location` model; the no-deletion sweep; 01's Auth.js rewrite; emailed payslips in 07.

### 2026-09-18 — Session 17: Step 5b begins — multi-tenancy specification and the device-auth correction

**Done**
- Wrote **`MULTI-TENANCY.md`**, a cross-cutting specification that governs all eleven features. Written as one document rather than eleven edits deliberately: adding a `tenantId` column is the easy part, and the risky parts — unique constraints, the unauthenticated device endpoint, per-tenant jobs, cross-tenant aggregates, platform vs tenant admin — are identical in every feature and must not get eleven different answers.
- Applied the concrete corrections the manual forced: 03's `commKeyHash`/`commKeyLast4`/`commKeySetAt` and the firmware/timezone/drift columns are **commented out with their reasoning**, not deleted; 03's device authentication sequence is rewritten with step 4 removed and tenant resolution added; 02's `employeeCode` is now 1–14 alphanumeric and unique per tenant.
- De-singletonised 03's `Company`: it becomes one tenant's *profile*, while the new `Tenant` model is how the platform knows about a customer. Keeping both is deliberate — they answer different questions.

**Decisions taken in `MULTI-TENANCY.md` (D-T-01…D-T-10)**
- **Shared schema with `tenantId`, plus PostgreSQL Row-Level Security.** RLS is what makes the cheap isolation option acceptable: a query that forgets its predicate returns nothing rather than everything. That matters more here than usual — this database holds salaries, national IDs and performance reviews for multiple companies, and a cross-tenant leak is an incident with two customers in it. Database-per-tenant remains a legitimate reversal **if taken before the first migration**.
- **One user, one tenant** (two accounts for someone who works for two). `users.email` stays globally unique so login needs no tenant hint; nearly everything else becomes composite.
- **`devices.serial_number` stays globally unique** — it resolves the tenant, so it cannot be composite. Two tenants registering the same serial is a meaningful error, and the message must not reveal who holds it.
- **Platform operator vs tenant super admin** are separate. The operator sees tenants, provisioning and health — not employee, payroll or performance data. Support access into a tenant is granted by that tenant, time-boxed and audited.
- **Timezone moves to `Tenant`.** OQ-302's "one company, one timezone" now means one *per tenant*, which makes 04's day opener, 05's digests and 06's accrual per-tenant-timezone jobs. Easy to miss and it buckets attendance into the wrong day for every tenant outside the server's zone.

**The security consequence worth stating plainly**
- With no comm key (OQ-307) the serial number is now **both the device identifier and the tenant selector**. It is not a secret. Anyone who reaches the ingestion port and knows a registered serial can write attendance into that tenant. The sequence still refuses unregistered serials and can disable a stolen terminal, but it can no longer prove the sender is the device it claims to be. **OQ-318 is therefore a prerequisite for building ingestion, not a nice-to-have**, and per-tenant network isolation now looks stronger than one shared port.
- A small vindication: feature 08 is almost unaffected by multi-tenancy, because it computes nothing (08 D-01).

**Outstanding in this pass**
- `tenantId` through the remaining data models: 01, 04, 05, 06, 07, 09, 10, 11.
- The `Location` model (OQ-206) across 02, 03, 04, 11.
- Removing the automatic retention sweeps (OQ-1002) from 01, 05, 10, 11 and turning 10's purge into a report.
- Rewriting 01's auth endpoints around Auth.js v5, keeping database sessions.
- Emailed payslip PDFs in 07 (OQ-709), with the two proposed mitigations.
- Six new questions raised by the tenancy work: OQ-T-01…T-06.

**Next session**
- Continue 5b: `tenantId` and composite uniqueness through the remaining eight data models, then Location, then the no-deletion sweep.

### 2026-09-18 — Session 16: Open questions answered; manual found; plan corrections

**Done**
- The user answered every open question; anything not named explicitly takes its proposed default, per their instruction. Recorded in `OPEN_QUESTIONS.md` under a new **ANSWERED** section at the top.
- **Found and read the SenseFace 2A manuals** (`C:\Dev\Datasheets`), closing OQ-000 after five sessions as the plan's longest-standing blocker. The PDFs are CID-encoded and no PDF tooling is installed, so the text was extracted with a stdlib-only Python script (kept in the scratchpad).
- Corrected the decisions the answers and the manual overturn, in place, with the reasoning preserved rather than deleted: 03 D-02 (single company), 03 D-08 (per-device comm keys), 10 D-02 (automatic candidate purge).
- Added a **revision-pending banner** to `00-overview.md` so the feature files are not read as current while they still say single-company.

**What the manual settled**
- **OQ-204:** the user ID is *"1 to 14 digits by default, supporting both numbers and alphabetic characters"* — the plan assumed numeric, ≤ 9 digits. 02's validation changes. The device also refuses to change an ID after registration, independently confirming 02's immutability decision.
- **OQ-307 — the significant one:** Cloud Server Settings offers only Enable Domain Name, Server Address, Server Port and Enable Proxy Server. **"Comm Key" appears nowhere in either manual.** 03 D-08 is therefore not implementable: the device cannot present a shared secret, so a push can be authenticated only by serial number (not a secret), network path, and HTTPS. `commKeyHash` and the shown-once flow should be **removed rather than left as security theatre**, and the replacement is raised as **OQ-318**.
- **OQ-312:** confirmed — no firmware version, device timezone, or clock drift in the protocol surface. Those speculative columns come out of 03's `Device` model.

**Decisions and their consequences**
- **Multi-tenancy (OQ-301) is the largest change.** The whole plan is written single-company. Every model needs a tenant discriminator and every query, scope check, settings read, scheduled job and report needs tenant scoping. This is exactly why the question was Tier 1 — answering it now costs perhaps 10–15%; retrofitting it would have been close to a rewrite.
- **No automatic deletion (OQ-1002/105/203/908)** revises 10 D-02 and removes retention sweeps from 01, 05, 10 and 11. Retention becomes tracked and reported, with a person deciding. A deliberate liability trade, recorded as such — and the candidate consent notice must stop promising a deletion the system will not perform.
- **Emailed payslip PDFs (OQ-709)** override 05 D-06 for that one case. Proceeding as asked; two mitigations proposed for confirmation — password-protected PDFs, and no figures in the email body itself.
- **NextAuth (OQ-101)**: 01 D-04 survives, since Auth.js supports database sessions via an adapter. JWT sessions must be avoided or immediate revocation is lost.
- Smaller reversals from the defaults: MFA yes (102), multiple roles yes (106), no remembered devices with enforced logout (108), shared rate-limit storage (109), completeness indicator (214), multiple locations (206), re-sync pull available (415), employees cannot see colleagues' leave (803), managers can see notification read state (508), ledger append-only at database level (613), cancelling past leave needs approval (615), no draft history (910), no "who read my review" (909).
- **Sequencing:** recruitment (10) moves to the end of the build order, after 11.

**Environment note**
- `C:\Dev\CLAUDE.md` states Python is not installed. **That is no longer true** — Anaconda Python is at `C:\Users\Intelliglow\anaconda3\python.exe`, which is what made the manual readable. It also means `ui-ux-pro-max`'s `search.py` should now work, rather than needing the CSVs queried by hand.

**Outstanding**
- **Step 5b — the revision pass.** Seven answers overturn written decisions and the feature files still read as originally planned. Multi-tenancy alone touches all 11 `data-model.md` files.
- **Two answers still needed, neither defaultable:** OQ-701 (the actual pay components) and OQ-601b (entitlement days for Annual/Casual/Medical).
- **OQ-318** — how the device push endpoint is protected without a comm key.
- One ambiguity: OQ-416 was answered twice, "default" and "yes". Taking the default (no future-dated attendance rows).
- Retro-pass of 01–04 `ui-ux.md` files, still pending.

**Next session**
- Step 5b: revise the feature files, starting with multi-tenancy across the data models, then the device-auth correction in 03/04, then the smaller reversals.

### 2026-09-15 — Session 15: Step 5 — consolidated open questions

**Done**
- Wrote `OPEN_QUESTIONS.md`: every open question from all eleven features in one place, tiered by cost of leaving it unanswered, each with a proposed default.
- **Corrected the count.** Earlier sessions said "~130 open questions"; the actual total is **175**. Fixed in `00-overview.md` §8 and reflected here. The larger number does not change the picture — the tiering is what makes it workable.
- Structured it so it can be answered rather than admired: **Tier 1 (12 blocking)** → **Tier 2 (23, answer before the relevant feature)** → **Tier 3 (~140, defaults proposed, silence accepts)**. Added an "if you only answer eight things" section ordered by what each unlocks.
- Flagged the two questions needing someone outside this project: **OQ-702** (who owns keeping tax rates current — a payroll/tax adviser) and **OQ-1002** (candidate retention period — a data-protection adviser). Neither is a product decision and the plan deliberately does not guess at either.
- Marked which Tier 1 questions are *company* decisions (7), *technical* decisions (1), *external* (2), and which depend on the device vendor (OQ-000).

**Decisions / notes**
- No new design decisions — Step 5 is a consolidation pass.
- The most useful reframing: of 175 questions, **only 12 actually block work**. The plan already carries a defensible default for nearly everything else, so the user's job is to skim for what looks wrong rather than to answer a survey.
- Three Tier 3 items are worth surfacing despite having defaults, because their defaults have consequences that are easy to miss: OQ-209 (the document volume and backups — a database-only backup silently loses every uploaded contract), OQ-1103 (a suppression threshold of 5 suppresses most department-level reporting in a 40-person company), and OQ-910 (draft history in performance reviews — cheap now, awkward after the form is built).
- `IMPLEMENTATION_LOG.md`'s open-items table now points at `OPEN_QUESTIONS.md` as the working document and remains only as the historical record of when things were raised.

**Outstanding**
- **Awaiting the user's answers.** Nothing further in the plan depends on them, but the build does.
- Retro-pass of 01–04 `ui-ux.md` files — offered in Sessions 7, 13, and 14; not yet taken up.

**Next session**
- **Step 6**, the last planning step: the definition-of-done checklist. Then planning is complete and the decision is whether to build, and in what order.

### 2026-09-15 — Session 14: Step 4 — the root overview

**Done**
- Rewrote `00-overview.md` from the Step 2 skeleton into the document that describes what was actually planned: system summary, features and build order with dependencies, the shared data model, cross-feature contracts, cross-cutting concerns, migration sequence, known gaps, and a suggested starting order.
- **Named the nine patterns that recur across the eleven features**, which is the most useful thing in the document. They emerged during Step 3 rather than being designed up front, so writing them down is what turns them from a habit into a convention a new feature can follow: code-declared catalogues · facts vs derived opinions · snapshotting the rules · preview before commit · one writer per table · explicit visibility states · separate read logs for sensitive data · scope and suppression enforced in the query · refusing automated judgement about individuals.
- Assembled the **cross-feature contract table** — the ~14 helpers features owe each other. These matter more than the endpoints, and two carry build-order consequences worth repeating: 04 and 06 ship calling an `isPeriodLocked` stub that 07 later replaces, and 05 arrives owing a notification backlog from 01–04.
- Counted the **~35 scheduled jobs** the plan defines across features 03–11, which reframes the job runner as critical infrastructure. Listed the five whose failure is effectively silent — the day opener (04) being the worst, since absent employees simply never appear in the absence report.

**Decisions / notes**
- No new design decisions. Step 4 is a synthesis pass; where it seemed to be inventing something, that was a sign the feature file needed revisiting instead.
- The overview records **known gaps in the plan itself** rather than leaving them to be discovered: 01–04's un-grounded UI specs, no testing strategy, no deployment/operations plan beyond the existing Docker Compose (and no backup procedure, though 02 already notes that a database-only backup loses every uploaded document), no estimates, and the stale `dev-plan/` copy inside `hrm-system`.
- Section 10 gives a concrete starting order, including the recommendation to build 04's engine against synthetic punches and defer ingestion until the manual arrives — which keeps OQ-000 from blocking anything.

**Outstanding**
- Step 5 (consolidate and prioritise ~130 open questions) and Step 6 (definition of done).
- Retro-pass of 01–04 `ui-ux.md` files — offered again this session and not yet taken up.

**Next session**
- **Step 5**: the consolidated open-questions pass. The overview's §8 already names the ten that block or reshape work; Step 5 should produce the full list in a form the user can answer in one sitting, grouped by who can answer it — company decisions, technical decisions, and the two that need outside expertise (OQ-1002 data protection, OQ-702 tax rules).

### 2026-09-15 — Session 13: Step 3, feature 11 (Reports & Analytics) — **Step 3 complete**

**Done**
- Wrote all five files for feature 11 in `11-reports-analytics/`.
- **Step 3 is finished: all 11 features planned, 56 files** (feature 04 carries a sixth, `shifts.md`).
- Identified the two problems specific to reporting and designed around both: **aggregation leaks** (an average salary for a department of two is a salary disclosure dressed as a statistic) and **reports disagreeing with the modules they read from**.
- Made the report definition the whole contract: a declaration in code whose `source` is a **function call into the owning feature**, never SQL written in the reporting layer. A report needing a figure the owning feature cannot supply is a request to that feature, not a new calculation here.
- Specified suppression properly, including the step usually omitted: if exactly one group falls below the threshold, its value is derivable from the total, so the next-smallest group — and where necessary the total — must be suppressed too. Without that, the mechanism is decorative.

**Decisions / notes**
- 10 decisions as D-01…D-10 (see Decisions table).
- **`report.read` grants nothing on its own.** Every report additionally requires its module's permission, enforced at startup validation — a report declared without one fails the boot rather than shipping unguarded. This is what stops reporting becoming the most powerful permission in the system.
- **D-06 repays feature 02's assignment history.** "Headcount by department for March" must use March's departments; the obvious query joins to today's and is silently wrong at every restructure. `data-model.md` shows the wrong and right query side by side, because this is the requirement most likely to be implemented incorrectly.
- **No ad-hoc query builder** (D-04). Over this data model, with row- and field-level permissions, a query builder is a permission-bypass engine. This is the fifth code-declared catalogue in the plan, after permissions, settings, notification types, and self-service fields.
- Chart choices are grounded in the skill's `charts.csv` rather than habit — and it ruled out pie charts entirely for this product (high accessibility risk, wrong when precise values matter, and in an HR system the precise value is always what is wanted). Part-to-whole uses a labelled 100% stacked bar.
- Third feature in a row to refuse individual-level judgement: 09 D-04 (no attendance-derived performance scoring), 10 D-07 (no automated candidate screening), 11 D-09 (no scoring, ranking or prediction about individuals). Worth keeping consistent if any of the three is challenged.

**Process note**
- `11-reports-analytics/ui-ux.md` is skill-grounded per `C:\Dev\CLAUDE.md`, drawing on both `ux-guidelines.csv` and `charts.csv`. It notes that whoever writes the actual chart components should load the workspace `dataviz` skill first — this document decides which charts and why, not how to render them.
- **Features 01–04's `ui-ux.md` files still predate the workspace design rules** — outstanding since Session 7, now across four features and seven sessions.

**Outstanding**
- ~130 open questions raised across the 11 features. Step 5 is the consolidation and prioritisation pass, and it is now the most valuable remaining planning work.
- The gating questions that recur: OQ-000 (SenseFace manual), OQ-201 (DEPARTMENT scope definition), OQ-301 (multi-company), OQ-601 (leave types), OQ-701 (pay components), OQ-802/805 (portal on phones vs kiosk; language), OQ-1002 (candidate retention — needs a qualified answer).
- Retro-pass of 01–04 `ui-ux.md` files, still pending.

**Next session**
- **Step 4**: write the root `00-overview.md` properly — system summary, the shared data model across all 11 features, cross-cutting concerns (auth, RBAC, the code-declared-catalogue pattern, snapshotting, the job runner, the SenseFace dependency), and the build-order rationale.
- Then Step 5 (consolidated open questions) and Step 6 (definition of done).

### 2026-09-15 — Session 12: Step 3, feature 10 (Recruitment & Onboarding)

**Done**
- Wrote all five files for feature 10 in `10-recruitment-onboarding/`. Only feature 11 remains in Step 3.
- Treated the feature as two halves joined by one hinge: recruitment (people outside the company) and onboarding (a new employee before and after day one), meeting at the **hire** — specified step by step in the README because that transition carries most of the feature's risk.
- Specified the hire as a **previewed conversion** that calls feature 02's employee-creation path rather than writing employee data itself, with an explicit "we won't" list (no salary, no shift, no leave policy) that also becomes onboarding tasks so the omissions are visible rather than silent.
- Carried feature 09's hidden-until-submitted discipline across to interview scorecards, including not revealing *who* is outstanding — naming the laggard turns structured interviewing into pressure to agree.

**Decisions / notes**
- 10 decisions as D-01…D-10 (see Decisions table).
- **D-02 inverts feature 02's retention stance, deliberately.** Employees are never deleted; candidates are deleted on a schedule unless hired or opted in. A rejected candidate's file kept indefinitely for no reason is the largest quiet liability this system could accumulate — hundreds of people's data held by a company they have no relationship with, that nobody remembers is there. The purge is the only scheduled job in the whole plan that deletes anything but logs: it warns HR a week ahead, previews with counts and never names, and requires the count typed back.
- **The public application form is the system's only unauthenticated, internet-facing write surface.** Feature 03's device endpoint is at least key-authenticated and LAN-reachable; this one is open. It is specified with its own handler, its own rate limiter, no shared code path with authenticated endpoints, content-sniffed uploads, and no CAPTCHA that blocks assistive technology.
- Rejection stores three separate fields — reason code, internal note, message sent — and the API refuses to send the internal note as the message. One field for both is how "weak on the technical round, poor culture fit" reaches a candidate.
- The pipeline board is the plan's clearest case of the drag-alternative guideline: every card has a keyboard-operable *Move to* menu, with drag as an enhancement.
- Consent stores the **notice text as shown**, not a version reference — proving what someone was told requires the words they saw.

**Process note**
- `10-recruitment-onboarding/ui-ux.md` is skill-grounded per `C:\Dev\CLAUDE.md`. The public form is the accessibility high-water mark of the whole system: applicants include people with disabilities, and an inaccessible application form is an accessible way to be sued.
- Features 01–04's `ui-ux.md` files still predate that instruction — outstanding since Session 7.

**Outstanding**
- All prior open items. New: OQ-1001…1015.
- **OQ-1002 needs a qualified answer, not a product decision**: the candidate retention period is jurisdiction-specific and the plan deliberately does not guess. It is configured as a setting so the answer can be applied without a migration.
- **OQ-1001 is worth asking before any of this is built**: a company hiring a handful of people a year runs recruitment in a spreadsheet and is right to. Onboarding alone may be the whole useful half.
- Retro-pass of 01–04 `ui-ux.md` files, still pending.

**Next session**
- Step 3, feature 11 (Reports & Analytics) — the last. It reads from every other module and must inherit each one's visibility rules, which is the main design problem. Then Steps 4–6: the root overview, the consolidated open-questions pass, and the definition-of-done checklist.

### 2026-09-15 — Session 11: Step 3, feature 09 (Performance Management)

**Done**
- Wrote all five files for feature 09 in `09-performance-management/`.
- Recognised that this feature records **opinions**, not facts, which changes the design problems: correctness is not a meaningful concept here, fairness is; data is deliberately withheld rather than shown as soon as it exists; and audit must answer "who read this" as well as "who changed it".
- Made visibility the governing section of `requirements.md` (FR-V), stated to override every other requirement where they conflict, and implemented it as a single `reviewVisibilityFilter` composed into every query — never a post-fetch filter.
- Separated **submit** from **share** for manager reviews, and blocked sharing before the self-review is in (unless the deadline passed or HR overrides) — otherwise the self-review is written in response to the rating.
- Specified acknowledgement as two equally-weighted options, "I've seen this" and "I've seen this and I don't agree". There is no API path that discards a disagreement and none that records agreement the employee did not give.

**Decisions / notes**
- 10 decisions as D-01…D-10 (see Decisions table).
- **D-04 is the one to defend:** no automatic scoring from attendance, leave, or payroll data — enforced structurally by the absence of any read path, not merely by not building a feature. `requirements.md` §FR-X records four explicit non-requirements (no attendance-derived scoring, no computed overall rating or ranking, no notifying managers that someone is drafting a self-review, no engagement metrics) so that "we decided not to" survives the first feature request, where "we didn't get to it" would not.
- **`performance.read_content` is granted to nobody by default — including HR admin.** Cycle progress and completion are visible without it; content is not. That should survive review.
- Anonymous feedback keeps its author in the database while hiding it in display. Anonymity is a display choice; an anonymous channel with no stored author cannot be investigated when it is used to abuse someone, and deleting the author would provide only deniability. The minimum-response threshold is what makes the anonymity real.
- Autosave is treated as a correctness requirement, not a convenience: a manager who loses forty minutes of writing does not write it again.
- This is the fourth feature to use the **snapshot** pattern (template and rating-label snapshots, after 07's payslip, 06's approval chain, 04's shift snapshot) and the second to need a **separate read log** distinct from 01's audit (after 07's payslip access log).

**Process note**
- `09-performance-management/ui-ux.md` is skill-grounded per `C:\Dev\CLAUDE.md` (multi-step progress, visible labels and required indicators, bounded line length for long text, confirmation for irreversible actions, drag alternatives, cancellable state transitions for autosave, realistic sample content).
- Features 01–04's `ui-ux.md` files still predate that instruction — outstanding since Session 7.

**Outstanding**
- All prior open items. New: OQ-901…912.
- **OQ-901 gates the shape of the build**: if there is no formal review cycle, most of this feature is unbuilt and it becomes goals plus feedback.
- Retro-pass of 01–04 `ui-ux.md` files, still pending.

**Next session**
- Step 3, feature 10 (Recruitment & Onboarding): job postings, candidates, interview pipeline, and the new-hire checklist that ends by creating an employee in feature 02. Last of the Step 3 features before 11 (Reports), then Steps 4–6.

### 2026-09-15 — Session 10: Step 3, feature 08 (Employee Self-Service) — all must-haves now planned

**Done**
- Wrote all five files for feature 08 in `08-employee-self-service/`.
- **All eight must-have features (01–08) are now planned.** Remaining: 09 and 10 (nice-to-have), 11 (should-have).
- Held the feature to one constraint: **no new domain logic**. The whole portal is three new endpoint groups (dashboard aggregate, profile, change requests) plus four `/api/me/*` façades. `api-design.md` carries an explicit table of every screen and what it calls, so the constraint is checkable during the build rather than aspirational.
- `data-model.md` is the shortest in the plan — two models — and that brevity is the evidence the constraint holds.
- Specified the plain-language translation layer as a table: every internal status from feature 04 mapped to what an employee reads. The one that matters most is `UNKNOWN` → "The terminal wasn't working — this isn't counted against you", which is where 04's care about never marking people absent becomes visible to the person it protects.

**Decisions / notes**
- 9 decisions as D-01…D-09 (see Decisions table).
- **This feature inverts most of the design assumptions made so far.** Every other feature serves trained HR staff at a desk, daily. This one serves the whole workforce, on phones, a few times a month, with no training — so: one thing per screen, bottom navigation, plain language, mobile-first at 375 px, and progressive disclosure over completeness.
- **Bank details are absent from the field catalogue entirely**, not set to read-only. Their absence is the enforcement — no configuration value exists that could enable editing them, which is stronger than a policy someone could change (07's fraud-vector reasoning).
- `fieldKey` on a change request is a string validated against a code-declared catalogue — the fourth appearance of that pattern after permissions (01), settings (03), and notification types (05). It is now the house answer to "user-referenced identifiers into code-defined things".
- Approving a change request applies it **through feature 02's normal update path**, so 02's validation and audit fire exactly as for an HR edit. If 02 rejects the value, the approval fails rather than marking the request approved over an unchanged record.
- Managers: approvals stay in features 04 and 06, reached via the admin shell; the portal dashboard only carries a prompt linking there. Revisit if managers turn out never to open the admin shell (OQ-806).
- `GET /api/directory` is specified in this feature but belongs to feature 02 — flagged as OQ-811 to be added there when built.

**Process note**
- `08-employee-self-service/ui-ux.md` is skill-grounded per `C:\Dev\CLAUDE.md`, with the mobile/responsive guidelines carrying most of the weight (mobile-first, touch targets and spacing, `dvh` not `100vh`, fixed-element and safe-area overlap, tap delay and overscroll, `inputmode`, 16 px body text, back-button behaviour).
- Features 01–04's `ui-ux.md` files still predate that instruction — outstanding since Session 7, now across four features.

**Outstanding**
- All prior open items. New: OQ-801…812.
- **OQ-802 (phones vs shared kiosk) and OQ-805 (language) both need answering before this feature is built** — either answer changes the design materially rather than at the margins.
- Retro-pass of 01–04 `ui-ux.md` files against the skill guidelines, still pending.

**Next session**
- Step 3, feature 09 (Performance Management): goals/KPIs, review cycles, feedback. First of the nice-to-haves, and the first feature with no dependency on the attendance/leave/payroll spine — so it is largely self-contained.

### 2026-09-15 — Session 9: Step 3, feature 07 (Payroll)

**Done**
- Wrote all five files for feature 07 in `07-payroll/`.
- Built the feature on **snapshotting**: at calculation, a payslip captures its inputs *and the rules* — compensation, component definitions, formula text, attendance and leave figures, period metadata. A finalised payslip references nothing live and renders identically years later after salaries, formulas, and departments have all changed.
- Specified the run lifecycle (`DRAFT` → `CALCULATED` → `APPROVED` → `FINALISED` → `PAID`) as a server-enforced state machine, with approval and finalisation as two distinct recorded acts even when one person holds both permissions.
- **Delivered the period lock that features 04 and 06 have been deferring to since Sessions 6 and 8.** Both shipped calling an `isPeriodLocked` stub; this feature supplies the table, the helper (returning the run and period, not just a boolean, so their error messages can name them), and the adjustment mechanism that makes their "create a payroll adjustment instead" advice true.
- Specified the formula language as a **closed grammar** — arithmetic, comparisons, `min`/`max`/`round`, bracket lookups — with validation and a test endpoint that shows intermediate values.

**Decisions / notes**
- 10 decisions as D-01…D-10 (see Decisions table).
- **D-02 is a security decision, not an ergonomic one:** a formula field that can execute code is a remote-code-execution hole with an HR admin's credentials in front of it. The closed grammar is also what makes formulas testable and explainable.
- **D-06 concentrates the sensitivity:** `MANAGER` holds no payroll permission at all — feature 01's role grid already reflects this, and it should be verified rather than assumed at review. Payslip *reads* are logged in their own table, not 01's audit log, because read-logging would otherwise drown the change records that log exists for.
- The `inputSnapshot` costs ~2–5 KB per payslip, about 1 MB a year at this headcount. Trivial cost for permanent explicability.
- Every notification this feature registers with 05 is figure-free (FR-L-05, 05 D-06). The denylist in 05's template validation must include every monetary variable — that is a concrete dependency, not a principle.
- No scheduled job calculates, approves, finalises, or pays. Every step that moves money is taken by a person.

**Process note**
- `07-payroll/ui-ux.md` is skill-grounded per `C:\Dev\CLAUDE.md` (progress indicators for the run lifecycle, bulk-action patterns for acknowledgements, wide-table handling for variance, chip reflow for formula identifiers, drag alternatives for calculation order, never colour alone for negative amounts).
- Features 01–04's `ui-ux.md` files still predate that instruction — outstanding since Session 7.

**Outstanding**
- All prior open items. New: OQ-701…718.
- **OQ-701 and OQ-702 gate the build**: the real components, and who owns keeping any configured tax table current. A stale tax table is worse than none, because it produces confidently wrong deductions.
- Retro-pass of 01–04 `ui-ux.md` files against the skill guidelines, still pending.

**Next session**
- Step 3, feature 08 (Employee Self-Service Portal). It is mostly a re-presentation of 02, 04, 06, and 07 for the employee's own record, so the main design questions are shell, navigation, and what an employee may see and do — not new domain logic.

### 2026-09-15 — Session 8: Step 3, feature 06 (Leave Management)

**Done**
- Wrote all five files for feature 06 in `06-leave-management/`.
- Built the feature on a **ledger**: every movement (grant, accrual, carry-over, reservation, taken, refund, adjustment, expiry, encashment) is an immutable signed entry, and the balance is their sum with a cached snapshot for speed. Same family of decision as 04's punches-vs-computed-days split.
- Specified the four-part balance — entitled / used / pending / available — because collapsing them into one number is what produces disputes.
- Specified the reservation mechanism: submitting writes a `RESERVATION` entry inside the same transaction as the balance check, so two concurrent requests against a 2-day balance cannot both succeed.
- Made every engine run (accrual, carry-over, expiry, settlement) idempotent by database constraint — `[employee, type, periodKey, kind]` unique — rather than by the engine remembering, and gave each a preview that writes nothing.
- Recorded the two obligations feature 04 left here (04 OQ-410) as **requirements, not notes**: FR-X-01 (leave marks attendance days dirty through 04's `markDirty`) and FR-X-03 (a bulk attendance recompute is part of this release, so days that computed `ABSENT` for want of leave data are corrected before anyone reports on them).

**Decisions / notes**
- 10 decisions as D-01…D-10 (see Decisions table).
- **D-09 keeps the 04/06 boundary clean:** leave never writes to `attendance_days`. One writer per table; two features writing attendance is how a recompute and a leave approval start fighting.
- **D-10 matches a promise feature 03 already made:** declaring a holiday over approved leave surfaces the impact but never silently refunds it. Automatic refunds would rewrite balances people have planned around.
- The partial unique index enforcing "one approved leave day per employee per date" **cannot be written as specified** — Postgres does not allow a subquery in an index predicate. It becomes a denormalised `is_approved` boolean on `LeaveRequestDay` maintained with the request status. Noted in `data-model.md` because it changes the model, not just the SQL.
- `LeaveRequestDay` is the integration surface: 04 and 07 both read it, neither re-derives a date range, neither touches the ledger.
- Carry-over previews put **forfeited days** next to carried-over days in the totals. A carry-over run is the moment a company discovers it is about to take 88 days off its staff, and that should be visible before the click.

**Process note**
- `06-leave-management/ui-ux.md` is skill-grounded per `C:\Dev\CLAUDE.md`, citing the `ui-ux-pro-max` guidelines it applies (drag alternatives for calendar range selection, minimum target size, never colour alone, inline announced errors with recovery paths, confirmation for destructive actions, reserved space for the live cost panel).
- Features 01–04's `ui-ux.md` files still predate that instruction and remain un-grounded — carried forward from Session 7 as an outstanding item.

**Outstanding**
- All prior open items. New: OQ-601…618.
- **OQ-601 gates everything**: without the real leave types and entitlements, policies cannot be seeded and every screen in this feature is illustrative.
- Retro-pass of 01–04 `ui-ux.md` files against the skill guidelines, still pending.

**Next session**
- Step 3, feature 07 (Payroll): configurable salary structures, allowances and deductions, pay runs, payslips. It consumes attendance hours and overtime (04), unpaid leave and encashment (06), and job/grade from 02 — and it owns the period lock that 04 and 06 both already defer to.

### 2026-09-15 — Session 7: Step 3, feature 05 (Notifications)

**Done**
- Wrote all five files for feature 05 in `05-notifications/`.
- Opened the README with the **backlog already owed**: 01 (invite + password reset, currently only logged), 02 (expiring documents), 03 (device offline, thin holiday coverage), 04 (unmatched PINs, device gaps, pending corrections). Feature 01's invite email is the urgent one — a real invite flow does not exist until this ships.
- Reduced the whole integration surface to one typed call, `notify(typeKey, { context, idempotencyKey, … }, tx)`, plus `supersede()` for notifications that someone else has already handled. Callers choose nothing about recipients, channels, wording, or timing.
- Specified recipient rules as composable helpers (`subjectEmployeeManager()`, `permissionHolders(key, scope)`, `union(...)`) that re-check visibility at send time, so a broadly-granted notification cannot become an information leak.
- Split `NotificationEvent` (one per trigger) from `NotificationDelivery` (one per recipient per channel), and deferred recipient resolution to the worker so `notify()` adds a single insert to the caller's transaction.

**Decisions / notes**
- 10 decisions as D-01…D-10 (see Decisions table).
- **D-05 (idempotency keys) is what keeps this feature from becoming spam:** without it, a device offline for a week sends 168 identical emails.
- **D-03/D-06 are the two that protect trust:** in-app records are always created even when email is muted (so "I was never told" stays answerable), and email bodies never carry sensitive data — the temptation for that arrives with feature 07.
- Every non-send writes a row with a reason (`SKIPPED` — muted, suppressed, no address). There is no path where a notification simply fails to appear, which is why the admin health strip puts `skipped` on the front page next to `failed`.
- `notification.deliveryMode` is an explicit setting, never inferred from `NODE_ENV` — a container pointed at a copy of production data with the wrong mode is how an entire company gets test emails.
- The migration **drops feature 01's temporary password-reset outbox** in the same release. Two outboxes in one system is how a reset email goes missing.

**Process note**
- `C:\Dev\CLAUDE.md` took effect this session and mandates skill-grounded UI work. `05-notifications/ui-ux.md` was written against `ui-ux-pro-max` and cites the specific guidelines it applies (live badge semantics, stable count slot, badge-vs-chip semantics, inline validation on blur, focus visibility). Python is not installed on this machine, so `search.py` is unusable — the skill's CSV data was queried directly with Grep, exactly as the workspace file instructs.
- Features 01–04's `ui-ux.md` files were written before that instruction existed. They are not wrong, but they are not skill-grounded either — worth a pass against the same guidelines before any of them is built.

**Outstanding**
- All prior open items. New: OQ-501…514.
- **OQ-502 is the one to answer first** — whether all employees actually have work email. It affects whether this feature reaches the workforce at all.

**Next session**
- Step 3, feature 06 (Leave Management): leave types, policies, balances, accrual, requests, approval flow, team calendar. It also carries an obligation from feature 04 (OQ-410): a bulk recompute of attendance once leave data exists, since days that should be `ON_LEAVE` currently compute as `ABSENT`.

### 2026-09-15 — Session 6: Step 3, feature 04 (Attendance Tracking)

**Done**
- Wrote **six** files for feature 04 — the five standard ones plus `shifts.md`, the split file agreed in Step 1 (Session 1 decision). The per-feature table above has five columns; `shifts.md` is additional and complete.
- The whole feature is built on one split: **punches are immutable facts, the daily record is a recomputable opinion**. Every other decision follows from it.
- Specified the classification engine as an ordered rule list (FR-C-08), a reason trail stored per day so any verdict can be explained in the UI, and a purity requirement (FR-C-20 / NFR-09) making the engine unit-testable with no database, device, or clock.
- Specified the shift-anchored day window that makes night shifts work, including the overlapping-window resolution rule and what happens to punches outside every window.
- Specified the dirty-day queue, its triggers across features (03 marks days dirty when a holiday moves; 06 when leave is approved, via one shared `markDirty` helper), and a recompute **preview** so HR sees what would change before committing.

**Decisions / notes**
- 10 decisions as D-01…D-10 plus 8 shift decisions as D-S-01…D-S-08 (see Decisions table).
- **D-08 is the one to defend hardest:** a device gap classifies days `UNKNOWN`, never `ABSENT`. Marking 200 people absent because a terminal lost power is the most damaging thing this system could do, and the UI is designed so `UNKNOWN` never reads as an absence.
- `resultHash` on the computed day is what makes recompute safe to run routinely — an unchanged recompute writes nothing at all, so a nightly full pass does not bury real changes under thousands of no-op audit rows.
- The **day opener** job is the least obvious piece: without it an absent employee generates no punch, so nothing marks their day dirty, so no row is written and they never appear in the absence report. Its failure is hard to notice, so its last-run time belongs on the status endpoint.
- The `attendance` → `attendance_punches` migration **must be hand-written**; an auto-generated Prisma migration will drop and recreate the table rather than rename it, losing the device-test rows.
- **OQ-000 (missing manual) hits hardest here.** Every ingestion detail in `api-design.md` is marked ⚠ and isolated behind one adapter module. Recommended build order *within* the feature: computation engine and shifts first against synthetic punches, ingestion last once the manual is in hand — which turns the blocker into a scheduling detail rather than a stop.

**Outstanding**
- All prior open items. New: OQ-401…420 and OQ-S-01…S-06.
- OQ-401 (night shifts) and OQ-S-01 (rotating patterns) both materially change the size of this build and are worth answering early.

**Next session**
- Step 3, feature 05 (Notifications): in-app and email alerts, templates, per-user preferences. Several features already queue notifications that currently only log — 02 (expiring documents), 03 (device offline, thin holiday coverage), 04 (unmatched PINs, device gaps, pending corrections) — so 05 has a backlog of defined senders to satisfy.

### 2026-09-15 — Session 5: Step 3, feature 03 (Admin & Settings)

**Done**
- Wrote all five files for feature 03 in `03-admin-settings/`: company profile, localisation, work week, holiday calendars, the settings registry, and SenseFace device registration/monitoring.
- Drew the 03/04 boundary explicitly in a table in the README: 03 authenticates the `/iclock/*` request and owns device identity and health; 04 parses the body and decides what a punch means.
- Specified the full device authentication sequence (serial lookup → status → constant-time key compare → `lastSeenAt` → hand off), so feature 04 builds ingestion on top without re-deciding any of it.
- Defined the helpers 04/06/07/11 will all depend on: `getSetting`/`setSetting`, `isWorkingDay`/`workingDaysBetween`, and the timezone helpers (`companyToday`, `toCompanyDate`, `startOfCompanyDay`).

**Decisions / notes**
- Nine decisions as D-01…D-09 (see Decisions table). The two with the longest reach: D-03/D-04 (UTC storage, timezone applied at the edges, changing it is change-controlled) and D-05 (one `isWorkingDay` implementation — three modules deciding independently what a holiday is guarantees three different answers, found at payroll time).
- **`device_events` is the fastest-growing table in the system** — a terminal polling every 30s would produce ~2 880 rows/day/device if every poll were logged. Two required mitigations are specified: log `CONNECTED` only on transitions after a stale gap, plus a nightly retention sweep. Without the first, one terminal writes over a million rows a year to record nothing.
- **The migration has a visible side effect worth putting in release notes:** existing `devices` rows get a null `comm_key_hash`, so push testing stops working until keys are issued. That is correct behaviour, but it will be reported as a bug otherwise.
- `.env`'s single `DEVICE_COMM_KEY` is superseded by per-device keys and should be removed — unless OQ-307 says the firmware only supports one shared key, in which case it returns as a setting rather than an env var so it can be rotated without a redeploy.
- This feature is the first to genuinely need a scheduled-job runner (device health, retention sweeps, recurring holiday generation), so the runner is built here. Where it lives is OQ-315.

**Outstanding**
- OQ-000 (missing SenseFace manual) now blocks two feature-03 items directly — OQ-307 (comm-key mechanism) and OQ-312 (which device fields the protocol exposes). It will block feature 04 considerably harder.
- New: OQ-301…317. OQ-301 (multi-company) is the cheapest decision on the list to make now.

**Next session**
- Step 3, feature 04 (Attendance Tracking): the largest feature so far — push ingestion, shifts (as the agreed split file `shifts.md`), daily attendance computation, and manual corrections. Expect OQ-000 to constrain the ingestion half of `api-design.md`.

### 2026-09-15 — Session 4: Step 3, feature 02 (Employee Management)

**Done**
- Wrote all five files for feature 02 in `02-employee-management/`, following the conventions set in Session 3.
- Specified the migration away from the scaffold's free-text `department`/`position` columns to FK models, including the backfill and the one irreversible step (dropping the original strings, which collapses case/whitespace variants).
- Defined the two helpers the rest of the system depends on: `resolveScopedEmployeeIds` (the recursive CTE behind feature 01's `employeeScopeFilter`) and `selectEmployeeFields` (permission-driven Prisma select, so sensitive columns are never fetched rather than fetched-and-stripped).
- Added 13 permission keys to the catalogue, with their default role grants.

**Decisions / notes**
- Nine decisions recorded as D-01…D-09 (see the Decisions table). The two worth challenging: D-03 (reporting line is its own field, not derived from department headship) and D-05 (salary lives in feature 07, not on the employee record).
- Feature 02 makes `DEPARTMENT` scope functional for the first time — which surfaced OQ-201, the most consequential open question in the plan so far. Feature 01 assumed the manager chain; a department head who does not line-manage everyone in their department would then see less than expected. The plan proposes the union of both, but this needs a real answer before either feature is built.
- The employee profile's tab structure is defined now (Overview / Job & history / Documents / Attendance / Leave / Payroll) so that features 04, 06, and 07 add tabs rather than redesigning the page.
- Document storage needs a volume in `docker-compose.yml` and coverage in the backup story — a database-only backup silently loses every contract (OQ-209).

**Outstanding**
- All of Session 3's open items remain. New: OQ-201…214.
- OQ-204 (`employeeCode` format) is blocked on OQ-000, the missing SenseFace manual — the same blocker that will hit feature 04 harder.

**Next session**
- Step 3, feature 03 (Admin & Settings): company profile, holiday calendar, work week, system config, and SenseFace device registration/monitoring.

### 2026-09-15 — Session 3: Step 3, feature 01 (Roles, Permissions & Auth)

**Done**
- Read the existing plan and the app scaffold; confirmed with the user that `C:\Dev\dev-plan` is the authoritative copy.
- Wrote all five files for feature 01 in `01-roles-permissions-auth/`: `README.md`, `requirements.md`, `data-model.md`, `api-design.md`, `ui-ux.md`, each with an Open Questions section.
- Established the documentation conventions the remaining ten features will follow: README carries purpose/scope/decisions/dependencies; requirements use ID-prefixed tables (FR-A/FR-Z/FR-U/FR-L, NFR) plus feature-level acceptance criteria; data-model gives real Prisma matching the existing schema's naming, plus seed and migration notes; api-design gives endpoint tables, payloads, error codes, and the shared server helpers; ui-ux gives a screen map, per-screen states, a11y, and a copy reference.

**Decisions / notes**
- Seven design decisions recorded as D-01…D-07 (see the Decisions table above). All are proposals awaiting user review — particularly D-04 (DB sessions, not JWTs) and D-03 (scope on the role→permission grant), since both are expensive to reverse later.
- Feature 01 defines the contract the other ten depend on: the permission catalogue (`module.action`), `requirePermission`, `employeeScopeFilter`, and `writeAudit(tx, …)`. Each later feature's Step 3 adds its own permission keys to the catalogue.
- `DEPARTMENT` scope deliberately resolves to an empty set until feature 02 supplies reporting lines — it must never fall back to "everyone".
- Environment additions the build will need: `SEED_ADMIN_EMAIL`, `SEED_ADMIN_PASSWORD`, `SESSION_IDLE_MINUTES`, `SESSION_ABSOLUTE_DAYS`.
- No application code was written; planning is not finished (Steps 4–6 remain).

**Outstanding**
- OQ-003 (token revocation) still open from Session 2.
- New this session: OQ-004 (duplicate plan copies), OQ-005 (no git on this machine), OQ-006 (missing source docs), OQ-101…115 (feature 01).

**Next session**
- Step 3, feature 02 (Employee Management): all five files. It resolves `DEPARTMENT` scope, so settle OQ-106 first if possible.

### 2026-09-14 — Session 2: Commit planning work, publish to GitHub

**Done**
- Committed Session 1's planning output (`dev-plan/` and the `CLAUDE.md` log rule) to `main` as `b851e80`.
- Installed GitHub CLI (`gh` 2.100.0) via Homebrew; user authenticated it themselves via the browser flow (account `intelliglowsolutions-afk`, scopes `repo`, `workflow`, `read:org`, `gist`).
- Confirmed the existing remote repo `intelliglowsolutions-afk/hrm-system` (private) was empty — no branches, size 0 — so nothing could be overwritten.
- Added `origin` and pushed `main`. Verified on the remote: both commits present, 32 files, and the only env file is `.env.example` (placeholder values only; real `.env` is ignored by the `.env*` rule).

**Decisions / notes**
- Claude does not authenticate as the user or handle access tokens; `gh auth login` was run by the user. This stays the pattern for future credential steps.
- Empty dirs are not tracked by git, so the 11 feature folders appear on the remote only once Step 3 writes files into them.

**Outstanding**
- The token pasted into chat on 2026-09-14 is still to be revoked (OQ-003).

**Next session**
- Step 3, feature 01 (Roles, Permissions & Auth): all five files plus Open Questions.

### 2026-09-11 — Session 1: Planning kickoff (Steps 0–2)

**Done**
- Read `HRM_SYSTEM_PLANNING_INSTRUCTIONS.md`, `HRM_SYSTEM_DEPLOYMENT.md`, and the current scaffold (`prisma/schema.prisma` with `Employee`, `Device`, `Attendance`; placeholder `/api/device/iclock` route).
- Created `dev-plan/` and this implementation log.
- Added a rule to `CLAUDE.md` requiring every session to update this log.
- Step 1: drafted the feature list and build order; user confirmed all 11 features, shifts inside Attendance, generic payroll.
- Step 2: created the 11 numbered folders and a `00-overview.md` skeleton that links to each one.

**Confirmed feature list (build order)**

| # | Feature | One-line description | Why this position |
|---|---|---|---|
| 01 | Roles, Permissions & Auth | Login, sessions, RBAC (roles → permissions), audit log | Every other feature checks permissions and writes audit entries |
| 02 | Employee Management | Employee records, departments, positions, reporting lines/org chart, documents | Core entity every module references |
| 03 | Admin & Settings | Company profile, holiday calendar, work-week and system config, SenseFace device registration/monitoring | Holidays + devices are needed before attendance and leave |
| 04 | Attendance Tracking | SenseFace push-mode ingestion, shifts/work schedules, daily attendance (late/absent/overtime), manual corrections | Needs employees, devices, holidays |
| 05 | Notifications | In-app + email alerts, templates, per-user preferences | Shared infra for leave approvals, missed punches, etc. |
| 06 | Leave Management | Leave types, policies, balances, requests, approval flow, team calendar | Needs employees, holidays, notifications; feeds payroll |
| 07 | Payroll | Configurable salary structures, allowances/deductions, pay runs, payslips | Needs employees, attendance, leave |
| 08 | Employee Self-Service Portal | Employee-facing view of own profile, attendance, leave, payslips | Front-end over 02/04/06/07 |
| 09 | Performance Management | Goals/KPIs, review cycles, feedback | Independent; lower priority |
| 10 | Recruitment & Onboarding | Job postings, candidates, interview pipeline, new-hire checklist → creates employee | Independent; lower priority |
| 11 | Reports & Analytics | Attendance/leave/payroll dashboards and exports | Needs data from all modules, so last |

**Next session**
- Step 3, feature 01 (Roles, Permissions & Auth): write all five files plus Open Questions.
- Then continue through features in build order, updating the per-feature table after each file.
