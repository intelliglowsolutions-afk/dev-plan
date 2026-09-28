# 13 — Launch & QA — Checklist

Items marked **[BLOCKING]** prevent launch. Everything else may ship as a known gap, recorded in
`IMPLEMENTATION_LOG.md` (D-1307).

---

## 1. Functional

- [ ] **[BLOCKING]** Every route in the sitemap returns 200
- [ ] **[BLOCKING]** Demo form: submits, writes to the database, sends both emails, redirects to `/thank-you/demo`
- [ ] **[BLOCKING]** Demo form: the sales notification arrives in the **shared** inbox (OQ-1002), not one person's
- [ ] **[BLOCKING]** Newsletter: double opt-in round trip completes; subscriber appears in Listmonk
- [ ] **[BLOCKING]** Gated download: email arrives, link works once, second use refused, expiry honoured
- [ ] Contact form: submits and routes by subject; `DATA_REQUEST` reaches the privacy address
- [ ] Every form works with **JavaScript disabled** (D-1001 progressive enhancement)
- [ ] Validation errors are clear, specific, and announced to assistive technology
- [ ] Honeypot, timing check, rate limit and Turnstile all verified working
- [ ] 404 and 500 pages render with working navigation
- [ ] All internal links resolve — no 404s, no redirect chains
- [ ] All external links resolve and open in a new tab with `rel="noopener"`
- [ ] Header nav, mega-menu and mobile sheet all work
- [ ] Blog: index, category filter, pagination, post page, related posts, RSS
- [ ] `Log in` goes to `app.vexalid.com` and lands somewhere sensible
- [ ] Theme switching works in both directions, with no flash of the wrong theme

## 2. Content

- [ ] **[BLOCKING]** No `TODO-COPY` anywhere in the production build (segment 05 T-513)
- [ ] **[BLOCKING]** Every claim appears in the claim register with an accurate status (segment 05 D-504)
- [ ] **[BLOCKING]** No module page describes an unbuilt feature as available
- [ ] **[BLOCKING]** No fabricated testimonials, logos, statistics or ratings (D-503)
- [ ] **[BLOCKING]** Legal text is real and reviewed — privacy, terms, DPA, cookies (OQ-303)
- [ ] Privacy policy accurately describes what the forms collect, the lawful basis and the retention
- [ ] Copy read-through by someone who did not write it (D-1306)
- [ ] Spelling consistent in the chosen English variant
- [ ] Every page has one `h1`, and heading levels never skip
- [ ] Every image has meaningful `alt`, or an explicit `alt=""`
- [ ] Company legal name and contact details correct in the footer (OQ-001)
- [ ] Every blog post has been reviewed by a human for factual accuracy (OQ-902) — particularly
      anything touching payroll, leave entitlement or employment law

## 3. Accessibility (WCAG 2.2 AA)

- [ ] **[BLOCKING]** Zero serious or critical axe violations on every route, both themes (D-1302)
- [ ] **[BLOCKING]** Full keyboard operability — no mouse, whole site (T-1307)
- [ ] Visible focus indicator on every interactive element, both themes
- [ ] Skip link present, and it works
- [ ] Contrast AA verified for every token pair, both themes (segment 02 T-209)
- [ ] Touch targets ≥ 44×44px throughout
- [ ] Forms: labels associated, errors announced, `autocomplete` set
- [ ] `prefers-reduced-motion` honoured — verified by actually enabling it
- [ ] Screen-reader spot check on home, a module page, and `/demo` (T-1308)
- [ ] Page zooms to 200% with no loss of content or function
- [ ] `<html lang>` set
- [ ] No keyboard trap in the mobile nav, dialogs, or the Turnstile widget

## 4. Performance

- [ ] **[BLOCKING]** Lighthouse mobile ≥ 90 on every tested page
- [ ] LCP < 1.8s, CLS < 0.05, INP < 200ms on home, product and blog post
- [ ] JS budgets met (segment 13 README)
- [ ] All images AVIF/WebP with explicit dimensions
- [ ] Fonts self-hosted, subset, preloaded, `font-display: swap`
- [ ] No render-blocking third-party resources
- [ ] Static assets served with long-lived immutable caching
- [ ] gzip/brotli active
- [ ] No layout shift from the sticky header or from late-loading fonts

## 5. SEO

- [ ] **[BLOCKING]** `robots.txt` on production does **not** disallow everything — check the live file, not the source
- [ ] **[BLOCKING]** No `noindex` accidentally shipped to production — check the built HTML
- [ ] **[BLOCKING]** Staging is `Disallow: /` **and** behind basic auth (D-1207)
- [ ] Unique title and description on every page, within bounds
- [ ] Self-referencing canonical on every page
- [ ] `sitemap.xml` valid, complete, excludes drafts and thank-you pages
- [ ] Structured data passes the Rich Results Test with no errors
- [ ] OG images render correctly in the LinkedIn, X and Slack preview tools
- [ ] Search Console and Bing verified; sitemap submitted
- [ ] `www` and apex resolve consistently with one 308
- [ ] RSS validates

## 6. Security

- [ ] **[BLOCKING]** HTTPS everywhere; HTTP redirects; no mixed content
- [ ] **[BLOCKING]** No secrets in the repository, the image, or client-side code
- [ ] **[BLOCKING]** The `vexalid_web` database role has **no access** to the HRM database — tested, not assumed
- [ ] Security headers set (HSTS, CSP, `nosniff`, `Referrer-Policy`, `X-Frame-Options`, `Permissions-Policy`)
- [ ] SSL Labs grade A
- [ ] `pnpm audit` — no high or critical vulnerabilities
- [ ] Rate limiting verified, and **not bypassable via a spoofed `X-Forwarded-For`** (segment 12 nginx config)
- [ ] Gated PDFs are not reachable by direct URL
- [ ] Download and confirmation tokens are stored hashed, expire, and are single-use where intended
- [ ] Error pages leak no stack traces, file paths or error ids
- [ ] SSH is key-only; the firewall allows 22/80/443 only
- [ ] Postgres is not listening on a public interface
- [ ] Turnstile fails closed on a verification error

## 7. Infrastructure

- [ ] **[BLOCKING]** Backup restore test completed successfully (09 T-1214 / D-1210)
- [ ] **[BLOCKING]** External uptime monitoring live, with alerts reaching a named person
- [ ] Deploy pipeline works end to end, including auto-rollback on a failed health check
- [ ] Rollback procedure rehearsed at least once
- [ ] TLS auto-renewal tested (`certbot renew --dry-run`)
- [ ] Scheduled jobs active and logging (retention, reconciliation, prune)
- [ ] Error monitoring receiving events
- [ ] Disk usage under 50%, with the prune timer active
- [ ] Resource headroom verified at 10× expected traffic
- [ ] All runbooks written and followable by someone who did not build the system

## 8. Email

- [ ] **[BLOCKING]** SPF, DKIM and DMARC configured on the sending domain (OQ-008)
- [ ] **[BLOCKING]** `mail-tester.com` score ≥ 9/10
- [ ] All five templates render correctly in Gmail, Outlook and Apple Mail
- [ ] Every template has a plain-text alternative
- [ ] `List-Unsubscribe` and `List-Unsubscribe-Post` headers on newsletter mail
- [ ] Unsubscribe works and is honoured
- [ ] No tracking pixels, no external images
- [ ] Reply-to addresses are monitored

## 9. Analytics

- [ ] Plausible receiving pageviews from production
- [ ] All conversion events fire and appear as goals
- [ ] No personal data in any event property (D-1110)
- [ ] Script proxied through the site's own domain
- [ ] No third-party tracking script anywhere — verified in the network panel

## 10. Legal and compliance

- [ ] **[BLOCKING]** Privacy policy, terms, cookie notice and DPA published and reviewed
- [ ] **[BLOCKING]** No cookie banner needed — confirmed no cookies are set beyond strictly necessary
- [ ] Lawful basis documented per form (segment 10)
- [ ] Retention periods implemented **and running**, not merely documented
- [ ] Erasure procedure documented and tested once
- [ ] Subject access procedure documented
- [ ] Consent records capture the exact wording shown, plus timestamp and IP
- [ ] Newsletter consent is never pre-ticked, and is never bundled with the download

---

## Cross-browser matrix (T-1310)

| Browser | Versions | Priority |
|---|---|---|
| Chrome | Latest, latest−1 | Critical |
| Safari | Latest macOS, latest iOS | Critical — iOS Safari is where layout and input bugs actually appear |
| Firefox | Latest | High |
| Edge | Latest | High |
| Samsung Internet | Latest | Medium — a large share of Android B2B traffic |

**Viewports:** 360×640, 390×844, 768×1024, 1280×800, 1920×1080.

**Test on each:** home, one module page, `/demo` (with a submission), a blog post, the mobile nav.

---

## Go / no-go

**Launch if:** every `[BLOCKING]` item is checked, and the demo form has been submitted successfully
from a real phone on mobile data — not from the office wifi, not from a laptop.

**Do not launch if:** any blocking item is open; if legal text is placeholder; if any claim
overpromises an unbuilt feature; or if the backup restore has not been tested.

**Soft-launch option (OQ-1304):** deploy with `noindex`, leave it a week, watch real traffic, fix
what surfaces, then remove `noindex` and announce. Recommended — it converts launch-day surprises
into ordinary Tuesday bugs.

---

## Post-launch monitoring (D-1308, T-1320)

**Days 1–7, daily:**
- Uptime and error rate
- Demo submissions — **any day with zero is investigated, not assumed**
- `EmailLog` failure rate
- Search Console for crawl errors
- Real-user Core Web Vitals

**Days 8–14, every other day:** the same, plus top entry pages and bounce behaviour.

**Week 3 onward:** the weekly and monthly cadence in segment 11.

**First-month review:** conversion rate per path, which pages actually drive demo requests, which
content earns subscriptions, and what the open questions in these plans now have real answers for.
Write the answers back into the plan documents — a plan that is never updated after contact with
reality stops being useful.
