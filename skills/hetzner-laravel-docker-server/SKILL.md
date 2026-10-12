---
name: hetzner-laravel-docker-server
description: Use when preparing a new Ubuntu/Hetzner server for Docker hosting of Laravel apps with Traefik and MySQL 8, or when hardening or auditing an existing one that already runs sites (exposed MySQL port, ufw not blocking Docker ports, old Traefik, docker.sock in Traefik, shared database network, root SSH, no DB backups, pending reboot). Executes over SSH when access is given, otherwise produces guided manual commands.
---

# Hetzner Laravel Docker Server

## Purpose

Prepare or harden the infrastructure layer for Laravel apps: Ubuntu with key-only SSH for a non-root sudo user, Hetzner Cloud Firewall, Docker, Traefik v3 behind a read-only Docker socket proxy, MySQL 8.4 reachable only from per-app networks and SSH tunnels, database dumps, capped logs and an update routine. Do not deploy the Laravel application itself; that is `laravel-docker-first-deploy`.

## Operating Modes

- **New server, agent-executed**: SSH access given and the user wants the agent to run it. Run the phases, verify each, stop on failed checks.
- **Existing server, agent-executed**: the server already runs sites. Every change can take all of them down. Run the existing-server preflight first, change one component at a time, keep a backup of each file before editing it, verify every site after each step, and agree a maintenance window for restarts of Docker, MySQL or the host.
- **Guided manual mode**: no SSH or unsafe to use. Give ordered command blocks and wait for reported output before risky steps.

Collect inputs from `references/inputs.md`. For lockout-prone, destructive or secret-changing steps follow `references/execution-safety.md`.

## Workflow

1. **Preflight** (`references/execution-safety.md`)
   - Host, SSH user and key, Ubuntu version, DNS, whether it is Hetzner Cloud (`curl -s http://169.254.169.254/hetzner/v1/metadata`).
   - Existing server: provider snapshot, external port scan, inventory of containers, compose dirs, Traefik labels, restart policies, MySQL users and who connects to published ports.
   - Recovery path before touching SSH (provider console needs a root password; Hetzner Rescue can reset it).

2. **Server hardening** (`references/server-hardening.md`)
   - Hetzner Cloud Firewall as the primary perimeter; ufw only for host ports, because Docker-published ports bypass it.
   - Non-root `deploy` user with sudo password (set by the user in an interactive terminal), not in the `docker` group; key-only SSH via a drop-in; fail2ban. Root keeps key login until step 6.
   - Unattended security updates, journald and Docker log caps.

3. **Docker, Traefik, MySQL** (`references/docker-traefik-mysql.md`)
   - Docker from the official repo, pinned image versions.
   - Traefik v3 behind `docker-socket-proxy`, TLS 1.2+, permanent HTTP to HTTPS redirect, dashboard off.
   - MySQL 8.4 bound to `127.0.0.1`, no `'%'` users, one internal network per app (`<app>_db`).

4. **Backups and updates** (`references/backups-and-updates.md`)
   - Daily `mysqldump` per app database before the provider disk backup.
   - Reboot after kernel or libc updates; Docker engine updated by hand after a compatibility check.

5. **Verification** (`references/verification.md`)
   - Check after each phase, from inside and from outside the server.

6. **Close root** (`references/server-hardening.md`, SSH Hardening)
   - The agent does all root work in steps 1-5 while root key login still works. Then the user confirms `ssh -t deploy@HOST sudo -v`, the agent sets `PermitRootLogin no` and confirms root is refused.
   - From here every privileged command needs the user's sudo password: hand the user ready `ssh -t deploy@HOST 'sudo ...'` commands. End with the readiness report.

## Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Relying on ufw for Docker ports | `ports: "9001:3306"` stays open to the internet with ufw active | Hetzner Cloud Firewall, and bind to `127.0.0.1:` |
| One shared DB network for all apps | Any compromised neighbour container reaches other apps' php-fpm on 9000 (FastCGI RCE) and their DB users | `<app>_db` internal network per app, MySQL joins each |
| `docker.sock:ro` in Traefik | `ro` does not limit the Docker API: Traefik RCE equals host root | `tecnativa/docker-socket-proxy`, GET-only endpoints |
| Floating tags `mysql:8`, `traefik:v3`, `nginx:latest` | Silent major jumps or years-old images | Pin minor versions, update deliberately |
| Upgrading Docker before Traefik | Docker 29 removed old API versions; Traefik older than v3.6 loses all routes | Upgrade Traefik first, then Docker |
| `PasswordAuthentication no` in `sshd_config` | A cloud-init drop-in with `yes` wins (first match) | Own drop-in `10-hardening.conf`, check `sshd -T` |
| Disabling root SSH before testing deploy + sudo | Lockout | Test login and `sudo -v` in a separate terminal first |
| `apt upgrade` without reboot | Kernel fixes not active; seen at 327 days uptime | Check `/var/run/reboot-required` |
| Only a provider disk backup | Crash-consistent image of a running MySQL | Daily `mysqldump --single-transaction` before it |

## Boundaries

This skill prepares the server. It must not:

- create the Laravel application's Compose file, clone the app or write its `.env`;
- configure queues, scheduler or app containers;
- publish MySQL on a public interface.

Creating an app's `<app>_db` network and attaching MySQL to it belongs to the app deployment, following the model in `references/docker-traefik-mysql.md`.

## Reporting

Report completed phases, files changed (high level), verification results with the external port scan, skipped items and why, open items (for example no off-site backup copy), and secrets created or changed without their values.
