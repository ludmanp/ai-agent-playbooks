# Templates

Generate templates into `/srv/devops/<app-name>`. Adjust versions and optional blocks from inputs. Pin image tags (`composer:2.8`, `php:8.3-fpm`, `nginx:1.27`): floating `latest` tags change without notice.

## Dockerfile

`/srv/devops/<app-name>/docker/app/Dockerfile`:

```dockerfile
FROM composer:2.8 AS composer

FROM php:8.3-fpm

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    libpng-dev \
    libwebp-dev \
    libjpeg62-turbo-dev \
    libfreetype6-dev \
    libzip-dev \
    zip unzip git curl \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

RUN docker-php-ext-configure gd --with-webp --with-freetype --with-jpeg \
    && docker-php-ext-install zip pdo_mysql exif pcntl bcmath gd

RUN cp /usr/local/etc/php/php.ini-production /usr/local/etc/php/php.ini && \
    sed -i \
      -e "s/^expose_php = .*/expose_php = Off/" \
      -e "s/max_execution_time = .*/max_execution_time = 60/" \
      -e "s/memory_limit = .*/memory_limit = 512M/" \
      -e "s/upload_max_filesize = .*/upload_max_filesize = 50M/" \
      -e "s/post_max_size = .*/post_max_size = 50M/" \
      /usr/local/etc/php/php.ini

COPY --from=composer /usr/bin/composer /usr/bin/composer
COPY run.sh /usr/local/bin/run.sh

ENTRYPOINT ["sh", "/usr/local/bin/run.sh"]
```

- `php.ini-production` ships `expose_php = On`, which adds `X-Powered-By: PHP/x.y`; the `sed` turns it off.
- Add image tools (`jpegoptim`, `optipng`), `supervisor` or build tools only when the project needs them.
- Upload limits: take them from the project; large values are a DoS surface.

## Optional Node Block

Add only when `package.json` exists and the user wants frontend assets built in the container:

```dockerfile
ARG NODE_VERSION=22.12.0
ARG NODE_PACKAGE=node-v$NODE_VERSION-linux-x64
ARG NODE_HOME=/opt/$NODE_PACKAGE

ENV NODE_PATH=$NODE_HOME/lib/node_modules
ENV PATH=$NODE_HOME/bin:$PATH

RUN curl -fsSL https://nodejs.org/dist/v$NODE_VERSION/$NODE_PACKAGE.tar.gz | tar -xzC /opt/
```

Add PM2 only for `worker_mode=pm2` (`RUN npm install -g pm2@<version>`).

## run.sh

`/srv/devops/<app-name>/docker/app/run.sh` (copied into the image). It fixes runtime permissions, then runs the given command, or php-fpm when none is given. Queue and scheduler services pass their command; without `exec "$@"` they would start php-fpm instead.

```sh
#!/bin/sh
set -eu

mkdir -p /app/storage /app/bootstrap/cache
chown -R www-data:www-data /app/storage /app/bootstrap/cache
chmod -R ug+rwX /app/storage /app/bootstrap/cache

if [ "$#" -gt 0 ]; then
    exec "$@"
fi

if [ -f /srv/devops/startup-www-data.sh ]; then
    su -s /bin/sh -c "sh /srv/devops/startup-www-data.sh" www-data
fi

exec php-fpm
```

Private Composer packages: pass the token only to the install command (`COMPOSER_AUTH`, see `first-deploy-workflow.md`); do not write it into the container's global `auth.json` on every start.

## nginx

`/srv/devops/<app-name>/nginx/default.conf`, mounted as a directory (see Compose). Use the detected web root, default `public`.

```nginx
# Paths carrying secrets (one-time tokens, signed links) are not logged.
# A `location` with `access_log off` does not work for them: try_files redirects
# internally to the PHP location, which then logs the request.
map $request_uri $app_loggable {
    ~^/<secret-path>/  0;
    default            1;
}

server {
    listen 80 default_server;
    listen [::]:80;

    root /app/public;
    index index.php;
    charset utf-8;
    server_tokens off;

    access_log /var/log/nginx/access.log combined if=$app_loggable;
    error_log /var/log/nginx/error.log;

    client_max_body_size 50M;

    # A location with its own add_header drops all of these: repeat them there.
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options DENY always;
    add_header X-Content-Type-Options nosniff always;
    add_header Referrer-Policy strict-origin-when-cross-origin always;
    add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;
    add_header Content-Security-Policy-Report-Only "default-src 'self'; frame-ancestors 'none'; object-src 'none'; base-uri 'self'; form-action 'self'" always;

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    location ~ /\.(?!well-known) { deny all; }

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass <app-name>_web:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_hide_header X-Powered-By;
        fastcgi_buffer_size 16k;
        fastcgi_buffers 4 16k;
    }
}
```

- Drop the `map` and `if=` when the app has no secret paths.
- CSP starts as Report-Only. Livewire and Alpine need inline scripts and `eval`; switch to an enforced `Content-Security-Policy` only after collecting violations from the browser console on the main screens.
- If a referer can carry a secret path (same-origin requests from that page), also mask `$http_referer` in a custom `log_format`.

## docker-compose.yml

`/srv/devops/<app-name>/docker-compose.yml`. Use project-level operations (`sudo docker compose up -d --build`).

```yaml
services:
  web:
    build:
      context: ./docker/app
    image: <app-name>/<app-name>:latest
    container_name: <app-name>_web
    restart: unless-stopped
    working_dir: /app
    volumes:
      - /var/www/<app-name>:/app
    labels:
      traefik.enable: "false"
    networks:
      - backend
      - db
      - egress        # only if composer/npm run in this container

  nginx:
    image: nginx:1.27
    container_name: <app-name>_nginx
    restart: unless-stopped
    volumes:
      - /var/www/<app-name>/public:/app/public:ro
      - ./nginx:/etc/nginx/conf.d:ro
      - ./nginx-logs:/var/log/nginx
    depends_on:
      - web
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=proxy"
      - "traefik.http.routers.<app-name>-secure.entrypoints=https"
      - "traefik.http.routers.<app-name>-secure.rule=Host(`<domain>`)"
      - "traefik.http.routers.<app-name>-secure.tls=true"
      - "traefik.http.routers.<app-name>-secure.tls.certresolver=http"
    networks:
      - proxy
      - backend

networks:
  proxy:
    external: true
    name: proxy
  backend:
    internal: true
  db:
    external: true
    name: <app-name>_db       # created in database.md, MySQL attached
  egress: {}
```

`egress` stays: every update deploy runs composer and npm in `web`. Leave it out only when the project builds elsewhere (CI image) and `web` needs no outbound access.

- The web container is on no network shared with another app. php-fpm listens on 9000 without authentication.
- `storage:link` targets live under `public/storage` -> `storage/app/public`. If public files are needed, mount `/var/www/<app-name>/storage/app/public:/app/storage/app/public:ro` into nginx, not all of `storage/app` (it holds private files).
- HTTP to HTTPS: the server's Traefik redirects on the entryPoint. If it does not (older server), add an `http` router with a `redirectscheme` middleware (`permanent=true`).
- The certificate resolver name must match the server's (`http` in `hetzner-laravel-docker-server`).
- nginx config is mounted as a directory, so `sudo docker exec <app-name>_nginx nginx -t && sudo docker exec <app-name>_nginx nginx -s reload` picks up edits. A single-file bind mount keeps the old inode after an editor or `git pull` replaces the file, and only `docker compose up -d --force-recreate nginx` would help.
