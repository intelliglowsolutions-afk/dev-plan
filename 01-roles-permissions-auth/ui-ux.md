# 01 — Roles, Permissions & Auth — UI / UX

## Principles

1. **The server decides; the UI only reflects.** Hiding a button is a courtesy, not a control
   (FR-Z-05, D-05). Every screen below assumes the endpoint behind it re-checks.
2. **Never reveal what the actor may not see.** No "you don't have access to *Payroll*" — a hidden
   module is simply absent from the navigation.
3. **Say what happened and what to do next.** Auth errors are the one place where a vague message is
   deliberate; everywhere else, vagueness is a defect.
4. **Destructive and access-changing actions confirm**, and the confirmation states the consequence
   in plain words ("Ayesha will be signed out immediately and cannot sign back in").

## Screen map

```
Public
  /login                     Sign in
  /forgot-password           Request a reset link
  /reset-password?token=...  Set a new password (also completes an invite)

Authenticated
  /change-password           Forced first-login change (blocks everything else)
  /dashboard                 Landing page; contents vary by permission
  /admin/users               User list                     user.read
  /admin/users/new           Invite user                   user.write
  /admin/users/:id           User detail: roles, status, sessions
  /admin/roles               Role list                     role.read
  /admin/roles/:id           Role editor (permission matrix)  role.write
  /admin/audit               Audit log                     audit.read
  /403                       Access denied
```

## Layout and navigation

A left sidebar with the module list, a top bar with the company name and a user menu (name, email,
role chips, *Change password*, *Sign out*).

Sidebar items are rendered from the session's effective permissions: *Users* and *Roles* appear only
with `user.read` / `role.read`; *Audit log* only with `audit.read`. A user with no admin permissions
sees no Administration group at all. The group header is hidden when it would be empty — an empty
section header reads as breakage.

Because permissions are resolved server-side in a Server Component, the sidebar renders correct on
first paint, with no flash of admin links.

## Screens

### `/login`

Centred card on a plain background. Company name and logo from settings (feature 03); until then, a
text wordmark.

- Fields: **Email**, **Password** (with a show/hide toggle), a **Sign in** button, and a
  *Forgot password?* link.
- Submit disables the button and shows an inline spinner. Nothing else moves.
- **Error:** one line above the fields — "Email or password is incorrect." — identical for unknown
  email, wrong password, and suspended account (FR-A-03). The fields keep the typed email and clear
  the password.
- **Locked:** "Too many attempts. Try again in 15 minutes, or ask an administrator to reset your
  password." This is the one case where extra detail is warranted: the user is almost always the
  legitimate owner, and silence here generates support calls.
- **Rate-limited (429):** "Too many attempts from this device. Wait a few minutes."
- Autofocus on Email. Enter submits. `autocomplete="email"` and `"current-password"` so password
  managers work.
- After success, redirect to `redirectTo` from the response, or to the originally requested URL if
  the user was bounced here from a protected page.

### `/forgot-password`

Single Email field. On submit, the form is **replaced** by a confirmation panel regardless of
outcome: "If an account exists for that address, a reset link is on its way. The link expires in 60
minutes." No hint that the address was or was not found (FR-A-09). A *Back to sign in* link.

### `/reset-password?token=...`

Fields: **New password**, **Confirm password**, with a live policy checklist (at least 12
characters; the checklist follows whatever OQ-103 settles on) that ticks as the user types. The
submit button stays disabled until both fields match and the policy passes.

- **Invalid or expired token:** the form is not rendered at all. Instead: "This link has expired or
  has already been used." with a button to request a new one.
- **Success:** "Your password has been set. You have been signed out of other devices." then a
  redirect to `/login` — or straight into the app if we choose to sign them in on completion (this
  is worth settling; see OQ-112).
- The same screen serves the invite flow, with the heading "Set your password" and a line naming the
  company.

### `/change-password` (forced)

Shown when `mustChangePassword` is set. The sidebar and top nav render but every link is inert; the
only working controls are the form and *Sign out* (FR-A-11). A short explanation at the top: "Before
you continue, choose a password only you know."

Fields: current (temporary) password, new, confirm. On success, land on `/dashboard`.

### `/admin/users`

Table, 25 per page:

| Column | Notes |
|---|---|
| Name | From the linked employee; falls back to the email's local part with a subtle *No employee record* tag |
| Email | |
| Roles | Chips; overflow collapses to "+2" |
| Status | Badge — `ACTIVE` neutral, `INVITED` blue, `SUSPENDED` grey, `LOCKED` amber |
| Last sign-in | Relative ("3 hours ago"), absolute UTC in the tooltip |

Toolbar: search box (email or name), status filter, role filter, **Invite user** button (only with
`user.write`). Row click opens the detail page.

Empty state, only ever true on a brand-new system: "Only your account exists so far. Invite your HR
team to get started." with the invite button.

### `/admin/users/new` — invite

A short form: Email, *Link to employee* (searchable select over employees without an account —
optional, with a note that the link can be added later), Roles (multi-select showing each role's
name and description), and a *Send invite email* checkbox, on by default.

On submit: a success toast naming the address, and a return to the list with the new row
highlighted. If email delivery is not yet wired up (before feature 05), the success panel shows the
invite link with a copy button, plus a caution that the link is valid for 60 minutes and grants
access to whoever holds it.

### `/admin/users/:id`

Header: name, email, status badge, role chips. Action buttons gated by permission —
*Edit*, *Change roles*, *Reset password*, *Suspend* / *Reactivate*.

Three panels:

- **Account** — email, linked employee (with a link to the employee record), created by and when,
  last sign-in.
- **Roles** — current roles with their descriptions, and a *Change roles* dialog with checkboxes.
  The dialog spells out what changes: "Adding *HR admin* grants access to all employee records,
  payroll, and settings." Saving shows a toast; the change is live immediately.
- **Sessions** — live sessions with device, IP, and last-used time, each with *Revoke*, plus
  *Sign out everywhere*.

Guard rails, shown as disabled buttons with an explaining tooltip rather than as errors after the
fact:

- Viewing your own account: *Change roles* and *Suspend* are disabled — "You cannot change your own
  roles or status." (FR-U-06)
- The last active super admin: *Suspend* and role removal disabled — "This is the only super admin.
  Give someone else the role first." (FR-U-08)

*Suspend* opens a confirmation naming the person and stating that they will be signed out
immediately and their records will be kept.

### `/admin/roles`

Cards or a compact table: role name, description, a *System* tag where applicable, permission count,
user count. *New role* button with `role.write`.

Deleting a role in use is blocked in the UI before the API refuses it: the delete control is disabled
with "3 users hold this role. Reassign them first," and the user count links to the filtered user
list.

### `/admin/roles/:id` — the permission matrix

The most important screen in this feature. Rows are permissions grouped by module (Employees,
Attendance, Leave, Payroll, …); each row has four radio options: **None · Self · Department · All**.

- Group headers collapse, and carry a summary chip ("Leave: 2 of 5"), so a 120-row catalogue stays
  navigable.
- A group-level control sets every row in the module at once.
- Each permission shows its description in plain language — "View leave requests", not
  `leave.read` — with the key itself in a tooltip for the technically minded.
- Unsaved changes mark the row and enable a sticky footer: "6 changes. Save · Discard." Navigating
  away warns.
- Saving explains the blast radius in the confirmation: "This role is held by 12 people. Their access
  changes immediately."
- `SUPER_ADMIN` opens read-only with a banner: "The super admin role always has full access and
  cannot be edited." Other system roles are editable, but renaming and deleting are not (FR-Z-08).
- Scopes wider than the role's other grants are not validated against each other — the matrix is the
  single source of truth and combinations are the admin's call.

### `/admin/audit`

A dense, chronological table: time (relative, absolute on hover), actor, action (as a readable
phrase, e.g. "Changed roles"), entity (type and a link to the record where one exists), and summary.

Filters across the top: date range, actor, action, entity type, free-text. Filters are reflected in
the URL so a filtered view can be shared or bookmarked.

A row expands in place to show the before/after diff: two columns, changed fields only, removed
values struck through in red, new values in green. Values that are objects render as formatted JSON.

Export to CSV sits behind the same `audit.read` permission and respects the active filters.

Empty state for an over-narrow filter: "No activity matches these filters" with a *Clear filters*
action — distinct from the genuinely empty log, which says "No activity recorded yet."

### `/403` and inline denials

A full-page denial says: "You don't have access to this page." plus a *Back to dashboard* button and
nothing else — no mention of which permission is missing, which would map the system for an attacker.

For an action denied mid-flow (a stale tab whose roles changed underneath it), a toast: "Your access
has changed. Refresh to continue." Session expiry mid-action shows a modal — "You've been signed out"
— with a sign-in link, and preserves the current URL for the return trip.

## States to design for every screen

Loading (skeleton rows, not a spinner over a blank page) · empty · error · permission-denied ·
saving/disabled · offline. A table that has only ever been designed full will look broken on day one,
when every list here is empty.

## Accessibility

- All controls reachable by keyboard; visible focus rings; the login form fully operable without a
  mouse.
- Status badges carry text, never colour alone — colour-blind users must be able to distinguish
  `SUSPENDED` from `ACTIVE`.
- Errors are associated with their fields via `aria-describedby` and announced in a live region.
- The permission matrix's radio groups are properly labelled per row and per column; a screen reader
  must be able to answer "what scope does *View leave requests* have?".
- Target contrast 4.5:1 for text.

## Responsive behaviour

The admin screens are desktop-first; HR works at a desk. But the self-service portal (feature 08)
shares this shell and will be used on phones, so: the sidebar collapses to a drawer below 768 px,
tables fall back to stacked cards, and the login, forgot, reset, and change-password screens are
built mobile-first from the start.

## Copy reference

| Situation | Text |
|---|---|
| Sign-in failure | Email or password is incorrect. |
| Account locked | Too many attempts. Try again in 15 minutes, or ask an administrator to reset your password. |
| Reset requested | If an account exists for that address, a reset link is on its way. The link expires in 60 minutes. |
| Bad reset token | This link has expired or has already been used. |
| Forced change | Before you continue, choose a password only you know. |
| Suspend confirm | {Name} will be signed out immediately and will not be able to sign back in. Their records are kept. |
| Last super admin | This is the only super admin. Give someone else the role first. |
| Self role change | You cannot change your own roles or status. |
| Role in use | {n} users hold this role. Reassign them first. |
| Access denied | You don't have access to this page. |
| Access changed | Your access has changed. Refresh to continue. |

## Open questions

| ID | Question |
|---|---|
| OQ-112 | After completing a reset or an invite, sign the user straight in, or send them to `/login`? Signing in is friendlier; sending them to login proves they know the password they just set. |
| OQ-113 | Does the dashboard differ per role, or is it one page with permission-gated cards? Affects feature 11 (Reports) more than this one. |
| OQ-114 | Branding — logo, colours, and whether the login screen carries company identity before settings (feature 03) exists. |
| OQ-115 | Language: English only, or is a second language needed? Retrofitting i18n after eleven features is expensive; deciding now costs little. |
