# Neo Next.js Tasks: Sprint 1.7 - History View & Persistence

**Sprint**: 1.7 - History View & Persistence
**Duration**: ~3.5 hours
**Role**: Lead Developer
**Agent**: Neo (Next.js Senior Developer)
**Support**: Ola (@ola), Trinity (@trinity), Sophia (@sophia)
**Review**: Julia (@julia)

---

## Development Tools

### shadcn/ui MCP Server Reference

For component discovery and retrieval during history UI development, use the **hx-shadcn MCP Server**:

| Property | Value |
|----------|-------|
| **Service Descriptor** | `project/0.8-references/hx-shadcn-service-descriptor.md` |
| **Host** | hx-shadcn.hx.dev.local (192.168.10.229) |
| **Port** | 7423 |
| **SSE Endpoint** | `http://hx-shadcn.hx.dev.local:7423/sse` |
| **Protocol** | SSE + MCP (Model Context Protocol) |
| **Status** | OPERATIONAL |

**Available MCP Tools:**
- `list_components` - List all available shadcn/ui components for React framework
- `get_component` - Retrieve component source code with dependencies
- `get_component_demo` - Get usage examples for Table, Dialog, Badge, Button, Select, Tabs components
- `get_component_metadata` - Get props, installation instructions, and documentation

**Example - Get Table Component Demo:**
```json
{
  "method": "tools/call",
  "params": {
    "name": "get_component_demo",
    "arguments": {"componentName": "table"}
  }
}
```

**Sprint 1.7 Component Usage:** This sprint heavily uses Table, Dialog, Badge, Button, Select, and Tabs components for the history view and job detail modal. Use `get_component_demo` to retrieve usage patterns for table layouts and modal dialogs.

---

## Overview

Create the history page with paginated job listing, job detail view, re-download functionality, cancel and resume endpoints, and session-based job filtering. This sprint completes the persistence and job management features.

---

## Tasks

### NEO-1.7-001: Create History Page Layout

**Priority**: P0 (Critical)
**Effort**: 20 minutes
**Dependencies**: Sprint 1.6 Complete
**Reference**: FR-701

**Description**:
Create the history page layout with proper structure, filters, and navigation. This is a Server Component that loads the initial job list.

**Acceptance Criteria**:
- [ ] Page at `/history` route
- [ ] Header with page title
- [ ] Filter controls (status filter, date range - future enhancement)
- [ ] Main content area for job list
- [ ] Server Component for initial data fetch
- [ ] Metadata for SEO

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/app/history/page.tsx`
- `/home/agent0/hx-docling-ui/src/app/history/loading.tsx`

**Technical Notes**:
```typescript
// src/app/history/page.tsx
import { Suspense } from 'react';
import { HistoryView } from '@/components/history/HistoryView';
import { HistorySkeleton } from '@/components/history/HistorySkeleton';

export const metadata = {
  title: 'Processing History | HX Docling',
  description: 'View your document processing history',
};

interface HistoryPageProps {
  searchParams: {
    page?: string;
    pageSize?: string;
    sortBy?: string;
    sortOrder?: string;
  };
}

export default async function HistoryPage({ searchParams }: HistoryPageProps) {
  const page = parseInt(searchParams.page || '1', 10);
  const pageSize = Math.min(parseInt(searchParams.pageSize || '20', 10), 50);
  const sortBy = searchParams.sortBy || 'createdAt';
  const sortOrder = searchParams.sortOrder || 'desc';

  return (
    <main className="container mx-auto py-8 px-4">
      <div className="mb-8">
        <h1 className="text-3xl font-bold">Processing History</h1>
        <p className="text-muted-foreground mt-2">
          View and manage your document processing jobs
        </p>
      </div>

      <Suspense fallback={<HistorySkeleton />}>
        <HistoryView
          initialPage={page}
          initialPageSize={pageSize}
          initialSortBy={sortBy as 'createdAt' | 'status'}
          initialSortOrder={sortOrder as 'asc' | 'desc'}
        />
      </Suspense>
    </main>
  );
}
```

---

### NEO-1.7-002: Implement HistoryView with Table

**Priority**: P0 (Critical)
**Effort**: 35 minutes
**Dependencies**: NEO-1.7-001
**Reference**: FR-701

**Description**:
Implement the main history view component with a table displaying job information including file/URL, status, date, and actions.

**Acceptance Criteria**:
- [ ] Table with columns: File/URL, Status, Date, Actions
- [ ] Jobs filtered by current session
- [ ] Sorted by createdAt DESC (default)
- [ ] Status sort order defined per spec
- [ ] Empty state when no jobs
- [ ] 'use client' directive (Client Component for interactivity)

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/components/history/HistoryView.tsx`

**Technical Notes**:
```typescript
'use client';

import { useState, useEffect } from 'react';
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from '@/components/ui/table';
import { JobRow } from './JobRow';
import { Pagination } from './Pagination';
import { HistorySkeleton } from './HistorySkeleton';
import { useHistory } from '@/hooks/useHistory';

interface HistoryViewProps {
  initialPage: number;
  initialPageSize: number;
  initialSortBy: 'createdAt' | 'status';
  initialSortOrder: 'asc' | 'desc';
}

export function HistoryView({
  initialPage,
  initialPageSize,
  initialSortBy,
  initialSortOrder,
}: HistoryViewProps) {
  const {
    jobs,
    pagination,
    isLoading,
    error,
    setPage,
    setPageSize,
    refresh,
  } = useHistory({
    initialPage,
    initialPageSize,
    sortBy: initialSortBy,
    sortOrder: initialSortOrder,
  });

  if (isLoading) {
    return <HistorySkeleton />;
  }

  if (error) {
    return (
      <div className="text-center py-8 text-destructive">
        <p>Failed to load history: {error.message}</p>
        <button onClick={refresh} className="mt-4 underline">
          Try again
        </button>
      </div>
    );
  }

  if (!jobs || jobs.length === 0) {
    return (
      <div className="text-center py-12 border rounded-lg">
        <p className="text-muted-foreground">No processing history yet.</p>
        <p className="text-sm text-muted-foreground mt-2">
          Process a document to see it here.
        </p>
      </div>
    );
  }

  return (
    <div className="space-y-4">
      <div className="border rounded-lg">
        <Table>
          <TableHeader>
            <TableRow>
              <TableHead className="w-[40%]">Document</TableHead>
              <TableHead className="w-[15%]">Status</TableHead>
              <TableHead className="w-[20%]">Date</TableHead>
              <TableHead className="w-[25%] text-right">Actions</TableHead>
            </TableRow>
          </TableHeader>
          <TableBody>
            {jobs.map((job) => (
              <JobRow key={job.id} job={job} onRefresh={refresh} />
            ))}
          </TableBody>
        </Table>
      </div>

      <Pagination
        currentPage={pagination.page}
        totalPages={pagination.totalPages}
        totalCount={pagination.totalCount}
        pageSize={pagination.pageSize}
        onPageChange={setPage}
        onPageSizeChange={setPageSize}
      />
    </div>
  );
}
```

---

### NEO-1.7-003: Create JobRow Component

**Priority**: P0 (Critical)
**Effort**: 20 minutes
**Dependencies**: NEO-1.7-002

**Description**:
Create the job row component displaying individual job information with status badge and action buttons.

**Acceptance Criteria**:
- [ ] Display file name or URL (truncated)
- [ ] Status badge with appropriate colors
- [ ] Formatted date (relative or absolute)
- [ ] Action buttons: View, Download, Cancel (if applicable), Delete
- [ ] Row clickable for detail view
- [ ] Accessible

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/components/history/JobRow.tsx`

**Technical Notes**:
```typescript
'use client';

import { FileText, Link, Eye, Download, X, Trash2 } from 'lucide-react';
import { TableCell, TableRow } from '@/components/ui/table';
import { Button } from '@/components/ui/button';
import { StatusBadge } from './StatusBadge';
import { formatRelativeTime } from '@/lib/utils/date';
import type { JobSummary } from '@/types/job';

interface JobRowProps {
  job: JobSummary;
  onRefresh: () => void;
}

export function JobRow({ job, onRefresh }: JobRowProps) {
  const isFile = job.inputType === 'FILE';
  const displayName = isFile ? job.fileName : job.url;
  const canCancel = ['PENDING', 'UPLOADING', 'PROCESSING', 'RETRY_1', 'RETRY_2', 'RETRY_3'].includes(job.status);
  const canResume = job.checkpointData && ['ERROR', 'CANCELLED'].includes(job.status);

  const handleCancel = async () => {
    try {
      await fetch(`/api/v1/jobs/${job.id}/cancel`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ preservePartialResults: true }),
      });
      onRefresh();
    } catch (error) {
      console.error('Failed to cancel job:', error);
    }
  };

  const handleResume = async () => {
    try {
      const response = await fetch(`/api/v1/jobs/${job.id}/resume`, { method: 'POST' });
      if (!response.ok) {
        const data = await response.json();
        const errorCode = data.error?.code;
        // Provide user-friendly messages for each checkpoint error
        const errorMessages: Record<string, string> = {
          'E703': 'Cannot resume: no checkpoint saved for this job.',
          'E704': 'Cannot resume: checkpoint has expired. Please restart the job.',
          'E705': 'Cannot resume: checkpoint data is corrupted. Please restart the job.',
        };
        const message = errorMessages[errorCode] || data.error?.message || 'Failed to resume job';
        // Display error to user (assumes toast or similar notification system)
        console.error(message);
        return;
      }
      onRefresh();
    } catch (error) {
      console.error('Failed to resume job:', error);
    }
  };

  return (
    <TableRow className="cursor-pointer hover:bg-muted/50">
      <TableCell>
        <div className="flex items-center gap-3">
          {isFile ? (
            <FileText className="w-4 h-4 text-muted-foreground shrink-0" />
          ) : (
            <Link className="w-4 h-4 text-muted-foreground shrink-0" />
          )}
          <div className="min-w-0">
            <p className="font-medium truncate" title={displayName || ''}>
              {displayName || 'Unknown'}
            </p>
            {isFile && job.fileSize && (
              <p className="text-sm text-muted-foreground">
                {formatFileSize(job.fileSize)}
              </p>
            )}
          </div>
        </div>
      </TableCell>
      <TableCell>
        <StatusBadge status={job.status} />
      </TableCell>
      <TableCell>
        <span title={new Date(job.createdAt).toLocaleString()}>
          {formatRelativeTime(job.createdAt)}
        </span>
      </TableCell>
      <TableCell className="text-right">
        <div className="flex items-center justify-end gap-2">
          <Button
            variant="ghost"
            size="icon"
            aria-label="View details"
            asChild
          >
            <a href={`/history/${job.id}`}>
              <Eye className="w-4 h-4" />
            </a>
          </Button>

          {job.status === 'COMPLETE' && (
            <Button
              variant="ghost"
              size="icon"
              aria-label="Download results"
            >
              <Download className="w-4 h-4" />
            </Button>
          )}

          {canCancel && (
            <Button
              variant="ghost"
              size="icon"
              aria-label="Cancel job"
              onClick={handleCancel}
            >
              <X className="w-4 h-4" />
            </Button>
          )}

          {canResume && (
            <Button
              variant="outline"
              size="sm"
              onClick={handleResume}
            >
              Resume
            </Button>
          )}
        </div>
      </TableCell>
    </TableRow>
  );
}

function formatFileSize(bytes: number): string {
  if (bytes < 1024) return `${bytes} B`;
  if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`;
  return `${(bytes / (1024 * 1024)).toFixed(1)} MB`;
}
```

---

### NEO-1.7-004: Implement Pagination Component

**Priority**: P0 (Critical)
**Effort**: 20 minutes
**Dependencies**: NEO-1.7-002
**Reference**: FR-702

**Description**:
Create pagination component with page numbers, prev/next controls, and page size selector.

**Acceptance Criteria**:
- [ ] Page numbers with ellipsis for large page counts
- [ ] Previous/Next buttons
- [ ] Page size selector (10, 20, 50)
- [ ] Total count displayed
- [ ] URL updates with page number (shallow routing)
- [ ] Keyboard accessible

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/components/history/Pagination.tsx`

**Technical Notes**:
```typescript
'use client';

import { useRouter, useSearchParams } from 'next/navigation';
import { ChevronLeft, ChevronRight } from 'lucide-react';
import { Button } from '@/components/ui/button';
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from '@/components/ui/select';

interface PaginationProps {
  currentPage: number;
  totalPages: number;
  totalCount: number;
  pageSize: number;
  onPageChange: (page: number) => void;
  onPageSizeChange: (size: number) => void;
}

export function Pagination({
  currentPage,
  totalPages,
  totalCount,
  pageSize,
  onPageChange,
  onPageSizeChange,
}: PaginationProps) {
  const router = useRouter();
  const searchParams = useSearchParams();

  const updateUrl = (page: number, size: number) => {
    const params = new URLSearchParams(searchParams.toString());
    params.set('page', page.toString());
    params.set('pageSize', size.toString());
    router.push(`?${params.toString()}`, { scroll: false });
  };

  const handlePageChange = (page: number) => {
    onPageChange(page);
    updateUrl(page, pageSize);
  };

  const handlePageSizeChange = (size: string) => {
    const newSize = parseInt(size, 10);
    onPageSizeChange(newSize);
    updateUrl(1, newSize); // Reset to page 1 on size change
  };

  const startItem = (currentPage - 1) * pageSize + 1;
  const endItem = Math.min(currentPage * pageSize, totalCount);

  // Generate page numbers with ellipsis
  const getPageNumbers = () => {
    const pages: (number | string)[] = [];
    const showEllipsis = totalPages > 7;

    if (!showEllipsis) {
      for (let i = 1; i <= totalPages; i++) pages.push(i);
      return pages;
    }

    pages.push(1);
    if (currentPage > 3) pages.push('...');

    const start = Math.max(2, currentPage - 1);
    const end = Math.min(totalPages - 1, currentPage + 1);

    for (let i = start; i <= end; i++) pages.push(i);

    if (currentPage < totalPages - 2) pages.push('...');
    if (totalPages > 1) pages.push(totalPages);

    return pages;
  };

  return (
    <div className="flex items-center justify-between">
      <div className="text-sm text-muted-foreground">
        Showing {startItem}-{endItem} of {totalCount} results
      </div>

      <div className="flex items-center gap-4">
        <div className="flex items-center gap-2">
          <span className="text-sm text-muted-foreground">Per page:</span>
          <Select value={pageSize.toString()} onValueChange={handlePageSizeChange}>
            <SelectTrigger className="w-20">
              <SelectValue />
            </SelectTrigger>
            <SelectContent>
              <SelectItem value="10">10</SelectItem>
              <SelectItem value="20">20</SelectItem>
              <SelectItem value="50">50</SelectItem>
            </SelectContent>
          </Select>
        </div>

        <div className="flex items-center gap-1">
          <Button
            variant="outline"
            size="icon"
            onClick={() => handlePageChange(currentPage - 1)}
            disabled={currentPage === 1}
            aria-label="Previous page"
          >
            <ChevronLeft className="w-4 h-4" />
          </Button>

          {getPageNumbers().map((page, index) =>
            page === '...' ? (
              <span key={`ellipsis-${index}`} className="px-2">
                ...
              </span>
            ) : (
              <Button
                key={page}
                variant={page === currentPage ? 'default' : 'outline'}
                size="icon"
                onClick={() => handlePageChange(page as number)}
                aria-label={`Page ${page}`}
                aria-current={page === currentPage ? 'page' : undefined}
              >
                {page}
              </Button>
            )
          )}

          <Button
            variant="outline"
            size="icon"
            onClick={() => handlePageChange(currentPage + 1)}
            disabled={currentPage === totalPages}
            aria-label="Next page"
          >
            <ChevronRight className="w-4 h-4" />
          </Button>
        </div>
      </div>
    </div>
  );
}
```

---

### NEO-1.7-005: Create History API with Pagination

**Priority**: P0 (Critical)
**Effort**: 35 minutes
**Dependencies**: None
**Reference**: API Spec 4.5

**Description**:
Implement GET /api/v1/history endpoint with pagination, session filtering, and sorting.

**Acceptance Criteria**:
- [ ] Default page size 20, max 50
- [ ] Filter by session ID from cookie
- [ ] Sort by createdAt (default) or status
- [ ] Status sort order per specification
- [ ] Return paginated response with metadata
- [ ] Validation with Zod schema

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/app/api/v1/history/route.ts`

**Technical Notes**:
```typescript
import { NextRequest, NextResponse } from 'next/server';
import { z } from 'zod';
import { prisma } from '@/lib/db/prisma';
import { getSession } from '@/lib/redis/session';
import { JobStatus } from '@prisma/client';

const querySchema = z.object({
  page: z.coerce.number().int().min(1).default(1),
  pageSize: z.coerce.number().int().min(1).max(50).default(20),
  sortBy: z.enum(['createdAt', 'status']).default('createdAt'),
  sortOrder: z.enum(['asc', 'desc']).default('desc'),
});

// Status sort order per specification
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

export async function GET(request: NextRequest) {
  const requestId = request.headers.get('X-Request-ID') || crypto.randomUUID();

  try {
    // Get session
    const sessionId = await getSession(request);
    if (!sessionId) {
      return NextResponse.json(
        { error: { code: 'E502', message: 'Invalid session' } },
        { status: 401, headers: { 'X-Request-ID': requestId } }
      );
    }

    // Parse and validate query params
    const params = Object.fromEntries(request.nextUrl.searchParams);
    const validation = querySchema.safeParse(params);

    if (!validation.success) {
      return NextResponse.json(
        {
          error: {
            code: 'E801',
            message: 'Invalid pagination parameters',
            details: validation.error.flatten(),
          },
        },
        { status: 400, headers: { 'X-Request-ID': requestId } }
      );
    }

    const { page, pageSize, sortBy, sortOrder } = validation.data;
    const skip = (page - 1) * pageSize;

    // Query jobs
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
        orderBy: sortBy === 'status'
          ? undefined // Handle status sort in memory
          : { [sortBy]: sortOrder },
        skip: sortBy === 'status' ? 0 : skip,
        take: sortBy === 'status' ? undefined : pageSize,
      }),
      prisma.job.count({ where: { sessionId } }),
    ]);

    // Sort by status if needed
    let sortedJobs = jobs;
    if (sortBy === 'status') {
      sortedJobs = [...jobs].sort((a, b) => {
        const orderA = STATUS_SORT_ORDER[a.status];
        const orderB = STATUS_SORT_ORDER[b.status];
        return sortOrder === 'asc' ? orderA - orderB : orderB - orderA;
      });
      sortedJobs = sortedJobs.slice(skip, skip + pageSize);
    }

    const totalPages = Math.ceil(totalCount / pageSize);

    return NextResponse.json({
      jobs: sortedJobs.map((job) => ({
        id: job.id,
        status: job.status,
        inputType: job.inputType,
        fileName: job.fileName,
        fileSize: job.fileSize,
        url: job.url,
        createdAt: job.createdAt.toISOString(),
        completedAt: job.completedAt?.toISOString(),
        hasCheckpoint: !!job.checkpointData,
      })),
      pagination: {
        page,
        pageSize,
        totalPages,
        totalCount,
        hasMore: page < totalPages,
      },
    }, { headers: { 'X-Request-ID': requestId } });
  } catch (error) {
    console.error('History API error:', error);
    return NextResponse.json(
      { error: { code: 'E401', message: 'Database error', retryable: true } },
      { status: 500, headers: { 'X-Request-ID': requestId } }
    );
  }
}
```

---

### NEO-1.7-006: Create Job Detail API

**Priority**: P0 (Critical)
**Effort**: 20 minutes
**Dependencies**: None
**Reference**: API Spec 4.6

**Description**:
Implement GET /api/v1/jobs/{id} endpoint to retrieve full job details with results.

**Acceptance Criteria**:
- [ ] Return full job details including results
- [ ] Session authorization check (user sees only own jobs)
- [ ] Return 404 for missing or unauthorized jobs
- [ ] Include all result formats

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/app/api/v1/jobs/[id]/route.ts`

**Technical Notes**:
```typescript
import { NextRequest, NextResponse } from 'next/server';
import { prisma } from '@/lib/db/prisma';
import { getSession } from '@/lib/redis/session';

export async function GET(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const requestId = request.headers.get('X-Request-ID') || crypto.randomUUID();
  const { id } = params;

  try {
    const sessionId = await getSession(request);
    if (!sessionId) {
      return NextResponse.json(
        { error: { code: 'E502', message: 'Invalid session' } },
        { status: 401, headers: { 'X-Request-ID': requestId } }
      );
    }

    const job = await prisma.job.findUnique({
      where: { id },
      include: {
        results: {
          select: {
            format: true,
            content: true,
            size: true,
          },
        },
      },
    });

    // Not found or unauthorized
    if (!job || job.sessionId !== sessionId) {
      return NextResponse.json(
        { error: { code: 'E501', message: 'Job not found' } },
        { status: 404, headers: { 'X-Request-ID': requestId } }
      );
    }

    return NextResponse.json({
      id: job.id,
      sessionId: job.sessionId,
      status: job.status,
      inputType: job.inputType,
      fileName: job.fileName,
      fileSize: job.fileSize,
      mimeType: job.mimeType,
      url: job.url,
      createdAt: job.createdAt.toISOString(),
      completedAt: job.completedAt?.toISOString(),
      error: job.error,
      errorCode: job.errorCode,
      results: job.results.map((r) => ({
        format: r.format,
        content: r.content,
        size: r.size,
      })),
    }, { headers: { 'X-Request-ID': requestId } });
  } catch (error) {
    console.error('Job detail error:', error);
    return NextResponse.json(
      { error: { code: 'E401', message: 'Database error' } },
      { status: 500, headers: { 'X-Request-ID': requestId } }
    );
  }
}
```

---

### NEO-1.7-007: Implement JobDetail Modal

**Priority**: P1 (High)
**Effort**: 30 minutes
**Dependencies**: NEO-1.7-003, NEO-1.7-006
**Reference**: FR-703

**Description**:
Create the job detail modal/view showing full job information with all result formats and download options.

**Acceptance Criteria**:
- [ ] Modal or dedicated view
- [ ] Show all result formats
- [ ] Download buttons available
- [ ] Status and timestamps displayed
- [ ] Error details if applicable
- [ ] Accessible modal with focus trap

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/components/history/JobDetail.tsx`

**Technical Notes**:
```typescript
'use client';

import { useState, useEffect } from 'react';
import { X, Download, FileText, Link, Clock, AlertCircle } from 'lucide-react';
import {
  Dialog,
  DialogContent,
  DialogHeader,
  DialogTitle,
} from '@/components/ui/dialog';
import { Button } from '@/components/ui/button';
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';
import { StatusBadge } from './StatusBadge';
import { MarkdownView } from '@/components/results/MarkdownView';
import { formatRelativeTime } from '@/lib/utils/date';
import type { JobDetail } from '@/types/job';

interface JobDetailProps {
  jobId: string | null;
  onClose: () => void;
}

export function JobDetailModal({ jobId, onClose }: JobDetailProps) {
  const [job, setJob] = useState<JobDetail | null>(null);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    if (!jobId) return;

    setIsLoading(true);
    setError(null);

    fetch(`/api/v1/jobs/${jobId}`)
      .then((res) => {
        if (!res.ok) throw new Error('Failed to load job');
        return res.json();
      })
      .then((data) => setJob(data))
      .catch((err) => setError(err.message))
      .finally(() => setIsLoading(false));
  }, [jobId]);

  const isOpen = !!jobId;

  return (
    <Dialog open={isOpen} onOpenChange={() => onClose()}>
      <DialogContent className="max-w-4xl max-h-[90vh] overflow-auto">
        <DialogHeader>
          <DialogTitle className="flex items-center gap-2">
            {job?.inputType === 'FILE' ? (
              <FileText className="w-5 h-5" />
            ) : (
              <Link className="w-5 h-5" />
            )}
            {job?.fileName || job?.url || 'Job Details'}
          </DialogTitle>
        </DialogHeader>

        {isLoading && <div>Loading...</div>}

        {error && (
          <div className="p-4 bg-destructive/10 text-destructive rounded-lg">
            {error}
          </div>
        )}

        {job && !isLoading && (
          <div className="space-y-6">
            {/* Job metadata */}
            <div className="grid grid-cols-2 gap-4 text-sm">
              <div className="flex items-center gap-2">
                <StatusBadge status={job.status} />
              </div>
              <div className="flex items-center gap-2 text-muted-foreground">
                <Clock className="w-4 h-4" />
                {formatRelativeTime(job.createdAt)}
              </div>
            </div>

            {/* Error display */}
            {job.error && (
              <div className="p-4 bg-destructive/10 rounded-lg flex items-start gap-3">
                <AlertCircle className="w-5 h-5 text-destructive shrink-0" />
                <div>
                  <p className="font-medium text-destructive">
                    Error {job.errorCode}
                  </p>
                  <p className="text-sm text-muted-foreground">{job.error}</p>
                </div>
              </div>
            )}

            {/* Results tabs */}
            {job.results && job.results.length > 0 && (
              <Tabs defaultValue={job.results[0].format.toLowerCase()}>
                <TabsList>
                  {job.results.map((result) => (
                    <TabsTrigger
                      key={result.format}
                      value={result.format.toLowerCase()}
                    >
                      {result.format}
                    </TabsTrigger>
                  ))}
                </TabsList>
                {job.results.map((result) => (
                  <TabsContent
                    key={result.format}
                    value={result.format.toLowerCase()}
                  >
                    <div className="max-h-[400px] overflow-auto">
                      <MarkdownView content={result.content} />
                    </div>
                  </TabsContent>
                ))}
              </Tabs>
            )}

            {/* Download button */}
            {job.status === 'COMPLETE' && (
              <div className="flex justify-end">
                <Button>
                  <Download className="w-4 h-4 mr-2" />
                  Download Results
                </Button>
              </div>
            )}
          </div>
        )}
      </DialogContent>
    </Dialog>
  );
}
```

---

### NEO-1.7-008: Add Re-Download Functionality

**Priority**: P1 (High)
**Effort**: 20 minutes
**Dependencies**: NEO-1.7-006, NEO-1.7-007
**Reference**: FR-704

**Description**:
Enable re-downloading of past job results with proper format selection and filename generation.

**Acceptance Criteria**:
- [ ] Results retrievable by job ID
- [ ] All formats available for download
- [ ] Returns 404 if job not found or expired
- [ ] Same download naming convention as live results

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/app/api/v1/jobs/[id]/download/route.ts`

**Technical Notes**:
```typescript
import { NextRequest, NextResponse } from 'next/server';
import { prisma } from '@/lib/db/prisma';
import { getSession } from '@/lib/redis/session';
import { generateDownloadFilename } from '@/lib/utils/download';

export async function GET(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const requestId = request.headers.get('X-Request-ID') || crypto.randomUUID();
  const format = request.nextUrl.searchParams.get('format') || 'markdown';

  try {
    const sessionId = await getSession(request);
    if (!sessionId) {
      return NextResponse.json(
        { error: { code: 'E502', message: 'Invalid session' } },
        { status: 401 }
      );
    }

    const job = await prisma.job.findUnique({
      where: { id: params.id },
      include: {
        results: {
          where: { format: format.toUpperCase() as any },
        },
      },
    });

    if (!job || job.sessionId !== sessionId) {
      return NextResponse.json(
        { error: { code: 'E501', message: 'Job not found' } },
        { status: 404 }
      );
    }

    const result = job.results[0];
    if (!result) {
      return NextResponse.json(
        { error: { code: 'E503', message: 'Format not available' } },
        { status: 404 }
      );
    }

    return NextResponse.json({
      data: {
        format: result.format,
        content: result.content,
        size: result.size,
        originalName: job.fileName || 'document',
      },
    }, { headers: { 'X-Request-ID': requestId } });
  } catch (error) {
    console.error('Download error:', error);
    return NextResponse.json(
      { error: { code: 'E401', message: 'Database error' } },
      { status: 500 }
    );
  }
}
```

---

### NEO-1.7-009: Create useHistory Hook

**Priority**: P0 (Critical)
**Effort**: 20 minutes
**Dependencies**: NEO-1.7-005

**Description**:
Create a React hook for managing history state including pagination, loading, and data fetching.

**Acceptance Criteria**:
- [ ] Manages page state
- [ ] Handles data fetching
- [ ] Provides loading and error states
- [ ] Supports refresh functionality
- [ ] Memoizes API calls

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/hooks/useHistory.ts`

**Technical Notes**:
```typescript
'use client';

import { useState, useEffect, useCallback } from 'react';
import type { JobSummary, PaginationMeta } from '@/types/job';

interface UseHistoryOptions {
  initialPage?: number;
  initialPageSize?: number;
  sortBy?: 'createdAt' | 'status';
  sortOrder?: 'asc' | 'desc';
}

interface HistoryState {
  jobs: JobSummary[];
  pagination: PaginationMeta;
  isLoading: boolean;
  error: Error | null;
}

export function useHistory(options: UseHistoryOptions = {}) {
  const {
    initialPage = 1,
    initialPageSize = 20,
    sortBy = 'createdAt',
    sortOrder = 'desc',
  } = options;

  const [state, setState] = useState<HistoryState>({
    jobs: [],
    pagination: {
      page: initialPage,
      pageSize: initialPageSize,
      totalPages: 0,
      totalCount: 0,
      hasMore: false,
    },
    isLoading: true,
    error: null,
  });

  const fetchHistory = useCallback(async (page: number, pageSize: number) => {
    setState((prev) => ({ ...prev, isLoading: true, error: null }));

    try {
      const params = new URLSearchParams({
        page: page.toString(),
        pageSize: pageSize.toString(),
        sortBy,
        sortOrder,
      });

      const response = await fetch(`/api/v1/history?${params}`);
      if (!response.ok) {
        throw new Error('Failed to fetch history');
      }

      const data = await response.json();
      setState({
        jobs: data.jobs,
        pagination: data.pagination,
        isLoading: false,
        error: null,
      });
    } catch (error) {
      setState((prev) => ({
        ...prev,
        isLoading: false,
        error: error instanceof Error ? error : new Error('Unknown error'),
      }));
    }
  }, [sortBy, sortOrder]);

  useEffect(() => {
    fetchHistory(initialPage, initialPageSize);
  }, [fetchHistory, initialPage, initialPageSize]);

  const setPage = useCallback((page: number) => {
    fetchHistory(page, state.pagination.pageSize);
  }, [fetchHistory, state.pagination.pageSize]);

  const setPageSize = useCallback((pageSize: number) => {
    fetchHistory(1, pageSize);
  }, [fetchHistory]);

  const refresh = useCallback(() => {
    fetchHistory(state.pagination.page, state.pagination.pageSize);
  }, [fetchHistory, state.pagination.page, state.pagination.pageSize]);

  return {
    ...state,
    setPage,
    setPageSize,
    refresh,
  };
}
```

---

### NEO-1.7-010: Implement Cancel Endpoint

**Priority**: P0 (Critical)
**Effort**: 45 minutes
**Dependencies**: Sprint 1.5a (MCP Client), Sprint 1.5b (SSE)
**Reference**: FR-406, API Spec 4.3.1

**Description**:
Implement POST /api/v1/jobs/{id}/cancel endpoint with AbortController integration, SSE cancelled event, and partial result preservation.

**Acceptance Criteria**:
- [ ] AbortController stops in-flight MCP request
- [ ] Job status updated to CANCELLED
- [ ] SSE `cancelled` event sent
- [ ] Partial results preserved if configured
- [ ] Returns success response with cancellation details
- [ ] Returns 409 if job not cancellable

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/app/api/v1/jobs/[id]/cancel/route.ts`

**Technical Notes**:
```typescript
import { NextRequest, NextResponse } from 'next/server';
import { z } from 'zod';
import { prisma } from '@/lib/db/prisma';
import { getSession } from '@/lib/redis/session';
import { MCPClient } from '@/lib/mcp/client';
import { publishSSEEvent } from '@/lib/sse/publisher';
import { JobStatus } from '@prisma/client';

const cancelSchema = z.object({
  preservePartialResults: z.boolean().default(false),
});

const CANCELLABLE_STATUSES: JobStatus[] = [
  'PENDING',
  'UPLOADING',
  'PROCESSING',
  'RETRY_1',
  'RETRY_2',
  'RETRY_3',
];

export async function POST(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const requestId = request.headers.get('X-Request-ID') || crypto.randomUUID();

  try {
    const sessionId = await getSession(request);
    if (!sessionId) {
      return NextResponse.json(
        { error: { code: 'E502', message: 'Invalid session' } },
        { status: 401, headers: { 'X-Request-ID': requestId } }
      );
    }

    const body = await request.json();
    const validation = cancelSchema.safeParse(body);
    if (!validation.success) {
      return NextResponse.json(
        { error: { code: 'E100', message: 'Invalid request body' } },
        { status: 400, headers: { 'X-Request-ID': requestId } }
      );
    }

    const { preservePartialResults } = validation.data;

    // Get job
    const job = await prisma.job.findUnique({
      where: { id: params.id },
    });

    if (!job || job.sessionId !== sessionId) {
      return NextResponse.json(
        { error: { code: 'E501', message: 'Job not found' } },
        { status: 404, headers: { 'X-Request-ID': requestId } }
      );
    }

    // Check if cancellable
    if (!CANCELLABLE_STATUSES.includes(job.status)) {
      return NextResponse.json(
        {
          error: {
            code: 'E702',
            message: `Job cannot be cancelled in ${job.status} state`,
          },
        },
        { status: 409, headers: { 'X-Request-ID': requestId } }
      );
    }

    // Abort MCP request if processing
    if (job.status === 'PROCESSING') {
      try {
        const mcpClient = await MCPClient.getInstance();
        mcpClient.abort();
      } catch (error) {
        console.warn('Failed to abort MCP request:', error);
      }
    }

    // Update job status
    const cancelledAt = new Date();
    await prisma.job.update({
      where: { id: params.id },
      data: {
        status: 'CANCELLED',
        completedAt: cancelledAt,
      },
    });

    // Clean up partial results if not preserving
    if (!preservePartialResults) {
      await prisma.result.deleteMany({
        where: { jobId: params.id },
      });
    }

    // Send SSE cancelled event
    await publishSSEEvent(params.id, 'cancelled', {
      jobId: params.id,
      cancelledAt: cancelledAt.toISOString(),
      partialResultsPreserved: preservePartialResults,
    });

    return NextResponse.json({
      jobId: params.id,
      status: 'CANCELLED',
      cancelledAt: cancelledAt.toISOString(),
      partialResultsPreserved: preservePartialResults,
      message: 'Job cancelled successfully',
    }, { headers: { 'X-Request-ID': requestId } });
  } catch (error) {
    console.error('Cancel error:', error);
    return NextResponse.json(
      { error: { code: 'E401', message: 'Failed to cancel job' } },
      { status: 500, headers: { 'X-Request-ID': requestId } }
    );
  }
}
```

---

### NEO-1.7-011: Implement Resume Endpoint

**Priority**: P1 (High)
**Effort**: 30 minutes
**Dependencies**: Sprint 1.5b (Checkpoint Manager)
**Reference**: Spec Section 5.5

**Description**:
Implement POST /api/v1/jobs/{id}/resume endpoint to resume processing from a saved checkpoint.

**Acceptance Criteria**:
- [ ] Load checkpoint from job record
- [ ] Resume processing from checkpoint stage
- [ ] Only available for jobs with checkpoints
- [ ] Update job status to PROCESSING
- [ ] Returns error if no checkpoint available

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/app/api/v1/jobs/[id]/resume/route.ts`

**Technical Notes**:
```typescript
import { NextRequest, NextResponse } from 'next/server';
import { prisma } from '@/lib/db/prisma';
import { getSession } from '@/lib/redis/session';
import { CheckpointManager } from '@/lib/checkpoint/manager';

const RESUMABLE_STATUSES = ['ERROR', 'CANCELLED'];

export async function POST(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const requestId = request.headers.get('X-Request-ID') || crypto.randomUUID();

  try {
    const sessionId = await getSession(request);
    if (!sessionId) {
      return NextResponse.json(
        { error: { code: 'E502', message: 'Invalid session' } },
        { status: 401, headers: { 'X-Request-ID': requestId } }
      );
    }

    const job = await prisma.job.findUnique({
      where: { id: params.id },
    });

    if (!job || job.sessionId !== sessionId) {
      return NextResponse.json(
        { error: { code: 'E501', message: 'Job not found' } },
        { status: 404, headers: { 'X-Request-ID': requestId } }
      );
    }

    // Check if resumable
    if (!RESUMABLE_STATUSES.includes(job.status)) {
      return NextResponse.json(
        { error: { code: 'E703', message: 'Job cannot be resumed' } },
        { status: 409, headers: { 'X-Request-ID': requestId } }
      );
    }

    // Check for checkpoint existence
    if (!job.checkpointData) {
      return NextResponse.json(
        { error: { code: 'E703', message: 'No valid checkpoint exists' } },
        { status: 409, headers: { 'X-Request-ID': requestId } }
      );
    }

    // Check for checkpoint expiry (TTL exceeded)
    if (job.checkpointExpiry && new Date(job.checkpointExpiry) < new Date()) {
      return NextResponse.json(
        { error: { code: 'E704', message: 'Checkpoint expired' } },
        { status: 409, headers: { 'X-Request-ID': requestId } }
      );
    }

    // Load and validate checkpoint integrity
    let checkpoint;
    try {
      checkpoint = CheckpointManager.deserialize(job.checkpointData);
      // Verify checksum integrity
      if (!CheckpointManager.verifyChecksum(checkpoint)) {
        return NextResponse.json(
          { error: { code: 'E705', message: 'Checkpoint corrupted' } },
          { status: 409, headers: { 'X-Request-ID': requestId } }
        );
      }
    } catch (error) {
      // Deserialization failure also indicates corruption
      return NextResponse.json(
        { error: { code: 'E705', message: 'Checkpoint corrupted' } },
        { status: 409, headers: { 'X-Request-ID': requestId } }
      );
    }

    // Update job status to PROCESSING
    await prisma.job.update({
      where: { id: params.id },
      data: {
        status: 'PROCESSING',
        error: null,
        errorCode: null,
      },
    });

    // Resume processing (typically triggers background job)
    // This could be done via a queue or direct call
    // For simplicity, we return and let the client reconnect to SSE

    return NextResponse.json({
      jobId: params.id,
      status: 'PROCESSING',
      resumeFromStage: checkpoint.stage,
      message: 'Job resumed successfully',
    }, { headers: { 'X-Request-ID': requestId } });
  } catch (error) {
    console.error('Resume error:', error);
    return NextResponse.json(
      { error: { code: 'E401', message: 'Failed to resume job' } },
      { status: 500, headers: { 'X-Request-ID': requestId } }
    );
  }
}
```

---

### NEO-1.7-012: Add Session Authorization Check

**Priority**: P1 (High)
**Effort**: 15 minutes
**Dependencies**: NEO-1.7-005, NEO-1.7-006

**Description**:
Ensure all history and job APIs properly authorize requests so users can only see their own jobs.

**Acceptance Criteria**:
- [ ] Session ID validated on all endpoints
- [ ] Job sessionId compared to request session
- [ ] Unauthorized access returns 404 (not 403, for security)
- [ ] Consistent authorization across all job endpoints

**Deliverables**:
- Authorization middleware/utility
- Updates to all job API routes

**Technical Notes**:
```typescript
// src/lib/auth/job-authorization.ts

import type { Job } from '@prisma/client';
import { prisma } from '@/lib/db/prisma';

export async function authorizeJobAccess(
  jobId: string,
  sessionId: string
): Promise<{ authorized: boolean; job: Job | null }> {
  const job = await prisma.job.findUnique({
    where: { id: jobId },
  });

  if (!job) {
    return { authorized: false, job: null };
  }

  // Return unauthorized as 404 (security best practice)
  if (job.sessionId !== sessionId) {
    return { authorized: false, job: null };
  }

  return { authorized: true, job };
}
```

---

### NEO-1.7-013: Write Unit Tests for History API

**Priority**: P1 (High)
**Effort**: 20 minutes
**Dependencies**: NEO-1.7-005

**Description**:
Write unit tests for history API covering pagination, filtering, and sorting.

**Acceptance Criteria**:
- [ ] Test default pagination
- [ ] Test custom page size
- [ ] Test session filtering
- [ ] Test status sorting
- [ ] Test empty results
- [ ] Tests achieve 80%+ coverage

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/app/api/v1/history/route.test.ts`

**Technical Notes**:
```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
// ... test implementation
```

---

### NEO-1.7-014: Write Unit Tests for Cancel/Resume

**Priority**: P1 (High)
**Effort**: 20 minutes
**Dependencies**: NEO-1.7-010, NEO-1.7-011

**Description**:
Write unit tests for cancel and resume endpoints covering state transitions and edge cases.

**Acceptance Criteria**:
- [ ] Test successful cancellation
- [ ] Test non-cancellable state rejection
- [ ] Test successful resume
- [ ] Test no-checkpoint rejection
- [ ] Test authorization checks
- [ ] Tests achieve 80%+ coverage

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/app/api/v1/jobs/[id]/cancel/route.test.ts`
- `/home/agent0/hx-docling-ui/src/app/api/v1/jobs/[id]/resume/route.test.ts`

---

## Summary

| Task ID | Task Title | Effort | Priority |
|---------|-----------|--------|----------|
| NEO-1.7-001 | Create History Page Layout | 20m | P0 |
| NEO-1.7-002 | Implement HistoryView | 35m | P0 |
| NEO-1.7-003 | Create JobRow Component | 20m | P0 |
| NEO-1.7-004 | Implement Pagination | 20m | P0 |
| NEO-1.7-005 | Create History API | 35m | P0 |
| NEO-1.7-006 | Create Job Detail API | 20m | P0 |
| NEO-1.7-007 | Implement JobDetail Modal | 30m | P1 |
| NEO-1.7-008 | Add Re-Download | 20m | P1 |
| NEO-1.7-009 | Create useHistory Hook | 20m | P0 |
| NEO-1.7-010 | Implement Cancel Endpoint | 45m | P0 |
| NEO-1.7-011 | Implement Resume Endpoint | 30m | P1 |
| NEO-1.7-012 | Add Session Authorization | 15m | P1 |
| NEO-1.7-013 | Write History API Tests | 20m | P1 |
| NEO-1.7-014 | Write Cancel/Resume Tests | 20m | P1 |

**Total Effort**: ~3.5 hours (350 minutes)
**Total Tasks**: 14

---

## Dependencies Graph

```
Sprint 1.6 Complete
    |
    +-> NEO-1.7-001 (Page Layout)
    |       |
    |       +-> NEO-1.7-002 (HistoryView)
    |               |
    |               +-> NEO-1.7-003 (JobRow)
    |               +-> NEO-1.7-004 (Pagination)
    |
    +-> NEO-1.7-005 (History API)
    |       |
    |       +-> NEO-1.7-009 (useHistory Hook)
    |       +-> NEO-1.7-012 (Authorization)
    |       +-> NEO-1.7-013 (API Tests)
    |
    +-> NEO-1.7-006 (Job Detail API)
    |       |
    |       +-> NEO-1.7-007 (JobDetail Modal)
    |       +-> NEO-1.7-008 (Re-Download)
    |
    +-> NEO-1.7-010 (Cancel Endpoint)
    |       |
    |       +-> NEO-1.7-014 (Cancel Tests)
    |
    +-> NEO-1.7-011 (Resume Endpoint)
            |
            +-> NEO-1.7-014 (Resume Tests)
```

---

## Parallel Execution Opportunities

**Parallel Group A** (Independent):
- [P] NEO-1.7-005 (History API)
- [P] NEO-1.7-006 (Job Detail API)
- [P] NEO-1.7-010 (Cancel Endpoint)
- [P] NEO-1.7-011 (Resume Endpoint)

**Parallel Group B** (After APIs):
- [P] NEO-1.7-009 (useHistory Hook) - depends on 005
- [P] NEO-1.7-012 (Authorization) - depends on 005, 006

**Parallel Group C** (After Group B):
- [P] NEO-1.7-013 (History Tests)
- [P] NEO-1.7-014 (Cancel/Resume Tests)
