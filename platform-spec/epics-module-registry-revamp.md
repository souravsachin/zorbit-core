# Platform Revamp — Epics, User Stories, Test Plans

**Initiative:** Module Registry, Manifest v2, Navigation Decoupling, FE Route Migration, Micro-Frontend  
**Conventions reference:** `platform-spec/uri-conventions.md` (read this before touching any endpoint)  
**Version:** 1.0  
**Date:** 2026-04-19  
**Status:** Approved for execution

---

## Execution Sequence

```
Phase 1 (parallel):
  EPIC 1 — zorbit-cor-module_registry (new repo)
  EPIC 2 — Manifest v2 schema + PCG4 migration

Phase 2 (after Epic 1 is running):
  EPIC 3 — Navigation service decoupling

Phase 3 (after Epic 2 manifests are done):
  EPIC 4 — FE route pattern migration

Phase 4 (after Epic 2 + 4):
  EPIC 5 — Micro-frontend lazy loading
```

---

## EPIC 1: `zorbit-cor-module_registry`

**Goal:** Ultra-core service. Every module announces its presence via Kafka. Registry validates, stores, resolves dependency order, and pushes WebSocket refresh to eligible users.

**Repo name:** `zorbit-cor-module_registry`  
**Port:** 3015 (dev), 3115 (server)  
**Namespace:** G (global — no org scope)

### Backend Endpoints (all nouns, per uri-conventions.md)

```
GET    /api/v1/G/health                                   → service health (public)
GET    /api/v1/G/modules                                  → list registered modules
GET    /api/v1/G/modules/:moduleId                        → one module record
GET    /api/v1/G/modules/:moduleId/manifest               → module manifest document
GET    /api/v1/G/modules/:moduleId/status                 → boot state + dependency graph
GET    /api/v1/G/modules/:moduleId/dependencies           → dependency list
GET    /api/v1/G/modules/:moduleId/users                  → users with access to this module
POST   /api/v1/G/modules                                  → register a module (manual, for admin)
DELETE /api/v1/G/modules/:moduleId                        → deregister
POST   /api/v1/G/modules/:moduleId/notifications          → trigger menu-refresh push to eligible users
```

### User Stories

**US-1.1 — Kafka Announcement Subscriber**  
When a module starts up, it publishes to Kafka topic `platform.module.announcements`. The registry validates the HMAC-SHA256 signature, fetches the manifest from `manifest_url`, stores it, and resolves dependency state.

Kafka message schema:
```json
{
  "moduleId": "app_pcg4",
  "manifestUrl": "https://cdn.onezippy.ai/modules/pcg4/zorbit-module-manifest.json",
  "version": "1.0.0",
  "signedToken": "<HMAC-SHA256(manifest_json, PLATFORM_MODULE_SECRET)>",
  "dependencies": ["zorbit-identity", "zorbit-authorization", "zorbit-event_bus"]
}
```

Acceptance:
- Valid signature → manifest fetched, module stored as `PENDING`
- Invalid signature → rejected, audit event `registry.module.rejected` written, not stored
- Duplicate announcement (same version) → idempotent, no duplicate record created

**US-1.2 — Dependency Graph Resolution**  
Registry maintains module state: `PENDING | READY | FAILED`. A module transitions to `READY` only when all declared dependencies are `READY`. Core modules (`zorbit-identity`, `zorbit-authorization`, `zorbit-event_bus`) declare no dependencies and go `READY` immediately on announcement.

When a module transitions to `READY`, registry publishes `platform.module.ready` to Kafka.

**US-1.3 — Registry REST API**  
All endpoints require JWT, privilege `registry.module.read`, except `GET /health` (public).  
`DELETE` and `POST /notifications` require `registry.module.admin`.

**US-1.4 — Module Trust Validation**  
`PLATFORM_MODULE_SECRET` is a shared env var across all modules and the registry.  
`signedToken = HMAC-SHA256(canonical_json(manifest), secret)`  
Canonical JSON = keys sorted alphabetically, no whitespace.

**US-1.5 — WebSocket Push to Eligible Users**  
When a module transitions to `READY`:
1. Registry calls `POST /api/v1/G/modules/:moduleId/notifications` (self-trigger, internal)
2. Registry calls Authorization service: `GET /api/v1/G/modules/:moduleId/users` — gets list of user IDs who hold any privilege from this module's `privileges[]`
3. Registry pushes WebSocket message to only those user sessions: `{ type: "MODULE_REGISTERED", moduleId, label, version }`
4. Unified Console receives → re-fetches `GET /api/v1/U/:userId/navigation/menu` → updates sidebar. No page reload.

Note: The module manifest contains no WebSocket declaration. This is orchestrated entirely by the registry + authorization service after registration.

### Test Plan

| ID | Scenario | Expected |
|----|----------|----------|
| TC-1.1 | Valid HMAC announcement published to Kafka | Module stored, status=PENDING or READY if deps met |
| TC-1.2 | Invalid HMAC in announcement | Module rejected, `registry.module.rejected` audit event |
| TC-1.3 | Module with unmet dependencies | Stays PENDING; transitions to READY when dep becomes READY |
| TC-1.4 | Duplicate announcement (same version) | Idempotent — no second record |
| TC-1.5 | `GET /api/v1/G/modules` without JWT | 401 |
| TC-1.6 | `GET /api/v1/G/modules` with valid JWT + privilege | Module list returned |
| TC-1.7 | Module becomes READY; user holds one of its privileges | WebSocket message received in browser |
| TC-1.8 | Module becomes READY; user holds none of its privileges | No WebSocket sent to that user |
| TC-1.9 | `GET /api/v1/G/health` | 200, no auth required |
| TC-1.10 | Core modules announce with no dependencies | Go READY immediately |

---

## EPIC 2: Module Manifest Standard v2

**Goal:** One canonical manifest file per module. Correct FE routes (no org scope). Full hierarchy placement. HMAC-ready. PCG4 is the reference implementation.

**Files:**
- `zorbit-core/schemas/module-manifest/manifest.schema.json` — already exists, extend it
- `zorbit-core/platform-spec/uri-conventions.md` — this document governs all route validation
- `zorbit-app-pcg4/zorbit-module-manifest.json` — update to v2
- `zorbit-app-pcg4/zorbit-manifest.json` — DELETE (duplicate)

### v2 Schema Changes from Current PCG4 Manifest

**Add `placement` block:**
```json
"placement": {
  "edition": "insurer",
  "businessLine": "distribution",
  "capabilityArea": "product-management",
  "sortOrder": 10
}
```

**Add `registration` block:**
```json
"registration": {
  "kafkaTopic": "platform.module.announcements",
  "manifestUrl": "https://cdn.onezippy.ai/modules/pcg4/zorbit-module-manifest.json"
}
```

**Fix `feRoute` — remove org scope:**
```
Before: "/org/{{org_id}}/app/pcg4/configs"
After:  "/m/pcg4/configs"
```

**Fix `frontend.bundleUrl` — versioned CDN URL:**
```
Before: "/modules/pcg4/bundle.js"
After:  "https://cdn.onezippy.ai/modules/pcg4/1.0.0/bundle.js"
```

**Fix `frontend.routes[]`:**
```
Before: ["/org/*/app/pcg4/*"]
After:  ["/m/pcg4/*"]
```

**Apply approved shortcuts in endpoint paths:**
```
Before: "/configurations"
After:  "/configs"
```

### User Stories

**US-2.1 — Canonical manifest schema v2 in zorbit-core**  
Extend `manifest.schema.json` with `placement`, `registration` blocks. Add validation rule: `feRoute` must match `/m/{slug}/...`.

**US-2.2 — PCG4 manifest migrated to v2**  
`zorbit-module-manifest.json` updated. `zorbit-manifest.json` deleted. PCG4 can announce itself to the registry.

**US-2.3 — zorbit-cli manifest validator**  
`zorbit validate-manifest ./zorbit-module-manifest.json` checks:
- Required fields present
- `feRoute` matches `/m/...` (no org scope)
- All privileges in nav items exist in `privileges[]`
- `bundleUrl` is a full URL (not relative path)
- No verb/adjective in any URI segment (blocklist from uri-conventions.md)
- Shortcuts: only approved ones used

**US-2.4 — All modules get manifest v2 (follow-up)**  
After PCG4 reference is complete, each other module gets its manifest updated. One soldier per module repo.

### Test Plan

| ID | Scenario | Expected |
|----|----------|----------|
| TC-2.1 | PCG4 v2 manifest against schema validator | 0 errors |
| TC-2.2 | Manifest with `feRoute: "/org/O-92AF/..."` | Validator error: org scope in feRoute |
| TC-2.3 | Missing `placement.edition` | Validator error: required field |
| TC-2.4 | Nav item privilege not in `privileges[]` | Validator error: undeclared privilege |
| TC-2.5 | `zorbit-manifest.json` still present | Validator warning: duplicate manifest detected |
| TC-2.6 | `bundleUrl` is `/modules/pcg4/bundle.js` | Validator error: bundleUrl must be a full URL |
| TC-2.7 | Non-approved shortcut used in beRoute | Validator error: unapproved shortcut |

---

## EPIC 3: Navigation Service Decoupling

**Goal:** `zorbit-navigation` stops owning module registrations. It becomes a read-only consumer of the module registry. Its own seeded menu DB is replaced by registry data. `/resolved` endpoint renamed and corrected.

### Corrected Endpoints

```
GET /api/v1/U/:userId/navigation/menu     → privilege-filtered menu tree (replaces /resolved)
GET /api/v1/O/:orgId/navigation/menus     → admin: all menus for org
GET /api/v1/O/:orgId/navigation/menus/:menuId              → one menu
GET /api/v1/O/:orgId/navigation/menus/:menuId/sections     → sections
GET /api/v1/O/:orgId/navigation/menus/:menuId/sections/:sectionId/items  → items
```

Removed endpoints (return `410 Gone`):
```
POST /api/v1/O/:orgId/navigation/menus    (module registration — now via Kafka to module registry)
POST /api/v1/O/:orgId/navigation/routes   (route registration — declared in manifest)
```

### User Stories

**US-3.1 — Navigation subscribes to `platform.module.ready`**  
On receipt: fetch manifest from registry `GET /api/v1/G/modules/:moduleId/manifest`. Cache locally with 5-minute TTL. Rebuild menu structure from registry data.

**US-3.2 — `/menu` builds tree from registry data, filtered by JWT privileges**  
Input: JWT (carries `privileges[]`). For each READY module: for each nav item: include only if user holds the required privilege. Apply `placement` hierarchy (edition → businessLine → capabilityArea → module → feature).

**US-3.3 — Seed files deleted**  
`src/config/menu-6level-seed.json`, `src/config/menuConfig.json` — deleted. Navigation no longer owns menu content. It only shapes and filters it.

**US-3.4 — Stale cache fallback**  
If registry is unreachable, return last cached menu with response header `X-Zorbit-Menu-Stale: true`. Log warning. Do not fail the request.

### Test Plan

| ID | Scenario | Expected |
|----|----------|----------|
| TC-3.1 | `/menu` with valid JWT; 3 READY modules in registry | Menu has items from all 3 modules |
| TC-3.2 | User lacks privilege for module X | Module X absent from menu response |
| TC-3.3 | New module becomes READY mid-session | After WebSocket refresh, user's menu includes it |
| TC-3.4 | `POST /navigation/menus` called | 410 Gone |
| TC-3.5 | Navigation seed files removed; service starts fresh | Menu builds from registry correctly |
| TC-3.6 | Registry unreachable | Last cached menu returned, `X-Zorbit-Menu-Stale: true` header set |
| TC-3.7 | `GET /api/v1/U/U-81F3/navigation/menu` without JWT | 401 |

---

## EPIC 4: FE Route Pattern Migration

**Goal:** All FE routes follow `/m/{module-slug}/{feature}/...`. No org scope in any FE URL. BE route resolved from manifest at call time using JWT org.

### Route Pattern

```
/m/{module-slug}/{feature-slug}[/{resource}/{resource-id}[/{sub-resource}/{sub-resource-id}]]
```

Full-word rules apply. Approved shortcuts from uri-conventions.md §5 apply.

### Correct examples

```
/m/pcg4/configs
/m/pcg4/configs/CFG-29F1
/m/pcg4/configs/CFG-29F1/plans
/m/pcg4/configs/CFG-29F1/plans/PLN-81F3/benefits
/m/pcg4/refs
/m/hi-quotation/quotes
/m/hi-quotation/quotes/QUO-92AF
/m/form-builder/forms
/m/form-builder/forms/FRM-44A1
/m/form-builder/forms/FRM-44A1/submissions
/m/identity/users
/m/identity/orgs
/m/authorization/roles
```

### User Stories

**US-4.1 — Unified Console router updated**  
All `<Route>` definitions match `/m/:moduleSlug/*`. Old `/org/:orgId/app/*` routes redirect to new pattern.

**US-4.2 — Org from JWT, not URL**  
Every module page that needs org context reads it from `useAuth().user.org`, not from route params.

**US-4.3 — Sidebar links use `feRoute` from manifest**  
Since manifest v2 carries correct `/m/...` routes, sidebar renders them directly. No transformation needed.

**US-4.4 — Old routes redirect**  
`/org/:orgId/app/:module/*` → `302` → `/m/:module/*` for backward compatibility with bookmarks.

### Test Plan

| ID | Scenario | Expected |
|----|----------|----------|
| TC-4.1 | Navigate to `/m/pcg4/configs` | PCG4 configs page loads |
| TC-4.2 | Navigate to old `/org/O-92AF/app/pcg4/configs` | Redirects to `/m/pcg4/configs` |
| TC-4.3 | PCG4 API call | URL contains org from JWT, not from URL |
| TC-4.4 | User with org O-A logged in; API called | BE URL contains `O-A` |
| TC-4.5 | Bookmark `/m/pcg4/configs`; open new tab | Loads correctly |

---

## EPIC 5: Micro-Frontend Lazy Loading

**Goal:** Unified Console is a thin shell. Module JS bundles load on first click from CDN. Shell always available. Module failures isolated.

### User Stories

**US-5.1 — Module bundle loader**  
On first navigation to `/m/pcg4/*`: dynamically import bundle from `manifest.frontend.bundleUrl`. Mount `entryComponent` inside shell `<main>`. Second visit uses cached bundle.

**US-5.2 — Loading state**  
Skeleton loader shown while bundle downloads. Progress indicator in sidebar item.

**US-5.3 — Bundle error isolation**  
If bundle fails (404, network error): show error card scoped to that module's area. Other modules unaffected. Error logged to audit.

**US-5.4 — Core services bundled in shell**  
Identity, Authorization, Navigation admin, Audit, PII Vault pages are in the main shell bundle. Always available, zero latency.

**US-5.5 — Bundle cache strategy**  
Bundles are versioned by URL (`/modules/pcg4/1.0.0/bundle.js`). Browser caches indefinitely. New version = new URL. Registry pushes `MODULE_UPDATED` WebSocket to prompt refresh.

### Test Plan

| ID | Scenario | Expected |
|----|----------|----------|
| TC-5.1 | First click on PCG4 in sidebar | Network request to CDN; module renders |
| TC-5.2 | Second click on PCG4 | No new CDN request (cached) |
| TC-5.3 | PCG4 bundle URL returns 404 | Error card shown; rest of app works |
| TC-5.4 | Direct navigation to `/m/pcg4/configs` | Bundle loads, correct page renders |
| TC-5.5 | Core page (Users) opened | No CDN request — from shell bundle |
| TC-5.6 | Module updated to v1.0.1; registry pushes WebSocket | Console prompts user to refresh |

---

## Cross-Cutting Rules (apply to all epics)

1. **Every endpoint in every PR is checked against `platform-spec/uri-conventions.md` before merge.**
2. **`zorbit validate-manifest` runs in CI** for every module repo. Fails build on violation.
3. **One soldier per module repo** when applying manifest v2 migrations (Epic 2). Each soldier reads uri-conventions.md first.
4. **No shortcuts outside the approved list.** A new shortcut requires a PR to uri-conventions.md first.
5. **State transitions use PATCH on noun sub-resources** — never a verb path segment.
