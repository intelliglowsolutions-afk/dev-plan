# Definition of Done

**Step 6 of the planning process.** What "done" means for a feature, for the system, and for going
live.

---

## How this document is organised

A generic checklist ("code reviewed, tests pass") would be true of any project and useful to none.
This one is built around the failure modes **this** system actually has, which came out of writing
the eleven features:

- Most of its dangerous failures are **silent** — a job stops, a snapshot drifts, a cache lies, a
  backup misses a volume. Nobody finds out for weeks, and by then payroll has run on bad data.
- Its two most sensitive boundaries — **tenant isolation** and **pay/performance visibility** — fail
  open if a single `WHERE` clause is forgotten.
- Several features **owe things to features built later**, so "done" for one is not "done" for all.

So: Part 1 is the per-feature checklist. **Part 2 is the part that matters** — the silent-failure
register. Parts 3–5 are the gates and the go-live list.

---

## Part 1 — A feature is done when

Applies to every feature 01–11. Anything unchecked means the feature is not done, not that it is
done with caveats.

### Data

- [ ] Prisma models match the feature's `data-model.md`, including the revisions in
      `MULTI-TENANCY.md` and `DEVICE-INGESTION-SECURITY.md`.
- [ ] Every tenant-owned table has `tenantId`, a composite index leading with it, and an **RLS
      policy**. A test proves a query without the predicate returns nothing rather than everything.
- [ ] Composite uniqueness applied per `MULTI-TENANCY.md` D-T-09 — and the deliberate exceptions
      (`users.email`, `devices.serial_number`, `job_postings.public_slug`) are still global.
- [ ] Migration written, reviewed, and **run forwards on a copy of real data**. The irreversible
      ones (02's column drop, 04's table rename) are hand-written and their SQL read before running.

### Access

- [ ] Every endpoint declares a permission. The shared wrapper makes an undeclared one impossible to
      register.
- [ ] Scope is applied **in the query**, never as a filter over fetched rows.
- [ ] A test proves an actor at `DEPARTMENT` scope sees exactly their transitive reports —
      across a three-level chain, two departments, and **two tenants**.
- [ ] Denied requests return 404 where existence itself is sensitive, 403 otherwise.
- [ ] Sensitive fields are absent from responses, not nulled.

### Behaviour

- [ ] The feature's own acceptance criteria — the numbered list at the end of its `requirements.md`
      — all pass as automated tests. These were written to be executable, not aspirational.
- [ ] Every state-changing endpoint writes an audit entry **inside the same transaction**.
- [ ] Every notification the feature promised is registered in 05's catalogue and fires.
- [ ] Every scheduled job the feature defines is registered, reports its last successful run, and is
      covered by Part 2.

### Interface

- [ ] All six states implemented: loading, empty, filtered-empty, error, permission-denied, saving —
      plus the feature-specific ones (`computation pending`, `quarantined`, `suppressed`).
- [ ] Accessibility: keyboard operable, visible focus, 4.5:1 contrast, no meaning by colour alone,
      `prefers-reduced-motion` honoured, 44×44px touch targets.
- [ ] Reviewed at 375px. For feature 08, **designed** at 375px.
- [ ] Copy matches the feature's copy reference table. That table is a specification, not a
      suggestion — it is where the plan's care about wording actually lands.
- [ ] UI work is grounded in the workspace skills per `C:\Dev\CLAUDE.md`.

### Honesty

- [ ] No figure displayed anywhere is computed by a feature that does not own it.
- [ ] Nothing is presented as a fact that is a guess. `UNKNOWN` never renders as absent; a
      quarantined punch never reaches a report; a suppressed cell is never a zero.
- [ ] Error messages name the next action, not just the refusal.

---

## Part 2 — The silent-failure register

**The most important part of this document.** Each of these breaks without anyone noticing. Each
needs a check that fails loudly, and no feature is done until its own entries here are covered.

| # | What breaks | How it looks | The check that catches it |
|---|---|---|---|
| 1 | **04's day opener stops** | Absent employees generate no punch, so no row is written, so they never appear in the absence report. Attendance looks *better* than reality | Status endpoint reports last successful run; alert if no run in 26 hours. **Plus** a daily assertion: every employed person has a row for yesterday |
| 2 | **04's recompute worker stalls** | Attendance quietly stops updating. The grid shows stale days that look real | Queue depth **and** last-run time on the status endpoint; alert if the oldest dirty day exceeds an hour |
| 3 | **05's notification worker stops** | Approvals wait forever. Nobody is told, including about this | Queue depth, last successful send, and a **failure-rate alert delivered in-app** — never by email, which is what may be broken |
| 4 | **06's accrual does not run** | Leave balances stop growing. Discovered when someone is refused leave they have earned | Per-tenant last-run record; alert on any tenant missing a due period |
| 5 | **A cache disagrees with its ledger** | 06's balance snapshot drifts from the entries that produce it. The number is wrong and self-consistent | Nightly consistency job **reports** drift rather than silently correcting it. A silent correction hides the bug that caused it |
| 6 | **A payslip does not reconcile** | Lines do not sum to the total. Rounding applied twice | Reconciliation is an exception at calculation time, blocking approval — already specified in 07 FR-X-06; verify it cannot be acknowledged away |
| 7 | **Cross-tenant leak** | Another company's data in a report or list. May never be noticed from inside | RLS as the enforcing layer, not the app. Per-feature tests asserting invisibility across tenants. A CI check that every tenant-owned table has a policy |
| 8 | **Backups miss the document volume** | Database restores fine; every contract, CV and payslip PDF is gone | A restore drill that opens an uploaded document, not just a table. **This must be tested, not assumed** |
| 9 | **The collector silently stops forwarding** | Attendance simply stops arriving, and looks like everyone being absent | Heartbeat with buffered-count; alert on missed heartbeat **and** on a buffer that stops draining |
| 10 | **Quarantined punches are ignored** | They accumulate unreviewed; real attendance is missing and nobody is chasing it | Quarantine count surfaced on the attendance dashboard, and an alert past a threshold age |
| 11 | **Retention reviews are ignored** | Now that nothing deletes automatically (OQ-1002), candidate and notification data grows without bound | The weekly review report is a notification, not a page someone must remember to visit. Track table growth as an operational metric |
| 12 | **A scheduled report keeps going to someone who lost access** | A quiet, ongoing disclosure | Recipients re-checked **at delivery**; the owner is told who was dropped; schedules die with their owner's account |
| 13 | **Timezone set after attendance exists** | Historical punches re-bucket into different days; reports before and after disagree | The setup checklist puts timezone before attendance; changing it later is change-controlled and states the record count affected |
| 14 | **A formula or template edit changes history** | Old payslips or reviews render differently than when issued | Snapshotting (07, 09). Test: render a finalised payslip, change the component, re-render, assert byte-identical |

---

## Part 3 — Prerequisite gate

A feature may not **start** until its blocking questions are answered. This table is the gate.

| Feature | Blocked by | Status |
|---|---|---|
| 01 | OQ-301 ✅, OQ-101 ✅ | **Clear to start** |
| 02 | OQ-201 ✅, OQ-204 ✅, OQ-206 ✅ | **Clear to start** |
| 03 | OQ-302 ✅, OQ-206 ✅ | **Clear to start** |
| 04 — engine & shifts | OQ-401 ✅, OQ-S-01 ✅, OQ-403–408 ✅ | **Clear to start** |
| 04 — ingestion | **OQ-319** (hosted vs per-company) | ⛔ **Blocked** |
| 05 | OQ-502 ✅, OQ-503 ✅ | Clear |
| 06 | OQ-602 ✅, OQ-604 ✅, OQ-606 ✅ · **OQ-601b** (entitlement days) blocks *seeding*, not building | 🟡 Partly |
| 07 | **OQ-701** (pay components) | ⛔ **Blocked** |
| 08 | OQ-802 ✅, OQ-805 ✅, OQ-801 ✅ | Clear |
| 09 | OQ-901 ✅, OQ-902 ✅ | Clear |
| 10 | OQ-1001 ✅ (built last), OQ-1002 ✅, OQ-1003 ✅ | Clear |
| 11 | OQ-1101 ✅, OQ-T-06 (per-tenant suppression) | Clear |

**Three answers are still outstanding, and two of them block features outright.** The engine-first
sequencing for 04 means its blocker costs nothing yet: build the computation and shifts against
synthetic punches while OQ-319 is settled.

---

## Part 4 — Go-live checklist

Before the first real tenant uses this for real payroll.

### Operational

- [ ] Backups cover the **database and the document volume**, and a restore has been performed and
      verified by opening a file (register #8).
- [ ] The job runner is monitored, with alerts on the five silent jobs (register #1–4, #9).
- [ ] Log review confirms **no secrets, no personal data, no salary figures** in application logs.
- [ ] HTTPS everywhere; the device ingestion path is on its own listener, firewalled separately.
- [ ] OQ-319's answer implemented: collector deployed, or LAN-only confirmed and enforced.

### First tenant

- [ ] Tenant provisioned, company profile complete.
- [ ] **Timezone set before any attendance exists** (register #13).
- [ ] Work week and this year's holidays configured; the thin-coverage warning is clear.
- [ ] Devices registered; a real punch has arrived and resolved to the right employee.
- [ ] Employees imported; the import's dry run was reviewed, not skipped.
- [ ] Leave types and entitlements configured (needs OQ-601b).
- [ ] Pay components configured and **tested with the formula tester** before anyone is paid
      (needs OQ-701).
- [ ] One full payroll cycle run in parallel with the existing process, and the figures reconciled
      line by line. **Do not cut over on the first run.**

### Handover

- [ ] Someone other than the builder can: register a device, correct an attendance day, approve
      leave, run payroll, and explain a payslip line to an employee.
- [ ] The super-admin recovery path is documented and tested.
- [ ] Known limitations are written down for the customer — including that **nothing is deleted
      automatically** and retention is their responsibility to action.

---

## Part 5 — Is the planning done?

| Step | Status |
|---|---|
| 0 — `dev-plan/` created | ✅ |
| 1 — Feature list and build order | ✅ (revised: 10 now builds last) |
| 2 — Folders and overview skeleton | ✅ |
| 3 — Five files per feature | ✅ 11 features, 56 files |
| 4 — Root overview | ✅ |
| 5 — Open questions | ✅ 175 raised, 172 answered |
| 5b — Revision pass | ✅ 2 cross-cutting docs + targeted edits |
| 6 — Definition of done | ✅ this document |

**Planning is complete.** What remains is not planning.

### Known gaps, carried forward honestly

These are real and were not closed. They are listed so they are decided rather than discovered:

1. ~~**No testing strategy document.**~~ ✅ **Closed 2026-09-18** —
   [TESTING_STRATEGY.md](./TESTING_STRATEGY.md).
2. **No deployment or operations plan** beyond the existing Docker Compose: no backup procedure, no
   monitoring stack, no upgrade path, no tenant-provisioning runbook.
3. **No estimates.** Nothing in this plan says how long any of it takes.
4. **Features 01–04's `ui-ux.md` predate the workspace design rules** and are not skill-grounded.
   05–11 are.
5. **Three open questions block work** — OQ-319, OQ-701, OQ-601b.
6. **The stale `dev-plan/` copy inside `hrm-system`** should be removed or reconciled (OQ-004), and
   `git` is still not on PATH on this machine (OQ-005).
7. **OQ-003 — the pasted GitHub token has still not been confirmed revoked.**

### What I would do next, in order

1. Answer OQ-319, OQ-701, OQ-601b.
2. ~~Write the testing strategy.~~ ✅ Done — [TESTING_STRATEGY.md](./TESTING_STRATEGY.md).
3. **Build the test harness first**: the canonical fixture, the two-tenant setup, and the injected
   clock. All three are expensive to retrofit and shape everything after them.
4. Build 01 and 02 together — they are hard to separate, and 02 is what makes 01's scope real.
5. Build 03, getting timezone and locations right before any attendance exists.
6. Build 04's engine against synthetic punches; ingestion when OQ-319 lands.
