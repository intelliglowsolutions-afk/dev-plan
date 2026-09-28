# 01 — Roles, Permissions & Auth — Data Model

Conventions follow the existing `prisma/schema.prisma`: `camelCase` fields mapped to `snake_case`
columns, `@@map` to plural snake_case tables, `Int @id @default(autoincrement())` primary keys,
`createdAt`/`updatedAt` on mutable models. All timestamps are UTC.

> **Revised 2026-09-18 — multi-tenant (OQ-301).** [MULTI-TENANCY.md](../MULTI-TENANCY.md) governs.
> For this feature specifically:
> - `User`, `Role`, `UserRole`, `Session`, `PasswordResetToken` and `AuditLog` all gain `tenantId`.
> - **`User.email` stays globally unique** (D-T-03) so login needs no tenant hint. `Role.key` becomes
>   unique per tenant — each tenant gets its own copy of the four system roles.
> - `Permission` is **not** tenant-owned: the catalogue is code-declared and global (D-T-02).
> - `SUPER_ADMIN` becomes the **tenant** super admin. The **platform operator** is a separate
>   concept that sees tenants, provisioning and health but no employee, payroll or performance data
>   (D-T-04).
> - The session carries `tenantId`, and it is what sets `app.tenant_id` for RLS on every request.
> - **The `Session` model below is replaced by Auth.js's adapter schema** (OQ-101) — reconcile the
>   two rather than shipping both.
> - Audit rows are retained, not swept (OQ-1002).

## Entity overview

```
Employee (feature 02) ──0..1───0..1── User ──*─── UserRole ───*── Role ──*── RolePermission ──*── Permission
                                       │                                          (scope)
                                       ├──*── Session
                                       ├──*── PasswordResetToken
                                       └──*── AuditLog (as actor)
```

- A `User` is a login account. An `Employee` is a person on the payroll. Most people are both, but
  neither requires the other (README D-01).
- `Permission` rows are seeded from code and never edited by users (D-02).
- The scope lives on `RolePermission`, not on `Permission` or `Role` (D-03).

## Enums

```prisma
enum UserStatus {
  INVITED    // created by an admin, has not set their own password yet
  ACTIVE     // may sign in
  SUSPENDED  // deactivated by an admin; only an admin can restore
  LOCKED     // too many failed attempts; clears itself when lockedUntil passes
}

enum PermissionScope {
  ALL         // every record in the company
  DEPARTMENT  // the actor's own reports, direct and transitive, plus the actor
  SELF        // only the actor's own records
}
```

`AuditLog.action` is a plain `String`, not an enum: every future feature adds actions, and an enum
would force a migration per feature. Values are constrained in code to a constant union type.

## Models

```prisma
model User {
  id                 Int        @id @default(autoincrement())
  email              String     @unique            // stored lower-cased
  passwordHash       String     @map("password_hash")
  status             UserStatus @default(INVITED)
  mustChangePassword Boolean    @default(true) @map("must_change_password")

  // Optional 1:1 link to the HR record. Nullable: the initial super admin has no employee row.
  employeeId         Int?       @unique @map("employee_id")

  lastLoginAt        DateTime?  @map("last_login_at")
  failedLoginCount   Int        @default(0) @map("failed_login_count")
  lockedUntil        DateTime?  @map("locked_until")

  createdById        Int?       @map("created_by_id")
  createdAt          DateTime   @default(now()) @map("created_at")
  updatedAt          DateTime   @updatedAt @map("updated_at")

  employee           Employee?  @relation(fields: [employeeId], references: [id], onDelete: SetNull)
  createdBy          User?      @relation("UserCreatedBy", fields: [createdById], references: [id])
  createdUsers       User[]     @relation("UserCreatedBy")

  roles              UserRole[]
  sessions           Session[]
  passwordResets     PasswordResetToken[]
  auditEntries       AuditLog[]

  @@index([status])
  @@map("users")
}

model Role {
  id          Int      @id @default(autoincrement())
  key         String   @unique                     // SUPER_ADMIN, HR_ADMIN, custom: SHIFT_SUPERVISOR
  name        String
  description String?
  isSystem    Boolean  @default(false) @map("is_system")
  createdAt   DateTime @default(now()) @map("created_at")
  updatedAt   DateTime @updatedAt @map("updated_at")

  users       UserRole[]
  permissions RolePermission[]

  @@map("roles")
}

model Permission {
  id          Int      @id @default(autoincrement())
  key         String   @unique                     // "employee.read"
  module      String                               // "employee"  — for grouping in the role editor
  action      String                               // "read"
  description String
  createdAt   DateTime @default(now()) @map("created_at")

  roles       RolePermission[]

  @@index([module])
  @@map("permissions")
}

model UserRole {
  userId       Int      @map("user_id")
  roleId       Int      @map("role_id")
  assignedById Int?     @map("assigned_by_id")
  assignedAt   DateTime @default(now()) @map("assigned_at")

  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  role         Role     @relation(fields: [roleId], references: [id], onDelete: Restrict)

  @@id([userId, roleId])
  @@index([roleId])
  @@map("user_roles")
}

model RolePermission {
  roleId       Int             @map("role_id")
  permissionId Int             @map("permission_id")
  scope        PermissionScope @default(SELF)

  role         Role            @relation(fields: [roleId], references: [id], onDelete: Cascade)
  permission   Permission      @relation(fields: [permissionId], references: [id], onDelete: Cascade)

  @@id([roleId, permissionId])
  @@index([permissionId])
  @@map("role_permissions")
}

model Session {
  id         Int       @id @default(autoincrement())
  userId     Int       @map("user_id")
  // SHA-256 of the opaque cookie token. The token itself is never stored.
  tokenHash  String    @unique @map("token_hash")
  expiresAt  DateTime  @map("expires_at")          // absolute expiry (created + 30d)
  lastUsedAt DateTime  @default(now()) @map("last_used_at")  // idle expiry is lastUsedAt + 12h
  revokedAt  DateTime? @map("revoked_at")
  ipAddress  String?   @map("ip_address")
  userAgent  String?   @map("user_agent")
  createdAt  DateTime  @default(now()) @map("created_at")

  user       User      @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@index([expiresAt])
  @@map("sessions")
}

model PasswordResetToken {
  id        Int       @id @default(autoincrement())
  userId    Int       @map("user_id")
  tokenHash String    @unique @map("token_hash")   // SHA-256, same as Session
  expiresAt DateTime  @map("expires_at")           // created + 60 min
  usedAt    DateTime? @map("used_at")
  createdAt DateTime  @default(now()) @map("created_at")

  user      User      @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@map("password_reset_tokens")
}

model AuditLog {
  id          Int      @id @default(autoincrement())
  // Null for system-initiated actions (device ingestion, scheduled jobs, the seed).
  actorUserId Int?     @map("actor_user_id")
  // Denormalised so the entry still reads correctly if the account is later renamed.
  actorLabel  String   @map("actor_label")

  action      String                               // "user.suspend", "auth.login_failed"
  entityType  String   @map("entity_type")         // "User", "Role", "Employee"
  entityId    String?  @map("entity_id")           // String: some entities are not Int-keyed
  summary     String?                              // one line for the list view

  before      Json?                                // changed fields only, sensitive values stripped
  after       Json?

  ipAddress   String?  @map("ip_address")
  userAgent   String?  @map("user_agent")
  createdAt   DateTime @default(now()) @map("created_at")

  actor       User?    @relation(fields: [actorUserId], references: [id], onDelete: SetNull)

  @@index([createdAt])
  @@index([entityType, entityId])
  @@index([actorUserId, createdAt])
  @@index([action])
  @@map("audit_logs")
}
```

## Change to an existing model

`Employee` (already in `prisma/schema.prisma`) gains the back-relation only — no column:

```prisma
model Employee {
  // ... existing fields unchanged
  user User?
}
```

## Design notes

**Why `tokenHash` and not the token.** A database dump, a backup, or a leaked read-only query
otherwise hands the attacker live sessions. Hashing means a leak yields nothing usable. SHA-256 is
adequate here — unlike a password, the token is 256 bits of entropy, so there is nothing to brute
force and no need for a slow hash.

**Why `actorLabel` is denormalised.** `actorUserId` is `SetNull` on delete, and users can change
their email. Two years later the log must still say who did it.

**Why `entityId` is a `String`.** Most entities are `Int`-keyed today, but composite keys
(`RolePermission`) and future non-integer keys need to be recordable. Stringifying at write time
keeps one column instead of several nullable ones.

**Why `before`/`after` hold only changed fields.** Storing whole records makes the table grow fast
and makes a diff harder to read, not easier. FR-L-03's redaction list is applied at write time, not
read time.

**Why `onDelete: Restrict` on `UserRole.role`.** Deleting a role that is still assigned would
silently strip access (FR-Z-09). The database refuses; the API returns a clear message first.

**Cascade summary.** `Session`, `PasswordResetToken`, and `UserRole` cascade from `User` — but users
are never hard-deleted (FR-U-05), so in practice these cascades only fire in tests and teardown.

## Indexes and expected volume

| Table | Rows (200-employee company, 2 years) | Hot query |
|---|---|---|
| `users` | ~250 | by `email` on sign-in — unique index |
| `roles`, `permissions`, `role_permissions` | <100, ~120, ~500 | loaded per request via the user's roles |
| `sessions` | ~2 000 live, pruned | by `tokenHash` on every request — unique index |
| `audit_logs` | ~500 000+ | by `createdAt` desc, filtered by actor/entity/action |

`audit_logs` is the only table that grows without bound, hence four indexes and OQ-105 on retention.
Session rows past `expiresAt` are deleted by a daily job; that job's own writes are not audited.

## Effective-permissions query

Loaded once per request (NFR-01) and cached for the request's lifetime:

```sql
SELECT p.key, MIN(rp.scope_rank) AS scope   -- ALL=0, DEPARTMENT=1, SELF=2; lowest wins (FR-Z-03)
FROM user_roles ur
JOIN role_permissions rp ON rp.role_id = ur.role_id
JOIN permissions p       ON p.id = rp.permission_id
WHERE ur.user_id = $1
GROUP BY p.key;
```

For a user holding `SUPER_ADMIN`, the helper short-circuits and returns the whole catalogue at `ALL`
without this query (requirements.md, "Role model").

## Seed data

Run by `prisma/seed.ts`, idempotent (NFR-08): upsert by `key`, never delete.

1. **Permissions** — the full catalogue from `requirements.md`, upserted by `key`. Keys removed from
   code are reported but not deleted, so a typo does not silently revoke access.
2. **Roles** — the four system roles with `isSystem: true`, upserted by `key`. Their `name` and
   `description` refresh; existing permission grants are *not* overwritten, since FR-Z-08 allows
   admins to edit three of them.
3. **Role permissions** — the default grid from `requirements.md`, inserted only for grants that do
   not already exist. `SUPER_ADMIN` gets no rows: its access is computed.
4. **Initial super admin** — created only if no user holds `SUPER_ADMIN`. Email and password come
   from `SEED_ADMIN_EMAIL` and `SEED_ADMIN_PASSWORD`; the run fails loudly if they are unset rather
   than falling back to a default password. Created with `mustChangePassword: true`.

New environment variables for `.env.example`:

```
SEED_ADMIN_EMAIL="admin@example.com"
SEED_ADMIN_PASSWORD="set-me-before-seeding"
SESSION_IDLE_MINUTES=720
SESSION_ABSOLUTE_DAYS=30
```

## Migration notes

- One migration, `add_auth_and_rbac`, additive only. It touches no existing table except adding
  `Employee`'s back-relation, which generates no SQL.
- Existing `employees`, `devices`, and `attendance` rows are unaffected; the first migration
  (`20260911065054_init`) stays as is.
- Seed runs after migrate. Order matters: permissions → roles → grants → admin user.
- `package.json` gains a `prisma.seed` entry and a `db:seed` script; `prisma.config.ts` may need the
  seed path declared — verify against the installed Prisma 6.19 behaviour before writing it.

## Open questions

| ID | Question |
|---|---|
| OQ-102 | If MFA lands later, it adds `UserMfaFactor` (userId, type, secretHash, confirmedAt) and `MfaRecoveryCode`. Nothing in this model blocks it. |
| OQ-103 | A password-reuse check (default: last 3) needs a `PasswordHistory` table. Included only if OQ-103 is answered "yes". |
| OQ-105 | Audit retention drives whether `audit_logs` needs partitioning by month. Below ~5 M rows, plain indexes are fine. |
| OQ-108 | `Session` currently has no "remember this device" concept. If wanted, it needs a device fingerprint column and a longer absolute expiry for trusted devices. |
