# Verification

Run verification before reporting completion.

## Containers

```bash
cd /srv/devops/<app-name>
docker compose ps
docker ps --filter name=<app-name>
```

Check logs:

```bash
docker compose logs --tail=100 web
docker compose logs --tail=100 nginx
```

If workers are enabled:

```bash
docker compose logs --tail=100 queue
docker compose logs --tail=100 scheduler
```

## Laravel

Inside web:

```bash
docker compose exec web su -s /bin/sh -c "php artisan --version" www-data
docker compose exec web su -s /bin/sh -c "php artisan about" www-data
```

Check logs without dumping secrets:

```bash
docker compose exec web sh -lc 'tail -100 storage/logs/*.log 2>/dev/null || true'
```

## Database

If migrations ran:

```bash
docker compose exec web su -s /bin/sh -c "php artisan migrate:status" www-data
```

## HTTP and HTTPS

```bash
curl -I http://<domain>
curl -I https://<domain>
```

Expected:

- HTTP redirects to HTTPS.
- HTTPS returns an application response, not Traefik 404.
- TLS certificate is valid after ACME succeeds.

## Files and Permissions

```bash
ls -ld /var/www/<app-name> /srv/devops/<app-name>
ls -l /var/www/<app-name>/.env
docker compose exec web sh -lc 'ls -ld storage bootstrap/cache'
```

## Final Checklist

- App checkout exists at `/var/www/<app-name>`.
- DevOps files exist at `/srv/devops/<app-name>`.
- `.env` exists and secrets were not printed.
- `docker compose up -d --build` completed.
- Composer install completed.
- npm build was run only if applicable.
- Laravel first-run commands completed or skipped with reason.
- Optional workers/scheduler are running if requested.
- Domain responds through Traefik over HTTPS.
