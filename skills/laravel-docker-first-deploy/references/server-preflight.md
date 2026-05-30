# Server Preflight

Before writing files, verify the target server is ready.

## SSH

```bash
whoami
hostname
sudo -n true && echo sudo-ok
```

If the user expects a non-root deploy user, verify it is in the `docker` group or can use `sudo docker`.

## Docker and Compose

```bash
docker --version
docker compose version
sudo systemctl status docker --no-pager
```

Use `sudo docker` if the SSH user is not in the Docker group.

## Networks

```bash
docker network ls
docker network inspect proxy >/dev/null
docker network inspect mysql8_network >/dev/null
```

If a required external network is missing, stop and ask whether to create it or use a different network.

## DNS and Traefik

Check the app domain resolves to the server:

```bash
dig +short app.example.com A
dig +short app.example.com AAAA
curl -I http://app.example.com
```

If DNS does not point at the server, stop before requesting certificates.

## First Deploy Safety

Check for existing app or devops directories:

```bash
test -e /var/www/<app-name> && echo app-path-exists
test -e /srv/devops/<app-name> && echo devops-path-exists
```

If either exists and is not empty, ask the user before reusing it. This skill is for first deploy, not update deploys.
