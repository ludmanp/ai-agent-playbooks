---
name: laravel-docker-first-deploy
description: Use when deploying a Laravel application to a Docker/Traefik server for the first time, and that server has no existing app checkout, Compose stack, or production .env for it. Not for routine update deploys, rollbacks, or release-symlink workflows.
---

# Laravel Docker First Deploy

## Overview

First deployment of a Laravel app onto an already-prepared Docker server. The runtime is PHP-FPM plus nginx behind Traefik, with MySQL reached over the app's own internal Docker network. Every Composer, npm, and Artisan command runs inside a container, never on the host. Docker runs through `sudo`; the deploy user is not in the `docker` group.

**REQUIRED PREREQUISITE:** the server must already have Docker, Traefik v3 and MySQL prepared by `hetzner-laravel-docker-server` (network model, `127.0.0.1`-only MySQL, dump timer). If it does not, use that skill first.

## Defaults

| Setting | Default |
|---|---|
| App code path | `/var/www/<app-name>` (root-owned checkout) |
| DevOps path | `/srv/devops/<app-name>` |
| Web root | `public` |
| Traefik network | `proxy` |
| Database network | `<app-name>_db` (internal, fixed /24 subnet, MySQL attached) |
| Outbound network | `<app-name>_egress` (web only, for composer and npm) |
| Database user | `<app-name>` limited to the `<app-name>_db` subnet |
| MySQL container | `mysql8` |
| Runtime user for Laravel commands | `www-data` |
| Worker mode | `none` |
| Scheduler mode | `none` |

Override any of these when the user or the project requires it.

## Workflow

Read the reference file at each phase. Do not work from this table alone: it names the phases, the reference files carry the commands.

| Phase | Reference | Stop condition |
|---|---|---|
| 1. Collect inputs | `references/inputs.md` | ask only for what the next phase needs |
| 2. Preflight server and app | `references/server-preflight.md` | stop when DNS, Docker, Traefik or MySQL are not ready |
| 3. Database and network | `references/database.md` | stop when the subnet is taken or MySQL cannot join the network |
| 4. Create layout | `references/layout.md` | stop when the app path exists and reuse is unconfirmed |
| 5. Prepare `.env` | `references/env-strategy.md` | stop when required keys cannot be resolved |
| 6. Generate Docker files | `references/templates.md` | - |
| 7. Workers and scheduler | `references/workers.md` | skip the phase entirely when both modes are `none` |
| 8. First deploy | `references/first-deploy-workflow.md` | stop on a failed build or install |
| 9. Verify | `references/verification.md` | stop and report the exact failed phase |

## Red Flags - Stop

- About to put a secret value into chat, a log, or a command line.
- About to request a certificate while DNS does not point at this server.
- About to run migrations without explicit approval.
- Reusing an existing app path because it looks like the right place.
- About to attach a PHP container to a network another app uses (`mysql8_network`, another app's `_db` or `_backend`).
- About to add the deploy user to the `docker` group.

Each of these means: stop and resolve it before continuing.

## Common Mistakes

| Mistake | Consequence | Instead |
|---|---|---|
| `composer`, `npm`, or `artisan` on the host | wrong PHP version and wrong file ownership | run inside the container |
| PHP container on a shared DB network | any neighbour container reaches php-fpm on 9000 (FastCGI code execution) | own `<app-name>_db` network, see `references/database.md` |
| One DB user for several apps | one compromised app reads every app's database | one user per app, limited to its subnet |
| `source /root/server-secrets/<app>.env` | shell metacharacters inside values execute | parse as data, see `references/env-strategy.md` |
| `docker compose up -d --build web` | nginx and workers keep stale project state | build the whole project |
| Assuming `npm` exists | the build fails on API-only apps | run npm only when `package.json` exists and assets are wanted |
| Running `php artisan` as root | `storage/` ends up root-owned and the app breaks later | run as `www-data` |
| `chown` on the host | host UID/GID does not match the container | chown inside the container |
| `nginx -s reload` after editing a single-file bind mount | the container keeps the old file (old inode) | `sudo docker compose up -d --force-recreate nginx` |
| Tokens in URL paths left in access logs | anyone with log access can reuse them | `map` + `access_log ... if=`, see `references/templates.md` |
| Inventing a supervisor or PM2 process list | wrong processes, silent breakage | use only a project-provided config, see `references/workers.md` |

## Report

Fill in every field. Write `skipped: <reason>` instead of dropping a field.

```text
App path:        /var/www/<app-name>
DevOps path:     /srv/devops/<app-name>
Domain:          <domain>
Containers:      <names>
Networks:        web=<list> nginx=<list>
DB:              <database>, user <app-name>@<subnet>
DB dumps:        <timer and database added, or open item>
Worker mode:     <none|artisan-queue|supervisor|pm2>
Scheduler mode:  <none|artisan-schedule-work|cron>
Completed:       <phases run>
Skipped:         <phase: reason>
Verification:    <result per check in references/verification.md>
Secret files:    <paths only, never values>
```
