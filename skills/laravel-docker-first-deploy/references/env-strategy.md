# Environment Strategy

Use a hybrid strategy.

## User-Provided `.env`

If the user provides a local `.env` file or approved content:

- copy it without printing it: `scp .env deploy@HOST:/tmp/<app-name>.env`, then `sudo install -o 0 -g 33 -m 640 /tmp/<app-name>.env /var/www/<app-name>/.env && rm /tmp/<app-name>.env`;
- do not print values; inspect only keys;
- check the production keys listed below and report any that differ.

Check keys by printing names only:

```bash
sudo sed -n 's/=.*//p' /var/www/<app-name>/.env | sort
```

## Generated `.env`

If no `.env` is provided, generate a minimal production `.env`. Take the DB values from `/root/server-secrets/<app-name>.env` (created in `database.md`) and generate `APP_KEY` up front, so the file never needs to be writable by the app:

```dotenv
APP_NAME=<app-name>
APP_ENV=production
APP_KEY=base64:<openssl rand -base64 32>
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

SESSION_DRIVER=database
SESSION_SECURE_COOKIE=true
SESSION_ENCRYPT=true
SESSION_LIFETIME=120

CACHE_STORE=file
QUEUE_CONNECTION=database
```

Write it from a root script that reads the secrets file as data and never echoes values, for example with `awk` substitutions into a heredoc template, then:

```bash
sudo chown 0:33 /var/www/<app-name>/.env     # root owner, group www-data (GID 33 in the PHP image and on Debian/Ubuntu hosts)
sudo chmod 640 /var/www/<app-name>/.env
```

The app reads it through the group and cannot rewrite it.

`SESSION_DRIVER=database` needs a sessions table: Laravel 11+ skeletons ship it in the default migrations. If the project has none, use `SESSION_DRIVER=file` (keep `SESSION_ENCRYPT=true`) instead of adding a migration during deploy. The same applies to `QUEUE_CONNECTION=database` and the jobs table. `SESSION_ENCRYPT=true` set later logs every user out once.

## Production Keys to Check in Any `.env`

`APP_ENV=production`, `APP_DEBUG=false`, `APP_URL` with `https`, `LOG_LEVEL` warning or higher, `SESSION_SECURE_COOKIE=true`, `SESSION_ENCRYPT=true`, `DB_HOST=mysql8` (resolved by Docker DNS on `<app-name>_db`), owner `root:33`, mode 640.

## Reading Server Secret Files

Never `source` secret files. Parse them as data because values may contain shell metacharacters:

```bash
sudo awk -F= '$1 == "DB_PASSWORD" { sub(/^[^=]*=/, ""); print; exit }' /root/server-secrets/<app-name>.env
```

Only inside a script that consumes the value; never let it reach the terminal output.
