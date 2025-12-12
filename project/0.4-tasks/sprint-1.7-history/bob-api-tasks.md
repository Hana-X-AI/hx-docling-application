# API Tasks: Sprint 1.7 - History View & Persistence

**Sprint**: 1.7 - History View & Persistence
**Author**: Bob Martin (@bob) - API Design SME
**Created**: 2025-12-12
**Status**: PENDING
**Dependencies**: Sprint 1.6 Complete

---

## Overview

This task file covers API-related deliverables for Sprint 1.7, including the History API, Jobs API, Cancel endpoint, and Resume endpoint for comprehensive job management.

---

## Tasks

### BOB-1.7-001: Implement History API Endpoint

| Field | Value |
|-------|-------|
| **Task ID** | BOB-1.7-001 |
| **Title** | Create GET /api/v1/history endpoint with pagination |
| **Priority** | P1 (Critical) |
| **Effort** | 35 minutes |
| **Dependencies** | BOB-1.2-002 (Validation Middleware), Trinity's Prisma schema |

#### Description

Implement the history endpoint that returns paginated job history for the current session. The endpoint supports sorting by createdAt or status with configurable page size.

#### Acceptance Criteria

- [ ] Endpoint accessible at `GET /api/v1/history`
- [ ] Accepts query parameters: page, pageSize, sortBy, sortOrder
- [ ] Default: page=1, pageSize=20, sortBy=createdAt, sortOrder=desc
- [ ] Max pageSize enforced at 50
- [ ] Filters jobs by session ID from cookie
- [ ] Returns jobs without full result content (summary only)
- [ ] Returns pagination metadata (totalPages, totalCount, hasMore)
- [ ] Supports status-based sorting with defined sort order
- [ ] Returns E801 error for invalid pagination parameters

#### Technical Notes

```typescript
// File: src/app/api/v1/history/route.ts

// Query Schema
const historyQuerySchema = z.object({
  page: z.coerce.number().int().min(1).default(1),
  pageSize: z.coerce.number().int().min(1).max(50).default(20),
  sortBy: z.enum(['createdAt', 'status']).default('createdAt'),
  sortOrder: z.enum(['asc', 'desc']).default('desc'),
});

// Status Sort Order (for sortBy=status)
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

// Response Structure
interface HistoryResponse {
  data: {
    jobs: JobSummary[];
    pagination: {
      page: number;
      pageSize: number;
      totalPages: number;
      totalCount: number;
      hasMore: boolean;
    };
  };
}

interface JobSummary {
  id: string;
  status: JobStatus;
  inputType: 'FILE' | 'URL';
  fileName?: string;
  url?: string;
  fileSize?: number;
  createdAt: string;
  completedAt?: string;
  formats: string[]; // Available export formats
}
```

**Prisma Query**:
```typescript
const jobs = await prisma.job.findMany({
  where: { sessionId },
  select: {
    id: true,
    status: true,
    inputType: true,
    fileName: true,
    url: true,
    fileSize: true,
    createdAt: true,
    completedAt: true,
    results: { select: { format: true } },
  },
  orderBy: sortBy === 'createdAt'
    ? { createdAt: sortOrder }
    : undefined, // Status sorting requires post-query sort
  skip: (page - 1) * pageSize,
  take: pageSize,
});
```

#### Deliverables

- `src/app/api/v1/history/route.ts` - History endpoint
- `src/lib/jobs/queries.ts` - Job query utilities

---

### BOB-1.7-002: Implement Job Detail API Endpoint

| Field | Value |
|-------|-------|
| **Task ID** | BOB-1.7-002 |
| **Title** | Create GET /api/v1/jobs/{id} endpoint |
| **Priority** | P1 (Critical) |
| **Effort** | 20 minutes |
| **Dependencies** | BOB-1.5a-003 (Job Access Validation) |

#### Description

Implement the job detail endpoint that returns full job information including all result content for a specific job ID.

#### Acceptance Criteria

- [ ] Endpoint accessible at `GET /api/v1/jobs/{id}`
- [ ] Validates job belongs to current session
- [ ] Returns full job details with all results
- [ ] Returns 404 with E501 if job not found or unauthorized
- [ ] Includes all result formats (markdown, html, json)
- [ ] Includes result content and sizes

#### Technical Notes

```typescript
// File: src/app/api/v1/jobs/[id]/route.ts

// Response Structure
interface JobDetailResponse {
  data: {
    id: string;
    sessionId: string;
    status: JobStatus;
    inputType: 'FILE' | 'URL';
    fileName?: string;
    url?: string;
    fileSize?: number;
    mimeType?: string;
    createdAt: string;
    completedAt?: string;
    results: {
      format: 'MARKDOWN' | 'HTML' | 'JSON';
      content: string;
      size: number;
    }[];
    checkpoint?: {
      stage: string;
      data: unknown;
    };
  };
}
```

#### Deliverables

- `src/app/api/v1/jobs/[id]/route.ts` - Job detail endpoint

---

### BOB-1.7-003: Implement Cancel API Endpoint

| Field | Value |
|-------|-------|
| **Task ID** | BOB-1.7-003 |
| **Title** | Create POST /api/v1/jobs/{id}/cancel endpoint |
| **Priority** | P0 (Critical - FR-406) |
| **Effort** | 45 minutes |
| **Dependencies** | James's MCP client (AbortController), William's SSE manager |

#### Description

Implement the job cancellation endpoint per FR-406. This endpoint must abort in-progress MCP operations, clean up partial results based on configuration, update job status to CANCELLED, and emit SSE cancelled event.

#### Acceptance Criteria

- [ ] Endpoint accessible at `POST /api/v1/jobs/{id}/cancel`
- [ ] Accepts optional `preservePartialResults` in body (default: false)
- [ ] Validates job belongs to current session
- [ ] Validates job is in cancellable state (PENDING, UPLOADING, PROCESSING, RETRY_*)
- [ ] Aborts MCP operation via AbortController
- [ ] Updates job status to CANCELLED
- [ ] Cleans up partial results if not preserved
- [ ] Emits `cancelled` SSE event
- [ ] Returns 200 OK with cancellation details
- [ ] Returns 404 with E501 if job not found
- [ ] Returns 409 with E702 if job not cancellable

#### Technical Notes

```typescript
// File: src/app/api/v1/jobs/[id]/cancel/route.ts

// Request Schema
const cancelSchema = z.object({
  preservePartialResults: z.boolean().default(false),
});

// Response Structure
interface CancelResponse {
  data: {
    jobId: string;
    status: 'CANCELLED';
    cancelledAt: string;
    partialResultsPreserved: boolean;
    message: string;
  };
}

// Cancellable States
const CANCELLABLE_STATES: JobStatus[] = [
  'PENDING',
  'UPLOADING',
  'PROCESSING',
  'RETRY_1',
  'RETRY_2',
  'RETRY_3',
];

// Cancellation Logic
async function cancelJob(
  jobId: string,
  preservePartialResults: boolean
): Promise<CancelResponse['data']> {
  const job = await prisma.job.findUnique({ where: { id: jobId } });

  if (!job) {
    throw new AppError('E501', 'Job not found', 404);
  }

  if (!CANCELLABLE_STATES.includes(job.status)) {
    throw new AppError('E702', 'Job not in cancellable state', 409);
  }

  // Abort MCP operation if processing
  if (['PROCESSING', 'RETRY_1', 'RETRY_2', 'RETRY_3'].includes(job.status)) {
    await mcpClient.abort(jobId);
  }

  // Update job status
  const cancelledAt = new Date().toISOString();
  await prisma.job.update({
    where: { id: jobId },
    data: { status: 'CANCELLED', cancelledAt },
  });

  // Clean up if not preserving
  if (!preservePartialResults) {
    await cleanupPartialResults(jobId);
  }

  // Emit SSE event
  await publishEvent(jobId, 'cancelled', {
    jobId,
    cancelledAt,
    partialResultsPreserved: preservePartialResults,
  });

  return {
    jobId,
    status: 'CANCELLED',
    cancelledAt,
    partialResultsPreserved: preservePartialResults,
    message: 'Job cancelled successfully',
  };
}
```

#### Deliverables

- `src/app/api/v1/jobs/[id]/cancel/route.ts` - Cancel endpoint
- `src/lib/jobs/cancel.ts` - Cancellation logic

---

### BOB-1.7-004: Implement Resume API Endpoint

| Field | Value |
|-------|-------|
| **Task ID** | BOB-1.7-004 |
| **Title** | Create POST /api/v1/jobs/{id}/resume endpoint |
| **Priority** | P1 (Critical - Spec Section 5.5) |
| **Effort** | 30 minutes |
| **Dependencies** | William's checkpoint manager (Sprint 1.5b) |

#### Description

Implement the job resume endpoint per Specification Section 5.5. This endpoint allows resuming a failed job from its last checkpoint.

#### Acceptance Criteria

- [ ] Endpoint accessible at `POST /api/v1/jobs/{id}/resume`
- [ ] Validates job belongs to current session
- [ ] Validates job has saved checkpoint
- [ ] Validates job is in resumable state (ERROR, CANCELLED with partial)
- [ ] Loads checkpoint and resumes processing
- [ ] Transitions job status to PROCESSING
- [ ] Returns 200 OK with new stream URL
- [ ] Returns 404 if job not found
- [ ] Returns 409 if job not resumable
- [ ] Returns 409 if no checkpoint available

#### Technical Notes

```typescript
// File: src/app/api/v1/jobs/[id]/resume/route.ts

// Response Structure
interface ResumeResponse {
  data: {
    jobId: string;
    status: 'PROCESSING';
    resumedFromStage: string;
    streamUrl: string;
  };
}

// Resumable States
const RESUMABLE_STATES: JobStatus[] = ['ERROR', 'CANCELLED'];

// Resume Logic
async function resumeJob(jobId: string): Promise<ResumeResponse['data']> {
  const job = await prisma.job.findUnique({
    where: { id: jobId },
    include: { checkpoint: true },
  });

  if (!job) {
    throw new AppError('E501', 'Job not found', 404);
  }

  if (!RESUMABLE_STATES.includes(job.status)) {
    throw new AppError('E703', 'Job not in resumable state', 409);
  }

  if (!job.checkpoint) {
    throw new AppError('E703', 'Job not in resumable state', 409);
  }

  // Check checkpoint TTL expiration (E704)
  const CHECKPOINT_TTL_HOURS = 24;
  const checkpointAge = Date.now() - new Date(job.checkpoint.savedAt).getTime();
  if (checkpointAge > CHECKPOINT_TTL_HOURS * 60 * 60 * 1000) {
    throw new AppError('E704', 'Checkpoint expired', 409);
  }

  // Update status and resume
  await prisma.job.update({
    where: { id: jobId },
    data: { status: 'PROCESSING' },
  });

  // Trigger async processing from checkpoint
  await resumeProcessing(jobId, job.checkpoint);

  return {
    jobId,
    status: 'PROCESSING',
    resumedFromStage: job.checkpoint.stage,
    streamUrl: `/api/v1/process/${jobId}/events`,
  };
}
```

#### Deliverables

- `src/app/api/v1/jobs/[id]/resume/route.ts` - Resume endpoint
- `src/lib/jobs/resume.ts` - Resume logic

---

### BOB-1.7-005: Implement Download API Endpoint

| Field | Value |
|-------|-------|
| **Task ID** | BOB-1.7-005 |
| **Title** | Create GET /api/v1/jobs/{id}/download endpoint |
| **Priority** | P1 (Critical) |
| **Effort** | 25 minutes |
| **Dependencies** | BOB-1.7-002 (Job Detail) |

#### Description

Implement the download endpoint that supports content negotiation for different export formats.

#### Acceptance Criteria

- [ ] Endpoint accessible at `GET /api/v1/jobs/{id}/download`
- [ ] Accepts `format` query parameter (markdown, html, json)
- [ ] Supports content negotiation via Accept header
- [ ] Returns JSON wrapper by default (Accept: application/json)
- [ ] Returns raw content for text/* Accept headers
- [ ] Sets appropriate Content-Disposition header for downloads
- [ ] Generates filename per convention: `{filename}_{timestamp}_{format}.{ext}`

#### Technical Notes

```typescript
// File: src/app/api/v1/jobs/[id]/download/route.ts

const downloadQuerySchema = z.object({
  format: z.enum(['markdown', 'html', 'json']).default('markdown'),
});

// Content negotiation
const CONTENT_TYPES: Record<string, string> = {
  markdown: 'text/markdown; charset=utf-8',
  html: 'text/html; charset=utf-8',
  json: 'application/json; charset=utf-8',
};

const FILE_EXTENSIONS: Record<string, string> = {
  markdown: 'md',
  html: 'html',
  json: 'json',
};

function generateDownloadFilename(
  originalName: string,
  format: string
): string {
  const baseName = originalName.replace(/\.[^/.]+$/, '');
  const timestamp = new Date().toISOString().replace(/[-:]/g, '').split('.')[0];
  const ext = FILE_EXTENSIONS[format];
  return `${baseName}_${timestamp}_${format}.${ext}`;
}
```

#### Deliverables

- `src/app/api/v1/jobs/[id]/download/route.ts` - Download endpoint

---

### BOB-1.7-006: Add Session Authorization to All Job Endpoints

| Field | Value |
|-------|-------|
| **Task ID** | BOB-1.7-006 |
| **Title** | Ensure session-based authorization on all job endpoints |
| **Priority** | P1 (Critical) |
| **Effort** | 15 minutes |
| **Dependencies** | BOB-1.5a-003 (Job Access Validation) |

#### Description

Review and ensure all job-related endpoints validate session ownership before returning data or performing actions.

#### Acceptance Criteria

- [ ] All /api/v1/jobs/* endpoints validate session
- [ ] History endpoint filters by session automatically
- [ ] Returns 404 (not 403) for unauthorized job access (security best practice)
- [ ] Consistent error handling across all job endpoints

#### Technical Notes

```typescript
// Reusable authorization wrapper
const withJobAuthorization = (
  handler: (req: NextRequest, job: Job) => Promise<Response>
) => {
  return async (req: NextRequest, { params }: { params: { id: string } }) => {
    const job = await validateJobAccess(params.id, req);
    if (!job) {
      return NextResponse.json(
        {
          error: {
            code: 'E501',
            message: 'Job not found',
            userMessage: 'The requested job could not be found.',
            retryable: false,
          },
        },
        { status: 404 }
      );
    }
    return handler(req, job);
  };
};
```

#### Deliverables

- Updated all job endpoints with authorization wrapper

---

### BOB-1.7-007: Write Unit Tests for History and Jobs APIs

| Field | Value |
|-------|-------|
| **Task ID** | BOB-1.7-007 |
| **Title** | Create comprehensive tests for history and job APIs |
| **Priority** | P1 (Critical) |
| **Effort** | 20 minutes |
| **Dependencies** | BOB-1.7-001 through BOB-1.7-006 |

#### Description

Write unit tests covering pagination, authorization, cancellation, and resume workflows.

#### Acceptance Criteria

- [ ] Tests for history pagination (page, pageSize validation)
- [ ] Tests for history sorting (createdAt, status)
- [ ] Tests for session filtering
- [ ] Tests for job detail retrieval
- [ ] Tests for cancel state transitions
- [ ] Tests for resume from checkpoint
- [ ] Tests for authorization failures (404 responses)

#### Technical Notes

```typescript
// File: src/app/api/v1/history/route.test.ts
describe('GET /api/v1/history', () => {
  it('should return paginated jobs for session', async () => {});
  it('should enforce max pageSize of 50', async () => {});
  it('should filter jobs by session ID', async () => {});
  it('should sort by status with defined order', async () => {});
});

// File: src/app/api/v1/jobs/[id]/cancel/route.test.ts
describe('POST /api/v1/jobs/{id}/cancel', () => {
  it('should cancel PROCESSING job', async () => {});
  it('should return 409 for COMPLETE job', async () => {});
  it('should preserve partial results when requested', async () => {});
});
```

#### Deliverables

- `src/app/api/v1/history/route.test.ts`
- `src/app/api/v1/jobs/[id]/route.test.ts`
- `src/app/api/v1/jobs/[id]/cancel/route.test.ts`
- `src/app/api/v1/jobs/[id]/resume/route.test.ts`
- `src/app/api/v1/jobs/[id]/download/route.test.ts`

---

## Sprint Summary

| Task ID | Title | Effort | Priority |
|---------|-------|--------|----------|
| BOB-1.7-001 | History API Endpoint | 35m | P1 |
| BOB-1.7-002 | Job Detail Endpoint | 20m | P1 |
| BOB-1.7-003 | Cancel Endpoint | 45m | P0 |
| BOB-1.7-004 | Resume Endpoint | 30m | P1 |
| BOB-1.7-005 | Download Endpoint | 25m | P1 |
| BOB-1.7-006 | Session Authorization | 15m | P1 |
| BOB-1.7-007 | Unit Tests | 20m | P1 |

**Total Effort**: 3 hours 10 minutes

---

## Dependencies Graph

```
Sprint 1.6 Complete
        |
        +-- William's Checkpoint Manager
        |
        +-- James's MCP Client (AbortController)
        |
        v
BOB-1.7-001 (History API)
        |
BOB-1.7-002 (Job Detail)
        |
        +----> BOB-1.7-005 (Download)
        |
BOB-1.7-003 (Cancel)
        |
BOB-1.7-004 (Resume)
        |
BOB-1.7-006 (Authorization)
        |
        v
BOB-1.7-007 (Tests)
```

---

## Error Codes Summary

| Code | HTTP Status | Description |
|------|-------------|-------------|
| E501 | 404 | Job not found (or unauthorized) |
| E702 | 409 | Job not cancellable |
| E703 | 409 | Job not resumable (status not in resumable states) |
| E704 | 409 | Checkpoint expired (TTL exceeded, default 24h) |
| E705 | 409 | Checkpoint corrupted (checksum mismatch) |
| E801 | 400 | Invalid pagination parameters |

### Resume Error Code Details

The resume endpoint (`POST /api/v1/jobs/{id}/resume`) returns three distinct checkpoint-related errors:

| Code | Condition | Sample Response |
|------|-----------|-----------------|
| E703 | Job status not in `RESUMABLE_STATES` or checkpoint is null | `{ "error": { "code": "E703", "message": "Job not in resumable state" } }` |
| E704 | Checkpoint exists but `savedAt + CHECKPOINT_TTL_HOURS` has passed | `{ "error": { "code": "E704", "message": "Checkpoint expired" } }` |
| E705 | Checkpoint exists but MD5 checksum validation failed | `{ "error": { "code": "E705", "message": "Checkpoint corrupted" } }` |

**Backend Validation Order**: E703 (status check) → E704 (expiry check) → E705 (integrity check)

---

## Coordination Notes

- **Trinity (@trinity)**: Prisma queries for history pagination
- **William (@william)**: Checkpoint manager integration for resume
- **James (@james)**: MCP abort functionality via AbortController
- **Ola (@ola)**: History UI will consume these APIs
- **Sophia (@sophia)**: Resume endpoint aligns with state machine transitions

---

## Changelog

| Version | Date | Change | Author |
|---------|------|--------|--------|
| v1.0.0 | 2025-12-12 | Initial task file creation | Bob Martin |
| v1.0.1 | 2025-12-12 | Fixed DEF-019: Error code semantics - E703 for null checkpoint, E704 for TTL expiration | Bob Martin |

---

**Version:** v1.0.1
**Last Updated:** 2025-12-12
