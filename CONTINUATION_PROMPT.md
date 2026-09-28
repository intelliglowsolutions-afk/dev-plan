# Continuation prompt — HRM System

Paste everything below the line into a new chat to pick the project up.

---

I'm continuing work on an HRM system. Read `C:\Dev\dev-plan\IMPLEMENTATION_LOG.md` first — it is the
single source of truth for history and decisions, and **you must add a dated entry to it before
your session ends** (this rule is in `hrm-system/CLAUDE.md`).

## Where things are

- `C:\Dev\dev-plan` — the plan. **Authoritative.** Git repo, 2 commits. Planning is complete
  (Steps 0–6): 11 features × 5 files, plus `00-overview.md`, `MULTI-TENANCY.md`,
  `DEVICE-INGESTION-SECURITY.md`, `TESTING_STRATEGY.md`, `DEFINITION_OF_DONE.md`,
  `OPEN_QUESTIONS.md`.
- `C:\Dev\hrm-system` — the app. Git repo, 3 commits. Remote is
  `github.com/intelliglowsolutions-afk/hrm-system`. **Nothing has been pushed** — push needs my
  credentials; ask me, don't handle tokens.
- `C:\Dev\Datasheets` — SenseFace 2A manuals (OQ-000, already read and mined).

**Built and working:** multi-tenant Prisma schema (features 01/02 + Location), 4 migrations applied,
Row-Level Security verified, three database roles, the permission/authorization/audit/scope contract
layer, Auth.js wiring, the seed, `protectedRoute`, and `GET /api/employees`.

**Test status: 113 unit tests, 25 database assertions, typecheck clean.** Keep it that way.

**The honest gap: nothing has ever been exercised over HTTP.** `next dev` has not been run. The
Auth.js integration is unproven end to end. That is step 1 below.

## Environment — read this before running anything

This machine has **no Node, no npm, no git** installed. Docker is installed and everything runs in
containers. Python exists at `C:\Users\Intelliglow\anaconda3\python.exe`.

**Docker Desktop is usually stopped** after a gap. Start it and wait for the daemon:

```powershell
Start-Process "$env:LOCALAPPDATA\Programs\DockerDesktop\Docker Desktop.exe"
# then poll `docker info` until it succeeds
docker compose -f 'C:\Dev\hrm-system\docker-compose.yml' up -d postgres
```

Commands that are known to work:

```powershell
# Unit tests
docker run --rm -v "C:\Dev\hrm-system:/app" -w /app node:20-alpine npx vitest run

# Typecheck — Vitest does NOT typecheck, and tsc has already caught a real runtime bug
docker run --rm -v "C:\Dev\hrm-system:/app" -w /app node:20-alpine npx tsc --noEmit

# Database tests (reloads the fixture; destructive to local data)
powershell -ExecutionPolicy Bypass -File 'C:\Dev\hrm-system\test\db\run.ps1'

# Migrations / seed — note the OWNER connection, and the compose network
$u='postgresql://hrm_user:hrm_password@postgres:5432/hrm_db?schema=public'
docker run --rm --network hrm-system_default -v "C:\Dev\hrm-system:/app" -w /app `
  -e DATABASE_URL="$u" node:20-alpine npx prisma migrate deploy

docker run --rm --network hrm-system_default -v "C:\Dev\hrm-system:/app" -w /app `
  -e DATABASE_URL="$u" -e SEED_ADMIN_EMAIL="admin@acme.test" `
  -e SEED_ADMIN_PASSWORD="change-me-on-first-login" node:20-alpine npx tsx prisma/seed.ts

# Git (containerised; no host install)
docker run --rm --entrypoint sh -v "C:\Dev\hrm-system:/repo" -w /repo alpine/git:latest `
  -c 'git config --global --add safe.directory /repo; git status --porcelain'
```

Database roles — this distinction is load-bearing:

| Role | Use | Note |
|---|---|---|
| `hrm_user` | migrations, seed, fixture | **superuser — bypasses RLS entirely** |
| `hrm_app` | the application | non-superuser, RLS enforced |
| `hrm_auth` | Auth.js sign-in only | `BYPASSRLS`, granted on auth tables only |

## Traps already discovered — do not rediscover these

1. **Superusers bypass RLS even with `FORCE`.** The app must connect as `hrm_app`. This shipped
   once and made every tenant policy silently inert.
2. **PowerShell `Set-Content -Encoding utf8` writes a BOM**, and Postgres rejects it. Use
   `[System.IO.File]::WriteAllText($p,$t,(New-Object System.Text.UTF8Encoding($false)))`.
3. **PowerShell 5.1 treats native stderr as failure.** `psql` writes `RAISE NOTICE` to stderr, so
   `$ErrorActionPreference='Stop'` aborts on the first *passing* assertion. Use `$LASTEXITCODE`.
4. **Postgres cannot use a new enum value in the transaction that adds it** — new values need their
   own earlier migration.
5. **`overlaps` is a reserved word** in PL/pgSQL and produces a misleading syntax error.
6. **`prisma migrate dev` is interactive** and fails in a container. Use `prisma migrate diff` →
   review the SQL → hand-write the migration folder → `prisma migrate deploy`.
7. **Auth.js v5 Credentials always issues a JWT** and never calls `adapter.createSession`. Hence the
   design: the JWT carries only a session id, and our own `sessions` row is authoritative. Do not
   "simplify" this back to plain JWT — it would lose immediate revocation (01 D-04).
8. **`promisify(scrypt)` silently drops the options argument**, which would ignore the tuned cost
   parameters.
9. **Two seeders for the same data drift.** Role grants come only from the code catalogue via
   `prisma/seed.ts`; the test fixture deliberately seeds none.
10. `node_modules` came from macOS; `prisma` and `typescript` were broken and were reinstalled.
    If a CLI reports a missing bin, reinstall that one package.

## Rules to follow

- **`C:\Dev\CLAUDE.md`** — any UI work (pages, components, layout, states, copy) must load the
  `ui-ux-pro-max` skill first and be grounded in it. Prefer `minimalist-ui` for this dense admin
  product. Query the skill's CSVs directly or use the Anaconda Python path.
- **`hrm-system/AGENTS.md`** — read `node_modules/next/dist/docs/` before writing Next.js code.
  This is Next.js 16 and diverges from training data.
- Update `IMPLEMENTATION_LOG.md` before finishing. Commit both repos.

## Remaining steps, in order

**1. Prove the stack over HTTP.** Bring up `next dev` (compose `app` service), sign in as
`admin@acme.test`, and call `GET /api/employees`. Confirm: the forced password change blocks access,
the response is tenant-scoped, and sensitive fields are absent without `employee.read_sensitive`.
This is the milestone that turns everything so far from "typechecks" into "works".

**2. Sign-in and change-password screens.** UI work — load the skill first. Copy is specified in
`dev-plan/01-roles-permissions-auth/ui-ux.md`.

**3. Add the integration test layer** (TypeScript + real database). It does not exist yet; currently
there are only pure unit tests and SQL tests. `TESTING_STRATEGY.md` describes what belongs there.

**4. Finish feature 01** — users, roles, audit endpoints and screens.

**5. Finish feature 02** — employees CRUD, departments, positions, dated assignments, documents,
CSV import, org chart.

**6. Then, in order:** 03 → 04 (build the engine against synthetic punches first; ingestion needs
OQ-319) → 05 → 06 → 07 → 08 → 09 → 11 → **10 last** (my sequencing decision).

**Also outstanding:** features 01–04's `ui-ux.md` files predate the workspace design rules and are
not skill-grounded; 05–11 are. A retro-pass is owed.

## Decisions I still owe you

Blocking:
- **OQ-701** — the real pay components and how each is calculated. Blocks feature 07 entirely.
- **OQ-601b** — entitlement days for Annual / Casual / Medical. Blocks seeding feature 06.
- **OQ-319** — one hosted installation serving many tenants, or one install per company? Blocks
  attendance ingestion, and decides whether the per-site collector is needed.

Not blocking, but ask me:
- **Argon2id or scrypt?** The plan says Argon2id; scrypt is implemented because Argon2 bindings are
  native modules and this builds in Alpine. Swapping is a one-file change.
- **OQ-201 confirmation** — the union answer means a department head sees everyone in their
  department *including their own manager*. Asserted as intended; confirm it is.
- **Device-event retention sweep** — "no automatic deletion" was answered globally; does it extend
  to machine-generated device logs? Without either the sweep or transition-only logging, one
  terminal writes over a million rows a year.
- **OQ-003** — a GitHub token was pasted into chat on 2026-09-14 and has never been confirmed
  revoked. Please revoke it.
