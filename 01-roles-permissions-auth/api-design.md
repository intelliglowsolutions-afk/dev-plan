# 01 — Roles, Permissions & Auth — API Design

## Conventions

- Next.js App Router **route handlers** under `src/app/api/**/route.ts`. Server Components read data
  directly through the same authorization helpers; the HTTP API exists for client interactions and
  for anything a future mobile client would need.
- JSON in, JSON out. `Content-Type: application/json`. Request bodies are validated at the boundary
  (Zod or equivalent); an invalid body is 422 with per-field messages.
- Errors share one shape:
  ```json
  { "error": { "code": "FORBIDDEN", "message": "You do not have access to this." } }
  ```
  `code` is machine-readable; `message` is safe to show a user. Field errors add
  `"fields": { "email": "Already in use" }`.
- Status codes: 200 ok · 201 created · 204 no content · 400 malformed · 401 not signed in ·
  403 signed in but not permitted · 404 not found *or* not visible to this actor (FR-Z-07) ·
  409 conflict · 422 validation · 429 rate-limited · 500 server error.
- List endpoints take `?page=1&pageSize=25` and return
  `{ "data": [...], "page": 1, "pageSize": 25, "total": 137 }`. `pageSize` is capped at 100.
- All request and response timestamps are ISO-8601 UTC.

> **Build note.** `AGENTS.md` warns that this Next.js version diverges from familiar conventions.
> Confirm route-handler and `cookies()` signatures against `node_modules/next/dist/docs/` before
> writing code — the existing device route already shows one such change (`params` is a `Promise`).

## Server-side helpers

These are the contract every other feature depends on. Getting them right matters more than the
endpoints themselves (NFR-06).

```ts
// src/lib/auth/session.ts
getSession(): Promise<SessionContext | null>
// Reads the cookie, looks up the session by token hash, checks revocation and both expiries,
// refreshes lastUsedAt (at most once a minute), loads the user and effective permissions.
// Returns null for absent, expired, revoked, or non-ACTIVE-user sessions — indistinguishable
// from the caller's point of view (FR-A-07).

type SessionContext = {
  user: { id: number; email: string; employeeId: number | null; mustChangePassword: boolean };
  roles: string[];                                  // role keys
  permissions: Map<PermissionKey, PermissionScope>; // effective, widest scope (FR-Z-03)
};
```

```ts
// src/lib/auth/authorize.ts
requireSession(): Promise<SessionContext>             // throws Unauthorized -> 401
requirePermission(key: PermissionKey): Promise<SessionContext & { scope: PermissionScope }>
                                                      // throws Forbidden -> 403
can(ctx: SessionContext, key: PermissionKey): boolean  // for conditional UI on the server

// Turns a scope into a query filter (FR-Z-05). Every scoped list query composes this in.
employeeScopeFilter(ctx, scope): Promise<{ employeeId: { in: number[] } } | {}>
// ALL        -> {}                  (no restriction)
// DEPARTMENT -> ids of the actor's reports, transitively, plus the actor
//               (empty array until feature 02 lands — never "everyone", FR-Z-06)
// SELF       -> [ctx.user.employeeId]
```

```ts
// src/lib/auth/route.ts
protectedRoute(key: PermissionKey, handler: (req, ctx) => Promise<Response>)
// The one wrapper every protected endpoint uses. It resolves the session, checks the permission,
// maps thrown AuthError/ValidationError to the standard error shape, and enforces the
// mustChangePassword redirect (FR-A-11). A handler registered without a permission key does not
// compile — the wrapper's signature requires one (FR-Z-04).
```

```ts
// src/lib/audit.ts
writeAudit(tx, entry: AuditEntry): Promise<void>
// Takes the *transaction client*, not the global prisma, so the audit write lives or dies with
// the change (FR-L-05). Strips the redaction list before writing (FR-L-03) and computes the
// changed-fields diff for before/after.
```

## Authentication endpoints

> **Revised 2026-09-18 (OQ-101): these are now implemented with Auth.js (NextAuth) v5**, not
> hand-rolled. What that changes, and what it does not:
>
> | Concern | With Auth.js |
> |---|---|
> | Session storage | **Database strategy via the Prisma adapter.** The JWT strategy is prohibited — it breaks D-04's immediate revocation |
> | Sign-in | A **Credentials provider** whose `authorize` callback holds the logic below: lookup, Argon2id verify, status check, lockout, generic failure (FR-A-03) |
> | Session cookie, CSRF | Auth.js owns them. FR-A-04 and FR-A-12 are satisfied by configuration rather than by our code — but the settings must be **verified**, not assumed |
> | Session shape | Auth.js's `Session` is extended in a callback to carry `permissions`, `roles`, `mustChangePassword`, and — post-OQ-301 — **`tenantId`** (`MULTI-TENANCY.md` D-T-05) |
> | Per-request re-read | FR-A-06 still applies. The session callback must re-read user status and permissions, **not** trust a cached session row, or a suspended user keeps working until expiry |
> | Password reset, invite, change | **Not** Auth.js's concern. These stay as the endpoints below, backed by feature 05 |
> | Tables | Auth.js's adapter schema replaces the hand-rolled `Session` model; `PasswordResetToken` and the rest of `data-model.md` are unchanged |
>
> Two risks to close early: **Next.js 16 compatibility is unverified** (`AGENTS.md` warns this
> Next.js diverges from training data), and the adapter's session table must be reconciled with
> `data-model.md`'s `Session` model rather than both existing.
>
> The endpoint contracts below still describe the required *behaviour*. Where Auth.js provides a
> route (`/api/auth/[...nextauth]`), these become its configuration rather than separate handlers.

All are public (no session required) except `session`, `logout`, and `password/change`.

### `POST /api/auth/login`

```json
{ "email": "hr@company.com", "password": "..." }
```

**200** — sets the `hrm_session` cookie (`httpOnly`, `SameSite=Lax`, `Secure` outside dev, `Path=/`).

```json
{ "user": { "id": 4, "email": "hr@company.com", "mustChangePassword": false },
  "roles": ["HR_ADMIN"], "redirectTo": "/dashboard" }
```

**401** — `{ "error": { "code": "INVALID_CREDENTIALS", "message": "Email or password is incorrect." } }`
for an unknown email, a wrong password, a suspended account, or a locked one alike (FR-A-03). The
response time is padded so a missing account cannot be detected by timing.

**429** — per-IP and per-account rate limit (FR-A-08, NFR-05).

Side effects: session row created; `lastLoginAt` set; `failedLoginCount` reset; audit
`auth.login` — or, on failure, `failedLoginCount` incremented, `lockedUntil` set at the 5th failure,
audit `auth.login_failed` / `auth.account_locked` with the attempted email and never the password.

### `POST /api/auth/logout`

No body. **204**. Revokes the current session row, clears the cookie, audits `auth.logout`.
Succeeds even without a valid session, so a stale tab can always "sign out".

### `GET /api/auth/session`

**200** with the same payload as login (plus `permissions` as a `{ key: scope }` object) or **401**.
Used by the client to rehydrate after a reload.

### `POST /api/auth/password/forgot`

```json
{ "email": "hr@company.com" }
```

**200** always — `{ "ok": true }` — whether or not the address exists (FR-A-09). If it does, a
single-use token valid 60 minutes is created and a reset link is queued for delivery. Until feature
05 exists, the link is written to the outbox table and logged at info level in development only.
Audit: `auth.password_reset_requested`.

### `POST /api/auth/password/reset`

```json
{ "token": "...", "password": "..." }
```

**200** on success: password replaced, `mustChangePassword` cleared, status `INVITED` → `ACTIVE`,
the token marked used, **all** sessions for that user revoked (FR-A-10). Audit
`auth.password_reset`.
**400** `INVALID_TOKEN` for an unknown, expired, or already-used token — one code for all three.
**422** if the new password fails policy (OQ-103).

The same endpoint completes an invite: the invite link carries a reset token.

### `POST /api/auth/password/change`

Requires a session. `{ "currentPassword": "...", "newPassword": "..." }`

**200** — revokes the user's *other* sessions, keeps the current one, clears
`mustChangePassword`. **401** if `currentPassword` is wrong. Audit `auth.password_changed`.

## User endpoints

| Method | Path | Permission | Notes |
|---|---|---|---|
| GET | `/api/users` | `user.read` | Filters: `q` (email/name), `status`, `roleKey`. Returns roles and linked employee inline. |
| POST | `/api/users` | `user.write` | Invite. See below. |
| GET | `/api/users/:id` | `user.read` | |
| PATCH | `/api/users/:id` | `user.write` | `email`, `employeeId`. Not roles, not status. |
| POST | `/api/users/:id/roles` | `user.assign_role` | Replaces the whole role set. |
| POST | `/api/users/:id/status` | `user.write` | `{ "status": "SUSPENDED" \| "ACTIVE" }` |
| POST | `/api/users/:id/reset-password` | `user.write` | Admin-initiated. Issues a reset token, sets `mustChangePassword`. |
| GET | `/api/users/:id/sessions` | `user.read` | Live sessions, for spotting shared accounts. |
| DELETE | `/api/users/:id/sessions` | `user.write` | Force sign-out everywhere. |

There is no `DELETE /api/users/:id` — accounts are suspended, never deleted (FR-U-05).

### `POST /api/users`

```json
{ "email": "new@company.com", "employeeId": 12, "roleIds": [3], "sendInvite": true }
```

**201** — creates the account `INVITED` with `mustChangePassword`, a random unusable password, and a
60-minute invite token; queues the invite email. Audit `user.invited`.

**409** — `EMAIL_TAKEN`, or `EMPLOYEE_ALREADY_LINKED` when that employee already has an account
(FR-U-04).

### `POST /api/users/:id/roles`

```json
{ "roleIds": [2, 5] }
```

**200** with the user's new roles. Rejects with **422** `SELF_ROLE_CHANGE` if `:id` is the actor
(FR-U-06), and **422** `LAST_SUPER_ADMIN` if the change would leave no active super admin (FR-U-08).
Audit `user.roles_changed` with before/after role keys — the single most useful entry in the log.

### `POST /api/users/:id/status`

**200**. Suspension revokes every session for that user immediately. Rejects `SELF_SUSPEND` and
`LAST_SUPER_ADMIN` as above. Audit `user.suspended` / `user.reactivated`.

Note that `LOCKED` is not settable here: it is set and cleared by the login flow (FR-U-02). An admin
clears a lockout early by reactivating, which resets `failedLoginCount` and `lockedUntil`.

## Role and permission endpoints

| Method | Path | Permission | Notes |
|---|---|---|---|
| GET | `/api/roles` | `role.read` | Includes `userCount` per role (US-13). |
| POST | `/api/roles` | `role.write` | Custom role. `key` is uppercase snake, unique, immutable after creation. |
| GET | `/api/roles/:id` | `role.read` | With its permission grants and scopes. |
| PATCH | `/api/roles/:id` | `role.write` | `name`, `description`. **409** `SYSTEM_ROLE` if `isSystem` (FR-Z-08). |
| PUT | `/api/roles/:id/permissions` | `role.write` | Replaces the full grant set — see below. |
| DELETE | `/api/roles/:id` | `role.write` | **409** `ROLE_IN_USE` if any user holds it; **409** `SYSTEM_ROLE` for system roles. |
| GET | `/api/permissions` | `role.read` | The catalogue, grouped by module. Read-only — there is no write endpoint by design (D-02). |

### `PUT /api/roles/:id/permissions`

```json
{ "grants": [ { "key": "employee.read", "scope": "DEPARTMENT" },
              { "key": "leave.approve", "scope": "DEPARTMENT" } ] }
```

Replace-whole-set rather than add/remove deltas: the role editor is a matrix, and a delta API makes
a concurrent edit silently merge two admins' intentions. **200** returns the stored grants.
Rejected with **409** `SUPER_ADMIN_IMMUTABLE` for that role (FR-Z-08), **422** `UNKNOWN_PERMISSION`
for a key not in the catalogue. Audit `role.permissions_changed` with the full before/after grant
sets. Users holding the role are affected on their next request (FR-A-06) — no sign-out needed.

## Audit endpoint

### `GET /api/audit-logs` — `audit.read`

Filters: `actorUserId`, `action`, `entityType`, `entityId`, `from`, `to`, `q`.
Default sort `createdAt` desc. Paginated as above; `pageSize` capped at 100.

```json
{ "data": [ { "id": 9812, "actorLabel": "hr@company.com", "action": "user.roles_changed",
              "entityType": "User", "entityId": "12",
              "summary": "Roles changed: EMPLOYEE -> MANAGER",
              "before": { "roles": ["EMPLOYEE"] }, "after": { "roles": ["MANAGER"] },
              "ipAddress": "10.0.0.14", "createdAt": "2026-09-15T08:41:02Z" } ],
  "page": 1, "pageSize": 25, "total": 4193 }
```

`GET /api/audit-logs/:id` returns one entry with full before/after. There is no POST, PATCH, or
DELETE — the log is append-only and written only from inside transactions (FR-L-04).

## Authorization middleware and the device exemption

Route protection is applied per handler via `protectedRoute`, not by a catch-all middleware matcher.
Middleware handles only the coarse redirect for unauthenticated *page* navigations, so that a direct
API call gets 401 rather than an HTML redirect (acceptance criterion 5).

`/iclock/*` and `/api/device/*` are on an explicit allowlist and never see the session logic
(FR-A-13). The allowlist is a literal array in one file with a comment pointing here; it is not a
regex that could widen by accident. Everything not on it requires a session.

Public paths: `/login`, `/forgot-password`, `/reset-password`, `/api/auth/login`,
`/api/auth/password/forgot`, `/api/auth/password/reset`, plus static assets and the health check.

## Rate limiting

| Endpoint | Limit |
|---|---|
| `POST /api/auth/login` | 10 / 15 min per IP; 5 consecutive failures per account → 15 min lock |
| `POST /api/auth/password/forgot` | 5 / hour per IP, 3 / hour per email |
| `POST /api/auth/password/reset` | 10 / hour per IP |
| everything else | 300 / min per session |

In-process counters are enough for a single-container deployment. If the app is ever scaled to more
than one instance, this must move to Postgres or Redis — noted in OQ-109.

## Open questions

| ID | Question |
|---|---|
| OQ-101 | Auth.js vs hand-rolled sessions — this document assumes hand-rolled (README). Switching would rewrite the auth endpoints but not the RBAC helpers. |
| OQ-109 | Rate-limit storage if the app is ever run as more than one container. In-process counters silently multiply the limit by the instance count. |
| OQ-110 | Should `GET /api/auth/session` also return the nav items the user may see, or should the client derive them from `permissions`? Deriving keeps one source of truth; returning them keeps the client dumb. |
| OQ-111 | API versioning — `/api/...` now, or `/api/v1/...` from the start? Cheap now, painful later. |
