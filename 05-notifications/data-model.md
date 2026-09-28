# 05 — Notifications — Data Model

Conventions as in features 01–04.

> **Revised 2026-09-18 — multi-tenant (OQ-301).** [MULTI-TENANCY.md](../MULTI-TENANCY.md) governs.
> - `NotificationEvent`, `NotificationDelivery`, `NotificationPreference`, `NotificationTemplate`,
>   `NotificationDigestBatch` and `EmailSuppression` all gain `tenantId`. The type catalogue stays
>   global and code-declared (D-T-02).
> - **Templates are per tenant** — each tenant may word its own — while their definitions are not.
> - **Digests run per tenant, at that tenant's configured hour in that tenant's timezone**
>   (D-T-06.4), and the worker isolates failure per tenant.
> - SMTP settings are per-tenant values of a global setting definition (D-T-07), so tenants can send
>   from their own domain.
> - **The nightly retention sweep is removed** (OQ-1002). `notification_deliveries` is the
>   fastest-growing table in this feature and now grows without bound — an operational concern to
>   plan for rather than one that solves itself.
> - Managers may now see whether their team read a notification (OQ-508 answered yes), which
>   reverses the plan's default and adds a small surveillance surface.

## Entity overview

```
(catalogue: declared in code, not a table)
         │
         ▼
NotificationEvent ──*── NotificationDelivery ──┬── channel IN_APP  → the user's notification list
   (one per trigger)     (one per recipient      └── channel EMAIL  → the outbox the worker drains
                          per channel)
User ──*── NotificationPreference
NotificationTemplate      (overrides only; defaults live in code)
EmailSuppression          (bounced / rejected addresses)
```

The split between **event** and **delivery** is the model's one real decision. One trigger — a
device going offline — is a single event, fanned out to however many recipients and channels the
rules produce. Keeping them separate means the idempotency key lives in one place (D-05), the log
can answer both "what happened" and "who was told", and a retry re-sends one delivery rather than
re-firing the event.

## Enums

```prisma
enum NotificationChannel {
  IN_APP
  EMAIL
  // SMS / PUSH deliberately absent until OQ-501 is answered — adding a value later is trivial.
}

enum DeliveryStatus {
  PENDING
  HELD          // quiet hours, or waiting for the next digest
  SENDING       // claimed by a worker (NFR-06)
  SENT
  FAILED        // retries exhausted
  SKIPPED       // suppressed address, muted preference, or no address
  SUPERSEDED    // no longer relevant (FR-A-05)
}

enum SuppressionReason {
  HARD_BOUNCE
  COMPLAINT
  INVALID_ADDRESS
  MANUAL
}
```

`SKIPPED` is a status, not an absence of a row. A muted preference and a suppressed address both
produce a record explaining why nothing was sent — which is what makes "I never got told" answerable.

## Models

```prisma
model NotificationEvent {
  id          Int      @id @default(autoincrement())
  // Key from the code-declared catalogue (D-01). Not an FK — the catalogue is not a table.
  typeKey     String   @map("type_key")
  category    String

  // Supplied by the caller; a second write with the same key is ignored (D-05, FR-D-03).
  idempotencyKey String @unique @map("idempotency_key")

  // The template variables for this event. Small, flat, no sensitive values (D-06, FR-D-11).
  context     Json

  // What this is about, for deep links and permission re-checks at send time.
  subjectEmployeeId Int? @map("subject_employee_id")
  entityType  String?  @map("entity_type")
  entityId    String?  @map("entity_id")
  linkPath    String?  @map("link_path")      // relative; absolute URL built at render (FR-D-12)

  triggeredByUserId Int? @map("triggered_by_user_id")
  createdAt   DateTime @default(now()) @map("created_at")
  // Null until the worker has resolved recipients — resolution is deferred out of the
  // caller's request path (NFR-01).
  resolvedAt  DateTime? @map("resolved_at")
  recipientCount Int?  @map("recipient_count")   // 0 is meaningful (FR-R-05)

  deliveries  NotificationDelivery[]

  @@index([typeKey, createdAt])
  @@index([resolvedAt])
  @@index([subjectEmployeeId])
  @@map("notification_events")
}

model NotificationDelivery {
  id          Int                 @id @default(autoincrement())
  eventId     Int                 @map("event_id")
  userId      Int                 @map("user_id")
  channel     NotificationChannel
  status      DeliveryStatus      @default(PENDING)

  // Rendered at send time and kept: the subject only, never the body (FR-N-06).
  renderedTitle String?           @map("rendered_title")
  renderedBody  String?           @map("rendered_body")   // IN_APP only — short, shown in the list
  emailAddress  String?           @map("email_address")   // resolved at send time

  // IN_APP read state (FR-A-02).
  readAt      DateTime?           @map("read_at")

  attempts    Int                 @default(0)
  lastError   String?             @map("last_error")
  skipReason  String?             @map("skip_reason")     // why SKIPPED, in plain words
  nextAttemptAt DateTime?         @map("next_attempt_at")
  // Set while a worker holds this row; lets a crashed worker's claims expire (NFR-06).
  claimedAt   DateTime?           @map("claimed_at")
  sentAt      DateTime?           @map("sent_at")

  digestBatchId Int?              @map("digest_batch_id")

  createdAt   DateTime            @default(now()) @map("created_at")

  event       NotificationEvent   @relation(fields: [eventId], references: [id], onDelete: Cascade)
  user        User                @relation(fields: [userId], references: [id], onDelete: Cascade)

  // The badge query: unread in-app for one user (NFR-02).
  @@index([userId, channel, readAt])
  @@index([userId, createdAt])
  // The worker's claim query.
  @@index([status, nextAttemptAt])
  @@index([digestBatchId])
  @@unique([eventId, userId, channel])        // one delivery per recipient per channel (FR-R-04)
  @@map("notification_deliveries")
}

model NotificationPreference {
  userId    Int                 @map("user_id")
  typeKey   String              @map("type_key")
  emailEnabled Boolean          @default(true) @map("email_enabled")
  useDigest Boolean             @default(false) @map("use_digest")
  updatedAt DateTime            @updatedAt @map("updated_at")

  user      User                @relation(fields: [userId], references: [id], onDelete: Cascade)

  // No row = the catalogue default (FR-P-02). Rows exist only for deliberate choices.
  @@id([userId, typeKey])
  @@map("notification_preferences")
}

model NotificationTemplate {
  id          Int                 @id @default(autoincrement())
  typeKey     String              @map("type_key")
  channel     NotificationChannel

  subject     String?                                   // EMAIL only
  bodyHtml    String?             @map("body_html")
  bodyText    String?             @map("body_text")

  updatedById Int?                @map("updated_by_id")
  updatedAt   DateTime            @updatedAt @map("updated_at")

  // Overrides only. The shipped default lives in code, so "reset" is a delete (D-10, FR-M-02).
  @@unique([typeKey, channel])
  @@map("notification_templates")
}

model NotificationDigestBatch {
  id        Int       @id @default(autoincrement())
  userId    Int       @map("user_id")
  scheduledFor DateTime @map("scheduled_for")
  sentAt    DateTime? @map("sent_at")
  itemCount Int       @default(0) @map("item_count")

  @@index([userId, scheduledFor])
  @@map("notification_digest_batches")
}

model EmailSuppression {
  emailAddress String            @id @map("email_address")   // lower-cased
  reason       SuppressionReason
  detail       String?
  suppressedAt DateTime          @default(now()) @map("suppressed_at")
  releasedAt   DateTime?         @map("released_at")
  releasedById Int?              @map("released_by_id")

  @@map("email_suppressions")
}
```

## Design notes

**Why the catalogue is not a table.** Types are referenced by key in code (`notify("device.offline",
…)`), their recipient rules *are* code, and their context shapes are TypeScript types. A table would
duplicate all of that with no way to keep the copies honest. The same reasoning produced code-declared
permissions in 01 and settings in 03; by the third occurrence it is the house pattern.

**Why recipient resolution is deferred to the worker.** `notify()` runs inside someone else's
transaction (FR-D-02). Resolving "everyone with `device.read`, scoped to this employee" there would
put a permission query and a scope query on the critical path of an unrelated action (NFR-01). The
event row is written with `resolvedAt: null`; the worker resolves, writes deliveries, and stamps it.

**Why `recipientCount` can be zero and is stored.** A rule that resolves to nobody is a real
outcome and an invisible failure (FR-R-05). `null` means not yet resolved; `0` means resolved to
nobody. Collapsing those two states hides the exact problem the column exists to expose.

**Why `renderedBody` exists for in-app but not email.** The in-app list needs its text to render;
storing it means the list is one query and an old notification still reads correctly after its
template is edited. Email bodies are not stored (FR-N-06) — they are large, already delivered, and
potentially a second copy of something better not duplicated.

**Why the unique constraint is `[eventId, userId, channel]`.** It enforces FR-R-04 in the database
rather than trusting de-duplication in the resolver. A manager who is also an HR admin matches two
rules and gets one row.

**Why `claimedAt` rather than a queue library.** A single Postgres table with
`UPDATE … WHERE status = 'PENDING' … RETURNING` (or `FOR UPDATE SKIP LOCKED`) is atomic, needs no
new infrastructure, and is honest about what it is. Introducing Redis or a job library for this
volume would add an operational dependency to a system that currently has exactly two containers.

**Why suppression is keyed by address, not user.** Addresses move between people, and a bounce is a
property of the mailbox. When feature 02 fixes a typo'd address, the old one stays suppressed and the
new one is clean — which is the behaviour wanted.

## Volume

| Table | Rows/year, 200 employees | Notes |
|---|---|---|
| `notification_events` | ~20 000 | One per trigger |
| `notification_deliveries` | ~60 000+ | Fan-out; a single system-health event to 5 admins × 2 channels is 10 rows |
| `notification_preferences` | hundreds | Only deliberate choices |
| `notification_templates` | tens | Overrides only |
| `email_suppressions` | few | |

`notification_deliveries` is the one that grows, and its retention (OQ-507) matters more than its
indexes. Note the interaction with feature 04: `attendance.marked_absent` to both the employee and
their manager, daily, is ~2 rows per absence — which is fine, but it is exactly the type that must be
digest-eligible or it becomes the reason people stop reading notifications.

## Preference resolution

At send time, for (user, typeKey, channel):

1. If the type is **mandatory** → send. Preferences are not consulted (FR-T-04, D-07).
2. If channel is `IN_APP` → always create (D-03, FR-A-01).
3. Otherwise look for a `NotificationPreference` row:
   - none → use the catalogue default for that type;
   - `emailEnabled: false` → write the delivery as `SKIPPED` with reason `"muted by user"`;
   - `useDigest: true` and the type is digest-eligible → `HELD`, attached to the user's next batch.
4. Then check suppression, then quiet hours (FR-D-13), either of which can move a `PENDING` to
   `SKIPPED` or `HELD`.

Every branch writes a row. There is no path where a notification simply does not appear.

## Seed and migration notes

Migration `notifications`:

1. Create the six tables above.
2. **Drop feature 01's temporary password-reset outbox table** (01 api-design, `password/forgot`) and
   switch that flow to `notify("account.password_reset", …)`. Doing this in the same migration is
   deliberate: two outboxes in one system is how a reset email goes missing.
3. Seed permission keys and role grants from `requirements.md`.
4. Seed **no templates**. Defaults live in code; the table holds overrides only (D-10).
5. Settings added to 03's catalogue: `notification.fromAddress`, `notification.fromName`,
   `notification.replyTo`, `notification.baseUrl`, `notification.smtpHost`, `notification.smtpPort`,
   `notification.smtpUser`, `notification.smtpPassword` (secret — 03 FR-S-06),
   `notification.smtpSecure`, `notification.deliveryMode` (`SMTP` | `LOCAL_OUTBOX`),
   `notification.digestHour` (8), `notification.quietHoursStart` / `End`,
   `notification.failureAlertThreshold`.

`notification.deliveryMode` is an explicit setting rather than a `NODE_ENV` check (FR-D-08). A
production container started against a copy of the production database for testing would otherwise
email the entire company.

## Open questions

| ID | Question |
|---|---|
| OQ-501 | SMS/WhatsApp. `NotificationChannel` gains a value and `NotificationDelivery` needs a `phoneNumber`; the rest of the model is unchanged. Cheap to add, which is why it is not pre-built. |
| OQ-503 | With a provider that supports delivery webhooks (SES, Postmark), `NotificationDelivery` gains a provider message id and real delivered/bounced states instead of "handed to SMTP". Worth knowing before building bounce handling. |
| OQ-507 | Retention for `notification_deliveries`. Deleting the delivery but keeping the event would preserve "this was sent" while shedding most of the volume. |
| OQ-509 | Should `notification_events.context` be capped in size? It is written for every trigger and a careless caller could put a whole record in it. Proposed: a validated shape per type from the catalogue, which caps it implicitly. |
