# First Deploy Workflow

Run all app build and setup commands inside containers, through `sudo docker`. Build steps (composer, npm) run as root in the container because the checkout is root-owned; Artisan runs as `www-data`.

From the devops directory:

```bash
cd /srv/devops/<app-name>
sudo docker compose config -q
sudo docker compose up -d --build
```

## Composer

```bash
sudo docker compose exec web composer install --no-dev --optimize-autoloader
```

Private dependencies: pass the token for this command only, read from the root-only secrets file, never printed:

```bash
sudo sh -c 'tok=$(awk -F= '"'"'$1=="GITHUB_OAUTH_TOKEN"{sub(/^[^=]*=/,"");print;exit}'"'"' /root/server-secrets/<app-name>.env);
  cd /srv/devops/<app-name> && COMPOSER_AUTH="{\"github-oauth\":{\"github.com\":\"$tok\"}}" docker compose exec -e COMPOSER_AUTH web composer install --no-dev --optimize-autoloader'
```

## Frontend Build

Do not assume npm exists. Only run npm when `package.json` exists and the user wants frontend assets built:

```bash
sudo docker compose exec web sh -c 'npm ci && npm run build'
```

If the project uses Yarn, pnpm or a custom build command, ask for it or infer it from `package.json` scripts.

## Permissions

`run.sh` sets them on every start. After the first install, once more:

```bash
sudo docker compose exec web sh -c 'chown -R www-data:www-data storage bootstrap/cache && chmod -R ug+rwX storage bootstrap/cache'
```

## Laravel First-Run Commands

`APP_KEY` is set in `.env` already (`env-strategy.md`). If a user-provided `.env` lacks it, generate one with `openssl rand -base64 32` and write `APP_KEY=base64:...` into the file as root; do not make `.env` writable for `key:generate`.

```bash
sudo docker compose exec -u www-data web php artisan storage:link
sudo docker compose exec -u www-data web php artisan migrate --force
sudo docker compose exec -u www-data web php artisan optimize
```

- Run migrations only when the user approves first-deploy database changes.
- If `optimize` fails due to app-specific config, run `config:cache`, `route:cache`, `view:cache` separately and report which one fails.

## Production Checks

- Debug tools must not load in production: `sudo docker compose exec -u www-data web php artisan about` and `composer show` for `laravel/telescope`, `barryvdh/laravel-debugbar`, `spatie/laravel-ignition`. Telescope in `require` (not `require-dev`) needs to be disabled or gated.
- `trustProxies(at: '*')` in `bootstrap/app.php` is safe behind the server's Traefik: Traefik replaces a client-sent `X-Forwarded-For` unless `forwardedHeaders.trustedIPs` or `insecure` is configured, and Laravel only trusts the nearest hop (nginx).
- Add the database to the server's dump timer (`database.md`, Daily Dumps).

## Project-Level Build Rule

Prefer `sudo docker compose up -d --build` over building only `web`, because nginx and worker services should be created from the same project state.

## Later Updates (for the README the user keeps)

```bash
cd /var/www/<app-name> && sudo git pull --ff-only
cd /srv/devops/<app-name>
sudo docker compose exec web composer install --no-dev -o
sudo docker compose exec web sh -c 'npm ci && npm run build'     # if applicable
sudo docker compose exec -u www-data web php artisan migrate --force
sudo docker compose exec -u www-data web php artisan optimize
sudo docker compose exec -u www-data web php artisan queue:restart   # if workers
```
