# Sprint 1.6: Frontend UI Tasks - Results Viewer & Display

**Agent**: Ola Mae Johnson (`@ola`)
**Role**: Frontend UI Developer & Accessibility Specialist
**Sprint**: 1.6 - Results Viewer & Display
**Dependencies**: Sprint 1.5b Complete
**Effort**: 1.25h (of 2.25h total sprint)

---

## Task Overview

| Task ID | Title | Effort | Status |
|---------|-------|--------|--------|
| OLA-1.6-001 | Create ResultsViewer with Tabbed Interface | 35m | Pending |
| OLA-1.6-002 | Implement Format-Specific View Components | 40m | Pending |
| OLA-1.6-003 | Create Download Button Component | 20m | Pending |
| OLA-1.6-004 | Add Partial Result Indicators | 20m | Pending |
| OLA-1.6-005 | Validate Responsive Design and Accessibility | 20m | Pending |

**Total Effort**: 1.25h (75 minutes)

---

## OLA-1.6-001: Create ResultsViewer with Tabbed Interface

### Description

Build the main results viewer component with a tabbed interface using shadcn/ui Tabs. Displays four tabs: Markdown, HTML, JSON, and Raw output formats.

### Acceptance Criteria

- [ ] Component created with shadcn/ui Tabs component
- [ ] Four tabs: Markdown (default), HTML, JSON, Raw
- [ ] Tab switching preserves content (no re-rendering)
- [ ] Active tab visually distinct
- [ ] Keyboard navigation between tabs (Arrow keys)
- [ ] URL hash parameter reflects active tab (#markdown, #html, #json, #raw)
- [ ] Tab content area scrollable with max height
- [ ] Empty state shown when no results available
- [ ] Loading skeleton shown while results load
- [ ] Accessibility: tabs accessible via keyboard, ARIA roles

### Dependencies

- Sprint 1.5b: ProcessingView complete
- Sprint 1.1: shadcn/ui Tabs component installed

### Deliverables

**File**: `src/components/results/ResultsViewer.tsx`

```typescript
'use client'

import { useState, useEffect } from 'react'
import { useRouter, useSearchParams } from 'next/navigation'
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs'
import { Card, CardContent } from '@/components/ui/card'
import { MarkdownView } from './MarkdownView'
import { HtmlView } from './HtmlView'
import { JsonView } from './JsonView'
import { RawView } from './RawView'
import { DownloadButton } from './DownloadButton'
import { PartialResultBadge } from './PartialResultBadge'
import { ResultsSkeleton } from './ResultsSkeleton'
import { Alert, AlertDescription } from '@/components/ui/alert'
import { FileX } from 'lucide-react'

export interface JobResult {
  jobId: string
  markdown?: string
  html?: string
  json?: any
  rawDocument?: any
  filename: string
  createdAt: string
  status: 'COMPLETED' | 'PARTIAL_COMPLETE'
  failedExports?: string[] // For partial results
}

interface ResultsViewerProps {
  result: JobResult | null
  isLoading?: boolean
}

type TabValue = 'markdown' | 'html' | 'json' | 'raw'

export function ResultsViewer({ result, isLoading }: ResultsViewerProps) {
  const router = useRouter()
  const searchParams = useSearchParams()
  const [activeTab, setActiveTab] = useState<TabValue>('markdown')

  // Sync tab with URL hash
  useEffect(() => {
    const tab = searchParams.get('tab') as TabValue
    if (tab && ['markdown', 'html', 'json', 'raw'].includes(tab)) {
      setActiveTab(tab)
    }
  }, [searchParams])

  const handleTabChange = (value: string) => {
    const newTab = value as TabValue
    setActiveTab(newTab)
    // Update URL without page reload
    const params = new URLSearchParams(searchParams.toString())
    params.set('tab', newTab)
    router.push(`?${params.toString()}`, { scroll: false })
  }

  if (isLoading) {
    return <ResultsSkeleton />
  }

  if (!result) {
    return (
      <Card>
        <CardContent className="p-12">
          <div className="flex flex-col items-center gap-4 text-center">
            <FileX className="h-16 w-16 text-muted-foreground" />
            <div>
              <h3 className="text-lg font-semibold mb-2">No Results Available</h3>
              <p className="text-muted-foreground">
                Process a document to see results here.
              </p>
            </div>
          </div>
        </CardContent>
      </Card>
    )
  }

  const isPartialResult = result.status === 'PARTIAL_COMPLETE'

  return (
    <div className="space-y-4">
      {/* Partial Result Warning */}
      {isPartialResult && result.failedExports && (
        <Alert>
          <AlertDescription>
            <PartialResultBadge failedExports={result.failedExports} />
          </AlertDescription>
        </Alert>
      )}

      {/* Tabs */}
      <Tabs value={activeTab} onValueChange={handleTabChange}>
        <div className="flex items-center justify-between mb-4">
          <TabsList>
            <TabsTrigger value="markdown" disabled={!result.markdown}>
              Markdown
            </TabsTrigger>
            <TabsTrigger value="html" disabled={!result.html}>
              HTML
            </TabsTrigger>
            <TabsTrigger value="json" disabled={!result.json}>
              JSON
            </TabsTrigger>
            <TabsTrigger value="raw" disabled={!result.rawDocument}>
              Raw
            </TabsTrigger>
          </TabsList>

          <DownloadButton result={result} activeFormat={activeTab} />
        </div>

        <Card>
          <CardContent className="p-0">
            <TabsContent value="markdown" className="m-0">
              <MarkdownView content={result.markdown} />
            </TabsContent>

            <TabsContent value="html" className="m-0">
              <HtmlView content={result.html} />
            </TabsContent>

            <TabsContent value="json" className="m-0">
              <JsonView content={result.json} />
            </TabsContent>

            <TabsContent value="raw" className="m-0">
              <RawView content={result.rawDocument} />
            </TabsContent>
          </CardContent>
        </Card>
      </Tabs>
    </div>
  )
}
```

### Technical Notes

- **Reference**: Specification FR-601, Implementation Plan Sprint 1.6 Task 1
- **Default Tab**: Markdown (most user-friendly format)
- **Tab Persistence**: Active tab stored in URL query parameter
- **Disabled Tabs**: Tabs disabled if export failed (partial results)
- **Keyboard Navigation**: shadcn/ui Tabs supports arrow key navigation
- **Content Preservation**: TabsContent uses CSS `hidden` instead of unmounting
- **Accessibility**:
  - `TabsList` has `role="tablist"`
  - `TabsTrigger` has `role="tab"`, `aria-selected`, `aria-controls`
  - `TabsContent` has `role="tabpanel"`, `aria-labelledby`
  - Disabled tabs have `aria-disabled="true"`

### Testing

```bash
# Unit test
npm run test src/components/results/ResultsViewer.test.tsx

# Manual testing:
# [ ] Click tabs - content switches correctly
# [ ] Arrow keys navigate between tabs
# [ ] URL updates when tab changes
# [ ] Reload page with ?tab=html - HTML tab active
# [ ] Disabled tabs (partial results) not selectable
# [ ] Empty state shows when no result
# [ ] Loading skeleton shows when isLoading=true
# [ ] Download button renders correctly
# [ ] Partial result badge shows for PARTIAL_COMPLETE
# [ ] Screen reader announces tab changes
# [ ] Dark mode renders correctly
```

---

## OLA-1.6-002: Implement Format-Specific View Components

### Description

Create specialized viewer components for each output format: Markdown with syntax highlighting, HTML in sandboxed iframe, JSON with collapsible tree, and Raw with copy button.

### Acceptance Criteria

- [ ] MarkdownView renders GitHub Flavored Markdown with syntax highlighting
- [ ] HtmlView displays HTML in sandboxed iframe (no script execution)
- [ ] JsonView shows pretty-printed JSON with collapsible tree structure
- [ ] RawView displays raw text with copy-to-clipboard button
- [ ] All views handle null/undefined content gracefully
- [ ] All views scrollable with max-height constraint
- [ ] Syntax highlighting works in dark mode
- [ ] Accessibility: code blocks have language labels, copy button accessible

### Dependencies

- OLA-1.6-001: ResultsViewer complete
- Sprint 1.1: Markdown and JSON libraries installed

### Deliverables

**File 1**: `src/components/results/MarkdownView.tsx`

```typescript
'use client'

import ReactMarkdown from 'react-markdown'
import remarkGfm from 'remark-gfm'
import { Prism as SyntaxHighlighter } from 'react-syntax-highlighter'
import { vscDarkPlus, vs } from 'react-syntax-highlighter/dist/esm/styles/prism'
import { useTheme } from 'next-themes'

interface MarkdownViewProps {
  content?: string
}

export function MarkdownView({ content }: MarkdownViewProps) {
  const { theme } = useTheme()

  if (!content) {
    return (
      <div className="p-8 text-center text-muted-foreground">
        No markdown content available
      </div>
    )
  }

  return (
    <div className="prose prose-sm dark:prose-invert max-w-none p-6 overflow-auto max-h-[600px]">
      <ReactMarkdown
        remarkPlugins={[remarkGfm]}
        components={{
          code({ node, inline, className, children, ...props }) {
            const match = /language-(\w+)/.exec(className || '')
            return !inline && match ? (
              <SyntaxHighlighter
                style={theme === 'dark' ? vscDarkPlus : vs}
                language={match[1]}
                PreTag="div"
                {...props}
              >
                {String(children).replace(/\n$/, '')}
              </SyntaxHighlighter>
            ) : (
              <code className={className} {...props}>
                {children}
              </code>
            )
          },
        }}
      >
        {content}
      </ReactMarkdown>
    </div>
  )
}
```

**File 2**: `src/components/results/HtmlView.tsx`

```typescript
'use client'

interface HtmlViewProps {
  content?: string
}

export function HtmlView({ content }: HtmlViewProps) {
  if (!content) {
    return (
      <div className="p-8 text-center text-muted-foreground">
        No HTML content available
      </div>
    )
  }

  return (
    <div className="p-6 overflow-auto max-h-[600px]">
      <iframe
        srcDoc={content}
        sandbox="allow-same-origin" // NO allow-scripts for security
        className="w-full min-h-[500px] border-0 bg-background"
        title="HTML Preview"
        aria-label="HTML content preview in sandboxed iframe"
      />
    </div>
  )
}
```

**File 3**: `src/components/results/JsonView.tsx`

```typescript
'use client'

import { useState } from 'react'
import { ChevronRight, ChevronDown } from 'lucide-react'
import { cn } from '@/lib/utils'

interface JsonViewProps {
  content?: any
}

export function JsonView({ content }: JsonViewProps) {
  if (!content) {
    return (
      <div className="p-8 text-center text-muted-foreground">
        No JSON content available
      </div>
    )
  }

  return (
    <div className="p-6 overflow-auto max-h-[600px] font-mono text-sm">
      <JsonTreeNode data={content} />
    </div>
  )
}

interface JsonTreeNodeProps {
  data: any
  depth?: number
}

function JsonTreeNode({ data, depth = 0 }: JsonTreeNodeProps) {
  const [isExpanded, setIsExpanded] = useState(depth < 2) // Auto-expand first 2 levels

  if (data === null) {
    return <span className="text-muted-foreground">null</span>
  }

  if (typeof data !== 'object') {
    return (
      <span className={cn(
        typeof data === 'string' && 'text-green-600 dark:text-green-400',
        typeof data === 'number' && 'text-blue-600 dark:text-blue-400',
        typeof data === 'boolean' && 'text-orange-600 dark:text-orange-400'
      )}>
        {JSON.stringify(data)}
      </span>
    )
  }

  const isArray = Array.isArray(data)
  const entries = Object.entries(data)

  if (entries.length === 0) {
    return <span className="text-muted-foreground">{isArray ? '[]' : '{}'}</span>
  }

  return (
    <div>
      <button
        onClick={() => setIsExpanded(!isExpanded)}
        className="inline-flex items-center hover:bg-accent rounded px-1"
        aria-expanded={isExpanded}
      >
        {isExpanded ? (
          <ChevronDown className="h-3 w-3 mr-1" />
        ) : (
          <ChevronRight className="h-3 w-3 mr-1" />
        )}
        <span className="text-muted-foreground">
          {isArray ? '[' : '{'} {entries.length} items {isArray ? ']' : '}'}
        </span>
      </button>

      {isExpanded && (
        <div className="ml-4 mt-1 border-l border-border pl-2">
          {entries.map(([key, value], index) => (
            <div key={key} className="py-0.5">
              {!isArray && (
                <span className="text-purple-600 dark:text-purple-400 mr-2">
                  "{key}":
                </span>
              )}
              <JsonTreeNode data={value} depth={depth + 1} />
              {index < entries.length - 1 && ','}
            </div>
          ))}
        </div>
      )}
    </div>
  )
}
```

**File 4**: `src/components/results/RawView.tsx`

```typescript
'use client'

import { useState } from 'react'
import { Button } from '@/components/ui/button'
import { Copy, Check } from 'lucide-react'

interface RawViewProps {
  content?: any
}

export function RawView({ content }: RawViewProps) {
  const [copied, setCopied] = useState(false)

  if (!content) {
    return (
      <div className="p-8 text-center text-muted-foreground">
        No raw content available
      </div>
    )
  }

  const rawText = typeof content === 'string'
    ? content
    : JSON.stringify(content, null, 2)

  const handleCopy = async () => {
    await navigator.clipboard.writeText(rawText)
    setCopied(true)
    setTimeout(() => setCopied(false), 2000)
  }

  return (
    <div className="p-6">
      <div className="flex items-center justify-between mb-4">
        <p className="text-sm text-muted-foreground">
          {rawText.split('\n').length} lines, {rawText.length} characters
        </p>
        <Button
          variant="outline"
          size="sm"
          onClick={handleCopy}
          aria-label={copied ? 'Copied to clipboard' : 'Copy to clipboard'}
        >
          {copied ? (
            <>
              <Check className="h-4 w-4 mr-1" />
              Copied
            </>
          ) : (
            <>
              <Copy className="h-4 w-4 mr-1" />
              Copy
            </>
          )}
        </Button>
      </div>
      <pre className="p-4 bg-muted rounded-lg overflow-auto max-h-[600px] text-sm font-mono">
        <code>{rawText}</code>
      </pre>
    </div>
  )
}
```

**File 5**: `src/components/results/ResultsSkeleton.tsx`

```typescript
import { Skeleton } from '@/components/ui/skeleton'
import { Card, CardContent } from '@/components/ui/card'

export function ResultsSkeleton() {
  return (
    <div className="space-y-4">
      <div className="flex items-center justify-between">
        <Skeleton className="h-10 w-64" />
        <Skeleton className="h-10 w-32" />
      </div>
      <Card>
        <CardContent className="p-6 space-y-4">
          <Skeleton className="h-6 w-3/4" />
          <Skeleton className="h-6 w-full" />
          <Skeleton className="h-6 w-5/6" />
          <Skeleton className="h-6 w-full" />
          <Skeleton className="h-6 w-2/3" />
        </CardContent>
      </Card>
    </div>
  )
}
```

### Technical Notes

- **Reference**: Specification FR-601, Implementation Plan Sprint 1.6 Tasks 2-5, 8
- **Markdown**: `react-markdown@^9.x` with `remark-gfm` for GitHub Flavored Markdown
- **Syntax Highlighting**: `react-syntax-highlighter` with Prism
- **HTML Sandbox**: `sandbox="allow-same-origin"` prevents script execution
- **JSON Tree**: Custom collapsible component (no external library needed)
- **Copy Button**: Uses Clipboard API with fallback
- **Max Height**: All views constrained to 600px with scroll
- **Dark Mode**: All syntax highlighting adapts to theme

### Testing

```bash
# Unit tests
npm run test src/components/results/*.test.tsx

# Manual testing:
# [ ] Markdown renders with headings, lists, code blocks
# [ ] Code blocks have syntax highlighting
# [ ] HTML renders in iframe without scripts executing
# [ ] JSON tree collapses/expands nodes
# [ ] JSON tree shows types with colors
# [ ] Raw view copy button copies to clipboard
# [ ] Raw view shows line/character count
# [ ] All views handle null content gracefully
# [ ] All views scrollable when content exceeds 600px
# [ ] Dark mode syntax highlighting works
# [ ] Screen reader reads content correctly
```

---

## OLA-1.6-003: Create Download Button Component

### Description

Build a download button with format selection dropdown. Generates correctly named files for each export format.

### Acceptance Criteria

- [ ] Button with dropdown showing available formats
- [ ] Download triggers browser download with correct filename
- [ ] Filename follows convention: `{original-filename}_{timestamp}.{ext}`
- [ ] Supports Markdown (.md), HTML (.html), JSON (.json), Raw (.txt)
- [ ] Disabled formats (partial results) not shown in dropdown
- [ ] Loading state during file preparation
- [ ] Download works across all browsers (Chrome, Firefox, Safari, Edge)
- [ ] Accessibility: dropdown keyboard navigable

### Dependencies

- OLA-1.6-001: ResultsViewer complete
- Sprint 1.1: shadcn/ui Select component installed

### Deliverables

**File**: `src/components/results/DownloadButton.tsx`

```typescript
'use client'

import { useState } from 'react'
import { Button } from '@/components/ui/button'
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuTrigger,
} from '@/components/ui/dropdown-menu'
import { Download, FileText, FileCode, FileJson, File } from 'lucide-react'
import type { JobResult } from './ResultsViewer'

interface DownloadButtonProps {
  result: JobResult
  activeFormat?: string
}

type DownloadFormat = {
  id: string
  label: string
  icon: React.ReactNode
  extension: string
  content?: string | object
}

export function DownloadButton({ result, activeFormat }: DownloadButtonProps) {
  const [isDownloading, setIsDownloading] = useState(false)

  /**
   * Generate standardized filename: {original}_{YYYYMMDD_HHmmss}_{format}.{ext}
   * Example: report_20240115_103045_markdown.md
   */
  const getFilename = (formatId: string, extension: string) => {
    const originalName = result.filename.replace(/\.[^.]+$/, '') // Remove extension
    const date = new Date(result.createdAt)
    const timestamp = [
      date.getFullYear(),
      String(date.getMonth() + 1).padStart(2, '0'),
      String(date.getDate()).padStart(2, '0'),
      '_',
      String(date.getHours()).padStart(2, '0'),
      String(date.getMinutes()).padStart(2, '0'),
      String(date.getSeconds()).padStart(2, '0'),
    ].join('')
    return `${originalName}_${timestamp}_${formatId}.${extension}`
  }

  const formats: DownloadFormat[] = [
    {
      id: 'markdown',
      label: 'Markdown',
      icon: <FileText className="h-4 w-4 mr-2" />,
      extension: 'md',
      content: result.markdown,
    },
    {
      id: 'html',
      label: 'HTML',
      icon: <FileCode className="h-4 w-4 mr-2" />,
      extension: 'html',
      content: result.html,
    },
    {
      id: 'json',
      label: 'JSON',
      icon: <FileJson className="h-4 w-4 mr-2" />,
      extension: 'json',
      content: result.json,
    },
    {
      id: 'raw',
      label: 'Raw Text',
      icon: <File className="h-4 w-4 mr-2" />,
      extension: 'txt',
      content: result.rawDocument,
    },
  ].filter((format) => format.content !== undefined && format.content !== null)

  const handleDownload = async (format: DownloadFormat) => {
    setIsDownloading(true)
    try {
      const content =
        typeof format.content === 'string'
          ? format.content
          : JSON.stringify(format.content, null, 2)

      const blob = new Blob([content], { type: 'text/plain;charset=utf-8' })
      const url = URL.createObjectURL(blob)
      const link = document.createElement('a')
      link.href = url
      link.download = getFilename(format.id, format.extension)
      document.body.appendChild(link)
      link.click()
      document.body.removeChild(link)
      URL.revokeObjectURL(url)
    } catch (error) {
      console.error('Download failed:', error)
    } finally {
      setIsDownloading(false)
    }
  }

  const currentFormat = formats.find((f) => f.id === activeFormat)

  return (
    <DropdownMenu>
      <DropdownMenuTrigger asChild>
        <Button variant="outline" disabled={isDownloading || formats.length === 0}>
          <Download className="h-4 w-4 mr-2" />
          {isDownloading ? 'Downloading...' : 'Download'}
        </Button>
      </DropdownMenuTrigger>
      <DropdownMenuContent align="end">
        {formats.map((format) => (
          <DropdownMenuItem
            key={format.id}
            onClick={() => handleDownload(format)}
            className="cursor-pointer"
          >
            {format.icon}
            {format.label} (.{format.extension})
          </DropdownMenuItem>
        ))}
      </DropdownMenuContent>
    </DropdownMenu>
  )
}
```

### Technical Notes

- **Reference**: Specification FR-602, Implementation Plan Sprint 1.6 Tasks 6-7
- **Filename Convention**: `{original}_{YYYYMMDD_HHmmss}_{format}.{ext}`
  - Example: `report_20240115_103045_markdown.md`
  - Format suffix disambiguates multiple downloads of the same document
- **Blob API**: Creates downloadable file in browser
- **URL Management**: `createObjectURL` and `revokeObjectURL` for memory cleanup
- **Format Detection**: Only shows formats with available content
- **Browser Compatibility**: Blob download works in all modern browsers
- **Accessibility**: Dropdown keyboard navigable with Arrow keys, Enter to select

### Testing

```bash
# Unit test
npm run test src/components/results/DownloadButton.test.tsx

# Manual testing:
# [ ] Click dropdown - formats list appears
# [ ] Select Markdown - file downloads with .md extension
# [ ] Select HTML - file downloads with .html extension
# [ ] Select JSON - file downloads pretty-printed
# [ ] Filename includes original name and timestamp
# [ ] Partial results only show available formats
# [ ] Loading state during download
# [ ] Downloaded files open correctly in editors
# [ ] Keyboard navigation works (Tab, Arrow, Enter)
# [ ] Works in Chrome, Firefox, Safari, Edge
```

---

## OLA-1.6-004: Add Partial Result Indicators

### Description

Create visual indicators for partial results, showing which export formats succeeded and which failed.

### Acceptance Criteria

- [ ] Badge component shows partial result status
- [ ] Lists failed export formats
- [ ] Visual distinction between complete and partial results
- [ ] Color-coded: green for success, red for failed, orange for partial
- [ ] Tooltip explains what partial result means
- [ ] Accessible: status communicated to screen readers

### Dependencies

- OLA-1.6-001: ResultsViewer complete
- Sprint 1.1: shadcn/ui Badge component

### Deliverables

**File**: `src/components/results/PartialResultBadge.tsx`

```typescript
'use client'

import { Badge } from '@/components/ui/badge'
import { AlertTriangle, CheckCircle, XCircle } from 'lucide-react'
import {
  Tooltip,
  TooltipContent,
  TooltipProvider,
  TooltipTrigger,
} from '@/components/ui/tooltip'

interface PartialResultBadgeProps {
  failedExports?: string[]
}

export function PartialResultBadge({ failedExports = [] }: PartialResultBadgeProps) {
  if (failedExports.length === 0) {
    return (
      <Badge
        variant="default"
        className="bg-green-600"
        role="status"
        aria-label="Export status: Complete"
      >
        <CheckCircle className="h-3 w-3 mr-1" aria-hidden="true" />
        Complete
      </Badge>
    )
  }

  const failedCount = failedExports.length
  const statusMessage = `Partial result: ${failedCount} export${failedCount > 1 ? 's' : ''} failed`

  return (
    <TooltipProvider>
      <Tooltip>
        <TooltipTrigger asChild>
          {/* tabIndex enables keyboard focus for tooltip accessibility */}
          <Badge
            variant="destructive"
            className="bg-orange-600 hover:bg-orange-700 cursor-pointer"
            role="status"
            aria-label={statusMessage}
            tabIndex={0}
          >
            <AlertTriangle className="h-3 w-3 mr-1" aria-hidden="true" />
            Partial Result
          </Badge>
        </TooltipTrigger>
        <TooltipContent role="tooltip" aria-live="polite">
          <div className="space-y-2">
            <p className="font-semibold">Some exports failed:</p>
            <ul className="list-disc list-inside space-y-1" aria-label="Failed exports">
              {failedExports.map((format) => (
                <li key={format} className="flex items-center gap-2">
                  <XCircle className="h-3 w-3 text-red-500" aria-hidden="true" />
                  {format}
                </li>
              ))}
            </ul>
          </div>
        </TooltipContent>
      </Tooltip>
    </TooltipProvider>
  )
}
```

### Technical Notes

- **Reference**: Specification FR-405, Implementation Plan Sprint 1.6 Task 9
- **Status Types**:
  - Complete: All exports successful (green badge)
  - Partial: Some exports failed (orange badge)
- **Tooltip**: Explains which formats failed
- **Color Coding**: Semantic colors per design system
- **Accessibility**: Badge has `role="status"`, tooltip accessible with focus/hover

### Testing

```bash
# Unit test
npm run test src/components/results/PartialResultBadge.test.tsx

# Manual testing:
# [ ] No failed exports - green "Complete" badge
# [ ] Failed exports - orange "Partial Result" badge
# [ ] Hover tooltip - shows failed export list
# [ ] Tooltip accessible with keyboard (Tab, focus)
# [ ] Screen reader announces status
# [ ] Dark mode colors appropriate
```

---

## OLA-1.6-005: Validate Responsive Design and Accessibility

### Description

Comprehensive validation of responsive design across breakpoints and accessibility compliance for all results viewer components.

### Acceptance Criteria

- [ ] Mobile (320px-767px): Vertical layout, full-width tabs, readable content
- [ ] Tablet (768px-1023px): Optimized layout, touch-friendly targets
- [ ] Desktop (1024px+): Horizontal layout, optimal content width
- [ ] All interactive elements >= 44x44px on mobile
- [ ] Tab navigation functional with keyboard
- [ ] ARIA roles and labels present
- [ ] Color contrast >= 4.5:1 for all text
- [ ] Screen reader announces tab changes and content updates
- [ ] Focus indicators visible
- [ ] Download dropdown accessible with keyboard

### Dependencies

- OLA-1.6-001: ResultsViewer complete
- OLA-1.6-002: Format-specific views complete
- OLA-1.6-003: Download button complete
- OLA-1.6-004: Partial result indicators complete

### Deliverables

**Responsive Breakpoint Tests**:

| Breakpoint | Layout | Font Size | Tab Size | Touch Target |
|------------|--------|-----------|----------|--------------|
| Mobile (< 768px) | Vertical, full-width | Base | Full-width | >= 44px |
| Tablet (768-1023px) | Optimized | Base | Auto | >= 44px |
| Desktop (>= 1024px) | Horizontal | Base | Auto | >= 24px (mouse) |

**Accessibility Checklist**:

```
Results Viewer:
[ ] TabsList has role="tablist"
[ ] TabsTrigger has role="tab", aria-selected, aria-controls
[ ] TabsContent has role="tabpanel", aria-labelledby
[ ] Keyboard navigation: Arrow keys between tabs
[ ] Focus visible on active tab

Markdown View:
[ ] Code blocks have language label (aria-label)
[ ] Prose content readable by screen reader

HTML View:
[ ] Iframe has title and aria-label
[ ] Sandbox prevents script execution

JSON View:
[ ] Expand/collapse buttons have aria-expanded
[ ] Tree structure navigable with keyboard

Raw View:
[ ] Copy button has aria-label
[ ] Button state (Copied) announced to screen reader

Download Button:
[ ] Dropdown menu has aria-haspopup
[ ] Menu items keyboard navigable
[ ] Selected item announced

Partial Result Badge:
[ ] Badge has role="status"
[ ] Tooltip accessible with keyboard
[ ] Failed exports list readable by screen reader
```

### Technical Notes

- **Reference**: Specification Section 9, Implementation Plan Sprint 1.6 Tasks 10-11
- **Testing Tools**:
  - Lighthouse accessibility audit (target score >= 90)
  - axe DevTools
  - NVDA or JAWS screen reader
  - Keyboard-only navigation
  - Chrome DevTools device emulation
- **Responsive Strategy**: Mobile-first design with progressive enhancement
- **Touch Targets**: Minimum 44x44px per WCAG 2.5.5
- **Focus Management**: Visible focus indicators using Tailwind `focus-visible:ring`

### Testing

```bash
# Automated accessibility testing
npm run test:a11y

# Lighthouse audit
npm run lighthouse

# Responsive testing
npm run test:responsive

# Manual testing checklist:
# Responsive:
# [ ] Test at 320px width - all content readable
# [ ] Test at 768px width - tabs optimized
# [ ] Test at 1024px+ width - desktop layout
# [ ] Touch targets measured >= 44x44px on mobile
# [ ] Text does not overflow containers
# [ ] Images/content scale appropriately

# Accessibility:
# [ ] Tab through all elements without mouse
# [ ] Arrow keys navigate between tabs
# [ ] Enter/Space activate buttons
# [ ] Escape closes dropdown
# [ ] Screen reader announces all content
# [ ] Screen reader announces tab changes
# [ ] Focus indicators visible
# [ ] Color contrast verified (WebAIM Contrast Checker)
# [ ] Lighthouse accessibility score >= 90
```

---

## Sprint Completion Checklist

### Acceptance Criteria (Sprint 1.6 Frontend)

- [ ] ResultsViewer with four tabs (Markdown, HTML, JSON, Raw)
- [ ] Default tab is Markdown
- [ ] Tab switching preserves content
- [ ] Markdown renders with syntax highlighting
- [ ] HTML renders in sandboxed iframe
- [ ] JSON displays as collapsible tree
- [ ] Raw view has copy button
- [ ] Download button generates correct filenames
- [ ] Partial result badge shows failed exports
- [ ] All components responsive (mobile, tablet, desktop)
- [ ] Accessibility checklist complete
- [ ] Unit tests pass with >= 80% coverage
- [ ] No TypeScript errors
- [ ] No console warnings

### Integration Points

**With Neo (Backend)**:
- Fetches job results from `/api/v1/jobs/{id}` endpoint
- Displays all available export formats
- Handles partial result status

**With William (Infrastructure)**:
- Results served from database
- Download generates files client-side (no server load)

**With Julia (Testing)**:
- Unit tests for all viewer components
- E2E tests for complete flow
- Accessibility tests passing
- Visual regression tests

### Files Delivered

```
src/components/results/
├── ResultsViewer.tsx
├── MarkdownView.tsx
├── HtmlView.tsx
├── JsonView.tsx
├── RawView.tsx
├── DownloadButton.tsx
├── PartialResultBadge.tsx
├── ResultsSkeleton.tsx
├── ResultsViewer.test.tsx
├── MarkdownView.test.tsx
├── HtmlView.test.tsx
├── JsonView.test.tsx
├── RawView.test.tsx
├── DownloadButton.test.tsx
└── PartialResultBadge.test.tsx
```

### Next Sprint Preview

**Sprint 1.7** will add:
- History page with job listing
- Pagination component
- Job detail modal
- Re-download functionality from history

---

## Notes

- **Security**: HTML iframe sandbox prevents XSS attacks
- **Performance**: Lazy loading for syntax highlighting libraries
- **UX**: Tab state persists in URL for shareable links
- **Accessibility**: Full keyboard navigation and screen reader support
- **Responsiveness**: Mobile-first design ensures usability on all devices
