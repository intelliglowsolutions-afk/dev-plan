# 08 — Employee Self-Service — API Design

Conventions as in [01 api-design.md](../01-roles-permissions-auth/api-design.md).

## The shape of this feature's API

Almost every screen calls an **existing endpoint** from features 02–07 at `SELF` scope. This
document specifies only what is genuinely new:

1. A **`/api/me/*` façade** — thin, permission-simplifying wrappers where the general endpoint would
   force the client to know an employee id it should never handle.
2. The **dashboard aggregate**, which exists for performance (NFR-01).
3. **Profile change requests**, the one new mechanism.

Everything else is reused as-is. Where this document names an existing endpoint, that is a
statement that nothing new is being built.

## Why a `/me` façade at all

Two reasons, both about safety rather than convenience:

- **No employee id in the request.** `/api/me/attendance` derives the employee from the session.
  `/api/attendance/employee/:id` would work — scope enforcement would reject another id — but a
  client that never *sends* an id cannot leak or fumble one, and a bug that would otherwise expose
  someone else's data becomes impossible rather than caught.
- **A stable contract for the portal.** The admin endpoints are shaped for HR's needs and will grow
  filters and fields that the portal should not follow.

The façade delegates. It contains no logic of its own (D-01).

| Façade | Delegates to |
|---|---|
| `GET /api/me` | 02 `GET /api/employees/:id` (self) |
| `GET /api/me/attendance` | 04 `GET /api/attendance/employee/:id` |
| `GET /api/me/attendance/:dayId` | 04 `GET /api/attendance/days/:id` |
| `GET /api/me/leave/balances` | 06 `GET /api/leave/balances` |
| `GET /api/me/leave/requests` | 06 `GET /api/leave/requests` |
| `GET /api/me/payslips` | 07 (already specified as `/api/me/payslips`) |
| `GET /api/me/documents` | 02 `GET /api/employees/:id/documents` |
| `GET /api/me/notifications` | 05 `GET /api/notifications` (already self-only) |

Write operations are **not** wrapped: requesting leave posts to 06's endpoint, requesting an
attendance correction posts to 04's. Those endpoints already derive the employee from the session
for self-scoped actors, and duplicating them would create two paths into the same state.

## `GET /api/me/dashboard` — the aggregate

One request, assembled server-side (FR-D-01, NFR-01). Sections the employee has no permission for
are **absent from the payload**, not present and empty (FR-D-07).

```json
{ "employee": { "firstName": "Ayesha", "photoUrl": "/api/employees/12/photo",
                "positionTitle": "Accountant", "departmentName": "Finance" },
  "actions": [
    { "kind": "LEAVE_REJECTED", "priority": 1,
      "title": "Your leave request was declined",
      "detail": "1–7 October — \"Two others are already off that week\"",
      "linkPath": "/portal/leave/requests/88" },
    { "kind": "DOCUMENT_EXPIRING", "priority": 2,
      "title": "Your visa expires in 21 days",
      "linkPath": "/portal/documents" },
    { "kind": "APPROVALS_WAITING", "priority": 1,
      "title": "3 leave requests need you",
      "linkPath": "/leave/approvals" } ],
  "facts": {
    "leave": { "typeName": "Annual leave", "available": 11.5,
               "expiring": { "days": 3, "on": "2027-03-31" } },
    "attendance": { "date": "2026-09-15", "statusLabel": "Signed in at 08:58",
                    "needsAttention": false },
    "payslip": { "periodName": "August 2026", "publishedAt": "2026-08-31T00:00:00Z" },
    "nextHoliday": { "name": "National Day", "date": "2026-10-06" } },
  "unreadNotifications": 2 }
```

Three things this payload deliberately does **not** contain:

- **No pay figure.** The latest payslip is named by period only (FR-Y-06, 07 D-06).
- **No internal status codes.** `statusLabel` is the rendered sentence; the client does not translate
  (D-05). Keeping the translation server-side means one vocabulary, not one per client.
- **No pending-computation notice.** If 04 has not computed today yet, `attendance` is simply absent
  and the card shows its own quiet state (D-06).

`APPROVALS_WAITING` is the only manager-facing item, and it links out of the portal into feature 06
(README, "Managers").

## Profile

| Method | Path | Permission | Notes |
|---|---|---|---|
| GET | `/api/me/profile` | `employee.read` (SELF) | Fields with their policy |
| PATCH | `/api/me/profile` | `employee.read` (SELF) | `SELF_EDIT` fields only |
| GET / POST / PATCH / DELETE | `/api/me/emergency-contacts` | `employee.read` (SELF) | Fully self-managed |
| POST | `/api/me/photo` | `employee.read` (SELF) | Where configured |

`GET /api/me/profile` returns each field with its policy, label, and help text, so the UI is
generated from the configuration rather than hard-coded — the same approach as 03's settings screen
and 05's preferences screen:

```json
{ "fields": [
    { "key": "personalPhone", "label": "Mobile number", "value": "+92 300 1234567",
      "policy": "SELF_EDIT" },
    { "key": "addressLine1", "label": "Home address", "value": "12 Mall Road",
      "policy": "REQUEST",
      "pendingRequest": { "id": 7, "proposedValue": "44 Jail Road",
                          "requestedAt": "2026-09-12T09:00:00Z" } },
    { "key": "employeeCode", "label": "Employee number", "value": "1042",
      "policy": "READ_ONLY",
      "helpText": "This is the number you use at the attendance terminal." } ] }
```

`HIDDEN` fields do not appear. Bank details and compensation are not in the catalogue at all
(FR-P-10), so no response, policy, or configuration can surface them here.

`PATCH /api/me/profile` accepts only `SELF_EDIT` fields. A `REQUEST` field in the body returns
**422** `REQUIRES_APPROVAL` naming the field and pointing at the request endpoint — rather than
silently ignoring it, which would look like a successful save.

## Change requests

| Method | Path | Permission |
|---|---|---|
| POST | `/api/me/change-requests` | `profile.change_request` |
| GET | `/api/me/change-requests` | `profile.change_request` |
| POST | `/api/me/change-requests/:id/withdraw` | requester |
| GET | `/api/change-requests` | `profile.change_request.decide` |
| POST | `/api/change-requests/:id/approve` | `profile.change_request.decide` |
| POST | `/api/change-requests/:id/reject` | `profile.change_request.decide` |

### `POST /api/me/change-requests`

```json
{ "fieldKey": "addressLine1", "proposedValue": "44 Jail Road",
  "note": "Moved last month", "attachmentDocumentId": null }
```

**201**. Validates that `fieldKey` is in the code-declared catalogue and its policy is `REQUEST`,
that the value passes the field's declared validator, and that no pending request exists for it
(FR-P-05). Snapshots `currentValue`. Notifies HR.

**409** `PENDING_EXISTS` names the existing request rather than creating a second.
**422** `NOT_REQUESTABLE` for a `SELF_EDIT` field (with a pointer to `PATCH`), a `READ_ONLY` field,
or any key not in the catalogue — which is the same response, deliberately: an unknown key and a
forbidden key are indistinguishable from outside.

### `POST /api/change-requests/:id/approve`

Applies the change **through feature 02's normal update path**, inside one transaction with the
request's status change — so 02's validation, its audit entry, and any downstream effect all happen
exactly as they would for an HR-entered edit (FR-P-04). The request records the asking; 02's audit
records the change (`data-model.md`).

If 02's validation rejects the value — a duplicate email, say — the approval fails with that error
rather than marking the request approved and silently not applying it. A request that says
"approved" over an unchanged record is the worst outcome available here.

Rejection requires a reason, shown to the employee (FR-P-06).

## Reused endpoints — the honest list

For the build's benefit, the portal's screens and what they call:

| Screen | Endpoints | New? |
|---|---|---|
| Dashboard | `GET /api/me/dashboard` | ✅ |
| My attendance | `GET /api/me/attendance`, `GET /api/me/attendance/:id` | façade |
| Report a problem | `POST /api/attendance/corrections` (04) | — |
| My leave | `GET /api/me/leave/balances`, `.../ledger` (06) | façade |
| Request leave | `POST /api/leave/requests/cost`, `POST /api/leave/requests` (06) | — |
| Cancel leave | `POST /api/leave/requests/:id/cancel` (06) | — |
| Team calendar | `GET /api/leave/calendar` (06) | — |
| My payslips | `GET /api/me/payslips`, `.../pdf` (07) | — |
| My profile | `GET/PATCH /api/me/profile` | ✅ |
| Change requests | `/api/me/change-requests` | ✅ |
| My documents | `GET /api/me/documents` | façade |
| Directory | `GET /api/employees/lookup`, `GET /api/org-chart` (02) | — |
| Notifications | `/api/notifications/*` (05) | — |

Three new endpoint groups and four façades, for an entire portal. If that list grows during the
build, D-01 is being broken and the new capability belongs in the feature that owns the data.

## Directory scope

The directory is the one place the portal shows other people, so its shape is stated explicitly:

`GET /api/employees/lookup` under `employee.read` at `SELF` scope returns **only the actor** — which
is correct for pickers and wrong for a directory. The directory therefore uses a dedicated
projection:

`GET /api/directory` — `department.read` (which everyone holds at `ALL`, per 02's permission table,
precisely so the org chart works) — returning name, position, department, work email, work phone,
and photo. Never personal contact details, never anything sensitive, never employment status beyond
"current employee" (FR-X-04).

This belongs in feature 02 rather than here, since it is a projection of 02's data, and is noted as
an addition to that feature's API when this one is built.

## Open questions

| ID | Question |
|---|---|
| OQ-810 | Should the dashboard aggregate be cached per user for a short window? It is the most-requested endpoint in the system once the portal ships. A 30-second cache would be invisible to users and would need invalidation on every action they take. |
| OQ-811 | `GET /api/directory` is specified here but belongs to feature 02. Add it there retrospectively, or leave it as this feature's one exception to "no new domain endpoints"? Proposed: add it to 02, and note it in 02's files when building. |
| OQ-807 | Document upload would add `POST /api/me/documents` against an HR request — deliberately not specified until there is a review queue for the results. |
