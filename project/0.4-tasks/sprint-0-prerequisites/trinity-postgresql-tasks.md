# Sprint 0: PostgreSQL Prerequisites Validation

**Version**: v1.1.0
**Sprint**: Sprint 0 (Prerequisites)
**Agent**: Trinity Smith (PostgreSQL DBA SME)
**Duration**: 1.5 hours
**Dependencies**: Infrastructure provisioning completed
**References**:
- Implementation Plan Section 4.1 (Sprint 0)
- Specification Section 12.1 (Database Prerequisites)

## Changelog

### v1.1.0 (2025-12-12)
- **FIX**: [DEF-014] Made connection pool settings configurable via environment variables
  - Changed DATABASE_URL to use `${DB_POOL_SIZE}` and `${DB_POOL_TIMEOUT_SEC}` instead of hardcoded values
  - Renamed `DB_POOL_TIMEOUT_MS` to `DB_POOL_TIMEOUT_SEC` for unit consistency (pool_timeout expects seconds)
  - Added comment clarifying pool settings are referenced in DATABASE_URL
  - Updated Technical Notes to document configurability

### v1.0.0 (Initial)
- Initial task definition for PostgreSQL prerequisites validation

---

## Task Overview

Validate all PostgreSQL prerequisites before Sprint 1.1 development begins. Ensure database server, users, permissions, SSL certificates, and connection pooling are correctly configured and accessible from hx-cc-server (192.168.10.224).

---

## Tasks

### TRI-0-001: Validate PostgreSQL User and Database Creation

**Priority**: P0 (Blocking)
**Effort**: 20 minutes
**Dependencies**: None

**Description**:
Verify that the `docling_app` PostgreSQL user and `docling_db` database have been created on hx-postgres-server with correct ownership and permissions.

**Acceptance Criteria**:
- [ ] Database `docling_db` exists and is listed in `\l` output
- [ ] User `docling_app` exists with expected roles (LOGIN, no SUPERUSER)
- [ ] User `docling_app` owns database `docling_db`
- [ ] Connection test succeeds: `psql -U docling_app -d docling_db -h hx-postgres-server.hx.dev.local -c '\conninfo'`

**Commands**:
```bash
# From hx-postgres-server or hx-cc-server with psql client
psql -U postgres -c "\l" | grep docling_db
psql -U postgres -c "\du docling_app"
psql -U postgres -c "SELECT datname, datdba::regrole FROM pg_database WHERE datname = 'docling_db';"
psql -U docling_app -d docling_db -h hx-postgres-server.hx.dev.local -c '\conninfo'
```

**Deliverables**:
- Screenshot or command output showing database and user existence
- Confirmation that `docling_app` owns `docling_db`

**Technical Notes**:
- Expected user attributes: `LOGIN`, `NOSUPERUSER`, `CREATEDB` (if needed for test databases)
- Database encoding should be UTF8, locale `en_US.UTF-8`
- If missing, escalate to William Chen for infrastructure provisioning

---

### TRI-0-002: Validate CRUD Permissions on Database

**Priority**: P0 (Blocking)
**Effort**: 15 minutes
**Dependencies**: TRI-0-001

**Description**:
Test that `docling_app` user has full CRUD (CREATE, READ, UPDATE, DELETE) permissions on `docling_db` by executing schema creation and data manipulation commands.

**Acceptance Criteria**:
- [ ] CREATE TABLE succeeds
- [ ] INSERT succeeds
- [ ] SELECT succeeds
- [ ] UPDATE succeeds
- [ ] DELETE succeeds
- [ ] DROP TABLE succeeds
- [ ] No permission denied errors

**Commands**:
```bash
psql -U docling_app -d docling_db -h hx-postgres-server.hx.dev.local << EOF
-- Test CRUD permissions
CREATE TABLE permission_test (
  id SERIAL PRIMARY KEY,
  test_data TEXT NOT NULL
);

INSERT INTO permission_test (test_data) VALUES ('test');

SELECT * FROM permission_test;

UPDATE permission_test SET test_data = 'updated' WHERE id = 1;

DELETE FROM permission_test WHERE id = 1;

DROP TABLE permission_test;
EOF
```

**Deliverables**:
- Command output showing all operations succeeded
- No "permission denied" errors in output

**Technical Notes**:
- User should have USAGE and CREATE privileges on the public schema
- If permissions are missing, grant with: `GRANT ALL PRIVILEGES ON DATABASE docling_db TO docling_app;`
- Verify schema-level permissions: `GRANT ALL ON SCHEMA public TO docling_app;`

---

### TRI-0-003: Validate SSL Certificate Availability

**Priority**: P0 (Blocking)
**Effort**: 10 minutes
**Dependencies**: None

**Description**:
Verify that PostgreSQL SSL/TLS CA certificate is available on hx-cc-server at the expected path and has correct permissions.

**Acceptance Criteria**:
- [ ] CA certificate file exists at `/etc/ssl/certs/hx-postgres-ca.crt` or documented path
- [ ] Certificate file is readable by the application user (agent0 or docling service user)
- [ ] Certificate is valid (not expired)
- [ ] Certificate format is PEM

**Commands**:
```bash
# From hx-cc-server
ls -la /etc/ssl/certs/hx-postgres-ca.crt

# Verify certificate validity
openssl x509 -in /etc/ssl/certs/hx-postgres-ca.crt -text -noout | grep -E 'Issuer|Subject|Not Before|Not After'

# Check file permissions (should be readable)
stat -c '%a %U %G' /etc/ssl/certs/hx-postgres-ca.crt
```

**Deliverables**:
- Certificate file path confirmation
- Certificate validity dates (must not be expired)
- File permissions output (e.g., 644 root root)

**Technical Notes**:
- Certificate should be part of HX-Infrastructure CA chain
- Alternative paths: `/etc/docling/ssl/ca.crt`, `/app/ssl/ca.crt`
- If certificate is missing or expired, escalate to William Chen

---

### TRI-0-004: Validate SSL Connection to PostgreSQL

**Priority**: P0 (Blocking)
**Effort**: 20 minutes
**Dependencies**: TRI-0-001, TRI-0-003

**Description**:
Test that SSL/TLS connections work correctly from hx-cc-server to hx-postgres-server using the CA certificate, with `sslmode=require` and `sslmode=verify-full`.

**Acceptance Criteria**:
- [ ] Connection with `sslmode=require` succeeds
- [ ] Connection with `sslmode=verify-full` succeeds
- [ ] Connection details show `SSL connection (protocol: TLSvX.X, cipher: ...)`
- [ ] `sslmode=disable` connection fails if SSL is enforced server-side (optional validation)

**Commands**:
```bash
# Test sslmode=require
psql "host=hx-postgres-server.hx.dev.local dbname=docling_db user=docling_app sslmode=require sslrootcert=/etc/ssl/certs/hx-postgres-ca.crt" -c '\conninfo'

# Test sslmode=verify-full (strict certificate validation)
psql "host=hx-postgres-server.hx.dev.local dbname=docling_db user=docling_app sslmode=verify-full sslrootcert=/etc/ssl/certs/hx-postgres-ca.crt" -c '\conninfo'

# Verify SSL is active
psql "host=hx-postgres-server.hx.dev.local dbname=docling_db user=docling_app sslmode=require sslrootcert=/etc/ssl/certs/hx-postgres-ca.crt" -c 'SELECT version(), pg_is_in_recovery(), current_setting('\''ssl'\'') as ssl_enabled;'
```

**Deliverables**:
- Connection info output showing SSL protocol and cipher
- Confirmation that both sslmode=require and sslmode=verify-full work

**Technical Notes**:
- If `sslmode=verify-full` fails, check that the server certificate CN matches `hx-postgres-server.hx.dev.local`
- Common issues: hostname mismatch, expired certificate, incorrect CA certificate
- Production configuration will use `sslmode=verify-full` for maximum security

---

### TRI-0-005: Validate PgBouncer Connection Pooling

**Priority**: P0 (Blocking)
**Effort**: 20 minutes
**Dependencies**: TRI-0-001, TRI-0-004

**Description**:
Verify that PgBouncer is accessible on port 6432 and correctly proxies connections to PostgreSQL (port 5432). Test that application connections via PgBouncer work correctly.

**Acceptance Criteria**:
- [ ] PgBouncer accessible at `hx-postgres-server.hx.dev.local:6432`
- [ ] Query via PgBouncer succeeds and returns expected result
- [ ] PgBouncer statistics show active pool
- [ ] Direct PostgreSQL connection (port 5432) also works for migration user

**Commands**:
```bash
# Test PgBouncer connection (port 6432)
psql -h hx-postgres-server.hx.dev.local -p 6432 -U docling_app -d docling_db -c "SELECT 1 AS test, current_database(), current_user;"

# Check PgBouncer stats (if pgbouncer database is accessible)
psql -h hx-postgres-server.hx.dev.local -p 6432 -U docling_app -d pgbouncer -c "SHOW POOLS;" || echo "pgbouncer admin db not accessible (expected)"

# Test direct PostgreSQL connection (for migrations - port 5432)
psql -h hx-postgres-server.hx.dev.local -p 5432 -U postgres -d docling_db -c "SELECT 1 AS direct_connection;"
```

**Deliverables**:
- Output showing successful query via PgBouncer (port 6432)
- Output showing direct PostgreSQL connection (port 5432) works
- PgBouncer pool statistics (if accessible)

**Technical Notes**:
- Application connections should ALWAYS use port 6432 (PgBouncer)
- Migrations use port 5432 (direct PostgreSQL) to avoid DDL conflicts with pooling
- PgBouncer should be configured with `pool_mode=transaction` for stateless connections
- If PgBouncer is not accessible, escalate to William Chen

---

### TRI-0-006: Document Connection Strings for .env Template

**Priority**: P1 (High)
**Effort**: 15 minutes
**Dependencies**: TRI-0-004, TRI-0-005

**Description**:
Create the correct `DATABASE_URL` and `DIRECT_DATABASE_URL` connection strings with all required parameters (SSL, pooling, timeouts) for inclusion in `.env.example`.

**Acceptance Criteria**:
- [ ] `DATABASE_URL` uses PgBouncer (port 6432) with connection pooling parameters
- [ ] `DIRECT_DATABASE_URL` uses direct PostgreSQL (port 5432) for migrations
- [ ] Both URLs include `sslmode=verify-full` and `sslrootcert` path
- [ ] Connection pool parameters documented (connection_limit, pool_timeout)
- [ ] Password placeholder uses environment variable syntax

**Template Output**:
```env
# ============================================================
# PostgreSQL Configuration
# ============================================================
# Application connection via PgBouncer (port 6432)
# Pool settings are configured via DB_POOL_SIZE and DB_POOL_TIMEOUT_SEC below
DATABASE_URL=postgresql://docling_app:${DB_PASSWORD}@hx-postgres-server.hx.dev.local:6432/docling_db?connection_limit=${DB_POOL_SIZE}&pool_timeout=${DB_POOL_TIMEOUT_SEC}&sslmode=verify-full&sslrootcert=/etc/ssl/certs/hx-postgres-ca.crt

# Direct connection for migrations (port 5432)
DIRECT_DATABASE_URL=postgresql://docling_migration:${MIGRATION_PASSWORD}@hx-postgres-server.hx.dev.local:5432/docling_db?sslmode=verify-full&sslrootcert=/etc/ssl/certs/hx-postgres-ca.crt

# Connection pool settings (referenced in DATABASE_URL)
DB_POOL_SIZE=5                         # Maximum pool size per instance
DB_POOL_TIMEOUT_SEC=20                 # Connection acquisition timeout in seconds
```

**Deliverables**:
- `.env.example` template snippet with documented connection strings
- Confirmation that paths match actual SSL certificate location
- Documentation of pool sizing rationale

**Technical Notes**:
- Pool settings are now configurable via environment variables (`DB_POOL_SIZE` and `DB_POOL_TIMEOUT_SEC`)
- Default `DB_POOL_SIZE=5` is sized for single-instance deployment (see Spec 5.1.3)
- Default `DB_POOL_TIMEOUT_SEC=20` prevents indefinite blocking on pool exhaustion
- Pool settings in DATABASE_URL are referenced via `${DB_POOL_SIZE}` and `${DB_POOL_TIMEOUT_SEC}` for flexibility
- Migration user (`docling_migration`) may be same as `docling_app` or separate with elevated privileges
- Coordinate with Neo Anderson to ensure `.env.example` includes this configuration

---

### TRI-0-007: Create Prerequisites Validation Report

**Priority**: P1 (High)
**Effort**: 10 minutes
**Dependencies**: TRI-0-001, TRI-0-002, TRI-0-003, TRI-0-004, TRI-0-005, TRI-0-006

**Description**:
Compile all validation results into a comprehensive report documenting that PostgreSQL prerequisites are met and the system is ready for Sprint 1.1 development.

**Acceptance Criteria**:
- [ ] Report includes pass/fail status for all 6 prerequisite checks (DB-01 through DB-06)
- [ ] Screenshots or command outputs attached as evidence
- [ ] Any issues or deviations documented with remediation steps
- [ ] Sign-off from Trinity confirming readiness
- [ ] Report shared with William Chen and Alex Rivera for review

**Report Template**:
```markdown
# PostgreSQL Prerequisites Validation Report

**Date**: YYYY-MM-DD
**Validator**: Trinity Smith
**Status**: PASS / PARTIAL / FAIL

## Summary

All PostgreSQL prerequisites for hx-docling-application have been validated and are ready for Sprint 1.1.

## Validation Results

| ID | Check | Status | Notes |
|----|-------|--------|-------|
| DB-01 | User `docling_app` created | PASS | User exists with LOGIN role |
| DB-02 | Database `docling_db` created | PASS | Database exists, UTF8 encoding |
| DB-03 | CRUD permissions verified | PASS | All operations succeeded |
| DB-04 | SSL certificate available | PASS | Certificate at /etc/ssl/certs/hx-postgres-ca.crt, valid until YYYY-MM-DD |
| DB-05 | SSL connection works | PASS | Both sslmode=require and verify-full succeed |
| DB-06 | PgBouncer accessible | PASS | Port 6432 connection successful |

## Connection Strings

Validated connection strings for `.env.example`:
[Include strings from TRI-0-006]

## Issues and Mitigations

[Document any issues encountered and how they were resolved]

## Sign-Off

- Trinity Smith (PostgreSQL DBA): [Date]
- William Chen (Infrastructure Lead): [Date]
- Alex Rivera (Architect): [Date]

## Next Steps

Prerequisites validated. Sprint 1.1 (Prisma schema design) is cleared to proceed.
```

**Deliverables**:
- Completed prerequisites validation report
- Evidence (screenshots, command outputs)
- Sign-off confirmation from Trinity, William, and Alex

**Technical Notes**:
- Report should be stored in `/home/agent0/hx-docling-application/project/0.4-tasks/sprint-0-prerequisites/postgresql-validation-report.md`
- Share report via team communication channel before Sprint 1.1 kickoff
- Any FAIL status is BLOCKING for Sprint 1.1 and must be resolved before proceeding

---

## Dependencies

**Upstream**:
- Infrastructure provisioning by William Chen (PostgreSQL server, PgBouncer, SSL certificates)

**Downstream**:
- Sprint 1.1 Prisma schema design (TRI-1.1-001)
- Sprint 1.2 Database connection implementation (TRI-1.2-001)

---

## Risk Mitigation

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| SSL certificate expired or invalid | Low | High | Validate certificate validity dates in TRI-0-003 |
| PgBouncer misconfigured | Medium | High | Test both pooled and direct connections in TRI-0-005 |
| Missing permissions | Medium | Medium | Comprehensive CRUD test in TRI-0-002 |
| Network connectivity issues | Low | High | Test from hx-cc-server, not local workstation |

---

## Total Effort Summary

**Total Estimated Effort**: 1.5 hours (90 minutes)
**Task Breakdown**:
- TRI-0-001: 20 minutes
- TRI-0-002: 15 minutes
- TRI-0-003: 10 minutes
- TRI-0-004: 20 minutes
- TRI-0-005: 20 minutes
- TRI-0-006: 15 minutes
- TRI-0-007: 10 minutes

**Critical Path**: TRI-0-001 → TRI-0-002 → TRI-0-004 → TRI-0-005 → TRI-0-006 → TRI-0-007
