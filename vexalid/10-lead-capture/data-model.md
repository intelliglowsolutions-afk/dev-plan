# 10 — Lead Capture — Data Model

Database: `vexalid_web` on the VPS's Postgres, with a dedicated role holding **no grants on the HRM
database** (segment 01 D-107). This isolation is not negotiable — a public marketing site must not be
a path to employee and payroll data.

## Entity overview

```
ServicesEnquiry ──┐
DemoRequest ──────┼──→ EmailLog   (every outbound message)
NewsletterSub ────┤
GatedDownload ────┘

RateLimit         (standalone, per IP + form + window)
```

No relations between lead tables. They are independent records of independent events; joining them
would imply an identity graph nobody asked for.

## Enums

```prisma
enum CompanySize        { SIZE_1_10  SIZE_11_50  SIZE_51_200  SIZE_201_500  SIZE_500_PLUS }
enum LeadStatus         { NEW  CONTACTED  QUALIFIED  CLOSED_WON  CLOSED_LOST  SPAM }
enum ServiceInterest    { CSV  AI_AUTOMATION  CUSTOM_SOFTWARE  NOT_SURE }
enum EngagementStage    { EXPLORING  SCOPING  READY_TO_START  URGENT }
enum ProductInterest    { HRM  STRATUMONE  BOTH }
enum SubscriptionStatus { PENDING  CONFIRMED  UNSUBSCRIBED  BOUNCED }
enum EmailStatus        { QUEUED  SENT  FAILED  BOUNCED }
enum EmailKind          { ENQUIRY_NOTIFICATION  ENQUIRY_AUTOREPLY
                          DEMO_NOTIFICATION     DEMO_AUTOREPLY
                          NEWSLETTER_CONFIRM    DOWNLOAD_LINK }
```

## Models

### `DemoRequest`

| Field | Type | Notes |
|---|---|---|
| `id` | `String @id @default(cuid())` | |
| `name` | `String` | |
| `email` | `String` | Not unique — the same person may request twice, legitimately |
| `company` | `String` | |
| `product` | `ProductInterest` | HRM, StratumOne, or both — without this the follow-up is guesswork |
| `companySize` | `CompanySize` | |
| `message` | `String?` | `@db.Text` |
| `phone` | `String?` | Reserved (OQ-1004) |
| `status` | `LeadStatus @default(NEW)` | Manual triage until a CRM exists |
| `internalNote` | `String?` | `@db.Text` |
| `sourcePath` | `String?` | Which page the form was submitted from |
| `referrer` | `String?` | |
| `utmSource` `utmMedium` `utmCampaign` `utmTerm` `utmContent` | `String?` | |
| `ipAddress` | `String?` | For abuse investigation and consent evidence — **purged at 90 days** |
| `userAgent` | `String?` | Purged at 90 days |
| `privacyNoticeVersion` | `String` | Which privacy notice was shown |
| `externalRef` | `String?` | Reserved for CRM id (D-1013) |
| `syncedAt` | `DateTime?` | Reserved |
| `createdAt` | `DateTime @default(now())` | |
| `updatedAt` | `DateTime @updatedAt` | |
| `contactedAt` | `DateTime?` | Starts the 24-month retention clock |

Indexes: `@@index([createdAt])`, `@@index([status])`, `@@index([email])`.

`ipAddress` and `userAgent` are separated from the rest by retention period on purpose: they are
needed for a fortnight of abuse investigation, not for two years of sales history. Keeping them the
full 24 months would be storing personal data with no remaining purpose.

### `ServicesEnquiry` (D-1014)

| Field | Type | Notes |
|---|---|---|
| `id` | `String @id @default(cuid())` | |
| `name` | `String` | |
| `email` | `String` | Not unique |
| `organisation` | `String` | "Organisation", not "company" — some regulated sites are institutes or authorities |
| `interest` | `ServiceInterest` | Includes `NOT_SURE`, deliberately |
| `stage` | `EngagementStage` | The routing field (D-1015) |
| `message` | `String?` | `@db.Text`, up to 2000 chars |
| `status` | `LeadStatus @default(NEW)` | |
| `internalNote` | `String?` | `@db.Text` |
| `assignedTo` | `String?` | Which consultant owns it (OQ-1002) |
| `sourcePath` | `String?` | An enquiry originating on the CSV page is a different lead from one off the home page |
| `referrer` | `String?` | |
| `utmSource` `utmMedium` `utmCampaign` `utmTerm` `utmContent` | `String?` | |
| `ipAddress` `userAgent` | `String?` | **Purged at 90 days** |
| `privacyNoticeVersion` | `String` | |
| `externalRef` `syncedAt` | — | Reserved for CRM (D-1013) |
| `createdAt` `updatedAt` | — | |
| `contactedAt` | `DateTime?` | Starts the 24-month retention clock |

Indexes: `@@index([createdAt])`, `@@index([status])`, `@@index([interest, stage])`,
`@@index([email])`.

The `[interest, stage]` composite is what makes *"everyone ready to start on a CSV engagement"* a
single fast query — the question this table exists to answer.

**GDPR data requests have no table.** They arrive by email to a published address
(requirements.md). A subject access request must not be able to fail silently inside a form
pipeline, and the 30-day statutory clock starts when the email arrives.

### `NewsletterSubscriber`

| Field | Type | Notes |
|---|---|---|
| `id` | `String @id @default(cuid())` | |
| `email` | `String @unique` | Stored lowercased and trimmed |
| `status` | `SubscriptionStatus @default(PENDING)` | |
| `confirmToken` | `String? @unique` | **SHA-256 hash of the token**, never the token itself |
| `confirmTokenExpiresAt` | `DateTime?` | 30 days |
| `confirmedAt` | `DateTime?` | |
| `unsubscribedAt` | `DateTime?` | |
| `consentText` | `String @db.Text` | The exact wording shown at signup |
| `consentIp` | `String?` | Consent evidence — retained while subscribed |
| `consentAt` | `DateTime` | |
| `source` | `String?` | `footer`, `blog-post`, `guide:<slug>`, `resources-hub` |
| `listmonkId` | `Int?` | Set once pushed to Listmonk |
| `lastConfirmSentAt` | `DateTime?` | Enforces the resend limit |
| `createdAt` `updatedAt` | — | |

Indexes: `@@index([status])`, `@@index([confirmTokenExpiresAt])`.

Tokens are stored hashed for the same reason passwords are: a database read must not yield working
confirmation links. `consentText` is stored verbatim because "we had consent" is only defensible if
you can produce what was agreed to (D-1005).

`PENDING` rows older than 7 days are deleted outright (D-1010) — an unconfirmed address is not a
lawful marketing contact, so there is nothing to keep.

### `GatedDownload`

| Field | Type | Notes |
|---|---|---|
| `id` | `String @id @default(cuid())` | |
| `firstName` `email` | `String` | |
| `guideSlug` | `String` | |
| `downloadToken` | `String @unique` | Hashed |
| `tokenExpiresAt` | `DateTime` | 7 days |
| `downloadedAt` | `DateTime?` | Single-use marker (D-1007) |
| `downloadCount` | `Int @default(0)` | Attempts after the first are logged, not served |
| `newsletterOptIn` | `Boolean @default(false)` | Separate, optional consent |
| `sourcePath` `ipAddress` | `String?` | IP purged at 90 days |
| `createdAt` | `DateTime @default(now())` | |

Indexes: `@@index([email])`, `@@index([guideSlug, createdAt])`, `@@index([tokenExpiresAt])`.

### `EmailLog` (D-1009)

| Field | Type | Notes |
|---|---|---|
| `id` | `String @id @default(cuid())` | |
| `kind` | `EmailKind` | |
| `to` | `String` | |
| `subject` | `String` | |
| `status` | `EmailStatus @default(QUEUED)` | |
| `providerId` | `String?` | Provider's message id, for support tickets |
| `error` | `String? @db.Text` | |
| `relatedType` | `String?` | `DemoRequest`, `GatedDownload`, … |
| `relatedId` | `String?` | Soft reference — an email log must survive its lead's deletion |
| `sentAt` | `DateTime?` | |
| `createdAt` | `DateTime @default(now())` | |

Indexes: `@@index([status])`, `@@index([to])`, `@@index([relatedType, relatedId])`,
`@@index([createdAt])`.

**Never stores the email body.** The subject line and status are enough to answer "was it sent?"
without duplicating personal data into a second table with a different retention period.

### `RateLimit` (D-1004)

| Field | Type | Notes |
|---|---|---|
| `id` | `String @id @default(cuid())` | |
| `key` | `String` | `<sha256(ip)>:<formName>` — the IP is hashed, so the table is not an IP log |
| `windowStart` | `DateTime` | Hour-truncated |
| `count` | `Int @default(1)` | |

`@@unique([key, windowStart])`, `@@index([windowStart])`.

Incremented with a single `upsert`; over-limit is a read of `count`. Rows older than 24 hours are
deleted nightly.

At tens of submissions a day this is comfortably fast. If the site ever sees real traffic spikes, the
same interface moves to Redis — the abstraction in `lib/rate-limit.ts` is why that stays a one-file
change.

## Expected volume (year one)

| Table | Rows/month | Year-one total |
|---|---|---|
| `DemoRequest` | 20–100 | ~1k |
| `ServicesEnquiry` | 10–60 | ~600 |
| `NewsletterSubscriber` | 30–200 | ~2k |
| `GatedDownload` | 30–150 | ~1.5k |
| `EmailLog` | 100–500 | ~5k |
| `RateLimit` | transient | < 1k live |

Total well under 100 MB. Postgres is chosen for correctness, backups and the migration path, not for
scale.

## Retention jobs (D-1010, T-1013)

Nightly, via a `node` script invoked by a systemd timer or cron (segment 12):

| Job | Rule |
|---|---|
| Purge unconfirmed subscribers | `status = PENDING AND createdAt < now() - 7 days` → delete |
| Purge lead IP/UA | `createdAt < now() - 90 days` → null `ipAddress`, `userAgent` |
| Purge old leads | `DemoRequest`/`ServicesEnquiry` where `coalesce(contactedAt, createdAt) < now() - 24 months` → delete |
| Purge expired download rows | `tokenExpiresAt < now() - 24 months` → delete |
| Trim email logs | `createdAt < now() - 12 months` → delete |
| Clear rate-limit rows | `windowStart < now() - 24 hours` → delete |

Each job logs a count. A retention job that silently stops running is the compliance failure nobody
notices until an audit, so the counts feed the uptime check in segment 12.

## Migration notes

- `prisma migrate deploy` runs as an **explicit deploy step**, never automatically on container
  start — two containers starting simultaneously must not race on migrations.
- Migrations are committed and reviewed.
- Backups: nightly `pg_dump` of `vexalid_web`, retained 30 days, **restore-tested before launch**.
  An untested backup is a hypothesis, not a backup.
- The initial migration creates every table above at once; there is no data to preserve.
