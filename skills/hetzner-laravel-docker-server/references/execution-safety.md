# Execution Safety

This workflow can lock users out of a server or take every site on it offline. Use these rules in all modes.

## Existing-Server Preflight

Before the first change on a server that already runs sites:

1. Provider snapshot (Hetzner Cloud: Server, Snapshots, Take snapshot). Note whether provider backups are enabled.
2. External port scan from the operator machine: `for p in 22 80 443 3306 9001 8080 2375; do nc -z -G 4 IP $p && echo "$p open"; done` (`-G` is macOS nc; `-w 4` on Linux).
3. Inventory: `docker ps`, the compose directory of each container (`docker inspect -f '{{index .Config.Labels "com.docker.compose.project.working_dir"}}' NAME`), restart policies, all Traefik labels, `docker network ls` with members.
4. MySQL: `select user,host from mysql.user`, and who connects through published ports (`performance_schema.accounts`; connections via docker-proxy show the network gateway as host).
5. Recovery path: provider console needs a root password (`passwd -S root`, `L` means locked); Hetzner Rescue can reset it.

## Change Discipline on Live Servers

- Copy a file to `FILE.bak-YYYYMMDD-HHMM` before editing it; `acme.json` copies keep mode 600.
- Validate before applying: `docker compose config -q`, `nginx -t`, `sshd -t`.
- After each change, request every site (`curl -sL -o /dev/null -w '%{http_code}' https://DOMAIN/`) and read the container logs.
- Restarting Docker, MySQL or the host affects all sites: agree the window, dump all databases first.
- Rollback for a Compose change: restore the `.bak` file and `docker compose up -d`.

## SSH and Firewall Gates

- Keep the current SSH session open until a new session on the new settings is verified.
- Test SSH config before reload (`sshd -t`), prefer `reload` over `restart`.
- When changing the SSH port, allow it in the provider firewall before removing 22.
- Disable root login only after login and `sudo -v` as the new user are confirmed by the user.

## Secrets

- Do not print secret values, including hashes (basic auth, `authentication_string`). When reading files that may contain them, mask the value in the command (`sed 's/=.*/=***/'`).
- No shell tracing (`set -x`) in commands that handle secrets.
- Generated passwords are marked as secrets for the user to store; never committed.
- `.env` files on the server have restrictive permissions and are parsed as data, never `source`d.
- Pipe SQL into `docker exec -i mysql8 sh -c 'MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysql -uroot'` through a heredoc; nested shell quoting of SQL breaks on backticks and `@`.

## Stop Conditions

Stop and ask if:

- SSH verification fails after a config change;
- DNS does not point to the server;
- Docker installation or upgrade fails;
- Traefik cannot obtain certificates or a site stops responding after a Traefik change;
- MySQL fails to start or its volume looks corrupted;
- disk usage is critically high.

## Remote Execution Pattern

Prefer small, verifiable SSH commands over one large script. Write remote files with heredocs (`sudo tee`), then show safe metadata or redacted content. Use non-interactive apt (`DEBIAN_FRONTEND=noninteractive`, `--force-confold`).
