# Server Preflight

Before writing files, verify the target server is ready.

## SSH and sudo

```bash
whoami
hostname
sudo -n true && echo sudo-ok || echo "sudo needs a password, see inputs.md"
id -nG | grep -qw docker && echo "WARNING: user is in the docker group"
```

Do not add the deploy user to the `docker` group: it is root without a password. Use `sudo docker`.

## Docker, Traefik, MySQL

```bash
sudo docker version -f '{{.Server.Version}}'
sudo docker compose version
sudo docker ps --format '{{.Names}} {{.Image}}' | grep -E 'traefik|mysql'
sudo docker network inspect proxy >/dev/null && echo proxy-ok
sudo docker port mysql8          # expect 127.0.0.1 only
```

Stop and send the user to `hetzner-laravel-docker-server` if:

- there is no Traefik v3 or no `proxy` network;
- MySQL is published on `0.0.0.0` or has users with host `'%'`;
- the server's model still has all apps on one shared DB network and the user wants that kept (then confirm explicitly; this skill does not put PHP containers on it).

Note the certificate resolver name existing sites use (`sudo docker inspect <some>_nginx | grep certresolver`).

## DNS

Check the app domain resolves to the server:

```bash
dig +short app.example.com A
dig +short app.example.com AAAA
curl -I http://app.example.com
```

The A record must equal the server's public IP (`curl -s https://ifconfig.me` on the server); an AAAA record must equal its IPv6 or be absent. Otherwise stop before requesting certificates.

## First Deploy Safety

Check for existing app or devops directories and a taken network name:

```bash
test -e /var/www/<app-name> && echo app-path-exists
test -e /srv/devops/<app-name> && echo devops-path-exists
sudo docker network inspect <app-name>_db >/dev/null 2>&1 && echo db-network-exists
```

If any exists, ask the user before reusing it. This skill is for first deploy, not update deploys.
