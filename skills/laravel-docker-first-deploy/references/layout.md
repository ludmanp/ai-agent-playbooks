# Layout

Default layout:

```text
/var/www/<app-name>/                    # Laravel app checkout, root-owned
/var/www/<app-name>/.env                # app env, www-data 600, not committed
/srv/devops/<app-name>/                 # Docker deployment files
/srv/devops/<app-name>/docker-compose.yml
/srv/devops/<app-name>/docker/app/Dockerfile
/srv/devops/<app-name>/docker/app/run.sh
/srv/devops/<app-name>/nginx/default.conf
/srv/devops/<app-name>/nginx-logs/
/root/server-secrets/<app-name>.env     # root-only operator secrets
```

The app repository holds no production secrets. The devops directory holds Docker files and no plaintext secrets.

## Clone

```bash
sudo mkdir -p /var/www
sudo git clone --branch <branch> <repo-url> /var/www/<app-name>
```

For a private repo use a read-only deploy key for root with an SSH host alias (`/root/.ssh/config`), and verify `sudo git ls-remote <repo-url>` before cloning.

## Ownership

The checkout stays `root:root`: the PHP process cannot rewrite its own code. Only Laravel's writable paths belong to `www-data`, set inside the PHP container:

```bash
chown -R www-data:www-data /app/storage /app/bootstrap/cache
chmod -R ug+rwX /app/storage /app/bootstrap/cache
```

Updates later: `sudo git pull` on the host, build steps in the container as root, Artisan as `www-data`.
