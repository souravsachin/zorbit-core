---
title: zorbit-cor-license — core licensing service spec
slug: zorbit-cor-license-spec
version: 0.1-draft
status: draft-for-review
type: SPEC
audience: [platform-engineers, owner]
last-verified: 2026-04-24
---

# zorbit-cor-license — Core Licensing Service

## Why this is a `cor` (core) module

Per the `cor`/`pfs` distinction captured in `memory/GHOSTS-KILLED.md` entry on `zorbit-cor-seeder`:
- `cor` = platform is dead without it
- `pfs` = capability you invoke

Without a valid license, **every Zorbit service refuses to start**. That meets the "platform is dead without it" bar. Hence `zorbit-cor-license`, not `zorbit-pfs-license`.

## Design goals

1. **Offline-first** — deployment at a client site with no outbound internet works for at least 4 rolling days per license rotation
2. **Tamper-evident** — any modification to the license file is detected (Ed25519 signature)
3. **Bundle-gate** — shipped dist tarballs/zip on GitHub Releases are AES-256-GCM encrypted; only a valid license unlocks them
4. **Operator-friendly entry** — on-site personnel can install a refreshed license via admin UI, USB, or a typed short code
5. **Owner-signable on laptop or phone** — Ed25519 private key never leaves owner's device

## Components

```
┌──────────────────────────── OWNER DEVICE (laptop + phone) ───────────────────┐
│  zorbit-license-signer                                                       │
│  - Ed25519 signing key in OS keychain                                        │
│  - CLI + tiny local PWA (served from laptop, phone opens via LAN)            │
│  - Inputs: customer_id, expiry_hard, grace_days, modules, refresh_counter   │
│  - Outputs: signed `.lic` (JWT-style), QR PNG, short-code (CBOR+base32)     │
└──────────────────────────────────────────────────────────────────────────────┘
                                  │ secure channel (email / USB / WhatsApp / paper)
                                  ▼
┌──────────────────────────── CLIENT INSTALL ──────────────────────────────────┐
│  zorbit-cor-license (runs on every env)                                      │
│  - Validator lib (Ed25519 pubkey baked into every service image)             │
│  - Receiver endpoint: POST /api/v1/G/license                                 │
│  - State file: /var/lib/zorbit/license-state.json                            │
│  - Active file: /opt/zorbit-platform/licenses/current.lic                    │
│  - Refresh inbox: /opt/zorbit-platform/licenses/refresh.lic                  │
│  - Admin UI: License page in zorbit-unified-console                          │
└──────────────────────────────────────────────────────────────────────────────┘
```

## License payload (signed with Ed25519)

```json
{
  "iss": "zorbit-license-authority",
  "sub": "C-88A2",
  "customer_name": "Acme Corp",
  "iat": 1714000000,
  "exp_hard": 1745536000,
  "grace_days": 4,
  "refresh_counter": 42,
  "modules": ["identity","audit","pii_vault","navigation","event_bus","..."],
  "limits": { "users": 1000, "orgs": 5, "transactions_per_day": 100000 },
  "bundle_kek_wrapped": "base64:..."
}
.ed25519_signature
```

## Runtime contract (shared middleware in `zorbit-sdk-node`)

Every Zorbit service calls `validateLicense()` on boot and every hour. Logic:

```
1. Read /opt/zorbit-platform/licenses/current.lic
2. Verify Ed25519 sig against baked-in public key — fatal if invalid
3. Check now < exp_hard — fatal if expired
4. If /opt/zorbit-platform/licenses/refresh.lic present:
     verify sig; if refresh_counter > current.refresh_counter:
        atomic swap: current.lic <- refresh.lic; delete refresh
5. Check /var/lib/zorbit/license-state.json:
     if license_state.refresh_counter_seen > current.refresh_counter:
        fatal — anti-rollback
     if now - license_state.last_sighted > grace_days * 86400:
        fatal — grace window exceeded
6. Update last_sighted = now; refresh_counter_seen = current.refresh_counter
7. Check requested module is in modules array — 403 if not
8. Check limits (pre-request hook in guard)
```

**Fatal on boot** = service refuses to start, logs reason, returns 503 on health probe.
**Fatal mid-run** = service continues running current in-flight requests but rejects new ones with 503 `license_invalid`.

## Entry channels for on-site operators

| Channel | How | Time to enter |
|---|---|---|
| Admin Console → License page | Paste base64 blob OR drag-drop `.lic` file OR scan QR with browser camera | 10 sec |
| USB stick | `zorbit license install /media/usb/license.lic` (CLI) | 30 sec |
| Typed short-code | CBOR+base32 compact form, 8 groups of 20 chars `XXXX-XXXX-... × 8` | ~2 min |
| Backend receiver endpoint | `POST /api/v1/G/license` with Bearer from owner SSO | 5 sec |

## Bundle encryption

- Each module's release tarball is encrypted with AES-256-GCM before upload to GitHub Releases
- AES key is wrapped by customer's Bundle-KEK: `HKDF(root_kek, customer_id || release_version)`
- Wrapped key ships inside the `.lic` file (field `bundle_kek_wrapped`)
- Installer flow:
  1. Read `current.lic`
  2. Verify sig; unwrap Bundle-KEK using customer_id + release
  3. Decrypt ciphertext tarball → plaintext archive
  4. Verify GCM tag (tamper detection)
  5. Extract + install

Without a license for THIS customer, the public tarball on GitHub Releases is ciphertext. Useless.

## Rolling 4-day cadence (offline clients)

| Day | Owner action | Client action |
|---|---|---|
| Day 0 | Issue `license_v1` (counter=1), delivered at install | Drop in place; services boot |
| Day 3 EOD | Issue `license_v2` (counter=2), deliver via email / WhatsApp / courier | — |
| Day 4 | — | Drop v2 in admin UI or /opt/.../refresh.lic; counter advances; timer resets 4 days |
| Miss day 4 | — | Already-running services stay up but log warnings; next service restart is blocked |

## Service structure (when built)

```
zorbit-cor-license/
  src/
    controllers/license.controller.ts          # POST /api/v1/G/license, GET /api/v1/G/license/status
    services/license.service.ts                # verify, adopt refresh, rotation
    services/state.service.ts                  # reads/writes license-state.json
    guards/license.guard.ts                    # @UseGuards() for all other services
    keys/public.pem                            # Ed25519 pubkey (built-in)
  dist/
  package.json                                 # exposes as @zorbit-platform/cor-license
  zorbit-module-manifest.json
  README.md
  CLAUDE.md
```

Mirror of runtime lives in `zorbit-sdk-node/src/license/` so every service can `@UseGuards(LicenseGuard)`.

## Defensive details

- Clock-skew doesn't bypass: `refresh_counter` is monotonic + server-side state
- Service already up stays up even if grace expires (prevents mid-demo outage); blocks only at next restart
- Panic-bypass: owner can issue emergency `unlock_license` with `grace_days=30` when a client is truly locked out
- Telemetry: each license validation emits an audit event to `zorbit-audit` (trail: who, when, license_counter, result)

## Endpoints

```
POST /api/v1/G/license            body: { license: "base64..." }        roles: admin
GET  /api/v1/G/license/status                                          roles: admin, system
GET  /api/v1/G/license/public-key                                      public (for operator verification)
```

## Admin Console — License page

- Current license summary: customer, expiry_hard, grace window remaining, modules, limits, refresh_counter, last_sighted
- Entry tools: paste-box, file-drop, QR camera, short-code typer
- Validation feedback: sig valid/invalid, expiry countdown, rollback warning
- "How to refresh" inline help

## Build order (when greenlit)

1. Finalise this spec (you review + amend)
2. `zorbit-license-signer` CLI (owner-side)
3. `zorbit-sdk-node/src/license/` middleware (client-side validator + guard)
4. `zorbit-cor-license` service (controller + state management)
5. Admin Console License page
6. Bundle encryption pipeline (integrated with `package-bundle.sh`)
7. End-to-end test: sign → install → boot → rotate

## Open questions for owner

1. **Module-granular licensing** — is it OK for the license to enumerate modules, or should it be feature-flag granular?
2. **Limits** — which limits matter day-one? (users, orgs, transactions_per_day — others?)
3. **Emergency bypass policy** — who's authorised to issue the `grace_days=30` override?
4. **Key rotation strategy** — if the Ed25519 signing key is ever compromised, how do we re-sign outstanding licenses?
5. **Telemetry opt-in** — do we phone home on every validate, once a day, or never (pure offline)?
