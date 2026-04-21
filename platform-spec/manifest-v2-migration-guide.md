# Module Manifest v2 Migration Guide

**Version:** 2.0
**Owner:** Platform Architecture
**Last updated:** 2026-04-18

---

## 1. What Changed in v2

### New required fields

Two blocks are now required on every manifest. Manifests missing either field will
fail schema validation and be rejected by the module registry.

| Field | Purpose |
|-------|---------|
| `placement` | Declares the module's position in the Edition > Business Line > Capability Area hierarchy. Used for sidebar grouping and admin discovery. |
| `registration` | Kafka self-registration config. The module emits this on startup so the registry can locate and fetch the full manifest. |

### `moduleType` enum updated

The `moduleType` value must now match the repository type prefix:

| Value | Meaning |
|-------|---------|
| `cor` | Core platform service (zorbit-cor-*) |
| `pfs` | Platform feature service (zorbit-pfs-*) |
| `app` | Business application (zorbit-app-*) |
| `tpm` | Third-party module integration (zorbit-tpm-*) |
| `sdk` | SDK library (zorbit-sdk-*) |

Old values `"platform-service"` and `"business-app"` are no longer valid.

### `feRoute` validation enforced

All `feRoute` values are now validated against the pattern `^/m/`. Any route that
does not start with `/m/` will fail schema validation.

### `frontend.bundleUrl` must be a full URL

The `bundleUrl` field now requires `format: uri`. Relative paths like
`/modules/pcg4/bundle.js` are invalid. Use a fully-qualified CDN URL.

---

## 2. Frontend Route Rule

> No org scope in FE routes. No exceptions.

### Pattern

```
/m/{module-slug}/{feature-slug}[/{resource}/{resource-id}]
```

The org ID is taken from the JWT at runtime. It never appears in the URL.

### Correct

```
/m/pcg4/overview
/m/pcg4/configs
/m/pcg4/configs/CFG-29F1
/m/pcg4/refs
/m/pcg4/deployments
/m/form-builder/forms/FRM-44A1/submissions
```

### Incorrect

```
/org/{{org_id}}/app/pcg4/overview       -- org scope in URL
/app/pcg4/overview                      -- missing /m prefix
/m/pcg4/c/CFG-29F1                      -- single-char abbreviation
/m/pcg4/configs/CFG-29F1/edit          -- "edit" is a verb
```

---

## 3. Approved Shortcuts (Section 5 of uri-conventions.md)

Only these shortenings are permitted. All other names use the full English word.

| Full word | Approved short form | Notes |
|-----------|-------------------|-------|
| `configurations` | `configs` | Universal in industry |
| `organizations` | `orgs` | Matches JWT `token.org` |
| `applications` | `apps` | Matches module type prefix |
| `references` | `refs` | Clear in context |
| `deployments` | `deployments` | NOT shortened — "deploys" is a verb |
| `privileges` | `privileges` | NOT shortened — security term |
| `dependencies` | `dependencies` | NOT shortened — "deps" is slang |
| `permissions` | `permissions` | NOT shortened — security term |
| `notifications` | `notifications` | NOT shortened |
| `registrations` | `registrations` | NOT shortened |

To propose a new shortcut: open a PR against `uri-conventions.md` with a rationale.
No one-off shortcuts in individual service manifests.

---

## 4. Migration Checklist

Work through this list for every module manifest being migrated from v1 to v2.

- [ ] Add `placement` block with `edition`, `businessLine`, `capabilityArea` (and optionally `sortOrder`)
- [ ] Add `registration` block with `kafkaTopic: "platform.module.announcements"` and a full `manifestUrl`
- [ ] Update `moduleType` from `"platform-service"` or `"business-app"` to the correct v2 value (`cor`, `pfs`, `app`, `tpm`, or `sdk`)
- [ ] Fix every `feRoute` — remove org scope, add `/m/` prefix
- [ ] Fix `frontend.routes[]` — replace `/org/*/app/{slug}/*` with `/m/{slug}/*`
- [ ] Fix `frontend.bundleUrl` — replace relative path with full CDN URL
- [ ] Apply approved shortcuts in `beRoute` and `backend.endpoints[].path` — replace `configurations` with `configs`, `references` with `refs`, `organizations` with `orgs`
- [ ] Check `backend.apiPrefix` — apply the same shortcut replacements
- [ ] Check dashboard `dataEndpoint` values — apply shortcut replacements
- [ ] Check reporting `dataEndpoint` values — apply shortcut replacements
- [ ] Delete any duplicate legacy manifest file (e.g. `zorbit-manifest.json`)
- [ ] Run `zorbit validate-manifest ./zorbit-module-manifest.json` to confirm schema compliance

---

## 5. Example placement + registration blocks

```json
"placement": {
  "edition": "insurer",
  "businessLine": "distribution",
  "capabilityArea": "product-management",
  "sortOrder": 10
},

"registration": {
  "kafkaTopic": "platform.module.announcements",
  "manifestUrl": "https://cdn.onezippy.ai/modules/pcg4/zorbit-module-manifest.json"
}
```

See `schemas/module-manifest/manifest.example.json` for a complete v2 reference.
