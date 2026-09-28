# 09 — Performance Management — API Design

Conventions as in [01 api-design.md](../01-roles-permissions-auth/api-design.md).

**Every endpoint in this feature enforces `requirements.md` §FR-V (Visibility) in the query itself,
not after fetching** (NFR-04). Where a visibility rule and a convenience conflict, the rule wins.

## The visibility helper

One function decides what an actor may see of a review instance. Every endpoint below routes through
it; none re-implements it.

```ts
// src/lib/performance/visibility.ts
reviewVisibilityFilter(ctx): Prisma.ReviewInstanceWhereInput
// Composes into every query. Produces, as a union:
//   - instances where the actor is the reviewer (any state, including DRAFT)
//   - instances where the actor is the subject AND state is SHARED or later
//   - self-review instances of the actor's direct reports where state is SUBMITTED or later
//   - everything, only with performance.read_content — and the read is logged
//
// Never returns a DRAFT instance to anyone but its author (FR-V-02, FR-V-03).

canShare(instance, ctx): { allowed: boolean; reason?: string }
// Enforces FR-V-04: a manager review cannot be shared before the subject's self-review is
// submitted, unless the deadline has passed or the actor is overriding with HR permission.
```

```ts
logReviewAccess(instanceId, userId): void
// Fire-and-forget (NFR-05). Never awaited in the response path; a logging failure must not
// fail a read.
```

## Goals

| Method | Path | Permission | Notes |
|---|---|---|---|
| GET | `/api/performance/goals` | `performance.goal.read` | Scoped: own, or reports' |
| POST | `/api/performance/goals` | `performance.goal.write` | Employee or their manager |
| GET | `/api/performance/goals/:id` | `performance.goal.read` | With check-ins and history |
| PATCH | `/api/performance/goals/:id` | `performance.goal.write` | Attributed (D-03) |
| POST | `/api/performance/goals/:id/check-ins` | `performance.goal.write` | Append-only |
| POST | `/api/performance/goals/:id/close` | `performance.goal.write` | Status + reason |
| GET | `/api/performance/goals/team` | `performance.goal.read` (DEPARTMENT) | Manager's view |

`GET /api/performance/goals/team` returns the manager's reports with their goals, progress, and
overdue flags in one request (NFR-02) — the manager's main screen, and the reason the endpoint
exists separately from the list.

Check-ins have no update or delete endpoint (FR-G-05). A correction is another check-in, and the
API's shape is what enforces that rather than a convention someone can forget.

## Cycles and templates

| Method | Path | Permission |
|---|---|---|
| GET / POST | `/api/performance/cycles` | `performance.cycle.read` / `.write` |
| GET / PATCH | `/api/performance/cycles/:id` | |
| POST | `/api/performance/cycles/:id/preview-participants` | `performance.cycle.write` |
| POST | `/api/performance/cycles/:id/open` | `performance.cycle.write` |
| POST | `/api/performance/cycles/:id/close` | `performance.cycle.write` |
| GET | `/api/performance/cycles/:id/progress` | `performance.cycle.read` |
| GET / POST | `/api/performance/templates` | `performance.cycle.write` |
| GET / POST | `/api/performance/rating-scales` | `performance.cycle.write` |

### `POST /api/performance/cycles/:id/preview-participants`

Resolves the participation rules without writing, and — importantly — lists the problems
(FR-C-05):

```json
{ "included": 187,
  "excluded": [ { "employeeId": 91, "name": "Sara Ahmed", "reason": "Hired 2 Sep, after the cut-off" } ],
  "problems": [ { "employeeId": 44, "name": "Bilal Aslam",
                  "issue": "No manager set — no manager review can be created.",
                  "fixPath": "/employees/44" } ] }
```

The same preview-before-commit pattern as features 04, 06, and 07. Opening a cycle for 200 people
without seeing who is in it is the kind of act that produces a week of corrections.

### `POST /api/performance/cycles/:id/open`

Resolves and **stores** participants and their managers (FR-C-02), creates review instances with
their template snapshots (FR-F-05), and notifies participants. **409** `HAS_PROBLEMS` unless
`"acknowledgeProblems": true` — employees without a manager get a self-review only, and that choice
must be deliberate.

### `GET /api/performance/cycles/:id/progress`

Completion only, no content (FR-V-06):

```json
{ "cycle": "2026 annual review",
  "stages": { "selfReview": { "complete": 142, "total": 187 },
              "managerReview": { "complete": 88, "total": 187 },
              "shared": { "complete": 71, "total": 187 },
              "acknowledged": { "complete": 64, "total": 187 } },
  "byDepartment": [ { "department": "Finance", "selfReview": 12, "managerReview": 9, "total": 14 } ],
  "overdue": { "selfReview": 12, "managerReview": 31 } }
```

An actor holding `performance.cycle.read` but not `performance.read_content` gets exactly this and
nothing deeper — no names attached to ratings, no answers, no drill-through to content.

## Review instances

| Method | Path | Permission | Notes |
|---|---|---|---|
| GET | `/api/performance/reviews/mine` | `performance.review.participate` | What is assigned to me |
| GET | `/api/performance/reviews/about-me` | `performance.review.participate` | Shared reviews of me |
| GET | `/api/performance/reviews/:id` | visibility helper | Logged on content read |
| PATCH | `/api/performance/reviews/:id/answers` | reviewer only | Autosave |
| POST | `/api/performance/reviews/:id/submit` | reviewer only | |
| POST | `/api/performance/reviews/:id/share` | `performance.review.manage_team` | Manager reviews |
| POST | `/api/performance/reviews/:id/acknowledge` | subject only | |
| POST | `/api/performance/reviews/:id/comments` | subject or reviewer | After acknowledgement |
| POST | `/api/performance/reviews/:id/unlock` | `performance.unlock` | Super admin, reason required |
| GET | `/api/performance/reviews/:id/access-log` | `performance.read_content` | Who read it |

### `PATCH /api/performance/reviews/:id/answers` — autosave

```json
{ "answers": [ { "questionId": 12, "textValue": "Ayesha led the migration…" },
               { "questionId": 13, "ratingValue": 4 } ] }
```

**200** with `{ "savedAt": "2026-09-15T10:42:03Z" }` — which the UI displays (FR-R-02, NFR-01).
Partial saves are expected and normal; there is no "save the whole form" operation.

Only the reviewer may write, in `DRAFT` state only. A `PATCH` to a submitted instance → **409**
`ALREADY_SUBMITTED`. Rating answers store the scale's label and definition alongside the value
(FR-F-07) — resolved server-side, so a client cannot supply a mismatched label.

### `POST /api/performance/reviews/:id/submit`

Validates required questions and returns **422** `INCOMPLETE` listing them by section, so the UI can
jump to each rather than saying "something is missing".

For a self-review, submission makes it visible to the manager and HR. For a manager review,
submission does **not** make it visible to the employee — that is `share`, and the separation is the
whole point (FR-V-02, D-02).

### `POST /api/performance/reviews/:id/share`

**409** `SELF_REVIEW_NOT_SUBMITTED` unless the self-review is in, the deadline has passed, or the
actor overrides with HR permission (FR-V-04):

```json
{ "error": { "code": "SELF_REVIEW_NOT_SUBMITTED",
             "message": "Ayesha hasn't submitted her self-review yet. Sharing now means she writes it after seeing your rating.",
             "selfReviewDueOn": "2026-09-20",
             "canOverride": false } }
```

The message explains the reasoning rather than just refusing. This rule is unusual enough that a
bare error reads as a bug.

Sharing notifies the employee — "your review is ready to discuss", with no content in the
notification (NFR-03).

### `POST /api/performance/reviews/:id/acknowledge`

```json
{ "agrees": true, "note": "Thanks — happy with this." }
```

`agrees: false` records `ACKNOWLEDGED_WITH_DISAGREEMENT` with the note (FR-R-06). Both outcomes are
acknowledgements; the API deliberately has no path that discards a disagreement, and no path that
records agreement the employee did not give.

After this the instance is immutable (FR-R-07); further input goes to `/comments`.

### `GET /api/performance/reviews/:id`

Returns the instance rendered **from its template snapshot**, with the answers the actor may see.
Calls `logReviewAccess` when content is returned and the actor is not the reviewer or the subject
(FR-V-07).

An instance the actor may not see returns **404**, never 403 — consistent with 01 FR-Z-07, and more
important here: a 403 confirms that a review exists and that someone wrote something.

## Feedback

| Method | Path | Permission |
|---|---|---|
| GET | `/api/performance/feedback/about-me` | `performance.feedback.give` |
| GET | `/api/performance/feedback/about/:employeeId` | manager or `performance.read_content` |
| POST | `/api/performance/feedback` | `performance.feedback.give` |
| POST | `/api/performance/feedback/requests` | `performance.feedback.give` |
| GET | `/api/performance/feedback/requests/mine` | `performance.feedback.give` |
| POST | `/api/performance/feedback/requests/:id/decline` | asked user |

`POST /api/performance/feedback` takes a visibility (FR-B-01). Where the author chooses
`MANAGER_ONLY`, the response carries the warning the UI must show before submission:

```json
{ "feedback": { "id": 55 },
  "notice": "This is visible to Ayesha's manager, not to Ayesha. She can ask HR to see it." }
```

That notice exists because FR-B-06 is a genuine constraint on the author's expectations: there is no
"the subject can never see this" option, and they should learn that before writing rather than after.

Anonymous peer feedback inside a cycle is returned **only in aggregate and only above the threshold**
(FR-V-11). Below it, the response omits the section entirely rather than returning an empty one — an
empty "anonymous feedback" section with two responses behind it tells the subject exactly how few
people replied, and in a small team, who.

## What this API deliberately does not have

Stated here because an absent endpoint is invisible in review (FR-X):

- **No endpoint that reads attendance, leave, or payroll data.** Not in the calculation of any
  figure, not as context on a review screen, not in a report. The feature has no dependency on 04,
  06, or 07 (D-04, FR-X-01).
- **No endpoint that computes an overall rating** from other ratings (FR-X-02).
- **No endpoint that ranks or distributes employees** (FR-X-02).
- **No notification or endpoint that tells a manager an employee has started, saved, or is working
  on a self-review** (FR-X-03).
- **No engagement or time-spent metrics** (FR-X-04).

## Scheduled jobs

| Job | Frequency | Does |
|---|---|---|
| Cycle stage reminders | daily | Notifies before each deadline, and once after it passes |
| Overdue goals | weekly | Notifies owner and manager only (FR-G-09) |
| Cycle auto-close | daily | Closes cycles past their final deadline, marking unfinished instances `INCOMPLETE` — never auto-submitting or auto-acknowledging (FR-C-07) |
| Feedback request reminders | weekly | One reminder, then stop |

"One reminder, then stop" is deliberate: a feedback request that nags indefinitely converts a
voluntary act into an obligation.

## Open questions

| ID | Question |
|---|---|
| OQ-910 | Draft answer history (raised in `data-model.md`) would need `PATCH` to append rather than overwrite. Cheap now, awkward later. |
| OQ-911 | Should `GET /api/performance/reviews/about-me` include reviews from previous employers of the same company — i.e. before a rehire? Interacts with 02's employment periods and 09 OQ-908. |
| OQ-903 | The visibility helper's skip-level branch is written but gated off by a setting. Turning it on is a company decision with real consequences for candour. |
