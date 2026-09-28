# 08 — Employee Self-Service — Data Model

This is the shortest data-model document in the plan, and that is the point. If it grew, decision
D-01 would have been broken.

## What this feature adds

Two models: profile change requests, and the configuration of which fields are self-editable.

## What this feature reads

Everything else, through the owning features' endpoints at `SELF` scope:

| Data | Owner | Read via |
|---|---|---|
| Employee record, documents, directory, org chart | 02 | `/api/employees/*`, `/api/org-chart` |
| Date and number formats, holidays, timezone | 03 | settings, `/api/holidays` |
| Attendance days, punches, corrections | 04 | `/api/attendance/*` |
| Notifications and preferences | 05 | `/api/notifications/*` |
| Leave balances, requests, calendar | 06 | `/api/leave/*` |
| Published payslips | 07 | `/api/me/payslips` |

No table in this list is written to by this feature, and no figure from it is recomputed here
(NFR-06).

## Enums

```prisma
enum ChangeRequestStatus {
  PENDING
  APPROVED
  REJECTED
  WITHDRAWN
}

enum SelfServiceFieldPolicy {
  SELF_EDIT     // the employee changes it directly, effective immediately
  REQUEST       // the employee proposes it; HR decides
  READ_ONLY     // visible, not changeable through the portal
  HIDDEN        // not shown in the portal at all
}
```

`READ_ONLY` and `HIDDEN` are distinct on purpose. An employee should see their employee code and
their job title without being able to change them (`READ_ONLY`); they should never see an internal
note field at all (`HIDDEN`).

## Models

```prisma
model ProfileChangeRequest {
  id           Int                 @id @default(autoincrement())
  employeeId   Int                 @map("employee_id")

  // Dotted path into the Employee model: "personalPhone", "addressLine1", "maritalStatus".
  // Validated against the field-policy catalogue, never used to build a query directly.
  fieldKey     String              @map("field_key")
  currentValue String?             @map("current_value")   // as at submission, for context
  proposedValue String?            @map("proposed_value")
  note         String?

  status       ChangeRequestStatus @default(PENDING)
  requestedById Int?               @map("requested_by_id")
  requestedAt  DateTime            @default(now()) @map("requested_at")
  decidedById  Int?                @map("decided_by_id")
  decidedAt    DateTime?           @map("decided_at")
  decisionNote String?             @map("decision_note")
  appliedAt    DateTime?           @map("applied_at")

  attachmentDocumentId Int?        @map("attachment_document_id")

  employee     Employee            @relation(fields: [employeeId], references: [id], onDelete: Cascade)

  // One pending request per field per employee (FR-P-05). Partial index, added in the
  // migration — the same pattern feature 06 needed for approved leave days.
  @@index([employeeId, status])
  @@index([status, requestedAt])
  @@map("profile_change_requests")
}

model SelfServiceFieldConfig {
  fieldKey  String                 @id @map("field_key")
  policy    SelfServiceFieldPolicy
  label     String                                    // plain-language, shown in the portal
  helpText  String?                @map("help_text")
  updatedById Int?                 @map("updated_by_id")
  updatedAt DateTime               @updatedAt @map("updated_at")

  @@map("self_service_field_config")
}
```

## Design notes

**Why `fieldKey` is a string, and why that is safe.** It looks like an injection risk, and it would
be if it were interpolated into a query. It is not: the key is looked up in a **code-declared
catalogue** of permitted fields — the same pattern as permissions (01), settings (03), and
notification types (05) — which supplies the Prisma field name, the type, the validator, and the
label. A key not in the catalogue is rejected before anything touches the database.

This is the fourth appearance of that pattern. By now it is the house answer to "user-referenced
identifiers into code-defined things", and using anything else here would be the surprising choice.

**Why `SelfServiceFieldConfig` is a table when the catalogue is code.** The catalogue defines what
*can* be configured and how each field is validated; the table records what the company *has*
chosen (OQ-801). Defaults come from the catalogue, so rows exist only for deliberate changes —
exactly how 03's settings and 05's notification preferences work.

**Why `currentValue` is snapshotted on the request.** HR deciding a request a week later needs to see
what it was when the employee asked, not what it is now — which may have changed by another route.
Without it, "change my phone from X to Y" becomes unreviewable once X is gone.

**Why there is no `ProfileChangeRequest` → applied-value audit.** There does not need to be: applying
a change writes through feature 02's normal update path, which is already audited with before and
after (02 FR-E-10). The request records the asking; 01's audit log records the change. Duplicating
it would create two histories that can disagree.

## Volume

| Table | Rows/year at 200 employees |
|---|---|
| `profile_change_requests` | ~100 |
| `self_service_field_config` | ~20, written rarely |

Negligible. This feature's cost is entirely in interface work, not in data.

## Seed and migration notes

Migration `self_service`:

1. Create the two tables.
2. Add the partial unique index for FR-P-05:
   ```sql
   CREATE UNIQUE INDEX profile_change_requests_one_pending_per_field
   ON profile_change_requests (employee_id, field_key)
   WHERE status = 'PENDING';
   ```
   This one *is* expressible as a partial index, unlike feature 06's equivalent, because the
   predicate is on the row's own column.
3. Seed `SelfServiceFieldConfig` from the proposed defaults in OQ-801:
   - `SELF_EDIT` — personal phone, personal email, emergency contacts, photo.
   - `REQUEST` — name, address, marital status, national ID.
   - `READ_ONLY` — employee code, job, department, manager, start date, work email.
   - **Never present in the catalogue at all** — bank details and compensation (FR-P-10). Their
     absence from the catalogue is the enforcement; there is no configuration that could enable
     them, which is stronger than a policy value that someone could change.
4. Seed permission keys `profile.change_request` (everyone, SELF) and
   `profile.change_request.decide` (HR, ALL).
5. Seed 05's notification types: `profile.change_requested` (to HR) and `profile.change_decided`
   (to the employee).

No new settings in 03's catalogue; the field config is this feature's own configuration surface.

## Open questions

| ID | Question |
|---|---|
| OQ-801 | The seeded field policies are proposals. They are a company decision about who owns which facts, not a technical one. |
| OQ-807 | Free-form document upload would add an `EmployeeDocumentRequest` model — HR asks for a document, the employee supplies it, someone reviews it. Deliberately not modelled until there is a queue to put the results in. |
| OQ-809 | Should a rejected change request keep the proposed value visible to the employee after rejection? Useful for re-submitting correctly; also preserves a value HR declined to record. Proposed: keep it, since the employee supplied it and already knows it. |
