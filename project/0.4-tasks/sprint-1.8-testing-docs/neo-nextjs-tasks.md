# Neo Next.js Tasks: Sprint 1.8 - Testing & Documentation (Support Role)

**Sprint**: 1.8 - Testing & Documentation
**Duration**: ~6.0 hours (Sprint Total)
**Role**: Support Developer
**Agent**: Neo (Next.js Senior Developer)
**Lead**: Julia (@julia)
**Support**: Neo (@neo), Trinity (@trinity), William (@william)
**Review**: Agent Zero (@agent-zero)

---

## Development Tools

### shadcn/ui MCP Server Reference

For component testing and documentation reference, the **hx-shadcn MCP Server** is available:

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
- `get_component_demo` - Get usage examples and demo code
- `get_component_metadata` - Get props, installation instructions, and documentation

**Example - Get Component Metadata for Testing:**
```json
{
  "method": "tools/call",
  "params": {
    "name": "get_component_metadata",
    "arguments": {"componentName": "button"}
  }
}
```

**Sprint 1.8 Usage:** When writing component tests, use `get_component_metadata` to understand component props and expected behaviors. This helps ensure comprehensive test coverage.

---

## Overview

Sprint 1.8 focuses on comprehensive testing and documentation. Julia leads the testing effort while Neo provides support for component testing, documentation writing, and final quality assurance. This sprint ensures the application meets all quality gates before Phase 1 completion.

---

## Neo's Support Tasks

### NEO-1.8-001: Write Component Tests (UploadZone, UrlInput)

**Priority**: P1 (High)
**Effort**: 30 minutes
**Dependencies**: Sprint 1.7 Complete

**Description**:
Write comprehensive component tests for input components including user interaction testing, validation feedback, and accessibility verification.

**Acceptance Criteria**:
- [ ] UploadZone drag-drop tests
- [ ] UploadZone click-to-browse tests
- [ ] UrlInput validation tests
- [ ] Error state display tests
- [ ] Accessibility tests (keyboard, ARIA)
- [ ] Coverage >= 80% for tested components

**Deliverables**:
- `src/components/upload/UploadZone.test.tsx`
- `src/components/upload/UrlInput.test.tsx`
- `src/components/upload/UploadForm.test.tsx`

**Technical Notes**:

> **Component Requirement**: The UploadZone component renders a Card, not a button element.
> Ensure the Card has `data-testid="upload-zone"` and `role="button"` attributes for proper
> test targeting and accessibility. Tests use `getByTestId('upload-zone')` instead of `getByRole('button')`.

```typescript
// UploadZone.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { describe, it, expect, vi } from 'vitest';
import { UploadZone } from './UploadZone';

describe('UploadZone', () => {
  describe('drag and drop', () => {
    it('accepts valid files on drop', async () => {
      const onFileSelect = vi.fn();
      render(<UploadZone onFileSelect={onFileSelect} />);

      const file = new File(['test'], 'test.pdf', { type: 'application/pdf' });
      const dropzone = screen.getByTestId('upload-zone');

      fireEvent.drop(dropzone, {
        dataTransfer: { files: [file] },
      });

      expect(onFileSelect).toHaveBeenCalledWith(file);
    });

    it('shows drag active state', () => {
      render(<UploadZone onFileSelect={vi.fn()} />);
      const dropzone = screen.getByTestId('upload-zone');

      fireEvent.dragEnter(dropzone);

      expect(screen.getByText(/drop the file/i)).toBeInTheDocument();
    });

    it('rejects invalid file types', () => {
      const onFileReject = vi.fn();
      render(
        <UploadZone
          onFileSelect={vi.fn()}
          onFileReject={onFileReject}
          accept={{ 'application/pdf': ['.pdf'] }}
        />
      );

      const file = new File(['test'], 'test.exe', { type: 'application/x-msdownload' });
      const dropzone = screen.getByTestId('upload-zone');

      fireEvent.drop(dropzone, {
        dataTransfer: { files: [file] },
      });

      expect(onFileReject).toHaveBeenCalled();
    });
  });

  describe('accessibility', () => {
    it('has accessible role and label', () => {
      render(<UploadZone onFileSelect={vi.fn()} />);

      const dropzone = screen.getByTestId('upload-zone');
      expect(dropzone).toHaveAttribute('role', 'button');
      expect(dropzone).toHaveAttribute('aria-label');
    });

    it('is keyboard accessible', () => {
      render(<UploadZone onFileSelect={vi.fn()} />);

      const dropzone = screen.getByTestId('upload-zone');
      expect(dropzone).toHaveAttribute('tabIndex', '0');
    });

    it('has tabIndex -1 when disabled', () => {
      render(<UploadZone onFileSelect={vi.fn()} disabled />);

      const dropzone = screen.getByTestId('upload-zone');
      expect(dropzone).toHaveAttribute('tabIndex', '-1');
    });
  });
});
```

---

### NEO-1.8-002: Write Component Tests (ProgressCard, ResultsViewer)

**Priority**: P1 (High)
**Effort**: 30 minutes
**Dependencies**: Sprint 1.7 Complete

**Description**:
Write comprehensive component tests for output components including state changes, tab switching, and loading states.

**Acceptance Criteria**:
- [ ] ProgressCard progress updates
- [ ] ProgressCard reconnection state
- [ ] ResultsViewer tab switching
- [ ] ResultsViewer content display
- [ ] Skeleton loading states
- [ ] Coverage >= 80% for tested components

**Deliverables**:
- `src/components/processing/ProgressCard.test.tsx`
- `src/components/results/ResultsViewer.test.tsx`

**Technical Notes**:
```typescript
// ProgressCard.test.tsx
import { render, screen } from '@testing-library/react';
import { describe, it, expect, vi } from 'vitest';
import { ProgressCard } from './ProgressCard';

describe('ProgressCard', () => {
  it('displays progress percentage', () => {
    render(
      <ProgressCard
        progress={{ stage: 'conversion', percent: 50, message: 'Converting...' }}
        connectionStatus="connected"
      />
    );

    expect(screen.getByText('50%')).toBeInTheDocument();
    expect(screen.getByText('Converting...')).toBeInTheDocument();
  });

  it('shows reconnecting state', () => {
    render(
      <ProgressCard
        progress={{ stage: 'conversion', percent: 50, message: 'Converting...' }}
        connectionStatus="reconnecting"
      />
    );

    expect(screen.getByText(/reconnecting/i)).toBeInTheDocument();
  });

  it('shows cancel button during processing', () => {
    const onCancel = vi.fn();
    render(
      <ProgressCard
        progress={{ stage: 'conversion', percent: 50, message: 'Converting...' }}
        connectionStatus="connected"
        onCancel={onCancel}
      />
    );

    expect(screen.getByRole('button', { name: /cancel/i })).toBeInTheDocument();
  });

  it('ensures progress never decreases (monotonic)', async () => {
    const { rerender } = render(
      <ProgressCard
        progress={{ stage: 'conversion', percent: 50, message: 'Processing...' }}
        connectionStatus="connected"
      />
    );

    expect(screen.getByText('50%')).toBeInTheDocument();

    // Simulate lower progress (should be ignored)
    rerender(
      <ProgressCard
        progress={{ stage: 'conversion', percent: 30, message: 'Processing...' }}
        connectionStatus="connected"
      />
    );

    // Should still show 50%
    expect(screen.getByText('50%')).toBeInTheDocument();
  });
});
```

---

### NEO-1.8-003: Write Store Tests (documentStore, uiStore)

**Priority**: P1 (High)
**Effort**: 30 minutes
**Dependencies**: Sprint 1.7 Complete

**Description**:
Write unit tests for Zustand stores covering state management, actions, and edge cases.

**Acceptance Criteria**:
- [ ] documentStore initial state
- [ ] documentStore mutual exclusion (file XOR URL)
- [ ] documentStore processing lock
- [ ] uiStore toast management
- [ ] uiStore modal management
- [ ] Coverage 100% for stores

**Deliverables**:
- `src/stores/documentStore.test.ts`
- `src/stores/uiStore.test.ts`

**Technical Notes**:
```typescript
// documentStore.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { useDocumentStore } from './documentStore';

describe('documentStore', () => {
  beforeEach(() => {
    useDocumentStore.getState().reset();
  });

  describe('mutual exclusion', () => {
    it('clears URL when file is set', () => {
      const store = useDocumentStore.getState();
      store.setUrl('https://example.com', 'job-1');
      store.setFile(new File(['test'], 'test.pdf'), 'job-2');

      const state = useDocumentStore.getState();
      expect(state.file).not.toBeNull();
      expect(state.url).toBeNull();
      expect(state.activeInput).toBe('file');
    });

    it('clears file when URL is set', () => {
      const store = useDocumentStore.getState();
      store.setFile(new File(['test'], 'test.pdf'), 'job-1');
      store.setUrl('https://example.com', 'job-2');

      const state = useDocumentStore.getState();
      expect(state.file).toBeNull();
      expect(state.url).toBe('https://example.com');
      expect(state.activeInput).toBe('url');
    });
  });

  describe('processing lock', () => {
    it('sets processing state', () => {
      useDocumentStore.getState().setProcessing(true);
      expect(useDocumentStore.getState().isProcessing).toBe(true);
    });
  });

  describe('reset', () => {
    it('returns to initial state', () => {
      const store = useDocumentStore.getState();
      store.setFile(new File(['test'], 'test.pdf'), 'job-1');
      store.setProcessing(true);
      store.reset();

      const state = useDocumentStore.getState();
      expect(state.file).toBeNull();
      expect(state.url).toBeNull();
      expect(state.isProcessing).toBe(false);
    });
  });
});
```

---

### NEO-1.8-004: Support API Route Testing

**Priority**: P1 (High)
**Effort**: 45 minutes
**Dependencies**: Sprint 1.7 Complete

**Description**:
Support Julia in writing API route tests, focusing on Next.js route handler testing patterns and MSW integration.

**Acceptance Criteria**:
- [ ] Upload route tests with file handling
- [ ] Process route tests with MCP mocking
- [ ] History route pagination tests
- [ ] Health route service status tests
- [ ] Error handling tests

**Deliverables**:
- Test utilities for API route testing
- Support for comprehensive API test coverage

**Technical Notes**:
```typescript
// test/utils/api-test-utils.ts

import { NextRequest } from 'next/server';

export function createMockRequest(
  url: string,
  options: {
    method?: string;
    headers?: Record<string, string>;
    body?: unknown;
    cookies?: Record<string, string>;
    isFormData?: boolean; // Flag to treat plain object as form data
  } = {}
): NextRequest {
  const { method = 'GET', headers = {}, body, cookies = {}, isFormData = false } = options;

  // Determine if body is FormData (native instance or flagged as form data)
  const isFormDataBody = body instanceof FormData || (isFormData && body && typeof body === 'object');

  const requestInit: RequestInit = {
    method,
    headers: isFormDataBody
      ? { ...headers } // Let browser/runtime set multipart/form-data boundary
      : {
          'Content-Type': 'application/json',
          ...headers,
        },
  };

  if (body && method !== 'GET') {
    if (body instanceof FormData) {
      // Use FormData instance directly
      requestInit.body = body;
    } else if (isFormData && typeof body === 'object') {
      // Convert plain object to FormData
      const formData = new FormData();
      Object.entries(body as Record<string, unknown>).forEach(([key, value]) => {
        if (value instanceof Blob || value instanceof File) {
          formData.append(key, value);
        } else if (value !== undefined && value !== null) {
          formData.append(key, String(value));
        }
      });
      requestInit.body = formData;
    } else {
      // JSON handling for regular objects
      requestInit.body = JSON.stringify(body);
    }
  }

  const request = new NextRequest(new URL(url, 'http://localhost:3000'), requestInit);

  // Add cookies
  Object.entries(cookies).forEach(([name, value]) => {
    request.cookies.set(name, value);
  });

  return request;
}

// Example usage in tests
// import { createMockRequest } from '@/test/utils/api-test-utils';
// const request = createMockRequest('/api/v1/history?page=1', {
//   cookies: { session: 'test-session-id' },
// });
```

---

### NEO-1.8-005: Write E2E Critical Path Test

**Priority**: P0 (Critical)
**Effort**: 30 minutes
**Dependencies**: All previous sprints

**Description**:
Write the primary E2E test covering the critical user journey: Upload document -> View progress -> View results.

**Acceptance Criteria**:
- [ ] Test file upload via drag-drop
- [ ] Test progress display during processing
- [ ] Test results display on completion
- [ ] Test download functionality
- [ ] Test passes in CI environment

**Deliverables**:
- `test/e2e/critical-path.spec.ts`

**Technical Notes**:
```typescript
// test/e2e/critical-path.spec.ts

import { test, expect } from '@playwright/test';
import path from 'path';
import { fileURLToPath } from 'url';

const __dirname = path.dirname(fileURLToPath(import.meta.url));

test.describe('Document Processing Critical Path', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/');
  });

  test('complete document processing flow', async ({ page }) => {
    // Step 1: Upload file
    const fileInput = page.locator('input[type="file"]');
    const testFile = path.join(__dirname, '../fixtures/test-document.pdf');
    await fileInput.setInputFiles(testFile);

    // Verify file preview
    await expect(page.getByText('test-document.pdf')).toBeVisible();

    // Step 2: Start processing
    const processButton = page.getByRole('button', { name: /process/i });
    await processButton.click();

    // Step 3: Wait for progress
    await expect(page.getByText(/processing/i)).toBeVisible({ timeout: 10000 });

    // Verify progress updates
    const progressBar = page.locator('[role="progressbar"]');
    await expect(progressBar).toBeVisible();

    // Step 4: Wait for completion (may take a while)
    await expect(page.getByText(/complete/i)).toBeVisible({ timeout: 60000 });

    // Step 5: Verify results display
    await expect(page.getByRole('tablist')).toBeVisible();
    await expect(page.getByRole('tab', { name: /markdown/i })).toHaveAttribute(
      'aria-selected',
      'true'
    );

    // Step 6: Test tab switching
    await page.getByRole('tab', { name: /html/i }).click();
    await expect(page.getByRole('tab', { name: /html/i })).toHaveAttribute(
      'aria-selected',
      'true'
    );

    // Step 7: Test download
    const downloadButton = page.getByRole('button', { name: /download/i });
    await expect(downloadButton).toBeVisible();

    // Trigger download and verify
    const [download] = await Promise.all([
      page.waitForEvent('download'),
      downloadButton.click(),
    ]);

    expect(download.suggestedFilename()).toContain('.md');
  });

  test('URL processing flow', async ({ page }) => {
    // Click URL tab or switch to URL input
    await page.getByRole('tab', { name: /url/i }).click();

    // Enter URL
    const urlInput = page.getByPlaceholder(/https/i);
    await urlInput.fill('https://example.com/page');

    // Submit
    await page.getByRole('button', { name: /process url/i }).click();

    // Wait for processing
    await expect(page.getByText(/processing/i)).toBeVisible({ timeout: 10000 });

    // Wait for completion
    await expect(page.getByText(/complete/i)).toBeVisible({ timeout: 30000 });
  });
});
```

---

### NEO-1.8-006: Run Accessibility Audit and Fix Issues

**Priority**: P1 (High)
**Effort**: 45 minutes
**Dependencies**: All previous sprints

**Description**:
Run comprehensive accessibility audit using axe-core and fix any identified issues to achieve Lighthouse accessibility score >= 90.

**Acceptance Criteria**:
- [ ] axe-core audit passes on all pages
- [ ] Lighthouse accessibility >= 90
- [ ] WAVE audit passes
- [ ] Keyboard navigation verified
- [ ] Screen reader testing completed
- [ ] Color contrast verified

**Deliverables**:
- Accessibility fixes across all components
- `test/a11y/accessibility.test.ts`

**Technical Notes**:
```typescript
// test/a11y/accessibility.test.ts

import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test.describe('Accessibility', () => {
  test('home page passes accessibility audit', async ({ page }) => {
    await page.goto('/');

    const accessibilityScanResults = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa', 'wcag21aa'])
      .analyze();

    expect(accessibilityScanResults.violations).toEqual([]);
  });

  test('history page passes accessibility audit', async ({ page }) => {
    await page.goto('/history');

    const accessibilityScanResults = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa', 'wcag21aa'])
      .analyze();

    expect(accessibilityScanResults.violations).toEqual([]);
  });

  test('keyboard navigation works on upload zone', async ({ page }) => {
    await page.goto('/');

    // Tab to upload zone
    await page.keyboard.press('Tab');
    const uploadZone = page.getByTestId('upload-zone');
    await expect(uploadZone).toBeFocused();

    // Can activate with Enter
    await page.keyboard.press('Enter');
    // File dialog would open (can't test in Playwright)

    // Can activate with Space
    await page.keyboard.press('Space');
  });

  test('focus visible on all interactive elements', async ({ page }) => {
    await page.goto('/');

    // Tab through elements and verify focus visible
    const interactiveElements = await page.locator('button, a, input, [tabindex="0"]').all();

    for (let i = 0; i < Math.min(interactiveElements.length, 10); i++) {
      await page.keyboard.press('Tab');
      const focused = await page.locator(':focus');
      const outline = await focused.evaluate(
        (el) => getComputedStyle(el).outline || getComputedStyle(el).boxShadow
      );
      expect(outline).not.toBe('none');
    }
  });
});
```

---

### NEO-1.8-007: Write README with Setup Instructions

**Priority**: P0 (Critical)
**Effort**: 30 minutes
**Dependencies**: All previous sprints

**Description**:
Write comprehensive README.md with installation, configuration, and running instructions.

**Acceptance Criteria**:
- [ ] Project overview and features
- [ ] Prerequisites (Node.js, environment)
- [ ] Installation steps
- [ ] Configuration (environment variables)
- [ ] Running locally (dev, build, start)
- [ ] Running tests
- [ ] Project structure overview
- [ ] Contributing guidelines (brief)

**Deliverables**:
- `README.md`

**Technical Notes**:
```markdown
# HX Docling UI

A modern web interface for document processing powered by the HX-Infrastructure Docling MCP Server.

## Features

- **File Upload**: Drag-drop or click-to-browse for PDF, Word, Excel, PowerPoint, and images
- **URL Processing**: Convert web pages to structured documents
- **Real-time Progress**: SSE-based progress tracking with automatic reconnection
- **Multiple Export Formats**: Markdown, HTML, JSON output
- **Processing History**: View past conversions with re-download capability
- **Dark Mode**: Full dark/light theme support

## Prerequisites

- Node.js 20.x or higher
- Access to HX-Infrastructure services:
  - hx-postgres-server (PostgreSQL database)
  - hx-redis-server (Redis session store)
  - hx-docling-mcp-server (Document processing)

## Installation

1. Clone the repository:
\`\`\`bash
git clone <repository-url>
cd hx-docling-ui
\`\`\`

2. Install dependencies:
\`\`\`bash
npm install
\`\`\`

3. Copy environment template:
\`\`\`bash
cp .env.example .env.local
\`\`\`

4. Configure environment variables (see Configuration section)

5. Generate Prisma client:
\`\`\`bash
npx prisma generate
\`\`\`

6. Run database migrations:
\`\`\`bash
npx prisma migrate deploy
\`\`\`

## Configuration

Create a `.env.local` file with the following variables:

\`\`\`env
# Database
DATABASE_URL="postgresql://user:password@hx-postgres-server:6432/docling_db?sslmode=require"
DIRECT_DATABASE_URL="postgresql://user:password@hx-postgres-server:5432/docling_db?sslmode=require"

# Redis
REDIS_URL="rediss://hx-redis-server:6379"

# MCP Server
MCP_SERVER_URL="http://hx-docling-server:8000"

# File Storage
UPLOAD_PATH="/data/docling-uploads"

# Session
SESSION_SECRET="your-session-secret-32-chars-min"

# Application
NEXT_PUBLIC_APP_URL="http://localhost:3000"
\`\`\`

## Running Locally

### Development Server
\`\`\`bash
npm run dev
\`\`\`
Open [http://localhost:3000](http://localhost:3000)

### Production Build
\`\`\`bash
npm run build
npm run start
\`\`\`

## Testing

### Unit Tests
\`\`\`bash
npm run test
npm run test:coverage
\`\`\`

### E2E Tests
\`\`\`bash
npm run test:e2e
\`\`\`

### Type Checking
\`\`\`bash
npm run typecheck
\`\`\`

### Linting
\`\`\`bash
npm run lint
\`\`\`

## Project Structure

\`\`\`
src/
  app/                  # Next.js App Router
    api/v1/            # API routes
    history/           # History page
  components/          # React components
    ui/               # shadcn/ui components
    upload/           # Upload components
    processing/       # Progress components
    results/          # Results components
    history/          # History components
  lib/                 # Utilities
    db/               # Database client
    redis/            # Redis client
    mcp/              # MCP client
    validation/       # Validation schemas
  stores/              # Zustand stores
  hooks/               # React hooks
  types/               # TypeScript types
test/
  e2e/                # Playwright E2E tests
  fixtures/           # Test fixtures
\`\`\`

## API Documentation

See API specification in `project/0.3-specification/0.3.1-detailed-specification.md`

## License

Proprietary - HX Infrastructure
```

---

### NEO-1.8-008: Update CLAUDE.md with Project Context

**Priority**: P0 (Critical)
**Effort**: 45 minutes
**Dependencies**: All previous sprints

**Description**:
Update CLAUDE.md with comprehensive project context including architecture, patterns, troubleshooting, and key implementation details for AI assistant guidance.

**Acceptance Criteria**:
- [ ] Project overview and purpose
- [ ] Development commands documented
- [ ] Architecture overview
- [ ] Key patterns documented
- [ ] Redis key patterns
- [ ] SSE reconnection sequence
- [ ] MCP error code mapping
- [ ] Circuit breaker states
- [ ] Progress interpolation
- [ ] Checkpoint serialization
- [ ] Troubleshooting guide

**Deliverables**:
- `CLAUDE.md`

**Technical Notes**:
```markdown
# CLAUDE.md

This file provides guidance to Claude Code when working with the HX Docling UI codebase.

## Project Overview

HX Docling UI is a Next.js 16 application providing a web interface for document processing via the HX-Infrastructure Docling MCP Server. Users can upload files or provide URLs for conversion to structured formats (Markdown, HTML, JSON).

## Development Commands

### Setup
\`\`\`bash
npm install
npx prisma generate
npx prisma migrate deploy
\`\`\`

### Building
\`\`\`bash
npm run build      # Production build
npm run typecheck  # TypeScript checking
npm run lint       # ESLint
\`\`\`

### Testing
\`\`\`bash
npm run test           # Unit tests (Vitest)
npm run test:coverage  # With coverage report
npm run test:e2e       # E2E tests (Playwright)
\`\`\`

### Running
\`\`\`bash
npm run dev   # Development server (port 3000)
npm run start # Production server
\`\`\`

## Architecture

### Overall Structure
- **App Router**: Next.js 16 with Server Components by default
- **Client Components**: Only for interactivity (forms, SSE, state)
- **API Routes**: /api/v1/* for all backend endpoints
- **State Management**: Zustand for client state
- **Database**: PostgreSQL via Prisma ORM
- **Sessions**: Redis with circuit breaker
- **Processing**: MCP protocol to hx-docling-mcp-server

### Key Components

1. **Upload Layer** (`components/upload/`)
   - UploadZone: Drag-drop file selection
   - UrlInput: URL entry with SSRF prevention
   - UploadForm: react-hook-form integration

2. **Processing Layer** (`components/processing/`)
   - ProgressCard: Real-time progress display
   - LoadingStates: Skeleton and spinner components

3. **Results Layer** (`components/results/`)
   - ResultsViewer: Tabbed interface (MD, HTML, JSON, Raw)
   - Format-specific renderers

4. **History Layer** (`components/history/`)
   - HistoryView: Paginated job list
   - JobDetail: Full job information modal

### Important Patterns

#### Redis Key Patterns
\`\`\`
hx-docling:session:{sessionId}     # Session data (24h TTL)
hx-docling:events:{jobId}          # SSE event buffer (5min TTL)
hx-docling:rate:{keyPrefix}:{id}   # Rate limiting counters
hx-docling:idempotency:{key}       # Idempotency records (1h TTL)
\`\`\`

#### SSE Reconnection Sequence
1. Client EventSource connects to /api/v1/process/{jobId}/events
2. Server sends `connected` event with connectionId
3. Server sends `progress` events with increasing event IDs
4. On disconnect, client reconnects with Last-Event-ID header
5. Server sends `state_sync` event with current full state
6. Server replays events from Last-Event-ID from Redis buffer
7. Processing continues normally

#### MCP Error Code Mapping
| JSON-RPC Code | App Code | HTTP Status | Meaning |
|---------------|----------|-------------|---------|
| -32700 | E201 | 400 | Parse error |
| -32600 | E202 | 400 | Invalid request |
| -32601 | E203 | 404 | Method not found |
| -32602 | E204 | 400 | Invalid params |
| -32603 | E205 | 500 | Internal error |
| -32000 to -32099 | E206 | 503 | Server errors |

#### Circuit Breaker States
\`\`\`
CLOSED -> (N failures) -> OPEN -> (timeout) -> HALF_OPEN
HALF_OPEN -> (success) -> CLOSED
HALF_OPEN -> (failure) -> OPEN
\`\`\`
Default: 5 failures to open, 30s timeout, 3 half-open attempts

#### Progress Interpolation
- Progress percentage NEVER decreases (monotonic guarantee)
- smoothProgress function tracks max seen percentage
- Stage transitions bump to stage minimum if higher

#### Checkpoint Serialization
\`\`\`typescript
interface Checkpoint {
  stage: string;           // Current processing stage
  completedStages: string[]; // Completed stages
  doclingDocument?: unknown; // Intermediate result
  results: Partial<ProcessingResult>; // Completed exports
  timestamp: number;       // Checkpoint creation time
}
// Stored as JSONB in Job.checkpointData
\`\`\`

## Troubleshooting

### Common Issues

1. **SSE Connection Drops**
   - Check Redis connectivity (circuit breaker may be open)
   - Verify no proxy timeouts (increase keep-alive)
   - Check event buffer not exhausted

2. **MCP Timeout**
   - Large files have longer timeouts (up to 300s)
   - Check MCP server health endpoint
   - Verify network connectivity

3. **Upload Failures**
   - Check file size limits (100MB PDF, 50MB Office, 25MB images)
   - Verify file type is supported
   - Check /data partition has space

4. **Session Issues**
   - Redis connection may be failing (check circuit breaker)
   - Session cookie may be expired (24h TTL)
   - Clear cookies and retry

### Debug Commands
\`\`\`bash
# Check health endpoint
curl http://localhost:3000/api/v1/health

# Check Redis
redis-cli -h hx-redis-server PING

# Check PostgreSQL
psql $DATABASE_URL -c "SELECT 1"

# Check MCP
curl http://hx-docling-server:8000/health
\`\`\`
```

---

### NEO-1.8-009: Create Test Fixtures

**Priority**: P1 (High)
**Effort**: 15 minutes
**Dependencies**: NEO-1.8-005

**Description**:
Create test fixtures including sample documents for E2E testing.

**Acceptance Criteria**:
- [ ] Sample PDF document
- [ ] Sample DOCX document
- [ ] Sample image (PNG/JPG)
- [ ] Sample URL fixture
- [ ] Fixtures are small but representative

**Deliverables**:
- `test/fixtures/test-document.pdf`
- `test/fixtures/test-document.docx`
- `test/fixtures/test-image.png`
- `test/fixtures/README.md`

**Technical Notes**:
- PDF: Simple 1-page document with text and a table
- DOCX: Simple document with headings and paragraphs
- Image: Small PNG with some text content
- Keep files under 1MB for fast tests

---

### NEO-1.8-010: Final Quality Gate Verification

**Priority**: P0 (Critical)
**Effort**: 20 minutes
**Dependencies**: All previous tasks

**Description**:
Run all quality gates and verify the application meets Phase 1 completion criteria.

**Acceptance Criteria**:
- [ ] `npm run typecheck` - 0 errors
- [ ] `npm run lint` - 0 errors, < 10 warnings
- [ ] `npm run test` - All pass
- [ ] `npm run test:coverage` - >= 80% lines, >= 75% branches
- [ ] `npm run test:e2e` - All pass
- [ ] Lighthouse accessibility >= 90
- [ ] Build succeeds

**Deliverables**:
- Quality gate verification report
- Any final fixes needed

**Technical Notes**:
```bash
# Quality Gate Checklist

# 1. TypeScript
npm run typecheck
# Expected: 0 errors

# 2. Lint
npm run lint
# Expected: 0 errors, < 10 warnings

# 3. Unit Tests
npm run test
# Expected: All pass

# 4. Coverage
npm run test:coverage
# Expected: >= 80% lines, >= 75% branches

# 5. E2E Tests
npm run test:e2e
# Expected: All pass

# 6. Build
npm run build
# Expected: Success

# 7. Accessibility (manual)
# Run Lighthouse on http://localhost:3000
# Expected: Accessibility >= 90
```

---

## Summary

| Task ID | Task Title | Effort | Priority |
|---------|-----------|--------|----------|
| NEO-1.8-001 | Component Tests (Input) | 30m | P1 |
| NEO-1.8-002 | Component Tests (Output) | 30m | P1 |
| NEO-1.8-003 | Store Tests | 30m | P1 |
| NEO-1.8-004 | Support API Route Testing | 45m | P1 |
| NEO-1.8-005 | E2E Critical Path Test | 30m | P0 |
| NEO-1.8-006 | Accessibility Audit | 45m | P1 |
| NEO-1.8-007 | Write README | 30m | P0 |
| NEO-1.8-008 | Update CLAUDE.md | 45m | P0 |
| NEO-1.8-009 | Create Test Fixtures | 15m | P1 |
| NEO-1.8-010 | Quality Gate Verification | 20m | P0 |

**Total Neo Effort**: ~5.3 hours (320 minutes)
**Total Tasks**: 10

---

## Dependencies

Neo's tasks in Sprint 1.8 support Julia's lead role:

```
Sprint 1.7 Complete
    |
    +-> JULIA: Testing Coordination (Lead)
    |       |
    |       +-> NEO-1.8-001 (Input Component Tests) [Support]
    |       +-> NEO-1.8-002 (Output Component Tests) [Support]
    |       +-> NEO-1.8-003 (Store Tests) [Support]
    |       +-> NEO-1.8-004 (API Test Support) [Support]
    |       +-> NEO-1.8-005 (E2E Test) [Support]
    |       +-> NEO-1.8-006 (Accessibility) [Support]
    |
    +-> NEO-1.8-007 (README) [Independent]
    |
    +-> NEO-1.8-008 (CLAUDE.md) [Independent]
    |
    +-> NEO-1.8-009 (Fixtures) [Depends on 005]
    |
    +-> NEO-1.8-010 (Quality Gates) [Final]
```

---

## Coordination Notes

- **Primary Lead**: Julia (@julia) owns testing coordination
- **Neo's Role**: Component testing, documentation, quality verification
- **Trinity's Role**: Database testing support
- **William's Role**: Infrastructure testing support
- **Handoff**: All tests must pass before Phase 1 signoff
- **Review**: Agent Zero performs final review
