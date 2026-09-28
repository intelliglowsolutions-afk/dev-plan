# 01 — Roles, Permissions & Auth — Requirements

## Actors

| Actor | Description |
|---|---|
| Super admin | Owns the system. Manages users, roles, and settings. Cannot be deleted or locked out (see FR-U-08). |
| HR admin | Day-to-day HR operator. Manages employees, attendance, leave, payroll across the whole company. |
| Manager | Leads a team. Reads and approves for their own reports only (`DEPARTMENT` scope). |
| Employee | Self-service only (`SELF` scope): own profile, own attendance, own leave, own payslips. |
| Anonymous visitor | Not signed in. May reach the login page, the forgot-password page, and a reset link. |
| Device | A SenseFace terminal pushing to `/iclock/*`. Authenticated by comm key, never by session. Out of scope here, listed so the exemption is explicit. |

## User stories

### Authentication

- **US-01** — As a user, I sign in with my email and password so I can reach the system.
- **US-02** — As a user, I stay signed in across page loads and browser restarts until my session
  expires or I sign out, so I do not re-authenticate constantly.
- **US-03** — As a user, I sign out, ending my session on the server, so the next person at this
  computer cannot use my account.
- **US-04** — As a user who forgot my password, I request a reset link by email and set a new
  password, so I can recover without contacting HR.
- **US-05** — As a newly invited user, I am forced to set my own password on first sign-in, so the
  admin-issued temporary password stops working.
- **US-06** — As a user, I change my password while signed in, which ends my other sessions.
- **US-07** — As a security-conscious owner, repeated failed sign-ins temporarily lock the account,
  so passwords cannot be guessed at speed.

### Users and roles

- **US-08** — As an admin, I invite a user by email, optionally linking them to an employee record,
  and assign roles, so they can sign in with the right access.
- **US-09** — As an admin, I see all user accounts with status, linked employee, roles, and last
  sign-in, so I know who has access.
- **US-10** — As an admin, I suspend or reactivate a user, so departing staff lose access immediately
  without their history being deleted.
- **US-11** — As an admin, I change a user's roles, and the change takes effect on their very next
  request, not at their next sign-in.
- **US-12** — As an admin, I create a custom role and choose exactly which permissions it grants and
  at which scope, so I can model roles the four defaults do not cover.
- **US-13** — As an admin, I see which users hold a role before I change or delete it, so I do not
  strip access by accident.
- **US-14** — As an admin, I reset another user's password, forcing a change at their next sign-in.

### Audit

- **US-15** — As an admin, I browse the audit log filtered by actor, action, entity, and date, so I
  can answer "who changed this salary, and when".
- **US-16** — As an admin, I open an audit entry and see the before and after values of the change.

## Functional requirements

### Authentication — FR-A

| ID | Requirement |
|---|---|
| FR-A-01 | Sign-in takes an email and a password. Email match is case-insensitive. |
| FR-A-02 | Passwords are stored only as an Argon2id hash (bcrypt acceptable fallback). Plaintext and reversible encryption are prohibited. |
| FR-A-03 | A failed sign-in returns one generic message — "Email or password is incorrect" — whether the email is unknown, the password wrong, or the account suspended. Never reveal which. |
| FR-A-04 | A successful sign-in creates a session row and sets an `httpOnly`, `SameSite=Lax`, `Secure` (outside local dev) cookie holding an opaque 256-bit token. Only a hash of the token is stored. |
| FR-A-05 | Sessions expire 12 hours after last use (idle) and 30 days after creation (absolute), whichever comes first. Each authenticated request refreshes `lastUsedAt`, at most once per minute. |
| FR-A-06 | Every authenticated request re-reads the session, its user, and the user's effective permissions from the database. A suspended user or a changed role takes effect on the next request. |
| FR-A-07 | Sign-out revokes the current session row and clears the cookie. A revoked or expired session is indistinguishable from an absent one. |
| FR-A-08 | After 5 consecutive failed attempts, the account is locked for 15 minutes. A successful sign-in resets the counter. The counter is per account, and a matching per-IP throttle applies to the login endpoint. |
| FR-A-09 | Password reset: the request endpoint always responds success, whether or not the email exists. The emailed token is single-use, valid 60 minutes, stored hashed, and invalidated once used or once the password changes by another route. |
| FR-A-10 | Changing a password — by reset, by self-service change, or by admin reset — revokes every other session belonging to that user. A self-service change keeps the current session alive. |
| FR-A-11 | While `mustChangePassword` is set, every route except sign-out and the change-password endpoint redirects to the change-password screen. |
| FR-A-12 | State-changing requests (POST/PATCH/DELETE) are protected against CSRF, either by an origin check or by a double-submit token. |
| FR-A-13 | `/iclock/*` is exempt from session authentication and is authorized by the device comm key instead. The exemption is an explicit allowlist of that path prefix, never a default-open rule. |

### Authorization — FR-Z

| ID | Requirement |
|---|---|
| FR-Z-01 | A permission key is `module.action`, e.g. `employee.read`. The catalogue is seeded from code; it is not user-editable. |
| FR-Z-02 | A role grants a set of permissions. Each grant carries a scope: `ALL`, `DEPARTMENT`, or `SELF`. |
| FR-Z-03 | A user may hold several roles. Their effective permissions are the union; where two roles grant the same key at different scopes, the widest scope wins (`ALL` > `DEPARTMENT` > `SELF`). |
| FR-Z-04 | Every server entry point that reads or writes data declares the permission it requires. An entry point with no declared permission is a bug, and the shared handler wrapper fails closed if one is missing. |
| FR-Z-05 | Scope is enforced as a **data filter**, not merely as a yes/no gate: a `DEPARTMENT`-scoped read returns only rows for employees in the actor's scope, and a `SELF`-scoped read only the actor's own. |
| FR-Z-06 | `DEPARTMENT` scope resolves to the set of employees reporting to the actor, directly or transitively, plus the actor. Until feature 02 provides reporting lines, it resolves to an empty set and must not fall back to "everyone". |
| FR-Z-07 | A denied request returns 403 with no detail about the record's existence; an unauthenticated one returns 401. API responses never leak whether a record the actor cannot see exists. |
| FR-Z-08 | System roles (`isSystem`) cannot be deleted or renamed. `SUPER_ADMIN`'s permission set cannot be edited. The other three system roles' permissions may be edited. |
| FR-Z-09 | A role that is still assigned to at least one user cannot be deleted; the admin must reassign those users first. |

### Users — FR-U

| ID | Requirement |
|---|---|
| FR-U-01 | A user has: email (unique), password hash, status, optional employee link, roles, timestamps, last sign-in. |
| FR-U-02 | Statuses are `INVITED`, `ACTIVE`, `SUSPENDED`, `LOCKED`. Only `ACTIVE` may sign in. `LOCKED` is set by FR-A-08 and clears itself; `SUSPENDED` is set by an admin and clears only by an admin. |
| FR-U-03 | Inviting a user creates the account in `INVITED` with `mustChangePassword` set and sends an invite link. The account becomes `ACTIVE` when the password is set. |
| FR-U-04 | An employee record may be linked to at most one user account, and a user to at most one employee. |
| FR-U-05 | Users are never hard-deleted; they are suspended. Their audit entries and their history in other modules must survive. |
| FR-U-06 | A user cannot change their own roles or their own status, and cannot suspend themselves. |
| FR-U-07 | Email changes require the new address to be unique and are audited. |
| FR-U-08 | At least one `ACTIVE` user holding `SUPER_ADMIN` must exist at all times. Any operation that would leave zero — suspension, role removal, role edit — is rejected. |

### Audit — FR-L

| ID | Requirement |
|---|---|
| FR-L-01 | Every create, update, and delete performed through the application is audited: actor, action, entity type, entity id, timestamp, IP, user agent. |
| FR-L-02 | Updates record `before` and `after` as JSON, limited to the fields that actually changed. |
| FR-L-03 | Sensitive values are never written to the audit log: password hashes, reset tokens, session tokens, the device comm key. Salary figures **are** audited — that is a primary reason the log exists. |
| FR-L-04 | Audit rows are append-only. The application exposes no update or delete path for them. |
| FR-L-05 | The audit write happens in the same database transaction as the change. If the audit write fails, the change rolls back. |
| FR-L-06 | Authentication events are audited too: sign-in success, sign-in failure, lockout, sign-out, password change, password reset. Failures record the attempted email but never the attempted password. |
| FR-L-07 | The log is readable only with `audit.read`, and is filterable by actor, action, entity type, and date range. |

## Non-functional requirements

| ID | Requirement |
|---|---|
| NFR-01 | The permission check on an authenticated request adds no more than ~10 ms. Effective permissions are loaded in one query, cached per request; no per-check round trip. |
| NFR-02 | Password hashing is deliberately slow (Argon2id, ≥64 MB memory cost). This is the one place latency is a feature. |
| NFR-03 | No secret is ever logged: passwords, tokens, hashes, and the comm key must not appear in application logs or error output. |
| NFR-04 | Error messages shown to users are generic; detail goes to server logs only. |
| NFR-05 | Sign-in and password-reset endpoints are rate-limited per IP as well as per account. |
| NFR-06 | The authorization helpers must be simple enough that adding a new protected endpoint is one wrapper call — otherwise modules 02–11 will skip them. |
| NFR-07 | All timestamps are stored UTC. Display timezone comes from company settings (feature 03). |
| NFR-08 | Seeding is idempotent: re-running the permission/role seed on an existing database adds new keys and leaves custom roles untouched. |

## Role model

Seeded system roles and their default grants. `—` means the role does not hold the permission.

| Permission area | SUPER_ADMIN | HR_ADMIN | MANAGER | EMPLOYEE |
|---|---|---|---|---|
| `user.*`, `role.*` | ALL | — | — | — |
| `audit.read` | ALL | ALL | — | — |
| `employee.read` | ALL | ALL | DEPARTMENT | SELF |
| `employee.write` | ALL | ALL | — | — |
| `settings.*` | ALL | ALL | — | — |
| `attendance.read` | ALL | ALL | DEPARTMENT | SELF |
| `attendance.write` (corrections) | ALL | ALL | DEPARTMENT | — |
| `leave.read` | ALL | ALL | DEPARTMENT | SELF |
| `leave.request` | ALL | ALL | SELF | SELF |
| `leave.approve` | ALL | ALL | DEPARTMENT | — |
| `payroll.read` | ALL | ALL | — | SELF |
| `payroll.run` | ALL | ALL | — | — |
| `report.read` | ALL | ALL | DEPARTMENT | — |

`SUPER_ADMIN` is defined as holding every permission in the catalogue at `ALL`, including permission
keys added by later features. It is computed, not stored as a fixed list, so a new module cannot
accidentally be invisible to the owner.

## Permission catalogue

Feature 01 seeds its own keys and reserves the namespaces later features will fill. Each feature's
`data-model.md` adds its keys to this catalogue in Step 3.

**Owned by this feature**

| Key | Meaning |
|---|---|
| `user.read` | View user accounts |
| `user.write` | Invite, edit, suspend, reactivate users; reset their passwords |
| `user.assign_role` | Change which roles a user holds |
| `role.read` | View roles and their permissions |
| `role.write` | Create, edit, delete custom roles; edit system-role permissions |
| `audit.read` | Read the audit log |

**Reserved namespaces** — `employee.*`, `settings.*`, `device.*`, `attendance.*`, `notification.*`,
`leave.*`, `payroll.*`, `performance.*`, `recruitment.*`, `report.*`.

## Acceptance criteria (feature-level)

1. A fresh database seeded and started yields exactly one `SUPER_ADMIN` account, created from
   environment variables, with `mustChangePassword` set.
2. That admin can sign in, is forced to change the password, and lands on the dashboard.
3. The admin can invite a user, assign `MANAGER`, and that user can sign in via the invite link.
4. A `MANAGER` requesting a list scoped `DEPARTMENT` receives only their own reports — verified by a
   test that seeds two departments.
5. An `EMPLOYEE` calling an admin endpoint directly with curl receives 403, not a redirect.
6. Suspending a user takes effect on that user's next request, without them signing out.
7. Five wrong passwords lock the account for 15 minutes; the sixth attempt with the *correct*
   password is still refused, with the same generic message.
8. Every user, role, and auth event above appears in the audit log with the correct actor.
9. Attempting to suspend the last remaining super admin is rejected with a clear message.
10. `/iclock/cdata?SN=...` still returns `OK` with no session cookie present.
