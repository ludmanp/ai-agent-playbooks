# Environment Strategy

Use a hybrid strategy.

## User-Provided `.env`

If the user provides a local `.env` file or approved content:

- copy it to `/var/www/<app-name>/.env`;
- set restrictive permissions;
- do not print values;
- inspect only keys, not values.

```bash
sudo chmod 600 /var/www/<app-name>/.env
```

Check required keys by printing names only:

```bash
sed -n 's/=.*//p' /var/www/<app-name>/.env | sort
```

## Generated `.env`

If no `.env` is provided, generate a minimal production `.env`.

Default database host for the prepared MySQL container:

```dotenv
DB_HOST=mysql8
DB_PORT=3306
```

Minimal shape:

```dotenv
APP_NAME=<app-name>
APP_ENV=production
APP_KEY=
APP_DEBUG=false
APP_URL=https://<domain>

LOG_CHANNEL=stack
LOG_LEVEL=warning

DB_CONNECTION=mysql
DB_HOST=mysql8
DB_PORT=3306
DB_DATABASE=<database>
DB_USERNAME=<username>
DB_PASSWORD=<password>

CACHE_STORE=file
SESSION_DRIVER=file
QUEUE_CONNECTION=database
```

After writing it, run inside the PHP container:

```bash
php artisan key:generate --force
```

Run the command as `www-data` if the `.env` file is writable by `www-data`; otherwise set `APP_KEY` before writing the file.

## Reading Server Secret Files

Never `source` secret files. Parse them as data because values may contain shell metacharacters:

```bash
awk -F= '$1 == "MYSQL_APP_PASSWORD" { sub(/^[^=]*=/, ""); print; exit }' /root/server-secrets/dashboard-credentials.env
```
