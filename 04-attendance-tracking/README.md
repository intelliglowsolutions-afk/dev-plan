# 04 — Attendance Tracking

**Priority:** Must-have · **Build order:** 4 of 11 · **Status:** planned (Step 3)

## Purpose

Turn raw face-scan punches from the SenseFace terminals into an answer to one question, per person
per day: **were they at work as expected, and if not, how not?**

That answer feeds leave (was this absence covered), payroll (hours, overtime, deductions), and every
attendance report. It is the feature with the most moving parts, the most edge cases, and the most
ways to be quietly wrong.

## The central problem

A punch is a fact: *PIN 1042 was seen at the main gate at 09:14:22 UTC.* Everything else is
interpretation, and the interpretation depends on the employee's shift, the holiday calendar, the
company timezone, approved leave, grace periods, and whether the person remembered to scan out.

So this feature keeps the two strictly apart:

- **Punches are immutable facts.** Never edited, never deleted, never invented. What the device sent
  is what is stored, forever, including punches that match no employee.
- **The daily attendance record is a derived opinion.** It can be recomputed from scratch at any
  time, and recomputing must produce the same answer — unless the inputs changed, in which case it
  *should* change.

Every design decision below follows from that split. It is what makes "the shift was wrong, fix it
and recompute last month" a routine operation rather than a data-recovery exercise.

## Scope

**In scope**

- **Ingestion**: parsing the ADMS/iClock push body into punch records, idempotently, behind the
  device authentication feature 03 defines.
- **Unmatched punches**: PINs that match no employee, surfaced rather than dropped.
- **Shifts and schedules** — see [shifts.md](./shifts.md), the agreed split file: shift definitions,
  patterns, rotations, and per-employee assignment.
- **Daily computation**: pairing punches into work sessions, applying grace, and classifying the day
  (present, late, absent, half day, on leave, holiday, missing punch, unknown).
- **Night shifts** that cross midnight, and the day a punch belongs to.
- **Overtime** detection and classification.
- **Manual corrections**: HR entering or adjusting a day, and employees requesting a correction
  (regularisation), with approval and a full trail.
- **Device gaps**: knowing when missing data is the device's fault rather than the employee's.
- **Recomputation**: the engine, its triggers, and its guarantees.
- Monthly attendance views and export.

**Out of scope (owned elsewhere)**

- Which devices may connect, their keys, health, and event log → [03 Admin & Settings](../03-admin-settings/README.md).
  03 authenticates the request; this feature parses the body. The line is drawn precisely in 03's
  README.
- Whether a date is a working day or a holiday → 03's `isWorkingDay`. This feature calls it and
  never re-implements it (03 D-05).
- Leave requests, balances, and approval → [06 Leave Management](../06-leave-management/README.md).
  This feature *reads* approved leave to classify a day; it never creates or consumes leave.
- Paying for the hours → [07 Payroll](../07-payroll/README.md). This feature produces hours and
  overtime; payroll decides what they are worth.
- The employee's own view of their attendance → [08 Employee Self-Service](../08-employee-self-service/README.md).
  The correction *request* flow is specified here; its employee-facing screens are built in 08.
- Cross-module dashboards → [11 Reports & Analytics](../11-reports-analytics/README.md).

## Files

| File | Contents |
|---|---|
| [requirements.md](./requirements.md) | User stories, functional and non-functional requirements, the classification rules, permission keys |
| [shifts.md](./shifts.md) | Shift definitions, patterns, rotations, assignment, and how a shift decides a day |
| [data-model.md](./data-model.md) | Prisma models, the migration of the existing `attendance` table, recomputation state |
| [api-design.md](./api-design.md) | Ingestion, attendance queries, corrections, shifts, and the recompute engine |
| [ui-ux.md](./ui-ux.md) | Daily grid, employee timeline, corrections queue, roster, states and copy |

## Key decisions taken in this plan

| # | Decision | Rationale |
|---|---|---|
| D-01 | **Raw punches and computed days are separate tables.** The existing `attendance` table becomes `attendance_punches`; a new `attendance_days` holds the derived record. | The whole feature rests on this. Mixing them means a recompute overwrites evidence, and "why was I marked absent" becomes unanswerable. |
| D-02 | **Punches are immutable and never deleted**, including punches that match no employee and punches that are obviously wrong (a double-scan, a test punch). | They are the evidence. A wrong punch is handled by classification and correction, not by deletion. |
| D-03 | **The daily record is fully recomputable and idempotent.** Recomputing an untouched day produces a byte-identical result. | It is the only way to safely fix a shift, a holiday, or a timezone after the fact — which will happen. |
| D-04 | **Manual corrections are stored as a separate overlay**, not by editing the computed day. Recompute re-applies them. | Otherwise every recompute silently discards HR's corrections, and the bug is discovered a month later. |
| D-05 | **A punch belongs to a shift-anchored day**, not to the calendar day of its timestamp. A night-shift punch at 01:30 belongs to the previous day's shift. | Calendar-day bucketing breaks every night shift. Deciding this at the end is a rewrite of the computation engine. |
| D-06 | **Default pairing is first-punch-in / last-punch-out per day**, with multi-session pairing available per shift. | People scan twice at the door, forget to scan for lunch, and scan again on the way past. First/last is robust against all three; strict pairing is not, and produces confidently wrong hours. |
| D-07 | **An unmatched PIN is queued, not dropped.** | A new hire whose code was mistyped produces punches nobody sees. Silently dropping them means their first week has no attendance and nobody knows why. |
| D-08 | **A device gap produces `UNKNOWN`, never `ABSENT`.** | Marking 200 people absent because a terminal lost power is the single most damaging thing this system could do. This is the rule to defend hardest (03 OQ-308). |
| D-09 | **Approved leave and holidays classify the day; they do not delete the punches.** Someone who came in on a holiday still has punches, and the day is flagged as worked-on-a-holiday for payroll. | Holiday working is real, and usually paid differently. |
| D-10 | **Computation runs asynchronously, triggered by a dirty-day queue**, not inline with ingestion. | Ingestion must answer the device fast (03 NFR-04). A slow computation must never cost a punch. |

## How a day is decided (the short version)

Long form in `requirements.md`; this is the shape, because everything else refers to it.

1. Resolve the employee's **shift** for the date (`shifts.md`), or the company work week if none.
2. Ask 03 whether it is a **working day**, and for how many hours.
3. Collect the **punches** in the shift-anchored window (D-05).
4. **Pair** them into sessions (D-06) and compute worked hours, less unpaid breaks.
5. Apply **grace** to decide late arrival and early departure.
6. Overlay **approved leave** for the date (06), and **holiday** status (03).
7. Classify: `PRESENT` · `LATE` · `EARLY_DEPARTURE` · `HALF_DAY` · `ABSENT` · `ON_LEAVE` ·
   `HOLIDAY` · `WEEKLY_OFF` · `MISSING_PUNCH` · `UNKNOWN` (D-08).
8. Compute **overtime** against the shift's rules.
9. Apply any **correction overlay** (D-04) and record which fields it changed.

Steps 1–8 are pure: same inputs, same output, no clock reads, no randomness (D-03).

## Dependencies

- **Depends on:** 01 (permissions, audit), 02 (employees — PIN resolution, scope), 03 (device auth,
  timezone, `isWorkingDay`, settings, job runner). All three must be built first; this feature
  cannot be sensibly started before them.
- **Soft dependency on 06 (Leave):** classification reads approved leave. Until 06 exists, the
  `ON_LEAVE` branch is a no-op and days that should be leave show as `ABSENT`. Building this feature
  before 06 is correct by build order, but the gap must be understood — see OQ-410.
- **Depended on by:** 06 (attendance informs leave), 07 (hours and overtime drive pay), 08, 11.
- **Touches existing code:** the placeholder `/api/device/iclock/[[...path]]/route.ts` gains its real
  implementation; the `Attendance` model is renamed and extended.

## The elephant: OQ-000

The SenseFace 2A manual is still missing. This feature's ingestion half — the exact request paths,
the ATTLOG record format, field order, the meaning of punch-state and verify-mode codes, the
handshake, and the command channel — **cannot be finalised without it.**

What has been done instead: `api-design.md` specifies ingestion against the *documented general
shape* of the ZKTeco ADMS/iClock protocol, marks every protocol-specific assumption explicitly, and
structures the parser so the uncertain parts sit behind one adapter module. The rest of the
feature — shifts, computation, corrections, recompute, UI — does not depend on the protocol at all
and can be built and tested against synthetic punches.

**Recommended build order within this feature:** computation engine and shifts first, with punches
injected by a test fixture; ingestion last, once the manual is in hand. That sequencing turns a
blocker into a scheduling detail.

## Open questions

| ID | Question | Why it matters | Proposed default |
|---|---|---|---|
| OQ-401 | Does the company run **night shifts** or any shift crossing midnight? | D-05 exists for this. If there are none, the computation simplifies considerably — but adding it later touches every part of the engine. | Assume yes; build the shift-anchored window from the start. |
| OQ-402 | Is there **one punch device at one door**, or several (entry/exit, multiple buildings)? | With several devices, pairing must consider which device a punch came from, and "last out" may be at a different terminal. | One or two devices, pairing ignores which. |
| OQ-403 | Do employees scan for **breaks/lunch**, or only at start and end of day? | Decides whether D-06's default (first/last) or multi-session pairing is right, and whether breaks are measured or assumed. | Start and end only; breaks are a fixed deduction per shift. |
| OQ-404 | What are the **grace periods** — late arrival, early departure — and does repeated lateness have a consequence the system should track (e.g. 3 lates = half-day deduction)? | Grace is a shift setting; the "3 lates" rule is a policy engine, which is a much bigger build. | 10 minutes each way; no cumulative-lateness rule in v1. |
| OQ-405 | Is **overtime** paid, and if so does it require pre-approval, or is any time beyond the shift automatically overtime? | Auto-overtime plus a terminal at the door means someone who lingers earns overtime. Most companies require approval. | Overtime is detected and recorded but marked `PENDING_APPROVAL`; payroll only pays approved overtime. |
| OQ-406 | What happens when someone **forgets to scan out**? | The most common real-world case. Options: leave the day `MISSING_PUNCH` and require correction; assume the shift end; assume nothing and pay nothing. | `MISSING_PUNCH`, no assumed hours, correction required. Never silently assume a departure time. |
| OQ-407 | Who may **approve corrections** — the line manager, HR, or both? | Decides the approval routing and the notification targets. | Manager approves, HR can override; HR-entered corrections are self-approved and audited. |
| OQ-408 | How far back may attendance be **corrected**? Is there a lock after payroll runs for a period? | Without a lock, a correction to a paid month silently desynchronises payroll. | Locked once payroll for that period is finalised (feature 07); corrections after that need a super admin and produce an adjustment, not a silent edit. |
| OQ-409 | Is there a **probation or trainee** group with different attendance rules? | Usually not, but it is cheap to ask now. | No. |
| OQ-410 | Feature 06 (Leave) is built *after* this one, so `ON_LEAVE` classification has nothing to read initially. Should historical days be recomputed once 06 lands? | A month of wrongly-`ABSENT` days would otherwise persist. | Yes — a bulk recompute is part of 06's release. Noted there too. |
| OQ-411 | Should the system record **which device** a punch came from in the daily record (first in at Gate A, last out at Gate B)? | Useful for multi-site; noise for single-site. | Stored on the punch, not summarised on the day. |
