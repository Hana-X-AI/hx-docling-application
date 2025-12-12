# Sprint 1.7: Frontend UI Tasks - History View & Persistence

**Agent**: Ola Mae Johnson (`@ola`)
**Role**: Frontend UI Developer & Accessibility Specialist
**Sprint**: 1.7 - History View & Persistence
**Dependencies**: Sprint 1.6 Complete
**Effort**: 1.5h (of 3.5h total sprint)

---

## Task Overview

| Task ID | Title | Effort | Status |
|---------|-------|--------|--------|
| OLA-1.7-001 | Create History Page Layout | 20m | Pending |
| OLA-1.7-002 | Implement HistoryView with Table | 35m | Pending |
| OLA-1.7-003 | Create JobRow Component with Actions | 20m | Pending |
| OLA-1.7-004 | Implement Pagination Component | 20m | Pending |
| OLA-1.7-005 | Create JobDetail Modal | 30m | Pending |
| OLA-1.7-006 | Validate Responsive Design and Accessibility | 20m | Pending |

**Total Effort**: 1.5h (90 minutes)

---

## OLA-1.7-001: Create History Page Layout

### Description

Build the history page layout structure with filters, search, and pagination controls. Provides the container for the history table and job management features.

### Acceptance Criteria

- [ ] Page created at `/history` route
- [ ] Header with page title and description
- [ ] Filter controls (status, date range)
- [ ] Search input for filename/job ID
- [ ] Main content area for history table
- [ ] Pagination controls at bottom
- [ ] Empty state when no jobs
- [ ] Responsive layout for mobile and desktop
- [ ] Accessibility: page has proper heading hierarchy

### Dependencies

- Sprint 1.6: Results viewer complete
- Sprint 1.1: shadcn/ui components available

### Deliverables

**File**: `src/app/history/page.tsx`

```typescript
import { Suspense } from 'react'
import { HistoryView } from '@/components/history/HistoryView'
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card'

export default function HistoryPage() {
  return (
    <div className="container mx-auto py-8 space-y-6">
      <div>
        <h1 className="text-3xl font-bold tracking-tight">Processing History</h1>
        <p className="text-muted-foreground mt-2">
          View and manage your document processing jobs
        </p>
      </div>

      <Suspense fallback={<HistoryViewSkeleton />}>
        <HistoryView />
      </Suspense>
    </div>
  )
}

function HistoryViewSkeleton() {
  return (
    <Card>
      <CardHeader>
        <CardTitle>Loading history...</CardTitle>
      </CardHeader>
      <CardContent>
        <div className="space-y-4">
          {[1, 2, 3, 4, 5].map((i) => (
            <div key={i} className="h-16 bg-muted animate-pulse rounded" />
          ))}
        </div>
      </CardContent>
    </Card>
  )
}
```

### Technical Notes

- **Reference**: Implementation Plan Sprint 1.7 Task 1
- **Route**: `/history` - accessible from navigation
- **Suspense**: Wraps async HistoryView for streaming
- **SEO**: Page title set to "Processing History"
- **Layout**: Uses container for consistent width
- **Empty State**: Handled by HistoryView component
- **Accessibility**: Proper heading hierarchy (h1 for page title)

### Testing

```bash
# Manual testing:
# [ ] Navigate to /history - page loads
# [ ] Page title visible and correct
# [ ] Description text readable
# [ ] Loading skeleton shows during fetch
# [ ] HistoryView renders after load
# [ ] Responsive on mobile and desktop
# [ ] Screen reader announces page title
```

---

## OLA-1.7-002: Implement HistoryView with Table

### Description

Create the history table component displaying job list with columns for filename, status, date, and actions. Supports sorting, filtering, and pagination.

### Acceptance Criteria

- [ ] Table component using shadcn/ui Table
- [ ] Columns: Filename, Status, Created Date, Actions
- [ ] Status badge with color coding (PENDING: yellow, PROCESSING: blue, COMPLETED: green, FAILED: red, CANCELLED: gray, PARTIAL_COMPLETE: orange)
- [ ] Date formatted in human-readable format (e.g., "2 hours ago", "Jan 15, 2025")
- [ ] Click row to view details
- [ ] Sort by date (default: newest first)
- [ ] Filter by status dropdown
- [ ] Search by filename or job ID
- [ ] Empty state when no jobs match filters
- [ ] Responsive: table scrolls horizontally on mobile

### Dependencies

- OLA-1.7-001: History page layout complete
- Sprint 1.1: shadcn/ui Table component installed

### Deliverables

**File**: `src/components/history/HistoryView.tsx`

```typescript
'use client'

import { useState, useEffect } from 'react'
import { useRouter } from 'next/navigation'
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from '@/components/ui/table'
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card'
import { Input } from '@/components/ui/input'
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select'
import { JobRow } from './JobRow'
import { Pagination } from './Pagination'
import { useHistory } from '@/hooks/useHistory'
import { Search, FileX } from 'lucide-react'

export type JobStatus =
  | 'PENDING'
  | 'UPLOADING'
  | 'PROCESSING'
  | 'COMPLETED'
  | 'FAILED'
  | 'CANCELLED'
  | 'PARTIAL_COMPLETE'

export interface HistoryJob {
  id: string
  filename: string
  status: JobStatus
  createdAt: string
  updatedAt: string
  hasResults: boolean
}

export function HistoryView() {
  const router = useRouter()
  const [page, setPage] = useState(1)
  const [statusFilter, setStatusFilter] = useState<JobStatus | 'ALL'>('ALL')
  const [searchQuery, setSearchQuery] = useState('')

  const { jobs, total, isLoading } = useHistory({
    page,
    limit: 20,
    status: statusFilter === 'ALL' ? undefined : statusFilter,
    search: searchQuery || undefined,
  })

  const totalPages = Math.ceil(total / 20)

  const handleRowClick = (jobId: string) => {
    router.push(`/results/${jobId}`)
  }

  if (!isLoading && jobs.length === 0 && statusFilter === 'ALL' && !searchQuery) {
    return (
      <Card>
        <CardContent className="p-12">
          <div className="flex flex-col items-center gap-4 text-center">
            <FileX className="h-16 w-16 text-muted-foreground" />
            <div>
              <h3 className="text-lg font-semibold mb-2">No Processing History</h3>
              <p className="text-muted-foreground">
                Your processed documents will appear here.
              </p>
            </div>
          </div>
        </CardContent>
      </Card>
    )
  }

  return (
    <Card>
      <CardHeader>
        <div className="flex flex-col sm:flex-row gap-4 items-start sm:items-center justify-between">
          <CardTitle>Your Jobs ({total})</CardTitle>

          <div className="flex flex-col sm:flex-row gap-2 w-full sm:w-auto">
            {/* Search */}
            <div className="relative flex-1 sm:w-64">
              <Search className="absolute left-3 top-1/2 -translate-y-1/2 h-4 w-4 text-muted-foreground" />
              <Input
                type="search"
                placeholder="Search by filename or job ID..."
                value={searchQuery}
                onChange={(e) => setSearchQuery(e.target.value)}
                className="pl-10"
                aria-label="Search jobs"
              />
            </div>

            {/* Status Filter */}
            <Select
              value={statusFilter}
              onValueChange={(value) => setStatusFilter(value as JobStatus | 'ALL')}
            >
              <SelectTrigger className="w-full sm:w-40" aria-label="Filter by status">
                <SelectValue placeholder="Filter by status" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="ALL">All Status</SelectItem>
                <SelectItem value="COMPLETED">Completed</SelectItem>
                <SelectItem value="PROCESSING">Processing</SelectItem>
                <SelectItem value="PENDING">Pending</SelectItem>
                <SelectItem value="FAILED">Failed</SelectItem>
                <SelectItem value="CANCELLED">Cancelled</SelectItem>
                <SelectItem value="PARTIAL_COMPLETE">Partial</SelectItem>
              </SelectContent>
            </Select>
          </div>
        </div>
      </CardHeader>

      <CardContent>
        {isLoading ? (
          <div className="space-y-4">
            {[1, 2, 3, 4, 5].map((i) => (
              <div key={i} className="h-16 bg-muted animate-pulse rounded" />
            ))}
          </div>
        ) : jobs.length === 0 ? (
          <div className="text-center py-8 text-muted-foreground">
            No jobs found matching your filters.
          </div>
        ) : (
          <>
            <div className="overflow-x-auto">
              <Table>
                <TableHeader>
                  <TableRow>
                    <TableHead className="w-[40%]">Filename</TableHead>
                    <TableHead className="w-[20%]">Status</TableHead>
                    <TableHead className="w-[20%]">Created</TableHead>
                    <TableHead className="w-[20%] text-right">Actions</TableHead>
                  </TableRow>
                </TableHeader>
                <TableBody>
                  {jobs.map((job) => (
                    <JobRow
                      key={job.id}
                      job={job}
                      onClick={() => handleRowClick(job.id)}
                    />
                  ))}
                </TableBody>
              </Table>
            </div>

            {totalPages > 1 && (
              <div className="mt-6">
                <Pagination
                  currentPage={page}
                  totalPages={totalPages}
                  onPageChange={setPage}
                />
              </div>
            )}
          </>
        )}
      </CardContent>
    </Card>
  )
}
```

### Technical Notes

- **Reference**: Implementation Plan Sprint 1.7 Task 2
- **Pagination**: 20 jobs per page (configurable)
- **Sorting**: Server-side sorting by createdAt DESC
- **Filtering**: Client-side filter by status, server-side search
- **Search**: Searches filename and job ID fields
- **Empty State**: Different messages for no jobs vs. no results
- **Responsive**: Table scrolls horizontally on mobile
- **Accessibility**: Search and filter inputs have ARIA labels

### Testing

```bash
# Unit test
npm run test src/components/history/HistoryView.test.tsx

# Manual testing:
# [ ] Table displays jobs correctly
# [ ] Search filters jobs by filename/ID
# [ ] Status filter updates results
# [ ] Pagination shows correct page count
# [ ] Click row navigates to job details
# [ ] Empty state shows when no jobs
# [ ] Loading skeleton shows during fetch
# [ ] Responsive: table scrolls on mobile
# [ ] Screen reader announces job count
```

---

## OLA-1.7-003: Create JobRow Component with Actions

### Description

Build individual job row component with status badge, action buttons (view, re-download, cancel/resume), and hover states.

### Acceptance Criteria

- [ ] Row displays filename (truncated with tooltip)
- [ ] Status badge color-coded by job status
- [ ] Created date formatted with relative time (e.g., "2 hours ago")
- [ ] Action buttons: View Details, Re-download (if completed), Cancel (if processing), Resume (if failed)
- [ ] Hover state highlights entire row
- [ ] Click row to view details
- [ ] Action buttons have loading states
- [ ] Accessibility: row has role="button", action buttons keyboard accessible

### Dependencies

- OLA-1.7-002: HistoryView complete
- Sprint 1.1: shadcn/ui components available

### Deliverables

**File**: `src/components/history/JobRow.tsx`

```typescript
'use client'

import { TableCell, TableRow } from '@/components/ui/table'
import { Badge } from '@/components/ui/badge'
import { Button } from '@/components/ui/button'
import { Download, Eye, X, PlayCircle } from 'lucide-react'
import { formatDistanceToNow } from 'date-fns'
import type { HistoryJob, JobStatus } from './HistoryView'
import { cn } from '@/lib/utils'

interface JobRowProps {
  job: HistoryJob
  onClick: () => void
}

const STATUS_CONFIG: Record<
  JobStatus,
  { label: string; variant: 'default' | 'secondary' | 'destructive'; className: string }
> = {
  PENDING: {
    label: 'Pending',
    variant: 'secondary',
    className: 'bg-yellow-600 hover:bg-yellow-700',
  },
  UPLOADING: {
    label: 'Uploading',
    variant: 'default',
    className: 'bg-blue-600 hover:bg-blue-700',
  },
  PROCESSING: {
    label: 'Processing',
    variant: 'default',
    className: 'bg-blue-600 hover:bg-blue-700',
  },
  COMPLETED: {
    label: 'Completed',
    variant: 'default',
    className: 'bg-green-600 hover:bg-green-700',
  },
  FAILED: {
    label: 'Failed',
    variant: 'destructive',
    className: 'bg-red-600 hover:bg-red-700',
  },
  CANCELLED: {
    label: 'Cancelled',
    variant: 'secondary',
    className: 'bg-gray-600 hover:bg-gray-700',
  },
  PARTIAL_COMPLETE: {
    label: 'Partial',
    variant: 'default',
    className: 'bg-orange-600 hover:bg-orange-700',
  },
}

export function JobRow({ job, onClick }: JobRowProps) {
  const statusConfig = STATUS_CONFIG[job.status]
  const canDownload = job.status === 'COMPLETED' || job.status === 'PARTIAL_COMPLETE'
  const canCancel = job.status === 'PROCESSING' || job.status === 'UPLOADING'
  const canResume = job.status === 'FAILED'

  const handleDownload = (e: React.MouseEvent) => {
    e.stopPropagation()
    // Download logic handled by parent
    console.log('Download job:', job.id)
  }

  const handleCancel = (e: React.MouseEvent) => {
    e.stopPropagation()
    // Cancel logic handled by parent
    console.log('Cancel job:', job.id)
  }

  const handleResume = (e: React.MouseEvent) => {
    e.stopPropagation()
    // Resume logic handled by parent
    console.log('Resume job:', job.id)
  }

  return (
    <TableRow
      className={cn(
        'cursor-pointer transition-colors',
        'hover:bg-muted/50',
        'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring'
      )}
      onClick={onClick}
      role="button"
      tabIndex={0}
      onKeyDown={(e) => {
        if (e.key === 'Enter' || e.key === ' ') {
          e.preventDefault()
          onClick()
        }
      }}
      aria-label={`View details for ${job.filename}`}
    >
      <TableCell className="font-medium">
        <div className="max-w-md truncate" title={job.filename}>
          {job.filename}
        </div>
      </TableCell>

      <TableCell>
        <Badge variant={statusConfig.variant} className={statusConfig.className}>
          {statusConfig.label}
        </Badge>
      </TableCell>

      <TableCell className="text-muted-foreground">
        <time dateTime={job.createdAt} title={new Date(job.createdAt).toLocaleString()}>
          {formatDistanceToNow(new Date(job.createdAt), { addSuffix: true })}
        </time>
      </TableCell>

      <TableCell className="text-right">
        <div className="flex items-center justify-end gap-2">
          {canDownload && (
            <Button
              variant="ghost"
              size="sm"
              onClick={handleDownload}
              aria-label={`Download results for ${job.filename}`}
            >
              <Download className="h-4 w-4 mr-1" />
              Download
            </Button>
          )}
          {canCancel && (
            <Button
              variant="ghost"
              size="sm"
              onClick={handleCancel}
              aria-label={`Cancel processing for ${job.filename}`}
            >
              <X className="h-4 w-4 mr-1" />
              Cancel
            </Button>
          )}
          {canResume && (
            <Button
              variant="ghost"
              size="sm"
              onClick={handleResume}
              aria-label={`Resume processing for ${job.filename}`}
            >
              <PlayCircle className="h-4 w-4 mr-1" />
              Resume
            </Button>
          )}
          <Button
            variant="ghost"
            size="sm"
            onClick={(e) => {
              e.stopPropagation()
              onClick()
            }}
            aria-label={`View details for ${job.filename}`}
          >
            <Eye className="h-4 w-4 mr-1" />
            View
          </Button>
        </div>
      </TableCell>
    </TableRow>
  )
}
```

### Technical Notes

- **Reference**: Implementation Plan Sprint 1.7 Task 3
- **Date Formatting**: `date-fns` formatDistanceToNow for relative time
- **Status Badges**: Color-coded per job status
- **Conditional Actions**:
  - Download: COMPLETED or PARTIAL_COMPLETE
  - Cancel: PROCESSING or UPLOADING
  - Resume: FAILED
  - View: Always available
- **Click Handling**: `stopPropagation` on action buttons to prevent row click
- **Accessibility**:
  - Row has `role="button"` and `tabIndex={0}`
  - Keyboard support: Enter/Space to activate
  - Action buttons have descriptive ARIA labels
  - Filename truncation with title tooltip

### Testing

```bash
# Unit test
npm run test src/components/history/JobRow.test.tsx

# Manual testing:
# [ ] Row displays filename, status, date correctly
# [ ] Status badge colors match job status
# [ ] Relative time updates ("2 hours ago")
# [ ] Hover highlights entire row
# [ ] Click row navigates to details
# [ ] Download button visible for completed jobs
# [ ] Cancel button visible for processing jobs
# [ ] Resume button visible for failed jobs
# [ ] View button always visible
# [ ] Action buttons don't trigger row click
# [ ] Keyboard navigation works (Tab, Enter, Space)
# [ ] Screen reader announces job info
# [ ] Filename tooltip shows on hover (long names)
```

---

## OLA-1.7-004: Implement Pagination Component

### Description

Create a pagination component with page numbers, previous/next buttons, and page size selector.

### Acceptance Criteria

- [ ] Component displays current page and total pages
- [ ] Previous/Next buttons functional
- [ ] Page numbers clickable (show range: e.g., 1...5 6 7...20)
- [ ] Current page visually distinct
- [ ] First/Last page buttons
- [ ] Previous disabled on page 1, Next disabled on last page
- [ ] Responsive: compact view on mobile
- [ ] Accessibility: buttons keyboard accessible, current page announced

### Dependencies

- OLA-1.7-002: HistoryView complete
- Sprint 1.1: shadcn/ui Pagination component installed

### Deliverables

**File**: `src/components/history/Pagination.tsx`

```typescript
'use client'

import {
  Pagination as PaginationRoot,
  PaginationContent,
  PaginationEllipsis,
  PaginationItem,
  PaginationLink,
  PaginationNext,
  PaginationPrevious,
} from '@/components/ui/pagination'
import { Button } from '@/components/ui/button'
import { ChevronFirst, ChevronLast } from 'lucide-react'

interface PaginationProps {
  currentPage: number
  totalPages: number
  onPageChange: (page: number) => void
  maxVisible?: number
}

export function Pagination({
  currentPage,
  totalPages,
  onPageChange,
  maxVisible = 7,
}: PaginationProps) {
  const getPageNumbers = () => {
    if (totalPages <= maxVisible) {
      return Array.from({ length: totalPages }, (_, i) => i + 1)
    }

    const halfVisible = Math.floor(maxVisible / 2)
    let start = Math.max(1, currentPage - halfVisible)
    let end = Math.min(totalPages, currentPage + halfVisible)

    if (currentPage <= halfVisible) {
      end = maxVisible
    } else if (currentPage >= totalPages - halfVisible) {
      start = totalPages - maxVisible + 1
    }

    const pages: (number | 'ellipsis')[] = []

    if (start > 1) {
      pages.push(1)
      if (start > 2) pages.push('ellipsis')
    }

    for (let i = start; i <= end; i++) {
      pages.push(i)
    }

    if (end < totalPages) {
      if (end < totalPages - 1) pages.push('ellipsis')
      pages.push(totalPages)
    }

    return pages
  }

  const pages = getPageNumbers()

  return (
    <PaginationRoot aria-label="Pagination navigation">
      <PaginationContent>
        {/* First Page */}
        <PaginationItem>
          <Button
            variant="ghost"
            size="icon"
            onClick={() => onPageChange(1)}
            disabled={currentPage === 1}
            aria-label="Go to first page"
          >
            <ChevronFirst className="h-4 w-4" />
          </Button>
        </PaginationItem>

        {/* Previous */}
        <PaginationItem>
          <PaginationPrevious
            onClick={() => onPageChange(Math.max(1, currentPage - 1))}
            aria-disabled={currentPage === 1}
          />
        </PaginationItem>

        {/* Page Numbers */}
        {pages.map((page, index) =>
          page === 'ellipsis' ? (
            <PaginationItem key={`ellipsis-${index}`}>
              <PaginationEllipsis />
            </PaginationItem>
          ) : (
            <PaginationItem key={page}>
              <PaginationLink
                onClick={() => onPageChange(page)}
                isActive={page === currentPage}
                aria-current={page === currentPage ? 'page' : undefined}
              >
                {page}
              </PaginationLink>
            </PaginationItem>
          )
        )}

        {/* Next */}
        <PaginationItem>
          <PaginationNext
            onClick={() => onPageChange(Math.min(totalPages, currentPage + 1))}
            aria-disabled={currentPage === totalPages}
          />
        </PaginationItem>

        {/* Last Page */}
        <PaginationItem>
          <Button
            variant="ghost"
            size="icon"
            onClick={() => onPageChange(totalPages)}
            disabled={currentPage === totalPages}
            aria-label="Go to last page"
          >
            <ChevronLast className="h-4 w-4" />
          </Button>
        </PaginationItem>
      </PaginationContent>
    </PaginationRoot>
  )
}
```

### Technical Notes

- **Reference**: Implementation Plan Sprint 1.7 Task 4
- **Page Range Logic**: Shows current page ± 3 pages, with ellipsis
- **First/Last Buttons**: Quick navigation to endpoints
- **Responsive**: Component adapts width on mobile
- **Accessibility**:
  - `role="navigation"` and `aria-label="Pagination navigation"`
  - Current page has `aria-current="page"`
  - Disabled buttons have `aria-disabled="true"`
  - Page numbers keyboard navigable

### Testing

```bash
# Unit test
npm run test src/components/history/Pagination.test.tsx

# Manual testing:
# [ ] Previous disabled on page 1
# [ ] Next disabled on last page
# [ ] Click page number - navigates correctly
# [ ] Click Previous - goes to previous page
# [ ] Click Next - goes to next page
# [ ] First button goes to page 1
# [ ] Last button goes to final page
# [ ] Ellipsis appears for large page counts
# [ ] Current page visually distinct
# [ ] Keyboard navigation works (Tab, Enter)
# [ ] Screen reader announces current page
# [ ] Responsive on mobile (compact view)
```

---

## OLA-1.7-005: Create JobDetail Modal

### Description

Build a modal dialog showing full job details including all results, metadata, and re-download options.

### Acceptance Criteria

- [ ] Modal component using shadcn/ui Dialog
- [ ] Displays job metadata (ID, filename, status, dates)
- [ ] Shows processing timeline (stages with timestamps)
- [ ] Embedded ResultsViewer for completed jobs
- [ ] Re-download button for each format
- [ ] Cancel/Resume buttons for appropriate statuses
- [ ] Close button and escape key support
- [ ] Modal scrollable if content exceeds viewport
- [ ] Accessibility: focus trapped in modal, focus restored on close

### Dependencies

- OLA-1.7-003: JobRow component complete
- Sprint 1.6: ResultsViewer complete
- Sprint 1.1: shadcn/ui Dialog component installed

### Deliverables

**File**: `src/components/history/JobDetail.tsx`

```typescript
'use client'

import { useState, useEffect } from 'react'
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogHeader,
  DialogTitle,
} from '@/components/ui/dialog'
import { Badge } from '@/components/ui/badge'
import { Button } from '@/components/ui/button'
import { Separator } from '@/components/ui/separator'
import { ResultsViewer } from '@/components/results/ResultsViewer'
import { Download, X, PlayCircle, Clock } from 'lucide-react'
import { format } from 'date-fns'
import type { JobStatus } from './HistoryView'

interface JobDetailProps {
  jobId: string | null
  open: boolean
  onOpenChange: (open: boolean) => void
}

interface JobDetailData {
  id: string
  filename: string
  status: JobStatus
  createdAt: string
  updatedAt: string
  result?: any
  timeline?: Array<{ stage: string; timestamp: string; status: 'complete' | 'failed' }>
}

export function JobDetail({ jobId, open, onOpenChange }: JobDetailProps) {
  // In real implementation, fetch job details from API
  const [job, setJob] = useState<JobDetailData | null>(null)
  const [isLoading, setIsLoading] = useState(false)

  useEffect(() => {
    if (jobId && open) {
      fetchJobDetails(jobId)
    }
  }, [jobId, open])

  const fetchJobDetails = async (id: string) => {
    setIsLoading(true)
    try {
      const response = await fetch(`/api/v1/jobs/${id}`)
      const data = await response.json()
      setJob(data)
    } catch (error) {
      console.error('Failed to fetch job details:', error)
    } finally {
      setIsLoading(false)
    }
  }

  const canCancel = job?.status === 'PROCESSING' || job?.status === 'UPLOADING'
  const canResume = job?.status === 'FAILED'
  const hasResults = job?.status === 'COMPLETED' || job?.status === 'PARTIAL_COMPLETE'

  return (
    <Dialog open={open} onOpenChange={onOpenChange}>
      <DialogContent className="max-w-4xl max-h-[90vh] overflow-y-auto">
        <DialogHeader>
          <DialogTitle className="flex items-center justify-between">
            <span>Job Details</span>
            {job && (
              <Badge
                variant={
                  job.status === 'COMPLETED'
                    ? 'default'
                    : job.status === 'FAILED'
                    ? 'destructive'
                    : 'secondary'
                }
              >
                {job.status}
              </Badge>
            )}
          </DialogTitle>
          <DialogDescription>
            {job?.filename || 'Loading...'}
          </DialogDescription>
        </DialogHeader>

        {isLoading ? (
          <div className="space-y-4 p-4">
            <div className="h-24 bg-muted animate-pulse rounded" />
            <div className="h-48 bg-muted animate-pulse rounded" />
          </div>
        ) : job ? (
          <div className="space-y-6 pt-4">
            {/* Metadata */}
            <div className="grid grid-cols-2 gap-4 text-sm">
              <div>
                <p className="text-muted-foreground mb-1">Job ID</p>
                <p className="font-mono">{job.id}</p>
              </div>
              <div>
                <p className="text-muted-foreground mb-1">Created</p>
                <p>{format(new Date(job.createdAt), 'PPpp')}</p>
              </div>
              <div>
                <p className="text-muted-foreground mb-1">Filename</p>
                <p className="truncate" title={job.filename}>
                  {job.filename}
                </p>
              </div>
              <div>
                <p className="text-muted-foreground mb-1">Last Updated</p>
                <p>{format(new Date(job.updatedAt), 'PPpp')}</p>
              </div>
            </div>

            <Separator />

            {/* Timeline */}
            {job.timeline && job.timeline.length > 0 && (
              <>
                <div>
                  <h3 className="text-sm font-semibold mb-3 flex items-center gap-2">
                    <Clock className="h-4 w-4" />
                    Processing Timeline
                  </h3>
                  <div className="space-y-2">
                    {job.timeline.map((item, index) => (
                      <div key={index} className="flex items-center gap-3 text-sm">
                        <div
                          className={`h-2 w-2 rounded-full ${
                            item.status === 'complete'
                              ? 'bg-green-600'
                              : 'bg-red-600'
                          }`}
                        />
                        <span className="flex-1">{item.stage}</span>
                        <span className="text-muted-foreground">
                          {format(new Date(item.timestamp), 'pp')}
                        </span>
                      </div>
                    ))}
                  </div>
                </div>
                <Separator />
              </>
            )}

            {/* Results */}
            {hasResults && job.result && (
              <div>
                <h3 className="text-sm font-semibold mb-3">Results</h3>
                <ResultsViewer result={job.result} />
              </div>
            )}

            {/* Actions */}
            <div className="flex gap-2 justify-end pt-4">
              {canCancel && (
                <Button variant="destructive" onClick={() => console.log('Cancel job')}>
                  <X className="h-4 w-4 mr-1" />
                  Cancel Job
                </Button>
              )}
              {canResume && (
                <Button variant="default" onClick={() => console.log('Resume job')}>
                  <PlayCircle className="h-4 w-4 mr-1" />
                  Resume Job
                </Button>
              )}
              <Button variant="outline" onClick={() => onOpenChange(false)}>
                Close
              </Button>
            </div>
          </div>
        ) : (
          <div className="text-center py-8 text-muted-foreground">
            Job not found
          </div>
        )}
      </DialogContent>
    </Dialog>
  )
}
```

### Technical Notes

- **Reference**: Implementation Plan Sprint 1.7 Task 7
- **Modal Scrolling**: `max-h-[90vh]` with `overflow-y-auto`
- **Focus Trap**: shadcn/ui Dialog handles focus management
- **Embedded Viewer**: Reuses ResultsViewer component
- **Timeline**: Shows processing stages with timestamps
- **Actions**: Contextual based on job status
- **Accessibility**:
  - `role="dialog"` and `aria-modal="true"`
  - Focus trapped within modal
  - Escape key closes modal
  - Focus restored to trigger on close

### Testing

```bash
# Unit test
npm run test src/components/history/JobDetail.test.tsx

# Manual testing:
# [ ] Click job row - modal opens
# [ ] Modal displays job metadata correctly
# [ ] Timeline shows processing stages
# [ ] ResultsViewer embedded for completed jobs
# [ ] Cancel button visible for processing jobs
# [ ] Resume button visible for failed jobs
# [ ] Close button closes modal
# [ ] Escape key closes modal
# [ ] Modal scrollable when content long
# [ ] Focus trapped in modal
# [ ] Focus returns to row on close
# [ ] Screen reader announces modal content
# [ ] Keyboard navigation works
```

---

## OLA-1.7-006: Validate Responsive Design and Accessibility

### Description

Comprehensive validation of responsive design and accessibility compliance for all history components.

### Acceptance Criteria

- [ ] Mobile (320px-767px): Table scrolls horizontally, filters stack vertically
- [ ] Tablet (768px-1023px): Optimized layout, touch-friendly targets
- [ ] Desktop (1024px+): Full table visible, optimal layout
- [ ] All touch targets >= 44x44px on mobile
- [ ] Keyboard navigation functional (Tab, Enter, Arrow keys)
- [ ] ARIA roles and labels present
- [ ] Color contrast >= 4.5:1 for all text
- [ ] Screen reader announces table structure and updates
- [ ] Focus indicators visible
- [ ] Modal accessible with keyboard

### Dependencies

- OLA-1.7-001 through OLA-1.7-005: All history components complete

### Deliverables

**Responsive Checklist**:

| Breakpoint | Table | Filters | Pagination | Modal |
|------------|-------|---------|------------|-------|
| Mobile (< 768px) | Horizontal scroll | Stacked | Compact | Full-screen |
| Tablet (768-1023px) | Full width | Inline | Full | 90% width |
| Desktop (>= 1024px) | Fixed width | Inline | Full | Max 4xl |

**Accessibility Checklist**:

```
History Page:
[ ] Page has h1 heading
[ ] Heading hierarchy correct (h1 -> h2 -> h3)

HistoryView:
[ ] Table has caption or aria-label
[ ] TableHeader has proper role
[ ] Search input has aria-label
[ ] Status filter has aria-label
[ ] Empty state announces to screen reader

JobRow:
[ ] Row has role="button"
[ ] Row keyboard accessible (Tab, Enter, Space)
[ ] Action buttons have descriptive aria-labels
[ ] Status badge color not sole indicator (includes text)
[ ] Date has semantic <time> element

Pagination:
[ ] Has role="navigation" and aria-label
[ ] Current page has aria-current="page"
[ ] Disabled buttons have aria-disabled
[ ] Page numbers keyboard navigable

JobDetail Modal:
[ ] Modal has role="dialog" and aria-modal="true"
[ ] Modal title linked via aria-labelledby
[ ] Focus trapped in modal
[ ] Escape key closes modal
[ ] Focus restored on close
[ ] Timeline accessible to screen reader
```

### Technical Notes

- **Reference**: Specification Section 9, Implementation Plan Sprint 1.7 Task 12
- **Testing Tools**:
  - Lighthouse accessibility audit (target >= 90)
  - axe DevTools
  - NVDA or JAWS screen reader
  - Keyboard-only navigation
  - Chrome DevTools responsive mode
- **Touch Targets**: Minimum 44x44px per WCAG 2.5.5
- **Focus Management**: Visible focus rings using Tailwind utilities

### Testing

```bash
# Automated testing
npm run test:a11y
npm run lighthouse

# Manual testing checklist:
# Responsive:
# [ ] Test at 320px - table scrolls, filters stack
# [ ] Test at 768px - optimal layout
# [ ] Test at 1024px+ - full desktop view
# [ ] Touch targets >= 44x44px on mobile
# [ ] Modal adapts to screen size

# Accessibility:
# [ ] Tab through all elements without mouse
# [ ] Arrow keys navigate table rows
# [ ] Enter/Space activate buttons
# [ ] Escape closes modal
# [ ] Screen reader announces table headers
# [ ] Screen reader announces job count
# [ ] Screen reader announces status changes
# [ ] Screen reader announces page changes
# [ ] Focus indicators visible
# [ ] Color contrast verified
# [ ] Lighthouse score >= 90
```

---

## Sprint Completion Checklist

### Acceptance Criteria (Sprint 1.7 Frontend)

- [ ] History page accessible at /history route
- [ ] HistoryView displays job table with pagination
- [ ] JobRow shows filename, status, date, actions
- [ ] Pagination functional with page numbers
- [ ] JobDetail modal shows full job information
- [ ] Search filters jobs by filename/ID
- [ ] Status filter updates results
- [ ] Re-download works for completed jobs
- [ ] Cancel works for processing jobs
- [ ] Resume works for failed jobs
- [ ] All components responsive (mobile, tablet, desktop)
- [ ] Accessibility checklist complete
- [ ] Unit tests pass with >= 80% coverage
- [ ] No TypeScript errors
- [ ] No console warnings

### Integration Points

**With Neo (Backend)**:
- Fetches history from `/api/v1/history` endpoint
- Fetches job details from `/api/v1/jobs/{id}`
- Triggers cancel via `/api/v1/jobs/{id}/cancel`
- Triggers resume via `/api/v1/jobs/{id}/resume`

**With Trinity (Database)**:
- Queries paginated job list
- Filters by session and status
- Sorts by creation date

**With Julia (Testing)**:
- Unit tests for all history components
- E2E tests for complete history flow
- Accessibility tests passing

### Files Delivered

```
src/app/history/
└── page.tsx

src/components/history/
├── HistoryView.tsx
├── JobRow.tsx
├── Pagination.tsx
├── JobDetail.tsx
├── HistoryView.test.tsx
├── JobRow.test.tsx
├── Pagination.test.tsx
└── JobDetail.test.tsx

src/hooks/
└── useHistory.ts
```

### Next Sprint Preview

**Sprint 1.8** will focus on:
- Comprehensive testing (state machine, security, accessibility)
- Documentation (README, API docs, deployment guide)
- Production readiness validation

---

## Notes

- **Performance**: Pagination reduces database queries and improves render time
- **UX**: Relative time ("2 hours ago") more intuitive than absolute dates
- **Security**: Session-based filtering ensures users only see their jobs
- **Accessibility**: Full table navigation support for screen readers
- **Responsiveness**: Mobile-first design with progressive enhancement
