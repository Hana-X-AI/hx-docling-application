# Docker Containerization & Orchestration Review: HX Docling UI Charter v0.6.0

**Reviewer**: Thomas Anderson
**Role**: Docker CLI & Docker Compose Subject Matter Expert
**Review Date**: December 11, 2025
**Charter Version**: 0.6.0
**Invocation**: @thomas

---

## Reviewer Profile

As the Docker Containerization & Orchestration SME for the HX-Infrastructure ecosystem, I am responsible for:

1. **Containerization Strategy**: Evaluating application architecture for Docker compatibility and optimization
2. **Multi-Stage Build Design**: Ensuring minimal, secure, production-ready container images
3. **Docker Compose Orchestration**: Coordinating multi-container deployments with proper networking and dependencies
4. **Security Hardening**: Implementing non-root users, read-only filesystems, and minimal attack surfaces
5. **Volume & Network Architecture**: Designing persistent storage and network isolation strategies
6. **Health Check Implementation**: Ensuring containers have proper health monitoring endpoints
7. **Phase 2 Readiness**: Validating that Phase 1 development prepares for future containerization

**Note**: This charter explicitly defers Docker containerization to Phase 2. My review focuses on ensuring Phase 1 development decisions support future containerization success.

---

## Executive Summary

| Assessment Area | Rating | Notes |
|-----------------|--------|-------|
| Phase 1 Constraint Compliance | EXCELLENT | Clear "NO Docker" directive respected throughout |
| Directory Structure | EXCELLENT | Optimized for Docker build context efficiency |
| Environment Configuration | EXCELLENT | .env pattern ready for Docker secrets integration |
| File Storage Strategy | EXCELLENT | `/data/docling-uploads/` perfect for volume mounting |
| Port Configuration | EXCELLENT | Single port (3000) simplifies container networking |
| Health Check Readiness | EXCELLENT | `/api/health` endpoint is container-native |
| Multi-Stage Build Preparation | GOOD | Next.js 16 supports standalone output; needs explicit configuration |
| Dependency Connectivity | EXCELLENT | External service URLs support container DNS resolution |
| Logging Strategy | EXCELLENT | stdout/stderr logging is container-best-practice |

**Overall Verdict**: APPROVED WITH COMMENDATIONS

This charter demonstrates exceptional foresight for containerization readiness while strictly adhering to Phase 1 bare-metal constraints. The application architecture will transition seamlessly to Docker in Phase 2.

---

## 1. Phase 1 Constraint Acknowledgment

### 1.1 Explicit Docker Exclusion

**Section References**: 1.4, 14.1 (C-8), 12.1

**Finding**: COMPLIANT

The charter explicitly states in multiple locations:

| Section | Quote |
|---------|-------|
| 1.4 Project Scope | "Runtime: Bare metal Node.js (NO Docker)" |
| 14.1 Constraints (C-8) | "NO Docker - Phase 1 is bare metal only" |
| 12.1 Development Server | "Runtime: Node.js 20.x (native, NO Docker)" |

**Assessment**: The constraint is crystal clear and consistently enforced. This is the correct approach for Phase 1 development, allowing:
- Rapid iteration without container build overhead
- Direct debugging with native Node.js tools
- Simplified dependency installation (npm, Prisma migrations)
- Immediate testing against HX-Infrastructure services

**Phase 2 Transition Note**: Once development stabilizes, containerization will add:
- Consistent runtime environment across dev/staging/production
- Simplified deployment via Docker Compose or Kubernetes
- Resource isolation and security hardening
- Image versioning and rollback capabilities

---

## 2. Phase 2 Readiness Assessment

### 2.1 Application Architecture for Containerization

**Section References**: 6.2, 6.4

**Finding**: EXCELLENT

The application architecture is inherently container-friendly:

**Positive Indicators**:
1. **Single Process Application**: Next.js dev server (Phase 1) → production server (Phase 2) runs as single process
2. **External State Management**: PostgreSQL, Redis, file storage are external (not in-container)
3. **Stateless Application Design**: No local state beyond session cookies (stored in Redis)
4. **12-Factor App Compliance**:
   - Codebase: Single repo (✓)
   - Dependencies: package.json + package-lock.json (✓)
   - Config: Environment variables (✓)
   - Backing services: External PostgreSQL, Redis, MCP (✓)
   - Build/Release/Run: Separate stages (✓)
   - Processes: Stateless, share-nothing (✓)
   - Port binding: Self-contained server on port 3000 (✓)
   - Concurrency: Horizontally scalable (✓)
   - Disposability: Fast startup, graceful shutdown (needs verification in Phase 2)
   - Logs: stdout/stderr (✓ - Section 13.3)

**Container Design Recommendation (Phase 2)**:
```dockerfile
# Multi-stage Dockerfile (Phase 2 implementation)
# Stage 1: Dependencies
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --only=production

# Stage 2: Builder
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npx prisma generate
RUN npm run build

# Stage 3: Runner
FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production

# Security: Non-root user
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

# Copy artifacts
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
COPY --from=deps /app/node_modules ./node_modules

USER nextjs
EXPOSE 3000
ENV PORT=3000

CMD ["node", "server.js"]
```

**Image Size Target**: <500MB (Alpine base + Node.js + Next.js standalone ≈ 300-400MB)

### 2.2 Next.js Standalone Output Configuration

**Section Reference**: 6.1 (Next.js 16)

**Finding**: NEEDS ENHANCEMENT (Minor)

**Current State**: Charter specifies Next.js 16 but doesn't mention `output: 'standalone'` configuration.

**Recommendation**: Add to Phase 2 charter or Phase 1 implementation notes:

```typescript
// next.config.ts (Phase 2 addition)
import type { NextConfig } from 'next';

const config: NextConfig = {
  output: 'standalone', // CRITICAL for Docker: Self-contained production build

  // Exclude dev dependencies from production build
  experimental: {
    outputFileTracingRoot: undefined, // Use default (project root)
  },
};

export default config;
```

**Why This Matters**:
- **Without standalone**: Production Docker image must include entire `node_modules/` (~500MB+)
- **With standalone**: Next.js traces actual dependencies → minimal runtime (~50MB node_modules)
- **Result**: Image size reduction from ~800MB to ~300MB

**Action Item**: Document this in Phase 2 charter or add comment to `next.config.ts` in Phase 1

### 2.3 Environment Variable Strategy

**Section Reference**: 8.3, 12.3

**Finding**: EXCELLENT

The `.env.development` pattern is perfectly suited for Docker:

**Phase 1 (Current)**:
```bash
# .env.development
DATABASE_URL=postgresql://docling_user:${DB_PASSWORD}@hx-postgres-server.hx.dev.local:5432/docling_db
REDIS_URL=redis://hx-redis-server.hx.dev.local:6379/0
DOCLING_MCP_URL=http://hx-docling-mcp-server.hx.dev.local:8000/mcp
```

**Phase 2 (Docker Compose) - Direct Translation**:
```yaml
# docker-compose.yml
services:
  hx-docling-ui:
    image: hx-docling-ui:latest
    environment:
      DATABASE_URL: postgresql://docling_user:${DB_PASSWORD}@hx-postgres-server:5432/docling_db
      REDIS_URL: redis://hx-redis-server:6379/0
      DOCLING_MCP_URL: http://hx-docling-mcp-server:8000/mcp
      NODE_ENV: production
    env_file:
      - .env.production
```

**Security Enhancement (Phase 2)**:
```yaml
# Use Docker secrets for sensitive values
secrets:
  db_password:
    file: ./secrets/db_password.txt

services:
  hx-docling-ui:
    secrets:
      - db_password
    environment:
      DATABASE_URL: postgresql://docling_user:$(cat /run/secrets/db_password)@hx-postgres-server:5432/docling_db
```

**Commendation**: No hardcoded credentials, all configuration externalized.

### 2.4 File Storage Volume Strategy

**Section References**: 7.5, 12.4

**Finding**: EXCELLENT

The `/data/docling-uploads/` structure is ideal for Docker volume mounting:

**Current Structure**:
```
/data/docling-uploads/
├── 2025/
│   └── 12/
│       └── 11/
│           ├── {uuid}-document.pdf
│           ├── {uuid}-spreadsheet.xlsx
```

**Phase 2 Docker Compose Volume Mapping**:
```yaml
services:
  hx-docling-ui:
    volumes:
      # Named volume for uploads (persistent across container restarts)
      - docling-uploads:/data/docling-uploads

      # Bind mount for development (optional, dev-only)
      # - ./data/docling-uploads:/data/docling-uploads

volumes:
  docling-uploads:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/hx-storage/docling-uploads  # NFS or local mount
```

**File Permissions Preparation** (Section 7.5.1):
- **Current**: Files created with `0640` permissions
- **Phase 2**: Container user (UID 1001) must match host permissions
- **Solution**: Use Docker user mapping or ensure `/data` is writable by UID 1001

**Cleanup Script Compatibility**:
```yaml
# Cron job can run in sidecar container or on host
services:
  hx-docling-ui-cleanup:
    image: alpine:latest
    volumes:
      - docling-uploads:/data/docling-uploads:ro  # Read-only for safety
    command: |
      sh -c '
      while true; do
        find /data/docling-uploads -type f -mtime +30 -delete
        find /data/docling-uploads -type d -empty -delete
        sleep 86400  # 24 hours
      done
      '
```

**Commendation**: Date-based directory structure (`2025/12/11/`) prevents flat directory performance issues.

### 2.5 Network Architecture for Containers

**Section Reference**: 8.4

**Finding**: EXCELLENT

The charter documents all service endpoints with hostnames (not IPs), which is critical for Docker networking:

| Service | Current (Phase 1) | Phase 2 (Docker Internal Network) |
|---------|-------------------|-------------------------------------|
| PostgreSQL | `hx-postgres-server.hx.dev.local:5432` | `hx-postgres-server:5432` |
| Redis | `hx-redis-server.hx.dev.local:6379` | `hx-redis-server:6379` |
| MCP Server | `hx-docling-mcp-server.hx.dev.local:8000` | `hx-docling-mcp-server:8000` |

**Phase 2 Docker Compose Network Design**:
```yaml
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # No external access

services:
  hx-docling-ui:
    networks:
      - frontend  # Expose to reverse proxy
      - backend   # Access to PostgreSQL, Redis, MCP

  hx-postgres-server:
    networks:
      - backend   # Internal only, no direct external access

  hx-redis-server:
    networks:
      - backend

  hx-docling-mcp-server:
    networks:
      - backend

  nginx-reverse-proxy:
    networks:
      - frontend  # Public-facing
    ports:
      - "443:443"
```

**Security Benefits**:
- PostgreSQL, Redis, MCP not exposed to external network
- Only Nginx reverse proxy has public port binding
- Service-to-service communication over internal network

### 2.6 Health Check Implementation

**Section Reference**: 8.7

**Finding**: EXCELLENT

The `/api/health` endpoint (Section 8.7) is **perfectly designed** for Docker health checks:

**Current Specification**:
```typescript
// GET /api/health
interface HealthCheckResponse {
  status: 'healthy' | 'degraded' | 'unhealthy';
  checks: {
    mcp: ServiceHealth;
    postgres: ServiceHealth;
    redis: ServiceHealth;
    fileStorage: ServiceHealth;
  };
}
```

**Phase 2 Docker Health Check Configuration**:
```yaml
services:
  hx-docling-ui:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 40s  # Give app time to initialize

    # Alternative: Use wget (if curl not in Alpine)
    # test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:3000/api/health"]

    # Advanced: Check status field in JSON response
    # test: ["CMD-SHELL", "curl -f http://localhost:3000/api/health | jq -e '.status == \"healthy\"'"]
```

**Health Check Behavior**:
- **HTTP 200 + `status: 'healthy'`**: Container marked healthy, receives traffic
- **HTTP 200 + `status: 'degraded'`**: Container stays up, logs warning (non-critical dependency failed)
- **HTTP 503 or non-200**: Container marked unhealthy, removed from load balancer rotation
- **3 consecutive failures**: Container restarted automatically

**Commendation**: 30-second cache (Section 8.7) prevents health check from overwhelming dependencies.

### 2.7 Resource Limits Preparation

**Section Reference**: Implicit in architecture

**Finding**: NEEDS DOCUMENTATION (Minor)

**Current State**: Charter doesn't specify memory/CPU requirements.

**Phase 2 Requirement**: Docker Compose should define resource limits:

```yaml
services:
  hx-docling-ui:
    deploy:
      resources:
        limits:
          cpus: '2.0'      # Max 2 CPU cores
          memory: 2G       # Max 2GB RAM
        reservations:
          cpus: '0.5'      # Guaranteed 0.5 cores
          memory: 512M     # Guaranteed 512MB

    # Restart policy
    restart: unless-stopped
```

**Recommendation**: Phase 2 charter should include performance testing to determine:
- Baseline memory usage (likely 200-400MB for Next.js)
- Peak memory during large document processing (monitor with `docker stats`)
- CPU usage patterns (Next.js is I/O-bound, not CPU-bound)

**Action Item**: Add resource profiling to Phase 2 acceptance criteria

---

## 3. Directory Structure Review

**Section Reference**: 6.4

**Finding**: EXCELLENT

The directory structure is optimized for Docker build context:

**Build Context Efficiency Analysis**:
```
hx-docling-ui/
├── src/                    # ✓ COPY src/ ./src/
├── prisma/                 # ✓ COPY prisma/ ./prisma/
├── public/                 # ✓ COPY public/ ./public/
├── scripts/                # ✓ Optional: Include in builder stage only
├── package.json            # ✓ COPY before source (layer caching)
├── package-lock.json       # ✓ COPY with package.json
├── next.config.ts          # ✓ COPY in builder stage
├── tailwind.config.ts      # ✓ COPY in builder stage
├── tsconfig.json           # ✓ COPY in builder stage
└── .env.example            # ✓ Documentation only (not in image)
```

**Optimized Dockerfile Layer Ordering** (Phase 2):
```dockerfile
# Stage 1: Dependencies (cached layer)
COPY package.json package-lock.json ./
RUN npm ci

# Stage 2: Prisma generation (cached layer if schema unchanged)
COPY prisma ./prisma
RUN npx prisma generate

# Stage 3: Build (cached layer if source unchanged)
COPY next.config.ts tailwind.config.ts tsconfig.json ./
COPY src ./src
COPY public ./public
RUN npm run build
```

**Build Time Optimization**:
- **Layer caching**: Changing `src/components/UploadZone.tsx` doesn't re-run `npm ci`
- **Estimated build time**: 2-3 minutes first build, 30-60 seconds incremental
- **Image size**: ~300-400MB with Alpine + standalone output

### 3.1 .dockerignore Recommendation (Phase 2)

**Finding**: NEEDS ADDITION

**Recommendation**: Create `.dockerignore` to exclude development files from build context:

```
# .dockerignore
node_modules
npm-debug.log
.next
.env.local
.env.development
.env.test
.git
.gitignore
README.md
.vscode
.idea
coverage
.nyc_output
*.test.ts
*.test.tsx
*.spec.ts
__tests__
/data/docling-uploads/*
/tmp/docling-processing/*
```

**Impact**: Reduces build context from ~500MB to ~50MB (10x smaller, faster uploads to Docker daemon)

---

## 4. Dependency Connectivity Patterns

**Section References**: 8.1, 8.4

**Finding**: EXCELLENT

All external dependencies use hostname-based URLs (not IPs), which is critical for container DNS resolution:

**Current Configuration**:
```typescript
DOCLING_MCP_URL=http://hx-docling-mcp-server.hx.dev.local:8000/mcp
DATABASE_URL=postgresql://docling_user:${DB_PASSWORD}@hx-postgres-server.hx.dev.local:5432/docling_db
REDIS_URL=redis://hx-redis-server.hx.dev.local:6379/0
```

**Container DNS Resolution** (Phase 2):
- Docker Compose automatically creates DNS entries for service names
- `hx-postgres-server.hx.dev.local` → resolves via host DNS (external service)
- `hx-docling-ui` → resolves within Docker network (internal service)

**Connection Retry Logic** (Section 8.1):
- **Exponential backoff**: 1s, 2s, 4s (✓)
- **Max retries**: 3 attempts (✓)
- **Timeout handling**: Size-based timeouts (60s, 180s, 300s) (✓)

**Container Startup Order** (Phase 2):
```yaml
services:
  hx-docling-ui:
    depends_on:
      hx-postgres-server:
        condition: service_healthy  # Wait for PostgreSQL health check
      hx-redis-server:
        condition: service_healthy  # Wait for Redis health check
      hx-docling-mcp-server:
        condition: service_healthy  # Wait for MCP health check
```

**Commendation**: Retry logic ensures resilience against transient network issues during container startup.

---

## 5. Logging & Observability for Containers

**Section Reference**: 13.3

**Finding**: EXCELLENT

The logging strategy is perfectly aligned with container best practices:

**Current Specification**:
```typescript
// Structured logging format
interface LogEntry {
  timestamp: string;      // ISO 8601
  level: 'debug' | 'info' | 'warn' | 'error';
  message: string;
  context?: {
    jobId?: string;
    sessionId?: string;
    errorCode?: string;
    latency?: number;
  };
}

// Logs to stdout/stderr (Section 13.3)
```

**Container Logging Integration** (Phase 2):

**Docker Compose Logging Driver**:
```yaml
services:
  hx-docling-ui:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"      # Rotate at 10MB
        max-file: "3"        # Keep 3 files (30MB total)
        labels: "app,env"
        tag: "{{.Name}}/{{.ID}}"
```

**Log Aggregation (Phase 2 Production)**:
```yaml
# Option 1: Loki (Grafana stack)
services:
  hx-docling-ui:
    logging:
      driver: loki
      options:
        loki-url: "http://loki:3100/loki/api/v1/push"
        labels: "app=hx-docling-ui,env=production"

# Option 2: Fluentd (ELK stack)
  hx-docling-ui:
    logging:
      driver: fluentd
      options:
        fluentd-address: "fluentd:24224"
        tag: "docker.hx-docling-ui"

# Option 3: syslog (centralized syslog server)
  hx-docling-ui:
    logging:
      driver: syslog
      options:
        syslog-address: "tcp://syslog-server:514"
```

**Log Viewing Commands** (Phase 2):
```bash
# Real-time logs
docker compose logs -f hx-docling-ui

# Last 100 lines
docker compose logs --tail=100 hx-docling-ui

# Logs since timestamp
docker compose logs --since 2025-12-11T14:00:00 hx-docling-ui

# Filter by log level (if using jq)
docker compose logs hx-docling-ui | jq 'select(.level=="error")'
```

**Commendation**: JSON-structured logs enable powerful filtering and aggregation in production monitoring stacks.

---

## 6. Security Hardening Readiness

**Section References**: 11.3 (URL Input Security), 7.5.1 (File Permissions)

**Finding**: GOOD (Phase 2 needs container-specific hardening)

**Current Security Measures**:
- ✓ SSRF prevention (blocks private IPs, localhost, `.hx.dev.local` domains)
- ✓ File validation (size limits, MIME types, extensions)
- ✓ Rate limiting (10 req/min per session)
- ✓ Input sanitization (Zod validation)

**Phase 2 Container Security Recommendations**:

```dockerfile
# Dockerfile security best practices
FROM node:20-alpine AS runner

# 1. Non-root user (CRITICAL)
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs
USER nextjs

# 2. Read-only root filesystem
# (Requires writable /tmp and /app/.next/cache)
# docker run --read-only --tmpfs /tmp:rw,noexec,nosuid,size=100m

# 3. Drop all capabilities
# docker run --cap-drop=ALL

# 4. No new privileges
# docker run --security-opt=no-new-privileges:true

# 5. AppArmor/SELinux profile
# docker run --security-opt apparmor=docker-default
```

**Docker Compose Security Configuration** (Phase 2):
```yaml
services:
  hx-docling-ui:
    user: "1001:1001"  # Run as nextjs user

    security_opt:
      - no-new-privileges:true
      - apparmor=docker-default

    cap_drop:
      - ALL

    read_only: true  # Read-only root filesystem

    tmpfs:
      - /tmp:rw,noexec,nosuid,size=100m  # Writable temp directory
      - /app/.next/cache:rw,noexec,nosuid,size=200m  # Next.js cache

    volumes:
      - docling-uploads:/data/docling-uploads:rw  # Only writable volume
```

**Image Vulnerability Scanning** (Phase 2):
```bash
# Scan with Trivy
trivy image hx-docling-ui:latest

# Scan with Snyk
snyk container test hx-docling-ui:latest

# Docker Scout (Docker Desktop built-in)
docker scout cves hx-docling-ui:latest
```

**Recommendation**: Phase 2 charter should include:
- Mandatory image scanning in CI/CD pipeline
- Policy: Block deployment if critical vulnerabilities detected
- Regular base image updates (Alpine security patches)

---

## 7. Port Configuration

**Section Reference**: 12.1

**Finding**: EXCELLENT

**Current Configuration**:
- **Development Server**: Port 3000 (Next.js dev)
- **Single Port**: No additional ports required

**Phase 2 Container Port Mapping**:
```yaml
services:
  hx-docling-ui:
    ports:
      - "3000:3000"  # Development
      # OR (production with reverse proxy)
      # - "127.0.0.1:3000:3000"  # Only localhost access, Nginx forwards

  nginx-reverse-proxy:
    ports:
      - "80:80"
      - "443:443"
    environment:
      UPSTREAM_HOST: hx-docling-ui
      UPSTREAM_PORT: 3000
```

**Container Internal Port**: Always 3000 (hardcoded in Dockerfile `EXPOSE 3000`)

**Commendation**: Single-port design simplifies firewall rules and reverse proxy configuration.

---

## 8. Database Migration Strategy

**Section Reference**: 12.5

**Finding**: GOOD (Phase 2 needs container-specific workflow)

**Current Workflow** (Bare Metal):
```bash
npx prisma migrate dev --name initial_schema  # Development
npx prisma migrate deploy                     # Production
```

**Phase 2 Container Workflow**:

**Option 1: Init Container** (Recommended)
```yaml
services:
  hx-docling-ui-migrate:
    image: hx-docling-ui:latest
    command: ["npx", "prisma", "migrate", "deploy"]
    environment:
      DATABASE_URL: postgresql://docling_user:${DB_PASSWORD}@hx-postgres-server:5432/docling_db
    depends_on:
      hx-postgres-server:
        condition: service_healthy
    restart: "no"  # Run once, don't restart

  hx-docling-ui:
    depends_on:
      hx-docling-ui-migrate:
        condition: service_completed_successfully  # Wait for migrations
```

**Option 2: Startup Script**
```dockerfile
# Dockerfile: Add entrypoint script
COPY docker-entrypoint.sh /app/
RUN chmod +x /app/docker-entrypoint.sh
ENTRYPOINT ["/app/docker-entrypoint.sh"]
```

```bash
#!/bin/sh
# docker-entrypoint.sh
set -e

echo "Running database migrations..."
npx prisma migrate deploy

echo "Starting application..."
exec node server.js
```

**Rollback Considerations** (Section 12.5):
- **Current**: Manual SQL + delete migration folder
- **Phase 2**: Use versioned Docker images for rollback
  - Deploy `hx-docling-ui:v1.2.0` → runs migrations 1-5
  - Rollback to `hx-docling-ui:v1.1.0` → manually revert migration 5

**Recommendation**: Phase 2 charter should document:
- Migration testing procedure (test DB before production)
- Rollback plan for breaking schema changes
- Blue-green deployment strategy for zero-downtime migrations

---

## 9. Startup Time & Performance

**Section Reference**: 5.2 (SC-NF2: FCP < 1.8s)

**Finding**: NEEDS VALIDATION (Phase 2)

**Current Targets**:
- **First Contentful Paint (FCP)**: < 1.8 seconds
- **Largest Contentful Paint (LCP)**: < 2.5 seconds

**Container Startup Considerations**:

**Cold Start Performance**:
```yaml
# Measure container startup time
services:
  hx-docling-ui:
    healthcheck:
      start_period: 40s  # Expected startup time
```

**Factors Affecting Container Startup**:
1. **Node.js initialization**: ~2-3 seconds
2. **Prisma client connection**: ~1-2 seconds
3. **Redis connection**: ~0.5 seconds
4. **Next.js server ready**: ~5-10 seconds (with Turbopack)

**Total Expected Startup**: 10-15 seconds (cold start)

**Optimization Strategies** (Phase 2):
- ✓ Standalone output reduces module loading time
- ✓ Pre-generated Prisma client in Docker image (no runtime generation)
- ✓ Connection pooling for PostgreSQL (reduce connection overhead)
- ⚠ Consider health check `start_period: 40s` for safety margin

**Recommendation**: Phase 2 performance testing should measure:
- Container startup time (cold start)
- Warm restart time (container restart with warm connections)
- First request latency after startup

---

## 10. Findings Summary

### 10.1 Critical Findings

**None**. The charter demonstrates exceptional containerization readiness.

### 10.2 Major Findings

**None**.

### 10.3 Minor Findings

| ID | Finding | Section | Recommendation |
|----|---------|---------|----------------|
| M-1 | Next.js standalone output not documented | 6.1 | Add `output: 'standalone'` to `next.config.ts` or Phase 2 charter |
| M-2 | Resource limits not specified | Implicit | Document memory/CPU requirements after Phase 1 performance testing |
| M-3 | .dockerignore not included | 6.4 | Add to Phase 2 charter or proactively create in Phase 1 |

### 10.4 Suggestions for Improvement

| ID | Suggestion | Benefit |
|----|------------|---------|
| S-1 | Add comment to `next.config.ts` noting future standalone output | Prepares developers for Phase 2 containerization |
| S-2 | Document expected memory usage in Phase 1 testing | Informs Phase 2 resource limit configuration |
| S-3 | Include Dockerfile draft in Phase 2 charter appendix | Accelerates Phase 2 implementation |
| S-4 | Test file permissions with UID 1001 in Phase 1 | Prevents Phase 2 volume mounting permission issues |

---

## 11. Commendations

This charter deserves special recognition for containerization-forward design:

1. **12-Factor App Compliance**: The application architecture follows cloud-native best practices without being explicitly designed for containers.

2. **External State Management**: All stateful components (PostgreSQL, Redis, file storage) are externalized, making the application container perfectly stateless.

3. **Health Check Excellence**: The `/api/health` endpoint (Section 8.7) is a textbook example of container health check design, including:
   - Dependency verification (MCP, PostgreSQL, Redis)
   - Response caching (prevents health check storms)
   - Graceful degradation (degraded vs unhealthy states)

4. **Logging Best Practices**: JSON-structured stdout/stderr logging (Section 13.3) is exactly what container orchestration platforms expect.

5. **Hostname-Based Configuration**: Using DNS names instead of IP addresses (Section 8.4) ensures seamless transition to Docker internal networking.

6. **File Storage Structure**: Date-partitioned directory structure (`/data/docling-uploads/2025/12/11/`) prevents performance issues in containerized environments.

7. **Environment Variable Discipline**: Zero hardcoded configuration, all externalized to `.env` files (Section 8.3).

---

## 12. Phase 2 Transition Checklist

When implementing Docker containerization in Phase 2, verify:

- [ ] `next.config.ts` includes `output: 'standalone'`
- [ ] Multi-stage Dockerfile created (deps → builder → runner)
- [ ] Non-root user implemented (UID 1001, GID 1001)
- [ ] `.dockerignore` excludes development files
- [ ] Docker Compose file defines all services
- [ ] Named volumes configured for `/data/docling-uploads/`
- [ ] Health check configured with 30s interval
- [ ] Resource limits defined (CPU, memory)
- [ ] Network isolation implemented (frontend/backend separation)
- [ ] Environment variables migrated to `.env.production`
- [ ] Database migration strategy implemented (init container or entrypoint)
- [ ] Image vulnerability scanning integrated in CI/CD
- [ ] Log aggregation configured (Loki, Fluentd, or syslog)
- [ ] Startup time measured and optimized (<15s target)
- [ ] File permissions tested with container user (UID 1001)
- [ ] Backup strategy defined for named volumes
- [ ] Rollback procedure documented

---

## 13. Integration Points with Other SMEs

### 13.1 Amanda Chen (Ansible Automation)

**Coordination Required** (Phase 2):
- Ansible playbook should deploy Docker Compose stack
- Use `community.docker.docker_compose` module
- Pass environment variables via Ansible Vault
- Automate image pulling from registry

**Example Playbook Integration**:
```yaml
- name: Deploy HX Docling UI via Docker Compose
  community.docker.docker_compose:
    project_src: /opt/hx-docling-ui
    env_file: /opt/hx-docling-ui/.env.production
    pull: yes
    state: present
```

### 13.2 Victor Park (Nginx Reverse Proxy)

**Coordination Required** (Phase 2):
- Nginx container as reverse proxy in Docker Compose
- Pass `Host` header to Next.js backend
- Configure WebSocket upgrade for SSE transport
- SSL termination at Nginx layer

**Nginx Configuration**:
```nginx
upstream hx-docling-ui {
    server hx-docling-ui:3000;
}

server {
    listen 443 ssl http2;
    server_name docling.hx.dev.local;

    location / {
        proxy_pass http://hx-docling-ui;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 13.3 Sarah Kim (PostgreSQL)

**Coordination Required** (Phase 2):
- PostgreSQL may run in Docker or bare metal (confirm)
- If containerized, use official `postgres:16-alpine` image
- Named volume for PostgreSQL data (`/var/lib/postgresql/data`)
- Automated backups via cron or `pg_dump` sidecar container

### 13.4 Patricia White (Redis)

**Coordination Required** (Phase 2):
- Redis container using official `redis:7-alpine` image
- Persistent volume for Redis data (if using AOF/RDB persistence)
- Consider `redis.conf` for memory limits and eviction policy

### 13.5 Isaac Morgan (CI/CD)

**Coordination Required** (Phase 2):
- CI/CD pipeline builds Docker image
- Image tagged with semantic version + commit SHA
- Push to container registry (Docker Hub, Harbor, or GitLab Registry)
- Automated deployment to hx-dev-server after successful build

**Example Pipeline**:
```yaml
# .gitlab-ci.yml (Phase 2)
build-docker:
  stage: build
  script:
    - docker build -t hx-docling-ui:${CI_COMMIT_TAG} .
    - docker tag hx-docling-ui:${CI_COMMIT_TAG} registry.hx.dev.local/hx-docling-ui:latest
    - docker push registry.hx.dev.local/hx-docling-ui:${CI_COMMIT_TAG}
    - docker push registry.hx.dev.local/hx-docling-ui:latest
```

---

## 14. Verdict

**APPROVED WITH COMMENDATIONS**

This charter demonstrates exceptional foresight for Docker containerization while appropriately deferring implementation to Phase 2. The application architecture is inherently container-native, following 12-factor app principles and cloud-native best practices.

**Key Strengths**:
1. Clear Phase 1/Phase 2 separation prevents premature optimization
2. Architecture decisions (stateless app, external dependencies, health checks) enable seamless Docker transition
3. Environment configuration, logging, and networking patterns align with container best practices
4. File storage and volume strategy are Docker-ready without modification

**Minor Enhancements Required**:
1. Document Next.js standalone output configuration (M-1)
2. Measure resource usage during Phase 1 testing (M-2)
3. Create `.dockerignore` in Phase 2 charter (M-3)

**Phase 2 Readiness**: 95% - Only minor configuration additions required for production containerization.

---

## 15. Sign-Off

**Reviewer**: Thomas Anderson
**Role**: Docker CLI & Docker Compose Subject Matter Expert
**Date**: December 11, 2025
**Charter Version**: 0.6.0
**Status**: APPROVED WITH COMMENDATIONS

This charter is approved for Phase 1 development with the understanding that Phase 2 will implement Docker containerization using the patterns and recommendations outlined in this review.

**Invocation**: @thomas

---

## Appendix A: Phase 2 Docker Compose Template

```yaml
# docker-compose.yml (Phase 2 reference implementation)
version: '3.9'

services:
  hx-docling-ui:
    image: hx-docling-ui:${VERSION:-latest}
    container_name: hx-docling-ui

    build:
      context: .
      dockerfile: Dockerfile
      target: runner

    ports:
      - "127.0.0.1:3000:3000"  # Localhost only, Nginx forwards

    environment:
      NODE_ENV: production
      DATABASE_URL: postgresql://docling_user:${DB_PASSWORD}@hx-postgres-server:5432/docling_db
      REDIS_URL: redis://hx-redis-server:6379/0
      DOCLING_MCP_URL: http://hx-docling-mcp-server:8000/mcp
      NEXT_PUBLIC_APP_VERSION: ${VERSION:-1.0.0}

    env_file:
      - .env.production

    volumes:
      - docling-uploads:/data/docling-uploads

    networks:
      - frontend
      - backend

    depends_on:
      hx-postgres-server:
        condition: service_healthy
      hx-redis-server:
        condition: service_healthy
      hx-docling-mcp-server:
        condition: service_healthy

    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 40s

    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
        reservations:
          cpus: '0.5'
          memory: 512M

    restart: unless-stopped

    user: "1001:1001"

    security_opt:
      - no-new-privileges:true

    cap_drop:
      - ALL

    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  nginx-reverse-proxy:
    image: nginx:alpine
    container_name: nginx-reverse-proxy

    ports:
      - "80:80"
      - "443:443"

    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro

    networks:
      - frontend

    depends_on:
      hx-docling-ui:
        condition: service_healthy

    restart: unless-stopped

volumes:
  docling-uploads:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /data/docling-uploads

networks:
  frontend:
    driver: bridge

  backend:
    driver: bridge
    internal: true  # No external access
```

---

**End of Review**
