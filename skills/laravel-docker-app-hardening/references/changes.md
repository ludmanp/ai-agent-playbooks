# Changes on a Live App

Apply one change at a time, cheapest and safest first. Before each: copy the file to `FILE.bak-YYYYMMDD-HHMM`. After each: request the site (`curl -sL -o /dev/null -w '%{http_code}' https://<domain>/`), read the container logs, and check the app's log for new errors. Before the first change, dump the app database.

Variables as in `inventory.md`: `APP`, `CODE`, `OPS`, `MYSQL` (the MySQL container name, `mysql8` or whatever the server uses).

## Before Any Change

1. Integrity: `sudo git status --porcelain` in the checkout is empty or explained (`inventory.md`). Otherwise stop.
2. Dump the app database:

   ```bash
   sudo sh -c 'umask 077; mkdir -p /root/backups/pre-hardening; docker exec $0 sh -c "MYSQL_PWD=\"\$MYSQL_ROOT_PASSWORD\" exec mysqldump -uroot --single-transaction --routines --triggers <database>" | gzip > /root/backups/pre-hardening/<database>-$(date -u +%F).sql.gz' "$MYSQL"
   ```

3. Order: 1 and 2 first (no downtime). If redirect-only stacks exist, do 6 next, so steps 3, 4, 7 and 8 touch one stack instead of several. Then 3, 4, 7, 5, 8, 9, 10.
4. Approval: every step except 1, 2 and 10's record keeping needs the user's yes (restart, logout, deploy-flow change, or a scheduler that starts running tasks).

## Running sudo Steps Through the User

When the deploy user's sudo needs a password and the agent's channel is non-interactive, send each step as one block the user runs in a terminal, ending with its own check, and ask for the output:

```bash
ssh -t deploy@HOST 'set -e
sudo cp -p /srv/devops/<app>/docker-compose.yml /srv/devops/<app>/docker-compose.yml.bak-$(date +%Y%m%d-%H%M)
# ... the step's commands ...
curl -s -o /dev/null -w "%{http_code}\n" https://<domain>/'
```

One block per step keeps the rollback point clear.

## 1. `.env` Permissions (no downtime)

```bash
sudo docker exec ${APP}_web id www-data          # GID 33 on Debian images; 82 on Alpine
sudo chown 0:33 $CODE/.env && sudo chmod 640 $CODE/.env
```

The app reads it through the group and cannot rewrite it. With cached config the change is invisible until the next `artisan config:cache`, so run one now as `www-data` to prove it still reads. Rollback: previous owner and mode from the inventory.

## 2. Debug Leftovers (no downtime)

- `LOG_LEVEL=debug` to `warning` in `.env`, then `sudo docker exec -u www-data ${APP}_web php artisan config:cache`.
- Remove `fastcgi_param APPLICATION_ENV development` and similar from nginx (applied with step 3).

## 3. nginx Headers and Mounts (seconds of downtime per nginx)

Take the server block from `laravel-docker-first-deploy` `references/templates.md` (headers, `server_tokens off`, `fastcgi_hide_header X-Powered-By`, dotfile deny, `map` for secret paths) and merge it into the existing config, keeping app-specific locations and redirects. Also:

- pin the image (`nginx:1.27` instead of `latest`);
- mounts read-only (`:ro`), and only `storage/app/public` instead of all of `storage/app`;
- prefer a directory mount for the config.
- run PHP only for `/index.php` (`location = /index.php`) and return 404 for any other `.php`: vendored plugins often ship sample PHP files under `public/`, and an upload bug would otherwise become code execution. Check first that the app has no other PHP entry points (`sudo find $CODE/public -name '*.php'`).

```bash
cd $OPS && sudo docker compose config -q
sudo docker compose up -d --force-recreate nginx
sudo docker exec ${APP}_nginx nginx -t
curl -sI https://<domain>/ | grep -iE '^(server|x-powered-by|strict-transport|x-frame|content-security)'
```

A site that sends its own `X-Powered-By` from PHP (some CMSs do, for example TypiCMS) needs `fastcgi_hide_header`; `expose_php=Off` only removes PHP's own header. CSP goes in as Report-Only. Rollback: restore `.bak` files, recreate nginx. Make `.bak` copies with `cp -p` before editing; with a single-file mount the recreate is what applies the edit.

## 4. `expose_php` (seconds of downtime)

Add `-e "s/^expose_php = .*/expose_php = Off/"` to the Dockerfile's `php.ini` edits, then:

```bash
sudo docker tag <image>:latest <image>:prev
cd $OPS && sudo docker compose build web && sudo docker compose up -d web
sudo docker exec ${APP}_web php -i | grep '^expose_php'
```

Rollback: `sudo docker tag <image>:prev <image>:latest && sudo docker compose up -d web`.

## 5. Root-Owned Checkout (no downtime, changes the deploy flow)

First find what the app writes outside `storage` and `bootstrap/cache` (uploads into `public/`, generated assets):

```bash
sudo find $CODE -path $CODE/storage -prune -o -path $CODE/bootstrap/cache -prune -o -path $CODE/vendor -prune -o -path $CODE/node_modules -prune -o -user www-data -newer $CODE/composer.lock -print | head
```

Then find files and directories the app can read today only because it owns them. After `chown root:root` they lock the app out (a `bootstrap` left at `700` by an old `chmod -R 700` gives HTTP 500 on every request):

```bash
sudo find $CODE -path $CODE/storage -prune -o -path $CODE/bootstrap/cache -prune -o -path $CODE/.git -prune -o -path $CODE/.env -prune -o \( -type d ! -perm -o=rx -o -type f ! -perm -o=r \) -print | head
```

Open each hit to `755` (directories) or `644` (files) as part of the change. Keep those paths `www-data`, chown the rest, and roll back automatically if the site stops answering:

```bash
sudo chown -R root:root $CODE
sudo chown 0:33 $CODE/.env && sudo chmod 640 $CODE/.env      # chown -R reset its group
sudo docker exec ${APP}_web sh -c 'chown -R www-data:www-data storage bootstrap/cache'
code=$(curl -s -o /dev/null -w '%{http_code}' https://<domain>/); echo $code
[ "$code" = 200 ] || { sudo chown -R www-data:www-data $CODE; sudo chown 0:33 $CODE/.env; echo ROLLED BACK; }
sudo git -C $CODE status --porcelain | head      # git now works for root without safe.directory
```

The scan only finds paths written since the last composer install; ask the user which features upload or generate files and test one upload after the chown. Update the deploy steps in the same change: `sudo git pull`, composer and npm as root in the container, Artisan as `www-data`. Rollback: `sudo chown -R www-data:www-data $CODE`.

## 6. Redirect-Only Domains (removes containers)

When extra stacks exist only to answer `return 301` for alias domains, let Traefik redirect and remove them. Labels on the main app's nginx (v2 rule syntax shown, valid with `core.defaultRuleSyntax: v2`; in v3 syntax use `Host(\`a\`) || Host(\`b\`)`):

```yaml
      - "traefik.http.routers.${APP}-alias.entrypoints=https"
      - "traefik.http.routers.${APP}-alias.rule=Host(`alias1.lv`,`www.alias1.lv`,`alias2.lv`,`www.alias2.lv`)"
      - "traefik.http.routers.${APP}-alias.tls.certresolver=http"
      - "traefik.http.routers.${APP}-alias.middlewares=${APP}-alias-redirect"
      - "traefik.http.middlewares.${APP}-alias-redirect.redirectregex.regex=^https?://[^/]+/(.*)"
      - "traefik.http.middlewares.${APP}-alias-redirect.redirectregex.replacement=https://main.lv/$${1}"
      - "traefik.http.middlewares.${APP}-alias-redirect.redirectregex.permanent=true"
```

HTTP requests for the aliases go through the entryPoint redirect to HTTPS first. Check it: `curl -sI http://alias1.lv/ | head -3` must show 301/308 to `https://alias1.lv/`. If it does not, add the same router on the `http` entryPoint. Use the rule syntax the other site labels on this server use (v2 `Host(`a`,`b`)` needs `core.defaultRuleSyntax: v2`). A router needs a service: it inherits the container's service, and the middleware answers before the request reaches it.

```bash
cd $OPS && sudo docker compose up -d nginx
for u in http://alias1.lv/x https://www.alias1.lv/x https://alias2.lv/a?b=1; do curl -s -o /dev/null -w "$u %{http_code} %{redirect_url}\n" $u; done
# then, per redirect stack:
cd /srv/devops/<alias-stack> && sudo docker compose down
sudo mv /srv/devops/<alias-stack> /srv/devops/<alias-stack>.disabled-YYYYMMDD
```

Run the `down` right after the new labels are verified: for those seconds the old and new routers match the same hosts, and both redirect to the same target, so the overlap is harmless. Certificates for the alias names are already in `acme.json` and are reused. Rollback: `docker compose up -d` in the old directory, remove the labels.

## 7. Own DB Network and User (seconds of downtime)

Follow `hetzner-laravel-docker-server` `references/docker-traefik-mysql.md`, "Moving an app off a shared DB network". Order:

1. `sudo docker network create --internal --subnet 172.30.N.0/24 ${APP}_db` and `sudo docker network connect ${APP}_db $MYSQL`; add it to the MySQL Compose file as external.
2. DB user:
   - user used only by this app (several stacks of the same code count as one app once step 6 removed the extras): `RENAME USER '<user>'@'<old host>' TO '<user>'@'172.30.N.%'` for one row, `DROP USER` the other container-IP rows; keep a `@'localhost'` row only if something on the MySQL host uses it;
   - user shared with another app: create a new user for this app (script in `laravel-docker-first-deploy` `references/database.md`), grant only its database, put the new credentials into `.env`, `config:cache`. Revoke the old user's access to this database once nothing uses it.
3. In the app Compose: web drops `mysql8_network` and `proxy`/`traefik_proxy`, joins `${APP}_db` and, if builds run there, `${APP}_egress`. Workers and scheduler join `backend` and `${APP}_db`.
4. `sudo docker compose up -d web` (and workers), then `sudo docker restart ${APP}_nginx` in the same step: nginx resolves `fastcgi_pass <app>_web:9000` once at start, and if it resolved the name through a network web just left (often `proxy`), every request now fails with 502 "No route to host". Then verify the DB connection (`php artisan db:show` as `www-data`) and that php-fpm is unreachable from a neighbour (`laravel-docker-first-deploy` `references/verification.md`).

Rollback: restore the app Compose `.bak`, `RENAME USER` back, `docker compose up -d web`.

If several stacks mount the same code and use the same DB user, do step 6 first, so only one stack moves.

## 8. Scheduler and Workers (functional)

- Check whether the app defines scheduled tasks (`routes/console.php`, `app/Console/Kernel.php`) and queued jobs (`QUEUE_CONNECTION` other than `sync`).
- Run each in exactly one container, as `www-data`: a `scheduler` service with `php artisan schedule:work` (`laravel-docker-first-deploy` `references/workers.md`), not a supervisor loop running `schedule:run` as root in every copy.
- Turning on a scheduler that has been off for months runs every due task from now on. Read the task list with the user first (`php artisan schedule:list`). Empty list: no scheduler needed, remove the dead supervisor config instead.
- With several stacks of the same code, the scheduler goes into the stack that stays after step 6.
- The scheduler container needs `${APP}_db` and, when tasks send mail or call APIs, `${APP}_egress`. Running it with `user: www-data` and the web image (`image: <web image name>`, check `docker inspect ${APP}_web -f '{{.Config.Image}}'`) avoids a rebuild and root-owned cache files.
- Root-owned files in `storage/framework/cache` or `bootstrap/cache` (from earlier root `artisan` runs) break `schedule:list` and some requests with "Permission denied": `chown -R www-data:www-data storage bootstrap/cache` inside the container first.

## 9. Sessions

`SESSION_SECURE_COOKIE=true`, `SESSION_ENCRYPT=true` in `.env`, `config:cache`. Enabling encryption invalidates existing sessions: everyone logs in again. Tell the user and pick a quiet time.

## 10. Backups and Records

- Add the app database to the dump service (`hetzner-laravel-docker-server` `references/backups-and-updates.md`).
- Copy the changed devops files into the app repository if it keeps one, and update its deploy steps (sudo, root-owned checkout, nginx recreate rule).
- Remove `.bak` files and `.disabled` stacks only after the user confirms a few days without issues.
