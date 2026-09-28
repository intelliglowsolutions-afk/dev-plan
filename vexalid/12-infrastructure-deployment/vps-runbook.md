# 12 — Infrastructure & Deployment — VPS Runbook

> Assumes Ubuntu 24.04 LTS, ≥ 2 vCPU / 4 GB RAM / 50 GB SSD. Confirm against OQ-1201 before executing.

## 1. Provisioning and hardening (T-1202, D-1213)

A VPS is port-scanned within minutes of coming online. This section is baseline, not optional.

| Step | Detail |
|---|---|
| Deploy user | Non-root `deploy` user, in the `docker` group, with an SSH public key |
| SSH | `PasswordAuthentication no`, `PermitRootLogin no`, key-only. Consider a non-standard port (noise reduction, not security) |
| Firewall | UFW default-deny inbound; allow 22, 80, 443 only. Postgres bound to `127.0.0.1` and **never** exposed |
| `fail2ban` | Enabled for `sshd` |
| Updates | `unattended-upgrades` for security patches; reboot window documented |
| Timezone / NTP | UTC, `systemd-timesyncd` on — token expiry and rate-limit windows depend on a correct clock |
| Swap | 2 GB swapfile — a Next build or a Plausible query spike will otherwise OOM-kill something |
| Monitoring agent | Node exporter or the provider's agent, for disk and memory alerts |

## 2. Base software (T-1203)

| Package | Version | Note |
|---|---|---|
| Docker Engine + compose plugin | latest stable | From Docker's own apt repository, not the Ubuntu one |
| nginx | 1.24+ | On the host (D-1203) |
| Certbot + `python3-certbot-nginx` | latest | Via apt or snap |
| PostgreSQL | 17 | On the host (D-1206) |
| `postgresql-client` | 17 | For `pg_dump` |
| Node.js | 22 LTS | For the scheduled retention scripts only — **not** for building |

## 3. DNS (T-1204)

| Record | Name | Value |
|---|---|---|
| A | `@` | VPS IPv4 |
| AAAA | `@` | VPS IPv6, if available |
| CNAME | `www` | `vexalid.com` → 308 to apex in nginx |
| A | `staging` | VPS IPv4 |
| A | `analytics` | VPS IPv4 |
| A | `mail` | VPS IPv4 (Listmonk's admin UI, not a mail server) |
| TXT | `@` | SPF for the email provider (OQ-008) |
| TXT | `<selector>._domainkey` | DKIM from the provider |
| TXT | `_dmarc` | `v=DMARC1; p=quarantine; rua=mailto:dmarc@vexalid.com` — start at `p=none`, tighten after a fortnight of reports |
| CAA | `@` | `0 issue "letsencrypt.org"` |

## 4. Database setup (T-1205)

```sql
CREATE ROLE vexalid_web LOGIN PASSWORD '<generated>';
CREATE DATABASE vexalid_web OWNER vexalid_web;
CREATE ROLE vexalid_web_staging LOGIN PASSWORD '<generated>';
CREATE DATABASE vexalid_web_staging OWNER vexalid_web_staging;

-- Isolation (segment 10 D-107): no access to anything else on this server
REVOKE ALL ON DATABASE postgres FROM vexalid_web;
REVOKE CONNECT ON DATABASE <hrm_database> FROM vexalid_web;
```

**Verify the isolation explicitly** — attempt a connection from `vexalid_web` to the HRM database and
confirm it is refused. An assumed boundary is not a boundary.

Also create `plausible` and `listmonk` databases with their own roles.

## 5. Dockerfile (T-1206, D-1202)

Four stages:

```dockerfile
# 1. deps      — node:22-alpine, pnpm, install with --frozen-lockfile
# 2. builder   — copy source, prisma generate, next build (output: standalone)
# 3. runner    — node:22-alpine, non-root `nextjs` user (uid 1001),
#                copy .next/standalone, .next/static, public/
# 4. CMD ["node", "server.js"]
```

Notes:
- `output: 'standalone'` in `next.config.ts` is what makes the ~150 MB image possible.
- `NEXT_PUBLIC_*` variables are **baked in at build time** — they are inlined into the client bundle.
  Staging and production therefore need **separate builds**, not one image with different runtime
  env. This surprises people; it is a property of how Next inlines public env vars, not a
  configuration mistake.
- `prisma generate` runs in the builder; the generated client is copied forward.
- `HEALTHCHECK` hitting `/api/health` (a trivial route returning 200 plus a database ping).
- `.dockerignore`: `node_modules`, `.next`, `.git`, `tests`, `*.md`, `.env*`.

## 6. `docker-compose.yml` (T-1207)

| Service | Image | Port | Notes |
|---|---|---|---|
| `web` | `ghcr.io/<owner>/vexalid-web:<tag>` | `127.0.0.1:3000` | `env_file: /srv/vexalid/.env.production`, `restart: unless-stopped`, log rotation via the `json-file` driver with `max-size: 10m` |
| `web-staging` | `…:staging-<sha>` | `127.0.0.1:3001` | `env_file: .env.staging` |
| `plausible` | official | `127.0.0.1:8000` | Needs ClickHouse as well — budget ~1 GB RAM |
| `listmonk` | `listmonk/listmonk` | `127.0.0.1:9000` | Points at the transactional provider's SMTP relay (segment 10 D-1006) |

Every port binds to `127.0.0.1` only. nginx is the sole public listener.

## 7. nginx (T-1208)

Per server block:

| Concern | Setting |
|---|---|
| TLS | `ssl_protocols TLSv1.2 TLSv1.3`, modern ciphers, `ssl_session_cache`, OCSP stapling |
| HSTS | `Strict-Transport-Security: max-age=63072000; includeSubDomains; preload` — only once you are certain every subdomain will always be HTTPS |
| Security headers | `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `X-Frame-Options: DENY`, `Permissions-Policy` denying camera/mic/geolocation |
| CSP | Start in `Content-Security-Policy-Report-Only`, then enforce. Must allow the Plausible origin and Cloudflare Turnstile. A CSP added after launch is far harder to get right than one tested from the start |
| Compression | gzip on; brotli if the module is available. `text/*`, `application/javascript`, `application/json`, `image/svg+xml` |
| Static caching | `/_next/static/` → `Cache-Control: public, max-age=31536000, immutable` (content-hashed filenames make this safe) |
| HTML caching | `Cache-Control: public, max-age=0, must-revalidate` — let the app decide |
| **Real IP** | `proxy_set_header X-Forwarded-For $remote_addr;` — **overwrite, not append** (`$proxy_add_x_forwarded_for`). Segment 10's rate limiting is bypassable if a client-supplied header survives. If Cloudflare is in front (OQ-1203), use `set_real_ip_from` with their ranges plus `real_ip_header CF-Connecting-IP` |
| Body size | `client_max_body_size 1m` — no uploads on this site |
| Redirects | `:80` → 308 `:443`; `www` → 308 apex |
| Staging | `auth_basic` + an `.htpasswd` file (D-1207) |
| Timeouts | `proxy_read_timeout 30s`; `proxy_connect_timeout 5s` |

## 8. CI — `.github/workflows/ci.yml` (T-1210)

On every push and pull request:

1. Checkout, setup pnpm + Node 22, restore cache.
2. `pnpm install --frozen-lockfile`.
3. `pnpm typecheck` · `pnpm lint` · `pnpm format:check`.
4. `pnpm test` (Vitest).
5. `pnpm build` — against a throwaway Postgres service container.
6. `pnpm test:e2e` (Playwright, Chromium + WebKit).
7. Lighthouse CI against the built output, with segment 13's budgets.
8. `lychee` link check.
9. **The `TODO-COPY` check** (segment 05 T-513) on production builds.

A `content:`-prefixed commit may skip steps 6–8 but never 3, 5 or 9.

## 9. CD — `.github/workflows/deploy.yml` (T-1211, D-1205)

On a push to `main`:

1. CI must pass.
2. `docker buildx build` with the production `NEXT_PUBLIC_*` values, tagged `:<sha>` and `:latest`.
3. Push to GHCR.
4. SSH to the VPS as `deploy`:
   ```
   docker compose pull web
   docker compose run --rm web pnpm db:deploy     # migrations, explicit step (D-1208)
   docker compose up -d --no-deps web
   ```
5. Health check `/api/health` — 10 attempts, 2 seconds apart.
6. **On failure: re-tag the previous image and restart** (auto-rollback), then fail the workflow
   loudly.
7. Post the deployed SHA to wherever the team will see it.

Roughly 3 seconds of downtime during the restart, accepted under D-1212.

On a pull request: the same, to `web-staging`, with a comment carrying the staging URL.

## 10. Secrets (D-1209)

| Location | Contents |
|---|---|
| `/srv/vexalid/.env.production` | Root-owned, mode `600`. Every server-side secret from segment 01's env table |
| `/srv/vexalid/.env.staging` | Same, with staging values and a **test** email provider key |
| GitHub Actions secrets | `SSH_PRIVATE_KEY`, `SSH_HOST`, `SSH_USER`, `GHCR_TOKEN`, and the build-time `NEXT_PUBLIC_*` values |

Rules: secrets are never in the image, never in the repository, never in a compose file. Rotate the
SSH deploy key and provider API keys annually, and immediately if anyone with access leaves. The
`.env` files are included in the backup set, encrypted separately from the database dumps.

## 11. Backups (T-1214, D-1210)

```
Nightly 02:00 UTC  — pg_dump -Fc vexalid_web > /srv/backups/vexalid_web-$(date +%F).dump
                   — gpg-encrypt, upload to off-box object storage (OQ-1204)
                   — prune local and remote copies older than 30 days
Weekly             — tar the content/ tree and public/ uploads (belt and braces; git already has these)
Monthly            — RESTORE TEST into a scratch database, verify row counts, then drop it
```

**The restore test is the deliverable, not the dump.** A backup that has never been restored is a
hypothesis. Do one before launch and record the result here.

Also backed up: `/srv/vexalid/.env.*` (encrypted, separately), the nginx configs, and the
`.htpasswd`.

## 12. Scheduled jobs (T-1215)

| Job | Schedule | Command |
|---|---|---|
| Retention cleanup | 03:00 daily | `docker compose run --rm web node scripts/retention.js` (segment 10 T-1013) |
| Listmonk reconciliation | hourly | Retry subscribers with a null `listmonkId` |
| Rate-limit table cleanup | 04:00 daily | Part of the retention script |
| Certificate renewal | twice daily | Certbot's systemd timer, with an nginx reload hook |
| Docker prune | 05:00 Sunday | `docker image prune -af --filter "until=168h"` — otherwise the disk fills with old image layers, which is the most common way a small VPS dies |
| Log rotation | daily | `logrotate` for nginx; the Docker `json-file` driver caps container logs |

Each job logs to `/var/log/vexalid/` and records a count. A retention job that silently stops is a
compliance failure nobody notices until an audit (segment 10 D-1010) — the uptime check should assert
that the job ran.

## 13. Monitoring (T-1216, T-1217, D-1211)

| Layer | Tool | Alerts on |
|---|---|---|
| External uptime | UptimeRobot / Better Stack free tier | `/` or `/demo` non-200, or > 3s response, from outside the VPS |
| Certificate expiry | Same service | < 14 days remaining |
| Application errors | Sentry or GlitchTip (OQ-105) | Any unhandled server exception; **any demo-form submission failure pages someone** — that is lost revenue |
| Host | Provider agent / node exporter | Disk > 80%, memory > 90%, CPU sustained > 80% |
| Database | A script in the retention job | Connection failure, `vexalid_web` size growth anomaly |
| Analytics sanity | Weekly manual (segment 11) | Zero demo requests in a week — almost always a broken form, not a quiet market |

Alerts go to a named person (OQ-1206). An alert channel nobody watches is decoration.

## 14. Runbooks (T-1218)

### Deploy
Push to `main`. Watch the Actions run. Verify the SHA at `/api/health`.

### Rollback
```bash
ssh deploy@vps
cd /srv/vexalid
docker compose pull web:<previous-sha>      # or use the local cached image
# edit the tag in .env / compose, then:
docker compose up -d --no-deps web
curl -f http://127.0.0.1:3000/api/health
```
**If the bad deploy included a migration**, roll the code back first, then assess the migration
separately. Never blind-revert a migration on live data — read it, decide, then act.

### Restore the database
```bash
createdb vexalid_web_restore
pg_restore -d vexalid_web_restore /srv/backups/vexalid_web-YYYY-MM-DD.dump
# verify row counts, then swap connection strings and restart
```

### Certificate renewal failed
Check `systemctl status certbot.timer` and `/var/log/letsencrypt/`. Usual cause: the ACME HTTP-01
challenge path is being proxied to the app. Ensure nginx serves `/.well-known/acme-challenge/` from
the webroot before any `proxy_pass`. Then `certbot renew --force-renewal && nginx -s reload`.

### Disk full
`docker image prune -af`, then check `/var/log/` and `/srv/backups/`. If it recurs, the weekly prune
timer has stopped — this is the single most common cause of a small VPS going down.

### Site down, cause unknown
1. `curl -I https://vexalid.com` from outside — DNS, TLS, or app?
2. `docker compose ps` — is the container running?
3. `docker compose logs --tail=200 web`
4. `systemctl status nginx` and `nginx -t`
5. `df -h` and `free -m`
6. `psql -U vexalid_web -c 'select 1'`
7. Still unclear → roll back to the last known-good image first, diagnose second. Restoring service
   is not the same job as finding the cause, and it comes first.
