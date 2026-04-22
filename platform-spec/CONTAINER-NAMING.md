# Zorbit Container Naming Convention

*Audience: platform owner + devops + anyone writing docker-compose, bootstrap scripts, or TPM wrappers.*

Version: 1.0
Last updated: 2026-04-23
Spec consumers: `zorbit-cli/scripts/bootstrap-env.sh`, `zorbit-core/platform-spec/all-repos.yaml`, `zorbit-core/platform-spec/environments.yaml`, every repo's `docker-compose*.yml`, every TPM wrapper service.

---

## The rule (one sentence)

```
Platform services per-environment:      z{e,q,d,u,p}-<module>
Shared infra per-tenancy (one instance services many envs):   zs-<service>
Prod-isolated TPM:                      zp-<service>
```

That's it. If you remember nothing else, remember that.

---

## Why the rule exists

- `zs-` = **Shared** across environments. One physical container instance serves dev + qa + demo + uat. Examples: `zs-pg`, `zs-kafka`, `zs-superset`.
- Environment-prefixed containers (`ze-`, `zq-`, `zd-`, `zu-`, `zp-`) are one-per-environment. Each tier gets its own copy.
- Production must never share TPM data with non-prod. Therefore prod TPMs use `zp-` (dedicated) and never `zs-`.

If a container could theoretically be shared across non-prod envs but someone deployed it as `zu-<service>`, that's **drift**. Fix it before production deploy.

---

## Reference tables

### Platform services per-environment

| Prefix | Tier          | Example            |
|--------|---------------|--------------------|
| `ze-`  | dev           | `ze-identity`      |
| `zq-`  | qa            | `zq-identity`      |
| `zd-`  | demo          | `zd-identity`      |
| `zu-`  | uat           | `zu-identity`      |
| `zp-`  | prod          | `zp-identity`      |

### Shared infra (non-prod, always `zs-`)

| Container          | Purpose                      |
|--------------------|------------------------------|
| `zs-pg`            | PostgreSQL (all services)    |
| `zs-mongo`         | MongoDB (datatable/forms/etc)|
| `zs-kafka`         | Kafka event bus              |
| `zs-redis`         | Redis cache                  |

### Shared third-party modules (TPMs) — non-prod, always `zs-`

| Container          | Purpose                                |
|--------------------|----------------------------------------|
| `zs-superset`      | BI dashboards (wraps Apache Superset)  |
| `zs-superset-redis`| Superset result cache / celery broker  |
| `zs-odoo`          | ERP (wraps Odoo Community)             |
| `zs-keycloak`      | SSO / federated identity               |
| `zs-jitsi`         | Meetings / video rooms                 |
| `zs-mailhog`       | Outbound mail capture for dev/test     |

### Prod-dedicated TPMs (per-customer isolation, always `zp-`)

| Container            | Purpose                       |
|----------------------|-------------------------------|
| `zp-superset-<cust>` | BI dashboards (per customer)  |
| `zp-odoo-<cust>`     | ERP (per customer)            |
| `zp-keycloak-<cust>` | SSO (per customer)            |

---

## Resolution algorithm (for compose generators)

Given an env (`dev`/`qa`/`demo`/`uat`/`prod`) and a logical service name, the physical container name is:

```
1. If service is declared in all-repos.yaml `infrastructure:` or `tpms_shared:`
   and env != prod:
       → use the `zs-<name>` as declared, unchanged.

2. If service is declared in all-repos.yaml `infrastructure:` or `tpms_shared:`
   and env == prod:
       → rewrite `zs-` to `zp-`. Operator must confirm a dedicated instance
         exists before deploy; no shared-with-non-prod fallback.

3. If service is a platform service (declared in all-repos.yaml `repos:`
   with type=service):
       → use `z{e,q,d,u,p}-<module>` where the prefix letter comes from
         environments.yaml `container_prefix` for the target env.
```

Reference implementation: `zorbit-cli/scripts/bootstrap-lib/services.sh`, `generate_compose_file()`.

---

## Checking for drift

```bash
# On any environment, list containers that violate the rule:
docker ps --format '{{.Names}}' | grep -vE '^(ze|zq|zd|zu|zp|zs)-'

# List zu- TPMs that should probably be zs-:
docker ps --format '{{.Names}}' \
  | grep -E '^zu-(superset|odoo|keycloak|jitsi|mailhog)'
```

If the second command finds anything, it's drift — either rename via `docker rename <old> <new>` during a maintenance window, or re-create from the updated compose file.

---

## Migration note — 2026-04-23

The UAT ARM server (`141.145.155.34`) was seeded with `zu-superset` + `zu-superset-redis` before this spec was written. Rename planned for the next maintenance window:

```bash
# Maintenance window only — requires nginx reload + restart of dependent PFS:
docker rename zu-superset zs-superset
docker rename zu-superset-redis zs-superset-redis
# Update nginx upstream in /etc/nginx/conf.d/zorbit-uat.conf to point at
# zs-superset:8088 instead of zu-superset:8088, then nginx -s reload.
# Restart zorbit-pfs-analytics_reporting container so it re-resolves DNS.
```

Until the rename is done, set `SUPERSET_URL=http://zu-superset:8088` in the
`zorbit-pfs-analytics_reporting` env on the UAT host as an override.

---

## Related specs

- `environments.yaml` v1.0 — tier definitions and `shared_tpms` logical lists
- `ENVIRONMENT-PROGRESSION.md` — tier-by-tier characteristics
- `all-repos.yaml` — `infrastructure:` and `tpms_shared:` blocks
