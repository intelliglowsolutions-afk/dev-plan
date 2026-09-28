# 12 — Infrastructure & Deployment

**Priority:** Must-have · **Build order:** 12 of 13 (but **start in week one**) · **Status:** planned

## Purpose

Get the site running on the user's own VPS, reproducibly, with TLS, automated deploys, backups and
enough monitoring to know when it breaks — without it becoming a second full-time system to
administer.

## Scope

**In scope**

- VPS provisioning, hardening and the base software set.
- Docker image, `docker-compose` topology, nginx reverse proxy, TLS.
- CI/CD: GitHub Actions → build → deploy over SSH.
- Staging environment.
- Postgres setup, the `vexalid_web` role, and backups.
- Supporting services on the VPS: Plausible, Listmonk, optionally GlitchTip.
- Secrets management.
- Monitoring, uptime checks, log rotation, scheduled jobs.
- The rollback procedure.

**Out of scope (owned elsewhere)**

- Application code and configuration → segments 01–08.
- The HRM product's own infrastructure — **separate concern, and deliberately separate isolation
  boundary** (segment 10 data-model).
- Domain registration → OQ-001.

## Files

| File | Contents |
|---|---|
| [vps-runbook.md](./vps-runbook.md) | Provisioning, topology, deploy pipeline, backups, monitoring, runbooks |

## Key decisions taken in this plan

| # | Decision | Rationale |
|---|---|---|
| D-1201 | **Docker + `docker-compose`**, not a bare `node` process under PM2. | Reproducible builds, a trivial rollback (re-tag and restart), and identical behaviour locally and in CI. The learning cost is repaid the first time a deploy has to be undone at 11pm. |
| D-1202 | **Multi-stage Dockerfile** producing a Next.js `standalone` image on `node:22-alpine`, running as a non-root user. | A ~150 MB image instead of ~1.2 GB, and no root in the container. |
| D-1203 | **nginx on the host**, not containerised, terminating TLS and proxying to the app container. | The host nginx likely already serves other things; Certbot integration is well-trodden; one fewer container in the critical path. |
| D-1204 | **Let's Encrypt via Certbot**, auto-renewing on a systemd timer. | Free, automatic, and an expired certificate on a marketing site is a total outage in every browser. |
| D-1205 | **CI builds the image; the VPS only pulls and restarts.** | The VPS never needs a build toolchain, a `node_modules` tree, or the source. Deploys are fast and the box stays clean. |
| D-1206 | **Postgres runs on the host, not in a container.** | Data durability and backup tooling are simpler outside Docker, and the HRM product may already be using it. |
| D-1207 | **A real staging environment** at `staging.vexalid.com`, behind basic auth **and** `Disallow: /`. | Content review needs a real URL. Without basic auth, staging gets indexed — a months-long SEO problem (segment 11). |
| D-1208 | **Migrations run as an explicit deploy step**, never on container start. | Two containers starting at once must not race on `prisma migrate deploy`. |
| D-1209 | **Secrets live in a root-owned `.env` on the VPS** (mode `600`), injected by compose. GitHub Actions holds only the SSH key and registry credentials. | Proportionate. A secrets manager is a service to run and secure for a site with about a dozen secrets. |
| D-1210 | **Nightly `pg_dump`, 30-day retention, copied off the box, and restore-tested before launch.** | A backup that has never been restored is a hypothesis. This is the single most important item in the segment. |
| D-1211 | **External uptime monitoring** (UptimeRobot or Better Stack free tier) checking `/` and `/demo`. | A monitor running on the same VPS cannot tell you the VPS is down. |
| D-1212 | **Zero-downtime deploys are not attempted in v1.** A ~3-second restart is acceptable. | Blue-green on a single VPS adds real complexity for a marketing site with modest traffic. Revisit if it ever matters. |
| D-1213 | **`fail2ban`, UFW, key-only SSH, unattended security upgrades** from day one. | A public VPS is scanned continuously within minutes of coming online. This is baseline, not paranoia. |

## Topology

```
                        Internet
                           │
                    ┌──────▼──────┐
                    │   nginx     │  :443 TLS (Let's Encrypt)
                    │  (host)     │  :80 → 308 → :443
                    └──┬───┬───┬──┘
       vexalid.com     │   │   │   analytics.vexalid.com
       staging.…       │   │   │   mail.vexalid.com (Listmonk)
                       │   │   │
        ┌──────────────▼┐ ┌▼──────────────┐ ┌▼────────────┐
        │ web  (:3000)  │ │ staging(:3001)│ │ plausible   │
        │ Next standalone│ │               │ │ listmonk    │
        └───────┬───────┘ └───────┬───────┘ └──────┬──────┘
                │                 │                │
        ┌───────▼─────────────────▼────────────────▼───────┐
        │        PostgreSQL (host)                         │
        │  vexalid_web · vexalid_web_staging · plausible ·  │
        │  listmonk        (separate roles, no cross-grants)│
        └──────────────────────────────────────────────────┘
```

The HRM product, if it shares this box, keeps its **own database and its own role**, and the
marketing site's role has no grants on it (segment 10). If resources allow, a separate VPS entirely
is better — a public marketing site is the more exposed surface of the two.

## Dependencies

- **Blocked on OQ-007** — VPS specs, OS, what already runs on it, and whether the HRM shares it.
  Everything here assumes Ubuntu 24.04, 2 vCPU / 4 GB minimum, Docker available.
- **Depends on:** OQ-001 (domain and DNS control), OQ-008 (email domain records).
- **Depended on by:** every segment that needs a deployed URL. **Stand staging up in week one** —
  environment problems discovered at launch are the most expensive kind.

## Tasks

| ID | Task | Done when |
|---|---|---|
| T-1201 | Confirm VPS specs and existing services (OQ-007). | Documented in [vps-runbook.md](./vps-runbook.md) |
| T-1202 | Harden the box: UFW (22/80/443 only), key-only SSH, `fail2ban`, unattended-upgrades, a non-root deploy user. | `ssh` as root with a password fails |
| T-1203 | Install Docker Engine + compose plugin, nginx, Certbot, Postgres 17. | Versions recorded |
| T-1204 | DNS: A/AAAA for apex, `www`, `staging`, `analytics`, `mail`. | All resolve |
| T-1205 | Create `vexalid_web` and `vexalid_web_staging` databases and roles; verify no cross-database grants. | A cross-database query is refused |
| T-1206 | Write the multi-stage Dockerfile (D-1202). | Image < 200 MB, runs as non-root |
| T-1207 | Write `docker-compose.yml` for production and staging. | `docker compose up -d` serves both |
| T-1208 | nginx config: TLS, HSTS, gzip + brotli, static caching, security headers, `x-forwarded-for` **overwritten** not appended (segment 10 depends on this). | SSL Labs grade A |
| T-1209 | Certbot certificates + renewal timer; test a dry-run renewal. | `certbot renew --dry-run` clean |
| T-1210 | GitHub Actions CI: typecheck, lint, test, build, E2E. | Fails on a broken commit |
| T-1211 | GitHub Actions CD: build and push the image, SSH deploy, `prisma migrate deploy`, restart, health-check, auto-rollback on failure. | A push to `main` deploys |
| T-1212 | Staging deploy on every PR + basic auth + `Disallow: /` (D-1207). | Staging returns 401 without credentials |
| T-1213 | Deploy Plausible (D-1101) and Listmonk (segment 10). | Both reachable, both behind TLS |
| T-1214 | Backups: nightly `pg_dump`, 30-day retention, off-box copy — **and a documented restore test** (D-1210). | Restore verified into a scratch database |
| T-1215 | Scheduled jobs: retention cleanup (segment 10 T-1013), Listmonk reconciliation, log rotation. | Systemd timers active, logging counts |
| T-1216 | External uptime monitoring on `/` and `/demo` (D-1211). | An alert fires during a deliberate test outage |
| T-1217 | Error monitoring (Sentry or GlitchTip — OQ-105). | A test exception appears |
| T-1218 | Write the runbooks: deploy, rollback, restore, certificate failure, disk full. | A person who did not build this could follow them |
| T-1219 | Resource headroom check under a synthetic load test. | Memory and CPU documented at 10× expected traffic |

## Open questions

| ID | Question | Why it matters | Proposed default |
|---|---|---|---|
| OQ-1201 | VPS provider, specs, OS, and what already runs there? (= OQ-007) | **Blocking.** Everything in this segment assumes an answer. | Ubuntu 24.04, 2 vCPU / 4 GB / 50 GB SSD minimum. Plausible and Listmonk together want ~1.5 GB |
| OQ-1202 | Does the HRM product share this VPS? | Isolation, resource contention, and blast radius. | Separate VPS strongly preferred. If shared: separate databases, roles, containers and nginx server blocks — and accept that a marketing-site compromise is then closer to the HRM than anyone would like |
| OQ-1203 | Is Cloudflare in front of the VPS (CDN, DDoS, caching)? | Changes the nginx real-IP configuration, which segment 10's rate limiting depends on. | Recommended — free tier, hides the origin IP, absorbs traffic spikes. Configure `set_real_ip_from` for Cloudflare ranges if so |
| OQ-1204 | Where do off-box backups go? | A backup on the same disk as the database is not a backup. | Provider object storage (S3-compatible) or a second host, encrypted at rest |
| OQ-1205 | Container registry: GHCR, Docker Hub, or self-hosted? | Affects CD credentials. | GHCR — free for private images, and already authenticated by GitHub Actions |
| OQ-1206 | Who is on call, and where do alerts go? | An alert nobody receives is decoration. | Email + a phone push via the monitoring service, to a named person |
| OQ-1207 | Is a CDN needed for images and static assets? | Next's static output is already cacheable; a VPS may be slow to distant regions. | Cloudflare's free CDN in front (OQ-1203) covers it. No separate CDN |
