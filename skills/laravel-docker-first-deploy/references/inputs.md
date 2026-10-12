# Inputs

Collect these before starting. Ask incrementally; do not demand every optional value up front.

## Required

- SSH target (deploy user with sudo) or active server session.
- App name, lowercase slug, for paths, container names, the DB network and the DB user.
- Production domain.
- Git repository URL and a read-only deploy key if it is private.
- Branch or tag to deploy.
- PHP version, default from project if known.
- Database source:
  - new database and user on the shared `mysql8` container (default, see `database.md`); or
  - existing database credentials, whose user must be limited to the app subnet.
- `.env` strategy:
  - user-provided `.env`; or
  - generate minimal production `.env`.

## Recommended

- App path, default `/var/www/<app-name>`.
- DevOps path, default `/srv/devops/<app-name>`.
- Web root, default `public`.
- Traefik network, default `proxy`; certificate resolver name, default `http` (check what existing sites use).
- DB network subnet, a free /24 (see `database.md`).
- Whether the web container needs outbound internet (composer, npm): then `<app-name>_egress`.
- Container name prefix, default `<app-name>`.
- Image tag, default `<app-name>/<app-name>:latest`.
- Node version, only if the project has a frontend build.
- Whether private Composer dependencies require `GITHUB_OAUTH_TOKEN`.
- URL paths that carry secrets (one-time tokens, signed links) to keep out of access logs.
- Whether migrations should run during first deploy.
- Worker mode: `none`, `artisan-queue`, `supervisor`, or `pm2`.
- Scheduler mode: `none`, `artisan-schedule-work`, or `cron`.
- Custom startup commands, if any, to run as `www-data`.

## Secret Handling

- Do not ask the user to paste secrets into chat unless unavoidable.
- Prefer generating or reading secrets on the server.
- Deployment secrets live in the app's own root-only file `/root/server-secrets/<app-name>.env` (700 directory, 600 file). One file per app; do not read another app's or a shared credentials file.
- The app `.env` lives at `/var/www/<app-name>/.env`, owner `root`, group `www-data` (GID 33), mode 640: readable by the app, not writable.
- Do not `source` `.env` files that contain secrets; parse them as data.

## sudo

The deploy user's sudo asks for a password. An agent's non-interactive SSH channel cannot type it. If `sudo -n true` fails, either the user runs the phase's commands in an interactive terminal (`ssh -t deploy@HOST 'sudo ...'`), or the agent works through a root session the user still allows. Agree which before phase 2.
