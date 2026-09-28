# 10 — Lead Capture — Requirements

## Services enquiry form (`/contact`) — D-1014

The consultative form. Its job is to start a conversation with enough context that the first reply
can be useful rather than a request for more information.

**Fields (6)**

| Field | Type | Required | Validation | Note |
|---|---|---|---|---|
| Full name | text | yes | 2–80 chars | — |
| Work email | email | yes | RFC-valid; free-provider domains allowed | — |
| Organisation | text | yes | 2–100 chars | "Organisation", not "Company" — some regulated sites are institutes or authorities |
| What do you need help with? | select | yes | `Computerized System Validation` · `AI Automation` · `Custom Software` · `Not sure yet` | **"Not sure yet" is a deliberate option.** Forcing a choice on someone still scoping their problem loses exactly the early-stage enquiries worth having |
| Where are you in the process? | select | yes | `Exploring` · `Scoping a project` · `Ready to start` · `Urgent / deadline-driven` | The most useful routing field on the site (D-1015) |
| Tell us about your systems | textarea | no | ≤ 2000 chars | Longer limit than the demo form — this audience writes more, and what they write is the qualifying information |

Beneath the button: the NDA line (D-1016) — *"Happy to sign an NDA before we discuss specifics."*
A statement, not a checkbox.

**Tone.** Labels and helper text are consultative, not transactional. The submit button reads
`Send enquiry`, not `Submit` or `Get a quote`.

## Demo request form (`/demo`)

**Fields (6 — D-1011)**

| Field | Type | Required | Validation | Note |
|---|---|---|---|---|
| Full name | text | yes | 2–80 chars | — |
| Work email | email | yes | RFC-valid; free-provider domains **allowed** | Blocking gmail.com loses real small-business owners — a genuine secondary buyer |
| Company | text | yes | 2–100 chars | — |
| Which product? | select | yes | `HRM` · `StratumOne` · `Both` | Two products now; without this the follow-up is guesswork |
| Company size | select | yes | `1-10`, `11-50`, `51-200`, `201-500`, `500+` | Drives prioritisation; one tap |
| What would you like to see? | textarea | no | ≤ 1000 chars | Optional. The people who fill it in are the best leads |

If the HRM is the only product with a demo at launch (OQ-702 / OQ-704), the product select still
ships — with StratumOne present and its option labelled — rather than being retrofitted later.

Hidden: `honeypot` (must be empty), `renderedAt` (timestamp), `turnstileToken`, `sourcePath`,
`utm_*` if present.

**UX**

- Labels above fields, always visible. placeholders show format examples only, never the label.
- Required fields marked with an asterisk **and** a legend; optional fields labelled "(optional)" —
  marking the optional ones reads better when most are required.
- Validation on blur after first interaction, then on every change once a field has errored. Never
  validate on first keystroke — showing "invalid email" while someone types the first letter is
  hostile.
- Errors: red text below the field, `aria-describedby`, `aria-invalid`, plus an icon so colour is not
  the only signal.
- On submit failure, focus moves to the first invalid field and a summary is announced via
  `role="alert"`.
- Submit button: `Book my demo`, full width on mobile, shows a spinner and `aria-busy` while
  pending, disabled only *during* the request — never disabled as a way to enforce validation, which
  strands keyboard and screen-reader users with no explanation.
- Success: redirect to `/thank-you/demo` (D-1012).
- Consent line beneath the button: "By submitting, you agree to our Privacy Policy. We'll only use
  your details to respond to this request." — a statement of fact, **not** a checkbox. Legitimate
  interest covers responding to an enquiry; a checkbox here would be consent theatre.

**States to design:** idle · focused · validating · error (per-field) · submitting · success ·
server error · rate-limited · Turnstile challenge · JavaScript disabled.

## Everything that is neither an enquiry nor a demo

Support, press, partnership and — critically — **GDPR data requests** still need a route. They must
not go through the services enquiry form, which is scoped to new business and routed to a consultant.

**Decision:** no third form. The `/contact` page carries a short "Other enquiries" block beneath the
services form, listing published addresses directly:

| Purpose | Route |
|---|---|
| Product support | `support@vexalid.com` (OQ-506) |
| Press and partnerships | `contact@vexalid.com` |
| **Data protection / GDPR requests** | A dedicated published address, stated explicitly |

Rationale: these are low-volume and not leads. A third form would add a table, a Server Action, spam
defence and a state machine to handle a handful of messages a month. A published address costs
nothing and is, for a data request, **more** reliable — a subject access request must not be able to
fail silently in a form pipeline, and the 30-day statutory clock starts when it arrives.

The data-protection address is also named in `/legal/privacy` and `/legal/dpa`, so it is discoverable
by someone who never visits `/contact`.

Success is inline (a `role="status"` region replacing the form), not a redirect — contact is a lower-
intent action and a page change is disproportionate.

## Newsletter (footer, `/resources`, blog posts)

| Field | Type | Required |
|---|---|---|
| Email | email | yes |
| Consent | checkbox | yes — **never pre-ticked** (D-1005) |

Consent label: *"Email me occasional HR articles and product updates. Unsubscribe any time."*

**Double opt-in flow**

1. Submit → row created with `status: PENDING` and a hashed confirmation token.
2. Confirmation email sent: one clear button, a plain-text URL fallback, and a line stating who is
   sending and why.
3. Click → `/api/newsletter/confirm?token=…` → `status: CONFIRMED`, `confirmedAt` set, subscriber
   pushed to Listmonk → redirect to `/thank-you/newsletter`.
4. Unconfirmed rows are deleted after 7 days (D-1010). An unconfirmed address is not a lawful
   marketing contact, so keeping it has no purpose.

**Edge cases**
- Already confirmed → success message, **no second email**, no disclosure that the address is on the
  list (which would make the form an address-enumeration oracle).
- Pending and re-submitted → resend, but at most once per hour per address.
- Expired token (30 days) → a friendly page offering to resend.
- Unsubscribe: Listmonk's link, plus `List-Unsubscribe` and `List-Unsubscribe-Post` headers so the
  one-click unsubscribe in Gmail and Outlook works. Missing these now hurts deliverability.

## Gated download (`/resources/guides/[slug]`)

| Field | Type | Required |
|---|---|---|
| First name | text | yes |
| Work email | email | yes |
| Newsletter consent | checkbox | **no — unticked, genuinely optional** |

The download is delivered whether or not the box is ticked. Bundling consent with the download makes
it not freely given, and therefore invalid under GDPR — lawful basis for the download itself is the
contract/request, and the newsletter is a separate, optional consent.

**Flow**

1. Submit → `GatedDownload` row, HMAC token generated (D-1007).
2. Email with the link → redirect to `/thank-you/guide`.
3. Link → `/api/download/[token]` → validate signature, expiry (7 days), single use → stream the PDF
   with `Content-Disposition: attachment`.
4. Second use → a page offering to re-send, not a raw error.

The PDF lives in `public/downloads/` but is **never linked from any page**, and `/downloads/` is
disallowed in `robots.txt`. (Note: files under `public/` are directly reachable by URL in Next.js —
if strict enforcement is required, move them outside `public/` and stream from disk in the route
handler. Recommended.)

## Spam and abuse defence (D-1003)

Layered, cheapest first, evaluated in order:

| # | Layer | Mechanism | Response |
|---|---|---|---|
| 1 | Honeypot | Hidden `website` field, off-screen via CSS (not `display:none`, which some bots detect), `tabindex="-1"`, `autocomplete="off"`, `aria-hidden` | Non-empty → return **success** without storing. Telling a bot it failed teaches it |
| 2 | Timing | `renderedAt` vs submission time | < 3 seconds → treat as bot, silent success |
| 3 | Turnstile | Cloudflare, server-side `siteverify` | Failure → generic error. **Fails closed** if the verification call errors |
| 4 | Rate limit | Per IP + per form, sliding window in Postgres (D-1004) | Demo/contact: 5/hour. Newsletter: 3/hour. Download: 10/hour. → 429 with a clear message |
| 5 | Disposable domains | Small blocklist on gated downloads only | Soft warning, not a hard block |

Not used: content keyword filtering (too many false positives on genuine HR enquiries), and IP
blocklists (they catch corporate NAT gateways — i.e. exactly Vexalid's buyers).

## GDPR requirements (T-1014)

| Form | Lawful basis | Retention | Notes |
|---|---|---|---|
| Demo request | Legitimate interest (responding to a request) | 24 months from last contact | Privacy notice at point of collection |
| Contact | Legitimate interest | 24 months | Same |
| Newsletter | **Consent** | Until unsubscribe + 12 months of proof-of-consent record | Double opt-in; consent timestamp, IP and wording version recorded |
| Gated download | Legitimate interest (fulfilling the request) | 24 months | Newsletter consent, if given, is separate and separately recorded |

**Required practices**
- A link to `/legal/privacy` at every point of collection.
- Consent records store the **exact wording shown**, plus timestamp and IP — "we have consent" is
  only defensible if you can show what was agreed to.
- An erasure procedure: a documented runbook, executable within 30 days, covering Postgres and
  Listmonk. A right-to-erasure request that cannot actually be fulfilled is the failure mode here.
- A subject access procedure: export all rows for an email address as JSON.
- No prospect data sent to any third party beyond the email provider (a processor) — no ad pixels,
  no data brokers.
- If a CRM is added later (OQ-1001), it becomes a processor and the privacy notice must be updated.

## Accessibility (all forms)

- Every input has a programmatically associated `<label>`. No exceptions, no placeholder-as-label.
- Errors: `aria-invalid`, `aria-describedby`, and a `role="alert"` summary on submit failure.
- Async results announced through `role="status"` (polite).
- Fieldsets and legends for grouped inputs.
- Full keyboard operability in visual order.
- Touch targets ≥ 44px; inputs at 16px font (prevents iOS zoom-on-focus).
- Correct `autocomplete` attributes (`name`, `email`, `organization`) — faster for everyone and
  materially so for users of assistive technology.
- Turnstile's own widget is keyboard-accessible; verify this in the a11y pass rather than assuming it.

## Notification and follow-up

| Trigger | Recipient | Content |
|---|---|---|
| Demo request | `SALES_NOTIFICATION_EMAIL` (shared alias — OQ-1002) | All fields, submission time, source path, UTM parameters, a `mailto:` reply link |
| Demo request | The prospect | Auto-reply confirming receipt, stating what happens next and when (OQ-505), from a named sender (OQ-1005) |
| Services enquiry | Shared alias (OQ-1002) | All fields; `stage` and `interest` lead the subject line |
| Newsletter | The subscriber | Confirmation email only |
| Gated download | The requester | Download link + a one-line related suggestion |

Notification emails are plain and scannable — they are read on a phone, usually while doing
something else. No marketing template, no images, no tracking pixel.
