# Docker, Traefik and MySQL 8.4

Default base directory: `/srv`. Pin image versions; check the current ones when installing (`docker run --rm traefik:v3 version`, `docker run --rm mysql:8.4 mysqld --version`) and write the exact tag into Compose.

## Network Model

| Network | Members | Internal | Purpose |
|---|---|---|---|
| `proxy` | Traefik, each app's nginx | no | HTTP routing |
| `traefik_socket` | Traefik, socket proxy | yes | read-only Docker API |
| `mysql8_admin` | MySQL only | no | carries the `127.0.0.1` published port (Docker cannot publish from internal-only networks); no app joins it |
| `<app>_db` | MySQL, that app's PHP containers | yes | database for one app |
| `<app>_egress` | that app's PHP container only | no | outbound internet for composer/npm, if needed |
| `<app>_backend` | that app's nginx, PHP, helpers | yes | FastCGI and internal services |

An app's PHP container is never on a network shared with another app. php-fpm listens on 9000 without authentication: anyone who can reach it can run code as the app (`PHP_ADMIN_VALUE` + `auto_prepend_file`).

## Docker Install

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
```

Do not add users to the `docker` group (see `server-hardening.md`).

## Traefik v3

```bash
sudo docker network create proxy        # keep it if it exists
sudo mkdir -p /srv/traefik/data
sudo touch /srv/traefik/data/acme.json && sudo chmod 600 /srv/traefik/data/acme.json
```

`/srv/traefik/traefik.yml`:

```yaml
api:
  dashboard: false

entryPoints:
  http:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: https
          scheme: https     # permanent (301/308) by default in v3
  https:
    address: ":443"

providers:
  docker:
    endpoint: "tcp://socket-proxy:2375"
    exposedByDefault: false
    network: proxy
  file:
    filename: /etc/traefik/dynamic.yml

certificatesResolvers:
  http:
    acme:
      email: ACME_EMAIL
      storage: /acme.json
      httpChallenge:
        entryPoint: http

accessLog: {}
```

`/srv/traefik/dynamic.yml`:

```yaml
tls:
  options:
    default:
      minVersion: VersionTLS12
```

`/srv/traefik/docker-compose.yml`:

```yaml
services:
  socket-proxy:
    # Read-only Docker API. A docker.sock mount, even :ro, gives full API access (host root).
    image: tecnativa/docker-socket-proxy:PINNED
    container_name: traefik_socket_proxy
    restart: unless-stopped
    environment:
      CONTAINERS: 1
      NETWORKS: 1
      EVENTS: 1
      VERSION: 1
      PING: 1
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - socket
    security_opt:
      - no-new-privileges:true

  traefik:
    image: traefik:v3.X.Y
    container_name: traefik
    restart: unless-stopped
    depends_on:
      - socket-proxy
    security_opt:
      - no-new-privileges:true
    networks:
      - proxy
      - socket
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - ./traefik.yml:/etc/traefik/traefik.yml:ro
      - ./dynamic.yml:/etc/traefik/dynamic.yml:ro
      - ./data/acme.json:/acme.json

networks:
  proxy:
    name: proxy
    external: true
  socket:
    name: traefik_socket
    internal: true
```

Declare pre-existing networks `external: true`. Without it Compose refuses a network it did not create ("has incorrect label com.docker.compose.network").

Dashboard: off by default. If the user needs it, set `api.dashboard: true` and add a router to `api@internal` with a basicauth middleware using a bcrypt hash (`htpasswd -nB USER`, `$` escaped as `$$` in labels).

```bash
cd /srv/traefik && sudo docker compose config -q && sudo docker compose up -d
sudo docker logs traefik --since 1m 2>&1 | grep -E 'ERR|WRN'
```

### Upgrading an existing Traefik v2

- Site labels like ``Host(`a.lv`,`www.a.lv`)`` are v2 rule syntax and invalid in v3. Either rewrite them as ``Host(`a.lv`) || Host(`www.a.lv`)`` or set in `traefik.yml`:

  ```yaml
  core:
    defaultRuleSyntax: v2
  ```

- Keep middlewares other sites reference (for example a redirect middleware defined in Traefik's own labels); add `redirectscheme.permanent=true` to it. The global entryPoint redirect in the template above is then optional; do not remove a middleware while labels still reference it.
- Keep the certificate resolver name the site labels use (`docker inspect` labels for `certresolver=`); rename it in `traefik.yml`, not in every site.
- Keep the existing container name if scripts or docs reference it (`grep -r traefik_v2 /srv /etc/systemd`).
- `acme.json` carries over; back it up first.
- Upgrade Traefik before the Docker engine: Docker 29 removed API versions below 1.44 and Traefik older than v3.6 stops seeing containers.
- Rollback: restore the backed-up `docker-compose.yml` and `traefik.yml`, `docker compose up -d`.

## MySQL 8.4

`/srv/mysql8/.env` (mode 600):

```dotenv
MYSQL_ROOT_PASSWORD=<root-secret>
MYSQL_LOCAL_PORT=3306
```

`/srv/mysql8/docker-compose.yml`:

```yaml
services:
  mysql:
    image: mysql:8.4      # LTS line; mysql:8 floats
    container_name: mysql8
    restart: unless-stopped
    env_file:
      - .env
    command:
      - --binlog-expire-logs-seconds=259200
      - --max-binlog-size=100M
    ports:
      - "127.0.0.1:${MYSQL_LOCAL_PORT}:3306"   # host and SSH tunnels only
    volumes:
      - mysql8_data:/var/lib/mysql
    networks:
      - admin   # not internal: Docker cannot publish a port of a container that is only on internal networks
      # one entry per app, see "Adding an app"
    healthcheck:
      test: ["CMD-SHELL", "MYSQL_PWD=\"$${MYSQL_ROOT_PASSWORD}\" mysqladmin ping -h 127.0.0.1 -uroot"]
      interval: 10s
      timeout: 5s
      retries: 10

volumes:
  mysql8_data:

networks:
  admin:
    name: mysql8_admin
```

Add `--mysql-native-password=ON` only if a client cannot use `caching_sha2_password`.

On an existing server keep the published port number clients already know (for example `127.0.0.1:9001:3306`); only the bind address changes, and external clients move to an SSH tunnel to that port. An existing MySQL in another directory or with another container name is adopted as is: apply the port binding, tag pin, users and networks to its own Compose file.

Users:

- `root` only `@'localhost'` (via `docker exec`) and, if the operator uses a tunnel, `@'<gateway IP>'`. Connections through the published port arrive from a Docker network gateway, not the client IP: log in once through the tunnel, read the host in `performance_schema.accounts`, and restrict the user to it.
- No user with host `'%'`. Check: `select user,host from mysql.user where host='%';` must be empty.
- Operator access: `ssh -N -L 3307:127.0.0.1:3306 deploy@SERVER`, client to `127.0.0.1:3307`.

Pipe SQL through a heredoc into `docker exec -i mysql8 sh -c 'MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysql -uroot'`.

### Adding an app

Done by the app deployment (`laravel-docker-first-deploy`), with a fixed subnet per app:

```bash
sudo docker network create --internal --subnet 172.30.N.0/24 <app>_db
sudo docker network connect <app>_db mysql8          # live, no MySQL restart
```

Then add `<app>_db` to the MySQL Compose file as an external network and to its `networks:` list, so it survives the next recreate. Create the app user limited to that subnet:

```sql
CREATE DATABASE <app>;
CREATE USER '<app>'@'172.30.N.%' IDENTIFIED BY '<secret>';
GRANT ALL PRIVILEGES ON <app>.* TO '<app>'@'172.30.N.%';
```

The app's Compose joins `<app>_db` as external and uses `DB_HOST=mysql8`.

### Moving an app off a shared DB network

1. `docker network create --internal --subnet ... <app>_db`, add it to the MySQL Compose file.
2. In the app Compose replace the shared network with `<app>_db` (plus `<app>_egress` if the container needs internet for composer/npm).
3. Recreate MySQL (or `docker network connect`), then `RENAME USER '<app>'@'<old>' TO '<app>'@'172.30.N.%'`, then recreate the app container and restart its nginx (nginx resolved the PHP container's name at start and keeps the old IP: 502 until restarted).
4. Check from a neighbour container that the app's php-fpm is unreachable: `docker exec NEIGHBOUR sh -c 'timeout 3 bash -c "</dev/tcp/IP/9000" && echo OPEN || echo blocked'`.

### Upgrading MySQL within 8.4, or from 8.0

`mysql:8` currently resolves to the 8.4 line, so a server on the floating tag may already run 8.4.x; check `select version()`. 8.0 to 8.4 is a supported in-place upgrade on the same volume; check deprecated settings in the MySQL 8.4 release notes (for example `default_authentication_plugin` is removed) before switching.


Dump all databases first (`mysqldump --all-databases --single-transaction --routines --triggers --events`), pull the pinned tag, `docker compose up -d`, and watch the log for `Server upgrade from ... completed`. All apps lose the database for 10-30 seconds.
