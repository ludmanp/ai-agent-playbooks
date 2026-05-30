# Workers

Workers are optional. Default: no workers and no scheduler.

All Laravel runtime commands should run as `www-data` unless the project explicitly uses another runtime user.

## Modes

### none

No worker services or startup commands.

### artisan-queue

Add a separate service using the same app image:

```yaml
  queue:
    image: <app-name>/<app-name>:latest
    container_name: <app-name>_queue
    restart: unless-stopped
    working_dir: /app
    volumes:
      - /var/www/<app-name>:/app
      - /srv/devops/<app-name>:/srv/devops:ro
    command: >
      sh -lc 'su -s /bin/sh -c "php artisan queue:work --sleep=3 --tries=3 --timeout=90" www-data'
    depends_on:
      - web
    networks:
      - backend
      - database
```

### supervisor

Use only when the project already has or the user provides a supervisor config. Do not invent complex supervisor process lists without project context.

Run supervisord in a dedicated service or as the web entrypoint only when requested.

### pm2

Use only when the project has a PM2 process file, commonly `pm2-worker.yml`.

Requirements:

- Node installed in the PHP image.
- `pm2` installed globally.
- `/var/www/.pm2` exists and belongs to `www-data`.
- PM2 commands run as `www-data`.

Example startup commands in `/srv/devops/<app-name>/startup-www-data.sh`:

```bash
mkdir -p /var/www/.pm2
pm2 start /app/pm2-worker.yml
php /app/artisan queue:restart
```

The parent `run.sh` must execute this script as `www-data`.

## Scheduler

Default: no scheduler.

For `artisan-schedule-work`, add a separate service:

```yaml
  scheduler:
    image: <app-name>/<app-name>:latest
    container_name: <app-name>_scheduler
    restart: unless-stopped
    working_dir: /app
    volumes:
      - /var/www/<app-name>:/app
      - /srv/devops/<app-name>:/srv/devops:ro
    command: >
      sh -lc 'su -s /bin/sh -c "php artisan schedule:work" www-data'
    depends_on:
      - web
    networks:
      - backend
      - database
```
