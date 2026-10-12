# Inventory (read-only)

Collect facts before proposing anything. Nothing here changes the server. Commands use `sudo`; if the deploy user's sudo needs a password and the agent's channel is non-interactive, give the user one combined command to run with `ssh -t` and read the output they paste.

Set `APP=<container prefix>`, `CODE=/var/www/<app>`, `OPS=/srv/devops/<app>`, and find the MySQL container name: `MYSQL=$(sudo docker ps --format '{{.Names}} {{.Image}}' | awk '/mysql/{print $1; exit}')`.

The server itself should already meet `hetzner-laravel-docker-server` (external scan shows only 22/80/443, MySQL on `127.0.0.1`, Traefik v3 without `docker.sock`). If a quick external scan shows otherwise, do that skill first.

## Stacks Sharing the Code

Several Compose stacks can mount the same checkout (one per domain). Find them first:

```bash
for c in $(sudo docker ps --format '{{.Names}}'); do
  sudo docker inspect $c -f '{{.Name}} {{index .Config.Labels "com.docker.compose.project.working_dir"}} {{range .Mounts}}{{.Source}}:{{.Destination}} {{end}}'
done | grep "$CODE"
```

## Networks and Exposure

```bash
for c in ${APP}_web ${APP}_nginx; do
  echo "$c: $(sudo docker inspect $c -f '{{range $k,$v := .NetworkSettings.Networks}}{{$k}} {{end}}')"
done
sudo docker inspect ${APP}_web -f '{{.HostConfig.PortBindings}}'
```

A web (php-fpm) container on `proxy`/`traefik_proxy`, `mysql8_network` or any network with another app's containers is a High finding. List who else is on each of its networks:

```bash
sudo docker network inspect <network> -f '{{range .Containers}}{{.Name}} {{end}}'
```

## Database User

```bash
sudo grep -E '^DB_(HOST|DATABASE|USERNAME)=' $CODE/.env
sudo docker exec -i $MYSQL sh -c 'MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysql -uroot -t' <<'SQL'
select user,host from mysql.user order by user;
SQL
```

Check: the app has its own user; no `'%'` host; which other apps use the same user name.

## Files and Integrity

```bash
stat -c '%U:%G %a %n' $CODE $CODE/.env $CODE/storage $CODE/bootstrap/cache
test -r $CODE/.env && echo ".env readable by $(whoami)"
sudo git -C $CODE status --porcelain | head -20
sudo git -C $CODE log --oneline -1
sudo find $CODE/public $CODE/storage -name '*.php' -newer $CODE/composer.lock | head
```

`git` may refuse with "dubious ownership" for an app-owned checkout; use `sudo git -c safe.directory=$CODE -C $CODE status`.

## Production Settings (names and non-secret values only)

```bash
sudo grep -E '^(APP_ENV|APP_DEBUG|APP_URL|LOG_LEVEL|SESSION_DRIVER|SESSION_SECURE_COOKIE|SESSION_ENCRYPT|QUEUE_CONNECTION|MAIL_MAILER)=' $CODE/.env
```

## nginx

```bash
sudo cat $OPS/docker/nginx.conf 2>/dev/null || sudo docker exec ${APP}_nginx cat /etc/nginx/conf.d/default.conf
sudo docker inspect ${APP}_nginx -f '{{.Config.Image}} {{range .Mounts}}{{.Source}}:{{.Destination}}:{{.Mode}} {{end}}'
curl -sI https://<domain>/ | grep -iE '^(server|x-powered-by|strict-transport|x-frame|x-content-type|referrer-policy|content-security|set-cookie)'
```

Look for: `return 301` server blocks (redirect-only domains), `fastcgi_param APPLICATION_ENV development`, missing headers, writable mounts, a single-file config mount, secret paths in `access.log`.

## PHP Image and Processes

```bash
sudo grep -nE 'FROM|php.ini|expose_php|ENTRYPOINT|CMD|supervisor' $OPS/docker/app/Dockerfile
sudo docker exec ${APP}_web php -i | grep -E '^expose_php'
ps -eo user,pid,etime,cmd | grep -E 'supervisord|schedule:(run|work)|queue:work|horizon' | grep -v grep
sudo cat $OPS/docker/supervisor/conf.d/* 2>/dev/null | grep -E '^\[|command|user'
```

Host `ps` shows container processes too. Check: is a scheduler expected (`app/Console/Kernel.php` or `routes/console.php` has `schedule`), does exactly one run, as `www-data`? A commented-out entrypoint that was supposed to start supervisor means nothing runs.

## Backups

```bash
systemctl cat backup-db.service 2>/dev/null | grep ExecStart
sudo ls -l /root/backups/*/ 2>/dev/null | tail -5
```

## Output

One line per check: fact, evidence, and whether it is a finding (rank from SKILL.md). No changes yet.
