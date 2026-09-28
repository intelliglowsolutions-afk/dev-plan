# 09 — Performance Management — Data Model

Conventions as in features 01–08.

> **Revised 2026-09-18 — multi-tenant (OQ-301).** [MULTI-TENANCY.md](../MULTI-TENANCY.md) governs.
> - Every model gains `tenantId`. `RatingScale.name`, `ReviewTemplate.name` and
>   `PerformanceCycle.name` become unique per tenant.
> - Cycles, templates and scales are per-tenant configuration; the example scale and template become
>   provisioning options rather than migration seeds (D-T-07).
> - `performance.read_content` remains granted to **nobody** by default — now meaning nobody in any
>   tenant, and explicitly not the platform operator (D-T-04).
> - Review content and access logs are retained, not swept (OQ-1002, and OQ-908's "employment plus
>   two years" becomes a review prompt rather than a deletion).
> - Answered since drafting: **no draft answer history** (OQ-910 — so an overwritten paragraph is
>   gone, accepted), **no "who read my review"** for employees (OQ-909), and a new manager sees past
>   reviews only where **HR grants it** (OQ-903).

## Entity overview

```
RatingScale ──*── RatingScaleLevel
     │
ReviewTemplate ──*── ReviewSection ──*── ReviewQuestion
     │
PerformanceCycle ──*── CycleParticipant ──*── ReviewInstance ──*── ReviewAnswer
                                                   │
                                                   ├── templateSnapshot (JSON)
                                                   ├──*── ReviewComment   (appendix after acknowledgement)
                                                   └──*── ReviewAccessLog (who read it)

Goal ──*── GoalCheckIn
  └── parentGoalId (self)

Feedback
FeedbackRequest
```

The two structural ideas carried over from earlier features: **snapshots** (feature 07's payslip,
applied here to templates and rating labels) and a **separate read log** (feature 07's payslip
access log, applied here to review content).

## Enums

```prisma
enum GoalStatus {
  DRAFT
  ACTIVE
  ACHIEVED
  PARTIALLY_ACHIEVED
  NOT_ACHIEVED
  CANCELLED
}

enum GoalVisibility {
  PRIVATE       // owner and manager only (default)
  TEAM          // the owner's department can see it (OQ-906)
}

enum CycleStatus {
  DRAFT
  OPEN
  CLOSED
}

enum ReviewerType {
  SELF
  MANAGER
  PEER
  SKIP_LEVEL
}

enum ReviewState {
  NOT_STARTED
  DRAFT                          // author only (FR-V-01)
  SUBMITTED                      // + HR, + manager for self-reviews
  SHARED                         // + the subject (manager reviews only)
  ACKNOWLEDGED
  ACKNOWLEDGED_WITH_DISAGREEMENT // FR-R-06
  DECLINED                       // a peer declined to review
  INCOMPLETE                     // the cycle closed without it being finished
}

enum QuestionType {
  FREE_TEXT
  RATING
  RATING_WITH_COMMENT
  SINGLE_CHOICE
  MULTI_CHOICE
  GOAL_REVIEW                    // auto-populated from the subject's goals
}

enum FeedbackVisibility {
  SUBJECT_ONLY
  SUBJECT_AND_MANAGER
  MANAGER_ONLY                   // the subject may request it via HR (FR-B-06)
}
```

`ACKNOWLEDGED_WITH_DISAGREEMENT` exists as its own state rather than a boolean, because it must be
impossible to render a disagreed review as a plain acknowledgement by forgetting a flag.

## Goals

```prisma
model Goal {
  id           Int            @id @default(autoincrement())
  employeeId   Int            @map("employee_id")
  cycleId      Int?           @map("cycle_id")
  parentGoalId Int?           @map("parent_goal_id")

  title        String
  description  String?
  status       GoalStatus     @default(ACTIVE)
  visibility   GoalVisibility @default(PRIVATE)

  startDate    DateTime?      @map("start_date") @db.Date
  targetDate   DateTime?      @map("target_date") @db.Date

  // Optional measure. Null unit means the goal is tracked by percentage only.
  targetValue  Decimal?       @map("target_value") @db.Decimal(14, 2)
  currentValue Decimal?       @map("current_value") @db.Decimal(14, 2)
  unit         String?
  progressPercent Int?        @map("progress_percent")

  closedAt     DateTime?      @map("closed_at")
  closeReason  String?        @map("close_reason")

  createdById  Int?           @map("created_by_id")
  createdAt    DateTime       @default(now()) @map("created_at")
  updatedById  Int?           @map("updated_by_id")
  updatedAt    DateTime       @updatedAt @map("updated_at")

  employee     Employee       @relation(fields: [employeeId], references: [id], onDelete: Cascade)
  cycle        PerformanceCycle? @relation(fields: [cycleId], references: [id])
  parent       Goal?          @relation("GoalTree", fields: [parentGoalId], references: [id])
  children     Goal[]         @relation("GoalTree")
  checkIns     GoalCheckIn[]

  @@index([employeeId, status])
  @@index([cycleId])
  @@map("goals")
}

model GoalCheckIn {
  id          Int      @id @default(autoincrement())
  goalId      Int      @map("goal_id")
  note        String
  progressPercent Int? @map("progress_percent")
  currentValue Decimal? @map("current_value") @db.Decimal(14, 2)
  statusAtCheckIn GoalStatus? @map("status_at_check_in")

  authorUserId Int?    @map("author_user_id")
  createdAt   DateTime @default(now()) @map("created_at")

  goal        Goal     @relation(fields: [goalId], references: [id], onDelete: Cascade)

  @@index([goalId, createdAt])
  @@map("goal_check_ins")
}
```

Check-ins are append-only (FR-G-05): there is no update path, and a correction is a new row. The
goal's `progressPercent` is a denormalised copy of the latest check-in, for listing — the same
pattern as feature 02's current-assignment columns.

## Scales and templates

```prisma
model RatingScale {
  id          Int                @id @default(autoincrement())
  name        String             @unique
  description String?
  isActive    Boolean            @default(true) @map("is_active")
  createdAt   DateTime           @default(now()) @map("created_at")
  updatedAt   DateTime           @updatedAt @map("updated_at")

  levels      RatingScaleLevel[]
  questions   ReviewQuestion[]

  @@map("rating_scales")
}

model RatingScaleLevel {
  id         Int         @id @default(autoincrement())
  scaleId    Int         @map("scale_id")
  value      Int                                  // ordered, e.g. 1..5
  label      String                               // "Exceeds expectations"
  definition String?                              // what that actually means here
  colour     String?

  scale      RatingScale @relation(fields: [scaleId], references: [id], onDelete: Cascade)

  @@unique([scaleId, value])
  @@map("rating_scale_levels")
}

model ReviewTemplate {
  id          Int             @id @default(autoincrement())
  name        String          @unique
  description String?
  isActive    Boolean         @default(true) @map("is_active")
  createdAt   DateTime        @default(now()) @map("created_at")
  updatedAt   DateTime        @updatedAt @map("updated_at")

  sections    ReviewSection[]
  cycles      PerformanceCycle[]

  @@map("review_templates")
}

model ReviewSection {
  id          Int              @id @default(autoincrement())
  templateId  Int              @map("template_id")
  sectionOrder Int             @map("section_order")
  title       String
  description String?

  template    ReviewTemplate   @relation(fields: [templateId], references: [id], onDelete: Cascade)
  questions   ReviewQuestion[]

  @@unique([templateId, sectionOrder])
  @@map("review_sections")
}

model ReviewQuestion {
  id           Int           @id @default(autoincrement())
  sectionId    Int           @map("section_id")
  questionOrder Int          @map("question_order")

  prompt       String
  helpText     String?       @map("help_text")
  type         QuestionType
  ratingScaleId Int?         @map("rating_scale_id")
  choices      Json?                               // for SINGLE_CHOICE / MULTI_CHOICE
  isRequired   Boolean       @default(false) @map("is_required")

  // Which reviewer types answer this question (FR-F-03).
  answeredBy   ReviewerType[] @map("answered_by")

  section      ReviewSection @relation(fields: [sectionId], references: [id], onDelete: Cascade)
  ratingScale  RatingScale?  @relation(fields: [ratingScaleId], references: [id])

  @@unique([sectionId, questionOrder])
  @@map("review_questions")
}
```

## Cycles and instances

```prisma
model PerformanceCycle {
  id            Int         @id @default(autoincrement())
  name          String      @unique                 // "2026 annual review"
  templateId    Int         @map("template_id")
  status        CycleStatus @default(DRAFT)

  periodStart   DateTime    @map("period_start") @db.Date
  periodEnd     DateTime    @map("period_end") @db.Date

  selfReviewOpensOn   DateTime? @map("self_review_opens_on") @db.Date
  selfReviewDueOn     DateTime? @map("self_review_due_on") @db.Date
  managerReviewDueOn  DateTime? @map("manager_review_due_on") @db.Date
  sharingDueOn        DateTime? @map("sharing_due_on") @db.Date
  acknowledgementDueOn DateTime? @map("acknowledgement_due_on") @db.Date

  // Peer reviews (OQ-904). Anonymity is only meaningful above the threshold (D-05, FR-V-11).
  peerReviewEnabled   Boolean @default(false) @map("peer_review_enabled")
  peerAnonymous       Boolean @default(false) @map("peer_anonymous")
  peerMinimumResponses Int    @default(3) @map("peer_minimum_responses")

  openedAt      DateTime?   @map("opened_at")
  closedAt      DateTime?   @map("closed_at")
  createdById   Int?        @map("created_by_id")
  createdAt     DateTime    @default(now()) @map("created_at")

  template      ReviewTemplate @relation(fields: [templateId], references: [id])
  participants  CycleParticipant[]
  goals         Goal[]

  @@map("performance_cycles")
}

model CycleParticipant {
  id           Int              @id @default(autoincrement())
  cycleId      Int              @map("cycle_id")
  employeeId   Int              @map("employee_id")
  // Resolved and stored when the cycle opens (FR-C-02) — not looked up later.
  managerEmployeeId Int?        @map("manager_employee_id")

  exclusionReason String?       @map("exclusion_reason")  // why someone was left out (FR-C-03)
  removedAt    DateTime?        @map("removed_at")        // left mid-cycle (FR-C-09)

  createdAt    DateTime         @default(now()) @map("created_at")

  cycle        PerformanceCycle @relation(fields: [cycleId], references: [id], onDelete: Cascade)
  employee     Employee         @relation(fields: [employeeId], references: [id], onDelete: Cascade)
  instances    ReviewInstance[]

  @@unique([cycleId, employeeId])
  @@map("cycle_participants")
}

model ReviewInstance {
  id            Int              @id @default(autoincrement())
  participantId Int              @map("participant_id")
  subjectEmployeeId Int          @map("subject_employee_id")
  reviewerUserId Int?            @map("reviewer_user_id")
  type          ReviewerType
  state         ReviewState      @default(NOT_STARTED)

  // The template as it was when this instance was created (FR-F-05).
  templateSnapshot Json          @map("template_snapshot")

  startedAt     DateTime?        @map("started_at")
  lastSavedAt   DateTime?        @map("last_saved_at")     // surfaced by autosave (FR-R-02)
  submittedAt   DateTime?        @map("submitted_at")
  sharedAt      DateTime?        @map("shared_at")
  acknowledgedAt DateTime?       @map("acknowledged_at")
  acknowledgementNote String?    @map("acknowledgement_note")

  unlockedById  Int?             @map("unlocked_by_id")
  unlockReason  String?          @map("unlock_reason")

  createdAt     DateTime         @default(now()) @map("created_at")

  participant   CycleParticipant @relation(fields: [participantId], references: [id], onDelete: Cascade)
  answers       ReviewAnswer[]
  comments      ReviewComment[]
  accessLogs    ReviewAccessLog[]

  @@unique([participantId, reviewerUserId, type])
  @@index([reviewerUserId, state])
  @@index([subjectEmployeeId])
  @@map("review_instances")
}

model ReviewAnswer {
  id           Int            @id @default(autoincrement())
  instanceId   Int            @map("instance_id")
  questionId   Int            @map("question_id")          // navigation only; the snapshot governs

  textValue    String?        @map("text_value")
  ratingValue  Int?           @map("rating_value")
  // Snapshotted so a "3" still means something after the scale is rewritten (D-07, FR-F-07).
  ratingLabel  String?        @map("rating_label")
  ratingDefinition String?    @map("rating_definition")
  choiceValues Json?          @map("choice_values")
  goalId       Int?           @map("goal_id")              // GOAL_REVIEW answers

  updatedAt    DateTime       @updatedAt @map("updated_at")

  instance     ReviewInstance @relation(fields: [instanceId], references: [id], onDelete: Cascade)

  @@unique([instanceId, questionId, goalId])
  @@map("review_answers")
}

model ReviewComment {
  id          Int            @id @default(autoincrement())
  instanceId  Int            @map("instance_id")
  authorUserId Int?          @map("author_user_id")
  body        String
  createdAt   DateTime       @default(now()) @map("created_at")

  instance    ReviewInstance @relation(fields: [instanceId], references: [id], onDelete: Cascade)

  @@index([instanceId, createdAt])
  @@map("review_comments")
}

model ReviewAccessLog {
  id         Int            @id @default(autoincrement())
  instanceId Int            @map("instance_id")
  userId     Int            @map("user_id")
  createdAt  DateTime       @default(now()) @map("created_at")

  instance   ReviewInstance @relation(fields: [instanceId], references: [id], onDelete: Cascade)

  @@index([instanceId, createdAt])
  @@index([userId, createdAt])
  @@map("review_access_logs")
}
```

## Feedback

```prisma
model Feedback {
  id          Int                @id @default(autoincrement())
  subjectEmployeeId Int          @map("subject_employee_id")
  // Always recorded, even when display is anonymised (D-05, FR-B-02).
  authorUserId Int?              @map("author_user_id")
  cycleId     Int?               @map("cycle_id")
  requestId   Int?               @map("request_id")

  body        String
  visibility  FeedbackVisibility @default(SUBJECT_AND_MANAGER)
  isAnonymousDisplay Boolean     @default(false) @map("is_anonymous_display")

  seenBySubjectAt DateTime?      @map("seen_by_subject_at")   // after this, no edits (FR-B-05)
  createdAt   DateTime           @default(now()) @map("created_at")

  subject     Employee           @relation(fields: [subjectEmployeeId], references: [id], onDelete: Cascade)

  @@index([subjectEmployeeId, createdAt])
  @@index([cycleId])
  @@map("feedback")
}

model FeedbackRequest {
  id          Int       @id @default(autoincrement())
  subjectEmployeeId Int @map("subject_employee_id")
  askedUserId Int       @map("asked_user_id")
  message     String?
  cycleId     Int?      @map("cycle_id")

  respondedAt DateTime? @map("responded_at")
  declinedAt  DateTime? @map("declined_at")       // no reason required (FR-B-03)
  createdAt   DateTime  @default(now()) @map("created_at")

  @@index([askedUserId, respondedAt])
  @@map("feedback_requests")
}
```

## Design notes

**Why `templateSnapshot` is on the instance.** Editing a template mid-cycle would otherwise change
questions people have already answered, orphaning their answers. The snapshot makes an instance
self-contained — the same reasoning as feature 07's payslip, and the fourth place in this plan where
snapshotting solves "the rules changed after the record was made".

**Why `ReviewAnswer` keeps a `questionId` at all, given the snapshot.** For navigation and
reporting — grouping answers to the same question across instances. The snapshot governs rendering;
the FK governs aggregation. When they disagree, the snapshot wins.

**Why the read log is separate from 01's audit.** Identical reasoning to feature 07's payslip access
log: reads are frequent, queried by instance and by reader, and would drown the change history if
mixed in (FR-V-07).

**Why `CycleParticipant.managerEmployeeId` is stored.** Reporting lines change mid-cycle. The
reviewer must not change underneath a half-written review, and "who was your manager for the 2026
cycle" must stay answerable afterwards (FR-C-02, D-09). Same reasoning as feature 06's stored
approval chain.

**Why `Feedback.authorUserId` is kept even for anonymous display.** Anonymity is a display choice,
not an absence of a record. An anonymous channel with no stored author cannot be investigated when
it is used to abuse someone — and it will be, eventually. The threshold rule (FR-V-11) provides the
protection; deleting the author would provide only deniability.

**What is deliberately absent from this model.** There is no field, table, or FK linking to
`attendance_days`, `leave_ledger_entries`, or `payslips`. That absence is the implementation of
D-04 and FR-X-01, and it should be noticed in review if anyone adds one.

## Volume

| Table | Rows/year at 200 employees |
|---|---|
| `goals` | ~800 (4 per person) |
| `goal_check_ins` | ~3 000 |
| `review_instances` | ~400 (self + manager, one cycle) |
| `review_answers` | ~8 000 |
| `review_access_logs` | ~2 000 |
| `feedback` | hundreds |

Small. As with payroll, the risk here is not volume; it is who can see what.

## Seed and migration notes

Migration `performance`:

1. Create the tables above.
2. Seed permission keys and role grants — **`performance.read_content` is granted to nobody**,
   including HR admin (FR-V-06, D-08). It is granted deliberately, per person, when needed.
3. Seed 05's notification types: `performance.cycle_opened`, `performance.self_review_due`,
   `performance.manager_review_due`, `performance.review_shared`,
   `performance.acknowledgement_due`, `performance.feedback_received`,
   `performance.feedback_requested`, `performance.goal_overdue`. **None may contain review content**
   (NFR-03) — the same denylist discipline payroll needed for figures.
4. Seed **one example** rating scale and one example template, clearly labelled as examples —
   following the precedent of 04's example shift, 06's example leave types, and 07's example
   components. Do not seed a cycle.
5. No settings in 03's catalogue beyond `performance.peerMinimumResponses` (3) and
   `performance.skipLevelAccess` (false, OQ-903).

## Open questions

| ID | Question |
|---|---|
| OQ-901 | The whole model assumes a cycle-based process. A company doing continuous check-ins with no formal review needs the goals and feedback halves and none of the cycle machinery — which is most of this document. |
| OQ-904 | Peer reviews are modelled (`ReviewerType.PEER`, anonymity threshold) but not built. Building them adds reviewer selection, nomination approval, and aggregate-only rendering. |
| OQ-908 | Retention. Performance records about former employees are the clearest case in the system for a deletion policy rather than indefinite retention. |
| OQ-910 | Should `ReviewAnswer` keep a version history while in draft? Autosave overwrites; a manager who deletes a paragraph and saves cannot recover it. A simple append-only draft history would fix that cheaply. |
