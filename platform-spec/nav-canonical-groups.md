# Canonical Navigation Groups

**Status:** Approved (cycle-106, 2026-04-27, MSG-101 issues #5/#6)
**Owner:** Platform Architecture
**Spec source of truth:** `schemas/module-manifest/manifest.schema.json`

---

## Why

Pre-cycle-106 every module's nav items rendered as flat siblings under the
module root. So a module that ships Intro / Presentation / Lifecycle /
Videos / Resources / Pricing **plus** working pages (e.g. Users,
Organizations, Sessions) showed all of them at the same depth — six
"Guide" siblings drowning out the working pages. Modules that need
Database Operations, Setup, or Deployment screens had nowhere canonical
to put them.

Owner pain (paraphrased): *"I have shouted at the top of my lungs so so
many times. Why this is still unfixed?"*

Cycle-106 fixes this at the schema layer.

## What

Every Zorbit module manifest may now declare nav sections that map to
**four canonical groups**:

| Group ID | Display Label | Purpose | Typical contents |
|----------|---------------|---------|------------------|
| `guide`      | Guide      | Module-as-content                  | Intro, Presentation, Lifecycle, Videos, Resources, Pricing |
| `data`       | Data       | Working business pages             | Users, Organizations, Sessions, Logs, Records, Reports |
| `setup`      | Setup      | Module configuration               | Configuration, Templates, Reference data, Privilege seeds |
| `deployment` | Deployment | Operations / health / DB ops       | Database operations, Health, Manifest, Versions, Containers |

Sections whose `id` is one of these four (or whose `groupId` is set) are
rendered by the SPA as a **named wrapper node** nested under the module.
Sections with any other `id`, or with no `id` at all, render flat — so
existing manifests continue to work without modification.

## How (manifest authors)

### Pattern A — section-as-group (recommended)

Place each canonical group as a top-level entry in
`navigation.sections[]` with the matching `id`:

```json
{
  "navigation": {
    "sections": [
      {
        "id": "guide",
        "label": "Guide",
        "sortOrder": 0,
        "icon": "menu_book",
        "items": [
          { "label": "Intro",        "feRoute": "/m/identity/guide/intro",        "sortOrder": 1, "feComponent": "@platform:GuideIntroView" },
          { "label": "Presentation", "feRoute": "/m/identity/guide/presentation", "sortOrder": 2, "feComponent": "@platform:GuideSlideDeck" },
          { "label": "Lifecycle",    "feRoute": "/m/identity/guide/lifecycle",    "sortOrder": 3, "feComponent": "@platform:GuideLifecycle" },
          { "label": "Videos",       "feRoute": "/m/identity/guide/videos",       "sortOrder": 4, "feComponent": "@platform:GuideVideos" },
          { "label": "Resources",    "feRoute": "/m/identity/guide/resources",    "sortOrder": 5, "feComponent": "@platform:GuideDocs" },
          { "label": "Pricing",      "feRoute": "/m/identity/guide/pricing",      "sortOrder": 6, "feComponent": "@platform:GuidePricing" }
        ]
      },
      {
        "id": "data",
        "label": "Data",
        "sortOrder": 10,
        "items": [
          { "label": "Users",         "feRoute": "/m/identity/users",   "sortOrder": 1 },
          { "label": "Organizations", "feRoute": "/m/identity/orgs",    "sortOrder": 2 },
          { "label": "Sessions",      "feRoute": "/m/identity/sessions","sortOrder": 3 }
        ]
      },
      {
        "id": "setup",
        "label": "Setup",
        "sortOrder": 20,
        "items": [
          { "label": "Configuration", "feRoute": "/m/identity/setup/config", "sortOrder": 1 }
        ]
      },
      {
        "id": "deployment",
        "label": "Deployment",
        "sortOrder": 30,
        "items": [
          { "label": "Database Operations", "feRoute": "/m/identity/db", "sortOrder": 1 }
        ]
      }
    ]
  }
}
```

The SPA renders this as:

```
Identity (module)
├── Guide
│   ├── Intro
│   ├── Presentation
│   ├── Lifecycle
│   ├── Videos
│   ├── Resources
│   └── Pricing
├── Data
│   ├── Users
│   ├── Organizations
│   └── Sessions
├── Setup
│   └── Configuration
└── Deployment
    └── Database Operations
```

### Pattern B — custom label, canonical group

If you want a custom section label but still want the SPA to nest items
under the canonical group wrapper, set `groupId` explicitly:

```json
{
  "id": "claims_registry",
  "label": "Claims Database",
  "groupId": "data",
  "items": [ ... ]
}
```

### Pattern C — single item override

A single item that needs to live in a canonical group without creating
a whole section can declare `groupId` at the item level:

```json
{
  "items": [
    {
      "label": "Database Operations",
      "feRoute": "/m/identity/db",
      "groupId": "deployment",
      "sortOrder": 99
    }
  ]
}
```

The item is hoisted out of its declared section and placed under the
named wrapper.

## Backwards compatibility

- **Sections without canonical id/groupId render flat** — existing
  manifests unchanged.
- The SPA continues to honour the legacy `feComponent`-based Guide
  detection (`@platform:GuideIntroView`, etc.) when no canonical id is
  declared. This means manifests that rely on the old auto-grouping keep
  working.
- New manifests should prefer the explicit canonical group ids — they
  are clearer, work for non-Guide groups, and don't depend on FE
  component naming.

## SPA implementation reference

`02_repos/zorbit-unified-console/src/components/layout/Sidebar6Level.tsx`,
function `groupItemsByCanonicalGroup()`. The renderer:

1. Inspects every flat item received from the cascade resolver.
2. Looks up its source section's `id` and `groupId` (passed alongside
   the item under `__sectionGroupId` by the cascade resolver).
3. Looks up the item's own `groupId`.
4. If any of those resolve to one of `guide` / `data` / `setup` /
   `deployment`, the item is collected into that group's bucket.
5. The four group buckets become wrapper nodes ("Guide", "Data",
   "Setup", "Deployment") in canonical order.
6. Remaining flat items keep their original positions between/after the
   wrapper nodes.

The sort order between wrapper nodes is fixed:
`guide < data < setup < deployment`. Within each wrapper, items sort by
their declared `sortOrder`.

## Cascade resolver contract

`02_repos/zorbit-navigation/src/services/nav-cascade-resolver.service.ts`
flattens module sections into a single `sections[]` array for the SPA.
Cycle-106 extends each flattened entry with:

| Field | Source |
|-------|--------|
| `sectionId`    | `manifest.navigation.sections[].id` |
| `sectionLabel` | `manifest.navigation.sections[].label` |
| `groupId`      | `manifest.navigation.sections[].groupId` (or canonical id if no explicit groupId) |

Each item in the flattened array also carries `groupId` (the item-level
override, if declared).

This is additive — earlier consumers ignore the new fields.
