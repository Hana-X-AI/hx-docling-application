# Infrastructure Tasks: Sprint 0 - Prerequisites

**Sprint**: 0 - Prerequisites Checklist
**Lead**: William Chen (`@william`)
**Support**: Trinity (`@trinity`)
**Validation**: Alex (`@alex`)
**Document Version**: 1.0.0
**Created**: 2025-12-12
**Reference**: `project/0.1-plan/0.1.1-implementation-plan.md` Section 4.0

---

## Overview

Sprint 0 validates all infrastructure prerequisites before development begins. This sprint ensures database, Redis, storage, and connectivity are ready for Sprint 1.1.

---

## Task WIL-0-001: PostgreSQL Database User Configuration

### Description

Create and configure the `docling_app` PostgreSQL user with appropriate permissions for the HX Docling UI Application. This includes CRUD permissions on the `docling_db` database.

### Acceptance Criteria

- [ ] PostgreSQL user `docling_app` created
- [ ] User has SELECT, INSERT, UPDATE, DELETE permissions on all tables in `docling_db`
- [ ] User can create indexes but NOT drop tables
- [ ] Connection verified via: `psql -U docling_app -d docling_db -c '\conninfo'`
- [ ] User roles verified via: `psql -c '\du docling_app'`
- [ ] Password stored in Ansible Vault (NOT in plain text)

### Dependencies

- PostgreSQL server accessible at hx-postgres-server
- Ansible Vault configured for credential storage

### Effort Estimate

**Duration**: 30 minutes
**Complexity**: Low

### Deliverables

1. Database user creation script (bash)
2. Role verification output
3. Credential entry in Ansible Vault

### Technical Notes

```bash
# Validation Commands
psql -U docling_app -d docling_db -c '\conninfo'
psql -c '\du docling_app'

# Expected privileges: SELECT, INSERT, UPDATE, DELETE on all tables
# User should NOT have SUPERUSER or CREATEDB privileges
```

---

## Task WIL-0-002: SSL Certificate Provisioning for PostgreSQL

### Description

Provision and configure SSL certificates for PostgreSQL connections. Ensure certificates are available at standard paths and database connection string includes `sslmode=require`.

### Acceptance Criteria

- [ ] SSL certificate exists at `/etc/ssl/certs/hx-postgres-ca.crt`
- [ ] Certificate is valid (not expired)
- [ ] Certificate permissions are 644 (readable by application)
- [ ] Database URL includes `sslmode=require` parameter
- [ ] SSL connection verified via query

### Dependencies

- Certificate Authority (CA) infrastructure
- PostgreSQL server configured for SSL

### Effort Estimate

**Duration**: 45 minutes
**Complexity**: Medium

### Deliverables

1. SSL certificate file at correct path
2. `.env.example` template with correct DATABASE_URL format
3. SSL connection verification log

### Technical Notes

```bash
# Certificate verification
ls -la /etc/ssl/certs/hx-postgres*
openssl x509 -in /etc/ssl/certs/hx-postgres-ca.crt -noout -dates

# Connection string format
DATABASE_URL="postgresql://docling_app:PASSWORD@hx-postgres-server:5432/docling_db?sslmode=require&sslrootcert=/etc/ssl/certs/hx-postgres-ca.crt"

# SSL verification via psql
psql "host=hx-postgres-server dbname=docling_db user=docling_app sslmode=require" -c "SELECT ssl_is_used();"
```

---

## Task WIL-0-003: Database Creation and Schema Preparation

### Description

Create the `docling_db` database and prepare it for Prisma migrations. Verify database is accessible and ready for schema deployment.

### Acceptance Criteria

- [ ] Database `docling_db` exists
- [ ] Database listed in `psql -c '\l'` output
- [ ] Database encoding is UTF8
- [ ] Database owner has necessary privileges for Prisma migrations
- [ ] Connection from hx-cc-server verified

### Dependencies

- Task WIL-0-001 completed
- PostgreSQL server accessible

### Effort Estimate

**Duration**: 20 minutes
**Complexity**: Low

### Deliverables

1. Database creation verification log
2. Connection test results from development server

### Technical Notes

```bash
# Database creation (if not exists)
createdb -U postgres docling_db

# Verification
psql -c '\l' | grep docling_db

# Remote connection test from hx-cc-server
psql -h hx-postgres-server -U docling_app -d docling_db -c 'SELECT version();'
```

---

## Task WIL-0-004: Data Partition Validation and Configuration

### Description

Validate the `/data` partition exists with sufficient capacity for document storage. Configure directory structure and permissions for the application.

### Acceptance Criteria

- [ ] `/data` partition is mounted
- [ ] `/data` partition has >= 500GB free space
- [ ] `/data/docling-uploads` directory exists
- [ ] Directory writable by application user
- [ ] Proper permissions (750) set on upload directory

### Dependencies

- Storage infrastructure provisioned
- Application user account exists

### Effort Estimate

**Duration**: 30 minutes
**Complexity**: Low

### Deliverables

1. Partition validation script
2. Directory structure creation commands
3. Permissions verification log

### Technical Notes

```bash
# Partition validation
df -h /data
# Expected: >= 500GB available

# Directory structure
mkdir -p /data/docling-uploads
chown app-user:app-group /data/docling-uploads
chmod 750 /data/docling-uploads

# Verification
ls -la /data/docling-uploads
touch /data/docling-uploads/test-write && rm /data/docling-uploads/test-write
```

---

## Task WIL-0-005: Redis Server Connectivity and TLS Configuration

### Description

Validate Redis server accessibility with TLS encryption. Ensure certificates are provisioned and connection is secure.

### Acceptance Criteria

- [ ] Redis server responds to PING at `hx-redis-server`
- [ ] Redis TLS certificates exist at `/etc/ssl/certs/hx-redis*`
- [ ] TLS connection verified via redis-cli
- [ ] Connection latency < 5ms from hx-cc-server
- [ ] Redis version >= 6.x confirmed

### Dependencies

- Redis server deployed
- TLS certificates issued

### Effort Estimate

**Duration**: 45 minutes
**Complexity**: Medium

### Deliverables

1. Redis connectivity verification script
2. TLS certificate validation log
3. Latency benchmark results

### Technical Notes

```bash
# Basic connectivity
redis-cli -h hx-redis-server PING
# Expected: PONG

# TLS certificate check
ls -la /etc/ssl/certs/hx-redis*

# TLS connection test
redis-cli -h hx-redis-server --tls \
  --cacert /etc/ssl/certs/hx-redis-ca.crt \
  --cert /etc/ssl/certs/hx-redis-client.crt \
  --key /etc/ssl/private/hx-redis-client.key \
  PING

# Version check
redis-cli -h hx-redis-server INFO server | grep redis_version

# Latency benchmark
redis-cli -h hx-redis-server --latency -c 100
```

---

## Task WIL-0-006: PgBouncer Connection Pooling Validation

### Description

Validate PgBouncer is accessible and properly configured for connection pooling to PostgreSQL.

### Acceptance Criteria

- [ ] PgBouncer accessible at `hx-postgres-server:6432`
- [ ] Connection via PgBouncer returns valid data
- [ ] Pool settings verified (max_client_conn, default_pool_size)
- [ ] Transaction pooling mode confirmed

### Dependencies

- PgBouncer deployed and configured
- Task WIL-0-001 completed

### Effort Estimate

**Duration**: 30 minutes
**Complexity**: Medium

### Deliverables

1. PgBouncer connectivity test script
2. Pool configuration verification log

### Technical Notes

```bash
# Connection via PgBouncer
psql -h hx-postgres-server -p 6432 -U docling_app -d docling_db -c '\conninfo'

# Check PgBouncer stats
psql -h hx-postgres-server -p 6432 -U docling_app -d pgbouncer -c 'SHOW POOLS;'
psql -h hx-postgres-server -p 6432 -U docling_app -d pgbouncer -c 'SHOW CONFIG;'
```

---

## Task WIL-0-007: MCP Server Health Check Validation

### Description

Validate the hx-docling-mcp-server is accessible and responding to health checks from the development environment.

### Acceptance Criteria

- [ ] MCP server health endpoint returns 200 OK
- [ ] Health check includes version information
- [ ] Response time < 500ms
- [ ] Server accessible from hx-cc-server network

### Dependencies

- MCP server deployed and running
- Network connectivity between servers

### Effort Estimate

**Duration**: 20 minutes
**Complexity**: Low

### Deliverables

1. Health check verification script
2. Response time measurements

### Technical Notes

```bash
# Health check
curl -v http://hx-docling-server:8000/health

# Response time measurement
curl -o /dev/null -s -w 'Total: %{time_total}s\n' http://hx-docling-server:8000/health

# Network connectivity
nc -zv hx-docling-server 8000
```

---

## Task WIL-0-008: Environment Configuration Template

### Description

Create comprehensive `.env.example` template with all required environment variables for Sprint 1.1 and beyond.

### Acceptance Criteria

- [ ] All database connection variables documented
- [ ] All Redis connection variables documented
- [ ] All MCP server variables documented
- [ ] File path variables documented
- [ ] SSL certificate paths documented
- [ ] No actual credentials in template (placeholders only)
- [ ] Comments explain each variable's purpose

### Dependencies

- Tasks WIL-0-001 through WIL-0-007 completed
- Understanding of all service endpoints

### Effort Estimate

**Duration**: 30 minutes
**Complexity**: Low

### Deliverables

1. `.env.example` file with all variables
2. Documentation of required vs optional variables

### Technical Notes

```bash
# Template structure
# Database Configuration
DATABASE_URL="postgresql://USER:PASSWORD@hx-postgres-server:5432/docling_db?sslmode=require"
DATABASE_POOL_URL="postgresql://USER:PASSWORD@hx-postgres-server:6432/docling_db"

# Redis Configuration
REDIS_URL="rediss://hx-redis-server:6379"
REDIS_TLS_CA_PATH="/etc/ssl/certs/hx-redis-ca.crt"

# MCP Configuration
MCP_SERVER_URL="http://hx-docling-server:8000"

# Storage Configuration
UPLOAD_DIR="/data/docling-uploads"
MAX_FILE_SIZE_MB="100"

# Session Configuration
SESSION_TTL_HOURS="24"
```

---

## Task WIL-0-009: Prerequisites Validation Report

### Description

Execute all validation checks and compile comprehensive prerequisites validation report for Alex review.

### Acceptance Criteria

- [ ] All 12 prerequisite checks documented
- [ ] Pass/Fail status for each check
- [ ] Evidence provided for each passing check
- [ ] Action items documented for any failing checks
- [ ] Report signed off by William and Trinity
- [ ] Report approved by Alex before Sprint 1.1

### Dependencies

- Tasks WIL-0-001 through WIL-0-008 completed
- Access to all infrastructure components

### Effort Estimate

**Duration**: 45 minutes
**Complexity**: Medium

### Deliverables

1. `project/0.6-governance/prerequisites-validation.md`
2. Validation evidence (screenshots, logs)
3. Sign-off confirmation

### Technical Notes

Report should follow this structure:
```markdown
# Prerequisites Validation Report

## Summary
- Total Checks: 12
- Passed: X
- Failed: X
- Date: YYYY-MM-DD

## Check Results
| # | Check | Status | Evidence | Notes |
|---|-------|--------|----------|-------|
| 1 | PostgreSQL user | PASS/FAIL | [link] | ... |
...

## Sign-Off
- William Chen: [signature] [date]
- Trinity: [signature] [date]
- Alex Rivera (Approval): [signature] [date]
```

---

## Sprint Summary

| Task ID | Title | Effort | Dependencies |
|---------|-------|--------|--------------|
| WIL-0-001 | PostgreSQL Database User Configuration | 30m | None |
| WIL-0-002 | SSL Certificate Provisioning for PostgreSQL | 45m | None |
| WIL-0-003 | Database Creation and Schema Preparation | 20m | WIL-0-001 |
| WIL-0-004 | Data Partition Validation and Configuration | 30m | None |
| WIL-0-005 | Redis Server Connectivity and TLS Configuration | 45m | None |
| WIL-0-006 | PgBouncer Connection Pooling Validation | 30m | WIL-0-001 |
| WIL-0-007 | MCP Server Health Check Validation | 20m | None |
| WIL-0-008 | Environment Configuration Template | 30m | WIL-0-001 through WIL-0-007 |
| WIL-0-009 | Prerequisites Validation Report | 45m | All |

**Total Tasks**: 9
**Total Effort**: 4 hours 55 minutes

---

## Risk Considerations

| Risk | Mitigation |
|------|------------|
| SSL certificates not provisioned | Escalate to security team immediately |
| Redis TLS configuration complex | Allow extra time, coordinate with Sri |
| PgBouncer not configured | Direct PostgreSQL connection as fallback (document limitation) |
| Data partition not mounted | Block Sprint 1.1 until resolved |
