# Backups and Updates

## Database Dumps

A provider disk backup (Hetzner Cloud Backups, 7 daily images) is a crash-consistent copy of a running MySQL. Add a logical dump per app database and schedule it before the provider backup window, so every disk image carries a consistent dump.

`/srv/backups/backup-db.sh` (mode 700, root):

```bash
#!/bin/bash
# Daily dumps of the listed databases, 14 days kept.
set -euo pipefail
dir=/root/backups/mysql
keep_days=14
umask 077
mkdir -p "$dir"
for db in "$@"; do
  out="$dir/$db-$(date -u +%F).sql.gz"
  tmp="$out.tmp"
  docker exec mysql8 sh -c 'MYSQL_PWD="$MYSQL_ROOT_PASSWORD" exec mysqldump -uroot --single-transaction --routines --triggers --no-tablespaces "$0"' "$db" | gzip > "$tmp"
  gzip -t "$tmp"
  zcat "$tmp" | tail -1 | grep -q "Dump completed"
  mv "$tmp" "$out"
  echo "$(date -u +%FT%TZ) ok $out $(stat -c %s "$out") bytes"
done
find "$dir" -name "*.sql.gz" -mtime +"$keep_days" -delete
```

`/etc/systemd/system/backup-db.service`:

```ini
[Unit]
Description=Dump MySQL databases
After=docker.service

[Service]
Type=oneshot
ExecStart=/srv/backups/backup-db.sh app1 app2
```

`/etc/systemd/system/backup-db.timer` (provider backup at 23:15 UTC in this example):

```ini
[Unit]
Description=Daily MySQL dumps before the provider disk backup

[Timer]
OnCalendar=*-*-* 22:45:00 UTC
Persistent=true

[Install]
WantedBy=timers.target
```

When an app is added, append its database to `ExecStart` and `systemctl daemon-reload`. The script lives in `/srv/backups` with the other server config; dumps go to `/root/backups` (700), which only root can read.

```bash
sudo systemctl daemon-reload && sudo systemctl enable --now backup-db.timer
sudo systemctl start backup-db.service && journalctl -u backup-db.service -n 5 -o cat
```

- Find the provider backup time in the console (Hetzner Cloud: Server, Backups, the timestamps of existing images).
- Restore test once: load a dump into a scratch database and count rows of a few tables.
- Off-site copy (another provider or a Storage Box, encrypted with `age` and a key the user keeps) protects against losing the provider account; it is optional, report it as an open item if skipped.
- No alerting comes with this. Report it as an open item unless the user has monitoring.

## OS Updates

- `unattended-upgrades` applies Ubuntu security updates daily but does not reboot. Kernel and libc fixes need a reboot: check `/var/run/reboot-required` and `uptime`, reboot in a window, then `apt-get autoremove --purge` for old kernels (Ubuntu keeps the running and the previous kernel, so a boot fallback remains).
- After the reboot: all containers up (`docker ps`), `systemctl --failed` empty, every site answers, timers listed.

## Docker Engine Updates

Keep the Docker apt repo out of unattended-upgrades: an engine upgrade restarts every container, and major versions break clients (Docker 29 removed API versions below 1.44, which stopped Traefik older than v3.6).

Monthly, by hand:

1. `apt list --upgradable 2>/dev/null | grep -E 'docker|containerd'` and read the release notes of the new major.
2. Check Traefik and other Docker API clients are compatible.
3. Dump all databases, then in the window: `sudo apt-get install --only-upgrade docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`.
4. Verify all containers and sites.

## Image Updates

Pinned tags do not update themselves. Monthly: check new patch releases of Traefik, MySQL 8.4, nginx and app base images, update the tag, `docker compose pull && docker compose up -d`, verify.
