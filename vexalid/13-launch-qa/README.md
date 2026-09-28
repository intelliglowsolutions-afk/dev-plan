# 13 — Launch & QA

**Priority:** Must-have · **Build order:** 13 of 13 · **Status:** planned

## Purpose

Define what "done" means, and verify it before the site is public. This is a **gate**, not a phase —
the checks here run continuously from the first commit; the checklist is what gets signed off on
launch day.

## Scope

**In scope**

- Performance budgets and their enforcement.
- Accessibility testing and the WCAG 2.2 AA conformance target.
- Cross-browser and cross-device matrix.
- Functional test coverage: routes, forms, content, error paths.
- Content and copy QA.
- Security review.
- The launch checklist and the go/no-go criteria.
- Post-launch monitoring for the first fortnight.

**Out of scope (owned elsewhere)**

- The test harness choice → [01 Foundation](../01-foundation-tech-stack/stack-decisions.md).
- Infrastructure monitoring → [12 Infrastructure](../12-infrastructure-deployment/README.md).
- Copy authorship → [05 Core Pages](../05-home-company/README.md).

## Files

| File | Contents |
|---|---|
| [checklist.md](./checklist.md) | The full launch checklist, budgets, test matrix, and go/no-go criteria |

## Key decisions taken in this plan

| # | Decision | Rationale |
|---|---|---|
| D-1301 | **Performance budgets fail the build**, they are not warnings. | A budget that only warns is a budget that gets ignored. Marketing-site performance degrades one "just this one library" at a time. |
| D-1302 | **Accessibility: zero serious or critical axe violations on every route** is a release gate. | WCAG 2.2 AA is the procurement baseline for software sold to HR departments; it will be asked about. And retrofitting is far more expensive than not regressing. |
| D-1303 | **E2E tests assert every route returns 200 and every form's happy *and* unhappy paths.** | The forms are the site's entire commercial purpose. A silently broken demo form is the worst possible failure, because nothing alerts you — the traffic looks normal. |
| D-1304 | **Real-device testing on at least one physical phone.** Emulation is not sufficient. | Touch targets, iOS input zoom, sticky-header behaviour with the mobile URL bar, and font rendering all differ from DevTools. |
| D-1305 | **QA runs against the staging deploy, not `pnpm dev`.** | Only a production build exposes SSG output, real caching, real image optimisation and real bundle sizes. |
| D-1306 | **A copy read-through by a human who did not write it** is a required gate. | Every site ships with at least one embarrassing typo in an `h1`. The only reliable defence is fresh eyes. |
| D-1307 | **The launch checklist has explicit blocking items.** Non-blocking items may ship as known gaps and be listed in the implementation log. | Distinguishing "must" from "should" before launch day prevents both shipping something broken and delaying over something cosmetic. |
| D-1308 | **A fortnight of heightened post-launch monitoring**, with a daily check for the first week. | Most launch defects surface within days, from real traffic patterns nobody tested. |

## Testing layers

| Layer | Tool | Scope | Runs |
|---|---|---|---|
| Types | `tsc --noEmit` | Whole repo | Every commit |
| Lint | ESLint 9 | Whole repo, including the `content/` import rule | Every commit |
| Unit | Vitest | Content adapter, Zod schemas, helpers, email templating, rate limiter | Every commit |
| Component | Vitest + Testing Library | Form validation states, nav behaviour | Every commit |
| E2E | Playwright | Every route, every form, error paths | Every PR |
| Accessibility | `@axe-core/playwright` | Every route, light and dark | Every PR |
| Performance | Lighthouse CI | Home, product, module, blog post, demo | Every PR |
| Links | `lychee` | Built output, internal and external | Every PR + weekly cron |
| Visual | Manual against the `/design` gallery | Components in every state | Before release |
| Security | `pnpm audit` + a manual review | Dependencies, headers, forms | Before release + weekly |
| Email rendering | Manual, real clients | All five templates | Before release |
| Real device | Manual | Home, module, demo, blog post | Before release |

## Performance budgets (D-1301)

Measured by Lighthouse CI against the **staging production build**, mobile preset, simulated 4G.

| Metric | Budget | Fails build |
|---|---|---|
| Performance score (mobile) | ≥ 95 | < 90 |
| Accessibility score | 100 | < 100 |
| Best Practices | ≥ 95 | < 90 |
| SEO | 100 | < 100 |
| LCP | < 1.8s | > 2.5s |
| CLS | < 0.05 | > 0.1 |
| INP | < 200ms | > 300ms |
| TTFB | < 600ms | > 1s |
| Total JS (home) | < 120 KB gz | > 160 KB |
| Total JS (blog post) | < 90 KB gz | > 130 KB |
| Total page weight (home) | < 600 KB | > 900 KB |
| Font files | ≤ 2 faces, ≤ 90 KB total | > 3 faces |

The gap between "budget" and "fails build" is deliberate headroom — a build that fails on a 1-point
fluctuation gets disabled within a week.

## Dependencies

- **Depends on:** every other segment, plus a working staging deploy (09 T-1212) — needed from week
  one, not at the end.
- **Blocks:** launch.

## Tasks

| ID | Task | Done when |
|---|---|---|
| T-1301 | Playwright suite: every route in [../04-site-structure-navigation/sitemap.md](../04-site-structure-navigation/sitemap.md) returns 200 with an `h1`. | Green |
| T-1302 | Playwright: demo form happy path → row in the database → `/thank-you/demo`. | Green against staging |
| T-1303 | Playwright: demo form validation, honeypot trip, rate limit, Turnstile failure. | All four asserted |
| T-1304 | Playwright: newsletter double opt-in round trip, including a second submission of a confirmed address. | Green |
| T-1305 | Playwright: gated download — link works once, second use refused, expired token handled. | Green |
| T-1306 | `@axe-core/playwright` on every route, both themes. | Zero serious/critical (D-1302) |
| T-1307 | Manual keyboard-only pass: whole site, no mouse. | Every interactive element reachable and operable; focus never lost or trapped |
| T-1308 | Screen-reader spot check: NVDA or VoiceOver on home, a module page, and `/demo`. | Forms and nav are comprehensible |
| T-1309 | Lighthouse CI wired with the budgets above. | Fails on a deliberate regression |
| T-1310 | Cross-browser matrix per [checklist.md](./checklist.md). | All pass |
| T-1311 | Real-device test on a physical phone (D-1304). | No input zoom, no horizontal scroll, sticky header behaves |
| T-1312 | Email rendering in Gmail (web + app), Outlook (web + desktop), Apple Mail. | All five templates legible; plain-text alternatives present |
| T-1313 | Security review per the checklist. | No open findings |
| T-1314 | Copy read-through by someone who did not write it (D-1306). | Sign-off recorded |
| T-1315 | Content QA: every claim checked against the register (segment 05), every `status` accurate. | No overpromising — the highest-consequence check in the list |
| T-1316 | Legal review of privacy, terms, DPA, cookies (OQ-303). | **Blocking** |
| T-1317 | 404 and 500 verified in production conditions. | Useful, navigable, no stack traces |
| T-1318 | Backup restore test (09 T-1214). | **Blocking** |
| T-1319 | Full launch checklist walk-through with the user. | Signed off |
| T-1320 | Post-launch monitoring plan in place (D-1308). | Owner and cadence agreed |

## Open questions

| ID | Question | Why it matters | Proposed default |
|---|---|---|---|
| OQ-1301 | Is there a second person to do the copy read-through and the keyboard pass? | Self-QA misses the obvious. | If not, do the copy read-through aloud, 24 hours after writing — the delay is what makes it work |
| OQ-1302 | Which physical devices are available? | D-1304 requires at least one real phone. | One iOS and one Android if possible; iOS matters most for the input-zoom and sticky-header issues |
| OQ-1303 | Is a formal WCAG audit needed, or is internal testing sufficient for now? | Enterprise HR procurement sometimes asks for a VPAT. | Internal testing + documented conformance claim at launch; a formal audit if a deal requires it |
| OQ-1304 | Is a soft launch (unlinked, `noindex`) wanted before the announcement? | Lets real traffic find defects before the site is indexed and shared. | Recommended: one week live and `noindex`, then remove `noindex` and announce |
| OQ-1305 | What is the rollback trigger — who decides to pull the site? | Needs deciding before it is needed, not during. | Any of: demo form broken, TLS failure, a factually wrong claim published, a data exposure. Named decision-maker (OQ-1206) |
