# Service Descriptor: hx-dev-server (AI/ML Development Server)

**Service Name**: hx-dev-server
**Node**: hx-dev-server.hx.dev.local
**IP Address**: 192.168.10.222
**Port**: 9443 (Portainer)
**Protocol**: HTTPS, Docker API
**Wildcard Routing**: Via hx-ssl-server:443 (*.dev.hx.dev.local)
**Status**: OPERATIONAL
**Operational Date**: 2025-12-06

---

## Table of Contents

1. [Overview](#overview)
2. [Service Architecture](#service-architecture)
3. [Docker Platform](#docker-platform)
4. [Portainer Management](#portainer-management)
5. [Developer Templates](#developer-templates)
6. [Wildcard Routing](#wildcard-routing)
7. [AI Service Integration](#ai-service-integration)
8. [Domain Integration](#domain-integration)
9. [PKI & Certificate Trust](#pki--certificate-trust)
10. [Administration](#administration)
11. [Monitoring & Health](#monitoring--health)
12. [Troubleshooting](#troubleshooting)
13. [Related Documentation](#related-documentation)

---

## Overview

### Description

Docker-based AI/ML development server providing isolated containerized development environments for the HX-Infrastructure ecosystem. Features AD-authenticated access, Portainer web UI for container management, pre-built development templates, and seamless integration with all HX AI services.

### Primary Value Proposition

- **Isolated Development**: Complete isolation from production systems for safe experimentation
- **Containerized Workflows**: Standardized Docker containers with pre-configured templates
- **AD Authentication**: Domain user SSH and Portainer access via SSSD/Kerberos
- **AI Service Connectivity**: Direct access to LiteLLM, Qdrant, PostgreSQL, Redis, FastMCP
- **Wildcard Subdomain Routing**: Developer services accessible via `*.dev.hx.dev.local`
- **Web-Based Management**: Portainer CE for intuitive container lifecycle management

### Key Capabilities

| Capability | Description |
|------------|-------------|
| Docker Engine | Container runtime (v29.1.2) |
| Docker Compose | Multi-container orchestration (v5.0.0) |
| Portainer CE | Web-based container management UI |
| Python Template | Pre-built Python 3.11 development container |
| Node.js Template | Pre-built Node.js 20 development container |
| Wildcard SSL | TLS-terminated routing via hx-ssl-server |
| AD Integration | Domain authentication for all access |

---

## Service Architecture

### Deployment Model

- **Type**: Bare-metal Ubuntu 24.04 with Docker Engine
- **Container Runtime**: Docker Engine 29.1.2
- **Compose**: Docker Compose v5.0.0 (plugin)
- **Management**: Portainer CE 2.33.5

### Component Stack

```
┌─────────────────────────────────────────────────────────────┐
│                    Developer Access                          │
│     (SSH, Portainer UI, *.dev.hx.dev.local URLs)            │
└─────────────────────────┬───────────────────────────────────┘
                          │
         ┌────────────────┼────────────────┐
         ▼                ▼                ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────────────────┐
│     SSH      │ │  Portainer   │ │      hx-ssl-server       │
│   Port 22    │ │  Port 9443   │ │  *.dev.hx.dev.local:443  │
│  AD Auth     │ │  HTTPS UI    │ │   TLS Termination        │
└──────────────┘ └──────────────┘ └─────────────┬────────────┘
                                                │
                          ┌─────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────┐
│              hx-dev-server (192.168.10.222)                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                Docker Engine 29.1.2                  │   │
│  │  ┌─────────────┬──────────────┬─────────────────┐   │   │
│  │  │ Development │  Portainer   │    Template     │   │   │
│  │  │ Containers  │  Container   │   Containers    │   │   │
│  │  └─────────────┴──────────────┴─────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              /mnt/docker-data (835 GB)              │   │
│  │  ┌─────────────┬──────────────┬─────────────────┐   │   │
│  │  │   Images    │   Volumes    │   Portainer     │   │   │
│  │  │   Storage   │   Storage    │   Data          │   │   │
│  │  └─────────────┴──────────────┴─────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────┬───────────────────────────────────┘
                          │
         ┌────────────────┼────────────────┬─────────────────┐
         ▼                ▼                ▼                 ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│   LiteLLM    │ │    Qdrant    │ │  PostgreSQL  │ │    Redis     │
│   :4000      │ │    :6333     │ │    :5432     │ │    :6379     │
│  LLM Proxy   │ │Vector Store  │ │   Database   │ │    Cache     │
└──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘
```

### Directory Structure

```
/mnt/docker-data/
├── docker/                     # Docker data root
│   ├── containers/            # Container data
│   ├── images/                # Image layers
│   ├── volumes/               # Named volumes
│   └── overlay2/              # Storage driver data
└── portainer/
    └── data/                  # Portainer persistent data

/opt/dev-templates/
├── python/                    # Python development template
│   ├── Dockerfile
│   ├── .dockerignore
│   ├── entrypoint.sh
│   ├── requirements.txt
│   └── README.md
├── nodejs/                    # Node.js development template
│   ├── Dockerfile
│   ├── .dockerignore
│   ├── entrypoint.sh
│   ├── package.json
│   └── README.md
└── common/
    ├── .env.template          # Environment variable template
    ├── docker-compose.yml     # Compose template
    ├── DEVELOPER-GUIDE.md     # Development guide
    └── TEMPLATE-INDEX.md      # Template catalog

/etc/docker/
└── daemon.json                # Docker daemon configuration

/usr/local/share/ca-certificates/
└── hx-ca.crt                  # HX-Infrastructure CA certificate
```

---

## Docker Platform

### Docker Engine Configuration

| Setting | Value |
|---------|-------|
| Version | 29.1.2 |
| Data Root | /mnt/docker-data/docker |
| Storage Driver | overlay2 |
| Log Driver | json-file |
| Log Max Size | 100m |
| Log Max Files | 3 |
| Live Restore | true |
| Userland Proxy | false |

### daemon.json

```json
{
  "data-root": "/mnt/docker-data/docker",
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  },
  "live-restore": true,
  "userland-proxy": false
}
```

### Docker Compose

| Setting | Value |
|---------|-------|
| Version | 5.0.0 (plugin) |
| Command | `docker compose` |
| Default File | docker-compose.yml |

### Storage Allocation

| Partition | Mount Point | Size | Purpose |
|-----------|-------------|------|---------|
| /dev/sda1 | /mnt/docker-data | 894 GB | Docker images, volumes, containers |
| Available | - | 835 GB | Free space for development |

---

## Portainer Management

### Access Information

| Property | Value |
|----------|-------|
| URL | https://hx-dev-server.hx.dev.local:9443 |
| Alternative | https://192.168.10.222:9443 |
| Version | Portainer CE 2.33.5 |
| Admin User | admin |
| Credentials | Ansible Vault |

### Container Configuration

```bash
docker run -d \
  --name=portainer \
  --restart=unless-stopped \
  --security-opt no-new-privileges:true \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /mnt/docker-data/portainer/data:/data \
  -v /etc/ssl/certs:/etc/ssl/certs:ro \
  --memory=512m \
  --cpus=0.5 \
  -e TZ=America/New_York \
  portainer/portainer-ce:latest
```

### Portainer Features

| Feature | Status |
|---------|--------|
| Container Management | Enabled |
| Image Management | Enabled |
| Volume Management | Enabled |
| Network Management | Enabled |
| Stack Deployment | Enabled |
| Console Access | Enabled |
| Log Viewing | Enabled |

### Credentials Location

```
/home/agent0/HX-Infrastructure/credentials/hx-dev-server/portainer.yml
```

---

## Developer Templates

### Python Template

| Property | Value |
|----------|-------|
| Base Image | python:3.11-slim |
| Build Type | Multi-stage |
| Image Size | ~161 MB |
| User | appuser (UID 1000) |
| Location | /opt/dev-templates/python/ |

**Files:**
- `Dockerfile` - Multi-stage build with security hardening
- `.dockerignore` - Build optimization
- `entrypoint.sh` - Container initialization
- `requirements.txt` - Python dependencies template
- `README.md` - Usage instructions

### Node.js Template

| Property | Value |
|----------|-------|
| Base Image | node:20-slim |
| Build Type | Multi-stage |
| Image Size | ~213 MB |
| User | nodeuser (UID 1001) |
| Location | /opt/dev-templates/nodejs/ |

**Files:**
- `Dockerfile` - Multi-stage build with security hardening
- `.dockerignore` - Build optimization
- `entrypoint.sh` - Container initialization
- `package.json` - Node.js dependencies template
- `README.md` - Usage instructions

### Common Templates

| File | Purpose |
|------|---------|
| `.env.template` | Environment variables for AI services |
| `docker-compose.yml` | Multi-container orchestration template |
| `DEVELOPER-GUIDE.md` | Comprehensive development guide (16 KB) |
| `TEMPLATE-INDEX.md` | Template catalog and reference (6.7 KB) |

### Template Usage

```bash
# Copy Python template
cp -r /opt/dev-templates/python/ ~/my-python-project/

# Build container
cd ~/my-python-project
docker build -t my-python-app .

# Run with AI service connectivity
docker run -d \
  --name my-python-app \
  -v /etc/ssl/certs:/etc/ssl/certs:ro \
  --env-file .env \
  my-python-app
```

---

## Wildcard Routing

### Architecture

Developer containers are accessible via wildcard subdomains through hx-ssl-server reverse proxy.

```
Developer Browser
       │
       ▼
https://myapp.dev.hx.dev.local
       │
       ▼ DNS Resolution
*.dev.hx.dev.local → 192.168.10.202 (hx-ssl-server)
       │
       ▼ TLS Termination + Reverse Proxy
http://192.168.10.222:$port (hx-dev-server container)
```

### DNS Configuration

| Record | Type | Value |
|--------|------|-------|
| *.dev | A | 192.168.10.202 |
| Zone | - | hx.dev.local |
| DNS Server | - | hx-dc-server (192.168.10.200) |

### SSL Certificate

| Property | Value |
|----------|-------|
| Subject CN | *.dev.hx.dev.local |
| SAN | DNS:*.dev.hx.dev.local, DNS:dev.hx.dev.local |
| Key Size | 4096-bit RSA |
| Validity | 365 days |
| Issuer | HX-Infrastructure Internal CA |
| Location | hx-ssl-server:/etc/nginx/ssl/ |

### Port Mapping

Developer services are routed based on subdomain-to-port mapping:

```nginx
# /etc/nginx/conf.d/dev-container-ports.conf on hx-ssl-server
map $subdomain $dev_container_port {
    default 8080;
    myapp   8081;
    api     8082;
    # Add custom mappings as needed
}
```

### Exposing a Container

```bash
# 1. Run container on specific port
docker run -d -p 8081:80 --name myapp nginx:alpine

# 2. Add port mapping on hx-ssl-server
ssh agent0@hx-ssl-server.hx.dev.local
sudo nano /etc/nginx/conf.d/dev-container-ports.conf
# Add: myapp 8081;
sudo nginx -t && sudo systemctl reload nginx

# 3. Access via HTTPS
curl https://myapp.dev.hx.dev.local
```

---

## AI Service Integration

### Connected Services

| Service | Hostname | Port | Purpose | Status |
|---------|----------|------|---------|--------|
| LiteLLM | hx-litellm-server.hx.dev.local | 4000 | LLM gateway | Connected |
| Qdrant | hx-qdrant-server.hx.dev.local | 6333 | Vector database | Connected |
| PostgreSQL | hx-postgres-server.hx.dev.local | 5432 | Relational database | Connected |
| Redis | hx-redis-server.hx.dev.local | 6379 | Cache/session store | Connected |
| FastMCP | hx-fastmcp-server.hx.dev.local | 8000 | MCP gateway | Connected |

### Environment Variables (.env.template)

```env
# AI Service Configuration
LITELLM_API_URL=http://hx-litellm-server.hx.dev.local:4000
LITELLM_API_KEY=<from-vault>

QDRANT_URL=http://hx-qdrant-server.hx.dev.local:6333
QDRANT_API_KEY=<from-vault>

POSTGRES_HOST=hx-postgres-server.hx.dev.local
POSTGRES_PORT=5432
POSTGRES_USER=<from-vault>
POSTGRES_PASSWORD=<from-vault>

REDIS_HOST=hx-redis-server.hx.dev.local
REDIS_PORT=6379
REDIS_PASSWORD=<from-vault>

FASTMCP_URL=http://hx-fastmcp-server.hx.dev.local:8000

# Timezone
TZ=America/New_York
```

### Container Integration Pattern

```dockerfile
# Mount CA certificates for HTTPS trust
COPY --from=base /etc/ssl/certs /etc/ssl/certs

# Or use volume mount at runtime
# docker run -v /etc/ssl/certs:/etc/ssl/certs:ro ...
```

---

## Domain Integration

### AD Configuration

| Property | Value |
|----------|-------|
| Domain | hx.dev.local |
| Realm | HX.DEV.LOCAL |
| Domain Controller | hx-dc-server.hx.dev.local (192.168.10.200) |
| Authentication | SSSD + Kerberos |

### SSSD Service

```bash
# Check SSSD status
systemctl status sssd

# Verify domain membership
realm list
```

### SSH Access

Domain users can SSH using AD credentials:

```bash
# SSH with domain credentials
ssh username@hx.dev.local@192.168.10.222

# Or with explicit domain
ssh username@192.168.10.222
# (SSSD resolves domain automatically)
```

### Docker Group Access

Domain users requiring Docker access must be added to the docker group:

```bash
# Add domain user to docker group
sudo usermod -aG docker username@hx.dev.local

# Verify group membership
id username@hx.dev.local
```

---

## PKI & Certificate Trust

### CA Certificate

| Property | Value |
|----------|-------|
| CA Name | HX-Infrastructure Internal CA |
| Location | /usr/local/share/ca-certificates/hx-ca.crt |
| Symlink | /etc/ssl/certs/hx-ca.pem |
| Trust Store | /etc/ssl/certs/ca-certificates.crt |

### Container CA Inheritance

Containers inherit CA trust via volume mount:

```bash
# Docker run
docker run -v /etc/ssl/certs:/etc/ssl/certs:ro ...

# Docker Compose
volumes:
  - /etc/ssl/certs:/etc/ssl/certs:ro
```

### Verification

```bash
# Verify CA installed on host
ls -la /etc/ssl/certs/hx-ca.pem

# Test HTTP from container (LiteLLM uses HTTP internally)
docker run --rm -v /etc/ssl/certs:/etc/ssl/certs:ro alpine \
  wget -qO- http://hx-litellm-server.hx.dev.local:4000/health
```

---

## Administration

### Service Management

```bash
# Docker service
ssh agent0@hx-dev-server.hx.dev.local 'systemctl status docker'
ssh agent0@hx-dev-server.hx.dev.local 'sudo systemctl restart docker'

# Portainer container
ssh agent0@hx-dev-server.hx.dev.local 'docker ps -f name=portainer'
ssh agent0@hx-dev-server.hx.dev.local 'docker restart portainer'

# View Docker logs
ssh agent0@hx-dev-server.hx.dev.local 'journalctl -u docker -n 50'
```

### Container Management

```bash
# List all containers
docker ps -a

# Container logs
docker logs <container_name>

# Container shell
docker exec -it <container_name> /bin/bash

# Stop all containers
docker stop $(docker ps -q)

# Cleanup unused resources
docker system prune -af
```

### Storage Management

```bash
# Check disk usage
df -h /mnt/docker-data

# Docker disk usage
docker system df

# Cleanup unused images
docker image prune -af

# Cleanup unused volumes
docker volume prune -f
```

---

## Monitoring & Health

### Health Check Endpoints

```bash
# Docker daemon
curl --unix-socket /var/run/docker.sock http://localhost/version

# Portainer
curl -k https://192.168.10.222:9443/api/system/status
```

### Key Metrics

| Metric | Check Command | Alert Threshold |
|--------|---------------|-----------------|
| Docker Service | `systemctl is-active docker` | != active |
| Portainer | `docker inspect portainer --format='{{.State.Status}}'` | != running |
| Disk Usage | `df -h /mnt/docker-data` | > 90% |
| Memory | `free -m` | > 90% used |
| Running Containers | `docker ps -q \| wc -l` | As expected |

### Connectivity Tests

```bash
# Test AI services from hx-dev-server
curl -s http://hx-litellm-server.hx.dev.local:4000/health
curl -s http://hx-qdrant-server.hx.dev.local:6333/health
redis-cli -h hx-redis-server.hx.dev.local ping
```

---

## Troubleshooting

### Docker Won't Start

```bash
# Check Docker status
systemctl status docker

# Check logs
journalctl -u docker -n 100 --no-pager

# Verify data root mount
mount | grep docker-data
df -h /mnt/docker-data
```

### Portainer Inaccessible

```bash
# Check container status
docker ps -a -f name=portainer

# Check container logs
docker logs portainer

# Restart container
docker restart portainer
```

### Container Can't Reach AI Services

```bash
# Verify DNS resolution
docker run --rm alpine nslookup hx-litellm-server.hx.dev.local

# Verify network connectivity
docker run --rm alpine ping -c 3 192.168.10.212

# Verify CA trust
docker run --rm -v /etc/ssl/certs:/etc/ssl/certs:ro alpine \
  wget -qO- http://hx-litellm-server.hx.dev.local:4000/health
```

### AD Authentication Fails

```bash
# Check SSSD status
systemctl status sssd

# Test user lookup
getent passwd username@hx.dev.local

# Check Kerberos
klist -k /etc/krb5.keytab
```

### Wildcard Routing Not Working

```bash
# Test DNS resolution
nslookup myapp.dev.hx.dev.local

# Test direct container access
curl http://192.168.10.222:8081/

# Check nginx on hx-ssl-server
ssh agent0@hx-ssl-server.hx.dev.local 'nginx -t'
ssh agent0@hx-ssl-server.hx.dev.local 'cat /etc/nginx/conf.d/dev-container-ports.conf'
```

---

## Related Documentation

### Node Documentation

| Document | Location |
|----------|----------|
| Charter | `nodes/hx-dev-server/charter/charter.md` |
| Specification | `nodes/hx-dev-server/specification/node-spec.md` |
| Task Breakdown | `nodes/hx-dev-server/tasks/task-breakdown.md` |
| Test Plan | `nodes/hx-dev-server/tests/test-plan.md` |
| Test Suite | `nodes/hx-dev-server/tests/test-suite-index.md` |
| Execution Tracking | `nodes/hx-dev-server/execution-tracking.md` |

### Developer Resources

| Document | Location |
|----------|----------|
| Developer Guide | `/opt/dev-templates/common/DEVELOPER-GUIDE.md` |
| Template Index | `/opt/dev-templates/common/TEMPLATE-INDEX.md` |
| CA Trust Pattern | `nodes/hx-dev-server/ca-trust-pattern.md` |

### HX-Infrastructure References

| Document | Location |
|----------|----------|
| Node Inventory | `inventory/nodes.md` |
| Naming Standards | `standards/naming-conventions.md` |
| Deployment Requirements | `standards/deployment-requirements.md` |
| Testing Requirements | `standards/testing-requirements.md` |

### Support Escalation

| Level | Contact | Scope |
|-------|---------|-------|
| L1 | Developer Self-Service | Portainer UI, Docker CLI |
| L2 | william-chen | Infrastructure, Docker, storage |
| L3 | frank-lucas | Security, PKI, domain auth |
| L4 | CAIO | Critical issues |

---

## Version History

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 1.0 | 2025-12-06 | Initial service descriptor | HX-Infrastructure Team |
| 1.1 | 2025-12-08 | Clarified port routing, fixed Portainer URL to use DNS, corrected HTTP/HTTPS in examples | Agent Zero |

---

**Last Updated**: 2025-12-08
**Maintained By**: HX-Infrastructure Team
**Approved By**: CAIO (Jarvis Richardson)
