# Server Hardening

## Update Base System

```bash
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install -y ca-certificates curl gnupg lsb-release ufw fail2ban unattended-upgrades apache2-utils dnsutils
```

## Optional Non-Root Sudo User

Create a deploy user only if requested or missing:

```bash
sudo adduser deploy
sudo usermod -aG sudo deploy
sudo mkdir -p /home/deploy/.ssh
sudo cp /root/.ssh/authorized_keys /home/deploy/.ssh/authorized_keys
sudo chown -R deploy:deploy /home/deploy/.ssh
sudo chmod 700 /home/deploy/.ssh
sudo chmod 600 /home/deploy/.ssh/authorized_keys
```

Verify login as that user before continuing.

## SSH Hardening

Edit `/etc/ssh/sshd_config` only after following the safety gates:

```text
Port 2222
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin prohibit-password
ChallengeResponseAuthentication no
UsePAM yes
```

Apply safely:

```bash
sudo sshd -t
sudo systemctl reload ssh
```

Verify new SSH access in a separate connection:

```bash
ssh -p 2222 deploy@SERVER_IP
```

## Firewall

If using Hetzner Cloud Firewall, make sure it allows:

- new SSH port from trusted source IPs;
- TCP `80` from anywhere for HTTP challenge and redirects;
- TCP `443` from anywhere for HTTPS.

If using `ufw`:

```bash
sudo ufw allow 2222/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status verbose
```

Do not remove access to port `22` until the new SSH port is verified.

## fail2ban

Create `/etc/fail2ban/jail.local`:

```ini
[DEFAULT]
bantime  = 1d
findtime = 10m
maxretry = 3

[sshd]
enabled = true
port    = 2222
```

Apply:

```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

## systemd Journal Cap

Create or edit `/etc/systemd/journald.conf`:

```ini
[Journal]
SystemMaxUse=500M
SystemKeepFree=1G
MaxRetentionSec=2week
```

Apply:

```bash
sudo systemctl restart systemd-journald
sudo journalctl --vacuum-size=500M
sudo journalctl --disk-usage
```

## Docker Log Rotation

Create `/etc/docker/daemon.json` after Docker is installed:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "5"
  }
}
```

Restart Docker after creating this file:

```bash
sudo systemctl restart docker
```

Existing containers must be recreated to pick up the setting.

## App Logrotate Baseline

Create `/etc/logrotate.d/app-logs`:

```text
/srv/apps/*/storage/logs/*.log
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
    create 0664 www-data www-data
}
```

Test:

```bash
sudo logrotate -d /etc/logrotate.d/app-logs
```

## Unattended Security Updates

```bash
sudo apt-get install -y unattended-upgrades
sudo dpkg-reconfigure -f noninteractive unattended-upgrades
sudo systemctl status unattended-upgrades --no-pager
```
