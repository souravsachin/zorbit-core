# Zorbit Module Manifest Schema v2

## Overview

Every Zorbit module (platform service or business application) registers itself with the platform by providing a **Module Manifest**. This is the contract between a module and the Zorbit platform.

The manifest file must be placed at the root of the module's repository as `zorbit-module-manifest.json`.

## Schema

- **Schema file:** `manifest.schema.json` (JSON Schema draft 2020-12)
- **Example:** `manifest.example.json` (zorbit-app-hi_quotation)

## Required Fields

| Field | Type | Description |
|---|---|---|
| moduleId | string | Unique identifier (snake_case, e.g. `app_hi_quotation`) |
| moduleName | string | Human-readable display name |
| moduleType | enum | `platform-service` or `business-app` |
| version | string | Semantic version (e.g. `1.0.0`) |

## Manifest Sections

| Section | Platform Action |
|---|---|
| `navigation` | zorbit-navigation auto-creates menu items + registers required privileges |
| `frontend` | Admin console / shell loads bundle on route match (eager, lazy, or on-demand) |
| `backend` | Health monitoring registered, nginx proxy auto-configured |
| `dashboard` | Dashboard framework discovers and renders widgets based on tier priority |
| `reporting` | Central reporting UI shows module's data sources for ad-hoc queries and scheduling |
| `seeding` | Seeding service discovers seed/flush endpoints, respects dependency order |
| `events` | Event registry validates topic names, documents contracts, enables discoverability |
| `database` | Platform SDK applies interceptors (PII auto-detection, audit trail, quota tracking) |
| `pii` | PII vault registers known fields + enables auto-detection for untagged fields |
| `analytics` | Observability service provisions Grafana dashboards + alerts from metric definitions |
| `dependencies` | Startup ordering, health check dependencies, circuit breaker configuration |

## Dashboard Widget Tiers

| Tier | Behavior |
|---|---|
| 1 | Always visible |
| 2 | Shown if space available |
| 3 | Collapsed/hidden by default |

## Module Registration API

Register a module manifest with the navigation service:

```
POST /api/v1/G/navigation/register-module
Body: { "manifest": <full manifest JSON> }
```

The endpoint validates the manifest, creates/updates menu items, and registers the module.

## Listing Registered Modules

```
GET /api/v1/G/navigation/modules
GET /api/v1/G/navigation/modules/:moduleId
DELETE /api/v1/G/navigation/modules/:moduleId
GET /api/v1/G/navigation/modules/:moduleId/health
```

## Migration from v1

The v1 schema (`zorbit-manifest.schema.json` in `/schemas/manifest/`) used a different structure with `module`, `menu`, `privileges`, `events`, and `health` top-level keys. The v2 schema is a complete rewrite that adds dashboard, reporting, seeding, database, PII, and analytics sections.

Modules using v1 manifests should migrate to v2. The `register-module` API only accepts v2 format.
