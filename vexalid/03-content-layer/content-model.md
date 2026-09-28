# 03 — Content Layer — Content Model

## Collections

| Collection | Path | Route | Type |
|---|---|---|---|
| **Services** | `content/services/<slug>.mdx` | `/services/<slug>` | `Service` |
| **Products** | `content/products/<slug>.mdx` | `/products/<slug>` | `Product` |
| **HRM modules** | `content/products/hrm/modules/<slug>.mdx` | `/products/hrm/<slug>` | `Module` |
| **Industries** | `content/industries/<slug>.mdx` | `/industries/<slug>` | `Industry` |
| Blog posts | `content/blog/YYYY-MM-<slug>.mdx` | `/resources/blog/<slug>` | `Post` |
| Guides | `content/guides/<slug>.mdx` | `/resources/guides/<slug>` | `Guide` |
| Legal | `content/legal/<slug>.mdx` | `/legal/<slug>` | `LegalPage` |
| Careers | `content/careers/<slug>.mdx` | `/careers/<slug>` | `JobPosting` (Phase 4) |
| Authors | `content/authors/<slug>.json` | — | `Author` |
| Site copy | `content/site/*.ts` | — | typed constants |

The module directory nests under its product (`content/products/hrm/modules/`) rather than sitting at
the root. If a second product later gains modules, the shape already accommodates it — and it keeps
the two-track structure visible in the content tree as well as the URLs.

The date prefix on blog filenames is for human ordering in a file listing only. The **slug is the
frontmatter `slug`**, or the filename with the date prefix stripped — never the raw filename, so a
post can be re-dated without breaking its URL.

## Frontmatter schemas

Shared base, on every MDX collection:

```ts
const BaseSchema = z.object({
  title:       z.string().min(10).max(70),   // also the <title>; 70 chars is the SERP cutoff
  description: z.string().min(70).max(160),  // meta description; 160 is the SERP cutoff
  slug:        z.string().regex(/^[a-z0-9]+(?:-[a-z0-9]+)*$/),
  draft:       z.boolean().default(false),
  ogImage:     z.string().optional(),        // path under /public/og/, falls back to generated
  noindex:     z.boolean().default(false),
})
```

The min/max bounds are deliberate: they turn "someone forgot the meta description" and "this title
gets truncated in Google" into build failures rather than SEO debt discovered months later.

### `Post` — blog

```ts
BaseSchema.extend({
  publishedAt: z.string().date(),                    // ISO YYYY-MM-DD
  updatedAt:   z.string().date().optional(),
  author:      z.string(),                           // → content/authors/<slug>.json, existence checked
  category:    z.enum([
                 'validation',      // CSV, GxP, data integrity — the services audience
                 'automation',      // workflow and AI automation
                 'hr-operations',   // HR process and policy — the products audience
                 'payroll',
                 'compliance',      // records, audit, data protection — spans both
                 'product',         // releases, how-tos
                 'company',
               ]),
  tags:        z.array(z.string()).max(6).default([]),
  featured:    z.boolean().default(false),           // at most 1 may be true — build-checked
  heroImage:   z.string().optional(),
  heroAlt:     z.string().optional(),                // required if heroImage is present
})
```

Derived by the adapter, never authored: `readingTime`, `tableOfContents`, `url`.

### `Guide` — long-form, optionally gated

```ts
BaseSchema.extend({
  publishedAt: z.string().date(),
  updatedAt:   z.string().date().optional(),
  gated:       z.boolean().default(false),
  pdfPath:     z.string().optional(),   // under /public/downloads/ — required when gated
  pageCount:   z.number().int().positive().optional(),
  summary:     z.array(z.string()).min(3).max(6),  // "what you'll learn" bullets on the gate
  formHeading: z.string().optional(),
})
```

`gated: true` without `pdfPath` fails the build. A gated guide renders its `summary`, an excerpt and
the capture form; the full body is **never sent to the client** for a gated guide — the excerpt is
split at an explicit `<!--more-->` marker server-side, so the gate cannot be bypassed by reading the
page source. (A gate that ships the whole document in the HTML is not a gate.)

### `Service` — service pages

```ts
BaseSchema.extend({
  name:         z.string(),                       // "Computerized System Validation"
  number:       z.enum(['01','02','03']),         // keeps the live site's numbering
  order:        z.number().int(),
  icon:         z.string(),
  tagline:      z.string().max(120),
  forWhom:      z.array(z.object({                // role + situation (06 D-604)
                  role: z.string(),
                  situation: z.string(),
                })).min(2).max(4),
  capabilities: z.array(z.object({
                  title: z.string(),
                  body:  z.string(),
                  icon:  z.string().optional(),
                })).min(3).max(6),
  deliverables: z.array(z.string()).min(3).max(10),   // WHAT THE CLIENT RECEIVES — 06 D-602
  process:      z.array(z.object({ title: z.string(), body: z.string() })).optional(),
  faqs:         z.array(z.object({ q: z.string(), a: z.string() })).min(3).max(8),
  relatedProducts: z.array(z.string()).max(2).default([]),   // cross-track link, 06 D-605
  reviewedBy:   z.string().min(2),                // practitioner sign-off — 06 D-607
  reviewedAt:   z.string().date(),
})
```

`reviewedBy` and `reviewedAt` are **required, not optional**. A service page with no recorded
reviewer fails the build. This is the mechanism that keeps unreviewed regulatory language off the
site — a convention nobody can forget, because the build enforces it.

`deliverables` has a minimum of three for the same reason `description` has a minimum length: a
service page that cannot name what the client receives is not finished.

### `Product` — product pages

```ts
BaseSchema.extend({
  name:         z.string(),                       // "StratumOne"
  order:        z.number().int(),
  icon:         z.string(),
  tagline:      z.string().max(120),
  status:       z.enum(['available','in-development','planned']),
  hasModules:   z.boolean().default(false),       // true for the HRM
  capabilities: z.array(z.object({
                  title: z.string(),
                  body:  z.string(),
                  icon:  z.string().optional(),
                })).min(3).max(8),
  deployment:   z.array(z.enum(['cloud','self-hosted','on-premises'])).min(1),
  faqs:         z.array(z.object({ q: z.string(), a: z.string() })).max(8).default([]),
  relatedServices: z.array(z.string()).min(1).max(3),   // cross-track link, 07 D-705
})
```

`relatedServices` has a **minimum of one**: a product page that does not link back into the services
track breaks the positioning (07 D-705), so the schema makes it impossible to omit.

`deployment` drives the self-hosting prominence required by 07 D-709.

### `Industry` — industry pages

```ts
BaseSchema.extend({
  name:            z.string(),                    // "Pharmaceutical"
  tagline:         z.string().max(120),
  relatedServices: z.array(z.string()).min(1),
  relatedProducts: z.array(z.string()).min(1),    // both tracks — 08 D-802
  faqs:            z.array(z.object({ q: z.string(), a: z.string() })).min(3).max(8),
  reviewedBy:      z.string().min(2),             // 08 D-803
  reviewedAt:      z.string().date(),
})
```

Both `relatedServices` and `relatedProducts` are required and non-empty. An industry page that links
into only one track is not doing the job the page exists for.

### `Module` — HRM product module pages

```ts
BaseSchema.extend({
  name:         z.string(),                          // "Attendance Tracking"
  order:        z.number().int(),                    // nav and overview ordering
  icon:         z.string(),                          // lucide icon name, validated against the set
  tagline:      z.string().max(120),
  status:       z.enum(['available','in-development','planned']),
  planRef:      z.string(),                          // e.g. "dev-plan/04-attendance-tracking"
  capabilities: z.array(z.object({
                  title: z.string(),
                  body:  z.string(),
                  icon:  z.string().optional(),
                })).min(3).max(8),
  faqs:         z.array(z.object({ q: z.string(), a: z.string() })).max(8).default([]),
  relatedModules: z.array(z.string()).max(3).default([]),
})
```

`status` and `planRef` are the honesty mechanism. `planRef` points at the HRM plan folder that
defines the module; `status` drives a visible `Coming soon` badge and suppresses "available today"
phrasing. **A module page for something not yet built must say so** — overpromising to an HR buyer
who then evaluates the product is worse than a smaller feature list. Cross-checked against the claim
register in [05 copy-guidelines.md](../05-home-company/copy-guidelines.md) and the page specs in
[07 Products Pages](../07-products-pages/README.md).

The eight module slugs, mapping to the HRM plan:

| Slug | HRM plan folder |
|---|---|
| `employee-records` | `dev-plan/02-employee-management` |
| `attendance` | `dev-plan/04-attendance-tracking` |
| `leave-management` | `dev-plan/06-leave-management` |
| `payroll` | `dev-plan/07-payroll` |
| `employee-self-service` | `dev-plan/08-employee-self-service` |
| `performance` | `dev-plan/09-performance-management` |
| `recruitment` | `dev-plan/10-recruitment-onboarding` |
| `reports` | `dev-plan/11-reports-analytics` |

### `LegalPage`

```ts
BaseSchema.extend({
  lastUpdated: z.string().date(),   // required (D-309), rendered at the top of the page
  version:     z.string().optional(),
})
```

### `Author`

```json
{
  "slug": "vexalid-team",
  "name": "Vexalid Team",
  "role": "",
  "bio": "",
  "avatar": "/images/authors/vexalid-team.png",
  "links": { "linkedin": "", "x": "" }
}
```

## Adapter API — `lib/content/index.ts`

The complete surface. Nothing else touches the file system.

```ts
// Blog
listPosts(opts?: { category?: Category; tag?: string; limit?: number; includeDrafts?: boolean }): Promise<PostMeta[]>
getPost(slug: string): Promise<Post | null>
getFeaturedPost(): Promise<PostMeta | null>
getRelatedPosts(slug: string, limit?: number): Promise<PostMeta[]>
listCategories(): Promise<{ category: Category; count: number }[]>

// Guides
listGuides(opts?: { includeDrafts?: boolean }): Promise<GuideMeta[]>
getGuide(slug: string): Promise<Guide | null>

// Services
listServices(): Promise<ServiceMeta[]>          // ordered by `order`
getService(slug: string): Promise<Service | null>

// Products
listProducts(): Promise<ProductMeta[]>          // ordered by `order`
getProduct(slug: string): Promise<Product | null>

// Modules (HRM)
listModules(product?: string): Promise<ModuleMeta[]>   // ordered by `order`; defaults to 'hrm'
getModule(slug: string, product?: string): Promise<Module | null>

// Industries
listIndustries(): Promise<IndustryMeta[]>
getIndustry(slug: string): Promise<Industry | null>

// Legal
listLegalPages(): Promise<LegalMeta[]>
getLegalPage(slug: string): Promise<LegalPage | null>

// Authors
getAuthor(slug: string): Promise<Author | null>

// Cross-cutting — used by sitemap.ts and rss
listAllContentUrls(): Promise<{ url: string; lastModified: Date; changeFrequency: string; priority: number }[]>
```

**Contract rules**
1. Every function is `async` even where the file implementation is synchronous. A CMS will be async;
   making callers async now avoids rewriting them later.
2. `list*` returns `*Meta` (frontmatter + derived fields, **no compiled body**). `get*` returns the
   full object including the serialised MDX. A list page must never compile 50 post bodies.
3. `get*` returns `null` for a missing or draft-in-production item; pages call `notFound()`.
4. Results are wrapped in `React.cache` so the content directory is read once per build.
5. Default ordering: `publishedAt` descending for posts and guides, `order` ascending for modules.
6. All returned dates are `Date` objects; all returned URLs are site-root-relative paths.

## MDX pipeline

**Remark**
- `remark-gfm` — tables, strikethrough, task lists, autolinks.
- `remark-reading-time` — populates the derived `readingTime`.
- A small custom plugin extracting `h2`/`h3` into `tableOfContents`.

**Rehype**
- `rehype-slug` — stable heading ids.
- `rehype-autolink-headings` — `behavior: 'append'`, a visually-hidden "Link to section" label so
  the anchor is announced rather than read as a bare "#".
- `rehype-pretty-code` (Shiki) — build-time syntax highlighting, so **no highlighter ships to the
  client**. Themes: `github-light` / `github-dark`, switched by CSS variables.
- A custom plugin marking external links `target="_blank" rel="noopener noreferrer"` and appending a
  visually-hidden "(opens in a new tab)".

## MDX component allowlist (D-305)

Available inside content without an import:

| Component | Use | Props |
|---|---|---|
| `Callout` | Note / tip / warning | `type: 'note'\|'tip'\|'warning'\|'important'`, `title?` |
| `Figure` | Image with caption | `src`, `alt` (**required**), `caption?`, `width?`, `height?` |
| `Cta` | Inline conversion block mid-article | `heading`, `body?`, `href`, `label` |
| `Stat` | Emphasised number | `value`, `label`, `source?` |
| `Video` | Self-hosted or privacy-mode embed | `src`, `title` (**required**), `poster?` |
| `Steps` | Numbered procedure | children |
| `Faq` | Q&A block that also emits FAQPage JSON-LD | `items[]` |

Standard elements (`h1`–`h4`, `p`, `a`, `ul`, `ol`, `blockquote`, `table`, `code`, `pre`, `img`, `hr`)
are mapped to token-styled components. Tables are auto-wrapped in `overflow-x: auto`. `img` is
rewritten to `MdxImage`.

Anything not on this list does not render. This is intentional (D-305).

## Images in content (D-306)

- Location: `public/images/content/<collection>/<slug>/<name>.<ext>`.
- Authors write `![Alt text](/images/content/blog/my-post/diagram.png)`.
- `MdxImage` reads real dimensions at build time via `sharp` (preventing layout shift), emits AVIF +
  WebP, lazy-loads everything except a `heroImage`, and applies `--vx-radius-lg`.
- **Missing `alt` fails the build.** Decorative images must be explicit: `![](...)` with an empty alt.
- Source images should be ≥ 1600px wide; a build warning fires below 1200px.

## CMS migration path (Phase 4)

When triggered (see 03 OQ-301):

1. Stand up **Payload CMS + Postgres** on the VPS at `cms.vexalid.com`, behind auth and IP-restricted
   if practical.
2. Model Payload collections to match the Zod schemas in this document exactly — they were written
   as a CMS schema in disguise.
3. Import the existing MDX as seed data via a one-off script.
4. Reimplement `lib/content/index.ts` against Payload's local/REST API. **The function signatures and
   return types do not change.** Rich text is stored as MDX or serialised to the same shape.
5. Add a webhook → `/api/revalidate` with `REVALIDATE_SECRET` (the env var is already reserved in
   [project-structure.md](../01-foundation-tech-stack/project-structure.md)), switching affected
   routes from pure SSG to ISR.
6. Delete `content/` once parity is verified, or keep it as the source for legal pages only.

Pages, components and tests are untouched by all of this — which is the entire return on D-302.

## Authoring workflow (v1)

Documented for a non-developer in the app repo's `CONTENT.md`:

1. Branch from `main` (`content/<short-description>`).
2. Add or edit the MDX file; add images under the matching `public/images/content/...` folder.
3. Push. CI validates frontmatter, builds, and posts a staging preview URL.
4. Review on staging (drafts are visible there).
5. Set `draft: false`, merge to `main` → production deploy.

A `content:`-prefixed commit skips the E2E suite in CI but **never** skips frontmatter validation.
