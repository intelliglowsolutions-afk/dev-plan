# 10 — Lead Capture

**Priority:** Must-have · **Build order:** 10 of 13 · **Status:** planned

## Purpose

Turn visitors into contactable leads through the three conversion paths: the **services enquiry**, the
**product demo request**, and the **newsletter / gated content** capture. This segment owns form logic,
validation, storage, spam defence, notification and compliance — everything behind the field.

This is the segment where the site either pays for itself or does not.

## Scope

**In scope**

- Services enquiry form: fields, validation, submission, storage, notification, auto-reply.
- Demo request form: the same, with product-specific fields.
- Newsletter subscription with double opt-in, via self-hosted Listmonk.
- Gated content: email capture → single-use expiring download link → PDF delivery.
- Spam and abuse defence: honeypot, timing check, Turnstile, rate limiting.
- The `vexalid_web` database schema.
- Transactional email templates and sending.
- GDPR obligations: consent, lawful basis, retention, subject access and erasure.
- Thank-you pages and conversion event tracking hand-off.

**Out of scope (owned elsewhere)**

- Visual form components (`Input`, `Button`, error styling) → [02 Design System](../02-design-system/README.md).
- Page copy around the forms → [05 Home & Company](../05-home-company/README.md).
- The gated guide's *page* → [09 Resources](../09-resources-blog/README.md).
- Analytics event definitions → [11 SEO & Analytics](../11-seo-analytics/README.md).
- CRM synchronisation — designed here, **not built in v1** (OQ-1001).

## Files

| File | Contents |
|---|---|
| [requirements.md](./requirements.md) | Field-level specs, validation rules, UX states, spam defence, GDPR |
| [data-model.md](./data-model.md) | Prisma schema, indexes, retention, migrations |
| [api-design.md](./api-design.md) | Server Actions, route handlers, email templates, error contracts |

## Key decisions taken in this plan

| # | Decision | Rationale |
|---|---|---|
| D-1001 | **Server Actions, not API routes**, for form submission. Forms use a real `<form action={...}>` so they work without JavaScript. | Fewer moving parts, no hand-written fetch, and progressive enhancement comes free. A form that fails silently when a script fails to load is a lost lead nobody ever hears about. |
| D-1002 | **One Zod schema per form**, shared by client and server. The server always re-validates. | Client validation is a convenience for the user; it is not a security control. |
| D-1003 | **Layered spam defence:** honeypot → time-to-submit floor → Cloudflare Turnstile → per-IP rate limit. In that order, cheapest first. | Most bots are stopped by the honeypot alone. Turnstile only runs for what gets past it, so real users rarely see a challenge. |
| D-1004 | **Rate limiting in Postgres**, not Redis. | At this traffic volume a `RateLimit` table with a unique key and a cleanup job is entirely adequate, and it avoids adding a service to the VPS for one feature. |
| D-1005 | **Double opt-in for the newsletter, always.** No pre-ticked consent boxes. | Legally required for consent-based marketing in the EU/UK, and it keeps the list clean. A pre-ticked box is invalid consent under GDPR — this is not a grey area. |
| D-1006 | **Newsletter list in self-hosted Listmonk**; transactional mail through a managed provider. | Subscriber data stays on the user's VPS; deliverability stays managed. Best of both. |
| D-1007 | **Gated downloads are HMAC-signed, single-use, 7-day-expiry tokens**, served by a route handler. PDFs are never publicly linked. | Otherwise the direct PDF URL circulates and the gate becomes decorative. |
| D-1008 | **Every lead is written to Postgres before any email is sent**, and email failures never fail the submission. | The lead is the asset. If the email provider is down, sales can still see the request. Reversing this loses leads silently. |
| D-1009 | **An `EmailLog` row is written for every outbound message.** | When a prospect says "I never got it", the answer must be checkable. |
| D-1010 | **Explicit retention and erasure:** demo requests kept 24 months, unconfirmed newsletter signups purged after 7 days, a documented erasure procedure. | GDPR requires a defined retention period. "Forever" is not one. |
| D-1011 | **Field count is minimised.** Demo form: 5 fields. Every added field measurably reduces completion. | Company size and role are worth the cost; phone number and "how did you hear about us" are not. |
| D-1012 | **Real thank-you pages at real URLs**, not client-side state swaps. | Measurable as conversions, linkable, and they survive a refresh. |
| D-1013 | **CRM sync is designed but not wired** (OQ-1001). An `externalRef` column and a documented webhook shape exist from day one. | Adding the column later means a migration on a live table; adding it now costs nothing. |
| D-1014 | **Two separate enquiry forms and two separate tables** — `ServicesEnquiry` and `DemoRequest` — not one table with a `type` column. | Their fields genuinely differ, their follow-up differs, and their retention may differ. A shared table would be half-empty in both directions and would push the divergence into application code. |
| D-1015 | **Both forms capture "what stage are you at"**, in their own vocabulary. | It is the single most useful qualifying field for routing and prioritisation, and it costs one tap. |
| D-1016 | **An NDA mention on the services form**, not a checkbox. | Pharma buyers discussing internal systems need to know it is available. Making it a form field implies bureaucracy before the first conversation. |

## Forms in v1

| Form | Location | Fields | Outcome |
|---|---|---|---|
| **Services enquiry** | `/contact` | 6 | Postgres + notification + auto-reply → `/thank-you/enquiry` |
| **Demo request** | `/demo` | 6 | Postgres + notification + auto-reply → `/thank-you/demo` |
| Newsletter | Footer, `/resources`, blog posts | 2 | Listmonk double opt-in → `/thank-you/newsletter` |
| Gated download | `/resources/guides/[slug]` | 3 | Postgres + email with download link → `/thank-you/guide` |

### Two enquiry forms, not one (D-1014)

The site has two buyers with two different asks, and one form cannot serve both:

| | Services enquiry (`/contact`) | Demo request (`/demo`) |
|---|---|---|
| Who | QA / IT / operations lead at a regulated manufacturer | HR / operations lead |
| Asking for | A conversation about a problem | To see a specific product |
| Needs to tell us | Which service, what stage they're at, what systems | Which product, headcount, what they want to see |
| Urgency | Weeks to months | Days to weeks |
| Follow-up | A consultant | A product conversation |
| Routing | Different inbox or different owner (OQ-1002) |

Merging them into a "contact us" form with a dropdown would fit neither — and would lose the
qualifying information that makes the follow-up useful. The cost is one extra Server Action and one
extra table; the benefit is that every lead arrives pre-qualified.

Both forms share the same pipeline, spam defence, rate limiting and email infrastructure. Only the
schema, the recipient and the copy differ.

## Dependencies

- **Depends on:** 01 (Prisma, Zod, env), 02 (form components), 05 (`/contact` and `/demo` pages),
  09 (guide pages), and OQ-010 (email provider + SPF/DKIM/DMARC — **blocking for production**).
- **Depended on by:** 11 (conversion events), 13 (E2E tests cover every form).

## Tasks

| ID | Task | Done when |
|---|---|---|
| T-1001 | Prisma schema per [data-model.md](./data-model.md); initial migration. | `prisma migrate deploy` clean |
| T-1002 | `lib/schemas/` — Zod schemas for all four forms. | Unit-tested including boundary cases |
| T-1003 | `lib/email/client.ts` provider wrapper + React Email templates. | A test send arrives, renders in Gmail and Outlook |
| T-1004 | Server Action `submitDemoRequest`. | Writes the row, sends both emails, redirects |
| T-1005 | Server Action `submitServicesEnquiry`. | Writes the row, sends both emails, redirects to `/thank-you/enquiry`; notification subject line leads with stage and interest |
| T-1006 | Server Action `subscribeNewsletter` + `/api/newsletter/confirm`. | Double opt-in round-trip works end to end |
| T-1007 | Server Action `requestGatedDownload` + `/api/download/[token]`. | Token expires; second use is refused |
| T-1008 | `lib/rate-limit.ts` + cleanup job. | 6th submission in an hour from one IP is blocked |
| T-1009 | Turnstile integration, both widget and server verification. | Fails closed if the verification call errors |
| T-1010 | Form components with all states: idle, validating, submitting, success, error, rate-limited. | Every state designed, not just the happy path |
| T-1011 | Listmonk on the VPS, list created, API credentials in env. | Confirmed subscriber appears in the list |
| T-1012 | SPF, DKIM, DMARC on the sending domain (OQ-010). | `mail-tester.com` score ≥ 9/10 |
| T-1013 | Retention cleanup job (D-1010). | Runs nightly; verified against seeded old rows |
| T-1014 | GDPR documentation: lawful basis per form, retention, erasure procedure. | Written into `/legal/privacy` |
| T-1015 | E2E tests: happy path, validation failure, honeypot trip, rate limit, expired token. | Playwright green |

## Open questions

| ID | Question | Why it matters | Proposed default |
|---|---|---|---|
| OQ-1001 | Which CRM, if any? | Decides whether leads are worked in an inbox or a pipeline. | Postgres + email in v1; `externalRef` column reserved (D-1013) |
| OQ-1002 | Who receives demo notifications, and is there a shared inbox? | A notification to one person's personal address is a single point of failure for revenue. | A shared alias (`sales@vexalid.com`) — set this up before launch |
| OQ-1003 | Should `/demo` offer calendar booking (Cal.com self-hosted) instead of a form? | Booking directly converts better but requires real availability and discipline. | Form in v1; self-hosted Cal.com is a strong Phase 4 candidate given the VPS |
| OQ-1004 | Is a phone number field wanted? | It raises lead quality and lowers completion rate. | Optional field, clearly marked optional (D-1011) |
| OQ-1005 | Auto-reply from a person or a generic address? | A reply from a named human converts better and gets replies. | Named sender, `reply-to` the shared alias |
| OQ-1006 | Is Cloudflare acceptable given the self-hosting stance (Turnstile is a third-party script)? | It is the one third-party script on the site. | Yes — Turnstile is privacy-preserving and needs no consent banner. Alternative: Altcha, self-hosted, if strict first-party is required |
| OQ-1007 | Retention: is 24 months right for demo requests? | Longer needs justification under GDPR minimisation. | 24 months from last contact, then delete |
