# Server Hardening

## Update Base System

```bash
sudo apt-get update
sudo DEBIAN_FRONTEND=noninteractive apt-get -y \
  -o Dpkg::Options::=--force-confdef -o Dpkg::Options::=--force-confold full-upgrade
sudo apt-get install -y ca-certificates curl gnupg ufw fail2ban unattended-upgrades apache2-utils dnsutils
test -f /var/run/reboot-required && echo "reboot needed"
```

On an existing server, run the upgrade in the maintenance window: it may upgrade Docker and restart every container. If `docker-ce` or `containerd.io` is in the upgrade list and Traefik is older than v3.6, hold them until Traefik is upgraded (`sudo apt-mark hold docker-ce docker-ce-cli containerd.io`, later `apt-mark unhold`); see `backups-and-updates.md`.

## Firewall

Docker writes its own iptables rules for published ports. ufw does not see them: with ufw active and only 22/80/443 allowed, a container published as `"9001:3306"` is still reachable from the internet. Use a perimeter firewall for the real policy.

### Hetzner Cloud Firewall (primary)

Cloud Console, Firewalls, Create Firewall:

- Inbound, source Any IPv4 + Any IPv6 (or trusted CIDRs for SSH): ICMP, TCP 22, TCP 80, TCP 443.
- Outbound: no rules, so all outbound traffic is allowed.
- Apply to: the server (Server or Label selector).

It filters before traffic reaches the host, so Docker cannot bypass it. On a dedicated server use the Robot firewall for the same rules.

### ufw (host ports only, optional)

If ufw is already active on an existing server, keep it: it does no harm for host ports. It just does not protect Docker ports.

```bash
sudo ufw default deny incoming
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

If there is no perimeter firewall, block Docker-published ports in the `DOCKER-USER` chain instead, for example:

```bash
ip -br link    # public interface name: eth0 on Hetzner Cloud, often enp*s0 on dedicated
sudo iptables -I DOCKER-USER -i eth0 -p tcp -m conntrack --ctorigdstport 9001 --ctdir ORIGINAL -j DROP
```

and persist it (`iptables-persistent`). Binding services to `127.0.0.1:` in Compose is still required either way.

Verify from outside the server, not only with `ufw status` (see `verification.md`).

## Non-Root Sudo User

```bash
sudo adduser --disabled-password --gecos "" deploy
sudo usermod -aG sudo deploy
sudo install -d -m 700 -o deploy -g deploy /home/deploy/.ssh
sudo install -m 600 -o deploy -g deploy /root/.ssh/authorized_keys /home/deploy/.ssh/authorized_keys
```

- The user sets the sudo password in an interactive terminal: `ssh -t HOST passwd deploy`. Commands run through a non-TTY channel (agent tools, `!` prefixes) cannot prompt for it.
- Do not add `deploy` to the `docker` group: that group is root without a password and cancels the sudo password.
- With a sudo password the agent cannot run `sudo` unattended afterwards. Do all root work first, disable root SSH last, then hand the user ready `ssh -t HOST 'sudo ...'` commands.

## SSH Hardening

Write a drop-in. sshd takes the first value it reads, and cloud-init may ship `50-cloud-init.conf` with `PasswordAuthentication yes`, so a lower number wins:

```bash
sudo tee /etc/ssh/sshd_config.d/10-hardening.conf >/dev/null <<'EOF'
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin prohibit-password
PubkeyAuthentication yes
X11Forwarding no
EOF
sudo sshd -t && sudo systemctl reload ssh
sudo sshd -T | grep -E '^(passwordauthentication|kbdinteractiveauthentication|permitrootlogin|x11forwarding) '
```

Verify in a new connection that key login works and password login is refused:

```bash
ssh deploy@SERVER_IP whoami
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no SERVER_IP true   # expect: Permission denied (publickey)
```

Last step, after the user confirmed `ssh -t deploy@SERVER_IP sudo -v` works: set `PermitRootLogin no` in the drop-in, `sshd -t`, reload, and confirm root is refused.

Moving SSH to another port (for example 2222) is optional. It cuts log noise but adds lockout risk; on an existing server keep 22 unless the user asks.

Ubuntu 24.04 names the unit `ssh.service`, not `sshd.service`.

## fail2ban

```bash
sudo tee /etc/fail2ban/jail.local >/dev/null <<'EOF'
[DEFAULT]
bantime  = 1d
findtime = 10m
maxretry = 5
ignoreip = 127.0.0.1/8 ::1 OPERATOR_IPS

[sshd]
enabled = true
backend = systemd
EOF
sudo systemctl enable --now fail2ban
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
```

Put the operator's current IP (`echo ${SSH_CLIENT%% *}` on the server) into `ignoreip`. If SSH moved to another port, add `port = 2222` to `[sshd]`.

## Logs

journald, as a drop-in:

```bash
sudo mkdir -p /etc/systemd/journald.conf.d
sudo tee /etc/systemd/journald.conf.d/10-cap.conf >/dev/null <<'EOF'
[Journal]
SystemMaxUse=500M
SystemKeepFree=1G
MaxRetentionSec=2week
EOF
sudo systemctl restart systemd-journald
```

Docker, `/etc/docker/daemon.json` (only if it does not exist; merge otherwise):

```json
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "50m", "max-file": "5" }
}
```

It needs a Docker restart and applies to containers created afterwards. On an existing server do it in the same window as a Docker upgrade or reboot.

App logs: logrotate for the real app log paths (ask, for example `/var/www/*/storage/logs/*.log`):

```text
/var/www/*/storage/logs/*.log
{
    daily
    rotate 7
    size 100M
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
    su www-data www-data
}
```

Test with `sudo logrotate -d /etc/logrotate.d/app-logs`.

## Unattended Security Updates

```bash
sudo dpkg-reconfigure -f noninteractive unattended-upgrades
systemctl is-active unattended-upgrades
```

Leave the Docker apt repo out of unattended-upgrades; see `backups-and-updates.md`.
