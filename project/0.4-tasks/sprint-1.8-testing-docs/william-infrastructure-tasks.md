# Infrastructure Tasks: Sprint 1.8 - Operational Documentation

**Sprint**: 1.8 - Testing & Documentation
**Lead**: Julia Santos (`@julia`)
**Support**: Neo (`@neo`), Trinity (`@trinity`), William Chen (`@william`)
**Review**: Agent Zero (`@agent-zero`)
**Document Version**: 1.0.0
**Created**: 2025-12-12
**Reference**: `project/0.1-plan/0.1.1-implementation-plan.md` Section 4.8

---

## Overview

Sprint 1.8 focuses on testing and documentation. William's responsibilities center on operational documentation, including CLAUDE.md updates with infrastructure patterns, runbooks, and troubleshooting guides.

**William's Tasks Effort**: 2.5 hours (of 6.0 hour sprint total)

---

## Task WIL-1.8-001: CLAUDE.md Redis Key Patterns Documentation

### Description

Document all Redis key patterns used by the application in CLAUDE.md. This enables developers to understand Redis data organization and troubleshoot issues.

### Acceptance Criteria

- [ ] All Redis key patterns documented
- [ ] TTL values documented for each key type
- [ ] Purpose and structure explained
- [ ] Example values provided
- [ ] Cleanup/expiration behavior documented

### Dependencies

- Sprint 1.2 Redis infrastructure complete
- Sprint 1.5b SSE infrastructure complete

### Effort Estimate

**Duration**: 15 minutes
**Complexity**: Low

### Deliverables

1. Redis key patterns section in CLAUDE.md

### Technical Notes

```markdown
## Redis Key Patterns

### Session Keys
- Pattern: `session:{uuid}`
- TTL: 24 hours (86400 seconds)
- Structure: JSON object with session data
- Example: `session:550e8400-e29b-41d4-a716-446655440000`

### Rate Limiting Keys
- Pattern: `ratelimit:{sessionId}:{endpoint}`
- TTL: Dynamic (window size)
- Structure: Sorted set with timestamps as scores
- Example: `ratelimit:550e8400:process`

### SSE Event Buffer Keys
- Pattern: `sse:events:{jobId}`
- TTL: 5 minutes (300 seconds)
- Structure: Sorted set with eventId as score
- Example: `sse:events:123e4567-e89b-12d3-a456-426614174000`

### Circuit Breaker State (if persisted)
- Pattern: `circuit:{serviceName}`
- TTL: None (manual management)
- Structure: JSON with state and failure count
```

---

## Task WIL-1.8-002: CLAUDE.md SSE Reconnection Sequence Documentation

### Description

Document the SSE reconnection sequence in CLAUDE.md, including client behavior, server handling, and state synchronization flow.

### Acceptance Criteria

- [ ] Client reconnection flow documented
- [ ] Server event replay documented
- [ ] Last-Event-ID handling explained
- [ ] State sync event format documented
- [ ] Fallback to polling documented

### Dependencies

- Sprint 1.5b SSE infrastructure complete

### Effort Estimate

**Duration**: 20 minutes
**Complexity**: Medium

### Deliverables

1. SSE reconnection section in CLAUDE.md
2. Sequence diagram

### Technical Notes

```markdown
## SSE Reconnection Sequence

### Client Reconnection Flow
1. EventSource detects connection loss (onerror event)
2. ReconnectionManager calculates next delay (exponential backoff)
3. Client waits for calculated delay
4. Client creates new EventSource with Last-Event-ID header
5. Server processes reconnection request

### Server Event Replay
1. Server receives request with Last-Event-ID
2. Server queries EventBuffer for events since ID
3. Server replays missed events in order
4. Server sends 'sync' event with current state
5. Normal streaming resumes

### Reconnection Timing
- Base delay: 1 second
- Max delay: 30 seconds
- Multiplier: 2x
- Max retries: 10
- Jitter: +/- 10%

### Fallback Behavior
After 10 failed reconnection attempts:
1. SSE connection abandoned
2. Polling fallback activated (2 second interval)
3. UI shows "Using backup connection" indicator
```

---

## Task WIL-1.8-003: CLAUDE.md MCP Error Code Mapping Documentation

### Description

Document the mapping between JSON-RPC error codes from MCP server and application E2xx error codes in CLAUDE.md.

### Acceptance Criteria

- [ ] All JSON-RPC error codes listed
- [ ] Corresponding E2xx codes documented
- [ ] User-friendly messages defined
- [ ] Recovery actions specified
- [ ] Error handling flow explained

### Dependencies

- Sprint 1.5a MCP client complete

### Effort Estimate

**Duration**: 15 minutes
**Complexity**: Low

### Deliverables

1. MCP error mapping section in CLAUDE.md

### Technical Notes

```markdown
## MCP Error Code Mapping

### JSON-RPC to Application Error Mapping

| JSON-RPC Code | JSON-RPC Message | App Code | User Message |
|---------------|------------------|----------|--------------|
| -32700 | Parse error | E200 | Server communication error |
| -32600 | Invalid request | E201 | Invalid request format |
| -32601 | Method not found | E202 | Unsupported operation |
| -32602 | Invalid params | E203 | Invalid parameters |
| -32603 | Internal error | E204 | Server processing error |
| -32000 to -32099 | Server errors | E205 | Document processing failed |

### Error Recovery Actions

| App Code | Recovery Action |
|----------|-----------------|
| E200 | Retry after 5 seconds |
| E201 | Check request format |
| E202 | Verify MCP server version |
| E203 | Validate input parameters |
| E204 | Contact support if persists |
| E205 | Retry with different document |

### Error Handling Flow
1. MCP client receives JSON-RPC error
2. Error mapped to application code via error-mapping.ts
3. Structured AppError created with user message
4. Error logged with original JSON-RPC details
5. User shown friendly error with recovery action
```

---

## Task WIL-1.8-004: CLAUDE.md Circuit Breaker State Transitions Documentation

### Description

Document the circuit breaker state machine and transitions in CLAUDE.md, including configuration and monitoring.

### Acceptance Criteria

- [ ] State machine diagram included
- [ ] Transition conditions documented
- [ ] Configuration parameters explained
- [ ] Monitoring/alerting recommendations
- [ ] Fallback behavior documented

### Dependencies

- Sprint 1.2 circuit breaker implementation complete

### Effort Estimate

**Duration**: 15 minutes
**Complexity**: Low

### Deliverables

1. Circuit breaker section in CLAUDE.md
2. State diagram

### Technical Notes

```markdown
## Circuit Breaker Pattern

### States
- **CLOSED**: Normal operation, requests flow through
- **OPEN**: Requests blocked, fallback used
- **HALF_OPEN**: Testing if service recovered

### State Transitions

```
        5 failures
CLOSED ──────────────> OPEN
   ^                     │
   │                     │ 30s timeout
   │    success          v
   └─────────────── HALF_OPEN
                         │
                    failure
                         └──> OPEN
```

### Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| failureThreshold | 5 | Failures before opening |
| resetTimeout | 30000ms | Time before half-open |
| monitoringPeriod | 60000ms | Window for failure count |

### Monitoring Recommendations
- Alert when circuit opens (state = OPEN)
- Track failure rate per service
- Monitor recovery success rate
- Log all state transitions

### Fallback Behavior
- Redis: Return null/empty, use in-memory cache
- MCP: Queue request, show degraded mode
- Database: Return error, no fallback available
```

---

## Task WIL-1.8-005: CLAUDE.md Progress Interpolation Algorithm Documentation

### Description

Document the progress interpolation algorithm in CLAUDE.md, including the monotonic guarantee and configuration options.

### Acceptance Criteria

- [ ] Algorithm explained clearly
- [ ] Monotonic guarantee documented
- [ ] Configuration parameters listed
- [ ] Edge cases documented
- [ ] Performance considerations noted

### Dependencies

- Sprint 1.5b progress interpolation complete

### Effort Estimate

**Duration**: 10 minutes
**Complexity**: Low

### Deliverables

1. Progress interpolation section in CLAUDE.md

### Technical Notes

```markdown
## Progress Interpolation

### Purpose
Smooths progress updates for better UX while ensuring progress never decreases.

### Algorithm
1. Server sends actual progress updates (may be sparse)
2. Client interpolates between updates every 100ms
3. Displayed progress increases by max 1% per interval
4. Progress pauses if no server updates for 5 seconds
5. On completion, immediately jumps to 100%

### Monotonic Guarantee
- Progress value is NEVER allowed to decrease
- Non-monotonic updates from server are ignored
- Ensures consistent user experience

### Configuration

| Parameter | Value | Description |
|-----------|-------|-------------|
| INTERPOLATION_INTERVAL | 100ms | Update frequency |
| MAX_RATE | 1% | Max increase per interval |
| STALE_THRESHOLD | 5000ms | Pause if no updates |

### Edge Cases
- Large jumps: If actual > displayed + 10%, catch up quickly
- Reconnection: Resume from current displayed value
- Completion: Immediate jump to 100%
```

---

## Task WIL-1.8-006: CLAUDE.md Checkpoint Serialization Format Documentation

### Description

Document the checkpoint serialization format in CLAUDE.md, including schema, size limits, and versioning.

### Acceptance Criteria

- [ ] JSON schema documented
- [ ] Size limits explained
- [ ] Version migration strategy noted
- [ ] Example checkpoint provided
- [ ] Cleanup behavior documented

### Dependencies

- Sprint 1.5b checkpoint manager complete

### Effort Estimate

**Duration**: 10 minutes
**Complexity**: Low

### Deliverables

1. Checkpoint format section in CLAUDE.md

### Technical Notes

```markdown
## Checkpoint Serialization Format

### Schema (Version 1)
```json
{
  "version": 1,
  "stage": "conversion",
  "progress": 65,
  "startedAt": 1702345678000,
  "lastUpdatedAt": 1702345690000,
  "partialResults": {
    "markdown": "# Document Title\n...",
    "html": null,
    "json": null
  },
  "metadata": {
    "mcpRequestId": "550e8400-e29b-41d4-a716-446655440000",
    "fileHash": "sha256:abc123..."
  }
}
```

### Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| version | number | Yes | Schema version for migration |
| stage | enum | Yes | Current processing stage |
| progress | number | Yes | Progress percentage (0-100) |
| startedAt | number | Yes | Job start timestamp (ms) |
| lastUpdatedAt | number | Yes | Last update timestamp (ms) |
| partialResults | object | No | Completed export outputs |
| metadata | object | No | Additional context |

### Size Limit
- Maximum checkpoint size: 1MB
- Exceeding limit throws error
- Large partial results should be stored separately

### Version Migration
- Always include version field
- Check version on restore
- Implement migration function for upgrades

### Cleanup
- Cleared on successful job completion
- Preserved on failure for resume capability
- Manual cleanup via CheckpointManager.clear()
```

---

## Task WIL-1.8-007: Operational Troubleshooting Guide

### Description

Create a troubleshooting guide in CLAUDE.md covering common operational issues and resolution steps.

### Acceptance Criteria

- [ ] Database connection issues covered
- [ ] Redis connection issues covered
- [ ] SSE connection issues covered
- [ ] MCP server issues covered
- [ ] Performance issues covered
- [ ] Log locations documented

### Dependencies

- All sprints complete

### Effort Estimate

**Duration**: 30 minutes
**Complexity**: Medium

### Deliverables

1. Troubleshooting section in CLAUDE.md

### Technical Notes

```markdown
## Troubleshooting Guide

### Database Connection Issues

**Symptom**: Health check shows database down
**Checks**:
1. Verify PostgreSQL is running: `systemctl status postgresql`
2. Check connection: `psql -h hx-postgres-server -U docling_app -d docling_db`
3. Verify SSL certificate: `openssl x509 -in /etc/ssl/certs/hx-postgres-ca.crt -noout -dates`
4. Check PgBouncer: `psql -p 6432 -c 'SHOW POOLS;'`

**Resolution**:
- Restart PostgreSQL if down
- Renew certificates if expired
- Check connection limits

### Redis Connection Issues

**Symptom**: Sessions not persisting, rate limiting not working
**Checks**:
1. Redis ping: `redis-cli -h hx-redis-server PING`
2. TLS verification: Check certificate paths in environment
3. Circuit breaker state: Check logs for "Circuit breaker OPEN"
4. Memory usage: `redis-cli INFO memory`

**Resolution**:
- Restart Redis if down
- Check/renew TLS certificates
- Increase memory if evictions occurring

### SSE Connection Issues

**Symptom**: Progress not updating, reconnection loop
**Checks**:
1. Nginx buffering disabled (X-Accel-Buffering header)
2. Proxy timeouts configured adequately
3. Client console for EventSource errors
4. Server logs for SSE handler errors

**Resolution**:
- Add `X-Accel-Buffering: no` header
- Increase proxy read timeout
- Check for firewall issues

### MCP Server Issues

**Symptom**: Processing fails, E2xx errors
**Checks**:
1. MCP health: `curl http://hx-docling-server:8000/health`
2. Check server logs for errors
3. Verify tool availability: Check tools/list response
4. Test with simple document

**Resolution**:
- Restart MCP server
- Check disk space for temporary files
- Review recent configuration changes

### Log Locations

| Component | Log Location |
|-----------|--------------|
| Application | `stdout/stderr` (systemd journal) |
| PostgreSQL | `/var/log/postgresql/` |
| Redis | `/var/log/redis/` |
| Nginx | `/var/log/nginx/` |
```

---

## Task WIL-1.8-008: Infrastructure Architecture Section for CLAUDE.md

### Description

Add comprehensive infrastructure architecture documentation to CLAUDE.md covering server topology, service dependencies, and configuration.

### Acceptance Criteria

- [ ] Server topology documented
- [ ] Service dependencies mapped
- [ ] Port assignments listed
- [ ] Environment variables documented
- [ ] Network diagram included

### Dependencies

- All sprints complete

### Effort Estimate

**Duration**: 25 minutes
**Complexity**: Medium

### Deliverables

1. Infrastructure architecture section in CLAUDE.md

### Technical Notes

```markdown
## Infrastructure Architecture

### Server Topology

| Server | Role | IP/Hostname |
|--------|------|-------------|
| hx-cc-server | Application host | 192.168.10.224 |
| hx-postgres-server | PostgreSQL + PgBouncer | Internal DNS |
| hx-redis-server | Redis cache/session | Internal DNS |
| hx-docling-server | MCP document processing | Internal DNS |

### Service Dependencies

```
User Browser
    │
    └── hx-cc-server:3000 (Next.js Application)
            │
            ├── hx-postgres-server:5432 (Direct) / :6432 (Pooled)
            │   └── PostgreSQL 15 + PgBouncer
            │
            ├── hx-redis-server:6379 (TLS)
            │   └── Redis 7.x
            │
            └── hx-docling-server:8000 (HTTP)
                └── Docling MCP Server
```

### Port Assignments

| Port | Service | Protocol |
|------|---------|----------|
| 3000 | Next.js Application | HTTP |
| 5432 | PostgreSQL Direct | TCP (TLS) |
| 6432 | PgBouncer Pool | TCP (TLS) |
| 6379 | Redis | TCP (TLS) |
| 8000 | MCP Server | HTTP |

### Required Environment Variables

```bash
# Application
NODE_ENV=production
APP_VERSION=1.0.0

# Database
DATABASE_URL=postgresql://...?sslmode=require
DATABASE_POOL_URL=postgresql://...:6432/...

# Redis
REDIS_HOST=hx-redis-server
REDIS_PORT=6379
REDIS_TLS_ENABLED=true
REDIS_TLS_CA_PATH=/etc/ssl/certs/hx-redis-ca.crt

# MCP
MCP_SERVER_URL=http://hx-docling-server:8000

# Storage
UPLOAD_DIR=/data/docling-uploads
MAX_FILE_SIZE_MB=100
```
```

---

## Task WIL-1.8-009: Health Monitoring and Alerting Recommendations

### Description

Document health monitoring and alerting recommendations in CLAUDE.md for production operations.

### Acceptance Criteria

- [ ] Key metrics to monitor listed
- [ ] Alert thresholds recommended
- [ ] Prometheus configuration examples
- [ ] Grafana dashboard recommendations
- [ ] Runbook reference for alerts

### Dependencies

- Sprint 1.2 health check complete

### Effort Estimate

**Duration**: 20 minutes
**Complexity**: Medium

### Deliverables

1. Monitoring section in CLAUDE.md

### Technical Notes

```markdown
## Health Monitoring & Alerting

### Key Metrics

| Metric | Source | Alert Threshold |
|--------|--------|-----------------|
| Health endpoint status | /api/v1/health | != healthy for 1 min |
| Database latency | Health check | > 100ms avg |
| Redis latency | Health check | > 50ms avg |
| Redis memory % | Health check | > 80% |
| MCP server status | Health check | down for 30s |
| Error rate | Application logs | > 5% of requests |
| SSE connections | Application metrics | > 1000 concurrent |

### Prometheus Configuration

```yaml
scrape_configs:
  - job_name: 'docling-ui'
    scrape_interval: 15s
    static_configs:
      - targets: ['hx-cc-server:3000']
    metrics_path: '/api/v1/metrics'
```

### Alerting Rules

```yaml
groups:
  - name: docling-ui
    rules:
      - alert: DoclingHealthDegraded
        expr: docling_health_status != 1
        for: 1m
        labels:
          severity: warning
      - alert: DoclingDatabaseDown
        expr: docling_db_status == 0
        for: 30s
        labels:
          severity: critical
      - alert: DoclingRedisHighMemory
        expr: docling_redis_memory_percent > 80
        for: 5m
        labels:
          severity: warning
```

### Grafana Dashboard Panels

1. **Health Overview**: Status of all services
2. **Latency**: Database, Redis, MCP response times
3. **Error Rate**: Errors per endpoint over time
4. **Active Jobs**: Processing jobs count
5. **SSE Connections**: Active connection count
6. **Redis Memory**: Memory usage over time
```

---

## Task WIL-1.8-010: Backup and Recovery Procedures

### Description

Document backup and recovery procedures in CLAUDE.md for disaster recovery scenarios.

### Acceptance Criteria

- [ ] Database backup procedure documented
- [ ] File storage backup documented
- [ ] Recovery procedures step-by-step
- [ ] RTO/RPO targets defined
- [ ] Backup verification process documented

### Dependencies

- All sprints complete

### Effort Estimate

**Duration**: 20 minutes
**Complexity**: Medium

### Deliverables

1. Backup and recovery section in CLAUDE.md

### Technical Notes

```markdown
## Backup & Recovery

### Backup Targets

| Component | Backup Method | Frequency | Retention |
|-----------|---------------|-----------|-----------|
| PostgreSQL | pg_dump | Daily | 30 days |
| Uploaded Files | rsync to backup storage | Daily | 90 days |
| Redis | Not backed up (cache) | N/A | N/A |

### Database Backup Procedure

```bash
#!/bin/bash
# /opt/scripts/backup-docling-db.sh

BACKUP_DIR="/data/backups/docling"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/docling_db_${TIMESTAMP}.sql.gz"

# Create backup
pg_dump -h hx-postgres-server -U docling_app docling_db | gzip > ${BACKUP_FILE}

# Verify backup integrity
gunzip -t ${BACKUP_FILE} || exit 1

# Retain 30 days
find ${BACKUP_DIR} -name "*.sql.gz" -mtime +30 -delete

echo "Backup complete: ${BACKUP_FILE}"
```

### Recovery Procedure

1. **Stop Application**
   ```bash
   systemctl stop docling-ui
   ```

2. **Restore Database**
   ```bash
   gunzip -c ${BACKUP_FILE} | psql -h hx-postgres-server -U postgres -d docling_db
   ```

3. **Verify Data**
   ```sql
   SELECT COUNT(*) FROM "Job";
   ```

4. **Restart Application**
   ```bash
   systemctl start docling-ui
   ```

### RTO/RPO Targets

| Metric | Target | Notes |
|--------|--------|-------|
| RPO | 24 hours | Daily backups |
| RTO | 1 hour | Database restore + verification |

### Backup Verification

- Monthly: Restore to test environment
- Quarterly: Full disaster recovery drill
- Document results in `project/0.6-governance/dr-tests/`
```

---

## Sprint Summary

| Task ID | Title | Effort | Dependencies |
|---------|-------|--------|--------------|
| WIL-1.8-001 | CLAUDE.md Redis Key Patterns Documentation | 15m | Sprint 1.2, 1.5b |
| WIL-1.8-002 | CLAUDE.md SSE Reconnection Sequence Documentation | 20m | Sprint 1.5b |
| WIL-1.8-003 | CLAUDE.md MCP Error Code Mapping Documentation | 15m | Sprint 1.5a |
| WIL-1.8-004 | CLAUDE.md Circuit Breaker State Transitions Documentation | 15m | Sprint 1.2 |
| WIL-1.8-005 | CLAUDE.md Progress Interpolation Algorithm Documentation | 10m | Sprint 1.5b |
| WIL-1.8-006 | CLAUDE.md Checkpoint Serialization Format Documentation | 10m | Sprint 1.5b |
| WIL-1.8-007 | Operational Troubleshooting Guide | 30m | All sprints |
| WIL-1.8-008 | Infrastructure Architecture Section for CLAUDE.md | 25m | All sprints |
| WIL-1.8-009 | Health Monitoring and Alerting Recommendations | 20m | Sprint 1.2 |
| WIL-1.8-010 | Backup and Recovery Procedures | 20m | All sprints |

**Total Tasks**: 10
**Total Effort**: 3 hours 0 minutes

---

## Risk Considerations

| Risk | Mitigation |
|------|------------|
| Documentation becomes outdated | Include in PR review checklist |
| Missing edge cases | Cross-reference with testing results |
| Incomplete coverage | Use implementation code as source of truth |
| Runbook accuracy | Test procedures during DR drills |
