---
name: laravel-docker-first-deploy
description: Use when deploying a Laravel application to a Docker/Traefik server for the first time, and that server has no existing app checkout, Compose stack, or production .env for it. Not for routine update deploys, rollbacks, or release-symlink workflows.
---

# Laravel Docker First Deploy

## Overview

First deployment of a Laravel app onto an already-prepared Docker server. The runtime is PHP-FPM plus nginx behind Traefik, with MySQL reached over an external Docker network. Every Composer, npm, and Artisan command runs inside a container, never on the host.

**REQUIRED PREREQUISITE:** the server must already have Docker, Docker Compose, Traefik, and optionally MySQL. If it does not, use `hetzner-laravel-docker-server` first.

## Defaults

| Setting | Default |
|---|---|
| App code path | `/var/www/<app-name>` |
| DevOps path | `/srv/devops/<app-name>` |
| Web root | `public` |
| Traefik network | `proxy` |
| MySQL network | `mysql8_network` |
| Runtime user for Laravel commands | `www-data` |
| Worker mode | `none` |
| Scheduler mode | `none` |

Override any of these when the user or the project requires it.

## Workflow

Read the reference file at each phase. Do not work from this table alone: it names the phases, the reference files carry the commands.

| Phase | Reference | Stop condition |
|---|---|---|
| 1. Collect inputs | `references/inputs.md` | ask only for what the next phase needs |
| 2. Preflight server and app | `references/server-preflight.md` | stop when DNS, Docker, or networks are not ready |
| 3. Create layout | `references/layout.md` | stop when the app path exists and reuse is unconfirmed |
| 4. Prepare `.env` | `references/env-strategy.md` | stop when required keys cannot be resolved |
| 5. Generate Docker files | `references/templates.md` | — |
| 6. Workers and scheduler | `references/workers.md` | skip the phase entirely when both modes are `none` |
| 7. First deploy | `references/first-deploy-workflow.md` | stop on a failed build or install |
| 8. Verify | `references/verification.md` | stop and report the exact failed phase |

## Red Flags — Stop

- About to put a secret value into chat, a log, or a command line.
- About to request a certificate while DNS does not point at this server.
- About to run migrations without explicit approval.
- Reusing an existing app path because it looks like the right place.

Each of these means: stop and resolve it before continuing.

## Common Mistakes

| Mistake | Consequence | Instead |
|---|---|---|
| `composer`, `npm`, or `artisan` on the host | wrong PHP version and wrong file ownership | run inside the container |
| `source /root/server-secrets/<app>.env` | shell metacharacters inside values execute | parse as data, see `references/env-strategy.md` |
| `docker compose up -d --build web` | nginx and workers keep stale project state | build the whole project |
| Assuming `npm` exists | the build fails on API-only apps | run npm only when `package.json` exists and assets are wanted |
| Running `php artisan` as root | `storage/` ends up root-owned and the app breaks later | run as `www-data` |
| `chown` on the host | host UID/GID does not match the container | chown inside the container |
| Inventing a supervisor or PM2 process list | wrong processes, silent breakage | use only a project-provided config, see `references/workers.md` |

## Report

Fill in every field. Write `skipped: <reason>` instead of dropping a field.

```text
App path:        /var/www/<app-name>
DevOps path:     /srv/devops/<app-name>
Domain:          <domain>
Containers:      <names>
Worker mode:     <none|artisan-queue|supervisor|pm2>
Scheduler mode:  <none|artisan-schedule-work|cron>
Completed:       <phases run>
Skipped:         <phase: reason>
Verification:    <result per check in references/verification.md>
Secret files:    <paths only, never values>
```
