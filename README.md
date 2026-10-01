# Self-hosted Linux Server Lab

A small self-hosted Linux server lab running on a Debian 12 VPS. This repository contains safe, public configuration files for the Home Lab environment deployed using Docker.

## Goals

- Learn Linux server administration
- Practice VPS hardening and secure service exposure
- Deploy and manage services with Docker Compose
- Monitor server resources and service availability
- Learn reverse proxy and container networking concepts
- Document a reproducible self-hosted setup

## Current Setup

- Debian 12 VPS
- Non-root sudo user
- SSH key-only authentication
- Root login disabled
- Password authentication disabled
- UFW firewall with default-deny incoming policy
- Fail2ban for SSH protection
- Docker and Docker Compose
- Caddy reverse proxy with automatic HTTPS
- HTTP/3 support via UDP/443
- Gitea for self-hosted Git
- Uptime Kuma for availability monitoring
- Netdata for system and container monitoring
- Homepage dashboard
- PostgreSQL for application data

## Home Lab Architecture

The lab is built around Docker containers connected through a common external Docker network called `lab_net`.

Caddy acts as the main public entry point and reverse proxy. It handles HTTPS certificates and forwards incoming requests to the corresponding containers using Docker's internal DNS.

Most administrative services are not directly exposed to the Internet. Their ports are bound to `127.0.0.1` and accessed through SSH local port forwarding.

```text
                    Internet
                       │
             ┌─────────┴─────────┐
             │                   │
          SSH :22          Caddy :80/:443
             │                   │
        Fail2ban            HTTPS / HTTP3
                                 │
                            Docker lab_net
                                 │
                    ┌────────────┼────────────┐
                    │            │            │
                  Gitea      Web Apps      Services
                                 │
                             PostgreSQL

        Localhost-only services
        ├── Homepage     :3000
        ├── Uptime Kuma  :3001
        ├── Gitea        :3002
        ├── Web App      :3003
        └── Netdata      :19999
```

## Available Services and Their Purpose

| Service | Purpose | Access |
|---|---|---|
| **Caddy** | Reverse proxy, automatic HTTPS, and HTTP/3 support | Public (80/tcp, 443/tcp, 443/udp) |
| **Gitea** | Lightweight self-hosted Git service | Public via Caddy / localhost management port |
| **Homepage** | Internal dashboard with shortcuts and service widgets | SSH tunnel (port 3000) |
| **Uptime Kuma** | Service availability and uptime monitoring | SSH tunnel (port 3001) |
| **Netdata** | Real-time system, resource, and container monitoring | SSH tunnel (port 19999) |
| **PostgreSQL** | Application database | Internal Docker network only |

## Docker Networks and Service Interaction

Services communicate through the external Docker network `lab_net`.

This allows containers to reach each other using their container names without exposing application ports directly to the Internet.

For example:

```text
Internet
   │
   ▼
Caddy
   │
   ├── gitea:3000
   └── application:3000
```

PostgreSQL is accessible only from containers that require database access and does not publish port `5432` on the host.

Administrative services such as Homepage, Uptime Kuma, and Netdata are accessible only through localhost and SSH port forwarding.

## Security Baseline & Access Model

The VPS uses a minimal public attack surface.

SSH is configured with:

- Public-key authentication only
- Password authentication disabled
- Root login disabled
- Fail2ban monitoring SSH authentication failures

UFW uses a default-deny incoming policy. Only the required public ports are allowed:

```text
22/tcp     SSH
80/tcp     HTTP
443/tcp    HTTPS / HTTP/2
443/udp    HTTP/3 (QUIC)
```

Internal service ports are bound to `127.0.0.1` whenever host access is required.

For example:

```bash
ssh \
  -L 3000:127.0.0.1:3000 \
  -L 3001:127.0.0.1:3001 \
  -L 3002:127.0.0.1:3002 \
  -L 19999:127.0.0.1:19999 \
  user@server
```

After establishing the tunnel, services can be accessed locally:

```text
http://localhost:3000    Homepage
http://localhost:3001    Uptime Kuma
http://localhost:3002    Gitea
http://localhost:19999   Netdata
```

## Monitoring

The lab uses two complementary monitoring tools.

### Netdata

Netdata provides real-time infrastructure monitoring, including:

- CPU usage
- Memory usage
- Disk usage and I/O
- Network traffic
- System load
- Processes
- Docker/container resource usage

Netdata listens only on:

```text
127.0.0.1:19999
```

and is therefore accessed through an SSH tunnel rather than being exposed publicly.

### Uptime Kuma

Uptime Kuma provides service-level availability monitoring.

It is used to check whether public applications and endpoints are reachable and can also monitor HTTPS/SSL certificate availability.

This separates infrastructure monitoring (Netdata) from service availability monitoring (Uptime Kuma).

## Docker Logging

Docker currently uses the standard `json-file` logging driver.

Container log sizes are monitored to prevent uncontrolled disk usage. Log rotation can be configured globally through `/etc/docker/daemon.json`.

Example:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "20m",
    "max-file": "5"
  }
}
```

## `docker` Folder Structure

The `docker` folder contains sanitized versions of service configurations without passwords, tokens, private keys, or other secrets:

```text
docker/
├── caddy/
│   ├── compose.yml
│   └── Caddyfile
├── gitea/
│   └── docker-compose.yml
├── homepage/
│   └── docker-compose.yml
└── uptime-kuma/
    └── docker-compose.yml
```

Netdata and host-level security services such as UFW and Fail2ban run directly on the VPS and are therefore not part of the Docker Compose stack.

## Basic Launch Instructions

### 1. Create the Docker network

```bash
docker network create lab_net
```

### 2. Start a service

```bash
cd docker/<service_name>
docker compose up -d
```

### 3. Reload Caddy

After changing the Caddyfile:

```bash
cd docker/caddy
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

### 4. Check running containers

```bash
docker ps
```

## Planned Improvements

Future improvements for the lab include:

- Automated PostgreSQL backups
- Gitea backups
- Off-site backup storage
- Docker log rotation
- Automatic security updates
- Netdata alert notifications
- PostgreSQL metrics in Netdata
- Caddy metrics and request monitoring
- Pinning Docker images to specific versions instead of relying on `latest`
