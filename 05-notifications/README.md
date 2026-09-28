# 05 — Notifications

**Priority:** Must-have · **Build order:** 5 of 11 · **Status:** planned (Step 3)

## Purpose

One way for the system to tell a person something happened. Every feature needs it; none of them
should own it.

This feature arrives with a **backlog already defined**. Features 02, 03, and 04 each specified
notifications they would send, and each currently writes to a log instead. Building this feature
means those senders start working, so its first job is to satisfy a contract already written:

| Feature | Notification | Spec |
|---|---|---|
| 02 | Employee document expiring within 60 days | 02 FR-D-07 |
| 03 | Device went offline | 03 FR-D-09 |
| 03 | Fewer than 60 days of holidays configured ahead | 03 FR-W-12 |
| 04 | Unmatched PIN older than the threshold | 04 FR-U-04 |
| 04 | Unresolved device gap | 04 FR-G-05 |
| 04 | Correction request awaiting your approval | 04 api-design, scheduled jobs |
| 04 | Your correction request was approved / rejected | 04 FR-X-08 |
| 01 | Password reset and account invite | 01 FR-A-09 — currently writes to a local outbox |

Feature 01's invite and reset emails are the most urgent of these: today they are logged, which
means a real invite flow does not exist until this feature ships.

## Scope

**In scope**

- **Channels**: in-app and email. The model is channel-agnostic so SMS or push can be added, but
  neither is built (OQ-501).
- **A notification type catalogue**, declared in code, one entry per thing the system can tell you.
- **Templates** per type and channel, with variables, editable by an admin, resettable to the
  shipped default.
- **Recipient resolution**: who should be told, expressed as a rule (the employee's manager, anyone
  holding a permission), not as a list of addresses.
- **Per-user preferences**: which types reach me, on which channel, immediately or in a digest.
- **Delivery**: an outbox, a worker, retries with backoff, bounce handling, and a dev mode that does
  not send real mail.
- **In-app notification centre**: unread counts, read state, deep links to the thing that happened.
- **Digests**: a daily summary for types that would otherwise arrive one at a time.
- **An admin log** of what was sent, to whom, and what happened to it.

**Out of scope (owned elsewhere)**

- **Deciding when something is worth telling someone.** Each feature owns its own trigger: 04 knows
  a device gap is unresolved; this feature knows how to tell someone about it. The boundary is a
  single call — `notify(typeKey, context)` — and that call lives in the owning feature.
- SMTP credentials are *stored* in 03's settings registry (03 README, out-of-scope note) and *used*
  here.
- Permissions and roles → 01. Recipient rules refer to permissions; they do not define them.
- Employee contact details → 02. This feature reads work email and never stores its own copy.

## Files

| File | Contents |
|---|---|
| [requirements.md](./requirements.md) | User stories, functional and non-functional requirements, the notification catalogue, permission keys |
| [data-model.md](./data-model.md) | Prisma models, the outbox, preference resolution, seed and migration notes |
| [api-design.md](./api-design.md) | The `notify()` contract, notification centre endpoints, templates, delivery worker |
| [ui-ux.md](./ui-ux.md) | Notification centre, preferences, template editor, delivery log |

## Key decisions taken in this plan

| # | Decision | Rationale |
|---|---|---|
| D-01 | Notification types are a **code-declared catalogue** — key, category, default channels, default template, recipient rule, whether it is mandatory. | The third time this pattern appears (permissions in 01, settings in 03). Same reasoning: a type nothing sends is dead weight, and a typo in a key should fail at seed time rather than silently notify nobody. |
| D-02 | **Nothing is sent inline with a request.** Every notification is written to an outbox and delivered by a worker. | An SMTP server that is slow or down must never slow down or fail the action that triggered it. Approving leave must not depend on a mail server. |
| D-03 | **In-app is always created; email is optional.** A user may silence email for a type but the in-app record still exists. | The in-app record is the system's memory of having told you. Letting it be disabled means "I was never informed" becomes unanswerable. |
| D-04 | **Recipients are resolved by rule, not by stored address lists.** Rules are expressed in terms of relationships (the employee's manager) and permissions (anyone with `device.read`). | A hardcoded list goes stale the day someone changes job. Permission-based rules mean access changes and notification changes stay in step automatically. |
| D-05 | Every notification carries an **idempotency key**. Sending the same key twice is a no-op. | Retries, recomputes, and re-runs of a scheduled job will all happen. Without this, a device that is offline for a week sends 168 identical emails. |
| D-06 | **No sensitive data in email bodies.** Emails say what happened and link to the system; they do not contain salary figures, national IDs, or document contents. **One documented exception, 2026-09-18 (OQ-709): payslip PDFs are emailed as attachments** — password-protected, with the email body itself still carrying no figures, and delivery audited as a disclosure (07 FR-L-05). | Email is forwarded, archived on third-party servers, and read on unlocked phones. The rule stands everywhere else; the exception is a deliberate, recorded trade rather than an erosion of it, and the template denylist still blocks monetary variables in **body text**. |
| D-07 | **Mandatory types cannot be silenced.** A small set — password reset, account invite, an approval waiting on you — ignore preferences. | A user who has muted "approval required" becomes a silent bottleneck for everyone else. |
| D-08 | **Digest by default for informational types**, immediate for actionable ones. | Twelve separate emails about expiring documents trains people to ignore the sender entirely. |
| D-09 | A **failed delivery is visible**, not silent. Bounces suppress future email to that address and surface as an admin task. | Otherwise the system believes it told someone for months after their mailbox stopped existing. |
| D-10 | Templates are **editable with a shipped default and a reset**. Edits do not change the variables available. | HR will want the company's wording. Letting them edit the variable set means a template that renders `{{undefined}}` in production. |

## The one call other features make

The entire integration surface is one function. Everything else in this feature exists to serve it:

```ts
notify("attendance.correction_pending", {
  employeeId: 12,
  attendanceDayId: 90124,
  date: "2026-09-15",
  idempotencyKey: `correction:${correctionId}:pending`,
});
```

The caller does not choose recipients, channels, wording, or timing. It states what happened, in the
vocabulary of the catalogue, and provides the context the template needs. That constraint is what
keeps eleven features from each inventing their own notion of who should be emailed.

## Dependencies

- **Depends on:** 01 (users, permissions, audit — recipient rules are permission-based), 02
  (employees, manager relationships, work email), 03 (settings for SMTP, the job runner, company
  timezone for quiet hours and digest timing).
- **Depended on by:** 06, 07, 09, 10 for their own notifications, and retroactively by 01–04 for the
  backlog above.
- **Touches existing code:** 01's password-reset outbox is replaced by this feature's outbox. That
  replacement should delete the temporary table rather than leave two.

## Open questions

| ID | Question | Why it matters | Proposed default |
|---|---|---|---|
| OQ-501 | Is SMS or WhatsApp needed for anything — shift changes, emergency notices? Factory staff often have no work email. | Changes the channel model from "email plus in-app" to something with a per-channel provider and cost. The model allows it; building it is real work, and for employees with no email, in-app may also be unreachable. | Email and in-app only in v1. |
| OQ-502 | **Do all employees have email addresses?** If a large share do not, the entire notification design needs rethinking for them. | This is the question that most affects whether this feature is useful. An HRM where half the workforce cannot be notified needs a different answer — printed notices, supervisor relay, or SMS. | Assume office staff have email; ask before building. |
| OQ-503 | What SMTP service will be used — a company mail server, or a provider (SES, SendGrid, Postmark)? | Affects bounce handling, rate limits, and whether webhooks for delivery status are available. | Generic SMTP, with bounce handling limited to what SMTP reports. |
| OQ-504 | What "from" address and display name should mail use, and is there a reply-to that reaches a human? | A no-reply address for approval requests guarantees someone replies to it anyway and is never heard. | `hr@company` as reply-to, not no-reply. |
| OQ-505 | Should digests be daily or weekly, and at what time? | Interacts with the company timezone and working hours. | Daily at 08:00 company time. |
| OQ-506 | Are notifications needed in a second language (01 OQ-115)? Templates are the most language-dependent surface in the system. | Retrofitting i18n into templates is far cheaper than into the whole UI, but it is not free. | English only, pending OQ-115. |
| OQ-507 | Retention for delivered notifications and the delivery log. | The in-app notification table grows with every event for every recipient. | In-app kept 12 months; delivery log 90 days. |
| OQ-508 | Should managers be able to see which of their team have unread notifications (e.g. "has she seen the schedule change")? | Useful operationally, uncomfortable as surveillance. | No. |
