# Layout

Default layout:

```text
/var/www/<app-name>/                  # Laravel app checkout
/var/www/<app-name>/.env              # app env, not committed
/srv/devops/<app-name>/               # Docker deployment files
/srv/devops/<app-name>/docker-compose.yml
/srv/devops/<app-name>/docker/app/Dockerfile
/srv/devops/<app-name>/docker/run.sh
/srv/devops/<app-name>/nginx.conf
/srv/devops/<app-name>/nginx-logs/
/root/server-secrets/<app-name>.env   # root-only operator secrets
```

The app repository should not contain generated production secrets. The devops directory may contain production Docker files but must not contain plaintext secrets unless they are root-only and intentionally excluded from Git.

## Clone

Use the requested branch or tag:

```bash
sudo mkdir -p /var/www
sudo git clone --branch <branch> <repo-url> /var/www/<app-name>
```

If the repo is private, verify SSH agent/key access before cloning.

## Ownership

For host checkout:

```bash
sudo chown -R deploy:deploy /var/www/<app-name>
```

Inside containers, Laravel writable paths should belong to `www-data`:

```bash
chown -R www-data:www-data /app/storage /app/bootstrap/cache
chmod -R ug+rwX /app/storage /app/bootstrap/cache
```

Run these inside the PHP container, not on the host, unless the host UID/GID mapping is known.
