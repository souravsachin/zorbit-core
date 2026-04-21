# Zorbit Platform — URI Conventions

**Version:** 1.0  
**Status:** Canonical — all modules must comply  
**Owner:** Platform Architecture  
**Last updated:** 2026-04-19

---

## 0. Guiding Principle

> **Every path segment is a noun. No exceptions.**

A URI identifies a *resource*. It does not describe an *action*. Actions are expressed by HTTP methods (GET, POST, PUT, PATCH, DELETE). If a path segment is a verb, a past participle, or an adjective, it is wrong.

This document is the single source of truth for all URI decisions on the Zorbit platform. When in doubt, consult this document. If this document does not answer the question, update it before writing code.

---

## 1. Module Naming Convention

### Pattern

```
zorbit-{type}-{module_name}
```

### Rules

- `{type}` — one of the registered type prefixes below (see table)
- `{module_name}` — lowercase, words separated by underscores (`_`), no hyphens within the name
- The hyphen between type and module_name is the only hyphen in the name

### Type Prefix Registry

| Prefix | Meaning | Examples |
|--------|---------|---------|
| `cor` | Core platform services | `zorbit-cor-identity`, `zorbit-cor-authorization`, `zorbit-cor-module_registry` |
| `pfs` | Platform Feature Services | `zorbit-pfs-datatable`, `zorbit-pfs-form_builder`, `zorbit-pfs-ai_gateway` |
| `app` | Business Applications | `zorbit-app-pcg4`, `zorbit-app-hi_quotation` |
| `tpm` | Third Party Module integrations | `zorbit-tpm-jitsi`, `zorbit-tpm-odoo` |
| `sdk` | SDK libraries | `zorbit-sdk-node`, `zorbit-sdk-react`, `zorbit-sdk-python` |

### Note on legacy names

`zorbit-identity`, `zorbit-authorization`, `zorbit-navigation`, etc. predate the `cor` prefix decision. These names are frozen — do not rename live repos. All new core services use `zorbit-cor-*`. This discrepancy must be documented in each legacy repo's CLAUDE.md.

### Correct examples

```
zorbit-cor-module_registry      ✓
zorbit-pfs-form_builder         ✓
zorbit-app-hi_quotation         ✓
zorbit-tpm-jitsi                ✓
```

### Incorrect examples

```
zorbit-module-registry          ✗  (missing type, hyphen in name)
zorbit-cor-moduleRegistry       ✗  (camelCase in name)
zorbit-cor-module-registry      ✗  (hyphen within name)
zorbit-form-builder             ✗  (missing type)
```

---

## 2. Backend URI Structure

### 2.1 Grammar

There are two forms of every backend URL: **external** (client-facing, routed through nginx) and **internal** (what the NestJS service actually registers).

#### External URL (client calls this)

```
/api/{module-slug}/api/v{N}/{scope}/{scope-id}/{resource-path}
```

#### Internal URL (NestJS controller prefix, inside the container)

```
/api/v{N}/{scope}/{scope-id}/{resource-path}
```

nginx strips `/api/{module-slug}/` before forwarding the request to the service container. The `{resource-path}` is **identical** in both external and internal forms.

| Segment | Rules |
|---------|-------|
| `/api` | Fixed outer prefix — nginx routing entry point |
| `/{module-slug}` | The module's short slug, **always present** — used by nginx to route to the correct service. Lowercase, hyphens between words. Same as repo name minus type prefix (e.g. `identity`, `pcg4`, `form-builder`, `module-registry`) |
| `/api` | Fixed inner prefix — NestJS global prefix |
| `/v{N}` | API version. Always immediately after the inner `/api`. Current: `v1` |
| `/{scope}` | One of: `G`, `O`, `D`, `U` (see namespace model). **Always present.** |
| `/{scope-id}` | The scoped entity identifier, e.g. `O-92AF`, `U-81F3`. For `G` (global) use literal `G` |
| `/{resource-path}` | One or more noun segments. See nesting rules (§7). |

#### FE ↔ BE alignment rule

The `{resource-path}` segment is **identical** in the FE URL and the BE external URL:

```
FE:  /m/{module-slug}/{resource-path}
BE:  /api/{module-slug}/api/v1/{scope}/{scope-id}/{resource-path}
```

Example:
```
FE:  /m/pcg4/configs/CFG-29F1/plans
BE:  /api/pcg4/api/v1/O/O-92AF/configs/CFG-29F1/plans
```

#### nginx routing rule

```nginx
location /api/identity/ {
    proxy_pass http://zu-core:3001/api/;
}
location /api/pcg4/ {
    proxy_pass http://zu-apps:3011/api/;
}
```

The trailing slash on `proxy_pass` causes nginx to strip the matched prefix before forwarding.

#### What was wrong before (do not repeat)

```
/api/app/pcg4/v1/O/{orgId}/configs     ✗  type prefix in path, version in wrong position
/api/v1/O/{orgId}/configurations       ✗  no module slug in external URL, unapproved full word
/api/v1/O/{orgId}/pcg4/configs         ✗  module slug inside service path (old hybrid — abolished)
```

### 2.2 Namespace selection per resource

| Namespace | When to use |
|-----------|-------------|
| `G` | Platform-wide resources. No org context needed. E.g. `modules`, `health` |
| `O` | Organization-scoped resources. Most business data lives here |
| `D` | Department-scoped resources. Use only when department boundary is meaningful |
| `U` | User-scoped resources. The user's personal context: their menu, their profile, their preferences |

Scope is **always present**. Never omit the scope segment even for simple resources.

### 2.3 Approved route list (all modules)

Format: `EXTERNAL URL` → internal (nginx strips `/api/{slug}/`)

#### zorbit-identity (slug: `identity`, port 3001)

```
GET    /api/identity/api/v1/G/health
GET    /api/identity/api/v1/O/{orgId}/users
POST   /api/identity/api/v1/O/{orgId}/users
GET    /api/identity/api/v1/O/{orgId}/users/{userId}
PATCH  /api/identity/api/v1/O/{orgId}/users/{userId}
DELETE /api/identity/api/v1/O/{orgId}/users/{userId}
GET    /api/identity/api/v1/U/{userId}/profile
PATCH  /api/identity/api/v1/U/{userId}/profile
GET    /api/identity/api/v1/O/{orgId}/orgs
POST   /api/identity/api/v1/O/{orgId}/orgs
GET    /api/identity/api/v1/O/{orgId}/orgs/{subOrgId}
POST   /api/identity/api/v1/G/auth/tokens               ← login (issues JWT)
DELETE /api/identity/api/v1/G/auth/tokens               ← logout (revoke)
POST   /api/identity/api/v1/G/auth/tokens/refresh       ← refresh
```

#### zorbit-authorization (slug: `authorization`, port 3002)

```
GET    /api/authorization/api/v1/G/health
GET    /api/authorization/api/v1/O/{orgId}/roles
POST   /api/authorization/api/v1/O/{orgId}/roles
GET    /api/authorization/api/v1/O/{orgId}/roles/{roleId}
PATCH  /api/authorization/api/v1/O/{orgId}/roles/{roleId}
DELETE /api/authorization/api/v1/O/{orgId}/roles/{roleId}
GET    /api/authorization/api/v1/O/{orgId}/roles/{roleId}/privileges
POST   /api/authorization/api/v1/O/{orgId}/roles/{roleId}/privileges
DELETE /api/authorization/api/v1/O/{orgId}/roles/{roleId}/privileges/{privilegeId}
GET    /api/authorization/api/v1/G/privileges
POST   /api/authorization/api/v1/G/privileges
GET    /api/authorization/api/v1/O/{orgId}/users/{userId}/privileges
GET    /api/authorization/api/v1/G/modules/{moduleId}/users  ← users with any privilege in module
```

#### zorbit-navigation (slug: `navigation`, port 3003)

```
GET    /api/navigation/api/v1/G/health
GET    /api/navigation/api/v1/U/{userId}/navigation/menu      ← privilege-filtered menu tree
GET    /api/navigation/api/v1/O/{orgId}/navigation/menus
GET    /api/navigation/api/v1/O/{orgId}/navigation/menus/{menuId}
GET    /api/navigation/api/v1/O/{orgId}/navigation/menus/{menuId}/sections
GET    /api/navigation/api/v1/O/{orgId}/navigation/menus/{menuId}/sections/{sectionId}/items
```

Deprecated (returns 410): `GET .../navigation/resolved`

#### zorbit-cor-module_registry (slug: `module-registry`, port 3036)

```
GET    /api/module-registry/api/v1/G/health
GET    /api/module-registry/api/v1/G/modules
POST   /api/module-registry/api/v1/G/modules
GET    /api/module-registry/api/v1/G/modules/{moduleId}
DELETE /api/module-registry/api/v1/G/modules/{moduleId}
GET    /api/module-registry/api/v1/G/modules/{moduleId}/manifest
GET    /api/module-registry/api/v1/G/modules/{moduleId}/status
GET    /api/module-registry/api/v1/G/modules/{moduleId}/dependencies
GET    /api/module-registry/api/v1/G/modules/{moduleId}/users
POST   /api/module-registry/api/v1/G/modules/{moduleId}/notifications
```

#### zorbit-audit (slug: `audit`, port 3006)

```
GET    /api/audit/api/v1/G/health
GET    /api/audit/api/v1/O/{orgId}/events
GET    /api/audit/api/v1/O/{orgId}/events/{eventId}
GET    /api/audit/api/v1/G/events                    ← platform-level audit (superadmin only)
```

#### zorbit-pii-vault (slug: `pii-vault`, port 3005)

```
GET    /api/pii-vault/api/v1/G/health
POST   /api/pii-vault/api/v1/O/{orgId}/tokens        ← tokenize PII value → returns token
GET    /api/pii-vault/api/v1/O/{orgId}/tokens/{tokenId}  ← resolve token → PII value
DELETE /api/pii-vault/api/v1/O/{orgId}/tokens/{tokenId}
```

#### zorbit-event_bus (slug: `event-bus`, port 3004)

```
GET    /api/event-bus/api/v1/G/health
GET    /api/event-bus/api/v1/G/topics
POST   /api/event-bus/api/v1/G/topics
GET    /api/event-bus/api/v1/G/topics/{topicId}
GET    /api/event-bus/api/v1/G/topics/{topicId}/messages
```

#### zorbit-pfs-form_builder (slug: `form-builder`, port 3014)

```
GET    /api/form-builder/api/v1/G/health
GET    /api/form-builder/api/v1/O/{orgId}/forms
POST   /api/form-builder/api/v1/O/{orgId}/forms
GET    /api/form-builder/api/v1/O/{orgId}/forms/{formId}
PATCH  /api/form-builder/api/v1/O/{orgId}/forms/{formId}
DELETE /api/form-builder/api/v1/O/{orgId}/forms/{formId}
GET    /api/form-builder/api/v1/O/{orgId}/forms/{formId}/submissions
POST   /api/form-builder/api/v1/O/{orgId}/forms/{formId}/submissions
GET    /api/form-builder/api/v1/O/{orgId}/forms/{formId}/submissions/{submissionId}
```

#### zorbit-pfs-datatable (slug: `datatable`, port 3013)

```
GET    /api/datatable/api/v1/G/health
GET    /api/datatable/api/v1/O/{orgId}/tables
POST   /api/datatable/api/v1/O/{orgId}/tables
GET    /api/datatable/api/v1/O/{orgId}/tables/{tableId}
PATCH  /api/datatable/api/v1/O/{orgId}/tables/{tableId}
DELETE /api/datatable/api/v1/O/{orgId}/tables/{tableId}
GET    /api/datatable/api/v1/O/{orgId}/tables/{tableId}/rows
POST   /api/datatable/api/v1/O/{orgId}/tables/{tableId}/rows
```

#### zorbit-app-pcg4 (slug: `pcg4`, port 3011)

```
GET    /api/pcg4/api/v1/G/health
GET    /api/pcg4/api/v1/O/{orgId}/configs
POST   /api/pcg4/api/v1/O/{orgId}/configs
GET    /api/pcg4/api/v1/O/{orgId}/configs/{configId}
PATCH  /api/pcg4/api/v1/O/{orgId}/configs/{configId}
DELETE /api/pcg4/api/v1/O/{orgId}/configs/{configId}
GET    /api/pcg4/api/v1/O/{orgId}/configs/{configId}/plans
POST   /api/pcg4/api/v1/O/{orgId}/configs/{configId}/plans
GET    /api/pcg4/api/v1/O/{orgId}/configs/{configId}/plans/{planId}
GET    /api/pcg4/api/v1/O/{orgId}/configs/{configId}/plans/{planId}/benefits
POST   /api/pcg4/api/v1/O/{orgId}/configs/{configId}/plans/{planId}/benefits
GET    /api/pcg4/api/v1/O/{orgId}/refs
```

#### zorbit-app-hi_quotation (slug: `hi-quotation`, port varies)

```
GET    /api/hi-quotation/api/v1/G/health
GET    /api/hi-quotation/api/v1/O/{orgId}/quotes
POST   /api/hi-quotation/api/v1/O/{orgId}/quotes
GET    /api/hi-quotation/api/v1/O/{orgId}/quotes/{quoteId}
PATCH  /api/hi-quotation/api/v1/O/{orgId}/quotes/{quoteId}
PATCH  /api/hi-quotation/api/v1/O/{orgId}/quotes/{quoteId}/status
```

#### zorbit-app-uw_workflow (slug: `uw-workflow`, port varies)

```
GET    /api/uw-workflow/api/v1/G/health
GET    /api/uw-workflow/api/v1/O/{orgId}/queues
POST   /api/uw-workflow/api/v1/O/{orgId}/queues
GET    /api/uw-workflow/api/v1/O/{orgId}/queues/{queueId}
GET    /api/uw-workflow/api/v1/O/{orgId}/queues/{queueId}/items
POST   /api/uw-workflow/api/v1/O/{orgId}/queues/{queueId}/items
PATCH  /api/uw-workflow/api/v1/O/{orgId}/queues/{queueId}/items/{itemId}/status
```

#### zorbit-app-hi_decisioning (slug: `hi-decisioning`, port varies)

```
GET    /api/hi-decisioning/api/v1/G/health
GET    /api/hi-decisioning/api/v1/O/{orgId}/decisions
POST   /api/hi-decisioning/api/v1/O/{orgId}/decisions
GET    /api/hi-decisioning/api/v1/O/{orgId}/decisions/{decisionId}
PATCH  /api/hi-decisioning/api/v1/O/{orgId}/decisions/{decisionId}/status
GET    /api/hi-decisioning/api/v1/O/{orgId}/decisions/{decisionId}/rules
```

### 2.4 Incorrect backend URI examples

```
GET /api/navigation/api/v1/U/U-81F3/navigation/resolved  ✗  ("resolved" is a past participle)
GET /api/identity/api/v1/O/O-92AF/getConfigurations      ✗  ("get" is a verb)
POST /api/module-registry/api/v1/G/modules/notify-users  ✗  ("notify" is a verb)
GET /api/pcg4/api/v1/O/O-92AF/active-configs             ✗  ("active" is adjective — use ?status=active)
POST /api/pcg4/api/v1/O/O-92AF/configs/publish           ✗  ("publish" is a verb — use PATCH /status)
GET /api/v1/O/O-92AF/pcg4/configs                        ✗  (no module-slug external prefix — old hybrid, abolished)
```

### 2.5 State transitions (the verb problem solved)

When you need to change a resource's state (approve, publish, archive, reject), use PATCH on a noun sub-resource:

```
PATCH /api/v1/O/O-92AF/configs/CFG-29F1/status     body: { "status": "published" }
PATCH /api/v1/O/O-92AF/configs/CFG-29F1/stage      body: { "stage": 5 }
PATCH /api/v1/O/O-92AF/claims/CLM-33F2/decision    body: { "decision": "approved" }
```

The sub-resource is the noun (`status`, `stage`, `decision`). The new value goes in the body. Never `POST /publish`, `POST /approve`, `POST /reject`.

---

## 3. Frontend URI Structure

### 3.1 Grammar

```
/m/{module-slug}/{feature-slug}[/{resource}/{resource-id}[/{sub-resource}/{sub-resource-id}]]
```

| Segment | Rules |
|---------|-------|
| `/m` | Fixed prefix — all module pages live under `/m` |
| `/{module-slug}` | The module's short slug, lowercase hyphens (same as module code, no underscores) |
| `/{feature-slug}` | The feature or page group within the module |
| `/{resource}` | Plural noun (singular form acceptable for detail pages) |
| `/{resource-id}` | Short hash identifier |

### 3.2 Key rules

- **No org scope in FE URIs.** Org ID comes from the JWT (`token.org`). It must never appear in the URL.
- **No user scope in FE URIs.** Same reason.
- **Full words.** No single-character abbreviations. Approved short forms (Section 5) are permitted.
- **Hyphens between words** in slugs (not underscores) — this is URL convention, not code convention.
- The BE route for each page is taken from `manifest.navigation.items[].beRoute` with `{{org_id}}` substituted from the JWT at call time.

### 3.3 Correct frontend URI examples

```
/m/pcg4/configs                              → PCG4 configs list
/m/pcg4/configs/CFG-29F1                     → one config detail
/m/pcg4/configs/CFG-29F1/plans              → plans within a config
/m/pcg4/configs/CFG-29F1/plans/PLN-81F3    → one plan detail
/m/pcg4/refs                                 → reference library
/m/pcg4/deploys                              → wait — "deploys" reads as verb; use "deployments"

/m/hi-quotation/quotes                       → quotes list
/m/hi-quotation/quotes/QUO-92AF             → one quote
/m/form-builder/forms                        → forms list
/m/form-builder/forms/FRM-44A1              → form detail
/m/form-builder/forms/FRM-44A1/submissions  → submissions for that form

/m/identity/users                            → user management
/m/identity/users/U-81F3                     → one user
/m/identity/orgs                             → organisation management
/m/identity/orgs/O-92AF                      → one org
```

### 3.4 Incorrect frontend URI examples

```
/org/O-92AF/app/pcg4/configurations     ✗  (org scope in URL)
/app/pcg4/overview                      ✗  (missing /m prefix)
/m/pcg4/c/CFG-29F1                      ✗  (single-character abbreviation)
/m/pcg4/configs/CFG-29F1/edit          ✗  ("edit" is a verb — just use the detail route; edit is a UI state)
```

---

## 4. HTTP Method Semantics

| Method | Meaning | Safe | Idempotent |
|--------|---------|------|------------|
| `GET` | Read a resource or collection | Yes | Yes |
| `POST` | Create a new resource | No | No |
| `PUT` | Replace a resource entirely | No | Yes |
| `PATCH` | Partial update (specific fields or state) | No | No |
| `DELETE` | Remove a resource | No | Yes |

### Rules

- `GET` must never modify state
- `POST` to a collection creates a new member
- `PUT` requires a full representation in the body
- `PATCH` is preferred for partial updates and state transitions
- `DELETE` is idempotent — deleting an already-deleted resource returns `200` or `204`, not `404`

---

## 5. Approved Shortcuts

These are the **only** approved shortenings. All other resource names use the full English word.

| Full word | Approved short form | Rationale |
|-----------|-------------------|-----------|
| `configurations` | `configs` | Universal in industry; zero ambiguity |
| `organizations` | `orgs` | Already used in JWT (`token.org`); consistent |
| `applications` | `apps` | Universal in industry; matches module type prefix |
| `references` | `refs` | Common technical shorthand; clear in context |
| `deployments` | `deployments` | **NOT shortened** — `deploys` reads as a verb |
| `privileges` | `privileges` | **NOT shortened** — security terms must be unambiguous |
| `dependencies` | `dependencies` | **NOT shortened** — `deps` is developer slang, not a URI contract |
| `permissions` | `permissions` | **NOT shortened** — security term |
| `notifications` | `notifications` | **NOT shortened** — not universally recognized as `notifs` |
| `registrations` | `registrations` | **NOT shortened** — too ambiguous |

**Adding new shortcuts:** Must be proposed in a PR to this file. Must include rationale and be approved by platform architecture. No one-off shortcuts in individual services.

---

## 6. Query Parameters

Query parameters are also nouns or noun phrases. No verb-style parameters.

### Filter parameters

```
?status=active          ✓  (noun: status)
?type=business-app      ✓  (noun: type)
?owner=O-92AF           ✓  (noun: owner)
?filter=active          ✗  ("filter" is a verb used as a noun — be specific: use ?status=)
?getOnly=published      ✗  (verb)
```

### Pagination parameters

Standard platform-wide pagination parameters (see `schemas/api/pagination.schema.json`):

```
?page=1                 page number (1-indexed)
?limit=20               items per page (default 20, max 100)
?sort=createdAt         sort field
?order=asc|desc         sort direction
```

### Search

```
?q=search+term          full-text search
?name=partial           field-specific search
```

---

## 7. Resource Nesting Depth

- Maximum nesting depth: **3 levels** of resource segments
- Beyond 3 levels: flatten by creating a top-level resource with a filter parameter

```
✓ /api/v1/O/O-92AF/configs/CFG-29F1/plans/PLN-81F3/benefits
  (3 levels: configs → plans → benefits)

✗ /api/v1/O/O-92AF/configs/CFG-29F1/plans/PLN-81F3/benefits/BEN-44A1/line-items
  (4 levels: too deep — create /api/v1/O/O-92AF/benefit-items?benefitId=BEN-44A1)
```

---

## 8. Versioning

- All APIs are versioned with a `v{N}` prefix immediately after `/api/`
- Current version: `v1`
- A new version (`v2`) is introduced only when breaking changes cannot be avoided
- `v1` must remain available for a documented deprecation period (minimum 6 months)
- Version is in the path, not in headers or query parameters

---

## 9. Response Envelope

All responses use the canonical envelope from `schemas/api/`. See those files for the full schema.

Error shape:
```json
{ "statusCode": 404, "error": "Not Found", "message": "Config CFG-29F1 not found" }
```

Paginated list shape:
```json
{
  "data": [...],
  "pagination": { "page": 1, "limit": 20, "total": 143, "pages": 8 }
}
```

---

## 10. Health Endpoint

Every service must expose a health endpoint at:

```
Internal:  GET /api/v1/G/health
External:  GET /api/{module-slug}/api/v1/G/health
```

No authentication required. Examples:

```
GET /api/identity/api/v1/G/health
GET /api/pcg4/api/v1/G/health
GET /api/module-registry/api/v1/G/health
```

Returns:

```json
{ "status": "ok", "service": "zorbit-cor-module_registry", "version": "1.0.0" }
```

---

## 11. Consistency Enforcement

### Manual review

Every PR that adds or modifies API endpoints must link to this document and confirm compliance.

### Automated validation (zorbit-cli)

```bash
zorbit validate-manifest ./zorbit-module-manifest.json
```

Checks performed:
- All `feRoute` values match `/m/{module-slug}/...` pattern (no org scope)
- All `beRoute` values match `/api/v1/{namespace}/...` pattern
- No path segment is a verb or past participle (blocklist enforced)
- All `beRoute` shortcuts are from the approved list only
- `feRoute` and `beRoute` are always declared together

### Blocklist — segments that will never appear in a URI

```
resolve, resolved, get, fetch, list, create, update, delete, remove,
notify, publish, activate, approve, reject, process, execute, run,
active, inactive, filtered, selected, pending (use ?status=pending instead)
```

---

## 12. nginx Routing Map

Every module has exactly one nginx location block that strips the module slug prefix before proxying.

```nginx
# Core services (zu-core bundle)
location /api/identity/      { proxy_pass http://zu-core:3001/api/; }
location /api/authorization/ { proxy_pass http://zu-core:3002/api/; }
location /api/navigation/    { proxy_pass http://zu-core:3003/api/; }
location /api/event-bus/     { proxy_pass http://zu-core:3004/api/; }
location /api/pii-vault/     { proxy_pass http://zu-core:3005/api/; }
location /api/audit/         { proxy_pass http://zu-core:3006/api/; }

# Module registry (standalone container)
location /api/module-registry/ { proxy_pass http://zu-module_registry:3036/api/; }

# PFS services (zu-pfs bundle)
location /api/form-builder/  { proxy_pass http://zu-pfs:3014/api/; }
location /api/datatable/     { proxy_pass http://zu-pfs:3013/api/; }
location /api/ai-gateway/    { proxy_pass http://zu-pfs:3009/api/; }

# Business apps (zu-apps bundle)
location /api/pcg4/          { proxy_pass http://zu-apps:3011/api/; }
location /api/hi-quotation/  { proxy_pass http://zu-apps:3xxx/api/; }
location /api/uw-workflow/   { proxy_pass http://zu-apps:3xxx/api/; }
location /api/hi-decisioning/{ proxy_pass http://zu-apps:3xxx/api/; }
```

The trailing `/api/` in `proxy_pass` is the key: nginx strips the matched prefix and appends `/api/`, making the internal path `GET /api/v1/...` as expected by NestJS.

---

## 13. Changelog

| Version | Date | Change |
|---------|------|--------|
| 1.0 | 2026-04-19 | Initial canonical document |
| 1.1 | 2026-04-19 | External URL grammar finalized: `/api/{module-slug}/api/v{N}/...` for ALL services (nginx strips slug prefix). Full approved route list added (§2.3). Old hybrid grammar abolished. |
