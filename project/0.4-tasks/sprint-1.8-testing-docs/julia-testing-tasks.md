# Julia Santos - Testing & QA Tasks: Sprint 1.8 (Comprehensive Testing & Documentation)

**Sprint**: 1.8 - Testing & Documentation
**Role**: Testing & Quality Assurance Specialist (Sprint Lead)
**Document Version**: 1.0.0
**Created**: 2025-12-12
**References**:
- Implementation Plan v1.2.0 Section 4.8
- Detailed Specification v1.2.0 Sections 10, 11

---

## Sprint 1.8 Overview

Sprint 1.8 is the final quality assurance sprint where I serve as the Sprint Lead. This sprint consolidates all testing efforts, fills coverage gaps, implements comprehensive E2E tests, security tests, state machine tests, and accessibility audits. The goal is to achieve production-ready quality with all quality gates passing.

**Planned Duration**: 6.0 hours (Sessions 9-11)
**Task Effort Total**: 10.0 hours (see task breakdown below)
**Lead**: Julia Santos (`@julia`)
**Support**: Neo (`@neo`), Trinity (`@trinity`), William (`@william`)
**Review**: Agent Zero (`@agent-zero`)

---

## Session 1: Comprehensive Unit & Integration Testing (~3.0 hours)

### JUL-1.8-001: Validation Library Coverage Completion

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.8-001 |
| **Title** | Complete Unit Test Coverage for Validation Libraries |
| **Priority** | P0 (Critical) |
| **Effort** | 0.75 hours |
| **Dependencies** | All Sprint 1.3-1.7 validation tasks complete |

#### Description

Complete unit test coverage for all validation libraries (file, URL, SSRF) ensuring all edge cases and error scenarios are covered.

#### Acceptance Criteria

- [ ] File validation: all type/size combinations tested
- [ ] URL validation: all protocol/format combinations tested
- [ ] SSRF validation: all blocked patterns verified
- [ ] Magic byte validation tested
- [ ] Content-Length validation tested
- [ ] Error messages verified for each validation failure
- [ ] Coverage >= 95% for all validation modules

#### Technical Notes

```typescript
// Additional validation tests to ensure complete coverage
describe('Validation Library Complete Coverage', () => {
  describe('File Validation Edge Cases', () => {
    it('handles filename with multiple dots', () => {
      const result = fileSchema.safeParse({
        name: 'document.backup.pdf',
        size: 1024,
        type: 'application/pdf',
      });
      expect(result.success).toBe(true);
    });

    it('handles filename with special characters', () => {
      const result = fileSchema.safeParse({
        name: 'doc (1) - copy.pdf',
        size: 1024,
        type: 'application/pdf',
      });
      expect(result.success).toBe(true);
    });

    it('handles exact size limits', () => {
      // PDF at exactly 100MB
      expect(fileSchema.safeParse({
        name: 'exact.pdf',
        size: 100 * 1024 * 1024,
        type: 'application/pdf',
      }).success).toBe(true);

      // PDF at 100MB + 1 byte
      expect(fileSchema.safeParse({
        name: 'over.pdf',
        size: 100 * 1024 * 1024 + 1,
        type: 'application/pdf',
      }).success).toBe(false);
    });
  });

  describe('URL Validation Edge Cases', () => {
    it('handles URLs with authentication', () => {
      const result = urlSchema.safeParse('https://user:pass@example.com/doc');
      expect(result.success).toBe(true);
    });

    it('handles URLs with port numbers', () => {
      const result = urlSchema.safeParse('https://example.com:8443/doc');
      expect(result.success).toBe(true);
    });

    it('handles URLs with special characters in path', () => {
      const result = urlSchema.safeParse('https://example.com/path%20with%20spaces');
      expect(result.success).toBe(true);
    });
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| Complete validation tests | `src/lib/validation/__tests__/*.test.ts` |
| Coverage report | `coverage/validation/` |

---

### JUL-1.8-002: MCP Client Complete Test Coverage

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.8-002 |
| **Title** | Complete Unit Test Coverage for MCP Client |
| **Priority** | P0 (Critical) |
| **Effort** | 0.75 hours |
| **Dependencies** | Sprint 1.5a MCP tasks complete |

#### Description

Complete unit test coverage for MCP client including initialization, tool invocations, error handling, and protocol compliance.

#### Acceptance Criteria

- [ ] Initialization sequence fully tested (3-step)
- [ ] All 8 tools tested with valid/invalid parameters
- [ ] All JSON-RPC error codes tested
- [ ] Retry logic with exponential backoff tested
- [ ] Circuit breaker states tested (if implemented)
- [ ] Timeout handling tested for all size categories
- [ ] Coverage >= 95% for MCP client

#### Technical Notes

```typescript
// Complete MCP client test coverage
describe('MCP Client Complete Coverage', () => {
  describe('Retry Logic', () => {
    it('retries on transient failure', async () => {
      let attempts = 0;
      server.use(
        http.post('http://hx-docling-server:8000/rpc', () => {
          attempts++;
          if (attempts < 3) {
            return HttpResponse.json({
              jsonrpc: '2.0',
              error: { code: -32603, message: 'Internal error' },
            });
          }
          return HttpResponse.json({
            jsonrpc: '2.0',
            result: { content: [] },
          });
        })
      );

      const client = new MCPClient('http://hx-docling-server:8000');
      await client.initialize();
      await client.invokeTool('convert_pdf', { file_path: '/test.pdf' });

      expect(attempts).toBe(3);
    });

    it('fails after max retries', async () => {
      server.use(
        http.post('http://hx-docling-server:8000/rpc', () => {
          return HttpResponse.json({
            jsonrpc: '2.0',
            error: { code: -32603, message: 'Internal error' },
          });
        })
      );

      const client = new MCPClient('http://hx-docling-server:8000');
      await client.initialize();

      await expect(
        client.invokeTool('convert_pdf', { file_path: '/test.pdf' })
      ).rejects.toThrow(/max retries/i);
    });

    it('uses exponential backoff', async () => {
      const delays: number[] = [];
      let lastTime = Date.now();

      server.use(
        http.post('http://hx-docling-server:8000/rpc', async () => {
          delays.push(Date.now() - lastTime);
          lastTime = Date.now();
          return HttpResponse.json({
            jsonrpc: '2.0',
            error: { code: -32603, message: 'Internal error' },
          });
        })
      );

      const client = new MCPClient('http://hx-docling-server:8000', {
        baseDelay: 100,
      });
      await client.initialize();

      try {
        await client.invokeTool('convert_pdf', { file_path: '/test.pdf' });
      } catch {}

      // Verify exponential increase (with some tolerance)
      expect(delays[1]).toBeGreaterThan(delays[0] * 1.5);
      expect(delays[2]).toBeGreaterThan(delays[1] * 1.5);
    });
  });

  describe('All Tools', () => {
    const TOOLS = [
      { name: 'convert_pdf', params: { file_path: '/test.pdf' } },
      { name: 'convert_docx', params: { file_path: '/test.docx' } },
      { name: 'convert_xlsx', params: { file_path: '/test.xlsx' } },
      { name: 'convert_pptx', params: { file_path: '/test.pptx' } },
      { name: 'convert_url', params: { url: 'https://example.com' } },
      { name: 'export_markdown', params: { document: {} } },
      { name: 'export_html', params: { document: {} } },
      { name: 'export_json', params: { document: {} } },
    ];

    TOOLS.forEach(({ name, params }) => {
      it(`invokes ${name} successfully`, async () => {
        const client = new MCPClient('http://hx-docling-server:8000');
        await client.initialize();

        const result = await client.invokeTool(name, params);
        expect(result).toBeDefined();
      });
    });
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| Complete MCP tests | `src/lib/mcp/__tests__/*.test.ts` |
| Coverage report | `coverage/mcp/` |

---

### JUL-1.8-003: Component Test Coverage Completion

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.8-003 |
| **Title** | Complete Component Test Coverage |
| **Priority** | P1 (High) |
| **Effort** | 0.5 hours |
| **Dependencies** | All component sprints complete |

#### Description

Review and complete component test coverage for all UI components ensuring user interactions and states are tested.

#### Acceptance Criteria

- [ ] UploadZone: all states and interactions tested
- [ ] UrlInput: all states and interactions tested
- [ ] ProgressCard: all progress states tested
- [ ] ResultsViewer: all tabs and formats tested
- [ ] HistoryView: all states and interactions tested
- [ ] Coverage >= 90% for all components

#### Deliverables

| Deliverable | Path |
|-------------|------|
| Component test updates | `src/components/**/__tests__/*.test.tsx` |

---

### JUL-1.8-004: Store Test Coverage Completion

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.8-004 |
| **Title** | Complete Store Test Coverage |
| **Priority** | P1 (High) |
| **Effort** | 0.5 hours |
| **Dependencies** | Sprint 1.4 store tasks complete |

#### Description

Complete test coverage for Zustand stores (documentStore, uiStore) including all state transitions and edge cases.

#### Acceptance Criteria

- [ ] documentStore: all actions and selectors tested
- [ ] uiStore: all toast/modal states tested
- [ ] State persistence verified (if applicable)
- [ ] SSR compatibility verified
- [ ] Coverage >= 95% for stores

#### Technical Notes

```typescript
// Complete store tests
describe('Store Complete Coverage', () => {
  describe('documentStore selectors', () => {
    it('hasInput returns true when file set', () => {
      useDocumentStore.getState().setFile(new File([''], 'test.pdf'));
      expect(useDocumentStore.getState().hasInput).toBe(true);
    });

    it('hasInput returns true when URL set', () => {
      useDocumentStore.getState().setUrl('https://example.com');
      expect(useDocumentStore.getState().hasInput).toBe(true);
    });

    it('hasInput returns false when empty', () => {
      useDocumentStore.getState().clearInputs();
      expect(useDocumentStore.getState().hasInput).toBe(false);
    });
  });

  describe('uiStore', () => {
    it('shows toast with correct duration', () => {
      vi.useFakeTimers();
      useUIStore.getState().showToast({ message: 'Test', type: 'success' });

      expect(useUIStore.getState().toast).toBeDefined();

      vi.advanceTimersByTime(5000);
      expect(useUIStore.getState().toast).toBeNull();

      vi.useRealTimers();
    });

    it('manages modal state', () => {
      useUIStore.getState().openModal('confirm');
      expect(useUIStore.getState().activeModal).toBe('confirm');

      useUIStore.getState().closeModal();
      expect(useUIStore.getState().activeModal).toBeNull();
    });
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| Complete store tests | `src/stores/__tests__/*.test.ts` |

---

### JUL-1.8-005: API Route Test Coverage Expansion

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.8-005 |
| **Title** | Expand API Route Test Coverage |
| **Priority** | P0 (Critical) |
| **Effort** | 1.5 hours |
| **Dependencies** | All API routes implemented |

#### Description

Complete integration tests for all API routes including validation, error handling, and edge cases.

#### Acceptance Criteria

- [ ] Upload route: all scenarios tested
- [ ] Process route: MCP integration tested
- [ ] History route: pagination and filtering tested
- [ ] Job detail route: authorization tested
- [ ] Cancel route: all state transitions tested
- [ ] Resume route: checkpoint handling tested
- [ ] Health route: all service statuses tested
- [ ] Coverage >= 90% for all API routes

#### Technical Notes

```typescript
// Complete API route tests
describe('API Route Complete Coverage', () => {
  describe('POST /api/v1/process', () => {
    it('initiates processing for file', async () => {
      // Create job first
      const job = await prisma.job.create({
        data: {
          sessionId: 'test-session',
          status: 'PENDING',
          fileName: 'test.pdf',
          filePath: '/data/uploads/test.pdf',
        },
      });

      const request = new Request(
        `http://localhost/api/v1/process`,
        {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            Cookie: 'session=test-session',
          },
          body: JSON.stringify({ jobId: job.id }),
        }
      );

      const response = await POST(request);
      expect(response.status).toBe(200);
    });

    it('returns SSE stream', async () => {
      const response = await fetch('/api/v1/process/job-123/events');
      expect(response.headers.get('Content-Type')).toBe('text/event-stream');
    });
  });

  describe('GET /api/v1/health', () => {
    it('returns status for all services', async () => {
      const response = await GET(new Request('http://localhost/api/v1/health'));
      const data = await response.json();

      expect(data).toHaveProperty('database');
      expect(data).toHaveProperty('redis');
      expect(data).toHaveProperty('mcp');
      expect(data).toHaveProperty('storage');
    });

    it('returns overall healthy status', async () => {
      const response = await GET(new Request('http://localhost/api/v1/health'));
      const data = await response.json();

      expect(data.status).toBe('healthy');
    });

    it('returns degraded when service unavailable', async () => {
      // Mock Redis failure
      vi.mocked(redis.ping).mockRejectedValueOnce(new Error('Connection refused'));

      const response = await GET(new Request('http://localhost/api/v1/health'));
      const data = await response.json();

      expect(data.status).toBe('degraded');
      expect(data.redis.status).toBe('unhealthy');
    });
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| Complete API tests | `src/app/api/**/__tests__/*.test.ts` |

---

### JUL-1.8-006: State Machine Transition Tests

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.8-006 |
| **Title** | Write State Machine Transition Tests for All 10 Job States |
| **Priority** | P0 (Critical) |
| **Effort** | 0.75 hours |
| **Dependencies** | Job status enum defined |

#### Description

Write comprehensive state machine tests covering all 10 job states and all valid/invalid state transitions as defined in the specification.

#### Acceptance Criteria

- [ ] All 10 states tested: PENDING, UPLOADING, PROCESSING, RETRY_PENDING, RETRY_PROCESSING, PARTIAL_COMPLETE, COMPLETED, FAILED, CANCELLED, EXPIRED
- [ ] All valid transitions tested
- [ ] Invalid transitions rejected with appropriate errors
- [ ] Transition side effects verified (e.g., timestamp updates)
- [ ] Concurrent transition handling tested
- [ ] Coverage 100% for state machine logic

#### Technical Notes

```typescript
// test/state-machine/job-status.test.ts
import { describe, it, expect } from 'vitest';
import { JobStateMachine, JobStatus } from '@/lib/state-machine/job';

describe('Job State Machine', () => {
  const VALID_TRANSITIONS: Record<JobStatus, JobStatus[]> = {
    PENDING: ['UPLOADING', 'CANCELLED'],
    UPLOADING: ['PROCESSING', 'FAILED', 'CANCELLED'],
    PROCESSING: ['COMPLETED', 'PARTIAL_COMPLETE', 'FAILED', 'RETRY_PENDING', 'CANCELLED'],
    RETRY_PENDING: ['RETRY_PROCESSING', 'CANCELLED'],
    RETRY_PROCESSING: ['COMPLETED', 'PARTIAL_COMPLETE', 'FAILED', 'CANCELLED'],
    PARTIAL_COMPLETE: ['COMPLETED'], // Can complete remaining exports
    COMPLETED: [], // Terminal state
    FAILED: [], // Terminal state (can resume via checkpoint)
    CANCELLED: [], // Terminal state
    EXPIRED: [], // Terminal state
  };

  describe('Valid Transitions', () => {
    Object.entries(VALID_TRANSITIONS).forEach(([from, toStates]) => {
      toStates.forEach((to) => {
        it(`allows transition from ${from} to ${to}`, () => {
          const sm = new JobStateMachine(from as JobStatus);
          expect(() => sm.transition(to as JobStatus)).not.toThrow();
          expect(sm.current).toBe(to);
        });
      });
    });
  });

  describe('Invalid Transitions', () => {
    const INVALID_TRANSITIONS = [
      { from: 'COMPLETED', to: 'PROCESSING' },
      { from: 'CANCELLED', to: 'PROCESSING' },
      { from: 'FAILED', to: 'COMPLETED' },
      { from: 'PENDING', to: 'COMPLETED' },
      { from: 'UPLOADING', to: 'COMPLETED' },
      { from: 'PROCESSING', to: 'PENDING' },
    ];

    INVALID_TRANSITIONS.forEach(({ from, to }) => {
      it(`rejects transition from ${from} to ${to}`, () => {
        const sm = new JobStateMachine(from as JobStatus);
        expect(() => sm.transition(to as JobStatus)).toThrow(/invalid transition/i);
      });
    });
  });

  describe('Terminal States', () => {
    const TERMINAL_STATES: JobStatus[] = ['COMPLETED', 'FAILED', 'CANCELLED', 'EXPIRED'];

    TERMINAL_STATES.forEach((state) => {
      it(`${state} is a terminal state`, () => {
        const sm = new JobStateMachine(state);
        expect(sm.isTerminal).toBe(true);
      });
    });
  });

  describe('Transition Side Effects', () => {
    it('updates timestamps on transition', () => {
      const sm = new JobStateMachine('PENDING');
      const before = new Date();

      sm.transition('UPLOADING');

      expect(sm.lastTransitionAt.getTime()).toBeGreaterThanOrEqual(before.getTime());
    });

    it('records transition history', () => {
      const sm = new JobStateMachine('PENDING');
      sm.transition('UPLOADING');
      sm.transition('PROCESSING');
      sm.transition('COMPLETED');

      expect(sm.history).toEqual(['PENDING', 'UPLOADING', 'PROCESSING', 'COMPLETED']);
    });
  });

  describe('Cancellation from Any Active State', () => {
    const CANCELLABLE_STATES: JobStatus[] = [
      'PENDING',
      'UPLOADING',
      'PROCESSING',
      'RETRY_PENDING',
      'RETRY_PROCESSING',
    ];

    CANCELLABLE_STATES.forEach((state) => {
      it(`can cancel from ${state}`, () => {
        const sm = new JobStateMachine(state);
        expect(() => sm.transition('CANCELLED')).not.toThrow();
      });
    });
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| State machine tests | `test/state-machine/job-status.test.ts` |

---

## Session 2: E2E, Security & Accessibility Testing (~2.5 hours)

### JUL-1.8-007: E2E Critical Path Tests

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.8-007 |
| **Title** | Write E2E Critical Path Tests |
| **Priority** | P0 (Critical) |
| **Effort** | 0.75 hours |
| **Dependencies** | All features implemented |

#### Description

Write comprehensive E2E tests for critical user paths using Playwright.

#### Acceptance Criteria

- [ ] Upload PDF -> Process -> View Results -> Download
- [ ] Upload DOCX -> Process -> View Results
- [ ] URL input -> Process -> View Results
- [ ] View History -> Re-download past job
- [ ] Cancel in-progress job
- [ ] Resume from checkpoint (if failed)
- [ ] All tests pass in Chromium, Firefox, WebKit

#### Technical Notes

```typescript
// test/e2e/critical-path.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Critical Path: Document Processing', () => {
  test('upload PDF, process, view results, download', async ({ page }) => {
    await page.goto('/');

    // Step 1: Upload file
    const fileInput = page.locator('input[type="file"]');
    await fileInput.setInputFiles('test/fixtures/sample.pdf');

    // Step 2: Verify preview
    await expect(page.locator('[data-testid="file-preview"]')).toContainText('sample.pdf');

    // Step 3: Start processing
    await page.click('button:has-text("Process")');

    // Step 4: Wait for progress
    await expect(page.locator('[data-testid="progress-card"]')).toBeVisible();

    // Step 5: Wait for completion (with timeout for processing)
    await expect(page.locator('[data-testid="progress-complete"]'))
      .toBeVisible({ timeout: 120000 });

    // Step 6: Verify results tabs
    await expect(page.locator('[role="tablist"]')).toBeVisible();
    await expect(page.getByRole('tab', { name: /markdown/i })).toHaveAttribute(
      'aria-selected',
      'true'
    );

    // Step 7: Verify content rendered
    const markdownContent = page.locator('[data-testid="markdown-content"]');
    await expect(markdownContent).not.toBeEmpty();

    // Step 8: Switch tabs
    await page.click('button:has-text("JSON")');
    await expect(page.locator('[data-testid="json-content"]')).toBeVisible();

    // Step 9: Download
    const [download] = await Promise.all([
      page.waitForEvent('download'),
      page.click('button:has-text("Download")'),
      page.click('[role="menuitem"]:has-text("Markdown")'),
    ]);

    expect(download.suggestedFilename()).toMatch(/sample_\d+\.md$/);
  });

  test('URL input processing', async ({ page }) => {
    await page.goto('/');

    // Enter URL
    await page.fill('input[placeholder*="URL"]', 'https://example.com/document.html');
    await page.click('button:has-text("Process")');

    // Wait for completion
    await expect(page.locator('[data-testid="progress-complete"]'))
      .toBeVisible({ timeout: 60000 });

    // Verify results
    await expect(page.locator('[data-testid="markdown-content"]')).not.toBeEmpty();
  });
});

test.describe('Critical Path: History & Re-download', () => {
  test('view history and re-download', async ({ page }) => {
    // First, create a job
    await page.goto('/');
    await page.locator('input[type="file"]').setInputFiles('test/fixtures/sample.pdf');
    await page.click('button:has-text("Process")');
    await expect(page.locator('[data-testid="progress-complete"]'))
      .toBeVisible({ timeout: 120000 });

    // Navigate to history
    await page.click('a:has-text("History")');

    // Verify job appears
    await expect(page.locator('table tbody tr')).toHaveCount(1);
    await expect(page.locator('table')).toContainText('sample.pdf');

    // Click to view details
    await page.click('table tbody tr:first-child');

    // Re-download
    const [download] = await Promise.all([
      page.waitForEvent('download'),
      page.click('[data-testid="job-detail"] button:has-text("Download")'),
    ]);

    expect(download.suggestedFilename()).toMatch(/\.md$/);
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| Critical path E2E tests | `test/e2e/critical-path.spec.ts` |

---

### JUL-1.8-008: E2E Error Handling & Edge Cases

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.8-008 |
| **Title** | Write E2E Error Handling Tests |
| **Priority** | P1 (High) |
| **Effort** | 0.5 hours |
| **Dependencies** | JUL-1.8-007 |

#### Description

Write E2E tests for error scenarios and edge cases.

#### Acceptance Criteria

- [ ] Invalid file type rejection
- [ ] Oversized file rejection
- [ ] Invalid URL rejection
- [ ] SSRF blocked URL rejection
- [ ] Network timeout handling
- [ ] MCP server unavailable handling
- [ ] Error recovery UI verified

#### Technical Notes

```typescript
// test/e2e/error-handling.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Error Handling', () => {
  test('rejects invalid file type', async ({ page }) => {
    await page.goto('/');

    const fileInput = page.locator('input[type="file"]');
    await fileInput.setInputFiles({
      name: 'script.exe',
      mimeType: 'application/octet-stream',
      buffer: Buffer.from('fake content'),
    });

    await expect(page.locator('[role="alert"]')).toContainText(/unsupported file type/i);
  });

  test('rejects oversized file', async ({ page }) => {
    await page.goto('/');

    // Note: Can't actually upload 100MB in test, mock the error
    await page.route('**/api/v1/upload', (route) => {
      route.fulfill({
        status: 400,
        body: JSON.stringify({ code: 'E001', message: 'File too large' }),
      });
    });

    const fileInput = page.locator('input[type="file"]');
    await fileInput.setInputFiles('test/fixtures/sample.pdf');
    await page.click('button:has-text("Process")');

    await expect(page.locator('[role="alert"]')).toContainText(/too large/i);
  });

  test('blocks SSRF URLs', async ({ page }) => {
    await page.goto('/');

    await page.fill('input[placeholder*="URL"]', 'http://192.168.1.1/admin');
    await page.click('button:has-text("Process")');

    await expect(page.locator('[role="alert"]')).toContainText(/blocked/i);
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| Error handling E2E tests | `test/e2e/error-handling.spec.ts` |

---

### JUL-1.8-009: E2E SSE Reconnection Test

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.8-009 |
| **Title** | Write E2E SSE Reconnection Test |
| **Priority** | P1 (High) |
| **Effort** | 0.33 hours |
| **Dependencies** | Sprint 1.5b SSE implemented |

#### Description

Write E2E test verifying SSE reconnection behavior during network interruptions.

#### Acceptance Criteria

- [ ] Test network offline -> online reconnection
- [ ] Verify "Reconnecting..." UI displayed
- [ ] Verify reconnection success
- [ ] Verify processing completes after reconnection
- [ ] Verify progress state preserved

#### Technical Notes

```typescript
// test/e2e/sse-reconnection.spec.ts
import { test, expect } from '@playwright/test';

test('SSE reconnection on network interruption', async ({ page, context }) => {
  await page.goto('/');

  // Start processing
  await page.locator('input[type="file"]').setInputFiles('test/fixtures/sample.pdf');
  await page.click('button:has-text("Process")');

  // Wait for streaming to start
  await expect(page.locator('[data-testid="connection-status"]'))
    .toContainText('Connected');

  // Record current progress
  const progressBefore = await page.locator('[role="progressbar"]')
    .getAttribute('aria-valuenow');

  // Simulate network interruption
  await context.setOffline(true);

  // Verify reconnection UI
  await expect(page.locator('[data-testid="connection-status"]'))
    .toContainText('Reconnecting', { timeout: 5000 });

  // Restore network
  await context.setOffline(false);

  // Verify reconnection success
  await expect(page.locator('[data-testid="connection-status"]'))
    .toContainText('Connected', { timeout: 30000 });

  // Verify progress continues (monotonic - should be >= before)
  const progressAfter = await page.locator('[role="progressbar"]')
    .getAttribute('aria-valuenow');
  expect(Number(progressAfter)).toBeGreaterThanOrEqual(Number(progressBefore));

  // Verify processing completes
  await expect(page.locator('[data-testid="progress-complete"]'))
    .toBeVisible({ timeout: 120000 });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| SSE reconnection E2E test | `test/e2e/sse-reconnection.spec.ts` |

---

### JUL-1.8-010: Security Test Suite

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.8-010 |
| **Title** | Write Security Test Suite |
| **Priority** | P0 (Critical - Security) |
| **Effort** | 0.75 hours |
| **Dependencies** | SSRF, input validation implemented |

#### Description

Write comprehensive security tests covering SSRF prevention, input validation, XSS prevention, and security headers.

#### Acceptance Criteria

- [ ] All SSRF patterns tested (localhost, private IPs, internal domains)
- [ ] All SSRF bypass attempts tested (encoding, decimal, octal)
- [ ] XSS prevention tested (filename, URL input)
- [ ] SQL injection prevention tested
- [ ] Path traversal prevention tested
- [ ] Security headers verified (CSP, X-Frame-Options, etc.)
- [ ] Rate limiting verified
- [ ] All security tests pass

#### Technical Notes

**Test Layer Guidance:**

- **Unit tests** (`src/**/__tests__/*.test.ts`): Call route handlers directly with `NextRequest`/`Request`, no server required
- **Integration tests** (`test/integration/*.test.ts`): Use MSW to mock external services, call handlers directly
- **E2E tests** (`test/e2e/*.spec.ts`): Use Playwright with running dev server, real `fetch()` calls OK

```typescript
// test/security/ssrf.test.ts - INTEGRATION TEST (calls route handler directly)
import { describe, it, expect } from 'vitest';
import { POST } from '@/app/api/v1/upload/route';
import { NextRequest } from 'next/server';

describe('Security: SSRF Prevention Complete', () => {
  const SSRF_PATTERNS = [
    // Standard localhost
    'http://localhost/',
    'http://127.0.0.1/',
    'http://[::1]/',
    'http://0.0.0.0/',

    // Private ranges
    'http://10.0.0.1/',
    'http://10.255.255.255/',
    'http://172.16.0.1/',
    'http://172.31.255.255/',
    'http://192.168.0.1/',
    'http://192.168.255.255/',

    // Link-local
    'http://169.254.169.254/', // AWS metadata
    'http://169.254.1.1/',

    // Internal domains
    'http://internal.hx.dev.local/',

    // Bypass attempts
    'http://127.0.0.1.nip.io/',
    'http://0x7f000001/', // Hex
    'http://2130706433/', // Decimal
    'http://017700000001/', // Octal
    'http://127.0.1/', // Shortened
    'http://127.1/',
    'http://0177.0.0.1/', // Mixed octal
    'http://[0:0:0:0:0:0:0:1]/', // IPv6 full form
    'http://[::ffff:127.0.0.1]/', // IPv4-mapped IPv6
  ];

  SSRF_PATTERNS.forEach((url) => {
    it(`blocks ${url}`, async () => {
      // Call route handler directly with NextRequest (no server needed)
      const request = new NextRequest('http://localhost/api/v1/upload', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ url }),
      });

      const response = await POST(request);

      expect(response.status).toBe(400);
      const data = await response.json();
      expect(data.code).toBe('E104');
    });
  });
});

// test/security/headers.test.ts - E2E TEST (requires running server)
// Note: Security headers are set by Next.js middleware/config, so E2E is appropriate
import { test, expect } from '@playwright/test';

test.describe('Security Headers (E2E)', () => {
  test('returns CSP header', async ({ request }) => {
    const response = await request.get('/');
    const csp = response.headers()['content-security-policy'];

    expect(csp).toContain("default-src 'self'");
    expect(csp).toContain("script-src 'self'");
    expect(csp).toContain("frame-ancestors 'none'");
  });

  test('returns X-Frame-Options', async ({ request }) => {
    const response = await request.get('/');
    expect(response.headers()['x-frame-options']).toBe('DENY');
  });

  test('returns X-Content-Type-Options', async ({ request }) => {
    const response = await request.get('/');
    expect(response.headers()['x-content-type-options']).toBe('nosniff');
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| SSRF security tests | `test/security/ssrf.test.ts` |
| Input validation security tests | `test/security/input-validation.test.ts` |
| Headers security tests | `test/security/headers.test.ts` |

---

### JUL-1.8-011: Accessibility Audit & Tests

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.8-011 |
| **Title** | Run Accessibility Audit and Write Tests |
| **Priority** | P0 (Critical) |
| **Effort** | 0.75 hours |
| **Dependencies** | All UI components implemented |

#### Description

Run comprehensive accessibility audit using axe-core and write accessibility tests for all pages.

#### Acceptance Criteria

- [ ] Main page: no WCAG 2.1 AA violations
- [ ] Results page: no WCAG 2.1 AA violations
- [ ] History page: no WCAG 2.1 AA violations
- [ ] Keyboard navigation works throughout
- [ ] Focus management verified
- [ ] Screen reader announcements verified
- [ ] Color contrast verified
- [ ] Lighthouse accessibility score >= 90

#### Technical Notes

```typescript
// test/a11y/accessibility.spec.ts
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test.describe('Accessibility Audit', () => {
  test('main page has no violations', async ({ page }) => {
    await page.goto('/');

    const results = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'])
      .analyze();

    expect(results.violations).toEqual([]);
  });

  test('results page has no violations', async ({ page }) => {
    // Navigate to results page with data
    await page.goto('/');
    await page.locator('input[type="file"]').setInputFiles('test/fixtures/sample.pdf');
    await page.click('button:has-text("Process")');
    await expect(page.locator('[data-testid="progress-complete"]'))
      .toBeVisible({ timeout: 120000 });

    const results = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa'])
      .analyze();

    expect(results.violations).toEqual([]);
  });

  test('history page has no violations', async ({ page }) => {
    await page.goto('/history');

    const results = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa'])
      .analyze();

    expect(results.violations).toEqual([]);
  });

  test('keyboard navigation works', async ({ page }) => {
    await page.goto('/');

    // Tab through all interactive elements
    const tabbableElements = [];
    let previousElement: string | null = null;

    for (let i = 0; i < 20; i++) {
      await page.keyboard.press('Tab');
      const focused = await page.evaluate(() => {
        const el = document.activeElement;
        return el ? `${el.tagName}#${el.id || 'no-id'}` : null;
      });

      if (focused === previousElement) break;
      if (focused) tabbableElements.push(focused);
      previousElement = focused;
    }

    // Verify key elements are tabbable
    expect(tabbableElements.length).toBeGreaterThan(5);
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| Accessibility tests | `test/a11y/accessibility.spec.ts` |
| Accessibility report | `reports/accessibility-audit.md` |

---

### JUL-1.8-012: Generate and Review Coverage Report

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.8-012 |
| **Title** | Generate Coverage Report and Identify Gaps |
| **Priority** | P1 (High) |
| **Effort** | 0.33 hours |
| **Dependencies** | JUL-1.8-001 through JUL-1.8-006 |

#### Description

Generate comprehensive coverage report, analyze gaps, and create action items for any coverage deficiencies.

#### Acceptance Criteria

- [ ] Coverage report generated with `npm run test:coverage`
- [ ] Line coverage >= 80%
- [ ] Branch coverage >= 75%
- [ ] Critical paths have >= 95% coverage
- [ ] Coverage gaps documented
- [ ] Action items created for any gaps

#### Deliverables

| Deliverable | Path |
|-------------|------|
| Coverage report | `coverage/lcov-report/index.html` |
| Gap analysis | `reports/coverage-analysis.md` |

---

### JUL-1.8-013: Create Test Fixtures

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.8-013 |
| **Title** | Create Test Fixtures |
| **Priority** | P2 (Medium) |
| **Effort** | 0.25 hours |
| **Dependencies** | E2E tests |

#### Description

Create and organize test fixtures for all test types including sample documents.

#### Acceptance Criteria

- [ ] sample.pdf - Valid PDF document
- [ ] sample.docx - Valid Word document
- [ ] sample.xlsx - Valid Excel document
- [ ] sample.pptx - Valid PowerPoint document
- [ ] sample.png - Valid image
- [ ] corrupted.pdf - Corrupted PDF for error testing
- [ ] All fixtures documented in README

#### Deliverables

| Deliverable | Path |
|-------------|------|
| Test fixtures | `test/fixtures/*` |
| Fixtures README | `test/fixtures/README.md` |

---

### JUL-1.8-014: Final Quality Gate Verification

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.8-014 |
| **Title** | Final Quality Gate Verification |
| **Priority** | P0 (Critical) |
| **Effort** | 0.33 hours |
| **Dependencies** | All testing tasks complete |

#### Description

Run all quality gates and verify Phase 1 completion criteria are met.

#### Acceptance Criteria

- [ ] `npm run lint` - 0 errors
- [ ] `npm run typecheck` - 0 errors
- [ ] `npm run test` - All tests pass
- [ ] `npm run test:coverage` - >= 80% lines, >= 75% branches
- [ ] `npm run test:e2e` - All E2E tests pass
- [ ] `npm run build` - Build succeeds
- [ ] Lighthouse Performance >= 80
- [ ] Lighthouse Accessibility >= 90
- [ ] All security tests pass
- [ ] Phase 1 sign-off checklist complete

#### Technical Notes

```bash
# Quality gate verification script
#!/bin/bash
set -e

echo "=== Quality Gate Verification ==="

echo "1. TypeScript Check..."
npm run typecheck

echo "2. ESLint Check..."
npm run lint

echo "3. Unit Tests..."
npm run test

echo "4. Coverage Check..."
npm run test:coverage

echo "5. E2E Tests..."
npm run test:e2e

echo "6. Build Check..."
npm run build

echo "7. Lighthouse Audit..."
npx lighthouse http://localhost:3000 --output=json --output-path=./reports/lighthouse.json

echo "=== All Quality Gates Passed ==="
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| Quality gate report | `reports/quality-gates.md` |
| Lighthouse report | `reports/lighthouse.json` |

---

## Sprint 1.8 Testing Summary

| Metric | Target | Notes |
|--------|--------|-------|
| Total Tasks | 14 | |
| Task Effort Total | 10.0 hours | Sum of all task estimates |
| Planned Duration | 6.0 hours | Allocated sprint sessions (9-11) |
| Session 1 Tasks | 6 | Unit/Integration tests (~4.75h effort) |
| Session 2 Tasks | 8 | E2E, Security, A11y, QA (~5.25h effort) |

### Coverage Targets (Phase 1 Complete)

| Category | Target | Verification |
|----------|--------|--------------|
| Line Coverage | >= 80% | `npm run test:coverage` |
| Branch Coverage | >= 75% | `npm run test:coverage` |
| Critical Paths | >= 95% | Manual review |
| Validation Libraries | >= 95% | Coverage report |
| State Machine | 100% | Coverage report |

### Quality Gate Summary

| Gate | Threshold | Tool |
|------|-----------|------|
| TypeScript Errors | 0 | `tsc --noEmit` |
| ESLint Errors | 0 | `eslint` |
| Unit Test Pass | 100% | Vitest |
| E2E Test Pass | 100% | Playwright |
| Accessibility Score | >= 90 | Lighthouse |
| Security Tests | 100% pass | Custom suite |

### Dependencies on Other Agents

| Agent | Support Required | Sprint Tasks |
|-------|------------------|--------------|
| Neo | Code fixes for failing tests | As needed |
| Trinity | Database test fixtures | JUL-1.8-005 |
| William | Infrastructure for E2E tests | JUL-1.8-007 |

---

## Phase 1 Completion Checklist

```markdown
## Phase 1 Testing Sign-Off

### Code Quality
- [ ] `npm run typecheck` - 0 TypeScript errors
- [ ] `npm run lint` - 0 ESLint errors
- [ ] `npm run build` - Build succeeds

### Testing
- [ ] Line coverage >= 80%
- [ ] Branch coverage >= 75%
- [ ] All unit tests pass
- [ ] All integration tests pass
- [ ] All E2E tests pass
- [ ] All security tests pass
- [ ] All accessibility tests pass

### Performance
- [ ] Lighthouse Performance >= 80
- [ ] Lighthouse Accessibility >= 90
- [ ] LCP < 2.5 seconds

### Security
- [ ] SSRF prevention verified
- [ ] Input validation verified
- [ ] Security headers verified
- [ ] No high/critical vulnerabilities

### Documentation
- [ ] Test documentation complete
- [ ] Coverage report generated
- [ ] Quality gate report generated

### Sign-Off
- [ ] Julia Santos (@julia) - Testing Lead
- [ ] Alex Rivera (@alex) - Technical Review
- [ ] Agent Zero (@agent-zero) - Final Approval
```

---

## Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0.0 | 2025-12-12 | Julia Santos | Initial creation |
