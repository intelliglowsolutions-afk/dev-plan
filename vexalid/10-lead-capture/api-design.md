# 10 — Lead Capture — API Design

## Conventions

- Submissions are **Server Actions** in `lib/actions/` (D-1001), not API routes. Route handlers exist
  only where a real URL is required — an email confirmation link, a file download.
- Every action returns a discriminated union, never a thrown error, so the form can render the
  failure:

```ts
type ActionResult =
  | { ok: true; redirectTo?: string }
  | { ok: false; kind: 'validation'; fieldErrors: Record<string, string[]> }
  | { ok: false; kind: 'rate_limit'; retryAfterSeconds: number }
  | { ok: false; kind: 'captcha' }
  | { ok: false; kind: 'server'; message: string }
```

- Error messages shown to users never leak internals. `kind: 'server'` renders a fixed string plus
  the contact email; the real error goes to Sentry with a correlation id.
- Every action follows the same order (D-1008): **validate → defend → persist → notify → redirect.**
  Persistence precedes notification, always. Email failures are logged and swallowed; the lead is
  already saved.

## Shared pipeline — `lib/actions/_pipeline.ts`

```ts
async function guard(formData: FormData, form: FormName, ip: string): Promise<GuardResult>
```

1. **Honeypot** — `formData.get('website')` non-empty → return `{ silentSuccess: true }`. The caller
   redirects to the thank-you page without writing anything (D-1003 layer 1).
2. **Timing** — `Date.now() - renderedAt < 3000` → `{ silentSuccess: true }`.
3. **Turnstile** — POST to `https://challenges.cloudflare.com/turnstile/v0/siteverify`. On a
   non-success response, or on a network error, return `{ ok: false, kind: 'captcha' }` — **fails
   closed**, deliberately.
4. **Rate limit** — `checkRateLimit(sha256(ip), form)`; over limit → `{ kind: 'rate_limit' }`.

`ip` comes from the `x-forwarded-for` header set by nginx. nginx must be configured to **overwrite**
that header, not append to a client-supplied one (segment 12) — otherwise the rate limiter is
trivially bypassed by a spoofed header.

---

## `submitDemoRequest(prev, formData)`

**Schema** — `lib/schemas/demo-request.ts`

```ts
export const DemoRequestSchema = z.object({
  name:        z.string().trim().min(2).max(80),
  email:       z.string().trim().toLowerCase().email().max(254),
  company:     z.string().trim().min(2).max(100),
  companySize: z.nativeEnum(CompanySize),
  message:     z.string().trim().max(1000).optional(),
  // hidden
  website:     z.string().max(0),            // honeypot
  renderedAt:  z.coerce.number().int(),
  sourcePath:  z.string().max(200).optional(),
  utmSource:   z.string().max(100).optional(),
  // … utmMedium, utmCampaign, utmTerm, utmContent
})
```

**Flow**

1. `guard()`.
2. Parse. On failure → `{ kind: 'validation', fieldErrors: flatten().fieldErrors }`.
3. `prisma.demoRequest.create()` with UTM, referrer, hashed-free IP, UA, and
   `privacyNoticeVersion` from a constant.
4. Fire both emails via `sendEmail()` — **awaited but individually try/caught**; a failure writes an
   `EmailLog` row with `FAILED` and reports to Sentry, and the action still succeeds.
5. `redirect('/thank-you/demo')` — Next's `redirect()` throws, so it must be the last statement,
   outside any try block.

**Emails**

| Kind | To | Subject | Body |
|---|---|---|---|
| `DEMO_NOTIFICATION` | `SALES_NOTIFICATION_EMAIL` | `New demo request — {company} ({companySize})` | Plain text table of all fields; source path; UTM; a `mailto:` reply link. Scannable on a phone. |
| `DEMO_AUTOREPLY` | the prospect | `Thanks for your interest in Vexalid` | Short, from a named human (OQ-1005), `reply-to` the shared alias. States what happens next and when. No images, no tracking pixel. |

---

## `submitServicesEnquiry(prev, formData)` — D-1014

**Schema** — `lib/schemas/services-enquiry.ts`

```ts
export const ServicesEnquirySchema = z.object({
  name:         z.string().trim().min(2).max(80),
  email:        z.string().trim().toLowerCase().email().max(254),
  organisation: z.string().trim().min(2).max(100),
  interest:     z.nativeEnum(ServiceInterest),     // incl. NOT_SURE
  stage:        z.nativeEnum(EngagementStage),
  message:      z.string().trim().max(2000).optional(),
  // hidden
  website:      z.string().max(0),
  renderedAt:   z.coerce.number().int(),
  sourcePath:   z.string().max(200).optional(),
  // … utm fields
})
```

**Flow** — identical to the demo request: `guard()` → parse → `prisma.servicesEnquiry.create()` →
two emails → `redirect('/thank-you/enquiry')`.

**Emails**

| Kind | To | Subject | Body |
|---|---|---|---|
| `ENQUIRY_NOTIFICATION` | `ENQUIRY_NOTIFICATION_EMAIL` (shared alias — OQ-1002) | `[{stage}] {interest} enquiry — {organisation}` | All fields, source path, UTM, a `mailto:` reply link. **The subject line carries `stage` and `interest` first** so an inbox triage is possible without opening anything — `[URGENT] CSV enquiry — …` is the one that gets opened on a phone. |
| `ENQUIRY_AUTOREPLY` | the enquirer | `Thanks for getting in touch` | Short, from a named person (OQ-1005), `reply-to` the shared alias. What happens next and when. Mentions that an NDA is available (D-1016). |

**Routing.** `interest` and `stage` are stored, not acted on automatically — v1 has no CRM
(OQ-1001), so routing is a human reading a well-formed subject line. Designing for that constraint
is why the subject line matters more than it would with a pipeline behind it.

## Other enquiry types

Support, press, partnership and GDPR data requests have **no Server Action and no table** — they go
to published email addresses listed on `/contact` (requirements.md). A subject access request must
not be able to fail silently inside a form pipeline.

---

## `subscribeNewsletter(prev, formData)`

**Schema**

```ts
export const NewsletterSchema = z.object({
  email:   z.string().trim().toLowerCase().email().max(254),
  consent: z.literal('on', { message: 'Please tick the box to subscribe.' }),
  source:  z.string().max(60).optional(),
  website: z.string().max(0),
  renderedAt: z.coerce.number().int(),
})
```

**Flow**

1. `guard()`.
2. Parse.
3. Look up by email:
   - **`CONFIRMED`** → return success. Send nothing. Reveal nothing — otherwise the form is an
     address-enumeration oracle.
   - **`PENDING`** and `lastConfirmSentAt` within an hour → return success, send nothing.
   - **`PENDING`**, older → regenerate the token, resend.
   - **`UNSUBSCRIBED`** → reset to `PENDING` with a fresh token and consent record; a
     re-subscription is a new consent event and is recorded as one.
   - **None** → create.
4. Token: `crypto.randomBytes(32).toString('base64url')`. Store `sha256(token)` (D-1005 / data-model).
   Expiry 30 days.
5. Send `NEWSLETTER_CONFIRM`.
6. `redirect('/thank-you/newsletter')`.

### `GET /api/newsletter/confirm?token=…`

1. `sha256` the token, look up by `confirmToken`.
2. Not found → `/newsletter/invalid` explaining the link may have expired, with a resend form.
3. Expired → same page.
4. Already `CONFIRMED` → redirect to `/thank-you/newsletter` (idempotent — people click twice).
5. Otherwise: set `CONFIRMED`, `confirmedAt`, clear the token, push to Listmonk
   (`POST {LISTMONK_URL}/api/subscribers` with `status: 'enabled'`, storing `listmonkId`).
   **A Listmonk failure does not fail the confirmation** — the row is marked confirmed with a
   `listmonkId` of null, and a nightly reconciliation job retries. The subscriber's consent is
   recorded either way.
6. Redirect to `/thank-you/newsletter?confirmed=1`.

`GET` is used because it is an email link. The action is idempotent and low-risk, which is what makes
that acceptable here.

---

## `requestGatedDownload(prev, formData)`

**Schema**

```ts
export const GatedDownloadSchema = z.object({
  firstName:       z.string().trim().min(1).max(60),
  email:           z.string().trim().toLowerCase().email().max(254),
  guideSlug:       z.string().regex(/^[a-z0-9-]+$/),
  newsletterOptIn: z.coerce.boolean().default(false),
  website:         z.string().max(0),
  renderedAt:      z.coerce.number().int(),
})
```

**Flow**

1. `guard()`.
2. Parse; verify `guideSlug` resolves via `getGuide()` and that `gated === true` and `pdfPath` exists.
   A request for a non-gated or unknown slug is a 404, not a download.
3. Token: `base64url(random(32))`; store `sha256(token)`; `tokenExpiresAt = now + 7 days`.
4. Create the `GatedDownload` row.
5. If `newsletterOptIn`, call the newsletter subscription path — **separately**, as its own consent
   record with its own `consentText` and `source: 'guide:<slug>'`.
6. Send `DOWNLOAD_LINK` with `{SITE_URL}/api/download/{token}`.
7. `redirect('/thank-you/guide')`.

### `GET /api/download/[token]`

1. `sha256`, look up. Not found → 404 page with a request-again link.
2. Expired → a page offering a fresh link, not a raw error.
3. `downloadedAt` already set → increment `downloadCount`, show a "this link has already been used"
   page with a re-send option (D-1007). Do not silently serve it again.
4. Otherwise: set `downloadedAt`, increment the count, stream the PDF with
   `Content-Type: application/pdf` and
   `Content-Disposition: attachment; filename="vexalid-<slug>.pdf"`.
5. Response headers: `Cache-Control: private, no-store`, `X-Robots-Tag: noindex, nofollow`.

The route reads the file from **outside `public/`** (see requirements.md) — anything under `public/`
is served directly by URL and cannot be gated.

---

## `lib/email/client.ts`

```ts
type SendArgs = {
  kind: EmailKind
  to: string
  subject: string
  react: React.ReactElement       // React Email template
  replyTo?: string
  relatedType?: string
  relatedId?: string
}

export async function sendEmail(args: SendArgs): Promise<{ ok: boolean; id?: string }>
```

1. Write an `EmailLog` row with `QUEUED` **before** dispatch (D-1009).
2. Dispatch through the provider SDK.
3. Update to `SENT` with `providerId`, or `FAILED` with the error text.
4. Never throw. Callers check `ok` and carry on.

Templates live in `lib/email/templates/`, built with React Email:

| Template | Notes |
|---|---|
| `demo-notification.tsx` | Internal. Plain, dense, scannable on a phone |
| `demo-autoreply.tsx` | Warm, brief, named sender, one clear next step |
| `contact-notification.tsx` | Internal |
| `newsletter-confirm.tsx` | One button, a plain-text URL fallback, who is sending and why, `List-Unsubscribe` headers |
| `download-link.tsx` | Button + plain URL, expiry stated explicitly, one related suggestion |

**Rules:** every template has a plain-text alternative (many corporate HR mail clients strip HTML);
no tracking pixels; no external images; table-based layout with inline styles, because Outlook;
tested in Gmail, Outlook and Apple Mail before launch (T-1003).

---

## `lib/rate-limit.ts`

```ts
export async function checkRateLimit(
  hashedIp: string,
  form: FormName,
): Promise<{ allowed: boolean; retryAfterSeconds: number }>
```

Hour-truncated window, single `upsert` incrementing `count`, compared against a per-form limit map
(`demo: 5`, `contact: 5`, `newsletter: 3`, `download: 10`). The IP is hashed before it reaches the
table, so the rate limiter is not itself an IP log.

Sliding-window precision is unnecessary here; a fixed hourly window is simpler and sufficient.

---

## Analytics events (hand-off to segment 11)

Fired from the thank-you pages, not from the action, so only genuinely completed conversions count:

| Page | Event | Props |
|---|---|---|
| `/thank-you/demo` | `demo_request_submitted` | `companySize`, `sourcePath` |
| `/thank-you/newsletter` | `newsletter_subscribed` | `source` |
| `/thank-you/guide` | `guide_download_requested` | `guideSlug` |
| `/api/newsletter/confirm` → redirect | `newsletter_confirmed` | `source` |

No personal data is ever sent to analytics — no email addresses, no names, no company names. Plausible
is cookieless and first-party; sending PII into it would undo that in one line.

---

## Error handling and observability

| Failure | Behaviour |
|---|---|
| Database unreachable | `{ kind: 'server' }`, Sentry alert, form keeps the user's input. **This is a lost lead — it must page someone.** |
| Email provider down | Lead saved, `EmailLog` `FAILED`, Sentry warning. Sales still sees the row. |
| Turnstile unreachable | Fails closed (`kind: 'captcha'`). Accepting submissions during an outage invites a spam flood. |
| Listmonk down | Confirmation still recorded; nightly reconciliation retries. |
| Malformed token | Friendly page, no stack trace, no distinction between "wrong" and "expired" in the copy. |

Weekly: a count of leads by source and the `EmailLog` failure rate. If demo requests drop to zero for
a week, that is far more likely to be a broken form than a quiet market — the check exists to catch
exactly that.
