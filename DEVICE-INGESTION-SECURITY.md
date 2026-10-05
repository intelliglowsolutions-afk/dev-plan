# Device Ingestion Security — resolving OQ-318

**Status:** written 2026-09-18. Resolves OQ-318 and replaces the withdrawn 03 D-08. **Built 2026-10-05** (Session 63): Layer 1 and Layer 3, as below; OQ-319 answered hosted.
**Spans:** features 03 (device identity) and 04 (ingestion), plus `MULTI-TENANCY.md` D-T-05.

---

## The problem, stated precisely

| What we have | What it gives us |
|---|---|
| The device sends `?SN=<serial>` | An **identifier**, not a secret. Serials are printed on the case and appear in support tickets |
| The device has no comm-key field (OQ-307, confirmed in the manual) | **No shared secret is possible at the device** |
| The device supports HTTPS, on by default | Encryption in transit, and the *device* can verify *the server*. It does not let the server verify the device |
| The serial resolves the tenant (`MULTI-TENANCY.md` D-T-05) | The identifier is also the **tenant selector** |

So: anyone who can reach the ingestion port and knows or guesses a registered serial can write
attendance into that tenant. HTTPS does not help — it authenticates the wrong direction.

**Why this is worse under multi-tenancy.** In a single-company LAN install, the blast radius of a
forged punch is one company whose network you already had to be on. In a hosted multi-tenant
install, the port is on the internet and the serial picks which customer you are writing to.

Forged attendance is not a trivial outcome. It feeds overtime, absence deductions, and ultimately
payroll — a fabricated punch is a fabricated payment.

---

## The decision that determines everything

**OQ-319 (new, and the first thing to answer): is the deployment one hosted installation serving
many tenants, or one installation per company?**

The answer changes which fix is right:

```
Is the ingestion port reachable from the internet?
│
├── NO — one install per company, devices on the same LAN
│     → The LAN is already the trust boundary.
│       Apply Layer 3 (defence in depth) only. No collector needed.
│
└── YES — hosted platform, devices push across the internet
      → Serial-only authentication is not acceptable.
        Apply Layer 1 (collector) or Layer 2 (tunnel), plus Layer 3.
```

Multi-tenancy strongly implies the hosted case, so the rest of this document assumes it — while
noting where a LAN-only deployment can skip work.

---

## Layer 1 — The recommended fix: a per-site collector

Put a small piece of software on the customer's network, between the terminals and the platform.

```
  Customer LAN                                    Internet            Platform
┌──────────────────────────────────┐                              ┌──────────────┐
│  [SenseFace]──iClock/HTTP(S)──►  │                              │              │
│  [SenseFace]──iClock/HTTP(S)──►  │ Collector ├──HTTPS + creds──►│  HRM ingest  │
│                                  │  (buffers)│                  │              │
└──────────────────────────────────┘                              └──────────────┘
      unauthenticatable hop,                    authenticated hop,
      confined to the LAN                       real rotatable secret
```

**What the collector does**

1. Speaks the iClock/ADMS protocol to the devices — it is the thing that must implement the
   protocol quirks, which also means the platform's ingestion code stops being protocol-shaped.
2. **Listens on the LAN only.** The weak hop never leaves the customer's network, where physical
   and network access is already the trust boundary.
3. **Buffers locally**, with a durable queue.
4. **Forwards to the platform over HTTPS with a per-collector credential** — an API key the platform
   issues, stores hashed, and can rotate or revoke. This is exactly what 03 D-08 wanted, moved to a
   layer that can actually hold a secret.
5. **Heartbeats**, so the platform can distinguish "collector down" from "device down".

**Why this is the right answer rather than a clever one**

- It restores real authentication without needing anything from the device. The device is not going
  to change; the architecture can.
- **The tenant stops being selected by the serial number.** The credential identifies the tenant, and
  the serial only identifies a device *within* it. That closes the multi-tenancy hole directly —
  a forged serial can now only affect the tenant whose credential you already stole.
- **It fixes a problem the plan already had.** Feature 04 classifies days as `UNKNOWN` when a device
  gap occurs (04 D-08), and a WAN outage currently produces exactly that. A buffering collector
  means an internet outage no longer costs anyone their attendance record — it drains when the link
  returns. That is worth building on its own merits.
- It is small: receive, buffer, forward, heartbeat. It ships as a container, and the customer
  already runs Docker Compose.

**Costs, honestly**

- A second deployable to install, monitor and update at each site.
- A customer who cannot run anything on-premise needs Layer 2 instead.
- The credential lives on a machine inside a customer's office, so it must be rotatable and scoped
  to one tenant and to ingestion only — never a general API key.

## Layer 2 — Equal-strength alternative: a network tunnel

A site-to-site VPN (WireGuard or IPSec) from the customer's network to the platform. The device
pushes across the tunnel; the ingestion port is never exposed to the internet at all.

**Security is equivalent to Layer 1** — arguably better, since nothing is exposed. The trade is
different: no software to write or maintain, but it needs competent network administration at each
site and a tunnel endpoint per customer. Where the customer's IT can do this, prefer it; the
collector exists for customers where they cannot.

It does **not** give the buffering benefit, so device gaps during a WAN outage remain.

## Layer 3 — Defence in depth, applied in every deployment

These are cheap, they apply whether or not a collector or tunnel is used, and several of them
belong in the plan regardless.

### 3.1 Source-IP pinning, trust on first use

On registration, the first source IP seen for a serial is **pinned** to that device. Later requests
from a different IP are not silently trusted:

- The punch is accepted into **quarantine** (below), not rejected outright — a genuine DHCP change
  must not destroy a day's attendance.
- An admin is notified and must approve the new address, which re-pins it.

This means an attacker needs the serial *and* the device's network position, not just the serial.

### 3.2 Quarantine, not accept-or-reject

A new punch state: punches from an unpinned IP, an unknown collector, or an implausible timestamp
are **stored, flagged, and not classified into attendance** until approved.

This follows the plan's existing instincts rather than inventing a new one — unmatched PINs are
queued rather than dropped (04 D-07), device gaps are `UNKNOWN` rather than `ABSENT` (04 D-08).
Evidence is kept; interpretation waits for a human.

### 3.3 Rate and plausibility limits

- Per-serial rate limit, sized well above a real terminal's polling rate.
- Reject timestamps implausibly far in the future or past (04 FR-I-11 already flags future ones).
- Cap batch size.

### 3.4 Alerting

Surface, through feature 05, to whoever holds `device.read`:

| Signal | Why it matters |
|---|---|
| Known serial from a **new source IP** | Either a network change or an impersonation attempt |
| Same serial from **two IPs concurrently** | Almost certainly an impersonation attempt |
| **Unregistered serial** attempts | Already planned (03 FR-D-04); now also a probe signal |
| Punch volume well outside the device's norm | Injection, or a device malfunction |

### 3.5 Never leak tenancy through errors

An unregistered serial, a serial belonging to another tenant, and a disabled device must be
**indistinguishable** in the response. Otherwise the endpoint becomes an oracle for enumerating
which serials are registered and to whom.

### 3.6 Bind the listener narrowly

Even in the hosted case, the ingestion path should be served on its own hostname or port, firewalled
separately from the application, and never share a listener with the user-facing app.

---

## What this changes in the plan

### Feature 03

- **D-08 is replaced**, not merely withdrawn: authentication moves from the device to the collector
  (or the tunnel). `Device.commKeyHash` stays deleted; `Collector.credentialHash` takes its place.
- New model:

```prisma
model Collector {
  id             Int       @id @default(autoincrement())
  tenantId       Int       @map("tenant_id")
  name           String                                   // "Head office", "Plant 2"
  // Argon2id hash of the issued credential. Shown once at issue, rotatable, revocable —
  // this is where 03 D-08's original intent now lives.
  credentialHash String    @map("credential_hash")
  credentialLast4 String?  @map("credential_last4")
  issuedAt       DateTime  @map("issued_at")
  rotatedAt      DateTime? @map("rotated_at")
  revokedAt      DateTime? @map("revoked_at")

  version        String?                                  // reported by heartbeat
  lastHeartbeatAt DateTime? @map("last_heartbeat_at")
  lastForwardAt  DateTime?  @map("last_forward_at")
  bufferedCount  Int       @default(0) @map("buffered_count")   // reported, for visibility

  devices        Device[]

  @@index([tenantId])
  @@map("collectors")
}
```

- `Device` gains: `collectorId` (nullable — direct-push deployments have none),
  `pinnedIpAddress`, `ipPinnedAt`, `ipPinApprovedById`.
- Device health gains a **collector health** dimension: "the collector is up, the device is silent"
  and "the collector is down" are different problems with different fixes, and today's plan cannot
  tell them apart.

### Feature 04

- A new `PunchFlag` value: `QUARANTINED`. Quarantined punches are stored, excluded from
  classification, and surfaced for approval — reusing the unmatched-PIN screen's pattern.
- Ingestion accepts either a **collector batch** (authenticated, many punches, many devices) or a
  **direct device push** (unauthenticated, LAN-only deployments). Same parser, different front door.
- The collector's buffer means `lastPushAt` can lag legitimately; the gap detector must treat a
  healthy collector with a full buffer as *delayed*, not *missing*.

### Feature 11

- No change, but worth noting: attendance-derived reports inherit whatever trust the ingestion path
  provides. A quarantined punch must never reach a report as though it were confirmed.

---

## Recommendation

1. **Answer OQ-319 first** — hosted or per-company install.
2. **If hosted: build the collector.** It is the only option that restores real authentication,
   and it pays for itself by removing WAN-outage attendance gaps.
3. **Offer the tunnel** as the alternative for customers whose IT prefers it.
4. **Implement Layer 3 in every case**, starting with IP pinning and quarantine. These are small,
   they fit the plan's existing patterns, and they are the parts that keep working when the
   assumptions above turn out to be wrong.
5. **Do not ship internet-exposed, serial-only ingestion.** If the collector is not ready, keep
   ingestion LAN-only until it is — feature 04's engine can be built and tested against synthetic
   punches regardless, so this blocks nothing else.

## Open questions

| ID | Question | Proposed |
|---|---|---|
| ~~OQ-319~~ | Hosted multi-tenant platform, or one installation per company? | **Answered 2026-10-05: hosted** |
| OQ-320 | Collector credential: API key, or mTLS client certificate? | **Built: API key** (OQ-174); mTLS if a security review asks |
| OQ-321 | Does the device's "Server Address" accept a path, not just a scheme and host? If it does, a per-tenant unguessable path becomes available as an extra layer | Assume not — the manual shows `http://www.XYZ.com` |
| OQ-322 | Who installs and updates the collector at each site — the customer's IT, or a supported appliance? | **Built for customer IT**: one image, one `docker run` (collector/README.md) |
| OQ-323 | How long should the collector buffer before alerting that it cannot reach the platform? | **Built:** buffers on disk with no limit; shown "Not reporting" and emailed once after `device.collectorSilentMinutes` (15) — Session 64 |
