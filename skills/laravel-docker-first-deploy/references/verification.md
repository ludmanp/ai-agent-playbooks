# Verification

Run verification before reporting completion.

## Containers and Networks

```bash
cd /srv/devops/<app-name>
sudo docker compose ps
sudo docker compose logs --tail=100 web nginx
for c in web nginx queue scheduler; do
  n=<app-name>_$c; sudo docker inspect $n >/dev/null 2>&1 || continue
  echo "$n: $(sudo docker inspect $n -f '{{range $k,$v := .NetworkSettings.Networks}}{{$k}} {{end}}')"
done
sudo docker network inspect <app-name>_db -f '{{range .Containers}}{{.Name}} {{end}}'
```

Expected: web on `backend`, `<app-name>_db` and optionally `<app-name>_egress`; nginx on `proxy` and `backend`; `<app-name>_db` holds only MySQL and this app's PHP containers.

php-fpm unreachable from another app's container (pick any neighbour with a shell):

```bash
ip=$(sudo docker inspect <app-name>_web -f '{{range .NetworkSettings.Networks}}{{.IPAddress}} {{end}}')
for a in $ip; do sudo docker exec <neighbour>_web sh -c "timeout 3 bash -c '</dev/tcp/$a/9000' 2>/dev/null && echo OPEN $a || echo blocked $a"; done
```

## Laravel

```bash
sudo docker compose exec -u www-data web php artisan --version
sudo docker compose exec -u www-data web php artisan about --only=environment
sudo docker compose exec web sh -c 'tail -50 storage/logs/*.log 2>/dev/null || true'
```

`about` must show production environment and debug off.

## Database

```bash
sudo docker compose exec -u www-data web php artisan db:show
sudo docker exec -i mysql8 sh -c 'MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysql -uroot -N' <<'SQL'
select user,host from mysql.user where user='<app-name>';
SQL
```

If migrations ran: `php artisan migrate:status`. The user's host must be the app subnet, not `%`.

## HTTP and Headers

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://<domain>/        # 301 or 308
curl -sI https://<domain>/ | grep -iE '^(HTTP|strict-transport|x-frame|x-content-type|referrer-policy|permissions-policy|content-security|server|x-powered-by)'
```

Expected: application response (not Traefik 404), HSTS, X-Frame-Options, nosniff, Referrer-Policy, Permissions-Policy, CSP (Report-Only) present; `server: nginx` without a version; no `x-powered-by`. If secret paths are configured, request one and check it is absent from `nginx-logs/access.log`.

## Files and Permissions

```bash
sudo ls -ld /var/www/<app-name> /srv/devops/<app-name>      # root:root
sudo stat -c '%u:%g %a %n' /var/www/<app-name>/.env          # 0:33 640
sudo stat -c '%a %n' /root/server-secrets/<app-name>.env     # 600
sudo docker compose exec web sh -c 'ls -ld storage bootstrap/cache'
```

## Backups

```bash
systemctl cat backup-db.service | grep ExecStart      # includes <app-name>
```

## Final Checklist

- Checkout at `/var/www/<app-name>`, root-owned; devops files at `/srv/devops/<app-name>`.
- `.env` root:www-data 640, production keys checked, secrets not printed.
- Own `<app-name>_db` network with MySQL attached live and in the MySQL Compose file; DB user limited to its subnet.
- Web container on no network shared with another app; php-fpm unreachable from a neighbour.
- `sudo docker compose up -d --build` completed; Composer install completed; npm only if applicable.
- Laravel first-run commands completed or skipped with reason; no debug tools active.
- Workers and scheduler run their own commands (not php-fpm) if requested.
- HTTPS through Traefik with security headers; no PHP version leak.
- Database added to the dump timer, or reported as open.
