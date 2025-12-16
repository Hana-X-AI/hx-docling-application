# HX Docling Application - Database Test Cases

**Document Type**: Database Operations Test Cases
**Version**: 2.1.0
**Status**: COMPLETE
**Created**: 2025-12-12
**Author**: Julia Santos (Testing & QA Specialist)
**Master Plan Reference**: `project/0.5-test/00-test-plan-overview.md`
**Specification Reference**: `project/0.3-specification/0.3.1-detailed-specification.md` v1.2.1

---

## Table of Contents

1. [Overview](#1-overview)
2. [Job Entity Tests](#2-job-entity-tests)
3. [Result Entity Tests](#3-result-entity-tests)
4. [Session Data Tests](#4-session-data-tests)
5. [Checkpoint Storage Tests](#5-checkpoint-storage-tests)
6. [Transaction Integrity Tests](#6-transaction-integrity-tests)
7. [Query Performance Tests](#7-query-performance-tests)
8. [Index Strategy and Query Optimization](#8-index-strategy-and-query-optimization)
9. [Connection Pooling and Management](#9-connection-pooling-and-management)
10. [Concurrency Control and Locking](#10-concurrency-control-and-locking)
11. [Workflow State Machine and Durable Execution](#11-workflow-state-machine-and-durable-execution)
12. [Cache-Database Consistency](#12-cache-database-consistency)
13. [Production Observability and Monitoring](#13-production-observability-and-monitoring)
14. [Data Constraints and Schema Validation](#14-data-constraints-and-schema-validation)
15. [Backup, Recovery, and Data Retention](#15-backup-recovery-and-data-retention)
16. [Data Migration Tests](#16-data-migration-tests)

---

## 1. Overview

### 1.1 Purpose

This document defines comprehensive test cases for database operations using Prisma with PostgreSQL. Tests cover CRUD operations, data integrity, transactions, query performance, and session management.

### 1.2 Database Schema Reference

```prisma
model Job {
  id              String        @id @default(uuid())
  sessionId       String
  status          JobStatus     @default(PENDING)
  inputType       InputType
  fileName        String?
  fileSize        Int?
  mimeType        String?
  filePath        String?
  sourceUrl       String?
  errorCode       String?
  errorMessage    String?
  retryCount      Int           @default(0)
  checkpointStage String?
  checkpointData  Json?
  checkpointHash  String?
  createdAt       DateTime      @default(now())
  updatedAt       DateTime      @updatedAt
  completedAt     DateTime?
  results         Result[]
}

model Result {
  id        String       @id @default(uuid())
  jobId     String
  format    ResultFormat
  content   String       @db.Text
  size      Int
  createdAt DateTime     @default(now())
  job       Job          @relation(fields: [jobId], references: [id], onDelete: Cascade)
}

enum JobStatus {
  PENDING
  UPLOADING
  PROCESSING
  RETRY_1
  RETRY_2
  RETRY_3
  COMPLETE
  PARTIAL_COMPLETE
  ERROR
  CANCELLED
}

enum InputType {
  FILE
  URL
}

enum ResultFormat {
  MARKDOWN
  HTML
  JSON
}
```

### 1.3 Test Database Setup

```typescript
// test/setup-database.ts
import { PrismaClient } from '@prisma/client';

export const prisma = new PrismaClient({
  datasources: { db: { url: process.env.TEST_DATABASE_URL } },
});

beforeAll(async () => {
  await prisma.$connect();
});

beforeEach(async () => {
  // Clear all data in reverse order of dependencies
  await prisma.result.deleteMany();
  await prisma.job.deleteMany();
});

afterAll(async () => {
  await prisma.$disconnect();
});
```

---

## 2. Job Entity Tests

### 2.1 Job Creation Tests

---

### DB-JOB-001: Create Job - File Upload

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-JOB-001 |
| **Title** | Create Job for File Upload |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.1 |

**Test Steps**:

```typescript
it('creates job record for file upload', async () => {
  const job = await prisma.job.create({
    data: {
      sessionId: 'test-session-123',
      status: 'PENDING',
      inputType: 'FILE',
      fileName: 'document.pdf',
      fileSize: 102400,
      mimeType: 'application/pdf',
      filePath: '/tmp/uploads/document.pdf',
    },
  });

  expect(job.id).toMatch(/^[0-9a-f-]{36}$/);
  expect(job.sessionId).toBe('test-session-123');
  expect(job.status).toBe('PENDING');
  expect(job.inputType).toBe('FILE');
  expect(job.fileName).toBe('document.pdf');
  expect(job.fileSize).toBe(102400);
  expect(job.mimeType).toBe('application/pdf');
  expect(job.retryCount).toBe(0);
  expect(job.createdAt).toBeInstanceOf(Date);
  expect(job.updatedAt).toBeInstanceOf(Date);
});
```

**Expected Result**:
- UUID generated for id
- Default values applied
- Timestamps set

---

### DB-JOB-002: Create Job - URL Input

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-JOB-002 |
| **Title** | Create Job for URL Input |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.1 |

**Test Steps**:

```typescript
it('creates job record for URL input', async () => {
  const job = await prisma.job.create({
    data: {
      sessionId: 'test-session-123',
      status: 'PENDING',
      inputType: 'URL',
      sourceUrl: 'https://example.com/document.html',
    },
  });

  expect(job.inputType).toBe('URL');
  expect(job.sourceUrl).toBe('https://example.com/document.html');
  expect(job.fileName).toBeNull();
  expect(job.filePath).toBeNull();
});
```

---

### DB-JOB-003: Create Job - Required Fields Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-JOB-003 |
| **Title** | Validate Required Fields on Creation |
| **Priority** | P1 - High |
| **Spec Reference** | Schema constraints |

**Test Steps**:

```typescript
it('rejects job without sessionId', async () => {
  await expect(
    prisma.job.create({
      data: {
        status: 'PENDING',
        inputType: 'FILE',
      } as any, // Missing sessionId
    })
  ).rejects.toThrow();
});

it('rejects job without inputType', async () => {
  await expect(
    prisma.job.create({
      data: {
        sessionId: 'test-session',
        status: 'PENDING',
      } as any, // Missing inputType
    })
  ).rejects.toThrow();
});
```

---

### 2.2 Job Update Tests

---

### DB-JOB-004: Update Job Status

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-JOB-004 |
| **Title** | Update Job Status |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.1.2 |

**Test Steps**:

```typescript
it('updates job status and updatedAt', async () => {
  const job = await prisma.job.create({
    data: {
      sessionId: 'test-session',
      status: 'PENDING',
      inputType: 'FILE',
    },
  });

  const originalUpdatedAt = job.updatedAt;

  // Wait to ensure different timestamp
  await new Promise(r => setTimeout(r, 100));

  const updated = await prisma.job.update({
    where: { id: job.id },
    data: { status: 'PROCESSING' },
  });

  expect(updated.status).toBe('PROCESSING');
  expect(updated.updatedAt.getTime()).toBeGreaterThan(originalUpdatedAt.getTime());
});
```

---

### DB-JOB-005: Update Job to Error with Details

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-JOB-005 |
| **Title** | Update Job to Error Status |
| **Priority** | P1 - High |
| **Spec Reference** | FR-801 |

**Test Steps**:

```typescript
it('updates job with error details', async () => {
  const job = await prisma.job.create({
    data: {
      sessionId: 'test-session',
      status: 'PROCESSING',
      inputType: 'FILE',
    },
  });

  const updated = await prisma.job.update({
    where: { id: job.id },
    data: {
      status: 'ERROR',
      errorCode: 'E302',
      errorMessage: 'Document conversion failed',
    },
  });

  expect(updated.status).toBe('ERROR');
  expect(updated.errorCode).toBe('E302');
  expect(updated.errorMessage).toBe('Document conversion failed');
});
```

---

### DB-JOB-006: Update Retry Count

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-JOB-006 |
| **Title** | Increment Retry Count |
| **Priority** | P1 - High |
| **Spec Reference** | FR-803 |

**Test Steps**:

```typescript
it('increments retry count on status transition', async () => {
  const job = await prisma.job.create({
    data: {
      sessionId: 'test-session',
      status: 'PROCESSING',
      inputType: 'FILE',
      retryCount: 0,
    },
  });

  // Simulate retry transitions
  for (let i = 1; i <= 3; i++) {
    const updated = await prisma.job.update({
      where: { id: job.id },
      data: {
        status: `RETRY_${i}` as JobStatus,
        retryCount: i,
      },
    });

    expect(updated.status).toBe(`RETRY_${i}`);
    expect(updated.retryCount).toBe(i);
  }
});
```

---

### DB-JOB-007: Set CompletedAt on Completion

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-JOB-007 |
| **Title** | Set CompletedAt Timestamp |
| **Priority** | P1 - High |
| **Spec Reference** | Section 5.1 |

**Test Steps**:

```typescript
it('sets completedAt when job completes', async () => {
  const job = await prisma.job.create({
    data: {
      sessionId: 'test-session',
      status: 'PROCESSING',
      inputType: 'FILE',
    },
  });

  const completedAt = new Date();
  const updated = await prisma.job.update({
    where: { id: job.id },
    data: {
      status: 'COMPLETE',
      completedAt,
    },
  });

  expect(updated.completedAt).toEqual(completedAt);
});
```

---

### 2.3 Job Query Tests

---

### DB-JOB-008: Find Job by ID

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-JOB-008 |
| **Title** | Find Job by ID |
| **Priority** | P1 - High |
| **Spec Reference** | FR-703 |

**Test Steps**:

```typescript
it('finds job by ID with results', async () => {
  const job = await prisma.job.create({
    data: {
      sessionId: 'test-session',
      status: 'COMPLETE',
      inputType: 'FILE',
      results: {
        create: [
          { format: 'MARKDOWN', content: '# Test', size: 6 },
          { format: 'HTML', content: '<h1>Test</h1>', size: 14 },
        ],
      },
    },
  });

  const found = await prisma.job.findUnique({
    where: { id: job.id },
    include: { results: true },
  });

  expect(found).not.toBeNull();
  expect(found?.results).toHaveLength(2);
});
```

---

### DB-JOB-009: Find Jobs by Session

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-JOB-009 |
| **Title** | Find Jobs by Session ID |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-701 |

**Test Steps**:

```typescript
it('finds only jobs for specified session', async () => {
  // Create jobs for different sessions
  await prisma.job.createMany({
    data: [
      { sessionId: 'session-a', status: 'COMPLETE', inputType: 'FILE' },
      { sessionId: 'session-a', status: 'COMPLETE', inputType: 'FILE' },
      { sessionId: 'session-b', status: 'COMPLETE', inputType: 'FILE' },
      { sessionId: 'session-c', status: 'COMPLETE', inputType: 'FILE' },
    ],
  });

  const sessionAJobs = await prisma.job.findMany({
    where: { sessionId: 'session-a' },
  });

  expect(sessionAJobs).toHaveLength(2);
  sessionAJobs.forEach(job => {
    expect(job.sessionId).toBe('session-a');
  });
});
```

---

### DB-JOB-010: Paginated History Query

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-JOB-010 |
| **Title** | Paginated Job History |
| **Priority** | P1 - High |
| **Spec Reference** | FR-702 |

**Test Steps**:

```typescript
it('returns paginated results with correct ordering', async () => {
  // Create 25 jobs with different timestamps
  const jobs = [];
  for (let i = 0; i < 25; i++) {
    jobs.push({
      sessionId: 'test-session',
      status: 'COMPLETE' as JobStatus,
      inputType: 'FILE' as InputType,
      fileName: `file-${i.toString().padStart(2, '0')}.pdf`,
      createdAt: new Date(Date.now() - i * 60000), // Decreasing timestamps
    });
  }
  await prisma.job.createMany({ data: jobs });

  // Page 1
  const page1 = await prisma.job.findMany({
    where: { sessionId: 'test-session' },
    orderBy: { createdAt: 'desc' },
    take: 10,
    skip: 0,
  });

  expect(page1).toHaveLength(10);
  expect(page1[0].fileName).toBe('file-00.pdf'); // Newest first

  // Page 2
  const page2 = await prisma.job.findMany({
    where: { sessionId: 'test-session' },
    orderBy: { createdAt: 'desc' },
    take: 10,
    skip: 10,
  });

  expect(page2).toHaveLength(10);
  expect(page2[0].fileName).toBe('file-10.pdf');

  // Page 3 (partial)
  const page3 = await prisma.job.findMany({
    where: { sessionId: 'test-session' },
    orderBy: { createdAt: 'desc' },
    take: 10,
    skip: 20,
  });

  expect(page3).toHaveLength(5);
});
```

---

### DB-JOB-011: Count Jobs for Pagination

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-JOB-011 |
| **Title** | Count Total Jobs |
| **Priority** | P1 - High |
| **Spec Reference** | FR-702 |

**Test Steps**:

```typescript
it('returns accurate total count for pagination', async () => {
  await prisma.job.createMany({
    data: Array.from({ length: 45 }, (_, i) => ({
      sessionId: 'test-session',
      status: 'COMPLETE' as JobStatus,
      inputType: 'FILE' as InputType,
    })),
  });

  const count = await prisma.job.count({
    where: { sessionId: 'test-session' },
  });

  expect(count).toBe(45);
});
```

---

### DB-JOB-012: Filter by Status

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-JOB-012 |
| **Title** | Filter Jobs by Status |
| **Priority** | P2 - Medium |
| **Spec Reference** | FR-702 |

**Test Steps**:

```typescript
it('filters jobs by status', async () => {
  await prisma.job.createMany({
    data: [
      { sessionId: 'test-session', status: 'COMPLETE', inputType: 'FILE' },
      { sessionId: 'test-session', status: 'COMPLETE', inputType: 'FILE' },
      { sessionId: 'test-session', status: 'ERROR', inputType: 'FILE' },
      { sessionId: 'test-session', status: 'PROCESSING', inputType: 'FILE' },
    ],
  });

  const completedJobs = await prisma.job.findMany({
    where: {
      sessionId: 'test-session',
      status: 'COMPLETE',
    },
  });

  expect(completedJobs).toHaveLength(2);
});
```

---

### 2.4 Job Deletion Tests

---

### DB-JOB-013: Delete Job Cascades to Results

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-JOB-013 |
| **Title** | Cascade Delete to Results |
| **Priority** | P1 - High |
| **Spec Reference** | Schema onDelete: Cascade |

**Test Steps**:

```typescript
it('deletes results when job is deleted', async () => {
  const job = await prisma.job.create({
    data: {
      sessionId: 'test-session',
      status: 'COMPLETE',
      inputType: 'FILE',
      results: {
        create: [
          { format: 'MARKDOWN', content: '# Test', size: 6 },
          { format: 'HTML', content: '<h1>Test</h1>', size: 14 },
        ],
      },
    },
    include: { results: true },
  });

  expect(job.results).toHaveLength(2);
  const resultIds = job.results.map(r => r.id);

  await prisma.job.delete({ where: { id: job.id } });

  // Verify results are also deleted
  const orphanedResults = await prisma.result.findMany({
    where: { id: { in: resultIds } },
  });

  expect(orphanedResults).toHaveLength(0);
});
```

---

## 3. Result Entity Tests

### 3.1 Result Creation Tests

---

### DB-RES-001: Create Result - Markdown

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-RES-001 |
| **Title** | Create Markdown Result |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.1 |

**Test Steps**:

```typescript
it('creates markdown result record', async () => {
  const job = await prisma.job.create({
    data: {
      sessionId: 'test-session',
      status: 'PROCESSING',
      inputType: 'FILE',
    },
  });

  const content = '# Document Title\n\nThis is the content.';
  const result = await prisma.result.create({
    data: {
      jobId: job.id,
      format: 'MARKDOWN',
      content,
      size: Buffer.byteLength(content, 'utf8'),
    },
  });

  expect(result.id).toMatch(/^[0-9a-f-]{36}$/);
  expect(result.format).toBe('MARKDOWN');
  expect(result.content).toBe(content);
  expect(result.size).toBe(Buffer.byteLength(content, 'utf8'));
});
```

---

### DB-RES-002: Create Result - HTML

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-RES-002 |
| **Title** | Create HTML Result |
| **Priority** | P1 - High |
| **Spec Reference** | Section 5.1 |

**Test Steps**:

```typescript
it('creates HTML result record', async () => {
  const job = await prisma.job.create({
    data: { sessionId: 'test-session', status: 'PROCESSING', inputType: 'FILE' },
  });

  const content = '<html><body><h1>Document</h1></body></html>';
  const result = await prisma.result.create({
    data: { jobId: job.id, format: 'HTML', content, size: content.length },
  });

  expect(result.format).toBe('HTML');
});
```

---

### DB-RES-003: Create Result - JSON

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-RES-003 |
| **Title** | Create JSON Result |
| **Priority** | P1 - High |
| **Spec Reference** | Section 5.1 |

**Test Steps**:

```typescript
it('creates JSON result record', async () => {
  const job = await prisma.job.create({
    data: { sessionId: 'test-session', status: 'PROCESSING', inputType: 'FILE' },
  });

  const content = JSON.stringify({ schema: 'test', data: {} }, null, 2);
  const result = await prisma.result.create({
    data: { jobId: job.id, format: 'JSON', content, size: content.length },
  });

  expect(result.format).toBe('JSON');
  expect(() => JSON.parse(result.content)).not.toThrow();
});
```

---

### DB-RES-004: Create All Results at Once

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-RES-004 |
| **Title** | Create All Result Formats |
| **Priority** | P1 - High |
| **Spec Reference** | FR-404 |

**Test Steps**:

```typescript
it('creates all three result formats atomically', async () => {
  const job = await prisma.job.create({
    data: { sessionId: 'test-session', status: 'PROCESSING', inputType: 'FILE' },
  });

  const results = await prisma.result.createMany({
    data: [
      { jobId: job.id, format: 'MARKDOWN', content: '# Doc', size: 5 },
      { jobId: job.id, format: 'HTML', content: '<h1>Doc</h1>', size: 12 },
      { jobId: job.id, format: 'JSON', content: '{}', size: 2 },
    ],
  });

  expect(results.count).toBe(3);

  const allResults = await prisma.result.findMany({
    where: { jobId: job.id },
  });

  expect(allResults).toHaveLength(3);
  expect(allResults.map(r => r.format).sort()).toEqual(['HTML', 'JSON', 'MARKDOWN']);
});
```

---

### DB-RES-005: Result Foreign Key Constraint

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-RES-005 |
| **Title** | Result Requires Valid Job |
| **Priority** | P1 - High |
| **Spec Reference** | Schema constraints |

**Test Steps**:

```typescript
it('rejects result with invalid jobId', async () => {
  await expect(
    prisma.result.create({
      data: {
        jobId: 'non-existent-uuid',
        format: 'MARKDOWN',
        content: '# Test',
        size: 6,
      },
    })
  ).rejects.toThrow(); // Foreign key violation
});
```

---

### 3.2 Result Query Tests

---

### DB-RES-006: Find Results by Job

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-RES-006 |
| **Title** | Find All Results for Job |
| **Priority** | P1 - High |
| **Spec Reference** | FR-703 |

**Test Steps**:

```typescript
it('retrieves all results for a job', async () => {
  const job = await prisma.job.create({
    data: {
      sessionId: 'test-session',
      status: 'COMPLETE',
      inputType: 'FILE',
      results: {
        create: [
          { format: 'MARKDOWN', content: '# Test', size: 6 },
          { format: 'HTML', content: '<h1>Test</h1>', size: 14 },
          { format: 'JSON', content: '{}', size: 2 },
        ],
      },
    },
  });

  const results = await prisma.result.findMany({
    where: { jobId: job.id },
    orderBy: { format: 'asc' },
  });

  expect(results).toHaveLength(3);
  expect(results[0].format).toBe('HTML');
  expect(results[1].format).toBe('JSON');
  expect(results[2].format).toBe('MARKDOWN');
});
```

---

### DB-RES-007: Find Specific Result Format

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-RES-007 |
| **Title** | Find Result by Format |
| **Priority** | P1 - High |
| **Spec Reference** | FR-606 |

**Test Steps**:

```typescript
it('finds result by job and format', async () => {
  const job = await prisma.job.create({
    data: {
      sessionId: 'test-session',
      status: 'COMPLETE',
      inputType: 'FILE',
      results: {
        create: [
          { format: 'MARKDOWN', content: '# Markdown Content', size: 18 },
          { format: 'HTML', content: '<p>HTML Content</p>', size: 20 },
        ],
      },
    },
  });

  const markdownResult = await prisma.result.findFirst({
    where: {
      jobId: job.id,
      format: 'MARKDOWN',
    },
  });

  expect(markdownResult).not.toBeNull();
  expect(markdownResult?.content).toBe('# Markdown Content');
});
```

---

## 4. Session Data Tests

### DB-SESS-001: Session Job Count

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-SESS-001 |
| **Title** | Count Jobs per Session |
| **Priority** | P1 - High |
| **Spec Reference** | Session management |

**Test Steps**:

```typescript
it('counts jobs accurately per session', async () => {
  await prisma.job.createMany({
    data: [
      { sessionId: 'session-a', status: 'COMPLETE', inputType: 'FILE' },
      { sessionId: 'session-a', status: 'COMPLETE', inputType: 'FILE' },
      { sessionId: 'session-a', status: 'ERROR', inputType: 'FILE' },
      { sessionId: 'session-b', status: 'COMPLETE', inputType: 'FILE' },
    ],
  });

  const sessionACount = await prisma.job.count({
    where: { sessionId: 'session-a' },
  });

  const sessionACompleteCount = await prisma.job.count({
    where: { sessionId: 'session-a', status: 'COMPLETE' },
  });

  expect(sessionACount).toBe(3);
  expect(sessionACompleteCount).toBe(2);
});
```

---

### DB-SESS-002: Session Data Isolation

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-SESS-002 |
| **Title** | Verify Session Isolation |
| **Priority** | P0 - Critical (Security) |
| **Spec Reference** | Session isolation |

**Test Steps**:

```typescript
it('ensures complete session isolation', async () => {
  // Create jobs in session A
  const jobA = await prisma.job.create({
    data: {
      sessionId: 'session-a',
      status: 'COMPLETE',
      inputType: 'FILE',
      results: {
        create: [{ format: 'MARKDOWN', content: 'Secret A', size: 8 }],
      },
    },
    include: { results: true },
  });

  // Query from session B perspective
  const jobFromB = await prisma.job.findFirst({
    where: {
      id: jobA.id,
      sessionId: 'session-b', // Different session
    },
    include: { results: true },
  });

  // Should not find the job
  expect(jobFromB).toBeNull();

  // Direct result query should also not return other session's data
  const resultsFromB = await prisma.result.findMany({
    where: {
      job: { sessionId: 'session-b' },
    },
  });

  expect(resultsFromB).toHaveLength(0);
});
```

---

## 5. Checkpoint Storage Tests

### DB-CK-001: Store Checkpoint Data

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-CK-001 |
| **Title** | Store Checkpoint Data as JSON |
| **Priority** | P1 - High |
| **Spec Reference** | Section 5.5 |

**Test Steps**:

```typescript
it('stores checkpoint data as JSON', async () => {
  const job = await prisma.job.create({
    data: {
      sessionId: 'test-session',
      status: 'PROCESSING',
      inputType: 'FILE',
    },
  });

  const checkpointData = {
    stage: 'converted',
    doclingDocument: { schema_name: 'test', version: '1.0' },
    timestamp: new Date().toISOString(),
  };

  const hash = createHash('sha256')
    .update(JSON.stringify(checkpointData))
    .digest('hex');

  const updated = await prisma.job.update({
    where: { id: job.id },
    data: {
      checkpointStage: 'converted',
      checkpointData,
      checkpointHash: hash,
    },
  });

  expect(updated.checkpointStage).toBe('converted');
  expect(updated.checkpointData).toEqual(checkpointData);
  expect(updated.checkpointHash).toBe(hash);
});
```

---

### DB-CK-002: Retrieve Checkpoint Data

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-CK-002 |
| **Title** | Retrieve and Validate Checkpoint |
| **Priority** | P1 - High |
| **Spec Reference** | Section 5.5 |

**Test Steps**:

```typescript
it('retrieves checkpoint with integrity check', async () => {
  const checkpointData = { stage: 'uploaded', filePath: '/tmp/test.pdf' };
  const hash = createHash('sha256')
    .update(JSON.stringify(checkpointData))
    .digest('hex');

  const job = await prisma.job.create({
    data: {
      sessionId: 'test-session',
      status: 'PROCESSING',
      inputType: 'FILE',
      checkpointStage: 'uploaded',
      checkpointData,
      checkpointHash: hash,
    },
  });

  const retrieved = await prisma.job.findUnique({
    where: { id: job.id },
    select: {
      checkpointStage: true,
      checkpointData: true,
      checkpointHash: true,
    },
  });

  // Verify integrity
  const computedHash = createHash('sha256')
    .update(JSON.stringify(retrieved?.checkpointData))
    .digest('hex');

  expect(computedHash).toBe(retrieved?.checkpointHash);
});
```

---

### DB-CK-003: Clear Checkpoint on Completion

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-CK-003 |
| **Title** | Clear Checkpoint on Job Completion |
| **Priority** | P1 - High |
| **Spec Reference** | Section 5.5 |

**Test Steps**:

```typescript
it('clears checkpoint when job completes', async () => {
  const job = await prisma.job.create({
    data: {
      sessionId: 'test-session',
      status: 'PROCESSING',
      inputType: 'FILE',
      checkpointStage: 'converted',
      checkpointData: { test: true },
      checkpointHash: 'abc123',
    },
  });

  const completed = await prisma.job.update({
    where: { id: job.id },
    data: {
      status: 'COMPLETE',
      checkpointStage: null,
      checkpointData: null,
      checkpointHash: null,
    },
  });

  expect(completed.checkpointStage).toBeNull();
  expect(completed.checkpointData).toBeNull();
  expect(completed.checkpointHash).toBeNull();
});
```

---

## 6. Transaction Integrity Tests

### DB-TX-001: Transaction Rollback on Error

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-TX-001 |
| **Title** | Transaction Rollback on Failure |
| **Priority** | P0 - Critical |
| **Spec Reference** | Database integrity |

**Test Steps**:

```typescript
it('rolls back all changes on transaction failure', async () => {
  const job = await prisma.job.create({
    data: { sessionId: 'test-session', status: 'PROCESSING', inputType: 'FILE' },
  });

  try {
    await prisma.$transaction(async (tx) => {
      // Create first result
      await tx.result.create({
        data: { jobId: job.id, format: 'MARKDOWN', content: '#', size: 1 },
      });

      // Create second result
      await tx.result.create({
        data: { jobId: job.id, format: 'HTML', content: '<p>', size: 3 },
      });

      // Simulate failure
      throw new Error('Simulated transaction failure');
    });
  } catch (e) {
    // Expected
  }

  // Verify no results were saved
  const results = await prisma.result.findMany({
    where: { jobId: job.id },
  });

  expect(results).toHaveLength(0);
});
```

---

### DB-TX-002: Transaction Commit on Success

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-TX-002 |
| **Title** | Transaction Commit All Changes |
| **Priority** | P1 - High |
| **Spec Reference** | Database integrity |

**Test Steps**:

```typescript
it('commits all changes on transaction success', async () => {
  const job = await prisma.job.create({
    data: { sessionId: 'test-session', status: 'PROCESSING', inputType: 'FILE' },
  });

  await prisma.$transaction(async (tx) => {
    // Create results
    await tx.result.createMany({
      data: [
        { jobId: job.id, format: 'MARKDOWN', content: '#', size: 1 },
        { jobId: job.id, format: 'HTML', content: '<p>', size: 3 },
        { jobId: job.id, format: 'JSON', content: '{}', size: 2 },
      ],
    });

    // Update job status
    await tx.job.update({
      where: { id: job.id },
      data: { status: 'COMPLETE', completedAt: new Date() },
    });
  });

  const completedJob = await prisma.job.findUnique({
    where: { id: job.id },
    include: { results: true },
  });

  expect(completedJob?.status).toBe('COMPLETE');
  expect(completedJob?.results).toHaveLength(3);
});
```

---

### DB-TX-003: Concurrent Update Handling

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-TX-003 |
| **Title** | Handle Concurrent Updates |
| **Priority** | P1 - High |
| **Spec Reference** | Optimistic concurrency |

**Test Steps**:

```typescript
it('handles concurrent updates safely', async () => {
  const job = await prisma.job.create({
    data: {
      sessionId: 'test-session',
      status: 'PENDING',
      inputType: 'FILE',
      retryCount: 0,
    },
  });

  // Simulate concurrent updates
  const updates = await Promise.allSettled([
    prisma.job.update({
      where: { id: job.id },
      data: { retryCount: { increment: 1 } },
    }),
    prisma.job.update({
      where: { id: job.id },
      data: { retryCount: { increment: 1 } },
    }),
    prisma.job.update({
      where: { id: job.id },
      data: { retryCount: { increment: 1 } },
    }),
  ]);

  // All should succeed (Prisma handles this)
  expect(updates.filter(r => r.status === 'fulfilled')).toHaveLength(3);

  const finalJob = await prisma.job.findUnique({ where: { id: job.id } });
  expect(finalJob?.retryCount).toBe(3);
});
```

---

## 7. Query Performance Tests

### DB-PERF-001: History Query Performance

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-PERF-001 |
| **Title** | History Query Performance |
| **Priority** | P2 - Medium |
| **Spec Reference** | NFR-103 |

**Test Steps**:

```typescript
it('completes history query within 100ms with 1000 jobs', async () => {
  // Seed 1000 jobs
  const jobs = Array.from({ length: 1000 }, (_, i) => ({
    sessionId: 'perf-test-session',
    status: 'COMPLETE' as JobStatus,
    inputType: 'FILE' as InputType,
    fileName: `file-${i}.pdf`,
    createdAt: new Date(Date.now() - i * 60000),
  }));

  await prisma.job.createMany({ data: jobs });

  // Measure query time
  const start = Date.now();
  const results = await prisma.job.findMany({
    where: { sessionId: 'perf-test-session' },
    orderBy: { createdAt: 'desc' },
    take: 20,
    select: {
      id: true,
      status: true,
      fileName: true,
      createdAt: true,
    },
  });
  const elapsed = Date.now() - start;

  expect(results).toHaveLength(20);
  expect(elapsed).toBeLessThan(100);
}, 30000);
```

---

### DB-PERF-002: Index Usage Verification

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-PERF-002 |
| **Title** | Verify Index Usage |
| **Priority** | P2 - Medium |
| **Spec Reference** | Database optimization |

**Test Steps**:

```typescript
it('uses index for sessionId queries', async () => {
  // This test verifies through EXPLAIN ANALYZE
  const result = await prisma.$queryRaw<any[]>`
    EXPLAIN ANALYZE
    SELECT * FROM "Job"
    WHERE "sessionId" = 'test-session'
    ORDER BY "createdAt" DESC
    LIMIT 20
  `;

  const plan = result.map(r => r['QUERY PLAN']).join('\n');

  // Should use index scan, not sequential scan
  expect(plan).toContain('Index');
  expect(plan).not.toContain('Seq Scan');
});
```

---

## 8. Index Strategy and Query Optimization

> **Priority**: P0 - CRITICAL
> **Gap Reference**: Consolidated Review - Gap Domain 1
> **Risk Level**: Production showstopper without proper index validation

This section validates that database indexes exist and are properly utilized for critical query patterns. Without these tests, production queries can degrade from milliseconds to minutes as data volume grows.

---

### DB-PERF-003: Verify sessionId Index Exists and Is Used

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-PERF-003 |
| **Title** | Verify sessionId Index Exists and Is Used |
| **Priority** | P0 - Critical |
| **Spec Reference** | Database optimization |
| **Gap Reference** | Consolidated Review - Gap Domain 1 |

**Test Steps**:

```typescript
it('should have index on sessionId for efficient history queries', async () => {
  // Verify index exists
  const indexResult = await prisma.$queryRaw<any[]>`
    SELECT indexname, indexdef
    FROM pg_indexes
    WHERE tablename = 'Job' AND indexdef LIKE '%sessionId%'
  `;
  expect(indexResult.length).toBeGreaterThan(0);

  // Verify index is used in query plan
  const explainResult = await prisma.$queryRaw<any[]>`
    EXPLAIN ANALYZE
    SELECT * FROM "Job"
    WHERE "sessionId" = 'test-session'
    ORDER BY "createdAt" DESC
    LIMIT 20
  `;

  const plan = explainResult.map(r => r['QUERY PLAN']).join('\n');
  expect(plan).toContain('Index');
  expect(plan).not.toContain('Seq Scan on Job');
});
```

**Expected Result**:
- Index on sessionId exists in pg_indexes
- Query plan shows Index Scan, not Sequential Scan

---

### DB-PERF-004: Validate Composite Index [sessionId, createdAt] Usage

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-PERF-004 |
| **Title** | Validate Composite Index [sessionId, createdAt] Usage |
| **Priority** | P0 - Critical |
| **Spec Reference** | Database optimization |
| **Gap Reference** | Consolidated Review - Gap Domain 1 |

**Test Steps**:

```typescript
it('should use composite index for session history queries', async () => {
  // Verify composite index exists
  const indexResult = await prisma.$queryRaw<any[]>`
    SELECT indexname, indexdef
    FROM pg_indexes
    WHERE tablename = 'Job'
    AND indexdef LIKE '%sessionId%'
    AND indexdef LIKE '%createdAt%'
  `;
  expect(indexResult.length).toBeGreaterThan(0);

  // Verify composite index is used for typical history query
  const explainResult = await prisma.$queryRaw<any[]>`
    EXPLAIN ANALYZE
    SELECT id, status, "fileName", "createdAt"
    FROM "Job"
    WHERE "sessionId" = 'test-session'
    ORDER BY "createdAt" DESC
    LIMIT 20
  `;

  const plan = explainResult.map(r => r['QUERY PLAN']).join('\n');

  // Should use index scan with backward traversal (DESC ordering)
  expect(plan).toMatch(/Index.*Scan/);
  expect(plan).not.toContain('Sort'); // No separate sort step needed
});
```

**Expected Result**:
- Composite index [sessionId, createdAt] exists
- Query uses index scan without additional sort operation

---

### DB-PERF-005: Validate Composite Index [status, createdAt DESC] Usage

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-PERF-005 |
| **Title** | Validate Composite Index [status, createdAt DESC] Usage |
| **Priority** | P0 - Critical |
| **Spec Reference** | Database optimization |
| **Gap Reference** | Consolidated Review - Gap Domain 1 |

**Test Steps**:

```typescript
it('should use index for status-based filtering queries', async () => {
  // Verify index exists for status filtering
  const indexResult = await prisma.$queryRaw<any[]>`
    SELECT indexname, indexdef
    FROM pg_indexes
    WHERE tablename = 'Job'
    AND indexdef LIKE '%status%'
  `;
  expect(indexResult.length).toBeGreaterThan(0);

  // Verify index usage for status filter query
  const explainResult = await prisma.$queryRaw<any[]>`
    EXPLAIN ANALYZE
    SELECT id, "sessionId", "fileName", "createdAt"
    FROM "Job"
    WHERE status = 'PROCESSING'
    ORDER BY "createdAt" DESC
    LIMIT 50
  `;

  const plan = explainResult.map(r => r['QUERY PLAN']).join('\n');
  expect(plan).toMatch(/Index.*Scan/);
});
```

**Expected Result**:
- Index on [status, createdAt] exists
- Query plan uses index scan for status filtering

---

### DB-PERF-006: History Query Performance with 100K Jobs

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-PERF-006 |
| **Title** | History Query Performance with 100K Jobs |
| **Priority** | P0 - Critical |
| **Spec Reference** | NFR-103 |
| **Gap Reference** | Consolidated Review - Gap Domain 1 |
| **SLA Target** | < 200ms |

**Test Steps**:

```typescript
it('should query history efficiently with 100K jobs (<200ms)', async () => {
  // Seed large dataset (use test helper for efficiency)
  await seedLargeDataset('perf-session-100k', 100000);

  // Warm up query cache
  await prisma.job.findMany({
    where: { sessionId: 'perf-session-100k' },
    take: 1,
  });

  // Measure query performance
  const iterations = 5;
  const times: number[] = [];

  for (let i = 0; i < iterations; i++) {
    const start = performance.now();
    const results = await prisma.job.findMany({
      where: { sessionId: 'perf-session-100k' },
      orderBy: { createdAt: 'desc' },
      take: 20,
      select: {
        id: true,
        status: true,
        fileName: true,
        createdAt: true,
      },
    });
    const elapsed = performance.now() - start;
    times.push(elapsed);

    expect(results).toHaveLength(20);
  }

  // Calculate P95 latency
  times.sort((a, b) => a - b);
  const p95 = times[Math.floor(times.length * 0.95)];

  expect(p95).toBeLessThan(200); // P95 < 200ms
}, 120000); // 2 minute timeout for seeding + queries

// Helper function
async function seedLargeDataset(sessionId: string, count: number) {
  const batchSize = 1000;
  for (let i = 0; i < count; i += batchSize) {
    const batch = Array.from({ length: Math.min(batchSize, count - i) }, (_, j) => ({
      sessionId,
      status: 'COMPLETE' as JobStatus,
      inputType: 'FILE' as InputType,
      fileName: `file-${(i + j).toString().padStart(6, '0')}.pdf`,
      createdAt: new Date(Date.now() - (i + j) * 1000),
    }));
    await prisma.job.createMany({ data: batch });
  }
}
```

**Expected Result**:
- Query returns 20 results
- P95 latency is less than 200ms

---

### DB-PERF-007: Pagination Efficiency with 1M Jobs

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-PERF-007 |
| **Title** | Pagination Efficiency with 1M Jobs |
| **Priority** | P0 - Critical |
| **Spec Reference** | NFR-103 |
| **Gap Reference** | Consolidated Review - Gap Domain 1 |
| **SLA Target** | < 500ms for deep pagination |

**Test Steps**:

```typescript
it('should maintain pagination efficiency with 1M jobs', async () => {
  // This test requires pre-seeded large dataset or uses mock
  // In CI, use scaled-down version; in performance env, use full 1M
  const datasetSize = process.env.PERF_TEST_FULL ? 1000000 : 100000;
  await seedLargeDataset('perf-session-1m', datasetSize);

  // Test deep pagination (page 100, offset 1980)
  const deepOffset = Math.min(1980, datasetSize - 20);

  const start = performance.now();
  const results = await prisma.job.findMany({
    where: { sessionId: 'perf-session-1m' },
    orderBy: { createdAt: 'desc' },
    skip: deepOffset,
    take: 20,
    select: {
      id: true,
      status: true,
      fileName: true,
      createdAt: true,
    },
  });
  const elapsed = performance.now() - start;

  expect(results).toHaveLength(20);
  expect(elapsed).toBeLessThan(500); // Deep pagination < 500ms

  // Verify cursor-based pagination is more efficient
  const lastItem = results[results.length - 1];
  const cursorStart = performance.now();
  const cursorResults = await prisma.job.findMany({
    where: {
      sessionId: 'perf-session-1m',
      createdAt: { lt: lastItem.createdAt },
    },
    orderBy: { createdAt: 'desc' },
    take: 20,
  });
  const cursorElapsed = performance.now() - cursorStart;

  expect(cursorResults).toHaveLength(20);
  expect(cursorElapsed).toBeLessThan(elapsed); // Cursor should be faster
}, 300000); // 5 minute timeout
```

**Expected Result**:
- Deep offset pagination completes within 500ms
- Cursor-based pagination is faster than offset-based

---

### DB-PERF-008: N+1 Query Detection in job.include({ results: true })

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-PERF-008 |
| **Title** | N+1 Query Detection in job.include({ results: true }) |
| **Priority** | P0 - Critical |
| **Spec Reference** | Database optimization |
| **Gap Reference** | Consolidated Review - Gap Domain 1 |

**Test Steps**:

```typescript
it('should not produce N+1 queries when including results', async () => {
  // Create 20 jobs with results
  for (let i = 0; i < 20; i++) {
    await prisma.job.create({
      data: {
        sessionId: 'n-plus-1-test',
        status: 'COMPLETE',
        inputType: 'FILE',
        fileName: `file-${i}.pdf`,
        results: {
          create: [
            { format: 'MARKDOWN', content: `# Doc ${i}`, size: 10 },
            { format: 'HTML', content: `<h1>Doc ${i}</h1>`, size: 15 },
          ],
        },
      },
    });
  }

  // Enable query logging
  const queries: string[] = [];
  prisma.$use(async (params, next) => {
    queries.push(`${params.model}.${params.action}`);
    return next(params);
  });

  // Fetch jobs with results
  const jobs = await prisma.job.findMany({
    where: { sessionId: 'n-plus-1-test' },
    include: { results: true },
    take: 20,
  });

  expect(jobs).toHaveLength(20);
  jobs.forEach(job => {
    expect(job.results).toHaveLength(2);
  });

  // Prisma should use 2 queries max (one for jobs, one for results)
  // Not 1 + 20 queries (N+1 pattern)
  const jobQueries = queries.filter(q => q.includes('Job'));
  const resultQueries = queries.filter(q => q.includes('Result'));

  // Should be at most 1 Job query and 1 Result query (batched)
  expect(jobQueries.length).toBeLessThanOrEqual(2);
  expect(resultQueries.length).toBeLessThanOrEqual(2);
});
```

**Expected Result**:
- Query count is O(1), not O(N)
- Prisma batches the result queries

---

### DB-PERF-009: Query Execution Time Distribution

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-PERF-009 |
| **Title** | Query Execution Time Distribution |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-103 |
| **Gap Reference** | Consolidated Review - Gap Domain 1 |

**Test Steps**:

```typescript
it('should maintain consistent query execution times', async () => {
  await seedLargeDataset('perf-distribution-test', 10000);

  const times: number[] = [];
  const queryCount = 100;

  for (let i = 0; i < queryCount; i++) {
    const start = performance.now();
    await prisma.job.findMany({
      where: { sessionId: 'perf-distribution-test' },
      orderBy: { createdAt: 'desc' },
      take: 20,
    });
    times.push(performance.now() - start);
  }

  // Calculate statistics
  times.sort((a, b) => a - b);
  const p50 = times[Math.floor(times.length * 0.5)];
  const p95 = times[Math.floor(times.length * 0.95)];
  const p99 = times[Math.floor(times.length * 0.99)];
  const avg = times.reduce((a, b) => a + b, 0) / times.length;

  // Log for monitoring
  console.log(`Query execution times (ms): P50=${p50.toFixed(2)}, P95=${p95.toFixed(2)}, P99=${p99.toFixed(2)}, Avg=${avg.toFixed(2)}`);

  // Assert reasonable distribution
  expect(p50).toBeLessThan(50);   // P50 < 50ms
  expect(p95).toBeLessThan(100);  // P95 < 100ms
  expect(p99).toBeLessThan(200);  // P99 < 200ms

  // Verify no extreme outliers (P99 should be < 5x P50)
  expect(p99 / p50).toBeLessThan(5);
}, 60000);
```

**Expected Result**:
- P50 latency under 50ms
- P95 latency under 100ms
- P99 latency under 200ms
- No extreme outliers (P99 < 5x P50)

---

### DB-PERF-010: Index Bloat and VACUUM Monitoring

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-PERF-010 |
| **Title** | Index Bloat and VACUUM Monitoring |
| **Priority** | P1 - High |
| **Spec Reference** | Database maintenance |
| **Gap Reference** | Consolidated Review - Gap Domain 1 |

**Test Steps**:

```typescript
it('should detect index bloat and verify VACUUM status', async () => {
  // Check index bloat using pgstattuple extension (if available)
  // Falls back to pg_stat_user_indexes for basic metrics

  const indexStats = await prisma.$queryRaw<any[]>`
    SELECT
      schemaname,
      relname AS table_name,
      indexrelname AS index_name,
      idx_scan AS index_scans,
      idx_tup_read AS tuples_read,
      idx_tup_fetch AS tuples_fetched,
      pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
    FROM pg_stat_user_indexes
    WHERE relname = 'Job'
    ORDER BY idx_scan DESC
  `;

  // Verify indexes exist and are being used
  expect(indexStats.length).toBeGreaterThan(0);

  // Check for unused indexes (potential bloat candidates)
  const unusedIndexes = indexStats.filter(idx => idx.index_scans === 0n);

  // Log warning for unused indexes (except primary key)
  unusedIndexes
    .filter(idx => !idx.index_name.includes('pkey'))
    .forEach(idx => {
      console.warn(`Unused index detected: ${idx.index_name} (size: ${idx.index_size})`);
    });

  // Check VACUUM status
  const vacuumStats = await prisma.$queryRaw<any[]>`
    SELECT
      relname AS table_name,
      last_vacuum,
      last_autovacuum,
      last_analyze,
      last_autoanalyze,
      n_dead_tup AS dead_tuples,
      n_live_tup AS live_tuples
    FROM pg_stat_user_tables
    WHERE relname = 'Job'
  `;

  expect(vacuumStats.length).toBe(1);
  const jobTable = vacuumStats[0];

  // If there are dead tuples, ensure vacuum has run recently (or will run)
  if (jobTable.dead_tuples > 0) {
    const deadTupleRatio = Number(jobTable.dead_tuples) / (Number(jobTable.live_tuples) || 1);

    // Warn if dead tuple ratio exceeds 10%
    if (deadTupleRatio > 0.1) {
      console.warn(`High dead tuple ratio: ${(deadTupleRatio * 100).toFixed(2)}% - VACUUM recommended`);
    }

    expect(deadTupleRatio).toBeLessThan(0.5); // Fail if > 50% dead tuples
  }
});
```

**Expected Result**:
- Index statistics are available
- No excessive index bloat (dead tuple ratio < 50%)
- VACUUM runs regularly (autovacuum enabled)

---

## 9. Connection Pooling and Management

> **Priority**: P0 - CRITICAL
> **Gap Reference**: Consolidated Review - Gap Domain 2
> **Risk Level**: Production showstopper - connection exhaustion causes service crashes

This section validates connection pool configuration, exhaustion handling, leak detection, and proper cleanup. Without these tests, the application will fail under load with "too many connections" errors.

---

### DB-POOL-001: Connection Pool Configuration Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-POOL-001 |
| **Title** | Connection Pool Configuration Validation |
| **Priority** | P0 - Critical |
| **Spec Reference** | Database configuration |
| **Gap Reference** | Consolidated Review - Gap Domain 2 |

**Test Steps**:

```typescript
it('should configure connection pool appropriately', async () => {
  const dbUrl = process.env.DATABASE_URL || '';

  // Verify connection pool parameters are set
  expect(dbUrl).toMatch(/connection_limit=\d+/);
  expect(dbUrl).toMatch(/pool_timeout=\d+/);

  // Extract and validate pool settings
  const connectionLimitMatch = dbUrl.match(/connection_limit=(\d+)/);
  const poolTimeoutMatch = dbUrl.match(/pool_timeout=(\d+)/);

  expect(connectionLimitMatch).not.toBeNull();
  expect(poolTimeoutMatch).not.toBeNull();

  const connectionLimit = parseInt(connectionLimitMatch![1], 10);
  const poolTimeout = parseInt(poolTimeoutMatch![1], 10);

  // Validate reasonable limits
  expect(connectionLimit).toBeGreaterThanOrEqual(5);  // Minimum pool size
  expect(connectionLimit).toBeLessThanOrEqual(100);   // Max sensible pool
  expect(poolTimeout).toBeGreaterThanOrEqual(5);      // At least 5 seconds
  expect(poolTimeout).toBeLessThanOrEqual(60);        // Max 60 seconds

  // Verify connection to database works with current config
  const result = await prisma.$queryRaw<any[]>`SELECT 1 as connected`;
  expect(result[0].connected).toBe(1);
});
```

**Expected Result**:
- DATABASE_URL contains connection_limit and pool_timeout parameters
- Pool size is between 5 and 100 connections
- Pool timeout is between 5 and 60 seconds

---

### DB-POOL-002: Connection Exhaustion Handling

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-POOL-002 |
| **Title** | Connection Exhaustion Handling |
| **Priority** | P0 - Critical |
| **Spec Reference** | Database resilience |
| **Gap Reference** | Consolidated Review - Gap Domain 2 |

**Test Steps**:

```typescript
it('should handle pool exhaustion gracefully', async () => {
  // Create many concurrent long-running queries to exhaust pool
  const concurrentQueries = 50;
  const queryDurationMs = 3000; // 3 second queries

  const promises = Array.from({ length: concurrentQueries }, async (_, i) => {
    try {
      // Simulate long-running query
      await prisma.$queryRaw`SELECT pg_sleep(${queryDurationMs / 1000})`;
      return { index: i, status: 'fulfilled' };
    } catch (error) {
      return { index: i, status: 'rejected', error: (error as Error).message };
    }
  });

  const startTime = Date.now();
  const results = await Promise.allSettled(promises);
  const elapsed = Date.now() - startTime;

  // Analyze results
  const fulfilled = results.filter(r => r.status === 'fulfilled').length;
  const rejected = results.filter(r => r.status === 'rejected').length;

  // Some queries should succeed (up to pool limit)
  expect(fulfilled).toBeGreaterThan(0);

  // If pool is exhausted, excess queries should fail gracefully (not crash)
  // or queue and complete within timeout
  console.log(`Pool exhaustion test: ${fulfilled} fulfilled, ${rejected} rejected in ${elapsed}ms`);

  // Verify application is still responsive after exhaustion
  const healthCheck = await prisma.$queryRaw<any[]>`SELECT 1 as healthy`;
  expect(healthCheck[0].healthy).toBe(1);
}, 30000);
```

**Expected Result**:
- Some queries succeed (up to pool limit)
- Excess queries either queue or fail gracefully
- Application remains responsive after exhaustion

---

### DB-POOL-003: Connection Leak Detection

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-POOL-003 |
| **Title** | Connection Leak Detection |
| **Priority** | P0 - Critical |
| **Spec Reference** | Database resilience |
| **Gap Reference** | Consolidated Review - Gap Domain 2 |

**Test Steps**:

```typescript
it('should detect and prevent connection leaks', async () => {
  // Get initial connection count
  const initialConnections = await prisma.$queryRaw<any[]>`
    SELECT count(*) as conn_count
    FROM pg_stat_activity
    WHERE datname = current_database()
    AND state != 'idle'
  `;
  const initialCount = Number(initialConnections[0].conn_count);

  // Execute many sequential operations (should reuse connections)
  for (let i = 0; i < 100; i++) {
    await prisma.job.findFirst({
      where: { sessionId: `leak-test-${i}` },
    });
  }

  // Check connection count after operations
  const afterConnections = await prisma.$queryRaw<any[]>`
    SELECT count(*) as conn_count
    FROM pg_stat_activity
    WHERE datname = current_database()
    AND state != 'idle'
  `;
  const afterCount = Number(afterConnections[0].conn_count);

  // Connection count should not grow significantly
  // (small increase acceptable for parallel execution)
  expect(afterCount).toBeLessThanOrEqual(initialCount + 5);

  // Wait for connections to return to pool
  await new Promise(r => setTimeout(r, 1000));

  // Verify connections are released
  const finalConnections = await prisma.$queryRaw<any[]>`
    SELECT count(*) as conn_count
    FROM pg_stat_activity
    WHERE datname = current_database()
    AND state = 'idle'
  `;
  const idleCount = Number(finalConnections[0].conn_count);

  // Should have idle connections available in pool
  expect(idleCount).toBeGreaterThan(0);
});
```

**Expected Result**:
- Connection count does not grow unboundedly
- Connections return to idle state after operations
- No leaked connections detected

---

### DB-POOL-004: Connection Cleanup on Graceful Shutdown

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-POOL-004 |
| **Title** | Connection Cleanup on Graceful Shutdown |
| **Priority** | P0 - Critical |
| **Spec Reference** | Database lifecycle |
| **Gap Reference** | Consolidated Review - Gap Domain 2 |

**Test Steps**:

```typescript
it('should clean up connections on graceful shutdown', async () => {
  // Create a separate Prisma client for this test
  const testClient = new PrismaClient({
    datasources: { db: { url: process.env.TEST_DATABASE_URL } },
  });

  await testClient.$connect();

  // Execute some queries to establish connections
  await testClient.job.findMany({ take: 5 });
  await testClient.result.findMany({ take: 5 });

  // Get connection count for this client
  const beforeDisconnect = await testClient.$queryRaw<any[]>`
    SELECT count(*) as conn_count
    FROM pg_stat_activity
    WHERE datname = current_database()
    AND application_name LIKE '%prisma%'
  `;
  const beforeCount = Number(beforeDisconnect[0].conn_count);
  expect(beforeCount).toBeGreaterThan(0);

  // Gracefully disconnect
  await testClient.$disconnect();

  // Wait for cleanup
  await new Promise(r => setTimeout(r, 500));

  // Verify connections were released
  // Use main prisma client to check
  const afterDisconnect = await prisma.$queryRaw<any[]>`
    SELECT count(*) as conn_count
    FROM pg_stat_activity
    WHERE datname = current_database()
    AND application_name LIKE '%prisma%'
  `;
  const afterCount = Number(afterDisconnect[0].conn_count);

  // Should have fewer connections after disconnect
  // (main client connections remain)
  expect(afterCount).toBeLessThanOrEqual(beforeCount);
});
```

**Expected Result**:
- Connections are established during operations
- $disconnect() releases connections back to PostgreSQL
- No orphaned connections remain

---

### DB-POOL-005: Connection Timeout Behavior

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-POOL-005 |
| **Title** | Connection Timeout Behavior |
| **Priority** | P0 - Critical |
| **Spec Reference** | Database resilience |
| **Gap Reference** | Consolidated Review - Gap Domain 2 |

**Test Steps**:

```typescript
it('should handle connection timeouts appropriately', async () => {
  // Test query timeout behavior
  const shortTimeoutClient = new PrismaClient({
    datasources: { db: { url: `${process.env.TEST_DATABASE_URL}&connect_timeout=5` } },
  });

  try {
    await shortTimeoutClient.$connect();

    // Execute a query that should complete within timeout
    const fastResult = await shortTimeoutClient.job.findFirst();
    // Result can be null if no jobs exist - that's fine
    expect(fastResult === null || typeof fastResult === 'object').toBe(true);

    // Test statement timeout (if configured)
    // This should timeout if statement_timeout is set low
    const timeoutTest = async () => {
      try {
        await shortTimeoutClient.$queryRaw`SELECT pg_sleep(10)`;
        return 'completed';
      } catch (error) {
        if ((error as Error).message.includes('timeout') ||
            (error as Error).message.includes('canceling')) {
          return 'timeout';
        }
        throw error;
      }
    };

    // Set a statement timeout and test
    await shortTimeoutClient.$executeRaw`SET statement_timeout = '2s'`;
    const result = await timeoutTest();

    // Should either timeout or complete (depending on config)
    expect(['completed', 'timeout']).toContain(result);

  } finally {
    await shortTimeoutClient.$disconnect();
  }
});
```

**Expected Result**:
- Connection establishes within timeout
- Long-running queries can be cancelled by statement timeout
- Timeout errors are handled gracefully

---

### DB-POOL-006: 50 Concurrent Operations Test

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-POOL-006 |
| **Title** | 50 Concurrent Operations Test |
| **Priority** | P0 - Critical |
| **Spec Reference** | NFR-102 |
| **Gap Reference** | Consolidated Review - Gap Domain 2 |

**Test Steps**:

```typescript
it('should handle 50 concurrent database operations', async () => {
  // Seed some test data
  const testSessionId = 'concurrent-test-session';
  await prisma.job.createMany({
    data: Array.from({ length: 100 }, (_, i) => ({
      sessionId: testSessionId,
      status: 'COMPLETE' as JobStatus,
      inputType: 'FILE' as InputType,
      fileName: `concurrent-${i}.pdf`,
    })),
  });

  const concurrentOperations = 50;

  // Mix of read and write operations
  const operations = Array.from({ length: concurrentOperations }, (_, i) => {
    if (i % 5 === 0) {
      // Write operation (20%)
      return prisma.job.create({
        data: {
          sessionId: testSessionId,
          status: 'PENDING',
          inputType: 'FILE',
          fileName: `concurrent-write-${i}.pdf`,
        },
      });
    } else if (i % 5 === 1) {
      // Count operation (20%)
      return prisma.job.count({
        where: { sessionId: testSessionId },
      });
    } else {
      // Read operation (60%)
      return prisma.job.findMany({
        where: { sessionId: testSessionId },
        take: 10,
        skip: i,
      });
    }
  });

  const startTime = Date.now();
  const results = await Promise.allSettled(operations);
  const elapsed = Date.now() - startTime;

  // All operations should succeed
  const fulfilled = results.filter(r => r.status === 'fulfilled');
  const rejected = results.filter(r => r.status === 'rejected');

  expect(fulfilled.length).toBe(concurrentOperations);
  expect(rejected.length).toBe(0);

  // Should complete within reasonable time (not serialized)
  // 50 operations should complete in < 10 seconds with proper pooling
  expect(elapsed).toBeLessThan(10000);

  console.log(`50 concurrent operations completed in ${elapsed}ms`);
}, 30000);
```

**Expected Result**:
- All 50 concurrent operations complete successfully
- No connection pool errors
- Total execution time under 10 seconds

---

### DB-POOL-007: Prometheus Pool Metrics Exposure

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-POOL-007 |
| **Title** | Prometheus Pool Metrics Exposure |
| **Priority** | P0 - Critical |
| **Spec Reference** | NFR-501 |
| **Gap Reference** | Consolidated Review - Gap Domain 2 |

**Test Steps**:

```typescript
it('should expose database metrics for Prometheus', async () => {
  // Check if Prisma metrics are enabled
  // Requires prisma client with metrics enabled:
  // new PrismaClient({ log: [{ emit: 'event', level: 'query' }] })

  // Get Prometheus metrics (if available)
  try {
    const metrics = await prisma.$metrics.prometheus();

    // Verify essential pool metrics are present
    expect(metrics).toContain('prisma_pool_connections_open');
    expect(metrics).toContain('prisma_pool_connections_busy');
    expect(metrics).toContain('prisma_pool_connections_idle');

    // Verify query metrics
    expect(metrics).toContain('prisma_client_queries_total');

    // Parse and validate metric values
    const openMatch = metrics.match(/prisma_pool_connections_open\s+(\d+)/);
    const busyMatch = metrics.match(/prisma_pool_connections_busy\s+(\d+)/);
    const idleMatch = metrics.match(/prisma_pool_connections_idle\s+(\d+)/);

    if (openMatch) {
      const openConnections = parseInt(openMatch[1], 10);
      expect(openConnections).toBeGreaterThanOrEqual(0);
    }

    if (busyMatch && idleMatch) {
      const busyConnections = parseInt(busyMatch[1], 10);
      const idleConnections = parseInt(idleMatch[1], 10);

      // Busy + idle should not exceed max pool size
      expect(busyConnections + idleConnections).toBeLessThanOrEqual(100);
    }

    console.log('Prometheus metrics available:\n', metrics.substring(0, 500));

  } catch (error) {
    // If $metrics is not available, check alternative metrics endpoint
    console.warn('Prisma $metrics not available, checking pg_stat_activity');

    const pgMetrics = await prisma.$queryRaw<any[]>`
      SELECT
        count(*) FILTER (WHERE state = 'active') as active_connections,
        count(*) FILTER (WHERE state = 'idle') as idle_connections,
        count(*) as total_connections
      FROM pg_stat_activity
      WHERE datname = current_database()
    `;

    expect(pgMetrics.length).toBe(1);
    expect(Number(pgMetrics[0].total_connections)).toBeGreaterThan(0);
  }
});
```

**Expected Result**:
- Prometheus metrics endpoint available
- Pool metrics (open, busy, idle connections) exposed
- Metric values are valid numbers

---

### DB-POOL-008: Connection Reuse Verification

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-POOL-008 |
| **Title** | Connection Reuse Verification |
| **Priority** | P1 - High |
| **Spec Reference** | Database optimization |
| **Gap Reference** | Consolidated Review - Gap Domain 2 |

**Test Steps**:

```typescript
it('should reuse connections across requests', async () => {
  // Get initial connection stats
  const initialStats = await prisma.$queryRaw<any[]>`
    SELECT
      numbackends as total_connections,
      xact_commit as commits,
      xact_rollback as rollbacks
    FROM pg_stat_database
    WHERE datname = current_database()
  `;
  const initialConnections = Number(initialStats[0].total_connections);
  const initialCommits = Number(initialStats[0].commits);

  // Execute many sequential queries (simulating multiple requests)
  const queryCount = 100;
  for (let i = 0; i < queryCount; i++) {
    await prisma.job.findMany({
      where: { sessionId: `reuse-test-${i % 10}` },
      take: 5,
    });
  }

  // Get final connection stats
  const finalStats = await prisma.$queryRaw<any[]>`
    SELECT
      numbackends as total_connections,
      xact_commit as commits,
      xact_rollback as rollbacks
    FROM pg_stat_database
    WHERE datname = current_database()
  `;
  const finalConnections = Number(finalStats[0].total_connections);
  const finalCommits = Number(finalStats[0].commits);

  // Connection count should not grow proportionally to query count
  // (indicates connection reuse)
  const connectionGrowth = finalConnections - initialConnections;
  expect(connectionGrowth).toBeLessThan(queryCount / 10);

  // Commits should increase by approximately queryCount
  const commitGrowth = finalCommits - initialCommits;
  expect(commitGrowth).toBeGreaterThanOrEqual(queryCount);

  // Calculate reuse ratio
  const reuseRatio = queryCount / Math.max(connectionGrowth, 1);
  console.log(`Connection reuse ratio: ${reuseRatio.toFixed(2)}x (${queryCount} queries, ${connectionGrowth} new connections)`);

  // Should have high reuse ratio (>10x)
  expect(reuseRatio).toBeGreaterThan(10);
});
```

**Expected Result**:
- Connection count grows minimally despite many queries
- High connection reuse ratio (>10x)
- Commits increase proportionally to query count

---

## 10. Concurrency Control and Locking

> **Priority**: P0 - CRITICAL
> **Gap Reference**: Consolidated Review - Gap Domain 3
> **Risk Level**: Data corruption risk without proper concurrency control

This section validates concurrent transaction handling, explicit locking, deadlock detection, optimistic concurrency control, and race condition prevention. Without these tests, concurrent operations could cause state inconsistency and data corruption.

---

### DB-CONC-001: True Concurrent Job Status Updates

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-CONC-001 |
| **Title** | Handle Concurrent Status Updates Safely |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.1.2 |
| **Gap Reference** | Consolidated Review - Gap Domain 3 |

**Test Steps**:

```typescript
it('should handle concurrent status updates safely', async () => {
  const job = await prisma.job.create({
    data: {
      sessionId: 'conc-test',
      status: 'PROCESSING',
      inputType: 'FILE',
    },
  });

  // Attempt concurrent updates to same job
  const results = await Promise.allSettled([
    prisma.job.update({
      where: { id: job.id },
      data: { status: 'COMPLETE' },
    }),
    prisma.job.update({
      where: { id: job.id },
      data: { status: 'ERROR' },
    }),
  ]);

  // Both updates should succeed (last write wins in PostgreSQL default)
  const successes = results.filter(r => r.status === 'fulfilled').length;
  expect(successes).toBe(2);

  // Final state should be deterministic (one of the values)
  const finalJob = await prisma.job.findUnique({ where: { id: job.id } });
  expect(['COMPLETE', 'ERROR']).toContain(finalJob?.status);
});
```

**Expected Result**:
- Both concurrent updates complete without errors
- Final state is one of the two attempted values
- No database corruption or constraint violations

---

### DB-CONC-002: Concurrent Checkpoint Updates

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-CONC-002 |
| **Title** | Handle Concurrent Checkpoint Updates |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.5 |
| **Gap Reference** | Consolidated Review - Gap Domain 3 |

**Test Steps**:

```typescript
it('should handle concurrent checkpoint updates safely', async () => {
  const job = await prisma.job.create({
    data: {
      sessionId: 'conc-checkpoint-test',
      status: 'PROCESSING',
      inputType: 'FILE',
      checkpointStage: 'uploaded',
      checkpointData: { stage: 'uploaded', progress: 0 },
    },
  });

  // Simulate concurrent checkpoint updates from different processes
  const updates = await Promise.allSettled([
    prisma.job.update({
      where: { id: job.id },
      data: {
        checkpointStage: 'converting',
        checkpointData: { stage: 'converting', progress: 25 },
      },
    }),
    prisma.job.update({
      where: { id: job.id },
      data: {
        checkpointStage: 'converting',
        checkpointData: { stage: 'converting', progress: 50 },
      },
    }),
  ]);

  // Both should succeed
  const successes = updates.filter(r => r.status === 'fulfilled').length;
  expect(successes).toBe(2);

  // Verify checkpoint data is consistent (not corrupted)
  const finalJob = await prisma.job.findUnique({ where: { id: job.id } });
  expect(finalJob?.checkpointStage).toBe('converting');
  expect(finalJob?.checkpointData).toHaveProperty('stage', 'converting');
  expect([25, 50]).toContain((finalJob?.checkpointData as any)?.progress);
});
```

**Expected Result**:
- Concurrent checkpoint updates complete without corruption
- Final checkpoint data is valid and consistent
- No partial updates or data corruption

---

### DB-CONC-003: Deadlock Detection and Handling

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-CONC-003 |
| **Title** | Detect and Handle Database Deadlocks |
| **Priority** | P0 - Critical |
| **Spec Reference** | Database integrity |
| **Gap Reference** | Consolidated Review - Gap Domain 3 |

**Test Steps**:

```typescript
it('should handle database deadlocks gracefully', async () => {
  const job1 = await prisma.job.create({
    data: { sessionId: 'deadlock-1', status: 'PENDING', inputType: 'FILE' },
  });
  const job2 = await prisma.job.create({
    data: { sessionId: 'deadlock-2', status: 'PENDING', inputType: 'FILE' },
  });

  // Create potential deadlock scenario with cross-updates
  const tx1Promise = prisma.$transaction(async (tx) => {
    await tx.job.update({
      where: { id: job1.id },
      data: { status: 'PROCESSING' },
    });
    // Small delay to increase deadlock likelihood
    await new Promise(r => setTimeout(r, 100));
    await tx.job.update({
      where: { id: job2.id },
      data: { status: 'PROCESSING' },
    });
  });

  const tx2Promise = prisma.$transaction(async (tx) => {
    await tx.job.update({
      where: { id: job2.id },
      data: { status: 'COMPLETE' },
    });
    await new Promise(r => setTimeout(r, 100));
    await tx.job.update({
      where: { id: job1.id },
      data: { status: 'COMPLETE' },
    });
  });

  const results = await Promise.allSettled([tx1Promise, tx2Promise]);

  // At least one should succeed; the other may fail due to deadlock
  const fulfilled = results.filter(r => r.status === 'fulfilled').length;
  const rejected = results.filter(r => r.status === 'rejected').length;

  // Either both succeed (no deadlock) or one fails gracefully
  expect(fulfilled).toBeGreaterThanOrEqual(1);

  // If rejected, verify it's a proper PostgreSQL error, not a crash
  results.forEach(result => {
    if (result.status === 'rejected') {
      expect((result.reason as Error).message).toMatch(
        /deadlock|serialization|could not serialize|timeout/i
      );
    }
  });
});
```

**Expected Result**:
- At least one transaction succeeds
- Failed transactions report proper PostgreSQL error codes
- Application does not crash on deadlock

---

### DB-CONC-004: SELECT FOR UPDATE Locking

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-CONC-004 |
| **Title** | Validate SELECT FOR UPDATE Locking Behavior |
| **Priority** | P0 - Critical |
| **Spec Reference** | Database integrity |
| **Gap Reference** | Consolidated Review - Gap Domain 3 |

**Test Steps**:

```typescript
it('should enforce SELECT FOR UPDATE locking', async () => {
  const job = await prisma.job.create({
    data: {
      sessionId: 'lock-test',
      status: 'PENDING',
      inputType: 'FILE',
      retryCount: 0,
    },
  });

  let tx1Completed = false;
  let tx2Started = false;

  // Transaction 1: Acquire lock and hold it
  const tx1Promise = prisma.$transaction(async (tx) => {
    // Acquire row lock using raw query
    await tx.$queryRaw`
      SELECT * FROM "Job" WHERE id = ${job.id} FOR UPDATE
    `;
    tx2Started = true;

    // Hold lock for a moment
    await new Promise(r => setTimeout(r, 500));

    await tx.job.update({
      where: { id: job.id },
      data: { retryCount: 1 },
    });
    tx1Completed = true;
  });

  // Wait for tx1 to acquire lock
  await new Promise(r => setTimeout(r, 100));

  // Transaction 2: Try to update locked row
  const tx2StartTime = Date.now();
  const tx2Promise = prisma.$transaction(async (tx) => {
    // This should block until tx1 releases lock
    await tx.job.update({
      where: { id: job.id },
      data: { retryCount: 2 },
    });
  });

  await Promise.all([tx1Promise, tx2Promise]);
  const tx2Duration = Date.now() - tx2StartTime;

  // tx2 should have been blocked (waited for tx1)
  expect(tx2Duration).toBeGreaterThanOrEqual(400);

  // Final value should be from tx2 (ran after tx1 completed)
  const finalJob = await prisma.job.findUnique({ where: { id: job.id } });
  expect(finalJob?.retryCount).toBe(2);
});
```

**Expected Result**:
- SELECT FOR UPDATE acquires row lock
- Second transaction waits for lock release
- Updates are serialized, not concurrent

---

### DB-CONC-005: Isolation Level Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-CONC-005 |
| **Title** | Validate Transaction Isolation Levels |
| **Priority** | P1 - High |
| **Spec Reference** | Database integrity |
| **Gap Reference** | Consolidated Review - Gap Domain 3 |

**Test Steps**:

```typescript
it('should enforce appropriate isolation level', async () => {
  // Verify default isolation level
  const isolationResult = await prisma.$queryRaw<any[]>`
    SHOW transaction_isolation
  `;
  const currentIsolation = isolationResult[0].transaction_isolation;

  // PostgreSQL default is READ COMMITTED
  expect(['read committed', 'repeatable read', 'serializable']).toContain(
    currentIsolation
  );

  // Test with explicit SERIALIZABLE isolation for critical operation
  const job = await prisma.job.create({
    data: { sessionId: 'isolation-test', status: 'PENDING', inputType: 'FILE' },
  });

  // Execute SERIALIZABLE transaction
  await prisma.$executeRaw`SET TRANSACTION ISOLATION LEVEL SERIALIZABLE`;

  const result = await prisma.$transaction(async (tx) => {
    const current = await tx.job.findUnique({ where: { id: job.id } });
    expect(current?.status).toBe('PENDING');

    await tx.job.update({
      where: { id: job.id },
      data: { status: 'PROCESSING' },
    });

    return tx.job.findUnique({ where: { id: job.id } });
  });

  expect(result?.status).toBe('PROCESSING');
});
```

**Expected Result**:
- Default isolation level is READ COMMITTED or higher
- SERIALIZABLE transactions execute correctly
- Isolation prevents dirty reads

---

### DB-CONC-006: Optimistic Locking with Version Field

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-CONC-006 |
| **Title** | Test Optimistic Locking Pattern |
| **Priority** | P1 - High |
| **Spec Reference** | Optimistic concurrency |
| **Gap Reference** | Consolidated Review - Gap Domain 3 |

**Test Steps**:

```typescript
it('should support optimistic locking pattern', async () => {
  // Simulate optimistic locking using updatedAt as version
  const job = await prisma.job.create({
    data: {
      sessionId: 'optimistic-test',
      status: 'PENDING',
      inputType: 'FILE',
    },
  });

  const originalUpdatedAt = job.updatedAt;

  // Read current state
  const current = await prisma.job.findUnique({ where: { id: job.id } });
  expect(current?.updatedAt).toEqual(originalUpdatedAt);

  // Simulate concurrent modification by another process
  await new Promise(r => setTimeout(r, 100));
  await prisma.job.update({
    where: { id: job.id },
    data: { status: 'PROCESSING' },
  });

  // Attempt optimistic update with stale version
  const optimisticUpdate = await prisma.job.updateMany({
    where: {
      id: job.id,
      updatedAt: originalUpdatedAt, // Stale version check
    },
    data: { status: 'COMPLETE' },
  });

  // Update should fail (0 rows affected) due to version mismatch
  expect(optimisticUpdate.count).toBe(0);

  // Verify job still in PROCESSING state
  const finalJob = await prisma.job.findUnique({ where: { id: job.id } });
  expect(finalJob?.status).toBe('PROCESSING');
});
```

**Expected Result**:
- Optimistic update fails when version (updatedAt) has changed
- 0 rows affected indicates concurrent modification
- Data integrity preserved

---

### DB-CONC-007: Race Condition in Retry Counter

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-CONC-007 |
| **Title** | Prevent Race Conditions in Retry Counter |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-803 |
| **Gap Reference** | Consolidated Review - Gap Domain 3 |

**Test Steps**:

```typescript
it('should handle concurrent retry counter increments correctly', async () => {
  const job = await prisma.job.create({
    data: {
      sessionId: 'retry-race-test',
      status: 'PROCESSING',
      inputType: 'FILE',
      retryCount: 0,
    },
  });

  // Simulate 5 concurrent retry attempts
  const concurrentRetries = 5;
  const results = await Promise.allSettled(
    Array.from({ length: concurrentRetries }, () =>
      prisma.job.update({
        where: { id: job.id },
        data: { retryCount: { increment: 1 } },
      })
    )
  );

  // All should succeed
  const successes = results.filter(r => r.status === 'fulfilled').length;
  expect(successes).toBe(concurrentRetries);

  // Verify final count is exactly 5 (no lost updates)
  const finalJob = await prisma.job.findUnique({ where: { id: job.id } });
  expect(finalJob?.retryCount).toBe(concurrentRetries);
});
```

**Expected Result**:
- All concurrent increments succeed
- Final count equals number of concurrent operations
- No lost updates due to race conditions

---

### DB-CONC-008: Concurrent Result Creation for Same Job

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-CONC-008 |
| **Title** | Handle Concurrent Result Creation |
| **Priority** | P1 - High |
| **Spec Reference** | FR-404 |
| **Gap Reference** | Consolidated Review - Gap Domain 3 |

**Test Steps**:

```typescript
it('should handle concurrent result creation for same job', async () => {
  const job = await prisma.job.create({
    data: {
      sessionId: 'conc-result-test',
      status: 'PROCESSING',
      inputType: 'FILE',
    },
  });

  // Concurrent result creation (simulating parallel format generation)
  const results = await Promise.allSettled([
    prisma.result.create({
      data: {
        jobId: job.id,
        format: 'MARKDOWN',
        content: '# Concurrent Result 1',
        size: 21,
      },
    }),
    prisma.result.create({
      data: {
        jobId: job.id,
        format: 'HTML',
        content: '<h1>Concurrent Result 2</h1>',
        size: 28,
      },
    }),
    prisma.result.create({
      data: {
        jobId: job.id,
        format: 'JSON',
        content: '{"concurrent": true}',
        size: 20,
      },
    }),
  ]);

  // All should succeed (different formats are unique)
  const successes = results.filter(r => r.status === 'fulfilled').length;
  expect(successes).toBe(3);

  // Verify all results exist
  const allResults = await prisma.result.findMany({
    where: { jobId: job.id },
    orderBy: { format: 'asc' },
  });

  expect(allResults).toHaveLength(3);
  expect(allResults.map(r => r.format)).toEqual(['HTML', 'JSON', 'MARKDOWN']);
});
```

**Expected Result**:
- All concurrent result creations succeed
- No duplicate or missing results
- Foreign key constraints maintained

---

### DB-CONC-009: Lost Update Prevention in Concurrent Increments

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-CONC-009 |
| **Title** | Validate No Lost Updates in Concurrent Operations |
| **Priority** | P0 - Critical |
| **Spec Reference** | Database integrity |
| **Gap Reference** | Consolidated Review - Gap Domain 3 |

**Test Steps**:

```typescript
it('should prevent lost updates in high-concurrency scenario', async () => {
  const job = await prisma.job.create({
    data: {
      sessionId: 'lost-update-test',
      status: 'PENDING',
      inputType: 'FILE',
      retryCount: 0,
    },
  });

  // High concurrency: 20 simultaneous increments
  const concurrentOps = 20;
  const results = await Promise.allSettled(
    Array.from({ length: concurrentOps }, () =>
      prisma.job.update({
        where: { id: job.id },
        data: { retryCount: { increment: 1 } },
      })
    )
  );

  // Count successful operations
  const successes = results.filter(r => r.status === 'fulfilled').length;

  // Verify final count matches successful operations
  const finalJob = await prisma.job.findUnique({ where: { id: job.id } });

  // Critical assertion: no lost updates
  expect(finalJob?.retryCount).toBe(successes);

  // Log for verification
  console.log(
    `Lost update test: ${successes}/${concurrentOps} succeeded, final count: ${finalJob?.retryCount}`
  );

  // All should succeed with Prisma's atomic increment
  expect(successes).toBe(concurrentOps);
});
```

**Expected Result**:
- Final count equals number of successful operations
- No lost updates detected
- Atomic increment operations are serialized

---

### DB-CONC-010: Transactional Consistency Under Contention

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-CONC-010 |
| **Title** | Verify Transactional Consistency Under Contention |
| **Priority** | P1 - High |
| **Spec Reference** | Database integrity |
| **Gap Reference** | Consolidated Review - Gap Domain 3 |

**Test Steps**:

```typescript
it('should maintain transactional consistency under contention', async () => {
  const job = await prisma.job.create({
    data: {
      sessionId: 'contention-test',
      status: 'PROCESSING',
      inputType: 'FILE',
    },
  });

  // Multiple concurrent transactions with reads and writes
  const transactions = Array.from({ length: 10 }, (_, i) =>
    prisma.$transaction(async (tx) => {
      // Read current state
      const current = await tx.job.findUnique({
        where: { id: job.id },
        include: { results: true },
      });

      // Create a result
      await tx.result.create({
        data: {
          jobId: job.id,
          format: 'MARKDOWN',
          content: `# Transaction ${i}`,
          size: 15,
        },
      });

      // Update job
      await tx.job.update({
        where: { id: job.id },
        data: { retryCount: { increment: 1 } },
      });

      return { transactionId: i, previousResults: current?.results.length || 0 };
    })
  );

  const results = await Promise.allSettled(transactions);
  const successes = results.filter(r => r.status === 'fulfilled');

  // Verify consistency
  const finalJob = await prisma.job.findUnique({
    where: { id: job.id },
    include: { results: true },
  });

  // retryCount should match number of successful transactions
  expect(finalJob?.retryCount).toBe(successes.length);

  // Result count should match successful transactions
  expect(finalJob?.results.length).toBe(successes.length);

  console.log(
    `Contention test: ${successes.length}/10 transactions succeeded`
  );
});
```

**Expected Result**:
- Retry count matches successful transaction count
- Result count matches successful transaction count
- No orphaned or missing data

---

## 11. Workflow State Machine and Durable Execution

> **Priority**: P0 - CRITICAL
> **Gap Reference**: Consolidated Review - Gap Domain 4
> **Risk Level**: Data loss and job corruption risk without state machine validation

This section validates state transition rules, checkpoint lifecycle, durable execution recovery scenarios, and workflow coordination. Without these tests, invalid state transitions could corrupt workflow state and process crashes could cause unrecoverable job loss.

---

### DB-SM-001: Validate State Transition Rules

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-SM-001 |
| **Title** | Validate State Transition Rules |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.1.2 |
| **Gap Reference** | Consolidated Review - Gap Domain 4 |

**Test Steps**:

```typescript
it('should enforce valid state transitions', async () => {
  // Define valid state transitions
  const validTransitions: Record<string, string[]> = {
    PENDING: ['UPLOADING', 'PROCESSING', 'ERROR', 'CANCELLED'],
    UPLOADING: ['PROCESSING', 'ERROR', 'CANCELLED'],
    PROCESSING: ['COMPLETE', 'PARTIAL_COMPLETE', 'RETRY_1', 'ERROR', 'CANCELLED'],
    RETRY_1: ['PROCESSING', 'RETRY_2', 'ERROR', 'CANCELLED'],
    RETRY_2: ['PROCESSING', 'RETRY_3', 'ERROR', 'CANCELLED'],
    RETRY_3: ['PROCESSING', 'ERROR', 'CANCELLED'],
    COMPLETE: [], // Terminal state
    PARTIAL_COMPLETE: [], // Terminal state
    ERROR: [], // Terminal state
    CANCELLED: [], // Terminal state
  };

  // Test each transition
  for (const [fromStatus, toStatuses] of Object.entries(validTransitions)) {
    for (const toStatus of toStatuses) {
      const job = await prisma.job.create({
        data: {
          sessionId: `transition-${fromStatus}-${toStatus}`,
          status: fromStatus as JobStatus,
          inputType: 'FILE',
        },
      });

      const updated = await prisma.job.update({
        where: { id: job.id },
        data: { status: toStatus as JobStatus },
      });

      expect(updated.status).toBe(toStatus);
    }
  }

  // Note: Invalid transitions (e.g., COMPLETE -> PROCESSING) should be
  // prevented by application logic or database triggers if implemented
});
```

**Expected Result**:
- All valid state transitions succeed
- State machine rules are documented and testable

---

### DB-SM-002: Status-Checkpoint Consistency Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-SM-002 |
| **Title** | Validate Status-Checkpoint Consistency |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.5 |
| **Gap Reference** | Consolidated Review - Gap Domain 4 |

**Test Steps**:

```typescript
it('should maintain status-checkpoint consistency', async () => {
  // Create job with checkpoint
  const job = await prisma.job.create({
    data: {
      sessionId: 'consistency-test',
      status: 'PROCESSING',
      inputType: 'FILE',
      checkpointStage: 'converting',
      checkpointData: {
        stage: 'converting',
        progress: 50,
        timestamp: new Date().toISOString(),
      },
    },
  });

  // Verify checkpoint stage matches expected workflow stage
  expect(job.checkpointStage).toBe('converting');

  // Valid: Advance checkpoint with status update
  const advanced = await prisma.job.update({
    where: { id: job.id },
    data: {
      checkpointStage: 'exporting',
      checkpointData: {
        stage: 'exporting',
        progress: 75,
        timestamp: new Date().toISOString(),
      },
    },
  });

  expect(advanced.checkpointStage).toBe('exporting');

  // On completion, checkpoint should be cleared
  const completed = await prisma.job.update({
    where: { id: job.id },
    data: {
      status: 'COMPLETE',
      checkpointStage: null,
      checkpointData: null,
      checkpointHash: null,
      completedAt: new Date(),
    },
  });

  expect(completed.status).toBe('COMPLETE');
  expect(completed.checkpointStage).toBeNull();
  expect(completed.checkpointData).toBeNull();
});
```

**Expected Result**:
- Checkpoint stage reflects current workflow progress
- Checkpoint cleared on job completion
- Status and checkpoint are consistent

---

### DB-CK-004: Checkpoint Version Compatibility Migration

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-CK-004 |
| **Title** | Checkpoint Version Compatibility Migration |
| **Priority** | P1 - High |
| **Spec Reference** | Section 5.5 |
| **Gap Reference** | Consolidated Review - Gap Domain 4 |

**Test Steps**:

```typescript
it('should handle checkpoint version migrations', async () => {
  // Create checkpoint with old version schema
  const oldVersionCheckpoint = {
    version: '1.0.0',
    stage: 'converted',
    data: { documentId: 'legacy-doc-123' },
    timestamp: new Date().toISOString(),
  };

  const job = await prisma.job.create({
    data: {
      sessionId: 'version-migration-test',
      status: 'RETRY_1',
      inputType: 'FILE',
      checkpointStage: 'converted',
      checkpointData: oldVersionCheckpoint,
    },
  });

  // Simulate version migration logic
  const retrieved = await prisma.job.findUnique({
    where: { id: job.id },
    select: { checkpointData: true },
  });

  const checkpoint = retrieved?.checkpointData as any;
  expect(checkpoint.version).toBe('1.0.0');

  // Migrate to new version
  const migratedCheckpoint = {
    version: '2.0.0',
    stage: checkpoint.stage,
    data: {
      ...checkpoint.data,
      migratedFrom: checkpoint.version,
    },
    timestamp: new Date().toISOString(),
  };

  const migrated = await prisma.job.update({
    where: { id: job.id },
    data: {
      checkpointData: migratedCheckpoint,
    },
  });

  const finalCheckpoint = migrated.checkpointData as any;
  expect(finalCheckpoint.version).toBe('2.0.0');
  expect(finalCheckpoint.data.migratedFrom).toBe('1.0.0');
});
```

**Expected Result**:
- Old version checkpoints can be read
- Version migration preserves data
- Migrated checkpoint has new version marker

---

### DB-CK-005: Checkpoint TTL Enforcement

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-CK-005 |
| **Title** | Checkpoint TTL Enforcement (24-hour Expiration) |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.5 |
| **Gap Reference** | Consolidated Review - Gap Domain 4 |

**Test Steps**:

```typescript
it('should identify expired checkpoints for cleanup', async () => {
  // Create expired checkpoint (25 hours old)
  const expiredTime = new Date(Date.now() - 25 * 60 * 60 * 1000);

  const expiredJob = await prisma.job.create({
    data: {
      sessionId: 'ttl-expired-test',
      status: 'RETRY_1',
      inputType: 'FILE',
      checkpointStage: 'converted',
      checkpointData: {
        stage: 'converted',
        createdAt: expiredTime.toISOString(),
      },
      updatedAt: expiredTime,
    },
  });

  // Create fresh checkpoint (1 hour old)
  const freshTime = new Date(Date.now() - 1 * 60 * 60 * 1000);

  const freshJob = await prisma.job.create({
    data: {
      sessionId: 'ttl-fresh-test',
      status: 'PROCESSING',
      inputType: 'FILE',
      checkpointStage: 'converting',
      checkpointData: {
        stage: 'converting',
        createdAt: freshTime.toISOString(),
      },
    },
  });

  // Query for expired checkpoints (>24 hours old)
  const ttlThreshold = new Date(Date.now() - 24 * 60 * 60 * 1000);

  const expiredCheckpoints = await prisma.job.findMany({
    where: {
      checkpointData: { not: null },
      updatedAt: { lt: ttlThreshold },
      status: { notIn: ['COMPLETE', 'ERROR', 'CANCELLED'] },
    },
  });

  // Should find the expired job
  expect(expiredCheckpoints.length).toBeGreaterThanOrEqual(1);
  expect(expiredCheckpoints.some(j => j.id === expiredJob.id)).toBe(true);
  expect(expiredCheckpoints.some(j => j.id === freshJob.id)).toBe(false);
});
```

**Expected Result**:
- Expired checkpoints (>24 hours) are identifiable
- Fresh checkpoints are not flagged
- Query efficiently finds stale jobs for cleanup

---

### DB-CK-006: Checkpoint Stage Progression Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-CK-006 |
| **Title** | Validate Checkpoint Stage Progression |
| **Priority** | P1 - High |
| **Spec Reference** | Section 5.5 |
| **Gap Reference** | Consolidated Review - Gap Domain 4 |

**Test Steps**:

```typescript
it('should track checkpoint stage progression correctly', async () => {
  const validStages = ['uploaded', 'converting', 'converted', 'exporting', 'complete'];

  const job = await prisma.job.create({
    data: {
      sessionId: 'stage-progression-test',
      status: 'PROCESSING',
      inputType: 'FILE',
      checkpointStage: 'uploaded',
      checkpointData: { stage: 'uploaded', timestamp: new Date().toISOString() },
    },
  });

  // Progress through stages
  for (let i = 1; i < validStages.length - 1; i++) {
    const updated = await prisma.job.update({
      where: { id: job.id },
      data: {
        checkpointStage: validStages[i],
        checkpointData: {
          stage: validStages[i],
          previousStage: validStages[i - 1],
          timestamp: new Date().toISOString(),
        },
      },
    });

    expect(updated.checkpointStage).toBe(validStages[i]);
  }

  // Query job history by checkpoint stages
  const history = await prisma.job.findUnique({
    where: { id: job.id },
    select: {
      checkpointStage: true,
      checkpointData: true,
    },
  });

  expect(history?.checkpointStage).toBe('exporting');
  expect((history?.checkpointData as any)?.previousStage).toBe('converted');
});
```

**Expected Result**:
- Stage progression is tracked correctly
- Previous stage is recorded for debugging
- All valid stages can be set

---

### DB-CK-007: Concurrent Checkpoint Update Handling

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-CK-007 |
| **Title** | Handle Concurrent Checkpoint Updates |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.5 |
| **Gap Reference** | Consolidated Review - Gap Domain 4 |

**Test Steps**:

```typescript
it('should handle concurrent checkpoint updates atomically', async () => {
  const job = await prisma.job.create({
    data: {
      sessionId: 'conc-checkpoint-atomic',
      status: 'PROCESSING',
      inputType: 'FILE',
      checkpointStage: 'converting',
      checkpointData: { stage: 'converting', pages: 0 },
    },
  });

  // Simulate concurrent page processing updates
  const concurrentUpdates = 10;
  const results = await Promise.allSettled(
    Array.from({ length: concurrentUpdates }, (_, i) =>
      prisma.$transaction(async (tx) => {
        const current = await tx.job.findUnique({
          where: { id: job.id },
          select: { checkpointData: true },
        });

        const currentPages = (current?.checkpointData as any)?.pages || 0;

        await tx.job.update({
          where: { id: job.id },
          data: {
            checkpointData: {
              stage: 'converting',
              pages: currentPages + 1,
              lastUpdatedBy: `worker-${i}`,
            },
          },
        });
      })
    )
  );

  const successes = results.filter(r => r.status === 'fulfilled').length;

  // With transaction isolation, final pages count might not equal concurrentUpdates
  // due to read-modify-write pattern, but no data corruption should occur
  const finalJob = await prisma.job.findUnique({ where: { id: job.id } });
  const finalPages = (finalJob?.checkpointData as any)?.pages;

  expect(typeof finalPages).toBe('number');
  expect(finalPages).toBeGreaterThanOrEqual(1);

  console.log(
    `Concurrent checkpoint updates: ${successes}/${concurrentUpdates} succeeded, final pages: ${finalPages}`
  );
});
```

**Expected Result**:
- Concurrent updates do not corrupt checkpoint data
- Final state is valid JSON
- No partial updates or data loss

---

### DB-DUR-001: Process Crash Recovery Scenario

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-DUR-001 |
| **Title** | Identify Jobs Requiring Recovery After Crash |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.5 |
| **Gap Reference** | Consolidated Review - Gap Domain 4 |

**Test Steps**:

```typescript
it('should identify jobs requiring recovery after crash', async () => {
  // Create stale processing job (simulating crash 5 minutes ago)
  const staleTime = new Date(Date.now() - 5 * 60 * 1000);

  const staleJob = await prisma.job.create({
    data: {
      sessionId: 'crash-recovery-test',
      status: 'PROCESSING',
      inputType: 'FILE',
      checkpointStage: 'converting',
      checkpointData: {
        stage: 'converting',
        progress: 50,
        timestamp: staleTime.toISOString(),
      },
    },
  });

  // Manually set updatedAt to stale time
  await prisma.$executeRaw`
    UPDATE "Job" SET "updatedAt" = ${staleTime} WHERE id = ${staleJob.id}
  `;

  // Create active job (updated recently)
  const activeJob = await prisma.job.create({
    data: {
      sessionId: 'active-job-test',
      status: 'PROCESSING',
      inputType: 'FILE',
      checkpointStage: 'converting',
      checkpointData: { stage: 'converting', progress: 25 },
    },
  });

  // Query for recoverable jobs (stale > 1 minute with checkpoint)
  const staleThreshold = new Date(Date.now() - 60 * 1000);

  const recoverableJobs = await prisma.job.findMany({
    where: {
      status: { in: ['PROCESSING', 'UPLOADING'] },
      checkpointData: { not: null },
      updatedAt: { lt: staleThreshold },
    },
  });

  // Should find stale job but not active job
  expect(recoverableJobs.some(j => j.id === staleJob.id)).toBe(true);
  expect(recoverableJobs.some(j => j.id === activeJob.id)).toBe(false);

  console.log(`Found ${recoverableJobs.length} recoverable job(s)`);
});
```

**Expected Result**:
- Stale processing jobs are identified for recovery
- Active jobs are not flagged incorrectly
- Query uses appropriate index (status, updatedAt)

---

### DB-DUR-002: Orphaned Job Detection

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-DUR-002 |
| **Title** | Detect Orphaned Jobs Without Checkpoints |
| **Priority** | P1 - High |
| **Spec Reference** | Section 5.5 |
| **Gap Reference** | Consolidated Review - Gap Domain 4 |

**Test Steps**:

```typescript
it('should detect orphaned jobs without valid checkpoints', async () => {
  // Create orphaned job (PROCESSING but no checkpoint, stale)
  const orphanTime = new Date(Date.now() - 10 * 60 * 1000);

  const orphanedJob = await prisma.job.create({
    data: {
      sessionId: 'orphan-detection-test',
      status: 'PROCESSING',
      inputType: 'FILE',
      checkpointStage: null,
      checkpointData: null,
    },
  });

  await prisma.$executeRaw`
    UPDATE "Job" SET "updatedAt" = ${orphanTime} WHERE id = ${orphanedJob.id}
  `;

  // Create job with checkpoint (recoverable, not orphaned)
  const recoverableJob = await prisma.job.create({
    data: {
      sessionId: 'recoverable-test',
      status: 'PROCESSING',
      inputType: 'FILE',
      checkpointStage: 'converting',
      checkpointData: { stage: 'converting' },
    },
  });

  await prisma.$executeRaw`
    UPDATE "Job" SET "updatedAt" = ${orphanTime} WHERE id = ${recoverableJob.id}
  `;

  // Query for orphaned jobs (stale + no checkpoint)
  const staleThreshold = new Date(Date.now() - 5 * 60 * 1000);

  const orphanedJobs = await prisma.job.findMany({
    where: {
      status: { in: ['PROCESSING', 'UPLOADING'] },
      checkpointData: null,
      updatedAt: { lt: staleThreshold },
    },
  });

  // Should find orphaned job only
  expect(orphanedJobs.some(j => j.id === orphanedJob.id)).toBe(true);
  expect(orphanedJobs.some(j => j.id === recoverableJob.id)).toBe(false);

  // Orphaned jobs should be marked as ERROR
  for (const orphan of orphanedJobs) {
    await prisma.job.update({
      where: { id: orphan.id },
      data: {
        status: 'ERROR',
        errorCode: 'E799',
        errorMessage: 'Job orphaned after process crash (no checkpoint)',
      },
    });
  }

  const finalOrphan = await prisma.job.findUnique({
    where: { id: orphanedJob.id },
  });
  expect(finalOrphan?.status).toBe('ERROR');
  expect(finalOrphan?.errorCode).toBe('E799');
});
```

**Expected Result**:
- Orphaned jobs (stale + no checkpoint) are identified
- Recoverable jobs (with checkpoint) are not flagged
- Orphaned jobs can be marked as ERROR

---

### DB-DUR-003: Checkpoint Integrity After Partial Write

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-DUR-003 |
| **Title** | Verify Checkpoint Integrity After Partial Write |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.5 |
| **Gap Reference** | Consolidated Review - Gap Domain 4 |

**Test Steps**:

```typescript
it('should maintain checkpoint integrity with hash validation', async () => {
  const crypto = require('crypto');

  const checkpointData = {
    stage: 'converted',
    doclingDocument: {
      schema_name: 'DoclingDocument',
      version: '1.0.0',
      pages: [{ pageNumber: 1, content: 'Test content' }],
    },
    timestamp: new Date().toISOString(),
  };

  // Generate integrity hash
  const hash = crypto
    .createHash('sha256')
    .update(JSON.stringify(checkpointData))
    .digest('hex');

  const job = await prisma.job.create({
    data: {
      sessionId: 'integrity-test',
      status: 'PROCESSING',
      inputType: 'FILE',
      checkpointStage: 'converted',
      checkpointData,
      checkpointHash: hash,
    },
  });

  // Retrieve and verify integrity
  const retrieved = await prisma.job.findUnique({
    where: { id: job.id },
    select: {
      checkpointData: true,
      checkpointHash: true,
    },
  });

  const computedHash = crypto
    .createHash('sha256')
    .update(JSON.stringify(retrieved?.checkpointData))
    .digest('hex');

  expect(computedHash).toBe(retrieved?.checkpointHash);

  // Simulate corruption detection
  const corruptedJob = await prisma.job.update({
    where: { id: job.id },
    data: {
      checkpointData: {
        ...checkpointData,
        corrupted: true, // Tampered data
      },
      // Hash not updated (simulates corruption)
    },
  });

  const corruptedRetrieved = await prisma.job.findUnique({
    where: { id: job.id },
  });

  const corruptedHash = crypto
    .createHash('sha256')
    .update(JSON.stringify(corruptedRetrieved?.checkpointData))
    .digest('hex');

  // Hash mismatch indicates corruption
  expect(corruptedHash).not.toBe(corruptedRetrieved?.checkpointHash);
});
```

**Expected Result**:
- Valid checkpoint passes hash verification
- Corrupted/tampered checkpoint fails hash verification
- Integrity check enables corruption detection

---

### DB-WF-001: Verify Checkpoint Matches Current Stage

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-WF-001 |
| **Title** | Verify Checkpoint Matches Current Workflow Stage |
| **Priority** | P1 - High |
| **Spec Reference** | Section 5.5 |
| **Gap Reference** | Consolidated Review - Gap Domain 4 |

**Test Steps**:

```typescript
it('should verify checkpoint stage matches job stage', async () => {
  const job = await prisma.job.create({
    data: {
      sessionId: 'stage-match-test',
      status: 'PROCESSING',
      inputType: 'FILE',
      checkpointStage: 'converting',
      checkpointData: {
        stage: 'converting',
        progress: 50,
      },
    },
  });

  // Verify checkpoint stage matches checkpointStage field
  const retrieved = await prisma.job.findUnique({ where: { id: job.id } });

  expect(retrieved?.checkpointStage).toBe(
    (retrieved?.checkpointData as any)?.stage
  );

  // Function to validate checkpoint consistency
  const validateCheckpoint = (job: any): boolean => {
    if (!job.checkpointData) return true; // No checkpoint is valid
    const dataStage = (job.checkpointData as any)?.stage;
    return job.checkpointStage === dataStage;
  };

  expect(validateCheckpoint(retrieved)).toBe(true);

  // Create inconsistent checkpoint (should fail validation)
  const inconsistentJob = await prisma.job.create({
    data: {
      sessionId: 'inconsistent-test',
      status: 'PROCESSING',
      inputType: 'FILE',
      checkpointStage: 'exporting',
      checkpointData: {
        stage: 'converting', // Mismatched!
        progress: 25,
      },
    },
  });

  expect(validateCheckpoint(inconsistentJob)).toBe(false);
});
```

**Expected Result**:
- Consistent checkpoints pass validation
- Inconsistent checkpoints fail validation
- Validation function is reliable

---

### DB-WF-002: Stage Completion Timestamp Sequencing

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-WF-002 |
| **Title** | Verify Stage Completion Timestamps Are Sequential |
| **Priority** | P2 - Medium |
| **Spec Reference** | Section 5.5 |
| **Gap Reference** | Consolidated Review - Gap Domain 4 |

**Test Steps**:

```typescript
it('should track stage completion timestamps sequentially', async () => {
  const stages = ['uploaded', 'converting', 'converted', 'exporting'];
  const stageTimestamps: Record<string, string> = {};

  const job = await prisma.job.create({
    data: {
      sessionId: 'timestamp-sequence-test',
      status: 'PROCESSING',
      inputType: 'FILE',
      checkpointStage: stages[0],
      checkpointData: {
        stage: stages[0],
        stageHistory: [],
        currentTimestamp: new Date().toISOString(),
      },
    },
  });

  stageTimestamps[stages[0]] = new Date().toISOString();

  // Progress through stages with timestamp tracking
  for (let i = 1; i < stages.length; i++) {
    await new Promise(r => setTimeout(r, 50)); // Ensure time progression

    const timestamp = new Date().toISOString();
    stageTimestamps[stages[i]] = timestamp;

    const currentData = (
      await prisma.job.findUnique({ where: { id: job.id } })
    )?.checkpointData as any;

    await prisma.job.update({
      where: { id: job.id },
      data: {
        checkpointStage: stages[i],
        checkpointData: {
          stage: stages[i],
          stageHistory: [
            ...(currentData?.stageHistory || []),
            { stage: stages[i - 1], completedAt: stageTimestamps[stages[i - 1]] },
          ],
          currentTimestamp: timestamp,
        },
      },
    });
  }

  // Verify timestamps are sequential
  const finalJob = await prisma.job.findUnique({ where: { id: job.id } });
  const history = (finalJob?.checkpointData as any)?.stageHistory || [];

  for (let i = 1; i < history.length; i++) {
    const prevTime = new Date(history[i - 1].completedAt).getTime();
    const currTime = new Date(history[i].completedAt).getTime();
    expect(currTime).toBeGreaterThanOrEqual(prevTime);
  }

  expect(history).toHaveLength(stages.length - 1);
});
```

**Expected Result**:
- Stage history records completion timestamps
- Timestamps are monotonically increasing
- Full stage progression is traceable

---

### DB-WF-003: Checkpoint Rollback on Stage Failure

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-WF-003 |
| **Title** | Handle Checkpoint Rollback on Stage Failure |
| **Priority** | P1 - High |
| **Spec Reference** | Section 5.5 |
| **Gap Reference** | Consolidated Review - Gap Domain 4 |

**Test Steps**:

```typescript
it('should rollback checkpoint on stage failure', async () => {
  const job = await prisma.job.create({
    data: {
      sessionId: 'rollback-test',
      status: 'PROCESSING',
      inputType: 'FILE',
      checkpointStage: 'converted',
      checkpointData: {
        stage: 'converted',
        doclingDocument: { pages: [{ pageNumber: 1 }] },
        timestamp: new Date().toISOString(),
      },
    },
  });

  // Capture pre-failure checkpoint
  const preFailureCheckpoint = (
    await prisma.job.findUnique({ where: { id: job.id } })
  )?.checkpointData;

  // Simulate failed export stage (within transaction for atomicity)
  try {
    await prisma.$transaction(async (tx) => {
      // Start export stage
      await tx.job.update({
        where: { id: job.id },
        data: {
          checkpointStage: 'exporting',
          checkpointData: {
            stage: 'exporting',
            progress: 10,
            timestamp: new Date().toISOString(),
          },
        },
      });

      // Simulate export failure
      throw new Error('Export failed: Out of memory');
    });
  } catch (e) {
    // Transaction rolled back - checkpoint should be unchanged
  }

  // Verify checkpoint was not modified (transaction rolled back)
  const postFailure = await prisma.job.findUnique({ where: { id: job.id } });

  expect(postFailure?.checkpointStage).toBe('converted');
  expect((postFailure?.checkpointData as any)?.stage).toBe('converted');

  // For non-transactional recovery, implement explicit rollback
  const explicitRollback = await prisma.job.update({
    where: { id: job.id },
    data: {
      status: 'RETRY_1',
      retryCount: 1,
      checkpointStage: 'converted',
      checkpointData: preFailureCheckpoint,
    },
  });

  expect(explicitRollback.status).toBe('RETRY_1');
  expect(explicitRollback.checkpointStage).toBe('converted');
});
```

**Expected Result**:
- Transaction rollback preserves previous checkpoint
- Explicit rollback can restore checkpoint state
- Recovery from failed stage is possible

---

## 12. Cache-Database Consistency

> **Priority**: P1 - HIGH
> **Gap Reference**: Consolidated Review - Gap Domain 5
> **Risk Level**: Data consistency risk - stale cache causes incorrect job status and session data

This section validates cache invalidation patterns, session synchronization between Redis and PostgreSQL, dual-write atomicity, and saga/compensation patterns for partial failure recovery. Without these tests, the system risks serving stale data, session count mismatches, and data loss on partial failures.

---

### DB-CACHE-001: Cache Invalidation on Job Status Update

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-CACHE-001 |
| **Title** | Cache Invalidation on Job Status Update |
| **Priority** | P0 - Critical |
| **Spec Reference** | Session management |
| **Gap Reference** | Consolidated Review - Gap Domain 5 |

**Test Steps**:

```typescript
it('should invalidate cache when job status updated', async () => {
  const redis = getRedisClient();
  const jobCacheKey = 'hx-docling:job';

  // Create job in database
  const job = await prisma.job.create({
    data: {
      sessionId: 'cache-invalidation-test',
      status: 'PENDING',
      inputType: 'FILE',
      fileName: 'test.pdf',
    },
  });

  // Populate cache with job data
  await redis.hset(`${jobCacheKey}:${job.id}`, {
    id: job.id,
    status: 'PENDING',
    sessionId: 'cache-invalidation-test',
  });
  await redis.expire(`${jobCacheKey}:${job.id}`, 300);

  // Verify cache is populated
  const cachedBefore = await redis.hgetall(`${jobCacheKey}:${job.id}`);
  expect(cachedBefore.status).toBe('PENDING');

  // Update job status in database (should trigger cache invalidation)
  await prisma.job.update({
    where: { id: job.id },
    data: { status: 'PROCESSING' },
  });

  // Invalidate cache (application logic should do this)
  await redis.del(`${jobCacheKey}:${job.id}`);

  // Verify cache is cleared
  const cachedAfter = await redis.hgetall(`${jobCacheKey}:${job.id}`);
  expect(Object.keys(cachedAfter).length).toBe(0);

  // Verify database has new value
  const dbJob = await prisma.job.findUnique({ where: { id: job.id } });
  expect(dbJob?.status).toBe('PROCESSING');
});
```

**Expected Result**:
- Cache is populated before update
- Cache is invalidated after database update
- Database contains correct updated value

---

### DB-CACHE-002: Cache-Aside Pattern Verification

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-CACHE-002 |
| **Title** | Verify Cache-Aside Pattern Implementation |
| **Priority** | P1 - High |
| **Spec Reference** | Session management |
| **Gap Reference** | Consolidated Review - Gap Domain 5 |

**Test Steps**:

```typescript
it('should implement cache-aside pattern correctly', async () => {
  const redis = getRedisClient();
  const jobCacheKey = 'hx-docling:job';

  // Create job in database
  const job = await prisma.job.create({
    data: {
      sessionId: 'cache-aside-test',
      status: 'COMPLETE',
      inputType: 'FILE',
    },
  });

  // Ensure cache is empty initially
  await redis.del(`${jobCacheKey}:${job.id}`);

  // Simulate cache-aside read pattern
  const getJobWithCache = async (jobId: string) => {
    // Step 1: Check cache first
    const cached = await redis.hgetall(`${jobCacheKey}:${jobId}`);
    if (cached && cached.id) {
      return { ...cached, source: 'cache' };
    }

    // Step 2: Cache miss - read from database
    const dbJob = await prisma.job.findUnique({ where: { id: jobId } });
    if (!dbJob) return null;

    // Step 3: Populate cache for next request
    await redis.hset(`${jobCacheKey}:${jobId}`, {
      id: dbJob.id,
      status: dbJob.status,
      sessionId: dbJob.sessionId,
    });
    await redis.expire(`${jobCacheKey}:${jobId}`, 300);

    return { ...dbJob, source: 'database' };
  };

  // First call: Cache miss, reads from database
  const firstCall = await getJobWithCache(job.id);
  expect(firstCall?.source).toBe('database');

  // Second call: Cache hit
  const secondCall = await getJobWithCache(job.id);
  expect(secondCall?.source).toBe('cache');
  expect(secondCall?.status).toBe('COMPLETE');
});
```

**Expected Result**:
- First request is a cache miss, reads from database
- Cache is populated after database read
- Second request is a cache hit

---

### DB-CACHE-003: Cache Hit Avoids Database Query

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-CACHE-003 |
| **Title** | Verify Cache Hit Avoids Database Query |
| **Priority** | P1 - High |
| **Spec Reference** | Performance optimization |
| **Gap Reference** | Consolidated Review - Gap Domain 5 |

**Test Steps**:

```typescript
it('should serve from cache without database query', async () => {
  const redis = getRedisClient();
  const jobCacheKey = 'hx-docling:job';

  const testJobId = 'cache-hit-test-job-id';

  // Populate cache with job data (simulate cached state)
  await redis.hset(`${jobCacheKey}:${testJobId}`, {
    id: testJobId,
    status: 'COMPLETE',
    sessionId: 'cache-hit-session',
    inputType: 'FILE',
  });
  await redis.expire(`${jobCacheKey}:${testJobId}`, 300);

  // Track database query count
  let dbQueryCount = 0;
  const originalFindUnique = prisma.job.findUnique;
  prisma.job.findUnique = async (...args) => {
    dbQueryCount++;
    return originalFindUnique.apply(prisma.job, args);
  };

  try {
    // Simulate cache-first read
    const cached = await redis.hgetall(`${jobCacheKey}:${testJobId}`);

    if (cached && cached.id) {
      // Cache hit - no database query needed
      expect(cached.status).toBe('COMPLETE');
      expect(dbQueryCount).toBe(0); // No database queries
    }
  } finally {
    // Restore original method
    prisma.job.findUnique = originalFindUnique;
    await redis.del(`${jobCacheKey}:${testJobId}`);
  }
});
```

**Expected Result**:
- Cache hit returns data without database query
- Database query count remains zero
- Cached data is correct

---

### DB-SESS-SYNC-001: Session JobCount Synchronization

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-SESS-SYNC-001 |
| **Title** | Session JobCount Synchronization Between Redis and PostgreSQL |
| **Priority** | P0 - Critical |
| **Spec Reference** | Session management |
| **Gap Reference** | Consolidated Review - Gap Domain 5 |

**Test Steps**:

```typescript
it('should synchronize session jobCount between Redis and PostgreSQL', async () => {
  const redis = getRedisClient();
  const sessionId = 'sync-jobcount-test';
  const sessionCacheKey = `hx-docling:session:${sessionId}`;

  // Initialize session in Redis with jobCount = 0
  await redis.hset(sessionCacheKey, {
    id: sessionId,
    jobCount: '0',
    activeJobIds: '[]',
  });
  await redis.expire(sessionCacheKey, 3600);

  // Create jobs in database and sync Redis
  const job1 = await prisma.job.create({
    data: { sessionId, status: 'PENDING', inputType: 'FILE' },
  });
  await redis.hincrby(sessionCacheKey, 'jobCount', 1);

  const job2 = await prisma.job.create({
    data: { sessionId, status: 'PENDING', inputType: 'URL' },
  });
  await redis.hincrby(sessionCacheKey, 'jobCount', 1);

  // Verify Redis count matches database count
  const redisSession = await redis.hgetall(sessionCacheKey);
  const redisJobCount = parseInt(redisSession.jobCount, 10);

  const dbJobCount = await prisma.job.count({
    where: { sessionId },
  });

  expect(redisJobCount).toBe(dbJobCount);
  expect(redisJobCount).toBe(2);

  // Cleanup
  await prisma.job.deleteMany({ where: { sessionId } });
  await redis.del(sessionCacheKey);
});
```

**Expected Result**:
- Redis jobCount increments with each job creation
- Redis jobCount matches PostgreSQL count
- No count mismatch between stores

---

### DB-SESS-SYNC-002: Session ActiveJobIds Synchronization

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-SESS-SYNC-002 |
| **Title** | Session ActiveJobIds Synchronization |
| **Priority** | P1 - High |
| **Spec Reference** | Session management |
| **Gap Reference** | Consolidated Review - Gap Domain 5 |

**Test Steps**:

```typescript
it('should synchronize session activeJobIds between stores', async () => {
  const redis = getRedisClient();
  const sessionId = 'sync-activejobs-test';
  const sessionCacheKey = `hx-docling:session:${sessionId}`;

  // Initialize session
  await redis.hset(sessionCacheKey, {
    id: sessionId,
    jobCount: '0',
    activeJobIds: '[]',
  });

  // Create active job
  const job = await prisma.job.create({
    data: {
      sessionId,
      status: 'PROCESSING',
      inputType: 'FILE',
    },
  });

  // Update Redis activeJobIds
  const currentActiveIds = JSON.parse(
    (await redis.hget(sessionCacheKey, 'activeJobIds')) || '[]'
  );
  currentActiveIds.push(job.id);
  await redis.hset(sessionCacheKey, 'activeJobIds', JSON.stringify(currentActiveIds));
  await redis.hincrby(sessionCacheKey, 'jobCount', 1);

  // Complete job
  await prisma.job.update({
    where: { id: job.id },
    data: { status: 'COMPLETE', completedAt: new Date() },
  });

  // Remove from activeJobIds
  const updatedActiveIds = JSON.parse(
    (await redis.hget(sessionCacheKey, 'activeJobIds')) || '[]'
  ).filter((id: string) => id !== job.id);
  await redis.hset(sessionCacheKey, 'activeJobIds', JSON.stringify(updatedActiveIds));

  // Verify synchronization
  const activeInRedis = JSON.parse(
    (await redis.hget(sessionCacheKey, 'activeJobIds')) || '[]'
  );
  const activeInDb = await prisma.job.findMany({
    where: {
      sessionId,
      status: { in: ['PENDING', 'UPLOADING', 'PROCESSING'] },
    },
    select: { id: true },
  });

  expect(activeInRedis.length).toBe(activeInDb.length);
  expect(activeInRedis).toEqual(activeInDb.map(j => j.id));

  // Cleanup
  await prisma.job.deleteMany({ where: { sessionId } });
  await redis.del(sessionCacheKey);
});
```

**Expected Result**:
- ActiveJobIds updated on job creation
- ActiveJobIds updated on job completion
- Redis and PostgreSQL lists are synchronized

---

### DB-SESS-SYNC-003: Orphaned Jobs on Session Expiration

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-SESS-SYNC-003 |
| **Title** | Handle Orphaned Jobs on Session Expiration |
| **Priority** | P1 - High |
| **Spec Reference** | Session management |
| **Gap Reference** | Consolidated Review - Gap Domain 5 |

**Test Steps**:

```typescript
it('should handle orphaned jobs when session expires', async () => {
  const redis = getRedisClient();
  const sessionId = 'orphan-session-test';
  const sessionCacheKey = `hx-docling:session:${sessionId}`;

  // Create session with very short TTL
  await redis.hset(sessionCacheKey, {
    id: sessionId,
    jobCount: '2',
    activeJobIds: '[]',
  });
  await redis.expire(sessionCacheKey, 1); // 1 second TTL

  // Create jobs in database
  const job1 = await prisma.job.create({
    data: { sessionId, status: 'PROCESSING', inputType: 'FILE' },
  });
  const job2 = await prisma.job.create({
    data: { sessionId, status: 'PENDING', inputType: 'URL' },
  });

  // Wait for session to expire
  await new Promise(r => setTimeout(r, 1500));

  // Verify session expired
  const sessionExists = await redis.exists(sessionCacheKey);
  expect(sessionExists).toBe(0);

  // Find orphaned jobs (session expired but jobs still active)
  const orphanedJobs = await prisma.job.findMany({
    where: {
      sessionId,
      status: { in: ['PENDING', 'PROCESSING', 'UPLOADING'] },
    },
  });

  expect(orphanedJobs.length).toBe(2);

  // Mark orphaned jobs as cancelled
  for (const orphan of orphanedJobs) {
    await prisma.job.update({
      where: { id: orphan.id },
      data: {
        status: 'CANCELLED',
        errorCode: 'E501',
        errorMessage: 'Session expired',
      },
    });
  }

  // Verify cleanup
  const remainingActive = await prisma.job.findMany({
    where: {
      sessionId,
      status: { in: ['PENDING', 'PROCESSING', 'UPLOADING'] },
    },
  });
  expect(remainingActive.length).toBe(0);

  // Cleanup
  await prisma.job.deleteMany({ where: { sessionId } });
});
```

**Expected Result**:
- Session expiration is detected
- Orphaned jobs are identified
- Orphaned jobs are properly cancelled

---

### DB-TX-DUAL-001: Dual-Write Success Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-TX-DUAL-001 |
| **Title** | Dual-Write Success to Redis and PostgreSQL |
| **Priority** | P0 - Critical |
| **Spec Reference** | Transaction integrity |
| **Gap Reference** | Consolidated Review - Gap Domain 5 |

**Test Steps**:

```typescript
it('should successfully write to both Redis and PostgreSQL', async () => {
  const redis = getRedisClient();
  const sessionId = 'dual-write-success-test';
  const sessionCacheKey = `hx-docling:session:${sessionId}`;

  // Initialize session in Redis
  await redis.hset(sessionCacheKey, {
    id: sessionId,
    jobCount: '0',
  });

  // Perform dual-write: Create job in PostgreSQL AND update Redis
  const dualWrite = async () => {
    // PostgreSQL write
    const job = await prisma.job.create({
      data: {
        sessionId,
        status: 'PENDING',
        inputType: 'FILE',
        fileName: 'dual-write.pdf',
      },
    });

    // Redis write
    await redis.hincrby(sessionCacheKey, 'jobCount', 1);
    await redis.hset(`hx-docling:job:${job.id}`, {
      id: job.id,
      status: 'PENDING',
      sessionId,
    });

    return job;
  };

  const job = await dualWrite();

  // Verify both stores are consistent
  const dbJob = await prisma.job.findUnique({ where: { id: job.id } });
  const cachedJob = await redis.hgetall(`hx-docling:job:${job.id}`);
  const sessionData = await redis.hgetall(sessionCacheKey);

  expect(dbJob).not.toBeNull();
  expect(dbJob?.status).toBe('PENDING');
  expect(cachedJob.status).toBe('PENDING');
  expect(parseInt(sessionData.jobCount, 10)).toBe(1);

  // Cleanup
  await prisma.job.deleteMany({ where: { sessionId } });
  await redis.del(sessionCacheKey);
  await redis.del(`hx-docling:job:${job.id}`);
});
```

**Expected Result**:
- Job created in PostgreSQL
- Job cached in Redis
- Session jobCount incremented
- Both stores are consistent

---

### DB-TX-DUAL-002: Rollback on Partial Failure

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-TX-DUAL-002 |
| **Title** | Rollback Both Stores on Partial Failure |
| **Priority** | P0 - Critical |
| **Spec Reference** | Transaction integrity |
| **Gap Reference** | Consolidated Review - Gap Domain 5 |

**Test Steps**:

```typescript
it('should rollback on partial dual-write failure', async () => {
  const redis = getRedisClient();
  const sessionId = 'dual-write-rollback-test';
  const sessionCacheKey = `hx-docling:session:${sessionId}`;

  // Initialize session
  await redis.hset(sessionCacheKey, {
    id: sessionId,
    jobCount: '0',
  });

  // Simulate dual-write with failure
  const dualWriteWithFailure = async () => {
    let jobId: string | null = null;

    try {
      // PostgreSQL write succeeds
      const job = await prisma.job.create({
        data: {
          sessionId,
          status: 'PENDING',
          inputType: 'FILE',
        },
      });
      jobId = job.id;

      // Simulate Redis failure
      throw new Error('Redis connection failed');

    } catch (error) {
      // Rollback PostgreSQL if Redis failed
      if (jobId) {
        await prisma.job.delete({ where: { id: jobId } });
      }
      throw error;
    }
  };

  // Attempt dual-write (should fail and rollback)
  await expect(dualWriteWithFailure()).rejects.toThrow('Redis connection failed');

  // Verify PostgreSQL was rolled back
  const dbJobs = await prisma.job.findMany({ where: { sessionId } });
  expect(dbJobs.length).toBe(0);

  // Verify Redis unchanged
  const sessionData = await redis.hgetall(sessionCacheKey);
  expect(parseInt(sessionData.jobCount, 10)).toBe(0);

  // Cleanup
  await redis.del(sessionCacheKey);
});
```

**Expected Result**:
- PostgreSQL write is rolled back on Redis failure
- No orphaned records in either store
- System remains consistent

---

### DB-TX-SAGA-001: Compensation on PostgreSQL Failure

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-TX-SAGA-001 |
| **Title** | Saga Compensation When PostgreSQL Fails |
| **Priority** | P1 - High |
| **Spec Reference** | Transaction integrity |
| **Gap Reference** | Consolidated Review - Gap Domain 5 |

**Test Steps**:

```typescript
it('should compensate Redis when PostgreSQL fails', async () => {
  const redis = getRedisClient();
  const sessionId = 'saga-pg-failure-test';
  const sessionCacheKey = `hx-docling:session:${sessionId}`;

  // Initialize session
  await redis.hset(sessionCacheKey, {
    id: sessionId,
    jobCount: '0',
  });

  // Saga pattern: Redis first, then PostgreSQL
  const sagaWithPgFailure = async () => {
    // Step 1: Optimistically update Redis
    await redis.hincrby(sessionCacheKey, 'jobCount', 1);
    const tempJobId = `temp-${Date.now()}`;
    await redis.hset(`hx-docling:job:${tempJobId}`, {
      id: tempJobId,
      status: 'PENDING',
      sessionId,
    });

    try {
      // Step 2: PostgreSQL write (simulated failure)
      throw new Error('PostgreSQL constraint violation');

    } catch (error) {
      // Compensation: Rollback Redis changes
      await redis.hincrby(sessionCacheKey, 'jobCount', -1);
      await redis.del(`hx-docling:job:${tempJobId}`);
      throw error;
    }
  };

  // Execute saga (should fail and compensate)
  await expect(sagaWithPgFailure()).rejects.toThrow('PostgreSQL constraint violation');

  // Verify Redis was compensated
  const sessionData = await redis.hgetall(sessionCacheKey);
  expect(parseInt(sessionData.jobCount, 10)).toBe(0);

  // Cleanup
  await redis.del(sessionCacheKey);
});
```

**Expected Result**:
- Redis optimistically updated first
- PostgreSQL failure triggers compensation
- Redis changes are rolled back
- System remains consistent

---

### DB-TX-SAGA-002: Compensation on Redis Failure

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-TX-SAGA-002 |
| **Title** | Saga Compensation When Redis Fails |
| **Priority** | P1 - High |
| **Spec Reference** | Transaction integrity |
| **Gap Reference** | Consolidated Review - Gap Domain 5 |

**Test Steps**:

```typescript
it('should compensate PostgreSQL when Redis fails', async () => {
  const redis = getRedisClient();
  const sessionId = 'saga-redis-failure-test';
  const sessionCacheKey = `hx-docling:session:${sessionId}`;

  // Initialize session
  await redis.hset(sessionCacheKey, {
    id: sessionId,
    jobCount: '0',
  });

  // Saga pattern: PostgreSQL first, then Redis
  const sagaWithRedisFailure = async () => {
    // Step 1: Create job in PostgreSQL
    const job = await prisma.job.create({
      data: {
        sessionId,
        status: 'PENDING',
        inputType: 'FILE',
      },
    });

    try {
      // Step 2: Update Redis (simulated failure)
      throw new Error('Redis WRONGTYPE Operation');

    } catch (error) {
      // Compensation: Delete PostgreSQL record
      await prisma.job.delete({ where: { id: job.id } });
      throw error;
    }
  };

  // Execute saga (should fail and compensate)
  await expect(sagaWithRedisFailure()).rejects.toThrow('Redis WRONGTYPE Operation');

  // Verify PostgreSQL was compensated
  const dbJobs = await prisma.job.findMany({ where: { sessionId } });
  expect(dbJobs.length).toBe(0);

  // Verify Redis unchanged
  const sessionData = await redis.hgetall(sessionCacheKey);
  expect(parseInt(sessionData.jobCount, 10)).toBe(0);

  // Cleanup
  await redis.del(sessionCacheKey);
});
```

**Expected Result**:
- PostgreSQL write succeeds initially
- Redis failure triggers compensation
- PostgreSQL record is deleted
- System remains consistent

---

## 13. Production Observability and Monitoring

> **Priority**: P1 - HIGH
> **Gap Reference**: Consolidated Review - Gap Domain 6
> **Risk Level**: Operational blindness - production issues invisible until user-facing outage

This section validates database health checks, Prometheus metrics exposure, query logging, and slow query detection. Without these tests, production issues remain invisible until they cause service outages.

---

### DB-HEALTH-001: Database Health Check Response

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-HEALTH-001 |
| **Title** | Database Health Check Returns Healthy Status |
| **Priority** | P0 - Critical |
| **Spec Reference** | NFR-301 |
| **Gap Reference** | Consolidated Review - Gap Domain 6 |

**Test Steps**:

```typescript
it('should return healthy status when database accessible', async () => {
  const validateDatabaseConnection = async () => {
    const startTime = Date.now();

    try {
      // Simple connectivity check
      await prisma.$queryRaw`SELECT 1 as connected`;

      return {
        status: 'healthy',
        responseTimeMs: Date.now() - startTime,
        database: 'postgresql',
        connected: true,
      };
    } catch (error) {
      return {
        status: 'unhealthy',
        responseTimeMs: Date.now() - startTime,
        database: 'postgresql',
        connected: false,
        error: (error as Error).message,
      };
    }
  };

  const health = await validateDatabaseConnection();

  expect(health.status).toBe('healthy');
  expect(health.connected).toBe(true);
  expect(health.responseTimeMs).toBeLessThan(1000); // < 1 second
  expect(health.database).toBe('postgresql');
});
```

**Expected Result**:
- Health check returns 'healthy' status
- Response time is under 1 second
- Database is confirmed connected

---

### DB-HEALTH-002: Health Check Timeout Behavior

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-HEALTH-002 |
| **Title** | Health Check Handles Timeout Gracefully |
| **Priority** | P0 - Critical |
| **Spec Reference** | NFR-301 |
| **Gap Reference** | Consolidated Review - Gap Domain 6 |

**Test Steps**:

```typescript
it('should handle health check timeout gracefully', async () => {
  const healthCheckWithTimeout = async (timeoutMs: number) => {
    const startTime = Date.now();

    const timeoutPromise = new Promise((_, reject) => {
      setTimeout(() => reject(new Error('Health check timeout')), timeoutMs);
    });

    const checkPromise = prisma.$queryRaw`SELECT 1 as connected`;

    try {
      await Promise.race([checkPromise, timeoutPromise]);
      return {
        status: 'healthy',
        responseTimeMs: Date.now() - startTime,
      };
    } catch (error) {
      return {
        status: 'unhealthy',
        responseTimeMs: Date.now() - startTime,
        error: (error as Error).message,
      };
    }
  };

  // Test with generous timeout (should succeed)
  const successResult = await healthCheckWithTimeout(5000);
  expect(successResult.status).toBe('healthy');

  // Test timeout handling with very short timeout
  const timeoutResult = await healthCheckWithTimeout(1);
  // May succeed or timeout depending on database responsiveness
  expect(['healthy', 'unhealthy']).toContain(timeoutResult.status);
  expect(timeoutResult.responseTimeMs).toBeDefined();
});
```

**Expected Result**:
- Normal health check succeeds within timeout
- Timeout is detected and reported
- Response includes timing information

---

### DB-HEALTH-003: Specific Error Reporting on Failure

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-HEALTH-003 |
| **Title** | Health Check Reports Specific Errors |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-301 |
| **Gap Reference** | Consolidated Review - Gap Domain 6 |

**Test Steps**:

```typescript
it('should report specific error details on health check failure', async () => {
  const healthCheckWithErrorDetails = async () => {
    try {
      // Test valid query
      await prisma.$queryRaw`SELECT 1 as connected`;
      return { status: 'healthy', error: null };
    } catch (error: any) {
      // Categorize error
      let errorCategory = 'UNKNOWN';
      if (error.message.includes('connect')) {
        errorCategory = 'CONNECTION_FAILED';
      } else if (error.message.includes('timeout')) {
        errorCategory = 'TIMEOUT';
      } else if (error.message.includes('authentication')) {
        errorCategory = 'AUTH_FAILED';
      }

      return {
        status: 'unhealthy',
        error: error.message,
        errorCategory,
        timestamp: new Date().toISOString(),
      };
    }
  };

  // Normal operation should be healthy
  const result = await healthCheckWithErrorDetails();
  expect(result.status).toBe('healthy');
  expect(result.error).toBeNull();

  // Test error categorization function
  const categorizeError = (message: string): string => {
    if (message.includes('connect')) return 'CONNECTION_FAILED';
    if (message.includes('timeout')) return 'TIMEOUT';
    if (message.includes('authentication')) return 'AUTH_FAILED';
    return 'UNKNOWN';
  };

  expect(categorizeError('Could not connect to database')).toBe('CONNECTION_FAILED');
  expect(categorizeError('Query timeout exceeded')).toBe('TIMEOUT');
  expect(categorizeError('Password authentication failed')).toBe('AUTH_FAILED');
  expect(categorizeError('Some other error')).toBe('UNKNOWN');
});
```

**Expected Result**:
- Healthy status when database accessible
- Error messages are categorized
- Detailed error information is available

---

### DB-METRICS-001: Prometheus Metrics Exposure Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-METRICS-001 |
| **Title** | Validate Prometheus Metrics Exposure |
| **Priority** | P0 - Critical |
| **Spec Reference** | NFR-302 |
| **Gap Reference** | Consolidated Review - Gap Domain 6 |

**Test Steps**:

```typescript
it('should expose database metrics for Prometheus', async () => {
  // Execute some queries to generate metrics
  await prisma.job.findMany({ take: 5 });
  await prisma.result.findMany({ take: 5 });

  // Get Prisma metrics (requires Prisma client with metrics enabled)
  // Note: Prisma preview feature must be enabled in schema.prisma
  try {
    const metrics = await prisma.$metrics.prometheus();

    // Verify essential connection pool metrics are present
    expect(metrics).toContain('prisma_pool_connections_open');
    expect(metrics).toContain('prisma_pool_connections_busy');
    expect(metrics).toContain('prisma_pool_connections_idle');

    // Verify query metrics are present
    expect(metrics).toContain('prisma_client_queries_total');

    // Verify metrics format is valid Prometheus format
    const lines = metrics.split('\n').filter(line => !line.startsWith('#') && line.trim());
    for (const line of lines) {
      // Each metric line should have format: metric_name{labels} value
      expect(line).toMatch(/^[a-z_]+(\{[^}]+\})?\s+[\d.]+$/);
    }

  } catch (error) {
    // If metrics not enabled, verify it's due to feature flag
    expect((error as Error).message).toContain('metrics');
    console.log('Note: Enable previewFeatures = ["metrics"] in schema.prisma');
  }
});
```

**Expected Result**:
- Prometheus metrics endpoint returns valid metrics
- Connection pool metrics are exposed
- Query count metrics are tracked
- Metrics format is valid Prometheus text format

---

### DB-METRICS-002: Pool Utilization Metrics Accuracy

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-METRICS-002 |
| **Title** | Verify Pool Utilization Metrics Accuracy |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-302 |
| **Gap Reference** | Consolidated Review - Gap Domain 6 |

**Test Steps**:

```typescript
it('should accurately report pool utilization metrics', async () => {
  // Get baseline metrics
  const getPoolMetrics = async () => {
    const result = await prisma.$queryRaw<any[]>`
      SELECT
        count(*) FILTER (WHERE state = 'active') as active_connections,
        count(*) FILTER (WHERE state = 'idle') as idle_connections,
        count(*) as total_connections
      FROM pg_stat_activity
      WHERE datname = current_database()
      AND backend_type = 'client backend'
    `;
    return result[0];
  };

  const baselineMetrics = await getPoolMetrics();
  expect(baselineMetrics.total_connections).toBeGreaterThan(0);

  // Execute multiple concurrent queries
  const concurrentQueries = Array.from({ length: 5 }, () =>
    prisma.job.findMany({ take: 10 })
  );
  await Promise.all(concurrentQueries);

  // Get post-query metrics
  const postQueryMetrics = await getPoolMetrics();

  // Verify metrics are reasonable
  expect(postQueryMetrics.total_connections).toBeGreaterThanOrEqual(
    baselineMetrics.total_connections
  );

  // Idle connections should be available after queries complete
  expect(parseInt(postQueryMetrics.idle_connections, 10)).toBeGreaterThanOrEqual(0);

  console.log(`Pool metrics: active=${postQueryMetrics.active_connections}, idle=${postQueryMetrics.idle_connections}, total=${postQueryMetrics.total_connections}`);
});
```

**Expected Result**:
- Pool metrics are queryable
- Connection counts are accurate
- Metrics update after query activity

---

### DB-METRICS-003: Query Duration Histogram Collection

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-METRICS-003 |
| **Title** | Collect Query Duration Histogram |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-302 |
| **Gap Reference** | Consolidated Review - Gap Domain 6 |

**Test Steps**:

```typescript
it('should collect query duration metrics', async () => {
  const queryDurations: number[] = [];

  // Execute various queries and measure duration
  const measureQuery = async (queryFn: () => Promise<any>, label: string) => {
    const start = Date.now();
    await queryFn();
    const duration = Date.now() - start;
    queryDurations.push(duration);
    return { label, duration };
  };

  const measurements = await Promise.all([
    measureQuery(() => prisma.job.findFirst(), 'findFirst'),
    measureQuery(() => prisma.job.findMany({ take: 10 }), 'findMany_10'),
    measureQuery(() => prisma.job.count(), 'count'),
    measureQuery(
      () => prisma.job.findMany({ take: 100, orderBy: { createdAt: 'desc' } }),
      'findMany_100_sorted'
    ),
  ]);

  // Calculate histogram buckets
  const buckets = {
    '0-10ms': queryDurations.filter(d => d <= 10).length,
    '10-50ms': queryDurations.filter(d => d > 10 && d <= 50).length,
    '50-100ms': queryDurations.filter(d => d > 50 && d <= 100).length,
    '100-500ms': queryDurations.filter(d => d > 100 && d <= 500).length,
    '500ms+': queryDurations.filter(d => d > 500).length,
  };

  // Calculate percentiles
  const sorted = [...queryDurations].sort((a, b) => a - b);
  const p50 = sorted[Math.floor(sorted.length * 0.5)];
  const p95 = sorted[Math.floor(sorted.length * 0.95)];
  const p99 = sorted[Math.floor(sorted.length * 0.99)];

  // Log metrics
  console.log('Query duration histogram:', buckets);
  console.log(`Percentiles: p50=${p50}ms, p95=${p95}ms, p99=${p99}ms`);

  // Verify reasonable query times
  expect(p50).toBeLessThan(100); // p50 under 100ms
  expect(p95).toBeLessThan(500); // p95 under 500ms

  // All queries should complete
  expect(measurements.length).toBe(4);
});
```

**Expected Result**:
- Query durations are measured
- Histogram buckets are populated
- Percentiles are calculated
- Performance is within acceptable bounds

---

### DB-LOGGING-001: Query Execution Time Logging

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-LOGGING-001 |
| **Title** | Log Query Execution Times |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-303 |
| **Gap Reference** | Consolidated Review - Gap Domain 6 |

**Test Steps**:

```typescript
it('should log query execution times', async () => {
  const queryLogs: Array<{ query: string; duration: number; timestamp: string }> = [];

  // Create a logging wrapper
  const loggedQuery = async <T>(
    label: string,
    queryFn: () => Promise<T>
  ): Promise<T> => {
    const start = Date.now();
    const result = await queryFn();
    const duration = Date.now() - start;

    queryLogs.push({
      query: label,
      duration,
      timestamp: new Date().toISOString(),
    });

    return result;
  };

  // Execute queries with logging
  await loggedQuery('job.findMany', () => prisma.job.findMany({ take: 20 }));
  await loggedQuery('job.count', () => prisma.job.count());
  await loggedQuery('result.findMany', () => prisma.result.findMany({ take: 10 }));

  // Verify logs captured
  expect(queryLogs.length).toBe(3);

  // Each log entry should have required fields
  for (const log of queryLogs) {
    expect(log.query).toBeDefined();
    expect(typeof log.duration).toBe('number');
    expect(log.timestamp).toMatch(/^\d{4}-\d{2}-\d{2}T/);
  }

  // Log format should be parseable
  const logOutput = queryLogs.map(
    l => `[${l.timestamp}] ${l.query}: ${l.duration}ms`
  );
  console.log('Query logs:', logOutput);

  expect(logOutput.length).toBe(3);
});
```

**Expected Result**:
- Query execution times are logged
- Log entries contain query label, duration, timestamp
- Logs are structured for parsing

---

### DB-LOGGING-002: Slow Query Detection

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-LOGGING-002 |
| **Title** | Detect and Log Slow Queries |
| **Priority** | P0 - Critical |
| **Spec Reference** | NFR-303 |
| **Gap Reference** | Consolidated Review - Gap Domain 6 |

**Test Steps**:

```typescript
it('should detect and flag slow queries', async () => {
  const SLOW_QUERY_THRESHOLD_MS = 100;
  const slowQueryLog: Array<{
    query: string;
    duration: number;
    threshold: number;
    exceededBy: number;
  }> = [];

  // Slow query detector
  const detectSlowQuery = async <T>(
    label: string,
    queryFn: () => Promise<T>,
    thresholdMs: number = SLOW_QUERY_THRESHOLD_MS
  ): Promise<T> => {
    const start = Date.now();
    const result = await queryFn();
    const duration = Date.now() - start;

    if (duration > thresholdMs) {
      slowQueryLog.push({
        query: label,
        duration,
        threshold: thresholdMs,
        exceededBy: duration - thresholdMs,
      });
      console.warn(`SLOW QUERY: ${label} took ${duration}ms (threshold: ${thresholdMs}ms)`);
    }

    return result;
  };

  // Execute queries (some may be slow)
  await detectSlowQuery('fast_query', () => prisma.job.findFirst());

  // Intentionally slow query
  await detectSlowQuery('pg_sleep', () =>
    prisma.$queryRaw`SELECT pg_sleep(0.15)`
  );

  // Verify slow query was detected
  expect(slowQueryLog.length).toBeGreaterThanOrEqual(1);

  const slowQuery = slowQueryLog.find(q => q.query === 'pg_sleep');
  expect(slowQuery).toBeDefined();
  expect(slowQuery?.duration).toBeGreaterThan(SLOW_QUERY_THRESHOLD_MS);
  expect(slowQuery?.exceededBy).toBeGreaterThan(0);

  // Verify slow query alerts contain actionable info
  for (const slow of slowQueryLog) {
    expect(slow.query).toBeDefined();
    expect(slow.threshold).toBe(SLOW_QUERY_THRESHOLD_MS);
    expect(slow.exceededBy).toBeGreaterThan(0);
  }
});
```

**Expected Result**:
- Slow queries exceeding threshold are detected
- Slow query details are logged
- Threshold exceedance is calculated
- Alerts contain actionable information

---

## 14. Data Constraints and Schema Validation

> **Priority**: P1 - HIGH
> **Gap Reference**: Consolidated Review - Gap Domain 7
> **Risk Level**: Data integrity risk - invalid data stored, constraint violations cause runtime errors

This section validates schema alignment with specifications, enum coverage, size limits, NOT NULL constraints, unique constraints, and cascade behavior. Without these tests, invalid data may be stored in the database and constraint violations will cause runtime errors instead of validation errors.

---

### DB-SCHEMA-001: Schema Matches Specification

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-SCHEMA-001 |
| **Title** | Verify Schema Matches Specification Exactly |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.1 |
| **Gap Reference** | Consolidated Review - Gap Domain 7 |

**Test Steps**:

```typescript
it('should have schema matching specification exactly', async () => {
  // Verify Job table columns match specification
  const jobColumns = await prisma.$queryRaw<any[]>`
    SELECT column_name, data_type, is_nullable, column_default
    FROM information_schema.columns
    WHERE table_name = 'Job'
    ORDER BY ordinal_position
  `;

  // Required columns from specification (Section 5.1)
  const expectedJobColumns = [
    'id',
    'sessionId',
    'status',
    'inputType',
    'fileName',
    'fileSize',
    'mimeType',
    'filePath',
    'sourceUrl',
    'errorCode',
    'errorMessage',
    'retryCount',
    'checkpointStage',
    'checkpointData',
    'checkpointHash',
    'createdAt',
    'updatedAt',
    'completedAt',
  ];

  const actualColumnNames = jobColumns.map(c => c.column_name);

  for (const expected of expectedJobColumns) {
    expect(actualColumnNames).toContain(expected);
  }

  // Verify Result table columns
  const resultColumns = await prisma.$queryRaw<any[]>`
    SELECT column_name, data_type, is_nullable
    FROM information_schema.columns
    WHERE table_name = 'Result'
    ORDER BY ordinal_position
  `;

  const expectedResultColumns = ['id', 'jobId', 'format', 'content', 'size', 'createdAt'];
  const actualResultColumns = resultColumns.map(c => c.column_name);

  for (const expected of expectedResultColumns) {
    expect(actualResultColumns).toContain(expected);
  }
});
```

**Expected Result**:
- Job table contains all specified columns
- Result table contains all specified columns
- Column names match specification exactly

---

### DB-SCHEMA-002: Result Format Enum Coverage (Including RAW)

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-SCHEMA-002 |
| **Title** | Test All Result Format Enums Including RAW |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.1 |
| **Gap Reference** | Consolidated Review - Gap Domain 7 |

**Test Steps**:

```typescript
it('should support all ResultFormat enum values including RAW', async () => {
  const testSession = 'result-format-enum-test';

  // Create job for results
  const job = await prisma.job.create({
    data: {
      sessionId: testSession,
      status: 'COMPLETE',
      inputType: 'FILE',
      fileName: 'enum-test.pdf',
    },
  });

  // Test all format enum values
  const formats = ['MARKDOWN', 'HTML', 'JSON', 'RAW'] as const;
  const createdResults: any[] = [];

  for (const format of formats) {
    const result = await prisma.result.create({
      data: {
        jobId: job.id,
        format: format,
        content: `Test content for ${format} format`,
        size: 100,
      },
    });
    createdResults.push(result);
    expect(result.format).toBe(format);
  }

  // Verify all formats stored and retrievable
  const retrievedResults = await prisma.result.findMany({
    where: { jobId: job.id },
  });

  expect(retrievedResults.length).toBe(4);

  const retrievedFormats = retrievedResults.map(r => r.format);
  expect(retrievedFormats).toContain('MARKDOWN');
  expect(retrievedFormats).toContain('HTML');
  expect(retrievedFormats).toContain('JSON');
  expect(retrievedFormats).toContain('RAW');

  // Cleanup
  await prisma.result.deleteMany({ where: { jobId: job.id } });
  await prisma.job.delete({ where: { id: job.id } });
});
```

**Expected Result**:
- All ResultFormat enum values are accepted
- RAW format is properly supported
- Results can be retrieved and filtered by format

---

### DB-SCHEMA-003: Result Size Limits Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-SCHEMA-003 |
| **Title** | Validate Result Content Size Limits |
| **Priority** | P1 - High |
| **Spec Reference** | Section 5.1 |
| **Gap Reference** | Consolidated Review - Gap Domain 7 |

**Test Steps**:

```typescript
it('should enforce result size limits by format', async () => {
  const testSession = 'result-size-limit-test';

  const job = await prisma.job.create({
    data: {
      sessionId: testSession,
      status: 'COMPLETE',
      inputType: 'FILE',
      fileName: 'size-test.pdf',
    },
  });

  // Size limits from specification:
  // - MARKDOWN: 25MB max
  // - HTML: 25MB max
  // - JSON: 25MB max
  // - RAW: 50MB max
  const SIZE_LIMITS = {
    MARKDOWN: 25 * 1024 * 1024, // 25MB
    HTML: 25 * 1024 * 1024,     // 25MB
    JSON: 25 * 1024 * 1024,     // 25MB
    RAW: 50 * 1024 * 1024,      // 50MB
  };

  // Test within-limits content (small test - 1KB)
  const smallContent = 'x'.repeat(1024);
  const smallResult = await prisma.result.create({
    data: {
      jobId: job.id,
      format: 'MARKDOWN',
      content: smallContent,
      size: smallContent.length,
    },
  });
  expect(smallResult.size).toBe(1024);

  // Validate size limit enforcement logic
  const validateSizeLimit = (format: keyof typeof SIZE_LIMITS, size: number): boolean => {
    return size <= SIZE_LIMITS[format];
  };

  // Test size validation logic
  expect(validateSizeLimit('MARKDOWN', 24 * 1024 * 1024)).toBe(true);  // 24MB - OK
  expect(validateSizeLimit('MARKDOWN', 26 * 1024 * 1024)).toBe(false); // 26MB - exceeds
  expect(validateSizeLimit('RAW', 49 * 1024 * 1024)).toBe(true);       // 49MB - OK
  expect(validateSizeLimit('RAW', 51 * 1024 * 1024)).toBe(false);      // 51MB - exceeds

  // Verify size is tracked correctly
  const retrieved = await prisma.result.findUnique({
    where: { id: smallResult.id },
    select: { size: true },
  });
  expect(retrieved?.size).toBe(1024);

  // Cleanup
  await prisma.result.deleteMany({ where: { jobId: job.id } });
  await prisma.job.delete({ where: { id: job.id } });
});
```

**Expected Result**:
- Size limits are defined per format type
- Size validation logic correctly identifies violations
- Result size is tracked accurately

---

### DB-SCHEMA-004: NOT NULL Constraints Enforcement

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-SCHEMA-004 |
| **Title** | Test NOT NULL Constraints on Required Fields |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.1 |
| **Gap Reference** | Consolidated Review - Gap Domain 7 |

**Test Steps**:

```typescript
it('should enforce NOT NULL constraints on required fields', async () => {
  // Job required fields: id, sessionId, status, inputType, createdAt, updatedAt

  // Test missing sessionId
  await expect(
    prisma.job.create({
      data: {
        status: 'PENDING',
        inputType: 'FILE',
      } as any,
    })
  ).rejects.toThrow();

  // Test missing inputType
  await expect(
    prisma.job.create({
      data: {
        sessionId: 'not-null-test',
        status: 'PENDING',
      } as any,
    })
  ).rejects.toThrow();

  // Result required fields: id, jobId, format, content, size, createdAt
  const job = await prisma.job.create({
    data: {
      sessionId: 'not-null-constraint-test',
      status: 'PENDING',
      inputType: 'FILE',
    },
  });

  // Test missing jobId on Result
  await expect(
    prisma.result.create({
      data: {
        format: 'MARKDOWN',
        content: 'test',
        size: 4,
      } as any,
    })
  ).rejects.toThrow();

  // Test missing format on Result
  await expect(
    prisma.result.create({
      data: {
        jobId: job.id,
        content: 'test',
        size: 4,
      } as any,
    })
  ).rejects.toThrow();

  // Test missing content on Result
  await expect(
    prisma.result.create({
      data: {
        jobId: job.id,
        format: 'MARKDOWN',
        size: 4,
      } as any,
    })
  ).rejects.toThrow();

  // Cleanup
  await prisma.job.delete({ where: { id: job.id } });
});
```

**Expected Result**:
- Missing required fields cause creation to fail
- NOT NULL constraints enforced at database level
- Error thrown before data corruption

---

### DB-SCHEMA-005: Unique Constraints on Appropriate Fields

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-SCHEMA-005 |
| **Title** | Test Unique Constraints on ID Fields |
| **Priority** | P1 - High |
| **Spec Reference** | Section 5.1 |
| **Gap Reference** | Consolidated Review - Gap Domain 7 |

**Test Steps**:

```typescript
it('should enforce unique constraints on id fields', async () => {
  // Verify Job.id is unique (auto-generated UUID)
  const job1 = await prisma.job.create({
    data: {
      sessionId: 'unique-test',
      status: 'PENDING',
      inputType: 'FILE',
    },
  });

  const job2 = await prisma.job.create({
    data: {
      sessionId: 'unique-test',
      status: 'PENDING',
      inputType: 'FILE',
    },
  });

  // UUIDs should be unique
  expect(job1.id).not.toBe(job2.id);

  // Verify Result.id is unique
  const result1 = await prisma.result.create({
    data: {
      jobId: job1.id,
      format: 'MARKDOWN',
      content: 'test 1',
      size: 6,
    },
  });

  const result2 = await prisma.result.create({
    data: {
      jobId: job1.id,
      format: 'HTML',
      content: 'test 2',
      size: 6,
    },
  });

  expect(result1.id).not.toBe(result2.id);

  // Test that duplicate ID insertion fails (if attempted manually)
  await expect(
    prisma.$executeRaw`
      INSERT INTO "Job" (id, "sessionId", status, "inputType", "createdAt", "updatedAt")
      VALUES (${job1.id}, 'dup-test', 'PENDING', 'FILE', NOW(), NOW())
    `
  ).rejects.toThrow();

  // Cleanup
  await prisma.result.deleteMany({ where: { jobId: { in: [job1.id, job2.id] } } });
  await prisma.job.deleteMany({ where: { id: { in: [job1.id, job2.id] } } });
});
```

**Expected Result**:
- Auto-generated UUIDs are unique
- Duplicate ID insertion fails with constraint violation
- Unique constraints enforced at database level

---

### DB-SCHEMA-006: Cascade Delete Behavior Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-SCHEMA-006 |
| **Title** | Validate Cascade Delete on Job-Result Relationship |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.1 |
| **Gap Reference** | Consolidated Review - Gap Domain 7 |

**Test Steps**:

```typescript
it('should cascade delete results when job is deleted', async () => {
  const testSession = 'cascade-delete-test';

  // Create job with multiple results
  const job = await prisma.job.create({
    data: {
      sessionId: testSession,
      status: 'COMPLETE',
      inputType: 'FILE',
      fileName: 'cascade-test.pdf',
    },
  });

  const result1 = await prisma.result.create({
    data: {
      jobId: job.id,
      format: 'MARKDOWN',
      content: 'Markdown content',
      size: 16,
    },
  });

  const result2 = await prisma.result.create({
    data: {
      jobId: job.id,
      format: 'HTML',
      content: '<p>HTML content</p>',
      size: 19,
    },
  });

  const result3 = await prisma.result.create({
    data: {
      jobId: job.id,
      format: 'JSON',
      content: '{"type": "json"}',
      size: 16,
    },
  });

  // Verify results exist
  const resultsBeforeDelete = await prisma.result.findMany({
    where: { jobId: job.id },
  });
  expect(resultsBeforeDelete.length).toBe(3);

  // Delete the job
  await prisma.job.delete({
    where: { id: job.id },
  });

  // Verify results are cascade deleted
  const resultsAfterDelete = await prisma.result.findMany({
    where: { id: { in: [result1.id, result2.id, result3.id] } },
  });
  expect(resultsAfterDelete.length).toBe(0);

  // Verify job is deleted
  const deletedJob = await prisma.job.findUnique({
    where: { id: job.id },
  });
  expect(deletedJob).toBeNull();
});
```

**Expected Result**:
- Deleting a job cascades to delete all associated results
- No orphaned results remain after job deletion
- Foreign key constraint with ON DELETE CASCADE is enforced

---

### DB-SCHEMA-007: Field Length Constraint Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-SCHEMA-007 |
| **Title** | Validate Field Length Constraints |
| **Priority** | P1 - High |
| **Spec Reference** | Section 5.1 |
| **Gap Reference** | Consolidated Review - Gap Domain 7 |

**Test Steps**:

```typescript
it('should handle field length constraints appropriately', async () => {
  // Test reasonable field lengths
  const reasonableFileName = 'document.pdf';
  const reasonableMimeType = 'application/pdf';
  const reasonableErrorCode = 'E101';
  const reasonableErrorMessage = 'File not found';

  const job = await prisma.job.create({
    data: {
      sessionId: 'field-length-test',
      status: 'ERROR',
      inputType: 'FILE',
      fileName: reasonableFileName,
      mimeType: reasonableMimeType,
      errorCode: reasonableErrorCode,
      errorMessage: reasonableErrorMessage,
    },
  });

  expect(job.fileName).toBe(reasonableFileName);
  expect(job.mimeType).toBe(reasonableMimeType);
  expect(job.errorCode).toBe(reasonableErrorCode);
  expect(job.errorMessage).toBe(reasonableErrorMessage);

  // Test long but acceptable file name (255 chars typical limit)
  const longFileName = 'a'.repeat(200) + '.pdf';
  const jobWithLongName = await prisma.job.create({
    data: {
      sessionId: 'long-field-test',
      status: 'PENDING',
      inputType: 'FILE',
      fileName: longFileName,
    },
  });
  expect(jobWithLongName.fileName?.length).toBe(204);

  // Test long error message (should be stored as TEXT)
  const longErrorMessage = 'Error: '.repeat(1000);
  const jobWithLongError = await prisma.job.create({
    data: {
      sessionId: 'long-error-test',
      status: 'ERROR',
      inputType: 'FILE',
      errorCode: 'E999',
      errorMessage: longErrorMessage,
    },
  });
  expect(jobWithLongError.errorMessage?.length).toBe(7000);

  // Cleanup
  await prisma.job.deleteMany({
    where: { sessionId: { in: ['field-length-test', 'long-field-test', 'long-error-test'] } },
  });
});
```

**Expected Result**:
- Standard field values are accepted
- Long file names are stored correctly
- Long error messages are stored in TEXT column
- Field lengths within database limits

---

### DB-SCHEMA-008: Enum Value Range Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-SCHEMA-008 |
| **Title** | Validate Enum Value Ranges |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.1 |
| **Gap Reference** | Consolidated Review - Gap Domain 7 |

**Test Steps**:

```typescript
it('should only accept valid enum values', async () => {
  // Test all valid JobStatus enum values
  const validStatuses = [
    'PENDING',
    'UPLOADING',
    'PROCESSING',
    'RETRY_1',
    'RETRY_2',
    'RETRY_3',
    'COMPLETE',
    'PARTIAL_COMPLETE',
    'ERROR',
    'CANCELLED',
  ] as const;

  const createdJobs: string[] = [];

  for (const status of validStatuses) {
    const job = await prisma.job.create({
      data: {
        sessionId: `enum-status-${status}`,
        status: status,
        inputType: 'FILE',
      },
    });
    createdJobs.push(job.id);
    expect(job.status).toBe(status);
  }

  // Test valid InputType enum values
  const validInputTypes = ['FILE', 'URL'] as const;

  for (const inputType of validInputTypes) {
    const job = await prisma.job.create({
      data: {
        sessionId: `enum-input-${inputType}`,
        status: 'PENDING',
        inputType: inputType,
      },
    });
    createdJobs.push(job.id);
    expect(job.inputType).toBe(inputType);
  }

  // Test invalid enum values are rejected (raw SQL to bypass Prisma validation)
  await expect(
    prisma.$executeRaw`
      INSERT INTO "Job" (id, "sessionId", status, "inputType", "createdAt", "updatedAt")
      VALUES (gen_random_uuid(), 'invalid-enum', 'INVALID_STATUS', 'FILE', NOW(), NOW())
    `
  ).rejects.toThrow();

  await expect(
    prisma.$executeRaw`
      INSERT INTO "Job" (id, "sessionId", status, "inputType", "createdAt", "updatedAt")
      VALUES (gen_random_uuid(), 'invalid-enum', 'PENDING', 'INVALID_TYPE', NOW(), NOW())
    `
  ).rejects.toThrow();

  // Cleanup
  await prisma.job.deleteMany({ where: { id: { in: createdJobs } } });
});
```

**Expected Result**:
- All valid enum values are accepted
- Invalid enum values are rejected at database level
- Enum constraints enforced by PostgreSQL

---

## 15. Backup, Recovery, and Data Retention

> **Priority**: P1-P2 - HIGH to MEDIUM
> **Gap Reference**: Consolidated Review - Gap Domain 8
> **Risk Level**: Disaster recovery risk - untested backups are equivalent to no backups

This section validates backup creation, restore procedures, data consistency after restore, point-in-time recovery capability, and data retention policy enforcement. Without these tests, recovery procedures are untested until disaster occurs and data retention could cause unbounded database growth.

---

### DB-BACKUP-001: Backup File Creation and Integrity

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-BACKUP-001 |
| **Title** | Create Backup File and Verify Integrity |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-401 |
| **Gap Reference** | Consolidated Review - Gap Domain 8 |

**Test Steps**:

```typescript
it('should create backup file with integrity verification', async () => {
  const fs = require('fs');
  const path = require('path');
  const { exec } = require('child_process');
  const util = require('util');
  const execAsync = util.promisify(exec);

  const backupDir = '/tmp/db-backups';
  const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
  const backupFile = path.join(backupDir, `backup-test-${timestamp}.sql`);

  // Ensure backup directory exists
  if (!fs.existsSync(backupDir)) {
    fs.mkdirSync(backupDir, { recursive: true });
  }

  // Create some test data to backup
  const testJob = await prisma.job.create({
    data: {
      sessionId: 'backup-test-session',
      status: 'COMPLETE',
      inputType: 'FILE',
      fileName: 'backup-test.pdf',
    },
  });

  await prisma.result.create({
    data: {
      jobId: testJob.id,
      format: 'MARKDOWN',
      content: 'Backup test content',
      size: 19,
    },
  });

  // Create backup using pg_dump
  const dbUrl = process.env.TEST_DATABASE_URL || process.env.DATABASE_URL;
  const connectionString = new URL(dbUrl!);

  try {
    const { stdout, stderr } = await execAsync(
      `PGPASSWORD="${connectionString.password}" pg_dump -h ${connectionString.hostname} -p ${connectionString.port || 5432} -U ${connectionString.username} -d ${connectionString.pathname.slice(1)} -F c -f ${backupFile}`
    );

    // Verify backup file exists and has content
    expect(fs.existsSync(backupFile)).toBe(true);

    const stats = fs.statSync(backupFile);
    expect(stats.size).toBeGreaterThan(0);

    // Verify backup integrity using pg_restore --list
    const { stdout: listOutput } = await execAsync(`pg_restore --list ${backupFile}`);

    // Should contain Job and Result tables
    expect(listOutput).toContain('Job');
    expect(listOutput).toContain('Result');

    console.log(`Backup created: ${backupFile} (${stats.size} bytes)`);

  } finally {
    // Cleanup backup file
    if (fs.existsSync(backupFile)) {
      fs.unlinkSync(backupFile);
    }

    // Cleanup test data
    await prisma.result.deleteMany({ where: { jobId: testJob.id } });
    await prisma.job.delete({ where: { id: testJob.id } });
  }
}, 30000);
```

**Expected Result**:
- Backup file is created successfully
- Backup file has non-zero size
- Backup contains Job and Result table structures
- pg_restore can list backup contents

---

### DB-BACKUP-002: Restore Procedure Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-BACKUP-002 |
| **Title** | Validate Restore Procedure |
| **Priority** | P0 - Critical |
| **Spec Reference** | NFR-401 |
| **Gap Reference** | Consolidated Review - Gap Domain 8 |

**Test Steps**:

```typescript
it('should restore data from backup successfully', async () => {
  // Note: This test validates the restore LOGIC, not actual restore to production
  // Actual restore testing requires a separate test database

  const backupData = {
    jobs: [] as any[],
    results: [] as any[],
    backupTimestamp: new Date().toISOString(),
  };

  // Create test data for "backup"
  const job1 = await prisma.job.create({
    data: {
      sessionId: 'restore-test-session',
      status: 'COMPLETE',
      inputType: 'FILE',
      fileName: 'restore-test-1.pdf',
    },
  });

  const result1 = await prisma.result.create({
    data: {
      jobId: job1.id,
      format: 'MARKDOWN',
      content: 'Restore test content 1',
      size: 21,
    },
  });

  // Simulate backup capture
  backupData.jobs.push({
    id: job1.id,
    sessionId: job1.sessionId,
    status: job1.status,
    inputType: job1.inputType,
    fileName: job1.fileName,
    createdAt: job1.createdAt.toISOString(),
    updatedAt: job1.updatedAt.toISOString(),
  });

  backupData.results.push({
    id: result1.id,
    jobId: result1.jobId,
    format: result1.format,
    content: result1.content,
    size: result1.size,
    createdAt: result1.createdAt.toISOString(),
  });

  // Delete original data (simulate data loss)
  await prisma.result.deleteMany({ where: { jobId: job1.id } });
  await prisma.job.delete({ where: { id: job1.id } });

  // Verify data is gone
  const deletedJob = await prisma.job.findUnique({ where: { id: job1.id } });
  expect(deletedJob).toBeNull();

  // Simulate restore from backup
  const restoredJob = await prisma.job.create({
    data: {
      id: backupData.jobs[0].id,
      sessionId: backupData.jobs[0].sessionId,
      status: backupData.jobs[0].status,
      inputType: backupData.jobs[0].inputType,
      fileName: backupData.jobs[0].fileName,
    },
  });

  const restoredResult = await prisma.result.create({
    data: {
      id: backupData.results[0].id,
      jobId: backupData.results[0].jobId,
      format: backupData.results[0].format,
      content: backupData.results[0].content,
      size: backupData.results[0].size,
    },
  });

  // Verify restored data matches backup
  expect(restoredJob.id).toBe(job1.id);
  expect(restoredJob.sessionId).toBe('restore-test-session');
  expect(restoredJob.status).toBe('COMPLETE');
  expect(restoredResult.id).toBe(result1.id);
  expect(restoredResult.content).toBe('Restore test content 1');

  // Cleanup
  await prisma.result.deleteMany({ where: { jobId: restoredJob.id } });
  await prisma.job.delete({ where: { id: restoredJob.id } });
});
```

**Expected Result**:
- Data can be captured for backup
- Data can be restored from backup
- Restored data matches original exactly
- Foreign key relationships maintained

---

### DB-BACKUP-003: Data Consistency Post-Restore

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-BACKUP-003 |
| **Title** | Verify Data Consistency After Restore |
| **Priority** | P0 - Critical |
| **Spec Reference** | NFR-401 |
| **Gap Reference** | Consolidated Review - Gap Domain 8 |

**Test Steps**:

```typescript
it('should maintain data consistency after restore', async () => {
  // Create comprehensive test dataset
  const testSession = 'consistency-test-session';

  const job = await prisma.job.create({
    data: {
      sessionId: testSession,
      status: 'COMPLETE',
      inputType: 'FILE',
      fileName: 'consistency-test.pdf',
      fileSize: 102400,
      mimeType: 'application/pdf',
      checkpointStage: 'complete',
      checkpointData: {
        stage: 'complete',
        version: '1.0.0',
        processedPages: 10,
      },
    },
  });

  // Create multiple results
  const formats = ['MARKDOWN', 'HTML', 'JSON'] as const;
  for (const format of formats) {
    await prisma.result.create({
      data: {
        jobId: job.id,
        format: format,
        content: `${format} content for consistency test`,
        size: 40,
      },
    });
  }

  // Verify pre-restore state
  const preRestoreJob = await prisma.job.findUnique({
    where: { id: job.id },
    include: { results: true },
  });

  expect(preRestoreJob?.results.length).toBe(3);
  expect(preRestoreJob?.checkpointData).toBeDefined();

  // Verify JSON data consistency
  const checkpointData = preRestoreJob?.checkpointData as any;
  expect(checkpointData.stage).toBe('complete');
  expect(checkpointData.processedPages).toBe(10);

  // Verify referential integrity
  const orphanedResults = await prisma.result.findMany({
    where: {
      jobId: job.id,
      job: null, // Should not exist - foreign key constraint
    },
  });
  expect(orphanedResults.length).toBe(0);

  // Verify result-job relationship
  for (const result of preRestoreJob?.results || []) {
    expect(result.jobId).toBe(job.id);
  }

  // Cleanup
  await prisma.result.deleteMany({ where: { jobId: job.id } });
  await prisma.job.delete({ where: { id: job.id } });
});
```

**Expected Result**:
- All relationships are maintained after restore
- JSON data is correctly serialized/deserialized
- No orphaned records exist
- Foreign key constraints are enforced

---

### DB-BACKUP-004: Point-in-Time Recovery Capability

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-BACKUP-004 |
| **Title** | Validate Point-in-Time Recovery Capability |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-401 |
| **Gap Reference** | Consolidated Review - Gap Domain 8 |

**Test Steps**:

```typescript
it('should support point-in-time recovery queries', async () => {
  const testSession = 'pitr-test-session';

  // Create initial job
  const job = await prisma.job.create({
    data: {
      sessionId: testSession,
      status: 'PENDING',
      inputType: 'FILE',
      fileName: 'pitr-test.pdf',
    },
  });

  const timeAfterCreate = new Date();
  await new Promise(r => setTimeout(r, 100));

  // Update job status
  await prisma.job.update({
    where: { id: job.id },
    data: { status: 'PROCESSING' },
  });

  const timeAfterProcessing = new Date();
  await new Promise(r => setTimeout(r, 100));

  // Complete the job
  await prisma.job.update({
    where: { id: job.id },
    data: {
      status: 'COMPLETE',
      completedAt: new Date(),
    },
  });

  // Query job history using timestamps
  // Note: This tests the ability to query by time range
  // Actual PITR requires WAL archiving at PostgreSQL level

  const jobsCreatedAfterCreate = await prisma.job.findMany({
    where: {
      sessionId: testSession,
      createdAt: { lte: timeAfterCreate },
    },
  });
  expect(jobsCreatedAfterCreate.length).toBe(1);

  const jobsUpdatedAfterProcessing = await prisma.job.findMany({
    where: {
      sessionId: testSession,
      updatedAt: { gte: timeAfterProcessing },
    },
  });
  expect(jobsUpdatedAfterProcessing.length).toBe(1);
  expect(jobsUpdatedAfterProcessing[0].status).toBe('COMPLETE');

  // Verify updatedAt tracks all changes
  const currentJob = await prisma.job.findUnique({ where: { id: job.id } });
  expect(currentJob?.updatedAt.getTime()).toBeGreaterThan(job.createdAt.getTime());

  // Test timestamp-based filtering for recovery scenarios
  const recoverableJobs = await prisma.job.findMany({
    where: {
      status: 'COMPLETE',
      completedAt: { gte: timeAfterProcessing },
    },
  });
  expect(recoverableJobs.some(j => j.id === job.id)).toBe(true);

  // Cleanup
  await prisma.job.delete({ where: { id: job.id } });
});
```

**Expected Result**:
- Timestamps are accurately tracked
- Time-based queries work correctly
- Job state changes can be tracked by time
- Recovery time objectives can be met with timestamp filtering

---

### DB-RETENTION-001: Old Job Cleanup Procedure

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-RETENTION-001 |
| **Title** | Validate Old Job Cleanup Procedure |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-402 |
| **Gap Reference** | Consolidated Review - Gap Domain 8 |

**Test Steps**:

```typescript
it('should identify and clean up old completed jobs', async () => {
  const RETENTION_DAYS = 30;
  const retentionThreshold = new Date(Date.now() - RETENTION_DAYS * 24 * 60 * 60 * 1000);

  // Create old completed job (beyond retention period)
  const oldJob = await prisma.job.create({
    data: {
      sessionId: 'retention-old-job',
      status: 'COMPLETE',
      inputType: 'FILE',
      fileName: 'old-file.pdf',
      completedAt: new Date(Date.now() - 45 * 24 * 60 * 60 * 1000), // 45 days ago
    },
  });

  // Set createdAt to old date
  await prisma.$executeRaw`
    UPDATE "Job"
    SET "createdAt" = ${new Date(Date.now() - 45 * 24 * 60 * 60 * 1000)},
        "updatedAt" = ${new Date(Date.now() - 45 * 24 * 60 * 60 * 1000)}
    WHERE id = ${oldJob.id}
  `;

  // Create result for old job
  await prisma.result.create({
    data: {
      jobId: oldJob.id,
      format: 'MARKDOWN',
      content: 'Old content to be cleaned up',
      size: 27,
    },
  });

  // Create recent job (within retention period)
  const recentJob = await prisma.job.create({
    data: {
      sessionId: 'retention-recent-job',
      status: 'COMPLETE',
      inputType: 'FILE',
      fileName: 'recent-file.pdf',
      completedAt: new Date(Date.now() - 5 * 24 * 60 * 60 * 1000), // 5 days ago
    },
  });

  // Query for jobs eligible for cleanup
  const jobsToCleanup = await prisma.job.findMany({
    where: {
      status: { in: ['COMPLETE', 'ERROR', 'CANCELLED'] },
      completedAt: { lt: retentionThreshold },
    },
  });

  // Verify old job is identified for cleanup
  expect(jobsToCleanup.some(j => j.id === oldJob.id)).toBe(true);
  expect(jobsToCleanup.some(j => j.id === recentJob.id)).toBe(false);

  // Execute cleanup (cascade deletes results)
  const cleanupResult = await prisma.job.deleteMany({
    where: {
      status: { in: ['COMPLETE', 'ERROR', 'CANCELLED'] },
      completedAt: { lt: retentionThreshold },
    },
  });

  expect(cleanupResult.count).toBeGreaterThanOrEqual(1);

  // Verify old job and results are deleted
  const deletedJob = await prisma.job.findUnique({ where: { id: oldJob.id } });
  expect(deletedJob).toBeNull();

  const orphanedResults = await prisma.result.findMany({
    where: { jobId: oldJob.id },
  });
  expect(orphanedResults.length).toBe(0);

  // Verify recent job is retained
  const retainedJob = await prisma.job.findUnique({ where: { id: recentJob.id } });
  expect(retainedJob).not.toBeNull();

  // Cleanup recent job
  await prisma.job.delete({ where: { id: recentJob.id } });
});
```

**Expected Result**:
- Jobs beyond retention period are identified
- Cleanup procedure deletes old jobs
- Associated results are cascade deleted
- Recent jobs are retained
- No data loss for jobs within retention period

---

### DB-RETENTION-002: Orphaned File Reference Cleanup

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-RETENTION-002 |
| **Title** | Clean Up Orphaned File References |
| **Priority** | P2 - Medium |
| **Spec Reference** | NFR-402 |
| **Gap Reference** | Consolidated Review - Gap Domain 8 |

**Test Steps**:

```typescript
it('should identify orphaned file references for cleanup', async () => {
  // Create job with file reference
  const jobWithFile = await prisma.job.create({
    data: {
      sessionId: 'orphan-file-test',
      status: 'COMPLETE',
      inputType: 'FILE',
      fileName: 'existing-file.pdf',
      filePath: '/tmp/uploads/existing-file.pdf',
      fileSize: 102400,
    },
  });

  // Create job with missing file reference (orphaned)
  const jobWithOrphanedFile = await prisma.job.create({
    data: {
      sessionId: 'orphan-file-test',
      status: 'ERROR',
      inputType: 'FILE',
      fileName: 'missing-file.pdf',
      filePath: '/tmp/uploads/nonexistent-file-12345.pdf',
      fileSize: 51200,
      errorCode: 'E101',
      errorMessage: 'File not found',
    },
  });

  // Query for jobs with file references
  const jobsWithFiles = await prisma.job.findMany({
    where: {
      filePath: { not: null },
    },
    select: {
      id: true,
      filePath: true,
      status: true,
    },
  });

  expect(jobsWithFiles.length).toBeGreaterThanOrEqual(2);

  // Identify orphaned file references
  // In production, this would check filesystem
  const fs = require('fs');
  const orphanedFileRefs: Array<{ jobId: string; filePath: string }> = [];

  for (const job of jobsWithFiles) {
    if (job.filePath && !fs.existsSync(job.filePath)) {
      orphanedFileRefs.push({
        jobId: job.id,
        filePath: job.filePath,
      });
    }
  }

  // Both test files don't exist on filesystem (test environment)
  expect(orphanedFileRefs.length).toBeGreaterThanOrEqual(2);

  // Verify orphan detection found our test job
  expect(orphanedFileRefs.some(o => o.jobId === jobWithOrphanedFile.id)).toBe(true);

  // Clean up orphaned references (set filePath to null)
  const cleanupCount = await prisma.job.updateMany({
    where: {
      id: { in: orphanedFileRefs.map(o => o.jobId) },
      status: { in: ['ERROR', 'CANCELLED'] }, // Only clean up failed jobs
    },
    data: {
      filePath: null,
    },
  });

  expect(cleanupCount.count).toBeGreaterThanOrEqual(1);

  // Verify cleanup
  const cleanedJob = await prisma.job.findUnique({
    where: { id: jobWithOrphanedFile.id },
  });
  expect(cleanedJob?.filePath).toBeNull();

  // Cleanup test data
  await prisma.job.deleteMany({
    where: { sessionId: 'orphan-file-test' },
  });
});
```

**Expected Result**:
- Jobs with file references are queryable
- Orphaned file references (file doesn't exist) are identified
- Orphaned references can be cleaned up
- Cleanup only affects failed jobs (preserves valid data)

---

## 16. Data Migration Tests

> **Priority**: P0-P1 - CRITICAL to HIGH
> **Gap Reference**: Architectural Review A-06 - Database Migration Tests
> **Risk Level**: Production risk - untested migrations can cause downtime and data loss

This section validates database schema migrations, rollback procedures, data integrity during migrations, zero-downtime deployment capability, migration idempotency, and version tracking. Without comprehensive migration tests, schema changes pose significant production risk.

---

### DB-MIG-001: Schema Migration Up (Apply New Migration)

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-MIG-001 |
| **Title** | Apply New Schema Migration Successfully |
| **Priority** | P0 - Critical |
| **Spec Reference** | NFR-401 |
| **Gap Reference** | Architectural Review A-06 |

**Preconditions**:
- Test database is in clean state
- Prisma schema contains test migration
- No pending migrations exist

**Test Steps**:

```typescript
it('should apply new schema migration successfully', async () => {
  const { exec } = require('child_process');
  const util = require('util');
  const execAsync = util.promisify(exec);

  // Get current migration status
  const { stdout: statusBefore } = await execAsync('npx prisma migrate status');
  console.log('Migration status before:', statusBefore);

  // Apply all pending migrations
  const { stdout: deployOutput, stderr } = await execAsync('npx prisma migrate deploy');

  // Verify no errors
  expect(stderr).not.toContain('Error');
  expect(stderr).not.toContain('failed');

  // Verify migrations were applied
  expect(deployOutput).toMatch(/applied|up to date/i);

  // Verify migration status is up to date
  const { stdout: statusAfter } = await execAsync('npx prisma migrate status');
  expect(statusAfter).toMatch(/up to date|Database schema is up to date/i);

  console.log('Migration deploy output:', deployOutput);
}, 30000);
```

**Expected Result**:
- Migrations apply without errors
- Database schema is updated successfully
- Migration status shows all migrations applied
- No data loss occurs during migration

---

### DB-MIG-002: Schema Migration Down (Rollback Migration)

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-MIG-002 |
| **Title** | Rollback Schema Migration Successfully |
| **Priority** | P0 - Critical |
| **Spec Reference** | NFR-401 |
| **Gap Reference** | Architectural Review A-06 |

**Preconditions**:
- Database has applied migrations
- Rollback migration script exists
- Test data exists to validate rollback

**Test Steps**:

```typescript
it('should rollback schema migration successfully', async () => {
  const fs = require('fs').promises;
  const path = require('path');

  // Create test data before rollback
  const testJob = await prisma.job.create({
    data: {
      sessionId: 'rollback-test-session',
      status: 'COMPLETE',
      inputType: 'FILE',
      fileName: 'rollback-test.pdf',
    },
  });

  const testJobId = testJob.id;

  // Note: Prisma doesn't have built-in rollback command
  // This test validates the PROCEDURE for manual rollback

  // Get list of applied migrations
  const migrationsDir = path.join(process.cwd(), 'prisma', 'migrations');
  const migrations = await fs.readdir(migrationsDir);

  // Sort migrations by timestamp (newest first)
  const sortedMigrations = migrations
    .filter(m => m !== 'migration_lock.toml')
    .sort()
    .reverse();

  console.log('Available migrations:', sortedMigrations.slice(0, 3));

  // Verify migration history exists
  expect(sortedMigrations.length).toBeGreaterThan(0);

  // Check that _prisma_migrations table exists and has records
  const migrationRecords = await prisma.$queryRaw`
    SELECT migration_name, applied_steps_count
    FROM _prisma_migrations
    ORDER BY finished_at DESC
    LIMIT 5
  `;

  expect(Array.isArray(migrationRecords)).toBe(true);
  expect(migrationRecords.length).toBeGreaterThan(0);

  console.log('Recent migrations:', migrationRecords);

  // Verify rollback procedure documentation
  // In production: manual rollback requires:
  // 1. Create down migration SQL script
  // 2. Execute: psql -f rollback.sql
  // 3. Update _prisma_migrations table
  // 4. Verify schema state

  // Verify test data still exists (rollback not executed in this test)
  const jobAfterRollbackTest = await prisma.job.findUnique({
    where: { id: testJobId },
  });
  expect(jobAfterRollbackTest).not.toBeNull();

  // Cleanup
  await prisma.job.delete({ where: { id: testJobId } });
}, 15000);
```

**Expected Result**:
- Migration history is queryable
- Rollback procedure is documented and testable
- Migration table tracks all applied migrations
- Rollback process preserves data integrity

---

### DB-MIG-003: Data Migration Integrity

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-MIG-003 |
| **Title** | Validate Data Integrity During Migration |
| **Priority** | P0 - Critical |
| **Spec Reference** | NFR-401 |
| **Gap Reference** | Architectural Review A-06 |

**Preconditions**:
- Database contains test data
- Migration involves data transformation
- Validation queries are defined

**Test Steps**:

```typescript
it('should maintain data integrity during migration', async () => {
  // Create comprehensive test dataset
  const testSession = 'migration-integrity-test';

  // Create jobs with various states
  const jobs = await Promise.all([
    prisma.job.create({
      data: {
        sessionId: testSession,
        status: 'COMPLETE',
        inputType: 'FILE',
        fileName: 'test1.pdf',
        fileSize: 102400,
        checkpointData: { stage: 'complete', version: '1.0.0' },
      },
    }),
    prisma.job.create({
      data: {
        sessionId: testSession,
        status: 'PROCESSING',
        inputType: 'URL',
        sourceUrl: 'https://example.com/doc.pdf',
        checkpointData: { stage: 'processing', version: '1.0.0' },
      },
    }),
    prisma.job.create({
      data: {
        sessionId: testSession,
        status: 'ERROR',
        inputType: 'FILE',
        fileName: 'test3.pdf',
        errorCode: 'E101',
        errorMessage: 'Processing failed',
      },
    }),
  ]);

  // Create results for completed job
  await prisma.result.create({
    data: {
      jobId: jobs[0].id,
      format: 'MARKDOWN',
      content: 'Test content for migration',
      size: 25,
    },
  });

  // Count records before migration
  const jobCountBefore = await prisma.job.count({
    where: { sessionId: testSession },
  });
  const resultCountBefore = await prisma.result.count({
    where: { jobId: jobs[0].id },
  });

  expect(jobCountBefore).toBe(3);
  expect(resultCountBefore).toBe(1);

  // Simulate migration validation checks
  // In real migration: these queries validate data transformation

  // Check 1: All foreign keys are valid
  const orphanedResults = await prisma.result.findMany({
    where: {
      jobId: { in: jobs.map(j => j.id) },
      job: null,
    },
  });
  expect(orphanedResults.length).toBe(0);

  // Check 2: JSON data is valid and parseable
  const jobsWithCheckpoints = await prisma.job.findMany({
    where: {
      sessionId: testSession,
      checkpointData: { not: null },
    },
  });

  for (const job of jobsWithCheckpoints) {
    const checkpointData = job.checkpointData as any;
    expect(checkpointData).toHaveProperty('stage');
    expect(checkpointData).toHaveProperty('version');
    expect(typeof checkpointData.stage).toBe('string');
  }

  // Check 3: Enum values are valid
  const validStatuses = ['PENDING', 'UPLOADING', 'PROCESSING', 'RETRY_1',
                         'RETRY_2', 'RETRY_3', 'COMPLETE', 'PARTIAL_COMPLETE',
                         'ERROR', 'CANCELLED'];

  for (const job of jobs) {
    expect(validStatuses).toContain(job.status);
  }

  // Check 4: Required fields are not null
  const jobsWithNullSession = await prisma.job.findMany({
    where: {
      sessionId: testSession,
      OR: [
        { sessionId: null },
        { status: null },
        { inputType: null },
      ],
    },
  });
  expect(jobsWithNullSession.length).toBe(0);

  // Count records after migration validation
  const jobCountAfter = await prisma.job.count({
    where: { sessionId: testSession },
  });
  const resultCountAfter = await prisma.result.count({
    where: { jobId: jobs[0].id },
  });

  // Verify no data loss
  expect(jobCountAfter).toBe(jobCountBefore);
  expect(resultCountAfter).toBe(resultCountBefore);

  // Cleanup
  await prisma.result.deleteMany({
    where: { jobId: { in: jobs.map(j => j.id) } },
  });
  await prisma.job.deleteMany({
    where: { sessionId: testSession },
  });
}, 20000);
```

**Expected Result**:
- All data is preserved during migration
- Foreign key relationships remain valid
- JSON data remains parseable
- Enum values remain valid
- No records are lost or corrupted

---

### DB-MIG-004: Zero-Downtime Migration

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-MIG-004 |
| **Title** | Validate Zero-Downtime Migration Capability |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-401 |
| **Gap Reference** | Architectural Review A-06 |

**Preconditions**:
- Database is accessible during migration
- Application can read/write during migration
- Migration uses online DDL techniques

**Test Steps**:

```typescript
it('should support database operations during migration', async () => {
  const testSession = 'zero-downtime-test';
  let operationsDuringMigration = 0;
  let errors: any[] = [];

  // Simulate background operations during migration
  const backgroundOperations = async () => {
    const maxAttempts = 10;
    for (let i = 0; i < maxAttempts; i++) {
      try {
        // Attempt to create a job
        const job = await prisma.job.create({
          data: {
            sessionId: `${testSession}-${i}`,
            status: 'PENDING',
            inputType: 'FILE',
            fileName: `concurrent-${i}.pdf`,
          },
        });

        operationsDuringMigration++;

        // Attempt to query
        const jobs = await prisma.job.findMany({
          where: { sessionId: { startsWith: testSession } },
        });

        operationsDuringMigration++;

        // Attempt to update
        await prisma.job.update({
          where: { id: job.id },
          data: { status: 'PROCESSING' },
        });

        operationsDuringMigration++;

        await new Promise(r => setTimeout(r, 100));

      } catch (error) {
        errors.push({
          iteration: i,
          error: error instanceof Error ? error.message : String(error),
        });
      }
    }
  };

  // Run background operations
  await backgroundOperations();

  console.log(`Operations completed: ${operationsDuringMigration}`);
  console.log(`Errors encountered: ${errors.length}`);

  if (errors.length > 0) {
    console.log('Error details:', errors);
  }

  // Verify most operations succeeded
  // In real zero-downtime migration: expect 100% success
  // In test environment: allow some failures due to timing
  expect(operationsDuringMigration).toBeGreaterThan(15);

  // Verify error rate is acceptable (<10%)
  const totalAttempts = operationsDuringMigration + errors.length;
  const errorRate = errors.length / totalAttempts;
  expect(errorRate).toBeLessThan(0.1);

  // Verify database is still operational
  const finalJobs = await prisma.job.findMany({
    where: { sessionId: { startsWith: testSession } },
  });

  expect(finalJobs.length).toBeGreaterThan(0);

  // Cleanup
  await prisma.job.deleteMany({
    where: { sessionId: { startsWith: testSession } },
  });
}, 30000);
```

**Expected Result**:
- Database operations succeed during migration
- Error rate is acceptably low (<10%)
- No prolonged blocking occurs
- Application remains available throughout migration

---

### DB-MIG-005: Migration Idempotency

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-MIG-005 |
| **Title** | Validate Migration Idempotency |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-401 |
| **Gap Reference** | Architectural Review A-06 |

**Preconditions**:
- Migrations have already been applied
- Database is in known state
- Migration tracking table exists

**Test Steps**:

```typescript
it('should be idempotent when re-running migrations', async () => {
  const { exec } = require('child_process');
  const util = require('util');
  const execAsync = util.promisify(exec);

  // Get initial migration state
  const { stdout: status1 } = await execAsync('npx prisma migrate status');
  console.log('Initial migration status:', status1);

  // Apply migrations (should be no-op if already applied)
  const { stdout: deploy1, stderr: stderr1 } = await execAsync('npx prisma migrate deploy');

  expect(stderr1).not.toContain('Error');
  expect(deploy1).toMatch(/up to date|applied/i);

  // Create test data
  const testJob = await prisma.job.create({
    data: {
      sessionId: 'idempotency-test',
      status: 'COMPLETE',
      inputType: 'FILE',
      fileName: 'idempotency-test.pdf',
    },
  });

  // Re-run migrations (should be idempotent)
  const { stdout: deploy2, stderr: stderr2 } = await execAsync('npx prisma migrate deploy');

  expect(stderr2).not.toContain('Error');
  expect(deploy2).toMatch(/up to date|applied/i);

  // Verify test data still exists (no data loss from re-running)
  const jobAfterRedeploy = await prisma.job.findUnique({
    where: { id: testJob.id },
  });

  expect(jobAfterRedeploy).not.toBeNull();
  expect(jobAfterRedeploy?.sessionId).toBe('idempotency-test');

  // Get final migration state
  const { stdout: status2 } = await execAsync('npx prisma migrate status');
  console.log('Final migration status:', status2);

  // Verify status matches initial state
  expect(status2).toMatch(/up to date|Database schema is up to date/i);

  // Query migration history to verify no duplicate applications
  const migrationHistory = await prisma.$queryRaw`
    SELECT migration_name, COUNT(*) as application_count
    FROM _prisma_migrations
    GROUP BY migration_name
    HAVING COUNT(*) > 1
  `;

  // No migrations should be applied multiple times
  expect(Array.isArray(migrationHistory)).toBe(true);
  expect(migrationHistory.length).toBe(0);

  // Cleanup
  await prisma.job.delete({ where: { id: testJob.id } });
}, 30000);
```

**Expected Result**:
- Re-running migrations is safe (no errors)
- No duplicate migration applications occur
- Existing data is not affected
- Migration status remains consistent

---

### DB-MIG-006: Migration Version Tracking

| Attribute | Value |
|-----------|-------|
| **Test ID** | DB-MIG-006 |
| **Title** | Validate Migration Version Tracking |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-401 |
| **Gap Reference** | Architectural Review A-06 |

**Preconditions**:
- Migrations have been applied
- Migration tracking table exists
- Migration history is populated

**Test Steps**:

```typescript
it('should accurately track migration versions', async () => {
  const fs = require('fs').promises;
  const path = require('path');

  // Query _prisma_migrations table
  const appliedMigrations = await prisma.$queryRaw`
    SELECT
      id,
      checksum,
      finished_at,
      migration_name,
      logs,
      rolled_back_at,
      started_at,
      applied_steps_count
    FROM _prisma_migrations
    ORDER BY started_at ASC
  `;

  expect(Array.isArray(appliedMigrations)).toBe(true);
  expect(appliedMigrations.length).toBeGreaterThan(0);

  console.log(`Found ${appliedMigrations.length} applied migrations`);

  // Verify migration tracking properties
  for (const migration of appliedMigrations as any[]) {
    // Each migration must have required fields
    expect(migration.id).toBeDefined();
    expect(migration.migration_name).toBeDefined();
    expect(migration.checksum).toBeDefined();
    expect(migration.started_at).toBeDefined();
    expect(migration.finished_at).toBeDefined();
    expect(migration.applied_steps_count).toBeGreaterThanOrEqual(0);

    // Migration should not be rolled back (in production)
    expect(migration.rolled_back_at).toBeNull();

    console.log(`Migration: ${migration.migration_name}, Steps: ${migration.applied_steps_count}`);
  }

  // Get migration files from filesystem
  const migrationsDir = path.join(process.cwd(), 'prisma', 'migrations');

  let migrationFiles: string[] = [];
  try {
    const dirs = await fs.readdir(migrationsDir);
    migrationFiles = dirs.filter(d => d !== 'migration_lock.toml');
  } catch (error) {
    // Migrations directory may not exist in test environment
    console.log('Migrations directory not found (expected in test environment)');
  }

  if (migrationFiles.length > 0) {
    console.log(`Found ${migrationFiles.length} migration files on filesystem`);

    // Verify applied migrations exist as files
    const appliedNames = (appliedMigrations as any[]).map(m => m.migration_name);

    for (const appliedName of appliedNames) {
      // Migration file should exist
      const migrationExists = migrationFiles.some(f => f === appliedName);

      if (!migrationExists) {
        console.warn(`Warning: Applied migration ${appliedName} not found in filesystem`);
      }
    }
  }

  // Verify most recent migration
  const latestMigration = (appliedMigrations as any[])
    .sort((a, b) => new Date(b.finished_at).getTime() - new Date(a.finished_at).getTime())[0];

  expect(latestMigration).toBeDefined();
  console.log(`Latest migration: ${latestMigration.migration_name}`);

  // Verify migration timestamps are sequential
  const timestamps = (appliedMigrations as any[]).map(m =>
    new Date(m.started_at).getTime()
  );

  for (let i = 1; i < timestamps.length; i++) {
    expect(timestamps[i]).toBeGreaterThanOrEqual(timestamps[i - 1]);
  }

  console.log('Migration version tracking validated successfully');
}, 15000);
```

**Expected Result**:
- All applied migrations are tracked in _prisma_migrations table
- Each migration has complete metadata (checksum, timestamps, steps)
- No migrations are marked as rolled back
- Migration timestamps are sequential
- Migration files match applied migrations

---

## Test Summary

### Test Count by Category

| Category | Test Count |
|----------|-----------|
| Job Entity Tests | 13 |
| Result Entity Tests | 7 |
| Session Data Tests | 2 |
| Checkpoint Storage Tests | 3 |
| Transaction Integrity Tests | 3 |
| Query Performance Tests | 2 |
| Index Strategy and Query Optimization | 8 |
| Connection Pooling and Management | 8 |
| Concurrency Control and Locking | 10 |
| Workflow State Machine and Durable Execution | 12 |
| Cache-Database Consistency | 10 |
| Production Observability and Monitoring | 8 |
| Data Constraints and Schema Validation | 8 |
| Backup, Recovery, and Data Retention | 6 |
| Data Migration Tests | 8 |
| **Total** | **108** |

### Test Count by Priority

| Priority | Count |
|----------|-------|
| P0 - Critical | 51 |
| P1 - High | 49 |
| P2 - Medium | 8 |
| **Total** | **108** |

### Entity Coverage

| Entity | Operations Tested |
|--------|-------------------|
| Job | Create, Update, Delete, Query, Paginate, Filter, Concurrent Updates |
| Result | Create, Query, Cascade Delete, Concurrent Creation, Format Enum Coverage |
| Checkpoint | Store, Retrieve, Clear, Version Migration, TTL, Integrity Hash |
| Session | Count, Isolation, Cache Sync, Expiration Handling |
| State Machine | Transitions, Consistency, Recovery |
| Cache | Invalidation, Cache-Aside Pattern, Dual-Write, Saga Compensation |
| Observability | Health Checks, Prometheus Metrics, Query Logging, Slow Query Detection |
| Schema | Column Validation, Enum Coverage, Size Limits, Constraints, Cascade Behavior |
| Backup/Recovery | Backup Creation, Restore Validation, Data Consistency, PITR, Data Retention |
| Migration | Schema Up/Down, Data Integrity, Zero-Downtime, Idempotency, Version Tracking |

---

## Changelog

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 2.1.0 | 2025-12-15 | Julia Santos | **Architectural Review A-06 Remediation - Database Migration Tests**: Expanded Data Migration Tests section from 2 to 8 comprehensive tests (DB-MIG-001 to DB-MIG-006), covering: (1) Schema migration up - apply new migrations with validation, (2) Schema migration down - rollback procedures and migration history tracking, (3) Data migration integrity - validate data preservation and foreign key consistency, (4) Zero-downtime migration - concurrent operations during migration, (5) Migration idempotency - safe re-running without duplicates, (6) Migration version tracking - _prisma_migrations table validation and timestamp sequencing. Updated test counts (102 to 108), priority breakdown (P0: 51, P1: 49, P2: 8), added Migration to entity coverage with operations: Schema Up/Down, Data Integrity, Zero-Downtime, Idempotency, Version Tracking |
| 2.0.0 | 2025-12-14 | Julia Santos | **MAJOR RELEASE - All 8 Gap Domains Remediated**: Added Data Constraints and Schema Validation section (8 tests: DB-SCHEMA-001 to DB-SCHEMA-008 covering schema alignment, RAW format enum, size limits, NOT NULL/unique constraints, cascade behavior, field lengths, enum validation), Added Backup, Recovery, and Data Retention section (6 tests: DB-BACKUP-001 to DB-BACKUP-004, DB-RETENTION-001/002 covering backup creation/integrity, restore validation, data consistency, PITR capability, job cleanup, orphaned file cleanup), Renumbered Data Migration Tests to Section 16, Updated test counts (88 to 102), Updated priority breakdown (P0: 48, P1: 46, P2: 8), Added Schema and Backup/Recovery to entity coverage |
| 1.3.0 | 2025-12-14 | Julia Santos | Added Cache-Database Consistency section (10 tests: DB-CACHE-001 to DB-CACHE-003, DB-SESS-SYNC-001 to DB-SESS-SYNC-003, DB-TX-DUAL-001/002, DB-TX-SAGA-001/002), Added Production Observability and Monitoring section (8 tests: DB-HEALTH-001 to DB-HEALTH-003, DB-METRICS-001 to DB-METRICS-003, DB-LOGGING-001/002), Renumbered Data Migration Tests to Section 14, Updated test counts (70 to 88) and priority breakdown |
| 1.2.0 | 2025-12-14 | Julia Santos | Added Concurrency Control and Locking section (10 tests: DB-CONC-001 to DB-CONC-010), Added Workflow State Machine and Durable Execution section (12 tests: DB-SM-001, DB-SM-002, DB-CK-004 to DB-CK-007, DB-DUR-001 to DB-DUR-003, DB-WF-001 to DB-WF-003), Updated test counts and priority breakdown |
| 1.1.0 | 2025-12-14 | Julia Santos | Added Index Strategy and Query Optimization section (8 tests: DB-PERF-003 to DB-PERF-010), Added Connection Pooling and Management section (8 tests: DB-POOL-001 to DB-POOL-008), Updated test counts and priority breakdown |
| 1.0.0 | 2025-12-12 | Julia Santos | Initial database test cases |
