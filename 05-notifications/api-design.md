# 05 — Notifications — API Design

Conventions as in [01 api-design.md](../01-roles-permissions-auth/api-design.md).

## The contract other features use

This is the feature's real API. The HTTP endpoints below serve the UI; **this** serves the other ten
features, and it is deliberately tiny.

```ts
// src/lib/notifications/notify.ts
notify<K extends NotificationTypeKey>(
  typeKey: K,
  input: {
    context: NotificationContext<K>;   // typed by the catalogue (FR-T-02)
    idempotencyKey: string;            // required (D-05)
    subjectEmployeeId?: number;
    entity?: { type: string; id: string };
    linkPath?: string;                 // relative; absolute URL built at render
  },
  tx?: PrismaTransaction,              // pass the caller's transaction (FR-D-02)
): Promise<void>
```

One insert into `notification_events`, and return (NFR-01). No recipient resolution, no rendering,
no sending. A duplicate idempotency key is a silent no-op, not an error — callers retry, and a retry
should not have to catch anything.

```ts
// Marking a notification irrelevant once someone else has handled it (FR-A-05).
supersede(entity: { type: string; id: string }, typeKeys?: NotificationTypeKey[]): Promise<void>
```

Called by feature 06 when a leave request is approved, so the other approver's "waiting on you"
notification stops counting as unread. Without it, the notification list fills with items that are
already done, and people stop reading it.

### The catalogue entry a feature adds

```ts
"device.offline": {
  module: "03",
  category: "SYSTEM_HEALTH",
  label: "A device went offline",
  description: "An attendance terminal stopped reporting.",
  mandatory: false,
  digestEligible: true,
  defaultChannels: ["IN_APP", "EMAIL"],
  recipients: permissionHolders("device.read"),
  context: z.object({ deviceName: z.string(), deviceId: z.number(),
                      lastSeenAt: z.string(), offlineHours: z.number() }),
  templates: { email: { subject: "…", html: "…", text: "…" }, inApp: { title: "…", body: "…" } },
}
```

Recipient rules are composable helpers (FR-R-01):

```ts
subjectEmployeeUser()            // the employee the event is about
subjectEmployeeManager()         // their line manager (02)
permissionHolders(key, scope?)   // everyone holding it, optionally scoped to the subject
namedUser(userId)
superAdmins()
union(...rules)                  // de-duplicated (FR-R-04)
```

`permissionHolders` re-checks visibility at send time (FR-R-03): a resolved recipient who cannot see
the subject employee is dropped. That check is what keeps a broadly-granted notification from
becoming an information leak.

## Notification centre — the user's own

| Method | Path | Permission | Notes |
|---|---|---|---|
| GET | `/api/notifications` | `notification.read` | Own only. `?category=&unread=&page=` |
| GET | `/api/notifications/unread-count` | `notification.read` | The badge (NFR-02) |
| POST | `/api/notifications/:id/read` | `notification.read` | |
| POST | `/api/notifications/:id/unread` | `notification.read` | |
| POST | `/api/notifications/read-all` | `notification.read` | Optional `?category=` |

"Own only" is enforced from the session, never from a parameter — there is no `userId` in any of
these requests.

```json
{ "data": [ { "id": 4412, "typeKey": "attendance.correction_pending",
              "category": "APPROVALS", "title": "Correction request from Ayesha Khan",
              "body": "15 September — asking to add a 17:05 sign-out.",
              "linkUrl": "/attendance/corrections?id=88",
              "createdAt": "2026-09-15T09:40:00Z", "readAt": null,
              "isSuperseded": false } ],
  "unreadCount": 3, "page": 1, "pageSize": 20, "total": 57 }
```

`unread-count` returns `{ "count": 3 }` and nothing else. It is requested on every page load by every
user, so it is an indexed count over `[userId, channel, readAt]` and is cached briefly per request.

## Preferences

| Method | Path | Permission |
|---|---|---|
| GET | `/api/notifications/preferences` | `notification.read` |
| PUT | `/api/notifications/preferences` | `notification.read` |
| GET | `/api/admin/notification-defaults` | `notification.admin` |
| PUT | `/api/admin/notification-defaults` | `notification.admin` |

`GET` returns the catalogue joined with the user's choices, grouped by category — so the screen is
generated from the catalogue and a new type appears without UI work:

```json
{ "categories": [
    { "key": "APPROVALS", "label": "Approvals",
      "types": [ { "typeKey": "leave.request_pending",
                   "label": "A request needs your approval",
                   "description": "Sent when someone's leave request is waiting on you.",
                   "mandatory": true,
                   "mandatoryReason": "You cannot turn this off — other people are waiting on you.",
                   "digestEligible": false,
                   "emailEnabled": true, "useDigest": false, "isDefault": true } ] } ] }
```

Types the user cannot receive — their permissions do not match any recipient rule — are omitted
entirely rather than shown greyed out (FR-P-06 keeps their stored rows for later).

`PUT` takes only changed entries. Attempting to disable a mandatory type → **422** `MANDATORY_TYPE`,
with the reason text, so a stale client cannot silence it either (acceptance criterion 7).

## Templates

| Method | Path | Permission | Notes |
|---|---|---|---|
| GET | `/api/admin/notification-templates` | `notification.template.read` | Catalogue + overrides |
| GET | `/api/admin/notification-templates/:typeKey` | `notification.template.read` | Both channels, default and override |
| PUT | `/api/admin/notification-templates/:typeKey/:channel` | `notification.template.write` | Save an override |
| DELETE | `/api/admin/notification-templates/:typeKey/:channel` | `notification.template.write` | Reset to default |
| POST | `/api/admin/notification-templates/:typeKey/preview` | `notification.template.read` | Render with sample data |
| POST | `/api/admin/notification-templates/:typeKey/test-send` | `notification.template.write` | Send to yourself (FR-N-04) |

`PUT` validates before storing (FR-M-03): every `{{variable}}` must exist in the type's declared
context, and no variable may be on the sensitive denylist (FR-D-11).

```json
{ "error": { "code": "UNKNOWN_VARIABLE",
             "message": "This template uses variables that do not exist for this notification.",
             "unknown": ["employee.salary"],
             "available": ["employee.name", "date", "linkUrl"] } }
```

Listing the available variables in the error is the difference between a usable editor and a
guessing game.

`preview` renders both HTML and text with sample data and returns them for side-by-side display. It
never sends.

## Delivery log and health

| Method | Path | Permission |
|---|---|---|
| GET | `/api/admin/notification-log` | `notification.log.read` |
| GET | `/api/admin/notification-log/:id` | `notification.log.read` |
| POST | `/api/admin/notification-log/:id/retry` | `notification.admin` |
| GET | `/api/admin/notification-health` | `notification.log.read` |
| GET | `/api/admin/email-suppressions` | `notification.log.read` |
| DELETE | `/api/admin/email-suppressions/:address` | `notification.admin` |

`notification-health` is what the admin dashboard and any monitoring watches:

```json
{ "queueDepth": 4, "oldestPendingAgeSeconds": 12, "held": 31,
  "sentLast24h": 412, "failedLast24h": 2, "skippedLast24h": 18,
  "suppressedAddresses": 1, "lastSuccessfulSendAt": "2026-09-15T09:41:22Z",
  "workerLastRunAt": "2026-09-15T09:41:20Z", "deliveryMode": "SMTP" }
```

`workerLastRunAt` matters as much as the queue depth: a queue of zero with a worker that last ran
yesterday means nothing is being sent and nothing is being queued either — the same two failure
modes feature 04's recompute status endpoint watches for.

`skippedLast24h` is deliberately on the front page. A high skip count usually means a suppression or
a muted type is quietly swallowing something important.

## The worker

Runs on 03's job runner every 30 seconds (FR-D-04).

**Pass 1 — resolve.** For each event with `resolvedAt: null`:

1. Evaluate the recipient rule → user ids.
2. Drop users who are not `ACTIVE`, except account-lifecycle types (FR-R-06).
3. Drop users who cannot see the subject (FR-R-03).
4. De-duplicate (FR-R-04).
5. For each remaining user × channel, apply preference resolution (`data-model.md`) and insert a
   delivery row in the appropriate state — `PENDING`, `HELD`, or `SKIPPED` with a reason.
6. Stamp `resolvedAt` and `recipientCount`, which may be `0` (FR-R-05).

All inserts for one event are one batch (NFR-04).

**Pass 2 — deliver.** Claim a batch:

```sql
UPDATE notification_deliveries SET status = 'SENDING', claimed_at = now()
WHERE id IN (
  SELECT id FROM notification_deliveries
  WHERE status = 'PENDING' AND next_attempt_at <= now()
  ORDER BY created_at LIMIT 50 FOR UPDATE SKIP LOCKED
) RETURNING *;
```

`FOR UPDATE SKIP LOCKED` is what makes two workers safe (NFR-06). Claims older than a few minutes are
returned to `PENDING` by the same job, so a worker that dies mid-batch does not strand its rows.

Then per row: render (falling back to the shipped default on error, and raising an alert — FR-M-06),
check suppression, send, and record. Failures set `attempts`, `lastError`, and `nextAttemptAt` from
the backoff schedule (FR-D-05); permanent failures skip straight to `FAILED`.

**Pass 3 — release held.** Items held for quiet hours become `PENDING` when the quiet window ends.

**Pass 4 — digests.** At the configured hour per user, gather their `HELD` digest items into a batch,
render one message, send it, and mark them `SENT`. A batch with no items sends nothing and writes no
row (FR-D-15).

### Delivery mode

`notification.deliveryMode` decides where mail goes:

- `SMTP` — the configured server.
- `LOCAL_OUTBOX` — written to a table and viewable at `/admin/notification-log` with a *View
  rendered email* action; nothing leaves the machine (FR-D-08).

Set explicitly, never inferred from `NODE_ENV`. A container started against a copy of production
data with the wrong mode set is how an entire company receives test emails.

## Scheduled jobs

| Job | Frequency | Does |
|---|---|---|
| Notification worker | 30 s | The four passes above |
| Digest sender | hourly | Sends digests for users whose configured hour has arrived |
| Stale claim release | 5 min | Returns `SENDING` rows claimed too long ago to `PENDING` |
| Failure-rate alert | 15 min | Raises `notification.delivery_failing` in-app to super admins (FR-N-05) |
| ~~Retention sweep~~ | — | **Removed 2026-09-18 (OQ-1002): no automatic deletion.** In-app deliveries and log rows are retained. `notification_deliveries` is the fastest-growing table here, so its size becomes an operational concern rather than a self-solving one — see OQ-507 |

The failure-rate alert is in-app only, by design: when email delivery is what is broken, an email
about it is the one message guaranteed not to arrive.

## Open questions

| ID | Question |
|---|---|
| OQ-503 | A provider with webhooks (SES/Postmark) would add `POST /api/webhooks/email` for delivery and bounce events, with signature verification. Worth deciding before bounce handling is built, since SMTP alone reports much less. |
| OQ-510 | Should `notify()` be callable outside a transaction at all? Requiring one would guarantee FR-D-02 by construction, but makes scheduled jobs (which have no natural transaction) awkward. Proposed: optional, documented. |
| OQ-511 | Should the in-app list support snooze ("remind me tomorrow")? Common request once people live in the notification centre; a small model addition now, a bigger one later. |
| OQ-512 | Whether `test-send` should be permitted for every type. Sending yourself a sample `account.suspended` is harmless; a sample payroll notification with real-looking figures may not be. |
