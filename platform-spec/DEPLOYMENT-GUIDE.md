# Zorbit Deployment Guide

*Audience: devops engineers deploying Zorbit on a fresh machine. No prior Zorbit knowledge required.*

Version: 1.0
Last updated: 2026-04-23
Spec: `zorbit-core/platform-spec/environments.yaml` v1.0

---

## Summary

Two scripts, run in order, from `zorbit-cli/scripts/`:

1. `bootstrap-env-root.sh` — run **once** as root. Creates the service account, installs system packages, sets up systemd + logrotate.
2. `bootstrap-env.sh` — run as the service account (`zorbit-deployer`). Clones repos, builds images, starts services, generates nginx config.

Anything requiring `sudo` is isolated to script 1. Script 2 does **zero** sudo calls.

---

## Prerequisites

- Linux server (Ubuntu 22.04 / 24.04 or Debian 12 tested)
- Root access for the one-time prep step
- 30 GB free disk at `/opt/zorbit-platform`
- Public DNS record pointing to the server (or Cloudflare)
- SSH key enrolled on GitHub for `souravsachin/zorbit-*` repos
- Outbound internet for docker + npm + apt

---

## Step-by-step (devops walking in cold)

### 1. Clone zorbit-cli + zorbit-core as root (temporary)

```
cd /root
git clone https://github.com/souravsachin/zorbit-cli.git
git clone https://github.com/souravsachin/zorbit-core.git
```

### 2. Dry-run the root prep script

**Always dry-run first.**

```
sudo bash /root/zorbit-cli/scripts/bootstrap-env-root.sh --dry-run
```

Review every command it would run. If anything looks wrong, stop and escalate.

### 3. Run the root prep script

```
sudo bash /root/zorbit-cli/scripts/bootstrap-env-root.sh
```

This creates:
- user `zorbit-deployer` (home: `/home/zorbit-deployer`, shell: `/bin/bash`)
- `/opt/zorbit-platform/{data,snapshots}` owned by `zorbit-deployer`
- `/var/log/zorbit-platform/` for logs
- systemd unit `zorbit-platform.service` (auto-start on boot)
- logrotate at `/etc/logrotate.d/zorbit-platform`
- installs: docker, docker-compose-plugin, nodejs 20, nginx, certbot, gh CLI, python3-yaml

### 4. Switch to the service account

```
sudo -u zorbit-deployer -i
```

### 5. Clone zorbit-cli + zorbit-core into the service account's workspace

```
mkdir -p ~/workspace/zorbit/02_repos
cd ~/workspace/zorbit/02_repos
git clone https://github.com/souravsachin/zorbit-cli.git
git clone https://github.com/souravsachin/zorbit-core.git
```

### 6. Authenticate `gh` and SSH as the service account

```
gh auth login                                # follow prompts
eval $(ssh-agent) && ssh-add ~/.ssh/id_ed25519
```

### 7. Dry-run the bootstrap

```
cd ~/workspace/zorbit/02_repos/zorbit-cli
./scripts/bootstrap-env.sh --dry-run
```

You will be prompted for:

| Prompt               | Default                        |
|----------------------|--------------------------------|
| Environment name     | `dev`                          |
| Full hostname        | `zorbit-dev.onezippy.ai`       |
| DNS configured?      | `y`                            |
| Data volume path     | `/opt/zorbit-platform/data`    |
| Git remote base URL  | `https://github.com/souravsachin` |
| Shared TPMs (y/n each)| `y`                           |
| Caffeinate?          | `n`                            |

The dry-run prints every command it would execute. Review it.

### 8. Real bootstrap

```
./scripts/bootstrap-env.sh
```

Takes 10 to 30 minutes depending on network + disk.

### 9. Install nginx config (needs sudo)

`bootstrap-env.sh` writes the nginx config to `/tmp/<hostname>.nginx.conf` and prints install instructions. Copy them into your sudo shell:

```
sudo mv /tmp/zorbit-dev.onezippy.ai.nginx.conf /etc/nginx/sites-available/zorbit-dev.onezippy.ai
sudo ln -sf /etc/nginx/sites-available/zorbit-dev.onezippy.ai /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

If no wildcard cert exists for `onezippy.ai`:

```
sudo certbot --nginx -d zorbit-dev.onezippy.ai
```

### 10. Verify

```
curl -I https://zorbit-dev.onezippy.ai/
./scripts/smoke-test.sh --env zorbit-dev    # if available
```

---

## Exit codes

| Code | Meaning                                     |
|------|---------------------------------------------|
| 0    | Success                                     |
| 1    | Preflight check failed                      |
| 2    | User cancelled at confirmation prompt       |
| 3    | Service-account guard rejected current user |
| 4    | Deploy step failed (build, compose, etc.)   |

---

## Common troubleshooting

### "This script must run as a Zorbit service account"

You are running as the wrong user. Switch: `sudo -u zorbit-deployer -i`, then re-run.

Override (at your own risk): `ZORBIT_SERVICE_USER=$(whoami) ./bootstrap-env.sh`

### "docker: permission denied"

The service account needs to be in the `docker` group. The root prep script does this; if you created the account manually: `sudo usermod -aG docker zorbit-deployer` then log out and back in.

### "Environment spec missing"

`zorbit-core` is not cloned, or is in an unexpected path. Set `REPO_ROOT_GUESS=<path>` env var.

### Ports busy

Something else is already listening on 80/443 or 3000-3005. Stop the conflicting service, or choose a different environment with a different port base (see `environments.yaml`).

### Postgres not ready

Check `docker logs <prefix>-pg`. Usually a volume permission issue — delete the volume and restart.

### Modules not announcing to registry

Kafka is probably not reachable. Check `docker logs <prefix>-kafka`. Service boot order matters: identity, authorization, module-registry must be healthy before dependent services.

---

## What `bootstrap-env.sh` does NOT do

- Does not run anything as root (by design)
- Does not push to GitHub
- Does not touch production without explicit `--allow-prod` flag
- Does not migrate data — fresh DBs only
- Does not install TLS certs — uses existing wildcard, or you run certbot
- Does not start the compose file automatically — review `/tmp/docker-compose.*.yml` first

---

## File reference

| File                                              | Purpose                                  |
|---------------------------------------------------|------------------------------------------|
| `zorbit-cli/scripts/bootstrap-env.sh`             | Main bootstrap (service account)         |
| `zorbit-cli/scripts/bootstrap-env-root.sh`        | Root-only prep (run once with sudo)      |
| `zorbit-cli/scripts/promote-env.sh`               | Tier progression (dev -> qa -> ...)      |
| `zorbit-cli/scripts/bootstrap-lib/*.sh`           | Modular helpers                          |
| `zorbit-cli/scripts/templates/Dockerfile.pm2-base`| Shared base image                        |
| `zorbit-core/platform-spec/environments.yaml`    | Env progression spec                     |
| `zorbit-core/platform-spec/all-repos.yaml`       | Repository manifest                      |
