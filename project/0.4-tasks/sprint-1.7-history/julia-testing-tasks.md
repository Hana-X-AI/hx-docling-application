# Julia Santos - Testing & QA Tasks: Sprint 1.7 (History View & Persistence)

**Sprint**: 1.7 - History View & Persistence
**Role**: Testing & Quality Assurance Specialist
**Document Version**: 1.0.0
**Created**: 2025-12-12
**References**:
- Implementation Plan v1.2.0 Section 4.7
- Detailed Specification v1.2.0 Section 10.4

---

## Sprint 1.7 Testing Objectives

Implement incremental unit tests for the history view, pagination, job detail, and the critical cancel/resume endpoints. Focus on state machine transitions and session authorization.

---

## Tasks

### JUL-1.7-001: Write Integration Tests for History API

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.7-001 |
| **Title** | Write Integration Tests for History API Route |
| **Priority** | P0 (Critical) |
| **Effort** | 1.25 hours |
| **Dependencies** | NEO-1.7-005 (History API created) |

#### Description

Write comprehensive integration tests for the history API route covering pagination, session filtering, and sorting.

#### Acceptance Criteria

- [ ] Test returns jobs filtered by session ID
- [ ] Test pagination default (20 items)
- [ ] Test pagination max (50 items)
- [ ] Test pagination page parameter
- [ ] Test sorted by createdAt DESC
- [ ] Test returns 401 for missing session (no authentication)
- [ ] Test returns empty array for different session (no jobs visible)
- [ ] Test empty result handling
- [ ] Test includes job status and metadata
- [ ] Coverage >= 90% for history route

#### Technical Notes

```typescript
// src/app/api/v1/history/__tests__/route.test.ts
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import { GET } from '../route';
import { prisma } from '@/lib/db/prisma';

describe('GET /api/v1/history', () => {
  const testSessionId = 'test-session-123';

  beforeEach(async () => {
    // Create test jobs for session
    await prisma.job.createMany({
      data: Array.from({ length: 30 }, (_, i) => ({
        id: `job-${i}`,
        sessionId: testSessionId,
        status: i % 3 === 0 ? 'COMPLETED' : 'PROCESSING',
        fileName: `file-${i}.pdf`,
        createdAt: new Date(Date.now() - i * 60000), // Staggered times
      })),
    });

    // Create jobs for different session (should not be returned)
    await prisma.job.createMany({
      data: Array.from({ length: 5 }, (_, i) => ({
        id: `other-job-${i}`,
        sessionId: 'other-session',
        status: 'COMPLETED',
        fileName: `other-${i}.pdf`,
      })),
    });
  });

  afterEach(async () => {
    await prisma.job.deleteMany();
  });

  describe('Session Filtering', () => {
    it('returns only jobs for requesting session', async () => {
      const request = new Request('http://localhost/api/v1/history', {
        headers: { Cookie: `session=${testSessionId}` },
      });

      const response = await GET(request);
      const data = await response.json();

      expect(response.status).toBe(200);
      data.jobs.forEach((job: any) => {
        expect(job.sessionId).toBe(testSessionId);
      });
    });

    it('returns 401 when no session provided', async () => {
      const request = new Request('http://localhost/api/v1/history');
      // No session cookie provided

      const response = await GET(request);

      expect(response.status).toBe(401); // No authentication
    });

    it('returns empty array for wrong session (no jobs visible)', async () => {
      const request = new Request('http://localhost/api/v1/history', {
        headers: { Cookie: 'session=different-session' },
      });

      const response = await GET(request);
      const data = await response.json();

      expect(response.status).toBe(200);
      expect(data.jobs).toHaveLength(0); // No jobs belong to this session
    });
  });

  describe('Pagination', () => {
    it('defaults to 20 items per page', async () => {
      const request = new Request('http://localhost/api/v1/history', {
        headers: { Cookie: `session=${testSessionId}` },
      });

      const response = await GET(request);
      const data = await response.json();

      expect(data.jobs).toHaveLength(20);
      expect(data.pagination.perPage).toBe(20);
    });

    it('respects limit parameter up to 50', async () => {
      const request = new Request('http://localhost/api/v1/history?limit=30', {
        headers: { Cookie: `session=${testSessionId}` },
      });

      const response = await GET(request);
      const data = await response.json();

      expect(data.jobs).toHaveLength(30);
    });

    it('caps limit at 50', async () => {
      const request = new Request('http://localhost/api/v1/history?limit=100', {
        headers: { Cookie: `session=${testSessionId}` },
      });

      const response = await GET(request);
      const data = await response.json();

      expect(data.jobs.length).toBeLessThanOrEqual(50);
    });

    it('returns correct page', async () => {
      const request = new Request('http://localhost/api/v1/history?page=2&limit=10', {
        headers: { Cookie: `session=${testSessionId}` },
      });

      const response = await GET(request);
      const data = await response.json();

      expect(data.pagination.page).toBe(2);
      expect(data.jobs).toHaveLength(10);
    });

    it('includes total count and pages', async () => {
      const request = new Request('http://localhost/api/v1/history', {
        headers: { Cookie: `session=${testSessionId}` },
      });

      const response = await GET(request);
      const data = await response.json();

      expect(data.pagination.total).toBe(30);
      expect(data.pagination.totalPages).toBe(2); // 30 / 20 = 2
    });
  });

  describe('Sorting', () => {
    it('sorts by createdAt DESC by default', async () => {
      const request = new Request('http://localhost/api/v1/history', {
        headers: { Cookie: `session=${testSessionId}` },
      });

      const response = await GET(request);
      const data = await response.json();

      const dates = data.jobs.map((j: any) => new Date(j.createdAt).getTime());
      expect(dates).toEqual([...dates].sort((a, b) => b - a));
    });
  });

  describe('Empty Results', () => {
    it('returns empty array for new session', async () => {
      const request = new Request('http://localhost/api/v1/history', {
        headers: { Cookie: 'session=new-session-no-jobs' },
      });

      const response = await GET(request);
      const data = await response.json();

      expect(data.jobs).toEqual([]);
      expect(data.pagination.total).toBe(0);
    });
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| History API tests | `src/app/api/v1/history/__tests__/route.test.ts` |

---

### JUL-1.7-002: Write Integration Tests for Cancel Endpoint

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.7-002 |
| **Title** | Write Integration Tests for Job Cancel Endpoint |
| **Priority** | P0 (Critical) |
| **Effort** | 1.5 hours |
| **Dependencies** | NEO-1.7-010 (Cancel endpoint created) |

#### Description

Write comprehensive integration tests for the job cancellation endpoint (POST /api/v1/jobs/{id}/cancel) covering all state transitions and cleanup behavior.

#### Acceptance Criteria

- [ ] Test cancellation of PENDING job
- [ ] Test cancellation of PROCESSING job
- [ ] Test cancellation aborts MCP request
- [ ] Test SSE `cancelled` event sent
- [ ] Test job status updated to CANCELLED
- [ ] Test partial results preserved if configured
- [ ] Test partial results cleaned if configured
- [ ] Test cannot cancel COMPLETED job
- [ ] Test cannot cancel CANCELLED job
- [ ] Test returns 401 for missing session (no authentication)
- [ ] Test returns 404 for wrong session (job not visible to requester)
- [ ] Test returns 404 for non-existent job
- [ ] Coverage >= 95% for cancel endpoint

#### Technical Notes

```typescript
// src/app/api/v1/jobs/[id]/cancel/__tests__/route.test.ts
import { describe, it, expect, beforeEach, afterEach, vi } from 'vitest';
import { POST } from '../route';
import { prisma } from '@/lib/db/prisma';
import { mcpClient } from '@/lib/mcp/client';
import { sseManager } from '@/lib/sse/manager';

vi.mock('@/lib/mcp/client');
vi.mock('@/lib/sse/manager');

describe('POST /api/v1/jobs/{id}/cancel', () => {
  const testSessionId = 'test-session-123';
  const testJobId = 'job-to-cancel';

  beforeEach(async () => {
    vi.clearAllMocks();
  });

  afterEach(async () => {
    await prisma.job.deleteMany();
  });

  describe('Successful Cancellation', () => {
    it('cancels PENDING job', async () => {
      await prisma.job.create({
        data: {
          id: testJobId,
          sessionId: testSessionId,
          status: 'PENDING',
          fileName: 'test.pdf',
        },
      });

      const request = new Request(
        `http://localhost/api/v1/jobs/${testJobId}/cancel`,
        {
          method: 'POST',
          headers: { Cookie: `session=${testSessionId}` },
        }
      );

      const response = await POST(request, { params: { id: testJobId } });
      const data = await response.json();

      expect(response.status).toBe(200);
      expect(data.status).toBe('CANCELLED');

      const job = await prisma.job.findUnique({ where: { id: testJobId } });
      expect(job?.status).toBe('CANCELLED');
    });

    it('cancels PROCESSING job and aborts MCP request', async () => {
      await prisma.job.create({
        data: {
          id: testJobId,
          sessionId: testSessionId,
          status: 'PROCESSING',
          fileName: 'test.pdf',
        },
      });

      const request = new Request(
        `http://localhost/api/v1/jobs/${testJobId}/cancel`,
        {
          method: 'POST',
          headers: { Cookie: `session=${testSessionId}` },
        }
      );

      const response = await POST(request, { params: { id: testJobId } });

      expect(response.status).toBe(200);
      expect(mcpClient.abort).toHaveBeenCalled();
    });

    it('sends SSE cancelled event', async () => {
      await prisma.job.create({
        data: {
          id: testJobId,
          sessionId: testSessionId,
          status: 'PROCESSING',
          fileName: 'test.pdf',
        },
      });

      const request = new Request(
        `http://localhost/api/v1/jobs/${testJobId}/cancel`,
        {
          method: 'POST',
          headers: { Cookie: `session=${testSessionId}` },
        }
      );

      await POST(request, { params: { id: testJobId } });

      expect(sseManager.send).toHaveBeenCalledWith(
        testJobId,
        expect.objectContaining({ type: 'cancelled' })
      );
    });
  });

  describe('Partial Results', () => {
    it('preserves partial results when configured', async () => {
      await prisma.job.create({
        data: {
          id: testJobId,
          sessionId: testSessionId,
          status: 'PROCESSING',
          fileName: 'test.pdf',
        },
      });

      await prisma.result.create({
        data: {
          jobId: testJobId,
          format: 'markdown',
          content: '# Partial content',
        },
      });

      const request = new Request(
        `http://localhost/api/v1/jobs/${testJobId}/cancel?preservePartial=true`,
        {
          method: 'POST',
          headers: { Cookie: `session=${testSessionId}` },
        }
      );

      await POST(request, { params: { id: testJobId } });

      const results = await prisma.result.findMany({ where: { jobId: testJobId } });
      expect(results).toHaveLength(1);
    });

    it('cleans up partial results by default', async () => {
      await prisma.job.create({
        data: {
          id: testJobId,
          sessionId: testSessionId,
          status: 'PROCESSING',
          fileName: 'test.pdf',
        },
      });

      await prisma.result.create({
        data: {
          jobId: testJobId,
          format: 'markdown',
          content: '# Partial content',
        },
      });

      const request = new Request(
        `http://localhost/api/v1/jobs/${testJobId}/cancel`,
        {
          method: 'POST',
          headers: { Cookie: `session=${testSessionId}` },
        }
      );

      await POST(request, { params: { id: testJobId } });

      const results = await prisma.result.findMany({ where: { jobId: testJobId } });
      expect(results).toHaveLength(0);
    });
  });

  describe('Invalid Cancellation', () => {
    it('rejects cancellation of COMPLETED job', async () => {
      await prisma.job.create({
        data: {
          id: testJobId,
          sessionId: testSessionId,
          status: 'COMPLETED',
          fileName: 'test.pdf',
        },
      });

      const request = new Request(
        `http://localhost/api/v1/jobs/${testJobId}/cancel`,
        {
          method: 'POST',
          headers: { Cookie: `session=${testSessionId}` },
        }
      );

      const response = await POST(request, { params: { id: testJobId } });

      expect(response.status).toBe(400);
      const data = await response.json();
      expect(data.code).toBe('E400'); // Invalid state transition
    });

    it('rejects cancellation of already CANCELLED job', async () => {
      await prisma.job.create({
        data: {
          id: testJobId,
          sessionId: testSessionId,
          status: 'CANCELLED',
          fileName: 'test.pdf',
        },
      });

      const request = new Request(
        `http://localhost/api/v1/jobs/${testJobId}/cancel`,
        {
          method: 'POST',
          headers: { Cookie: `session=${testSessionId}` },
        }
      );

      const response = await POST(request, { params: { id: testJobId } });

      expect(response.status).toBe(400);
    });
  });

  describe('Authorization', () => {
    it('returns 401 when no session provided', async () => {
      await prisma.job.create({
        data: {
          id: testJobId,
          sessionId: testSessionId,
          status: 'PROCESSING',
          fileName: 'test.pdf',
        },
      });

      const request = new Request(
        `http://localhost/api/v1/jobs/${testJobId}/cancel`,
        {
          method: 'POST',
          // No session cookie
        }
      );

      const response = await POST(request, { params: { id: testJobId } });

      expect(response.status).toBe(401); // No authentication
    });

    it('returns 404 for wrong session (job not visible)', async () => {
      await prisma.job.create({
        data: {
          id: testJobId,
          sessionId: testSessionId,
          status: 'PROCESSING',
          fileName: 'test.pdf',
        },
      });

      const request = new Request(
        `http://localhost/api/v1/jobs/${testJobId}/cancel`,
        {
          method: 'POST',
          headers: { Cookie: 'session=different-session' },
        }
      );

      const response = await POST(request, { params: { id: testJobId } });

      expect(response.status).toBe(404); // Job not found for this session
    });

    it('returns 404 for non-existent job', async () => {
      const request = new Request(
        'http://localhost/api/v1/jobs/non-existent/cancel',
        {
          method: 'POST',
          headers: { Cookie: `session=${testSessionId}` },
        }
      );

      const response = await POST(request, { params: { id: 'non-existent' } });

      expect(response.status).toBe(404);
    });
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| Cancel endpoint tests | `src/app/api/v1/jobs/[id]/cancel/__tests__/route.test.ts` |

---

### JUL-1.7-003: Write Integration Tests for Resume Endpoint

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.7-003 |
| **Title** | Write Integration Tests for Job Resume Endpoint |
| **Priority** | P0 (Critical) |
| **Effort** | 1.25 hours |
| **Dependencies** | NEO-1.7-011 (Resume endpoint created) |

#### Description

Write comprehensive integration tests for the job resume endpoint (POST /api/v1/jobs/{id}/resume) covering checkpoint restoration and processing continuation.

#### Acceptance Criteria

- [ ] Test resume from checkpoint
- [ ] Test checkpoint loaded correctly
- [ ] Test processing continues from correct stage
- [ ] Test job status transitions to PROCESSING
- [ ] Test SSE connection re-established
- [ ] Test resume only available with checkpoint
- [ ] Test cannot resume COMPLETED job
- [ ] Test cannot resume CANCELLED job (without checkpoint)
- [ ] Test session authorization required
- [ ] Coverage >= 95% for resume endpoint

#### Technical Notes

```typescript
// src/app/api/v1/jobs/[id]/resume/__tests__/route.test.ts
import { describe, it, expect, beforeEach, afterEach, vi } from 'vitest';
import { POST } from '../route';
import { prisma } from '@/lib/db/prisma';
import { checkpointManager } from '@/lib/checkpoint/manager';

vi.mock('@/lib/checkpoint/manager');

describe('POST /api/v1/jobs/{id}/resume', () => {
  const testSessionId = 'test-session-123';
  const testJobId = 'job-to-resume';

  beforeEach(async () => {
    vi.clearAllMocks();
  });

  afterEach(async () => {
    await prisma.job.deleteMany();
    await prisma.checkpoint.deleteMany();
  });

  describe('Successful Resume', () => {
    it('resumes job from checkpoint', async () => {
      // Create job with checkpoint
      await prisma.job.create({
        data: {
          id: testJobId,
          sessionId: testSessionId,
          status: 'FAILED',
          fileName: 'test.pdf',
        },
      });

      await prisma.checkpoint.create({
        data: {
          jobId: testJobId,
          data: JSON.stringify({
            stage: 'conversion',
            percent: 50,
            result: { documentId: 'doc-123' },
            timestamp: new Date().toISOString(),
          }),
        },
      });

      vi.mocked(checkpointManager.restore).mockResolvedValue({
        stage: 'conversion',
        percent: 50,
        result: { documentId: 'doc-123' },
        timestamp: new Date().toISOString(),
      });

      const request = new Request(
        `http://localhost/api/v1/jobs/${testJobId}/resume`,
        {
          method: 'POST',
          headers: { Cookie: `session=${testSessionId}` },
        }
      );

      const response = await POST(request, { params: { id: testJobId } });
      const data = await response.json();

      expect(response.status).toBe(200);
      expect(data.status).toBe('PROCESSING');
      expect(data.resumedFrom).toBe('conversion');
    });

    it('loads checkpoint correctly', async () => {
      await prisma.job.create({
        data: {
          id: testJobId,
          sessionId: testSessionId,
          status: 'FAILED',
          fileName: 'test.pdf',
        },
      });

      const checkpointData = {
        stage: 'export',
        percent: 80,
        result: { markdown: '# Content' },
        timestamp: new Date().toISOString(),
      };

      await prisma.checkpoint.create({
        data: {
          jobId: testJobId,
          data: JSON.stringify(checkpointData),
        },
      });

      vi.mocked(checkpointManager.restore).mockResolvedValue(checkpointData);

      const request = new Request(
        `http://localhost/api/v1/jobs/${testJobId}/resume`,
        {
          method: 'POST',
          headers: { Cookie: `session=${testSessionId}` },
        }
      );

      await POST(request, { params: { id: testJobId } });

      expect(checkpointManager.restore).toHaveBeenCalledWith(testJobId);
    });

    it('continues processing from correct stage', async () => {
      await prisma.job.create({
        data: {
          id: testJobId,
          sessionId: testSessionId,
          status: 'FAILED',
          fileName: 'test.pdf',
        },
      });

      vi.mocked(checkpointManager.restore).mockResolvedValue({
        stage: 'export',
        percent: 80,
        result: { documentId: 'doc-123' },
        timestamp: new Date().toISOString(),
      });

      const request = new Request(
        `http://localhost/api/v1/jobs/${testJobId}/resume`,
        {
          method: 'POST',
          headers: { Cookie: `session=${testSessionId}` },
        }
      );

      const response = await POST(request, { params: { id: testJobId } });
      const data = await response.json();

      expect(data.resumedFrom).toBe('export');
      // Processing should skip earlier stages
    });
  });

  describe('Invalid Resume', () => {
    it('rejects resume without checkpoint', async () => {
      await prisma.job.create({
        data: {
          id: testJobId,
          sessionId: testSessionId,
          status: 'FAILED',
          fileName: 'test.pdf',
        },
      });

      vi.mocked(checkpointManager.restore).mockResolvedValue(null);

      const request = new Request(
        `http://localhost/api/v1/jobs/${testJobId}/resume`,
        {
          method: 'POST',
          headers: { Cookie: `session=${testSessionId}` },
        }
      );

      const response = await POST(request, { params: { id: testJobId } });

      expect(response.status).toBe(400);
      const data = await response.json();
      expect(data.code).toBe('E401'); // No checkpoint available
    });

    it('rejects resume of COMPLETED job', async () => {
      await prisma.job.create({
        data: {
          id: testJobId,
          sessionId: testSessionId,
          status: 'COMPLETED',
          fileName: 'test.pdf',
        },
      });

      const request = new Request(
        `http://localhost/api/v1/jobs/${testJobId}/resume`,
        {
          method: 'POST',
          headers: { Cookie: `session=${testSessionId}` },
        }
      );

      const response = await POST(request, { params: { id: testJobId } });

      expect(response.status).toBe(400);
      const data = await response.json();
      expect(data.message).toContain('already completed');
    });
  });

  describe('Authorization', () => {
    it('requires session authorization', async () => {
      await prisma.job.create({
        data: {
          id: testJobId,
          sessionId: testSessionId,
          status: 'FAILED',
          fileName: 'test.pdf',
        },
      });

      const request = new Request(
        `http://localhost/api/v1/jobs/${testJobId}/resume`,
        {
          method: 'POST',
          headers: { Cookie: 'session=different-session' },
        }
      );

      const response = await POST(request, { params: { id: testJobId } });

      expect(response.status).toBe(404);
    });
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| Resume endpoint tests | `src/app/api/v1/jobs/[id]/resume/__tests__/route.test.ts` |

---

### JUL-1.7-004: Write Component Tests for History View

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.7-004 |
| **Title** | Write Component Tests for History View Components |
| **Priority** | P1 (High) |
| **Effort** | 1.0 hour |
| **Dependencies** | NEO-1.7-002, NEO-1.7-003, NEO-1.7-004 |

#### Description

Write component tests for HistoryView, JobRow, and Pagination components.

#### Acceptance Criteria

- [ ] Test HistoryView renders table
- [ ] Test JobRow displays job info
- [ ] Test JobRow status badge colors
- [ ] Test action buttons (view, cancel, resume)
- [ ] Test Pagination page switching
- [ ] Test Pagination boundary cases
- [ ] Test empty state display
- [ ] Test loading state
- [ ] Coverage >= 90% for history components

#### Technical Notes

```typescript
// src/components/history/__tests__/HistoryView.test.tsx
import { render, screen, within } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, it, expect, vi } from 'vitest';
import { HistoryView } from '../HistoryView';

const mockJobs = [
  {
    id: 'job-1',
    fileName: 'document.pdf',
    status: 'COMPLETED',
    createdAt: '2025-12-12T10:00:00Z',
  },
  {
    id: 'job-2',
    fileName: 'report.docx',
    status: 'PROCESSING',
    createdAt: '2025-12-12T09:00:00Z',
  },
  {
    id: 'job-3',
    fileName: 'data.xlsx',
    status: 'FAILED',
    createdAt: '2025-12-12T08:00:00Z',
  },
];

describe('HistoryView', () => {
  it('renders job table', () => {
    render(
      <HistoryView
        jobs={mockJobs}
        pagination={{ page: 1, totalPages: 1, total: 3 }}
      />
    );

    expect(screen.getByRole('table')).toBeInTheDocument();
    expect(screen.getAllByRole('row')).toHaveLength(4); // Header + 3 jobs
  });

  it('displays job information', () => {
    render(
      <HistoryView
        jobs={mockJobs}
        pagination={{ page: 1, totalPages: 1, total: 3 }}
      />
    );

    expect(screen.getByText('document.pdf')).toBeInTheDocument();
    expect(screen.getByText('report.docx')).toBeInTheDocument();
    expect(screen.getByText('data.xlsx')).toBeInTheDocument();
  });

  it('shows status badges with correct colors', () => {
    render(
      <HistoryView
        jobs={mockJobs}
        pagination={{ page: 1, totalPages: 1, total: 3 }}
      />
    );

    const completedBadge = screen.getByText('COMPLETED');
    const processingBadge = screen.getByText('PROCESSING');
    const failedBadge = screen.getByText('FAILED');

    expect(completedBadge).toHaveClass('bg-green');
    expect(processingBadge).toHaveClass('bg-blue');
    expect(failedBadge).toHaveClass('bg-red');
  });

  it('shows empty state when no jobs', () => {
    render(
      <HistoryView
        jobs={[]}
        pagination={{ page: 1, totalPages: 0, total: 0 }}
      />
    );

    expect(screen.getByText(/no processing history/i)).toBeInTheDocument();
  });

  it('shows loading state', () => {
    render(<HistoryView jobs={[]} isLoading pagination={{ page: 1, totalPages: 0, total: 0 }} />);

    expect(screen.getByTestId('history-skeleton')).toBeInTheDocument();
  });
});

// src/components/history/__tests__/JobRow.test.tsx
describe('JobRow', () => {
  it('shows cancel button for PROCESSING jobs', () => {
    render(<JobRow job={{ ...mockJobs[1] }} />);
    expect(screen.getByRole('button', { name: /cancel/i })).toBeInTheDocument();
  });

  it('shows resume button for FAILED jobs with checkpoint', () => {
    render(<JobRow job={{ ...mockJobs[2], hasCheckpoint: true }} />);
    expect(screen.getByRole('button', { name: /resume/i })).toBeInTheDocument();
  });

  it('hides resume button for FAILED jobs without checkpoint', () => {
    render(<JobRow job={{ ...mockJobs[2], hasCheckpoint: false }} />);
    expect(screen.queryByRole('button', { name: /resume/i })).not.toBeInTheDocument();
  });
});

// src/components/history/__tests__/Pagination.test.tsx
describe('Pagination', () => {
  it('renders page numbers', () => {
    render(
      <Pagination
        currentPage={1}
        totalPages={5}
        onPageChange={vi.fn()}
      />
    );

    expect(screen.getByText('1')).toBeInTheDocument();
    expect(screen.getByText('5')).toBeInTheDocument();
  });

  it('calls onPageChange when page clicked', async () => {
    const user = userEvent.setup();
    const onPageChange = vi.fn();

    render(
      <Pagination
        currentPage={1}
        totalPages={5}
        onPageChange={onPageChange}
      />
    );

    await user.click(screen.getByText('3'));
    expect(onPageChange).toHaveBeenCalledWith(3);
  });

  it('disables previous on first page', () => {
    render(
      <Pagination
        currentPage={1}
        totalPages={5}
        onPageChange={vi.fn()}
      />
    );

    expect(screen.getByRole('button', { name: /previous/i })).toBeDisabled();
  });

  it('disables next on last page', () => {
    render(
      <Pagination
        currentPage={5}
        totalPages={5}
        onPageChange={vi.fn()}
      />
    );

    expect(screen.getByRole('button', { name: /next/i })).toBeDisabled();
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| HistoryView tests | `src/components/history/__tests__/HistoryView.test.tsx` |
| JobRow tests | `src/components/history/__tests__/JobRow.test.tsx` |
| Pagination tests | `src/components/history/__tests__/Pagination.test.tsx` |

---

### JUL-1.7-005: Write Tests for useHistory Hook

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.7-005 |
| **Title** | Write Tests for useHistory Hook |
| **Priority** | P1 (High) |
| **Effort** | 0.5 hours |
| **Dependencies** | NEO-1.7-009 (useHistory hook created) |

#### Description

Write tests for the useHistory hook covering data fetching, pagination state management, and refetch functionality.

#### Acceptance Criteria

- [ ] Test initial data fetch
- [ ] Test pagination state management
- [ ] Test page change triggers refetch
- [ ] Test loading states
- [ ] Test error handling
- [ ] Test refetch function
- [ ] Coverage >= 90% for useHistory hook

#### Technical Notes

```typescript
// src/hooks/__tests__/useHistory.test.tsx
import { renderHook, waitFor } from '@testing-library/react';
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { useHistory } from '../useHistory';

describe('useHistory', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it('fetches history on mount', async () => {
    const { result } = renderHook(() => useHistory());

    await waitFor(() => {
      expect(result.current.isLoading).toBe(false);
    });

    expect(result.current.jobs).toBeDefined();
  });

  it('manages pagination state', async () => {
    const { result } = renderHook(() => useHistory());

    await waitFor(() => expect(result.current.isLoading).toBe(false));

    expect(result.current.pagination.page).toBe(1);

    result.current.setPage(2);

    await waitFor(() => {
      expect(result.current.pagination.page).toBe(2);
    });
  });

  it('refetches on page change', async () => {
    const fetchSpy = vi.spyOn(global, 'fetch');
    const { result } = renderHook(() => useHistory());

    await waitFor(() => expect(result.current.isLoading).toBe(false));

    const initialCallCount = fetchSpy.mock.calls.length;

    result.current.setPage(2);

    await waitFor(() => {
      expect(fetchSpy.mock.calls.length).toBeGreaterThan(initialCallCount);
    });
  });

  it('handles errors', async () => {
    vi.spyOn(global, 'fetch').mockRejectedValueOnce(new Error('Network error'));

    const { result } = renderHook(() => useHistory());

    await waitFor(() => {
      expect(result.current.error).toBeDefined();
    });
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| useHistory hook tests | `src/hooks/__tests__/useHistory.test.tsx` |

---

## Sprint 1.7 Testing Summary

| Metric | Target | Notes |
|--------|--------|-------|
| Total Tasks | 5 | |
| Total Effort | 5.5 hours | |
| Critical Tasks | 3 | History API, Cancel, Resume |
| High Priority Tasks | 2 | Components, Hook |

### Coverage Targets

| Component | Target Coverage |
|-----------|-----------------|
| `api/v1/history/route.ts` | >= 90% |
| `api/v1/jobs/[id]/cancel/route.ts` | >= 95% |
| `api/v1/jobs/[id]/resume/route.ts` | >= 95% |
| `HistoryView.tsx` | >= 90% |
| `useHistory.ts` | >= 90% |

### Quality Gates for Sprint 1.7

- [ ] Session authorization verified
- [ ] Pagination tested
- [ ] Cancel workflow complete
- [ ] Resume from checkpoint verified
- [ ] State transitions valid
- [ ] History components render correctly

---

## Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0.0 | 2025-12-12 | Julia Santos | Initial creation |
