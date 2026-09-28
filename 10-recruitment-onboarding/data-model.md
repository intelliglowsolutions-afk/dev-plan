# 10 — Recruitment & Onboarding — Data Model

Conventions as in features 01–09.

> **Revised 2026-09-18.** [MULTI-TENANCY.md](../MULTI-TENANCY.md) governs.
> - Every model gains `tenantId` — **except that `JobPosting.publicSlug` stays globally unique**,
>   because it resolves the tenant for the unauthenticated public form before any tenant is known
>   (D-T-05). It must therefore be unguessable, not sequential.
> - Candidate documents are stored under a per-tenant prefix (D-T-10).
> - **Automatic deletion is removed** (OQ-1002): `retentionUntil` still drives a weekly *review*
>   report, but nothing is deleted without an HR action. `anonymisedAt` and the erasure path stay.
>   The consent notice must be reworded to describe what actually happens.
> - Confirmed since drafting: both intake routes exist — public form **and** manual entry (OQ-1003);
>   no CV parsing or AI shortlisting (OQ-1005); and **feature 10 is built last**, after 11 (OQ-1001).

## Entity overview

```
Requisition ──*── JobPosting ──*── Application ──*── Candidate
     │                                  │                │
     │                                  │                ├──*── CandidateDocument
     │                                  │                └──*── CandidateConsent
     │                                  │
     │                                  ├──*── ApplicationStageEvent   (append-only history)
     │                                  ├──*── Interview ──*── InterviewScorecard ──*── ScorecardAnswer
     │                                  └──0..1─ Offer
     │
     └── PipelineTemplate ──*── PipelineStage

Employee (02) ──0..1── Candidate        (set at hire — the hinge)
     └──*── OnboardingTask  ←── OnboardingTemplate ──*── OnboardingTaskDefinition
```

Two things to notice. First, `Candidate` and `Employee` are separate tables joined by a nullable
link set at hire (D-01). Second, this is the **only feature in the plan whose data is designed to be
deleted** — `CandidateConsent` and the retention fields exist for that purpose (D-02).

## Enums

```prisma
enum RequisitionStatus {
  DRAFT
  PENDING_APPROVAL
  APPROVED
  DECLINED
  ON_HOLD
  FILLED
  CANCELLED
}

enum PostingStatus {
  DRAFT
  PUBLISHED
  CLOSED
}

enum ApplicationOutcome {
  IN_PROGRESS
  HIRED
  REJECTED       // we declined them
  WITHDRAWN      // they withdrew — a different fact (FR-P-05)
}

enum InterviewType {
  PHONE_SCREEN
  TECHNICAL
  PANEL
  FINAL
  OTHER
}

enum InterviewStatus {
  SCHEDULED
  COMPLETED
  CANCELLED
  NO_SHOW
}

enum Recommendation {
  STRONG_YES
  YES
  NO
  STRONG_NO
}

enum OfferStatus {
  DRAFT
  SENT
  ACCEPTED
  DECLINED
  WITHDRAWN
  EXPIRED
}

enum OnboardingOwnerRole {
  HR
  MANAGER
  IT
  EMPLOYEE
}

enum OnboardingTaskStatus {
  PENDING
  DONE
  NOT_APPLICABLE
  BLOCKED
}

enum CandidateSource {
  DIRECT_APPLICATION
  REFERRAL
  AGENCY
  JOB_BOARD
  OTHER
}
```

## Requisitions, postings, pipeline

```prisma
model Requisition {
  id             Int               @id @default(autoincrement())
  title          String
  departmentId   Int?              @map("department_id")
  positionId     Int?              @map("position_id")
  hiringManagerId Int?             @map("hiring_manager_id")     // Employee
  employmentType EmploymentType?   @map("employment_type")

  headcount      Int               @default(1)
  headcountFilled Int              @default(0) @map("headcount_filled")
  isReplacement  Boolean           @default(false) @map("is_replacement")
  replacingEmployeeId Int?         @map("replacing_employee_id")
  reason         String?
  targetStartDate DateTime?        @map("target_start_date") @db.Date

  status         RequisitionStatus @default(DRAFT)
  approvedById   Int?              @map("approved_by_id")
  approvedAt     DateTime?         @map("approved_at")
  decisionNote   String?           @map("decision_note")

  pipelineTemplateId Int?          @map("pipeline_template_id")

  createdById    Int?              @map("created_by_id")
  createdAt      DateTime          @default(now()) @map("created_at")
  updatedAt      DateTime          @updatedAt @map("updated_at")

  postings       JobPosting[]

  @@index([status])
  @@map("requisitions")
}

model JobPosting {
  id             Int           @id @default(autoincrement())
  requisitionId  Int           @map("requisition_id")

  title          String
  summary        String?
  // Sanitised on save — rendered to the open internet (FR-Q-08).
  descriptionHtml String?      @map("description_html")
  location       String?
  showSalary     Boolean       @default(false) @map("show_salary")
  salaryText     String?       @map("salary_text")

  // Unguessable public identifier: the URL is public, the row id is not exposed (NFR-03).
  publicSlug     String        @unique @map("public_slug")
  status         PostingStatus @default(DRAFT)
  publishedAt    DateTime?     @map("published_at")
  closedAt       DateTime?     @map("closed_at")

  createdAt      DateTime      @default(now()) @map("created_at")
  updatedAt      DateTime      @updatedAt @map("updated_at")

  requisition    Requisition   @relation(fields: [requisitionId], references: [id], onDelete: Cascade)
  applications   Application[]

  @@index([status])
  @@map("job_postings")
}

model PipelineTemplate {
  id        Int             @id @default(autoincrement())
  name      String          @unique
  isDefault Boolean         @default(false) @map("is_default")
  isActive  Boolean         @default(true) @map("is_active")
  createdAt DateTime        @default(now()) @map("created_at")

  stages    PipelineStage[]

  @@map("pipeline_templates")
}

model PipelineStage {
  id            Int              @id @default(autoincrement())
  templateId    Int              @map("template_id")
  stageOrder    Int              @map("stage_order")
  name          String
  requiresInterview Boolean      @default(false) @map("requires_interview")
  requiresScorecard Boolean      @default(false) @map("requires_scorecard")
  // Days after which a candidate sitting here is surfaced to the recruiter (FR-P-06).
  stallAfterDays Int?            @map("stall_after_days")

  template      PipelineTemplate @relation(fields: [templateId], references: [id], onDelete: Cascade)

  @@unique([templateId, stageOrder])
  @@map("pipeline_stages")
}
```

## Candidates and applications

```prisma
model Candidate {
  id            Int             @id @default(autoincrement())
  firstName     String          @map("first_name")
  lastName      String          @map("last_name")
  email         String
  phone         String?
  location      String?
  linkedinUrl   String?         @map("linkedin_url")
  source        CandidateSource @default(DIRECT_APPLICATION)
  referredByEmployeeId Int?     @map("referred_by_employee_id")

  salaryExpectation String?     @map("salary_expectation")   // free text: "45–50k", "negotiable"
  noticePeriod  String?         @map("notice_period")

  // The hinge (D-01). Set at hire; also exempts this row from the purge (FR-R-04).
  hiredEmployeeId Int?          @unique @map("hired_employee_id")

  // Retention (D-02, FR-R-04). Computed from the last decision date plus the configured period.
  retentionUntil DateTime?      @map("retention_until") @db.Date
  keepOnFile    Boolean         @default(false) @map("keep_on_file")   // explicit opt-in
  anonymisedAt  DateTime?       @map("anonymised_at")

  createdById   Int?            @map("created_by_id")       // null = applied through the public form
  createdAt     DateTime        @default(now()) @map("created_at")
  updatedAt     DateTime        @updatedAt @map("updated_at")

  hiredEmployee Employee?       @relation(fields: [hiredEmployeeId], references: [id])
  applications  Application[]
  documents     CandidateDocument[]
  consents      CandidateConsent[]
  notes         CandidateNote[]

  // Not unique: the same person may apply years apart, and matching is a warning,
  // not an automatic merge (FR-C-02).
  @@index([email])
  @@index([retentionUntil])
  @@map("candidates")
}

model Application {
  id            Int                @id @default(autoincrement())
  candidateId   Int                @map("candidate_id")
  postingId     Int                @map("posting_id")

  currentStageId Int?              @map("current_stage_id")
  currentStageEnteredAt DateTime?  @map("current_stage_entered_at")
  outcome       ApplicationOutcome @default(IN_PROGRESS)

  coverNote     String?            @map("cover_note")

  // Internal reason and the message actually sent are separate fields (D-09, FR-C-06).
  rejectionReasonCode String?      @map("rejection_reason_code")
  rejectionInternalNote String?    @map("rejection_internal_note")
  rejectionMessageSent String?     @map("rejection_message_sent")

  decidedAt     DateTime?          @map("decided_at")
  createdAt     DateTime           @default(now()) @map("created_at")
  updatedAt     DateTime           @updatedAt @map("updated_at")

  candidate     Candidate          @relation(fields: [candidateId], references: [id], onDelete: Cascade)
  posting       JobPosting         @relation(fields: [postingId], references: [id])
  stageEvents   ApplicationStageEvent[]
  interviews    Interview[]
  offer         Offer?

  @@unique([candidateId, postingId])
  @@index([postingId, outcome])
  @@index([currentStageId])
  @@map("applications")
}

model ApplicationStageEvent {
  id            Int         @id @default(autoincrement())
  applicationId Int         @map("application_id")
  fromStageId   Int?        @map("from_stage_id")
  toStageId     Int?        @map("to_stage_id")
  note          String?
  movedById     Int?        @map("moved_by_id")
  createdAt     DateTime    @default(now()) @map("created_at")

  application   Application @relation(fields: [applicationId], references: [id], onDelete: Cascade)

  // Append-only (FR-C-05). No update or delete path exists.
  @@index([applicationId, createdAt])
  @@map("application_stage_events")
}

model CandidateDocument {
  id               Int       @id @default(autoincrement())
  candidateId      Int       @map("candidate_id")
  kind             String                                  // "CV", "COVER_LETTER", "PORTFOLIO"
  storageKey       String    @unique @map("storage_key")
  originalFilename String    @map("original_filename")
  mimeType         String    @map("mime_type")
  sizeBytes        Int       @map("size_bytes")
  checksumSha256   String    @map("checksum_sha256")
  uploadedAt       DateTime  @default(now()) @map("uploaded_at")
  deletedAt        DateTime? @map("deleted_at")            // set by the purge

  candidate        Candidate @relation(fields: [candidateId], references: [id], onDelete: Cascade)

  @@index([candidateId])
  @@map("candidate_documents")
}

model CandidateConsent {
  id            Int       @id @default(autoincrement())
  candidateId   Int       @map("candidate_id")
  // The notice text as shown, not a reference to a document that may later change (FR-R-01).
  noticeVersion String    @map("notice_version")
  noticeText    String    @map("notice_text")
  retentionMonths Int     @map("retention_months")
  keepOnFileOptIn Boolean @default(false) @map("keep_on_file_opt_in")
  givenAt       DateTime  @default(now()) @map("given_at")
  ipAddress     String?   @map("ip_address")

  candidate     Candidate @relation(fields: [candidateId], references: [id], onDelete: Cascade)

  @@index([candidateId])
  @@map("candidate_consents")
}

model CandidateNote {
  id          Int       @id @default(autoincrement())
  candidateId Int       @map("candidate_id")
  body        String
  authorUserId Int?     @map("author_user_id")
  createdAt   DateTime  @default(now()) @map("created_at")

  candidate   Candidate @relation(fields: [candidateId], references: [id], onDelete: Cascade)

  @@index([candidateId, createdAt])
  @@map("candidate_notes")
}
```

## Interviews and scorecards

```prisma
model Interview {
  id            Int             @id @default(autoincrement())
  applicationId Int             @map("application_id")
  type          InterviewType
  status        InterviewStatus @default(SCHEDULED)

  scheduledAt   DateTime        @map("scheduled_at")
  durationMinutes Int           @default(60) @map("duration_minutes")
  location      String?
  meetingLink   String?         @map("meeting_link")
  instructions  String?

  scorecardTemplateId Int?      @map("scorecard_template_id")
  cancelledReason String?       @map("cancelled_reason")

  createdById   Int?            @map("created_by_id")
  createdAt     DateTime        @default(now()) @map("created_at")

  application   Application     @relation(fields: [applicationId], references: [id], onDelete: Cascade)
  scorecards    InterviewScorecard[]

  @@index([applicationId])
  @@index([scheduledAt])
  @@map("interviews")
}

model InterviewScorecard {
  id             Int             @id @default(autoincrement())
  interviewId    Int             @map("interview_id")
  interviewerUserId Int          @map("interviewer_user_id")

  // Not visible to other interviewers until submittedAt is set (D-04, FR-I-04).
  submittedAt    DateTime?       @map("submitted_at")
  lastSavedAt    DateTime?       @map("last_saved_at")

  recommendation Recommendation?
  notes          String?

  interview      Interview       @relation(fields: [interviewId], references: [id], onDelete: Cascade)
  answers        ScorecardAnswer[]

  @@unique([interviewId, interviewerUserId])
  @@index([interviewerUserId, submittedAt])
  @@map("interview_scorecards")
}

model ScorecardAnswer {
  id           Int                @id @default(autoincrement())
  scorecardId  Int                @map("scorecard_id")
  criterionKey String             @map("criterion_key")
  criterionLabel String           @map("criterion_label")   // snapshotted
  ratingValue  Int?               @map("rating_value")
  ratingLabel  String?            @map("rating_label")      // snapshotted (09 D-07's discipline)
  comment      String?

  scorecard    InterviewScorecard @relation(fields: [scorecardId], references: [id], onDelete: Cascade)

  @@unique([scorecardId, criterionKey])
  @@map("scorecard_answers")
}

model Offer {
  id             Int             @id @default(autoincrement())
  applicationId  Int             @unique @map("application_id")

  positionId     Int?            @map("position_id")
  departmentId   Int?            @map("department_id")
  managerEmployeeId Int?         @map("manager_employee_id")
  employmentType EmploymentType? @map("employment_type")
  proposedStartDate DateTime?    @map("proposed_start_date") @db.Date

  // Stored here, NOT written to feature 07 until the hire is complete (FR-O-02).
  proposedSalary Decimal?        @map("proposed_salary") @db.Decimal(14, 2)
  currency       String?
  salaryNote     String?         @map("salary_note")

  status         OfferStatus     @default(DRAFT)
  sentAt         DateTime?       @map("sent_at")
  expiresOn      DateTime?       @map("expires_on") @db.Date
  respondedAt    DateTime?       @map("responded_at")
  declineReason  String?         @map("decline_reason")     // the most useful data here (FR-O-04)

  createdById    Int?            @map("created_by_id")
  createdAt      DateTime        @default(now()) @map("created_at")
  updatedAt      DateTime        @updatedAt @map("updated_at")

  application    Application     @relation(fields: [applicationId], references: [id], onDelete: Cascade)

  @@map("offers")
}
```

## Onboarding

```prisma
model OnboardingTemplate {
  id             Int                        @id @default(autoincrement())
  name           String                     @unique
  departmentId   Int?                       @map("department_id")
  employmentType EmploymentType?            @map("employment_type")
  isDefault      Boolean                    @default(false) @map("is_default")
  isActive       Boolean                    @default(true) @map("is_active")
  createdAt      DateTime                   @default(now()) @map("created_at")

  definitions    OnboardingTaskDefinition[]

  @@map("onboarding_templates")
}

model OnboardingTaskDefinition {
  id          Int                 @id @default(autoincrement())
  templateId  Int                 @map("template_id")
  taskOrder   Int                 @map("task_order")
  title       String
  description String?
  ownerRole   OnboardingOwnerRole
  // Relative to the start date (D-06): -2 = two days before, +5 = five days after.
  dueOffsetDays Int               @map("due_offset_days")
  requiresDocument Boolean        @default(false) @map("requires_document")
  linkPath    String?             @map("link_path")   // e.g. the payroll compensation screen

  template    OnboardingTemplate  @relation(fields: [templateId], references: [id], onDelete: Cascade)

  @@unique([templateId, taskOrder])
  @@map("onboarding_task_definitions")
}

model OnboardingTask {
  id          Int                  @id @default(autoincrement())
  employeeId  Int                  @map("employee_id")
  definitionId Int?                @map("definition_id")   // null = added ad hoc (FR-N-10)

  title       String
  description String?
  ownerUserId Int?                 @map("owner_user_id")   // resolved at instantiation (FR-N-03)
  ownerRole   OnboardingOwnerRole
  dueOn       DateTime?            @map("due_on") @db.Date
  linkPath    String?              @map("link_path")

  status      OnboardingTaskStatus @default(PENDING)
  statusNote  String?              @map("status_note")
  completedById Int?               @map("completed_by_id")
  completedAt DateTime?            @map("completed_at")
  documentId  Int?                 @map("document_id")     // EmployeeDocument (02)

  createdAt   DateTime             @default(now()) @map("created_at")

  employee    Employee             @relation(fields: [employeeId], references: [id], onDelete: Cascade)

  @@index([employeeId, status])
  @@index([ownerUserId, status])
  @@index([dueOn])
  @@map("onboarding_tasks")
}
```

## Design notes

**Why `publicSlug` exists.** The posting URL is public. Exposing `/jobs/14` leaks how many
requisitions exist and invites enumeration of drafts. An unguessable slug costs nothing (NFR-03).

**Why `Candidate.email` is indexed but not unique.** The same person may apply for two roles, or
apply again in three years. Uniqueness would force a merge decision at the worst moment — during an
unauthenticated public form submission — and merging people automatically is hard to undo (FR-C-02).

**Why the three rejection fields.** `rejectionReasonCode` is for statistics, `rejectionInternalNote`
is for the team, `rejectionMessageSent` is what the candidate received. One field for all three
means the internal note eventually gets sent (D-09), which is the kind of mistake that ends up on
social media.

**Why consent stores the notice *text*, not a version reference.** A reference points at a document
that will be edited. Proving what someone was actually told requires the words they saw (FR-R-01).

**Why `Offer.proposedSalary` lives here rather than in feature 07.** An offered figure is not a paid
figure — offers get declined and renegotiated. Writing it to payroll before the hire would create
compensation records for people who never join (FR-O-02).

**Why onboarding tasks hang off `Employee`, not `Candidate`.** D-10. The employee exists from the
hire, with a future start date, and feature 02 already supports that. A parallel pre-employee
identity would need its own permissions, its own portal access, and its own migration into the real
record.

**Why `OnboardingTask` copies its title and description from the definition.** Editing a template
must not rewrite checklists already issued — the same snapshot discipline as 07's payslip lines and
09's template snapshots.

## Retention mechanics

The purge job (FR-R-04) runs daily:

1. Select candidates where `retentionUntil < today`, `hiredEmployeeId IS NULL`,
   `keepOnFile = false`, and no application has `outcome = IN_PROGRESS`.
2. **Dry run first**: count and summarise. The scheduled job reports to HR a week ahead (FR-R-08).
3. On the real run, per candidate: delete documents from the volume, clear personal fields
   (name, email, phone, location, URLs, notes, consents), set `anonymisedAt`.
4. Keep the `Application` and `ApplicationStageEvent` rows with their posting, stage, outcome, and
   dates, so hiring statistics survive (FR-R-05).
5. Log what was deleted — counts and ids, never content (FR-R-06).

`retentionUntil` is set when an application reaches a terminal outcome, from
`recruitment.retentionMonths`. A candidate with several applications takes the latest date.

## Volume

| Table | Rows/year, a company hiring ~20 people |
|---|---|
| `candidates` | ~600 (30 applicants per hire) |
| `applications` | ~650 |
| `application_stage_events` | ~2 500 |
| `interviews` | ~200 |
| `interview_scorecards` | ~400 |
| `onboarding_tasks` | ~300 |

Small, and shrinking on a schedule — the only feature in the plan where that is true.

## Seed and migration notes

Migration `recruitment_onboarding`:

1. Create the tables above.
2. Add the nullable `hiredEmployeeId` relation; `Employee` gains a `candidate` back-relation only.
3. Seed permission keys and role grants.
4. Seed 05's notification types: `recruitment.requisition_pending_approval`,
   `recruitment.application_received`, `recruitment.interview_scheduled`,
   `recruitment.scorecard_due`, `recruitment.candidate_stalled`, `recruitment.offer_response`,
   `onboarding.task_due`, `onboarding.task_overdue`, `recruitment.retention_purge_upcoming`.
   **Candidate personal data must not appear in notification bodies** (FR-R-09) — the same denylist
   discipline payroll needed for figures.
5. Seed one **example** pipeline template, one scorecard template, and one onboarding template,
   labelled as examples — the now-standard precedent from 04, 06, 07, and 09.
6. Settings in 03's catalogue: `recruitment.retentionMonths` (6), `recruitment.requireRequisitionApproval`
   (true), `recruitment.publicFormEnabled` (true), `recruitment.stallWarningDays` (14),
   `recruitment.privacyNoticeVersion`, `recruitment.privacyNoticeText`.

## Open questions

| ID | Question |
|---|---|
| OQ-1002 | `recruitment.retentionMonths` defaults to 6 as a placeholder. The real value is a legal question, not a product one, and the model is built so changing it is a setting rather than a migration. |
| OQ-1011 | Should anonymised applications keep the posting link, or be aggregated into counts? Keeping rows preserves per-role statistics; aggregating removes even the possibility of re-identification by correlating dates. |
| OQ-1012 | Should `CandidateNote` be purged earlier than the rest? Free-text notes about people who were not hired are the most sensitive thing here and the least useful to keep. |
| OQ-1008 | Interviewers without accounts would need a tokenised scorecard link, which means an unauthenticated write endpoint — a second hostile surface after the public form. Worth avoiding if possible. |
