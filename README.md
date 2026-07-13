# Self-hosted Linux Server Lab

A small self-hosted Linux server lab running on a Debian 12 VPS. This repository contains safe, public configuration files for the Home Lab environment deployed using Docker.

## Goals

- Learn Linux server administration
- Practice basic VPS hardening
- Deploy services with Docker Compose
- Monitor internal services
- Document a reproducible self-hosted setup

## Current Setup

- Debian VPS
- Non-root sudo user
- SSH key-only authentication
- Root login disabled
- Password authentication disabled
- UFW firewall enabled
- Docker and Docker Compose
- Caddy as a reverse proxy for public services
- Gitea for self-hosted Git
- Uptime Kuma monitoring
- Homepage dashboard

## Home Lab Architecture

The architecture is built around a single external Docker network called `lab_net`, which connects all services. All services run in separate containers as microservices. Access to public services is provided through a reverse proxy (Caddy), which automatically obtains SSL certificates and proxies requests to the corresponding internal containers.

## Available Services and Their Purpose

| Service | Purpose | Access |
|---|---|---|
| **Caddy** | Web server and reverse proxy handling routing and SSL (HTTPS). | Public (Ports 80/443) |
| **Gitea** | Lightweight self-hosted Git service for source code management. | Public (via Caddy proxy) & SSH tunnel (port 3002) |
| **Homepage** | Start page (dashboard) with customizable widgets and quick access. | SSH tunnel (port 3000) |
| **Uptime Kuma** | Uptime monitoring for services and infrastructure. | SSH tunnel (port 3001) |

## Docker Networks and Service Interaction

All services are connected to a common external Docker network `lab_net`. This network ensures isolated interaction between containers:
- Caddy receives external requests and redirects them using internal container names (e.g., to `gitea:3000`).
- Internal services bind to `127.0.0.1` and are accessed securely via SSH local port forwarding.

## Security Baseline & Access Model

- SSH key-only login (root & password login disabled)
- UFW default deny incoming (only SSH, 80, and 443 exposed publicly)
- Internal services (Homepage, Uptime Kuma, and Gitea SSH) are bound to `127.0.0.1` and accessed via SSH tunnel:

```bash
ssh -L 3000:127.0.0.1:3000 -L 3001:127.0.0.1:3001 -L 3002:127.0.0.1:3002 user@server
```

## `docker` Folder Structure

The `docker` folder contains safe versions of the configurations (without passwords and secrets):

```
docker/
├── caddy/
│   ├── compose.yml          # Docker Compose for Caddy
│   └── Caddyfile            # Caddy routing rules (sanitized)
├── gitea/
│   └── docker-compose.yml   # Gitea configuration
├── homepage/
│   └── docker-compose.yml   # Homepage configuration
└── uptime-kuma/
    └── docker-compose.yml   # Uptime Kuma configuration
```

## Basic Launch Instructions

1. **Creating the network:**
   ```bash
   docker network create lab_net
   ```
2. **Launching a service:**
   ```bash
   cd docker/<service_name>
   docker compose up -d
   ```
3. **Reloading Caddy (after changing the Caddyfile):**
   ```bash
   cd docker/caddy
   docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
   ```

## ⚠️ Warnings Regarding `.env` and Secrets

- The files presented here are **templates** and contain placeholder values (e.g., `${DOMAIN}`). 
- **NEVER** store real passwords, API keys, certificates, exact IP addresses, or private domains in public repositories.
- To pass secrets into Docker Compose, always use `.env` files (which must be added to `.gitignore`) or the Docker Secrets mechanism.