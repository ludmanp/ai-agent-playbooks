# Verification

Run checks after every phase. Do not claim completion while a critical check fails. A check from inside the server does not prove what the internet sees: firewall results come from outside.

## From Outside (operator machine)

```bash
IP=SERVER_IP
# macOS nc: -G 4; Linux (OpenBSD nc): -w 4
for p in 22 80 443 3306 9001 8080 2375 2376 3000; do printf "%s " $p; nc -z -G 4 $IP $p && echo open || echo closed; done
for d in DOMAIN1 DOMAIN2; do printf "%s %s\n" $d "$(curl -sL -o /dev/null -m 15 -w '%{http_code} %{url_effective}' https://$d/)"; done
curl -s -o /dev/null -w '%{http_code}\n' http://DOMAIN1/                       # 301 or 308
echo | openssl s_client -connect DOMAIN1:443 -servername DOMAIN1 -tls1_1 2>&1 | grep -E 'alert|Protocol'   # alert expected
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no $IP true       # Permission denied (publickey)
ssh root@$IP true                                                                 # refused once root is disabled
```

Expected: only 22, 80, 443 open; every site 200 after redirects; TLS 1.0/1.1 refused.

## Base System

```bash
lsb_release -a; df -h /; free -h
systemctl --failed --no-pager
test -f /var/run/reboot-required && echo "reboot needed"; uptime -p
systemctl is-active unattended-upgrades
```

## SSH and fail2ban

```bash
sudo sshd -T | grep -E '^(passwordauthentication|kbdinteractiveauthentication|permitrootlogin|x11forwarding) '
sudo fail2ban-client status sshd
```

## Logging

```bash
journalctl --disk-usage
sudo cat /etc/docker/daemon.json
sudo logrotate -d /etc/logrotate.d/app-logs
```

## Docker and Traefik

```bash
sudo docker version -f '{{.Server.Version}}'
sudo docker ps --format '{{.Names}} {{.Image}} {{.Status}}'
sudo docker inspect traefik -f '{{range .Mounts}}{{.Source}} {{end}}' | grep -c docker.sock    # 0
sudo docker logs traefik --since 10m 2>&1 | grep -E 'ERR|WRN'
```

## MySQL and Networks

```bash
sudo docker port mysql8                                   # 127.0.0.1:... only
sudo docker exec -i mysql8 sh -c 'MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysql -uroot -N' <<'SQL'
select version();
select user,host from mysql.user where host='%';
SQL
for n in $(sudo docker network ls --format '{{.Name}}' | grep _db$); do
  echo "$n: $(sudo docker network inspect $n -f '{{range .Containers}}{{.Name}} {{end}}')"
done
```

Expected: no rows for `host='%'`; each `<app>_db` holds only MySQL and that app's PHP containers.

## Backups

```bash
systemctl list-timers --no-pager | grep backup-db
sudo ls -l /root/backups/mysql | tail -5
```

## Final Readiness Checklist

- External scan: only 22, 80, 443 open; Cloud Firewall (or Robot firewall) attached.
- SSH: key-only, `deploy` with sudo password, root login refused, fail2ban active.
- Unattended security updates active, no pending reboot.
- journald and Docker logs capped.
- Traefik v3 pinned, socket proxy in place, no `docker.sock` in Traefik, dashboard off or bcrypt-protected, TLS 1.2+, HTTP redirects permanent.
- MySQL 8.4 pinned, published on `127.0.0.1` only, no `'%'` users, one internal `<app>_db` network per app.
- Daily dump timer before the provider backup window; restore tested or listed as open.
- Open items stated: off-site backup copy, alerting, Docker engine update routine.
