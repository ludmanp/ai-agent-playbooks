# Inputs

Collect these before setup. If a value is missing, ask only for the values needed for the next safe phase.

## Required

- Server IP or hostname.
- SSH username for initial access.
- SSH port for initial access.
- Authentication method: key path, agent, or already-open session.
- Target SSH port after hardening, usually `2222`.
- Traefik dashboard domain, for example `traefik.example.com`.
- ACME/Let's Encrypt email address.
- MySQL root password or permission to generate one.
- MySQL application database name.
- MySQL application username.
- MySQL application password or permission to generate one.

## Recommended

- Whether the server is Hetzner Cloud.
- Whether Hetzner Cloud Firewall is already configured.
- Whether to use host `ufw` in addition to provider firewall.
- Allowed SSH source IPs or CIDR ranges.
- Non-root sudo username, usually `deploy`.
- Base directory, default `/srv`.
- Traefik project directory, default `/srv/traefik`.
- MySQL project directory, default `/srv/mysql8`.
- Traefik basic auth username.
- Traefik basic auth password hash or permission to generate one with `htpasswd`.

## Preflight DNS

Before Traefik ACME setup, verify the dashboard domain resolves to the server:

```bash
dig +short traefik.example.com A
dig +short traefik.example.com AAAA
```

If the server has only IPv4, avoid an incorrect AAAA record.
