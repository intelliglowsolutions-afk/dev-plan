# 08 — Employee Self-Service — UI / UX

> **This is the feature.** Everything else in feature 08 is plumbing over endpoints that already
> exist. The interface *is* the deliverable, so this is the longest and most specific of the eight
> `ui-ux.md` documents so far.

> **Design grounding.** Written against the workspace `ui-ux-pro-max` guideline set per
> `C:\Dev\CLAUDE.md`, with the mobile and responsive guidelines carrying most of the weight here.
> Cited inline as *[UX-nn]* and listed at the end.

## Who this is for

Not an HR professional. Someone who:

- opens this **three or four times a month**, usually to check one thing;
- is on a **phone**, possibly on mobile data they pay for, possibly on a two-year-old Android;
- has **never been trained** and will not read a guide;
- does not know what a "shift window", "pro-rated entitlement", or "missing punch" is;
- forms their entire opinion of "the HR system" from this screen.

Every decision below follows from that reader.

## Principles

1. **One thing per screen.** The admin side earns its density; this does not.
2. **Answer first, working out second** *[D-07]*. "11.5 days left" is the answer. The ledger is one
   tap away, not on the card.
3. **Plain language, no exceptions** *[D-05]*. Not one internal status name reaches the screen.
4. **Never show a problem the employee cannot act on** *[D-06]*. If attendance has not been computed
   yet, the card is quiet — it does not report the queue.
5. **Mobile-first, genuinely** *[UX-64]*. Built at 375 px, then widened. Not the reverse.

## The shell

**Bottom navigation** on mobile, five items maximum, because that is where thumbs are and the
sidebar pattern used everywhere else in the system is wrong on a phone:

```
  Home      My time     Leave      Pay      More
```

*More* holds profile, documents, directory, and notifications. Above 768 px the bottom bar becomes
the standard left sidebar, so the portal and admin shells converge on wide screens.

Shell details that are easy to get wrong and expensive to fix later:

- Fixed bottom nav plus any sticky element must account for **safe areas** and not overlap
  *[UX-17]*; `env(safe-area-inset-bottom)` padding, and nothing else fixed at the bottom of a
  scrolling screen.
- Full-height layouts use `dvh`, never `100vh` — mobile browser chrome makes `vh` wrong *[UX-20]*.
- The browser **back button works everywhere**, including out of modals and multi-step flows
  *[UX-4]*. A portal that traps back is abandoned.
- `touch-action: manipulation` to remove tap delay *[UX-25]*; `overscroll-behavior: contain` so a
  scroll at the top of a list does not reload the page *[UX-26]*.
- No horizontal swipe gestures on main content *[UX-24]*.
- URLs reflect state, so a notification's deep link lands on the exact thing *[UX-5]*.

**Admin context switching** *[FR-S-03]*: a user with admin permissions sees a clearly-labelled
switch between "My portal" and the admin shell, and the current context is named in the header at
all times. Nobody should have to work out whose leave they are looking at.

## Home

The screen most people will see and close within eight seconds.

```
Good morning, Ayesha

┌──────────────────────────────────────┐
│ ⚠ Your leave request was declined    │
│   1–7 October                        │
│   "Two others are already off that   │
│    week"                          →  │
└──────────────────────────────────────┘
┌──────────────────────────────────────┐
│ ⚠ Your visa expires in 21 days    →  │
└──────────────────────────────────────┘

Annual leave
11.5 days left          3 days expire 31 Mar

Today
Signed in at 08:58

August payslip
Available                             →

Next holiday
National Day · Tue 6 Oct

[ Request leave ]  [ Report a problem ]
```

- **Action cards first**, ordered by priority, each stating the thing and the consequence in one
  line *[FR-D-02]*. A rejection shows its reason inline — being told "declined" and having to tap to
  find out why is the single most irritating pattern in portals of this kind.
- **Facts** second, as plain rows. The payslip row names the period and **never an amount**
  *[FR-Y-06]*.
- **Two quick actions**, not six. These are the things people come to do.
- **Nothing pending** is a complete screen, not an apology: "Nothing needs your attention right
  now." followed by the same facts *[FR-D-06]*.
- **First run**, before anything is configured: a short welcome explaining what the portal is for,
  and which sections will fill in as HR sets things up *[D-09, FR-S-07]*.

Body text is at least 16 px *[UX-67]*; contrast at least 4.5:1 *[UX-36, UX-76]*; a consistent type
scale rather than arbitrary sizes *[UX-74]*.

## My time

A month, one row per day, newest first — not a calendar grid. On a phone a list is scannable and a
grid is a puzzle.

```
Mon 15 Sep    Present          08:58 – 17:03    8h 5m
Fri 12 Sep    You didn't       09:01 – —        —          ⚠
              sign out
Thu 11 Sep    On leave         Annual leave
Wed 10 Sep    Terminal         —                —          ⓘ
              wasn't working
```

The **translations are the work here** *[D-05]*, and one of them matters more than the rest:

| Internal (feature 04) | What the employee reads |
|---|---|
| `PRESENT` | Present |
| `LATE` | Late — 14 minutes |
| `MISSING_PUNCH` | You didn't sign out |
| `ABSENT` | No record for this day |
| `ON_LEAVE` | On leave — annual leave |
| `HOLIDAY` | Public holiday — National Day |
| `WEEKLY_OFF` | Day off |
| `UNKNOWN` | **The terminal wasn't working** |

`UNKNOWN` expands to: *"The attendance terminal wasn't recording that day. This isn't counted
against you — HR will sort it out."* *[FR-A-04]*. Feature 04 went to some trouble never to mark
these people absent; this sentence is where that care becomes visible to the person it protects.

Tapping a day shows what was expected, what was recorded, and — in plain sentences — how it was
decided. Feature 04's reason trail is translated, never rendered raw *[FR-A-03]*.

**Report a problem** is on every day and opens a short form: what was wrong (I forgot to sign out /
I was here but there's no record / the times are wrong), the correct time if relevant, and a note.
It creates exactly the correction request an HR admin would create *[FR-A-05]*. Status afterwards is
shown on the day itself, with the rejection reason if there is one.

Month totals sit at the top: days present, days off, late arrivals, overtime. No trends, no
comparisons, no history beyond last month *[FR-A-07, OQ-804]*.

## Leave

**Balance first**, one card per type, the available number given the most visual weight *[D-07]*:

```
Annual leave
11.5 days left

Entitled 20 · Taken 6.5 · Requested 2
⚠ 3 days expire on 31 March 2027
How is this worked out? →
```

"How is this worked out?" opens the ledger as dated plain-language lines: *"1 January — 20 days
added for 2026"*, *"14 February — 2 days taken, 12–13 Feb"*, *"2 September — 2 days held while your
request is decided"*.

**Requesting** reuses feature 06's flow, restyled for one column. The live cost panel becomes a
**sticky bar at the bottom of the viewport** while dates are being chosen — the number must stay
visible while it changes *[06 ui-ux, UX-17: it and the bottom nav cannot both be fixed, so the nav
hides while the form is active]*.

```
┌──────────────────────────────────────┐
│ 4.5 days · 7 left afterwards         │
│ Weekend and National Day not counted │
│              [ Request leave ]       │
└──────────────────────────────────────┘
```

Errors carry their fix *[UX-80]*: *"You have 11.5 days; this needs 12. Try ending a half day
earlier."* — with a control that does exactly that.

**My requests** lists status, dates, days, who has it, and any reason. Withdraw and cancel where the
rules allow.

**Who else is off** (subject to OQ-803) is a simple list by date for the employee's own department —
names and dates, no leave types. The grid belongs to HR; employees want "is anyone else off that
week", and a list answers it.

## Pay

The most sensitive screen, and the plainest.

A list of published payslips by period with the net amount. Tapping one opens the payslip: earnings,
deductions, net, each line tappable for its explanation in arithmetic a person can check by hand
(07's calculation trail, subject to OQ-717).

**Compare with last month** is offered directly, because it is the actual question:

```
August → September
Net pay      118,400 → 105,600    −12,800
  Basic         same
  Overtime    4,200 → 0           −4,200
  Unpaid leave    0 → −8,600      −8,600
```

That table prevents most pay queries, and each line links to the reason — the unpaid leave row links
to the leave request that caused it.

PDF download is prominent; it is what people came for when they came for a mortgage application.

No payroll figure appears anywhere outside this section *[FR-Y-06]*.

## Profile

Fields grouped as **Your details**, **Contact**, **Emergency contacts**, **Documents**, with each
field rendered by its configured policy *[FR-P-02, FR-P-03]*:

- `SELF_EDIT` — an inline edit control, saving immediately with clear confirmation *[UX-34]*.
- `REQUEST` — an edit control that opens a request: "Ask HR to change this". The submitted state is
  shown on the field itself: *"Requested 12 Sep — waiting for HR"*.
- `READ_ONLY` — plain text, with help text where useful: *"This is the number you use at the
  attendance terminal."*

Numeric and phone inputs set `inputmode` so the right keyboard appears *[UX-63]* — a small thing
that is noticed every single time it is missing.

**Emergency contacts** are fully self-managed with add/edit/remove *[FR-P-08]*, and the section
carries one line of motivation: *"We'll call these people if something happens to you at work."*
That sentence is why the data gets kept current.

**Documents** lists what HR holds, with expiry dates and a clear marker for anything expiring, plus
download. Upload only against a specific request *[FR-X-03, OQ-807]*.

**Bank details do not appear here at all** *[FR-P-10]* — not as a read-only field, not as a
"contact HR" row. A short line under Contact explains where to go: *"To change your bank details,
speak to HR in person."*

## Directory

Search-first: a single search field, results as name, position, department, photo, and work contact
details. Tapping a person shows their card and their place in the org chart — the indented list at
narrow widths, never the pan-and-zoom canvas *[02 ui-ux, FR-X-05]*.

This is consistently the most-used screen in portals like this, and it is cheap to build because
feature 02 already has the data and the endpoint.

## States

Every screen needs: loading (stable skeletons, `aria-busy`, no layout jump) *[UX-78, UX-19]*, empty,
error with a retry *[UX-80]*, and the portal-specific **not configured yet** *[D-09]*:

| Situation | What it says |
|---|---|
| No leave types configured | "Leave isn't set up yet. You'll see your balance here once HR has configured it." |
| No payslips | "Your first payslip will appear here after {month} payroll." |
| No shift assigned | "Your working hours haven't been set yet. Your attendance is still being recorded." |
| No attendance record today | Nothing — the card is simply absent *[D-06]* |

That last row is the discipline: silence, not an explanation of internal state.

## Accessibility

Some of this workforce will need it and none of them will ask.

- Touch targets ≥ 44×44 px with ≥ 8 px spacing *[UX-22, UX-23, UX-66]*; the WCAG 24 px minimum is
  the floor, not the target, on a surface this casual *[UX-104]*.
- Body text ≥ 16 px *[UX-67]*; usable at 200% zoom with no horizontal scroll *[NFR-02]*.
- Contrast ≥ 4.5:1 for all text, including the muted secondary lines that this design uses a lot
  *[UX-36, UX-76]*.
- Status never by colour alone — every state carries its word *[UX-37]*.
- Full keyboard operation with visible focus *[UX-28, UX-102]*; focus never hidden behind the fixed
  bottom nav *[UX-100]*.
- `prefers-reduced-motion` honoured *[UX-9]*.
- Tested at 320, 375, 414, and 768 px *[UX-65]*.

## Copy reference

The portal's copy *is* its design, so this table is a specification, not a suggestion.

| Situation | Text |
|---|---|
| Nothing pending | Nothing needs your attention right now. |
| Device gap | The attendance terminal wasn't recording that day. This isn't counted against you — HR will sort it out. |
| Missing sign-out | You didn't sign out. Tell us what time you left and HR will fix it. |
| No record | There's no record for this day. If you were at work, let us know. |
| Leave balance help | How is this worked out? |
| Ledger line | 2 September — 2 days held while your request is decided |
| Cost summary | {n} days · {m} left afterwards. Weekend and {holiday} not counted. |
| Insufficient balance | You have {a} days; this needs {b}. Try ending a half day earlier. |
| Request declined | Your leave request was declined — "{reason}" |
| Waiting | With {name} since {day}. |
| Emergency contacts | We'll call these people if something happens to you at work. |
| Change requested | Requested {date} — waiting for HR. |
| Change declined | HR declined this change — "{reason}" |
| Bank details | To change your bank details, speak to HR in person. |
| Employee code help | This is the number you use at the attendance terminal. |
| Leave not configured | Leave isn't set up yet. You'll see your balance here once HR has configured it. |
| No payslips | Your first payslip will appear here after {month} payroll. |
| No shift | Your working hours haven't been set yet. Your attendance is still being recorded. |

## Guidelines applied

From `.claude/skills/ui-ux-pro-max/data/ux-guidelines.csv`:

| Ref | Guideline | Where |
|---|---|---|
| UX-64 / UX-65 | Mobile-first; test 320/375/414/768 | The whole feature |
| UX-67 / UX-74 / UX-36 / UX-76 | 16 px body, consistent scale, 4.5:1 contrast | Typography throughout |
| UX-22 / UX-23 / UX-66 / UX-104 | Touch target size and spacing | Every control |
| UX-17 | Fixed elements must not overlap; safe areas | Bottom nav + sticky cost bar |
| UX-20 | `dvh`, not `100vh` | Full-height layouts |
| UX-4 / UX-5 | Back behaviour; URLs reflect state | Navigation and deep links |
| UX-24 / UX-25 / UX-26 | No gesture conflicts, no tap delay, contained overscroll | Touch behaviour |
| UX-63 | `inputmode` for the right keyboard | Profile and correction forms |
| UX-78 / UX-19 | Stable skeletons, no layout jump | All loading states |
| UX-80 | Errors carry a recovery path | Every validation message |
| UX-37 | Never colour alone | Attendance and request statuses |
| UX-28 / UX-100 / UX-102 | Visible focus, never obscured | Keyboard operation |
| UX-9 | Reduced motion honoured | Transitions |

## Open questions

| ID | Question |
|---|---|
| OQ-805 | **Language.** This screen is the strongest argument for i18n in the system: HR can work in English; the whole workforce may not. The copy table above is the thing that would be translated, and it is written as a table for that reason. |
| OQ-802 | If there is a shared kiosk rather than personal phones, this design is wrong in a specific way — sessions must be short, pay must be hidden or PIN-gated, and nothing personal may persist on screen. Worth knowing before the shell is built. |
| OQ-812 | Should the portal be installable as a PWA (home-screen icon, offline shell)? It is a small addition that markedly changes how often people open it. |
| OQ-804 | How much attendance detail to show. The design shows the record and this month's totals, and deliberately no trend. |
