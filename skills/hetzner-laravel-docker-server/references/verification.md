# Verification

Run verification after every phase. Do not claim completion if any critical check fails.

## Base System

```bash
lsb_release -a
df -h
free -h
sudo systemctl --failed --no-pager
```

## SSH

From the operator machine, verify the final SSH path:

```bash
ssh -p 2222 deploy@SERVER_IP 'whoami && hostname'
```

On the server:

```bash
sudo sshd -t
sudo systemctl status ssh --no-pager
```

## Firewall

```bash
sudo ufw status verbose || true
```

If using Hetzner Cloud Firewall, ask the user to confirm provider rules allow the final SSH port, `80`, and `443`.

## fail2ban

```bash
sudo systemctl status fail2ban --no-pager
sudo fail2ban-client status sshd
```

## Logging Controls

```bash
sudo journalctl --disk-usage
sudo test -f /etc/docker/daemon.json && sudo cat /etc/docker/daemon.json
sudo logrotate -d /etc/logrotate.d/app-logs
```

## Docker

```bash
docker --version
docker compose version
sudo systemctl status docker --no-pager
sudo docker network ls
```

## Traefik

```bash
sudo docker ps --filter name=traefik
sudo docker logs traefik --tail=100
curl -I http://TRAEFIK_DOMAIN
curl -Ik https://TRAEFIK_DOMAIN
```

Expected:

- HTTP redirects to HTTPS.
- HTTPS responds with a valid certificate after ACME succeeds.
- Dashboard is protected by basic auth.

## MySQL

```bash
sudo docker ps --filter name=mysql8
sudo docker inspect --format='{{json .State.Health}}' mysql8
sudo docker logs mysql8 --tail=100
sudo docker exec mysql8 mysqladmin ping -h 127.0.0.1 -uroot -p"$MYSQL_ROOT_PASSWORD"
```

Avoid printing passwords in logs or final reports.

## Final Readiness Checklist

- SSH hardened and verified on final port.
- Provider firewall or `ufw` allows only intended public ports.
- fail2ban is active for SSH.
- Unattended security updates are installed.
- journald and Docker log growth are capped.
- Docker and Docker Compose plugin are installed.
- `proxy` network exists.
- Traefik is running with Docker provider and `exposedByDefault=false`.
- Traefik dashboard domain has HTTPS and basic auth.
- MySQL 8 is running with persistent volume and no public bind.
- Disk usage is healthy.
- Server is ready for a separate Laravel deployment workflow.
