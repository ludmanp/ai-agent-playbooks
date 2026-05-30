---
name: hetzner-laravel-docker-server
description: Use this skill when preparing a new Ubuntu/Hetzner server for secure Docker hosting with Traefik, MySQL 8, log controls, and readiness for later Laravel application deployment. It can execute setup over SSH when explicit access is provided, or produce guided manual commands when direct access is unavailable.
---

# Hetzner Laravel Docker Server

## Purpose

Prepare the infrastructure layer for future Laravel apps: a hardened Ubuntu server with Docker, Docker Compose, Traefik, MySQL 8, safe logging defaults, and verification checks. Do not deploy the Laravel application itself unless the user explicitly invokes a separate deployment workflow.

## Operating Modes

- **Agent-executed mode**: If the user provides SSH access and explicitly wants the agent to perform setup, connect to the server, run the phases, verify each phase, and stop on failed checks.
- **Guided manual mode**: If SSH access is unavailable or unsafe, provide commands in ordered blocks and ask the user to report outputs before continuing through risky steps.

Before acting, collect required inputs from `references/inputs.md`. For any destructive, lockout-prone, or secret-changing operation, follow `references/execution-safety.md`.

## Workflow

1. **Preflight**
   - Confirm target host, SSH user, SSH port, authentication method, Ubuntu version, domain DNS, and whether Hetzner Cloud Firewall or host firewall will be used.
   - Verify DNS A/AAAA records for Traefik dashboard point to the server before starting ACME/TLS work.
   - Confirm there is a safe SSH rollback path before hardening SSH.

2. **Server hardening**
   - Apply OS updates.
   - Create or verify a non-root sudo user when requested.
   - Harden SSH only after the new login path is verified.
   - Configure firewall assumptions, fail2ban, unattended security updates, journald caps, and logrotate.
   - See `references/server-hardening.md`.

3. **Docker, Traefik, and MySQL**
   - Install Docker from the official Docker apt repository and use the Docker Compose plugin.
   - Create the shared external Docker network for Traefik-routed services.
   - Install Traefik with Docker provider, ACME HTTP challenge, dashboard protected by basic auth, and `exposedByDefault=false`.
   - Install MySQL 8 in Docker with persistent storage and local-only/default-safe exposure.
   - See `references/docker-traefik-mysql.md`.

4. **Verification**
   - Run the phase checks immediately after each phase.
   - End with a readiness report: SSH status, firewall status, fail2ban status, Docker status, Traefik TLS/dashboard status, MySQL container health, disk usage, and log limits.
   - See `references/verification.md`.

## Boundaries

This skill prepares the server. It must not:

- create the Laravel application's Docker Compose file;
- clone or deploy the application repository;
- write an application `.env`;
- configure Laravel queues, scheduler, workers, or app containers;
- expose MySQL publicly unless the user explicitly requests it and accepts the risk.

The expected output is a server ready for a later Laravel deployment workflow using Traefik labels and the shared Docker network.

## Reporting

When finished, provide a concise report with:

- completed phases;
- commands or files changed at a high level;
- verification results;
- any skipped items and why;
- secrets created or changed, without printing secret values.
