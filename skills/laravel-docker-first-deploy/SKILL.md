---
name: laravel-docker-first-deploy
description: Use this skill when deploying a Laravel application to a Docker/Traefik server for the first time. It creates the initial server-side app layout, Docker Compose stack, PHP-FPM/nginx runtime, environment file, optional workers, and first-run Laravel setup. It is for first deploy only, not routine updates or rollback workflows.
---

# Laravel Docker First Deploy

## Purpose

Perform the first deployment of a Laravel app onto a prepared Docker server. The default runtime is PHP-FPM plus nginx behind Traefik, with MySQL reached over an external Docker network. All Composer, npm, and Artisan commands run inside containers, not on the host.

This skill assumes the server already has Docker, Docker Compose, Traefik, and optionally MySQL prepared. For server preparation, use a separate server setup playbook.

## Scope

This skill does:

- clone or place the first app checkout under `/var/www/<app-name>`;
- create Docker/deployment files under `/srv/devops/<app-name>`;
- create or install a production `.env` with secret-safe handling;
- generate PHP-FPM, nginx, and Docker Compose configuration;
- build and start the full Compose project with `docker compose up -d --build`;
- run first-deploy app setup inside containers;
- optionally configure workers or scheduler;
- verify HTTPS, containers, Laravel health, and logs.

This skill does not:

- implement routine update deploys;
- implement rollbacks or release symlink deployment;
- assume npm exists in every project;
- hardcode app-specific startup commands unless the user provides them;
- print secrets into chat or logs.

## Defaults

- App code path: `/var/www/<app-name>`.
- DevOps path: `/srv/devops/<app-name>`.
- Web root: `public`.
- Traefik network: `proxy`.
- MySQL network: `mysql8_network`.
- Runtime user for Laravel commands: `www-data`.
- Worker mode: `none`.
- Scheduler mode: `none`.

Allow explicit overrides when the user or project requires them.

## Workflow

1. **Collect inputs**
   - Read `references/inputs.md`.
   - Ask only for missing values needed for the next phase.
   - Treat secrets as sensitive. Do not echo them.

2. **Preflight server and app**
   - Read `references/server-preflight.md`.
   - Verify SSH, Docker, networks, DNS, app path, and whether this is really a first deploy.

3. **Create layout**
   - Read `references/layout.md`.
   - Create `/var/www/<app-name>` and `/srv/devops/<app-name>`.
   - Clone the app repo or use the existing checkout only after confirming it is expected.

4. **Prepare `.env`**
   - Read `references/env-strategy.md`.
   - Use provided `.env` when available; otherwise generate a minimal production `.env`.
   - Use server-side secret files where available.

5. **Generate Docker files**
   - Read `references/templates.md`.
   - Generate Dockerfile, nginx config, Docker Compose, and optional run script.
   - Use project-level Compose commands; do not build only one service unless debugging.

6. **Workers**
   - Read `references/workers.md` if workers or scheduler are requested.
   - Runtime commands such as `php artisan queue:restart` and `pm2 start` must run as `www-data`.

7. **First deploy**
   - Read `references/first-deploy-workflow.md`.
   - Run `docker compose up -d --build` for the whole project.
   - Run Composer, optional npm, and Artisan setup inside containers.

8. **Verify**
   - Read `references/verification.md`.
   - Stop on failed verification and report the exact failed phase.

## Reporting

Final reports should include:

- app path and devops path;
- domain and container names;
- worker/scheduler mode;
- commands completed at a high level;
- verification results;
- location of secret files without printing secret values;
- any skipped optional phase and why.
