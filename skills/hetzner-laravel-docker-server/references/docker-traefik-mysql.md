# Docker, Traefik, and MySQL 8

Default base directory: `/srv`.

## Docker Install

Install Docker from the official apt repository:

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo \"$VERSION_CODENAME\") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
docker --version
docker compose version
```

Optionally allow the deploy user to run Docker:

```bash
sudo usermod -aG docker deploy
```

The user must log out and back in before group membership applies.

## Shared Docker Network

Create a shared network for Traefik-routed services:

```bash
sudo docker network create proxy
```

If it already exists, keep it.

## Traefik

Create directories:

```bash
sudo mkdir -p /srv/traefik/data
sudo touch /srv/traefik/data/acme.json
sudo chmod 600 /srv/traefik/data/acme.json
```

Generate basic auth when needed:

```bash
htpasswd -nbB USERNAME 'PASSWORD'
```

In Docker Compose labels, each `$` in the generated hash must be escaped as `$$`.

Create `/srv/traefik/traefik.yml`:

```yaml
api:
  dashboard: true

entryPoints:
  http:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: https
          scheme: https
  https:
    address: ":443"

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
    network: proxy

certificatesResolvers:
  http:
    acme:
      email: ACME_EMAIL
      storage: /acme.json
      httpChallenge:
        entryPoint: http

accessLog: {}
```

Create `/srv/traefik/docker-compose.yml`:

```yaml
services:
  traefik:
    image: traefik:v3
    container_name: traefik
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    networks:
      - proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik.yml:/etc/traefik/traefik.yml:ro
      - ./data/acme.json:/acme.json
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.traefik.rule=Host(`TRAEFIK_DOMAIN`)"
      - "traefik.http.routers.traefik.entrypoints=https"
      - "traefik.http.routers.traefik.tls=true"
      - "traefik.http.routers.traefik.tls.certresolver=http"
      - "traefik.http.routers.traefik.service=api@internal"
      - "traefik.http.routers.traefik.middlewares=traefik-auth"
      - "traefik.http.middlewares.traefik-auth.basicauth.users=USERNAME:HASH_WITH_ESCAPED_DOLLARS"

networks:
  proxy:
    external: true
```

Start:

```bash
cd /srv/traefik
sudo docker compose up -d
sudo docker logs traefik --tail=100
```

## MySQL 8

Create `/srv/mysql8/.env` with restrictive permissions:

```dotenv
MYSQL_ROOT_PASSWORD=<root-secret>
MYSQL_DATABASE=app_database
MYSQL_USER=app_user
MYSQL_PASSWORD=<app-secret>
```

```bash
sudo chmod 600 /srv/mysql8/.env
```

Create `/srv/mysql8/docker-compose.yml`:

```yaml
services:
  mysql:
    image: mysql:8.4
    container_name: mysql8
    restart: unless-stopped
    env_file:
      - .env
    command:
      - --mysql-native-password=ON
      - --binlog-expire-logs-seconds=259200
      - --max-binlog-size=100M
    volumes:
      - mysql8_data:/var/lib/mysql
    networks:
      - mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "127.0.0.1", "-uroot", "-p$${MYSQL_ROOT_PASSWORD}"]
      interval: 10s
      timeout: 5s
      retries: 10

volumes:
  mysql8_data:

networks:
  mysql:
    name: mysql8_network
```

Start:

```bash
cd /srv/mysql8
sudo docker compose up -d
sudo docker ps --filter name=mysql8
sudo docker logs mysql8 --tail=100
```

Do not publish MySQL to `0.0.0.0`. Prefer app containers on the Docker network or SSH tunnels for local database clients.

To inspect container IPs:

```bash
sudo docker network inspect mysql8_network --format '{{range .Containers}}{{.Name}}: {{.IPv4Address}}{{"\n"}}{{end}}'
```
