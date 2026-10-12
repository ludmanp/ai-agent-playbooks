# Database and Network

Each app gets its own internal network to MySQL and its own MySQL user. The model is defined in `hetzner-laravel-docker-server` (`references/docker-traefik-mysql.md`, Network Model and Adding an app).

Why: php-fpm on port 9000 has no authentication, so any container on the same network can run code as the app. And a DB user shared between apps lets one compromised app read every database.

## Pick a Subnet

```bash
sudo docker network ls --format '{{.Name}}' | xargs sudo docker network inspect -f '{{.Name}} {{range .IPAM.Config}}{{.Subnet}}{{end}}'
```

Choose a /24 that overlaps no listed subnet. Convention from `hetzner-laravel-docker-server`: app DB networks take `172.30.N.0/24`, N counting up. Record it in the report. The user's host `172.30.N.%` then covers only MySQL and this app's PHP containers.

## Create the Network and Attach MySQL

```bash
sudo docker network create --internal --subnet 172.30.N.0/24 <app-name>_db
sudo docker network connect <app-name>_db mysql8      # live, no MySQL restart
```

Then make it permanent in the MySQL Compose file (back it up first), so the next MySQL recreate keeps the network:

```yaml
networks:
  <app-name>_db:
    name: <app-name>_db
    external: true

services:
  mysql:
    networks:
      - <app-name>_db
```

Run `sudo docker compose config -q` in the MySQL directory. Do not recreate MySQL now; the live `network connect` already applies.

## Create the Database and User

One root script: generates the password into the app's root-only secrets file and creates the user. The password never appears in a command line or the output. Set `app` and `subnet` first.

```bash
sudo bash -s <<'EOF'
set -euo pipefail
app=<app-name>
subnet=172.30.N.%
umask 077
install -d -m 700 /root/server-secrets
f=/root/server-secrets/$app.env
if ! grep -q '^DB_PASSWORD=' "$f" 2>/dev/null; then
  pw=$(openssl rand -base64 36 | tr -dc 'A-Za-z0-9' | cut -c1-32)
  printf 'DB_DATABASE=%s\nDB_USERNAME=%s\nDB_PASSWORD=%s\n' "$app" "$app" "$pw" >> "$f"
fi
pw=$(awk -F= '$1=="DB_PASSWORD"{sub(/^[^=]*=/,"");print;exit}' "$f")
docker exec -i mysql8 sh -c 'MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysql -uroot' <<SQL
CREATE DATABASE IF NOT EXISTS \`$app\` CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER IF NOT EXISTS '$app'@'$subnet' IDENTIFIED BY '$pw';
GRANT ALL PRIVILEGES ON \`$app\`.* TO '$app'@'$subnet';
SQL
echo "user $app@$subnet ready"
EOF
```

The generated password is alphanumeric, so it is safe inside the SQL quotes.

If the user supplied credentials for an existing database instead, check the user's host is the app subnet, not `'%'`.

## Rollback

If a later phase fails and the deploy is abandoned:

```bash
sudo docker network disconnect <app-name>_db mysql8
sudo docker network rm <app-name>_db
```

Restore the MySQL Compose backup, and drop the user and database only if the user confirms (`DROP USER '<app-name>'@'172.30.N.%'; DROP DATABASE <app-name>;`).

## Daily Dumps

Add the database to the server's dump service (`hetzner-laravel-docker-server`, `references/backups-and-updates.md`): append it to `ExecStart` in `/etc/systemd/system/backup-db.service`, `sudo systemctl daemon-reload`, run the service once and check the new file. If the server has no dump timer, report it as an open item.
