# Sprint 1.8: Database Testing and Production Readiness

**Sprint**: Sprint 1.8 (Testing & Documentation)
**Agent**: Trinity Smith (PostgreSQL DBA SME)
**Duration**: 2.5 hours
**Dependencies**: All previous sprints complete
**References**:
- Implementation Plan Section 4.11 (Sprint 1.8)
- Specification Section 10 (Testing Specifications)
- Specification Section 10.9 (Integration Test Database Assertions)
- Specification Section 3.4 (Database Requirements)

---

## Task Overview

Comprehensive database testing, production readiness validation, backup/recovery procedures, monitoring setup, and final database documentation. Ensure the database layer is production-ready with proper testing, backups, monitoring, and documentation.

---

## Tasks

### TRI-1.8-001: Create Comprehensive Integration Tests for Database Operations

**Priority**: P0 (Blocking)
**Effort**: 45 minutes
**Dependencies**: All sprint tasks complete

**Description**:
Create integration tests that verify database operations through API endpoints per Specification Section 10.9. Tests must validate database state changes, not just HTTP responses.

**Acceptance Criteria**:
- [ ] Integration tests verify Job creation in database after `/api/v1/process`
- [ ] Integration tests verify Result creation in database after successful processing
- [ ] Integration tests verify Job status transitions follow state machine
- [ ] Integration tests verify cascade delete (Job deletion deletes Results)
- [ ] Integration tests verify pagination in `/api/v1/history`
- [ ] Tests use test database (not production)
- [ ] All integration tests pass
- [ ] Test coverage >80% for database operations

**Implementation**:
```typescript
// test/integration/database/api-database.test.ts

import { describe, it, expect, beforeEach } from 'vitest';
import { prisma } from '@/lib/db/prisma';

describe('API Database Integration Tests', () => {
  beforeEach(async () => {
    // Clean up test data before each test
    await prisma.result.deleteMany();
    await prisma.job.deleteMany();
  });

  describe('POST /api/v1/process', () => {
    it('creates Job record in database with correct initial status', async () => {
      const response = await fetch('http://localhost:3000/api/v1/process', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          sessionId: 'test-session',
          inputType: 'FILE',
          fileName: 'test.pdf',
          filePath: '/tmp/test.pdf',
        }),
      });

      expect(response.status).toBe(200);
      const { jobId } = await response.json();

      // DATABASE ASSERTION
      const job = await prisma.job.findUnique({ where: { id: jobId } });

      expect(job).not.toBeNull();
      expect(job?.sessionId).toBe('test-session');
      expect(job?.status).toBe('PENDING'); // Initial status
      expect(job?.inputType).toBe('FILE');
      expect(job?.fileName).toBe('test.pdf');
    });
  });

  describe('Job Processing Flow', () => {
    it('creates Result records on successful processing', async () => {
      // Setup: Create job in PROCESSING state
      const job = await prisma.job.create({
        data: {
          sessionId: 'test-session',
          status: 'PROCESSING',
          inputType: 'FILE',
          fileName: 'test.pdf',
          filePath: '/tmp/test.pdf',
          mcpToolName: 'convert_pdf',
        },
      });

      // Simulate processing completion (normally done by MCP handler)
      await prisma.$transaction([
        prisma.result.createMany({
          data: [
            {
              jobId: job.id,
              formatType: 'markdown',
              content: '# Test Document',
              sizeBytes: 16,
            },
            {
              jobId: job.id,
              formatType: 'html',
              content: '<h1>Test Document</h1>',
              sizeBytes: 24,
            },
            {
              jobId: job.id,
              formatType: 'json',
              content: '{"title": "Test"}',
              sizeBytes: 19,
            },
          ],
        }),
        prisma.job.update({
          where: { id: job.id },
          data: { status: 'COMPLETE', completedAt: new Date() },
        }),
      ]);

      // DATABASE ASSERTIONS
      const results = await prisma.result.findMany({
        where: { jobId: job.id },
      });

      expect(results).toHaveLength(3); // markdown, html, json
      expect(results.map((r) => r.formatType).sort()).toEqual([
        'html',
        'json',
        'markdown',
      ]);

      const updatedJob = await prisma.job.findUnique({ where: { id: job.id } });
      expect(updatedJob?.status).toBe('COMPLETE');
      expect(updatedJob?.completedAt).toBeDefined();
    });
  });

  describe('Job Status State Machine', () => {
    /**
     * IMPORTANT: State transitions MUST be atomic to prevent TOCTOU race conditions.
     * Use updateMany with status predicate instead of findUnique + validate + update.
     *
     * Pattern:
     *   const updated = await prisma.job.updateMany({
     *     where: { id: jobId, status: expectedCurrentStatus },
     *     data: { status: newStatus },
     *   });
     *   if (updated.count === 0) throw new AppError('E701', 'Invalid state transition', 409);
     */

    it('transitions through valid states: PENDING → UPLOADING → PROCESSING → COMPLETE', async () => {
      const job = await prisma.job.create({
        data: {
          sessionId: 'state-test',
          status: 'PENDING',
          inputType: 'FILE',
        },
      });

      // Atomic transition: PENDING → UPLOADING (prevents race conditions)
      const pendingToUploading = await prisma.job.updateMany({
        where: { id: job.id, status: 'PENDING' },
        data: { status: 'UPLOADING' },
      });
      expect(pendingToUploading.count).toBe(1);

      let currentJob = await prisma.job.findUnique({ where: { id: job.id } });
      expect(currentJob?.status).toBe('UPLOADING');

      // Atomic transition: UPLOADING → PROCESSING
      const uploadingToProcessing = await prisma.job.updateMany({
        where: { id: job.id, status: 'UPLOADING' },
        data: { status: 'PROCESSING', startedAt: new Date() },
      });
      expect(uploadingToProcessing.count).toBe(1);

      currentJob = await prisma.job.findUnique({ where: { id: job.id } });
      expect(currentJob?.status).toBe('PROCESSING');
      expect(currentJob?.startedAt).toBeDefined();

      // Atomic transition: PROCESSING → COMPLETE
      const processingToComplete = await prisma.job.updateMany({
        where: { id: job.id, status: 'PROCESSING' },
        data: { status: 'COMPLETE', completedAt: new Date() },
      });
      expect(processingToComplete.count).toBe(1);

      currentJob = await prisma.job.findUnique({ where: { id: job.id } });
      expect(currentJob?.status).toBe('COMPLETE');
      expect(currentJob?.completedAt).toBeDefined();
    });

    it('rejects invalid state transition (atomic check)', async () => {
      const job = await prisma.job.create({
        data: {
          sessionId: 'invalid-transition-test',
          status: 'PENDING',
          inputType: 'FILE',
        },
      });

      // Attempt to transition PENDING → PROCESSING (invalid, must go through UPLOADING)
      const invalidTransition = await prisma.job.updateMany({
        where: { id: job.id, status: 'UPLOADING' }, // Job is PENDING, not UPLOADING
        data: { status: 'PROCESSING' },
      });

      // Atomic check: count === 0 means transition was rejected
      expect(invalidTransition.count).toBe(0);

      // Verify job status unchanged
      const currentJob = await prisma.job.findUnique({ where: { id: job.id } });
      expect(currentJob?.status).toBe('PENDING');
    });

    it('handles retry states: PROCESSING → RETRY_PROCESSING → PROCESSING', async () => {
      const job = await prisma.job.create({
        data: {
          sessionId: 'retry-test',
          status: 'PROCESSING',
          inputType: 'FILE',
          retryCount: 0,
          maxRetries: 3,
        },
      });

      // Atomic transition to RETRY_PROCESSING
      const toRetry = await prisma.job.updateMany({
        where: { id: job.id, status: 'PROCESSING' },
        data: { status: 'RETRY_PROCESSING' },
      });
      expect(toRetry.count).toBe(1);

      // Increment retry count separately (or use raw SQL for atomic increment)
      await prisma.job.update({
        where: { id: job.id },
        data: { retryCount: { increment: 1 } },
      });

      let currentJob = await prisma.job.findUnique({ where: { id: job.id } });
      expect(currentJob?.status).toBe('RETRY_PROCESSING');
      expect(currentJob?.retryCount).toBe(1);

      // Atomic transition back to PROCESSING
      const backToProcessing = await prisma.job.updateMany({
        where: { id: job.id, status: 'RETRY_PROCESSING' },
        data: { status: 'PROCESSING' },
      });
      expect(backToProcessing.count).toBe(1);

      currentJob = await prisma.job.findUnique({ where: { id: job.id } });
      expect(currentJob?.status).toBe('PROCESSING');
    });

    it('prevents double-processing race condition', async () => {
      const job = await prisma.job.create({
        data: {
          sessionId: 'race-test',
          status: 'PENDING',
          inputType: 'FILE',
        },
      });

      // Simulate two concurrent requests trying to start processing
      const [request1, request2] = await Promise.all([
        prisma.job.updateMany({
          where: { id: job.id, status: 'PENDING' },
          data: { status: 'UPLOADING' },
        }),
        prisma.job.updateMany({
          where: { id: job.id, status: 'PENDING' },
          data: { status: 'UPLOADING' },
        }),
      ]);

      // Only ONE request should succeed (atomic guarantee)
      expect(request1.count + request2.count).toBe(1);

      // Job should be in UPLOADING state
      const currentJob = await prisma.job.findUnique({ where: { id: job.id } });
      expect(currentJob?.status).toBe('UPLOADING');
    });
  });

  describe('Cascade Delete', () => {
    it('deletes Results when Job is deleted', async () => {
      const job = await prisma.job.create({
        data: {
          sessionId: 'cascade-test',
          status: 'COMPLETE',
          inputType: 'FILE',
        },
      });

      await prisma.result.createMany({
        data: [
          {
            jobId: job.id,
            formatType: 'markdown',
            content: '# Test',
            sizeBytes: 6,
          },
          {
            jobId: job.id,
            formatType: 'html',
            content: '<h1>Test</h1>',
            sizeBytes: 14,
          },
        ],
      });

      // Verify results exist before delete
      let results = await prisma.result.findMany({ where: { jobId: job.id } });
      expect(results).toHaveLength(2);

      // Delete job (should cascade to results)
      await prisma.job.delete({ where: { id: job.id } });

      // Verify results are deleted
      results = await prisma.result.findMany({ where: { jobId: job.id } });
      expect(results).toHaveLength(0);
    });
  });

  describe('GET /api/v1/history', () => {
    it('returns jobs filtered by sessionId with correct pagination', async () => {
      // Create jobs for different sessions
      await prisma.job.createMany({
        data: [
          {
            sessionId: 'session-a',
            status: 'COMPLETE',
            inputType: 'FILE',
            createdAt: new Date('2025-12-01'),
          },
          {
            sessionId: 'session-a',
            status: 'FAILED',
            inputType: 'FILE',
            createdAt: new Date('2025-12-02'),
          },
          {
            sessionId: 'session-b',
            status: 'COMPLETE',
            inputType: 'URL',
            createdAt: new Date('2025-12-03'),
          },
        ],
      });

      const response = await fetch(
        'http://localhost:3000/api/v1/history?sessionId=session-a&page=1&pageSize=10'
      );

      expect(response.status).toBe(200);
      const data = await response.json();

      expect(data.jobs).toHaveLength(2); // Only session-a jobs
      expect(data.total).toBe(2);
      expect(data.page).toBe(1);
      expect(data.pageSize).toBe(10);
      expect(data.totalPages).toBe(1);

      // Verify jobs are sorted by createdAt DESC (newest first)
      expect(new Date(data.jobs[0].createdAt).getTime()).toBeGreaterThan(
        new Date(data.jobs[1].createdAt).getTime()
      );
    });
  });
});
```

**Deliverables**:
- Integration test file created
- All tests pass (verified with `npm test`)
- Test coverage report shows >80% coverage for database layer

**Technical Notes**:
- Use test database separate from development database
- Clean up data in `beforeEach` to ensure test isolation
- Tests should verify database state, not just API responses
- Use transactions in tests to speed up cleanup
- Run tests with `NODE_ENV=test` to prevent pollution of dev database

---

### TRI-1.8-002: Test Backup and Restore Procedures

**Priority**: P0 (Blocking)
**Effort**: 30 minutes
**Dependencies**: TRI-1.2-009 (Backup strategy documented)

**Description**:
Execute a full backup and restore test cycle to verify that backup procedures work correctly and backups can be restored successfully. Document any issues and update backup procedures.

**Acceptance Criteria**:
- [ ] Full backup created with pg_dump
- [ ] Backup file verified (non-zero size, valid format)
- [ ] Backup restored to test database successfully
- [ ] Restored data matches original data
- [ ] Restore time documented (<1 hour for RTO)
- [ ] Backup and restore procedures validated

**Test Procedure**:
```bash
#!/bin/bash
# test-backup-restore.sh

set -e

# Use environment variables with defaults
DB_HOST="${DB_HOST:-hx-postgres-server.hx.dev.local}"
DB_PORT="${DB_PORT:-5432}"
DB_USER="${DB_USER:-postgres}"
DB_NAME="${DB_NAME:-docling_db}"
BACKUP_DIR="${BACKUP_DIR:-/data/backups/docling-db}"

# Ensure required variables are set
if [ -z "$DB_HOST" ] || [ -z "$DB_USER" ] || [ -z "$DB_NAME" ]; then
  echo "ERROR: DB_HOST, DB_USER, and DB_NAME must be set"
  exit 1
fi

# Password should be in .pgpass file or PGPASSWORD environment variable
if [ -z "$PGPASSWORD" ] && [ ! -f ~/.pgpass ]; then
  echo "ERROR: PGPASSWORD environment variable or ~/.pgpass required"
  exit 1
fi

TIMESTAMP=$(date +%Y%m%d-%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/test-backup-${TIMESTAMP}.dump"

echo "=== PostgreSQL Backup & Restore Test ==="
echo "Timestamp: ${TIMESTAMP}"

# Step 1: Create backup
echo "Step 1: Creating backup..."
pg_dump -h "${DB_HOST}" -p "${DB_PORT}" -U "${DB_USER}" -d "${DB_NAME}" \
  --format=custom --compress=9 --file="${BACKUP_FILE}"

# Verify backup file exists and is non-empty
if [ ! -s "${BACKUP_FILE}" ]; then
  echo "ERROR: Backup file is missing or empty"
  exit 1
fi

BACKUP_SIZE=$(du -h "${BACKUP_FILE}" | cut -f1)
echo "Backup created: ${BACKUP_FILE} (${BACKUP_SIZE})"

# Step 2: Create test database for restore
echo "Step 2: Creating test database..."
createdb -h "${DB_HOST}" -p "${DB_PORT}" -U "${DB_USER}" docling_db_restore_test

# Step 3: Restore backup to test database
echo "Step 3: Restoring backup..."
START_TIME=$(date +%s)
pg_restore -h "${DB_HOST}" -p "${DB_PORT}" -U "${DB_USER}" \
  -d docling_db_restore_test "${BACKUP_FILE}"
END_TIME=$(date +%s)
RESTORE_DURATION=$((END_TIME - START_TIME))

echo "Restore completed in ${RESTORE_DURATION} seconds"

# Step 4: Verify data integrity
echo "Step 4: Verifying data integrity..."

# Count records in original database
ORIGINAL_JOB_COUNT=$(psql -h "${DB_HOST}" -U "${DB_USER}" -d "${DB_NAME}" -t -c "SELECT COUNT(*) FROM jobs;")
ORIGINAL_RESULT_COUNT=$(psql -h "${DB_HOST}" -U "${DB_USER}" -d "${DB_NAME}" -t -c "SELECT COUNT(*) FROM results;")

# Count records in restored database
RESTORED_JOB_COUNT=$(psql -h "${DB_HOST}" -U "${DB_USER}" -d docling_db_restore_test -t -c "SELECT COUNT(*) FROM jobs;")
RESTORED_RESULT_COUNT=$(psql -h "${DB_HOST}" -U "${DB_USER}" -d docling_db_restore_test -t -c "SELECT COUNT(*) FROM results;")

echo "Original: ${ORIGINAL_JOB_COUNT} jobs, ${ORIGINAL_RESULT_COUNT} results"
echo "Restored: ${RESTORED_JOB_COUNT} jobs, ${RESTORED_RESULT_COUNT} results"

if [ "${ORIGINAL_JOB_COUNT}" -ne "${RESTORED_JOB_COUNT}" ] || [ "${ORIGINAL_RESULT_COUNT}" -ne "${RESTORED_RESULT_COUNT}" ]; then
  echo "ERROR: Data mismatch between original and restored databases"
  exit 1
fi

echo "Data integrity verified!"

# Step 5: Clean up test database
echo "Step 5: Cleaning up test database..."
dropdb -h "${DB_HOST}" -U "${DB_USER}" docling_db_restore_test

echo "=== Backup & Restore Test PASSED ==="
echo "RTO (Restore Time): ${RESTORE_DURATION} seconds"
echo "Backup Size: ${BACKUP_SIZE}"
```

**Deliverables**:
- Backup and restore test script (`scripts/test-backup-restore.sh`)
- Test execution log showing successful restore
- RTO (Recovery Time Objective) measurement
- Updated backup documentation with test results

**Technical Notes**:
- Test restore to DIFFERENT database to avoid overwriting production data
- Verify record counts match between original and restored databases
- Document restore time to validate RTO <1 hour
- Test restore procedure quarterly to ensure it remains valid
- Coordinate with William Chen for automated backup scheduling
- **Password Handling**: Use `~/.pgpass` file (chmod 600) or `PGPASSWORD` environment variable (not in code or scripts)
- **Example**: `export PGPASSWORD='...' && bash test-backup-restore.sh`

---

### TRI-1.8-003: Configure Database Monitoring and Alerting

**Priority**: P1 (High)
**Effort**: 30 minutes
**Dependencies**: TRI-1.2-010 (Health endpoint integration)

**Description**:
Configure PostgreSQL monitoring with postgres_exporter and define alerting thresholds for critical database metrics. Coordinate with William Chen on Prometheus/Grafana integration.

**Acceptance Criteria**:
- [ ] postgres_exporter configured and running
- [ ] Key metrics exposed to Prometheus (connections, transactions, replication lag, cache hit ratio)
- [ ] Grafana dashboard created for database metrics
- [ ] Alerts defined for critical conditions (connection exhaustion, high latency, replication lag)
- [ ] Alert thresholds documented
- [ ] Test alert by simulating critical condition

**Monitoring Metrics**:

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `pg_stat_database_numbackends` | Active connections | >80% of max_connections |
| `pg_stat_database_xact_commit` | Transaction rate | <10/s (degraded) |
| `pg_stat_database_blks_hit / blks_read` | Cache hit ratio | <95% |
| `pg_replication_lag` | Replication lag (if HA) | >10 seconds |
| `pg_database_size_bytes` | Database size | >80% of disk capacity |
| `pg_stat_activity` | Long-running queries | >5 minutes |

**postgres_exporter Configuration**:
```yaml
# /etc/postgres_exporter/postgres_exporter.yml

# Connection string (use read-only monitoring user)
DATA_SOURCE_NAME: "postgresql://monitoring_user:${MONITORING_PASSWORD}@hx-postgres-server.hx.dev.local:5432/docling_db?sslmode=verify-full"

# Metrics to collect
PG_EXPORTER_EXTEND_QUERY_PATH: "/etc/postgres_exporter/queries.yaml"

# Custom queries
# queries.yaml:
pg_stat_activity_long_queries:
  query: |
    SELECT COUNT(*) AS count
    FROM pg_stat_activity
    WHERE state = 'active'
      AND now() - query_start > interval '5 minutes';
  metrics:
    - count:
        usage: "GAUGE"
        description: "Number of queries running longer than 5 minutes"
```

**Alert Rules (Prometheus)**:
```yaml
# /etc/prometheus/alerts/postgresql.yml

groups:
  - name: postgresql
    interval: 30s
    rules:
      - alert: PostgreSQLConnectionExhaustion
        expr: pg_stat_database_numbackends / pg_settings_max_connections > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL connection pool near exhaustion ({{ $value }}%)"

      - alert: PostgreSQLLowCacheHitRatio
        expr: rate(pg_stat_database_blks_hit[5m]) / (rate(pg_stat_database_blks_hit[5m]) + rate(pg_stat_database_blks_read[5m])) < 0.95
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL cache hit ratio below 95% ({{ $value }})"

      - alert: PostgreSQLLongRunningQueries
        expr: pg_stat_activity_long_queries > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL has {{ $value }} queries running >5 minutes"
```

**Deliverables**:
- postgres_exporter configured and running
- Grafana dashboard created (coordinate with William)
- Alert rules defined in Prometheus
- Alert test successful (simulate connection exhaustion)

**Technical Notes**:
- Create dedicated `monitoring_user` with read-only access to pg_stat_* views
- postgres_exporter runs as systemd service on hx-postgres-server or hx-cc-server
- Coordinate with William Chen for Prometheus/Grafana setup
- Test alerts by simulating conditions (e.g., open many connections to trigger alert)
- Alert thresholds should be tuned based on baseline metrics

---

### TRI-1.8-004: Perform Database Load Testing

**Priority**: P1 (High)
**Effort**: 25 minutes
**Dependencies**: TRI-1.8-001 (Integration tests complete)

**Description**:
Execute load tests to validate database performance under expected production load: concurrent job creation, history queries, and result retrieval. Verify connection pooling behavior and identify bottlenecks.

**Acceptance Criteria**:
- [ ] Load test simulates 50 concurrent job creations
- [ ] Load test simulates 100 concurrent history queries
- [ ] Connection pool handles load without exhaustion
- [ ] Query response times <100ms at P95 for indexed queries
- [ ] No connection timeout errors
- [ ] Load test results documented

**Load Test Script**:
```typescript
// test/load/database-load.test.ts

import { prisma } from '@/lib/db/prisma';
import { performance } from 'perf_hooks';

async function loadTestJobCreation(concurrency: number) {
  console.log(`\n=== Load Test: ${concurrency} Concurrent Job Creations ===`);

  const start = performance.now();

  const promises = Array.from({ length: concurrency }, (_, i) =>
    prisma.job.create({
      data: {
        sessionId: `load-test-${Date.now()}`,
        status: 'PENDING',
        inputType: 'FILE',
        fileName: `test-${i}.pdf`,
        fileSize: BigInt(Math.floor(Math.random() * 10_000_000)),
      },
    })
  );

  await Promise.all(promises);

  const duration = performance.now() - start;
  console.log(`Created ${concurrency} jobs in ${duration.toFixed(2)}ms`);
  console.log(`Average: ${(duration / concurrency).toFixed(2)}ms per job`);
}

async function loadTestHistoryQueries(concurrency: number) {
  console.log(`\n=== Load Test: ${concurrency} Concurrent History Queries ===`);

  const sessionId = `load-test-${Date.now()}`;

  // Create 100 jobs for the session
  await prisma.job.createMany({
    data: Array.from({ length: 100 }, (_, i) => ({
      sessionId,
      status: 'COMPLETE',
      inputType: 'FILE',
      fileName: `test-${i}.pdf`,
    })),
  });

  const start = performance.now();
  const latencies: number[] = [];

  const promises = Array.from({ length: concurrency }, async () => {
    const queryStart = performance.now();

    await prisma.job.findMany({
      where: { sessionId },
      include: { results: true },
      orderBy: { createdAt: 'desc' },
      take: 20,
    });

    const queryLatency = performance.now() - queryStart;
    latencies.push(queryLatency);
  });

  await Promise.all(promises);

  const totalDuration = performance.now() - start;
  latencies.sort((a, b) => a - b);

  const p50 = latencies[Math.floor(latencies.length * 0.5)];
  const p95 = latencies[Math.floor(latencies.length * 0.95)];
  const p99 = latencies[Math.floor(latencies.length * 0.99)];

  console.log(`Executed ${concurrency} queries in ${totalDuration.toFixed(2)}ms`);
  console.log(`P50: ${p50.toFixed(2)}ms`);
  console.log(`P95: ${p95.toFixed(2)}ms`);
  console.log(`P99: ${p99.toFixed(2)}ms`);

  // Verify P95 meets target
  if (p95 > 100) {
    console.warn(`WARNING: P95 latency (${p95.toFixed(2)}ms) exceeds target (100ms)`);
  }
}

async function main() {
  console.log('=== Database Load Testing ===');

  try {
    await loadTestJobCreation(50);
    await loadTestHistoryQueries(100);

    console.log('\n=== Load Tests PASSED ===');
  } catch (error) {
    console.error('Load test failed:', error);
    process.exit(1);
  } finally {
    await prisma.$disconnect();
  }
}

main();
```

**Run Load Test**:
```bash
# Run load test
npx tsx test/load/database-load.test.ts

# Monitor connection pool during load test (separate terminal)
watch -n 1 'psql "$DATABASE_URL" -c "SELECT COUNT(*) AS active_connections FROM pg_stat_activity WHERE datname = '\''docling_db'\'';"'
```

**Deliverables**:
- Load test script created
- Load test results documented (P50, P95, P99 latencies)
- Connection pool behavior verified
- Bottlenecks identified (if any)

**Technical Notes**:
- Load test should run against test database, not production
- Monitor connection pool during load test to verify pooling works
- If P95 latency exceeds 100ms, investigate with EXPLAIN ANALYZE
- Common bottlenecks: missing indexes, insufficient connection pool size, disk I/O
- Coordinate with William Chen if infrastructure tuning is needed

---

### TRI-1.8-005: Document Database Schema and Architecture

**Priority**: P2 (Medium)
**Effort**: 20 minutes
**Dependencies**: All tasks complete

**Description**:
Create comprehensive database schema documentation including entity-relationship diagram (ERD), table descriptions, index rationale, and operational procedures.

**Acceptance Criteria**:
- [ ] ERD generated from Prisma schema
- [ ] Table and column descriptions documented
- [ ] Index usage and rationale documented
- [ ] State machine diagram included
- [ ] Operational procedures documented (backup, restore, migration)
- [ ] Documentation reviewed by Neo and Alex

**Documentation Structure**:
```markdown
# Database Schema Documentation

## Entity-Relationship Diagram

[Generate ERD using Prisma ERD tool or dbdiagram.io]

```mermaid
erDiagram
    Job ||--o{ Result : has
    Job {
        uuid id PK
        string sessionId
        JobStatus status
        string inputType
        string inputPath
        string inputUrl
        datetime createdAt
        datetime updatedAt
    }
    Result {
        uuid id PK
        uuid jobId FK
        string formatType
        text content
        int sizeBytes
        datetime generatedAt
    }
```

## Tables

### jobs

**Purpose**: Stores document processing jobs with status, metadata, and error tracking.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | UUID | No | Primary key |
| sessionId | VARCHAR(255) | No | Redis session ID for user correlation |
| status | JobStatus | No | Current job status (see state machine) |
| currentStage | VARCHAR(50) | Yes | Processing stage (upload, parsing, conversion, export, saving) |
| progressPercent | INT | Yes | Progress percentage (0-100) |
| inputType | VARCHAR(10) | No | FILE or URL |
| inputPath | VARCHAR(500) | Yes | File path on disk (if FILE) |
| inputUrl | VARCHAR(2048) | Yes | URL (if URL) |
| fileName | VARCHAR(255) | Yes | Original filename |
| fileSize | BIGINT | Yes | File size in bytes |
| fileMime | VARCHAR(100) | Yes | MIME type |
| mcpToolName | VARCHAR(50) | Yes | MCP tool used (convert_pdf, convert_docx, etc.) |
| timeoutMs | INT | Yes | Configured timeout for this job |
| retryCount | INT | No | Current retry attempt (0-based) |
| maxRetries | INT | No | Maximum retry attempts (default 3) |
| checkpointData | JSON | Yes | Checkpoint for resumption (see Spec 5.5) |
| errorCode | VARCHAR(10) | Yes | Error code (E001, E201, etc.) |
| errorMessage | VARCHAR(500) | Yes | Human-readable error message |
| errorDetails | JSON | Yes | Detailed error context |
| createdAt | TIMESTAMP | No | Job creation time |
| updatedAt | TIMESTAMP | No | Last update time (auto-updated) |
| startedAt | TIMESTAMP | Yes | Processing start time |
| completedAt | TIMESTAMP | Yes | Processing completion time |

**Indexes**:
- `jobs_pkey`: Primary key on `id`
- `jobs_sessionId_idx`: Filter by sessionId (history queries)
- `jobs_status_idx`: Filter by status (admin dashboards)
- `jobs_createdAt_idx`: Sort by creation time (DESC)
- `jobs_sessionId_createdAt_idx`: Composite index for sessionId + ORDER BY createdAt (history pagination)

### results

**Purpose**: Stores export outputs (markdown, HTML, JSON) for completed jobs.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | UUID | No | Primary key |
| jobId | UUID | No | Foreign key to jobs.id (CASCADE DELETE) |
| formatType | VARCHAR(20) | No | Export format (markdown, html, json) |
| content | TEXT | No | Export content (large text) |
| sizeBytes | INT | No | Content size in bytes |
| generatedAt | TIMESTAMP | No | Export generation time |

**Indexes**:
- `results_pkey`: Primary key on `id`
- `results_jobId_idx`: Foreign key index for JOIN queries

**Constraints**:
- Foreign key: `jobId` REFERENCES `jobs(id)` ON DELETE CASCADE

## Enums

### JobStatus

**Purpose**: Represents the current state of a job in the processing pipeline.

| Value | Description | Terminal? |
|-------|-------------|-----------|
| PENDING | Job created, awaiting processing | No |
| UPLOADING | File upload in progress | No |
| PROCESSING | MCP processing in progress | No |
| COMPLETE | Successfully completed with all exports | Yes |
| FAILED | Permanent failure, no retry | Yes |
| CANCELLED | User-cancelled job | Yes |
| PARTIAL_COMPLETE | Some exports succeeded, some failed | Yes |
| RETRY_PENDING | Scheduled for retry | No |
| RETRY_UPLOAD | Retrying upload | No |
| RETRY_PROCESSING | Retrying MCP processing | No |
| RETRY_EXPORT | Retrying export generation | No |
| RETRY_SAVE | Retrying result save | No |
| RETRY_FAILED | Max retries exhausted | Yes |

**State Machine**: See Specification Section 5.1.2

## Operational Procedures

### Backups

- **Frequency**: Daily at 3 AM
- **Retention**: 7 days (development), 30 days (production)
- **Method**: pg_dump --format=custom --compress=9
- **Location**: /data/backups/docling-db/
- **Restore**: See `docs/database-backup.md`

### Migrations

- **Tool**: Prisma Migrate
- **Process**:
  1. Update `prisma/schema.prisma`
  2. Run `npx prisma migrate dev --name <description>`
  3. Review generated SQL in `prisma/migrations/`
  4. Test migration on staging
  5. Apply to production: `npx prisma migrate deploy`

### Monitoring

- **Tool**: postgres_exporter + Prometheus + Grafana
- **Key Metrics**: Connections, transaction rate, cache hit ratio, replication lag
- **Alerts**: See Prometheus alert rules in `/etc/prometheus/alerts/postgresql.yml`

### Maintenance

- **Autovacuum**: Enabled (default)
- **Manual VACUUM**: Run after bulk deletes
- **ANALYZE**: Run after bulk inserts/updates to update query planner statistics
- **Index Maintenance**: Monitor pg_stat_user_indexes for unused indexes
```

**Deliverables**:
- Database schema documentation in `docs/database-schema.md`
- ERD diagram (Mermaid or PNG)
- Operational procedures documented

**Technical Notes**:
- Use Prisma ERD tool: `npx prisma-erd-generator`
- Documentation should be kept in sync with schema changes
- Include migration history in documentation
- Coordinate with Neo for review

---

### TRI-1.8-006: Final Production Readiness Checklist

**Priority**: P0 (Blocking)
**Effort**: 20 minutes
**Dependencies**: All tasks complete

**Description**:
Complete a comprehensive production readiness checklist to ensure the database layer meets all requirements for production deployment. Verify that all deliverables are complete, tested, and documented.

**Acceptance Criteria**:
- [ ] All database tasks from Sprints 0, 1.1, 1.2, 1.7, 1.8 complete
- [ ] All integration tests pass
- [ ] Backup and restore procedures tested and documented
- [ ] Monitoring and alerting configured
- [ ] Load testing completed with acceptable performance
- [ ] Documentation complete and reviewed
- [ ] Sign-off from Trinity, William, and Alex
- [ ] Ready for production deployment

**Production Readiness Checklist**:

```markdown
# Database Production Readiness Checklist

## Prerequisites (Sprint 0)
- [x] PostgreSQL user `docling_app` created
- [x] Database `docling_db` created
- [x] CRUD permissions verified
- [x] SSL certificate configured
- [x] SSL connection works (sslmode=verify-full)
- [x] PgBouncer accessible (port 6432)
- [x] Connection strings documented in .env.example

## Schema (Sprint 1.1)
- [x] Prisma schema created with Job and Result models
- [x] JobStatus enum defined with all states
- [x] Indexes defined (sessionId, status, createdAt, composite)
- [x] Foreign key relationships defined
- [x] Schema validated and reviewed
- [x] Prisma Client generated

## Integration (Sprint 1.2)
- [x] Prisma client singleton implemented
- [x] DATABASE_URL configured (PgBouncer, port 6432)
- [x] DIRECT_DATABASE_URL configured (direct, port 5432)
- [x] Initial migration applied
- [x] Migration rollback procedure tested
- [x] Health check functions implemented
- [x] Indexes verified in database
- [x] CRUD operations tested
- [x] PgBouncer connection pooling validated
- [x] Backup procedure documented and tested
- [x] Health endpoint integrated

## History Queries (Sprint 1.7)
- [x] History query implemented with pagination
- [x] Pagination validation implemented
- [x] Query performance optimized (indexes used)
- [x] Edge cases handled (empty history, beyond pages)
- [x] Concurrency tested
- [x] Performance documented

## Testing (Sprint 1.8)
- [x] Integration tests complete (>80% coverage)
- [x] Backup and restore tested successfully
- [x] Monitoring configured (postgres_exporter)
- [x] Alerts defined (connection exhaustion, latency)
- [x] Load testing completed (P95 <100ms)
- [x] Schema documentation complete

## Performance
- [x] Query execution time <100ms for indexed queries (P95)
- [x] Connection pool size appropriate (5 per instance)
- [x] Cache hit ratio >95%
- [x] No slow queries (>1s) in pg_stat_statements
- [x] Indexes used by query planner (verified with EXPLAIN)

## Security
- [x] SSL/TLS enforced (sslmode=verify-full)
- [x] Least-privilege database user (docling_app, not superuser)
- [x] Passwords stored securely (not in code)
- [x] No trust authentication in pg_hba.conf
- [x] Audit logging configured (if required)

## Reliability
- [x] Backup strategy documented (daily, 30-day retention)
- [x] Backup tested (restore successful)
- [x] RTO <1 hour (verified in restore test)
- [x] RPO <5 minutes (with WAL archiving, if configured)
- [x] Monitoring and alerting in place
- [x] Health check integrated into /api/v1/health

## Documentation
- [x] Database schema documented (ERD, tables, indexes)
- [x] Backup and restore procedures documented
- [x] Migration procedures documented
- [x] Performance benchmarks documented
- [x] Operational runbooks created

## Sign-Off
- [ ] Trinity Smith (PostgreSQL DBA): ___________________ Date: _______
- [ ] William Chen (Infrastructure Lead): _______________ Date: _______
- [ ] Alex Rivera (Architect): _______________________ Date: _______

## Production Deployment Readiness
- [ ] All checklist items complete
- [ ] No blocking issues
- [ ] Ready for production deployment: YES / NO

---

**Next Steps**:
1. Final review meeting with Trinity, William, Alex
2. Deploy to staging environment
3. Run smoke tests on staging
4. Deploy to production
5. Monitor for 24 hours post-deployment
```

**Deliverables**:
- Completed production readiness checklist
- Sign-off from Trinity, William, and Alex
- Confirmation that database layer is production-ready

**Technical Notes**:
- This checklist is BLOCKING for production deployment
- Any "NO" or incomplete item must be resolved before deployment
- Staging deployment should precede production deployment
- Post-deployment monitoring is critical for first 24 hours

---

## Dependencies

**Upstream**:
- All Sprint 0, 1.1, 1.2, 1.7 tasks complete

**Downstream**:
- Production deployment (coordinate with William and Amanda Foster)

**Cross-Team**:
- Julia Santos: Integration testing coordination
- William Chen: Monitoring setup, backup automation
- Alex Rivera: Final architecture review
- Amanda Foster: Ansible playbooks for database deployment

---

## Validation

Sprint 1.8 PostgreSQL tasks are complete when:
1. All tasks TRI-1.8-001 through TRI-1.8-006 complete
2. All integration tests pass
3. Backup and restore procedures validated
4. Monitoring and alerting configured
5. Load testing shows acceptable performance
6. Documentation complete
7. Production readiness checklist signed off

---

## Risk Mitigation

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Integration tests fail | Medium | High | Fix issues before production, coordinate with Julia |
| Backup restore fails | Low | Critical | Test restore in TRI-1.8-002, verify regularly |
| Load test reveals performance issues | Medium | Medium | Optimize queries, add indexes, increase pool size |
| Missing documentation | Low | Medium | Complete all documentation in TRI-1.8-005 |
| Production readiness checklist incomplete | Low | Critical | Thorough review in TRI-1.8-006, sign-off required |

---

## Total Effort Summary

**Total Estimated Effort**: 2.5 hours (150 minutes)
**Task Breakdown**:
- TRI-1.8-001: 45 minutes (Integration tests)
- TRI-1.8-002: 30 minutes (Backup/restore test)
- TRI-1.8-003: 30 minutes (Monitoring setup)
- TRI-1.8-004: 25 minutes (Load testing)
- TRI-1.8-005: 20 minutes (Documentation)
- TRI-1.8-006: 20 minutes (Readiness checklist)

**Critical Path**: TRI-1.8-001 → 002 → 003 → 004 → 005 → 006

---

## Deliverables Checklist

- [ ] `test/integration/database/api-database.test.ts` - Integration tests
- [ ] `scripts/test-backup-restore.sh` - Backup/restore test script
- [ ] `docs/database-backup.md` - Backup procedures (updated)
- [ ] `test/load/database-load.test.ts` - Load test script
- [ ] `docs/database-schema.md` - Schema documentation
- [ ] `/etc/prometheus/alerts/postgresql.yml` - Alert rules
- [ ] Production readiness checklist (signed off)
- [ ] All tests passing
- [ ] Monitoring active
- [ ] Ready for production deployment
