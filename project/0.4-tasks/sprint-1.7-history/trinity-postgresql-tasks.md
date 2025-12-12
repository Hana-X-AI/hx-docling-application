# Sprint 1.7: History Queries and Data Persistence

**Sprint**: Sprint 1.7 (History)
**Agent**: Trinity Smith (PostgreSQL DBA SME)
**Duration**: 2.0 hours
**Dependencies**: Sprint 1.2 database integration complete
**References**:
- Implementation Plan Section 4.9 (Sprint 1.7)
- Specification Section 4.5 (History Endpoint)
- Specification Section 5.1.2 (Job Status State Machine)
- Specification Section 2.7 (History View Requirements)

---

## Task Overview

Implement efficient database queries for the history view, optimize query performance with proper indexes, implement pagination, handle edge cases (empty history, failed jobs), and ensure data consistency. Support filtering by session ID, sorting by creation time, and efficient retrieval of related results.

---

## Tasks

### TRI-1.7-001: Design History Query with Pagination

**Priority**: P0 (Blocking)
**Effort**: 30 minutes
**Dependencies**: TRI-1.2-007 (CRUD operations tested)

**Description**:
Design the SQL query for the history endpoint that retrieves jobs for a given session ID with pagination, sorting, and related results. Ensure the query uses indexes efficiently and returns data in the format expected by the frontend.

**Acceptance Criteria**:
- [ ] Query retrieves jobs by `sessionId` with pagination (page, pageSize)
- [ ] Results sorted by `createdAt DESC` (newest first)
- [ ] Related results included (eager loading with Prisma `include`)
- [ ] Query uses indexes (sessionId, createdAt composite index)
- [ ] Total count returned for pagination UI
- [ ] Query execution time <100ms for typical dataset (100 jobs)
- [ ] EXPLAIN ANALYZE shows index scan, not sequential scan

**Query Design**:
```typescript
// lib/db/queries/history.ts

import { prisma } from '@/lib/db/prisma';
import type { Job, Result } from '@prisma/client';

export interface HistoryQueryParams {
  sessionId: string;
  page: number;       // 1-indexed
  pageSize: number;   // 1-50, default 20
}

export interface HistoryResult {
  jobs: (Job & { results: Result[] })[];
  pagination: {
    page: number;
    pageSize: number;
    totalPages: number;
    totalCount: number;
    hasMore: boolean;
  };
}

/**
 * Query jobs for a session with pagination and related results.
 *
 * Uses composite index: jobs_sessionId_createdAt_idx
 * Reference: Specification Section 4.5
 */
export async function getJobHistory(
  params: HistoryQueryParams
): Promise<HistoryResult> {
  const { sessionId, page, pageSize } = params;
  const skip = (page - 1) * pageSize;

  // Parallel queries: jobs + total count
  const [jobs, totalCount] = await Promise.all([
    prisma.job.findMany({
      where: { sessionId },
      include: { results: true }, // Eager load results
      orderBy: { createdAt: 'desc' },
      skip,
      take: pageSize,
    }),
    prisma.job.count({
      where: { sessionId },
    }),
  ]);

  const totalPages = Math.ceil(totalCount / pageSize);

  return {
    jobs,
    pagination: {
      page,
      pageSize,
      totalPages,
      totalCount,
      hasMore: page < totalPages,
    },
  };
}
```

**Deliverables**:
- `lib/db/queries/history.ts` file created
- `getJobHistory()` function implemented
- TypeScript types defined for query params and result
- Query design documented

**Technical Notes**:
- Composite index `jobs_sessionId_createdAt_idx` supports WHERE sessionId + ORDER BY createdAt
- Eager loading with `include: { results: true }` avoids N+1 query problem
- Parallel execution of jobs query + count query improves performance
- `skip` and `take` implement offset pagination (suitable for <10,000 records)
- For very large datasets (>10,000 jobs), consider cursor-based pagination

---

### TRI-1.7-002: Implement Pagination Validation

**Priority**: P0 (Blocking)
**Effort**: 15 minutes
**Dependencies**: TRI-1.7-001

**Description**:
Implement input validation for pagination parameters (page, pageSize) using Zod schema per Specification Section 4.5. Ensure page is ≥1, pageSize is 1-50, and defaults are applied.

**Acceptance Criteria**:
- [ ] Zod schema validates page (min 1, default 1)
- [ ] Zod schema validates pageSize (min 1, max 50, default 20)
- [ ] Invalid inputs return validation errors
- [ ] Defaults applied when params omitted
- [ ] Coercion handles string inputs (query params)

**Implementation**:
```typescript
// lib/validation/pagination.ts

import { z } from 'zod';

/**
 * Pagination validation schema.
 * Reference: Specification Section 4.5 (MAJ-21)
 */
export const paginationSchema = z.object({
  page: z.coerce.number().int().min(1).default(1),
  pageSize: z.coerce.number().int().min(1).max(50).default(20),
});

export type PaginationParams = z.infer<typeof paginationSchema>;

/**
 * Validate and parse pagination parameters.
 * Returns validated params or throws ZodError.
 */
export function validatePagination(input: unknown): PaginationParams {
  return paginationSchema.parse(input);
}
```

**Test Cases**:
```typescript
// test/validation/pagination.test.ts

import { validatePagination } from '@/lib/validation/pagination';
import { describe, it, expect } from 'vitest';

describe('Pagination Validation', () => {
  it('applies defaults when params omitted', () => {
    const result = validatePagination({});
    expect(result.page).toBe(1);
    expect(result.pageSize).toBe(20);
  });

  it('accepts valid pagination params', () => {
    const result = validatePagination({ page: 2, pageSize: 10 });
    expect(result.page).toBe(2);
    expect(result.pageSize).toBe(10);
  });

  it('coerces string inputs to numbers', () => {
    const result = validatePagination({ page: '3', pageSize: '25' });
    expect(result.page).toBe(3);
    expect(result.pageSize).toBe(25);
  });

  it('rejects page < 1', () => {
    expect(() => validatePagination({ page: 0 })).toThrow();
    expect(() => validatePagination({ page: -1 })).toThrow();
  });

  it('rejects pageSize > 50', () => {
    expect(() => validatePagination({ pageSize: 51 })).toThrow();
    expect(() => validatePagination({ pageSize: 100 })).toThrow();
  });

  it('rejects non-integer values', () => {
    expect(() => validatePagination({ page: 1.5 })).toThrow();
    expect(() => validatePagination({ pageSize: 20.7 })).toThrow();
  });
});
```

**Deliverables**:
- `lib/validation/pagination.ts` file created
- Pagination validation schema implemented
- Unit tests pass

**Technical Notes**:
- `z.coerce.number()` handles string inputs from URL query params
- Max pageSize of 50 prevents large query result sets
- Default values simplify client code (no need to always specify page/pageSize)
- Validation errors should be caught in API route and return HTTP 400

---

### TRI-1.7-003: Optimize Query Performance with Index Analysis

**Priority**: P1 (High)
**Effort**: 20 minutes
**Dependencies**: TRI-1.7-001

**Description**:
Analyze the history query performance using EXPLAIN ANALYZE, verify that the composite index is used, and measure query execution time with various dataset sizes (10, 100, 1000 jobs).

**Acceptance Criteria**:
- [ ] EXPLAIN ANALYZE output shows "Index Scan using jobs_sessionId_createdAt_idx"
- [ ] No sequential scans on jobs table
- [ ] Query execution time <100ms for 100 jobs
- [ ] Query execution time <200ms for 1000 jobs
- [ ] Results included via JOIN (not N+1 queries)
- [ ] Performance benchmarks documented

**Analysis Commands**:
```bash
# Insert test data (100 jobs for session-1)
psql "$DATABASE_URL" << EOF
INSERT INTO jobs (id, "sessionId", status, "inputType", "createdAt")
SELECT
  gen_random_uuid(),
  'session-1',
  'COMPLETE',
  'FILE',
  NOW() - (random() * INTERVAL '30 days')
FROM generate_series(1, 100);
EOF

# Analyze query plan for pagination query
psql "$DATABASE_URL" << EOF
EXPLAIN ANALYZE
SELECT j.*, r.*
FROM jobs j
LEFT JOIN results r ON r."jobId" = j.id
WHERE j."sessionId" = 'session-1'
ORDER BY j."createdAt" DESC
LIMIT 20;
EOF

# Measure query execution time
psql "$DATABASE_URL" << EOF
\timing on
SELECT j.*, r.*
FROM jobs j
LEFT JOIN results r ON r."jobId" = j.id
WHERE j."sessionId" = 'session-1'
ORDER BY j."createdAt" DESC
LIMIT 20;
EOF
```

**Expected EXPLAIN Output**:
```
Limit  (cost=... rows=20 width=...)
  ->  Nested Loop Left Join  (cost=... rows=100 width=...)
        ->  Index Scan using jobs_sessionId_createdAt_idx on jobs j
              Index Cond: (sessionId = 'session-1'::text)
        ->  Index Scan using results_jobId_idx on results r
              Index Cond: (jobId = j.id)

Execution time: 15.234 ms
```

**Deliverables**:
- EXPLAIN ANALYZE output showing index usage
- Query execution time benchmarks (10, 100, 1000 jobs)
- Performance documentation in `docs/database-performance.md`

**Technical Notes**:
- Composite index `jobs_sessionId_createdAt_idx` covers WHERE + ORDER BY clauses
- Left join to results should use `results_jobId_idx` (foreign key index)
- If query is slow, check:
  - Are indexes present? (`\d jobs`)
  - Are statistics up to date? (run `ANALYZE jobs;`)
  - Is work_mem too low? (increase for complex sorts)
- Query time should scale logarithmically with dataset size (due to index)

---

### TRI-1.7-004: Implement Edge Case Handling (Empty History)

**Priority**: P1 (High)
**Effort**: 15 minutes
**Dependencies**: TRI-1.7-001

**Description**:
Ensure the history query correctly handles edge cases: empty history (no jobs for session), invalid sessionId, and pagination beyond available pages. Return appropriate responses for each case.

**Acceptance Criteria**:
- [ ] Query returns empty array when no jobs exist for sessionId
- [ ] Total count is 0 for empty history
- [ ] totalPages is 0 for empty history
- [ ] Requesting page beyond totalPages returns empty array (not error)
- [ ] Invalid sessionId format handled gracefully
- [ ] Edge cases tested

**Edge Case Handling**:
```typescript
// lib/db/queries/history.ts (add edge case handling)

export async function getJobHistory(
  params: HistoryQueryParams
): Promise<HistoryResult> {
  const { sessionId, page, pageSize } = params;
  const skip = (page - 1) * pageSize;

  // Parallel queries
  const [jobs, total] = await Promise.all([
    prisma.job.findMany({
      where: { sessionId },
      include: { results: true },
      orderBy: { createdAt: 'desc' },
      skip,
      take: pageSize,
    }),
    prisma.job.count({
      where: { sessionId },
    }),
  ]);

  const totalPages = total > 0 ? Math.ceil(total / pageSize) : 0;

  return {
    jobs,
    total,
    page,
    pageSize,
    totalPages,
  };
}
```

**Test Cases**:
```typescript
// test/db/queries/history.test.ts

describe('History Query Edge Cases', () => {
  it('returns empty array for non-existent session', async () => {
    const result = await getJobHistory({
      sessionId: 'non-existent-session',
      page: 1,
      pageSize: 20,
    });

    expect(result.jobs).toEqual([]);
    expect(result.total).toBe(0);
    expect(result.totalPages).toBe(0);
  });

  it('returns empty array when requesting page beyond totalPages', async () => {
    // Create 5 jobs for session-test
    await prisma.job.createMany({
      data: Array.from({ length: 5 }, () => ({
        sessionId: 'session-test',
        status: 'COMPLETE',
        inputType: 'FILE',
      })),
    });

    // Request page 2 with pageSize 10 (only 5 jobs exist)
    const result = await getJobHistory({
      sessionId: 'session-test',
      page: 2,
      pageSize: 10,
    });

    expect(result.jobs).toEqual([]);
    expect(result.total).toBe(5);
    expect(result.totalPages).toBe(1);
  });

  it('handles very long sessionId gracefully', async () => {
    const longSessionId = 'x'.repeat(300);

    const result = await getJobHistory({
      sessionId: longSessionId,
      page: 1,
      pageSize: 20,
    });

    expect(result.jobs).toEqual([]);
    expect(result.total).toBe(0);
  });
});
```

**Deliverables**:
- Edge case handling in `getJobHistory()`
- Unit tests for edge cases pass
- Documentation of expected behavior

**Technical Notes**:
- Empty history is a valid state (new session, or all jobs deleted)
- Requesting page beyond totalPages should not error (UX: show "no more results")
- SessionId validation should be in API route (not database layer)
- Graceful degradation: if database query fails, return error response (not crash)

---

### TRI-1.7-005: Implement Job Status Filtering (Optional Enhancement)

**Priority**: P2 (Medium)
**Effort**: 20 minutes
**Dependencies**: TRI-1.7-001

**Description**:
Add optional job status filtering to the history query to allow users to filter by status (e.g., show only COMPLETE jobs, or only FAILED jobs). Use the status index for efficient filtering.

**Acceptance Criteria**:
- [ ] Query accepts optional `status` filter parameter
- [ ] Status filter uses `jobs_status_idx` index
- [ ] Multiple status values supported (e.g., COMPLETE, FAILED)
- [ ] Filter validated against JobStatus enum
- [ ] Query performance maintained with status filter

**Implementation**:
```typescript
// lib/db/queries/history.ts (enhanced with status filter)

export interface HistoryQueryParams {
  sessionId: string;
  page: number;
  pageSize: number;
  status?: JobStatus | JobStatus[]; // Optional status filter
}

export async function getJobHistory(
  params: HistoryQueryParams
): Promise<HistoryResult> {
  const { sessionId, page, pageSize, status } = params;
  const skip = (page - 1) * pageSize;

  // Build where clause with optional status filter
  const where = {
    sessionId,
    ...(status && {
      status: Array.isArray(status) ? { in: status } : status,
    }),
  };

  const [jobs, total] = await Promise.all([
    prisma.job.findMany({
      where,
      include: { results: true },
      orderBy: { createdAt: 'desc' },
      skip,
      take: pageSize,
    }),
    prisma.job.count({ where }),
  ]);

  const totalPages = total > 0 ? Math.ceil(total / pageSize) : 0;

  return {
    jobs,
    total,
    page,
    pageSize,
    totalPages,
  };
}
```

**Validation**:
```typescript
// lib/validation/pagination.ts (add status filter)

import { JobStatus } from '@prisma/client';

export const historyQuerySchema = paginationSchema.extend({
  status: z.nativeEnum(JobStatus).or(z.array(z.nativeEnum(JobStatus))).optional(),
});
```

**Deliverables**:
- Status filter added to `getJobHistory()`
- Validation schema updated
- Tests for status filtering

**Technical Notes**:
- Status filter uses `jobs_status_idx` index for efficient filtering
- Multiple status values use `IN` clause in SQL
- Status filter is optional: omitting it returns all jobs
- Coordinate with Neo/Ola if UI needs status filter controls

---

### TRI-1.7-006: Test Data Consistency During Concurrent Operations

**Priority**: P1 (High)
**Effort**: 25 minutes
**Dependencies**: TRI-1.7-001

**Description**:
Test data consistency when jobs are created, updated, or deleted while history queries are running. Ensure that queries return consistent snapshots and handle race conditions gracefully.

**Acceptance Criteria**:
- [ ] Concurrent job creation does not cause duplicate results
- [ ] Concurrent job deletion does not cause missing foreign keys
- [ ] Pagination consistency: page 1 + page 2 don't overlap or skip records
- [ ] Transaction isolation level understood and documented
- [ ] Race conditions tested and documented

**Concurrency Test**:
```typescript
// test/db/queries/history-concurrency.test.ts

describe('History Query Concurrency', () => {
  it('handles concurrent job creation during pagination', async () => {
    // Create initial jobs
    await prisma.job.createMany({
      data: Array.from({ length: 30 }, () => ({
        sessionId: 'concurrent-test',
        status: 'COMPLETE',
        inputType: 'FILE',
      })),
    });

    // Query page 1
    const page1Promise = getJobHistory({
      sessionId: 'concurrent-test',
      page: 1,
      pageSize: 20,
    });

    // Concurrently create new jobs
    const createPromise = prisma.job.createMany({
      data: Array.from({ length: 10 }, () => ({
        sessionId: 'concurrent-test',
        status: 'PENDING',
        inputType: 'FILE',
      })),
    });

    const [page1, _created] = await Promise.all([page1Promise, createPromise]);

    // Query page 2
    const page2 = await getJobHistory({
      sessionId: 'concurrent-test',
      page: 2,
      pageSize: 20,
    });

    // Verify no duplicates between pages
    const page1Ids = new Set(page1.jobs.map((j) => j.id));
    const page2Ids = new Set(page2.jobs.map((j) => j.id));

    const overlap = [...page1Ids].filter((id) => page2Ids.has(id));
    expect(overlap).toHaveLength(0); // No duplicates
  });

  it('handles concurrent job deletion', async () => {
    const job = await prisma.job.create({
      data: {
        sessionId: 'delete-test',
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

    // Concurrent query + delete
    const [historyResult, _deleted] = await Promise.all([
      getJobHistory({ sessionId: 'delete-test', page: 1, pageSize: 20 }),
      prisma.job.delete({ where: { id: job.id } }),
    ]);

    // Query may or may not include deleted job (depends on timing)
    // But should NOT crash with foreign key errors
    expect(historyResult).toBeDefined();
  });
});
```

**Deliverables**:
- Concurrency tests written and passing
- Transaction isolation level documented
- Race condition handling validated

**Technical Notes**:
- PostgreSQL default isolation level: READ COMMITTED (each query sees committed data)
- Pagination consistency: new jobs inserted during pagination may appear in unexpected positions
- Cascade delete prevents orphaned Results when Job is deleted
- For strict consistency, consider REPEATABLE READ isolation or cursor-based pagination
- Document known race conditions and acceptable behavior

---

### TRI-1.7-007: Document History Query Performance and Limitations

**Priority**: P2 (Medium)
**Effort**: 15 minutes
**Dependencies**: TRI-1.7-003

**Description**:
Document the performance characteristics of the history query, index usage, query execution times, and known limitations (e.g., offset pagination not suitable for very large datasets).

**Acceptance Criteria**:
- [ ] Performance benchmarks documented (10, 100, 1000, 10000 jobs)
- [ ] Index usage explained
- [ ] Pagination limitations documented (offset vs cursor)
- [ ] Recommendations for scaling beyond 10,000 jobs
- [ ] Query optimization tips included

**Documentation**:
```markdown
# History Query Performance

## Query Design

The history query retrieves jobs for a given session ID with pagination and related results.

**Query**:
```sql
SELECT j.*, r.*
FROM jobs j
LEFT JOIN results r ON r."jobId" = j.id
WHERE j."sessionId" = $1
ORDER BY j."createdAt" DESC
LIMIT $2 OFFSET $3;
```

**Indexes Used**:
- `jobs_sessionId_createdAt_idx` (composite): Supports WHERE sessionId + ORDER BY createdAt
- `results_jobId_idx`: Supports LEFT JOIN on jobId

## Performance Benchmarks

| Dataset Size | Query Time | Notes |
|--------------|------------|-------|
| 10 jobs | 5ms | Warm cache |
| 100 jobs | 15ms | Typical user history |
| 1,000 jobs | 50ms | Power user |
| 10,000 jobs | 200ms | Large dataset (offset pagination limit) |

**Test Environment**: PostgreSQL 16, 16GB RAM, SSD storage

## Pagination Strategy

**Current**: Offset pagination (`LIMIT/OFFSET`)
- Pros: Simple implementation, supports random page access
- Cons: Performance degrades at high offsets (>10,000 records)

**Recommendation for >10,000 jobs**: Cursor-based pagination
```typescript
// Cursor-based pagination example
const jobs = await prisma.job.findMany({
  where: { sessionId, createdAt: { lt: cursor } },
  orderBy: { createdAt: 'desc' },
  take: pageSize,
});
```

## Optimization Tips

1. **Keep indexes up to date**: Run `ANALYZE jobs;` after bulk inserts
2. **Monitor query plans**: Use EXPLAIN ANALYZE periodically
3. **Increase work_mem for large sorts**: If sorting >1000 jobs, increase work_mem
4. **Consider partitioning**: If jobs table exceeds 1M rows, partition by createdAt
5. **Archive old jobs**: Delete jobs older than retention period (e.g., 90 days)

## Known Limitations

- Offset pagination performance degrades at high offsets
- Concurrent inserts may cause pagination inconsistencies
- Very large result sets (>1MB) may impact response time
```

**Deliverables**:
- Documentation file `docs/database-performance.md` created
- Performance benchmarks included
- Scaling recommendations documented

**Technical Notes**:
- Offset pagination is acceptable for <10,000 records
- For very large datasets, cursor-based pagination is more efficient
- Partitioning by `createdAt` can improve query performance for time-based queries
- Archive old jobs to keep table size manageable

---

### TRI-1.7-008: Implement Database-Level Status Sorting

**Priority**: P1 (High)
**Effort**: 25 minutes
**Dependencies**: TRI-1.7-001, TRI-1.7-005

**Description**:
Implement status-based ordering at the database level using a CASE expression in ORDER BY, replacing the current in-memory sorting approach. The current implementation fetches all jobs when `sortBy === 'status'` and sorts in application memory, which loads entire result sets and breaks pagination. This task moves sorting to the database for efficiency and proper pagination support.

**Problem Statement**:
The current implementation (in `neo-nextjs-tasks.md` lines 690-707) has these issues:
1. Sets `orderBy: undefined` and `skip: 0` and `take: undefined` when `sortBy === 'status'`
2. Fetches ALL jobs for the session into memory
3. Sorts in memory with JavaScript `.sort()`
4. Slices for pagination with `.slice(skip, skip + pageSize)`

This breaks pagination semantics and causes performance degradation for sessions with many jobs.

**Acceptance Criteria**:
- [ ] Status sorting performed at database level using CASE expression
- [ ] skip/take (LIMIT/OFFSET) applied correctly to DB query for status sorting
- [ ] Only the requested page of results returned from database
- [ ] Pagination works correctly with status sorting (no duplicate/missing records)
- [ ] Query performance <100ms for 100 jobs with status sorting
- [ ] Composite index added for status + createdAt ordering
- [ ] EXPLAIN ANALYZE shows index usage (no sequential scan)
- [ ] Unit tests verify correct status sort order

**Implementation**:

**Option A: Raw SQL with CASE Expression (Recommended)**
```typescript
// lib/db/queries/history.ts

import { prisma } from '@/lib/db/prisma';
import { Prisma, JobStatus } from '@prisma/client';

// Status sort order per specification Section 5.1.2
const STATUS_SORT_ORDER: Record<JobStatus, number> = {
  PROCESSING: 1,
  PENDING: 2,
  UPLOADING: 3,
  RETRY_1: 4,
  RETRY_2: 5,
  RETRY_3: 6,
  COMPLETE: 7,
  PARTIAL_COMPLETE: 8,
  CANCELLED: 9,
  ERROR: 10,
};

export interface HistoryQueryParams {
  sessionId: string;
  page: number;
  pageSize: number;
  sortBy: 'createdAt' | 'status';
  sortOrder: 'asc' | 'desc';
}

export async function getJobHistory(params: HistoryQueryParams) {
  const { sessionId, page, pageSize, sortBy, sortOrder } = params;
  const skip = (page - 1) * pageSize;

  // Build orderBy clause
  let orderByClause: Prisma.JobOrderByWithRelationInput[];

  if (sortBy === 'status') {
    // Use raw SQL for CASE-based status ordering
    // Prisma doesn't support CASE in orderBy, so use $queryRaw
    // Note: sortOrder is safely handled via Prisma.sql fragments (ASC/DESC), not string interpolation
    const jobs = await prisma.$queryRaw<Array<{
      id: string;
      status: JobStatus;
      inputType: string;
      fileName: string | null;
      fileSize: number | null;
      url: string | null;
      createdAt: Date;
      completedAt: Date | null;
      checkpointData: string | null;
    }>>`
      SELECT id, status, "inputType", "fileName", "fileSize", url, "createdAt", "completedAt", "checkpointData"
      FROM jobs
      WHERE "sessionId" = ${sessionId}
      ORDER BY
        CASE status
          WHEN 'PROCESSING' THEN 1
          WHEN 'PENDING' THEN 2
          WHEN 'UPLOADING' THEN 3
          WHEN 'RETRY_1' THEN 4
          WHEN 'RETRY_2' THEN 5
          WHEN 'RETRY_3' THEN 6
          WHEN 'COMPLETE' THEN 7
          WHEN 'PARTIAL_COMPLETE' THEN 8
          WHEN 'CANCELLED' THEN 9
          WHEN 'ERROR' THEN 10
          ELSE 99
        END ${sortOrder === 'asc' ? Prisma.sql`ASC` : Prisma.sql`DESC`},
        "createdAt" DESC
      LIMIT ${pageSize}
      OFFSET ${skip}
    `;

    const totalCount = await prisma.job.count({ where: { sessionId } });
    const totalPages = Math.ceil(totalCount / pageSize);

    return {
      jobs,
      pagination: {
        page,
        pageSize,
        totalPages,
        totalCount,
        hasMore: page < totalPages,
      },
    };
  }

  // Standard createdAt sorting using Prisma
  const [jobs, totalCount] = await Promise.all([
    prisma.job.findMany({
      where: { sessionId },
      select: {
        id: true,
        status: true,
        inputType: true,
        fileName: true,
        fileSize: true,
        url: true,
        createdAt: true,
        completedAt: true,
        checkpointData: true,
      },
      orderBy: { [sortBy]: sortOrder },
      skip,
      take: pageSize,
    }),
    prisma.job.count({ where: { sessionId } }),
  ]);

  const totalPages = Math.ceil(totalCount / pageSize);

  return {
    jobs,
    pagination: {
      page,
      pageSize,
      totalPages,
      totalCount,
      hasMore: page < totalPages,
    },
  };
}
```

**Option B: Prisma with Computed Column (Alternative)**
```sql
-- Add a computed/generated column for status sort order
-- Migration: prisma/migrations/YYYYMMDD_add_status_sort_column.sql

ALTER TABLE jobs ADD COLUMN status_sort_order INTEGER GENERATED ALWAYS AS (
  CASE status
    WHEN 'PROCESSING' THEN 1
    WHEN 'PENDING' THEN 2
    WHEN 'UPLOADING' THEN 3
    WHEN 'RETRY_1' THEN 4
    WHEN 'RETRY_2' THEN 5
    WHEN 'RETRY_3' THEN 6
    WHEN 'COMPLETE' THEN 7
    WHEN 'PARTIAL_COMPLETE' THEN 8
    WHEN 'CANCELLED' THEN 9
    WHEN 'ERROR' THEN 10
    ELSE 99
  END
) STORED;

-- Then Prisma can use: orderBy: { statusSortOrder: sortOrder }
```

**Composite Index for Status Sorting**:
```sql
-- Add composite index for efficient status + createdAt ordering
-- This index supports queries that filter by sessionId and sort by status/createdAt

CREATE INDEX CONCURRENTLY jobs_sessionId_status_createdAt_idx
ON jobs ("sessionId", status, "createdAt" DESC);

-- EXPLAIN ANALYZE should show this index being used for status sorting queries
```

**Prisma Schema Update**:
```prisma
// prisma/schema.prisma

model Job {
  // ... existing fields ...

  @@index([sessionId, createdAt(sort: Desc)], name: "jobs_sessionId_createdAt_idx")
  @@index([sessionId, status, createdAt(sort: Desc)], name: "jobs_sessionId_status_createdAt_idx")
  @@index([status], name: "jobs_status_idx")
}
```

**Test Cases**:
```typescript
// test/db/queries/history-status-sort.test.ts

import { prisma } from '@/lib/db/prisma';
import { getJobHistory } from '@/lib/db/queries/history';
import { JobStatus } from '@prisma/client';

describe('History Query - Status Sorting at Database Level', () => {
  const sessionId = 'test-session-status-sort';

  beforeAll(async () => {
    // Create test jobs with various statuses
    const statuses: JobStatus[] = [
      'COMPLETE', 'ERROR', 'PROCESSING', 'PENDING',
      'CANCELLED', 'UPLOADING', 'RETRY_1', 'PARTIAL_COMPLETE',
    ];

    await prisma.job.createMany({
      data: statuses.map((status, i) => ({
        sessionId,
        status,
        inputType: 'FILE',
        createdAt: new Date(Date.now() - i * 1000), // Different timestamps
      })),
    });
  });

  afterAll(async () => {
    await prisma.job.deleteMany({ where: { sessionId } });
  });

  it('sorts by status ascending at database level', async () => {
    const result = await getJobHistory({
      sessionId,
      page: 1,
      pageSize: 10,
      sortBy: 'status',
      sortOrder: 'asc',
    });

    // Verify order: PROCESSING (1) should come first
    expect(result.jobs[0].status).toBe('PROCESSING');
    expect(result.jobs[1].status).toBe('PENDING');
    expect(result.jobs[2].status).toBe('UPLOADING');
  });

  it('sorts by status descending at database level', async () => {
    const result = await getJobHistory({
      sessionId,
      page: 1,
      pageSize: 10,
      sortBy: 'status',
      sortOrder: 'desc',
    });

    // Verify order: ERROR (10) should come first when descending
    expect(result.jobs[0].status).toBe('ERROR');
    expect(result.jobs[1].status).toBe('CANCELLED');
  });

  it('applies pagination correctly with status sorting', async () => {
    // Create more jobs for pagination test
    const extraJobs = Array.from({ length: 25 }, (_, i) => ({
      sessionId: `${sessionId}-pagination`,
      status: i % 2 === 0 ? 'COMPLETE' : 'PENDING' as JobStatus,
      inputType: 'FILE' as const,
    }));
    await prisma.job.createMany({ data: extraJobs });

    const page1 = await getJobHistory({
      sessionId: `${sessionId}-pagination`,
      page: 1,
      pageSize: 10,
      sortBy: 'status',
      sortOrder: 'asc',
    });

    const page2 = await getJobHistory({
      sessionId: `${sessionId}-pagination`,
      page: 2,
      pageSize: 10,
      sortBy: 'status',
      sortOrder: 'asc',
    });

    // Verify pagination works (no overlapping IDs)
    const page1Ids = new Set(page1.jobs.map(j => j.id));
    const page2Ids = new Set(page2.jobs.map(j => j.id));
    const overlap = [...page1Ids].filter(id => page2Ids.has(id));
    expect(overlap).toHaveLength(0);

    // Verify we got the expected number of results
    expect(page1.jobs).toHaveLength(10);
    expect(page2.jobs).toHaveLength(10);
    expect(page1.pagination.totalCount).toBe(25);

    // Cleanup
    await prisma.job.deleteMany({ where: { sessionId: `${sessionId}-pagination` } });
  });

  it('does NOT load all jobs into memory', async () => {
    // This test verifies the fix: pagination should be at DB level
    // We can verify by checking that only pageSize records are returned
    const result = await getJobHistory({
      sessionId,
      page: 1,
      pageSize: 3,
      sortBy: 'status',
      sortOrder: 'asc',
    });

    // Should return exactly pageSize, not all jobs
    expect(result.jobs).toHaveLength(3);
    expect(result.pagination.totalCount).toBe(8); // We created 8 jobs
  });
});
```

**Performance Verification**:
```bash
# Verify index is used for status sorting
psql "$DATABASE_URL" << EOF
EXPLAIN ANALYZE
SELECT id, status, "inputType", "fileName", "fileSize", url, "createdAt", "completedAt", "checkpointData"
FROM jobs
WHERE "sessionId" = 'test-session'
ORDER BY
  CASE status
    WHEN 'PROCESSING' THEN 1
    WHEN 'PENDING' THEN 2
    WHEN 'UPLOADING' THEN 3
    WHEN 'RETRY_1' THEN 4
    WHEN 'RETRY_2' THEN 5
    WHEN 'RETRY_3' THEN 6
    WHEN 'COMPLETE' THEN 7
    WHEN 'PARTIAL_COMPLETE' THEN 8
    WHEN 'CANCELLED' THEN 9
    WHEN 'ERROR' THEN 10
    ELSE 99
  END ASC,
  "createdAt" DESC
LIMIT 20 OFFSET 0;
EOF

# Expected: Index Scan using jobs_sessionId_status_createdAt_idx
```

**Deliverables**:
- Updated `lib/db/queries/history.ts` with DB-level status sorting
- Prisma migration for composite index `jobs_sessionId_status_createdAt_idx`
- Unit tests for status sorting with pagination
- EXPLAIN ANALYZE output confirming index usage
- Documentation update in `docs/database-performance.md`

**Technical Notes**:
- CASE expression in ORDER BY allows custom sort ordering by status
- Secondary sort by `createdAt DESC` ensures deterministic ordering within same status
- Composite index `(sessionId, status, createdAt DESC)` supports efficient status-based queries
- Raw SQL via `$queryRaw` is required since Prisma doesn't support CASE in orderBy
- Alternative: Add a generated column `status_sort_order` for native Prisma support
- Memory sorting approach is O(n) where n is total jobs; DB sorting is O(log n) with index

---

## Dependencies

**Upstream**:
- Sprint 1.2 Database integration complete (TRI-1.2-010)
- Sprint 1.2 CRUD operations tested (TRI-1.2-007)

**Downstream**:
- Sprint 1.8 Integration testing (history endpoint tests)

**Cross-Team**:
- Neo Anderson: History endpoint implementation using `getJobHistory()`
- Ola Mae Johnson: History UI component using pagination data

---

## Validation

Sprint 1.7 PostgreSQL tasks are complete when:
1. All tasks TRI-1.7-001 through TRI-1.7-008 complete
2. History query implemented and tested
3. Pagination validation working correctly
4. Query performance meets targets (<100ms for 100 jobs)
5. Edge cases handled gracefully
6. Concurrency tests pass
7. Documentation complete
8. Status sorting uses DB-level CASE expression (not in-memory sorting)

---

## Risk Mitigation

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Query performance degrades at scale | Medium | Medium | Benchmark with large datasets, document cursor pagination alternative |
| Pagination inconsistencies | Low | Low | Document known race conditions, acceptable for this use case |
| Missing index | Low | High | Verify indexes in TRI-1.7-003, use EXPLAIN ANALYZE |
| N+1 query problem | Low | High | Use `include: { results: true }` to eager load |
| In-memory sorting breaks pagination | High | High | Use CASE expression in ORDER BY for status sorting (TRI-1.7-008) |

---

## Total Effort Summary

**Total Estimated Effort**: 2.5 hours (165 minutes)
**Task Breakdown**:
- TRI-1.7-001: 30 minutes (Query design)
- TRI-1.7-002: 15 minutes (Pagination validation)
- TRI-1.7-003: 20 minutes (Performance analysis)
- TRI-1.7-004: 15 minutes (Edge case handling)
- TRI-1.7-005: 20 minutes (Status filtering - optional)
- TRI-1.7-006: 25 minutes (Concurrency testing)
- TRI-1.7-007: 15 minutes (Documentation)
- TRI-1.7-008: 25 minutes (DB-level status sorting)

**Critical Path**: TRI-1.7-001 → 002 → 003 → 004 → 006 → 007 → 008

---

## Deliverables Checklist

- [ ] `lib/db/queries/history.ts` - History query implementation
- [ ] `lib/validation/pagination.ts` - Pagination validation
- [ ] `docs/database-performance.md` - Performance documentation
- [ ] `test/db/queries/history.test.ts` - Unit tests
- [ ] `test/db/queries/history-concurrency.test.ts` - Concurrency tests
- [ ] `test/db/queries/history-status-sort.test.ts` - Status sorting tests (TRI-1.7-008)
- [ ] Prisma migration for `jobs_sessionId_status_createdAt_idx` composite index
- [ ] DB-level status sorting with CASE expression (not in-memory)
- [ ] Query performance verified (<100ms for 100 jobs)
- [ ] Edge cases tested and documented
- [ ] Ready for integration into `/api/v1/history` endpoint
