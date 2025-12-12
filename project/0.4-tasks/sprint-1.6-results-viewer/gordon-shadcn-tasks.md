# Tasks: shadcn/ui Components for Results Viewer (Sprint 1.6)

**Agent**: Gordon Zain (`@gordon`) - shadcn/ui Specialist
**Sprint**: 1.6 - Results Viewer & Display
**Duration**: ~0.75h
**Prerequisites**: Sprint 1.5b complete, SSE progress working

## MCP Integration

**hx-shadcn MCP Server**: Use the MCP server for tabbed interface and download components.

**Service Reference**: `project/0.8-references/hx-shadcn-service-descriptor.md`
**Endpoint**: `http://hx-shadcn.hx.dev.local:7423/sse`

**MCP Tools for This Sprint**:
| Tool | Usage in Sprint 1.6 |
|------|---------------------|
| `get_component` | Retrieve Tabs, Badge, DropdownMenu, Skeleton source |
| `get_component_demo` | View Tabs component usage patterns for multi-format display |
| `get_component_metadata` | Review DropdownMenu props for download format selection |

**Example: Retrieve Tabs Component with Demo**
```json
{
  "method": "tools/call",
  "params": {
    "name": "get_component_demo",
    "arguments": {"componentName": "tabs"}
  }
}
```

**Component Installation Notes**:
- Tabs and Skeleton pre-installed in Sprint 1.1
- Badge: `npx shadcn@latest add badge` (if not already added)
- DropdownMenu: `npx shadcn@latest add dropdown-menu`
- Use MCP server to preview components before installation

## Task List

### GOR-1.6-001: Implement ResultsViewer with shadcn/ui Tabs Component
**Status**: Pending
**Effort**: 25 minutes
**Dependencies**: Sprint 1.6 Task 1 (ResultsViewer component created)
**Priority**: P0 - Critical Path

#### Description
Create a tabbed interface for viewing conversion results in multiple formats (Markdown, HTML, JSON, Raw) using shadcn/ui Tabs component. Ensure tab state persistence and smooth transitions.

#### Acceptance Criteria
- [ ] Tabs component displays four tabs: Markdown, HTML, JSON, Raw
- [ ] Default tab is Markdown (as per specification)
- [ ] Tab switching preserves content (no re-fetching)
- [ ] Active tab visually distinct (highlight, underline)
- [ ] Tab content area has consistent height and scrolling
- [ ] Dark mode styling works
- [ ] Keyboard navigation (Arrow keys to switch tabs)
- [ ] Accessibility: ARIA labels, screen reader announces tab changes

#### Technical Implementation
```tsx
// src/components/results/ResultsViewer.tsx
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
import { MarkdownView } from './MarkdownView';
import { HtmlView } from './HtmlView';
import { JsonView } from './JsonView';
import { RawView } from './RawView';
import { DownloadButton } from './DownloadButton';
import { Badge } from '@/components/ui/badge';

interface ResultsViewerProps {
  jobId: string;
  results: {
    markdown?: string;
    html?: string;
    json?: string;
    raw?: string;
  };
  partialResults?: {
    markdown: boolean;
    html: boolean;
    json: boolean;
  };
  fileName: string;
}

export function ResultsViewer({ jobId, results, partialResults, fileName }: ResultsViewerProps) {
  return (
    <Card>
      <CardHeader>
        <div className="flex items-center justify-between">
          <div className="space-y-1">
            <CardTitle>Conversion Results</CardTitle>
            <CardDescription>
              {fileName}
              {partialResults && (
                <Badge variant="outline" className="ml-2">
                  Partial Results
                </Badge>
              )}
            </CardDescription>
          </div>
          <DownloadButton
            jobId={jobId}
            results={results}
            fileName={fileName}
          />
        </div>
      </CardHeader>
      <CardContent>
        <Tabs defaultValue="markdown" className="w-full">
          <TabsList className="grid w-full grid-cols-4">
            <TabsTrigger value="markdown" className="relative">
              Markdown
              {partialResults?.markdown === false && (
                <span className="ml-1 inline-block h-2 w-2 rounded-full bg-destructive" aria-label="Export failed" />
              )}
            </TabsTrigger>
            <TabsTrigger value="html" className="relative">
              HTML
              {partialResults?.html === false && (
                <span className="ml-1 inline-block h-2 w-2 rounded-full bg-destructive" aria-label="Export failed" />
              )}
            </TabsTrigger>
            <TabsTrigger value="json" className="relative">
              JSON
              {partialResults?.json === false && (
                <span className="ml-1 inline-block h-2 w-2 rounded-full bg-destructive" aria-label="Export failed" />
              )}
            </TabsTrigger>
            <TabsTrigger value="raw">
              Raw
            </TabsTrigger>
          </TabsList>

          <div className="mt-4 min-h-[400px] max-h-[600px] overflow-auto rounded-md border border-border bg-muted/30 p-4">
            <TabsContent value="markdown" className="mt-0">
              {results.markdown ? (
                <MarkdownView content={results.markdown} />
              ) : (
                <EmptyState message="Markdown export not available" />
              )}
            </TabsContent>

            <TabsContent value="html" className="mt-0">
              {results.html ? (
                <HtmlView content={results.html} />
              ) : (
                <EmptyState message="HTML export not available" />
              )}
            </TabsContent>

            <TabsContent value="json" className="mt-0">
              {results.json ? (
                <JsonView content={results.json} />
              ) : (
                <EmptyState message="JSON export not available" />
              )}
            </TabsContent>

            <TabsContent value="raw" className="mt-0">
              {results.raw ? (
                <RawView content={results.raw} />
              ) : (
                <EmptyState message="Raw output not available" />
              )}
            </TabsContent>
          </div>
        </Tabs>
      </CardContent>
    </Card>
  );
}

// Empty state component
function EmptyState({ message }: { message: string }) {
  return (
    <div className="flex h-[200px] items-center justify-center text-muted-foreground">
      <p>{message}</p>
    </div>
  );
}
```

**Tab Layout**:
```
┌─────────────────────────────────────────────────┐
│ Conversion Results            [Download Button] │
│ document.pdf                  [Partial Results] │
├─────────────────────────────────────────────────┤
│ [Markdown] [HTML] [JSON] [Raw]                  │
├─────────────────────────────────────────────────┤
│                                                 │
│  Tab Content Area (scrollable)                  │
│  min-height: 400px                              │
│  max-height: 600px                              │
│                                                 │
└─────────────────────────────────────────────────┘
```

**Partial Results Indicator** (per FR-405):
- Red dot next to tab if export failed
- "Partial Results" badge in header
- Empty state message if content unavailable

#### Validation
```bash
# Test tab functionality
# 1. Load results page → Markdown tab active by default
# 2. Click HTML tab → Content switches, Markdown preserved
# 3. Click JSON tab → JSON content displays
# 4. Click Raw tab → Raw text displays
# 5. Click Markdown tab → Returns to Markdown (no re-fetch)
# 6. Keyboard: Arrow keys switch tabs
# 7. Toggle dark mode → Tabs styling adapts
# 8. Screen reader: Announces tab changes
```

#### Deliverables
- ResultsViewer component with Tabs
- Partial results indicators
- Empty state handling

---

### GOR-1.6-002: Style Download Button with shadcn/ui Button Component
**Status**: Pending
**Effort**: 15 minutes
**Dependencies**: Sprint 1.6 Task 6 (DownloadButton component created)
**Priority**: P1 - High

#### Description
Create a download button with format selection dropdown using shadcn/ui Button and DropdownMenu components. Allow users to download results in their preferred format.

#### Acceptance Criteria
- [ ] Button component with download icon
- [ ] Dropdown menu for format selection
- [ ] Formats: Markdown (.md), HTML (.html), JSON (.json), Raw (.txt)
- [ ] Download triggers browser download with correct filename
- [ ] Disabled state when no results available
- [ ] Dark mode styling works
- [ ] Accessibility: Keyboard navigation, ARIA labels

#### Technical Implementation
```tsx
// src/components/results/DownloadButton.tsx
import { Button } from '@/components/ui/button';
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuLabel,
  DropdownMenuSeparator,
  DropdownMenuTrigger,
} from '@/components/ui/dropdown-menu';
import { Download, FileText, FileCode, FileJson } from 'lucide-react';

interface DownloadButtonProps {
  jobId: string;
  results: {
    markdown?: string;
    html?: string;
    json?: string;
    raw?: string;
  };
  fileName: string;
}

export function DownloadButton({ jobId, results, fileName }: DownloadButtonProps) {
  const downloadFile = (content: string, format: string, extension: string) => {
    // Sprint 1.6 Task 7: Download naming convention
    const timestamp = new Date().toISOString().replace(/[:.]/g, '-').slice(0, -5);
    const baseFileName = fileName.replace(/\.[^/.]+$/, ''); // Remove original extension
    const downloadFileName = `${baseFileName}_${timestamp}.${extension}`;

    const blob = new Blob([content], { type: 'text/plain' });
    const url = URL.createObjectURL(blob);
    const link = document.createElement('a');
    link.href = url;
    link.download = downloadFileName;
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
    URL.revokeObjectURL(url);
  };

  const hasResults = Object.values(results).some(Boolean);

  return (
    <DropdownMenu>
      <DropdownMenuTrigger asChild>
        <Button variant="outline" disabled={!hasResults} aria-label="Download results">
          <Download className="mr-2 h-4 w-4" />
          Download
        </Button>
      </DropdownMenuTrigger>
      <DropdownMenuContent align="end" className="w-48">
        <DropdownMenuLabel>Select Format</DropdownMenuLabel>
        <DropdownMenuSeparator />

        <DropdownMenuItem
          disabled={!results.markdown}
          onClick={() => results.markdown && downloadFile(results.markdown, 'Markdown', 'md')}
        >
          <FileText className="mr-2 h-4 w-4" />
          Markdown (.md)
        </DropdownMenuItem>

        <DropdownMenuItem
          disabled={!results.html}
          onClick={() => results.html && downloadFile(results.html, 'HTML', 'html')}
        >
          <FileCode className="mr-2 h-4 w-4" />
          HTML (.html)
        </DropdownMenuItem>

        <DropdownMenuItem
          disabled={!results.json}
          onClick={() => results.json && downloadFile(results.json, 'JSON', 'json')}
        >
          <FileJson className="mr-2 h-4 w-4" />
          JSON (.json)
        </DropdownMenuItem>

        <DropdownMenuItem
          disabled={!results.raw}
          onClick={() => results.raw && downloadFile(results.raw, 'Plain Text', 'txt')}
        >
          <FileText className="mr-2 h-4 w-4" />
          Raw (.txt)
        </DropdownMenuItem>
      </DropdownMenuContent>
    </DropdownMenu>
  );
}
```

**Download Filename Convention** (per Sprint 1.6 Task 7):
```
Format: {original_filename}_{timestamp}.{extension}

Examples:
- document_2025-12-12T10-30-45.md
- report_2025-12-12T10-30-45.html
- data_2025-12-12T10-30-45.json
- output_2025-12-12T10-30-45.txt
```

#### Validation
```bash
# Test download functionality
# 1. Click Download button → Dropdown opens
# 2. Click "Markdown (.md)" → File downloads
# 3. Verify filename: {original}_{timestamp}.md
# 4. Open downloaded file → Content correct
# 5. Try other formats (HTML, JSON, Raw)
# 6. Partial results: Unavailable formats disabled (gray)
# 7. Keyboard: Arrow keys navigate dropdown
```

#### Deliverables
- DownloadButton component with dropdown
- File download functionality
- Filename timestamp formatting

---

### GOR-1.6-003: Add Skeleton Loaders for Loading States
**Status**: Pending
**Effort**: 15 minutes
**Dependencies**: Sprint 1.6 Task 8 (Skeleton loaders)
**Priority**: P1 - High

#### Description
Implement skeleton loading states using shadcn/ui Skeleton component for each tab while results are loading. Provide smooth loading experience.

#### Acceptance Criteria
- [ ] Skeleton component displays in each tab during loading
- [ ] Skeleton matches content layout (lines for text, blocks for code)
- [ ] Skeleton animates (shimmer effect)
- [ ] Different skeleton layouts for each format
- [ ] Dark mode styling works
- [ ] Accessibility: Screen reader announces "Loading results"

#### Technical Implementation
```tsx
// src/components/results/ResultsSkeleton.tsx
import { Skeleton } from '@/components/ui/skeleton';
import { Card, CardContent, CardHeader } from '@/components/ui/card';

export function MarkdownSkeleton() {
  return (
    <div className="space-y-4" role="status" aria-label="Loading markdown content">
      <Skeleton className="h-8 w-3/4" /> {/* Heading */}
      <Skeleton className="h-4 w-full" />  {/* Paragraph line 1 */}
      <Skeleton className="h-4 w-5/6" />  {/* Paragraph line 2 */}
      <Skeleton className="h-4 w-4/6" />  {/* Paragraph line 3 */}
      <Skeleton className="h-6 w-2/4 mt-6" /> {/* Subheading */}
      <Skeleton className="h-4 w-full" />
      <Skeleton className="h-4 w-4/5" />
      <Skeleton className="h-32 w-full mt-4" /> {/* Code block */}
    </div>
  );
}

export function JsonSkeleton() {
  return (
    <div className="space-y-2" role="status" aria-label="Loading JSON content">
      <Skeleton className="h-4 w-24" /> {/* { */}
      <Skeleton className="h-4 w-48 ml-4" /> {/* "key": "value", */}
      <Skeleton className="h-4 w-56 ml-4" />
      <Skeleton className="h-4 w-40 ml-4" />
      <Skeleton className="h-4 w-32 ml-8" /> {/* Nested object */}
      <Skeleton className="h-4 w-36 ml-8" />
      <Skeleton className="h-4 w-24" /> {/* } */}
    </div>
  );
}

export function TableSkeleton() {
  return (
    <div className="space-y-2" role="status" aria-label="Loading table content">
      <Skeleton className="h-10 w-full" /> {/* Table header */}
      <Skeleton className="h-8 w-full" />  {/* Row 1 */}
      <Skeleton className="h-8 w-full" />  {/* Row 2 */}
      <Skeleton className="h-8 w-full" />  {/* Row 3 */}
      <Skeleton className="h-8 w-full" />  {/* Row 4 */}
    </div>
  );
}

// Usage in ResultsViewer
export function ResultsViewer({ jobId, results, isLoading }: ResultsViewerProps) {
  return (
    <Tabs defaultValue="markdown">
      <TabsContent value="markdown">
        {isLoading ? (
          <MarkdownSkeleton />
        ) : results.markdown ? (
          <MarkdownView content={results.markdown} />
        ) : (
          <EmptyState message="Markdown export not available" />
        )}
      </TabsContent>
      {/* ... other tabs */}
    </Tabs>
  );
}
```

**Skeleton Layouts**:
| Format | Skeleton Pattern |
|--------|------------------|
| Markdown | Heading + paragraphs + code block |
| HTML | Similar to Markdown (structural) |
| JSON | Nested key-value pairs |
| Raw | Simple text lines |

#### Validation
```bash
# Test loading states
# 1. Start processing → Skeleton appears
# 2. Observe: Shimmer animation on skeleton
# 3. Results arrive → Skeleton replaced with content
# 4. Toggle tabs → Each tab has appropriate skeleton
# 5. Dark mode → Skeleton adapts to dark background
# 6. Screen reader: Announces "Loading markdown content"
```

#### Deliverables
- Skeleton components for each format
- Loading state integration
- Shimmer animation

---

### GOR-1.6-004: Add Badge Component for Partial Results Indicator
**Status**: Pending
**Effort**: 10 minutes
**Dependencies**: Sprint 1.6 Task 9 (Partial result indicators)
**Priority**: P1 - High

#### Description
Use shadcn/ui Badge component to indicate partial results when some export formats fail. Provide clear visual feedback per FR-405.

#### Acceptance Criteria
- [ ] Badge component shows "Partial Results" in header
- [ ] Badge uses "outline" or "secondary" variant
- [ ] Red dot indicator on failed export tabs
- [ ] Tooltip explains which exports failed (optional)
- [ ] Badge only appears when partialResults prop is true
- [ ] Dark mode styling works
- [ ] Accessibility: Badge text read by screen readers

#### Technical Implementation
```tsx
// Enhanced ResultsViewer with partial results badge
import { Badge } from '@/components/ui/badge';
import { Alert, AlertDescription } from '@/components/ui/alert';
import { AlertCircle } from 'lucide-react';

export function ResultsViewer({ jobId, results, partialResults, fileName }: ResultsViewerProps) {
  const failedExports = partialResults
    ? Object.entries(partialResults)
        .filter(([_, success]) => !success)
        .map(([format]) => format)
    : [];

  return (
    <Card>
      <CardHeader>
        <div className="space-y-2">
          <div className="flex items-center justify-between">
            <CardTitle>Conversion Results</CardTitle>
            <DownloadButton jobId={jobId} results={results} fileName={fileName} />
          </div>
          <div className="flex items-center gap-2">
            <CardDescription>{fileName}</CardDescription>
            {failedExports.length > 0 && (
              <Badge variant="outline" className="bg-destructive/10 text-destructive border-destructive">
                Partial Results
              </Badge>
            )}
          </div>

          {/* Alert for failed exports */}
          {failedExports.length > 0 && (
            <Alert variant="destructive">
              <AlertCircle className="h-4 w-4" />
              <AlertDescription>
                Some exports failed: {failedExports.join(', ')}. Available formats can still be downloaded.
              </AlertDescription>
            </Alert>
          )}
        </div>
      </CardHeader>
      {/* ... rest of component */}
    </Card>
  );
}
```

**Badge Variants**:
| State | Badge Text | Variant | Color |
|-------|-----------|---------|-------|
| All exports succeeded | None | - | - |
| Some exports failed | "Partial Results" | outline | Destructive |
| All exports failed | "Processing Failed" | destructive | Red |

#### Validation
```bash
# Test partial results indicators
# 1. Process file with partial success
# 2. Observe: "Partial Results" badge in header
# 3. Observe: Red dots on failed export tabs
# 4. Observe: Alert message listing failed exports
# 5. Click failed tab → Empty state message
# 6. Click successful tab → Content displays
# 7. Download → Only successful formats enabled
```

#### Deliverables
- Badge component for partial results
- Alert message for failed exports
- Red dot indicators on tabs

---

## Summary

**Total Tasks**: 4
**Total Effort**: 1.08 hours
**Critical Path**: GOR-1.6-001 → GOR-1.6-002

**Success Criteria**:
- [ ] Tabs component for result formats
- [ ] Download button with format selection
- [ ] Skeleton loaders for loading states
- [ ] Partial results indicators (badge, red dots)
- [ ] Zero TypeScript errors
- [ ] WCAG 2.1 AA accessibility compliance
- [ ] Dark mode styling works

**Coordination Points**:
- **Neo (Sprint 1.6 Lead)**: Coordinate ResultsViewer implementation (Tasks 1-5)
- **Ola (Accessibility)**: Review keyboard navigation and screen reader support
- **William (TailwindCSS)**: Verify custom skeleton animations if needed

**Anti-Patterns Prevented**:
- ✅ Named imports for lucide-react icons
- ✅ CSS variables for badge colors
- ✅ Proper loading states with Skeleton
- ✅ ARIA labels for accessibility
- ✅ TypeScript types for props

**Notes**:
- Tabs component uses Radix UI primitives (built-in accessibility)
- Partial results handling per FR-405 specification
- Download functionality uses browser Blob API
- Skeleton components match content layout for smooth transitions
- DropdownMenu requires installation: `npx shadcn@latest add dropdown-menu`
