# Inputs

Collect these before setup. If a value is missing, ask only for the values needed for the next safe phase.

## Required

- Server IP or hostname.
- SSH username, port and key for initial access.
- New server or existing server with live sites.
- Whether the server is Hetzner Cloud (Cloud Firewall, snapshots, Rescue available) or dedicated (Robot firewall).
- Non-root sudo username, usually `deploy`.
- ACME/Let's Encrypt email address.
- MySQL root password or permission to generate one.

## Recommended

- Allowed SSH source IPs or CIDR ranges (Cloud Firewall rule, fail2ban `ignoreip`).
- Whether a Traefik dashboard is needed at all (default: off). If yes: its domain and a bcrypt hash from `htpasswd -nB USER`.
- Who connects to MySQL from outside today (clients, BI tools) so they can move to an SSH tunnel.
- Off-site backup target, if any (Hetzner Storage Box, S3-compatible bucket) and an encryption key owner.
- Base directory, default `/srv`; Traefik `/srv/traefik`; MySQL `/srv/mysql8`.
- App log locations for logrotate (for example `/var/www/*/storage/logs/*.log`), taken from the real layout.
- Maintenance window for Docker, MySQL and host restarts on an existing server.

## Preflight DNS

Before ACME setup, verify every domain resolves to the server:

```bash
dig +short example.com A
dig +short example.com AAAA
```

If the server has only IPv4, avoid an incorrect AAAA record.
