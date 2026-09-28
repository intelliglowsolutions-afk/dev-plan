# 11 — Reports & Analytics

**Priority:** Should-have · **Build order:** 11 of 11 · **Status:** planned (Step 3)

## Purpose

Answer the questions that span modules — how much overtime did Operations work last quarter, what is
our absence rate, what did payroll cost by department, how long does hiring take — without becoming
a way to see things you are not allowed to see.

## The two problems

**1. Aggregation leaks.** This is the problem specific to reporting, and it is easy to miss.

A manager cannot see individual salaries. But a report showing *average salary by department* for a
department of two people is a salary disclosure, dressed as a statistic. Filters make it worse: an
aggregate narrowed by department, then grade, then start date, eventually describes one person.

Every other feature protects data by row. A reporting feature must also protect it by **inference**
(D-03), and that is a different discipline.

**2. Reports must agree with the system.** A report saying an employee worked 168 hours when their
payslip says 164 destroys confidence in both. There is only one defence: reports must not compute
anything. They read what the owning feature already produced (D-01).

Nearly every reporting feature in every system fails this eventually, because reimplementing a
calculation in SQL is faster than threading a helper through — and then the two drift silently.

## Scope

**In scope**

- A **catalogue of defined reports**, declared in code, each with parameters, columns, and a
  visibility rule.
- **Running** them with filters, and paging through the results.
- **Charts** for those that benefit — a small set, chosen deliberately.
- **Dashboards**: role-appropriate compositions of the same reports.
- **Exports** to CSV and PDF, audited as disclosures.
- **Saved views**: a user's filters, named and reusable.
- **Scheduled delivery**: a link on a schedule, never data in an email.
- **Suppression** of aggregates over small populations.

**Out of scope**

- **An ad-hoc query builder.** Letting users compose arbitrary queries is, in a system with
  row-level and field-level permissions, a permission-bypass engine (D-04). If it is wanted later, it
  needs its own security design, not an afterthought.
- **A data warehouse or BI tool integration.** Direct database access by an external tool bypasses
  every permission rule in features 01–10 (OQ-1105).
- **Any business logic.** Overtime hours come from 04, leave balances from 06, pay figures from 07.
  This feature computes none of them (D-01).
- **Predictive analytics, attrition scoring, flight-risk modelling.** Same stance as 09 D-04 and
  10 D-07, extended: the system does not produce judgements about individuals (D-09).

## Files

| File | Contents |
|---|---|
| [requirements.md](./requirements.md) | User stories, functional and non-functional requirements, the report catalogue, permission keys |
| [data-model.md](./data-model.md) | The few models, what it reads, and how historical accuracy is preserved |
| [api-design.md](./api-design.md) | The report contract, running, exporting, scheduling |
| [ui-ux.md](./ui-ux.md) | Report viewer, dashboards, charts, states and copy |

## Key decisions taken in this plan

| # | Decision | Rationale |
|---|---|---|
| D-01 | **Reports compute nothing.** Every figure comes from the owning feature's own query or helper. A report that needs a number the owning feature cannot supply is a request to that feature, not a new calculation here. | The only durable defence against reports and payslips disagreeing. |
| D-02 | **Every report inherits the actor's scope and the owning feature's permissions**, enforced in the query. There is no reporting permission that grants a wider view than the underlying module. | Otherwise `report.read` quietly becomes the most powerful permission in the system. |
| D-03 | **Aggregates over sensitive measures are suppressed below a minimum population** (default 5). | Aggregation leaks (see above). A threshold is crude but effective, and its absence is indefensible once pointed out. |
| D-04 | **Reports are declared in code**, with fixed parameters and columns. No user-composed queries in v1. | A query builder over this data model is a permission-bypass engine. This is the fifth code-declared catalogue in the plan, after permissions, settings, notification types, and self-service fields. |
| D-05 | **Every result states its as-of time, its filters, and its sources.** | A figure with no provenance cannot be reconciled, and reconciling figures is most of what reporting is for. |
| D-06 | **Historical reports use historical structure.** "Headcount by department for March" uses the department each person was in during March (02's assignment history), not today's. | The default — joining to the current department — is silently wrong for every restructure, and nobody notices until an auditor does. |
| D-07 | **Query live; no pre-aggregation in v1.** With indexes, this system's volumes are small. | A materialised layer adds staleness, invalidation, and a second source of truth. Revisit at scale, not before (OQ-1104). |
| D-08 | **Scheduled reports deliver a link, never the data.** | 05 D-06. An emailed payroll summary is a pay disclosure with no access control at all. |
| D-09 | **No scoring, ranking, or prediction about individuals.** Aggregates and lists, not judgements. | Consistent with 09 D-04 and 10 D-07. An HR system that ranks people by "risk" is one nobody should have to work under. |
| D-10 | **Exports are audited as disclosures**, with the filters used and the row count. | Already required by 02 FR-I-07 and 07 FR-E-05; this feature is where it becomes systematic. |

## Dependencies

- **Depends on:** every other feature. This one is last for good reason — each module must exist
  before it can be reported on, and each contributes its own report definitions.
- **Depended on by:** nothing.
- **Degrades cleanly:** reports whose source feature is not built simply do not appear in the
  catalogue. Building this feature early with three modules in place is possible and produces three
  modules' worth of reports.

## What each feature contributes

This feature is mostly an assembly of definitions owned elsewhere. The initial catalogue:

| Source | Reports |
|---|---|
| 02 | Headcount, joiners and leavers, turnover, headcount by department, tenure distribution, org changes |
| 03 | Device uptime, holiday calendar coverage |
| 04 | Attendance summary, absence rate, lateness summary, overtime by department, device gaps, correction volume |
| 06 | Leave taken and remaining, leave liability, absence by type, approval turnaround |
| 07 | Payroll cost by department and component, period comparison, variance history, adjustment volume |
| 09 | Review completion, goal completion — **never ratings by person, and no rating distribution** (D-09) |
| 10 | Time to hire, pipeline conversion, source effectiveness, offer acceptance rate |

Each definition lives with the feature that owns the data, and this feature provides the machinery.

## Open questions

| ID | Question | Why it matters | Proposed default |
|---|---|---|---|
| OQ-1101 | **Which reports does the company actually need?** The catalogue above is what such systems usually have, which is not the same as what anyone reads. | Building thirty reports of which four are used is the standard outcome here. The machinery is cheap; each report is not. | Start with six, chosen by asking. Add on request. |
| OQ-1102 | Is there an **external reporting obligation** — statutory returns, industry filings — with a fixed format? | A required format is a hard requirement rather than a nice report, and may change the data model if something is not currently captured. | None assumed. |
| OQ-1103 | What is the **minimum population** for suppression, and which measures count as sensitive? | D-03. Five is conventional; the right answer depends on company size. In a 40-person company a threshold of 5 suppresses most department-level reporting. | 5, with pay and performance always sensitive. Needs sanity-checking against actual headcount. |
| OQ-1104 | At what volume does live querying stop being adequate? | D-07. Attendance is the only large table; three years of 200 employees is ~900 000 punch rows, which is still comfortable. | Revisit past 2 million rows or 1 000 employees. |
| OQ-1105 | Will anyone want to connect **Excel, Power BI, or similar** directly to the database? | This bypasses every permission rule in the system. It is also the single most common request once reporting exists. | Refuse direct database access; offer exports and, if needed, a permission-respecting API later. |
| OQ-1106 | Should reports be **exportable to PDF** as well as CSV, and do they need company branding? | PDF is wanted for anything that leaves the building. Branding requires 03's logo and a layout pass. | CSV first, PDF for the handful that are shared externally. |
| OQ-1107 | Who may see **company-wide** aggregates — only HR and super admin, or department heads too? | A department head seeing company-wide pay cost is a policy decision, not a technical one. | HR and super admin only; managers see their own scope. |
| OQ-1108 | Should there be a **turnover / attrition** report at all, given D-09? | Aggregate turnover is a legitimate and useful business metric. Turnover *predictions about named individuals* are not, and the line between them is one product decision away. | Aggregate only, with the prediction line explicitly out of scope. |
