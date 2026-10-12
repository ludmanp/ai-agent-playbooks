# Workers

Workers are optional. Default: no workers and no scheduler.

All Laravel runtime commands run as `www-data` unless the project explicitly uses another runtime user. Worker services use the same image as `web`; `run.sh` fixes permissions and then `exec`s the service's `command` (see `templates.md`). They join `db`, plus `<app-name>_egress` when their jobs send mail or call external APIs (SMTP, Telegram, payment providers), and `backend` only if they talk to services there. Never a network shared with another app.

## Modes

### none

No worker services or startup commands.

### artisan-queue

```yaml
  queue:
    image: <app-name>/<app-name>:latest
    container_name: <app-name>_queue
    restart: unless-stopped
    working_dir: /app
    volumes:
      - /var/www/<app-name>:/app
    command: ["su", "-s", "/bin/sh", "-c", "php artisan queue:work --sleep=3 --tries=3 --timeout=90", "www-data"]
    depends_on:
      - web
    networks:
      - backend
      - db
```

After a deploy, `php artisan queue:restart` (as `www-data`) makes workers load the new code.

### supervisor

Use only when the project already has or the user provides a supervisor config. Do not invent complex supervisor process lists without project context. Install `supervisor` in the image only for this mode, and run it as the service `command`.

### pm2

Use only when the project has a PM2 process file, commonly `pm2-worker.yml`.

Requirements:

- Node and `pm2` installed in the image (see `templates.md`).
- `/var/www/.pm2` exists and belongs to `www-data`.
- PM2 commands run as `www-data`.

Startup commands in `/srv/devops/<app-name>/startup-www-data.sh`, mounted into `web` read-only:

```yaml
    volumes:
      - /var/www/<app-name>:/app
      - ./startup-www-data.sh:/srv/devops/startup-www-data.sh:ro
```

```bash
mkdir -p /var/www/.pm2
pm2 start /app/pm2-worker.yml
php /app/artisan queue:restart
```

`run.sh` runs this script as `www-data` before php-fpm.

## Scheduler

Default: no scheduler.

For `artisan-schedule-work`:

```yaml
  scheduler:
    image: <app-name>/<app-name>:latest
    container_name: <app-name>_scheduler
    restart: unless-stopped
    working_dir: /app
    volumes:
      - /var/www/<app-name>:/app
    command: ["su", "-s", "/bin/sh", "-c", "php artisan schedule:work", "www-data"]
    depends_on:
      - web
    networks:
      - db
      - egress     # if scheduled tasks send mail or call APIs
```

Before enabling a scheduler on an app that already has data, read `php artisan schedule:list` and each task's selection logic: a task that acts on everything "due" can fire a backlog at once on its first run. Run risky ones once by hand first.
