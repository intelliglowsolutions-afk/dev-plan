# 05 — Notifications — Requirements

## Actors

| Actor | What they do here |
|---|---|
| Any signed-in user | Receives notifications, reads them, sets their own preferences. |
| HR admin | Edits templates, watches the delivery log, resends a failed notification. |
| Super admin | SMTP configuration (via 03's settings), suppression list, mandatory-type policy. |
| Other features | Call `notify()`. They are the highest-volume "actor" in the feature and the one whose needs shape it. |
| The worker | Drains the outbox. Like feature 04's engine, its behaviour is specified as carefully as any user's. |

## User stories

### Receiving

- **US-01** — As a user, I see a badge when something needs my attention, and a list of what it is.
- **US-02** — As a user, I click a notification and land on the thing it is about, not on a
  dashboard.
- **US-03** — As a user, I mark notifications read, individually or all at once, and the badge
  clears.
- **US-04** — As a manager, I am emailed when something waits on my approval, so I do not have to
  remember to check.
- **US-05** — As an employee, I am told when my request was approved or rejected, and why.
- **US-06** — As a user, I get one daily summary of routine things rather than a stream of separate
  emails.

### Controlling

- **US-07** — As a user, I choose which notifications reach me by email, without losing the in-app
  record.
- **US-08** — As a user, I understand from the preferences screen what each type actually is, in
  terms of events I recognise.
- **US-09** — As a user, I cannot accidentally silence something I am required to act on, and the
  screen explains why.

### Administering

- **US-10** — As HR, I edit the wording of a notification to match how we speak, and preview it with
  real sample data.
- **US-11** — As HR, I reset a template to the shipped default when my edit turns out worse.
- **US-12** — As HR, I see what was sent, to whom, and whether it arrived.
- **US-13** — As HR, I see when email is failing — a wrong SMTP password, a bouncing address — and
  am told, rather than discovering it weeks later.
- **US-14** — As HR, I send a test notification to myself to check the configuration works before
  relying on it.

## Functional requirements

### The catalogue — FR-T

| ID | Requirement |
|---|---|
| FR-T-01 | Each notification type is declared in code with: key, owning module, category, human label and description, default channels, mandatory flag, digest eligibility, default templates, recipient rule, and the context variables its templates may use (D-01). |
| FR-T-02 | `notify(typeKey, context)` is typed against the catalogue: an unknown key, or a context missing a required variable, is a compile-time error. |
| FR-T-03 | Categories group types for the preferences UI: *Account*, *Approvals*, *Attendance*, *Leave*, *Payroll*, *System health*. Categories are how users reason about this; individual types are how the system does. |
| FR-T-04 | A type marked mandatory ignores user preferences on all channels (D-07). The catalogue is where that is decided, not the UI. |
| FR-T-05 | Adding a type requires no migration. Removing one leaves orphaned preference rows, which are ignored and cleaned by a maintenance task rather than cascading deletes. |

### Recipient resolution — FR-R

| ID | Requirement |
|---|---|
| FR-R-01 | A recipient rule is one of: a named user, the subject employee's linked user, the subject employee's manager, the holders of a permission (optionally narrowed by scope over the subject employee), or a combination. |
| FR-R-02 | Rules resolve at send time, not at trigger time, so a role change between the two takes effect. |
| FR-R-03 | Resolution never produces a recipient who lacks permission to see the thing being notified about. A notification whose deep link the recipient cannot open must not be sent — this is the rule that stops "Sara's salary was updated" reaching someone without payroll access. |
| FR-R-04 | Resolution de-duplicates: someone who is both the manager and an HR admin receives one notification, not two. |
| FR-R-05 | A rule resolving to nobody is recorded as such and surfaced in the admin log. Silence is a legitimate outcome and an invisible failure mode; it must be distinguishable from "not attempted". |
| FR-R-06 | Users with status other than `ACTIVE` are excluded from resolution, except for account-lifecycle types (invite, reset) which are precisely for users who are not yet active. |
| FR-R-07 | An employee with no linked user account cannot receive in-app notifications. Where a work email exists, email is still attempted; where neither exists, the notification is recorded as undeliverable and counted, so the gap is visible (OQ-502). |

### Delivery — FR-D

| ID | Requirement |
|---|---|
| FR-D-01 | `notify()` writes to the outbox and returns. It never sends inline and never fails the caller's transaction for a delivery reason (D-02). |
| FR-D-02 | `notify()` participates in the caller's transaction where one exists: if the action rolls back, the notification is not queued. Nobody should be told about a leave approval that did not happen. |
| FR-D-03 | Every notification carries an idempotency key supplied by the caller. A second write with the same key is ignored (D-05). |
| FR-D-04 | The worker drains the outbox on a schedule (default every 30 seconds), oldest first, in batches. |
| FR-D-05 | Delivery failures retry with exponential backoff — 1 min, 5, 30, 2 h, 12 h — then stop and are marked failed. Permanent failures (invalid address, rejected recipient) do not retry at all. |
| FR-D-06 | A hard bounce adds the address to a suppression list. Suppressed addresses are skipped, and the skip is recorded rather than silently dropped (D-09). |
| FR-D-07 | Suppression is per address and reversible by an admin, since the most common cause is a typo that gets fixed in feature 02. |
| FR-D-08 | In dev and test, email delivery writes to a local outbox viewer and never contacts an SMTP server. The mode is explicit configuration, never inferred solely from `NODE_ENV`, so a misconfigured production deployment cannot quietly email real staff from a test database. |
| FR-D-09 | Emails are sent as HTML with a plain-text alternative. Both come from the same template source. |
| FR-D-10 | Every email includes: what happened, who it concerns, a link into the system, and — for non-mandatory types — how to change these preferences. |
| FR-D-11 | Email bodies never contain sensitive data (D-06). Templates are validated against a denylist of variables at seed time, so a future template cannot introduce a salary figure into an email by accident. |
| FR-D-12 | Deep links are absolute URLs built from a configured base URL, and they survive being opened by a signed-out user: the login redirect returns them to the link's destination. |
| FR-D-13 | Quiet hours: non-urgent email is held until the next working hour in the company timezone. Mandatory and approval types ignore quiet hours. |
| FR-D-14 | Digest-eligible notifications accumulate and are sent as one message at the configured time (D-08, OQ-505). A user's digest contains only types they have not set to immediate. |
| FR-D-15 | A digest with nothing in it is not sent. An empty daily email is how people learn to filter the sender. |
| FR-D-16 | Sending is rate-limited to stay within the provider's limits, with the queue draining steadily rather than in bursts. |

### In-app — FR-A

| ID | Requirement |
|---|---|
| FR-A-01 | An in-app notification is created for every resolved recipient with a user account, regardless of email preference (D-03). |
| FR-A-02 | Each carries a title, a short body, a category, a deep link, a created timestamp, and a read timestamp. |
| FR-A-03 | The unread count is available cheaply — it appears on every page — and is not a scan over the notification table. |
| FR-A-04 | Reading a notification marks it read; the user can also mark all read, and mark an individual one unread again. |
| FR-A-05 | Notifications that have become irrelevant — an approval someone else handled — are **superseded**: the record remains, marked resolved, and no longer contributes to the unread count. A queue full of stale "approve this" items is how the feature stops being read. |
| FR-A-06 | The list is filterable by category and read state, and paginated. |
| FR-A-07 | In-app notifications older than the retention period are deleted by a maintenance job (OQ-507). |

### Templates — FR-M

| ID | Requirement |
|---|---|
| FR-M-01 | Each type has a shipped default template per channel, defined in code. |
| FR-M-02 | An admin may override a template. Overrides are stored; the default remains available for reset (D-10). |
| FR-M-03 | Templates use named variables from the type's declared context only. An unknown variable fails validation at save time, not at send time. |
| FR-M-04 | All variable interpolation is escaped for its channel. HTML email escapes HTML; a name containing `<` must not break the message or enable injection. |
| FR-M-05 | Preview renders a template with sample data for that type, in both HTML and plain text. |
| FR-M-06 | A template failing to render at send time does not lose the notification: it falls back to the shipped default, delivers, and raises an admin alert naming the broken template. |
| FR-M-07 | Template edits are audited with before and after. |

### Preferences — FR-P

| ID | Requirement |
|---|---|
| FR-P-01 | A user sets, per type: email on or off, and immediate or digest where the type allows digest. |
| FR-P-02 | Defaults come from the catalogue. A user with no stored preference gets the default; no row is written until they change something. |
| FR-P-03 | Mandatory types render as fixed, with a short explanation of why (FR-T-04, D-07). |
| FR-P-04 | Preferences are grouped by category and described in plain language (US-08). |
| FR-P-05 | An admin may set organisation-wide defaults per type, which apply to users who have not chosen. |
| FR-P-06 | A user's preferences survive role changes. Types they can no longer receive are hidden rather than deleted, so access restored later restores their choices. |

### Administration — FR-N

| ID | Requirement |
|---|---|
| FR-N-01 | The delivery log lists every notification: type, recipient, channel, status, attempts, last error, timestamps. |
| FR-N-02 | Filterable by type, recipient, channel, status, and date. |
| FR-N-03 | A failed notification can be retried manually. |
| FR-N-04 | A test notification of any type can be sent to the acting admin, rendered with sample data (US-14). |
| FR-N-05 | When the failure rate over a rolling window exceeds a threshold, an alert is raised — in-app to super admins, since email is precisely what may be broken (D-09, US-13). |
| FR-N-06 | The log records the rendered subject but not the full body, to keep it small and to avoid a second copy of anything sensitive. |
| FR-N-07 | Health is visible: queue depth, oldest unsent, failures in the last 24 hours, suppressed addresses, last successful send. |

## The notification catalogue (initial)

Category · key · recipient rule · channels · mandatory · digest.

**Account** — mandatory, immediate, email-first (the recipient may have no session yet)

| Key | Recipient | Notes |
|---|---|---|
| `account.invited` | the invited user | Carries the invite link (01 FR-U-03) |
| `account.password_reset` | the requesting user | 01 FR-A-09 |
| `account.password_changed` | the user | Security notice; also alerts on a change they did not make |
| `account.suspended` | the user | |

**Approvals** — mandatory, immediate

| Key | Recipient |
|---|---|
| `attendance.correction_pending` | the employee's manager, plus HR |
| `attendance.correction_decided` | the requester |
| `leave.request_pending` | the approver (feature 06) |
| `leave.request_decided` | the requester |

**Attendance**

| Key | Recipient | Channels | Digest |
|---|---|---|---|
| `attendance.missing_punch` | the employee | in-app, email | ✅ |
| `attendance.marked_absent` | the employee, their manager | in-app | ✅ |
| `attendance.overtime_pending` | the manager | in-app, email | ✅ |

**System health** — to permission holders, not to individuals

| Key | Recipient | Notes |
|---|---|---|
| `device.offline` | holders of `device.read` | 03 FR-D-09. One per transition, not per check (D-05) |
| `device.gap_unresolved` | holders of `attendance.write` | 04 FR-G-05 |
| `attendance.unmatched_pin` | holders of `attendance.manage_unmatched` | 04 FR-U-04 |
| `holiday.coverage_low` | holders of `holiday.write` | 03 FR-W-12 |
| `employee.document_expiring` | holders of `employee.document.write`, and the employee | 02 FR-D-07 |
| `notification.delivery_failing` | super admins, **in-app only** | FR-N-05 — email may be the thing that is broken |
| `attendance.queue_stalled` | super admins, in-app | 04 FR-R-10 |

Features 06, 07, 09, and 10 add their own keys in their Step 3 files.

## Non-functional requirements

| ID | Requirement |
|---|---|
| NFR-01 | `notify()` adds at most one insert to the caller's transaction. Recipient resolution happens in the worker, not in the caller's path — a notification to "everyone with `device.read`" must not make a device health check slow. |
| NFR-02 | The unread badge is served from a counter or an indexed count, not a table scan (FR-A-03). It is requested on every page load by every user. |
| NFR-03 | The worker processes a batch without N+1 queries: recipients, preferences, and templates are loaded per batch. |
| NFR-04 | A single trigger fanning out to 200 recipients produces one outbox row per recipient per channel, written in one batch insert. |
| NFR-05 | No notification content is written to application logs — bodies may contain names, dates, and links that identify people. |
| NFR-06 | The worker is safe to run alongside itself: claiming a row for delivery must be atomic, so two workers cannot send the same email twice. |
| NFR-07 | An SMTP outage must not grow the queue unboundedly in memory; the queue is the database table, and the worker holds only its current batch. |
| NFR-08 | Template rendering is sandboxed: templates are data, not code. No arbitrary expression evaluation. |

## Permission keys added by this feature

| Key | Meaning | Default holders |
|---|---|---|
| `notification.read` | See your own notifications and set your own preferences | Everyone: SELF |
| `notification.template.read` | View templates | HR: ALL |
| `notification.template.write` | Edit and reset templates | HR: ALL |
| `notification.log.read` | View the delivery log and health | HR: ALL |
| `notification.admin` | Retry sends, manage suppression, set org-wide defaults | Super admin |

## Acceptance criteria (feature-level)

1. `notify()` called inside a transaction that then rolls back queues nothing.
2. The same idempotency key submitted twice produces one notification.
3. A device offline for a week produces one `device.offline` notification, not 168.
4. A notification whose recipient cannot see the linked record is never created, verified by a test
   where a manager is outside the subject employee's scope.
5. A manager who is also an HR admin receives one copy.
6. Disabling email for `attendance.missing_punch` still produces the in-app record.
7. Attempting to disable `leave.request_pending` is not possible in the UI, and an API call that
   tries is rejected.
8. With SMTP unreachable, the triggering action still completes; the outbox shows the item retrying,
   and after the final attempt it is marked failed and appears in the log.
9. A hard bounce suppresses the address; a later notification to it is recorded as skipped, with a
   reason, not silently dropped.
10. A template edited to reference an undeclared variable is rejected at save time.
11. A template that throws at send time still delivers, using the shipped default, and raises an
    alert naming the template.
12. Digest-eligible items accumulate into one 08:00 email; a day with none sends nothing.
13. Feature 01's invite email is delivered by this feature, and the temporary outbox table from 01 is
    removed in the same release.
14. An employee with no user account and no work email produces an undeliverable record that is
    counted and visible, not a silent no-op.
