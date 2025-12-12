# Sprint 1.2: PostgreSQL Integration and Migrations

**Sprint**: Sprint 1.2 (Database & Redis)
**Agent**: Trinity Smith (PostgreSQL DBA SME)
**Duration**: 3.5 hours
**Dependencies**: Sprint 1.1 Prisma schema complete
**References**:
- Implementation Plan Section 4.3 (Sprint 1.2)
- Specification Section 5.1 (Database Schema)
- Specification Section 5.1.1 (Connection Validation)
- Specification Section 5.1.3 (Connection Pool Configuration)
- Specification Section 7.3 (PostgreSQL Integration)

---

## Task Overview

Implement PostgreSQL database integration for the hx-docling-application: create Prisma client singleton with connection pooling, run initial migrations, implement health checks, validate indexes, test rollback procedures, and establish backup strategy. Ensure production-grade configuration with SSL, connection pooling via PgBouncer, and comprehensive error handling.

---

## Tasks

### TRI-1.2-001: Create Prisma Client Singleton with Connection Pooling

**Priority**: P0 (Blocking)
**Effort**: 30 minutes
**Dependencies**: TRI-1.1-007 (Prisma schema complete)

**Description**:
Implement the Prisma client singleton pattern with proper connection pooling configuration, logging, and graceful shutdown handling. Configure for development and production environments with appropriate log levels.

**Acceptance Criteria**:
- [ ] File `src/lib/db/prisma.ts` created
- [ ] Singleton pattern prevents multiple PrismaClient instances in development (hot reload)
- [ ] Connection pooling parameters configured (connection_limit=5, pool_timeout=20s)
- [ ] Logging configured: query/error/warn in development, error only in production
- [ ] Graceful shutdown handler disconnects Prisma on process exit
- [ ] TypeScript types correctly exported
- [ ] ESLint passes with no warnings

**Implementation**:
```typescript
// src/lib/db/prisma.ts

import { PrismaClient } from '@prisma/client';

/**
 * Prisma Client Singleton
 *
 * Prevents multiple instances in development (Next.js hot reload).
 * Uses connection pooling via PgBouncer (port 6432).
 *
 * Connection Pool Configuration:
 * - connection_limit=5 (per instance, set in DATABASE_URL)
 * - pool_timeout=20s (connection acquisition timeout)
 *
 * Reference: Specification Section 7.3.1
 */

// Global storage for singleton in development
const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined;
};

// Create or reuse PrismaClient
export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    log:
      process.env.NODE_ENV === 'development'
        ? ['query', 'error', 'warn']
        : ['error'],

    // Error formatting for better debugging
    errorFormat: process.env.NODE_ENV === 'development' ? 'pretty' : 'minimal',
  });

// Store singleton in development to survive hot reload
if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = prisma;
}

/**
 * Graceful shutdown: disconnect Prisma on process exit
 * Ensures connections are properly closed.
 */
async function gracefulShutdown() {
  await prisma.$disconnect();
  console.log('[Database] Prisma disconnected');
}

// Register shutdown handlers
if (typeof process !== 'undefined') {
  process.on('SIGINT', gracefulShutdown);
  process.on('SIGTERM', gracefulShutdown);
  process.on('beforeExit', gracefulShutdown);
}

// Export Prisma types for application use
export type { Job, Result, JobStatus, Prisma } from '@prisma/client';
```

**Deliverables**:
- `src/lib/db/prisma.ts` file created
- Singleton pattern implemented
- Graceful shutdown handlers registered
- TypeScript types exported

**Technical Notes**:
- Singleton pattern prevents "too many clients" errors during Next.js hot reload
- `connection_limit=5` is set in `DATABASE_URL`, not in PrismaClient options
- `pool_timeout=20` (seconds) prevents indefinite blocking on pool exhaustion
- Logging in development helps debug query issues; production uses error-only for performance
- Graceful shutdown ensures connections are released on server stop

---

### TRI-1.2-002: Configure DATABASE_URL Environment Variables

**Priority**: P0 (Blocking)
**Effort**: 15 minutes
**Dependencies**: TRI-0-006 (Connection string template)

**Description**:
Configure the `.env` file with `DATABASE_URL` and `DIRECT_DATABASE_URL` using the validated connection strings from Sprint 0. Ensure SSL certificate path is correct and connection parameters match production requirements.

**Acceptance Criteria**:
- [ ] `.env` file created (or updated) with database connection strings
- [ ] `DATABASE_URL` uses PgBouncer (port 6432)
- [ ] `DIRECT_DATABASE_URL` uses direct PostgreSQL (port 5432)
- [ ] `sslmode=verify-full` and `sslrootcert` path configured
- [ ] `connection_limit=5` and `pool_timeout=20` in DATABASE_URL
- [ ] Passwords are sourced from environment or secrets manager (not hardcoded)
- [ ] `.env.example` updated with template (passwords redacted)

**Implementation**:
```bash
# .env (local development - NOT committed to git)
# Copy from .env.example and fill in actual passwords

# ============================================================
# PostgreSQL Configuration
# ============================================================
# Application connection via PgBouncer (port 6432)
DATABASE_URL="postgresql://docling_app:ACTUAL_PASSWORD@hx-postgres-server.hx.dev.local:6432/docling_db?connection_limit=5&pool_timeout=20&sslmode=verify-full&sslrootcert=/etc/ssl/certs/hx-postgres-ca.crt"

# Direct connection for migrations (port 5432)
DIRECT_DATABASE_URL="postgresql://docling_migration:ACTUAL_MIGRATION_PASSWORD@hx-postgres-server.hx.dev.local:5432/docling_db?sslmode=verify-full&sslrootcert=/etc/ssl/certs/hx-postgres-ca.crt"

# Connection pool settings (documented for reference)
DB_POOL_SIZE=5
DB_POOL_TIMEOUT_MS=20000
```

**Deliverables**:
- `.env` file configured with actual credentials
- `.env.example` updated with template (passwords as `${DB_PASSWORD}`)
- SSL certificate path verified to match system

**Technical Notes**:
- `.env` should be in `.gitignore` to prevent password leaks
- Production passwords should come from environment variables or secrets manager (Ansible Vault, HashiCorp Vault)
- Verify SSL certificate path: `ls -la /etc/ssl/certs/hx-postgres-ca.crt`
- If certificate path differs, update connection strings accordingly

---

### TRI-1.2-003: Create Initial Prisma Migration

**Priority**: P0 (Blocking)
**Effort**: 20 minutes
**Dependencies**: TRI-1.2-001, TRI-1.2-002

**Description**:
Generate and apply the initial Prisma migration that creates the `jobs` and `results` tables, `JobStatus` enum, and all indexes in the PostgreSQL database. Verify migration files for correctness before applying.

**Acceptance Criteria**:
- [ ] Migration created: `npx prisma migrate dev --name init`
- [ ] Migration file generated in `prisma/migrations/YYYYMMDDHHMMSS_init/migration.sql`
- [ ] Migration SQL reviewed and validated (CREATE TABLE, ENUM, INDEX statements)
- [ ] Migration applied successfully to database
- [ ] Tables `jobs` and `results` exist in database
- [ ] Enum `JobStatus` created with all values
- [ ] Indexes created on `sessionId`, `status`, `createdAt`, `jobId`
- [ ] `_prisma_migrations` table created for migration tracking

**Commands**:
```bash
# Generate and apply migration
npx prisma migrate dev --name init

# Verify migration was applied
npx prisma migrate status

# Inspect generated SQL
cat prisma/migrations/*/migration.sql

# Verify tables in database
psql "$DATABASE_URL" -c "\dt"
psql "$DATABASE_URL" -c "\d jobs"
psql "$DATABASE_URL" -c "\d results"
psql "$DATABASE_URL" -c "\dT JobStatus"
```

**Deliverables**:
- Migration file `prisma/migrations/YYYYMMDDHHMMSS_init/migration.sql`
- Tables `jobs` and `results` created in database
- Migration status output showing success

**Technical Notes**:
- `migrate dev` uses `DIRECT_DATABASE_URL` (port 5432) to bypass PgBouncer (DDL operations need direct connection)
- Review generated SQL before applying to catch any schema issues
- Migration is idempotent: re-running `migrate dev` will not duplicate tables
- If migration fails, check error message and fix schema issues in `prisma/schema.prisma`
- Migration history is tracked in `_prisma_migrations` table

---

### TRI-1.2-004: Test Migration Rollback Procedure

**Priority**: P1 (High)
**Effort**: 15 minutes
**Dependencies**: TRI-1.2-003

**Description**:
Test the Prisma migration rollback procedure using `prisma migrate reset` to ensure that migrations can be safely reverted in development and staging environments. Document rollback steps for production use.

**Acceptance Criteria**:
- [ ] Test database reset: `npx prisma migrate reset --skip-seed` succeeds
- [ ] All tables dropped and recreated
- [ ] Migration re-applied successfully
- [ ] Database returns to clean state
- [ ] Rollback procedure documented for production
- [ ] Production rollback warnings documented (destructive operation)

**Commands**:
```bash
# WARNING: This drops all data. Only run in development/staging.
npx prisma migrate reset --skip-seed

# Verify reset worked (tables recreated)
npx prisma migrate status
psql "$DATABASE_URL" -c "\dt"

# For production rollback (manual steps):
# 1. Identify migration to rollback: psql -c "SELECT * FROM _prisma_migrations ORDER BY finished_at DESC;"
# 2. Manually write and test DOWN migration SQL
# 3. Apply in transaction with backup ready
# 4. Update _prisma_migrations table to remove reverted migration
```

**Deliverables**:
- Successful reset test output
- Rollback procedure documented in `prisma/ROLLBACK.md`
- Production rollback warnings and checklist

**Technical Notes**:
- `migrate reset` is DESTRUCTIVE: it drops all tables and data
- NEVER run `migrate reset` in production
- Production rollback requires manual intervention: backup → manual DOWN SQL → restore if needed
- For production, prefer forward fixes (new migration) over rollback when possible
- Document rollback steps in `prisma/ROLLBACK.md` for emergency use

---

### TRI-1.2-005: Implement Database Health Check Functions

**Priority**: P1 (High)
**Effort**: 30 minutes
**Dependencies**: TRI-1.2-001

**Description**:
Implement database health check functions per Specification Section 5.1.1: `validateDatabaseConnection()` for startup checks and `ensureDatabaseConnection()` for fail-fast behavior on application startup.

**Acceptance Criteria**:
- [ ] File `src/lib/db/health.ts` created
- [ ] `validateDatabaseConnection()` executes `SELECT 1` and measures latency
- [ ] `ensureDatabaseConnection()` retries 3 times with 2s delay, exits on failure
- [ ] Health check returns status object: `{ healthy: boolean, latencyMs: number, error?: string }`
- [ ] Error handling logs meaningful messages
- [ ] TypeScript types exported
- [ ] Unit tests pass (mock Prisma for testing)

**Implementation**:
```typescript
// src/lib/db/health.ts

import { prisma } from './prisma';

/**
 * Database Health Status
 */
export interface DatabaseHealthStatus {
  healthy: boolean;
  latencyMs: number;
  error?: string;
}

/**
 * Validate database connection health.
 * Executes a simple query and measures latency.
 *
 * Reference: Specification Section 5.1.1
 */
export async function validateDatabaseConnection(): Promise<DatabaseHealthStatus> {
  const start = Date.now();

  try {
    // Simple query to validate connection
    await prisma.$queryRaw`SELECT 1`;

    return {
      healthy: true,
      latencyMs: Date.now() - start,
    };
  } catch (error) {
    const errorMessage = error instanceof Error ? error.message : 'Unknown database error';

    console.error('[Database] Connection validation failed:', errorMessage);

    return {
      healthy: false,
      latencyMs: Date.now() - start,
      error: errorMessage,
    };
  }
}

/**
 * Ensure database connection on startup (fail-fast).
 * Retries 3 times with 2-second delay, exits process on failure.
 *
 * Call from instrumentation.ts or application startup.
 */
export async function ensureDatabaseConnection(): Promise<void> {
  const MAX_RETRIES = 3;
  const RETRY_DELAY_MS = 2000;

  for (let attempt = 1; attempt <= MAX_RETRIES; attempt++) {
    const status = await validateDatabaseConnection();

    if (status.healthy) {
      console.log(`[Database] Connection established (${status.latencyMs}ms)`);
      return;
    }

    if (attempt < MAX_RETRIES) {
      console.warn(
        `[Database] Retry ${attempt}/${MAX_RETRIES} in ${RETRY_DELAY_MS}ms... Error: ${status.error}`
      );
      await new Promise((resolve) => setTimeout(resolve, RETRY_DELAY_MS));
    }
  }

  console.error('[Database] Failed to connect after maximum retries. Exiting.');
  process.exit(1); // Fail-fast: stop application if database is unavailable
}
```

**Deliverables**:
- `src/lib/db/health.ts` file created
- Health check functions implemented
- Error handling and logging in place
- Ready for integration into `/api/v1/health` endpoint

**Technical Notes**:
- `SELECT 1` is a lightweight query that tests connectivity without loading data
- Latency measurement helps diagnose network or database performance issues
- `process.exit(1)` fail-fast behavior prevents application from starting with broken database
- Health check should be called from `instrumentation.ts` (Next.js startup hook) or `server.ts`
- Coordinate with William Chen to integrate into `/api/v1/health` endpoint

---

### TRI-1.2-006: Verify Database Indexes Exist and Are Used

**Priority**: P1 (High)
**Effort**: 15 minutes
**Dependencies**: TRI-1.2-003

**Description**:
Verify that all indexes defined in the Prisma schema were created in PostgreSQL and test that they are used by query planner for expected queries (sessionId, status, createdAt).

**Acceptance Criteria**:
- [ ] Indexes exist in database: `\d jobs` shows indexes on `sessionId`, `status`, `createdAt`
- [ ] Index on `results.jobId` exists
- [ ] Composite index `jobs_sessionId_createdAt_idx` exists
- [ ] EXPLAIN ANALYZE shows indexes are used for history queries
- [ ] No sequential scans on indexed queries
- [ ] Index usage documented

**Commands**:
```bash
# Verify indexes exist
psql "$DATABASE_URL" -c "\d jobs"
psql "$DATABASE_URL" -c "\d results"

# Test query plan for history query (should use index)
psql "$DATABASE_URL" << EOF
EXPLAIN ANALYZE
SELECT id, status, createdAt, fileName
FROM jobs
WHERE sessionId = 'test-session'
ORDER BY createdAt DESC
LIMIT 20;
EOF

# Test query plan for status filter (should use index)
psql "$DATABASE_URL" << EOF
EXPLAIN ANALYZE
SELECT id, status, createdAt
FROM jobs
WHERE status = 'COMPLETE'
ORDER BY createdAt DESC
LIMIT 20;
EOF
```

**Deliverables**:
- Index verification output showing all indexes exist
- EXPLAIN ANALYZE output showing index scans (not sequential scans)
- Documentation of index usage patterns

**Technical Notes**:
- Indexes should show in `\d jobs` output as: `jobs_sessionId_idx`, `jobs_status_idx`, `jobs_createdAt_idx`
- Composite index `jobs_sessionId_createdAt_idx` supports sessionId + ORDER BY createdAt queries
- EXPLAIN ANALYZE should show "Index Scan" or "Index Only Scan", NOT "Seq Scan" on large tables
- If indexes are missing, check migration SQL and re-run `npx prisma migrate dev`
- Indexes are critical for performance at scale (>1000 jobs)

---

### TRI-1.2-007: Test All Database CRUD Operations

**Priority**: P1 (High)
**Effort**: 30 minutes
**Dependencies**: TRI-1.2-001, TRI-1.2-003

**Description**:
Write and execute comprehensive tests for all CRUD operations (Create, Read, Update, Delete) on Job and Result models. Verify that Prisma client works correctly with the database.

**Acceptance Criteria**:
- [ ] CREATE: Job and Result records created successfully
- [ ] READ: Jobs queried by sessionId, status, and createdAt
- [ ] UPDATE: Job status updated, fields modified
- [ ] DELETE: Cascade delete works (deleting Job deletes Results)
- [ ] Foreign key constraint enforced (cannot create Result without Job)
- [ ] Transactions work correctly
- [ ] Error handling tested (duplicate ID, missing foreign key)

**Test Script**:
```typescript
// test/database/crud.test.ts

import { prisma } from '@/lib/db/prisma';
import { describe, it, expect, beforeEach, afterEach } from 'vitest';

describe('Database CRUD Operations', () => {
  beforeEach(async () => {
    // Clean up test data
    await prisma.result.deleteMany();
    await prisma.job.deleteMany();
  });

  afterEach(async () => {
    // Clean up after tests
    await prisma.result.deleteMany();
    await prisma.job.deleteMany();
  });

  it('CREATE: creates job with all fields', async () => {
    const job = await prisma.job.create({
      data: {
        sessionId: 'test-session',
        status: 'PENDING',
        inputType: 'FILE',
        fileName: 'test.pdf',
        fileSize: BigInt(1024),
        fileMime: 'application/pdf',
      },
    });

    expect(job.id).toBeDefined();
    expect(job.sessionId).toBe('test-session');
    expect(job.status).toBe('PENDING');
  });

  it('READ: queries jobs by sessionId', async () => {
    await prisma.job.createMany({
      data: [
        { sessionId: 'session-1', status: 'COMPLETE', inputType: 'FILE' },
        { sessionId: 'session-1', status: 'FAILED', inputType: 'FILE' },
        { sessionId: 'session-2', status: 'COMPLETE', inputType: 'URL' },
      ],
    });

    const jobs = await prisma.job.findMany({
      where: { sessionId: 'session-1' },
      orderBy: { createdAt: 'desc' },
    });

    expect(jobs).toHaveLength(2);
    expect(jobs[0].sessionId).toBe('session-1');
  });

  it('UPDATE: updates job status', async () => {
    const job = await prisma.job.create({
      data: {
        sessionId: 'test',
        status: 'PENDING',
        inputType: 'FILE',
      },
    });

    const updated = await prisma.job.update({
      where: { id: job.id },
      data: { status: 'PROCESSING', startedAt: new Date() },
    });

    expect(updated.status).toBe('PROCESSING');
    expect(updated.startedAt).toBeDefined();
  });

  it('DELETE: cascade deletes results when job deleted', async () => {
    const job = await prisma.job.create({
      data: {
        sessionId: 'test',
        status: 'COMPLETE',
        inputType: 'FILE',
      },
    });

    await prisma.result.create({
      data: {
        jobId: job.id,
        formatType: 'markdown',
        content: '# Test',
        sizeBytes: 6,
      },
    });

    // Delete job (should cascade to results)
    await prisma.job.delete({ where: { id: job.id } });

    const results = await prisma.result.findMany({ where: { jobId: job.id } });
    expect(results).toHaveLength(0); // Results deleted
  });

  it('TRANSACTION: rolls back on error', async () => {
    await expect(
      prisma.$transaction(async (tx) => {
        await tx.job.create({
          data: { sessionId: 'test', status: 'PENDING', inputType: 'FILE' },
        });

        // Force error to test rollback
        throw new Error('Rollback test');
      })
    ).rejects.toThrow('Rollback test');

    // Job should not exist (rolled back)
    const jobs = await prisma.job.findMany();
    expect(jobs).toHaveLength(0);
  });
});
```

**Deliverables**:
- Test file `test/database/crud.test.ts` created
- All CRUD tests pass
- Cascade delete verified
- Transaction rollback verified

**Technical Notes**:
- Tests should use a dedicated test database or clean up data in `beforeEach`/`afterEach`
- Cascade delete is critical: deleting Job should delete associated Results
- Foreign key constraint prevents orphaned Results (Result without Job)
- Transactions ensure atomicity: all operations succeed or all fail

---

### TRI-1.2-008: Validate PgBouncer Connection Pooling Behavior

**Priority**: P1 (High)
**Effort**: 15 minutes
**Dependencies**: TRI-1.2-001, TRI-1.2-002

**Description**:
Test that the application correctly uses PgBouncer connection pooling and verify pool behavior under load. Ensure that connection limits are respected and connections are reused.

**Acceptance Criteria**:
- [ ] Application connects via PgBouncer (port 6432), not direct PostgreSQL (port 5432)
- [ ] Connection string in DATABASE_URL uses port 6432
- [ ] Multiple queries reuse pooled connections
- [ ] Pool exhaustion handled gracefully (queries wait for available connection)
- [ ] Connection timeout (20s) prevents indefinite blocking
- [ ] PgBouncer stats show active connections

**Test Commands**:
```bash
# Verify application uses port 6432
grep "6432" .env

# Run queries and check PgBouncer stats
psql -h hx-postgres-server.hx.dev.local -p 6432 -U docling_app -d pgbouncer -c "SHOW POOLS;" || echo "PgBouncer admin not accessible"

# Test connection pooling with multiple queries
node -e "
const { PrismaClient } = require('@prisma/client');
const prisma = new PrismaClient();

(async () => {
  const promises = Array.from({ length: 10 }, (_, i) =>
    prisma.\$queryRaw\`SELECT \${i} AS query_id, pg_backend_pid() AS pid\`
  );

  const results = await Promise.all(promises);
  console.log('Backend PIDs (should reuse connections):', results.map(r => r[0].pid));
  await prisma.\$disconnect();
})();
"
```

**Deliverables**:
- Confirmation that DATABASE_URL uses port 6432
- Test output showing connection pooling behavior
- PgBouncer statistics (if accessible)

**Technical Notes**:
- PgBouncer `pool_mode=transaction` means connections are released after each transaction
- `connection_limit=5` in DATABASE_URL limits Prisma to 5 connections
- If multiple queries use same backend PID, connection pooling is working
- Pool exhaustion causes queries to wait up to `pool_timeout=20s` before failing
- Coordinate with William Chen if PgBouncer is misconfigured

---

### TRI-1.2-009: Create Backup Procedure Before Initial Data Load

**Priority**: P1 (High)
**Effort**: 20 minutes
**Dependencies**: TRI-1.2-003

**Description**:
Document and execute a database backup procedure before loading initial test data. Establish backup strategy for development, staging, and production environments per Specification Section 5.1.3.

**Acceptance Criteria**:
- [ ] Backup procedure documented in `docs/database-backup.md`
- [ ] Base backup created with `pg_dump`
- [ ] Backup tested with restore to verify integrity
- [ ] Backup retention policy documented (7 days dev, 30 days prod)
- [ ] Automated backup schedule planned (coordinate with William)
- [ ] Point-in-time recovery (PITR) strategy documented

**Backup Commands**:
```bash
# Create backup directory
mkdir -p /data/backups/docling-db

# Base backup with pg_dump (schema + data)
pg_dump -h hx-postgres-server.hx.dev.local -p 5432 -U postgres -d docling_db \
  --format=custom --compress=9 --file=/data/backups/docling-db/docling-db-$(date +%Y%m%d-%H%M%S).dump

# Schema-only backup (for documentation)
pg_dump -h hx-postgres-server.hx.dev.local -p 5432 -U postgres -d docling_db \
  --schema-only --file=/data/backups/docling-db/schema-$(date +%Y%m%d).sql

# Test restore to verify backup integrity (on test database)
createdb -h hx-postgres-server.hx.dev.local -U postgres docling_db_test
pg_restore -h hx-postgres-server.hx.dev.local -p 5432 -U postgres -d docling_db_test \
  /data/backups/docling-db/docling-db-YYYYMMDD-HHMMSS.dump

# Clean up test database
dropdb -h hx-postgres-server.hx.dev.local -U postgres docling_db_test
```

**Backup Strategy Documentation**:
```markdown
# Database Backup Strategy

## Backup Types

1. **Base Backups** (pg_dump):
   - Frequency: Daily (3 AM)
   - Retention: 7 days (development), 30 days (production)
   - Format: Custom compressed (pg_dump --format=custom)
   - Location: /data/backups/docling-db/

2. **WAL Archiving** (Point-in-Time Recovery):
   - Frequency: Continuous
   - Retention: 7 days
   - Location: /data/wal-archive/docling-db/
   - Enables recovery to any point within retention window

## Restore Procedures

### Full Restore (from base backup):
```bash
pg_restore -h hx-postgres-server.hx.dev.local -U postgres -d docling_db \
  /data/backups/docling-db/docling-db-YYYYMMDD.dump
```

### Point-in-Time Recovery (PITR):
[Document PITR procedure if WAL archiving is configured]

## Recovery Objectives

- RTO (Recovery Time Objective): < 1 hour
- RPO (Recovery Point Objective): < 5 minutes (with WAL archiving)

## Testing

- Backup integrity tested monthly (restore to staging environment)
- Disaster recovery drill quarterly
```

**Deliverables**:
- Backup documentation in `docs/database-backup.md`
- Initial base backup created and verified
- Restore tested successfully
- Backup automation plan (coordinate with William)

**Technical Notes**:
- `pg_dump --format=custom` creates compressed binary backup (faster restore than SQL)
- WAL archiving requires PostgreSQL configuration: `archive_mode=on`, `archive_command` (coordinate with William)
- Production backups should be stored off-site (S3, network storage)
- Never rely on untested backups: restore test is critical
- Coordinate with William Chen for backup automation (cron, Ansible playbook)

---

### TRI-1.2-010: Integration Test with Health Check Endpoint

**Priority**: P2 (Medium)
**Effort**: 20 minutes
**Dependencies**: TRI-1.2-005, William's health endpoint implementation

**Description**:
Integrate database health check functions into the `/api/v1/health` endpoint and verify that health status is correctly reported. Coordinate with William Chen on health endpoint implementation.

**Acceptance Criteria**:
- [ ] `/api/v1/health` endpoint returns database status
- [ ] Response includes `database: { healthy: true/false, latencyMs: number }`
- [ ] Unhealthy status (e.g., database down) returns HTTP 503
- [ ] Health check executes in <500ms
- [ ] Integration test passes

**Health Endpoint Integration**:
```typescript
// app/api/v1/health/route.ts (coordinated with William)

import { NextResponse } from 'next/server';
import { validateDatabaseConnection } from '@/lib/db/health';
import { validateRedisConnection } from '@/lib/redis/health'; // William's implementation

export async function GET() {
  const [dbHealth, redisHealth] = await Promise.all([
    validateDatabaseConnection(),
    validateRedisConnection(),
  ]);

  const isHealthy = dbHealth.healthy && redisHealth.healthy;

  return NextResponse.json(
    {
      status: isHealthy ? 'healthy' : 'degraded',
      timestamp: new Date().toISOString(),
      services: {
        database: {
          healthy: dbHealth.healthy,
          latencyMs: dbHealth.latencyMs,
          error: dbHealth.error,
        },
        redis: {
          healthy: redisHealth.healthy,
          latencyMs: redisHealth.latencyMs,
          error: redisHealth.error,
        },
      },
    },
    { status: isHealthy ? 200 : 503 }
  );
}
```

**Test**:
```bash
# Test health endpoint
curl http://localhost:3000/api/v1/health | jq

# Expected output:
# {
#   "status": "healthy",
#   "timestamp": "2025-12-12T10:00:00.000Z",
#   "services": {
#     "database": { "healthy": true, "latencyMs": 5 },
#     "redis": { "healthy": true, "latencyMs": 2 }
#   }
# }
```

**Deliverables**:
- Database health integrated into `/api/v1/health` endpoint
- Successful health check test
- HTTP 503 returned on unhealthy status

**Technical Notes**:
- Coordinate with William Chen to ensure consistent health endpoint format
- Health check should be fast (<500ms) to avoid blocking monitoring probes
- HTTP 503 status signals load balancers to remove unhealthy instances
- Health endpoint used by Kubernetes liveness/readiness probes (future)

---

## Dependencies

**Upstream**:
- Sprint 1.1 Prisma schema complete (TRI-1.1-007)
- Sprint 0 PostgreSQL prerequisites validated (TRI-0-007)

**Downstream**:
- Sprint 1.3 File upload (uses Job model)
- Sprint 1.7 History queries (uses Job/Result models)
- Sprint 1.8 Database testing (integration tests)

**Cross-Team**:
- William Chen: Health endpoint integration, backup automation, PgBouncer configuration
- Neo Anderson: Prisma client usage in API routes, environment variable configuration

---

## Validation

Sprint 1.2 PostgreSQL tasks are complete when:
1. All tasks TRI-1.2-001 through TRI-1.2-010 complete
2. Prisma client singleton works correctly
3. Initial migration applied and verified
4. All indexes exist and are used by queries
5. Health check integrated into `/api/v1/health`
6. Backup procedure documented and tested
7. Connection pooling via PgBouncer validated

---

## Risk Mitigation

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Migration fails on apply | Low | High | Review SQL before applying, test rollback |
| Connection pool exhausted | Medium | Medium | Monitor pool usage, increase limit if needed |
| SSL certificate path incorrect | Medium | High | Verify path in TRI-1.2-002, test connection |
| PgBouncer misconfigured | Low | High | Validate in TRI-1.2-008, coordinate with William |
| Backup restore fails | Low | Critical | Test restore in TRI-1.2-009 |

---

## Total Effort Summary

**Total Estimated Effort**: 3.5 hours (210 minutes)
**Task Breakdown**:
- TRI-1.2-001: 30 minutes (Prisma singleton)
- TRI-1.2-002: 15 minutes (Environment config)
- TRI-1.2-003: 20 minutes (Initial migration)
- TRI-1.2-004: 15 minutes (Rollback test)
- TRI-1.2-005: 30 minutes (Health checks)
- TRI-1.2-006: 15 minutes (Index verification)
- TRI-1.2-007: 30 minutes (CRUD tests)
- TRI-1.2-008: 15 minutes (PgBouncer validation)
- TRI-1.2-009: 20 minutes (Backup procedure)
- TRI-1.2-010: 20 minutes (Health endpoint integration)

**Critical Path**: TRI-1.2-001 → 002 → 003 → 004 → 005 → 010

---

## Deliverables Checklist

- [ ] `src/lib/db/prisma.ts` - Prisma client singleton
- [ ] `src/lib/db/health.ts` - Health check functions
- [ ] `.env` - Database connection strings configured
- [ ] `.env.example` - Template with redacted passwords
- [ ] `prisma/migrations/` - Initial migration
- [ ] `docs/database-backup.md` - Backup strategy
- [ ] `test/database/crud.test.ts` - CRUD tests
- [ ] `/api/v1/health` - Health endpoint with database status
- [ ] Database tables created and indexed
- [ ] Backup created and restore tested
