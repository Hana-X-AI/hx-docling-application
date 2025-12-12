# Tasks: shadcn/ui Components for History View (Sprint 1.7)

**Version**: 1.0.1
**Agent**: Gordon Zain (`@gordon`) - shadcn/ui Specialist
**Sprint**: 1.7 - History View & Persistence
**Duration**: ~1.0h
**Prerequisites**: Sprint 1.6 complete, Results viewer functional

---

## Changelog

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0.1 | 2025-12-12 | Gordon Zain | Fixed DEF-017: JobRow keyboard navigation guard to prevent double-triggering when action buttons are focused |
| 1.0.0 | - | Gordon Zain | Initial task document |

## MCP Integration

**hx-shadcn MCP Server**: Use the MCP server for table, dialog, and pagination components.

**Service Reference**: `project/0.8-references/hx-shadcn-service-descriptor.md`
**Endpoint**: `http://hx-shadcn.hx.dev.local:7423/sse`

**MCP Tools for This Sprint**:
| Tool | Usage in Sprint 1.7 |
|------|---------------------|
| `get_component` | Retrieve Table, Dialog, Badge, Separator component source |
| `get_component_demo` | View Table component usage with sorting and pagination |
| `get_component_metadata` | Review Dialog props for modal implementation |

**Example: Retrieve Table Component with Demo**
```json
{
  "method": "tools/call",
  "params": {
    "name": "get_component_demo",
    "arguments": {"componentName": "table"}
  }
}
```

**Component Installation Notes**:
- Table, Dialog, Pagination pre-installed in Sprint 1.1
- Badge already installed in Sprint 1.6
- Separator: `npx shadcn@latest add separator`
- MCP server provides reference for responsive table patterns

## Task List

### GOR-1.7-001: Implement HistoryView with shadcn/ui Table Component
**Status**: Pending
**Effort**: 30 minutes
**Dependencies**: Sprint 1.7 Task 2 (HistoryView component created)
**Priority**: P0 - Critical Path

#### Description
Create a responsive table view for job history using shadcn/ui Table component. Display file/URL, status, date, and actions with proper sorting and filtering.

#### Acceptance Criteria
- [ ] Table component displays job history
- [ ] Columns: File/URL, Status, Date, Actions
- [ ] Status badge with color coding (pending, processing, completed, failed, cancelled)
- [ ] Rows sorted by createdAt DESC (most recent first)
- [ ] Responsive design (collapses on mobile)
- [ ] Hover state on rows
- [ ] Dark mode styling works
- [ ] Accessibility: Table headers, row descriptions

#### Technical Implementation
```tsx
// src/components/history/HistoryView.tsx
import {
  Table,
  TableBody,
  TableCaption,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from '@/components/ui/table';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
import { JobRow } from './JobRow';
import { Pagination } from './Pagination';
import { Skeleton } from '@/components/ui/skeleton';

interface Job {
  id: string;
  inputType: 'FILE' | 'URL';
  fileName?: string;
  url?: string;
  status: 'PENDING' | 'PROCESSING' | 'COMPLETED' | 'FAILED' | 'CANCELLED';
  createdAt: string;
  updatedAt: string;
}

interface HistoryViewProps {
  jobs: Job[];
  isLoading: boolean;
  currentPage: number;
  totalPages: number;
  onPageChange: (page: number) => void;
  onJobClick: (jobId: string) => void;
}

export function HistoryView({
  jobs,
  isLoading,
  currentPage,
  totalPages,
  onPageChange,
  onJobClick,
}: HistoryViewProps) {
  if (isLoading) {
    return <HistoryTableSkeleton />;
  }

  if (jobs.length === 0) {
    return (
      <Card>
        <CardContent className="flex h-[400px] items-center justify-center">
          <div className="text-center">
            <p className="text-lg font-medium text-muted-foreground">No jobs found</p>
            <p className="text-sm text-muted-foreground mt-2">
              Upload a file or process a URL to get started
            </p>
          </div>
        </CardContent>
      </Card>
    );
  }

  return (
    <Card>
      <CardHeader>
        <CardTitle>Processing History</CardTitle>
        <CardDescription>
          View your recent document processing jobs
        </CardDescription>
      </CardHeader>
      <CardContent>
        <div className="rounded-md border">
          <Table>
            <TableCaption className="sr-only">
              List of document processing jobs, sorted by date (newest first)
            </TableCaption>
            <TableHeader>
              <TableRow>
                <TableHead className="w-[40%]">File/URL</TableHead>
                <TableHead className="w-[20%]">Status</TableHead>
                <TableHead className="w-[25%]">Date</TableHead>
                <TableHead className="w-[15%] text-right">Actions</TableHead>
              </TableRow>
            </TableHeader>
            <TableBody>
              {jobs.map((job) => (
                <JobRow
                  key={job.id}
                  job={job}
                  onClick={() => onJobClick(job.id)}
                />
              ))}
            </TableBody>
          </Table>
        </div>

        {/* Pagination */}
        {totalPages > 1 && (
          <div className="mt-4">
            <Pagination
              currentPage={currentPage}
              totalPages={totalPages}
              onPageChange={onPageChange}
            />
          </div>
        )}
      </CardContent>
    </Card>
  );
}

// Loading skeleton
function HistoryTableSkeleton() {
  return (
    <Card>
      <CardHeader>
        <Skeleton className="h-6 w-48" />
        <Skeleton className="h-4 w-64 mt-2" />
      </CardHeader>
      <CardContent>
        <div className="space-y-2">
          <Skeleton className="h-10 w-full" /> {/* Header */}
          <Skeleton className="h-12 w-full" /> {/* Row 1 */}
          <Skeleton className="h-12 w-full" /> {/* Row 2 */}
          <Skeleton className="h-12 w-full" /> {/* Row 3 */}
          <Skeleton className="h-12 w-full" /> {/* Row 4 */}
          <Skeleton className="h-12 w-full" /> {/* Row 5 */}
        </div>
      </CardContent>
    </Card>
  );
}
```

**Table Layout**:
```
┌──────────────────────────────────────────────────────────────┐
│ Processing History                                           │
│ View your recent document processing jobs                    │
├──────────────────────────────────────────────────────────────┤
│ File/URL           │ Status      │ Date            │ Actions │
├──────────────────────────────────────────────────────────────┤
│ document.pdf       │ ● Completed │ 2 hours ago     │ [View]  │
│ report.docx        │ ● Processing│ 3 hours ago     │ [View]  │
│ https://...        │ ● Failed    │ 1 day ago       │ [Retry] │
└──────────────────────────────────────────────────────────────┘
                      [Pagination]
```

**Responsive Behavior**:
| Breakpoint | Layout |
|------------|--------|
| Desktop (>1024px) | Full table with all columns |
| Tablet (768-1024px) | Stack status and date, reduce padding |
| Mobile (<768px) | Card view (replace table with stacked cards) |

#### Validation
```bash
# Test table functionality
# 1. Load history page → Table displays jobs
# 2. Verify: Most recent job at top
# 3. Verify: Status badges color-coded
# 4. Hover over row → Background highlights
# 5. Click row → Job detail modal opens
# 6. Resize to mobile → Table adapts or switches to card view
# 7. Toggle dark mode → Table styling adapts
```

#### Deliverables
- HistoryView component with Table
- Loading skeleton for table
- Empty state for no jobs
- Responsive table layout

---

### GOR-1.7-002: Create JobRow Component with Status Badge
**Status**: Pending
**Effort**: 20 minutes
**Dependencies**: Sprint 1.7 Task 3 (JobRow component created)
**Priority**: P0 - Critical Path

#### Description
Build individual job row component with status badge, action buttons, and formatted date display using shadcn/ui components.

#### Acceptance Criteria
- [ ] TableRow component displays job data
- [ ] Status badge color-coded by state
- [ ] File name/URL truncated if too long
- [ ] Date formatted as relative time (e.g., "2 hours ago")
- [ ] Action buttons (View, Download, Cancel/Retry)
- [ ] Hover state on row
- [ ] Dark mode styling works
- [ ] Accessibility: Row has descriptive label

#### Technical Implementation
```tsx
// src/components/history/JobRow.tsx
import { TableRow, TableCell } from '@/components/ui/table';
import { Badge } from '@/components/ui/badge';
import { Button } from '@/components/ui/button';
import { Eye, Download, RefreshCw, XCircle } from 'lucide-react';
import { formatDistanceToNow } from 'date-fns';
import { cn } from '@/lib/utils';

interface Job {
  id: string;
  inputType: 'FILE' | 'URL';
  fileName?: string;
  url?: string;
  status: 'PENDING' | 'PROCESSING' | 'COMPLETED' | 'FAILED' | 'CANCELLED';
  createdAt: string;
  updatedAt: string;
}

interface JobRowProps {
  job: Job;
  onClick: () => void;
}

export function JobRow({ job, onClick }: JobRowProps) {
  const getStatusBadge = (status: Job['status']) => {
    const variants = {
      PENDING: { variant: 'secondary' as const, label: 'Pending', className: 'bg-yellow-500/10 text-yellow-500 border-yellow-500/20' },
      PROCESSING: { variant: 'default' as const, label: 'Processing', className: 'bg-blue-500/10 text-blue-500 border-blue-500/20' },
      COMPLETED: { variant: 'default' as const, label: 'Completed', className: 'bg-green-500/10 text-green-500 border-green-500/20' },
      FAILED: { variant: 'destructive' as const, label: 'Failed', className: 'bg-destructive/10 text-destructive border-destructive/20' },
      CANCELLED: { variant: 'outline' as const, label: 'Cancelled', className: 'bg-muted text-muted-foreground' },
    };

    const config = variants[status];

    return (
      <Badge variant="outline" className={cn('gap-1', config.className)}>
        <span className="inline-block h-2 w-2 rounded-full bg-current" aria-hidden="true" />
        {config.label}
      </Badge>
    );
  };

  const displayName = job.inputType === 'FILE'
    ? job.fileName || 'Unnamed file'
    : job.url || 'Unknown URL';

  const truncatedName = displayName.length > 50
    ? `${displayName.slice(0, 47)}...`
    : displayName;

  const relativeTime = formatDistanceToNow(new Date(job.createdAt), { addSuffix: true });

  return (
    <TableRow
      className="cursor-pointer hover:bg-muted/50 transition-colors"
      onClick={onClick}
      role="button"
      tabIndex={0}
      aria-label={`Job ${truncatedName}, status ${job.status}, created ${relativeTime}`}
      onKeyDown={(e) => {
        if (e.key === 'Enter' || e.key === ' ') {
          // Don't trigger row click if an interactive element inside the row is focused
          if (e.target === e.currentTarget) {
            e.preventDefault();
            onClick();
          }
        }
      }}
    >
      <TableCell className="font-medium">
        <div className="flex items-center gap-2">
          {job.inputType === 'FILE' ? (
            <span className="text-xs text-muted-foreground">FILE</span>
          ) : (
            <span className="text-xs text-muted-foreground">URL</span>
          )}
          <span className="truncate">{truncatedName}</span>
        </div>
      </TableCell>

      <TableCell>
        {getStatusBadge(job.status)}
      </TableCell>

      <TableCell className="text-muted-foreground">
        <time dateTime={job.createdAt}>
          {relativeTime}
        </time>
      </TableCell>

      <TableCell className="text-right">
        <div className="flex items-center justify-end gap-2">
          <Button
            variant="ghost"
            size="sm"
            onClick={(e) => {
              e.stopPropagation();
              onClick();
            }}
            aria-label={`View job ${truncatedName}`}
          >
            <Eye className="h-4 w-4" />
          </Button>

          {job.status === 'COMPLETED' && (
            <Button
              variant="ghost"
              size="sm"
              onClick={(e) => {
                e.stopPropagation();
                // Download functionality
              }}
              aria-label={`Download results for ${truncatedName}`}
            >
              <Download className="h-4 w-4" />
            </Button>
          )}

          {job.status === 'FAILED' && (
            <Button
              variant="ghost"
              size="sm"
              onClick={(e) => {
                e.stopPropagation();
                // Retry functionality
              }}
              aria-label={`Retry job ${truncatedName}`}
            >
              <RefreshCw className="h-4 w-4" />
            </Button>
          )}

          {job.status === 'PROCESSING' && (
            <Button
              variant="ghost"
              size="sm"
              onClick={(e) => {
                e.stopPropagation();
                // Cancel functionality (Sprint 1.7 Task 10)
              }}
              aria-label={`Cancel job ${truncatedName}`}
            >
              <XCircle className="h-4 w-4" />
            </Button>
          )}
        </div>
      </TableCell>
    </TableRow>
  );
}
```

**Status Badge Colors**:
| Status | Badge Color | Dot Color | Border |
|--------|-------------|-----------|--------|
| PENDING | Yellow tint | Yellow | Yellow |
| PROCESSING | Blue tint | Blue | Blue |
| COMPLETED | Green tint | Green | Green |
| FAILED | Red tint | Red | Red |
| CANCELLED | Gray tint | Gray | Gray |

**Action Button Logic**:
| Status | Actions Available |
|--------|-------------------|
| PENDING | View |
| PROCESSING | View, Cancel |
| COMPLETED | View, Download |
| FAILED | View, Retry |
| CANCELLED | View |

#### Validation
```bash
# Test job row functionality
# 1. View completed job → Green badge, View + Download buttons
# 2. View processing job → Blue badge, View + Cancel buttons
# 3. View failed job → Red badge, View + Retry buttons
# 4. Hover over row → Background highlights
# 5. Click row → Job detail modal opens
# 6. Click action button → Event fires, modal doesn't open
# 7. Keyboard: Tab to row, Enter to open
```

#### Deliverables
- JobRow component with status badge
- Action buttons (View, Download, Cancel, Retry)
- Formatted relative time display

---

### GOR-1.7-003: Implement Pagination Component
**Status**: Pending
**Effort**: 20 minutes
**Dependencies**: Sprint 1.7 Task 4 (Pagination component created)
**Priority**: P1 - High

#### Description
Create pagination controls using shadcn/ui Pagination component (or custom Button-based pagination). Support page numbers, previous/next navigation, and page jump.

#### Acceptance Criteria
- [ ] Previous/Next buttons
- [ ] Page numbers displayed (show 5 pages max with ellipsis)
- [ ] Current page highlighted
- [ ] Disabled state for first/last page boundaries
- [ ] Keyboard navigation (Tab, Enter)
- [ ] Dark mode styling works
- [ ] Accessibility: ARIA labels for navigation

#### Technical Implementation
```tsx
// src/components/history/Pagination.tsx
import { Button } from '@/components/ui/button';
import { ChevronLeft, ChevronRight, ChevronsLeft, ChevronsRight } from 'lucide-react';
import { cn } from '@/lib/utils';

interface PaginationProps {
  currentPage: number;
  totalPages: number;
  onPageChange: (page: number) => void;
}

export function Pagination({ currentPage, totalPages, onPageChange }: PaginationProps) {
  const getPageNumbers = () => {
    const pages: (number | 'ellipsis')[] = [];
    const showEllipsis = totalPages > 7;

    if (!showEllipsis) {
      // Show all pages if 7 or fewer
      return Array.from({ length: totalPages }, (_, i) => i + 1);
    }

    // Always show first page
    pages.push(1);

    if (currentPage > 3) {
      pages.push('ellipsis');
    }

    // Show pages around current page
    const start = Math.max(2, currentPage - 1);
    const end = Math.min(totalPages - 1, currentPage + 1);

    for (let i = start; i <= end; i++) {
      pages.push(i);
    }

    if (currentPage < totalPages - 2) {
      pages.push('ellipsis');
    }

    // Always show last page
    if (totalPages > 1) {
      pages.push(totalPages);
    }

    return pages;
  };

  const pageNumbers = getPageNumbers();

  return (
    <nav
      role="navigation"
      aria-label="Pagination navigation"
      className="flex items-center justify-center gap-1"
    >
      {/* First page button */}
      <Button
        variant="outline"
        size="icon"
        onClick={() => onPageChange(1)}
        disabled={currentPage === 1}
        aria-label="Go to first page"
      >
        <ChevronsLeft className="h-4 w-4" />
      </Button>

      {/* Previous page button */}
      <Button
        variant="outline"
        size="icon"
        onClick={() => onPageChange(currentPage - 1)}
        disabled={currentPage === 1}
        aria-label="Go to previous page"
      >
        <ChevronLeft className="h-4 w-4" />
      </Button>

      {/* Page numbers */}
      {pageNumbers.map((page, index) => (
        page === 'ellipsis' ? (
          <span key={`ellipsis-${index}`} className="px-2 text-muted-foreground">
            ...
          </span>
        ) : (
          <Button
            key={page}
            variant={currentPage === page ? 'default' : 'outline'}
            size="icon"
            onClick={() => onPageChange(page)}
            aria-label={`Go to page ${page}`}
            aria-current={currentPage === page ? 'page' : undefined}
            className={cn(
              currentPage === page && 'pointer-events-none'
            )}
          >
            {page}
          </Button>
        )
      ))}

      {/* Next page button */}
      <Button
        variant="outline"
        size="icon"
        onClick={() => onPageChange(currentPage + 1)}
        disabled={currentPage === totalPages}
        aria-label="Go to next page"
      >
        <ChevronRight className="h-4 w-4" />
      </Button>

      {/* Last page button */}
      <Button
        variant="outline"
        size="icon"
        onClick={() => onPageChange(totalPages)}
        disabled={currentPage === totalPages}
        aria-label="Go to last page"
      >
        <ChevronsRight className="h-4 w-4" />
      </Button>
    </nav>
  );
}
```

**Pagination Display Logic**:
| Total Pages | Display Pattern | Example |
|-------------|-----------------|---------|
| 1-7 pages | All pages | [1] [2] [3] [4] [5] [6] [7] |
| 8+ pages, current near start | 1 2 3 4 5 ... 20 | [1] [2] [3] [4] [5] ... [20] |
| 8+ pages, current in middle | 1 ... 9 10 11 ... 20 | [1] ... [9] [10] [11] ... [20] |
| 8+ pages, current near end | 1 ... 16 17 18 19 20 | [1] ... [16] [17] [18] [19] [20] |

**Default Pagination Settings** (per Sprint 1.7 Task 5):
- Default items per page: 20
- Max items per page: 50
- Total pages calculated from item count

#### Validation
```bash
# Test pagination functionality
# 1. View page 1 → Previous/First disabled
# 2. Click Next → Navigate to page 2
# 3. Click page 5 → Jump to page 5
# 4. Click Last → Navigate to last page, Next/Last disabled
# 5. Keyboard: Tab through buttons, Enter to navigate
# 6. Toggle dark mode → Pagination styling adapts
# 7. Screen reader: Announces current page
```

#### Deliverables
- Pagination component with page numbers
- Previous/Next/First/Last navigation
- Ellipsis for long page lists

---

### GOR-1.7-004: Create JobDetail Modal with Dialog Component
**Status**: Pending
**Effort**: 25 minutes
**Dependencies**: Sprint 1.7 Task 7 (JobDetail modal)
**Priority**: P1 - High

#### Description
Implement a modal dialog for viewing job details and results using shadcn/ui Dialog component. Integrate ResultsViewer from Sprint 1.6.

#### Acceptance Criteria
- [ ] Dialog component displays job details
- [ ] Modal shows file/URL, status, timestamps, results
- [ ] ResultsViewer embedded for completed jobs
- [ ] Close button and overlay click to dismiss
- [ ] Keyboard navigation (Esc to close, Tab trap)
- [ ] Dark mode styling works
- [ ] Accessibility: Focus trap, ARIA labels

#### Technical Implementation
```tsx
// src/components/history/JobDetail.tsx
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogHeader,
  DialogTitle,
  DialogTrigger,
} from '@/components/ui/dialog';
import { Badge } from '@/components/ui/badge';
import { Button } from '@/components/ui/button';
import { ResultsViewer } from '@/components/results/ResultsViewer';
import { Separator } from '@/components/ui/separator';
import { format } from 'date-fns';

interface JobDetailProps {
  job: {
    id: string;
    inputType: 'FILE' | 'URL';
    fileName?: string;
    url?: string;
    status: string;
    createdAt: string;
    updatedAt: string;
    results?: {
      markdown?: string;
      html?: string;
      json?: string;
      raw?: string;
    };
  };
  open: boolean;
  onOpenChange: (open: boolean) => void;
}

export function JobDetail({ job, open, onOpenChange }: JobDetailProps) {
  return (
    <Dialog open={open} onOpenChange={onOpenChange}>
      <DialogContent className="max-w-4xl max-h-[90vh] overflow-y-auto">
        <DialogHeader>
          <DialogTitle>Job Details</DialogTitle>
          <DialogDescription>
            {job.inputType === 'FILE' ? job.fileName : job.url}
          </DialogDescription>
        </DialogHeader>

        {/* Job metadata */}
        <div className="space-y-4">
          <div className="grid grid-cols-2 gap-4">
            <div>
              <p className="text-sm font-medium text-muted-foreground">Job ID</p>
              <p className="text-sm font-mono">{job.id}</p>
            </div>
            <div>
              <p className="text-sm font-medium text-muted-foreground">Status</p>
              <Badge variant="outline" className="mt-1">
                {job.status}
              </Badge>
            </div>
            <div>
              <p className="text-sm font-medium text-muted-foreground">Created</p>
              <p className="text-sm">
                {format(new Date(job.createdAt), 'PPpp')}
              </p>
            </div>
            <div>
              <p className="text-sm font-medium text-muted-foreground">Updated</p>
              <p className="text-sm">
                {format(new Date(job.updatedAt), 'PPpp')}
              </p>
            </div>
          </div>

          <Separator />

          {/* Results viewer (only for completed jobs) */}
          {job.status === 'COMPLETED' && job.results ? (
            <ResultsViewer
              jobId={job.id}
              results={job.results}
              fileName={job.fileName || job.url || 'Unknown'}
            />
          ) : job.status === 'FAILED' ? (
            <div className="rounded-md border border-destructive bg-destructive/10 p-4">
              <p className="text-sm text-destructive">
                Processing failed. Please try again or contact support.
              </p>
            </div>
          ) : job.status === 'PROCESSING' ? (
            <div className="rounded-md border border-blue-500 bg-blue-500/10 p-4">
              <p className="text-sm text-blue-500">
                Job is currently processing. Refresh to see updates.
              </p>
            </div>
          ) : (
            <div className="rounded-md border border-border bg-muted/30 p-4">
              <p className="text-sm text-muted-foreground">
                No results available for this job.
              </p>
            </div>
          )}
        </div>
      </DialogContent>
    </Dialog>
  );
}
```

**Modal Layout**:
```
┌─────────────────────────────────────────┐
│ Job Details                        [X]  │
│ document.pdf                            │
├─────────────────────────────────────────┤
│ Job ID: abc-123-def                     │
│ Status: ● Completed                     │
│ Created: Dec 12, 2025 10:30 AM         │
│ Updated: Dec 12, 2025 10:35 AM         │
├─────────────────────────────────────────┤
│ [ResultsViewer Component]               │
│ - Tabs: Markdown, HTML, JSON, Raw      │
│ - Download button                       │
└─────────────────────────────────────────┘
```

#### Validation
```bash
# Test modal functionality
# 1. Click job row → Modal opens
# 2. Verify: Job metadata displayed
# 3. Completed job: ResultsViewer shown
# 4. Processing job: "Processing" message shown
# 5. Failed job: Error message shown
# 6. Click overlay → Modal closes
# 7. Press Esc → Modal closes
# 8. Keyboard: Tab through modal, focus trapped
```

#### Deliverables
- JobDetail modal component
- ResultsViewer integration
- Status-based content display

---

## Summary

**Total Tasks**: 4
**Total Effort**: 1.58 hours
**Critical Path**: GOR-1.7-001 → GOR-1.7-002 → GOR-1.7-004

**Success Criteria**:
- [ ] Table component displays job history
- [ ] JobRow with status badge and action buttons
- [ ] Pagination with page numbers
- [ ] JobDetail modal with ResultsViewer
- [ ] Zero TypeScript errors
- [ ] WCAG 2.1 AA accessibility compliance
- [ ] Responsive design (mobile, tablet, desktop)

**Coordination Points**:
- **Neo (Sprint 1.7 Lead)**: Coordinate history API integration (Task 5)
- **Trinity (Database)**: Review job query performance for pagination
- **Ola (Accessibility)**: Review table and modal accessibility
- **Sophia (State Management)**: Review useHistory hook integration

**Anti-Patterns Prevented**:
- ✅ Named imports for lucide-react icons
- ✅ CSS variables for badge colors
- ✅ Proper table semantics (TableHeader, TableBody)
- ✅ ARIA labels for navigation
- ✅ TypeScript types for all props
- ✅ Focus trap in modal (Dialog primitive)

**Notes**:
- Table component uses semantic HTML for accessibility
- Status badges use consistent color coding across the application
- Pagination supports default 20 items per page (max 50)
- JobDetail modal reuses ResultsViewer from Sprint 1.6
- date-fns used for relative time formatting (already in dependencies)
- Dialog component requires installation: `npx shadcn@latest add dialog`
- Separator component requires installation: `npx shadcn@latest add separator`
