# 10 — Recruitment & Onboarding — API Design

Conventions as in [01 api-design.md](../01-roles-permissions-auth/api-design.md). Every endpoint is
wrapped in `protectedRoute` **except the two public ones**, which are specified first because they
are the only unauthenticated, internet-facing write surface in the entire system.

## The public surface

Feature 03's device endpoint is unauthenticated by session but authenticated by a comm key and
reachable only from the local network. **These two endpoints are reachable by anyone on the
internet, with no credential of any kind.** They are designed accordingly (D-08, NFR-01).

### `GET /jobs/:publicSlug` — the public posting

Returns only what a posting is for: title, summary, sanitised description, location, and the salary
text if the posting says to show it. No requisition id, no internal ids, no hiring manager, no
headcount, no stage names (NFR-03).

A `DRAFT` or `CLOSED` posting returns **404** — the same response as a slug that never existed, so
drafts cannot be discovered by trying slugs.

### `POST /api/public/applications` — the application form

`multipart/form-data`: name, email, phone, location, cover note, CV file, posting slug, and the
consent acknowledgement.

Defences, all required rather than advisable:

| Concern | Measure |
|---|---|
| Volume | Rate limit per IP (5/hour) and a global cap per posting per hour |
| Bots | A honeypot field and a minimum form-fill time; **no CAPTCHA that blocks assistive technology** without an accessible alternative |
| File type | Content sniffing, not extension; PDF and common document formats only |
| File size | 5 MB, enforced while streaming, not after buffering |
| Malicious files | Stored with a generated key outside the web root, never executed, never served statically (NFR-02) |
| Injection | All text sanitised; nothing from this endpoint is ever rendered as HTML |
| Enumeration | Always the same response, whether or not the email is already known (FR-C-03 acceptance) |
| Data leakage | The response contains no ids — internal or otherwise |

```json
{ "received": true,
  "message": "Thank you. We've received your application and will be in touch." }
```

That is the entire response. No application id, no candidate id, no "you have applied before".

The consent text in force is **stored with the application** (FR-R-01), along with the retention
period the applicant was shown and their optional keep-on-file choice — which is a separate
checkbox, unticked, never bundled with submitting (FR-R-03).

This endpoint shares no code path with any authenticated endpoint. It has its own handler, its own
validation, and its own rate limiter.

## Requisitions and postings

| Method | Path | Permission |
|---|---|---|
| GET / POST | `/api/recruitment/requisitions` | `recruitment.requisition.read` / `.write` |
| GET / PATCH | `/api/recruitment/requisitions/:id` | |
| POST | `/api/recruitment/requisitions/:id/submit` | `recruitment.requisition.write` |
| POST | `/api/recruitment/requisitions/:id/approve` | `recruitment.requisition.approve` |
| POST | `/api/recruitment/requisitions/:id/decline` | `recruitment.requisition.approve` |
| GET / POST | `/api/recruitment/postings` | `recruitment.posting.write` |
| POST | `/api/recruitment/postings/:id/publish` | `recruitment.posting.write` |
| POST | `/api/recruitment/postings/:id/close` | `recruitment.posting.write` |

`publish` → **409** `REQUISITION_NOT_APPROVED` where approval is required and absent (FR-Q-02). The
error names the requisition and its current status, so the recruiter knows whether to chase an
approver or enable the posting.

A hiring manager sees their own requisitions; HR sees all. Scope here is by
`hiringManagerId`, not by department — a manager hiring for another team is common and the reporting
line is the wrong test.

## Candidates and applications

| Method | Path | Permission | Notes |
|---|---|---|---|
| GET | `/api/recruitment/candidates` | `recruitment.candidate.read` | Search by name, email, posting |
| POST | `/api/recruitment/candidates` | `recruitment.candidate.write` | Manual entry (US-06) |
| GET | `/api/recruitment/candidates/:id` | `recruitment.candidate.read` | Full history across applications |
| PATCH | `/api/recruitment/candidates/:id` | `recruitment.candidate.write` | |
| GET | `/api/recruitment/candidates/:id/documents/:docId` | `recruitment.candidate.read` | Permission-checked stream |
| POST | `/api/recruitment/candidates/:id/notes` | `recruitment.candidate.write` | |
| GET | `/api/recruitment/applications` | `recruitment.candidate.read` | The pipeline query |
| POST | `/api/recruitment/applications/:id/move` | `recruitment.candidate.write` | Change stage |
| POST | `/api/recruitment/applications/:id/reject` | `recruitment.candidate.write` | |
| POST | `/api/recruitment/applications/bulk-reject` | `recruitment.candidate.write` | Reason required |
| POST | `/api/recruitment/applications/:id/withdraw` | `recruitment.candidate.write` | They withdrew |

### `POST /api/recruitment/candidates` — manual entry

Returns **200** with a `possibleDuplicates` array when the email or name matches an existing
candidate, and creates nothing until `"confirmNotDuplicate": true` (FR-C-02):

```json
{ "possibleDuplicates": [ { "candidateId": 88, "name": "Sara Ahmed", "email": "sara@…",
                            "lastApplied": "2024-03-11", "lastOutcome": "REJECTED",
                            "postingTitle": "Accountant" } ] }
```

Showing the previous outcome matters: "we interviewed her last year and liked her" is exactly the
context that gets lost, and it is why this is a warning rather than a silent merge.

### `GET /api/recruitment/applications` — the pipeline

`?postingId=&stageId=&outcome=&stalledOnly=true`. Returns candidates grouped by stage with the
minimum needed for the board — name, days in stage, interview status, whether a scorecard is
outstanding — and **not** their documents (NFR-04).

```json
{ "stages": [ { "stageId": 3, "name": "Interview", "count": 6,
                "applications": [ { "applicationId": 141, "candidateName": "Imran Qadir",
                                    "daysInStage": 18, "isStalled": true,
                                    "nextInterviewAt": null,
                                    "scorecardsOutstanding": 0 } ] } ] }
```

`isStalled` is computed from the stage's `stallAfterDays` (FR-P-06). It is the field that stops
candidates being forgotten, which is the most common failure of a hiring process and the one that
does most reputational damage.

### `POST /api/recruitment/applications/:id/reject`

```json
{ "reasonCode": "SKILLS_MISMATCH",
  "internalNote": "Strong communicator, but no experience with the reporting stack.",
  "messageToCandidate": "Thank you for your time. We've decided to move forward with other candidates whose experience is closer to the role.",
  "sendMessage": true }
```

Two separate fields, and the API **will not** accept the internal note as the message (D-09,
FR-C-06). If `sendMessage` is true and `messageToCandidate` is empty, **422** `MESSAGE_REQUIRED` —
the system does not fill it in from the internal note, ever.

Sets `retentionUntil` from the configured period (FR-R-04) and notifies the candidate via 05 if
requested.

`bulk-reject` takes an array of application ids plus one reason code and one message, and requires
`"confirmCount": n` matching the selection — bulk rejection is the operation most likely to be
performed on the wrong selection.

## Interviews and scorecards

| Method | Path | Permission |
|---|---|---|
| POST | `/api/recruitment/interviews` | `recruitment.interview.schedule` |
| PATCH | `/api/recruitment/interviews/:id` | `recruitment.interview.schedule` |
| POST | `/api/recruitment/interviews/:id/cancel` | `recruitment.interview.schedule` |
| GET | `/api/recruitment/interviews/mine` | `recruitment.scorecard.submit` |
| GET | `/api/recruitment/interviews/:id` | assigned interviewer or `candidate.read` |
| PATCH | `/api/recruitment/scorecards/:id` | the scorecard's interviewer only |
| POST | `/api/recruitment/scorecards/:id/submit` | the scorecard's interviewer only |
| GET | `/api/recruitment/interviews/:id/scorecards` | see below |

### Scorecard visibility — the rule that matters

`GET /api/recruitment/interviews/:id/scorecards` returns:

- **Always**: the requester's own scorecard, submitted or not.
- **Other interviewers' scorecards**: only once **every** assigned interviewer has submitted
  (FR-I-04, FR-I-05).
- **Before that**: a count of how many are outstanding, and **not who they are waiting on** — naming
  the laggard turns a structured process into social pressure to agree.

```json
{ "mine": { "id": 22, "submittedAt": "2026-09-15T14:02:00Z", "recommendation": "YES", … },
  "othersVisible": false,
  "outstandingCount": 2,
  "message": "Other interviewers' feedback will appear once everyone has submitted." }
```

Enforced in the query, not in the UI (NFR-06). An interviewer who requests another's scorecard id
directly gets **404** — not 403, which would confirm that one exists and has content.

`PATCH /api/recruitment/scorecards/:id` autosaves, returning `savedAt`, exactly as feature 09's
review form does. Submission is irreversible (FR-I-06); additions are appended comments.

Scheduling calls feature 06 to check interviewers' approved leave and returns conflicts as warnings
in the response (FR-I-08) — it never blocks, because interviewing during leave is the interviewer's
decision to make.

## Offers

| Method | Path | Permission |
|---|---|---|
| GET / POST | `/api/recruitment/offers` | `recruitment.offer.read` / `.write` |
| PATCH | `/api/recruitment/offers/:id` | `recruitment.offer.write` |
| POST | `/api/recruitment/offers/:id/send` | `recruitment.offer.write` |
| POST | `/api/recruitment/offers/:id/respond` | `recruitment.offer.write` |
| POST | `/api/recruitment/offers/:id/withdraw` | `recruitment.offer.write` |

`respond` records `ACCEPTED` or `DECLINED` with the candidate's reason where given (FR-O-04).
Offers contain compensation and are gated by their own permission, separate from
`recruitment.candidate.read` (FR-O-05) — a hiring manager may run the pipeline without seeing the
salary discussion.

Nothing here writes to feature 07 (FR-O-02).

## The hire

| Method | Path | Permission |
|---|---|---|
| POST | `/api/recruitment/applications/:id/hire/preview` | `recruitment.hire` |
| POST | `/api/recruitment/applications/:id/hire` | `recruitment.hire` |

### `hire/preview`

Writes nothing. Returns the employee that would be created, with what is missing (FR-H-02):

```json
{ "employee": { "firstName": "Sara", "lastName": "Ahmed", "workEmail": null,
                "personalEmail": "sara@…", "phone": "+92…",
                "departmentId": 3, "positionId": 8, "managerId": 5,
                "employmentType": "FULL_TIME", "hireDate": "2026-11-01" },
  "required": [ { "field": "employeeCode",
                  "message": "Needed for the attendance terminal. Numbers only, up to 9 digits." },
                { "field": "workEmail", "message": "Needed to create their login." } ],
  "willAlsoDo": [ "Create 11 onboarding tasks, first due 30 October",
                  "Keep this candidate's record permanently (it becomes part of their employment history)" ],
  "willNotDo": [ "Set their salary — do that in Payroll after hiring",
                 "Assign a shift — do that in Attendance",
                 "Assign a leave policy — do that in Leave" ] }
```

`willNotDo` is the most useful part of this response. Those three omissions are deliberate (FR-H-06)
and are exactly what gets forgotten; naming them here, and again as onboarding tasks, is how they
stop being forgotten.

### `hire`

Takes the preview's values plus what HR supplied. One transaction (FR-H-04): create the employee
**through feature 02's creation service** (FR-H-03), link the candidate, mark the application
`HIRED`, decrement the requisition headcount, instantiate onboarding tasks, and set the retention
exemption.

A validation failure from 02 — duplicate code, invalid PIN format — fails the whole thing with that
error and writes nothing (FR-H-07):

```json
{ "error": { "code": "EMPLOYEE_CODE_TAKEN",
             "message": "Code 1042 belonged to Imran Sheikh (left Mar 2025). Codes are never reused.",
             "source": "employee-management" } }
```

The `source` field marks errors surfaced from another feature, so the message's vocabulary being
different is explicable rather than confusing.

The user invite is **not** part of this transaction — it is offered in the response and confirmed
separately (FR-H-05).

## Onboarding

| Method | Path | Permission |
|---|---|---|
| GET / POST | `/api/onboarding/templates` | `onboarding.write` |
| PUT | `/api/onboarding/templates/:id/tasks` | `onboarding.write` |
| GET | `/api/onboarding/tasks` | `onboarding.read` |
| GET | `/api/onboarding/board` | `onboarding.read` |
| POST | `/api/onboarding/employees/:id/tasks` | `onboarding.write` |
| POST | `/api/onboarding/tasks/:id/status` | owner or `onboarding.write` |
| GET | `/api/me/onboarding-tasks` | `onboarding.read` (SELF) |

`GET /api/onboarding/board` is HR's screen: everyone starting within a window, with their tasks
grouped by status and overdue counts (FR-N-7), in one request.

`POST /api/onboarding/tasks/:id/status` accepts `DONE`, `NOT_APPLICABLE` (reason required), or
`BLOCKED` (note required). A task marked not applicable without a reason is indistinguishable from
one that was skipped.

`/api/me/onboarding-tasks` is the portal's endpoint (08), and is the new hire's first interaction
with the system — it must work before their start date, which means feature 01's invite has to go
out before then.

## Retention

| Method | Path | Permission |
|---|---|---|
| GET | `/api/recruitment/retention/preview` | `recruitment.retention` |
| POST | `/api/recruitment/retention/purge` | `recruitment.retention` |
| POST | `/api/recruitment/candidates/:id/erase` | `recruitment.retention` |

`preview` lists what is due for deletion, with counts by posting and the earliest and latest
application dates — never the candidates' names, since a preview of a privacy action should not
itself be a list of personal data (FR-R-08).

`purge` requires `"confirmCount": n` matching the preview. It deletes documents from the volume,
clears personal fields, keeps anonymised application rows, and logs counts (FR-R-05, FR-R-06).

`erase` does the same for one candidate, immediately, for an erasure request (FR-R-07).

## Scheduled jobs

| Job | Frequency | Does |
|---|---|---|
| Retention **review** | weekly | **Revised 2026-09-18 (OQ-1002): reports what is past its retention period. Deletes nothing.** HR reviews and acts |
| Stalled candidates | daily | Surfaces candidates past their stage's threshold |
| Interview reminders | hourly | Reminds interviewers before, and about outstanding scorecards after |
| Onboarding due | daily | Notifies task owners of due and overdue tasks |
| Offer expiry | daily | Marks expired offers and notifies the recruiter |

**Revised 2026-09-18 (OQ-1002): no scheduled job in this system deletes anything.** The retention
job became a weekly *review* that reports what is past its period; `POST /api/recruitment/retention/purge`
stays, but only as an HR-initiated action against a preview they have seen.

Two consequences worth stating rather than discovering:

- **Candidate data now accumulates unless someone acts.** The liability the feature was designed to
  avoid is now carried deliberately. The review report is the only thing standing between the
  company and an unbounded archive of rejected applicants' personal data.
- **The consent notice must stop promising deletion the system will not perform.** Its wording
  changes from "we'll delete it after 6 months" to what actually happens — the company reviews and
  decides. Promising an automatic deletion that never runs is worse than promising nothing.

## Open questions

| ID | Question |
|---|---|
| OQ-1003 | If there is no public form, `POST /api/public/applications` and `GET /jobs/:slug` disappear and with them the system's only internet-facing write surface. That is a meaningful reduction in risk. |
| OQ-1008 | Tokenised scorecard links for interviewers without accounts would add a second unauthenticated write endpoint. Worth resisting. |
| OQ-1013 | Should `hire` be reversible within a window, mirroring 02's 24-hour mistake rule, or is 02's own window sufficient? Currently the latter. |
| OQ-1014 | Should the public form support applying without a CV? Some roles do not need one, and requiring a document excludes people applying from a phone. |
