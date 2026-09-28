# 05 — Notifications — UI / UX

Builds on the shell and principles in [01 ui-ux.md](../01-roles-permissions-auth/ui-ux.md).

> **Design grounding.** Written against the workspace `ui-ux-pro-max` guideline set rather than from
> memory, per `C:\Dev\CLAUDE.md`. The guidelines that shaped this document are cited inline as
> *[UX-nn]* and listed at the end. The visual direction stays with the dense-admin default already
> established in features 01–04 — no new palette, no motion-heavy treatment; this is an internal HR
> product where scanning speed beats delight.

## Principles specific to this feature

1. **The badge is a promise.** A number in the top bar says "something here needs you". If it counts
   things that do not, people stop looking, and every other feature's approvals stall behind it.
   This is why superseded items stop counting (FR-A-05) and why informational types default to
   digest.
2. **Notifications are a record, not an interruption.** The centre is where you go to find out what
   happened. Toasts are for the thing you just did. Conflating them means either missing what
   matters or being interrupted by what does not.
3. **Never make the user guess what a setting does.** "Attendance reminders" means nothing. "Sent
   when you have not scanned out by the end of your shift" means something. The description comes
   from the catalogue so it stays true.
4. **Every notification lands somewhere specific.** A notification that opens a dashboard has
   wasted the click it just earned.

## Screen map

```
(global)  Notification bell + panel — in the top bar       notification.read
/notifications                  Full list                  notification.read
/settings/notifications         Own preferences            notification.read
/admin/notifications/templates  Template list + editor     notification.template.read
/admin/notifications/log        Delivery log + health      notification.log.read
```

## The bell and the badge

Top bar, left of the user menu. A bell icon with a count.

**Accessibility, which is most of the design here:**

- The count is announced as one atomic status message — "3 unread notifications" — not as a bare
  number, and it is the *only* live region in the shell. A badge that announces "3" tells a screen
  reader user nothing, and several competing live regions tell them less than one *[UX-118]*.
- The bell is a `<button>` with an accessible name that includes the count, not an icon alone
  *[UX-117, UX-1]*.
- The badge occupies a **stable slot** whether or not it is showing, so appearing does not shove the
  user menu sideways *[UX-19]*.
- The count is a status, rendered as static text — not a chip, not clickable on its own *[UX-114]*.
- Counts above 99 render as "99+" on one line, never wrapping *[UX-116]*.

**The panel** opens on click (not hover — hover-only disclosure is unusable by touch and keyboard
*[UX-117]*): the 10 most recent, grouped under *New* and *Earlier*, each with an icon by category,
title, one line of body, and relative time. Footer: *Mark all read* and *See all*.

Keyboard: Escape closes and returns focus to the bell; arrow keys move through items; Enter
activates. The panel must not obscure the focused element behind it when closing *[UX-100]*.

Polling every 60 seconds for the count, matching feature 04's grid. The panel animates open in
~150 ms, and not at all under `prefers-reduced-motion` *[UX-9]*.

## `/notifications` — the full list

A table-density list, not cards: title, body line, category, time, and read state. Unread rows carry
a weight change and a leading marker — never colour alone *[UX-114]*.

Filters as a row of toggle buttons (real `<button aria-pressed>`, not styled divs *[UX-117]*):
*All · Unread · Approvals · Attendance · Leave · System*. Counts on each.

Row actions: open (the whole row is the target), mark read/unread, and — where the underlying thing
is still actionable — the action itself inline, so an approver can clear three correction requests
without leaving the list.

**Superseded items** render dimmed with a short explanation: "Handled by Ravi Menon" or "No longer
needed". They stay visible, because "who dealt with this" is useful, and they do not count *[FR-A-05]*.

**Empty states** *[UX-79]*, and they differ:

- Nothing at all: "Nothing here yet. You'll see approvals, reminders, and system alerts as they
  happen."
- Nothing unread: "You're all caught up." — with a link to read history rather than a blank panel.
- A filter with no matches: "No notifications in *Leave*." with *Clear filter*.

Loading is a skeleton of stable-height rows so the list does not jump as it fills *[UX-78, UX-19]*.

## Toasts vs notifications

Worth stating once, because the two get conflated and the consequences are quiet:

| | Toast | Notification |
|---|---|---|
| For | The action you just took | Something that happened, possibly elsewhere |
| Lifetime | 3–5 s, auto-dismiss *[UX-82]* | Until read |
| Carries an action | Rarely — and never *only* there | Often |
| Record kept | No | Yes |

A toast is **never** the only place an actionable thing appears. If a toast is the only notice that
an approval is waiting, it is gone in four seconds and nobody is accountable for it *[UX-34, UX-82]*.

Toasts announce through the same atomic status region pattern, and never steal focus.

## `/settings/notifications` — preferences

Generated from the catalogue, grouped by category. One row per type:

```
Approvals
  A request needs your approval                          Email  [always on]
    Sent when someone's leave or correction request is waiting on you.
    You can't turn this off — other people are waiting on you.

Attendance
  You didn't scan out                                    Email  ( ●  )   Digest ▾
    Sent when your shift ends and no sign-out was recorded.
```

- The **description is the label that matters**; the type name alone is not enough *[UX-8 forms /
  helper text]*. Both come from the catalogue, so wording cannot drift from behaviour.
- **Mandatory types** show a fixed "always on" state with the reason inline — not a disabled toggle
  with no explanation, which reads as a bug *[FR-P-03]*.
- The digest selector appears only for digest-eligible types, and says what it means: "In the daily
  8:00 summary" vs "As it happens".
- A line at the top sets the expectation the whole screen depends on: **"These settings control
  email only. Everything still appears in your notifications list."** Without it, a user who mutes a
  type believes they will never hear about it *[D-03]*.
- Types the user cannot receive are absent entirely, not greyed *[FR-P-06]* — consistent with
  feature 01's rule that hidden means hidden.
- Saving is per-toggle with an inline confirmation, not a page-level Save — these are independent
  preferences, and a Save button invites half-saved state *[UX-34]*.

Footer: the digest time, and where to change it if the user may.

## `/admin/notifications/templates`

**List** — type, category, both channels, whether each is customised, last edited by and when. A
*Customised* marker is the useful column: it answers "what have we changed from the defaults".

**Editor** — two panes. Left: subject and body, HTML and plain text as tabs. Right: live preview
with sample data, toggleable between rendered HTML and plain text.

Beneath the fields, the **available variables** as insertable chips — real buttons, each inserting
at the cursor, with the sample value shown *[UX-117]*. This is the difference between an editor
someone can use and one they are guessing at.

Validation on blur, not on save *[UX-56]*: an unknown `{{variable}}` is underlined in place with the
message beside the field — "`employee.salary` isn't available here" — plus the list of what is
*[FR-M-03]*. Errors never appear only in a summary at the top.

Actions: *Save*, *Reset to default* (with a confirmation naming what is lost), *Send test to
myself*. The test-send result appears inline — "Sent to hr@company.com" or the actual SMTP error,
which is what makes this the fastest way to diagnose a misconfiguration *[FR-N-04]*.

Unsaved changes warn on navigation, as elsewhere in the system.

## `/admin/notifications/log` — delivery and health

**Health strip first**, because this screen is opened when something is suspected to be broken:

```
Queue 4   ·   Oldest pending 12s   ·   Sent 24h 412   ·   Failed 24h 2   ·   Skipped 24h 18
Worker last ran 20 seconds ago            Mode: SMTP
```

`Worker last ran` is given equal weight to the queue depth: a queue of zero with a stale worker is
the failure that looks like health. Both go amber past their thresholds, with the word, not just the
colour *[UX-114]*.

`Skipped` is clickable and lands on the skipped rows. A high skip count is the quiet failure mode —
a suppression or a muted preference swallowing something that mattered.

**The log table**: time, type, recipient, channel, status, attempts, last error. Filters across the
top, reflected in the URL. A row expands in place to show the rendered subject, the resolution
reason, and — in `LOCAL_OUTBOX` mode — *View rendered email*, which is how the whole notification
system gets tested before any mail server exists *[FR-D-08]*.

Failed rows offer *Retry*. Suppressed addresses have their own tab with the bounce reason and a
*Release* action, worded honestly: "Release hr@old-domain.com? Email will be attempted again. If the
address is still invalid it will bounce and be suppressed again."

When email delivery is failing, a banner sits at the top of **every admin screen**, not only this
one — because an admin who is not looking at the notification log is exactly who needs to know
*[FR-N-05]*.

## States, accessibility, responsive

- Six states everywhere (loading, empty, filtered-empty, error, denied, saving), plus *skipped* and
  *superseded* as first-class visual states here.
- Every status — read, unread, failed, skipped, suppressed, superseded — carries a word. Colour is
  reinforcement only *[UX-114]*.
- Full keyboard operation: bell, panel, list, filters, and the template editor's variable chips
  *[UX-41]*. Visible focus rings throughout, never removed *[UX-28]*; sticky headers offset with
  `scroll-padding` so focus is never hidden behind them *[UX-100]*.
- One live region for the unread count, atomic, with meaningful text *[UX-118]*.
- Motion is minimal and respects `prefers-reduced-motion` *[UX-9]*.
- Touch targets ≥ 44×44 px — particularly the bell and the per-row mark-read control, which are the
  two most-tapped things in the feature.
- Below 768 px the panel becomes a full-height sheet; the preferences rows stack with the toggle
  beneath the description; the log table falls back to stacked cards keeping time, type, recipient,
  and status.

## Copy reference

| Situation | Text |
|---|---|
| Badge, announced | {n} unread notifications |
| All caught up | You're all caught up. |
| Nothing ever | Nothing here yet. You'll see approvals, reminders, and system alerts as they happen. |
| Superseded | Handled by {name} — no action needed. |
| Preferences header | These settings control email only. Everything still appears in your notifications list. |
| Mandatory type | You can't turn this off — other people are waiting on you. |
| Digest option | In the daily 8:00 summary / As it happens |
| Unknown variable | `{{variable}}` isn't available for this notification. Available: {list}. |
| Reset template | This replaces your wording with the original. Your version isn't kept. |
| Test send result | Sent to {address}. / Couldn't send: {error}. |
| Release suppression | Release {address}? Email will be attempted again. If the address is still invalid it will bounce and be suppressed again. |
| Delivery failing banner | Email delivery is failing — {n} of the last {m} messages could not be sent. People may not be receiving approvals. *(View log)* |
| No recipients | This notification resolved to nobody. Check who holds {permission}. |

## Guidelines applied

From `.claude/skills/ui-ux-pro-max/data/ux-guidelines.csv`:

| Ref | Guideline | Where it shows up here |
|---|---|---|
| UX-118 | Contextual live badge updates — one atomic status message, never a bare number | The unread badge, and toasts |
| UX-19 | Content jumping — stable count slot | Badge does not shift the top bar when it appears |
| UX-114 | Compact label semantics — badges are state, not buttons; never colour alone | All status indicators |
| UX-116 | Compact label overflow — no wrapping, no hover-only recovery | "99+", category chips |
| UX-117 | Compact control semantics — real buttons with accessible names and pressed state | Filter toggles, variable chips, the bell |
| UX-82 / UX-34 | Toasts auto-dismiss 3–5 s; confirm success | Toast vs notification split |
| UX-79 | Empty states guide rather than sit blank | Three distinct empty states |
| UX-56 | Inline validation on blur, error beside the field | Template editor |
| UX-78 | Loading matched to expected wait, stable skeletons | List and panel loading |
| UX-28 / UX-41 / UX-100 | Visible focus, full keyboard nav, focus never obscured | Panel, list, editor |
| UX-9 | Honour `prefers-reduced-motion` | Panel and toast motion |

## Open questions

| ID | Question |
|---|---|
| OQ-513 | Should the bell poll, or should the count be pushed (SSE/websocket)? Polling at 60 s is honest and cheap; approvers may find it slow. Consistent with feature 04's grid either way — both should make the same choice. |
| OQ-511 | Snooze on a notification — raised in `api-design.md`; it is primarily a UI affordance and worth deciding with this screen rather than after it. |
| OQ-514 | Does the notification panel need a "only show things I can act on" mode? Approvers with broad permissions will receive a lot of system-health noise alongside their approvals. |
| OQ-508 | Whether managers see their team's read state — currently no, deliberately. The UI would be trivial; the question is whether it should exist. |
