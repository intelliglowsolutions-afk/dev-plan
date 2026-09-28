# 01 — Roles, Permissions & Auth

**Priority:** Must-have · **Build order:** 1 of 11 · **Status:** planned (Step 3)

## Purpose

Establish who may sign in, what each signed-in person is allowed to do, and a durable record of
what they did. Every other feature in the system depends on this one: each module declares its own
permission keys, checks them on every server entry point, and writes audit entries for changes.

## Scope

**In scope**

- Email + password sign-in, sign-out, and server-side sessions.
- Password lifecycle: admin-issued invite, forced first-login change, self-service reset by email,
  password policy, failed-attempt lockout.
- RBAC: permissions (fixed catalogue, seeded), roles (system + custom), role→permission assignment
  with a data scope, user→role assignment.
- Authorization enforcement helpers used by every other module's server code.
- Audit log: who did what to which record, with before/after values.
- Admin UI for users, roles, and the audit log.

**Out of scope (owned elsewhere)**

- The `Employee` record itself, departments, and reporting lines → [02 Employee Management](../02-employee-management/README.md).
  This feature stores only the `User` account and its optional link to an employee.
- Company profile, system config, device registration → [03 Admin & Settings](../03-admin-settings/README.md).
- Sending the invite / reset emails → [05 Notifications](../05-notifications/README.md) owns delivery.
  Until 05 exists, feature 01 writes the message to a local outbox table and logs the link.
- The device push endpoint (`/iclock/*`), which is authenticated by the device comm key, not by a
  user session. It is explicitly exempt from session auth → [04 Attendance](../04-attendance-tracking/README.md).

## Files

| File | Contents |
|---|---|
| [requirements.md](./requirements.md) | User stories, functional and non-functional requirements, the role model, the permission catalogue |
| [data-model.md](./data-model.md) | Prisma models, enums, indexes, seed data, migration notes |
| [api-design.md](./api-design.md) | Route handlers, request/response shapes, the server-side authorization helpers |
| [ui-ux.md](./ui-ux.md) | Screens, navigation gating, states, and copy |

## Key decisions taken in this plan

| # | Decision | Rationale |
|---|---|---|
| D-01 | `User` (login account) is a separate model from `Employee`, linked optionally 1:1. | Some accounts have no employee record (the initial super admin, an external auditor); some employees never log in (factory staff who only punch at the terminal). |
| D-02 | Permissions are a **fixed, seeded catalogue** of `module.action` keys. Only roles are user-editable. | Permission keys are referenced in code. Letting users invent keys produces permissions nothing checks. |
| D-03 | Each role→permission grant carries a **scope**: `ALL`, `DEPARTMENT`, or `SELF`. | An HRM needs "a manager may read attendance — of their own team". Without a scope, the same permission key would have to be triplicated. |
| D-04 | Sessions are **stored in the database**, in an `httpOnly` cookie — not stateless JWTs. **Confirmed 2026-09-18 and now implemented via Auth.js** (OQ-101): its database-session strategy, through a Prisma adapter, satisfies this. **The JWT session strategy must not be used** — it would reintroduce exactly the delay this decision exists to prevent. | Revocation must be immediate: deactivating a user or changing a role has to take effect on the next request, not at token expiry. |
| D-11 | **Auth.js (NextAuth) v5 with the Credentials provider and a Prisma adapter**, replacing the hand-rolled sessions this plan originally proposed (OQ-101, answered by the user). | Their call. It brings a maintained implementation of the session cookie, CSRF, and callback plumbing, and leaves room for SSO later (OQ-104) without another rewrite. The cost is a framework-version dependency — Next.js 16 compatibility is unverified and `AGENTS.md` warns this Next.js differs from training data, so **verify it works before building on it**. |
| D-05 | Authorization is enforced **server-side on every entry point**. The UI only hides what the user cannot use. | Hiding a nav item is not a security control. |
| D-06 | The audit log is **append-only** and written in the same transaction as the change it records. | An audit entry that can be missing or edited is not evidence. |
| D-07 | Four seeded system roles: `SUPER_ADMIN`, `HR_ADMIN`, `MANAGER`, `EMPLOYEE`. Custom roles allowed. | Covers the common org without forcing setup work on day one. |

## Dependencies

- **Depends on:** nothing. This is the first feature built.
- **Depended on by:** all ten other features — each contributes permission keys to the catalogue and
  calls the authorization helpers and the audit writer.
- **Soft dependency:** 02 Employee Management, for the `User.employeeId` link and for `DEPARTMENT`
  scope resolution (which needs departments and reporting lines). Feature 01 ships with the link
  column nullable and `DEPARTMENT` scope resolving to an empty set until 02 lands.

## Open questions

| ID | Question | Why it matters | Proposed default |
|---|---|---|---|
| OQ-101 | Auth library: Auth.js (NextAuth) v5, or a hand-rolled session on top of `cookies()`? `.env.example` already carries `NEXTAUTH_SECRET`/`NEXTAUTH_URL`, but Next.js 16 compatibility is unverified (see `AGENTS.md`). | Shapes every route handler in this feature. | Hand-rolled: ~200 lines, no framework-version risk, and D-04 already rules out the JWT path that is Auth.js's main draw. Revisit if SSO is wanted (OQ-104). |
| OQ-102 | Is multi-factor authentication required for admin accounts in this version? | TOTP adds two models, an enrolment screen, and recovery codes. | Not in v1; the data model leaves room for it. |
| OQ-103 | Password policy specifics — minimum length, complexity, expiry, reuse history. | Affects validation and the change-password UI. | 12 characters minimum, no composition rules, no forced expiry, last 3 passwords blocked. |
| OQ-104 | Will staff sign in with company Google/Microsoft accounts (SSO), now or later? | Changes D-01 and the login screen; retrofitting SSO is far cheaper if planned for. | Password-only in v1. |
| OQ-105 | Audit log retention — keep forever, or purge after N months? | The table grows with every write in the system. | Keep 24 months, then archive; needs a decision before 11 Reports. |
| OQ-106 | Should a user be able to hold more than one role? | A `MANAGER` who is also the `HR_ADMIN` is common in a small company. | Yes — many-to-many, permissions union, most permissive scope wins. |
| OQ-107 | Self-service portal (08) users: same `User` table and login screen, or a separate employee portal login? | Affects routing and session shape. | Same table, same login; landing page differs by permission. |
