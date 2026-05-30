# Execution Safety

This workflow can lock users out of a server if performed carelessly. Use these rules in both agent-executed and guided manual modes.

## SSH and Firewall Gates

- Keep the current SSH path working until the new SSH path has been verified.
- Before changing SSH config, confirm either a second SSH session is open or the user has console access through the provider.
- Always test SSH config before reload:

```bash
sudo sshd -t
```

- When changing SSH ports, allow the new port in the provider firewall or `ufw` before removing port `22`.
- After reload, verify a new SSH connection on the new port before disabling the old path.
- Prefer `sudo systemctl reload ssh` over restart.

## Secrets

- Do not print secret values in the final report.
- Do not use shell tracing (`set -x` or `set -o xtrace`) in commands that assign, write, or pass secrets.
- If generating passwords, generate strong random values and clearly mark them as secrets for the user to store.
- Do not commit secrets into files intended for Git.
- Use `.env` files on the server with restrictive permissions.
- Do not `source` or execute `.env` files that contain secrets; parse them as data because passwords may contain shell metacharacters.

## Stop Conditions

Stop and ask for guidance if:

- SSH verification fails after a config change;
- DNS does not point to the target server;
- Docker installation fails;
- Traefik cannot obtain certificates after DNS and port checks;
- MySQL fails to start or its volume appears corrupted;
- disk usage is critically high before setup continues.

## Remote Execution Pattern

For agent-executed setup, prefer small, verifiable SSH commands over a large opaque script. After each phase, run verification commands and inspect output before proceeding.

Use non-interactive commands where possible:

```bash
sudo apt-get update
sudo apt-get install -y package-name
```

When writing remote files, use heredocs with `sudo tee`, then immediately show safe file metadata or redacted content for verification.
