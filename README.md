# Linux Server Lab

A personal Linux server lab running on a Debian 12 VPS.

The lab is used to practice Linux server administration, security hardening, Docker, networking, monitoring, databases, reverse proxies, and self-hosted services in a real server environment.

This repository contains sanitized configuration files and documentation for the lab. Passwords, tokens, private keys, and other secrets are not included.

## Goals

- Learn Linux server administration
- Practice VPS hardening and secure service exposure
- Learn Docker and Docker Compose
- Understand container networking
- Learn reverse proxy concepts
- Deploy and manage self-hosted services
- Work with PostgreSQL
- Monitor server resources and service availability
- Practice host-level security monitoring
- Build a reproducible Linux server environment

## Server

The lab currently runs on a Debian 12 VPS.

The base server configuration includes:

- Debian 12
- Non-root sudo user
- SSH key-only authentication
- Root login disabled
- Password authentication disabled
- UFW firewall
- Fail2ban
- Wazuh Agent
- Docker Engine
- Docker Compose

## Current Services

| Service | Purpose | Deployment | Access |
|---|---|---|---|
| **Caddy** | Reverse proxy, HTTPS and HTTP/3 | Docker | Public |
| **Gitea** | Self-hosted Git service | Docker | Caddy / localhost |
| **Homepage** | Server dashboard | Docker | SSH tunnel |
| **Uptime Kuma** | Service availability monitoring | Docker | SSH tunnel |
| **PostgreSQL** | Application database | Docker | Docker network only |
| **Netdata** | System and container monitoring | Host | SSH tunnel |
| **Wazuh Agent** | Host security monitoring | Host | Wazuh Manager |
| **Fail2ban** | Protection against repeated authentication failures | Host | Host-level |
| **UFW** | Host firewall | Host | Host-level |

## Architecture

Docker services communicate through a shared external Docker network called `lab_net`.

Caddy is the main public entry point for HTTP/HTTPS traffic. It handles TLS certificates and forwards requests to containers through Docker's internal DNS.

Administrative services are not directly exposed to the Internet. When host access is required, their ports are bound to `127.0.0.1` and accessed through SSH local port forwarding.

```text id="x3fmti"
                         Internet
                            │
                ┌───────────┴───────────┐
                │                       │
             SSH :22             HTTP/HTTPS
                │                  :80/:443
                │                       │
             sshd                    Caddy
                │                       │
            Fail2ban              Docker lab_net
                                        │
                         ┌──────────────┼──────────────┐
                         │              │              │
                       Gitea        Web Apps       Services
                                        │
                                    PostgreSQL


                    Debian 12 VPS
                         │
            ┌────────────┼─────────────┐
            │            │             │
           UFW        Netdata      Wazuh Agent
                                      │
                                      ▼
                                Wazuh Manager
```

## Network Exposure

The server follows a minimal public exposure model.

UFW uses a default-deny incoming policy.

Only the required public ports are allowed:

```text id="8b8hmm"
22/tcp     SSH
80/tcp     HTTP
443/tcp    HTTPS / HTTP/2
443/udp    HTTP/3 / QUIC
```

Application and administration ports are not directly exposed to the Internet.

Where host port mappings are required, services are bound to the loopback interface:

```text id="oy10j3"
127.0.0.1:<port>
```

PostgreSQL does not publish port `5432` on the host and is reachable only through the Docker network by containers that require database access.

## SSH

SSH is the primary administration interface for the VPS.

The SSH configuration follows several basic hardening practices:

- Public-key authentication only
- Password authentication disabled
- Root login disabled
- Non-root administrative user
- Fail2ban protection
- UFW firewall restrictions

SSH is also used to access internal web interfaces through local port forwarding.

Example:

```bash id="j3cxtm"
ssh \
  -L 3000:127.0.0.1:3000 \
  -L 3001:127.0.0.1:3001 \
  -L 3002:127.0.0.1:3002 \
  -L 19999:127.0.0.1:19999 \
  user@server
```

The services can then be accessed from the local machine:

```text id="22r45b"
http://localhost:3000    Homepage
http://localhost:3001    Uptime Kuma
http://localhost:3002    Gitea
http://localhost:19999   Netdata
```

## Docker

Docker is used to run most application-level services.

Services are managed using Docker Compose and are separated into individual directories.

A shared external Docker network is used for communication between services:

```bash id="q5jyk1"
docker network create lab_net
```

Containers connected to `lab_net` can communicate using Docker DNS.

For example:

```text id="ggn6js"
Caddy
  │
  ├── gitea:3000
  │
  └── application:3000
```

This means backend containers do not need to expose their application ports publicly.

## Reverse Proxy

Caddy acts as the main reverse proxy.

Its responsibilities include:

- HTTP and HTTPS entry point
- Automatic TLS certificates
- HTTP/2
- HTTP/3 / QUIC
- Reverse proxy routing
- Routing traffic to Docker containers

Public traffic follows approximately this path:

```text id="6z23hf"
Internet
   │
   ▼
80 / 443
   │
   ▼
Caddy
   │
   ▼
Docker network
   │
   ▼
Application container
```

Caddy can reach other containers through the shared `lab_net` network.

## PostgreSQL

PostgreSQL is used as the database for applications running in the lab.

The database is not directly accessible from the Internet.

```text id="59wbs3"
Application
     │
     ▼
 Docker lab_net
     │
     ▼
 PostgreSQL
```

Port `5432` is not published on the VPS.

Only containers that require database access are connected to the appropriate Docker network.

## Monitoring

The lab uses different tools for infrastructure, service, and security monitoring.

### Netdata

Netdata provides real-time infrastructure monitoring.

It is used to monitor:

- CPU usage
- Memory usage
- Disk usage
- Disk I/O
- Network traffic
- System load
- Processes
- Docker containers

Netdata listens only on:

```text id="d4bb7x"
127.0.0.1:19999
```

It is accessed through an SSH tunnel rather than being exposed publicly.

### Uptime Kuma

Uptime Kuma provides service-level availability monitoring.

It is used to monitor whether applications and endpoints are reachable.

It can monitor:

- HTTP/HTTPS endpoints
- Service availability
- Response times
- SSL/TLS certificate availability

The Uptime Kuma interface is available only through localhost and an SSH tunnel.

### Wazuh Agent

A Wazuh agent runs directly on the Debian host.

It provides host-level security telemetry and communicates with a Wazuh Manager.

It can provide information about:

- Security events
- System logs
- Authentication activity
- File integrity changes
- Security configuration
- Vulnerabilities
- Suspicious system activity

Wazuh is therefore used for security visibility rather than infrastructure performance monitoring.

The different monitoring layers have separate responsibilities:

```text id="2ifpfh"
Netdata       → system and resource monitoring
Uptime Kuma   → service availability monitoring
Wazuh Agent   → host security monitoring
```

## Host Security

Security controls are implemented at several levels.

### UFW

UFW provides network-level access control.

The server uses a default-deny incoming policy and explicitly allows only required public services.

Check the current configuration with:

```bash id="ih38st"
sudo ufw status verbose
```

### Fail2ban

Fail2ban monitors authentication failures and can temporarily block hosts that repeatedly fail authentication.

It is primarily used to protect SSH.

Check Fail2ban:

```bash id="gmw8at"
sudo fail2ban-client status
```

Check the SSH jail:

```bash id="pl8zbe"
sudo fail2ban-client status sshd
```

### Wazuh

Wazuh provides another security layer focused on monitoring and detection.

```text id="msj63g"
UFW
 │
 └── Controls network access

Fail2ban
 │
 └── Reacts to repeated authentication failures

Wazuh Agent
 │
 └── Monitors host security events and changes
```

Check the agent:

```bash id="xkl71u"
sudo systemctl status wazuh-agent
```

## Docker Logging

Docker currently uses the standard `json-file` logging driver.

Without rotation, container logs can grow over time and consume significant disk space.

Global log rotation can be configured in:

```text id="6vvwwb"
/etc/docker/daemon.json
```

Example configuration:

```json id="3z0h8w"
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "20m",
    "max-file": "5"
  }
}
```

This limits the amount of disk space used by container logs.

## Repository Structure

The repository contains sanitized configuration files for the lab.

```text id="0rsh90"
.
├── docker/
│   ├── caddy/
│   │   ├── compose.yml
│   │   └── Caddyfile
│   │
│   ├── gitea/
│   │   └── docker-compose.yml
│   │
│   ├── homepage/
│   │   └── docker-compose.yml
│   │
│   └── uptime-kuma/
│       └── docker-compose.yml
│
└── README.md
```

Host-level services such as UFW, Fail2ban, Netdata, and the Wazuh agent are installed directly on the Debian VPS and are not part of the Docker Compose stack.

## Basic Operations

### Start a Docker Service

```bash id="mk0vdk"
cd docker/<service>
docker compose up -d
```

### Stop a Docker Service

```bash id="r05c91"
docker compose down
```

### View Running Containers

```bash id="jq2pjz"
docker ps
```

### View Container Logs

```bash id="b6dkro"
docker logs <container>
```

Or with Docker Compose:

```bash id="7ezr3u"
docker compose logs
```

Follow logs in real time:

```bash id="6bqt1c"
docker compose logs -f
```

### Inspect the Lab Network

```bash id="24ty1h"
docker network inspect lab_net
```

### Reload Caddy

After modifying the Caddyfile:

```bash id="3k8lv9"
cd docker/caddy

docker compose exec caddy \
  caddy reload --config /etc/caddy/Caddyfile
```

## Security of This Repository

This repository contains only configurations that are safe to publish.

Sensitive information should never be committed.

This includes:

- SSH private keys
- Passwords
- Database credentials
- API tokens
- Application secrets
- Wazuh enrollment credentials
- TLS private keys
- `.env` files containing secrets
- Private backups
- Production database dumps

Configuration files committed to the repository should be sanitized before publishing.

## Planned Improvements

Future improvements for the server lab include:

- Automated PostgreSQL backups
- Automated Gitea backups
- Off-site backup storage
- Backup restore testing
- Docker log rotation
- Automatic security updates
- Netdata alert notifications
- PostgreSQL monitoring
- Caddy metrics and request monitoring
- Additional Wazuh monitoring
- Docker security monitoring
- Container health checks
- Improved secrets management
- Docker image version pinning
- Additional security hardening
- Infrastructure documentation

## Purpose

This lab is primarily a learning environment.

The goal is not only to run services, but to understand how the underlying Linux server works: networking, security, containers, databases, monitoring, logging, service management, and troubleshooting.

The VPS provides an always-available Linux environment where these concepts can be tested and documented in practice.
