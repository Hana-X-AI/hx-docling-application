# Neo Next.js Tasks: Sprint 1.6 - Results Viewer & Display

**Sprint**: 1.6 - Results Viewer & Display
**Duration**: ~2.25 hours
**Role**: Lead Developer
**Agent**: Neo (Next.js Senior Developer)
**Support**: Ola (@ola)
**Review**: Julia (@julia)

---

## Development Tools

### shadcn/ui MCP Server Reference

For component discovery and retrieval during results UI development, use the **hx-shadcn MCP Server**:

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
- `get_component_demo` - Get usage examples for Tabs, Card, Button, Dropdown, Skeleton, Badge, Tooltip components
- `get_component_metadata` - Get props, installation instructions, and documentation

**Example - Get Tabs Component Demo:**
```json
{
  "method": "tools/call",
  "params": {
    "name": "get_component_demo",
    "arguments": {"componentName": "tabs"}
  }
}
```

**Sprint 1.6 Component Usage:** This sprint heavily uses Tabs, Card, Button, DropdownMenu, Skeleton, Badge, and Tooltip components for the results viewer interface. Use `get_component_demo` to retrieve usage patterns for tabbed interfaces and tooltips.

---

## Overview

Create the tabbed results viewer with format-specific renderers (Markdown, HTML, JSON, Raw), download functionality, partial result handling, and responsive design. This sprint delivers the output display layer for processed documents.

---

## Tasks

### NEO-1.6-001: Create ResultsViewer with Tabbed Interface

**Priority**: P0 (Critical)
**Effort**: 35 minutes
**Dependencies**: Sprint 1.5b Complete
**Reference**: FR-601

**Description**:
Create the main ResultsViewer component with a tabbed interface for displaying conversion results in four formats: Markdown, HTML, JSON, and Raw.

**Acceptance Criteria**:
- [ ] Four tabs: Markdown (default), HTML, JSON, Raw
- [ ] Tab order fixed (not reorderable)
- [ ] Tab switching preserves content
- [ ] Last selected tab remembered in session
- [ ] Loading state with skeleton while fetching
- [ ] Empty state when no results
- [ ] 'use client' directive (Client Component for tabs)
- [ ] Keyboard navigation for tabs (arrow keys)

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/components/results/ResultsViewer.tsx`

**Technical Notes**:
```typescript
'use client';

import { useState, useEffect } from 'react';
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { MarkdownView } from './MarkdownView';
import { HtmlView } from './HtmlView';
import { JsonView } from './JsonView';
import { RawView } from './RawView';
import { DownloadButton } from './DownloadButton';
import { PartialResultBadge } from './PartialResultBadge';
import { ResultsSkeleton } from './ResultsSkeleton';
import type { ProcessingResult } from '@/types/mcp';

interface ResultsViewerProps {
  jobId: string;
  results: ProcessingResult | null;
  isLoading?: boolean;
  isPartial?: boolean;
  failedFormats?: string[];
}

const TAB_STORAGE_KEY = 'hx-docling-last-tab';

export function ResultsViewer({
  jobId,
  results,
  isLoading = false,
  isPartial = false,
  failedFormats = [],
}: ResultsViewerProps) {
  const [activeTab, setActiveTab] = useState<string>(() => {
    if (typeof window !== 'undefined') {
      return sessionStorage.getItem(TAB_STORAGE_KEY) || 'markdown';
    }
    return 'markdown';
  });

  useEffect(() => {
    sessionStorage.setItem(TAB_STORAGE_KEY, activeTab);
  }, [activeTab]);

  if (isLoading) {
    return <ResultsSkeleton />;
  }

  if (!results) {
    return (
      <Card>
        <CardContent className="p-8 text-center text-muted-foreground">
          <p>No results available. Process a document to see results.</p>
        </CardContent>
      </Card>
    );
  }

  const hasMarkdown = !!results.markdown;
  const hasHtml = !!results.html;
  const hasJson = !!results.json;
  const hasRaw = !!results.raw;

  return (
    <Card>
      <CardHeader className="flex flex-row items-center justify-between">
        <div className="flex items-center gap-2">
          <CardTitle>Results</CardTitle>
          {isPartial && <PartialResultBadge failedFormats={failedFormats} />}
        </div>
        <DownloadButton
          jobId={jobId}
          format={activeTab}
          disabled={!results}
        />
      </CardHeader>
      <CardContent>
        <Tabs value={activeTab} onValueChange={setActiveTab}>
          <TabsList className="grid w-full grid-cols-4">
            <TabsTrigger
              value="markdown"
              disabled={!hasMarkdown || failedFormats.includes('markdown')}
            >
              Markdown
            </TabsTrigger>
            <TabsTrigger
              value="html"
              disabled={!hasHtml || failedFormats.includes('html')}
            >
              HTML
            </TabsTrigger>
            <TabsTrigger
              value="json"
              disabled={!hasJson || failedFormats.includes('json')}
            >
              JSON
            </TabsTrigger>
            <TabsTrigger
              value="raw"
              disabled={!hasRaw || failedFormats.includes('raw')}
            >
              Raw
            </TabsTrigger>
          </TabsList>

          <TabsContent value="markdown" className="mt-4">
            {hasMarkdown ? (
              <MarkdownView content={results.markdown!} />
            ) : (
              <FormatUnavailable format="Markdown" />
            )}
          </TabsContent>

          <TabsContent value="html" className="mt-4">
            {hasHtml ? (
              <HtmlView content={results.html!} />
            ) : (
              <FormatUnavailable format="HTML" />
            )}
          </TabsContent>

          <TabsContent value="json" className="mt-4">
            {hasJson ? (
              <JsonView content={results.json!} />
            ) : (
              <FormatUnavailable format="JSON" />
            )}
          </TabsContent>

          <TabsContent value="raw" className="mt-4">
            {hasRaw ? (
              <RawView content={results.raw!} />
            ) : (
              <FormatUnavailable format="Raw" />
            )}
          </TabsContent>
        </Tabs>
      </CardContent>
    </Card>
  );
}

function FormatUnavailable({ format }: { format: string }) {
  return (
    <div className="p-8 text-center text-muted-foreground border rounded-lg border-dashed">
      <p>{format} export failed or is unavailable.</p>
    </div>
  );
}
```

---

### NEO-1.6-002: Implement MarkdownView with GFM Support

**Priority**: P0 (Critical)
**Effort**: 25 minutes
**Dependencies**: NEO-1.6-001
**Reference**: FR-602

**Description**:
Implement Markdown rendering with GitHub Flavored Markdown (GFM) support, syntax highlighting for code blocks, and proper table rendering.

**Acceptance Criteria**:
- [ ] GFM syntax supported (tables, strikethrough, task lists)
- [ ] Code blocks with syntax highlighting
- [ ] Tables rendered correctly
- [ ] Images rendered (base64 data URIs supported)
- [ ] Links open in new tab
- [ ] Scrollable container for long content
- [ ] Server Component (no client-side interactivity)

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/components/results/MarkdownView.tsx`

**Technical Notes**:
```typescript
import ReactMarkdown from 'react-markdown';
import remarkGfm from 'remark-gfm';
import { cn } from '@/lib/utils';

interface MarkdownViewProps {
  content: string;
  className?: string;
}

export function MarkdownView({ content, className }: MarkdownViewProps) {
  return (
    <div
      className={cn(
        'prose prose-sm dark:prose-invert max-w-none',
        'overflow-auto max-h-[600px] p-4 rounded-lg bg-muted/50',
        className
      )}
    >
      <ReactMarkdown
        remarkPlugins={[remarkGfm]}
        components={{
          // Open links in new tab
          a: ({ href, children }) => (
            <a
              href={href}
              target="_blank"
              rel="noopener noreferrer"
              className="text-primary hover:underline"
            >
              {children}
            </a>
          ),
          // Code blocks with syntax highlighting
          code: ({ className, children, ...props }) => {
            const isInline = !className;
            if (isInline) {
              return (
                <code
                  className="px-1.5 py-0.5 bg-muted rounded text-sm font-mono"
                  {...props}
                >
                  {children}
                </code>
              );
            }
            return (
              <code
                className={cn(
                  'block p-4 bg-muted rounded-lg overflow-x-auto text-sm',
                  className
                )}
                {...props}
              >
                {children}
              </code>
            );
          },
          // Tables
          table: ({ children }) => (
            <div className="overflow-x-auto">
              <table className="min-w-full border-collapse border border-border">
                {children}
              </table>
            </div>
          ),
          th: ({ children }) => (
            <th className="border border-border px-4 py-2 bg-muted font-medium text-left">
              {children}
            </th>
          ),
          td: ({ children }) => (
            <td className="border border-border px-4 py-2">{children}</td>
          ),
          // Images
          img: ({ src, alt }) => (
            // eslint-disable-next-line @next/next/no-img-element
            <img
              src={src}
              alt={alt || ''}
              className="max-w-full h-auto rounded-lg"
              loading="lazy"
            />
          ),
        }}
      >
        {content}
      </ReactMarkdown>
    </div>
  );
}
```

---

### NEO-1.6-003: Implement HtmlView with Sandboxed Iframe

**Priority**: P0 (Critical)
**Effort**: 25 minutes
**Dependencies**: NEO-1.6-001
**Reference**: FR-603

**Description**:
Display HTML output in a sandboxed iframe for security, preventing script execution and external resource loading.

**Acceptance Criteria**:
- [ ] HTML rendered in iframe element
- [ ] sandbox attribute applied (prevent scripts)
- [ ] Scrollable container
- [ ] srcDoc used (no external URL)
- [ ] Responsive height adjustment
- [ ] Server Component

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/components/results/HtmlView.tsx`

**Technical Notes**:
```typescript
import { cn } from '@/lib/utils';

interface HtmlViewProps {
  content: string;
  className?: string;
}

/**
 * HtmlView renders untrusted HTML content from document conversion in a sandboxed iframe.
 *
 * SECURITY THREAT MODEL:
 * - Content is generated by Docling from uploaded documents (PDFs, DOCX, etc.)
 * - Content may contain malicious scripts, event handlers, or external resource references
 * - We treat all converted HTML as UNTRUSTED
 *
 * SANDBOX POLICY:
 * - Empty sandbox attribute provides maximum isolation:
 *   - No JavaScript execution (no allow-scripts)
 *   - No form submission (no allow-forms)
 *   - No same-origin access (no allow-same-origin) - prevents access to parent page cookies/storage
 *   - No popups (no allow-popups)
 *   - No top navigation (no allow-top-navigation)
 * - This is intentionally restrictive; the HTML preview is read-only display only
 */
export function HtmlView({ content, className }: HtmlViewProps) {
  // Wrap content with basic styles for consistent rendering
  // All styles are inline since external stylesheets would require allow-same-origin
  const wrappedHtml = `
    <!DOCTYPE html>
    <html>
      <head>
        <meta charset="utf-8">
        <meta name="viewport" content="width=device-width, initial-scale=1">
        <style>
          body {
            font-family: system-ui, -apple-system, sans-serif;
            font-size: 14px;
            line-height: 1.6;
            color: #333;
            margin: 0;
            padding: 16px;
          }
          img { max-width: 100%; height: auto; }
          table { border-collapse: collapse; width: 100%; }
          th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
          th { background: #f5f5f5; }
          pre { overflow-x: auto; background: #f5f5f5; padding: 12px; border-radius: 4px; }
          code { font-family: monospace; background: #f5f5f5; padding: 2px 4px; border-radius: 2px; }
        </style>
      </head>
      <body>
        ${content}
      </body>
    </html>
  `;

  return (
    <div
      className={cn(
        'relative rounded-lg border overflow-hidden',
        className
      )}
    >
      {/*
        SECURITY: Empty sandbox provides maximum isolation for untrusted content.
        Do NOT add allow-same-origin as it would allow the iframe to:
        - Access parent page cookies and localStorage
        - Execute scripts that could exfiltrate data
        - Bypass the same-origin policy protections
      */}
      <iframe
        srcDoc={wrappedHtml}
        sandbox=""
        title="HTML Preview"
        className="w-full min-h-[400px] max-h-[600px] bg-white"
        style={{ border: 'none' }}
        aria-label="HTML document preview"
      />
    </div>
  );
}
```

**Test Requirements**:
```typescript
// tests/components/HtmlView.test.tsx
import { render, screen } from '@testing-library/react';
import { HtmlView } from '@/components/results/HtmlView';

describe('HtmlView Security', () => {
  it('should render iframe with empty sandbox attribute for maximum isolation', () => {
    render(<HtmlView content="<p>Test</p>" />);
    const iframe = screen.getByTitle('HTML Preview');

    // Empty sandbox attribute provides maximum isolation
    expect(iframe).toHaveAttribute('sandbox', '');
  });

  it('should NOT include allow-same-origin in sandbox', () => {
    render(<HtmlView content="<p>Test</p>" />);
    const iframe = screen.getByTitle('HTML Preview');

    const sandbox = iframe.getAttribute('sandbox') || '';
    expect(sandbox).not.toContain('allow-same-origin');
  });

  it('should NOT include allow-scripts in sandbox', () => {
    render(<HtmlView content="<p>Test</p>" />);
    const iframe = screen.getByTitle('HTML Preview');

    const sandbox = iframe.getAttribute('sandbox') || '';
    expect(sandbox).not.toContain('allow-scripts');
  });

  it('should render srcDoc with wrapped HTML content', () => {
    render(<HtmlView content="<p>Hello World</p>" />);
    const iframe = screen.getByTitle('HTML Preview');

    expect(iframe).toHaveAttribute('srcDoc');
    expect(iframe.getAttribute('srcDoc')).toContain('<p>Hello World</p>');
  });
});
```

---

### NEO-1.6-004: Implement JsonView with Syntax Highlighting

**Priority**: P0 (Critical)
**Effort**: 25 minutes
**Dependencies**: NEO-1.6-001
**Reference**: FR-604

**Description**:
Display JSON output with syntax highlighting and optional collapsible nodes for large documents.

**Acceptance Criteria**:
- [ ] Pretty-printed (2-space indent)
- [ ] Syntax highlighting (keys, strings, numbers, booleans)
- [ ] Collapsible nodes for objects/arrays (optional enhancement)
- [ ] Scrollable container
- [ ] Copy button for raw JSON
- [ ] Server Component

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/components/results/JsonView.tsx`

**Technical Notes**:
```typescript
'use client';

import { useState, useMemo, ReactNode } from 'react';
import { Copy, Check, ChevronDown, ChevronRight } from 'lucide-react';
import { Button } from '@/components/ui/button';
import { cn } from '@/lib/utils';

interface JsonViewProps {
  content: string;
  className?: string;
}

// Token types for JSON syntax highlighting
type TokenType = 'key' | 'string' | 'number' | 'boolean' | 'null' | 'punctuation';

interface Token {
  type: TokenType;
  value: string;
}

// Token-to-class mapping for syntax highlighting (XSS-safe)
const TOKEN_CLASSES: Record<TokenType, string> = {
  key: 'text-blue-600 dark:text-blue-400',
  string: 'text-green-600 dark:text-green-400',
  number: 'text-orange-600 dark:text-orange-400',
  boolean: 'text-purple-600 dark:text-purple-400',
  null: 'text-gray-500',
  punctuation: '',
};

/**
 * Tokenize JSON string for safe syntax highlighting.
 * Renders React elements instead of using dangerouslySetInnerHTML.
 */
function tokenizeJson(json: string): Token[] {
  const tokens: Token[] = [];
  const regex = /("(?:[^"\\]|\\.)*")\s*:|("(?:[^"\\]|\\.)*")|(-?\d+\.?\d*(?:[eE][+-]?\d+)?)|(\btrue\b|\bfalse\b)|(\bnull\b)|([{}\[\]:,])|(\s+)/g;

  let match: RegExpExecArray | null;
  while ((match = regex.exec(json)) !== null) {
    if (match[1]) {
      // Key (string followed by colon)
      tokens.push({ type: 'key', value: match[1] });
      tokens.push({ type: 'punctuation', value: ':' });
    } else if (match[2]) {
      // String value
      tokens.push({ type: 'string', value: match[2] });
    } else if (match[3]) {
      // Number
      tokens.push({ type: 'number', value: match[3] });
    } else if (match[4]) {
      // Boolean
      tokens.push({ type: 'boolean', value: match[4] });
    } else if (match[5]) {
      // Null
      tokens.push({ type: 'null', value: match[5] });
    } else if (match[6]) {
      // Punctuation
      tokens.push({ type: 'punctuation', value: match[6] });
    } else if (match[7]) {
      // Whitespace (preserve formatting)
      tokens.push({ type: 'punctuation', value: match[7] });
    }
  }

  return tokens;
}

/**
 * Render tokens as React elements (XSS-safe - no innerHTML)
 */
function renderTokens(tokens: Token[]): ReactNode[] {
  return tokens.map((token, index) => {
    const className = TOKEN_CLASSES[token.type];
    if (className) {
      return (
        <span key={index} className={className}>
          {token.value}
        </span>
      );
    }
    // Punctuation and whitespace rendered as plain text
    return token.value;
  });
}

export function JsonView({ content, className }: JsonViewProps) {
  const [copied, setCopied] = useState(false);

  const formattedJson = useMemo(() => {
    try {
      const parsed = JSON.parse(content);
      return JSON.stringify(parsed, null, 2);
    } catch {
      return content;
    }
  }, [content]);

  const handleCopy = async () => {
    await navigator.clipboard.writeText(formattedJson);
    setCopied(true);
    setTimeout(() => setCopied(false), 2000);
  };

  // Safe syntax highlighting using React elements (no dangerouslySetInnerHTML)
  const highlightedContent = useMemo(() => {
    const tokens = tokenizeJson(formattedJson);
    return renderTokens(tokens);
  }, [formattedJson]);

  return (
    <div className={cn('relative rounded-lg', className)}>
      <div className="absolute top-2 right-2 z-10">
        <Button
          variant="ghost"
          size="sm"
          onClick={handleCopy}
          aria-label={copied ? 'Copied!' : 'Copy JSON'}
        >
          {copied ? (
            <Check className="w-4 h-4 text-green-500" />
          ) : (
            <Copy className="w-4 h-4" />
          )}
        </Button>
      </div>
      <pre
        className={cn(
          'p-4 bg-muted rounded-lg overflow-auto max-h-[600px]',
          'text-sm font-mono leading-relaxed'
        )}
      >
        <code className="whitespace-pre">
          {highlightedContent}
        </code>
      </pre>
    </div>
  );
}
```

---

### NEO-1.6-005: Implement RawView with Copy Button

**Priority**: P1 (High)
**Effort**: 15 minutes
**Dependencies**: NEO-1.6-001
**Reference**: FR-605

**Description**:
Display the raw DoclingDocument JSON for debugging purposes with a copy-to-clipboard button.

**Acceptance Criteria**:
- [ ] Full DoclingDocument JSON displayed
- [ ] Pretty-printed
- [ ] Copy-to-clipboard button
- [ ] Scrollable container
- [ ] Monospace font

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/components/results/RawView.tsx`

**Technical Notes**:
```typescript
'use client';

import { useState, useMemo } from 'react';
import { Copy, Check } from 'lucide-react';
import { Button } from '@/components/ui/button';
import { cn } from '@/lib/utils';
import type { DoclingDocument } from '@/types/mcp';

interface RawViewProps {
  content: DoclingDocument;
  className?: string;
}

export function RawView({ content, className }: RawViewProps) {
  const [copied, setCopied] = useState(false);

  const formattedContent = useMemo(() => {
    return JSON.stringify(content, null, 2);
  }, [content]);

  const handleCopy = async () => {
    await navigator.clipboard.writeText(formattedContent);
    setCopied(true);
    setTimeout(() => setCopied(false), 2000);
  };

  return (
    <div className={cn('relative rounded-lg', className)}>
      <div className="absolute top-2 right-2 z-10">
        <Button
          variant="ghost"
          size="sm"
          onClick={handleCopy}
          aria-label={copied ? 'Copied!' : 'Copy raw document'}
        >
          {copied ? (
            <Check className="w-4 h-4 text-green-500" />
          ) : (
            <Copy className="w-4 h-4" />
          )}
          <span className="ml-2">{copied ? 'Copied!' : 'Copy'}</span>
        </Button>
      </div>
      <pre
        className={cn(
          'p-4 bg-muted rounded-lg overflow-auto max-h-[600px]',
          'text-sm font-mono leading-relaxed text-muted-foreground'
        )}
      >
        {formattedContent}
      </pre>
    </div>
  );
}
```

---

### NEO-1.6-006: Create DownloadButton Component

**Priority**: P0 (Critical)
**Effort**: 20 minutes
**Dependencies**: NEO-1.6-001
**Reference**: FR-606

**Description**:
Create a download button component that allows users to download results in the selected format with proper filename convention.

**Acceptance Criteria**:
- [ ] Download button per format
- [ ] File naming: {filename}_{timestamp}_{format}.{ext}
- [ ] Browser download dialog triggered
- [ ] Loading state during download
- [ ] Error handling for failed downloads

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/components/results/DownloadButton.tsx`

**Technical Notes**:
```typescript
'use client';

import { useState } from 'react';
import { Download, Loader2 } from 'lucide-react';
import { Button } from '@/components/ui/button';
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuTrigger,
} from '@/components/ui/dropdown-menu';

interface DownloadButtonProps {
  jobId: string;
  format?: string;
  disabled?: boolean;
}

const FORMAT_EXTENSIONS: Record<string, string> = {
  markdown: 'md',
  html: 'html',
  json: 'json',
  raw: 'json',
};

function generateFilename(originalName: string, format: string): string {
  const baseName = originalName.replace(/\.[^/.]+$/, '') || 'document';
  const timestamp = new Date().toISOString().replace(/[:.]/g, '-').slice(0, 19);
  const ext = FORMAT_EXTENSIONS[format] || format;
  return `${baseName}_${timestamp}_${format}.${ext}`;
}

export function DownloadButton({
  jobId,
  format = 'markdown',
  disabled = false,
}: DownloadButtonProps) {
  const [isDownloading, setIsDownloading] = useState(false);

  const handleDownload = async (downloadFormat: string) => {
    setIsDownloading(true);
    try {
      const response = await fetch(
        `/api/v1/jobs/${jobId}/download?format=${downloadFormat}`
      );

      if (!response.ok) {
        throw new Error('Download failed');
      }

      const data = await response.json();
      const content = data.data.content;
      const filename = generateFilename(data.data.originalName || 'document', downloadFormat);

      // Create blob and download
      const blob = new Blob([content], {
        type: downloadFormat === 'json' || downloadFormat === 'raw'
          ? 'application/json'
          : 'text/plain',
      });
      const url = URL.createObjectURL(blob);
      const link = document.createElement('a');
      link.href = url;
      link.download = filename;
      document.body.appendChild(link);
      link.click();
      document.body.removeChild(link);
      URL.revokeObjectURL(url);
    } catch (error) {
      console.error('Download failed:', error);
      // Could show toast notification here
    } finally {
      setIsDownloading(false);
    }
  };

  return (
    <DropdownMenu>
      <DropdownMenuTrigger asChild>
        <Button variant="outline" size="sm" disabled={disabled || isDownloading}>
          {isDownloading ? (
            <Loader2 className="w-4 h-4 mr-2 animate-spin" />
          ) : (
            <Download className="w-4 h-4 mr-2" />
          )}
          Download
        </Button>
      </DropdownMenuTrigger>
      <DropdownMenuContent align="end">
        <DropdownMenuItem onClick={() => handleDownload('markdown')}>
          Markdown (.md)
        </DropdownMenuItem>
        <DropdownMenuItem onClick={() => handleDownload('html')}>
          HTML (.html)
        </DropdownMenuItem>
        <DropdownMenuItem onClick={() => handleDownload('json')}>
          JSON (.json)
        </DropdownMenuItem>
        <DropdownMenuItem onClick={() => handleDownload('raw')}>
          Raw Document (.json)
        </DropdownMenuItem>
      </DropdownMenuContent>
    </DropdownMenu>
  );
}
```

---

### NEO-1.6-007: Implement Download Naming Convention

**Priority**: P1 (High)
**Effort**: 10 minutes
**Dependencies**: NEO-1.6-006

**Description**:
Implement the download filename generation utility per specification: {filename}_{timestamp}_{format}.{ext}

**Acceptance Criteria**:
- [ ] Filename extracted from original (without extension)
- [ ] Timestamp in YYYYMMDD_HHmmss format
- [ ] Format suffix added
- [ ] Correct extension per format
- [ ] Utility function reusable

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/lib/utils/download.ts`

**Technical Notes**:
```typescript
// src/lib/utils/download.ts

export const FORMAT_EXTENSIONS: Record<string, string> = {
  markdown: 'md',
  html: 'html',
  json: 'json',
  raw: 'json',
};

export function generateDownloadFilename(
  originalName: string,
  format: 'markdown' | 'html' | 'json' | 'raw'
): string {
  // Remove extension from original name
  const baseName = originalName.replace(/\.[^/.]+$/, '') || 'document';

  // Generate timestamp in YYYYMMDD_HHmmss format
  const now = new Date();
  const timestamp = [
    now.getFullYear(),
    String(now.getMonth() + 1).padStart(2, '0'),
    String(now.getDate()).padStart(2, '0'),
    '_',
    String(now.getHours()).padStart(2, '0'),
    String(now.getMinutes()).padStart(2, '0'),
    String(now.getSeconds()).padStart(2, '0'),
  ].join('');

  // Get extension for format
  const ext = FORMAT_EXTENSIONS[format] || format;

  return `${baseName}_${timestamp}_${format}.${ext}`;
}

export function downloadContent(
  content: string,
  filename: string,
  mimeType: string = 'text/plain'
): void {
  const blob = new Blob([content], { type: mimeType });
  const url = URL.createObjectURL(blob);

  const link = document.createElement('a');
  link.href = url;
  link.download = filename;

  document.body.appendChild(link);
  link.click();

  document.body.removeChild(link);
  URL.revokeObjectURL(url);
}
```

---

### NEO-1.6-008: Add Skeleton Loaders

**Priority**: P1 (High)
**Effort**: 15 minutes
**Dependencies**: NEO-1.6-001

**Description**:
Create skeleton loaders for the results viewer that show while content is loading, providing visual feedback during data fetching.

**Acceptance Criteria**:
- [ ] Skeleton matches results viewer layout
- [ ] Tab skeleton shapes
- [ ] Content area skeleton
- [ ] Animated pulse effect
- [ ] Accessible (aria-busy)

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/components/results/ResultsSkeleton.tsx`

**Technical Notes**:
```typescript
import { Card, CardContent, CardHeader } from '@/components/ui/card';
import { Skeleton } from '@/components/ui/skeleton';

export function ResultsSkeleton() {
  return (
    <Card aria-busy="true" aria-label="Loading results">
      <CardHeader className="flex flex-row items-center justify-between">
        <Skeleton className="h-6 w-24" />
        <Skeleton className="h-9 w-28" />
      </CardHeader>
      <CardContent className="space-y-4">
        {/* Tab bar skeleton */}
        <div className="flex gap-2">
          <Skeleton className="h-10 w-24" />
          <Skeleton className="h-10 w-20" />
          <Skeleton className="h-10 w-20" />
          <Skeleton className="h-10 w-16" />
        </div>

        {/* Content area skeleton */}
        <div className="space-y-3 p-4 border rounded-lg">
          <Skeleton className="h-4 w-full" />
          <Skeleton className="h-4 w-5/6" />
          <Skeleton className="h-4 w-4/6" />
          <Skeleton className="h-20 w-full" />
          <Skeleton className="h-4 w-full" />
          <Skeleton className="h-4 w-3/4" />
        </div>
      </CardContent>
    </Card>
  );
}
```

---

### NEO-1.6-009: Implement Partial Result Indicators

**Priority**: P1 (High)
**Effort**: 20 minutes
**Dependencies**: NEO-1.6-001
**Reference**: FR-405

**Description**:
Create visual indicators showing which export formats succeeded and which failed when job completes with PARTIAL_COMPLETE status.

**Acceptance Criteria**:
- [ ] Badge showing partial completion status
- [ ] List of failed formats on hover/click
- [ ] Warning color scheme
- [ ] Accessible tooltip/popover
- [ ] Integration with ResultsViewer

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/components/results/PartialResultBadge.tsx`

**Technical Notes**:
```typescript
'use client';

import { AlertTriangle } from 'lucide-react';
import {
  Tooltip,
  TooltipContent,
  TooltipProvider,
  TooltipTrigger,
} from '@/components/ui/tooltip';
import { Badge } from '@/components/ui/badge';

interface PartialResultBadgeProps {
  failedFormats: string[];
}

const FORMAT_LABELS: Record<string, string> = {
  markdown: 'Markdown',
  html: 'HTML',
  json: 'JSON',
  raw: 'Raw Document',
};

export function PartialResultBadge({ failedFormats }: PartialResultBadgeProps) {
  if (failedFormats.length === 0) return null;

  return (
    <TooltipProvider>
      <Tooltip>
        <TooltipTrigger asChild>
          <Badge
            variant="outline"
            className="border-yellow-500 text-yellow-700 dark:text-yellow-400 gap-1"
          >
            <AlertTriangle className="w-3 h-3" />
            Partial
          </Badge>
        </TooltipTrigger>
        <TooltipContent side="bottom" className="max-w-xs">
          <p className="font-medium mb-1">Some exports failed:</p>
          <ul className="text-sm list-disc list-inside">
            {failedFormats.map((format) => (
              <li key={format}>{FORMAT_LABELS[format] || format}</li>
            ))}
          </ul>
        </TooltipContent>
      </Tooltip>
    </TooltipProvider>
  );
}
```

---

### NEO-1.6-010: Validate Responsive Design

**Priority**: P1 (High)
**Effort**: 15 minutes
**Dependencies**: NEO-1.6-001 through NEO-1.6-009

**Description**:
Validate and adjust responsive design for mobile, tablet, and desktop breakpoints. Ensure results viewer is usable across all device sizes.

**Acceptance Criteria**:
- [ ] Mobile (< 640px): Stacked tabs, full-width content
- [ ] Tablet (640px - 1024px): Horizontal tabs, adjusted padding
- [ ] Desktop (> 1024px): Full layout with optimal spacing
- [ ] Touch-friendly tap targets (44x44px minimum)
- [ ] No horizontal scrolling on mobile (except code blocks)

**Deliverables**:
- Responsive adjustments to all results components

**Technical Notes**:
```typescript
// Responsive classes to apply:
// Tabs: grid-cols-2 sm:grid-cols-4
// Content: p-2 sm:p-4
// Max heights: max-h-[400px] sm:max-h-[600px]
// Font sizes: text-sm sm:text-base
```

---

### NEO-1.6-011: Add Accessibility (Tab Navigation, ARIA)

**Priority**: P1 (High)
**Effort**: 20 minutes
**Dependencies**: All previous tasks

**Description**:
Ensure all results viewer components meet WCAG 2.1 AA accessibility standards with proper ARIA labels, keyboard navigation, and screen reader support.

**Acceptance Criteria**:
- [ ] Keyboard tab navigation works
- [ ] Arrow keys switch between tabs
- [ ] ARIA labels on all interactive elements
- [ ] aria-selected on active tab
- [ ] Screen reader announces tab content changes
- [ ] Color contrast meets 4.5:1 ratio
- [ ] Focus visible indicators present

**Deliverables**:
- Accessibility improvements to all results components

**Technical Notes**:
```typescript
// shadcn/ui Tabs already handle most ARIA attributes
// Ensure custom components have:
// - role="tabpanel" on content areas
// - aria-labelledby linking to tab trigger
// - aria-busy on loading states
// - aria-live="polite" for dynamic content
```

---

### NEO-1.6-012: Write Unit Tests for ResultsViewer

**Priority**: P1 (High)
**Effort**: 15 minutes
**Dependencies**: NEO-1.6-001

**Description**:
Write unit tests for ResultsViewer covering tab switching, content display, and partial result handling.

**Acceptance Criteria**:
- [ ] Test default tab selection (Markdown)
- [ ] Test tab switching preserves content
- [ ] Test loading state displays skeleton
- [ ] Test empty state message
- [ ] Test partial result badge display
- [ ] Tests achieve 80%+ coverage

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/components/results/ResultsViewer.test.tsx`

**Technical Notes**:
```typescript
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, it, expect } from 'vitest';
import { ResultsViewer } from './ResultsViewer';

describe('ResultsViewer', () => {
  const mockResults = {
    markdown: '# Test',
    html: '<h1>Test</h1>',
    json: '{"test": true}',
    raw: { name: 'test', body: {} },
  };

  it('renders with Markdown tab selected by default', () => {
    render(<ResultsViewer jobId="job-1" results={mockResults} />);
    expect(screen.getByRole('tab', { name: /markdown/i })).toHaveAttribute(
      'aria-selected',
      'true'
    );
  });

  it('switches tabs on click', async () => {
    render(<ResultsViewer jobId="job-1" results={mockResults} />);
    const htmlTab = screen.getByRole('tab', { name: /html/i });

    await userEvent.click(htmlTab);

    expect(htmlTab).toHaveAttribute('aria-selected', 'true');
  });

  it('shows loading state', () => {
    render(<ResultsViewer jobId="job-1" results={null} isLoading />);
    expect(screen.getByLabelText(/loading/i)).toBeInTheDocument();
  });

  it('shows empty state when no results', () => {
    render(<ResultsViewer jobId="job-1" results={null} />);
    expect(screen.getByText(/no results available/i)).toBeInTheDocument();
  });

  it('shows partial result badge', () => {
    render(
      <ResultsViewer
        jobId="job-1"
        results={{ ...mockResults, html: undefined }}
        isPartial
        failedFormats={['html']}
      />
    );
    expect(screen.getByText(/partial/i)).toBeInTheDocument();
  });
});
```

---

## Summary

| Task ID | Task Title | Effort | Priority |
|---------|-----------|--------|----------|
| NEO-1.6-001 | Create ResultsViewer with Tabs | 35m | P0 |
| NEO-1.6-002 | Implement MarkdownView | 25m | P0 |
| NEO-1.6-003 | Implement HtmlView | 25m | P0 |
| NEO-1.6-004 | Implement JsonView | 25m | P0 |
| NEO-1.6-005 | Implement RawView | 15m | P1 |
| NEO-1.6-006 | Create DownloadButton | 20m | P0 |
| NEO-1.6-007 | Implement Download Naming | 10m | P1 |
| NEO-1.6-008 | Add Skeleton Loaders | 15m | P1 |
| NEO-1.6-009 | Implement Partial Result Indicators | 20m | P1 |
| NEO-1.6-010 | Validate Responsive Design | 15m | P1 |
| NEO-1.6-011 | Add Accessibility | 20m | P1 |
| NEO-1.6-012 | Write Unit Tests | 15m | P1 |

**Total Effort**: ~2.25 hours (240 minutes)
**Total Tasks**: 12

---

## Dependencies Graph

```
Sprint 1.5b Complete
    |
    +-> NEO-1.6-001 (ResultsViewer)
            |
            +-> NEO-1.6-002 (MarkdownView)
            +-> NEO-1.6-003 (HtmlView)
            +-> NEO-1.6-004 (JsonView)
            +-> NEO-1.6-005 (RawView)
            +-> NEO-1.6-006 (DownloadButton)
            |       |
            |       +-> NEO-1.6-007 (Download Naming)
            +-> NEO-1.6-008 (Skeleton)
            +-> NEO-1.6-009 (Partial Indicators)
            |
            +-> NEO-1.6-010 (Responsive) [After all views]
            +-> NEO-1.6-011 (Accessibility) [After all views]
            +-> NEO-1.6-012 (Tests) [After main implementation]
```

---

## Parallel Execution Opportunities

**Parallel Group A** (After NEO-1.6-001):
- [P] NEO-1.6-002 (MarkdownView)
- [P] NEO-1.6-003 (HtmlView)
- [P] NEO-1.6-004 (JsonView)
- [P] NEO-1.6-005 (RawView)
- [P] NEO-1.6-008 (Skeleton)
- [P] NEO-1.6-009 (Partial Indicators)

**Parallel Group B** (After NEO-1.6-006):
- [P] NEO-1.6-007 (Download Naming)

**Sequential** (After all views complete):
- NEO-1.6-010 (Responsive)
- NEO-1.6-011 (Accessibility)
- NEO-1.6-012 (Tests)
