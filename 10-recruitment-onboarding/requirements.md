# 10 — Recruitment & Onboarding — Requirements

## Actors

| Actor | What they do here |
|---|---|
| Hiring manager | Raises a requisition, reviews candidates for their own roles, interviews, decides. |
| Recruiter / HR | Runs the pipeline, schedules, communicates with candidates, makes offers, hires, runs onboarding. |
| Interviewer | Fills in a scorecard for interviews they are assigned to. Sees their own assessment, not others', until submitted. |
| Approver | Approves a requisition before it is advertised (OQ-1004). |
| Candidate | Applies through the public form. Has no account, no session, and no access to anything else. |
| New hire | An employee before their start date, who sees their onboarding tasks in the portal (08). |
| Task owner | Whoever is responsible for an onboarding task — HR, the manager, sometimes IT (OQ-1010). |

## User stories

### Opening a role

- **US-01** — As a hiring manager, I raise a request to hire, saying what the role is and why.
- **US-02** — As an approver, I approve or decline a requisition, with a reason.
- **US-03** — As a recruiter, I write the job posting and publish it.
- **US-04** — As a recruiter, I close a posting when we have enough candidates or the role is filled.

### Candidates

- **US-05** — As a candidate, I apply for a role with my details and my CV, and I know it was
  received and what will happen to my data.
- **US-06** — As a recruiter, I enter a candidate who applied by email or was referred.
- **US-07** — As a recruiter, I see everyone who applied for a role, at which stage, and how long
  they have been there.
- **US-08** — As a recruiter, I move a candidate to the next stage, or reject them with a reason.
- **US-09** — As a recruiter, I see a candidate's whole history with us — every application, every
  interview, every outcome.
- **US-10** — As a recruiter, I am warned when someone applies who has applied before.

### Interviews

- **US-11** — As a recruiter, I schedule an interview with named interviewers and a time.
- **US-12** — As an interviewer, I am told what I am interviewing for and what to assess.
- **US-13** — As an interviewer, I fill in my scorecard without seeing what anyone else said first.
- **US-14** — As a hiring manager, once everyone has submitted, I see all the scorecards together.

### Deciding and hiring

- **US-15** — As a recruiter, I record an offer and its outcome.
- **US-16** — As a recruiter, I turn an accepted candidate into an employee, without re-typing
  everything they already told us.
- **US-17** — As a recruiter, I reject candidates with a message that is decent to receive.

### Onboarding

- **US-18** — As HR, I define an onboarding checklist once, and it applies to every new hire.
- **US-19** — As HR, I see what is outstanding for everyone starting in the next month.
- **US-20** — As a manager, I see what I need to do before my new report arrives.
- **US-21** — As a new hire, I see what I need to do and bring, before my first day.

### Data protection

- **US-22** — As HR, I know what we hold about candidates and for how long.
- **US-23** — As HR, I delete a candidate's data on request, and I can prove it happened.

## Functional requirements

### Requisitions and postings — FR-Q

| ID | Requirement |
|---|---|
| FR-Q-01 | A requisition has: title, department, position, hiring manager, headcount, employment type, reason (new role or replacement), target start date, and optional budget notes. |
| FR-Q-02 | Requisitions require approval before a posting can be published, unless approval is disabled by configuration (OQ-1004). Approval records the approver, time, and any note. |
| FR-Q-03 | A declined requisition keeps its reason and can be revised and resubmitted. |
| FR-Q-04 | A posting belongs to a requisition and carries the public-facing text: summary, responsibilities, requirements, location, and whether salary is stated. |
| FR-Q-05 | A posting has a state: `DRAFT`, `PUBLISHED`, `CLOSED`. Only `PUBLISHED` postings accept applications. |
| FR-Q-06 | Closing a posting stops new applications but does not affect candidates already in the pipeline. |
| FR-Q-07 | A requisition tracks how many of its headcount are filled, and closes when complete. |
| FR-Q-08 | Public posting text is sanitised on save. It is rendered to the open internet, so it is treated as content that could carry injection. |

### Candidates and applications — FR-C

| ID | Requirement |
|---|---|
| FR-C-01 | A **candidate** is a person; an **application** is that person applying for one posting. One candidate may have many applications over time (US-09). |
| FR-C-02 | Candidate identity is matched on email, with a warning rather than an automatic merge when a returning applicant is detected (US-10). Automatic merging of people is a mistake that is hard to undo. |
| FR-C-03 | Candidate fields: name, email, phone, location, CV document, cover note, source (how they found us), and optional salary expectation and notice period. |
| FR-C-04 | An application records its posting, its current stage, its stage history with timestamps, and its outcome. |
| FR-C-05 | Stage history is append-only. "How long were they sitting at screening" must be answerable. |
| FR-C-06 | Rejection records an **internal reason** (structured, from a configurable list, plus a note) and separately the **message sent** to the candidate (D-09). These are different fields and are never the same text. |
| FR-C-07 | Candidates cannot be deleted while an application is active; deletion is by the retention process or by an explicit erasure request (FR-R). |
| FR-C-08 | Candidate notes are visible to recruiters and the hiring manager for that requisition, and to nobody else. There is no company-wide candidate read permission. |
| FR-C-09 | A candidate's documents follow feature 02's document rules: volume storage, content sniffing, permission-checked download, never a static path. |

### Pipeline — FR-P

| ID | Requirement |
|---|---|
| FR-P-01 | Stages are configurable per requisition, from a template (D-03): typically Applied → Screening → Interview → Offer → Hired, with Rejected and Withdrawn as terminal outcomes. |
| FR-P-02 | Each stage declares whether it requires an interview, a scorecard, or an approval before moving on. |
| FR-P-03 | Moving a candidate between stages is recorded with actor, time, and optional note. |
| FR-P-04 | Bulk stage moves are supported for the common "reject these six" case, and require a reason. |
| FR-P-05 | A candidate may be withdrawn (by them) or rejected (by us), and the two are distinct outcomes — conflating them misstates the company's own record. |
| FR-P-06 | Candidates sitting in a stage beyond a configured threshold are surfaced to the recruiter. Silence is the most common failure of a hiring process. |
| FR-P-07 | The system computes no candidate score, rank, or shortlist (D-07). Sorting by stage, date, and name only. |

### Interviews and scorecards — FR-I

| ID | Requirement |
|---|---|
| FR-I-01 | An interview has: application, type (phone, technical, panel, final), scheduled time, duration, location or link, and interviewers. |
| FR-I-02 | Interviewers are notified with what they need: the role, the candidate, the CV, and the scorecard to complete (05). |
| FR-I-03 | A scorecard is a template of criteria, each rated on a configurable scale with a comment, plus an overall recommendation (`STRONG_YES`, `YES`, `NO`, `STRONG_NO`) and free-text notes. |
| FR-I-04 | **An interviewer cannot see another interviewer's scorecard until their own is submitted** (D-04). This includes not being able to see whether one exists. |
| FR-I-05 | Once every interviewer has submitted, all scorecards for that interview become visible to the interviewers and the hiring manager. |
| FR-I-06 | A submitted scorecard cannot be edited. An addition is an appended, dated comment — the same discipline as feature 09's reviews. |
| FR-I-07 | The system computes no aggregate score from scorecards and produces no recommendation (D-07, mirroring 09 FR-X-02). |
| FR-I-08 | Interview scheduling checks the interviewers' approved leave (06) and warns about conflicts. It warns; it does not block. |
| FR-I-09 | Cancelling or rescheduling an interview notifies everyone involved and keeps the original in the history. |

### Offers — FR-O

| ID | Requirement |
|---|---|
| FR-O-01 | An offer records: proposed position, department, manager, employment type, start date, proposed compensation, expiry, and any notes. |
| FR-O-02 | Offer compensation is **stored on the offer**, not written to feature 07, until the hire is complete and a human sets compensation there. An offered figure is not a paid figure. |
| FR-O-03 | Offer states: `DRAFT`, `SENT`, `ACCEPTED`, `DECLINED`, `WITHDRAWN`, `EXPIRED`. |
| FR-O-04 | A declined offer records the candidate's reason where they give one — the most useful recruitment data the company will ever collect. |
| FR-O-05 | Offer details are visible only to those with `recruitment.offer.read`; they contain compensation and inherit feature 07's sensitivity (07 D-06). |

### Hiring — FR-H

| ID | Requirement |
|---|---|
| FR-H-01 | Hiring is available only from an `ACCEPTED` offer. |
| FR-H-02 | The hire **preview** shows the employee record that will be created, with candidate fields mapped and gaps flagged, before anything is written (D-05, README "The hire transition"). |
| FR-H-03 | The employee is created through feature 02's creation path, with its validation and audit — never by writing employee tables directly. |
| FR-H-04 | On success, in one transaction: employee created, candidate linked, application marked `HIRED`, requisition headcount decremented, onboarding tasks instantiated, and the retention exemption applied. |
| FR-H-05 | The user-account invite (01) is offered as part of the flow but is a separate, confirmed step. |
| FR-H-06 | Compensation, shift, and leave policy are **not** set automatically. They appear as onboarding tasks with links to the right screens, so they are visibly outstanding rather than silently missing. |
| FR-H-07 | If employee creation fails validation — a duplicate employee code, an invalid PIN format — the hire fails cleanly with that error and nothing is written. |
| FR-H-08 | A hire can be reversed only by feature 02's 24-hour mistake window (02 FR-E-06); after that, the person is an employee and leaving is a termination. |

### Onboarding — FR-N

| ID | Requirement |
|---|---|
| FR-N-01 | An onboarding template is an ordered set of task definitions: title, description, owner role (HR, manager, IT, the employee), and a due offset relative to the start date (D-06). |
| FR-N-02 | Templates may be scoped to department or employment type, so a factory hire and an office hire get different lists. |
| FR-N-03 | Instantiating resolves owner roles to actual people — the manager from the employee record, HR from a configured user or permission holders. |
| FR-N-04 | Tasks have states: `PENDING`, `DONE`, `NOT_APPLICABLE` (with a reason), `BLOCKED` (with a note). |
| FR-N-05 | Tasks assigned to the employee appear in the portal (08) and are the new hire's first experience of the system. |
| FR-N-06 | Task owners are notified when tasks become due and when they are overdue (05). |
| FR-N-07 | HR sees a board of everyone starting in a window with their outstanding tasks (US-19). |
| FR-N-08 | A task may require a document upload (signed contract, ID copy), which stores against the employee's documents in 02. |
| FR-N-09 | Onboarding completes when all tasks are `DONE` or `NOT_APPLICABLE`; completion is recorded but nothing depends on it. |
| FR-N-10 | Tasks may be added ad hoc to one person's checklist without changing the template. |

### Consent, retention, and erasure — FR-R

This section is the one that most needs a qualified answer rather than a sensible guess (OQ-1002).

| ID | Requirement |
|---|---|
| FR-R-01 | Every candidate record stores what they were told about data use, and when — the privacy notice version in force at the time of their application. |
| FR-R-02 | The public application form states the retention period and requires an explicit action to submit. The consent text is stored, not merely referenced. |
| FR-R-03 | Candidates may opt in to being kept on file for future roles beyond the standard period; this is a separate, optional choice, never bundled with applying. |
| FR-R-04 | A retention job deletes candidate data whose period has expired, unless: they were hired, they opted in, or an application is still active (D-02). |
| FR-R-05 | Deletion removes personal data and documents, and retains an anonymised application record — posting, stage, outcome, dates — so hiring statistics survive without identifying anyone. |
| FR-R-06 | Deletion is logged with what was deleted and when, without the deleted content (US-23). |
| FR-R-07 | An erasure request can be actioned immediately by HR, with the same anonymisation. |
| FR-R-08 | Before each scheduled purge, HR receives a summary of what is about to be deleted, with time to intervene. Automatic deletion with no warning is how something needed gets lost. |
| FR-R-09 | Candidate data is never included in general exports or reports that identify individuals, and never in notification bodies. |

## Non-functional requirements

| ID | Requirement |
|---|---|
| NFR-01 | The public application endpoint is unauthenticated and therefore hostile-facing (D-08): rate-limited per IP, size-capped, file-type-checked by content sniffing, and sanitised. It shares no code path with authenticated endpoints. |
| NFR-02 | Uploaded CVs are treated as untrusted files: never executed, never served from a static path, scanned as feature 02 requires, and stored outside the web root. |
| NFR-03 | The public form must not leak internal information — no requisition ids, no internal stage names, no error messages revealing whether an email is already known. |
| NFR-04 | The pipeline board for a posting with 200 candidates loads in one request and does not fetch all candidates' documents. |
| NFR-05 | Candidate personal data must not appear in application logs. |
| NFR-06 | Scorecard visibility (FR-I-04) is enforced in the query, not by hiding in the UI — the same rule as feature 09's review drafts. |
| NFR-07 | The retention job must be resumable and must never delete more than its intended set; a dry run reports counts before any destructive run. |

## Permission keys added by this feature

| Key | Meaning | Default holders |
|---|---|---|
| `recruitment.requisition.read` / `.write` | Requisitions | HR: ALL · Manager: own |
| `recruitment.requisition.approve` | Approve a requisition | Configured approver |
| `recruitment.posting.write` | Write and publish postings | HR: ALL |
| `recruitment.candidate.read` | See candidates and applications | HR: ALL · Hiring manager: own requisitions |
| `recruitment.candidate.write` | Add and edit candidates, move stages | HR: ALL |
| `recruitment.interview.schedule` | Schedule interviews | HR: ALL · Hiring manager: own |
| `recruitment.scorecard.submit` | Fill in a scorecard you are assigned | Assigned interviewers |
| `recruitment.offer.read` / `.write` | Offers, including compensation | HR: ALL |
| `recruitment.hire` | Convert a candidate to an employee | HR: ALL |
| `recruitment.retention` | Run purges, action erasure requests | HR: ALL |
| `onboarding.read` | See onboarding tasks | HR: ALL · Manager: own reports · Employee: SELF |
| `onboarding.write` | Configure templates, add and complete tasks | HR: ALL · task owners for their own |

## Acceptance criteria (feature-level)

1. A posting cannot be published until its requisition is approved, where approval is enabled.
2. A public application creates a candidate and application, stores the consent text and version,
   and returns no internal identifiers.
3. Submitting the public form twice with the same email does not reveal that the first application
   exists.
4. An oversized or wrong-type CV upload is rejected without creating a partial candidate.
5. A returning applicant is flagged to the recruiter, and nothing is merged automatically.
6. An interviewer cannot retrieve another interviewer's unsubmitted scorecard through any endpoint,
   and cannot tell whether one exists.
7. Once all scorecards are submitted, they become visible together.
8. The system produces no candidate score, ranking, or recommendation anywhere.
9. Rejection stores the internal reason and the sent message separately, and the internal reason is
   never sent.
10. The hire preview shows the employee to be created, and a duplicate employee code fails the hire
    with that error and writes nothing.
11. A completed hire creates the employee through feature 02's path, links the candidate, and
    instantiates onboarding tasks with dates derived from the start date.
12. Compensation, shift, and leave policy are not set by the hire, and appear as outstanding tasks.
13. The retention job's dry run reports what would be deleted; the real run deletes personal data
    and documents, keeps an anonymised application record, and logs the deletion.
14. A hired candidate is exempt from the purge.
