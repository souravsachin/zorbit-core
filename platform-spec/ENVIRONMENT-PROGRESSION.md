# Zorbit Environment Progression Model

*Audience: platform owner + devops + release managers.*

Version: 1.0
Last updated: 2026-04-23
Spec: `environments.yaml` v1.0

---

## The 5-tier model

Zorbit environments graduate through five tiers. Each tier gates promotion to the next. You cannot skip a tier.

```
tier 1: zorbit-dev     -->  tier 2: zorbit-qa
                                 |
                                 v
tier 5: zorbit-prod   <--  tier 4: zorbit-uat  <--  tier 3: zorbit-demo
```

| Tier | Name        | Purpose                               | Auto-deploy | Auto-seed demo data |
|------|-------------|---------------------------------------|-------------|---------------------|
| 1    | zorbit-dev  | Developer workstations + shared dev   | yes         | yes                 |
| 2    | zorbit-qa   | CI + automated tests                  | yes         | yes                 |
| 3    | zorbit-demo | Client demos, stakeholder walkthroughs| no (manual) | yes                 |
| 4    | zorbit-uat  | Paying customer UAT + pre-prod        | no          | no                  |
| 5    | zorbit-prod | Live production                       | no          | no                  |

---

## Tier characteristics

### Tier 1 — dev
- Container prefix: `ze-`
- Port base: 3100
- Shared TPMs: superset, odoo-shared, keycloak-shared, jitsi-shared
  (physical container names use `zs-` prefix — one instance serves all
  non-prod tiers; see `CONTAINER-NAMING.md`)
- Rollback retention: 7 days
- Developers push feature branches here freely.

### Tier 2 — qa
- Container prefix: `zq-`
- Port base: 3200
- Shared TPMs: same as dev
- Gated by: dev
- Rollback retention: 14 days
- CI runs the full test suite here.

### Tier 3 — demo
- Container prefix: `zd-`
- Port base: 3300
- Shared TPMs: superset, odoo-shared, keycloak-shared
- Gated by: qa
- Rollback retention: 30 days
- Pre-seeded with realistic demo data. No real customer data.

### Tier 4 — uat
- Container prefix: `zu-`
- Port base: 3100 (matches existing UAT)
- Shared TPMs: superset only. odoo + keycloak dedicated per customer.
- Gated by: demo
- Approval required: yes
- Rollback retention: 30 days

### Tier 5 — prod
- Container prefix: `zp-`
- Port base: 3400
- Shared TPMs: none — all dedicated. TPM containers use `zp-` prefix,
  never `zs-`. Production TPM data must never be shared with non-prod.
- Gated by: uat
- `--allow-prod` flag required
- `ZORBIT_PROD_APPROVAL_TOKEN` env var required
- Rollback retention: 90 days

---

## Promoting between tiers

Use `zorbit-cli/scripts/promote-env.sh`:

```
# dev -> qa
./promote-env.sh --from zorbit-dev --to zorbit-qa

# qa -> demo
./promote-env.sh --from zorbit-qa --to zorbit-demo

# demo -> uat  (approval required)
./promote-env.sh --from zorbit-demo --to zorbit-uat

# uat -> prod  (flag + token required)
ZORBIT_PROD_APPROVAL_TOKEN=<token> ./promote-env.sh \
    --from zorbit-uat --to zorbit-prod --allow-prod
```

### What `promote-env.sh` does

1. Validates source + target are adjacent tiers (no skipping).
2. For prod, validates `--allow-prod` + `ZORBIT_PROD_APPROVAL_TOKEN`.
3. Snapshots the source env's module-registry state to `/opt/zorbit-platform/snapshots/<from>-to-<to>-<timestamp>.yaml`.
4. Runs smoke tests against the source. Aborts if any fail.
5. Checks out the exact git SHAs that produced the source's running code.
6. Builds + deploys those SHAs to the target using `bootstrap-env.sh --env <to> --yes`.
7. Verifies the target matches the source's version table. Reports drift.
8. Emits a promotion report.

### Dry-run

Always dry-run before the real thing:

```
./promote-env.sh --from zorbit-dev --to zorbit-qa --dry-run
```

---

## Rollback

Snapshots in `/opt/zorbit-platform/snapshots/` retain the exact SHAs that were deployed to each tier at each promotion. To roll back:

```
# List snapshots
ls -lh /opt/zorbit-platform/snapshots/

# Identify the snapshot you want to restore
# Example: zorbit-qa-to-zorbit-demo-20260423-143000.yaml

# Re-run promotion using that snapshot as input (WIP — planned for v1.1)
```

Note: `--rollback` flag is reserved for v1.1. For now, manual checkout + bootstrap-env.sh is the workflow.

---

## Safety rules

- **Never auto-promote to prod.** Even with `--allow-prod`, the script requires an approval token.
- **Smoke-test failures abort promotion.** Fix the source before retrying.
- **Rollback snapshots are retained for 30+ days** (90 for prod). Do not delete.
- **No tier skipping.** qa cannot promote directly to uat.
- **Cross-tier DB migrations** are handled manually — promote-env.sh does not migrate data.

---

## When to use each tier

| Scenario                          | Tier        |
|-----------------------------------|-------------|
| Developing a new module           | dev         |
| Running integration tests         | qa          |
| Showing a prospect a live demo    | demo        |
| Paying customer integration test  | uat         |
| Real users, real money            | prod        |

---

## Exit codes (promote-env.sh)

| Code | Meaning                                       |
|------|-----------------------------------------------|
| 0    | Promotion succeeded                           |
| 1    | Spec file missing or unreadable               |
| 2    | Invalid args (unknown env, skipped tier, etc.)|
| 4    | Smoke test failed or deploy failed            |
