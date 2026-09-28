# 11 — Reports & Analytics — UI / UX

Builds on the shell and principles in [01 ui-ux.md](../01-roles-permissions-auth/ui-ux.md).

> **Design grounding.** Written against the workspace `ui-ux-pro-max` guideline set per
> `C:\Dev\CLAUDE.md`, using both `ux-guidelines.csv` and `charts.csv`. Cited inline as *[UX-nn]* and
> *[CH-n]* and listed at the end.
>
> **At implementation time**, the workspace also has a `dataviz` skill whose description says to
> load it before writing any chart code. This document decides *which* charts and why; whoever
> writes the chart components should load that skill before the first line of chart code.

## Principles specific to this feature

1. **A number without provenance is a rumour.** Every result states its as-of time, filters, scope,
   and sources — on screen and in every export *[D-05]*. This is the feature's defining habit.
2. **Say what is missing and why.** Empty, suppressed, and failed are three different outcomes with
   three different messages *[FR-R-06]*. A blank cell that might mean zero is worse than no report.
3. **The table is the report; the chart is a summary of it.** Every chart has a visible, sortable
   data table — not a hidden accessibility fallback, but the primary artefact *[CH-1, CH-2]*.
4. **Never imply a capability the reader lacks.** A report they cannot run is absent, not greyed.
   A drill-through they cannot follow is not offered *[FR-C-03, FR-P-06]*.

## Screen map

```
/reports                      Catalogue, grouped by area        report.read
/reports/:key                 Report viewer                     report.read + the report's own
/reports/views                My saved views                    report.read
/reports/schedules            Scheduled reports                 report.schedule
/dashboard                    Role dashboard                    (composition)
```

## `/reports` — the catalogue

Grouped by area (People, Attendance, Leave, Payroll, Hiring), each report a row with its title and a
plain-language description of the question it answers — "How much approved overtime did each
department work?" rather than "Overtime by department".

Only reports the actor can run appear *[FR-C-03]*. An empty group is not rendered. There is no
"upgrade" or "request access" affordance, because a list of things you cannot see is itself
information.

Saved views appear beneath their report, so the monthly ritual is one click from the catalogue.

## `/reports/:key` — the viewer

Generated from the report's declaration *[FR-C-06]*, so a new report needs no UI work. Three regions,
top to bottom: **filters**, **provenance**, **result**.

### Filters

Rendered from the parameter declaration: date ranges, department (tree picker with an
include-sub-departments toggle), employment type, status. Real labels on everything *[UX-54, UX-43]*,
required parameters marked *[UX-59]*.

Filters live in the URL so a view can be shared or bookmarked *[UX-5]* — and sharing a URL shares
the filters, never the data, so the recipient sees it under their own permissions.

Applied filters sit above the result as removable chips that wrap rather than clip *[UX-115, UX-116]*.

### Provenance — the strip that always appears

```
As of 15 Sep 2026, 11:02 UTC · 1 Jan – 30 Sep 2026 · Finance (incl. sub-departments)
Showing your team (14 people) · Sources: Attendance, Employees
```

Small, permanent, never collapsed. The scope line is the one that prevents the most common
misreading — a manager taking their team's figure for the company's *[FR-R-03]*.

### Result

A table first *[principle 3]*: right-aligned tabular figures, consistent decimals, currency in the
header rather than every cell, sortable columns with `aria-sort` *[CH-2]*. Totals in a distinct
footer row, computed over the whole result rather than the page *[FR-R-02]*.

**Suppressed cells** show a marker and the reason, never a blank and never a zero *[FR-S-03]*:

```
Department    Employees   Average salary
Finance              14          84,200
Legal                 3               —   Hidden (fewer than 5 people)
Executive             4               —   Hidden (fewer than 5 people)
Total                21               —   Hidden
```

The explanation appears once above the table and as a per-cell hint. Note the total is hidden too —
the UI must not present a total that would let a suppressed row be derived, and the API is
responsible for that *[FR-S-04]*. If the interface ever shows exactly one suppressed row, something
is wrong upstream.

**Drill-through** is a link on the aggregate value, present only where permitted *[FR-P-06]*.

**Comparison**, where supported, adds previous-period columns and a change column with sign and
arrow — direction carried by the arrow and the sign, not colour *[UX-37]*.

### Charts

Charts are optional per report and never replace the table. Choices are made from the guidance,
not by habit:

| Report shape | Chart | Why |
|---|---|---|
| Headcount, absence rate, cost over months | **Line** *[CH-1]* | A time axis with rise and fall. Fewer than 4 points renders a stat card instead, not a two-dot line |
| Overtime by department, cost by component | **Bar, sorted descending** *[CH-2]* | Comparing discrete categories where ranking is the insight. Over 15 categories: table only |
| Recruitment pipeline conversion | **Funnel** *[CH-7]* | Sequential stages with drop-off; stage names and values always visible, not encoded in a gradient |

**No pie or donut charts anywhere in this system.** The guidance rates them high-risk for
accessibility and unsuitable when precise values matter *[CH-3]* — and in an HR product the precise
value is always what is wanted. Part-to-whole uses a 100% stacked bar with labels.

Chart rules, all of them non-negotiable:

- Series are distinguished by **line style and direct labels**, never by hue alone *[CH-1, UX-37]*.
- Every chart has its data table visible beneath it — not behind a toggle *[CH-1, CH-2]*.
- Keyboard: focus moves through points and reveals values; sortable table headers respond to
  Enter/Space and report `aria-sort` *[CH-2, UX-41]*.
- A concise text summary sits with the chart — "Overtime rose from 180 to 212 hours between January
  and September, with the largest increase in Operations" — which serves screen-reader users and
  also the majority of sighted readers, who read the sentence and skip the chart.
- Axes start at zero for bar charts; a truncated axis on a magnitude comparison is a lie.

### States — three different empties *[FR-R-06]*

| Outcome | Message |
|---|---|
| No matching data | "No data matches these filters." + *Clear filters* |
| Everything suppressed | "All figures are hidden because each group covers fewer than 5 people. Try a wider date range or fewer filters." |
| Failed | "This report couldn't be produced." + what to try + *Retry* *[UX-80]* |
| Still running | Progress with `aria-busy` and a stable layout; a long report says so rather than appearing hung *[UX-78]* |

Conflating the first two would be the worst failure available here: a reader who believes there is
no overtime, when in fact the figures are withheld, has been actively misled.

## Export

A button that states what it will produce: "Export 1,204 rows to CSV". Over the row limit it refuses
with the count and a suggestion to narrow *[FR-E-01]*, rather than starting a download that fails.

The dialog states plainly that the export carries the same restrictions:

> This file contains what you can see — your team (14 people) — with hidden figures left out. It
> will be recorded as a data export.

Saying that a disclosure is recorded is a mild and effective control *[D-10]*.

## `/reports/schedules`

A list: report, filters, frequency, recipients, next run, owner, status. Creating one is a short
form, and its recipient picker warns immediately about anyone lacking the permission *[FR-E-05]*:

> Imran Qadir can't see this report. He'll be skipped unless his access changes.

A disabled schedule shows why in the row — "Disabled: owner's account was suspended"
*[FR-E-06]* — rather than sitting silently inactive.

The delivery notification is stated at creation, so nobody expects an attachment:

> Recipients get a link, not the data. They'll need to sign in to open it.

## `/dashboard`

Tiles composed per role *[FR-D-01]*: a headline figure, a small chart where it helps, and a link into
the full report with the same filters *[FR-D-03]*.

Tiles render as they arrive rather than the page waiting for the slowest *[FR-D-05, UX-78]*, each in
a fixed-height container so nothing jumps as figures land *[UX-19]*.

Tiles the actor cannot see are absent and the layout closes up *[FR-D-02]* — no gaps, no placeholders
hinting at what others see.

Each tile carries its own as-of time. A dashboard of figures calculated at different moments must say
so, or someone will reconcile two of them and lose an afternoon.

## Accessibility

Reporting is where accessibility is most often abandoned, because tables and charts are harder.

- Data tables use real `<table>` markup with `<caption>`, scoped headers, and `aria-sort` *[CH-2]*.
- Wide tables scroll horizontally within their container rather than breaking the page *[UX-71]*.
- Every chart has a visible table and a text summary *[CH-1, CH-2, CH-7]*.
- Nothing is encoded by colour alone — series, changes, and suppression all carry text or shape
  *[UX-37]*.
- Contrast ≥ 4.5:1 including on chart labels and axis text *[UX-36, UX-76]*.
- Full keyboard operation for filters, sorting, drill-through, and chart point inspection
  *[UX-41, UX-28]*; focus never hidden behind the sticky filter bar *[UX-100]*.
- Reduced motion honoured — chart entry animations are the first thing to drop *[UX-9]*.
- Number formatting follows 03's settings, and figures use tabular numerals so columns align.

## Copy reference

| Situation | Text |
|---|---|
| Provenance | As of {time} · {period} · {filters} |
| Scope note | Showing your team ({n} people) |
| Suppression, above table | Some figures are hidden because they cover fewer than {n} people. |
| Suppression, in cell | Hidden (fewer than {n} people) |
| No data | No data matches these filters. |
| All suppressed | All figures are hidden because each group covers fewer than {n} people. Try a wider date range or fewer filters. |
| Failed | This report couldn't be produced. {reason} |
| Export button | Export {n} rows to CSV |
| Export too large | This export has {n} rows, over the {limit} limit. Narrow the date range or filter by department. |
| Export notice | This file contains what you can see — {scope} — with hidden figures left out. It will be recorded as a data export. |
| Schedule recipient warning | {Name} can't see this report. They'll be skipped unless their access changes. |
| Schedule delivery | Recipients get a link, not the data. They'll need to sign in to open it. |
| Schedule disabled | Disabled: owner's account was suspended. |
| Chart summary example | Overtime rose from {a} to {b} hours between {start} and {end}, with the largest increase in {group}. |

## Guidelines applied

From `.claude/skills/ui-ux-pro-max/data/ux-guidelines.csv` and `charts.csv`:

| Ref | Guideline | Where |
|---|---|---|
| CH-1 | Trend over time → line; never hue alone; visible data table; keyboard point values | Time-series reports |
| CH-2 | Compare categories → bar sorted descending; >15 categories → table; sortable table with `aria-sort` | Department comparisons |
| CH-3 | Part-to-whole → pie is high accessibility risk and wrong when precise values matter | **No pie charts in this system** |
| CH-7 | Funnel → 3–8 stages, names and values visible, not gradient-encoded | Recruitment pipeline |
| UX-37 | Never colour alone | Series, changes, suppression |
| UX-71 | Wide tables scroll, not break | Result tables |
| UX-78 / UX-19 | Stable loading, reserved space | Long runs, dashboard tiles |
| UX-80 | Errors carry a recovery path | Failed runs, oversized exports |
| UX-54 / UX-43 / UX-59 | Real labels, required marked | Filters |
| UX-5 | URLs reflect state | Shareable filtered views |
| UX-115 / UX-116 | Chips wrap, labels stay whole | Applied filters |
| UX-36 / UX-76 | Contrast, including chart text | Charts and tables |
| UX-41 / UX-28 / UX-100 | Keyboard, focus visible and unobscured | Throughout |
| UX-9 | Reduced motion | Chart animation |

## Open questions

| ID | Question |
|---|---|
| OQ-1101 | Which six reports to start with. Each report's viewer is generated, but its definition, its source query in the owning feature, and its chart choice are real work. |
| OQ-1106 | PDF export needs a layout and 03's branding. Worth doing only for reports that leave the building. |
| OQ-1109 | User-arranged dashboards. Deliberately absent; the per-role composition covers most of the need at none of the cost. |
| OQ-1112 | Should the chart text summary be generated from the data (templated) or written per report? Templated scales and reads mechanically; hand-written reads well and goes stale. |
