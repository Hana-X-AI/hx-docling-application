# Infrastructure & Operations Review: HX Docling UI Charter

**Charter Reviewed**: `0.1-hx-docling-ui-charter.md` (v0.6.0)
**Review Date**: December 11, 2025
**Reviewer**: William Chen, Infrastructure & Operations Specialist
**Invocation**: @william

---

## Reviewer Profile

**Role**: Infrastructure & Operations Specialist
**Primary Responsibilities**:
- Bare metal server deployment and systemd service management
- PostgreSQL and Redis infrastructure configuration
- File storage architecture and capacity planning
- Monitoring, alerting, and health check implementation
- Backup and disaster recovery procedures
- Operational runbook development

**Review Focus Areas**:
1. PostgreSQL configuration (hx-postgres-server, docling_db)
2. Redis configuration (hx-redis-server, session management)
3. File storage architecture (/data/docling-uploads/)
4. Infrastructure dependencies (server IPs, ports, connectivity)
5. Database schema (Prisma migrations, indexes, retention)
6. Health check specifications
7. Operational concerns (monitoring, alerting, disk usage)

---

## Executive Summary

| Category | Rating | Assessment |
|----------|--------|------------|
| **PostgreSQL Infrastructure** | GOOD | Well-defined connection details, schema, and retention policies |
| **Redis Infrastructure** | GOOD | Clear session management strategy with appropriate TTLs |
| **File Storage Architecture** | GOOD | Comprehensive lifecycle management and cleanup procedures |
| **Health Check Design** | EXCELLENT | Thorough implementation with proper caching and status logic |
| **Monitoring & Alerting** | GOOD | Comprehensive metrics and alert thresholds defined |
| **Operational Readiness** | NEEDS WORK | Some gaps in runbook details and credential management |

**Overall Infrastructure Assessment**: APPROVED WITH CONDITIONS

The charter demonstrates solid infrastructure planning with well-defined server configurations, database schemas, and operational procedures. A few operational gaps need to be addressed before deployment.

---

## Section-by-Section Infrastructure Assessment

### 1. PostgreSQL Configuration Review

**Reference**: Section 7.1-7.3, Section 8.4, Section 12.5

#### Strengths

| Aspect | Assessment | Reference |
|--------|------------|-----------|
| Connection Details | Server IP (192.168.10.208), port (5432), database name (docling_db) clearly specified | Section 8.4 |
| Database Schema | Well-designed Prisma schema with appropriate relations, enums, and indexes | Section 7.3 |
| Index Strategy | Correct indexes on `sessionId`, `createdAt`, `status`, `jobId`, `format` | Section 7.3, 7.6.1 |
| Migration Workflow | Clear naming convention and migration procedures documented | Section 12.5 |
| Retention Policy | 90-day retention for jobs and results with cleanup script | Section 7.6 |
| Cascade Deletes | Results properly cascade on Job deletion | Section 7.3 |

#### Observations

**DATABASE_URL Pattern (Section 8.3)**:
```
DATABASE_URL=postgresql://docling_user:${DB_PASSWORD}@hx-postgres-server.hx.dev.local:5432/docling_db
```

This is appropriate - using environment variable substitution for the password prevents credential exposure. However, the actual credential storage mechanism needs clarification.

**Prisma Client Singleton (Section 12.3.1)**: Properly prevents connection pool exhaustion during hot reload. This is a critical pattern for Next.js development.

**Rollback Procedure (Section 12.5)**: The manual rollback procedure is honest about Prisma's limitations. This matches our operational philosophy of manual procedures with documentation.

#### Minor Findings

| ID | Finding | Section | Recommendation |
|----|---------|---------|----------------|
| INF-PG-01 | Missing connection pool size specification | 8.3 | Add `connection_limit` parameter to DATABASE_URL or Prisma config |
| INF-PG-02 | No database backup procedure specified | 7.6, 12.5 | Add pg_dump backup script with systemd timer |
| INF-PG-03 | Test database on same server as development | 12.5 | Acceptable for Phase 1, document isolation strategy |

---

### 2. Redis Configuration Review

**Reference**: Section 7.1, 7.4, 8.3, 8.4

#### Strengths

| Aspect | Assessment | Reference |
|--------|------------|-----------|
| Connection Details | Server IP (192.168.10.209), port (6379), database 0 | Section 8.4 |
| Session Structure | Clear interface definition with all required fields | Section 7.4 |
| TTL Strategy | 24-hour session expiry is appropriate for anonymous sessions | Section 7.4 |
| Key Pattern | `session:{sessionId}` is simple and effective | Section 7.4 |
| Edge Cases | Session expiry scenarios well-documented | Section 7.4.1 |

#### Observations

**Redis URL Pattern (Section 8.3)**:
```
REDIS_URL=redis://hx-redis-server.hx.dev.local:6379/0
```

No authentication specified. This is acceptable for internal network deployment but should be documented as a security consideration.

**Session Timeout Cascade (Section 7.4.1)**: Excellent documentation of the cascade effect when sessions expire. The separation between session lifetime (24h) and data retention (90 days) is well thought out.

**Multi-Tab Handling (Section 7.4.1)**: Rate limiting per session across tabs is correctly specified. This prevents abuse while allowing legitimate multi-tab usage.

#### Minor Findings

| ID | Finding | Section | Recommendation |
|----|---------|---------|----------------|
| INF-RD-01 | No Redis authentication configured | 8.3 | Document security posture; consider Redis AUTH for Phase 2 |
| INF-RD-02 | No Redis persistence mode specified | 7.4 | Verify hx-redis-server has AOF or RDB persistence enabled |
| INF-RD-03 | No Redis connection retry strategy | 7.4 | Add ioredis retry configuration in client setup |

---

### 3. File Storage Architecture Review

**Reference**: Section 7.1, 7.5, 7.5.1, 7.6

#### Strengths

| Aspect | Assessment | Reference |
|--------|------------|-----------|
| Directory Structure | Date-based hierarchy (`YYYY/MM/DD/`) enables efficient cleanup | Section 7.5 |
| File Permissions | `0640` with owner/group separation is appropriate | Section 7.5.1 |
| Cleanup Strategy | 30-day retention with daily cron job | Section 7.5.1 |
| Orphan Handling | Weekly reconciliation with 7-day orphan retention | Section 7.5.1 |
| Capacity Planning | 500GB minimum with 80%/95% alert thresholds | Section 7.5.1 |
| UUID Naming | Prevents filename collisions and directory traversal | Section 7.5 |

#### Observations

**Cleanup Script (Section 7.5.1)**:
```bash
find /data/docling-uploads -type f -mtime +30 -delete
find /data/docling-uploads -type d -empty -delete
```

This is correct but needs error handling and logging enhancements for production use.

**Storage Estimate (Section 7.5.1)**:
- 30 days x 100 files/day x 50 MB avg = ~150 GB
- 500 GB minimum is appropriate with growth headroom

**Temp vs Persistent Storage (Section 8.3)**:
- `/tmp/docling-processing/` for upload staging
- `/data/docling-uploads/` for persistent storage

This separation is operationally sound.

#### Minor Findings

| ID | Finding | Section | Recommendation |
|----|---------|---------|----------------|
| INF-FS-01 | Missing `/data` partition mount verification | 7.5.1 | Add pre-flight check for mount point and permissions |
| INF-FS-02 | No disk monitoring integration specified | 7.5.1 | Define Prometheus node_exporter metrics for disk usage |
| INF-FS-03 | Orphan alert threshold (>100) needs monitoring integration | 7.5.1 | Define how alert is triggered and routed |

---

### 4. Infrastructure Dependencies Review

**Reference**: Section 8.4

#### Server Inventory Assessment

| Server | IP | Port | Status | Assessment |
|--------|-----|------|--------|------------|
| hx-cc-server | 192.168.10.224 | 3000 | Operational | Development server - appropriate |
| hx-docling-mcp-server | 192.168.10.217 | 8000 | Operational | Core dependency - critical |
| hx-postgres-server | 192.168.10.208 | 5432 | Operational | Data persistence - critical |
| hx-redis-server | 192.168.10.209 | 6379 | Operational | Session management - critical |
| hx-litellm-server | 192.168.10.212 | 4000 | Operational | Transitive dependency |
| hx-ollama3-server | 192.168.10.206 | 11434 | Operational | Transitive dependency |

#### Network Connectivity Concerns

| ID | Finding | Recommendation |
|----|---------|----------------|
| INF-NET-01 | No firewall rules specified | Document required UFW/iptables rules for each connection |
| INF-NET-02 | DNS resolution via `.hx.dev.local` assumed | Verify internal DNS is configured on hx-cc-server |
| INF-NET-03 | No latency requirements specified | A-4 assumes <50ms - add monitoring for this |

---

### 5. Database Schema & Indexes Review

**Reference**: Section 7.3, 7.6.1

#### Schema Assessment

**Job Model**: Comprehensive with all necessary fields:
- UUID primary key (good for distributed systems)
- Session tracking via `sessionId`
- Processing state for SSE reconnection (`currentStage`, `currentPercent`, `currentMessage`)
- Error tracking with `errorCode` and `retryCount`
- Proper timestamps with `@updatedAt`

**Result Model**: Clean one-to-many relationship:
- Cascade delete is appropriate
- `@db.Text` for content handles large documents
- Size tracking enables monitoring

#### Index Analysis

```prisma
@@index([sessionId])      // History queries by session
@@index([createdAt])      // Retention cleanup
@@index([status])         // Status filtering
@@index([jobId])          // Result lookups
@@index([format])         // Format filtering (may be underutilized)
```

**Composite Index Recommendation** (Section 7.6.1):
```sql
CREATE INDEX idx_job_session_created ON "Job"("sessionId", "createdAt" DESC);
```

This composite index is documented and appropriate for the pagination query pattern.

#### Major Finding

| ID | Finding | Section | Impact | Recommendation |
|----|---------|---------|--------|----------------|
| INF-DB-01 | No index on `Job.completedAt` | 7.3 | Slow queries for completed job filtering | Add `@@index([completedAt])` to schema |

---

### 6. Health Check Implementation Review

**Reference**: Section 8.5, 8.7

#### Strengths

| Aspect | Assessment | Reference |
|--------|------------|-----------|
| Endpoint Specification | Clear interface with status, checks, latency | Section 8.7 |
| Status Logic | Correct healthy/degraded/unhealthy determination | Section 8.7 |
| Caching | 30-second cache prevents health check storms | Section 8.7 |
| HTTP Status Codes | 503 for unhealthy, 200 otherwise | Section 8.7 |
| Timeout Handling | 5-second timeout with AbortSignal | Section 8.7 |

#### Health Check Matrix

| Service | Check Method | Timeout | Critical |
|---------|--------------|---------|----------|
| MCP | HTTP GET /health | 5s | Yes |
| PostgreSQL | Prisma connection test | 5s | Yes |
| Redis | PING command | 5s | Yes |
| File Storage | (not specified) | - | No |

#### Minor Findings

| ID | Finding | Section | Recommendation |
|----|---------|---------|----------------|
| INF-HC-01 | File storage health check not implemented | 8.7 | Add disk space and write permission checks |
| INF-HC-02 | No health check logging | 8.7 | Log health transitions for operational awareness |
| INF-HC-03 | Check frequency (30s) not configurable | 8.5 | Add environment variable for check interval |

---

### 7. Monitoring & Alerting Review

**Reference**: Section 13.5

#### Metrics Assessment

**Application Metrics**: Well-defined counter and histogram metrics:
- `jobs_total`, `jobs_duration_seconds` - essential for capacity planning
- `uploads_total`, `uploads_size_bytes` - file processing metrics
- `errors_total` - error rate monitoring
- `active_sessions` - usage tracking
- `sse_reconnections_total` - resilience monitoring

**Health Metrics**: Appropriate operational metrics:
- `health_check_duration_seconds` - latency monitoring
- `health_check_status` - availability tracking
- `db_connections_active` - connection pool monitoring
- `disk_usage_bytes` - capacity monitoring

#### Alert Rules Assessment

| Condition | Severity | Assessment |
|-----------|----------|------------|
| MCP unavailable > 1 minute | Critical | Appropriate - core dependency |
| Error rate > 5% (5 min) | High | Good threshold for early warning |
| Processing time > 5 minutes | Medium | Appropriate given 300s timeout |
| Disk usage > 80% | High | Good early warning threshold |
| Disk usage > 95% | Critical | Appropriate with upload rejection |
| Rate limit hits > 50/hour | Low | Good for abuse detection |

#### Major Finding

| ID | Finding | Section | Impact | Recommendation |
|----|---------|---------|--------|----------------|
| INF-MON-01 | No Prometheus scrape endpoint specified | 13.5 | Cannot integrate with monitoring stack | Add `/api/metrics` endpoint implementation details |

#### Minor Findings

| ID | Finding | Section | Recommendation |
|----|---------|---------|----------------|
| INF-MON-02 | Alerting destination not specified | 13.5 | Define Alertmanager webhook or Slack integration |
| INF-MON-03 | No Grafana dashboard specification | 13.5 | Add dashboard JSON or reference |

---

### 8. Operational Concerns Review

#### Credential Management

**Reference**: Section 8.3

The charter uses environment variable substitution (`${DB_PASSWORD}`) which is appropriate. However:

| ID | Finding | Recommendation |
|----|---------|----------------|
| INF-CRED-01 | No Ansible Vault integration documented | Add credential retrieval procedure for DB_PASSWORD |
| INF-CRED-02 | .env.local in .gitignore but no .env.example | Create .env.example with placeholder values |

#### Backup Strategy

**Reference**: Section 7.6, 12.5

| Backup Type | Specified | Assessment |
|-------------|-----------|------------|
| PostgreSQL backup | Not specified | MISSING - Critical gap |
| File backup | Not specified | MISSING - Should backup /data/docling-uploads |
| Redis backup | Not specified | Acceptable - session data is transient |

| ID | Finding | Impact | Recommendation |
|----|---------|--------|----------------|
| INF-BK-01 | No PostgreSQL backup procedure | Data loss risk | Add pg_dump script with systemd timer |
| INF-BK-02 | No file backup procedure | Document loss risk | Add rsync or restic backup for /data/docling-uploads |

#### Disaster Recovery

No explicit disaster recovery runbook is specified in the charter. For Phase 1 development, this is acceptable, but should be documented.

| ID | Finding | Recommendation |
|----|---------|----------------|
| INF-DR-01 | No DR runbook referenced | Create basic DR runbook for Phase 1 completion |

---

## Findings Summary

### Critical Findings (0)

No critical infrastructure findings. The charter adequately addresses core infrastructure requirements.

### Major Findings (3)

| ID | Finding | Section | Recommendation |
|----|---------|---------|----------------|
| INF-DB-01 | Missing index on `Job.completedAt` | 7.3 | Add index for completed job queries |
| INF-MON-01 | No Prometheus scrape endpoint implementation | 13.5 | Add `/api/metrics` endpoint |
| INF-BK-01 | No PostgreSQL backup procedure | 7.6 | Add pg_dump backup with systemd timer |

### Minor Findings (12)

| ID | Category | Finding |
|----|----------|---------|
| INF-PG-01 | PostgreSQL | Missing connection pool size |
| INF-PG-02 | PostgreSQL | No backup procedure |
| INF-PG-03 | PostgreSQL | Test DB isolation strategy |
| INF-RD-01 | Redis | No authentication documented |
| INF-RD-02 | Redis | Persistence mode not specified |
| INF-RD-03 | Redis | No connection retry strategy |
| INF-FS-01 | File Storage | Missing mount verification |
| INF-FS-02 | File Storage | No disk monitoring integration |
| INF-FS-03 | File Storage | Orphan alert routing undefined |
| INF-HC-01 | Health Check | File storage check not implemented |
| INF-HC-02 | Health Check | No health transition logging |
| INF-MON-02 | Monitoring | Alert destination not specified |

---

## Infrastructure Pre-Deployment Checklist

Before Phase 1 deployment, verify:

### Database

- [ ] hx-postgres-server accessible from hx-cc-server
- [ ] `docling_user` role created with appropriate permissions
- [ ] `docling_db` database created
- [ ] Connection pool size configured in Prisma
- [ ] pg_dump backup script created and tested

### Redis

- [ ] hx-redis-server accessible from hx-cc-server
- [ ] Database 0 available for session storage
- [ ] Persistence mode verified (AOF recommended)
- [ ] ioredis retry configuration documented

### File Storage

- [ ] `/data/docling-uploads/` directory created with correct permissions
- [ ] `docling-app` user and `docling-files` group created
- [ ] Cron job for cleanup installed and tested
- [ ] Disk usage monitoring configured

### Network

- [ ] Internal DNS resolution working for `.hx.dev.local`
- [ ] Firewall rules allow required connections
- [ ] Network latency verified (<50ms)

### Monitoring

- [ ] Prometheus scrape target configured
- [ ] Grafana dashboard created
- [ ] Alertmanager rules deployed
- [ ] Alert notification channel configured

---

## Suggested Charter Amendments

### Amendment 1: Add PostgreSQL Backup Procedure

Add to Section 7.6:

```markdown
#### 7.6.2 Database Backup Procedure

**Backup Script (`scripts/backup-database.sh`):**
```bash
#!/bin/bash
# PostgreSQL backup for docling_db
BACKUP_DIR=/var/backups/docling
RETENTION_DAYS=30
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR
pg_dump -h hx-postgres-server.hx.dev.local -U docling_user -d docling_db \
  | gzip > $BACKUP_DIR/docling_db_$TIMESTAMP.sql.gz

# Cleanup old backups
find $BACKUP_DIR -type f -mtime +$RETENTION_DAYS -delete

echo "$(date -Iseconds) Backup completed: docling_db_$TIMESTAMP.sql.gz"
```

**Systemd Timer:**
```ini
# /etc/systemd/system/docling-backup.timer
[Unit]
Description=Daily Docling database backup

[Timer]
OnCalendar=*-*-* 01:00:00
Persistent=true

[Install]
WantedBy=timers.target
```
```

### Amendment 2: Add Prometheus Metrics Endpoint

Add to Section 8.7:

```markdown
#### 8.7.1 Prometheus Metrics Endpoint

**Endpoint**: `GET /api/metrics`

**Format**: Prometheus text exposition format

**Implementation**: Use `prom-client` library with default collectors plus custom metrics defined in Section 13.5.
```

### Amendment 3: Add Index to Schema

Update Section 7.3 Prisma schema:

```prisma
@@index([sessionId])
@@index([createdAt])
@@index([status])
@@index([completedAt])  // ADD THIS
```

---

## Verdict

### APPROVED WITH CONDITIONS

The hx-docling-ui charter demonstrates comprehensive infrastructure planning with well-defined database schemas, file storage architecture, health checks, and monitoring specifications. The charter is ready for Phase 1 development with the following conditions:

**Conditions for Approval**:

1. **Before Sprint 1.1**: Add PostgreSQL backup procedure (INF-BK-01)
2. **Before Sprint 1.2**: Verify database index on `completedAt` (INF-DB-01)
3. **Before Sprint 1.8**: Implement `/api/metrics` Prometheus endpoint (INF-MON-01)

**Recommendations for Phase 2 Charter**:

1. Define comprehensive disaster recovery runbook
2. Specify Redis authentication for production
3. Add detailed alerting integration (Alertmanager/Slack)
4. Include database connection pooling configuration

---

## Sign-Off

**Reviewer**: William Chen
**Role**: Infrastructure & Operations Specialist
**Invocation**: @william
**Date**: December 11, 2025
**Verdict**: APPROVED WITH CONDITIONS

---

*This review focuses on infrastructure and operational concerns. For security review, see Frank Lucas (@frank). For architecture review, see Alex Rivera (@alex). For testing review, see Julia Santos (@julia).*
