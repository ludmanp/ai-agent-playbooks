# First Deploy Workflow

Run all app build/setup commands inside containers.

From the devops directory:

```bash
cd /srv/devops/<app-name>
docker compose up -d --build
```

Use `sudo docker` if required by the server.

## Composer

Run Composer inside the web container:

```bash
docker compose exec web composer install --no-dev --optimize-autoloader
```

If private Composer dependencies require GitHub OAuth, pass `GITHUB_OAUTH_TOKEN` via environment or Docker Compose. Do not print token values.

## Frontend Build

Do not assume npm exists.

Only run npm when `package.json` exists and the user wants frontend assets built:

```bash
docker compose exec web npm ci
docker compose exec web npm run build
```

If the project uses Yarn, pnpm, Vite, Mix, or a custom build command, ask for the command or infer it from `package.json` scripts.

## Permissions

Run inside the web container:

```bash
docker compose exec web sh -lc 'chown -R www-data:www-data storage bootstrap/cache && chmod -R ug+rwX storage bootstrap/cache'
```

## Laravel First-Run Commands

Run Laravel runtime commands as `www-data`:

```bash
docker compose exec web su -s /bin/sh -c "php artisan key:generate --force" www-data
docker compose exec web su -s /bin/sh -c "php artisan storage:link" www-data
docker compose exec web su -s /bin/sh -c "php artisan migrate --force" www-data
docker compose exec web su -s /bin/sh -c "php artisan optimize" www-data
```

Only run `key:generate` if `APP_KEY` is missing or generated `.env` intentionally left it blank.

Only run migrations when the user approves first-deploy database changes.

If `optimize` fails due to app-specific config, capture the error and decide whether to run narrower commands:

```bash
php artisan config:cache
php artisan route:cache
php artisan view:cache
```

## Project-Level Build Rule

Prefer:

```bash
docker compose up -d --build
```

over building only `web`, because nginx and optional worker services should be created from the same project state.
