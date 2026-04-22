# Zorbit Deployment Guide

*Audience: devops engineers deploying Zorbit on a fresh machine. No prior Zorbit knowledge required.*

Version: 1.1
Last updated: 2026-04-23
Spec: `zorbit-core/platform-spec/environments.yaml` v1.0, `all-repos.yaml` v1.1

---

## Summary

Two scripts, run in order, from `zorbit-cli/scripts/`:

1. `bootstrap-env-root.sh --env <e>` — run **once per env** as root. Creates the env-specific service account (e.g. `zorbit-dev`), sets up per-env systemd + logrotate, installs shared system packages.
2. `bootstrap-env.sh --env <e>` — run as **the env-specific service account** (e.g. `zorbit-dev`). Pulls pre-built images (or unpacks a bundle), starts services, generates nginx config from the pre-cooked template.

Anything requiring `sudo` is isolated to script 1. Script 2 does **zero** sudo calls.

---

## What changed on 2026-04-23 (five design-flaw fixes)

The installer was redesigned around owner feedback. If you are familiar with an older version, note:

1. **No git clone + source build.** The installer pulls pre-built Docker images from the container registry (default `ghcr.io/souravsachin/<module>:<tag>`) or unpacks a pre-built tarball bundle for air-gapped installs. Source checkout is never required on a deploy target.
2. **No caffeinate prompt.** That was a macOS-laptop artifact and has been removed.
3. **Every y/n prompt shows help.** A two-line help block prints above each prompt explaining what "yes" and "no" will do before you choose.
4. **Per-env service account + install dir.** Retired: the umbrella `zorbit-deployer` account. Now each environment has its own isolated account (`zorbit-dev`, `zorbit-qa`, `zorbit-demo`, `zorbit-uat`, `zorbit-prod`) whose `$HOME` *is* that environment's eco-system root.
5. **Pre-cooked nginx template.** Location blocks for all 50+ modules are baked into `scripts/templates/nginx-precooked.conf`. The installer substitutes only `{{HOSTNAME}}` and `{{ENV_PREFIX}}` via `sed`.

---

## Per-environment account + install dir matrix

| Env           | Service account | Install dir (==$HOME) | Container prefix |
|---------------|-----------------|-----------------------|------------------|
| `zorbit-dev`  | `zorbit-dev`    | `/home/zorbit-dev/`   | `ze-`            |
| `zorbit-qa`   | `zorbit-qa`     | `/home/zorbit-qa/`    | `zq-`            |
| `zorbit-demo` | `zorbit-demo`   | `/home/zorbit-demo/`  | `zd-`            |
| `zorbit-uat`  | `zorbit-uat`    | `/home/zorbit-uat/`   | `zu-`            |
| `zorbit-prod` | `zorbit-prod`   | `/home/zorbit-prod/`  | `zp-`            |

Complete isolation rules:
- Each account's `$HOME` is that env's root (artifacts, compose files, generated nginx config, install journal all live there).
- Each account can ONLY touch its own env's resources (enforced by filesystem permissions + the bootstrap account-guard).
- Docker group membership is required on every account (docker CLI must work for the account).
- There is no umbrella account. Trying to run `bootstrap-env.sh --env dev` as `zorbit-qa` exits with code 3.

---

## Prerequisites

- Linux server (Ubuntu 22.04 / 24.04 or Debian 12 tested)
- Root access for the one-time prep step
- 30 GB free disk at `/opt/zorbit-platform`
- Public DNS record pointing to the server (or Cloudflare)
- Outbound internet for `docker pull` (registry mode) OR a pre-built bundle file/URL (bundle mode)

---

## Step-by-step (devops walking in cold)

### 1. Clone zorbit-cli + zorbit-core as root (temporary, one-time)

```
cd /root
git clone https://github.com/souravsachin/zorbit-cli.git
git clone https://github.com/souravsachin/zorbit-core.git
```

### 2. Dry-run the root prep script for the envs you want to provision

**Always dry-run first.** Batch supported via comma-separated `--env`.

```
sudo bash /root/zorbit-cli/scripts/bootstrap-env-root.sh --env dev --dry-run
sudo bash /root/zorbit-cli/scripts/bootstrap-env-root.sh --env dev,qa,demo,uat,prod --dry-run
```

Review every command. If anything looks wrong, stop and escalate.

### 3. Run the root prep script

```
sudo bash /root/zorbit-cli/scripts/bootstrap-env-root.sh --env dev
```

Per env, this creates:
- user `zorbit-<env>` with `$HOME=/home/zorbit-<env>` (shell: `/bin/bash`, in docker group)
- `${HOME}/artifacts/`, `${HOME}/compose/`, `${HOME}/config/` (install dir subtree)
- `/opt/zorbit-platform/<env-name>/` (data volumes, owned by the env's account only)
- `/var/log/zorbit-platform/<env-name>/` (logs, owned by the env's account only)
- systemd unit `<env-name>.service` (auto-start on boot, runs as the env's account)
- logrotate at `/etc/logrotate.d/<env-name>`

Shared one-time installs (only installed if missing):
- docker + docker-compose-plugin, nodejs 20, nginx, certbot, gh CLI, python3-yaml

### 4. Switch to the env-specific service account

```
sudo -u zorbit-dev -i
```

Note: do **not** switch to an umbrella account — there is no such thing any more.

### 5. Get zorbit-cli + zorbit-core available to the service account

The service account needs to read `zorbit-core/platform-spec/*` and run the `bootstrap-env.sh` script. Easiest path:

```
cp -r /root/zorbit-cli /home/zorbit-dev/
cp -r /root/zorbit-core /home/zorbit-dev/
chown -R zorbit-dev:zorbit-dev /home/zorbit-dev/zorbit-cli /home/zorbit-dev/zorbit-core
```

Or clone inside the service account shell if internet + gh auth are available there.

### 6. Dry-run the bootstrap

```
cd ~/zorbit-cli
./scripts/bootstrap-env.sh --env dev --dry-run
```

You will be prompted for:

| Prompt                              | Default                             |
|-------------------------------------|-------------------------------------|
| Full hostname                       | `zorbit-dev.onezippy.ai`            |
| DNS configured? (y/n + help)        | `y`                                 |
| Data volume path                    | `/opt/zorbit-platform/data`         |
| Artifact source (1=registry, 2=bundle) | `1`                              |
| Registry base (if 1)                | `ghcr.io/souravsachin`              |
| Image tag (if 1)                    | `latest`                            |
| Bundle path/URL (if 2)              | `(required)`                        |
| Superset shared? (y/n + help)       | `y`                                 |
| Odoo shared? (y/n + help)           | `y`                                 |
| Keycloak shared? (y/n + help)       | `y`                                 |
| Jitsi shared? (y/n + help)          | `y`                                 |
| Confirm summary? (y/n + help)       | `n`                                 |

The dry-run prints every command it would execute. Review it.

#### What "no" does on each y/n prompt

| Question | yes | no |
|----------|-----|-----|
| DNS configured? | Proceed to nginx step. | Print DNS-setup instructions (A record for hostname → server IP) and abort (exit 1). |
| Superset shared? | Point env's `SUPERSET_URL` at `http://zs-superset:8088` | Skip — no BI analytics for this env until a dedicated Superset is added later. |
| Odoo shared? | Point env's `ODOO_URL` at `http://zs-odoo:8069` | No ERP integration for this env. |
| Keycloak shared? | Point env's `KEYCLOAK_URL` at `http://zs-keycloak:8080` | Fall back to Zorbit's native identity service only (no SSO). |
| Jitsi shared? | Point env's `JITSI_URL` at `http://zs-jitsi:8443` | No video-conferencing module in this env. |
| Confirm summary? | Execute the install. | Abort with exit 2, no state changes. |

### 7. Real bootstrap

```
./scripts/bootstrap-env.sh --env dev
```

Registry-mode installs take 2-5 minutes (time-dominated by `docker pull`s).
Bundle-mode installs depend on bundle size + download speed.

### 8. Install nginx config (needs sudo)

`bootstrap-env.sh` renders the pre-cooked template to `${HOME}/config/<hostname>.nginx.conf` and prints install instructions. Copy them into your sudo shell:

```
sudo cp /home/zorbit-dev/config/zorbit-dev.onezippy.ai.nginx.conf \
        /etc/nginx/sites-available/zorbit-dev.onezippy.ai
sudo ln -sf /etc/nginx/sites-available/zorbit-dev.onezippy.ai \
            /etc/nginx/sites-enabled/zorbit-dev.onezippy.ai
sudo nginx -t && sudo systemctl reload nginx
```

If no wildcard cert exists:

```
sudo certbot --nginx -d zorbit-dev.onezippy.ai
```

### 9. Verify

```
curl -I https://zorbit-dev.onezippy.ai/
./scripts/smoke-test.sh --env zorbit-dev
```

---

## Artifact source — registry vs bundle

### Registry (recommended)

- Default. Pulls images from `ghcr.io/souravsachin/<module>:<tag>` for every runtime module in `all-repos.yaml`.
- Requires outbound internet + registry authentication (`docker login ghcr.io` beforehand if repos are private).
- `ZORBIT_IMAGE_TAG` env var overrides the default `latest` tag globally.

### Bundle (air-gapped)

- For client environments with no outbound internet.
- Operator supplies a tarball (local path or HTTPS URL) at the "Bundle location" prompt.
- Bundle layout:
  ```
  zorbit-v<version>-<arch>.tar.gz
    ├── images.tar         (docker save of all runtime images)
    └── manifest.json      (image -> repo map, versions)
  ```
- The installer extracts the tarball, runs `docker load -i images.tar`, then proceeds identically to registry mode.

---

## Exit codes

| Code | Meaning                                           |
|------|---------------------------------------------------|
| 0    | Success                                           |
| 1    | Preflight check failed (or DNS answered `no`)     |
| 2    | User cancelled at confirmation prompt             |
| 3    | Service-account guard rejected current user       |
| 4    | Deploy step failed (pull, compose, etc.)          |

---

## Common troubleshooting

### "BLOCKED — This script must run as zorbit-dev for env zorbit-dev"

You are logged in as the wrong account. Switch: `sudo -u zorbit-dev -i`, then re-run.

Override (at your own risk, repair/forensic work only): `ZORBIT_SERVICE_USER=$(whoami) ./bootstrap-env.sh --env dev`.

### "docker pull: unauthorized"

Registry credentials are missing. Run `docker login ghcr.io` as the service account before retrying. Or switch to bundle mode (`--env ... ` then choose `2` at the artifact-source prompt).

### "docker: permission denied"

The service account is not in the `docker` group. The root prep script does this; if you created the account manually: `sudo usermod -aG docker zorbit-<env>`, then log out and back in.

### "Environment spec missing"

`zorbit-core` is not in `$REPO_ROOT_GUESS`. Default: `$HOME/workspace/zorbit/02_repos`. Set `REPO_ROOT_GUESS=<path>` env var to override.

### Ports busy

Something is already on 80/443 or the env's port base (3100/3200/3300/3100/3400). Stop the conflicting service, or choose a different environment.

### Bundle not found

Bundle-mode requires either a local file path or an HTTPS URL. Check the path exists and the service account can read it. For URLs, verify the service account has outbound HTTPS to that host.

---

## What `bootstrap-env.sh` does NOT do

- Does not run anything as root (by design).
- Does not push to GitHub.
- Does not clone source code or build from source (owner rule 2026-04-23).
- Does not touch production without explicit `--allow-prod` flag.
- Does not migrate data — fresh DBs only.
- Does not install TLS certs — uses existing wildcard, or you run certbot.
- Does not start the compose file automatically — review `${HOME}/compose/docker-compose.*.yml` first.

---

## File reference

| File                                              | Purpose                                  |
|---------------------------------------------------|------------------------------------------|
| `zorbit-cli/scripts/bootstrap-env.sh`             | Main bootstrap (env-specific service account) |
| `zorbit-cli/scripts/bootstrap-env-root.sh`        | Root-only prep (creates per-env account) |
| `zorbit-cli/scripts/bootstrap-lib/*.sh`           | Modular helpers                          |
| `zorbit-cli/scripts/templates/nginx-precooked.conf` | Pre-cooked nginx template (all modules)|
| `zorbit-cli/scripts/templates/Dockerfile.pm2-base`| Legacy shared base image (unused after flaw 1 fix) |
| `zorbit-cli/scripts/promote-env.sh`               | Tier progression (dev -> qa -> ...)      |
| `zorbit-core/platform-spec/environments.yaml`     | Env spec + per-env account + install dir |
| `zorbit-core/platform-spec/all-repos.yaml`        | Repository manifest v1.1 (with `image:` per runtime repo) |
| `zorbit-core/platform-spec/CONTAINER-NAMING.md`   | `z{e,q,d,u,p}-` vs `zs-`/`zp-` rule      |
