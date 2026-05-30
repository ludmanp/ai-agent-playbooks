# Inputs

Collect these before starting. Ask incrementally; do not demand every optional value up front.

## Required

- SSH target or active server session.
- App name, lowercase slug, for paths and container names.
- Production domain.
- Git repository URL.
- Branch or tag to deploy.
- PHP version, default from project if known.
- Database source:
  - existing MySQL container/network credentials file; or
  - DB host, database, username, and password.
- `.env` strategy:
  - user-provided `.env`; or
  - generate minimal production `.env`.

## Recommended

- App path, default `/var/www/<app-name>`.
- DevOps path, default `/srv/devops/<app-name>`.
- Web root, default `public`.
- Traefik network, default `proxy`.
- MySQL network, default `mysql8_network`.
- Container name prefix, default `<app-name>`.
- Image tag, default `<app-name>/<app-name>:latest`.
- Node version, only if the project has a frontend build.
- Whether private Composer dependencies require `GITHUB_OAUTH_TOKEN`.
- Whether migrations should run during first deploy.
- Worker mode: `none`, `artisan-queue`, `supervisor`, or `pm2`.
- Scheduler mode: `none`, `artisan-schedule-work`, or `cron`.
- Custom startup commands, if any, to run as `www-data`.

## Secret Handling

- Do not ask the user to paste secrets into chat unless unavoidable.
- Prefer generating or reading secrets on the server.
- Store deployment secrets in a root-only server file such as `/root/server-secrets/<app-name>.env`.
- Store app `.env` at `/var/www/<app-name>/.env` with restrictive permissions.
- Do not `source` `.env` files that contain secrets; parse them as data.
