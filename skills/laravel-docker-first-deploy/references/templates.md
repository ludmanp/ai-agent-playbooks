# Templates

Generate templates into `/srv/devops/<app-name>`. Adjust versions and optional blocks from inputs.

## Dockerfile

Baseline PHP-FPM image:

```dockerfile
FROM composer:latest AS composer

FROM php:8.3-fpm

WORKDIR /app

RUN apt-get update && apt-get install -y \
    build-essential \
    libpng-dev \
    libwebp-dev \
    libjpeg62-turbo-dev \
    libfreetype6-dev \
    locales \
    zip \
    libzip-dev \
    jpegoptim optipng pngquant gifsicle \
    vim \
    unzip \
    git \
    curl \
    supervisor \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

RUN docker-php-ext-install zip pdo_mysql exif pcntl bcmath
RUN docker-php-ext-configure gd --with-webp=/usr/include --with-freetype=/usr/include/ --with-jpeg=/usr/include/
RUN docker-php-ext-install gd

RUN cp /usr/local/etc/php/php.ini-production /usr/local/etc/php/php.ini && \
    sed -i "s/max_execution_time = .*/max_execution_time = 60/" /usr/local/etc/php/php.ini && \
    sed -i "s/memory_limit = .*/memory_limit = 1024M/" /usr/local/etc/php/php.ini && \
    sed -i "s/upload_max_filesize = .*/upload_max_filesize = 500M/" /usr/local/etc/php/php.ini && \
    sed -i "s/post_max_size = .*/post_max_size = 500M/" /usr/local/etc/php/php.ini

COPY --from=composer /usr/bin/composer /usr/bin/composer

EXPOSE 9000
ENTRYPOINT ["sh", "/srv/devops/run.sh"]
```

Only add Node/npm/PM2 if the project requires it.

## Optional Node Block

Add only when `package.json` exists and the user wants frontend assets built in the container:

```dockerfile
ARG NODE_VERSION=24.8.0
ARG NODE_PACKAGE=node-v$NODE_VERSION-linux-x64
ARG NODE_HOME=/opt/$NODE_PACKAGE

ENV NODE_PATH=$NODE_HOME/lib/node_modules
ENV PATH=$NODE_HOME/bin:$PATH

RUN curl https://nodejs.org/dist/v$NODE_VERSION/$NODE_PACKAGE.tar.gz | tar -xzC /opt/
RUN npm install -g npm@latest
```

Add PM2 only for `worker_mode=pm2`:

```dockerfile
RUN npm install pm2@latest -g
```

## run.sh

`run.sh` should perform runtime permissions and optional startup commands, then foreground `php-fpm`.

```bash
#!/bin/sh
set -eu

if [ -n "${GITHUB_OAUTH_TOKEN:-}" ]; then
    composer config --global --auth github-oauth.github.com "$GITHUB_OAUTH_TOKEN"
fi

mkdir -p /app/storage /app/bootstrap/cache
chown -R www-data:www-data /app/storage /app/bootstrap/cache
chmod -R ug+rwX /app/storage /app/bootstrap/cache

if [ -f /srv/devops/startup-www-data.sh ]; then
    su -s /bin/sh -c "sh /srv/devops/startup-www-data.sh" www-data
fi

php-fpm
```

## nginx.conf

Use the detected web root, default `public`.

```nginx
server {
    listen 80 default_server;
    listen [::]:80;

    root /app/public;
    index index.php;
    charset utf-8;

    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    sendfile off;
    client_max_body_size 500M;

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass <app-name>_web:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_intercept_errors off;
        fastcgi_buffer_size 16k;
        fastcgi_buffers 4 16k;
        fastcgi_param PHP_VALUE "max_execution_time=60 \n upload_max_filesize=500M \n post_max_size=500M";
    }
}
```

## docker-compose.yml

Use project-level Compose operations such as `docker compose up -d --build`.

```yaml
services:
  web:
    build:
      context: ./docker/app
      dockerfile: Dockerfile
    image: <app-name>/<app-name>:latest
    container_name: <app-name>_web
    restart: unless-stopped
    working_dir: /app
    volumes:
      - /var/www/<app-name>:/app
      - /srv/devops/<app-name>:/srv/devops:ro
    labels:
      traefik.enable: "false"
    networks:
      - backend
      - database

  nginx:
    image: nginx:latest
    container_name: <app-name>_nginx
    restart: unless-stopped
    volumes:
      - /var/www/<app-name>/public:/app/public:ro
      - /var/www/<app-name>/storage/app:/app/storage/app:ro
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - ./nginx-logs:/var/log/nginx
    depends_on:
      - web
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=proxy"
      - "traefik.http.routers.<app-name>.entrypoints=http"
      - "traefik.http.routers.<app-name>.rule=Host(`<domain>`)"
      - "traefik.http.routers.<app-name>.middlewares=<app-name>-https-redirect"
      - "traefik.http.middlewares.<app-name>-https-redirect.redirectscheme.scheme=https"
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
    driver: bridge
    internal: true
  database:
    external: true
    name: mysql8_network
```
