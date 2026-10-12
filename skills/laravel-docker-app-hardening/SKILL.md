---
name: laravel-docker-app-hardening
description: Use when an already deployed Laravel app on a Docker/Traefik server needs a security review or hardening - php-fpm container on a network shared with other apps, world-readable .env, app-owned checkout, missing security headers or X-Powered-By leaks, shared or '%' DB users, scheduler or workers not running or running in several copies, redirect-only domains running full app copies. Not for a first deploy or a routine update deploy.
---

# Laravel Docker App Hardening

## Overview

Bring an existing Laravel deployment (PHP-FPM + nginx behind Traefik, MySQL in a shared container) to the standard of `laravel-docker-first-deploy` without breaking it. The target state is defined there and in `hetzner-laravel-docker-server`; this skill adds the inventory, the order of changes on a live app, and rollbacks.

**REQUIRED BACKGROUND:** `laravel-docker-first-deploy` (`references/templates.md`, `references/database.md`) for the target files, and `hetzner-laravel-docker-server` (`references/docker-traefik-mysql.md`, Network Model) for networks and DB users. If the server itself is not hardened (MySQL on `0.0.0.0`, old Traefik, root SSH), do that skill first.

## Workflow

| Phase | Reference | Output |
|---|---|---|
| 1. Inventory, read-only | `references/inventory.md` | facts per check, no changes |
| 2. Findings | this file, Findings | ranked table with evidence and proposed change |
| 3. Agree the plan | - | which changes, which window, who runs sudo |
| 4. Apply in order | `references/changes.md` | one change at a time, verified, with rollback ready |
| 5. Verify | `laravel-docker-first-deploy` `references/verification.md` | every check passes or is reported |
| 6. Record | - | devops copy in the repo synced, deploy steps updated |

Phase 1 changes nothing. Phases 3-4 need the user's approval for each change that restarts a container, logs users out, or changes how deploys run.

## Findings

Rank each finding and give the evidence line (command and output, secrets masked).

| Finding | Typical evidence | Rank |
|---|---|---|
| PHP container on a network another app uses | `docker inspect` shows `mysql8_network`, `proxy` or a neighbour's network on `<app>_web` | High |
| DB user shared between apps, between several stacks of the same code, or with host `'%'` | `mysql.user` rows, `DB_USERNAME` in each stack | High |
| `.env` readable by others | `stat` shows `644`, or the deploy user can read it | High |
| Checkout owned by `www-data` | `stat /var/www/<app>` | Medium |
| Code changed outside git | `sudo git status --porcelain` not empty, unknown PHP files in `public/` | High, stop and investigate |
| No security headers, `X-Powered-By`, nginx version | `curl -sI` | Medium |
| nginx executes any `.php` under `public/` | `location ~ \.php$` passes everything to php-fpm; a non-index PHP URL returns 200 | Medium |
| Scheduler not running, or running in several containers | `ps` on the host, entrypoint commented out, supervisor configs | Medium (functional) |
| Redirect-only domains running a full app copy | nginx `return 301` in front of a php-fpm with DB access | Medium |
| Debug settings in production | `APP_DEBUG=true`, `LOG_LEVEL=debug`, `APPLICATION_ENV development` | Medium / Low |
| nginx mounts app paths writable | volumes without `:ro`, all of `storage/app` | Low |
| Floating image tags | `nginx:latest`, `mysql:8` | Low |
| App DB missing from dump timer | `systemctl cat backup-db.service` | Medium |

## Red Flags - Stop

- `git status` in the checkout shows changes nobody can explain, or unknown PHP files in `public/` or `storage/`: possible compromise. Stop hardening, report, preserve evidence.
- About to print `.env` values or a DB password while inspecting.
- About to recreate MySQL or Traefik for an app change: use `docker network connect` and per-app recreates.
- About to switch `SESSION_ENCRYPT` or the session driver without telling the user that everyone gets logged out.
- About to enable a scheduler in more than one container.

## Common Mistakes

| Mistake | Consequence | Instead |
|---|---|---|
| Moving the web container to `<app>_db` before the DB user allows the new subnet | app loses the database | rename the user right after MySQL joins the network, then recreate web |
| Removing `egress`-like access the deploy needs | next `composer install` fails | keep `<app>_egress` for web if builds run there |
| `chown -R root:root` on the checkout without fixing writable dirs | app cannot write cache, sessions, uploads | `storage` and `bootstrap/cache` back to `www-data` in the container |
| Editing a single-file nginx bind mount and reloading | old config stays (old inode) | recreate nginx, or switch to a directory mount |
| Changing the deploy flow (root-owned checkout, sudo) without updating the README | next deploy done the old way, ownership drifts back | update the deploy steps in the same change |

## Report

```text
App:            <name>, code path, devops path, domains
Findings:       <rank: finding: evidence>
Applied:        <change: verification result>
Not applied:    <change: reason or waiting for approval>
Downtime:       <container: seconds>
Deploy steps:   <updated where>
Open items:     <list>
```
