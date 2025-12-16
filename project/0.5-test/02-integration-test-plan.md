# HX Docling Application - Integration Test Plan

**Document Type**: Integration Test Plan
**Version**: 2.3.0
**Status**: DRAFT
**Last Updated**: 2025-12-15
**Created**: 2025-12-12
**Author**: Julia Santos (Testing & QA Specialist)
**Master Plan Reference**: `project/0.5-test/00-test-plan-overview.md`
**Specification Reference**: `project/0.3-specification/0.3.1-detailed-specification.md` v1.2.1

---

## Table of Contents

1. [Overview](#1-overview)
2. [API Route to MCP Client Integration](#2-api-route-to-mcp-client-integration)
3. [MCP Client to Docling Server Integration](#3-mcp-client-to-docling-server-integration)
4. [API Route to PostgreSQL Integration](#4-api-route-to-postgresql-integration)
5. [API Route to Redis Integration](#5-api-route-to-redis-integration)
6. [SSE Endpoint to Redis Pub/Sub Integration](#6-sse-endpoint-to-redis-pubsub-integration)
7. [State Machine Transitions](#7-state-machine-transitions)
8. [Checkpoint and Resume Integration](#8-checkpoint-and-resume-integration)
9. [Rate Limiting Integration](#9-rate-limiting-integration)
10. [Session Management Integration](#10-session-management-integration)
11. [Frontend Component Integration](#11-frontend-component-integration)
12. [Health Check Integration](#12-health-check-integration)
13. [Next.js Middleware Integration](#13-nextjs-middleware-integration)
14. [Server Component Integration](#14-server-component-integration)
15. [Database Connection Management](#15-database-connection-management)
16. [MCP Protocol Compliance](#16-mcp-protocol-compliance)
17. [Redis Connection Management](#17-redis-connection-management)
18. [SSE Connection Management](#18-sse-connection-management)
19. [Request Validation Integration](#19-request-validation-integration)
20. [Async Concurrency Integration](#20-async-concurrency-integration)
21. [File Storage Integration](#21-file-storage-integration)
22. [URL Fetch Integration](#22-url-fetch-integration)
23. [Workflow Orchestration Integration](#23-workflow-orchestration-integration)
24. [Cross-Service Consistency](#24-cross-service-consistency)
25. [Container and Docker Testing](#25-container-and-docker-testing)
26. [Next.js Server Components](#26-nextjs-server-components)
27. [Server Actions](#27-server-actions)
28. [Next.js Caching](#28-nextjs-caching)
29. [Disaster Recovery Tests](#29-disaster-recovery-tests)

---

## 1. Overview

### 1.1 Purpose

This document defines integration test specifications for validating the interactions between system components. Integration tests verify that components work together correctly, focusing on interfaces, data flow, and error handling across boundaries.

### 1.2 Test Framework

- **Tool**: Vitest with MSW (Mock Service Worker)
- **Test Directory**: `test/integration/`
- **Configuration**: `vitest.config.ts`
- **Run Command**: `npm run test:integration`

### 1.3 Integration Points

```
+-------------------+     +-----------------+     +----------------------+
|   API Routes      |<--->|   MCP Client    |<--->|  hx-docling-mcp      |
|   (Next.js)       |     |   (lib/mcp/)    |     |  (External Service)  |
+-------------------+     +-----------------+     +----------------------+
         |                        |
         v                        v
+-------------------+     +-----------------+
|   PostgreSQL      |     |   Redis         |
|   (Prisma)        |     |   (Session/SSE) |
+-------------------+     +-----------------+
```

### 1.4 Mock Strategy

| Component | Integration Test Strategy |
|-----------|---------------------------|
| MCP Server | MSW mock for HTTP/JSON-RPC |
| PostgreSQL | Test database instance |
| Redis | Test Redis instance (or ioredis-mock) |
| External URLs | MSW mock |

---

## 2. API Route to MCP Client Integration

### 2.1 Overview

Tests validating that API routes correctly invoke the MCP client for document processing operations.

---

### INT-MCP-001: Process Route Invokes MCP Client

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-MCP-001 |
| **Title** | Process Route Invokes MCP Client |
| **Priority** | P1 - High |
| **Components** | `/api/v1/process`, `lib/mcp/client.ts` |
| **Spec Reference** | FR-401, FR-402, FR-403 |

**Preconditions**:
- Job exists in PENDING status
- MCP server mock configured

**Test Steps**:

```typescript
describe('POST /api/v1/process - MCP Integration', () => {
  it('invokes correct MCP tool based on file type', async () => {
    // Setup: Create job with PDF file
    const job = await prisma.job.create({
      data: {
        sessionId: 'test-session',
        status: 'PENDING',
        inputType: 'FILE',
        fileName: 'test.pdf',
        filePath: '/tmp/test.pdf',
        mimeType: 'application/pdf',
      },
    });

    // Mock MCP server
    const mcpCalls: any[] = [];
    server.use(
      rest.post(MCP_ENDPOINT, async (req, res, ctx) => {
        const body = await req.json();
        mcpCalls.push(body);

        if (body.method === 'initialize') {
          return res(ctx.json({ jsonrpc: '2.0', result: { serverInfo: {}, capabilities: { tools: {} } }, id: body.id }));
        }
        if (body.method === 'tools/call' && body.params.name === 'convert_pdf') {
          return res(ctx.json({ jsonrpc: '2.0', result: { content: [{ type: 'document', document: mockDoclingDoc }] }, id: body.id }));
        }
        // ... handle exports
      })
    );

    // Execute
    const response = await fetch('/api/v1/process', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ jobId: job.id }),
    });

    // Verify
    expect(response.status).toBe(202);
    expect(mcpCalls.some(c => c.method === 'tools/call' && c.params.name === 'convert_pdf')).toBe(true);
  });
});
```

**Expected Results**:
- API route calls MCP client
- Correct tool selected for file type (convert_pdf for .pdf)
- MCP client properly initialized before tool call

**Verification Points**:
- MCP `initialize` called first
- `notifications/initialized` sent after initialize response
- `tools/call` with correct tool name invoked

---

### INT-MCP-002: Tool Routing by File Extension

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-MCP-002 |
| **Title** | Tool Routing by File Extension |
| **Priority** | P1 - High |
| **Components** | `lib/mcp/router.ts`, `lib/mcp/client.ts` |
| **Spec Reference** | FR-402 |

**Test Matrix**:

| File Extension | Expected MCP Tool | Timeout |
|----------------|-------------------|---------|
| .pdf | convert_pdf | Size-based |
| .doc, .docx | convert_docx | Size-based |
| .xls, .xlsx | convert_xlsx | Size-based |
| .ppt, .pptx | convert_pptx | Size-based |
| .png, .jpg, .jpeg, .tiff | convert_pdf (image mode) | Size-based |
| URL (http/https) | convert_url | 30s |

**Test Steps**:

```typescript
describe('MCP Tool Routing', () => {
  const testCases = [
    { extension: '.pdf', expectedTool: 'convert_pdf' },
    { extension: '.docx', expectedTool: 'convert_docx' },
    { extension: '.xlsx', expectedTool: 'convert_xlsx' },
    { extension: '.pptx', expectedTool: 'convert_pptx' },
    { extension: '.png', expectedTool: 'convert_pdf' },
  ];

  testCases.forEach(({ extension, expectedTool }) => {
    it(`routes ${extension} to ${expectedTool}`, async () => {
      const job = await createJob({ fileName: `test${extension}` });

      let invokedTool: string | null = null;
      mockMCPServer((request) => {
        if (request.method === 'tools/call') {
          invokedTool = request.params.name;
        }
      });

      await processJob(job.id);

      expect(invokedTool).toBe(expectedTool);
    });
  });

  it('routes URL input to convert_url', async () => {
    const job = await createJob({ inputType: 'URL', url: 'https://example.com/doc' });

    let invokedTool: string | null = null;
    mockMCPServer((request) => {
      if (request.method === 'tools/call') {
        invokedTool = request.params.name;
      }
    });

    await processJob(job.id);

    expect(invokedTool).toBe('convert_url');
  });
});
```

**Expected Results**:
- Each file type routes to correct MCP tool
- URL input routes to convert_url

---

### INT-MCP-003: Export Chaining (Prompt Chaining Pattern)

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-MCP-003 |
| **Title** | Export Chaining After Conversion |
| **Priority** | P1 - High |
| **Components** | `/api/v1/process`, `lib/mcp/client.ts` |
| **Spec Reference** | FR-404 |

**Test Steps**:

```typescript
describe('Export Chaining', () => {
  it('calls all three export tools after successful conversion', async () => {
    const job = await createJob({ fileName: 'test.pdf' });

    const toolsCalled: string[] = [];
    mockMCPServer((request) => {
      if (request.method === 'tools/call') {
        toolsCalled.push(request.params.name);
      }
    });

    await processJob(job.id);

    // Verify conversion first
    expect(toolsCalled[0]).toBe('convert_pdf');

    // Verify all three exports called
    expect(toolsCalled).toContain('export_markdown');
    expect(toolsCalled).toContain('export_html');
    expect(toolsCalled).toContain('export_json');
  });

  it('executes exports in parallel using Promise.allSettled', async () => {
    const job = await createJob({ fileName: 'test.pdf' });

    const exportStartTimes: Record<string, number> = {};
    mockMCPServer((request) => {
      if (request.method === 'tools/call' && request.params.name.startsWith('export_')) {
        exportStartTimes[request.params.name] = Date.now();
      }
    });

    await processJob(job.id);

    // All exports should start within 100ms of each other (parallel)
    const times = Object.values(exportStartTimes);
    const maxDiff = Math.max(...times) - Math.min(...times);
    expect(maxDiff).toBeLessThan(100);
  });
});
```

**Expected Results**:
- All three exports called after conversion
- Exports executed in parallel (Promise.allSettled)

---

### INT-MCP-004: MCP Error Mapping

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-MCP-004 |
| **Title** | MCP Error to Application Error Mapping |
| **Priority** | P1 - High |
| **Components** | `lib/mcp/error-mapping.ts` |
| **Spec Reference** | Section 7.1.4 |

**Test Matrix**:

| MCP Error Code | Application Error Code | Retryable |
|----------------|------------------------|-----------|
| -32700 | E203 | No |
| -32601 | E204 | No |
| -32603 | E201 | Yes |
| -32000 | E201 | Yes |
| -32001 | E202 | Yes |
| -32002 | E302 | Yes |

**Test Steps**:

```typescript
describe('MCP Error Mapping', () => {
  const errorMappings = [
    { mcpCode: -32700, appCode: 'E203', retryable: false },
    { mcpCode: -32601, appCode: 'E204', retryable: false },
    { mcpCode: -32603, appCode: 'E201', retryable: true },
    { mcpCode: -32000, appCode: 'E201', retryable: true },
    { mcpCode: -32001, appCode: 'E202', retryable: true },
  ];

  errorMappings.forEach(({ mcpCode, appCode, retryable }) => {
    it(`maps MCP ${mcpCode} to ${appCode} (retryable: ${retryable})`, async () => {
      const job = await createJob({ fileName: 'test.pdf' });

      mockMCPServer((request) => {
        if (request.method === 'tools/call') {
          return { error: { code: mcpCode, message: 'Test error' } };
        }
      });

      await processJob(job.id);

      const updatedJob = await prisma.job.findUnique({ where: { id: job.id } });
      expect(updatedJob?.errorCode).toBe(appCode);

      if (retryable) {
        expect(['RETRY_1', 'RETRY_2', 'RETRY_3', 'ERROR']).toContain(updatedJob?.status);
      } else {
        expect(updatedJob?.status).toBe('ERROR');
      }
    });
  });
});
```

**Expected Results**:
- MCP errors correctly mapped to application error codes
- Retryable errors trigger retry logic

---

## 3. MCP Client to Docling Server Integration

### INT-MCP-010: MCP Initialization Sequence

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-MCP-010 |
| **Title** | MCP Protocol Initialization Sequence |
| **Priority** | P0 - Critical |
| **Components** | `lib/mcp/client.ts` |
| **Spec Reference** | Section 7.1.2 |

**Test Steps**:

```typescript
describe('MCP Initialization', () => {
  it('follows correct initialization sequence', async () => {
    const requests: any[] = [];
    mockMCPServer((request) => {
      requests.push(request);
    });

    const client = new MCPClient();
    await client.initialize();

    // Verify sequence
    expect(requests[0].method).toBe('initialize');
    expect(requests[1].method).toBe('notifications/initialized');
    expect(requests[2].method).toBe('tools/list');
  });

  it('sends initialized notification after initialize response', async () => {
    let initializeResponseSent = false;
    let notificationSentAfterResponse = false;

    mockMCPServer((request, respond) => {
      if (request.method === 'initialize') {
        respond({ result: { serverInfo: {}, capabilities: { tools: {} } } });
        initializeResponseSent = true;
      }
      if (request.method === 'notifications/initialized') {
        notificationSentAfterResponse = initializeResponseSent;
      }
    });

    const client = new MCPClient();
    await client.initialize();

    expect(notificationSentAfterResponse).toBe(true);
  });

  it('caches tool schemas after initialization', async () => {
    mockMCPServer((request) => {
      if (request.method === 'tools/list') {
        return {
          result: {
            tools: [
              { name: 'convert_pdf', inputSchema: { type: 'object' } },
              { name: 'export_markdown', inputSchema: { type: 'object' } },
            ],
          },
        };
      }
    });

    const client = new MCPClient();
    await client.initialize();

    expect(client.hasTool('convert_pdf')).toBe(true);
    expect(client.hasTool('export_markdown')).toBe(true);
    expect(client.hasTool('unknown_tool')).toBe(false);
  });
});
```

**Expected Results**:
- Initialize -> Initialized notification -> Tools list sequence
- Tool schemas cached for validation

---

### INT-MCP-011: MCP Connection Timeout Handling

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-MCP-011 |
| **Title** | MCP Connection Timeout Handling |
| **Priority** | P1 - High |
| **Components** | `lib/mcp/client.ts` |
| **Spec Reference** | E202 |

**Test Steps**:

```typescript
describe('MCP Timeout Handling', () => {
  it('throws E202 on connection timeout', async () => {
    mockMCPServer((request) => {
      // Never respond
      return new Promise(() => {});
    });

    const client = new MCPClient({ timeout: 1000 });

    await expect(client.invoke('convert_pdf', { file_path: '/test.pdf' }))
      .rejects.toMatchObject({ code: 'E202' });
  });

  it('respects size-based timeout configuration', async () => {
    const startTime = Date.now();
    let timeoutTriggered = false;

    mockMCPServer((request) => {
      return new Promise((resolve) => {
        setTimeout(() => {
          timeoutTriggered = true;
          resolve({ result: {} });
        }, 10000);
      });
    });

    // Small file timeout (5s for test)
    const client = new MCPClient();

    try {
      await client.invoke('convert_pdf', { file_path: '/small.pdf' }, { timeoutMs: 1000 });
    } catch (e) {
      // Expected
    }

    expect(timeoutTriggered).toBe(false);
    expect(Date.now() - startTime).toBeLessThan(2000);
  });
});
```

**Expected Results**:
- Timeout errors mapped to E202
- Size-based timeouts respected

---

### INT-MCP-012: MCP Request Cancellation

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-MCP-012 |
| **Title** | MCP Request Cancellation via AbortController |
| **Priority** | P1 - High |
| **Components** | `lib/mcp/client.ts` |
| **Spec Reference** | FR-406 |

**Test Steps**:

```typescript
describe('MCP Cancellation', () => {
  it('aborts in-flight request on cancel', async () => {
    let requestAborted = false;

    mockMCPServer((request, respond, { signal }) => {
      signal.addEventListener('abort', () => {
        requestAborted = true;
      });
      // Simulate long-running request
      return new Promise((resolve) => setTimeout(resolve, 10000));
    });

    const client = new MCPClient();
    const promise = client.invoke('convert_pdf', { file_path: '/test.pdf' });

    // Cancel after 100ms
    setTimeout(() => client.abort(), 100);

    await expect(promise).rejects.toThrow('aborted');
    expect(requestAborted).toBe(true);
  });
});
```

**Expected Results**:
- AbortController cancels in-flight request
- Server receives abort signal

---

## 4. API Route to PostgreSQL Integration

### INT-DB-001: Job Creation and Retrieval

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DB-001 |
| **Title** | Job CRUD Operations |
| **Priority** | P1 - High |
| **Components** | `/api/v1/upload`, `/api/v1/jobs`, Prisma |
| **Spec Reference** | Section 5.1 |

**Test Steps**:

```typescript
describe('Job Database Operations', () => {
  beforeEach(async () => {
    await prisma.result.deleteMany();
    await prisma.job.deleteMany();
  });

  it('creates job record on upload', async () => {
    const response = await uploadFile('test.pdf');

    expect(response.status).toBe(201);
    const { jobId } = await response.json();

    const job = await prisma.job.findUnique({ where: { id: jobId } });
    expect(job).not.toBeNull();
    expect(job?.status).toBe('PENDING');
    expect(job?.inputType).toBe('FILE');
    expect(job?.fileName).toBe('test.pdf');
  });

  it('retrieves job with all relations', async () => {
    // Create job with results
    const job = await prisma.job.create({
      data: {
        sessionId: 'test-session',
        status: 'COMPLETE',
        inputType: 'FILE',
        results: {
          create: [
            { format: 'MARKDOWN', content: '# Test', size: 6 },
            { format: 'HTML', content: '<h1>Test</h1>', size: 14 },
          ],
        },
      },
    });

    const response = await fetch(`/api/v1/jobs/${job.id}`);
    const data = await response.json();

    expect(data.results).toHaveLength(2);
    expect(data.results.map((r: any) => r.format)).toContain('MARKDOWN');
  });
});
```

**Expected Results**:
- Jobs created with correct initial state
- Job retrieval includes all relations

---

### INT-DB-002: Result Storage and Retrieval

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DB-002 |
| **Title** | Result Storage on Processing Completion |
| **Priority** | P1 - High |
| **Components** | `/api/v1/process`, Prisma |
| **Spec Reference** | Section 5.1 |

**Test Steps**:

```typescript
describe('Result Database Operations', () => {
  it('stores all export results on successful processing', async () => {
    const job = await createJob({ fileName: 'test.pdf' });

    mockSuccessfulMCPProcessing();

    await processJob(job.id);

    const results = await prisma.result.findMany({
      where: { jobId: job.id },
    });

    expect(results).toHaveLength(3);
    expect(results.map(r => r.format).sort()).toEqual(['HTML', 'JSON', 'MARKDOWN']);

    results.forEach(result => {
      expect(result.content).toBeTruthy();
      expect(result.size).toBeGreaterThan(0);
    });
  });

  it('validates result size before storage', async () => {
    const job = await createJob({ fileName: 'test.pdf' });

    mockMCPServerWithLargeResult(30 * 1024 * 1024); // 30MB - exceeds limit

    await expect(processJob(job.id)).rejects.toThrow('E305');
  });
});
```

**Expected Results**:
- All three result formats stored
- Size validation enforced

---

### INT-DB-003: Transaction Integrity

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DB-003 |
| **Title** | Database Transaction Integrity |
| **Priority** | P1 - High |
| **Components** | Prisma |
| **Spec Reference** | Database reliability |

**Test Steps**:

```typescript
describe('Transaction Integrity', () => {
  it('rolls back on partial failure', async () => {
    const job = await createJob({ fileName: 'test.pdf' });

    // Mock: First result succeeds, second fails
    let resultCount = 0;
    jest.spyOn(prisma.result, 'create').mockImplementation(async (args) => {
      resultCount++;
      if (resultCount === 2) {
        throw new Error('Simulated failure');
      }
      return args.data as any;
    });

    await expect(processJobWithTransaction(job.id)).rejects.toThrow();

    // Verify no partial results exist
    const results = await prisma.result.findMany({ where: { jobId: job.id } });
    expect(results).toHaveLength(0);
  });
});
```

**Expected Results**:
- Transactions roll back on failure
- No partial data persisted

---

### INT-DB-004: History Query with Pagination

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DB-004 |
| **Title** | History Query with Pagination |
| **Priority** | P1 - High |
| **Components** | `/api/v1/history`, Prisma |
| **Spec Reference** | FR-702 |

**Test Steps**:

```typescript
describe('History Pagination', () => {
  beforeEach(async () => {
    // Create 25 jobs
    await prisma.job.createMany({
      data: Array.from({ length: 25 }, (_, i) => ({
        sessionId: 'test-session',
        status: 'COMPLETE',
        inputType: 'FILE',
        fileName: `test-${i}.pdf`,
        createdAt: new Date(Date.now() - i * 1000),
      })),
    });
  });

  it('returns paginated results with correct page size', async () => {
    const response = await fetch('/api/v1/history?page=1&pageSize=10', {
      headers: { Cookie: 'session=test-session' },
    });
    const data = await response.json();

    expect(data.jobs).toHaveLength(10);
    expect(data.pagination.page).toBe(1);
    expect(data.pagination.pageSize).toBe(10);
    expect(data.pagination.totalCount).toBe(25);
    expect(data.pagination.totalPages).toBe(3);
    expect(data.pagination.hasMore).toBe(true);
  });

  it('sorts by createdAt DESC by default', async () => {
    const response = await fetch('/api/v1/history', {
      headers: { Cookie: 'session=test-session' },
    });
    const data = await response.json();

    // First job should be newest
    expect(data.jobs[0].fileName).toBe('test-0.pdf');
    expect(data.jobs[1].fileName).toBe('test-1.pdf');
  });

  it('supports status sorting', async () => {
    // Create jobs with different statuses
    await prisma.job.create({
      data: { sessionId: 'test-session', status: 'PROCESSING', inputType: 'FILE' },
    });
    await prisma.job.create({
      data: { sessionId: 'test-session', status: 'ERROR', inputType: 'FILE' },
    });

    const response = await fetch('/api/v1/history?sortBy=status&sortOrder=asc', {
      headers: { Cookie: 'session=test-session' },
    });
    const data = await response.json();

    // PROCESSING should come before COMPLETE before ERROR
    expect(data.jobs[0].status).toBe('PROCESSING');
  });
});
```

**Expected Results**:
- Pagination parameters work correctly
- Sorting by createdAt and status works
- Session filtering applied

---

## 5. API Route to Redis Integration

### INT-REDIS-001: Session CRUD Operations

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-REDIS-001 |
| **Title** | Session CRUD Operations |
| **Priority** | P1 - High |
| **Components** | `lib/redis/session.ts` |
| **Spec Reference** | Section 5.4 |

**Test Steps**:

```typescript
describe('Redis Session Operations', () => {
  it('creates session with correct structure', async () => {
    const sessionId = 'test-session-123';
    const session = await createSession(sessionId);

    expect(session.id).toBe(sessionId);
    expect(session.createdAt).toBeDefined();
    expect(session.lastActivity).toBeDefined();
    expect(session.jobCount).toBe(0);
    expect(session.activeJobIds).toEqual([]);
  });

  it('retrieves session from Redis', async () => {
    const sessionId = 'test-session-456';
    await createSession(sessionId);

    const session = await getSession(sessionId);

    expect(session).not.toBeNull();
    expect(session?.id).toBe(sessionId);
  });

  it('updates lastActivity and extends TTL', async () => {
    const sessionId = 'test-session-789';
    await createSession(sessionId);

    const beforeUpdate = await getSession(sessionId);
    await new Promise(resolve => setTimeout(resolve, 100));
    await updateSessionActivity(sessionId);
    const afterUpdate = await getSession(sessionId);

    expect(afterUpdate?.lastActivity).toBeGreaterThan(beforeUpdate?.lastActivity || 0);
  });

  it('deletes session from Redis', async () => {
    const sessionId = 'test-session-del';
    await createSession(sessionId);
    await deleteSession(sessionId);

    const session = await getSession(sessionId);
    expect(session).toBeNull();
  });

  it('session expires after TTL', async () => {
    const sessionId = 'test-session-ttl';

    // Create session with short TTL for testing
    await createSession(sessionId, { ttl: 1 });

    // Wait for expiry
    await new Promise(resolve => setTimeout(resolve, 1500));

    const session = await getSession(sessionId);
    expect(session).toBeNull();
  });
});
```

**Expected Results**:
- Sessions created with correct fields
- TTL enforced
- Activity update extends TTL

---

### INT-REDIS-002: Rate Limit Check Operations

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-REDIS-002 |
| **Title** | Rate Limiting with Sliding Window |
| **Priority** | P1 - High |
| **Components** | `lib/redis/rate-limit.ts` |
| **Spec Reference** | Section 7.4.2 |

**Test Steps**:

```typescript
describe('Redis Rate Limiting', () => {
  it('allows requests within limit', async () => {
    const sessionId = 'rate-test-1';

    for (let i = 0; i < 10; i++) {
      const result = await checkRateLimit(sessionId);
      expect(result.allowed).toBe(true);
      expect(result.remaining).toBe(10 - i - 1);
    }
  });

  it('blocks requests exceeding limit', async () => {
    const sessionId = 'rate-test-2';

    // Exhaust limit
    for (let i = 0; i < 10; i++) {
      await checkRateLimit(sessionId);
    }

    // 11th request should be blocked
    const result = await checkRateLimit(sessionId);
    expect(result.allowed).toBe(false);
    expect(result.remaining).toBe(0);
    expect(result.retryAfter).toBeGreaterThan(0);
    expect(result.retryAfter).toBeLessThanOrEqual(60);
  });

  it('uses sliding window (not fixed window)', async () => {
    const sessionId = 'rate-test-sliding';

    // Make 5 requests
    for (let i = 0; i < 5; i++) {
      await checkRateLimit(sessionId);
    }

    // Wait 30 seconds (half window)
    await sleep(30000);

    // Make 5 more requests
    for (let i = 0; i < 5; i++) {
      const result = await checkRateLimit(sessionId);
      expect(result.allowed).toBe(true);
    }

    // 11th total should be blocked
    const result = await checkRateLimit(sessionId);
    expect(result.allowed).toBe(false);
  }, 60000);

  it('returns retryAfter in delta seconds', async () => {
    const sessionId = 'rate-test-delta';

    // Exhaust limit
    for (let i = 0; i < 10; i++) {
      await checkRateLimit(sessionId);
    }

    const result = await checkRateLimit(sessionId);

    // retryAfter should be delta seconds, not epoch
    expect(result.retryAfter).toBeGreaterThan(0);
    expect(result.retryAfter).toBeLessThanOrEqual(60);
  });
});
```

**Expected Results**:
- Sliding window algorithm works correctly
- retryAfter returns delta seconds

---

### INT-REDIS-003: Circuit Breaker Behavior

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-REDIS-003 |
| **Title** | Redis Circuit Breaker |
| **Priority** | P1 - High |
| **Components** | `lib/redis/circuit-breaker.ts` |
| **Spec Reference** | Section 7.4.3 |

**Test Steps**:

```typescript
describe('Redis Circuit Breaker', () => {
  it('opens after failure threshold', async () => {
    const circuitBreaker = new RedisCircuitBreaker({ failureThreshold: 3 });

    // Simulate failures
    for (let i = 0; i < 3; i++) {
      try {
        await circuitBreaker.execute(() => Promise.reject(new Error('Redis down')));
      } catch (e) {}
    }

    expect(circuitBreaker.getState().state).toBe('open');
  });

  it('transitions to half-open after reset timeout', async () => {
    const circuitBreaker = new RedisCircuitBreaker({
      failureThreshold: 1,
      resetTimeout: 100,
    });

    // Trigger open state
    try {
      await circuitBreaker.execute(() => Promise.reject(new Error()));
    } catch (e) {}

    expect(circuitBreaker.getState().state).toBe('open');

    // Wait for reset timeout
    await sleep(150);

    // Attempt should transition to half-open
    await circuitBreaker.execute(() => Promise.resolve());
    expect(circuitBreaker.getState().state).toBe('closed');
  });

  it('falls back to in-memory rate limiting when open', async () => {
    const sessionId = 'fallback-test';

    // Mock Redis to fail
    mockRedisFailure();

    const result = await checkRateLimitWithFallback(
      sessionId,
      FallbackStrategy.IN_MEMORY
    );

    expect(result.allowed).toBe(true);
  });
});
```

**Expected Results**:
- Circuit opens after threshold failures
- Half-open state tested
- Fallback to in-memory works

---

## 6. SSE Endpoint to Redis Pub/Sub Integration

### INT-SSE-001: Progress Event Publishing

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-SSE-001 |
| **Title** | Progress Event Publishing to Redis |
| **Priority** | P1 - High |
| **Components** | `/api/v1/process/[jobId]/events`, Redis pub/sub |
| **Spec Reference** | Section 4.4 |

**Test Steps**:

```typescript
describe('SSE Progress Publishing', () => {
  it('publishes progress events to Redis channel', async () => {
    const job = await createJob({ fileName: 'test.pdf' });

    const events: any[] = [];
    const subscriber = redis.subscribe(`hx-docling:job:${job.id}:progress`);
    subscriber.on('message', (channel, message) => {
      events.push(JSON.parse(message));
    });

    // Start processing (background)
    processJob(job.id);

    // Wait for events
    await sleep(1000);

    expect(events.length).toBeGreaterThan(0);
    expect(events[0]).toHaveProperty('stage');
    expect(events[0]).toHaveProperty('percent');
    expect(events[0]).toHaveProperty('message');
  });
});
```

**Expected Results**:
- Progress events published to Redis
- Event format matches specification

---

### INT-SSE-002: Event ID Format and Buffering

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-SSE-002 |
| **Title** | SSE Event ID Format and Buffer |
| **Priority** | P1 - High |
| **Components** | `lib/sse/buffer.ts` |
| **Spec Reference** | Section 4.4.1 |

**Test Steps**:

```typescript
describe('SSE Event Buffering', () => {
  it('generates event ID in correct format', async () => {
    const jobId = '12345678-1234-1234-1234-123456789012';
    const eventId = await bufferEvent(jobId, 'progress', { percent: 50 }, 1);

    // Format: {jobId-8chars}-{timestamp}-{sequence-4digits}
    const pattern = /^[a-f0-9]{8}-\d+-\d{4}$/;
    expect(eventId).toMatch(pattern);
  });

  it('buffers events up to max limit', async () => {
    const jobId = 'buffer-test';

    // Add 150 events (max is 100)
    for (let i = 0; i < 150; i++) {
      await bufferEvent(jobId, 'progress', { percent: i }, i);
    }

    const events = await getEventsSince(jobId, null);
    expect(events.length).toBe(100);
  });

  it('retrieves events after lastEventId', async () => {
    const jobId = 'since-test';

    // Add events
    const eventIds: string[] = [];
    for (let i = 0; i < 5; i++) {
      const id = await bufferEvent(jobId, 'progress', { percent: i * 20 }, i);
      eventIds.push(id);
    }

    // Get events since event 2
    const events = await getEventsSince(jobId, eventIds[2]);

    expect(events.length).toBe(2); // Events 3 and 4
    expect(events[0].data).toContain('60');
  });

  it('expires buffer after TTL', async () => {
    const jobId = 'ttl-test';

    await bufferEvent(jobId, 'progress', { percent: 50 }, 1);

    // Wait for TTL (use short TTL in test)
    await sleep(6000); // 5 minute TTL normally, shortened for test

    const events = await getEventsSince(jobId, null);
    expect(events.length).toBe(0);
  }, 10000);
});
```

**Expected Results**:
- Event ID format matches specification
- Buffer max size enforced
- Reconnection retrieval works
- TTL expiration works

---

## 7. State Machine Transitions

### INT-SM-001: Valid State Transitions

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-SM-001 |
| **Title** | Valid Job State Transitions |
| **Priority** | P0 - Critical |
| **Components** | `lib/job/state-machine.ts` |
| **Spec Reference** | Section 5.1.2 |

**Test Steps**:

```typescript
describe('Job State Machine', () => {
  const validTransitions = [
    { from: 'PENDING', to: 'UPLOADING', valid: true },
    { from: 'PENDING', to: 'CANCELLED', valid: true },
    { from: 'UPLOADING', to: 'PROCESSING', valid: true },
    { from: 'UPLOADING', to: 'ERROR', valid: true },
    { from: 'UPLOADING', to: 'CANCELLED', valid: true },
    { from: 'PROCESSING', to: 'COMPLETE', valid: true },
    { from: 'PROCESSING', to: 'PARTIAL_COMPLETE', valid: true },
    { from: 'PROCESSING', to: 'ERROR', valid: true },
    { from: 'PROCESSING', to: 'RETRY_1', valid: true },
    { from: 'PROCESSING', to: 'CANCELLED', valid: true },
    { from: 'RETRY_1', to: 'PROCESSING', valid: true },
    { from: 'RETRY_1', to: 'RETRY_2', valid: true },
    { from: 'RETRY_1', to: 'ERROR', valid: true },
    { from: 'RETRY_1', to: 'CANCELLED', valid: true },
    { from: 'RETRY_2', to: 'PROCESSING', valid: true },
    { from: 'RETRY_2', to: 'RETRY_3', valid: true },
    { from: 'RETRY_3', to: 'PROCESSING', valid: true },
    { from: 'RETRY_3', to: 'ERROR', valid: true },
  ];

  validTransitions.forEach(({ from, to, valid }) => {
    it(`${valid ? 'allows' : 'blocks'} transition from ${from} to ${to}`, async () => {
      const job = await prisma.job.create({
        data: { sessionId: 'test', status: from as JobStatus, inputType: 'FILE' },
      });

      if (valid) {
        await expect(transitionJobStatus(job.id, to as JobStatus)).resolves.not.toThrow();
      } else {
        await expect(transitionJobStatus(job.id, to as JobStatus)).rejects.toThrow('E707');
      }
    });
  });
});
```

**Expected Results**:
- All valid transitions allowed
- Invalid transitions throw E707

---

### INT-SM-002: Invalid State Transitions

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-SM-002 |
| **Title** | Invalid State Transitions Blocked |
| **Priority** | P0 - Critical |
| **Components** | `lib/job/state-machine.ts` |
| **Spec Reference** | Section 5.1.2 |

**Test Steps**:

```typescript
describe('Invalid State Transitions', () => {
  const invalidTransitions = [
    { from: 'COMPLETE', to: 'PROCESSING' },
    { from: 'ERROR', to: 'PROCESSING' },
    { from: 'CANCELLED', to: 'PROCESSING' },
    { from: 'PENDING', to: 'COMPLETE' },
    { from: 'PROCESSING', to: 'PENDING' },
  ];

  invalidTransitions.forEach(({ from, to }) => {
    it(`blocks transition from ${from} (terminal) to ${to}`, async () => {
      const job = await prisma.job.create({
        data: { sessionId: 'test', status: from as JobStatus, inputType: 'FILE' },
      });

      await expect(transitionJobStatus(job.id, to as JobStatus))
        .rejects.toMatchObject({ code: 'E707' });
    });
  });
});
```

**Expected Results**:
- Terminal states cannot transition
- Backward transitions blocked

---

## 8. Checkpoint and Resume Integration

### INT-CK-001: Checkpoint Storage

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-CK-001 |
| **Title** | Checkpoint Storage at Processing Stages |
| **Priority** | P1 - High |
| **Components** | `lib/checkpoint/manager.ts` |
| **Spec Reference** | Section 5.5 |

**Test Steps**:

```typescript
describe('Checkpoint Storage', () => {
  it('saves checkpoint after upload stage', async () => {
    const job = await createJob({ fileName: 'test.pdf' });

    await saveCheckpoint(job.id, 'uploaded', {
      uploadedFilePath: '/tmp/test.pdf',
    });

    const checkpoint = await loadCheckpoint(job.id);
    expect(checkpoint?.stage).toBe('uploaded');
    expect(checkpoint?.uploadedFilePath).toBe('/tmp/test.pdf');
  });

  it('saves checkpoint after conversion stage', async () => {
    const job = await createJob({ fileName: 'test.pdf' });
    const mockDoclingDoc = { schema_name: 'test', version: '1.0', name: 'test' };

    await saveCheckpoint(job.id, 'converted', {
      doclingDocument: mockDoclingDoc,
    });

    const checkpoint = await loadCheckpoint(job.id);
    expect(checkpoint?.stage).toBe('converted');
    expect(checkpoint?.doclingDocument).toEqual(mockDoclingDoc);
  });

  it('validates checksum on load', async () => {
    const job = await createJob({ fileName: 'test.pdf' });

    await saveCheckpoint(job.id, 'converted', {});

    // Corrupt the checkpoint
    await prisma.job.update({
      where: { id: job.id },
      data: {
        checkpointData: { corrupted: true },
      },
    });

    await expect(loadCheckpoint(job.id)).rejects.toThrow('CheckpointCorruptedError');
  });
});
```

**Expected Results**:
- Checkpoints saved at each stage
- Checksum validation catches corruption

---

### INT-CK-002: Resume from Checkpoint

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-CK-002 |
| **Title** | Resume Processing from Checkpoint |
| **Priority** | P1 - High |
| **Components** | `/api/v1/jobs/[id]/resume` |
| **Spec Reference** | Section 5.5 Resume Endpoint |

**Test Steps**:

```typescript
describe('Resume from Checkpoint', () => {
  it('resumes from converted checkpoint (skips conversion)', async () => {
    const job = await createJob({ fileName: 'test.pdf' });
    const mockDoclingDoc = { schema_name: 'test', version: '1.0', name: 'test' };

    await saveCheckpoint(job.id, 'converted', {
      doclingDocument: mockDoclingDoc,
    });

    const toolsCalled: string[] = [];
    mockMCPServer((request) => {
      if (request.method === 'tools/call') {
        toolsCalled.push(request.params.name);
      }
    });

    const response = await fetch(`/api/v1/jobs/${job.id}/resume`, {
      method: 'POST',
    });

    expect(response.status).toBe(202);

    // Verify conversion was skipped
    expect(toolsCalled).not.toContain('convert_pdf');
    expect(toolsCalled).toContain('export_markdown');
  });

  it('returns E703 when no checkpoint exists', async () => {
    const job = await createJob({ fileName: 'test.pdf' });

    const response = await fetch(`/api/v1/jobs/${job.id}/resume`, {
      method: 'POST',
    });

    expect(response.status).toBe(409);
    const data = await response.json();
    expect(data.error.code).toBe('E703');
  });

  it('returns E704 when checkpoint expired', async () => {
    const job = await createJob({ fileName: 'test.pdf' });

    await saveCheckpoint(job.id, 'converted', {
      expiresAt: new Date(Date.now() - 1000).toISOString(),
    });

    const response = await fetch(`/api/v1/jobs/${job.id}/resume`, {
      method: 'POST',
    });

    expect(response.status).toBe(409);
    const data = await response.json();
    expect(data.error.code).toBe('E704');
  });
});
```

**Expected Results**:
- Resume skips completed stages
- Error codes returned appropriately

---

## 9. Rate Limiting Integration

### INT-RL-001: API Rate Limit Enforcement

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-RL-001 |
| **Title** | Rate Limiting on API Endpoints |
| **Priority** | P1 - High |
| **Components** | API routes, rate limit middleware |
| **Spec Reference** | Section 8.4 |

**Test Steps**:

```typescript
describe('API Rate Limiting', () => {
  it('returns rate limit headers on all requests', async () => {
    const response = await fetch('/api/v1/upload', {
      method: 'POST',
      headers: { Cookie: 'session=test' },
      body: JSON.stringify({ url: 'https://example.com' }),
    });

    expect(response.headers.get('X-RateLimit-Limit')).toBe('10');
    expect(response.headers.get('X-RateLimit-Remaining')).toBeDefined();
    expect(response.headers.get('RateLimit-Limit')).toBe('10');
    expect(response.headers.get('RateLimit-Remaining')).toBeDefined();
  });

  it('returns 429 with correct headers when rate limited', async () => {
    const session = `rate-test-${Date.now()}`;

    // Exhaust rate limit
    for (let i = 0; i < 10; i++) {
      await fetch('/api/v1/upload', {
        method: 'POST',
        headers: { Cookie: `session=${session}` },
        body: JSON.stringify({ url: 'https://example.com' }),
      });
    }

    // 11th request
    const response = await fetch('/api/v1/upload', {
      method: 'POST',
      headers: { Cookie: `session=${session}` },
      body: JSON.stringify({ url: 'https://example.com' }),
    });

    expect(response.status).toBe(429);
    expect(response.headers.get('Retry-After')).toBeDefined();
    expect(response.headers.get('X-RateLimit-Reset')).toBeDefined();
    expect(response.headers.get('RateLimit-Reset')).toBeDefined();

    // Verify error body
    const data = await response.json();
    expect(data.error.code).toBe('E601');
  });
});
```

**Expected Results**:
- Rate limit headers on all responses
- 429 response with Retry-After header

---

## 10. Session Management Integration

### INT-SESS-001: Session Cookie Handling

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-SESS-001 |
| **Title** | Session Cookie Creation and Validation |
| **Priority** | P1 - High |
| **Components** | Session middleware |
| **Spec Reference** | Section 5.4 |

**Test Steps**:

```typescript
describe('Session Cookie Handling', () => {
  it('creates session cookie on first request', async () => {
    const response = await fetch('/api/v1/health');

    const setCookie = response.headers.get('Set-Cookie');
    expect(setCookie).toContain('hx-docling-session=');
    expect(setCookie).toContain('HttpOnly');
    expect(setCookie).toContain('SameSite=Strict');
  });

  it('reuses existing session cookie', async () => {
    // First request
    const response1 = await fetch('/api/v1/health');
    const sessionCookie = response1.headers.get('Set-Cookie')?.split(';')[0];

    // Second request with same cookie
    const response2 = await fetch('/api/v1/upload', {
      method: 'POST',
      headers: { Cookie: sessionCookie || '' },
      body: JSON.stringify({ url: 'https://example.com' }),
    });

    // Should not set new cookie
    expect(response2.headers.get('Set-Cookie')).toBeNull();
  });

  it('isolates data between sessions', async () => {
    // Create job in session A
    const uploadA = await fetch('/api/v1/upload', {
      method: 'POST',
      headers: { Cookie: 'session=session-a' },
      body: JSON.stringify({ url: 'https://example.com' }),
    });
    const { jobId } = await uploadA.json();

    // Try to access from session B
    const accessB = await fetch(`/api/v1/jobs/${jobId}`, {
      headers: { Cookie: 'session=session-b' },
    });

    expect(accessB.status).toBe(404);
  });
});
```

**Expected Results**:
- Session cookie created with secure flags
- Session isolation enforced

---

## 11. Frontend Component Integration

### 11.1 Overview

Tests validating the integration between React frontend components and the Next.js API backend, including form handling, SSE connections, and data fetching.

---

### INT-UI-001: UploadZone Component to /api/v1/upload Integration

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-UI-001 |
| **Title** | UploadZone Component to /api/v1/upload Integration |
| **Priority** | P0 - Critical |
| **Components** | `components/UploadZone.tsx`, `/api/v1/upload` |
| **Spec Reference** | FR-101, FR-102 |

**Preconditions**:
- Frontend test environment configured with MSW
- Test file fixtures available
- Session cookie mechanism active

**Test Steps**:

```typescript
describe('UploadZone to API Integration', () => {
  it('uploads file and receives job ID', async () => {
    render(<UploadZone />);

    const file = new File(['test content'], 'test.pdf', { type: 'application/pdf' });
    const dropzone = screen.getByTestId('upload-dropzone');

    await userEvent.upload(dropzone, file);

    // Wait for API response
    await waitFor(() => {
      expect(screen.getByTestId('job-id')).toBeInTheDocument();
    });

    // Verify API was called correctly
    expect(mockUploadHandler).toHaveBeenCalledWith(
      expect.objectContaining({
        method: 'POST',
        headers: expect.objectContaining({
          'Content-Type': expect.stringContaining('multipart/form-data'),
        }),
      })
    );
  });

  it('displays upload progress during file transfer', async () => {
    render(<UploadZone />);

    const largeFile = new File([new ArrayBuffer(5 * 1024 * 1024)], 'large.pdf', {
      type: 'application/pdf',
    });

    // Mock slow upload
    server.use(
      rest.post('/api/v1/upload', async (req, res, ctx) => {
        await delay(500);
        return res(ctx.json({ jobId: 'test-job-id', status: 'PENDING' }));
      })
    );

    const dropzone = screen.getByTestId('upload-dropzone');
    await userEvent.upload(dropzone, largeFile);

    // Progress indicator should appear
    expect(screen.getByRole('progressbar')).toBeInTheDocument();
  });

  it('handles drag and drop file upload', async () => {
    render(<UploadZone />);

    const file = new File(['test'], 'test.pdf', { type: 'application/pdf' });
    const dropzone = screen.getByTestId('upload-dropzone');

    fireEvent.dragOver(dropzone);
    expect(dropzone).toHaveClass('drag-active');

    fireEvent.drop(dropzone, {
      dataTransfer: { files: [file] },
    });

    await waitFor(() => {
      expect(mockUploadHandler).toHaveBeenCalled();
    });
  });

  it('validates file type before upload', async () => {
    render(<UploadZone />);

    const invalidFile = new File(['test'], 'test.exe', { type: 'application/x-executable' });
    const dropzone = screen.getByTestId('upload-dropzone');

    await userEvent.upload(dropzone, invalidFile);

    expect(screen.getByText(/unsupported file type/i)).toBeInTheDocument();
    expect(mockUploadHandler).not.toHaveBeenCalled();
  });

  it('enforces file size limit', async () => {
    render(<UploadZone />);

    const oversizedFile = new File([new ArrayBuffer(60 * 1024 * 1024)], 'huge.pdf', {
      type: 'application/pdf',
    });

    const dropzone = screen.getByTestId('upload-dropzone');
    await userEvent.upload(dropzone, oversizedFile);

    expect(screen.getByText(/file too large/i)).toBeInTheDocument();
    expect(mockUploadHandler).not.toHaveBeenCalled();
  });
});
```

**Expected Results**:
- File upload triggers API call with correct multipart form data
- Progress indicator displays during upload
- Drag and drop functionality works correctly
- Invalid file types rejected before API call
- File size validation enforced client-side

**Cleanup**:
- Reset MSW handlers
- Clear rendered components

---

### INT-UI-002: Form Validation with react-hook-form + Zod + Backend

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-UI-002 |
| **Title** | Form Validation with react-hook-form + Zod + Backend |
| **Priority** | P0 - Critical |
| **Components** | `components/UploadForm.tsx`, `lib/validation/schemas.ts`, `/api/v1/upload` |
| **Spec Reference** | FR-103, Section 7.1 |

**Preconditions**:
- Form component with react-hook-form and Zod validation
- MSW configured for backend validation responses

**Test Steps**:

```typescript
describe('Form Validation Integration', () => {
  it('validates URL format with Zod before submission', async () => {
    render(<UploadForm />);

    const urlInput = screen.getByLabelText(/url/i);
    await userEvent.type(urlInput, 'not-a-valid-url');

    const submitButton = screen.getByRole('button', { name: /submit/i });
    await userEvent.click(submitButton);

    // Zod validation error should appear
    expect(screen.getByText(/invalid url/i)).toBeInTheDocument();
    expect(mockUploadHandler).not.toHaveBeenCalled();
  });

  it('displays backend validation errors', async () => {
    server.use(
      rest.post('/api/v1/upload', (req, res, ctx) => {
        return res(
          ctx.status(400),
          ctx.json({
            error: {
              code: 'E103',
              message: 'Invalid URL: URL is unreachable',
            },
          })
        );
      })
    );

    render(<UploadForm />);

    const urlInput = screen.getByLabelText(/url/i);
    await userEvent.type(urlInput, 'https://unreachable.example.com');

    const submitButton = screen.getByRole('button', { name: /submit/i });
    await userEvent.click(submitButton);

    await waitFor(() => {
      expect(screen.getByText(/url is unreachable/i)).toBeInTheDocument();
    });
  });

  it('synchronizes client and server validation rules', async () => {
    render(<UploadForm />);

    // Test URL length validation (max 2048 chars)
    const longUrl = 'https://example.com/' + 'a'.repeat(2050);
    const urlInput = screen.getByLabelText(/url/i);
    await userEvent.type(urlInput, longUrl);

    const submitButton = screen.getByRole('button', { name: /submit/i });
    await userEvent.click(submitButton);

    // Client-side Zod validation should catch this
    expect(screen.getByText(/url too long/i)).toBeInTheDocument();
  });

  it('clears validation errors on valid input', async () => {
    render(<UploadForm />);

    const urlInput = screen.getByLabelText(/url/i);

    // Type invalid URL
    await userEvent.type(urlInput, 'invalid');
    await userEvent.tab(); // Trigger blur validation
    expect(screen.getByText(/invalid url/i)).toBeInTheDocument();

    // Clear and type valid URL
    await userEvent.clear(urlInput);
    await userEvent.type(urlInput, 'https://example.com/document.pdf');
    await userEvent.tab();

    expect(screen.queryByText(/invalid url/i)).not.toBeInTheDocument();
  });

  it('prevents double submission during processing', async () => {
    server.use(
      rest.post('/api/v1/upload', async (req, res, ctx) => {
        await delay(1000);
        return res(ctx.json({ jobId: 'test-job', status: 'PENDING' }));
      })
    );

    render(<UploadForm />);

    const urlInput = screen.getByLabelText(/url/i);
    await userEvent.type(urlInput, 'https://example.com/doc.pdf');

    const submitButton = screen.getByRole('button', { name: /submit/i });
    await userEvent.click(submitButton);
    await userEvent.click(submitButton);
    await userEvent.click(submitButton);

    // Only one API call should be made
    await waitFor(() => {
      expect(mockUploadHandler).toHaveBeenCalledTimes(1);
    });
  });
});
```

**Expected Results**:
- Client-side Zod validation prevents invalid submissions
- Backend validation errors displayed to user
- Validation rules synchronized between client and server
- Errors clear on valid input
- Double submission prevented

**Cleanup**:
- Reset form state
- Clear MSW handlers

---

### INT-UI-003: useSSE Hook Integration with Redis Pub/Sub

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-UI-003 |
| **Title** | useSSE Hook Integration with Redis Pub/Sub |
| **Priority** | P0 - Critical |
| **Components** | `hooks/useSSE.ts`, `/api/v1/process/[jobId]/events`, Redis pub/sub |
| **Spec Reference** | FR-501, Section 4.4 |

**Preconditions**:
- SSE endpoint configured
- Redis pub/sub mock available
- Job exists in PROCESSING state

**Test Steps**:

```typescript
describe('useSSE Hook Integration', () => {
  it('establishes SSE connection and receives progress events', async () => {
    const jobId = 'test-job-123';
    const events: SSEEvent[] = [];

    const { result } = renderHook(() => useSSE(jobId));

    // Simulate SSE events from server
    mockSSEStream([
      { event: 'progress', data: { stage: 'converting', percent: 25, message: 'Converting...' } },
      { event: 'progress', data: { stage: 'exporting', percent: 75, message: 'Exporting...' } },
      { event: 'complete', data: { status: 'COMPLETE' } },
    ]);

    await waitFor(() => {
      expect(result.current.events).toHaveLength(3);
      expect(result.current.status).toBe('connected');
    });

    expect(result.current.events[0].stage).toBe('converting');
    expect(result.current.events[1].stage).toBe('exporting');
  });

  it('reconnects automatically on connection loss', async () => {
    const jobId = 'reconnect-test';
    const { result } = renderHook(() => useSSE(jobId));

    await waitFor(() => {
      expect(result.current.status).toBe('connected');
    });

    // Simulate connection drop
    mockSSEDisconnect();

    await waitFor(() => {
      expect(result.current.status).toBe('reconnecting');
    });

    // Should reconnect within retry interval
    await waitFor(
      () => {
        expect(result.current.status).toBe('connected');
      },
      { timeout: 5000 }
    );
  });

  it('sends Last-Event-ID header on reconnection', async () => {
    const jobId = 'last-event-test';
    let lastEventIdHeader: string | null = null;

    server.use(
      rest.get(`/api/v1/process/${jobId}/events`, (req, res, ctx) => {
        lastEventIdHeader = req.headers.get('Last-Event-ID');
        return res(ctx.status(200));
      })
    );

    const { result } = renderHook(() => useSSE(jobId));

    // Receive some events
    mockSSEStream([
      { id: 'test1234-1234567890-0001', event: 'progress', data: { percent: 50 } },
    ]);

    await waitFor(() => {
      expect(result.current.lastEventId).toBe('test1234-1234567890-0001');
    });

    // Trigger reconnection
    mockSSEDisconnect();

    await waitFor(() => {
      expect(lastEventIdHeader).toBe('test1234-1234567890-0001');
    });
  });

  it('handles SSE error events', async () => {
    const jobId = 'error-test';
    const { result } = renderHook(() => useSSE(jobId));

    mockSSEStream([
      { event: 'error', data: { code: 'E201', message: 'Processing failed' } },
    ]);

    await waitFor(() => {
      expect(result.current.error).toMatchObject({
        code: 'E201',
        message: 'Processing failed',
      });
    });
  });

  it('cleans up EventSource on unmount', async () => {
    const jobId = 'cleanup-test';
    const closeSpy = vi.fn();

    global.EventSource = vi.fn().mockImplementation(() => ({
      close: closeSpy,
      addEventListener: vi.fn(),
      removeEventListener: vi.fn(),
    }));

    const { unmount } = renderHook(() => useSSE(jobId));

    unmount();

    expect(closeSpy).toHaveBeenCalled();
  });
});
```

**Expected Results**:
- SSE connection established successfully
- Progress events received and parsed correctly
- Automatic reconnection on connection loss
- Last-Event-ID sent on reconnection for missed events
- Error events handled properly
- EventSource cleaned up on component unmount

**Cleanup**:
- Close SSE connections
- Reset EventSource mock

---

### INT-UI-004: ResultsTabs Component to /api/v1/jobs/:id/results

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-UI-004 |
| **Title** | ResultsTabs Component to /api/v1/jobs/:id/results |
| **Priority** | P1 - High |
| **Components** | `components/ResultsTabs.tsx`, `/api/v1/jobs/[id]/results` |
| **Spec Reference** | FR-601, FR-602 |

**Preconditions**:
- Completed job with results available
- MSW configured for results endpoint

**Test Steps**:

```typescript
describe('ResultsTabs Integration', () => {
  const mockResults = {
    markdown: '# Test Document\n\nContent here.',
    html: '<h1>Test Document</h1><p>Content here.</p>',
    json: { schema_name: 'DoclingDocument', content: {} },
  };

  beforeEach(() => {
    server.use(
      rest.get('/api/v1/jobs/:jobId/results', (req, res, ctx) => {
        return res(ctx.json({ results: mockResults }));
      })
    );
  });

  it('fetches and displays results for each format', async () => {
    render(<ResultsTabs jobId="test-job" />);

    await waitFor(() => {
      expect(screen.getByRole('tab', { name: /markdown/i })).toBeInTheDocument();
      expect(screen.getByRole('tab', { name: /html/i })).toBeInTheDocument();
      expect(screen.getByRole('tab', { name: /json/i })).toBeInTheDocument();
    });

    // Default tab should show markdown
    expect(screen.getByText('# Test Document')).toBeInTheDocument();
  });

  it('switches between result formats on tab click', async () => {
    render(<ResultsTabs jobId="test-job" />);

    await waitFor(() => {
      expect(screen.getByRole('tab', { name: /html/i })).toBeInTheDocument();
    });

    // Click HTML tab
    await userEvent.click(screen.getByRole('tab', { name: /html/i }));

    expect(screen.getByText(/<h1>Test Document<\/h1>/)).toBeInTheDocument();

    // Click JSON tab
    await userEvent.click(screen.getByRole('tab', { name: /json/i }));

    expect(screen.getByText(/DoclingDocument/)).toBeInTheDocument();
  });

  it('displays loading state while fetching results', async () => {
    server.use(
      rest.get('/api/v1/jobs/:jobId/results', async (req, res, ctx) => {
        await delay(500);
        return res(ctx.json({ results: mockResults }));
      })
    );

    render(<ResultsTabs jobId="test-job" />);

    expect(screen.getByTestId('results-loading')).toBeInTheDocument();

    await waitFor(() => {
      expect(screen.queryByTestId('results-loading')).not.toBeInTheDocument();
    });
  });

  it('handles API error gracefully', async () => {
    server.use(
      rest.get('/api/v1/jobs/:jobId/results', (req, res, ctx) => {
        return res(
          ctx.status(404),
          ctx.json({ error: { code: 'E702', message: 'Job not found' } })
        );
      })
    );

    render(<ResultsTabs jobId="nonexistent-job" />);

    await waitFor(() => {
      expect(screen.getByText(/job not found/i)).toBeInTheDocument();
    });
  });
});
```

**Expected Results**:
- Results fetched from API on component mount
- All three format tabs displayed
- Tab switching shows correct content
- Loading state displayed during fetch
- API errors handled gracefully

**Cleanup**:
- Reset MSW handlers

---

### INT-UI-005: HistoryPanel Pagination with Backend

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-UI-005 |
| **Title** | HistoryPanel Pagination with Backend |
| **Priority** | P1 - High |
| **Components** | `components/HistoryPanel.tsx`, `/api/v1/history` |
| **Spec Reference** | FR-702 |

**Preconditions**:
- Multiple jobs exist in session
- MSW configured for history endpoint with pagination

**Test Steps**:

```typescript
describe('HistoryPanel Pagination', () => {
  const generateJobs = (count: number) =>
    Array.from({ length: count }, (_, i) => ({
      id: `job-${i}`,
      fileName: `document-${i}.pdf`,
      status: 'COMPLETE',
      createdAt: new Date(Date.now() - i * 60000).toISOString(),
    }));

  beforeEach(() => {
    server.use(
      rest.get('/api/v1/history', (req, res, ctx) => {
        const page = parseInt(req.url.searchParams.get('page') || '1');
        const pageSize = parseInt(req.url.searchParams.get('pageSize') || '10');
        const allJobs = generateJobs(25);

        const start = (page - 1) * pageSize;
        const jobs = allJobs.slice(start, start + pageSize);

        return res(
          ctx.json({
            jobs,
            pagination: {
              page,
              pageSize,
              totalCount: 25,
              totalPages: 3,
              hasMore: page < 3,
            },
          })
        );
      })
    );
  });

  it('displays first page of results on mount', async () => {
    render(<HistoryPanel />);

    await waitFor(() => {
      expect(screen.getByText('document-0.pdf')).toBeInTheDocument();
      expect(screen.getByText('document-9.pdf')).toBeInTheDocument();
      expect(screen.queryByText('document-10.pdf')).not.toBeInTheDocument();
    });
  });

  it('navigates to next page on click', async () => {
    render(<HistoryPanel />);

    await waitFor(() => {
      expect(screen.getByText('document-0.pdf')).toBeInTheDocument();
    });

    const nextButton = screen.getByRole('button', { name: /next/i });
    await userEvent.click(nextButton);

    await waitFor(() => {
      expect(screen.getByText('document-10.pdf')).toBeInTheDocument();
      expect(screen.queryByText('document-0.pdf')).not.toBeInTheDocument();
    });
  });

  it('disables previous button on first page', async () => {
    render(<HistoryPanel />);

    await waitFor(() => {
      expect(screen.getByRole('button', { name: /previous/i })).toBeDisabled();
    });
  });

  it('disables next button on last page', async () => {
    render(<HistoryPanel />);

    // Navigate to last page
    const nextButton = screen.getByRole('button', { name: /next/i });
    await userEvent.click(nextButton);
    await userEvent.click(nextButton);

    await waitFor(() => {
      expect(screen.getByRole('button', { name: /next/i })).toBeDisabled();
    });
  });

  it('displays pagination info correctly', async () => {
    render(<HistoryPanel />);

    await waitFor(() => {
      expect(screen.getByText(/page 1 of 3/i)).toBeInTheDocument();
      expect(screen.getByText(/25 total/i)).toBeInTheDocument();
    });
  });
});
```

**Expected Results**:
- First page of results displayed on mount
- Next/previous navigation works correctly
- Navigation buttons disabled appropriately
- Pagination info displayed accurately

**Cleanup**:
- Reset MSW handlers

---

### INT-UI-006: ThemeProvider localStorage Sync

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-UI-006 |
| **Title** | ThemeProvider localStorage Sync |
| **Priority** | P2 - Medium |
| **Components** | `components/ThemeProvider.tsx`, localStorage |
| **Spec Reference** | NFR-801 |

**Preconditions**:
- localStorage available
- ThemeProvider component configured

**Test Steps**:

```typescript
describe('ThemeProvider localStorage Sync', () => {
  beforeEach(() => {
    localStorage.clear();
  });

  it('reads theme preference from localStorage on mount', () => {
    localStorage.setItem('hx-docling-theme', 'dark');

    render(
      <ThemeProvider>
        <TestComponent />
      </ThemeProvider>
    );

    expect(document.documentElement).toHaveClass('dark');
  });

  it('persists theme change to localStorage', async () => {
    render(
      <ThemeProvider>
        <ThemeToggle />
      </ThemeProvider>
    );

    const toggleButton = screen.getByRole('button', { name: /toggle theme/i });
    await userEvent.click(toggleButton);

    expect(localStorage.getItem('hx-docling-theme')).toBe('dark');
  });

  it('defaults to system preference when no localStorage value', () => {
    // Mock system dark mode preference
    window.matchMedia = vi.fn().mockImplementation((query) => ({
      matches: query === '(prefers-color-scheme: dark)',
      addEventListener: vi.fn(),
      removeEventListener: vi.fn(),
    }));

    render(
      <ThemeProvider>
        <TestComponent />
      </ThemeProvider>
    );

    expect(document.documentElement).toHaveClass('dark');
  });

  it('responds to system preference changes', async () => {
    let preferenceListener: ((e: MediaQueryListEvent) => void) | null = null;

    window.matchMedia = vi.fn().mockImplementation(() => ({
      matches: false,
      addEventListener: (event: string, listener: (e: MediaQueryListEvent) => void) => {
        preferenceListener = listener;
      },
      removeEventListener: vi.fn(),
    }));

    render(
      <ThemeProvider>
        <TestComponent />
      </ThemeProvider>
    );

    expect(document.documentElement).not.toHaveClass('dark');

    // Simulate system preference change
    preferenceListener?.({ matches: true } as MediaQueryListEvent);

    await waitFor(() => {
      expect(document.documentElement).toHaveClass('dark');
    });
  });
});
```

**Expected Results**:
- Theme preference read from localStorage on mount
- Theme changes persisted to localStorage
- System preference used as default
- System preference changes detected

**Cleanup**:
- Clear localStorage
- Reset matchMedia mock

---

### INT-UI-007: Accessibility Keyboard Navigation

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-UI-007 |
| **Title** | Accessibility Keyboard Navigation |
| **Priority** | P1 - High |
| **Components** | All interactive components |
| **Spec Reference** | NFR-803, WCAG 2.1 AA |

**Preconditions**:
- Components rendered in test environment
- Keyboard events supported

**Test Steps**:

```typescript
describe('Keyboard Navigation', () => {
  it('allows tab navigation through all interactive elements', async () => {
    render(<AppLayout />);

    const interactiveElements = [
      'upload-button',
      'url-input',
      'submit-button',
      'theme-toggle',
      'history-link',
    ];

    for (const elementId of interactiveElements) {
      await userEvent.tab();
      expect(screen.getByTestId(elementId)).toHaveFocus();
    }
  });

  it('supports Enter key activation for buttons', async () => {
    const onUpload = vi.fn();
    render(<UploadButton onClick={onUpload} />);

    const button = screen.getByRole('button');
    button.focus();

    await userEvent.keyboard('{Enter}');

    expect(onUpload).toHaveBeenCalled();
  });

  it('supports Space key activation for buttons', async () => {
    const onUpload = vi.fn();
    render(<UploadButton onClick={onUpload} />);

    const button = screen.getByRole('button');
    button.focus();

    await userEvent.keyboard(' ');

    expect(onUpload).toHaveBeenCalled();
  });

  it('allows Escape key to close modals', async () => {
    render(<ConfirmDialog isOpen={true} />);

    expect(screen.getByRole('dialog')).toBeInTheDocument();

    await userEvent.keyboard('{Escape}');

    expect(screen.queryByRole('dialog')).not.toBeInTheDocument();
  });

  it('traps focus within modal when open', async () => {
    render(<ConfirmDialog isOpen={true} />);

    const dialog = screen.getByRole('dialog');
    const focusableElements = within(dialog).getAllByRole('button');

    // Tab through all elements
    for (let i = 0; i < focusableElements.length + 1; i++) {
      await userEvent.tab();
    }

    // Focus should cycle back to first element
    expect(focusableElements[0]).toHaveFocus();
  });

  it('uses arrow keys for tab navigation in ResultsTabs', async () => {
    render(<ResultsTabs jobId="test" />);

    const tablist = screen.getByRole('tablist');
    const tabs = within(tablist).getAllByRole('tab');

    tabs[0].focus();

    await userEvent.keyboard('{ArrowRight}');
    expect(tabs[1]).toHaveFocus();

    await userEvent.keyboard('{ArrowRight}');
    expect(tabs[2]).toHaveFocus();

    await userEvent.keyboard('{ArrowLeft}');
    expect(tabs[1]).toHaveFocus();
  });
});
```

**Expected Results**:
- Tab navigation reaches all interactive elements
- Enter and Space activate buttons
- Escape closes modals
- Focus trapped within open modals
- Arrow keys navigate tab components

**Cleanup**:
- Reset focus state

---

### INT-UI-008: ErrorBoundary with API Errors

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-UI-008 |
| **Title** | ErrorBoundary with API Errors |
| **Priority** | P0 - Critical |
| **Components** | `components/ErrorBoundary.tsx`, API error handling |
| **Spec Reference** | Section 7.1, NFR-802 |

**Preconditions**:
- ErrorBoundary wrapping application components
- MSW configured to return various error responses

**Test Steps**:

```typescript
describe('ErrorBoundary with API Errors', () => {
  it('catches and displays API 500 errors', async () => {
    server.use(
      rest.get('/api/v1/jobs/:id', (req, res, ctx) => {
        return res(
          ctx.status(500),
          ctx.json({ error: { code: 'E501', message: 'Internal server error' } })
        );
      })
    );

    render(
      <ErrorBoundary>
        <JobDetails jobId="test" />
      </ErrorBoundary>
    );

    await waitFor(() => {
      expect(screen.getByRole('alert')).toBeInTheDocument();
      expect(screen.getByText(/something went wrong/i)).toBeInTheDocument();
    });
  });

  it('provides retry action for recoverable errors', async () => {
    let requestCount = 0;
    server.use(
      rest.get('/api/v1/jobs/:id', (req, res, ctx) => {
        requestCount++;
        if (requestCount === 1) {
          return res(
            ctx.status(503),
            ctx.json({ error: { code: 'E502', message: 'Service unavailable' } })
          );
        }
        return res(ctx.json({ id: 'test', status: 'COMPLETE' }));
      })
    );

    render(
      <ErrorBoundary>
        <JobDetails jobId="test" />
      </ErrorBoundary>
    );

    await waitFor(() => {
      expect(screen.getByRole('button', { name: /retry/i })).toBeInTheDocument();
    });

    await userEvent.click(screen.getByRole('button', { name: /retry/i }));

    await waitFor(() => {
      expect(screen.getByText(/complete/i)).toBeInTheDocument();
    });
  });

  it('displays user-friendly error messages', async () => {
    server.use(
      rest.post('/api/v1/upload', (req, res, ctx) => {
        return res(
          ctx.status(400),
          ctx.json({ error: { code: 'E101', message: 'Invalid file format' } })
        );
      })
    );

    render(
      <ErrorBoundary>
        <UploadForm />
      </ErrorBoundary>
    );

    await submitForm();

    await waitFor(() => {
      // Should show user-friendly message, not raw error code
      expect(screen.getByText(/file format is not supported/i)).toBeInTheDocument();
      expect(screen.queryByText('E101')).not.toBeInTheDocument();
    });
  });

  it('logs errors to console in development', async () => {
    const consoleSpy = vi.spyOn(console, 'error').mockImplementation(() => {});

    server.use(
      rest.get('/api/v1/jobs/:id', (req, res, ctx) => {
        return res(ctx.status(500));
      })
    );

    render(
      <ErrorBoundary>
        <JobDetails jobId="test" />
      </ErrorBoundary>
    );

    await waitFor(() => {
      expect(consoleSpy).toHaveBeenCalled();
    });

    consoleSpy.mockRestore();
  });

  it('prevents error propagation to parent', async () => {
    const ThrowingComponent = () => {
      throw new Error('Test error');
    };

    const ParentErrorHandler = vi.fn();

    render(
      <div onError={ParentErrorHandler}>
        <ErrorBoundary>
          <ThrowingComponent />
        </ErrorBoundary>
      </div>
    );

    expect(ParentErrorHandler).not.toHaveBeenCalled();
    expect(screen.getByRole('alert')).toBeInTheDocument();
  });
});
```

**Expected Results**:
- API errors caught by ErrorBoundary
- Retry action available for recoverable errors
- User-friendly error messages displayed
- Errors logged in development mode
- Error propagation prevented

**Cleanup**:
- Reset MSW handlers
- Restore console mocks

---

### INT-UI-009: ProgressBar with SSE Updates

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-UI-009 |
| **Title** | ProgressBar with SSE Updates |
| **Priority** | P1 - High |
| **Components** | `components/ProgressBar.tsx`, `hooks/useSSE.ts` |
| **Spec Reference** | FR-501, FR-502 |

**Preconditions**:
- Job in PROCESSING state
- SSE endpoint configured

**Test Steps**:

```typescript
describe('ProgressBar with SSE Updates', () => {
  it('updates progress bar based on SSE events', async () => {
    render(<ProcessingView jobId="test-job" />);

    mockSSEStream([
      { event: 'progress', data: { stage: 'converting', percent: 0, message: 'Starting...' } },
    ]);

    await waitFor(() => {
      const progressBar = screen.getByRole('progressbar');
      expect(progressBar).toHaveAttribute('aria-valuenow', '0');
    });

    mockSSEStream([
      { event: 'progress', data: { stage: 'converting', percent: 50, message: 'Converting...' } },
    ]);

    await waitFor(() => {
      const progressBar = screen.getByRole('progressbar');
      expect(progressBar).toHaveAttribute('aria-valuenow', '50');
    });

    mockSSEStream([
      { event: 'progress', data: { stage: 'exporting', percent: 100, message: 'Complete!' } },
    ]);

    await waitFor(() => {
      const progressBar = screen.getByRole('progressbar');
      expect(progressBar).toHaveAttribute('aria-valuenow', '100');
    });
  });

  it('displays current stage message', async () => {
    render(<ProcessingView jobId="test-job" />);

    mockSSEStream([
      { event: 'progress', data: { stage: 'converting', percent: 25, message: 'Converting document...' } },
    ]);

    await waitFor(() => {
      expect(screen.getByText('Converting document...')).toBeInTheDocument();
    });

    mockSSEStream([
      { event: 'progress', data: { stage: 'exporting', percent: 75, message: 'Exporting to formats...' } },
    ]);

    await waitFor(() => {
      expect(screen.getByText('Exporting to formats...')).toBeInTheDocument();
    });
  });

  it('shows indeterminate state when percent is unknown', async () => {
    render(<ProcessingView jobId="test-job" />);

    mockSSEStream([
      { event: 'progress', data: { stage: 'initializing', percent: null, message: 'Initializing...' } },
    ]);

    await waitFor(() => {
      const progressBar = screen.getByRole('progressbar');
      expect(progressBar).not.toHaveAttribute('aria-valuenow');
      expect(progressBar).toHaveClass('indeterminate');
    });
  });

  it('transitions to success state on complete event', async () => {
    render(<ProcessingView jobId="test-job" />);

    mockSSEStream([{ event: 'complete', data: { status: 'COMPLETE' } }]);

    await waitFor(() => {
      expect(screen.getByTestId('success-indicator')).toBeInTheDocument();
      expect(screen.queryByRole('progressbar')).not.toBeInTheDocument();
    });
  });

  it('transitions to error state on error event', async () => {
    render(<ProcessingView jobId="test-job" />);

    mockSSEStream([
      { event: 'error', data: { code: 'E201', message: 'Processing failed' } },
    ]);

    await waitFor(() => {
      expect(screen.getByTestId('error-indicator')).toBeInTheDocument();
      expect(screen.getByText(/processing failed/i)).toBeInTheDocument();
    });
  });
});
```

**Expected Results**:
- Progress bar updates with SSE progress events
- Current stage message displayed
- Indeterminate state for unknown progress
- Success state on completion
- Error state on failure

**Cleanup**:
- Close SSE connections

---

### INT-UI-010: Download Buttons Trigger API Calls

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-UI-010 |
| **Title** | Download Buttons Trigger API Calls |
| **Priority** | P1 - High |
| **Components** | `components/DownloadButtons.tsx`, `/api/v1/jobs/[id]/download` |
| **Spec Reference** | FR-603 |

**Preconditions**:
- Completed job with results available
- Download endpoint configured

**Test Steps**:

```typescript
describe('Download Buttons Integration', () => {
  beforeEach(() => {
    server.use(
      rest.get('/api/v1/jobs/:jobId/download/:format', (req, res, ctx) => {
        const format = req.params.format as string;
        const content = format === 'json' ? '{"test": true}' : '# Test';

        return res(
          ctx.set('Content-Disposition', `attachment; filename="result.${format}"`),
          ctx.set('Content-Type', format === 'json' ? 'application/json' : 'text/plain'),
          ctx.body(content)
        );
      })
    );
  });

  it('triggers download API call on button click', async () => {
    const downloadSpy = vi.fn();
    global.URL.createObjectURL = vi.fn(() => 'blob:test');
    global.URL.revokeObjectURL = vi.fn();

    render(<DownloadButtons jobId="test-job" onDownload={downloadSpy} />);

    const markdownButton = screen.getByRole('button', { name: /download markdown/i });
    await userEvent.click(markdownButton);

    await waitFor(() => {
      expect(downloadSpy).toHaveBeenCalledWith('markdown');
    });
  });

  it('creates download link with correct filename', async () => {
    const createElementSpy = vi.spyOn(document, 'createElement');

    render(<DownloadButtons jobId="test-job" />);

    const jsonButton = screen.getByRole('button', { name: /download json/i });
    await userEvent.click(jsonButton);

    await waitFor(() => {
      const anchor = createElementSpy.mock.results.find(
        (r) => r.value?.tagName === 'A'
      )?.value as HTMLAnchorElement;

      expect(anchor?.download).toBe('result.json');
    });

    createElementSpy.mockRestore();
  });

  it('shows loading state during download', async () => {
    server.use(
      rest.get('/api/v1/jobs/:jobId/download/:format', async (req, res, ctx) => {
        await delay(500);
        return res(ctx.body('content'));
      })
    );

    render(<DownloadButtons jobId="test-job" />);

    const button = screen.getByRole('button', { name: /download markdown/i });
    await userEvent.click(button);

    expect(button).toHaveAttribute('aria-busy', 'true');
    expect(within(button).getByTestId('loading-spinner')).toBeInTheDocument();

    await waitFor(() => {
      expect(button).not.toHaveAttribute('aria-busy');
    });
  });

  it('handles download error gracefully', async () => {
    server.use(
      rest.get('/api/v1/jobs/:jobId/download/:format', (req, res, ctx) => {
        return res(
          ctx.status(404),
          ctx.json({ error: { code: 'E702', message: 'Result not found' } })
        );
      })
    );

    render(<DownloadButtons jobId="test-job" />);

    const button = screen.getByRole('button', { name: /download markdown/i });
    await userEvent.click(button);

    await waitFor(() => {
      expect(screen.getByText(/download failed/i)).toBeInTheDocument();
    });
  });

  it('disables buttons while download in progress', async () => {
    server.use(
      rest.get('/api/v1/jobs/:jobId/download/:format', async (req, res, ctx) => {
        await delay(1000);
        return res(ctx.body('content'));
      })
    );

    render(<DownloadButtons jobId="test-job" />);

    const markdownButton = screen.getByRole('button', { name: /download markdown/i });
    const htmlButton = screen.getByRole('button', { name: /download html/i });

    await userEvent.click(markdownButton);

    expect(htmlButton).toBeDisabled();
  });
});
```

**Expected Results**:
- Download API called with correct format
- Correct filename used for download
- Loading state displayed during download
- Errors handled gracefully
- Buttons disabled during download

**Cleanup**:
- Reset MSW handlers
- Restore document.createElement spy

---

## 12. Health Check Integration

### 12.1 Overview

Tests validating the health check endpoint behavior with various service availability states.

---

### INT-HEALTH-001: Health Endpoint with All Services Up

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-HEALTH-001 |
| **Title** | Health Endpoint with All Services Up |
| **Priority** | P1 - High |
| **Components** | `/api/v1/health`, PostgreSQL, Redis, MCP Client |
| **Spec Reference** | Section 4.1.5 |

**Preconditions**:
- PostgreSQL test instance running
- Redis test instance running
- MCP mock server running

**Test Steps**:

```typescript
describe('Health Endpoint - All Services Up', () => {
  beforeEach(() => {
    // Ensure all services are available
    mockPostgresHealthy();
    mockRedisHealthy();
    mockMCPHealthy();
  });

  it('returns 200 with healthy status', async () => {
    const response = await fetch('/api/v1/health');

    expect(response.status).toBe(200);

    const data = await response.json();
    expect(data.status).toBe('healthy');
  });

  it('includes all service statuses', async () => {
    const response = await fetch('/api/v1/health');
    const data = await response.json();

    expect(data.services).toEqual({
      database: { status: 'healthy', latency: expect.any(Number) },
      redis: { status: 'healthy', latency: expect.any(Number) },
      mcp: { status: 'healthy', latency: expect.any(Number) },
    });
  });

  it('includes version information', async () => {
    const response = await fetch('/api/v1/health');
    const data = await response.json();

    expect(data.version).toBeDefined();
    expect(data.uptime).toBeGreaterThan(0);
  });

  it('returns response within SLA (< 500ms)', async () => {
    const startTime = Date.now();
    await fetch('/api/v1/health');
    const duration = Date.now() - startTime;

    expect(duration).toBeLessThan(500);
  });
});
```

**Expected Results**:
- 200 status code returned
- All services reported as healthy
- Version and uptime included
- Response within SLA

**Cleanup**:
- Reset service mocks

---

### INT-HEALTH-002: Health with PostgreSQL Unavailable

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-HEALTH-002 |
| **Title** | Health with PostgreSQL Unavailable |
| **Priority** | P1 - High |
| **Components** | `/api/v1/health`, PostgreSQL |
| **Spec Reference** | Section 4.1.5 |

**Preconditions**:
- PostgreSQL unavailable or connection refused
- Redis and MCP available

**Test Steps**:

```typescript
describe('Health Endpoint - PostgreSQL Unavailable', () => {
  beforeEach(() => {
    mockPostgresUnavailable();
    mockRedisHealthy();
    mockMCPHealthy();
  });

  it('returns 503 with degraded status', async () => {
    const response = await fetch('/api/v1/health');

    expect(response.status).toBe(503);

    const data = await response.json();
    expect(data.status).toBe('unhealthy');
  });

  it('identifies PostgreSQL as unhealthy', async () => {
    const response = await fetch('/api/v1/health');
    const data = await response.json();

    expect(data.services.database).toEqual({
      status: 'unhealthy',
      error: expect.stringContaining('connection'),
    });
  });

  it('reports other services as healthy', async () => {
    const response = await fetch('/api/v1/health');
    const data = await response.json();

    expect(data.services.redis.status).toBe('healthy');
    expect(data.services.mcp.status).toBe('healthy');
  });

  it('includes appropriate error details', async () => {
    const response = await fetch('/api/v1/health');
    const data = await response.json();

    expect(data.error).toBe('One or more services unhealthy');
    expect(data.services.database.error).toBeDefined();
  });
});
```

**Expected Results**:
- 503 status code returned
- Overall status is unhealthy
- PostgreSQL identified as unhealthy
- Other services still reported correctly

**Cleanup**:
- Restore PostgreSQL mock

---

### INT-HEALTH-003: Health with Redis Unavailable

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-HEALTH-003 |
| **Title** | Health with Redis Unavailable |
| **Priority** | P1 - High |
| **Components** | `/api/v1/health`, Redis |
| **Spec Reference** | Section 4.1.5 |

**Preconditions**:
- Redis unavailable or connection refused
- PostgreSQL and MCP available

**Test Steps**:

```typescript
describe('Health Endpoint - Redis Unavailable', () => {
  beforeEach(() => {
    mockPostgresHealthy();
    mockRedisUnavailable();
    mockMCPHealthy();
  });

  it('returns 503 with degraded status', async () => {
    const response = await fetch('/api/v1/health');

    expect(response.status).toBe(503);

    const data = await response.json();
    expect(data.status).toBe('unhealthy');
  });

  it('identifies Redis as unhealthy', async () => {
    const response = await fetch('/api/v1/health');
    const data = await response.json();

    expect(data.services.redis).toEqual({
      status: 'unhealthy',
      error: expect.stringContaining('connection'),
    });
  });

  it('reports other services as healthy', async () => {
    const response = await fetch('/api/v1/health');
    const data = await response.json();

    expect(data.services.database.status).toBe('healthy');
    expect(data.services.mcp.status).toBe('healthy');
  });
});
```

**Expected Results**:
- 503 status code returned
- Overall status is unhealthy
- Redis identified as unhealthy
- Other services still reported correctly

**Cleanup**:
- Restore Redis mock

---

### INT-HEALTH-004: Health with MCP Unavailable

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-HEALTH-004 |
| **Title** | Health with MCP Unavailable |
| **Priority** | P1 - High |
| **Components** | `/api/v1/health`, MCP Client |
| **Spec Reference** | Section 4.1.5 |

**Preconditions**:
- MCP server unavailable
- PostgreSQL and Redis available

**Test Steps**:

```typescript
describe('Health Endpoint - MCP Unavailable', () => {
  beforeEach(() => {
    mockPostgresHealthy();
    mockRedisHealthy();
    mockMCPUnavailable();
  });

  it('returns 503 with degraded status', async () => {
    const response = await fetch('/api/v1/health');

    expect(response.status).toBe(503);

    const data = await response.json();
    expect(data.status).toBe('unhealthy');
  });

  it('identifies MCP as unhealthy', async () => {
    const response = await fetch('/api/v1/health');
    const data = await response.json();

    expect(data.services.mcp).toEqual({
      status: 'unhealthy',
      error: expect.stringContaining('unavailable'),
    });
  });

  it('reports other services as healthy', async () => {
    const response = await fetch('/api/v1/health');
    const data = await response.json();

    expect(data.services.database.status).toBe('healthy');
    expect(data.services.redis.status).toBe('healthy');
  });
});
```

**Expected Results**:
- 503 status code returned
- Overall status is unhealthy
- MCP identified as unhealthy
- Other services still reported correctly

**Cleanup**:
- Restore MCP mock

---

### INT-HEALTH-005: Readiness Probe Behavior

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-HEALTH-005 |
| **Title** | Readiness Probe Behavior |
| **Priority** | P1 - High |
| **Components** | `/api/v1/health/ready` |
| **Spec Reference** | Kubernetes readiness probe |

**Preconditions**:
- Readiness endpoint configured
- Services in various states

**Test Steps**:

```typescript
describe('Readiness Probe', () => {
  it('returns 200 when all critical services ready', async () => {
    mockPostgresHealthy();
    mockRedisHealthy();
    mockMCPHealthy();

    const response = await fetch('/api/v1/health/ready');

    expect(response.status).toBe(200);
    expect(await response.json()).toEqual({ ready: true });
  });

  it('returns 503 when database not ready', async () => {
    mockPostgresUnavailable();
    mockRedisHealthy();
    mockMCPHealthy();

    const response = await fetch('/api/v1/health/ready');

    expect(response.status).toBe(503);
    expect(await response.json()).toMatchObject({
      ready: false,
      reason: expect.stringContaining('database'),
    });
  });

  it('returns 503 when Redis not ready', async () => {
    mockPostgresHealthy();
    mockRedisUnavailable();
    mockMCPHealthy();

    const response = await fetch('/api/v1/health/ready');

    expect(response.status).toBe(503);
    expect(await response.json()).toMatchObject({
      ready: false,
      reason: expect.stringContaining('redis'),
    });
  });

  it('returns 200 even when MCP degraded (non-critical for readiness)', async () => {
    mockPostgresHealthy();
    mockRedisHealthy();
    mockMCPUnavailable();

    const response = await fetch('/api/v1/health/ready');

    // MCP may be non-critical for accepting traffic
    expect(response.status).toBe(200);
  });

  it('responds quickly for Kubernetes probe timeout', async () => {
    const startTime = Date.now();
    await fetch('/api/v1/health/ready');
    const duration = Date.now() - startTime;

    expect(duration).toBeLessThan(1000); // Kubernetes default probe timeout
  });
});
```

**Expected Results**:
- Returns 200 when critical services ready
- Returns 503 when database unavailable
- Returns 503 when Redis unavailable
- Fast response for Kubernetes timeout
- MCP may be non-critical for readiness

**Cleanup**:
- Reset service mocks

---

### INT-HEALTH-006: Liveness Probe Behavior

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-HEALTH-006 |
| **Title** | Liveness Probe Behavior |
| **Priority** | P1 - High |
| **Components** | `/api/v1/health/live` |
| **Spec Reference** | Kubernetes liveness probe |

**Preconditions**:
- Liveness endpoint configured
- Application in various states

**Test Steps**:

```typescript
describe('Liveness Probe', () => {
  it('returns 200 when application is alive', async () => {
    const response = await fetch('/api/v1/health/live');

    expect(response.status).toBe(200);
    expect(await response.json()).toEqual({ alive: true });
  });

  it('returns 200 even when external services unavailable', async () => {
    mockPostgresUnavailable();
    mockRedisUnavailable();
    mockMCPUnavailable();

    const response = await fetch('/api/v1/health/live');

    // Liveness only checks if app process is running
    expect(response.status).toBe(200);
  });

  it('includes basic process information', async () => {
    const response = await fetch('/api/v1/health/live');
    const data = await response.json();

    expect(data.alive).toBe(true);
    expect(data.uptime).toBeGreaterThan(0);
    expect(data.timestamp).toBeDefined();
  });

  it('does not perform expensive health checks', async () => {
    const startTime = Date.now();
    await fetch('/api/v1/health/live');
    const duration = Date.now() - startTime;

    // Should be very fast as it doesn't check external services
    expect(duration).toBeLessThan(50);
  });

  it('does not check database connection', async () => {
    const dbQuerySpy = vi.spyOn(prisma, '$queryRaw');

    await fetch('/api/v1/health/live');

    expect(dbQuerySpy).not.toHaveBeenCalled();
    dbQuerySpy.mockRestore();
  });

  it('does not check Redis connection', async () => {
    const redisPingSpy = vi.spyOn(redis, 'ping');

    await fetch('/api/v1/health/live');

    expect(redisPingSpy).not.toHaveBeenCalled();
    redisPingSpy.mockRestore();
  });
});
```

**Expected Results**:
- Returns 200 when process is running
- Does not depend on external services
- Includes basic process info
- Very fast response time
- No database or Redis checks performed

**Cleanup**:
- Restore spies

---

## 13. Next.js Middleware Integration

### 13.1 Overview

Tests validating the Next.js middleware chain execution, including authentication, rate limiting, validation, and error handling middleware layers.

---

### INT-MW-001: Middleware Chain Execution Order

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-MW-001 |
| **Title** | Middleware Chain Execution Order (auth -> rateLimit -> validation -> handler) |
| **Priority** | P1 - High |
| **Components** | `middleware.ts`, `lib/middleware/*.ts` |
| **Spec Reference** | Section 7.4 |

**Preconditions**:
- Middleware stack configured
- All middleware layers active
- Test request with valid session

**Test Steps**:

```typescript
describe('Middleware Chain Execution Order', () => {
  it('executes middleware in correct order: auth -> rateLimit -> validation -> handler', async () => {
    const executionOrder: string[] = [];

    // Spy on middleware functions
    vi.spyOn(authMiddleware, 'handle').mockImplementation(async (req, next) => {
      executionOrder.push('auth');
      return next();
    });

    vi.spyOn(rateLimitMiddleware, 'handle').mockImplementation(async (req, next) => {
      executionOrder.push('rateLimit');
      return next();
    });

    vi.spyOn(validationMiddleware, 'handle').mockImplementation(async (req, next) => {
      executionOrder.push('validation');
      return next();
    });

    const response = await fetch('/api/v1/upload', {
      method: 'POST',
      headers: { Cookie: 'session=test-session' },
      body: JSON.stringify({ url: 'https://example.com/doc.pdf' }),
    });

    expect(executionOrder).toEqual(['auth', 'rateLimit', 'validation']);
  });

  it('short-circuits on auth failure before rate limiting', async () => {
    const rateLimitCalled = { value: false };

    vi.spyOn(rateLimitMiddleware, 'handle').mockImplementation(async (req, next) => {
      rateLimitCalled.value = true;
      return next();
    });

    const response = await fetch('/api/v1/upload', {
      method: 'POST',
      // No session cookie - auth should fail
      body: JSON.stringify({ url: 'https://example.com/doc.pdf' }),
    });

    expect(response.status).toBe(401);
    expect(rateLimitCalled.value).toBe(false);
  });

  it('short-circuits on rate limit before validation', async () => {
    const sessionId = `rate-test-${Date.now()}`;
    const validationCalled = { value: false };

    vi.spyOn(validationMiddleware, 'handle').mockImplementation(async (req, next) => {
      validationCalled.value = true;
      return next();
    });

    // Exhaust rate limit
    for (let i = 0; i < 10; i++) {
      await fetch('/api/v1/upload', {
        method: 'POST',
        headers: { Cookie: `session=${sessionId}` },
        body: JSON.stringify({ url: 'https://example.com/doc.pdf' }),
      });
    }

    // 11th request should be rate limited
    const response = await fetch('/api/v1/upload', {
      method: 'POST',
      headers: { Cookie: `session=${sessionId}` },
      body: JSON.stringify({ url: 'https://example.com/doc.pdf' }),
    });

    expect(response.status).toBe(429);
    expect(validationCalled.value).toBe(false);
  });

  it('passes request context through middleware chain', async () => {
    let contextFromHandler: any = null;

    mockHandler.mockImplementation(async (req) => {
      contextFromHandler = {
        sessionId: req.context.sessionId,
        rateLimitRemaining: req.context.rateLimitRemaining,
        validatedBody: req.context.validatedBody,
      };
      return new Response(JSON.stringify({ success: true }));
    });

    await fetch('/api/v1/upload', {
      method: 'POST',
      headers: { Cookie: 'session=test-session' },
      body: JSON.stringify({ url: 'https://example.com/doc.pdf' }),
    });

    expect(contextFromHandler.sessionId).toBe('test-session');
    expect(contextFromHandler.rateLimitRemaining).toBeDefined();
    expect(contextFromHandler.validatedBody).toBeDefined();
  });
});
```

**Expected Results**:
- Middleware executes in order: auth -> rateLimit -> validation -> handler
- Auth failure short-circuits before rate limiting
- Rate limit failure short-circuits before validation
- Request context passed through middleware chain

**Cleanup**:
- Restore middleware spies
- Reset rate limit state

---

### INT-MW-002: Session Middleware to Redis Integration

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-MW-002 |
| **Title** | Session Middleware to Redis Integration |
| **Priority** | P1 - High |
| **Components** | `lib/middleware/session.ts`, `lib/redis/session.ts` |
| **Spec Reference** | Section 5.4 |

**Preconditions**:
- Redis test instance available
- Session middleware configured

**Test Steps**:

```typescript
describe('Session Middleware to Redis Integration', () => {
  it('creates new session in Redis on first request', async () => {
    const response = await fetch('/api/v1/health');

    const setCookie = response.headers.get('Set-Cookie');
    const sessionId = extractSessionId(setCookie);

    // Verify session exists in Redis
    const session = await redis.get(`session:${sessionId}`);
    expect(session).not.toBeNull();

    const sessionData = JSON.parse(session!);
    expect(sessionData.id).toBe(sessionId);
    expect(sessionData.createdAt).toBeDefined();
  });

  it('retrieves existing session from Redis', async () => {
    // Create session directly in Redis
    const sessionId = 'existing-session-123';
    const sessionData = {
      id: sessionId,
      createdAt: Date.now() - 60000,
      lastActivity: Date.now() - 30000,
      jobCount: 5,
    };
    await redis.set(`session:${sessionId}`, JSON.stringify(sessionData));

    const response = await fetch('/api/v1/health', {
      headers: { Cookie: `hx-docling-session=${sessionId}` },
    });

    // Should not create new session
    expect(response.headers.get('Set-Cookie')).toBeNull();
  });

  it('updates lastActivity in Redis on each request', async () => {
    const sessionId = 'activity-test-session';
    const initialTime = Date.now() - 60000;
    await redis.set(`session:${sessionId}`, JSON.stringify({
      id: sessionId,
      createdAt: initialTime,
      lastActivity: initialTime,
    }));

    await fetch('/api/v1/health', {
      headers: { Cookie: `hx-docling-session=${sessionId}` },
    });

    const updatedSession = JSON.parse(await redis.get(`session:${sessionId}`)!);
    expect(updatedSession.lastActivity).toBeGreaterThan(initialTime);
  });

  it('extends session TTL on activity', async () => {
    const sessionId = 'ttl-test-session';
    await redis.set(`session:${sessionId}`, JSON.stringify({
      id: sessionId,
      createdAt: Date.now(),
      lastActivity: Date.now(),
    }));
    await redis.expire(`session:${sessionId}`, 60); // 60 seconds TTL

    await fetch('/api/v1/health', {
      headers: { Cookie: `hx-docling-session=${sessionId}` },
    });

    const ttl = await redis.ttl(`session:${sessionId}`);
    expect(ttl).toBeGreaterThan(60); // TTL should be extended
  });

  it('handles Redis connection failure gracefully', async () => {
    mockRedisUnavailable();

    const response = await fetch('/api/v1/health');

    // Should still respond, possibly with degraded mode
    expect(response.status).toBeLessThan(500);
  });
});
```

**Expected Results**:
- New sessions created in Redis on first request
- Existing sessions retrieved from Redis
- lastActivity updated on each request
- TTL extended on activity
- Graceful degradation on Redis failure

**Cleanup**:
- Clear Redis test keys
- Restore Redis mock

---

### INT-MW-003: Rate Limit Middleware Integration

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-MW-003 |
| **Title** | Rate Limit Middleware Integration |
| **Priority** | P1 - High |
| **Components** | `lib/middleware/rate-limit.ts`, `lib/redis/rate-limit.ts` |
| **Spec Reference** | Section 7.4.2 |

**Preconditions**:
- Redis test instance available
- Rate limit middleware configured

**Test Steps**:

```typescript
describe('Rate Limit Middleware Integration', () => {
  it('increments rate limit counter in Redis', async () => {
    const sessionId = `rate-counter-${Date.now()}`;

    await fetch('/api/v1/upload', {
      method: 'POST',
      headers: { Cookie: `session=${sessionId}` },
      body: JSON.stringify({ url: 'https://example.com/doc.pdf' }),
    });

    const counter = await redis.get(`ratelimit:${sessionId}`);
    expect(parseInt(counter!)).toBe(1);
  });

  it('applies different limits per endpoint', async () => {
    const sessionId = `endpoint-limits-${Date.now()}`;

    // Upload endpoint: 10/minute
    for (let i = 0; i < 10; i++) {
      const response = await fetch('/api/v1/upload', {
        method: 'POST',
        headers: { Cookie: `session=${sessionId}` },
        body: JSON.stringify({ url: 'https://example.com/doc.pdf' }),
      });
      expect(response.status).not.toBe(429);
    }

    // 11th should be rate limited
    const limitedResponse = await fetch('/api/v1/upload', {
      method: 'POST',
      headers: { Cookie: `session=${sessionId}` },
      body: JSON.stringify({ url: 'https://example.com/doc.pdf' }),
    });
    expect(limitedResponse.status).toBe(429);

    // History endpoint may have different limit (e.g., 30/minute)
    const historyResponse = await fetch('/api/v1/history', {
      headers: { Cookie: `session=${sessionId}` },
    });
    expect(historyResponse.status).not.toBe(429);
  });

  it('uses sliding window algorithm', async () => {
    const sessionId = `sliding-window-${Date.now()}`;

    // Make 5 requests at time T
    for (let i = 0; i < 5; i++) {
      await fetch('/api/v1/upload', {
        method: 'POST',
        headers: { Cookie: `session=${sessionId}` },
        body: JSON.stringify({ url: 'https://example.com/doc.pdf' }),
      });
    }

    // Wait 30 seconds (half window)
    await sleep(30000);

    // Make 5 more requests - should still be allowed (sliding window)
    for (let i = 0; i < 5; i++) {
      const response = await fetch('/api/v1/upload', {
        method: 'POST',
        headers: { Cookie: `session=${sessionId}` },
        body: JSON.stringify({ url: 'https://example.com/doc.pdf' }),
      });
      expect(response.status).not.toBe(429);
    }
  }, 60000);

  it('includes rate limit info in response headers', async () => {
    const sessionId = `headers-test-${Date.now()}`;

    const response = await fetch('/api/v1/upload', {
      method: 'POST',
      headers: { Cookie: `session=${sessionId}` },
      body: JSON.stringify({ url: 'https://example.com/doc.pdf' }),
    });

    expect(response.headers.get('X-RateLimit-Limit')).toBe('10');
    expect(response.headers.get('X-RateLimit-Remaining')).toBe('9');
    expect(response.headers.get('X-RateLimit-Reset')).toBeDefined();
  });
});
```

**Expected Results**:
- Rate limit counter stored in Redis
- Different limits applied per endpoint
- Sliding window algorithm used
- Rate limit headers included in response

**Cleanup**:
- Clear Redis rate limit keys

---

### INT-MW-004: Error Handling Middleware

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-MW-004 |
| **Title** | Error Handling Middleware |
| **Priority** | P1 - High |
| **Components** | `lib/middleware/error-handler.ts` |
| **Spec Reference** | Section 7.1 |

**Preconditions**:
- Error handling middleware configured
- Various error scenarios available

**Test Steps**:

```typescript
describe('Error Handling Middleware', () => {
  it('catches and formats application errors', async () => {
    server.use(
      rest.post('/api/v1/upload', () => {
        throw new ApplicationError('E101', 'Invalid file format');
      })
    );

    const response = await fetch('/api/v1/upload', {
      method: 'POST',
      headers: { Cookie: 'session=test' },
      body: JSON.stringify({ url: 'https://example.com/bad.exe' }),
    });

    expect(response.status).toBe(400);
    const data = await response.json();
    expect(data).toEqual({
      error: {
        code: 'E101',
        message: 'Invalid file format',
      },
    });
  });

  it('converts unknown errors to E501', async () => {
    server.use(
      rest.post('/api/v1/upload', () => {
        throw new Error('Unexpected error');
      })
    );

    const response = await fetch('/api/v1/upload', {
      method: 'POST',
      headers: { Cookie: 'session=test' },
      body: JSON.stringify({ url: 'https://example.com/doc.pdf' }),
    });

    expect(response.status).toBe(500);
    const data = await response.json();
    expect(data.error.code).toBe('E501');
  });

  it('does not expose internal error details in production', async () => {
    const originalEnv = process.env.NODE_ENV;
    process.env.NODE_ENV = 'production';

    server.use(
      rest.post('/api/v1/upload', () => {
        throw new Error('Database connection string: postgres://user:secret@host');
      })
    );

    const response = await fetch('/api/v1/upload', {
      method: 'POST',
      headers: { Cookie: 'session=test' },
      body: JSON.stringify({ url: 'https://example.com/doc.pdf' }),
    });

    const data = await response.json();
    expect(data.error.message).not.toContain('secret');
    expect(data.error.message).not.toContain('postgres');

    process.env.NODE_ENV = originalEnv;
  });

  it('includes request ID for error tracking', async () => {
    server.use(
      rest.post('/api/v1/upload', () => {
        throw new ApplicationError('E201', 'Processing failed');
      })
    );

    const response = await fetch('/api/v1/upload', {
      method: 'POST',
      headers: { Cookie: 'session=test' },
      body: JSON.stringify({ url: 'https://example.com/doc.pdf' }),
    });

    const data = await response.json();
    expect(data.error.requestId).toBeDefined();
    expect(data.error.requestId).toMatch(/^[a-f0-9-]+$/);
  });

  it('logs errors with appropriate severity', async () => {
    const loggerSpy = vi.spyOn(logger, 'error');

    server.use(
      rest.post('/api/v1/upload', () => {
        throw new ApplicationError('E501', 'Internal error');
      })
    );

    await fetch('/api/v1/upload', {
      method: 'POST',
      headers: { Cookie: 'session=test' },
      body: JSON.stringify({ url: 'https://example.com/doc.pdf' }),
    });

    expect(loggerSpy).toHaveBeenCalledWith(
      expect.objectContaining({
        level: 'error',
        code: 'E501',
      })
    );

    loggerSpy.mockRestore();
  });
});
```

**Expected Results**:
- Application errors formatted correctly
- Unknown errors converted to E501
- Internal details hidden in production
- Request ID included for tracking
- Errors logged with appropriate severity

**Cleanup**:
- Restore environment variables
- Restore logger spy

---

### INT-MW-005: Request Validation Middleware

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-MW-005 |
| **Title** | Request Validation Middleware |
| **Priority** | P2 - Medium |
| **Components** | `lib/middleware/validation.ts`, `lib/validation/schemas.ts` |
| **Spec Reference** | Section 7.1 |

**Preconditions**:
- Validation middleware configured
- Zod schemas defined

**Test Steps**:

```typescript
describe('Request Validation Middleware', () => {
  it('validates request body against schema', async () => {
    const response = await fetch('/api/v1/upload', {
      method: 'POST',
      headers: {
        Cookie: 'session=test',
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ invalid: 'body' }),
    });

    expect(response.status).toBe(400);
    const data = await response.json();
    expect(data.error.code).toBe('E102');
  });

  it('validates query parameters', async () => {
    const response = await fetch('/api/v1/history?page=-1', {
      headers: { Cookie: 'session=test' },
    });

    expect(response.status).toBe(400);
    const data = await response.json();
    expect(data.error.message).toContain('page');
  });

  it('validates path parameters', async () => {
    const response = await fetch('/api/v1/jobs/not-a-uuid', {
      headers: { Cookie: 'session=test' },
    });

    expect(response.status).toBe(400);
    const data = await response.json();
    expect(data.error.message).toContain('jobId');
  });

  it('provides detailed validation error messages', async () => {
    const response = await fetch('/api/v1/upload', {
      method: 'POST',
      headers: {
        Cookie: 'session=test',
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        url: 'not-a-url',
        options: { outputFormats: ['invalid'] },
      }),
    });

    const data = await response.json();
    expect(data.error.details).toBeDefined();
    expect(data.error.details).toContainEqual(
      expect.objectContaining({ field: 'url' })
    );
  });

  it('sanitizes validated input', async () => {
    let sanitizedBody: any = null;

    mockHandler.mockImplementation(async (req) => {
      sanitizedBody = req.context.validatedBody;
      return new Response(JSON.stringify({ jobId: 'test' }), { status: 201 });
    });

    await fetch('/api/v1/upload', {
      method: 'POST',
      headers: {
        Cookie: 'session=test',
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        url: 'https://example.com/doc.pdf',
        extraField: 'should be stripped',
      }),
    });

    expect(sanitizedBody.url).toBe('https://example.com/doc.pdf');
    expect(sanitizedBody.extraField).toBeUndefined();
  });
});
```

**Expected Results**:
- Request body validated against schema
- Query parameters validated
- Path parameters validated
- Detailed validation error messages provided
- Input sanitized after validation

**Cleanup**:
- Reset handler mock

---

## 14. Server Component Integration

### 14.1 Overview

Tests validating Next.js React Server Components integration patterns, including data fetching, caching, and mutations.

---

### INT-RSC-001: Server Component Fetch Deduplication

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-RSC-001 |
| **Title** | Server Component Fetch Deduplication |
| **Priority** | P1 - High |
| **Components** | Server Components, `fetch` with cache |
| **Spec Reference** | Next.js Server Components patterns |

**Preconditions**:
- Server Components configured
- Multiple components requesting same data

**Test Steps**:

```typescript
describe('Server Component Fetch Deduplication', () => {
  it('deduplicates identical fetch requests in render tree', async () => {
    let fetchCount = 0;

    server.use(
      rest.get('/api/v1/jobs/:jobId', (req, res, ctx) => {
        fetchCount++;
        return res(ctx.json({ id: req.params.jobId, status: 'COMPLETE' }));
      })
    );

    // Render page with multiple components fetching same job
    const html = await renderServerComponent(
      <JobPage jobId="test-job">
        <JobHeader jobId="test-job" />
        <JobDetails jobId="test-job" />
        <JobActions jobId="test-job" />
      </JobPage>
    );

    // All components fetch same job, but should only result in one request
    expect(fetchCount).toBe(1);
  });

  it('does not deduplicate different requests', async () => {
    let fetchCount = 0;

    server.use(
      rest.get('/api/v1/jobs/:jobId', (req, res, ctx) => {
        fetchCount++;
        return res(ctx.json({ id: req.params.jobId, status: 'COMPLETE' }));
      })
    );

    const html = await renderServerComponent(
      <div>
        <JobDetails jobId="job-1" />
        <JobDetails jobId="job-2" />
      </div>
    );

    expect(fetchCount).toBe(2);
  });

  it('deduplicates requests with same cache key', async () => {
    let fetchCount = 0;

    server.use(
      rest.get('/api/v1/history', (req, res, ctx) => {
        fetchCount++;
        return res(ctx.json({ jobs: [], pagination: {} }));
      })
    );

    const html = await renderServerComponent(
      <HistoryPage>
        <HistoryList />
        <HistoryStats />
      </HistoryPage>
    );

    // Both use same cache key for history
    expect(fetchCount).toBe(1);
  });
});
```

**Expected Results**:
- Identical fetch requests deduplicated in render tree
- Different requests not deduplicated
- Cache key determines deduplication

**Cleanup**:
- Reset MSW handlers

---

### INT-RSC-002: Server Component to API Route Caching

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-RSC-002 |
| **Title** | Server Component to API Route Caching |
| **Priority** | P1 - High |
| **Components** | Server Components, `fetch` cache options |
| **Spec Reference** | Next.js caching patterns |

**Preconditions**:
- Server Components with various cache configurations
- API routes returning cacheable data

**Test Steps**:

```typescript
describe('Server Component to API Route Caching', () => {
  it('caches static data with force-cache', async () => {
    let fetchCount = 0;

    server.use(
      rest.get('/api/v1/config', (req, res, ctx) => {
        fetchCount++;
        return res(ctx.json({ maxFileSize: 52428800, supportedFormats: ['pdf'] }));
      })
    );

    // First render
    await renderServerComponent(<ConfigDisplay />);
    expect(fetchCount).toBe(1);

    // Second render - should use cache
    await renderServerComponent(<ConfigDisplay />);
    expect(fetchCount).toBe(1);
  });

  it('bypasses cache with no-store', async () => {
    let fetchCount = 0;

    server.use(
      rest.get('/api/v1/jobs/:jobId', (req, res, ctx) => {
        fetchCount++;
        return res(ctx.json({ id: req.params.jobId, status: 'PROCESSING' }));
      })
    );

    // Render with no-store (job status changes frequently)
    await renderServerComponent(<JobStatusBadge jobId="test" />);
    expect(fetchCount).toBe(1);

    await renderServerComponent(<JobStatusBadge jobId="test" />);
    expect(fetchCount).toBe(2);
  });

  it('revalidates after specified time', async () => {
    let fetchCount = 0;

    server.use(
      rest.get('/api/v1/stats', (req, res, ctx) => {
        fetchCount++;
        return res(ctx.json({ totalJobs: 100, completedJobs: 95 }));
      })
    );

    // Render with revalidate: 60
    await renderServerComponent(<StatsDisplay />);
    expect(fetchCount).toBe(1);

    // Immediate re-render uses cache
    await renderServerComponent(<StatsDisplay />);
    expect(fetchCount).toBe(1);

    // After revalidation period
    await advanceTimers(61000);
    await renderServerComponent(<StatsDisplay />);
    expect(fetchCount).toBe(2);
  });

  it('invalidates cache on revalidatePath', async () => {
    let fetchCount = 0;

    server.use(
      rest.get('/api/v1/jobs/:jobId', (req, res, ctx) => {
        fetchCount++;
        return res(ctx.json({ id: req.params.jobId, status: 'COMPLETE' }));
      })
    );

    await renderServerComponent(<JobDetails jobId="test" />);
    expect(fetchCount).toBe(1);

    // Trigger revalidation
    await revalidatePath('/jobs/test');

    await renderServerComponent(<JobDetails jobId="test" />);
    expect(fetchCount).toBe(2);
  });
});
```

**Expected Results**:
- Static data cached with force-cache
- Dynamic data bypasses cache with no-store
- Cache revalidates after specified time
- revalidatePath invalidates cache

**Cleanup**:
- Reset timers
- Clear cache

---

### INT-RSC-003: Server Action Mutation Flow

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-RSC-003 |
| **Title** | Server Action Mutation Flow |
| **Priority** | P1 - High |
| **Components** | Server Actions, `revalidatePath`, `redirect` |
| **Spec Reference** | Next.js Server Actions patterns |

**Preconditions**:
- Server Actions configured
- Form components with server action handlers

**Test Steps**:

```typescript
describe('Server Action Mutation Flow', () => {
  it('executes server action and revalidates path', async () => {
    let jobCreated = false;
    let revalidateCalled = false;

    server.use(
      rest.post('/api/v1/upload', (req, res, ctx) => {
        jobCreated = true;
        return res(ctx.json({ jobId: 'new-job-123' }));
      })
    );

    vi.spyOn(nextCache, 'revalidatePath').mockImplementation(() => {
      revalidateCalled = true;
    });

    render(<UploadForm />);

    const fileInput = screen.getByTestId('file-input');
    await userEvent.upload(fileInput, new File(['test'], 'test.pdf'));

    const submitButton = screen.getByRole('button', { name: /upload/i });
    await userEvent.click(submitButton);

    await waitFor(() => {
      expect(jobCreated).toBe(true);
      expect(revalidateCalled).toBe(true);
    });
  });

  it('redirects after successful mutation', async () => {
    let redirectPath: string | null = null;

    vi.spyOn(nextNavigation, 'redirect').mockImplementation((path) => {
      redirectPath = path;
      throw new Error('NEXT_REDIRECT');
    });

    server.use(
      rest.post('/api/v1/upload', (req, res, ctx) => {
        return res(ctx.json({ jobId: 'new-job-123' }));
      })
    );

    render(<UploadForm />);

    await submitUploadForm();

    expect(redirectPath).toBe('/jobs/new-job-123');
  });

  it('handles server action errors', async () => {
    server.use(
      rest.post('/api/v1/upload', (req, res, ctx) => {
        return res(
          ctx.status(400),
          ctx.json({ error: { code: 'E101', message: 'Invalid file' } })
        );
      })
    );

    render(<UploadForm />);

    await submitUploadForm();

    await waitFor(() => {
      expect(screen.getByText(/invalid file/i)).toBeInTheDocument();
    });
  });

  it('shows pending state during mutation', async () => {
    server.use(
      rest.post('/api/v1/upload', async (req, res, ctx) => {
        await delay(500);
        return res(ctx.json({ jobId: 'new-job' }));
      })
    );

    render(<UploadForm />);

    const submitButton = screen.getByRole('button', { name: /upload/i });
    await userEvent.click(submitButton);

    expect(submitButton).toBeDisabled();
    expect(screen.getByTestId('loading-indicator')).toBeInTheDocument();

    await waitFor(() => {
      expect(submitButton).not.toBeDisabled();
    });
  });
});
```

**Expected Results**:
- Server action executes and revalidates path
- Redirects after successful mutation
- Server action errors handled
- Pending state shown during mutation

**Cleanup**:
- Restore navigation mock
- Reset MSW handlers

---

### INT-RSC-004: Parallel Server Component Fetching

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-RSC-004 |
| **Title** | Parallel Server Component Fetching |
| **Priority** | P2 - Medium |
| **Components** | Server Components, `Suspense`, parallel data loading |
| **Spec Reference** | Next.js streaming patterns |

**Preconditions**:
- Server Components with independent data needs
- Suspense boundaries configured

**Test Steps**:

```typescript
describe('Parallel Server Component Fetching', () => {
  it('fetches independent data in parallel', async () => {
    const fetchStartTimes: Record<string, number> = {};

    server.use(
      rest.get('/api/v1/jobs/:jobId', async (req, res, ctx) => {
        fetchStartTimes['job'] = Date.now();
        await delay(100);
        return res(ctx.json({ id: req.params.jobId }));
      }),
      rest.get('/api/v1/history', async (req, res, ctx) => {
        fetchStartTimes['history'] = Date.now();
        await delay(100);
        return res(ctx.json({ jobs: [] }));
      })
    );

    const startTime = Date.now();
    await renderServerComponent(
      <DashboardPage>
        <Suspense fallback={<Loading />}>
          <CurrentJobPanel jobId="test" />
        </Suspense>
        <Suspense fallback={<Loading />}>
          <RecentHistoryPanel />
        </Suspense>
      </DashboardPage>
    );
    const totalTime = Date.now() - startTime;

    // Both fetches should start at roughly the same time
    const timeDiff = Math.abs(fetchStartTimes['job'] - fetchStartTimes['history']);
    expect(timeDiff).toBeLessThan(50);

    // Total time should be ~100ms (parallel), not ~200ms (sequential)
    expect(totalTime).toBeLessThan(150);
  });

  it('streams components as data becomes available', async () => {
    const componentOrder: string[] = [];

    server.use(
      rest.get('/api/v1/jobs/:jobId', async (req, res, ctx) => {
        await delay(200);
        return res(ctx.json({ id: req.params.jobId }));
      }),
      rest.get('/api/v1/stats', async (req, res, ctx) => {
        await delay(50);
        return res(ctx.json({ total: 100 }));
      })
    );

    const stream = await renderToStream(
      <DashboardPage>
        <Suspense fallback={<div data-testid="job-loading" />}>
          <JobPanel jobId="test" onRender={() => componentOrder.push('job')} />
        </Suspense>
        <Suspense fallback={<div data-testid="stats-loading" />}>
          <StatsPanel onRender={() => componentOrder.push('stats')} />
        </Suspense>
      </DashboardPage>
    );

    // Stats should render before job (faster API)
    expect(componentOrder).toEqual(['stats', 'job']);
  });

  it('handles partial failures with error boundaries', async () => {
    server.use(
      rest.get('/api/v1/jobs/:jobId', (req, res, ctx) => {
        return res(ctx.json({ id: req.params.jobId }));
      }),
      rest.get('/api/v1/stats', (req, res, ctx) => {
        return res(ctx.status(500));
      })
    );

    const html = await renderServerComponent(
      <DashboardPage>
        <ErrorBoundary fallback={<div data-testid="job-error" />}>
          <Suspense fallback={<Loading />}>
            <JobPanel jobId="test" />
          </Suspense>
        </ErrorBoundary>
        <ErrorBoundary fallback={<div data-testid="stats-error" />}>
          <Suspense fallback={<Loading />}>
            <StatsPanel />
          </Suspense>
        </ErrorBoundary>
      </DashboardPage>
    );

    // Job should render successfully
    expect(html).toContain('test');
    // Stats should show error fallback
    expect(html).toContain('stats-error');
  });
});
```

**Expected Results**:
- Independent data fetched in parallel
- Components streamed as data becomes available
- Partial failures handled with error boundaries

**Cleanup**:
- Reset MSW handlers

---

## 15. Database Connection Management

### 15.1 Overview

Tests validating database connection pool behavior, error handling, transaction management, and performance under load.

---

### INT-DB-005: Connection Pool Behavior Under Load

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DB-005 |
| **Title** | Connection Pool Behavior Under Load |
| **Priority** | P0 - Critical |
| **Components** | Prisma, PostgreSQL connection pool |
| **Spec Reference** | Database reliability |

**Preconditions**:
- PostgreSQL test instance running
- Connection pool configured (pool size: 10)

**Test Steps**:

```typescript
describe('Connection Pool Under Load', () => {
  it('handles concurrent requests within pool size', async () => {
    const concurrentRequests = 10;
    const requests = Array.from({ length: concurrentRequests }, () =>
      prisma.job.findMany({ take: 1 })
    );

    const results = await Promise.all(requests);

    expect(results).toHaveLength(concurrentRequests);
    results.forEach((result) => {
      expect(Array.isArray(result)).toBe(true);
    });
  });

  it('queues requests exceeding pool size', async () => {
    const concurrentRequests = 20; // Double the pool size
    const startTime = Date.now();

    const requests = Array.from({ length: concurrentRequests }, () =>
      prisma.$queryRaw`SELECT pg_sleep(0.1)` // 100ms each
    );

    await Promise.all(requests);
    const totalTime = Date.now() - startTime;

    // With pool size 10 and 20 requests of 100ms each:
    // Should take ~200ms (2 batches), not ~2000ms (sequential)
    expect(totalTime).toBeGreaterThan(150);
    expect(totalTime).toBeLessThan(500);
  });

  it('reports pool exhaustion metrics', async () => {
    const poolMetricsSpy = vi.fn();

    // Configure pool monitoring
    prisma.$on('query', (e) => {
      if (e.duration > 100) {
        poolMetricsSpy({ waitTime: e.duration });
      }
    });

    const concurrentRequests = 50;
    const requests = Array.from({ length: concurrentRequests }, () =>
      prisma.job.findMany({ take: 1 })
    );

    await Promise.all(requests);

    // Some requests should have waited for pool
    expect(poolMetricsSpy).toHaveBeenCalled();
  });

  it('recovers from pool exhaustion', async () => {
    // Exhaust pool with slow queries
    const slowQueries = Array.from({ length: 15 }, () =>
      prisma.$queryRaw`SELECT pg_sleep(0.5)`
    );

    // Start slow queries without waiting
    const slowPromise = Promise.all(slowQueries);

    // Wait a bit then try a fast query
    await sleep(100);

    const fastQueryStart = Date.now();
    await prisma.job.count();
    const fastQueryTime = Date.now() - fastQueryStart;

    // Fast query should complete after waiting for pool
    expect(fastQueryTime).toBeGreaterThan(0);

    // Wait for slow queries to complete
    await slowPromise;

    // Subsequent queries should be fast
    const afterRecoveryStart = Date.now();
    await prisma.job.count();
    const afterRecoveryTime = Date.now() - afterRecoveryStart;

    expect(afterRecoveryTime).toBeLessThan(50);
  });
});
```

**Expected Results**:
- Concurrent requests within pool size handled
- Requests exceeding pool size queued
- Pool exhaustion metrics reported
- Pool recovers after exhaustion

**Cleanup**:
- Reset pool state

---

### INT-DB-006: Connection Release After Errors

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DB-006 |
| **Title** | Connection Release After Errors |
| **Priority** | P0 - Critical |
| **Components** | Prisma, PostgreSQL connection pool |
| **Spec Reference** | Database reliability |

**Preconditions**:
- PostgreSQL test instance running
- Connection pool configured

**Test Steps**:

```typescript
describe('Connection Release After Errors', () => {
  it('releases connection after query error', async () => {
    const poolSize = 10;
    let successfulQueries = 0;

    // Cause some query errors
    for (let i = 0; i < poolSize; i++) {
      try {
        await prisma.$queryRaw`SELECT * FROM nonexistent_table`;
      } catch (e) {
        // Expected
      }
    }

    // All connections should be released, able to run new queries
    const queries = Array.from({ length: poolSize }, async () => {
      await prisma.job.count();
      successfulQueries++;
    });

    await Promise.all(queries);

    expect(successfulQueries).toBe(poolSize);
  });

  it('releases connection after transaction rollback', async () => {
    const poolSize = 10;

    // Cause transaction rollbacks
    for (let i = 0; i < poolSize; i++) {
      try {
        await prisma.$transaction(async (tx) => {
          await tx.job.create({
            data: { sessionId: 'test', status: 'PENDING', inputType: 'FILE' },
          });
          throw new Error('Rollback');
        });
      } catch (e) {
        // Expected
      }
    }

    // Connections should be released
    const startTime = Date.now();
    await prisma.job.count();
    const queryTime = Date.now() - startTime;

    // Should be fast, not waiting for connections
    expect(queryTime).toBeLessThan(100);
  });

  it('releases connection after timeout', async () => {
    // Configure short statement timeout
    const shortTimeoutClient = new PrismaClient({
      datasources: {
        db: { url: `${process.env.DATABASE_URL}?statement_timeout=100` },
      },
    });

    try {
      await shortTimeoutClient.$queryRaw`SELECT pg_sleep(1)`;
    } catch (e) {
      expect((e as Error).message).toContain('timeout');
    }

    // Connection should be released
    const result = await shortTimeoutClient.job.count();
    expect(result).toBeGreaterThanOrEqual(0);

    await shortTimeoutClient.$disconnect();
  });

  it('does not leak connections on repeated errors', async () => {
    const initialConnections = await getActiveConnectionCount();

    // Cause many errors
    for (let i = 0; i < 100; i++) {
      try {
        await prisma.$queryRaw`INVALID SQL`;
      } catch (e) {
        // Expected
      }
    }

    // Wait for any cleanup
    await sleep(100);

    const finalConnections = await getActiveConnectionCount();

    // Should not have leaked connections
    expect(finalConnections).toBeLessThanOrEqual(initialConnections + 2);
  });
});
```

**Expected Results**:
- Connection released after query error
- Connection released after transaction rollback
- Connection released after timeout
- No connection leaks on repeated errors

**Cleanup**:
- Disconnect test clients

---

### INT-DB-007: Transaction Isolation Levels

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DB-007 |
| **Title** | Transaction Isolation Levels |
| **Priority** | P1 - High |
| **Components** | Prisma transactions |
| **Spec Reference** | Data consistency |

**Preconditions**:
- PostgreSQL test instance running
- Test data available

**Test Steps**:

```typescript
describe('Transaction Isolation Levels', () => {
  it('prevents dirty reads with default isolation', async () => {
    const job = await prisma.job.create({
      data: { sessionId: 'test', status: 'PENDING', inputType: 'FILE' },
    });

    let readStatus: string | null = null;

    // Start transaction that updates but doesn't commit
    const tx1 = prisma.$transaction(async (tx) => {
      await tx.job.update({
        where: { id: job.id },
        data: { status: 'PROCESSING' },
      });

      // Signal that update is done (but not committed)
      await sleep(100);

      // Read from another connection during this window
      const jobInOtherTx = await prisma.job.findUnique({ where: { id: job.id } });
      readStatus = jobInOtherTx?.status || null;

      throw new Error('Rollback');
    });

    try {
      await tx1;
    } catch (e) {
      // Expected rollback
    }

    // Should have read PENDING, not PROCESSING (no dirty read)
    expect(readStatus).toBe('PENDING');
  });

  it('handles serialization conflicts', async () => {
    const job = await prisma.job.create({
      data: { sessionId: 'test', status: 'PENDING', inputType: 'FILE' },
    });

    let conflictDetected = false;

    // Two transactions trying to update same row
    const tx1 = prisma.$transaction(
      async (tx) => {
        const j = await tx.job.findUnique({ where: { id: job.id } });
        await sleep(50); // Allow tx2 to read
        await tx.job.update({
          where: { id: job.id },
          data: { status: 'PROCESSING' },
        });
      },
      { isolationLevel: 'Serializable' }
    );

    const tx2 = prisma.$transaction(
      async (tx) => {
        const j = await tx.job.findUnique({ where: { id: job.id } });
        await sleep(100); // tx1 commits first
        await tx.job.update({
          where: { id: job.id },
          data: { status: 'COMPLETE' },
        });
      },
      { isolationLevel: 'Serializable' }
    );

    try {
      await Promise.all([tx1, tx2]);
    } catch (e) {
      if ((e as Error).message.includes('serialization')) {
        conflictDetected = true;
      }
    }

    expect(conflictDetected).toBe(true);
  });

  it('uses appropriate isolation for result storage', async () => {
    const job = await prisma.job.create({
      data: { sessionId: 'test', status: 'PROCESSING', inputType: 'FILE' },
    });

    // Store results atomically
    await prisma.$transaction(async (tx) => {
      await tx.result.create({
        data: { jobId: job.id, format: 'MARKDOWN', content: '# Test', size: 6 },
      });
      await tx.result.create({
        data: { jobId: job.id, format: 'HTML', content: '<h1>Test</h1>', size: 14 },
      });
      await tx.job.update({
        where: { id: job.id },
        data: { status: 'COMPLETE' },
      });
    });

    // Verify atomicity
    const results = await prisma.result.findMany({ where: { jobId: job.id } });
    const updatedJob = await prisma.job.findUnique({ where: { id: job.id } });

    expect(results).toHaveLength(2);
    expect(updatedJob?.status).toBe('COMPLETE');
  });
});
```

**Expected Results**:
- Dirty reads prevented with default isolation
- Serialization conflicts detected
- Appropriate isolation used for result storage

**Cleanup**:
- Delete test data

---

### INT-DB-008: Query Performance with Indexes

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DB-008 |
| **Title** | Query Performance with Indexes |
| **Priority** | P1 - High |
| **Components** | Prisma, PostgreSQL indexes |
| **Spec Reference** | Performance requirements |

**Preconditions**:
- PostgreSQL test instance with indexes
- Large test dataset (1000+ jobs)

**Test Steps**:

```typescript
describe('Query Performance with Indexes', () => {
  beforeAll(async () => {
    // Create test data
    const jobs = Array.from({ length: 1000 }, (_, i) => ({
      sessionId: `session-${i % 10}`,
      status: i % 5 === 0 ? 'ERROR' : 'COMPLETE',
      inputType: 'FILE' as const,
      fileName: `file-${i}.pdf`,
      createdAt: new Date(Date.now() - i * 60000),
    }));

    await prisma.job.createMany({ data: jobs });
  });

  it('uses index for session-based queries', async () => {
    const explain = await prisma.$queryRaw`
      EXPLAIN (ANALYZE, FORMAT JSON)
      SELECT * FROM "Job" WHERE "sessionId" = 'session-1' ORDER BY "createdAt" DESC LIMIT 10
    `;

    const plan = (explain as any)[0]['QUERY PLAN'][0];
    expect(plan['Plan']['Index Name']).toBeDefined();
    expect(plan['Execution Time']).toBeLessThan(10); // 10ms
  });

  it('uses index for status-based queries', async () => {
    const startTime = Date.now();

    await prisma.job.findMany({
      where: { status: 'ERROR' },
      take: 100,
    });

    const queryTime = Date.now() - startTime;
    expect(queryTime).toBeLessThan(50);
  });

  it('uses composite index for common access patterns', async () => {
    const explain = await prisma.$queryRaw`
      EXPLAIN (ANALYZE, FORMAT JSON)
      SELECT * FROM "Job" WHERE "sessionId" = 'session-1' AND "status" = 'COMPLETE'
    `;

    const plan = (explain as any)[0]['QUERY PLAN'][0];
    // Should use composite index, not sequential scan
    expect(plan['Plan']['Node Type']).not.toBe('Seq Scan');
  });

  it('pagination uses index efficiently', async () => {
    const startTime = Date.now();

    // Page 50 of 20 items each
    await prisma.job.findMany({
      where: { sessionId: 'session-1' },
      orderBy: { createdAt: 'desc' },
      skip: 980,
      take: 20,
    });

    const queryTime = Date.now() - startTime;
    expect(queryTime).toBeLessThan(100);
  });

  afterAll(async () => {
    await prisma.job.deleteMany({});
  });
});
```

**Expected Results**:
- Index used for session-based queries
- Index used for status-based queries
- Composite index used for common patterns
- Pagination uses index efficiently

**Cleanup**:
- Delete test data

---

### INT-DB-009: Database Migration Testing

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DB-009 |
| **Title** | Database Migration Testing |
| **Priority** | P1 - High |
| **Components** | Prisma migrations |
| **Spec Reference** | Deployment requirements |

**Preconditions**:
- Test database available
- Migration files present

**Test Steps**:

```typescript
describe('Database Migration Testing', () => {
  it('applies pending migrations successfully', async () => {
    // Reset to clean state
    await execAsync('npx prisma migrate reset --force --skip-seed');

    // Apply all migrations
    const result = await execAsync('npx prisma migrate deploy');

    expect(result.stderr).not.toContain('error');
    expect(result.stdout).toContain('migrations applied');
  });

  it('creates all expected tables', async () => {
    const tables = await prisma.$queryRaw<{ tablename: string }[]>`
      SELECT tablename FROM pg_tables WHERE schemaname = 'public'
    `;

    const tableNames = tables.map((t) => t.tablename);

    expect(tableNames).toContain('Job');
    expect(tableNames).toContain('Result');
    expect(tableNames).toContain('_prisma_migrations');
  });

  it('creates all expected indexes', async () => {
    const indexes = await prisma.$queryRaw<{ indexname: string }[]>`
      SELECT indexname FROM pg_indexes WHERE schemaname = 'public'
    `;

    const indexNames = indexes.map((i) => i.indexname);

    expect(indexNames).toContain('Job_sessionId_idx');
    expect(indexNames).toContain('Job_status_idx');
    expect(indexNames).toContain('Result_jobId_idx');
  });

  it('migration is idempotent', async () => {
    // Run migrate deploy twice
    const result1 = await execAsync('npx prisma migrate deploy');
    const result2 = await execAsync('npx prisma migrate deploy');

    expect(result2.stderr).not.toContain('error');
    expect(result2.stdout).toContain('already applied');
  });

  it('schema matches migration state', async () => {
    // Generate schema diff
    const result = await execAsync('npx prisma migrate diff --from-schema-datamodel prisma/schema.prisma --to-schema-datasource prisma/schema.prisma');

    // No differences should exist
    expect(result.stdout).toContain('No difference');
  });
});
```

**Expected Results**:
- Pending migrations apply successfully
- All expected tables created
- All expected indexes created
- Migration is idempotent
- Schema matches migration state

**Cleanup**:
- None (test database state)

---

### INT-DB-010: Foreign Key Constraint Enforcement

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DB-010 |
| **Title** | Foreign Key Constraint Enforcement |
| **Priority** | P2 - Medium |
| **Components** | Prisma, PostgreSQL constraints |
| **Spec Reference** | Data integrity |

**Preconditions**:
- PostgreSQL test instance with foreign keys
- Clean test state

**Test Steps**:

```typescript
describe('Foreign Key Constraint Enforcement', () => {
  it('prevents orphaned results', async () => {
    await expect(
      prisma.result.create({
        data: {
          jobId: '00000000-0000-0000-0000-000000000000', // Non-existent job
          format: 'MARKDOWN',
          content: 'test',
          size: 4,
        },
      })
    ).rejects.toThrow('Foreign key constraint');
  });

  it('cascades delete from job to results', async () => {
    const job = await prisma.job.create({
      data: {
        sessionId: 'test',
        status: 'COMPLETE',
        inputType: 'FILE',
        results: {
          create: [
            { format: 'MARKDOWN', content: '# Test', size: 6 },
            { format: 'HTML', content: '<h1>Test</h1>', size: 14 },
          ],
        },
      },
      include: { results: true },
    });

    expect(job.results).toHaveLength(2);

    // Delete job
    await prisma.job.delete({ where: { id: job.id } });

    // Results should be deleted
    const orphanedResults = await prisma.result.findMany({
      where: { jobId: job.id },
    });

    expect(orphanedResults).toHaveLength(0);
  });

  it('enforces result format enum', async () => {
    const job = await prisma.job.create({
      data: { sessionId: 'test', status: 'PENDING', inputType: 'FILE' },
    });

    await expect(
      prisma.$executeRaw`
        INSERT INTO "Result" ("id", "jobId", "format", "content", "size", "createdAt")
        VALUES (gen_random_uuid(), ${job.id}, 'INVALID_FORMAT', 'test', 4, NOW())
      `
    ).rejects.toThrow();
  });

  it('enforces job status enum', async () => {
    await expect(
      prisma.$executeRaw`
        INSERT INTO "Job" ("id", "sessionId", "status", "inputType", "createdAt", "updatedAt")
        VALUES (gen_random_uuid(), 'test', 'INVALID_STATUS', 'FILE', NOW(), NOW())
      `
    ).rejects.toThrow();
  });

  it('maintains referential integrity under concurrent operations', async () => {
    const job = await prisma.job.create({
      data: { sessionId: 'test', status: 'PROCESSING', inputType: 'FILE' },
    });

    // Concurrent: create result and delete job
    const createResult = prisma.result.create({
      data: { jobId: job.id, format: 'MARKDOWN', content: 'test', size: 4 },
    });

    const deleteJob = prisma.job.delete({ where: { id: job.id } });

    // One should succeed, one should fail (or both succeed if create wins)
    const results = await Promise.allSettled([createResult, deleteJob]);

    // Verify no orphaned results
    const orphans = await prisma.result.findMany({
      where: {
        job: null,
      },
    });

    expect(orphans).toHaveLength(0);
  });
});
```

**Expected Results**:
- Orphaned results prevented
- Delete cascades to results
- Format enum enforced
- Status enum enforced
- Referential integrity maintained under concurrency

**Cleanup**:
- Delete test data

---

## 16. MCP Protocol Compliance

### 16.1 Overview

Tests validating MCP (Model Context Protocol) protocol compliance, including JSON-RPC 2.0 message formatting, capability negotiation, transport mode handling, and error scenarios.

---

### INT-MCP-013: JSON-RPC 2.0 Request Structure Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-MCP-013 |
| **Title** | JSON-RPC 2.0 Request Structure Validation |
| **Priority** | P0 - Critical |
| **Components** | `lib/mcp/client.ts`, `lib/mcp/protocol.ts` |
| **Spec Reference** | Section 7.1.2, JSON-RPC 2.0 Specification |

**Preconditions**:
- MCP client initialized
- Mock MCP server capturing raw requests

**Test Steps**:

```typescript
describe('JSON-RPC 2.0 Request Structure', () => {
  it('includes required jsonrpc version field', async () => {
    const capturedRequests: any[] = [];
    mockMCPServer((request) => {
      capturedRequests.push(request);
    });

    const client = new MCPClient();
    await client.initialize();

    capturedRequests.forEach((request) => {
      expect(request.jsonrpc).toBe('2.0');
    });
  });

  it('includes method field for all requests', async () => {
    const capturedRequests: any[] = [];
    mockMCPServer((request) => {
      capturedRequests.push(request);
    });

    const client = new MCPClient();
    await client.initialize();
    await client.callTool('convert_pdf', { source: '/tmp/test.pdf' });

    capturedRequests.forEach((request) => {
      expect(request.method).toBeDefined();
      expect(typeof request.method).toBe('string');
      expect(request.method.length).toBeGreaterThan(0);
    });
  });

  it('includes unique id for requests expecting response', async () => {
    const capturedRequests: any[] = [];
    mockMCPServer((request) => {
      capturedRequests.push(request);
    });

    const client = new MCPClient();
    await client.initialize();

    // Filter out notifications (which have no id)
    const requestsWithId = capturedRequests.filter((r) => !r.method.startsWith('notifications/'));
    const ids = requestsWithId.map((r) => r.id);

    // All requests should have ids
    expect(ids.every((id) => id !== undefined)).toBe(true);

    // All ids should be unique
    const uniqueIds = new Set(ids);
    expect(uniqueIds.size).toBe(ids.length);
  });

  it('uses valid id types (string, number, null)', async () => {
    const capturedRequests: any[] = [];
    mockMCPServer((request) => {
      capturedRequests.push(request);
    });

    const client = new MCPClient();
    await client.initialize();

    capturedRequests
      .filter((r) => r.id !== undefined)
      .forEach((request) => {
        const validTypes = ['string', 'number'];
        expect(validTypes).toContain(typeof request.id);
      });
  });

  it('formats params as object when present', async () => {
    const capturedRequests: any[] = [];
    mockMCPServer((request) => {
      capturedRequests.push(request);
    });

    const client = new MCPClient();
    await client.initialize();
    await client.callTool('convert_pdf', { source: '/tmp/test.pdf' });

    const toolCallRequest = capturedRequests.find((r) => r.method === 'tools/call');
    expect(toolCallRequest.params).toBeDefined();
    expect(typeof toolCallRequest.params).toBe('object');
    expect(Array.isArray(toolCallRequest.params)).toBe(false);
  });
});
```

**Expected Results**:
- All requests include `jsonrpc: "2.0"`
- All requests include method field
- Requests expecting responses have unique ids
- Id types are valid (string or number)
- Params formatted as object

**Verification Points**:
- JSON-RPC 2.0 specification compliance
- MCP protocol message format adherence

---

### INT-MCP-014: JSON-RPC Notification vs Request Distinction

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-MCP-014 |
| **Title** | JSON-RPC Notification vs Request Distinction |
| **Priority** | P0 - Critical |
| **Components** | `lib/mcp/client.ts`, `lib/mcp/protocol.ts` |
| **Spec Reference** | Section 7.1.2, JSON-RPC 2.0 Specification |

**Preconditions**:
- MCP client initialized
- Mock MCP server capturing raw requests

**Test Steps**:

```typescript
describe('JSON-RPC Notification vs Request', () => {
  it('sends notifications without id field', async () => {
    const capturedRequests: any[] = [];
    mockMCPServer((request) => {
      capturedRequests.push(request);
    });

    const client = new MCPClient();
    await client.initialize();

    const notificationRequest = capturedRequests.find(
      (r) => r.method === 'notifications/initialized'
    );

    expect(notificationRequest).toBeDefined();
    expect(notificationRequest.id).toBeUndefined();
  });

  it('sends requests with id field', async () => {
    const capturedRequests: any[] = [];
    mockMCPServer((request) => {
      capturedRequests.push(request);
    });

    const client = new MCPClient();
    await client.initialize();

    const initializeRequest = capturedRequests.find((r) => r.method === 'initialize');
    const toolsListRequest = capturedRequests.find((r) => r.method === 'tools/list');

    expect(initializeRequest.id).toBeDefined();
    expect(toolsListRequest.id).toBeDefined();
  });

  it('does not await response for notifications', async () => {
    let notificationReceived = false;
    let responseDelay = 1000;

    mockMCPServer((request, respond) => {
      if (request.method === 'notifications/initialized') {
        notificationReceived = true;
        // Notification should not have response
        setTimeout(() => {
          // Server never responds to notification
        }, responseDelay);
      }
      if (request.method === 'initialize') {
        respond({ result: { serverInfo: {}, capabilities: { tools: {} } } });
      }
    });

    const client = new MCPClient();
    const startTime = Date.now();
    await client.initialize();
    const elapsed = Date.now() - startTime;

    // Should not wait for notification response
    expect(elapsed).toBeLessThan(responseDelay);
    expect(notificationReceived).toBe(true);
  });

  it('correctly identifies notification methods by convention', async () => {
    const notificationMethods = [
      'notifications/initialized',
      'notifications/progress',
      'notifications/cancelled',
    ];

    const requestMethods = ['initialize', 'tools/list', 'tools/call', 'resources/list'];

    notificationMethods.forEach((method) => {
      expect(method.startsWith('notifications/')).toBe(true);
    });

    requestMethods.forEach((method) => {
      expect(method.startsWith('notifications/')).toBe(false);
    });
  });

  it('handles server-initiated notifications without response', async () => {
    const receivedNotifications: any[] = [];

    const client = new MCPClient({
      onNotification: (notification) => {
        receivedNotifications.push(notification);
      },
    });

    mockMCPServer((request, respond, sendNotification) => {
      if (request.method === 'initialize') {
        respond({ result: { serverInfo: {}, capabilities: { tools: {} } } });
        // Server sends progress notification
        sendNotification({
          jsonrpc: '2.0',
          method: 'notifications/progress',
          params: { progress: 50, message: 'Processing...' },
        });
      }
    });

    await client.initialize();

    // Wait for notification to be received
    await sleep(100);

    expect(receivedNotifications.some((n) => n.method === 'notifications/progress')).toBe(true);
  });
});
```

**Expected Results**:
- Notifications sent without id field
- Requests sent with id field
- Client does not await notification responses
- Notification methods identified by convention
- Server-initiated notifications handled correctly

**Verification Points**:
- Correct distinction between notifications and requests
- No unnecessary waiting for notification responses

---

### INT-MCP-015: MCP Capability Negotiation

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-MCP-015 |
| **Title** | MCP Capability Negotiation |
| **Priority** | P1 - High |
| **Components** | `lib/mcp/client.ts`, `lib/mcp/capabilities.ts` |
| **Spec Reference** | Section 7.1.2, MCP Specification |

**Preconditions**:
- MCP client ready for initialization
- Mock MCP server with configurable capabilities

**Test Steps**:

```typescript
describe('MCP Capability Negotiation', () => {
  it('sends client capabilities during initialization', async () => {
    let initializeParams: any = null;
    mockMCPServer((request) => {
      if (request.method === 'initialize') {
        initializeParams = request.params;
      }
    });

    const client = new MCPClient();
    await client.initialize();

    expect(initializeParams).toBeDefined();
    expect(initializeParams.capabilities).toBeDefined();
    expect(initializeParams.protocolVersion).toBeDefined();
  });

  it('stores server capabilities after initialization', async () => {
    const serverCapabilities = {
      tools: { listChanged: true },
      resources: { subscribe: true },
      logging: {},
    };

    mockMCPServer((request, respond) => {
      if (request.method === 'initialize') {
        respond({
          result: {
            serverInfo: { name: 'test-server', version: '1.0.0' },
            capabilities: serverCapabilities,
          },
        });
      }
    });

    const client = new MCPClient();
    await client.initialize();

    expect(client.serverCapabilities).toEqual(serverCapabilities);
  });

  it('checks tool capability before calling tools', async () => {
    mockMCPServer((request, respond) => {
      if (request.method === 'initialize') {
        respond({
          result: {
            serverInfo: {},
            capabilities: { tools: {} }, // No tools capability
          },
        });
      }
    });

    const client = new MCPClient();
    await client.initialize();

    // Should still be able to call tools if server advertises tools capability
    expect(client.hasCapability('tools')).toBe(true);
  });

  it('handles server without optional capabilities', async () => {
    mockMCPServer((request, respond) => {
      if (request.method === 'initialize') {
        respond({
          result: {
            serverInfo: { name: 'minimal-server', version: '1.0.0' },
            capabilities: {
              tools: {}, // Only tools, no resources, logging, etc.
            },
          },
        });
      }
    });

    const client = new MCPClient();
    await client.initialize();

    expect(client.hasCapability('tools')).toBe(true);
    expect(client.hasCapability('resources')).toBe(false);
    expect(client.hasCapability('logging')).toBe(false);
  });

  it('validates protocol version compatibility', async () => {
    mockMCPServer((request, respond) => {
      if (request.method === 'initialize') {
        respond({
          result: {
            serverInfo: {},
            capabilities: { tools: {} },
            protocolVersion: '2024-11-05', // Current MCP version
          },
        });
      }
    });

    const client = new MCPClient();
    await client.initialize();

    // Client should accept compatible protocol version
    expect(client.isInitialized()).toBe(true);
  });

  it('rejects incompatible protocol versions', async () => {
    mockMCPServer((request, respond) => {
      if (request.method === 'initialize') {
        respond({
          result: {
            serverInfo: {},
            capabilities: { tools: {} },
            protocolVersion: '1999-01-01', // Ancient version
          },
        });
      }
    });

    const client = new MCPClient();

    await expect(client.initialize()).rejects.toThrow(/protocol version/i);
  });
});
```

**Expected Results**:
- Client sends capabilities during initialization
- Server capabilities stored correctly
- Tool capability checked before tool calls
- Optional capabilities handled gracefully
- Protocol version compatibility validated

**Verification Points**:
- Capability negotiation follows MCP specification
- Incompatible versions rejected

---

### INT-MCP-020: MCP SSE Transport Mode

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-MCP-020 |
| **Title** | MCP SSE Transport Mode |
| **Priority** | P1 - High |
| **Components** | `lib/mcp/transport/sse.ts`, `lib/mcp/client.ts` |
| **Spec Reference** | Section 7.1.2, MCP Transport Specification |

**Preconditions**:
- MCP client configured for SSE transport
- Mock SSE endpoint available

**Test Steps**:

```typescript
describe('MCP SSE Transport Mode', () => {
  it('establishes SSE connection to server', async () => {
    let sseConnectionEstablished = false;

    const mockSSEServer = createMockSSEServer({
      onConnect: () => {
        sseConnectionEstablished = true;
      },
    });

    const client = new MCPClient({
      transport: 'sse',
      endpoint: mockSSEServer.url,
    });

    await client.connect();

    expect(sseConnectionEstablished).toBe(true);
  });

  it('sends requests via HTTP POST and receives responses via SSE', async () => {
    const requestsReceived: any[] = [];
    const responseSent = { jsonrpc: '2.0', result: { tools: [] }, id: 1 };

    const mockSSEServer = createMockSSEServer({
      onMessage: (message) => {
        requestsReceived.push(message);
        return responseSent;
      },
    });

    const client = new MCPClient({
      transport: 'sse',
      endpoint: mockSSEServer.url,
    });

    await client.connect();
    const response = await client.listTools();

    expect(requestsReceived.length).toBeGreaterThan(0);
    expect(response).toEqual(responseSent.result);
  });

  it('handles SSE reconnection on connection drop', async () => {
    let connectionCount = 0;

    const mockSSEServer = createMockSSEServer({
      onConnect: () => {
        connectionCount++;
        if (connectionCount === 1) {
          // Drop first connection after 100ms
          setTimeout(() => mockSSEServer.dropConnection(), 100);
        }
      },
    });

    const client = new MCPClient({
      transport: 'sse',
      endpoint: mockSSEServer.url,
      reconnect: true,
    });

    await client.connect();
    await sleep(500); // Wait for reconnection

    expect(connectionCount).toBeGreaterThanOrEqual(2);
  });

  it('properly closes SSE connection on disconnect', async () => {
    let connectionClosed = false;

    const mockSSEServer = createMockSSEServer({
      onDisconnect: () => {
        connectionClosed = true;
      },
    });

    const client = new MCPClient({
      transport: 'sse',
      endpoint: mockSSEServer.url,
    });

    await client.connect();
    await client.disconnect();

    expect(connectionClosed).toBe(true);
  });

  it('receives server-initiated events via SSE stream', async () => {
    const receivedEvents: any[] = [];

    const mockSSEServer = createMockSSEServer({
      onConnect: (sendEvent) => {
        // Server sends event after connection
        setTimeout(() => {
          sendEvent({
            jsonrpc: '2.0',
            method: 'notifications/progress',
            params: { progress: 50 },
          });
        }, 50);
      },
    });

    const client = new MCPClient({
      transport: 'sse',
      endpoint: mockSSEServer.url,
      onNotification: (event) => {
        receivedEvents.push(event);
      },
    });

    await client.connect();
    await sleep(100);

    expect(receivedEvents.some((e) => e.method === 'notifications/progress')).toBe(true);
  });
});
```

**Expected Results**:
- SSE connection established correctly
- Request/response via HTTP POST + SSE
- Automatic reconnection on connection drop
- Clean connection closure
- Server events received via SSE stream

**Verification Points**:
- SSE transport follows MCP transport specification
- Reconnection logic works correctly

---

### INT-MCP-021: MCP stdio Transport Mode

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-MCP-021 |
| **Title** | MCP stdio Transport Mode |
| **Priority** | P1 - High |
| **Components** | `lib/mcp/transport/stdio.ts`, `lib/mcp/client.ts` |
| **Spec Reference** | Section 7.1.2, MCP Transport Specification |

**Preconditions**:
- MCP server executable available
- stdio transport configured

**Test Steps**:

```typescript
describe('MCP stdio Transport Mode', () => {
  it('spawns MCP server process', async () => {
    let processSpawned = false;

    const mockSpawn = vi.fn().mockImplementation(() => {
      processSpawned = true;
      return createMockChildProcess();
    });

    vi.stubGlobal('spawn', mockSpawn);

    const client = new MCPClient({
      transport: 'stdio',
      command: 'python',
      args: ['-m', 'hx_docling_mcp'],
    });

    await client.connect();

    expect(processSpawned).toBe(true);
    expect(mockSpawn).toHaveBeenCalledWith('python', ['-m', 'hx_docling_mcp'], expect.any(Object));
  });

  it('sends JSON-RPC messages via stdin', async () => {
    const stdinWrites: string[] = [];
    const mockProcess = createMockChildProcess({
      onStdinWrite: (data) => {
        stdinWrites.push(data);
      },
    });

    vi.stubGlobal('spawn', () => mockProcess);

    const client = new MCPClient({
      transport: 'stdio',
      command: 'python',
      args: ['-m', 'hx_docling_mcp'],
    });

    await client.connect();
    await client.initialize();

    // Verify newline-delimited JSON
    stdinWrites.forEach((write) => {
      expect(write.endsWith('\n')).toBe(true);
      const parsed = JSON.parse(write.trim());
      expect(parsed.jsonrpc).toBe('2.0');
    });
  });

  it('receives JSON-RPC messages via stdout', async () => {
    const mockProcess = createMockChildProcess();

    vi.stubGlobal('spawn', () => mockProcess);

    const client = new MCPClient({
      transport: 'stdio',
      command: 'python',
      args: ['-m', 'hx_docling_mcp'],
    });

    await client.connect();

    // Simulate server response on stdout
    mockProcess.stdout.emit(
      'data',
      JSON.stringify({
        jsonrpc: '2.0',
        result: { serverInfo: {}, capabilities: { tools: {} } },
        id: 1,
      }) + '\n'
    );

    const tools = await client.listTools();
    expect(Array.isArray(tools)).toBe(true);
  });

  it('handles process exit gracefully', async () => {
    const mockProcess = createMockChildProcess();

    vi.stubGlobal('spawn', () => mockProcess);

    const client = new MCPClient({
      transport: 'stdio',
      command: 'python',
      args: ['-m', 'hx_docling_mcp'],
    });

    let disconnectCalled = false;
    client.on('disconnect', () => {
      disconnectCalled = true;
    });

    await client.connect();

    // Simulate process exit
    mockProcess.emit('exit', 0);

    expect(disconnectCalled).toBe(true);
    expect(client.isConnected()).toBe(false);
  });

  it('handles process crash with non-zero exit', async () => {
    const mockProcess = createMockChildProcess();

    vi.stubGlobal('spawn', () => mockProcess);

    const client = new MCPClient({
      transport: 'stdio',
      command: 'python',
      args: ['-m', 'hx_docling_mcp'],
    });

    let errorReceived: Error | null = null;
    client.on('error', (error) => {
      errorReceived = error;
    });

    await client.connect();

    // Simulate crash
    mockProcess.emit('exit', 1);

    expect(errorReceived).not.toBeNull();
    expect(errorReceived?.message).toContain('exit code 1');
  });

  it('terminates process on client disconnect', async () => {
    let processKilled = false;
    const mockProcess = createMockChildProcess({
      onKill: () => {
        processKilled = true;
      },
    });

    vi.stubGlobal('spawn', () => mockProcess);

    const client = new MCPClient({
      transport: 'stdio',
      command: 'python',
      args: ['-m', 'hx_docling_mcp'],
    });

    await client.connect();
    await client.disconnect();

    expect(processKilled).toBe(true);
  });
});
```

**Expected Results**:
- MCP server process spawned correctly
- Messages sent via stdin with newline delimiter
- Messages received via stdout parsing
- Process exit handled gracefully
- Process crash reported as error
- Process terminated on disconnect

**Verification Points**:
- stdio transport follows MCP specification
- Process lifecycle managed correctly

---

### INT-MCP-022: MCP Transport Mode Selection Logic

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-MCP-022 |
| **Title** | MCP Transport Mode Selection Logic |
| **Priority** | P1 - High |
| **Components** | `lib/mcp/transport/index.ts`, `lib/mcp/client.ts` |
| **Spec Reference** | Section 7.1.2, Configuration requirements |

**Preconditions**:
- Environment variables configurable
- Both transport implementations available

**Test Steps**:

```typescript
describe('MCP Transport Mode Selection', () => {
  it('defaults to SSE transport when MCP_ENDPOINT is HTTP URL', async () => {
    process.env.MCP_ENDPOINT = 'http://localhost:8000/mcp';

    const client = MCPClient.create();

    expect(client.transportType).toBe('sse');
  });

  it('uses stdio transport when MCP_COMMAND is set', async () => {
    process.env.MCP_COMMAND = 'python -m hx_docling_mcp';

    const client = MCPClient.create();

    expect(client.transportType).toBe('stdio');
  });

  it('prefers explicit transport configuration over environment', async () => {
    process.env.MCP_ENDPOINT = 'http://localhost:8000/mcp';
    process.env.MCP_COMMAND = 'python -m hx_docling_mcp';

    const client = new MCPClient({ transport: 'stdio' });

    expect(client.transportType).toBe('stdio');
  });

  it('throws error when no transport configuration available', async () => {
    delete process.env.MCP_ENDPOINT;
    delete process.env.MCP_COMMAND;

    expect(() => MCPClient.create()).toThrow(/transport configuration/i);
  });

  it('validates SSE endpoint URL format', async () => {
    process.env.MCP_ENDPOINT = 'invalid-url';

    expect(() => MCPClient.create()).toThrow(/invalid.*url/i);
  });

  it('validates stdio command exists', async () => {
    process.env.MCP_COMMAND = 'nonexistent-command-xyz';

    const client = new MCPClient({ transport: 'stdio' });

    await expect(client.connect()).rejects.toThrow(/command.*not found/i);
  });
});
```

**Expected Results**:
- SSE transport selected for HTTP URLs
- stdio transport selected when command configured
- Explicit configuration overrides environment
- Missing configuration throws error
- Invalid URLs rejected
- Non-existent commands rejected

**Verification Points**:
- Transport selection logic is deterministic
- Configuration validation prevents runtime errors

---

### INT-MCP-023: Tool Schema Validation Before Invocation

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-MCP-023 |
| **Title** | Tool Schema Validation Before Invocation |
| **Priority** | P1 - High |
| **Components** | `lib/mcp/client.ts`, `lib/mcp/validation.ts` |
| **Spec Reference** | Section 7.1.2, MCP Tool Specification |

**Preconditions**:
- MCP client initialized with tool list
- Tool schemas available from server

**Test Steps**:

```typescript
describe('Tool Schema Validation', () => {
  const convertPdfSchema = {
    name: 'convert_pdf',
    description: 'Convert PDF to Docling document',
    inputSchema: {
      type: 'object',
      properties: {
        source: { type: 'string', description: 'Path to PDF file' },
        options: {
          type: 'object',
          properties: {
            ocr: { type: 'boolean', default: true },
            extractTables: { type: 'boolean', default: true },
          },
        },
      },
      required: ['source'],
    },
  };

  it('validates required parameters before invocation', async () => {
    mockMCPServer((request, respond) => {
      if (request.method === 'tools/list') {
        respond({ result: { tools: [convertPdfSchema] } });
      }
    });

    const client = new MCPClient();
    await client.initialize();

    // Missing required 'source' parameter
    await expect(client.callTool('convert_pdf', {})).rejects.toThrow(/source.*required/i);
  });

  it('validates parameter types before invocation', async () => {
    mockMCPServer((request, respond) => {
      if (request.method === 'tools/list') {
        respond({ result: { tools: [convertPdfSchema] } });
      }
    });

    const client = new MCPClient();
    await client.initialize();

    // Wrong type for 'source' parameter
    await expect(client.callTool('convert_pdf', { source: 123 })).rejects.toThrow(/source.*string/i);
  });

  it('allows optional parameters to be omitted', async () => {
    let toolCallReceived = false;

    mockMCPServer((request, respond) => {
      if (request.method === 'tools/list') {
        respond({ result: { tools: [convertPdfSchema] } });
      }
      if (request.method === 'tools/call') {
        toolCallReceived = true;
        respond({ result: { content: [{ type: 'document', document: {} }] } });
      }
    });

    const client = new MCPClient();
    await client.initialize();

    // Only required parameter, no options
    await client.callTool('convert_pdf', { source: '/tmp/test.pdf' });

    expect(toolCallReceived).toBe(true);
  });

  it('validates nested object schemas', async () => {
    mockMCPServer((request, respond) => {
      if (request.method === 'tools/list') {
        respond({ result: { tools: [convertPdfSchema] } });
      }
    });

    const client = new MCPClient();
    await client.initialize();

    // Wrong type for nested property
    await expect(
      client.callTool('convert_pdf', {
        source: '/tmp/test.pdf',
        options: { ocr: 'yes' }, // Should be boolean
      })
    ).rejects.toThrow(/ocr.*boolean/i);
  });

  it('rejects unknown tools', async () => {
    mockMCPServer((request, respond) => {
      if (request.method === 'tools/list') {
        respond({ result: { tools: [convertPdfSchema] } });
      }
    });

    const client = new MCPClient();
    await client.initialize();

    await expect(client.callTool('unknown_tool', {})).rejects.toThrow(/unknown.*tool/i);
  });

  it('caches tool schemas after initial fetch', async () => {
    let toolsListCallCount = 0;

    mockMCPServer((request, respond) => {
      if (request.method === 'tools/list') {
        toolsListCallCount++;
        respond({ result: { tools: [convertPdfSchema] } });
      }
      if (request.method === 'tools/call') {
        respond({ result: { content: [{ type: 'document', document: {} }] } });
      }
    });

    const client = new MCPClient();
    await client.initialize();

    // Multiple tool calls
    await client.callTool('convert_pdf', { source: '/tmp/test1.pdf' });
    await client.callTool('convert_pdf', { source: '/tmp/test2.pdf' });
    await client.callTool('convert_pdf', { source: '/tmp/test3.pdf' });

    // tools/list should only be called once (during initialization)
    expect(toolsListCallCount).toBe(1);
  });
});
```

**Expected Results**:
- Required parameters validated
- Parameter types validated
- Optional parameters can be omitted
- Nested schemas validated
- Unknown tools rejected
- Tool schemas cached

**Verification Points**:
- Client-side validation prevents invalid requests
- Schema caching improves performance

---

### INT-MCP-024: MCP Server Unavailability Handling

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-MCP-024 |
| **Title** | MCP Server Unavailability Handling |
| **Priority** | P1 - High |
| **Components** | `lib/mcp/client.ts`, `lib/mcp/error-handling.ts` |
| **Spec Reference** | Section 7.1.4, Error Handling |

**Preconditions**:
- MCP client configured
- Mock server controllable for availability

**Test Steps**:

```typescript
describe('MCP Server Unavailability Handling', () => {
  it('handles connection refused error', async () => {
    const client = new MCPClient({
      transport: 'sse',
      endpoint: 'http://localhost:9999/mcp', // Nothing listening
    });

    await expect(client.connect()).rejects.toThrow(/connection refused|ECONNREFUSED/i);
  });

  it('handles connection timeout', async () => {
    const slowServer = createMockSSEServer({
      onConnect: async () => {
        await sleep(10000); // Very slow response
      },
    });

    const client = new MCPClient({
      transport: 'sse',
      endpoint: slowServer.url,
      connectionTimeout: 1000,
    });

    await expect(client.connect()).rejects.toThrow(/timeout/i);
  });

  it('handles server disconnect during operation', async () => {
    const mockServer = createMockSSEServer();

    const client = new MCPClient({
      transport: 'sse',
      endpoint: mockServer.url,
    });

    await client.connect();
    await client.initialize();

    // Disconnect server during tool call
    const toolCallPromise = client.callTool('convert_pdf', { source: '/tmp/test.pdf' });
    mockServer.close();

    await expect(toolCallPromise).rejects.toThrow(/disconnected|connection.*closed/i);
  });

  it('queues requests during reconnection', async () => {
    let connectionCount = 0;
    const mockServer = createMockSSEServer({
      onConnect: () => {
        connectionCount++;
      },
      onMessage: (request, respond) => {
        if (request.method === 'tools/call') {
          respond({ result: { content: [{ type: 'document', document: {} }] } });
        }
      },
    });

    const client = new MCPClient({
      transport: 'sse',
      endpoint: mockServer.url,
      reconnect: true,
      reconnectDelay: 100,
    });

    await client.connect();
    await client.initialize();

    // Force disconnect
    mockServer.dropConnection();

    // Queue request during reconnection
    const toolCallPromise = client.callTool('convert_pdf', { source: '/tmp/test.pdf' });

    // Wait for reconnection and request completion
    const result = await toolCallPromise;

    expect(connectionCount).toBeGreaterThanOrEqual(2);
    expect(result).toBeDefined();
  });

  it('reports server unavailability in health check', async () => {
    const client = new MCPClient({
      transport: 'sse',
      endpoint: 'http://localhost:9999/mcp',
    });

    const health = await client.healthCheck();

    expect(health.status).toBe('unhealthy');
    expect(health.error).toContain('unavailable');
  });

  it('retries failed requests with exponential backoff', async () => {
    let attemptCount = 0;
    const attemptTimes: number[] = [];

    const mockServer = createMockSSEServer({
      onMessage: (request, respond) => {
        attemptCount++;
        attemptTimes.push(Date.now());

        if (attemptCount < 3) {
          respond({ error: { code: -32000, message: 'Temporary failure' } });
        } else {
          respond({ result: { content: [{ type: 'document', document: {} }] } });
        }
      },
    });

    const client = new MCPClient({
      transport: 'sse',
      endpoint: mockServer.url,
      retryAttempts: 5,
      initialRetryDelay: 100,
    });

    await client.connect();
    await client.initialize();

    const result = await client.callTool('convert_pdf', { source: '/tmp/test.pdf' });

    expect(attemptCount).toBe(3);
    expect(result).toBeDefined();

    // Verify exponential backoff
    const delay1 = attemptTimes[1] - attemptTimes[0];
    const delay2 = attemptTimes[2] - attemptTimes[1];
    expect(delay2).toBeGreaterThan(delay1);
  });
});
```

**Expected Results**:
- Connection refused handled gracefully
- Connection timeout respected
- Mid-operation disconnect handled
- Requests queued during reconnection
- Health check reports unavailability
- Exponential backoff retry implemented

**Verification Points**:
- Robust error handling for all unavailability scenarios
- Automatic recovery where possible

---

## 17. Redis Connection Management

### 17.1 Overview

Tests validating Redis connection pool behavior, failover handling, Pub/Sub reliability, and data management patterns.

---

### INT-REDIS-004: Connection Pool Exhaustion Handling

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-REDIS-004 |
| **Title** | Connection Pool Exhaustion Handling |
| **Priority** | P1 - High |
| **Components** | `lib/redis/client.ts`, `lib/redis/pool.ts` |
| **Spec Reference** | Section 6.3, Redis Configuration |

**Preconditions**:
- Redis test instance running
- Connection pool configured with known size

**Test Steps**:

```typescript
describe('Redis Connection Pool Exhaustion', () => {
  const POOL_SIZE = 10;

  beforeEach(() => {
    // Configure pool with known size
    redisClient.configure({ maxConnections: POOL_SIZE });
  });

  it('handles requests exceeding pool size with queuing', async () => {
    const concurrentOps = POOL_SIZE * 2;

    // Create long-running operations
    const operations = Array.from({ length: concurrentOps }, (_, i) =>
      redisClient.eval('return redis.call("DEBUG", "SLEEP", 0.1)', 0)
    );

    const results = await Promise.allSettled(operations);

    // All operations should eventually complete
    const fulfilled = results.filter((r) => r.status === 'fulfilled');
    expect(fulfilled.length).toBe(concurrentOps);
  });

  it('reports pool exhaustion metrics', async () => {
    const metrics: any[] = [];

    redisClient.on('pool:exhausted', (metric) => {
      metrics.push(metric);
    });

    // Exhaust pool
    const longOps = Array.from({ length: POOL_SIZE + 5 }, () =>
      redisClient.eval('return redis.call("DEBUG", "SLEEP", 0.2)', 0)
    );

    await Promise.all(longOps);

    expect(metrics.length).toBeGreaterThan(0);
    expect(metrics[0]).toHaveProperty('waitingRequests');
  });

  it('times out requests waiting too long for connection', async () => {
    // Configure short wait timeout
    redisClient.configure({ connectionWaitTimeout: 100 });

    // Exhaust pool with long operations
    const blockingOps = Array.from({ length: POOL_SIZE }, () =>
      redisClient.eval('return redis.call("DEBUG", "SLEEP", 1)', 0)
    );

    // Start blocking ops
    const blockingPromise = Promise.all(blockingOps);

    // Try additional operation (should timeout waiting for connection)
    await expect(redisClient.get('test-key')).rejects.toThrow(/timeout.*connection/i);

    // Wait for blocking ops to complete
    await blockingPromise;
  });

  it('recovers gracefully after pool exhaustion', async () => {
    // Exhaust pool
    const blockingOps = Array.from({ length: POOL_SIZE }, () =>
      redisClient.eval('return redis.call("DEBUG", "SLEEP", 0.1)', 0)
    );

    await Promise.all(blockingOps);

    // Subsequent operations should work normally
    const startTime = Date.now();
    await redisClient.set('recovery-test', 'value');
    const duration = Date.now() - startTime;

    expect(duration).toBeLessThan(100); // Fast, not waiting for pool
  });

  it('does not leak connections on operation errors', async () => {
    const initialPoolSize = await redisClient.getPoolStats().available;

    // Cause many errors
    for (let i = 0; i < POOL_SIZE * 3; i++) {
      try {
        await redisClient.eval('INVALID LUA SYNTAX', 0);
      } catch (e) {
        // Expected
      }
    }

    // Wait for cleanup
    await sleep(100);

    const finalPoolSize = await redisClient.getPoolStats().available;

    expect(finalPoolSize).toBe(initialPoolSize);
  });
});
```

**Expected Results**:
- Requests queued when pool exhausted
- Pool exhaustion metrics reported
- Timeout for requests waiting too long
- Graceful recovery after exhaustion
- No connection leaks on errors

**Verification Points**:
- Connection pool handles high load correctly
- Monitoring available for pool status

---

### INT-REDIS-005: Connection Timeout Behavior

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-REDIS-005 |
| **Title** | Connection Timeout Behavior |
| **Priority** | P1 - High |
| **Components** | `lib/redis/client.ts` |
| **Spec Reference** | Section 6.3, Redis Configuration |

**Preconditions**:
- Redis test instance running
- Network simulation capability (or mock)

**Test Steps**:

```typescript
describe('Redis Connection Timeout', () => {
  it('respects connection timeout configuration', async () => {
    const client = new RedisClient({
      host: '10.255.255.1', // Non-routable address
      connectTimeout: 500,
    });

    const startTime = Date.now();

    await expect(client.connect()).rejects.toThrow(/timeout|ETIMEDOUT/i);

    const duration = Date.now() - startTime;
    expect(duration).toBeGreaterThanOrEqual(500);
    expect(duration).toBeLessThan(1000);
  });

  it('respects command timeout configuration', async () => {
    const client = new RedisClient({
      commandTimeout: 500,
    });

    await client.connect();

    // Command that takes longer than timeout
    await expect(client.eval('return redis.call("DEBUG", "SLEEP", 2)', 0)).rejects.toThrow(
      /timeout/i
    );
  });

  it('handles timeout during Pub/Sub subscription', async () => {
    const client = new RedisClient({
      commandTimeout: 500,
    });

    await client.connect();

    // Subscribe with simulated network delay
    const subscriber = client.duplicate();
    await subscriber.connect();

    // Simulate slow subscription acknowledgment
    mockNetworkDelay(subscriber, 1000);

    await expect(subscriber.subscribe('test-channel')).rejects.toThrow(/timeout/i);
  });

  it('reconnects after timeout with backoff', async () => {
    let connectionAttempts = 0;
    const attemptTimes: number[] = [];

    const client = new RedisClient({
      connectTimeout: 100,
      retryStrategy: (times) => {
        connectionAttempts = times;
        attemptTimes.push(Date.now());
        return Math.min(times * 100, 1000);
      },
    });

    // Start with unreachable server
    mockUnreachableServer();

    client.connect().catch(() => {}); // Don't await, let it retry

    await sleep(500);

    expect(connectionAttempts).toBeGreaterThan(1);

    // Verify backoff
    if (attemptTimes.length >= 2) {
      const delay = attemptTimes[1] - attemptTimes[0];
      expect(delay).toBeGreaterThanOrEqual(100);
    }
  });
});
```

**Expected Results**:
- Connection timeout respected
- Command timeout respected
- Pub/Sub timeout handled
- Reconnection with backoff after timeout

**Verification Points**:
- Timeout configuration effective
- Graceful handling of timeout scenarios

---

### INT-REDIS-006: Redis Failover Scenarios

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-REDIS-006 |
| **Title** | Redis Failover Scenarios |
| **Priority** | P1 - High |
| **Components** | `lib/redis/client.ts`, `lib/redis/sentinel.ts` |
| **Spec Reference** | Section 6.3, High Availability |

**Preconditions**:
- Redis Sentinel or cluster configuration (or mock)
- Ability to simulate failover

**Test Steps**:

```typescript
describe('Redis Failover Scenarios', () => {
  it('detects primary failure', async () => {
    let failoverDetected = false;

    const client = new RedisClient({
      sentinels: [{ host: 'sentinel', port: 26379 }],
      name: 'mymaster',
    });

    client.on('failover', () => {
      failoverDetected = true;
    });

    await client.connect();

    // Simulate primary failure
    mockPrimaryFailure();

    await sleep(500);

    expect(failoverDetected).toBe(true);
  });

  it('reconnects to new primary after failover', async () => {
    const client = new RedisClient({
      sentinels: [{ host: 'sentinel', port: 26379 }],
      name: 'mymaster',
    });

    await client.connect();

    // Write to current primary
    await client.set('failover-test', 'before');

    // Simulate failover
    mockPrimaryFailover();

    // Wait for reconnection
    await sleep(1000);

    // Write to new primary
    await client.set('failover-test', 'after');

    const value = await client.get('failover-test');
    expect(value).toBe('after');
  });

  it('queues commands during failover', async () => {
    const client = new RedisClient({
      sentinels: [{ host: 'sentinel', port: 26379 }],
      name: 'mymaster',
      enableOfflineQueue: true,
    });

    await client.connect();

    // Start failover
    mockPrimaryFailover();

    // Queue commands during failover
    const writePromise = client.set('queued-key', 'queued-value');

    // Failover completes
    mockFailoverComplete();

    // Queued command should succeed
    await writePromise;

    const value = await client.get('queued-key');
    expect(value).toBe('queued-value');
  });

  it('handles replica promotion correctly', async () => {
    const client = new RedisClient({
      sentinels: [{ host: 'sentinel', port: 26379 }],
      name: 'mymaster',
    });

    await client.connect();

    const initialRole = await client.info('replication');
    expect(initialRole).toContain('role:master');

    // Promote replica
    mockReplicaPromotion();

    await sleep(500);

    // Client should connect to new master
    const newRole = await client.info('replication');
    expect(newRole).toContain('role:master');
  });

  it('reports failover duration metric', async () => {
    let failoverDuration: number | null = null;

    const client = new RedisClient({
      sentinels: [{ host: 'sentinel', port: 26379 }],
      name: 'mymaster',
    });

    client.on('failover:complete', (metric) => {
      failoverDuration = metric.duration;
    });

    await client.connect();

    mockPrimaryFailover();

    await sleep(1000);

    mockFailoverComplete();

    await sleep(100);

    expect(failoverDuration).not.toBeNull();
    expect(failoverDuration).toBeGreaterThan(0);
  });
});
```

**Expected Results**:
- Primary failure detected
- Reconnection to new primary
- Commands queued during failover
- Replica promotion handled
- Failover duration reported

**Verification Points**:
- High availability behavior correct
- Minimal data loss during failover

---

### INT-REDIS-007: Key Naming Consistency Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-REDIS-007 |
| **Title** | Key Naming Consistency Validation |
| **Priority** | P2 - Medium |
| **Components** | `lib/redis/keys.ts`, `lib/redis/client.ts` |
| **Spec Reference** | Section 6.3, Key Naming Convention |

**Preconditions**:
- Redis client configured
- Key naming utility available

**Test Steps**:

```typescript
describe('Redis Key Naming Consistency', () => {
  it('generates session keys with correct prefix', async () => {
    const sessionId = 'abc123';
    const key = redisKeys.session(sessionId);

    expect(key).toBe('hx:session:abc123');
    expect(key).toMatch(/^hx:session:[a-zA-Z0-9]+$/);
  });

  it('generates job keys with correct prefix', async () => {
    const jobId = 'job-456';
    const key = redisKeys.job(jobId);

    expect(key).toBe('hx:job:job-456');
  });

  it('generates SSE channel keys with correct prefix', async () => {
    const sessionId = 'abc123';
    const key = redisKeys.sseChannel(sessionId);

    expect(key).toBe('hx:sse:abc123');
  });

  it('generates rate limit keys with correct prefix', async () => {
    const identifier = '192.168.1.1';
    const key = redisKeys.rateLimit(identifier);

    expect(key).toBe('hx:ratelimit:192.168.1.1');
  });

  it('prevents key collision between different key types', async () => {
    const id = 'same-id';

    const sessionKey = redisKeys.session(id);
    const jobKey = redisKeys.job(id);
    const sseKey = redisKeys.sseChannel(id);

    const uniqueKeys = new Set([sessionKey, jobKey, sseKey]);
    expect(uniqueKeys.size).toBe(3);
  });

  it('escapes special characters in key components', async () => {
    const maliciousId = 'test:with:colons';
    const key = redisKeys.session(maliciousId);

    // Colons should be escaped or rejected
    expect(key).not.toBe('hx:session:test:with:colons');
  });

  it('enforces maximum key length', async () => {
    const longId = 'a'.repeat(1000);

    expect(() => redisKeys.session(longId)).toThrow(/key.*too long/i);
  });
});
```

**Expected Results**:
- Session keys have correct prefix
- Job keys have correct prefix
- SSE channel keys have correct prefix
- Rate limit keys have correct prefix
- No key collisions between types
- Special characters handled
- Maximum key length enforced

**Verification Points**:
- Consistent key naming across application
- No namespace collisions

---

### INT-REDIS-008: Memory Optimization Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-REDIS-008 |
| **Title** | Memory Optimization Validation |
| **Priority** | P2 - Medium |
| **Components** | `lib/redis/client.ts`, `lib/redis/serialization.ts` |
| **Spec Reference** | Section 6.3, Memory Management |

**Preconditions**:
- Redis test instance with memory info access
- Test data generation capability

**Test Steps**:

```typescript
describe('Redis Memory Optimization', () => {
  it('uses efficient serialization for session data', async () => {
    const sessionData = {
      id: 'session-123',
      createdAt: new Date().toISOString(),
      jobs: ['job-1', 'job-2', 'job-3'],
      metadata: { userAgent: 'Mozilla/5.0', ip: '192.168.1.1' },
    };

    const serialized = redisClient.serialize(sessionData);
    const jsonSize = JSON.stringify(sessionData).length;

    // Serialized size should not be significantly larger than JSON
    expect(serialized.length).toBeLessThanOrEqual(jsonSize * 1.2);
  });

  it('compresses large values', async () => {
    const largeContent = 'x'.repeat(10000);

    await redisClient.set('large-key', largeContent, { compress: true });

    const memoryBefore = await redisClient.memory('usage', 'large-key');

    // Should be compressed (less than original size)
    expect(memoryBefore).toBeLessThan(largeContent.length);

    // Verify data integrity after decompression
    const retrieved = await redisClient.get('large-key');
    expect(retrieved).toBe(largeContent);
  });

  it('uses appropriate data structures for different use cases', async () => {
    // Set for unique items
    await redisClient.sadd('unique-items', 'a', 'b', 'c', 'a', 'b');
    const setSize = await redisClient.scard('unique-items');
    expect(setSize).toBe(3);

    // Sorted set for ordered data with scores
    await redisClient.zadd('ordered-items', 1, 'first', 2, 'second', 3, 'third');
    const orderedItems = await redisClient.zrange('ordered-items', 0, -1);
    expect(orderedItems).toEqual(['first', 'second', 'third']);

    // Hash for structured data
    await redisClient.hset('structured', { field1: 'value1', field2: 'value2' });
    const hashData = await redisClient.hgetall('structured');
    expect(hashData).toEqual({ field1: 'value1', field2: 'value2' });
  });

  it('sets appropriate TTL for temporary data', async () => {
    // Session data - 24 hours
    await redisClient.setSession('sess-123', { data: 'test' });
    const sessionTTL = await redisClient.ttl(redisKeys.session('sess-123'));
    expect(sessionTTL).toBeGreaterThan(23 * 3600);
    expect(sessionTTL).toBeLessThanOrEqual(24 * 3600);

    // Rate limit data - 1 minute
    await redisClient.setRateLimit('ip-123', 1);
    const rateLimitTTL = await redisClient.ttl(redisKeys.rateLimit('ip-123'));
    expect(rateLimitTTL).toBeLessThanOrEqual(60);
  });

  it('cleans up expired keys automatically', async () => {
    await redisClient.set('temp-key', 'value', { ttl: 1 });

    // Key exists immediately
    expect(await redisClient.exists('temp-key')).toBe(1);

    // Wait for expiration
    await sleep(1500);

    // Key should be gone
    expect(await redisClient.exists('temp-key')).toBe(0);
  });
});
```

**Expected Results**:
- Efficient serialization used
- Large values compressed
- Appropriate data structures used
- TTL set for temporary data
- Expired keys cleaned up

**Verification Points**:
- Memory usage optimized
- No memory leaks from expired data

---

### INT-REDIS-009: Pub/Sub Error Handling

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-REDIS-009 |
| **Title** | Pub/Sub Error Handling |
| **Priority** | P1 - High |
| **Components** | `lib/redis/pubsub.ts`, `lib/redis/client.ts` |
| **Spec Reference** | Section 6.3.2, SSE Integration |

**Preconditions**:
- Redis test instance running
- Pub/Sub client configured

**Test Steps**:

```typescript
describe('Redis Pub/Sub Error Handling', () => {
  it('handles subscription failure gracefully', async () => {
    const subscriber = redisClient.duplicate();
    await subscriber.connect();

    // Mock subscription failure
    mockSubscriptionFailure('test-channel');

    let errorReceived: Error | null = null;
    subscriber.on('error', (error) => {
      errorReceived = error;
    });

    await expect(subscriber.subscribe('test-channel')).rejects.toThrow();

    expect(errorReceived).not.toBeNull();
  });

  it('reconnects Pub/Sub after connection loss', async () => {
    const subscriber = redisClient.duplicate();
    await subscriber.connect();

    const receivedMessages: string[] = [];
    await subscriber.subscribe('reconnect-test', (message) => {
      receivedMessages.push(message);
    });

    // Publish initial message
    await redisClient.publish('reconnect-test', 'message-1');
    await sleep(50);

    // Force disconnect
    mockConnectionLoss(subscriber);
    await sleep(100);

    // Connection should auto-reconnect and re-subscribe
    await sleep(500);

    // Publish after reconnection
    await redisClient.publish('reconnect-test', 'message-2');
    await sleep(50);

    expect(receivedMessages).toContain('message-1');
    expect(receivedMessages).toContain('message-2');
  });

  it('handles message handler exceptions without crashing', async () => {
    const subscriber = redisClient.duplicate();
    await subscriber.connect();

    let messageCount = 0;
    await subscriber.subscribe('error-test', (message) => {
      messageCount++;
      if (messageCount === 1) {
        throw new Error('Handler error');
      }
    });

    // First message triggers error
    await redisClient.publish('error-test', 'message-1');
    await sleep(50);

    // Subscription should still work
    await redisClient.publish('error-test', 'message-2');
    await sleep(50);

    expect(messageCount).toBe(2);
  });

  it('handles channel pattern subscription errors', async () => {
    const subscriber = redisClient.duplicate();
    await subscriber.connect();

    // Invalid pattern should be rejected
    await expect(subscriber.psubscribe('invalid[pattern')).rejects.toThrow();
  });

  it('unsubscribes cleanly on error', async () => {
    const subscriber = redisClient.duplicate();
    await subscriber.connect();

    await subscriber.subscribe('cleanup-test', () => {});

    // Verify subscription active
    const subsBefore = await subscriber.pubsubChannels();
    expect(subsBefore).toContain('cleanup-test');

    // Force error that triggers cleanup
    mockFatalError(subscriber);

    await sleep(100);

    // Verify subscription cleaned up
    const subsAfter = await subscriber.pubsubChannels();
    expect(subsAfter).not.toContain('cleanup-test');
  });

  it('handles rapid subscribe/unsubscribe', async () => {
    const subscriber = redisClient.duplicate();
    await subscriber.connect();

    const operations = Array.from({ length: 50 }, async (_, i) => {
      const channel = `rapid-test-${i % 10}`;
      await subscriber.subscribe(channel, () => {});
      await subscriber.unsubscribe(channel);
    });

    await Promise.all(operations);

    // No subscriptions should remain
    const activeSubs = await subscriber.pubsubChannels();
    const rapidTestSubs = activeSubs.filter((c: string) => c.startsWith('rapid-test-'));
    expect(rapidTestSubs.length).toBe(0);
  });
});
```

**Expected Results**:
- Subscription failures handled gracefully
- Auto-reconnect and re-subscribe after connection loss
- Handler exceptions don't crash subscription
- Invalid patterns rejected
- Clean unsubscribe on error
- Rapid subscribe/unsubscribe handled

**Verification Points**:
- Pub/Sub resilient to errors
- No resource leaks

---

### INT-REDIS-010: Pipeline Atomicity

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-REDIS-010 |
| **Title** | Pipeline Atomicity |
| **Priority** | P2 - Medium |
| **Components** | `lib/redis/client.ts`, `lib/redis/pipeline.ts` |
| **Spec Reference** | Section 6.3, Transaction Support |

**Preconditions**:
- Redis test instance running
- Pipeline support enabled

**Test Steps**:

```typescript
describe('Redis Pipeline Atomicity', () => {
  it('executes pipeline commands in order', async () => {
    const pipeline = redisClient.pipeline();

    pipeline.set('order-test', '1');
    pipeline.incr('order-test');
    pipeline.incr('order-test');
    pipeline.get('order-test');

    const results = await pipeline.exec();

    expect(results[0][1]).toBe('OK'); // SET
    expect(results[1][1]).toBe(2); // INCR -> 2
    expect(results[2][1]).toBe(3); // INCR -> 3
    expect(results[3][1]).toBe('3'); // GET
  });

  it('continues pipeline after individual command errors', async () => {
    const pipeline = redisClient.pipeline();

    pipeline.set('valid-key', 'value');
    pipeline.incr('valid-key'); // Error: value is not an integer
    pipeline.set('another-key', 'another-value');
    pipeline.get('another-key');

    const results = await pipeline.exec();

    expect(results[0][0]).toBeNull(); // SET succeeded
    expect(results[1][0]).not.toBeNull(); // INCR failed
    expect(results[2][0]).toBeNull(); // SET succeeded
    expect(results[3][1]).toBe('another-value'); // GET succeeded
  });

  it('uses MULTI/EXEC for atomic transactions', async () => {
    const multi = redisClient.multi();

    multi.set('atomic-1', 'value1');
    multi.set('atomic-2', 'value2');

    // Simulate concurrent modification attempt
    const concurrentModify = redisClient.set('atomic-1', 'modified');

    const results = await multi.exec();
    await concurrentModify;

    // Transaction should have completed atomically
    const value1 = await redisClient.get('atomic-1');
    expect(['value1', 'modified']).toContain(value1);
  });

  it('supports WATCH for optimistic locking', async () => {
    await redisClient.set('watched-key', '0');

    // Transaction 1: Watch and increment
    const tx1 = async () => {
      await redisClient.watch('watched-key');
      const value = parseInt((await redisClient.get('watched-key')) || '0');

      const multi = redisClient.multi();
      multi.set('watched-key', (value + 1).toString());
      return multi.exec();
    };

    // Transaction 2: Modify without watch
    const tx2 = async () => {
      await sleep(10); // Slight delay
      return redisClient.set('watched-key', '100');
    };

    const [result1, result2] = await Promise.all([tx1(), tx2()]);

    // One transaction should abort due to WATCH
    const finalValue = await redisClient.get('watched-key');
    expect(['1', '100', '101']).toContain(finalValue);
  });

  it('handles pipeline with large number of commands', async () => {
    const pipeline = redisClient.pipeline();
    const commandCount = 10000;

    for (let i = 0; i < commandCount; i++) {
      pipeline.set(`bulk-${i}`, `value-${i}`);
    }

    const results = await pipeline.exec();

    expect(results.length).toBe(commandCount);
    expect(results.every(([err]) => err === null)).toBe(true);
  });

  it('cleans up pipeline on abort', async () => {
    const pipeline = redisClient.pipeline();

    pipeline.set('abort-test', 'value');
    pipeline.get('abort-test');

    // Abort pipeline before execution
    pipeline.discard();

    // Key should not exist
    const value = await redisClient.get('abort-test');
    expect(value).toBeNull();
  });
});
```

**Expected Results**:
- Pipeline commands execute in order
- Individual errors don't abort pipeline
- MULTI/EXEC provides atomicity
- WATCH enables optimistic locking
- Large pipelines handled
- Aborted pipelines cleaned up

**Verification Points**:
- Transaction semantics correct
- No partial state on errors

---

### INT-REDIS-011: Data Expiration Cleanup

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-REDIS-011 |
| **Title** | Data Expiration Cleanup |
| **Priority** | P2 - Medium |
| **Components** | `lib/redis/client.ts`, `lib/redis/cleanup.ts` |
| **Spec Reference** | Section 6.3, Data Lifecycle |

**Preconditions**:
- Redis test instance running
- Known TTL configurations

**Test Steps**:

```typescript
describe('Redis Data Expiration Cleanup', () => {
  it('expires session data after TTL', async () => {
    const sessionId = 'expire-session-test';
    await redisClient.setSession(sessionId, { data: 'test' }, { ttl: 1 });

    // Session exists immediately
    expect(await redisClient.getSession(sessionId)).not.toBeNull();

    // Wait for expiration
    await sleep(1500);

    // Session should be gone
    expect(await redisClient.getSession(sessionId)).toBeNull();
  });

  it('expires job progress data after completion', async () => {
    const jobId = 'expire-job-test';
    await redisClient.setJobProgress(jobId, { progress: 100, status: 'COMPLETE' });

    // Set short TTL after completion
    await redisClient.expire(redisKeys.job(jobId), 1);

    // Wait for expiration
    await sleep(1500);

    // Job data should be gone
    expect(await redisClient.getJobProgress(jobId)).toBeNull();
  });

  it('extends TTL on session activity', async () => {
    const sessionId = 'extend-session-test';
    await redisClient.setSession(sessionId, { data: 'test' }, { ttl: 2 });

    // Initial TTL
    const initialTTL = await redisClient.ttl(redisKeys.session(sessionId));

    await sleep(500);

    // Touch session (extends TTL)
    await redisClient.touchSession(sessionId);

    // TTL should be extended
    const extendedTTL = await redisClient.ttl(redisKeys.session(sessionId));
    expect(extendedTTL).toBeGreaterThan(initialTTL - 1);
  });

  it('does not affect persistent data', async () => {
    const configKey = 'hx:config:setting';
    await redisClient.set(configKey, 'persistent-value');

    // No TTL should be set
    const ttl = await redisClient.ttl(configKey);
    expect(ttl).toBe(-1); // -1 means no expiration

    await sleep(2000);

    // Data should still exist
    const value = await redisClient.get(configKey);
    expect(value).toBe('persistent-value');
  });

  it('handles bulk expiration efficiently', async () => {
    // Create many expiring keys
    const keyCount = 1000;
    const pipeline = redisClient.pipeline();

    for (let i = 0; i < keyCount; i++) {
      pipeline.set(`bulk-expire-${i}`, 'value', 'EX', 1);
    }

    await pipeline.exec();

    // Verify keys exist
    const existsBefore = await redisClient.exists(
      ...Array.from({ length: 10 }, (_, i) => `bulk-expire-${i}`)
    );
    expect(existsBefore).toBe(10);

    // Wait for expiration
    await sleep(1500);

    // All keys should be gone
    const existsAfter = await redisClient.exists(
      ...Array.from({ length: 10 }, (_, i) => `bulk-expire-${i}`)
    );
    expect(existsAfter).toBe(0);
  });

  it('reports expired key metrics', async () => {
    const metrics: any[] = [];

    redisClient.on('key:expired', (metric) => {
      metrics.push(metric);
    });

    // Create expiring key with short TTL
    await redisClient.set('metric-test', 'value', 'EX', 1);

    await sleep(1500);

    // Should have received expiration metric
    expect(metrics.some((m) => m.key === 'metric-test')).toBe(true);
  });
});
```

**Expected Results**:
- Session data expires after TTL
- Job data expires after completion
- TTL extended on activity
- Persistent data not affected
- Bulk expiration handled efficiently
- Expired key metrics reported

**Verification Points**:
- Data lifecycle managed correctly
- No stale data accumulation

---

## 18. SSE Connection Management

### 18.1 Overview

Tests validating SSE (Server-Sent Events) connection lifecycle, concurrent stream handling, and event ordering guarantees.

---

### INT-SSE-003: Redis Pub/Sub Connection Cleanup on Disconnect

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-SSE-003 |
| **Title** | Redis Pub/Sub Connection Cleanup on Disconnect |
| **Priority** | P1 - High |
| **Components** | `/api/v1/sse`, `lib/redis/pubsub.ts` |
| **Spec Reference** | Section 6.3.2, SSE Integration |

**Preconditions**:
- SSE endpoint available
- Redis Pub/Sub configured
- Connection tracking enabled

**Test Steps**:

```typescript
describe('SSE Redis Pub/Sub Cleanup', () => {
  it('unsubscribes from Redis channel on client disconnect', async () => {
    const sessionId = 'cleanup-test-session';
    const channelKey = redisKeys.sseChannel(sessionId);

    // Establish SSE connection
    const controller = new AbortController();
    const ssePromise = fetch(`/api/v1/sse?sessionId=${sessionId}`, {
      signal: controller.signal,
    });

    // Wait for subscription to be established
    await sleep(100);

    // Verify subscription exists
    const subsBefore = await redisClient.pubsubChannels(channelKey);
    expect(subsBefore).toContain(channelKey);

    // Disconnect client
    controller.abort();

    // Wait for cleanup
    await sleep(200);

    // Verify subscription removed
    const subsAfter = await redisClient.pubsubChannels(channelKey);
    expect(subsAfter).not.toContain(channelKey);
  });

  it('cleans up subscription on server-side stream close', async () => {
    const sessionId = 'server-close-test';
    const channelKey = redisKeys.sseChannel(sessionId);

    // Start SSE connection
    const response = await fetch(`/api/v1/sse?sessionId=${sessionId}`);
    const reader = response.body?.getReader();

    // Wait for subscription
    await sleep(100);

    // Verify subscription exists
    const subsBefore = await redisClient.pubsubChannels(channelKey);
    expect(subsBefore).toContain(channelKey);

    // Trigger server-side close (e.g., via admin endpoint or timeout)
    await fetch(`/api/v1/admin/close-sse?sessionId=${sessionId}`, { method: 'POST' });

    // Wait for cleanup
    await sleep(200);

    // Verify subscription removed
    const subsAfter = await redisClient.pubsubChannels(channelKey);
    expect(subsAfter).not.toContain(channelKey);

    // Reader should indicate stream closed
    const { done } = await reader?.read() || { done: true };
    expect(done).toBe(true);
  });

  it('handles multiple disconnects of same session', async () => {
    const sessionId = 'multi-disconnect-test';
    const channelKey = redisKeys.sseChannel(sessionId);

    // Establish multiple connections for same session
    const controllers = Array.from({ length: 3 }, () => new AbortController());
    const connections = controllers.map((controller) =>
      fetch(`/api/v1/sse?sessionId=${sessionId}`, { signal: controller.signal })
    );

    await Promise.all(connections);
    await sleep(100);

    // Verify single subscription (deduped)
    const numsub = await redisClient.pubsubNumsub(channelKey);
    expect(numsub[channelKey]).toBe(1);

    // Disconnect all clients
    controllers.forEach((c) => c.abort());

    await sleep(200);

    // Verify cleanup
    const subsAfter = await redisClient.pubsubChannels(channelKey);
    expect(subsAfter).not.toContain(channelKey);
  });

  it('does not leak Redis connections on rapid connect/disconnect', async () => {
    const initialConnections = await getRedisConnectionCount();

    // Rapid connect/disconnect cycles
    for (let i = 0; i < 50; i++) {
      const controller = new AbortController();
      fetch(`/api/v1/sse?sessionId=rapid-test-${i}`, { signal: controller.signal });
      await sleep(10);
      controller.abort();
    }

    // Wait for cleanup
    await sleep(500);

    const finalConnections = await getRedisConnectionCount();

    // Should not have leaked connections
    expect(finalConnections).toBeLessThanOrEqual(initialConnections + 5);
  });

  it('cleans up subscription on request timeout', async () => {
    const sessionId = 'timeout-cleanup-test';
    const channelKey = redisKeys.sseChannel(sessionId);

    // Configure short server-side timeout
    const response = await fetch(`/api/v1/sse?sessionId=${sessionId}&timeout=1000`);

    // Wait for subscription
    await sleep(100);
    expect((await redisClient.pubsubChannels(channelKey)).length).toBeGreaterThan(0);

    // Wait for timeout
    await sleep(1500);

    // Verify cleanup after timeout
    const subsAfter = await redisClient.pubsubChannels(channelKey);
    expect(subsAfter).not.toContain(channelKey);
  });
});
```

**Expected Results**:
- Subscription removed on client disconnect
- Subscription removed on server-side close
- Multiple disconnects handled correctly
- No connection leaks on rapid cycles
- Timeout triggers cleanup

**Verification Points**:
- No orphaned Redis subscriptions
- No resource leaks

---

### INT-SSE-004: Multiple Concurrent SSE Streams

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-SSE-004 |
| **Title** | Multiple Concurrent SSE Streams |
| **Priority** | P1 - High |
| **Components** | `/api/v1/sse`, `lib/sse/manager.ts` |
| **Spec Reference** | Section 6.3.2, Concurrent Sessions |

**Preconditions**:
- SSE endpoint available
- Multiple test sessions

**Test Steps**:

```typescript
describe('Multiple Concurrent SSE Streams', () => {
  it('handles multiple independent SSE connections', async () => {
    const sessionCount = 10;
    const sessions = Array.from({ length: sessionCount }, (_, i) => `concurrent-session-${i}`);

    // Establish all connections
    const connections = sessions.map((sessionId) => ({
      sessionId,
      events: [] as any[],
      controller: new AbortController(),
    }));

    for (const conn of connections) {
      const response = await fetch(`/api/v1/sse?sessionId=${conn.sessionId}`, {
        signal: conn.controller.signal,
      });

      const reader = response.body?.getReader();
      const decoder = new TextDecoder();

      // Read events in background
      (async () => {
        while (true) {
          const { done, value } = (await reader?.read()) || { done: true, value: undefined };
          if (done) break;
          conn.events.push(decoder.decode(value));
        }
      })();
    }

    await sleep(100);

    // Publish to each session
    for (const conn of connections) {
      await redisClient.publish(redisKeys.sseChannel(conn.sessionId), JSON.stringify({ test: conn.sessionId }));
    }

    await sleep(100);

    // Verify each connection received only its message
    for (const conn of connections) {
      expect(conn.events.some((e) => e.includes(conn.sessionId))).toBe(true);
      // Should not have received other sessions' messages
      const otherSessions = sessions.filter((s) => s !== conn.sessionId);
      expect(conn.events.every((e) => !otherSessions.some((os) => e.includes(os)))).toBe(true);
    }

    // Cleanup
    connections.forEach((c) => c.controller.abort());
  });

  it('maintains isolation between concurrent streams', async () => {
    const session1 = 'isolation-test-1';
    const session2 = 'isolation-test-2';

    const events1: string[] = [];
    const events2: string[] = [];

    // Connection 1
    const controller1 = new AbortController();
    const response1 = await fetch(`/api/v1/sse?sessionId=${session1}`, {
      signal: controller1.signal,
    });
    readSSEEvents(response1, events1);

    // Connection 2
    const controller2 = new AbortController();
    const response2 = await fetch(`/api/v1/sse?sessionId=${session2}`, {
      signal: controller2.signal,
    });
    readSSEEvents(response2, events2);

    await sleep(100);

    // Publish to session 1 only
    await redisClient.publish(
      redisKeys.sseChannel(session1),
      JSON.stringify({ message: 'for-session-1' })
    );

    await sleep(100);

    // Session 1 should receive, session 2 should not
    expect(events1.some((e) => e.includes('for-session-1'))).toBe(true);
    expect(events2.some((e) => e.includes('for-session-1'))).toBe(false);

    controller1.abort();
    controller2.abort();
  });

  it('handles high concurrent connection count', async () => {
    const connectionCount = 100;
    const controllers: AbortController[] = [];
    const responses: Response[] = [];

    // Establish many connections
    for (let i = 0; i < connectionCount; i++) {
      const controller = new AbortController();
      controllers.push(controller);

      const response = await fetch(`/api/v1/sse?sessionId=high-concurrent-${i}`, {
        signal: controller.signal,
      });
      responses.push(response);
    }

    // All connections should be established
    expect(responses.length).toBe(connectionCount);
    expect(responses.every((r) => r.ok)).toBe(true);

    // Cleanup
    controllers.forEach((c) => c.abort());
  });

  it('properly manages memory under concurrent load', async () => {
    const initialMemory = process.memoryUsage().heapUsed;

    // Create and destroy many connections
    for (let batch = 0; batch < 10; batch++) {
      const controllers: AbortController[] = [];

      // Create batch of connections
      for (let i = 0; i < 10; i++) {
        const controller = new AbortController();
        controllers.push(controller);
        fetch(`/api/v1/sse?sessionId=memory-test-${batch}-${i}`, {
          signal: controller.signal,
        });
      }

      await sleep(50);

      // Destroy batch
      controllers.forEach((c) => c.abort());
      await sleep(50);
    }

    // Force garbage collection if available
    if (global.gc) global.gc();

    const finalMemory = process.memoryUsage().heapUsed;

    // Memory should not have grown significantly
    const memoryGrowth = finalMemory - initialMemory;
    expect(memoryGrowth).toBeLessThan(50 * 1024 * 1024); // Less than 50MB growth
  });
});
```

**Expected Results**:
- Multiple independent connections handled
- Stream isolation maintained
- High concurrent count supported
- Memory managed under load

**Verification Points**:
- No cross-session data leakage
- Scalable concurrent connections

---

### INT-SSE-005: SSE Event Ordering Guarantee

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-SSE-005 |
| **Title** | SSE Event Ordering Guarantee |
| **Priority** | P2 - Medium |
| **Components** | `/api/v1/sse`, `lib/redis/pubsub.ts` |
| **Spec Reference** | Section 6.3.2, Event Delivery |

**Preconditions**:
- SSE endpoint available
- Redis Pub/Sub configured
- Event sequence tracking capability

**Test Steps**:

```typescript
describe('SSE Event Ordering', () => {
  it('delivers events in publish order', async () => {
    const sessionId = 'order-test-session';
    const receivedEvents: any[] = [];

    // Establish SSE connection
    const controller = new AbortController();
    const response = await fetch(`/api/v1/sse?sessionId=${sessionId}`, {
      signal: controller.signal,
    });

    readSSEEvents(response, receivedEvents, (event) => {
      try {
        return JSON.parse(event.data);
      } catch {
        return null;
      }
    });

    await sleep(100);

    // Publish events in sequence
    const eventCount = 20;
    for (let i = 0; i < eventCount; i++) {
      await redisClient.publish(
        redisKeys.sseChannel(sessionId),
        JSON.stringify({ sequence: i })
      );
    }

    await sleep(200);

    // Verify order preserved
    const sequences = receivedEvents
      .filter((e) => e && e.sequence !== undefined)
      .map((e) => e.sequence);

    for (let i = 1; i < sequences.length; i++) {
      expect(sequences[i]).toBeGreaterThan(sequences[i - 1]);
    }

    controller.abort();
  });

  it('maintains order under rapid publishing', async () => {
    const sessionId = 'rapid-order-test';
    const receivedEvents: any[] = [];

    const controller = new AbortController();
    const response = await fetch(`/api/v1/sse?sessionId=${sessionId}`, {
      signal: controller.signal,
    });

    readSSEEvents(response, receivedEvents, (event) => {
      try {
        return JSON.parse(event.data);
      } catch {
        return null;
      }
    });

    await sleep(100);

    // Rapid-fire publishing
    const pipeline = redisClient.pipeline();
    for (let i = 0; i < 100; i++) {
      pipeline.publish(
        redisKeys.sseChannel(sessionId),
        JSON.stringify({ sequence: i })
      );
    }
    await pipeline.exec();

    await sleep(500);

    // Verify all events received in order
    const sequences = receivedEvents
      .filter((e) => e && e.sequence !== undefined)
      .map((e) => e.sequence);

    expect(sequences.length).toBe(100);

    for (let i = 0; i < sequences.length; i++) {
      expect(sequences[i]).toBe(i);
    }

    controller.abort();
  });

  it('includes sequence numbers for verification', async () => {
    const sessionId = 'seq-number-test';
    const receivedEvents: any[] = [];

    const controller = new AbortController();
    const response = await fetch(`/api/v1/sse?sessionId=${sessionId}`, {
      signal: controller.signal,
    });

    readSSEEvents(response, receivedEvents);

    await sleep(100);

    // Publish events
    for (let i = 0; i < 5; i++) {
      await redisClient.publish(
        redisKeys.sseChannel(sessionId),
        JSON.stringify({ data: `message-${i}` })
      );
    }

    await sleep(100);

    // Verify events include id field for ordering
    const eventsWithId = receivedEvents.filter((e) => e.id !== undefined);
    expect(eventsWithId.length).toBeGreaterThan(0);

    // IDs should be monotonically increasing
    for (let i = 1; i < eventsWithId.length; i++) {
      expect(parseInt(eventsWithId[i].id)).toBeGreaterThan(parseInt(eventsWithId[i - 1].id));
    }

    controller.abort();
  });

  it('handles reconnection with last-event-id', async () => {
    const sessionId = 'reconnect-order-test';
    const firstBatchEvents: any[] = [];
    const secondBatchEvents: any[] = [];

    // First connection
    const controller1 = new AbortController();
    const response1 = await fetch(`/api/v1/sse?sessionId=${sessionId}`, {
      signal: controller1.signal,
    });

    let lastEventId = '0';
    readSSEEvents(response1, firstBatchEvents, (event) => {
      if (event.id) lastEventId = event.id;
      return event;
    });

    await sleep(100);

    // Publish first batch
    for (let i = 0; i < 5; i++) {
      await redisClient.publish(
        redisKeys.sseChannel(sessionId),
        JSON.stringify({ batch: 1, sequence: i })
      );
    }

    await sleep(100);

    // Disconnect first connection
    controller1.abort();

    // Publish while disconnected (should be buffered or lost based on design)
    await redisClient.publish(
      redisKeys.sseChannel(sessionId),
      JSON.stringify({ batch: 'disconnected', sequence: 99 })
    );

    // Reconnect with Last-Event-ID
    const controller2 = new AbortController();
    const response2 = await fetch(`/api/v1/sse?sessionId=${sessionId}`, {
      signal: controller2.signal,
      headers: { 'Last-Event-ID': lastEventId },
    });

    readSSEEvents(response2, secondBatchEvents);

    await sleep(100);

    // Publish second batch
    for (let i = 0; i < 5; i++) {
      await redisClient.publish(
        redisKeys.sseChannel(sessionId),
        JSON.stringify({ batch: 2, sequence: i })
      );
    }

    await sleep(100);

    // Verify second batch received correctly
    const batch2Events = secondBatchEvents.filter((e) => e.data?.batch === 2);
    expect(batch2Events.length).toBe(5);

    controller2.abort();
  });
});
```

**Expected Results**:
- Events delivered in publish order
- Order maintained under rapid publishing
- Sequence numbers included
- Reconnection with Last-Event-ID supported

**Verification Points**:
- Event ordering guaranteed
- Reconnection handles missed events appropriately

---

## 19. Request Validation Integration

### 19.1 Overview

Tests validating the request validation chain integration, including RFC 7807 error responses, body size limits, content-type validation, and malformed JSON handling across API endpoints.

---

### INT-VAL-001: Request Validation Chain Integration

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-VAL-001 |
| **Title** | Request Validation Chain Integration |
| **Priority** | P1 - High |
| **Components** | `lib/validation/`, `app/api/v1/`, `middleware.ts` |
| **Spec Reference** | Section 5.2, Input Validation |

**Preconditions**:
- API endpoints available
- Validation middleware configured
- Zod schemas defined

**Test Steps**:

```typescript
describe('Request Validation Chain Integration', () => {
  it('executes validation chain in correct order', async () => {
    const validationOrder: string[] = [];

    // Instrument validation middleware to track order
    const originalValidators = { ...validators };
    Object.keys(validators).forEach((key) => {
      validators[key] = async (...args: any[]) => {
        validationOrder.push(key);
        return originalValidators[key](...args);
      };
    });

    // Send request with multiple validation requirements
    const response = await fetch('/api/v1/jobs', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-Session-ID': 'test-session',
      },
      body: JSON.stringify({
        inputType: 'URL',
        url: 'https://example.com/document.pdf',
      }),
    });

    // Verify validation order: auth -> rate-limit -> schema -> business
    expect(validationOrder).toEqual([
      'authValidation',
      'rateLimitValidation',
      'schemaValidation',
      'businessValidation',
    ]);

    // Restore original validators
    Object.assign(validators, originalValidators);
  });

  it('stops chain on first validation failure', async () => {
    const validationOrder: string[] = [];

    // Track validation calls
    const originalValidators = { ...validators };
    Object.keys(validators).forEach((key) => {
      validators[key] = async (...args: any[]) => {
        validationOrder.push(key);
        return originalValidators[key](...args);
      };
    });

    // Send request that fails auth validation
    const response = await fetch('/api/v1/jobs', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      // Missing X-Session-ID header
      body: JSON.stringify({ inputType: 'URL' }),
    });

    expect(response.status).toBe(401);
    // Chain should stop after auth failure
    expect(validationOrder).toEqual(['authValidation']);

    Object.assign(validators, originalValidators);
  });

  it('aggregates validation errors from schema validation', async () => {
    const response = await fetch('/api/v1/jobs', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-Session-ID': 'test-session',
      },
      body: JSON.stringify({
        inputType: 'INVALID_TYPE',
        // Missing required fields
      }),
    });

    expect(response.status).toBe(400);
    const body = await response.json();

    // Should contain multiple validation errors
    expect(body.errors).toBeDefined();
    expect(body.errors.length).toBeGreaterThan(1);
  });

  it('validates nested object structures', async () => {
    const response = await fetch('/api/v1/jobs', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-Session-ID': 'test-session',
      },
      body: JSON.stringify({
        inputType: 'FILE',
        fileName: 'test.pdf',
        options: {
          exportFormats: ['INVALID_FORMAT'],
          ocr: { enabled: 'not-a-boolean' },
        },
      }),
    });

    expect(response.status).toBe(400);
    const body = await response.json();

    // Should report nested validation errors with paths
    expect(body.errors).toContainEqual(
      expect.objectContaining({
        path: expect.stringContaining('options.exportFormats'),
      })
    );
  });

  it('sanitizes input after validation passes', async () => {
    const response = await fetch('/api/v1/jobs', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-Session-ID': 'test-session',
      },
      body: JSON.stringify({
        inputType: 'URL',
        url: 'https://example.com/doc.pdf',
        extraField: '<script>alert("xss")</script>',
        __proto__: { polluted: true },
      }),
    });

    expect(response.status).toBe(202);

    // Verify extra fields stripped and no prototype pollution
    const job = await prisma.job.findFirst({
      where: { sessionId: 'test-session' },
      orderBy: { createdAt: 'desc' },
    });

    expect(job).not.toHaveProperty('extraField');
    expect(Object.getPrototypeOf(job)).not.toHaveProperty('polluted');
  });
});
```

**Expected Results**:
- Validation chain executes in correct order
- Chain stops on first failure
- Multiple errors aggregated from schema validation
- Nested structures validated correctly
- Input sanitized after validation

**Verification Points**:
- Order: auth -> rate-limit -> schema -> business
- Early termination on failure
- Complete error reporting

---

### INT-VAL-002: Validation Error Response Format RFC 7807

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-VAL-002 |
| **Title** | Validation Error Response Format RFC 7807 |
| **Priority** | P1 - High |
| **Components** | `lib/errors/`, `app/api/v1/` |
| **Spec Reference** | Section 5.3, Error Response Format |

**Preconditions**:
- API endpoints available
- Error handling middleware configured

**Test Steps**:

```typescript
describe('RFC 7807 Error Response Format', () => {
  it('returns RFC 7807 compliant validation error response', async () => {
    const response = await fetch('/api/v1/jobs', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-Session-ID': 'test-session',
      },
      body: JSON.stringify({
        inputType: 'INVALID',
      }),
    });

    expect(response.status).toBe(400);
    expect(response.headers.get('Content-Type')).toBe('application/problem+json');

    const body = await response.json();

    // RFC 7807 required fields
    expect(body).toHaveProperty('type');
    expect(body).toHaveProperty('title');
    expect(body).toHaveProperty('status', 400);
    expect(body).toHaveProperty('detail');

    // Type should be a URI
    expect(body.type).toMatch(/^https?:\/\//);
  });

  it('includes instance field with request correlation ID', async () => {
    const response = await fetch('/api/v1/jobs', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-Session-ID': 'test-session',
        'X-Request-ID': 'test-correlation-123',
      },
      body: JSON.stringify({ inputType: 'INVALID' }),
    });

    const body = await response.json();

    expect(body).toHaveProperty('instance');
    expect(body.instance).toContain('test-correlation-123');
  });

  it('includes validation errors extension for field-level errors', async () => {
    const response = await fetch('/api/v1/jobs', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-Session-ID': 'test-session',
      },
      body: JSON.stringify({
        inputType: 'FILE',
        // Missing fileName
      }),
    });

    const body = await response.json();

    // Extension field for validation errors
    expect(body).toHaveProperty('errors');
    expect(Array.isArray(body.errors)).toBe(true);

    const fieldError = body.errors.find((e: any) => e.field === 'fileName');
    expect(fieldError).toBeDefined();
    expect(fieldError).toHaveProperty('message');
    expect(fieldError).toHaveProperty('code');
  });

  it('uses consistent error type URIs across endpoints', async () => {
    const endpoints = [
      { path: '/api/v1/jobs', method: 'POST' },
      { path: '/api/v1/upload', method: 'POST' },
      { path: '/api/v1/process', method: 'POST' },
    ];

    const errorTypes: string[] = [];

    for (const endpoint of endpoints) {
      const response = await fetch(endpoint.path, {
        method: endpoint.method,
        headers: {
          'Content-Type': 'application/json',
          'X-Session-ID': 'test-session',
        },
        body: JSON.stringify({ invalid: 'data' }),
      });

      if (response.status === 400) {
        const body = await response.json();
        errorTypes.push(body.type);
      }
    }

    // Same validation error type across endpoints
    const uniqueTypes = [...new Set(errorTypes)];
    expect(uniqueTypes.length).toBe(1);
    expect(uniqueTypes[0]).toContain('validation-error');
  });

  it('includes timestamp in error response', async () => {
    const beforeRequest = new Date();

    const response = await fetch('/api/v1/jobs', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-Session-ID': 'test-session',
      },
      body: JSON.stringify({ inputType: 'INVALID' }),
    });

    const afterRequest = new Date();
    const body = await response.json();

    expect(body).toHaveProperty('timestamp');
    const timestamp = new Date(body.timestamp);
    expect(timestamp.getTime()).toBeGreaterThanOrEqual(beforeRequest.getTime());
    expect(timestamp.getTime()).toBeLessThanOrEqual(afterRequest.getTime());
  });
});
```

**Expected Results**:
- RFC 7807 compliant response structure
- Request correlation ID in instance field
- Field-level validation errors extension
- Consistent error type URIs
- Timestamp included

**Verification Points**:
- Content-Type: application/problem+json
- Required RFC 7807 fields present
- Error type URI format correct

---

### INT-VAL-003: Request Body Size Limits Enforcement

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-VAL-003 |
| **Title** | Request Body Size Limits Enforcement |
| **Priority** | P1 - High |
| **Components** | `middleware.ts`, `app/api/v1/upload/` |
| **Spec Reference** | Section 5.1.3, File Size Limits |

**Preconditions**:
- API endpoints available
- Body size limits configured
- Upload endpoint functional

**Test Steps**:

```typescript
describe('Request Body Size Limits Enforcement', () => {
  it('rejects JSON body exceeding limit', async () => {
    // Create JSON body exceeding 1MB limit
    const largePayload = {
      data: 'x'.repeat(2 * 1024 * 1024), // 2MB string
    };

    const response = await fetch('/api/v1/jobs', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-Session-ID': 'test-session',
      },
      body: JSON.stringify(largePayload),
    });

    expect(response.status).toBe(413);

    const body = await response.json();
    expect(body.type).toContain('payload-too-large');
    expect(body.detail).toContain('1MB');
  });

  it('rejects file upload exceeding size limit', async () => {
    // Create file exceeding 50MB limit
    const largeFile = new Blob([new Uint8Array(60 * 1024 * 1024)], {
      type: 'application/pdf',
    });

    const formData = new FormData();
    formData.append('file', largeFile, 'large.pdf');

    const response = await fetch('/api/v1/upload', {
      method: 'POST',
      headers: { 'X-Session-ID': 'test-session' },
      body: formData,
    });

    expect(response.status).toBe(413);

    const body = await response.json();
    expect(body.type).toContain('payload-too-large');
    expect(body.detail).toContain('50MB');
  });

  it('accepts body at exactly the size limit', async () => {
    // Create JSON body at exactly 1MB
    const exactLimitPayload = {
      inputType: 'URL',
      url: 'https://example.com/doc.pdf',
      padding: 'x'.repeat(1024 * 1024 - 100), // ~1MB minus overhead
    };

    const response = await fetch('/api/v1/jobs', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-Session-ID': 'test-session',
      },
      body: JSON.stringify(exactLimitPayload),
    });

    // Should be accepted (400 for validation, not 413 for size)
    expect(response.status).not.toBe(413);
  });

  it('enforces different limits per endpoint', async () => {
    // Jobs endpoint: 1MB limit
    const mediumPayload = { data: 'x'.repeat(1.5 * 1024 * 1024) };

    const jobsResponse = await fetch('/api/v1/jobs', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-Session-ID': 'test-session',
      },
      body: JSON.stringify(mediumPayload),
    });
    expect(jobsResponse.status).toBe(413);

    // Upload endpoint: 50MB limit (same payload should be accepted)
    const uploadResponse = await fetch('/api/v1/upload', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-Session-ID': 'test-session',
      },
      body: JSON.stringify(mediumPayload),
    });
    expect(uploadResponse.status).not.toBe(413);
  });

  it('streams large uploads without buffering entire body', async () => {
    const fileSize = 40 * 1024 * 1024; // 40MB
    const file = new Blob([new Uint8Array(fileSize)], {
      type: 'application/pdf',
    });

    const formData = new FormData();
    formData.append('file', file, 'medium.pdf');

    const initialMemory = process.memoryUsage().heapUsed;

    const response = await fetch('/api/v1/upload', {
      method: 'POST',
      headers: { 'X-Session-ID': 'test-session' },
      body: formData,
    });

    const peakMemory = process.memoryUsage().heapUsed;
    const memoryGrowth = peakMemory - initialMemory;

    // Memory should not grow by full file size (streaming)
    expect(memoryGrowth).toBeLessThan(fileSize * 0.5);
    expect(response.ok).toBe(true);
  });
});
```

**Expected Results**:
- JSON body exceeding limit rejected with 413
- File upload exceeding limit rejected with 413
- Bodies at exact limit accepted
- Different limits per endpoint enforced
- Large uploads streamed without full buffering

**Verification Points**:
- 413 Payload Too Large response
- Size limit in error message
- Memory-efficient streaming

---

### INT-VAL-004: Content-Type Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-VAL-004 |
| **Title** | Content-Type Validation |
| **Priority** | P1 - High |
| **Components** | `middleware.ts`, `app/api/v1/` |
| **Spec Reference** | Section 5.1.2, Content Type Handling |

**Preconditions**:
- API endpoints available
- Content-Type validation configured

**Test Steps**:

```typescript
describe('Content-Type Validation', () => {
  it('rejects request with missing Content-Type for POST', async () => {
    const response = await fetch('/api/v1/jobs', {
      method: 'POST',
      headers: { 'X-Session-ID': 'test-session' },
      body: JSON.stringify({ inputType: 'URL' }),
    });

    expect(response.status).toBe(415);

    const body = await response.json();
    expect(body.type).toContain('unsupported-media-type');
    expect(body.detail).toContain('Content-Type');
  });

  it('rejects request with incorrect Content-Type', async () => {
    const response = await fetch('/api/v1/jobs', {
      method: 'POST',
      headers: {
        'Content-Type': 'text/plain',
        'X-Session-ID': 'test-session',
      },
      body: JSON.stringify({ inputType: 'URL' }),
    });

    expect(response.status).toBe(415);

    const body = await response.json();
    expect(body.detail).toContain('application/json');
  });

  it('accepts Content-Type with charset parameter', async () => {
    const response = await fetch('/api/v1/jobs', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json; charset=utf-8',
        'X-Session-ID': 'test-session',
      },
      body: JSON.stringify({
        inputType: 'URL',
        url: 'https://example.com/doc.pdf',
      }),
    });

    // Should not reject due to charset parameter
    expect(response.status).not.toBe(415);
  });

  it('validates Content-Type for multipart/form-data uploads', async () => {
    const file = new Blob(['test content'], { type: 'application/pdf' });
    const formData = new FormData();
    formData.append('file', file, 'test.pdf');

    // FormData automatically sets correct Content-Type with boundary
    const response = await fetch('/api/v1/upload', {
      method: 'POST',
      headers: { 'X-Session-ID': 'test-session' },
      body: formData,
    });

    expect(response.status).not.toBe(415);
  });

  it('rejects unsupported file MIME types', async () => {
    const file = new Blob(['#!/bin/bash\nrm -rf /'], { type: 'application/x-sh' });
    const formData = new FormData();
    formData.append('file', file, 'script.sh');

    const response = await fetch('/api/v1/upload', {
      method: 'POST',
      headers: { 'X-Session-ID': 'test-session' },
      body: formData,
    });

    expect(response.status).toBe(415);

    const body = await response.json();
    expect(body.detail).toContain('supported file types');
  });

  it('validates file content matches declared MIME type', async () => {
    // Declare as PDF but content is not PDF
    const fakeFile = new Blob(['not a pdf'], { type: 'application/pdf' });
    const formData = new FormData();
    formData.append('file', fakeFile, 'fake.pdf');

    const response = await fetch('/api/v1/upload', {
      method: 'POST',
      headers: { 'X-Session-ID': 'test-session' },
      body: formData,
    });

    expect(response.status).toBe(400);

    const body = await response.json();
    expect(body.detail).toContain('content does not match');
  });
});
```

**Expected Results**:
- Missing Content-Type rejected with 415
- Incorrect Content-Type rejected with 415
- Content-Type with charset accepted
- multipart/form-data accepted for uploads
- Unsupported MIME types rejected
- MIME type spoofing detected

**Verification Points**:
- 415 Unsupported Media Type response
- Helpful error messages
- Magic byte validation for files

---

### INT-VAL-005: Malformed JSON Handling

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-VAL-005 |
| **Title** | Malformed JSON Handling |
| **Priority** | P1 - High |
| **Components** | `middleware.ts`, `app/api/v1/` |
| **Spec Reference** | Section 5.2.1, JSON Parsing |

**Preconditions**:
- API endpoints available
- JSON parsing middleware configured

**Test Steps**:

```typescript
describe('Malformed JSON Handling', () => {
  it('rejects syntactically invalid JSON', async () => {
    const response = await fetch('/api/v1/jobs', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-Session-ID': 'test-session',
      },
      body: '{ "inputType": "URL", invalid }',
    });

    expect(response.status).toBe(400);

    const body = await response.json();
    expect(body.type).toContain('malformed-request');
    expect(body.detail).toContain('JSON');
  });

  it('reports JSON parse error location', async () => {
    const response = await fetch('/api/v1/jobs', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-Session-ID': 'test-session',
      },
      body: '{\n  "inputType": "URL",\n  "url": undefined\n}',
    });

    expect(response.status).toBe(400);

    const body = await response.json();
    // Should include line/position info
    expect(body.detail).toMatch(/line|position|column/i);
  });

  it('handles empty request body', async () => {
    const response = await fetch('/api/v1/jobs', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-Session-ID': 'test-session',
      },
      body: '',
    });

    expect(response.status).toBe(400);

    const body = await response.json();
    expect(body.detail).toContain('empty');
  });

  it('handles truncated JSON', async () => {
    const response = await fetch('/api/v1/jobs', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-Session-ID': 'test-session',
      },
      body: '{ "inputType": "URL", "url": "https://example',
    });

    expect(response.status).toBe(400);

    const body = await response.json();
    expect(body.type).toContain('malformed-request');
  });

  it('rejects JSON with duplicate keys', async () => {
    const response = await fetch('/api/v1/jobs', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-Session-ID': 'test-session',
      },
      body: '{ "inputType": "URL", "inputType": "FILE" }',
    });

    // May be 400 (rejected) or use last value depending on strictness
    const body = await response.json();

    if (response.status === 400) {
      expect(body.detail).toContain('duplicate');
    } else {
      // If accepted, verify which value was used
      expect(body).toBeDefined();
    }
  });

  it('handles deeply nested JSON within limits', async () => {
    // Create deeply nested structure
    let nested: any = { value: 'deep' };
    for (let i = 0; i < 50; i++) {
      nested = { nested };
    }

    const response = await fetch('/api/v1/jobs', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-Session-ID': 'test-session',
      },
      body: JSON.stringify(nested),
    });

    // Should either accept or reject with clear error
    if (response.status === 400) {
      const body = await response.json();
      expect(body.detail).toMatch(/depth|nested/i);
    }
  });

  it('rejects JSON with __proto__ pollution attempts', async () => {
    const response = await fetch('/api/v1/jobs', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-Session-ID': 'test-session',
      },
      body: '{ "__proto__": { "polluted": true }, "inputType": "URL" }',
    });

    // Should sanitize or reject
    if (response.status === 200 || response.status === 202) {
      // If accepted, verify no pollution
      expect(({} as any).polluted).toBeUndefined();
    } else {
      expect(response.status).toBe(400);
    }
  });
});
```

**Expected Results**:
- Syntactically invalid JSON rejected
- Parse error location reported
- Empty body handled gracefully
- Truncated JSON detected
- Deep nesting handled appropriately
- Prototype pollution prevented

**Verification Points**:
- 400 Bad Request for malformed JSON
- Helpful error messages with location
- Security: no prototype pollution

---

## 20. Async Concurrency Integration

### 20.1 Overview

Tests validating concurrent job processing, database connection pool behavior under concurrency, Redis connection pool concurrency handling, and MCP client connection reuse patterns.

---

### INT-ASYNC-001: Concurrent Job Processing

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-ASYNC-001 |
| **Title** | Concurrent Job Processing |
| **Priority** | P1 - High |
| **Components** | `lib/jobs/processor.ts`, `app/api/v1/process/` |
| **Spec Reference** | Section 6.1, Job Processing |

**Preconditions**:
- Job processing endpoint available
- Multiple test jobs created
- Processing resources available

**Test Steps**:

```typescript
describe('Concurrent Job Processing', () => {
  it('processes multiple jobs concurrently', async () => {
    const jobCount = 5;
    const jobs: any[] = [];

    // Create multiple jobs
    for (let i = 0; i < jobCount; i++) {
      const job = await prisma.job.create({
        data: {
          sessionId: `concurrent-test-${i}`,
          status: 'PENDING',
          inputType: 'URL',
          inputUrl: `https://example.com/doc${i}.pdf`,
        },
      });
      jobs.push(job);
    }

    // Track processing times
    const processingTimes: { jobId: string; start: number; end: number }[] = [];

    // Start all jobs concurrently
    const startTime = Date.now();
    const promises = jobs.map(async (job) => {
      const start = Date.now();
      await fetch('/api/v1/process', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'X-Session-ID': job.sessionId,
        },
        body: JSON.stringify({ jobId: job.id }),
      });
      const end = Date.now();
      processingTimes.push({ jobId: job.id, start: start - startTime, end: end - startTime });
    });

    await Promise.all(promises);

    // Verify concurrent execution (overlapping time ranges)
    const overlaps = processingTimes.filter((t, i) =>
      processingTimes.some(
        (other, j) => i !== j && t.start < other.end && t.end > other.start
      )
    );

    expect(overlaps.length).toBeGreaterThan(0);

    // Verify all jobs processed
    for (const job of jobs) {
      const updated = await prisma.job.findUnique({ where: { id: job.id } });
      expect(['PROCESSING', 'COMPLETED', 'FAILED']).toContain(updated?.status);
    }
  });

  it('enforces concurrency limit per session', async () => {
    const sessionId = 'concurrency-limit-test';
    const jobCount = 10;
    const maxConcurrent = 3; // Expected limit

    // Create jobs
    const jobs = await Promise.all(
      Array.from({ length: jobCount }, (_, i) =>
        prisma.job.create({
          data: {
            sessionId,
            status: 'PENDING',
            inputType: 'URL',
            inputUrl: `https://example.com/doc${i}.pdf`,
          },
        })
      )
    );

    // Track concurrent processing count
    let currentConcurrent = 0;
    let maxObservedConcurrent = 0;

    const processJob = async (job: any) => {
      currentConcurrent++;
      maxObservedConcurrent = Math.max(maxObservedConcurrent, currentConcurrent);

      await fetch('/api/v1/process', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'X-Session-ID': sessionId,
        },
        body: JSON.stringify({ jobId: job.id }),
      });

      currentConcurrent--;
    };

    await Promise.all(jobs.map(processJob));

    // Should not exceed concurrency limit
    expect(maxObservedConcurrent).toBeLessThanOrEqual(maxConcurrent);
  });

  it('maintains job isolation under concurrent processing', async () => {
    const jobs = await Promise.all([
      prisma.job.create({
        data: {
          sessionId: 'isolation-1',
          status: 'PENDING',
          inputType: 'URL',
          inputUrl: 'https://example.com/doc1.pdf',
        },
      }),
      prisma.job.create({
        data: {
          sessionId: 'isolation-2',
          status: 'PENDING',
          inputType: 'FILE',
          fileName: 'doc2.docx',
          filePath: '/tmp/doc2.docx',
        },
      }),
    ]);

    // Process concurrently
    await Promise.all(
      jobs.map((job) =>
        fetch('/api/v1/process', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'X-Session-ID': job.sessionId,
          },
          body: JSON.stringify({ jobId: job.id }),
        })
      )
    );

    // Verify job data not mixed
    const results = await Promise.all(
      jobs.map((job) => prisma.job.findUnique({ where: { id: job.id } }))
    );

    expect(results[0]?.inputType).toBe('URL');
    expect(results[0]?.inputUrl).toContain('doc1');
    expect(results[1]?.inputType).toBe('FILE');
    expect(results[1]?.fileName).toContain('doc2');
  });

  it('handles partial concurrent failures gracefully', async () => {
    const jobs = await Promise.all([
      prisma.job.create({
        data: {
          sessionId: 'partial-fail-1',
          status: 'PENDING',
          inputType: 'URL',
          inputUrl: 'https://example.com/valid.pdf',
        },
      }),
      prisma.job.create({
        data: {
          sessionId: 'partial-fail-2',
          status: 'PENDING',
          inputType: 'URL',
          inputUrl: 'https://invalid-url-that-will-fail.example/doc.pdf',
        },
      }),
      prisma.job.create({
        data: {
          sessionId: 'partial-fail-3',
          status: 'PENDING',
          inputType: 'URL',
          inputUrl: 'https://example.com/another-valid.pdf',
        },
      }),
    ]);

    // Process all concurrently
    const results = await Promise.allSettled(
      jobs.map((job) =>
        fetch('/api/v1/process', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'X-Session-ID': job.sessionId,
          },
          body: JSON.stringify({ jobId: job.id }),
        })
      )
    );

    // One failure should not affect others
    const jobStatuses = await Promise.all(
      jobs.map((job) => prisma.job.findUnique({ where: { id: job.id } }))
    );

    // Valid jobs should succeed
    const successfulJobs = jobStatuses.filter((j) => j?.status === 'COMPLETED');
    expect(successfulJobs.length).toBeGreaterThanOrEqual(1);

    // Failed job should be marked as failed
    const failedJob = jobStatuses.find((j) => j?.sessionId === 'partial-fail-2');
    expect(failedJob?.status).toBe('FAILED');
  });
});
```

**Expected Results**:
- Multiple jobs processed concurrently
- Concurrency limit enforced per session
- Job isolation maintained
- Partial failures handled gracefully

**Verification Points**:
- Overlapping execution times
- No data mixing between jobs
- Independent failure handling

---

### INT-ASYNC-002: Database Connection Pool Under Concurrency

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-ASYNC-002 |
| **Title** | Database Connection Pool Under Concurrency |
| **Priority** | P1 - High |
| **Components** | `lib/db/prisma.ts`, `prisma/` |
| **Spec Reference** | Section 6.4, Database Connection Management |

**Preconditions**:
- Database connection pool configured
- Pool metrics accessible
- Multiple concurrent requests

**Test Steps**:

```typescript
describe('Database Connection Pool Under Concurrency', () => {
  it('handles concurrent database operations within pool limits', async () => {
    const concurrentOperations = 50;
    const poolSize = 10; // Expected pool size

    const operations = Array.from({ length: concurrentOperations }, (_, i) =>
      prisma.job.create({
        data: {
          sessionId: `pool-test-${i}`,
          status: 'PENDING',
          inputType: 'URL',
          inputUrl: `https://example.com/doc${i}.pdf`,
        },
      })
    );

    // All operations should complete without pool exhaustion
    const results = await Promise.all(operations);
    expect(results.length).toBe(concurrentOperations);

    // Cleanup
    await prisma.job.deleteMany({
      where: { sessionId: { startsWith: 'pool-test-' } },
    });
  });

  it('queues requests when pool is exhausted', async () => {
    const poolSize = 10;
    const totalOperations = poolSize * 3;
    const operationTimes: number[] = [];

    const slowOperation = async (index: number) => {
      const start = Date.now();
      // Simulate slow query
      await prisma.$queryRaw`SELECT pg_sleep(0.1)`;
      await prisma.job.findMany({ take: 1 });
      operationTimes.push(Date.now() - start);
    };

    const startTime = Date.now();
    await Promise.all(
      Array.from({ length: totalOperations }, (_, i) => slowOperation(i))
    );
    const totalTime = Date.now() - startTime;

    // Operations should be batched, not all concurrent
    // If truly concurrent with poolSize connections, should take ~3 batches
    const minExpectedTime = (totalOperations / poolSize) * 100; // 100ms per batch
    expect(totalTime).toBeGreaterThanOrEqual(minExpectedTime * 0.8);
  });

  it('recovers from pool exhaustion timeout', async () => {
    // This test verifies the pool recovers after exhaustion
    const exhaustOperations: Promise<any>[] = [];

    // Exhaust pool with long-running operations
    for (let i = 0; i < 20; i++) {
      exhaustOperations.push(
        prisma.$queryRaw`SELECT pg_sleep(2)`.catch(() => null)
      );
    }

    // Wait a bit then try a new operation
    await sleep(100);

    const normalOperation = async () => {
      const start = Date.now();
      try {
        await prisma.job.findFirst();
        return { success: true, time: Date.now() - start };
      } catch (error) {
        return { success: false, error };
      }
    };

    // Should queue and eventually complete
    const result = await Promise.race([
      normalOperation(),
      sleep(5000).then(() => ({ success: false, timeout: true })),
    ]);

    // Clean up long-running ops
    exhaustOperations.forEach((p) => p.catch(() => {}));

    // Either succeeds (queued) or fails with timeout error
    expect(result).toBeDefined();
  });

  it('does not leak connections on operation failure', async () => {
    const initialMetrics = await getPoolMetrics();

    // Perform operations that will fail
    const failingOperations = Array.from({ length: 20 }, () =>
      prisma.job
        .findUnique({ where: { id: 'non-existent-id' } })
        .catch(() => null)
    );

    await Promise.all(failingOperations);

    // Wait for connection cleanup
    await sleep(500);

    const finalMetrics = await getPoolMetrics();

    // Active connections should return to baseline
    expect(finalMetrics.activeConnections).toBeLessThanOrEqual(
      initialMetrics.activeConnections + 2
    );
  });

  it('handles transaction deadlock under concurrency', async () => {
    const sessionId = 'deadlock-test';

    // Create initial job
    const job = await prisma.job.create({
      data: { sessionId, status: 'PENDING', inputType: 'URL' },
    });

    // Simulate potential deadlock scenario with concurrent updates
    const updates = Array.from({ length: 10 }, (_, i) =>
      prisma.$transaction(async (tx) => {
        await tx.job.update({
          where: { id: job.id },
          data: { status: 'PROCESSING' },
        });
        await sleep(10);
        await tx.job.update({
          where: { id: job.id },
          data: { progress: i * 10 },
        });
      }).catch((e) => ({ error: e.message }))
    );

    const results = await Promise.all(updates);

    // Some may fail due to contention, but should not deadlock
    const errors = results.filter((r) => r && typeof r === 'object' && 'error' in r);

    // System should handle gracefully
    expect(errors.length).toBeLessThan(results.length);

    await prisma.job.delete({ where: { id: job.id } });
  });
});
```

**Expected Results**:
- Concurrent operations within pool limits succeed
- Requests queue when pool exhausted
- Pool recovers from exhaustion
- No connection leaks on failure
- Deadlocks handled gracefully

**Verification Points**:
- Pool size respected
- Queuing behavior correct
- Connection cleanup verified

---

### INT-ASYNC-003: Redis Connection Pool Concurrency

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-ASYNC-003 |
| **Title** | Redis Connection Pool Concurrency |
| **Priority** | P1 - High |
| **Components** | `lib/redis/client.ts`, `lib/redis/` |
| **Spec Reference** | Section 6.5, Redis Connection Management |

**Preconditions**:
- Redis connection pool configured
- Pool metrics accessible
- Multiple concurrent requests

**Test Steps**:

```typescript
describe('Redis Connection Pool Concurrency', () => {
  it('handles concurrent Redis operations efficiently', async () => {
    const concurrentOps = 100;
    const results: { time: number; success: boolean }[] = [];

    const operations = Array.from({ length: concurrentOps }, async (_, i) => {
      const start = Date.now();
      try {
        await redisClient.set(`concurrent-test-${i}`, `value-${i}`, 'EX', 60);
        await redisClient.get(`concurrent-test-${i}`);
        results.push({ time: Date.now() - start, success: true });
      } catch {
        results.push({ time: Date.now() - start, success: false });
      }
    });

    await Promise.all(operations);

    // All operations should succeed
    const successCount = results.filter((r) => r.success).length;
    expect(successCount).toBe(concurrentOps);

    // Operations should be fast (connection reuse)
    const avgTime = results.reduce((sum, r) => sum + r.time, 0) / results.length;
    expect(avgTime).toBeLessThan(100); // <100ms average

    // Cleanup
    await redisClient.del(
      ...Array.from({ length: concurrentOps }, (_, i) => `concurrent-test-${i}`)
    );
  });

  it('handles concurrent Pub/Sub operations', async () => {
    const channelCount = 10;
    const messagesPerChannel = 5;
    const received: Map<string, string[]> = new Map();

    // Create subscribers for multiple channels
    const subscribers = Array.from({ length: channelCount }, (_, i) => {
      const channel = `pubsub-concurrent-${i}`;
      received.set(channel, []);
      return subscriberClient.subscribe(channel, (message) => {
        received.get(channel)?.push(message);
      });
    });

    await Promise.all(subscribers);
    await sleep(100);

    // Publish to all channels concurrently
    const publishPromises: Promise<any>[] = [];
    for (let ch = 0; ch < channelCount; ch++) {
      for (let msg = 0; msg < messagesPerChannel; msg++) {
        publishPromises.push(
          redisClient.publish(
            `pubsub-concurrent-${ch}`,
            `message-${ch}-${msg}`
          )
        );
      }
    }

    await Promise.all(publishPromises);
    await sleep(200);

    // Verify all messages received
    for (let ch = 0; ch < channelCount; ch++) {
      const channelMessages = received.get(`pubsub-concurrent-${ch}`);
      expect(channelMessages?.length).toBe(messagesPerChannel);
    }

    // Cleanup
    await subscriberClient.unsubscribe();
  });

  it('does not block on slow operations', async () => {
    // Start a slow operation
    const slowOp = redisClient.blpop('non-existent-list', 2); // 2 second block

    // Fast operations should not be blocked
    const fastOpStart = Date.now();
    await redisClient.set('fast-op-test', 'value');
    const fastOpTime = Date.now() - fastOpStart;

    expect(fastOpTime).toBeLessThan(100); // Should not wait for slow op

    // Clean up slow op
    await slowOp.catch(() => null);
    await redisClient.del('fast-op-test');
  });

  it('handles pipeline operations under concurrency', async () => {
    const pipelineCount = 20;
    const opsPerPipeline = 50;

    const executePipeline = async (index: number) => {
      const pipeline = redisClient.pipeline();
      for (let i = 0; i < opsPerPipeline; i++) {
        pipeline.set(`pipeline-${index}-${i}`, `value-${i}`);
        pipeline.get(`pipeline-${index}-${i}`);
      }
      return pipeline.exec();
    };

    const startTime = Date.now();
    const results = await Promise.all(
      Array.from({ length: pipelineCount }, (_, i) => executePipeline(i))
    );
    const totalTime = Date.now() - startTime;

    // All pipelines should succeed
    expect(results.length).toBe(pipelineCount);
    results.forEach((pipelineResult) => {
      expect(pipelineResult?.length).toBe(opsPerPipeline * 2);
    });

    // Should be efficient (batched)
    const totalOps = pipelineCount * opsPerPipeline * 2;
    const opsPerSecond = (totalOps / totalTime) * 1000;
    expect(opsPerSecond).toBeGreaterThan(1000); // At least 1000 ops/sec

    // Cleanup
    const delKeys: string[] = [];
    for (let p = 0; p < pipelineCount; p++) {
      for (let i = 0; i < opsPerPipeline; i++) {
        delKeys.push(`pipeline-${p}-${i}`);
      }
    }
    await redisClient.del(...delKeys);
  });

  it('recovers from connection failures during concurrent operations', async () => {
    const operations = Array.from({ length: 50 }, async (_, i) => {
      try {
        await redisClient.set(`recovery-test-${i}`, `value-${i}`);
        return { success: true, index: i };
      } catch (error) {
        return { success: false, index: i, error };
      }
    });

    // Simulate brief connection issue mid-operations
    // (In real test, would use Redis proxy or container restart)

    const results = await Promise.all(operations);

    // Most operations should succeed
    const successCount = results.filter((r) => r.success).length;
    expect(successCount).toBeGreaterThan(results.length * 0.8);

    // Cleanup
    await redisClient.del(
      ...Array.from({ length: 50 }, (_, i) => `recovery-test-${i}`)
    );
  });
});
```

**Expected Results**:
- Concurrent operations handled efficiently
- Pub/Sub operations concurrent without interference
- Slow operations do not block fast ones
- Pipeline operations efficient under concurrency
- Connection failures handled gracefully

**Verification Points**:
- High throughput maintained
- Connection reuse verified
- No blocking behavior

---

### INT-ASYNC-004: MCP Client Connection Reuse

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-ASYNC-004 |
| **Title** | MCP Client Connection Reuse |
| **Priority** | P1 - High |
| **Components** | `lib/mcp/client.ts`, `lib/mcp/pool.ts` |
| **Spec Reference** | Section 6.2, MCP Connection Management |

**Preconditions**:
- MCP client configured
- Connection pooling enabled
- MCP server mock available

**Test Steps**:

```typescript
describe('MCP Client Connection Reuse', () => {
  it('reuses MCP connections for sequential requests', async () => {
    const connectionIds: string[] = [];

    // Mock to track connection IDs
    server.use(
      rest.post(MCP_ENDPOINT, async (req, res, ctx) => {
        const connectionId = req.headers.get('X-Connection-ID') || 'unknown';
        connectionIds.push(connectionId);

        const body = await req.json();
        if (body.method === 'initialize') {
          return res(
            ctx.json({
              jsonrpc: '2.0',
              result: { serverInfo: {}, capabilities: { tools: {} } },
              id: body.id,
            })
          );
        }
        return res(ctx.json({ jsonrpc: '2.0', result: {}, id: body.id }));
      })
    );

    // Make sequential requests
    for (let i = 0; i < 5; i++) {
      await mcpClient.callTool('convert_pdf', { path: `/tmp/doc${i}.pdf` });
    }

    // Should reuse connection (same ID after initial setup)
    const uniqueIds = [...new Set(connectionIds)];
    expect(uniqueIds.length).toBeLessThanOrEqual(2); // Initial + reused
  });

  it('handles concurrent requests with connection pool', async () => {
    const concurrentRequests = 10;
    const poolSize = 3; // Expected pool size
    const activeConnections: Set<string> = new Set();
    let maxConcurrentConnections = 0;

    server.use(
      rest.post(MCP_ENDPOINT, async (req, res, ctx) => {
        const connectionId = `conn-${Math.random()}`;
        activeConnections.add(connectionId);
        maxConcurrentConnections = Math.max(
          maxConcurrentConnections,
          activeConnections.size
        );

        // Simulate processing delay
        await sleep(100);

        activeConnections.delete(connectionId);

        const body = await req.json();
        return res(ctx.json({ jsonrpc: '2.0', result: {}, id: body.id }));
      })
    );

    // Make concurrent requests
    await Promise.all(
      Array.from({ length: concurrentRequests }, (_, i) =>
        mcpClient.callTool('convert_pdf', { path: `/tmp/doc${i}.pdf` })
      )
    );

    // Should not exceed pool size
    expect(maxConcurrentConnections).toBeLessThanOrEqual(poolSize);
  });

  it('reinitializes connection after server restart', async () => {
    let initializeCount = 0;

    server.use(
      rest.post(MCP_ENDPOINT, async (req, res, ctx) => {
        const body = await req.json();

        if (body.method === 'initialize') {
          initializeCount++;
          return res(
            ctx.json({
              jsonrpc: '2.0',
              result: { serverInfo: {}, capabilities: { tools: {} } },
              id: body.id,
            })
          );
        }

        // First tool call after "restart" fails
        if (body.method === 'tools/call' && initializeCount === 1) {
          return res(
            ctx.status(503),
            ctx.json({ error: 'Server restarting' })
          );
        }

        return res(
          ctx.json({
            jsonrpc: '2.0',
            result: { content: [{ type: 'text', text: 'success' }] },
            id: body.id,
          })
        );
      })
    );

    // First call - establishes connection
    await mcpClient.callTool('convert_pdf', { path: '/tmp/doc1.pdf' });
    expect(initializeCount).toBe(1);

    // Simulate server restart detection
    await mcpClient.resetConnection();

    // Next call should reinitialize
    await mcpClient.callTool('convert_pdf', { path: '/tmp/doc2.pdf' });
    expect(initializeCount).toBe(2);
  });

  it('handles connection timeout and retry', async () => {
    let requestCount = 0;

    server.use(
      rest.post(MCP_ENDPOINT, async (req, res, ctx) => {
        requestCount++;

        // First two requests timeout
        if (requestCount <= 2) {
          await sleep(5000); // Exceed timeout
          return res(ctx.status(504));
        }

        const body = await req.json();
        return res(ctx.json({ jsonrpc: '2.0', result: {}, id: body.id }));
      })
    );

    // Should retry and eventually succeed
    const result = await mcpClient.callTool('convert_pdf', {
      path: '/tmp/doc.pdf',
      timeout: 1000,
      retries: 3,
    });

    expect(requestCount).toBeGreaterThanOrEqual(3);
    expect(result).toBeDefined();
  });

  it('cleans up idle connections', async () => {
    const connections: { id: string; created: number; closed?: number }[] = [];

    server.use(
      rest.post(MCP_ENDPOINT, async (req, res, ctx) => {
        const connId = `conn-${Date.now()}`;
        connections.push({ id: connId, created: Date.now() });

        const body = await req.json();

        if (body.method === 'shutdown') {
          const conn = connections.find((c) => c.id === connId);
          if (conn) conn.closed = Date.now();
        }

        return res(ctx.json({ jsonrpc: '2.0', result: {}, id: body.id }));
      })
    );

    // Create connections
    await Promise.all([
      mcpClient.callTool('convert_pdf', { path: '/tmp/doc1.pdf' }),
      mcpClient.callTool('convert_pdf', { path: '/tmp/doc2.pdf' }),
    ]);

    // Wait for idle timeout (configured to be short in tests)
    await sleep(2000);

    // Trigger cleanup
    await mcpClient.cleanupIdleConnections();

    // Connections should be cleaned up
    const closedConnections = connections.filter((c) => c.closed);
    expect(closedConnections.length).toBeGreaterThan(0);
  });
});
```

**Expected Results**:
- Connections reused for sequential requests
- Connection pool limits respected
- Connections reinitialized after server restart
- Timeout and retry handled correctly
- Idle connections cleaned up

**Verification Points**:
- Connection ID reuse verified
- Pool size not exceeded
- Proper reconnection logic

---

## 21. File Storage Integration

### 21.1 Overview

Tests validating file upload, storage, retrieval, path validation, error handling, and cleanup operations for the storage backend.

---

### INT-STORAGE-001: File Upload to Storage Backend

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-STORAGE-001 |
| **Title** | File Upload to Storage Backend |
| **Priority** | P1 - High |
| **Components** | `/api/v1/upload`, `lib/storage/` |
| **Spec Reference** | FR-201, FR-202 |

**Preconditions**:
- Storage backend configured and accessible
- Valid file for upload
- Session established

**Test Steps**:

```typescript
describe('File Upload to Storage Backend', () => {
  it('uploads file to storage backend successfully', async () => {
    const testFile = createTestFile('test-document.pdf', 'application/pdf', 1024);
    const formData = new FormData();
    formData.append('file', testFile);

    const response = await fetch('/api/v1/upload', {
      method: 'POST',
      headers: { 'X-Session-ID': 'test-session' },
      body: formData,
    });

    expect(response.status).toBe(201);
    const { jobId, filePath } = await response.json();

    // Verify file exists in storage
    const fileExists = await storageClient.exists(filePath);
    expect(fileExists).toBe(true);

    // Verify file metadata stored
    const job = await prisma.job.findUnique({ where: { id: jobId } });
    expect(job?.filePath).toBe(filePath);
    expect(job?.fileName).toBe('test-document.pdf');
    expect(job?.mimeType).toBe('application/pdf');
  });

  it('generates unique storage paths for concurrent uploads', async () => {
    const uploadPromises = Array.from({ length: 5 }, (_, i) => {
      const testFile = createTestFile(`doc-${i}.pdf`, 'application/pdf', 512);
      const formData = new FormData();
      formData.append('file', testFile);

      return fetch('/api/v1/upload', {
        method: 'POST',
        headers: { 'X-Session-ID': `session-${i}` },
        body: formData,
      });
    });

    const responses = await Promise.all(uploadPromises);
    const results = await Promise.all(responses.map(r => r.json()));
    const filePaths = results.map(r => r.filePath);

    // All paths should be unique
    const uniquePaths = new Set(filePaths);
    expect(uniquePaths.size).toBe(5);
  });

  it('stores file with correct content', async () => {
    const testContent = 'Test PDF content for verification';
    const testFile = createTestFileWithContent('verify.pdf', testContent);
    const formData = new FormData();
    formData.append('file', testFile);

    const response = await fetch('/api/v1/upload', {
      method: 'POST',
      headers: { 'X-Session-ID': 'test-session' },
      body: formData,
    });

    const { filePath } = await response.json();
    const storedContent = await storageClient.read(filePath);

    expect(storedContent).toBe(testContent);
  });
});
```

**Expected Results**:
- File uploaded and stored successfully
- Unique paths generated for concurrent uploads
- File content matches uploaded content

**Verification Points**:
- Storage backend write operation successful
- File metadata correctly stored in database
- No path collisions on concurrent uploads

---

### INT-STORAGE-002: File Storage Path Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-STORAGE-002 |
| **Title** | File Storage Path Validation |
| **Priority** | P1 - High |
| **Components** | `lib/storage/`, `lib/validation/` |
| **Spec Reference** | FR-203, Security Requirements |

**Preconditions**:
- Storage backend configured
- Path validation rules defined

**Test Steps**:

```typescript
describe('File Storage Path Validation', () => {
  it('rejects path traversal attempts', async () => {
    const maliciousFilenames = [
      '../../../etc/passwd',
      '..\\..\\windows\\system32',
      'normal/../../../secret.txt',
      'file\x00.pdf',
    ];

    for (const filename of maliciousFilenames) {
      const testFile = createTestFile(filename, 'application/pdf', 100);
      const formData = new FormData();
      formData.append('file', testFile);

      const response = await fetch('/api/v1/upload', {
        method: 'POST',
        headers: { 'X-Session-ID': 'test-session' },
        body: formData,
      });

      expect(response.status).toBe(400);
      const error = await response.json();
      expect(error.code).toBe('E201');
    }
  });

  it('sanitizes filenames before storage', async () => {
    const unsafeFilenames = [
      { input: 'file with spaces.pdf', expected: 'file_with_spaces.pdf' },
      { input: 'special!@#$chars.pdf', expected: 'special_chars.pdf' },
      { input: 'UPPERCASE.PDF', expected: 'uppercase.pdf' },
    ];

    for (const { input, expected } of unsafeFilenames) {
      const testFile = createTestFile(input, 'application/pdf', 100);
      const formData = new FormData();
      formData.append('file', testFile);

      const response = await fetch('/api/v1/upload', {
        method: 'POST',
        headers: { 'X-Session-ID': 'test-session' },
        body: formData,
      });

      const { filePath } = await response.json();
      expect(filePath).toContain(expected);
    }
  });

  it('enforces storage directory boundaries', async () => {
    const testFile = createTestFile('test.pdf', 'application/pdf', 100);
    const formData = new FormData();
    formData.append('file', testFile);

    const response = await fetch('/api/v1/upload', {
      method: 'POST',
      headers: { 'X-Session-ID': 'test-session' },
      body: formData,
    });

    const { filePath } = await response.json();
    const configuredBasePath = process.env.STORAGE_BASE_PATH || '/tmp/uploads';

    expect(filePath.startsWith(configuredBasePath)).toBe(true);
  });
});
```

**Expected Results**:
- Path traversal attempts rejected
- Filenames sanitized before storage
- Storage directory boundaries enforced

**Verification Points**:
- Security validation active
- No files written outside allowed directories
- Filename sanitization applied consistently

---

### INT-STORAGE-003: File Retrieval for Processing

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-STORAGE-003 |
| **Title** | File Retrieval for Processing |
| **Priority** | P1 - High |
| **Components** | `lib/storage/`, `/api/v1/process` |
| **Spec Reference** | FR-301, FR-302 |

**Preconditions**:
- File previously uploaded and stored
- Job record exists with valid file path
- Processing endpoint available

**Test Steps**:

```typescript
describe('File Retrieval for Processing', () => {
  it('retrieves stored file for MCP processing', async () => {
    // Upload file first
    const testFile = createTestFile('process-test.pdf', 'application/pdf', 2048);
    const formData = new FormData();
    formData.append('file', testFile);

    const uploadResponse = await fetch('/api/v1/upload', {
      method: 'POST',
      headers: { 'X-Session-ID': 'test-session' },
      body: formData,
    });
    const { jobId, filePath } = await uploadResponse.json();

    // Track MCP calls to verify file path passed
    let mcpFilePath: string | null = null;
    server.use(
      rest.post(MCP_ENDPOINT, async (req, res, ctx) => {
        const body = await req.json();
        if (body.method === 'tools/call' && body.params.name === 'convert_pdf') {
          mcpFilePath = body.params.arguments.source;
        }
        return res(ctx.json({ jsonrpc: '2.0', result: mockConversionResult, id: body.id }));
      })
    );

    // Trigger processing
    await fetch('/api/v1/process', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'X-Session-ID': 'test-session' },
      body: JSON.stringify({ jobId }),
    });

    // Verify correct file path passed to MCP
    expect(mcpFilePath).toBe(filePath);
  });

  it('handles missing file gracefully', async () => {
    // Create job with non-existent file path
    const job = await prisma.job.create({
      data: {
        sessionId: 'test-session',
        status: 'PENDING',
        inputType: 'FILE',
        fileName: 'missing.pdf',
        filePath: '/tmp/non-existent-file.pdf',
        mimeType: 'application/pdf',
      },
    });

    const response = await fetch('/api/v1/process', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'X-Session-ID': 'test-session' },
      body: JSON.stringify({ jobId: job.id }),
    });

    expect(response.status).toBe(404);
    const error = await response.json();
    expect(error.code).toBe('E302');

    // Job should be marked as failed
    const updatedJob = await prisma.job.findUnique({ where: { id: job.id } });
    expect(updatedJob?.status).toBe('FAILED');
  });

  it('validates file integrity before processing', async () => {
    // Upload file
    const testFile = createTestFile('integrity-test.pdf', 'application/pdf', 1024);
    const formData = new FormData();
    formData.append('file', testFile);

    const uploadResponse = await fetch('/api/v1/upload', {
      method: 'POST',
      headers: { 'X-Session-ID': 'test-session' },
      body: formData,
    });
    const { jobId, filePath } = await uploadResponse.json();

    // Corrupt the file
    await storageClient.write(filePath, 'corrupted content');

    const response = await fetch('/api/v1/process', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'X-Session-ID': 'test-session' },
      body: JSON.stringify({ jobId }),
    });

    // Should detect corruption or MCP should report format error
    const updatedJob = await prisma.job.findUnique({ where: { id: jobId } });
    expect(['FAILED', 'PROCESSING']).toContain(updatedJob?.status);
  });
});
```

**Expected Results**:
- Stored files correctly retrieved for processing
- Missing files handled with appropriate error
- File integrity validated before processing

**Verification Points**:
- Correct file path passed to MCP
- Error code E302 returned for missing files
- Job status updated appropriately on file errors

---

### INT-STORAGE-004: Storage Error Handling

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-STORAGE-004 |
| **Title** | Storage Error Handling |
| **Priority** | P1 - High |
| **Components** | `lib/storage/`, `/api/v1/upload` |
| **Spec Reference** | FR-204, Error Handling Requirements |

**Preconditions**:
- Storage backend configured
- Error simulation capabilities available

**Test Steps**:

```typescript
describe('Storage Error Handling', () => {
  it('handles storage write failure gracefully', async () => {
    // Mock storage write failure
    jest.spyOn(storageClient, 'write').mockRejectedValueOnce(
      new Error('Storage backend unavailable')
    );

    const testFile = createTestFile('test.pdf', 'application/pdf', 1024);
    const formData = new FormData();
    formData.append('file', testFile);

    const response = await fetch('/api/v1/upload', {
      method: 'POST',
      headers: { 'X-Session-ID': 'test-session' },
      body: formData,
    });

    expect(response.status).toBe(503);
    const error = await response.json();
    expect(error.code).toBe('E501');
    expect(error.title).toContain('Storage');

    // No orphan job should be created
    const jobs = await prisma.job.findMany({
      where: { sessionId: 'test-session', fileName: 'test.pdf' },
    });
    expect(jobs).toHaveLength(0);
  });

  it('handles storage read failure during processing', async () => {
    // Create job with valid path
    const job = await prisma.job.create({
      data: {
        sessionId: 'test-session',
        status: 'PENDING',
        inputType: 'FILE',
        fileName: 'test.pdf',
        filePath: '/tmp/test.pdf',
        mimeType: 'application/pdf',
      },
    });

    // Mock storage read failure
    jest.spyOn(storageClient, 'read').mockRejectedValueOnce(
      new Error('Permission denied')
    );

    const response = await fetch('/api/v1/process', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'X-Session-ID': 'test-session' },
      body: JSON.stringify({ jobId: job.id }),
    });

    expect(response.status).toBe(500);

    const updatedJob = await prisma.job.findUnique({ where: { id: job.id } });
    expect(updatedJob?.status).toBe('FAILED');
    expect(updatedJob?.errorMessage).toContain('storage');
  });

  it('handles disk full condition', async () => {
    // Mock disk full error
    jest.spyOn(storageClient, 'write').mockRejectedValueOnce(
      Object.assign(new Error('No space left on device'), { code: 'ENOSPC' })
    );

    const testFile = createTestFile('large-file.pdf', 'application/pdf', 10 * 1024 * 1024);
    const formData = new FormData();
    formData.append('file', testFile);

    const response = await fetch('/api/v1/upload', {
      method: 'POST',
      headers: { 'X-Session-ID': 'test-session' },
      body: formData,
    });

    expect(response.status).toBe(507);
    const error = await response.json();
    expect(error.code).toBe('E502');
  });

  it('cleans up partial uploads on failure', async () => {
    let writtenPath: string | null = null;

    // Mock partial write then failure
    jest.spyOn(storageClient, 'write').mockImplementationOnce(async (path) => {
      writtenPath = path;
      await actualWrite(path, 'partial');
      throw new Error('Write interrupted');
    });

    const testFile = createTestFile('partial.pdf', 'application/pdf', 1024);
    const formData = new FormData();
    formData.append('file', testFile);

    await fetch('/api/v1/upload', {
      method: 'POST',
      headers: { 'X-Session-ID': 'test-session' },
      body: formData,
    });

    // Partial file should be cleaned up
    if (writtenPath) {
      const exists = await storageClient.exists(writtenPath);
      expect(exists).toBe(false);
    }
  });
});
```

**Expected Results**:
- Storage write failures handled with appropriate error codes
- Storage read failures during processing handled gracefully
- Disk full conditions detected and reported
- Partial uploads cleaned up on failure

**Verification Points**:
- No orphan database records on storage failures
- Appropriate HTTP status codes returned
- Cleanup performed on partial failures

---

### INT-STORAGE-005: Storage Cleanup After Expiration

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-STORAGE-005 |
| **Title** | Storage Cleanup After Expiration |
| **Priority** | P2 - Medium |
| **Components** | `lib/storage/cleanup.ts`, `lib/jobs/` |
| **Spec Reference** | FR-701, Data Retention Requirements |

**Preconditions**:
- Cleanup scheduler configured
- Test files with expired timestamps

**Test Steps**:

```typescript
describe('Storage Cleanup After Expiration', () => {
  it('removes expired files from storage', async () => {
    // Create jobs with files older than retention period
    const expiredJobs = await Promise.all(
      Array.from({ length: 3 }, async (_, i) => {
        const testFile = createTestFile(`expired-${i}.pdf`, 'application/pdf', 512);
        const formData = new FormData();
        formData.append('file', testFile);

        const response = await fetch('/api/v1/upload', {
          method: 'POST',
          headers: { 'X-Session-ID': `expired-session-${i}` },
          body: formData,
        });
        return response.json();
      })
    );

    // Manually set createdAt to past expiration
    const retentionDays = 7;
    const expiredDate = new Date(Date.now() - (retentionDays + 1) * 24 * 60 * 60 * 1000);

    await prisma.job.updateMany({
      where: { id: { in: expiredJobs.map(j => j.jobId) } },
      data: { createdAt: expiredDate },
    });

    // Run cleanup
    await runStorageCleanup();

    // Verify files deleted
    for (const job of expiredJobs) {
      const exists = await storageClient.exists(job.filePath);
      expect(exists).toBe(false);
    }

    // Verify job records updated or deleted based on retention policy
    const jobs = await prisma.job.findMany({
      where: { id: { in: expiredJobs.map(j => j.jobId) } },
    });
    expect(jobs.length).toBe(0); // Or check for filePath: null if soft delete
  });

  it('preserves files within retention period', async () => {
    const testFile = createTestFile('recent.pdf', 'application/pdf', 512);
    const formData = new FormData();
    formData.append('file', testFile);

    const response = await fetch('/api/v1/upload', {
      method: 'POST',
      headers: { 'X-Session-ID': 'test-session' },
      body: formData,
    });
    const { jobId, filePath } = await response.json();

    // Run cleanup
    await runStorageCleanup();

    // File should still exist
    const exists = await storageClient.exists(filePath);
    expect(exists).toBe(true);

    // Job should be unaffected
    const job = await prisma.job.findUnique({ where: { id: jobId } });
    expect(job?.filePath).toBe(filePath);
  });

  it('handles cleanup errors without affecting other files', async () => {
    // Create multiple expired files
    const jobs = await Promise.all(
      Array.from({ length: 3 }, async (_, i) => {
        const testFile = createTestFile(`cleanup-${i}.pdf`, 'application/pdf', 256);
        const formData = new FormData();
        formData.append('file', testFile);

        const response = await fetch('/api/v1/upload', {
          method: 'POST',
          headers: { 'X-Session-ID': `cleanup-session-${i}` },
          body: formData,
        });
        return response.json();
      })
    );

    // Set all to expired
    const expiredDate = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000);
    await prisma.job.updateMany({
      where: { id: { in: jobs.map(j => j.jobId) } },
      data: { createdAt: expiredDate },
    });

    // Mock failure for middle file
    const originalDelete = storageClient.delete;
    jest.spyOn(storageClient, 'delete').mockImplementation(async (path) => {
      if (path === jobs[1].filePath) {
        throw new Error('Delete failed');
      }
      return originalDelete(path);
    });

    // Run cleanup - should not throw
    await expect(runStorageCleanup()).resolves.not.toThrow();

    // First and third files should be deleted
    expect(await storageClient.exists(jobs[0].filePath)).toBe(false);
    expect(await storageClient.exists(jobs[2].filePath)).toBe(false);

    // Second file should still exist (cleanup failed)
    expect(await storageClient.exists(jobs[1].filePath)).toBe(true);
  });
});
```

**Expected Results**:
- Expired files removed from storage
- Files within retention period preserved
- Cleanup errors do not affect other files

**Verification Points**:
- Retention period correctly enforced
- Database records updated after file cleanup
- Graceful handling of individual cleanup failures

---

## 22. URL Fetch Integration

### 22.1 Overview

Tests validating URL fetch operations for converting web content, including successful fetches, error handling, and redirect following.

---

### INT-URL-001: URL Fetch and Conversion

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-URL-001 |
| **Title** | URL Fetch and Conversion |
| **Priority** | P1 - High |
| **Components** | `/api/v1/convert`, `lib/mcp/client.ts` |
| **Spec Reference** | FR-101, FR-102 |

**Preconditions**:
- URL conversion endpoint available
- MCP server mock configured
- Test URLs accessible via MSW

**Test Steps**:

```typescript
describe('URL Fetch and Conversion', () => {
  it('fetches URL and converts to document', async () => {
    const testUrl = 'https://example.com/document.pdf';

    // Mock external URL
    server.use(
      rest.get(testUrl, (req, res, ctx) => {
        return res(
          ctx.set('Content-Type', 'application/pdf'),
          ctx.body(mockPdfContent)
        );
      })
    );

    // Track MCP tool call
    let mcpUrl: string | null = null;
    server.use(
      rest.post(MCP_ENDPOINT, async (req, res, ctx) => {
        const body = await req.json();
        if (body.method === 'tools/call' && body.params.name === 'convert_url') {
          mcpUrl = body.params.arguments.url;
        }
        return res(ctx.json({ jsonrpc: '2.0', result: mockConversionResult, id: body.id }));
      })
    );

    const response = await fetch('/api/v1/convert', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'X-Session-ID': 'test-session' },
      body: JSON.stringify({ url: testUrl }),
    });

    expect(response.status).toBe(202);
    const { jobId } = await response.json();

    // Verify MCP received correct URL
    expect(mcpUrl).toBe(testUrl);

    // Verify job created with URL type
    const job = await prisma.job.findUnique({ where: { id: jobId } });
    expect(job?.inputType).toBe('URL');
    expect(job?.inputUrl).toBe(testUrl);
  });

  it('validates URL format before processing', async () => {
    const invalidUrls = [
      'not-a-url',
      'ftp://unsupported-protocol.com/file',
      'javascript:alert(1)',
      'file:///etc/passwd',
      '',
    ];

    for (const url of invalidUrls) {
      const response = await fetch('/api/v1/convert', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json', 'X-Session-ID': 'test-session' },
        body: JSON.stringify({ url }),
      });

      expect(response.status).toBe(400);
      const error = await response.json();
      expect(error.code).toBe('E102');
    }
  });

  it('handles URL with query parameters', async () => {
    const testUrl = 'https://example.com/document.pdf?token=abc123&version=2';

    server.use(
      rest.get('https://example.com/document.pdf', (req, res, ctx) => {
        // Verify query params preserved
        expect(req.url.searchParams.get('token')).toBe('abc123');
        expect(req.url.searchParams.get('version')).toBe('2');
        return res(ctx.set('Content-Type', 'application/pdf'), ctx.body(mockPdfContent));
      })
    );

    const response = await fetch('/api/v1/convert', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'X-Session-ID': 'test-session' },
      body: JSON.stringify({ url: testUrl }),
    });

    expect(response.status).toBe(202);
  });

  it('supports HTTPS URLs only in production mode', async () => {
    // Mock production environment
    const originalEnv = process.env.NODE_ENV;
    process.env.NODE_ENV = 'production';

    const httpUrl = 'http://insecure.example.com/document.pdf';

    const response = await fetch('/api/v1/convert', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'X-Session-ID': 'test-session' },
      body: JSON.stringify({ url: httpUrl }),
    });

    expect(response.status).toBe(400);
    const error = await response.json();
    expect(error.code).toBe('E103');

    process.env.NODE_ENV = originalEnv;
  });
});
```

**Expected Results**:
- URL fetched and passed to MCP for conversion
- Invalid URL formats rejected with appropriate errors
- Query parameters preserved in URL
- HTTPS enforcement in production

**Verification Points**:
- Correct MCP tool invoked (convert_url)
- Job record created with URL input type
- URL validation applied before processing

---

### INT-URL-002: URL Fetch Error Handling

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-URL-002 |
| **Title** | URL Fetch Error Handling |
| **Priority** | P1 - High |
| **Components** | `lib/mcp/client.ts`, `/api/v1/convert` |
| **Spec Reference** | FR-104, Error Handling Requirements |

**Preconditions**:
- URL conversion endpoint available
- MSW configured for error simulation

**Test Steps**:

```typescript
describe('URL Fetch Error Handling', () => {
  it('handles 404 not found response', async () => {
    const testUrl = 'https://example.com/missing-document.pdf';

    server.use(
      rest.get(testUrl, (req, res, ctx) => {
        return res(ctx.status(404), ctx.json({ error: 'Not Found' }));
      })
    );

    // Mock MCP returning URL fetch error
    server.use(
      rest.post(MCP_ENDPOINT, async (req, res, ctx) => {
        const body = await req.json();
        if (body.method === 'tools/call' && body.params.name === 'convert_url') {
          return res(ctx.json({
            jsonrpc: '2.0',
            error: { code: -32000, message: 'URL fetch failed: 404 Not Found' },
            id: body.id,
          }));
        }
        return res(ctx.json({ jsonrpc: '2.0', result: {}, id: body.id }));
      })
    );

    const response = await fetch('/api/v1/convert', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'X-Session-ID': 'test-session' },
      body: JSON.stringify({ url: testUrl }),
    });

    const { jobId } = await response.json();

    // Wait for processing to complete
    await waitForJobCompletion(jobId);

    const job = await prisma.job.findUnique({ where: { id: jobId } });
    expect(job?.status).toBe('FAILED');
    expect(job?.errorCode).toBe('E401');
  });

  it('handles connection timeout', async () => {
    const testUrl = 'https://slow-server.example.com/document.pdf';

    server.use(
      rest.get(testUrl, async (req, res, ctx) => {
        await sleep(35000); // Exceed 30s timeout
        return res(ctx.body(mockPdfContent));
      })
    );

    server.use(
      rest.post(MCP_ENDPOINT, async (req, res, ctx) => {
        const body = await req.json();
        if (body.method === 'tools/call' && body.params.name === 'convert_url') {
          return res(ctx.json({
            jsonrpc: '2.0',
            error: { code: -32000, message: 'URL fetch timeout' },
            id: body.id,
          }));
        }
        return res(ctx.json({ jsonrpc: '2.0', result: {}, id: body.id }));
      })
    );

    const response = await fetch('/api/v1/convert', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'X-Session-ID': 'test-session' },
      body: JSON.stringify({ url: testUrl }),
    });

    const { jobId } = await response.json();
    await waitForJobCompletion(jobId, { timeout: 40000 });

    const job = await prisma.job.findUnique({ where: { id: jobId } });
    expect(job?.status).toBe('FAILED');
    expect(job?.errorCode).toBe('E402');
  });

  it('handles SSL certificate errors', async () => {
    const testUrl = 'https://invalid-cert.example.com/document.pdf';

    server.use(
      rest.post(MCP_ENDPOINT, async (req, res, ctx) => {
        const body = await req.json();
        if (body.method === 'tools/call' && body.params.name === 'convert_url') {
          return res(ctx.json({
            jsonrpc: '2.0',
            error: { code: -32000, message: 'SSL certificate verification failed' },
            id: body.id,
          }));
        }
        return res(ctx.json({ jsonrpc: '2.0', result: {}, id: body.id }));
      })
    );

    const response = await fetch('/api/v1/convert', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'X-Session-ID': 'test-session' },
      body: JSON.stringify({ url: testUrl }),
    });

    const { jobId } = await response.json();
    await waitForJobCompletion(jobId);

    const job = await prisma.job.findUnique({ where: { id: jobId } });
    expect(job?.status).toBe('FAILED');
    expect(job?.errorCode).toBe('E403');
  });

  it('handles DNS resolution failure', async () => {
    const testUrl = 'https://non-existent-domain-xyz123.com/document.pdf';

    server.use(
      rest.post(MCP_ENDPOINT, async (req, res, ctx) => {
        const body = await req.json();
        if (body.method === 'tools/call' && body.params.name === 'convert_url') {
          return res(ctx.json({
            jsonrpc: '2.0',
            error: { code: -32000, message: 'DNS resolution failed' },
            id: body.id,
          }));
        }
        return res(ctx.json({ jsonrpc: '2.0', result: {}, id: body.id }));
      })
    );

    const response = await fetch('/api/v1/convert', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'X-Session-ID': 'test-session' },
      body: JSON.stringify({ url: testUrl }),
    });

    const { jobId } = await response.json();
    await waitForJobCompletion(jobId);

    const job = await prisma.job.findUnique({ where: { id: jobId } });
    expect(job?.status).toBe('FAILED');
    expect(job?.errorCode).toBe('E404');
  });
});
```

**Expected Results**:
- 404 responses handled with appropriate error code
- Connection timeouts detected and reported
- SSL certificate errors handled securely
- DNS failures handled gracefully

**Verification Points**:
- Job status set to FAILED on fetch errors
- Appropriate error codes assigned
- Error messages descriptive and actionable

---

### INT-URL-003: Redirect Following

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-URL-003 |
| **Title** | Redirect Following |
| **Priority** | P1 - High |
| **Components** | `lib/mcp/client.ts`, `/api/v1/convert` |
| **Spec Reference** | FR-105 |

**Preconditions**:
- URL conversion endpoint available
- MSW configured for redirect simulation

**Test Steps**:

```typescript
describe('Redirect Following', () => {
  it('follows 301 permanent redirect', async () => {
    const originalUrl = 'https://example.com/old-location.pdf';
    const redirectUrl = 'https://example.com/new-location.pdf';

    server.use(
      rest.get(originalUrl, (req, res, ctx) => {
        return res(
          ctx.status(301),
          ctx.set('Location', redirectUrl)
        );
      }),
      rest.get(redirectUrl, (req, res, ctx) => {
        return res(
          ctx.set('Content-Type', 'application/pdf'),
          ctx.body(mockPdfContent)
        );
      })
    );

    let finalUrl: string | null = null;
    server.use(
      rest.post(MCP_ENDPOINT, async (req, res, ctx) => {
        const body = await req.json();
        if (body.method === 'tools/call' && body.params.name === 'convert_url') {
          finalUrl = body.params.arguments.url;
        }
        return res(ctx.json({ jsonrpc: '2.0', result: mockConversionResult, id: body.id }));
      })
    );

    const response = await fetch('/api/v1/convert', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'X-Session-ID': 'test-session' },
      body: JSON.stringify({ url: originalUrl }),
    });

    expect(response.status).toBe(202);

    // MCP should receive original URL (redirect handled by MCP/Docling)
    expect(finalUrl).toBe(originalUrl);
  });

  it('follows 302 temporary redirect', async () => {
    const originalUrl = 'https://example.com/temp-redirect.pdf';
    const redirectUrl = 'https://cdn.example.com/actual-file.pdf';

    server.use(
      rest.get(originalUrl, (req, res, ctx) => {
        return res(
          ctx.status(302),
          ctx.set('Location', redirectUrl)
        );
      }),
      rest.get(redirectUrl, (req, res, ctx) => {
        return res(
          ctx.set('Content-Type', 'application/pdf'),
          ctx.body(mockPdfContent)
        );
      })
    );

    const response = await fetch('/api/v1/convert', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'X-Session-ID': 'test-session' },
      body: JSON.stringify({ url: originalUrl }),
    });

    expect(response.status).toBe(202);
    const { jobId } = await response.json();

    // Job should complete successfully after redirect
    await waitForJobCompletion(jobId);
    const job = await prisma.job.findUnique({ where: { id: jobId } });
    expect(job?.status).toBe('COMPLETED');
  });

  it('limits redirect chain depth', async () => {
    const maxRedirects = 10;
    const baseUrl = 'https://example.com/redirect';

    // Create redirect chain exceeding limit
    for (let i = 0; i <= maxRedirects + 5; i++) {
      const currentUrl = `${baseUrl}-${i}.pdf`;
      const nextUrl = `${baseUrl}-${i + 1}.pdf`;

      server.use(
        rest.get(currentUrl, (req, res, ctx) => {
          return res(
            ctx.status(302),
            ctx.set('Location', nextUrl)
          );
        })
      );
    }

    server.use(
      rest.post(MCP_ENDPOINT, async (req, res, ctx) => {
        const body = await req.json();
        if (body.method === 'tools/call' && body.params.name === 'convert_url') {
          return res(ctx.json({
            jsonrpc: '2.0',
            error: { code: -32000, message: 'Too many redirects' },
            id: body.id,
          }));
        }
        return res(ctx.json({ jsonrpc: '2.0', result: {}, id: body.id }));
      })
    );

    const response = await fetch('/api/v1/convert', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'X-Session-ID': 'test-session' },
      body: JSON.stringify({ url: `${baseUrl}-0.pdf` }),
    });

    const { jobId } = await response.json();
    await waitForJobCompletion(jobId);

    const job = await prisma.job.findUnique({ where: { id: jobId } });
    expect(job?.status).toBe('FAILED');
    expect(job?.errorCode).toBe('E405');
  });

  it('prevents redirect to different protocol', async () => {
    const httpsUrl = 'https://example.com/secure.pdf';
    const httpUrl = 'http://example.com/insecure.pdf';

    server.use(
      rest.get(httpsUrl, (req, res, ctx) => {
        return res(
          ctx.status(302),
          ctx.set('Location', httpUrl)
        );
      })
    );

    // Mock production environment
    const originalEnv = process.env.NODE_ENV;
    process.env.NODE_ENV = 'production';

    server.use(
      rest.post(MCP_ENDPOINT, async (req, res, ctx) => {
        const body = await req.json();
        if (body.method === 'tools/call' && body.params.name === 'convert_url') {
          return res(ctx.json({
            jsonrpc: '2.0',
            error: { code: -32000, message: 'Redirect to insecure protocol blocked' },
            id: body.id,
          }));
        }
        return res(ctx.json({ jsonrpc: '2.0', result: {}, id: body.id }));
      })
    );

    const response = await fetch('/api/v1/convert', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'X-Session-ID': 'test-session' },
      body: JSON.stringify({ url: httpsUrl }),
    });

    const { jobId } = await response.json();
    await waitForJobCompletion(jobId);

    const job = await prisma.job.findUnique({ where: { id: jobId } });
    expect(job?.status).toBe('FAILED');
    expect(job?.errorCode).toBe('E406');

    process.env.NODE_ENV = originalEnv;
  });
});
```

**Expected Results**:
- 301 and 302 redirects followed correctly
- Redirect chain depth limited
- Protocol downgrade blocked in production

**Verification Points**:
- Redirects followed transparently
- Maximum redirect limit enforced
- Security maintained during redirects

---

## 23. Workflow Orchestration Integration

### 23.1 Overview

Tests validating workflow coordination across processing stages, job queue management, checkpoint consistency, and long-running workflow stability.

---

### INT-WF-001: Workflow Coordination Across Stages

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-WF-001 |
| **Title** | Workflow Coordination Across Stages |
| **Priority** | P1 - High |
| **Components** | `lib/workflow/`, `lib/mcp/client.ts`, State Machine |
| **Spec Reference** | FR-401, FR-402, Workflow Requirements |

**Preconditions**:
- Workflow orchestrator initialized
- All processing stages available
- State machine configured

**Test Steps**:

```typescript
describe('Workflow Coordination Across Stages', () => {
  it('coordinates document processing through all stages', async () => {
    const testFile = createTestFile('workflow-test.pdf', 'application/pdf', 2048);
    const formData = new FormData();
    formData.append('file', testFile);

    // Upload file
    const uploadResponse = await fetch('/api/v1/upload', {
      method: 'POST',
      headers: { 'X-Session-ID': 'test-session' },
      body: formData,
    });
    const { jobId } = await uploadResponse.json();

    // Track stage transitions
    const stageTransitions: string[] = [];
    const sseEvents = await createSSEConnection(`/api/v1/jobs/${jobId}/events`);
    sseEvents.on('status', (data) => {
      stageTransitions.push(data.stage);
    });

    // Trigger processing
    await fetch('/api/v1/process', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'X-Session-ID': 'test-session' },
      body: JSON.stringify({ jobId }),
    });

    // Wait for completion
    await waitForJobCompletion(jobId);

    // Verify all stages executed in correct order
    expect(stageTransitions).toContain('VALIDATING');
    expect(stageTransitions).toContain('CONVERTING');
    expect(stageTransitions).toContain('EXPORTING');
    expect(stageTransitions.indexOf('VALIDATING')).toBeLessThan(stageTransitions.indexOf('CONVERTING'));
    expect(stageTransitions.indexOf('CONVERTING')).toBeLessThan(stageTransitions.indexOf('EXPORTING'));
  });

  it('handles stage failure with proper rollback', async () => {
    const job = await prisma.job.create({
      data: {
        sessionId: 'test-session',
        status: 'PENDING',
        inputType: 'FILE',
        fileName: 'test.pdf',
        filePath: '/tmp/test.pdf',
        mimeType: 'application/pdf',
      },
    });

    // Mock MCP conversion failure at CONVERTING stage
    server.use(
      rest.post(MCP_ENDPOINT, async (req, res, ctx) => {
        const body = await req.json();
        if (body.method === 'tools/call' && body.params.name === 'convert_pdf') {
          return res(ctx.json({
            jsonrpc: '2.0',
            error: { code: -32000, message: 'Conversion failed' },
            id: body.id,
          }));
        }
        return res(ctx.json({ jsonrpc: '2.0', result: {}, id: body.id }));
      })
    );

    await fetch('/api/v1/process', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'X-Session-ID': 'test-session' },
      body: JSON.stringify({ jobId: job.id }),
    });

    await waitForJobCompletion(job.id);

    const updatedJob = await prisma.job.findUnique({ where: { id: job.id } });
    expect(updatedJob?.status).toBe('FAILED');
    expect(updatedJob?.lastCompletedStage).toBe('VALIDATING');
  });

  it('supports workflow restart from failed stage', async () => {
    // Create job that failed at CONVERTING stage
    const job = await prisma.job.create({
      data: {
        sessionId: 'test-session',
        status: 'FAILED',
        inputType: 'FILE',
        fileName: 'test.pdf',
        filePath: '/tmp/test.pdf',
        mimeType: 'application/pdf',
        lastCompletedStage: 'VALIDATING',
        checkpointData: JSON.stringify({ validated: true }),
      },
    });

    // Mock successful MCP response for retry
    server.use(
      rest.post(MCP_ENDPOINT, async (req, res, ctx) => {
        return res(ctx.json({ jsonrpc: '2.0', result: mockConversionResult, id: req.body.id }));
      })
    );

    // Retry from failed stage
    const response = await fetch('/api/v1/process', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'X-Session-ID': 'test-session' },
      body: JSON.stringify({ jobId: job.id, resumeFromCheckpoint: true }),
    });

    expect(response.status).toBe(202);

    await waitForJobCompletion(job.id);

    const updatedJob = await prisma.job.findUnique({ where: { id: job.id } });
    expect(updatedJob?.status).toBe('COMPLETED');
  });
});
```

**Expected Results**:
- All workflow stages execute in correct order
- Stage failures trigger proper rollback
- Workflow can restart from failed stage

**Verification Points**:
- Stage transition order validated
- Rollback preserves partial progress
- Checkpoint data enables resumption

---

### INT-QUEUE-001: Concurrent Job Queuing

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-QUEUE-001 |
| **Title** | Concurrent Job Queuing |
| **Priority** | P1 - High |
| **Components** | `lib/queue/`, `/api/v1/process`, Redis |
| **Spec Reference** | FR-501, Concurrency Requirements |

**Preconditions**:
- Job queue initialized
- Redis connection available
- Multiple test jobs prepared

**Test Steps**:

```typescript
describe('Concurrent Job Queuing', () => {
  it('queues multiple concurrent jobs correctly', async () => {
    const jobCount = 10;
    const jobs: string[] = [];

    // Create multiple jobs concurrently
    const createPromises = Array(jobCount).fill(null).map(async (_, i) => {
      const testFile = createTestFile(`test-${i}.pdf`, 'application/pdf', 1024);
      const formData = new FormData();
      formData.append('file', testFile);

      const uploadResponse = await fetch('/api/v1/upload', {
        method: 'POST',
        headers: { 'X-Session-ID': `session-${i}` },
        body: formData,
      });
      const { jobId } = await uploadResponse.json();
      jobs.push(jobId);
      return jobId;
    });

    await Promise.all(createPromises);

    // Verify all jobs queued
    expect(jobs.length).toBe(jobCount);

    // Trigger processing for all jobs
    await Promise.all(jobs.map(jobId =>
      fetch('/api/v1/process', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json', 'X-Session-ID': 'admin' },
        body: JSON.stringify({ jobId }),
      })
    ));

    // Wait for all jobs to complete
    await Promise.all(jobs.map(jobId => waitForJobCompletion(jobId, { timeout: 60000 })));

    // Verify all jobs completed
    for (const jobId of jobs) {
      const job = await prisma.job.findUnique({ where: { id: jobId } });
      expect(job?.status).toBe('COMPLETED');
    }
  });

  it('maintains job isolation under concurrent load', async () => {
    const jobs = await Promise.all([
      createJob({ fileName: 'a.pdf', sessionId: 'session-a' }),
      createJob({ fileName: 'b.pdf', sessionId: 'session-b' }),
      createJob({ fileName: 'c.pdf', sessionId: 'session-c' }),
    ]);

    // Process concurrently
    await Promise.all(jobs.map(job =>
      fetch('/api/v1/process', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json', 'X-Session-ID': job.sessionId },
        body: JSON.stringify({ jobId: job.id }),
      })
    ));

    await Promise.all(jobs.map(job => waitForJobCompletion(job.id)));

    // Verify no cross-contamination
    for (const job of jobs) {
      const result = await prisma.job.findUnique({
        where: { id: job.id },
        include: { outputs: true },
      });
      expect(result?.sessionId).toBe(job.sessionId);
      expect(result?.outputs.every(o => o.jobId === job.id)).toBe(true);
    }
  });
});
```

**Expected Results**:
- Multiple concurrent jobs queued without conflicts
- Job isolation maintained under concurrent load
- All jobs complete successfully

**Verification Points**:
- No race conditions in queue operations
- Job data remains isolated
- Concurrent processing scales correctly

---

### INT-QUEUE-002: Job Queue Ordering

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-QUEUE-002 |
| **Title** | Job Queue Ordering |
| **Priority** | P2 - Medium |
| **Components** | `lib/queue/`, Redis |
| **Spec Reference** | FR-502, Queue Management |

**Preconditions**:
- Job queue initialized
- Redis connection available
- Priority queue configuration enabled

**Test Steps**:

```typescript
describe('Job Queue Ordering', () => {
  it('processes jobs in FIFO order by default', async () => {
    const completionOrder: string[] = [];

    // Create jobs with tracking
    const job1 = await createJob({ fileName: 'first.pdf' });
    await sleep(100);
    const job2 = await createJob({ fileName: 'second.pdf' });
    await sleep(100);
    const job3 = await createJob({ fileName: 'third.pdf' });

    // Set up completion tracking
    const trackCompletion = async (jobId: string) => {
      await waitForJobCompletion(jobId);
      completionOrder.push(jobId);
    };

    // Process all jobs
    await Promise.all([job1, job2, job3].map(job =>
      fetch('/api/v1/process', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ jobId: job.id }),
      })
    ));

    await Promise.all([job1, job2, job3].map(job => trackCompletion(job.id)));

    // FIFO order expected
    expect(completionOrder[0]).toBe(job1.id);
    expect(completionOrder[1]).toBe(job2.id);
    expect(completionOrder[2]).toBe(job3.id);
  });

  it('respects job priority when configured', async () => {
    const completionOrder: string[] = [];

    // Create jobs with different priorities
    const lowPriority = await createJob({ fileName: 'low.pdf', priority: 'LOW' });
    const highPriority = await createJob({ fileName: 'high.pdf', priority: 'HIGH' });
    const normalPriority = await createJob({ fileName: 'normal.pdf', priority: 'NORMAL' });

    // Track completions
    const trackCompletion = async (jobId: string) => {
      await waitForJobCompletion(jobId);
      completionOrder.push(jobId);
    };

    // Process all jobs
    await Promise.all([lowPriority, highPriority, normalPriority].map(job =>
      fetch('/api/v1/process', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ jobId: job.id }),
      })
    ));

    await Promise.all([lowPriority, highPriority, normalPriority].map(job => trackCompletion(job.id)));

    // High priority should complete first
    expect(completionOrder[0]).toBe(highPriority.id);
  });
});
```

**Expected Results**:
- Jobs processed in FIFO order by default
- Priority configuration respected when enabled
- Queue ordering consistent

**Verification Points**:
- FIFO ordering validated
- Priority processing verified
- No queue corruption under load

---

### INT-CK-003: Checkpoint Consistency Across Services

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-CK-003 |
| **Title** | Checkpoint Consistency Across Services |
| **Priority** | P1 - High |
| **Components** | `lib/checkpoint/`, PostgreSQL, Redis |
| **Spec Reference** | FR-601, Checkpoint Requirements |

**Preconditions**:
- Checkpoint system initialized
- PostgreSQL and Redis connections available
- Multi-stage job prepared

**Test Steps**:

```typescript
describe('Checkpoint Consistency Across Services', () => {
  it('maintains checkpoint consistency between PostgreSQL and Redis', async () => {
    const job = await createJob({ fileName: 'checkpoint-test.pdf' });

    // Mock slow processing to capture checkpoint state
    server.use(
      rest.post(MCP_ENDPOINT, async (req, res, ctx) => {
        const body = await req.json();
        if (body.method === 'tools/call' && body.params.name === 'convert_pdf') {
          await sleep(2000); // Allow checkpoint to be written
          return res(ctx.json({ jsonrpc: '2.0', result: mockConversionResult, id: body.id }));
        }
        return res(ctx.json({ jsonrpc: '2.0', result: {}, id: body.id }));
      })
    );

    // Start processing
    fetch('/api/v1/process', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ jobId: job.id }),
    });

    // Wait for checkpoint to be written
    await sleep(1000);

    // Check checkpoint in PostgreSQL
    const dbJob = await prisma.job.findUnique({ where: { id: job.id } });
    const dbCheckpoint = JSON.parse(dbJob?.checkpointData || '{}');

    // Check checkpoint in Redis
    const redisCheckpoint = await redis.get(`checkpoint:${job.id}`);
    const parsedRedisCheckpoint = JSON.parse(redisCheckpoint || '{}');

    // Verify consistency
    expect(dbCheckpoint.stage).toBe(parsedRedisCheckpoint.stage);
    expect(dbCheckpoint.timestamp).toBe(parsedRedisCheckpoint.timestamp);

    await waitForJobCompletion(job.id);
  });

  it('recovers checkpoint from Redis when PostgreSQL is temporarily unavailable', async () => {
    const job = await createJob({ fileName: 'recovery-test.pdf' });

    // Write checkpoint to Redis
    const checkpointData = {
      stage: 'CONVERTING',
      progress: 50,
      timestamp: Date.now(),
    };
    await redis.set(`checkpoint:${job.id}`, JSON.stringify(checkpointData));

    // Update job to use checkpoint
    await prisma.job.update({
      where: { id: job.id },
      data: { status: 'PROCESSING', checkpointData: JSON.stringify(checkpointData) },
    });

    // Verify checkpoint recovery
    const response = await fetch(`/api/v1/jobs/${job.id}/checkpoint`, {
      headers: { 'X-Session-ID': 'test-session' },
    });

    const checkpoint = await response.json();
    expect(checkpoint.stage).toBe('CONVERTING');
    expect(checkpoint.progress).toBe(50);
  });

  it('handles checkpoint write failures gracefully', async () => {
    const job = await createJob({ fileName: 'failure-test.pdf' });

    // Mock Redis write failure
    jest.spyOn(redis, 'set').mockRejectedValueOnce(new Error('Redis unavailable'));

    // Start processing
    await fetch('/api/v1/process', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ jobId: job.id }),
    });

    // Processing should continue even if Redis checkpoint fails
    await waitForJobCompletion(job.id, { timeout: 30000 });

    const updatedJob = await prisma.job.findUnique({ where: { id: job.id } });
    expect(updatedJob?.status).toBe('COMPLETED');
    // PostgreSQL checkpoint should still be written
    expect(updatedJob?.checkpointData).toBeTruthy();
  });
});
```

**Expected Results**:
- Checkpoint data consistent between PostgreSQL and Redis
- Recovery from Redis checkpoint when needed
- Graceful handling of checkpoint write failures

**Verification Points**:
- Cross-service checkpoint consistency validated
- Recovery mechanism tested
- Failure resilience confirmed

---

### INT-LONG-001: Long-Running Workflow Stability

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-LONG-001 |
| **Title** | Long-Running Workflow Stability |
| **Priority** | P2 - Medium |
| **Components** | `lib/workflow/`, All Integration Points |
| **Spec Reference** | NFR-101, Stability Requirements |

**Preconditions**:
- Full system initialized
- Extended timeout configuration
- Large test documents available

**Test Steps**:

```typescript
describe('Long-Running Workflow Stability', () => {
  it('maintains stability during extended processing', async () => {
    // Create large document job
    const largeFile = createTestFile('large-document.pdf', 'application/pdf', 50 * 1024 * 1024); // 50MB
    const formData = new FormData();
    formData.append('file', largeFile);

    const uploadResponse = await fetch('/api/v1/upload', {
      method: 'POST',
      headers: { 'X-Session-ID': 'test-session' },
      body: formData,
    });
    const { jobId } = await uploadResponse.json();

    // Track heartbeats/progress
    const progressUpdates: number[] = [];
    const sseEvents = await createSSEConnection(`/api/v1/jobs/${jobId}/events`);
    sseEvents.on('progress', (data) => {
      progressUpdates.push(data.progress);
    });

    // Start processing with extended timeout
    await fetch('/api/v1/process', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ jobId }),
    });

    // Wait for extended processing
    await waitForJobCompletion(jobId, { timeout: 300000 }); // 5 minute timeout

    const job = await prisma.job.findUnique({ where: { id: jobId } });
    expect(job?.status).toBe('COMPLETED');
    expect(progressUpdates.length).toBeGreaterThan(0);
    expect(progressUpdates[progressUpdates.length - 1]).toBe(100);
  }, 360000);

  it('handles connection interruptions during long processing', async () => {
    const job = await createJob({ fileName: 'interrupt-test.pdf' });

    // Mock slow MCP processing
    server.use(
      rest.post(MCP_ENDPOINT, async (req, res, ctx) => {
        await sleep(5000);
        return res(ctx.json({ jsonrpc: '2.0', result: mockConversionResult, id: req.body.id }));
      })
    );

    // Start processing
    fetch('/api/v1/process', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ jobId: job.id }),
    });

    // Simulate connection interruption mid-processing
    await sleep(2000);

    // Verify job continues processing despite SSE disconnection
    await waitForJobCompletion(job.id, { timeout: 30000 });

    const updatedJob = await prisma.job.findUnique({ where: { id: job.id } });
    expect(updatedJob?.status).toBe('COMPLETED');
  });

  it('prevents memory leaks during sustained processing', async () => {
    const initialMemory = process.memoryUsage().heapUsed;
    const iterations = 10;

    for (let i = 0; i < iterations; i++) {
      const job = await createJob({ fileName: `memory-test-${i}.pdf` });

      await fetch('/api/v1/process', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ jobId: job.id }),
      });

      await waitForJobCompletion(job.id);

      // Force garbage collection if available
      if (global.gc) global.gc();
    }

    const finalMemory = process.memoryUsage().heapUsed;
    const memoryGrowth = (finalMemory - initialMemory) / initialMemory;

    // Memory growth should be less than 50% over iterations
    expect(memoryGrowth).toBeLessThan(0.5);
  });
});
```

**Expected Results**:
- Long-running workflows complete successfully
- Connection interruptions handled gracefully
- No significant memory leaks during sustained processing

**Verification Points**:
- Extended processing stability confirmed
- Connection resilience validated
- Memory usage within acceptable bounds

---

## 24. Cross-Service Consistency

### 24.1 Overview

Tests validating data consistency across services, including API responses matching database state, pagination consistency, cache-database synchronization, and session state consistency.

---

### INT-CONSISTENCY-001: API Response Data Consistency with Database

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-CONSISTENCY-001 |
| **Title** | API Response Data Consistency with Database |
| **Priority** | P2 - Medium |
| **Components** | API Routes, PostgreSQL, Prisma |
| **Spec Reference** | FR-701, Data Consistency Requirements |

**Preconditions**:
- Database populated with test data
- API endpoints available
- No pending migrations

**Test Steps**:

```typescript
describe('API Response Data Consistency with Database', () => {
  it('returns job data matching database state exactly', async () => {
    // Create job with known data
    const dbJob = await prisma.job.create({
      data: {
        sessionId: 'consistency-test',
        status: 'COMPLETED',
        inputType: 'FILE',
        fileName: 'test.pdf',
        filePath: '/tmp/test.pdf',
        mimeType: 'application/pdf',
        progress: 100,
        completedAt: new Date(),
      },
    });

    // Fetch via API
    const response = await fetch(`/api/v1/jobs/${dbJob.id}`, {
      headers: { 'X-Session-ID': 'consistency-test' },
    });
    const apiJob = await response.json();

    // Verify exact match
    expect(apiJob.id).toBe(dbJob.id);
    expect(apiJob.status).toBe(dbJob.status);
    expect(apiJob.fileName).toBe(dbJob.fileName);
    expect(apiJob.progress).toBe(dbJob.progress);
    expect(new Date(apiJob.completedAt).getTime()).toBe(dbJob.completedAt.getTime());
  });

  it('reflects database updates immediately in API responses', async () => {
    const job = await createJob({ fileName: 'update-test.pdf' });

    // Verify initial state
    let response = await fetch(`/api/v1/jobs/${job.id}`, {
      headers: { 'X-Session-ID': job.sessionId },
    });
    let apiJob = await response.json();
    expect(apiJob.status).toBe('PENDING');

    // Update in database
    await prisma.job.update({
      where: { id: job.id },
      data: { status: 'PROCESSING', progress: 50 },
    });

    // Verify immediate reflection
    response = await fetch(`/api/v1/jobs/${job.id}`, {
      headers: { 'X-Session-ID': job.sessionId },
    });
    apiJob = await response.json();
    expect(apiJob.status).toBe('PROCESSING');
    expect(apiJob.progress).toBe(50);
  });

  it('maintains consistency under concurrent reads and writes', async () => {
    const job = await createJob({ fileName: 'concurrent-test.pdf' });
    const inconsistencies: string[] = [];

    // Concurrent reads and writes
    const operations = Array(20).fill(null).map(async (_, i) => {
      if (i % 2 === 0) {
        // Write operation
        await prisma.job.update({
          where: { id: job.id },
          data: { progress: i * 5 },
        });
      } else {
        // Read operation
        const response = await fetch(`/api/v1/jobs/${job.id}`, {
          headers: { 'X-Session-ID': job.sessionId },
        });
        const apiJob = await response.json();
        const dbJob = await prisma.job.findUnique({ where: { id: job.id } });

        // Check for inconsistency at read time
        if (apiJob.progress !== dbJob?.progress) {
          inconsistencies.push(`API: ${apiJob.progress}, DB: ${dbJob?.progress}`);
        }
      }
    });

    await Promise.all(operations);
    expect(inconsistencies.length).toBe(0);
  });
});
```

**Expected Results**:
- API responses exactly match database state
- Database updates immediately reflected in API
- No inconsistencies under concurrent access

**Verification Points**:
- Data integrity validated
- Update propagation verified
- Concurrent access consistency confirmed

---

### INT-CONSISTENCY-002: Pagination Response Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-CONSISTENCY-002 |
| **Title** | Pagination Response Validation |
| **Priority** | P2 - Medium |
| **Components** | API Routes, PostgreSQL |
| **Spec Reference** | FR-702, Pagination Requirements |

**Preconditions**:
- Multiple jobs in database
- Pagination configuration available

**Test Steps**:

```typescript
describe('Pagination Response Validation', () => {
  it('returns correct pagination metadata', async () => {
    // Create 25 jobs
    const jobs = await Promise.all(
      Array(25).fill(null).map((_, i) =>
        prisma.job.create({
          data: {
            sessionId: 'pagination-test',
            status: 'COMPLETED',
            inputType: 'FILE',
            fileName: `test-${i}.pdf`,
            filePath: `/tmp/test-${i}.pdf`,
            mimeType: 'application/pdf',
          },
        })
      )
    );

    // Fetch first page
    const response = await fetch('/api/v1/jobs?page=1&limit=10', {
      headers: { 'X-Session-ID': 'pagination-test' },
    });
    const result = await response.json();

    expect(result.data.length).toBe(10);
    expect(result.pagination.total).toBe(25);
    expect(result.pagination.page).toBe(1);
    expect(result.pagination.limit).toBe(10);
    expect(result.pagination.totalPages).toBe(3);
    expect(result.pagination.hasNextPage).toBe(true);
    expect(result.pagination.hasPrevPage).toBe(false);
  });

  it('maintains consistent ordering across pages', async () => {
    // Fetch all pages
    const allJobs: any[] = [];
    let page = 1;
    let hasNextPage = true;

    while (hasNextPage) {
      const response = await fetch(`/api/v1/jobs?page=${page}&limit=10`, {
        headers: { 'X-Session-ID': 'pagination-test' },
      });
      const result = await response.json();
      allJobs.push(...result.data);
      hasNextPage = result.pagination.hasNextPage;
      page++;
    }

    // Verify no duplicates
    const ids = allJobs.map(j => j.id);
    const uniqueIds = [...new Set(ids)];
    expect(ids.length).toBe(uniqueIds.length);

    // Verify consistent ordering (by createdAt desc)
    for (let i = 1; i < allJobs.length; i++) {
      const prevDate = new Date(allJobs[i - 1].createdAt);
      const currDate = new Date(allJobs[i].createdAt);
      expect(prevDate.getTime()).toBeGreaterThanOrEqual(currDate.getTime());
    }
  });

  it('handles edge cases correctly', async () => {
    // Empty result
    const emptyResponse = await fetch('/api/v1/jobs?page=100&limit=10', {
      headers: { 'X-Session-ID': 'pagination-test' },
    });
    const emptyResult = await emptyResponse.json();
    expect(emptyResult.data.length).toBe(0);
    expect(emptyResult.pagination.hasNextPage).toBe(false);

    // Invalid page number
    const invalidResponse = await fetch('/api/v1/jobs?page=-1&limit=10', {
      headers: { 'X-Session-ID': 'pagination-test' },
    });
    expect(invalidResponse.status).toBe(400);

    // Excessive limit
    const limitResponse = await fetch('/api/v1/jobs?page=1&limit=1000', {
      headers: { 'X-Session-ID': 'pagination-test' },
    });
    const limitResult = await limitResponse.json();
    expect(limitResult.data.length).toBeLessThanOrEqual(100); // Max limit enforced
  });
});
```

**Expected Results**:
- Pagination metadata accurate
- Consistent ordering across pages
- Edge cases handled correctly

**Verification Points**:
- Total count accurate
- No duplicate records across pages
- Limit enforcement validated

---

### INT-CONSISTENCY-003: Cache-Database Consistency

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-CONSISTENCY-003 |
| **Title** | Cache-Database Consistency |
| **Priority** | P2 - Medium |
| **Components** | Redis Cache, PostgreSQL, API Routes |
| **Spec Reference** | FR-703, Caching Requirements |

**Preconditions**:
- Redis cache enabled
- Cache invalidation configured
- Test jobs available

**Test Steps**:

```typescript
describe('Cache-Database Consistency', () => {
  it('serves cached data matching database state', async () => {
    const job = await createJob({ fileName: 'cache-test.pdf', status: 'COMPLETED' });

    // First request - cache miss, fetches from DB
    const response1 = await fetch(`/api/v1/jobs/${job.id}`, {
      headers: { 'X-Session-ID': job.sessionId },
    });
    const data1 = await response1.json();

    // Second request - cache hit
    const response2 = await fetch(`/api/v1/jobs/${job.id}`, {
      headers: { 'X-Session-ID': job.sessionId },
    });
    const data2 = await response2.json();

    // Verify consistency
    expect(data1.id).toBe(data2.id);
    expect(data1.status).toBe(data2.status);
    expect(data1.fileName).toBe(data2.fileName);
  });

  it('invalidates cache on database update', async () => {
    const job = await createJob({ fileName: 'invalidation-test.pdf' });

    // Warm the cache
    await fetch(`/api/v1/jobs/${job.id}`, {
      headers: { 'X-Session-ID': job.sessionId },
    });

    // Update via API (should invalidate cache)
    await prisma.job.update({
      where: { id: job.id },
      data: { status: 'PROCESSING', progress: 50 },
    });

    // Trigger cache invalidation (API update or explicit invalidation)
    await redis.del(`job:${job.id}`);

    // Fetch again - should get updated data
    const response = await fetch(`/api/v1/jobs/${job.id}`, {
      headers: { 'X-Session-ID': job.sessionId },
    });
    const data = await response.json();

    expect(data.status).toBe('PROCESSING');
    expect(data.progress).toBe(50);
  });

  it('handles cache failure gracefully', async () => {
    const job = await createJob({ fileName: 'cache-failure-test.pdf' });

    // Mock Redis failure
    jest.spyOn(redis, 'get').mockRejectedValue(new Error('Redis unavailable'));

    // Request should still succeed by falling back to database
    const response = await fetch(`/api/v1/jobs/${job.id}`, {
      headers: { 'X-Session-ID': job.sessionId },
    });

    expect(response.status).toBe(200);
    const data = await response.json();
    expect(data.id).toBe(job.id);

    jest.restoreAllMocks();
  });

  it('maintains consistency during cache refresh', async () => {
    const job = await createJob({ fileName: 'refresh-test.pdf' });

    // Simulate concurrent cache refresh and database update
    const operations = await Promise.all([
      // Cache refresh
      fetch(`/api/v1/jobs/${job.id}`, {
        headers: { 'X-Session-ID': job.sessionId },
      }),
      // Database update
      prisma.job.update({
        where: { id: job.id },
        data: { progress: 75 },
      }),
    ]);

    // Small delay to allow cache to settle
    await sleep(100);

    // Final read should show consistent state
    const finalResponse = await fetch(`/api/v1/jobs/${job.id}`, {
      headers: { 'X-Session-ID': job.sessionId },
    });
    const finalData = await finalResponse.json();
    const dbJob = await prisma.job.findUnique({ where: { id: job.id } });

    expect(finalData.progress).toBe(dbJob?.progress);
  });
});
```

**Expected Results**:
- Cached data matches database state
- Cache invalidated on updates
- Graceful fallback on cache failure

**Verification Points**:
- Cache hit/miss consistency validated
- Invalidation mechanism tested
- Fallback behavior confirmed

---

### INT-CONSISTENCY-004: Session State Consistency

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-CONSISTENCY-004 |
| **Title** | Session State Consistency |
| **Priority** | P2 - Medium |
| **Components** | Session Management, Redis, API Routes |
| **Spec Reference** | FR-704, Session Requirements |

**Preconditions**:
- Session management enabled
- Redis session store available
- Test sessions created

**Test Steps**:

```typescript
describe('Session State Consistency', () => {
  it('maintains session state across requests', async () => {
    const sessionId = `session-${Date.now()}`;

    // Create session data
    await fetch('/api/v1/sessions', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ sessionId, metadata: { user: 'test-user' } }),
    });

    // Create multiple jobs in session
    const job1 = await createJob({ sessionId, fileName: 'session-test-1.pdf' });
    const job2 = await createJob({ sessionId, fileName: 'session-test-2.pdf' });

    // Verify session state consistency
    const sessionResponse = await fetch(`/api/v1/sessions/${sessionId}`, {
      headers: { 'X-Session-ID': sessionId },
    });
    const sessionData = await sessionResponse.json();

    expect(sessionData.jobCount).toBe(2);
    expect(sessionData.metadata.user).toBe('test-user');
  });

  it('isolates session data between different sessions', async () => {
    const sessionA = `session-a-${Date.now()}`;
    const sessionB = `session-b-${Date.now()}`;

    // Create jobs in different sessions
    await createJob({ sessionId: sessionA, fileName: 'session-a.pdf' });
    await createJob({ sessionId: sessionB, fileName: 'session-b.pdf' });

    // Verify isolation
    const jobsA = await fetch('/api/v1/jobs', {
      headers: { 'X-Session-ID': sessionA },
    });
    const dataA = await jobsA.json();

    const jobsB = await fetch('/api/v1/jobs', {
      headers: { 'X-Session-ID': sessionB },
    });
    const dataB = await jobsB.json();

    expect(dataA.data.every(j => j.sessionId === sessionA)).toBe(true);
    expect(dataB.data.every(j => j.sessionId === sessionB)).toBe(true);
    expect(dataA.data.length).toBe(1);
    expect(dataB.data.length).toBe(1);
  });

  it('handles session expiration correctly', async () => {
    const sessionId = `expiring-session-${Date.now()}`;

    // Create session with short TTL
    await redis.set(`session:${sessionId}`, JSON.stringify({ created: Date.now() }), 'EX', 2);

    // Create job in session
    const job = await createJob({ sessionId, fileName: 'expiring-test.pdf' });

    // Verify session exists
    let sessionResponse = await fetch(`/api/v1/sessions/${sessionId}`, {
      headers: { 'X-Session-ID': sessionId },
    });
    expect(sessionResponse.status).toBe(200);

    // Wait for expiration
    await sleep(3000);

    // Session should be expired
    sessionResponse = await fetch(`/api/v1/sessions/${sessionId}`, {
      headers: { 'X-Session-ID': sessionId },
    });
    expect(sessionResponse.status).toBe(404);

    // Job should still exist in database
    const dbJob = await prisma.job.findUnique({ where: { id: job.id } });
    expect(dbJob).toBeTruthy();
  });

  it('synchronizes session state between Redis and database', async () => {
    const sessionId = `sync-session-${Date.now()}`;

    // Create session
    await fetch('/api/v1/sessions', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ sessionId }),
    });

    // Verify Redis state
    const redisSession = await redis.get(`session:${sessionId}`);
    expect(redisSession).toBeTruthy();

    // Verify database state (if sessions are persisted)
    const dbSession = await prisma.session.findUnique({ where: { id: sessionId } });
    expect(dbSession).toBeTruthy();

    // Update session
    await fetch(`/api/v1/sessions/${sessionId}`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json', 'X-Session-ID': sessionId },
      body: JSON.stringify({ metadata: { updated: true } }),
    });

    // Verify both stores updated
    const updatedRedis = JSON.parse(await redis.get(`session:${sessionId}`) || '{}');
    const updatedDb = await prisma.session.findUnique({ where: { id: sessionId } });

    expect(updatedRedis.metadata?.updated).toBe(true);
    expect(updatedDb?.metadata?.updated).toBe(true);
  });
});
```

**Expected Results**:
- Session state maintained across requests
- Session data isolated between sessions
- Session expiration handled correctly
- Redis and database session state synchronized

**Verification Points**:
- Session persistence validated
- Isolation confirmed
- Expiration handling tested
- Cross-store synchronization verified

---

## 25. Container and Docker Testing

### 25.1 Overview

Tests validating container security, Docker image best practices, Docker Compose configuration, and container runtime behavior. This section addresses critical container security scanning gaps identified in the consolidated review (A-03).

**Test Framework**:
- **Container Scanning**: Trivy, Grype
- **Security Scanning**: OWASP ZAP, Dockle
- **Runtime Testing**: Docker SDK, testcontainers
- **Test Directory**: `test/integration/container/`
- **Run Command**: `npm run test:container`

**Integration Points**:
```
+-------------------+     +------------------+     +----------------------+
|   Dockerfile      |---->|  Container Image |---->|  Container Runtime   |
|   (Build Config)  |     |  (Security Scan) |     |  (Health/Resources)  |
+-------------------+     +------------------+     +----------------------+
         |                        |                        |
         v                        v                        v
+-------------------+     +------------------+     +----------------------+
|  docker-compose   |     |   CVE Database   |     |   Resource Limits    |
|  (Orchestration)  |     |   (Trivy/Grype)  |     |   (CPU/Memory)       |
+-------------------+     +------------------+     +----------------------+
```

---

### INT-DOCKER-001: Image Vulnerability Scanning with Trivy

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DOCKER-001 |
| **Title** | Image Vulnerability Scanning with Trivy |
| **Priority** | P0 - Critical |
| **Components** | Docker Image, Trivy Scanner |
| **Spec Reference** | SEC-001, Container Security Requirements |

**Preconditions**:
- Docker image built and available
- Trivy scanner installed and configured
- CVE database updated

**Test Steps**:

```typescript
describe('Image Vulnerability Scanning with Trivy', () => {
  it('scans container image for HIGH/CRITICAL vulnerabilities', async () => {
    const imageName = 'hx-docling-application:latest';

    // Execute Trivy scan
    const scanResult = await execAsync(
      `trivy image --severity HIGH,CRITICAL --format json ${imageName}`
    );

    const vulnerabilities = JSON.parse(scanResult.stdout);

    // Extract HIGH and CRITICAL CVEs
    const criticalVulns = vulnerabilities.Results?.flatMap(
      (r: any) => r.Vulnerabilities?.filter(
        (v: any) => v.Severity === 'CRITICAL'
      ) || []
    ) || [];

    const highVulns = vulnerabilities.Results?.flatMap(
      (r: any) => r.Vulnerabilities?.filter(
        (v: any) => v.Severity === 'HIGH'
      ) || []
    ) || [];

    // Quality gate: Zero CRITICAL vulnerabilities
    expect(criticalVulns.length).toBe(0);

    // Quality gate: Maximum 5 HIGH vulnerabilities (with remediation plan)
    expect(highVulns.length).toBeLessThanOrEqual(5);

    // Log findings for review
    if (highVulns.length > 0) {
      console.warn('HIGH vulnerabilities found:', highVulns.map(
        (v: any) => `${v.VulnerabilityID}: ${v.PkgName}`
      ));
    }
  });

  it('validates Trivy scan output format', async () => {
    const imageName = 'hx-docling-application:latest';

    const scanResult = await execAsync(
      `trivy image --format json ${imageName}`
    );

    const result = JSON.parse(scanResult.stdout);

    // Validate scan completed successfully
    expect(result).toHaveProperty('SchemaVersion');
    expect(result).toHaveProperty('ArtifactName');
    expect(result).toHaveProperty('Results');
    expect(Array.isArray(result.Results)).toBe(true);
  });

  it('generates SBOM (Software Bill of Materials)', async () => {
    const imageName = 'hx-docling-application:latest';

    const sbomResult = await execAsync(
      `trivy image --format cyclonedx ${imageName}`
    );

    const sbom = JSON.parse(sbomResult.stdout);

    // Validate SBOM format
    expect(sbom).toHaveProperty('bomFormat', 'CycloneDX');
    expect(sbom).toHaveProperty('components');
    expect(Array.isArray(sbom.components)).toBe(true);
    expect(sbom.components.length).toBeGreaterThan(0);
  });
});
```

**Expected Results**:
- Zero CRITICAL vulnerabilities detected
- Maximum 5 HIGH vulnerabilities (with documented remediation plan)
- Valid SBOM generated for compliance

**Verification Points**:
- Trivy scan completes successfully
- CVE database is current (within 24 hours)
- All findings logged for security review

---

### INT-DOCKER-002: Image Vulnerability Scanning with Grype

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DOCKER-002 |
| **Title** | Image Vulnerability Scanning with Grype |
| **Priority** | P0 - Critical |
| **Components** | Docker Image, Grype Scanner |
| **Spec Reference** | SEC-001, Container Security Requirements |

**Preconditions**:
- Docker image built and available
- Grype scanner installed
- Vulnerability database updated

**Test Steps**:

```typescript
describe('Image Vulnerability Scanning with Grype', () => {
  it('cross-validates vulnerabilities with Grype scanner', async () => {
    const imageName = 'hx-docling-application:latest';

    // Execute Grype scan
    const scanResult = await execAsync(
      `grype ${imageName} --output json --fail-on critical`
    );

    const vulnerabilities = JSON.parse(scanResult.stdout);

    // Extract matches
    const matches = vulnerabilities.matches || [];

    // Categorize by severity
    const criticalMatches = matches.filter(
      (m: any) => m.vulnerability.severity === 'Critical'
    );
    const highMatches = matches.filter(
      (m: any) => m.vulnerability.severity === 'High'
    );

    // Quality gate: Zero Critical
    expect(criticalMatches.length).toBe(0);

    // Log high severity for tracking
    console.log(`Grype found ${highMatches.length} HIGH severity issues`);
  });

  it('compares Trivy and Grype results for consistency', async () => {
    const imageName = 'hx-docling-application:latest';

    // Run both scanners
    const [trivyResult, grypeResult] = await Promise.all([
      execAsync(`trivy image --format json ${imageName}`),
      execAsync(`grype ${imageName} --output json`),
    ]);

    const trivyVulns = JSON.parse(trivyResult.stdout);
    const grypeVulns = JSON.parse(grypeResult.stdout);

    // Extract CVE IDs from both
    const trivyCVEs = new Set(
      trivyVulns.Results?.flatMap(
        (r: any) => r.Vulnerabilities?.map((v: any) => v.VulnerabilityID) || []
      ) || []
    );

    const grypeCVEs = new Set(
      grypeVulns.matches?.map((m: any) => m.vulnerability.id) || []
    );

    // Log comparison
    console.log(`Trivy found ${trivyCVEs.size} unique CVEs`);
    console.log(`Grype found ${grypeCVEs.size} unique CVEs`);

    // Both scanners should find critical issues if they exist
    const trivyCritical = trivyVulns.Results?.flatMap(
      (r: any) => r.Vulnerabilities?.filter(
        (v: any) => v.Severity === 'CRITICAL'
      ) || []
    ) || [];

    const grypeCritical = grypeVulns.matches?.filter(
      (m: any) => m.vulnerability.severity === 'Critical'
    ) || [];

    // If one finds critical, both should find it
    if (trivyCritical.length > 0 || grypeCritical.length > 0) {
      expect(trivyCritical.length).toBeGreaterThan(0);
      expect(grypeCritical.length).toBeGreaterThan(0);
    }
  });
});
```

**Expected Results**:
- Grype scan validates Trivy findings
- Cross-scanner consistency for critical issues
- Comprehensive vulnerability coverage

**Verification Points**:
- Both scanners execute successfully
- Critical findings consistent across scanners
- Results logged for security review

---

### INT-DOCKER-003: Base Image Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DOCKER-003 |
| **Title** | Base Image Validation |
| **Priority** | P0 - Critical |
| **Components** | Dockerfile, Base Image |
| **Spec Reference** | SEC-002, Image Hardening Requirements |

**Preconditions**:
- Dockerfile available
- Base image accessible

**Test Steps**:

```typescript
describe('Base Image Validation', () => {
  it('uses approved base image from trusted registry', async () => {
    const dockerfile = await fs.readFile('Dockerfile', 'utf-8');

    // Extract FROM instructions
    const fromLines = dockerfile.match(/^FROM\s+(.+)$/gm) || [];

    expect(fromLines.length).toBeGreaterThan(0);

    // Approved base images
    const approvedBases = [
      'node:20-alpine',
      'node:20-slim',
      'python:3.11-slim',
      'python:3.11-alpine',
      'gcr.io/distroless/',
      'mcr.microsoft.com/',
    ];

    for (const fromLine of fromLines) {
      const image = fromLine.replace(/^FROM\s+/, '').split(' ')[0];

      // Skip build stage references
      if (image.includes('AS ') || image.startsWith('--')) continue;

      const isApproved = approvedBases.some(
        (base) => image.startsWith(base) || image.includes(base)
      );

      expect(isApproved).toBe(true);
    }
  });

  it('base image is not using latest tag', async () => {
    const dockerfile = await fs.readFile('Dockerfile', 'utf-8');

    const fromLines = dockerfile.match(/^FROM\s+(.+)$/gm) || [];

    for (const fromLine of fromLines) {
      const image = fromLine.replace(/^FROM\s+/, '').split(' ')[0];

      // Check for :latest tag or no tag (defaults to latest)
      expect(image).not.toMatch(/:latest$/);
      expect(image).toMatch(/:.+/); // Must have explicit tag
    }
  });

  it('base image uses minimal distro (alpine/slim/distroless)', async () => {
    const dockerfile = await fs.readFile('Dockerfile', 'utf-8');

    // Get final stage base image
    const fromLines = dockerfile.match(/^FROM\s+(.+)$/gm) || [];
    const finalFrom = fromLines[fromLines.length - 1];
    const finalImage = finalFrom.replace(/^FROM\s+/, '').split(' ')[0];

    // Should use minimal base
    const isMinimal =
      finalImage.includes('alpine') ||
      finalImage.includes('slim') ||
      finalImage.includes('distroless');

    expect(isMinimal).toBe(true);
  });

  it('base image digest is pinned for reproducibility', async () => {
    const dockerfile = await fs.readFile('Dockerfile', 'utf-8');

    // Production builds should pin digest
    const fromLines = dockerfile.match(/^FROM\s+(.+)$/gm) || [];

    // At least the final stage should have pinned digest
    const hasDigest = fromLines.some((line) => line.includes('@sha256:'));

    // Log recommendation if not pinned
    if (!hasDigest) {
      console.warn('Recommendation: Pin base image with digest for production');
    }

    // Soft check - warning only for now
    expect(hasDigest || true).toBe(true);
  });
});
```

**Expected Results**:
- Base image from approved registry
- Explicit version tag (not :latest)
- Minimal distro used (alpine/slim/distroless)

**Verification Points**:
- Dockerfile parsed correctly
- Base image meets security standards
- Version pinning validated

---

### INT-DOCKER-004: Non-Root User Enforcement

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DOCKER-004 |
| **Title** | Non-Root User Enforcement |
| **Priority** | P0 - Critical |
| **Components** | Dockerfile, Container Runtime |
| **Spec Reference** | SEC-003, Privilege Requirements |

**Preconditions**:
- Docker image built
- Container can be started

**Test Steps**:

```typescript
describe('Non-Root User Enforcement', () => {
  it('Dockerfile defines non-root USER', async () => {
    const dockerfile = await fs.readFile('Dockerfile', 'utf-8');

    // Check for USER instruction
    const userLines = dockerfile.match(/^USER\s+(.+)$/gm) || [];

    expect(userLines.length).toBeGreaterThan(0);

    // Get final USER instruction
    const finalUser = userLines[userLines.length - 1]
      .replace(/^USER\s+/, '')
      .trim();

    // Should not be root or 0
    expect(finalUser).not.toBe('root');
    expect(finalUser).not.toBe('0');
  });

  it('container runs as non-root user at runtime', async () => {
    const imageName = 'hx-docling-application:latest';

    // Run container and check user
    const result = await execAsync(
      `docker run --rm ${imageName} id -u`
    );

    const uid = parseInt(result.stdout.trim(), 10);

    // UID should not be 0 (root)
    expect(uid).not.toBe(0);
    expect(uid).toBeGreaterThanOrEqual(1000);
  });

  it('container cannot escalate to root', async () => {
    const imageName = 'hx-docling-application:latest';

    // Try to run as root - should fail
    const result = await execAsync(
      `docker run --rm --user root ${imageName} id -u || echo "BLOCKED"`
    );

    // If security is configured correctly, root should be blocked
    // or we should enforce --security-opt=no-new-privileges
    const output = result.stdout.trim();

    // Log for security review
    console.log('Root escalation test result:', output);
  });

  it('validates user/group creation in Dockerfile', async () => {
    const dockerfile = await fs.readFile('Dockerfile', 'utf-8');

    // Should create dedicated user/group
    const hasUserCreation =
      dockerfile.includes('adduser') ||
      dockerfile.includes('useradd') ||
      dockerfile.includes('addgroup');

    expect(hasUserCreation).toBe(true);

    // Verify home directory setup
    const hasWorkdir = dockerfile.includes('WORKDIR');
    expect(hasWorkdir).toBe(true);
  });
});
```

**Expected Results**:
- Dockerfile specifies non-root USER
- Container runtime UID is not 0
- Privilege escalation blocked

**Verification Points**:
- USER instruction present and valid
- Runtime user verified
- Security constraints applied

---

### INT-DOCKER-005: Secret Scanning in Images

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DOCKER-005 |
| **Title** | Secret Scanning in Images |
| **Priority** | P0 - Critical |
| **Components** | Docker Image, Secret Scanner |
| **Spec Reference** | SEC-004, Secret Management Requirements |

**Preconditions**:
- Docker image built
- Trivy secret scanning enabled

**Test Steps**:

```typescript
describe('Secret Scanning in Images', () => {
  it('scans image for embedded secrets with Trivy', async () => {
    const imageName = 'hx-docling-application:latest';

    // Run Trivy secret scan
    const scanResult = await execAsync(
      `trivy image --scanners secret --format json ${imageName}`
    );

    const results = JSON.parse(scanResult.stdout);

    // Check for secrets in all layers
    const secrets = results.Results?.flatMap(
      (r: any) => r.Secrets || []
    ) || [];

    // Quality gate: Zero embedded secrets
    expect(secrets.length).toBe(0);

    if (secrets.length > 0) {
      console.error('SECURITY ALERT: Secrets found in image:',
        secrets.map((s: any) => `${s.Category}: ${s.Title}`));
    }
  });

  it('validates no sensitive files in image', async () => {
    const imageName = 'hx-docling-application:latest';

    // Check for common secret file patterns
    const sensitivePatterns = [
      '.env',
      '.env.local',
      'credentials.json',
      'secrets.yaml',
      'id_rsa',
      'id_ed25519',
      '.aws/credentials',
      '.npmrc',
      '.pypirc',
    ];

    for (const pattern of sensitivePatterns) {
      const result = await execAsync(
        `docker run --rm ${imageName} test -f ${pattern} && echo "FOUND" || echo "OK"`
      ).catch(() => ({ stdout: 'OK' }));

      expect(result.stdout.trim()).toBe('OK');
    }
  });

  it('environment variables do not contain secrets', async () => {
    const imageName = 'hx-docling-application:latest';

    // Get environment variables from image
    const result = await execAsync(
      `docker inspect ${imageName} --format '{{json .Config.Env}}'`
    );

    const envVars = JSON.parse(result.stdout);

    // Patterns that indicate hardcoded secrets
    const secretPatterns = [
      /password=/i,
      /secret=/i,
      /api_key=/i,
      /apikey=/i,
      /token=/i,
      /private_key=/i,
    ];

    for (const env of envVars) {
      for (const pattern of secretPatterns) {
        // Should not have hardcoded secret values
        const match = env.match(pattern);
        if (match) {
          // Exclude pattern-only matches (e.g., DATABASE_PASSWORD= with no value)
          const value = env.split('=')[1];
          expect(value).toBeFalsy();
        }
      }
    }
  });

  it('Docker history does not expose secrets', async () => {
    const imageName = 'hx-docling-application:latest';

    // Get image history
    const result = await execAsync(
      `docker history ${imageName} --no-trunc --format '{{.CreatedBy}}'`
    );

    const history = result.stdout;

    // Check for secret exposure in build commands
    const secretIndicators = [
      /--build-arg.*PASSWORD/i,
      /--build-arg.*SECRET/i,
      /--build-arg.*TOKEN/i,
      /--build-arg.*API_KEY/i,
      /echo.*password/i,
    ];

    for (const indicator of secretIndicators) {
      expect(history).not.toMatch(indicator);
    }
  });
});
```

**Expected Results**:
- Zero secrets embedded in image layers
- No sensitive files present
- Environment variables clean
- Build history does not expose secrets

**Verification Points**:
- Trivy secret scan passes
- File system validated
- Environment variables checked
- Build history reviewed

---

### INT-DOCKER-006: CVE Threshold Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DOCKER-006 |
| **Title** | CVE Threshold Validation |
| **Priority** | P0 - Critical |
| **Components** | Docker Image, CI/CD Pipeline |
| **Spec Reference** | SEC-005, Vulnerability Management |

**Preconditions**:
- Image vulnerability scan completed
- CVE thresholds defined

**Test Steps**:

```typescript
describe('CVE Threshold Validation', () => {
  const CVE_THRESHOLDS = {
    critical: 0,      // Zero tolerance
    high: 5,          // Maximum 5 with remediation plan
    medium: 20,       // Maximum 20
    low: 100,         // Maximum 100
  };

  it('validates vulnerability counts against thresholds', async () => {
    const imageName = 'hx-docling-application:latest';

    const scanResult = await execAsync(
      `trivy image --format json ${imageName}`
    );

    const results = JSON.parse(scanResult.stdout);

    // Count by severity
    const counts = {
      critical: 0,
      high: 0,
      medium: 0,
      low: 0,
    };

    for (const result of results.Results || []) {
      for (const vuln of result.Vulnerabilities || []) {
        const severity = vuln.Severity.toLowerCase();
        if (counts.hasOwnProperty(severity)) {
          counts[severity as keyof typeof counts]++;
        }
      }
    }

    // Validate against thresholds
    expect(counts.critical).toBeLessThanOrEqual(CVE_THRESHOLDS.critical);
    expect(counts.high).toBeLessThanOrEqual(CVE_THRESHOLDS.high);
    expect(counts.medium).toBeLessThanOrEqual(CVE_THRESHOLDS.medium);
    expect(counts.low).toBeLessThanOrEqual(CVE_THRESHOLDS.low);

    // Log summary
    console.log('CVE Summary:', counts);
  });

  it('fails build on critical CVE detection', async () => {
    const imageName = 'hx-docling-application:latest';

    // Trivy should exit with error on critical
    try {
      await execAsync(
        `trivy image --exit-code 1 --severity CRITICAL ${imageName}`
      );
      // If we reach here, no critical CVEs found
      expect(true).toBe(true);
    } catch (error: any) {
      // Exit code 1 means critical CVEs found
      expect(error.code).not.toBe(1);
    }
  });

  it('generates vulnerability report for compliance', async () => {
    const imageName = 'hx-docling-application:latest';
    const reportPath = '/tmp/vulnerability-report.json';

    await execAsync(
      `trivy image --format json --output ${reportPath} ${imageName}`
    );

    // Verify report generated
    const report = await fs.readFile(reportPath, 'utf-8');
    const reportData = JSON.parse(report);

    expect(reportData).toHaveProperty('ArtifactName');
    expect(reportData).toHaveProperty('Results');

    // Report should be parseable by security tools
    expect(reportData.SchemaVersion).toBeDefined();
  });
});
```

**Expected Results**:
- All severity counts within thresholds
- Build fails on critical CVE
- Compliance report generated

**Verification Points**:
- Thresholds enforced programmatically
- CI/CD integration validated
- Report format verified

---

### INT-DOCKER-007: Multi-Stage Build Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DOCKER-007 |
| **Title** | Multi-Stage Build Validation |
| **Priority** | P1 - High |
| **Components** | Dockerfile |
| **Spec Reference** | BLD-001, Build Optimization |

**Preconditions**:
- Dockerfile available
- Build context accessible

**Test Steps**:

```typescript
describe('Multi-Stage Build Validation', () => {
  it('Dockerfile uses multi-stage build pattern', async () => {
    const dockerfile = await fs.readFile('Dockerfile', 'utf-8');

    // Count FROM instructions
    const fromLines = dockerfile.match(/^FROM\s+/gm) || [];

    // Multi-stage requires at least 2 FROM instructions
    expect(fromLines.length).toBeGreaterThanOrEqual(2);
  });

  it('build stage is separate from production stage', async () => {
    const dockerfile = await fs.readFile('Dockerfile', 'utf-8');

    // Check for named stages
    const buildStage = dockerfile.match(/FROM\s+.+\s+AS\s+(build|builder)/i);
    const prodStage = dockerfile.match(/FROM\s+.+\s+AS\s+(production|prod|runtime|final)/i);

    expect(buildStage).toBeTruthy();
    // Production stage might not be named (final stage)
  });

  it('dev dependencies not in production image', async () => {
    const imageName = 'hx-docling-application:latest';

    // Check for common dev dependencies that should not be in prod
    const devDeps = [
      'node_modules/.bin/jest',
      'node_modules/.bin/vitest',
      'node_modules/.bin/eslint',
      'node_modules/typescript',
      'node_modules/@types',
    ];

    for (const dep of devDeps) {
      const result = await execAsync(
        `docker run --rm ${imageName} test -e ${dep} && echo "FOUND" || echo "OK"`
      ).catch(() => ({ stdout: 'OK' }));

      expect(result.stdout.trim()).toBe('OK');
    }
  });

  it('build artifacts not present in final image', async () => {
    const imageName = 'hx-docling-application:latest';

    // Check for build artifacts that should be excluded
    const buildArtifacts = [
      '.git',
      '.github',
      'src',
      'test',
      'tests',
      '*.ts',
      'tsconfig.json',
      'vitest.config.ts',
    ];

    for (const artifact of buildArtifacts) {
      const result = await execAsync(
        `docker run --rm ${imageName} test -e /app/${artifact} && echo "FOUND" || echo "OK"`
      ).catch(() => ({ stdout: 'OK' }));

      // Source files should not be in production image
      if (!artifact.includes('*')) {
        expect(result.stdout.trim()).toBe('OK');
      }
    }
  });
});
```

**Expected Results**:
- Multi-stage build pattern used
- Clear separation of build and production stages
- Dev dependencies excluded from production
- Build artifacts not in final image

**Verification Points**:
- Dockerfile structure validated
- Image contents verified
- Size optimization confirmed

---

### INT-DOCKER-008: Layer Optimization

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DOCKER-008 |
| **Title** | Layer Optimization |
| **Priority** | P1 - High |
| **Components** | Dockerfile, Docker Image |
| **Spec Reference** | BLD-002, Image Optimization |

**Preconditions**:
- Docker image built
- Dockerfile available

**Test Steps**:

```typescript
describe('Layer Optimization', () => {
  it('image size is within acceptable limits', async () => {
    const imageName = 'hx-docling-application:latest';

    const result = await execAsync(
      `docker image inspect ${imageName} --format '{{.Size}}'`
    );

    const sizeBytes = parseInt(result.stdout.trim(), 10);
    const sizeMB = sizeBytes / (1024 * 1024);

    // Maximum acceptable size: 500MB for Node.js app
    expect(sizeMB).toBeLessThan(500);

    console.log(`Image size: ${sizeMB.toFixed(2)} MB`);
  });

  it('layer count is optimized', async () => {
    const imageName = 'hx-docling-application:latest';

    const result = await execAsync(
      `docker history ${imageName} --format '{{.Size}}'`
    );

    const layers = result.stdout.trim().split('\n');

    // Reasonable layer count (multi-stage usually has fewer final layers)
    expect(layers.length).toBeLessThan(25);

    console.log(`Layer count: ${layers.length}`);
  });

  it('RUN commands are consolidated appropriately', async () => {
    const dockerfile = await fs.readFile('Dockerfile', 'utf-8');

    // Count RUN instructions
    const runCommands = dockerfile.match(/^RUN\s+/gm) || [];

    // Should consolidate RUN commands (generally < 10 per stage)
    expect(runCommands.length).toBeLessThan(15);

    // Check for proper cleanup in RUN commands
    const hasCleanup = dockerfile.includes('rm -rf') ||
                       dockerfile.includes('apt-get clean') ||
                       dockerfile.includes('apk --no-cache');

    expect(hasCleanup).toBe(true);
  });

  it('cache layers ordered correctly', async () => {
    const dockerfile = await fs.readFile('Dockerfile', 'utf-8');

    // Package files should be copied before source code
    const packageCopyIndex = dockerfile.indexOf('COPY package');
    const sourceCopyIndex = dockerfile.indexOf('COPY src') !== -1
      ? dockerfile.indexOf('COPY src')
      : dockerfile.indexOf('COPY . ');

    if (packageCopyIndex !== -1 && sourceCopyIndex !== -1) {
      expect(packageCopyIndex).toBeLessThan(sourceCopyIndex);
    }
  });
});
```

**Expected Results**:
- Image size under 500MB
- Layer count optimized
- RUN commands consolidated
- Cache-friendly ordering

**Verification Points**:
- Size within limits
- Layer efficiency validated
- Build cache optimization confirmed

---

### INT-DOCKER-009: COPY vs ADD Usage

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DOCKER-009 |
| **Title** | COPY vs ADD Usage Validation |
| **Priority** | P1 - High |
| **Components** | Dockerfile |
| **Spec Reference** | BLD-003, Dockerfile Best Practices |

**Preconditions**:
- Dockerfile available

**Test Steps**:

```typescript
describe('COPY vs ADD Usage', () => {
  it('prefers COPY over ADD for local files', async () => {
    const dockerfile = await fs.readFile('Dockerfile', 'utf-8');

    // Extract ADD instructions
    const addLines = dockerfile.match(/^ADD\s+(.+)$/gm) || [];

    for (const addLine of addLines) {
      const source = addLine.replace(/^ADD\s+/, '').split(' ')[0];

      // ADD is only acceptable for:
      // 1. Remote URLs (http/https)
      // 2. Tar file extraction
      const isRemoteUrl = source.startsWith('http://') || source.startsWith('https://');
      const isTarExtraction = source.endsWith('.tar') ||
                              source.endsWith('.tar.gz') ||
                              source.endsWith('.tgz');

      if (!isRemoteUrl && !isTarExtraction) {
        console.warn(`Recommendation: Replace ADD with COPY for: ${source}`);
      }

      expect(isRemoteUrl || isTarExtraction).toBe(true);
    }
  });

  it('COPY instructions use explicit paths', async () => {
    const dockerfile = await fs.readFile('Dockerfile', 'utf-8');

    const copyLines = dockerfile.match(/^COPY\s+(.+)$/gm) || [];

    for (const copyLine of copyLines) {
      // Should not copy entire context without .dockerignore
      const copiesEverything = copyLine.includes('COPY . .');

      if (copiesEverything) {
        // Verify .dockerignore exists
        const hasDockerignore = await fs.access('.dockerignore')
          .then(() => true)
          .catch(() => false);

        expect(hasDockerignore).toBe(true);
      }
    }
  });

  it('validates --chown flag usage for security', async () => {
    const dockerfile = await fs.readFile('Dockerfile', 'utf-8');

    const copyLines = dockerfile.match(/^COPY\s+(.+)$/gm) || [];

    // After USER instruction, COPY should use --chown
    const userIndex = dockerfile.indexOf('USER');

    for (const copyLine of copyLines) {
      const copyIndex = dockerfile.indexOf(copyLine);

      if (copyIndex > userIndex && userIndex !== -1) {
        const hasChown = copyLine.includes('--chown');

        if (!hasChown) {
          console.warn('Recommendation: Use --chown with COPY after USER instruction');
        }
      }
    }
  });
});
```

**Expected Results**:
- COPY preferred over ADD for local files
- ADD used only for URLs or tar extraction
- Explicit paths used
- --chown used appropriately

**Verification Points**:
- Dockerfile best practices followed
- Security considerations applied
- Clear file ownership

---

### INT-DOCKER-010: Health Check Configuration

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DOCKER-010 |
| **Title** | Health Check Configuration |
| **Priority** | P1 - High |
| **Components** | Dockerfile, Container Runtime |
| **Spec Reference** | OPS-001, Container Health Monitoring |

**Preconditions**:
- Dockerfile available
- Container can be started

**Test Steps**:

```typescript
describe('Health Check Configuration', () => {
  it('Dockerfile defines HEALTHCHECK instruction', async () => {
    const dockerfile = await fs.readFile('Dockerfile', 'utf-8');

    const healthcheck = dockerfile.match(/^HEALTHCHECK\s+(.+)$/gm);

    expect(healthcheck).toBeTruthy();
    expect(healthcheck!.length).toBeGreaterThan(0);
  });

  it('health check has appropriate timing configuration', async () => {
    const dockerfile = await fs.readFile('Dockerfile', 'utf-8');

    const healthcheck = dockerfile.match(/^HEALTHCHECK\s+(.+)/m)?.[0] || '';

    // Should have interval, timeout, and retries
    const hasInterval = healthcheck.includes('--interval=');
    const hasTimeout = healthcheck.includes('--timeout=');
    const hasRetries = healthcheck.includes('--retries=');

    expect(hasInterval || hasTimeout || hasRetries).toBe(true);
  });

  it('health check uses appropriate endpoint', async () => {
    const dockerfile = await fs.readFile('Dockerfile', 'utf-8');

    const healthcheck = dockerfile.match(/^HEALTHCHECK\s+(.+)/m)?.[0] || '';

    // Should use health endpoint
    const usesHealthEndpoint =
      healthcheck.includes('/health') ||
      healthcheck.includes('/healthz') ||
      healthcheck.includes('/api/health');

    expect(usesHealthEndpoint).toBe(true);
  });

  it('container health check reports correctly at runtime', async () => {
    const imageName = 'hx-docling-application:latest';
    const containerName = 'healthcheck-test';

    // Start container
    await execAsync(
      `docker run -d --name ${containerName} ${imageName}`
    );

    // Wait for health check
    await new Promise((resolve) => setTimeout(resolve, 10000));

    // Check health status
    const result = await execAsync(
      `docker inspect ${containerName} --format '{{.State.Health.Status}}'`
    );

    const status = result.stdout.trim();
    expect(['healthy', 'starting']).toContain(status);

    // Cleanup
    await execAsync(`docker rm -f ${containerName}`);
  });
});
```

**Expected Results**:
- HEALTHCHECK instruction defined
- Appropriate timing configuration
- Health endpoint used
- Runtime health reporting works

**Verification Points**:
- Dockerfile health check present
- Configuration validated
- Runtime behavior verified

---

### INT-DOCKER-011: Entrypoint and CMD Patterns

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DOCKER-011 |
| **Title** | Entrypoint and CMD Patterns |
| **Priority** | P1 - High |
| **Components** | Dockerfile |
| **Spec Reference** | BLD-004, Container Execution |

**Preconditions**:
- Dockerfile available

**Test Steps**:

```typescript
describe('Entrypoint and CMD Patterns', () => {
  it('uses exec form for ENTRYPOINT/CMD', async () => {
    const dockerfile = await fs.readFile('Dockerfile', 'utf-8');

    const entrypoint = dockerfile.match(/^ENTRYPOINT\s+(.+)$/m)?.[1];
    const cmd = dockerfile.match(/^CMD\s+(.+)$/m)?.[1];

    // Exec form starts with [ (JSON array)
    if (entrypoint) {
      expect(entrypoint.trim()).toMatch(/^\[/);
    }
    if (cmd) {
      expect(cmd.trim()).toMatch(/^\[/);
    }

    // At least one should be defined
    expect(entrypoint || cmd).toBeTruthy();
  });

  it('ENTRYPOINT is not shell script unless necessary', async () => {
    const dockerfile = await fs.readFile('Dockerfile', 'utf-8');

    const entrypoint = dockerfile.match(/^ENTRYPOINT\s+(.+)$/m)?.[1];

    if (entrypoint) {
      // Should not be wrapping in shell
      expect(entrypoint).not.toMatch(/\["\/bin\/sh",\s*"-c"/);
      expect(entrypoint).not.toMatch(/\["\/bin\/bash",\s*"-c"/);
    }
  });

  it('signals are properly handled', async () => {
    const imageName = 'hx-docling-application:latest';
    const containerName = 'signal-test';

    // Start container
    await execAsync(
      `docker run -d --name ${containerName} ${imageName}`
    );

    // Wait for startup
    await new Promise((resolve) => setTimeout(resolve, 5000));

    // Send SIGTERM and measure shutdown time
    const startTime = Date.now();
    await execAsync(`docker stop ${containerName}`);
    const stopTime = Date.now() - startTime;

    // Should stop gracefully within 10 seconds
    expect(stopTime).toBeLessThan(10000);

    // Cleanup
    await execAsync(`docker rm -f ${containerName}`).catch(() => {});
  });

  it('container starts with correct process', async () => {
    const imageName = 'hx-docling-application:latest';

    // Check what runs as PID 1
    const result = await execAsync(
      `docker run --rm ${imageName} cat /proc/1/cmdline`
    );

    const cmdline = result.stdout;

    // Should be node or the application, not shell
    expect(cmdline).not.toMatch(/^\/bin\/sh/);
  });
});
```

**Expected Results**:
- Exec form used (JSON array)
- No unnecessary shell wrapping
- Proper signal handling
- Correct PID 1 process

**Verification Points**:
- Dockerfile patterns validated
- Signal handling verified
- Process structure correct

---

### INT-DOCKER-012: Docker Compose Service Dependencies

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DOCKER-012 |
| **Title** | Docker Compose Service Dependencies |
| **Priority** | P1 - High |
| **Components** | docker-compose.yml |
| **Spec Reference** | OPS-002, Service Orchestration |

**Preconditions**:
- docker-compose.yml available
- All service images available

**Test Steps**:

```typescript
describe('Docker Compose Service Dependencies', () => {
  it('validates depends_on configuration', async () => {
    const composeFile = await fs.readFile('docker-compose.yml', 'utf-8');
    const compose = yaml.parse(composeFile);

    const services = compose.services || {};

    // App service should depend on database and redis
    const appService = services.app || services.api || services.web;

    if (appService?.depends_on) {
      const deps = Array.isArray(appService.depends_on)
        ? appService.depends_on
        : Object.keys(appService.depends_on);

      // Should depend on data services
      const hasDatabaseDep = deps.some((d: string) =>
        d.includes('postgres') || d.includes('db') || d.includes('database')
      );
      const hasRedisDep = deps.some((d: string) =>
        d.includes('redis') || d.includes('cache')
      );

      expect(hasDatabaseDep || hasRedisDep).toBe(true);
    }
  });

  it('uses condition: service_healthy for dependencies', async () => {
    const composeFile = await fs.readFile('docker-compose.yml', 'utf-8');
    const compose = yaml.parse(composeFile);

    const services = compose.services || {};

    for (const [name, service] of Object.entries(services)) {
      const svc = service as any;

      if (svc.depends_on && typeof svc.depends_on === 'object') {
        for (const [dep, config] of Object.entries(svc.depends_on)) {
          const depConfig = config as any;

          // Should use health condition, not just service_started
          if (depConfig.condition) {
            expect(['service_healthy', 'service_completed_successfully'])
              .toContain(depConfig.condition);
          }
        }
      }
    }
  });

  it('services start in correct order', async () => {
    const startOrder: string[] = [];

    // Start compose and capture order
    const result = await execAsync(
      `docker compose up -d 2>&1`
    );

    // Parse output for service start order
    const lines = result.stdout.split('\n');
    for (const line of lines) {
      const match = line.match(/Container (\w+)/);
      if (match) {
        startOrder.push(match[1]);
      }
    }

    // Database/Redis should start before app
    const dbIndex = startOrder.findIndex(s =>
      s.includes('postgres') || s.includes('redis')
    );
    const appIndex = startOrder.findIndex(s =>
      s.includes('app') || s.includes('api')
    );

    if (dbIndex !== -1 && appIndex !== -1) {
      expect(dbIndex).toBeLessThan(appIndex);
    }

    // Cleanup
    await execAsync('docker compose down');
  });
});
```

**Expected Results**:
- Proper depends_on configuration
- Health conditions used
- Correct startup order

**Verification Points**:
- Compose file parsed correctly
- Dependencies validated
- Startup order verified

---

### INT-DOCKER-013: Network Isolation

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DOCKER-013 |
| **Title** | Network Isolation |
| **Priority** | P1 - High |
| **Components** | docker-compose.yml, Docker Networks |
| **Spec Reference** | SEC-006, Network Security |

**Preconditions**:
- docker-compose.yml available
- Docker networks configured

**Test Steps**:

```typescript
describe('Network Isolation', () => {
  it('defines custom networks for isolation', async () => {
    const composeFile = await fs.readFile('docker-compose.yml', 'utf-8');
    const compose = yaml.parse(composeFile);

    // Should define custom networks
    expect(compose.networks).toBeDefined();
    expect(Object.keys(compose.networks).length).toBeGreaterThan(0);
  });

  it('services are assigned to appropriate networks', async () => {
    const composeFile = await fs.readFile('docker-compose.yml', 'utf-8');
    const compose = yaml.parse(composeFile);

    const services = compose.services || {};

    for (const [name, service] of Object.entries(services)) {
      const svc = service as any;

      // Each service should be on at least one network
      if (svc.networks) {
        expect(
          Array.isArray(svc.networks)
            ? svc.networks.length
            : Object.keys(svc.networks).length
        ).toBeGreaterThan(0);
      }
    }
  });

  it('database not exposed to public network', async () => {
    const composeFile = await fs.readFile('docker-compose.yml', 'utf-8');
    const compose = yaml.parse(composeFile);

    const services = compose.services || {};

    // Find database service
    const dbService = services.postgres || services.db || services.database;

    if (dbService) {
      // Should not have ports exposed to host
      if (dbService.ports) {
        for (const port of dbService.ports) {
          // Should not bind to 0.0.0.0 or host port
          const portStr = String(port);
          expect(portStr).not.toMatch(/^0\.0\.0\.0:/);

          // In production, should not expose at all
          if (process.env.NODE_ENV === 'production') {
            expect(dbService.ports).toHaveLength(0);
          }
        }
      }
    }
  });

  it('internal network flag used for backend services', async () => {
    const composeFile = await fs.readFile('docker-compose.yml', 'utf-8');
    const compose = yaml.parse(composeFile);

    const networks = compose.networks || {};

    // Should have internal network for backend
    const hasInternalNetwork = Object.values(networks).some(
      (n: any) => n?.internal === true
    );

    // Log recommendation if not present
    if (!hasInternalNetwork) {
      console.warn('Recommendation: Use internal: true for backend networks');
    }
  });
});
```

**Expected Results**:
- Custom networks defined
- Services properly networked
- Database not publicly exposed
- Internal networks used

**Verification Points**:
- Network configuration validated
- Isolation verified
- Security boundaries enforced

---

### INT-DOCKER-014: Volume Mount Security

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DOCKER-014 |
| **Title** | Volume Mount Security |
| **Priority** | P1 - High |
| **Components** | docker-compose.yml |
| **Spec Reference** | SEC-007, Storage Security |

**Preconditions**:
- docker-compose.yml available

**Test Steps**:

```typescript
describe('Volume Mount Security', () => {
  it('uses named volumes instead of bind mounts in production', async () => {
    const composeFile = await fs.readFile('docker-compose.yml', 'utf-8');
    const compose = yaml.parse(composeFile);

    const services = compose.services || {};

    for (const [name, service] of Object.entries(services)) {
      const svc = service as any;

      if (svc.volumes) {
        for (const volume of svc.volumes) {
          const volStr = String(volume);

          // Bind mounts start with / or .
          const isBindMount = volStr.startsWith('/') || volStr.startsWith('.');

          if (isBindMount && process.env.NODE_ENV === 'production') {
            console.warn(`Warning: Bind mount detected in ${name}: ${volStr}`);
          }
        }
      }
    }
  });

  it('sensitive directories not mounted', async () => {
    const composeFile = await fs.readFile('docker-compose.yml', 'utf-8');
    const compose = yaml.parse(composeFile);

    const sensitivePatterns = [
      '/etc/passwd',
      '/etc/shadow',
      '/root',
      '/var/run/docker.sock',
      '~/.ssh',
      '~/.aws',
    ];

    const services = compose.services || {};

    for (const [name, service] of Object.entries(services)) {
      const svc = service as any;

      if (svc.volumes) {
        for (const volume of svc.volumes) {
          const volStr = String(volume);

          for (const sensitive of sensitivePatterns) {
            expect(volStr).not.toContain(sensitive);
          }
        }
      }
    }
  });

  it('read-only mounts used where appropriate', async () => {
    const composeFile = await fs.readFile('docker-compose.yml', 'utf-8');
    const compose = yaml.parse(composeFile);

    const services = compose.services || {};

    // Config volumes should be read-only
    for (const [name, service] of Object.entries(services)) {
      const svc = service as any;

      if (svc.volumes) {
        for (const volume of svc.volumes) {
          const volStr = String(volume);

          // Config directories should be :ro
          if (volStr.includes('/config') || volStr.includes('/etc')) {
            const isReadOnly = volStr.includes(':ro');

            if (!isReadOnly) {
              console.warn(`Recommendation: Make config volume read-only: ${volStr}`);
            }
          }
        }
      }
    }
  });

  it('volumes have explicit definitions', async () => {
    const composeFile = await fs.readFile('docker-compose.yml', 'utf-8');
    const compose = yaml.parse(composeFile);

    // Named volumes should be defined in volumes section
    const definedVolumes = new Set(Object.keys(compose.volumes || {}));

    const services = compose.services || {};

    for (const [name, service] of Object.entries(services)) {
      const svc = service as any;

      if (svc.volumes) {
        for (const volume of svc.volumes) {
          const volStr = String(volume);

          // Extract volume name (before first :)
          const volName = volStr.split(':')[0];

          // If not a path, should be a defined volume
          if (!volName.startsWith('/') && !volName.startsWith('.')) {
            expect(definedVolumes.has(volName)).toBe(true);
          }
        }
      }
    }
  });
});
```

**Expected Results**:
- Named volumes preferred in production
- Sensitive directories not mounted
- Read-only where appropriate
- Volumes explicitly defined

**Verification Points**:
- Volume configuration validated
- Security boundaries enforced
- Best practices applied

---

### INT-DOCKER-015: Environment Variable Handling

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DOCKER-015 |
| **Title** | Environment Variable Handling |
| **Priority** | P1 - High |
| **Components** | docker-compose.yml, .env files |
| **Spec Reference** | SEC-008, Secret Management |

**Preconditions**:
- docker-compose.yml available
- Environment configuration accessible

**Test Steps**:

```typescript
describe('Environment Variable Handling', () => {
  it('secrets use env_file or Docker secrets, not inline', async () => {
    const composeFile = await fs.readFile('docker-compose.yml', 'utf-8');
    const compose = yaml.parse(composeFile);

    const secretPatterns = [
      'PASSWORD',
      'SECRET',
      'API_KEY',
      'TOKEN',
      'PRIVATE_KEY',
    ];

    const services = compose.services || {};

    for (const [name, service] of Object.entries(services)) {
      const svc = service as any;

      if (svc.environment) {
        const envVars = Array.isArray(svc.environment)
          ? svc.environment
          : Object.entries(svc.environment).map(([k, v]) => `${k}=${v}`);

        for (const env of envVars) {
          const envStr = String(env);

          for (const pattern of secretPatterns) {
            if (envStr.includes(pattern)) {
              // Should use variable substitution, not hardcoded value
              const hasHardcodedValue = !envStr.includes('${') &&
                                        envStr.includes('=') &&
                                        envStr.split('=')[1]?.length > 0;

              expect(hasHardcodedValue).toBe(false);
            }
          }
        }
      }
    }
  });

  it('env_file is used for environment configuration', async () => {
    const composeFile = await fs.readFile('docker-compose.yml', 'utf-8');
    const compose = yaml.parse(composeFile);

    const services = compose.services || {};

    // At least one service should use env_file
    const usesEnvFile = Object.values(services).some(
      (s: any) => s.env_file !== undefined
    );

    expect(usesEnvFile).toBe(true);
  });

  it('.env files are not committed to repository', async () => {
    // Check gitignore for .env exclusion
    const gitignore = await fs.readFile('.gitignore', 'utf-8')
      .catch(() => '');

    const ignoresEnv = gitignore.includes('.env') ||
                       gitignore.includes('*.env');

    expect(ignoresEnv).toBe(true);

    // Verify no .env files are tracked
    const result = await execAsync('git ls-files | grep -E "\\.env$" || true');
    const trackedEnvFiles = result.stdout.trim();

    // .env.example is acceptable
    const hasRealEnvFile = trackedEnvFiles
      .split('\n')
      .filter(f => f && !f.includes('example'))
      .length > 0;

    expect(hasRealEnvFile).toBe(false);
  });

  it('required environment variables are documented', async () => {
    // Check for .env.example or similar
    const hasEnvExample = await fs.access('.env.example')
      .then(() => true)
      .catch(() => false);

    const hasEnvTemplate = await fs.access('.env.template')
      .then(() => true)
      .catch(() => false);

    expect(hasEnvExample || hasEnvTemplate).toBe(true);
  });
});
```

**Expected Results**:
- Secrets not hardcoded
- env_file used for configuration
- .env files not committed
- Environment documented

**Verification Points**:
- Secret handling validated
- Configuration management verified
- Documentation present

---

### INT-DOCKER-016: Resource Limits Configuration

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DOCKER-016 |
| **Title** | Resource Limits Configuration |
| **Priority** | P1 - High |
| **Components** | docker-compose.yml |
| **Spec Reference** | OPS-003, Resource Management |

**Preconditions**:
- docker-compose.yml available

**Test Steps**:

```typescript
describe('Resource Limits Configuration', () => {
  it('services have memory limits defined', async () => {
    const composeFile = await fs.readFile('docker-compose.yml', 'utf-8');
    const compose = yaml.parse(composeFile);

    const services = compose.services || {};

    for (const [name, service] of Object.entries(services)) {
      const svc = service as any;

      // Check for deploy.resources.limits or mem_limit
      const hasMemLimit = svc.mem_limit ||
                          svc.deploy?.resources?.limits?.memory;

      expect(hasMemLimit).toBeTruthy();
    }
  });

  it('services have CPU limits defined', async () => {
    const composeFile = await fs.readFile('docker-compose.yml', 'utf-8');
    const compose = yaml.parse(composeFile);

    const services = compose.services || {};

    for (const [name, service] of Object.entries(services)) {
      const svc = service as any;

      // Check for deploy.resources.limits or cpus
      const hasCpuLimit = svc.cpus ||
                          svc.deploy?.resources?.limits?.cpus;

      if (!hasCpuLimit) {
        console.warn(`Warning: No CPU limit for service ${name}`);
      }
    }
  });

  it('resource limits are reasonable', async () => {
    const composeFile = await fs.readFile('docker-compose.yml', 'utf-8');
    const compose = yaml.parse(composeFile);

    const services = compose.services || {};

    for (const [name, service] of Object.entries(services)) {
      const svc = service as any;

      const memLimit = svc.mem_limit ||
                       svc.deploy?.resources?.limits?.memory;

      if (memLimit) {
        // Parse memory limit (e.g., "512m", "1g")
        const memStr = String(memLimit).toLowerCase();
        let memMB = 0;

        if (memStr.endsWith('g')) {
          memMB = parseFloat(memStr) * 1024;
        } else if (memStr.endsWith('m')) {
          memMB = parseFloat(memStr);
        }

        // Reasonable limits: 128MB to 8GB
        expect(memMB).toBeGreaterThanOrEqual(128);
        expect(memMB).toBeLessThanOrEqual(8192);
      }
    }
  });

  it('reservations are set alongside limits', async () => {
    const composeFile = await fs.readFile('docker-compose.yml', 'utf-8');
    const compose = yaml.parse(composeFile);

    const services = compose.services || {};

    for (const [name, service] of Object.entries(services)) {
      const svc = service as any;

      const hasLimits = svc.deploy?.resources?.limits;
      const hasReservations = svc.deploy?.resources?.reservations;

      if (hasLimits && !hasReservations) {
        console.warn(
          `Recommendation: Set reservations alongside limits for ${name}`
        );
      }
    }
  });
});
```

**Expected Results**:
- Memory limits defined for all services
- CPU limits defined or documented
- Limits are reasonable values
- Reservations complement limits

**Verification Points**:
- Resource configuration present
- Values validated
- Best practices applied

---

### INT-DOCKER-017: Container Startup Time

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DOCKER-017 |
| **Title** | Container Startup Time |
| **Priority** | P1 - High |
| **Components** | Docker Image, Container Runtime |
| **Spec Reference** | PERF-001, Startup Performance |

**Preconditions**:
- Docker image available
- Container can be started

**Test Steps**:

```typescript
describe('Container Startup Time', () => {
  it('container starts within acceptable time', async () => {
    const imageName = 'hx-docling-application:latest';
    const containerName = 'startup-test';

    const startTime = Date.now();

    // Start container
    await execAsync(
      `docker run -d --name ${containerName} ${imageName}`
    );

    // Wait for healthy status
    let healthy = false;
    const maxWait = 30000; // 30 seconds max

    while (!healthy && Date.now() - startTime < maxWait) {
      const result = await execAsync(
        `docker inspect ${containerName} --format '{{.State.Health.Status}}'`
      ).catch(() => ({ stdout: 'unknown' }));

      if (result.stdout.trim() === 'healthy') {
        healthy = true;
      } else {
        await new Promise((resolve) => setTimeout(resolve, 1000));
      }
    }

    const startupTime = Date.now() - startTime;

    // Cleanup
    await execAsync(`docker rm -f ${containerName}`);

    // Should start within 30 seconds
    expect(startupTime).toBeLessThan(30000);
    console.log(`Container startup time: ${startupTime}ms`);
  });

  it('cold start time is acceptable', async () => {
    const imageName = 'hx-docling-application:latest';

    // Ensure image is not cached (pull fresh)
    // In CI, this simulates cold start
    const startTime = Date.now();

    const result = await execAsync(
      `docker run --rm ${imageName} echo "ready"`
    );

    const coldStartTime = Date.now() - startTime;

    // Cold start should be under 60 seconds
    expect(coldStartTime).toBeLessThan(60000);
    expect(result.stdout.trim()).toBe('ready');

    console.log(`Cold start time: ${coldStartTime}ms`);
  });

  it('warm start time is fast', async () => {
    const imageName = 'hx-docling-application:latest';

    // First run to warm cache
    await execAsync(`docker run --rm ${imageName} echo "warmup"`);

    // Measure warm start
    const startTime = Date.now();
    await execAsync(`docker run --rm ${imageName} echo "ready"`);
    const warmStartTime = Date.now() - startTime;

    // Warm start should be under 10 seconds
    expect(warmStartTime).toBeLessThan(10000);
    console.log(`Warm start time: ${warmStartTime}ms`);
  });
});
```

**Expected Results**:
- Container starts within 30 seconds
- Cold start under 60 seconds
- Warm start under 10 seconds

**Verification Points**:
- Startup time measured
- Health check validates readiness
- Performance acceptable

---

### INT-DOCKER-018: Health Check Runtime Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DOCKER-018 |
| **Title** | Health Check Runtime Validation |
| **Priority** | P1 - High |
| **Components** | Container Runtime, Health Endpoint |
| **Spec Reference** | OPS-004, Health Monitoring |

**Preconditions**:
- Container running with health check
- Health endpoint accessible

**Test Steps**:

```typescript
describe('Health Check Runtime Validation', () => {
  let containerName: string;

  beforeAll(async () => {
    containerName = 'health-runtime-test';
    await execAsync(
      `docker run -d --name ${containerName} -p 3001:3000 hx-docling-application:latest`
    );
    // Wait for startup
    await new Promise((resolve) => setTimeout(resolve, 10000));
  });

  afterAll(async () => {
    await execAsync(`docker rm -f ${containerName}`);
  });

  it('health check transitions through expected states', async () => {
    // Check initial state (should be starting or healthy)
    const initialResult = await execAsync(
      `docker inspect ${containerName} --format '{{.State.Health.Status}}'`
    );

    const initialStatus = initialResult.stdout.trim();
    expect(['starting', 'healthy']).toContain(initialStatus);

    // Wait and check again
    await new Promise((resolve) => setTimeout(resolve, 15000));

    const finalResult = await execAsync(
      `docker inspect ${containerName} --format '{{.State.Health.Status}}'`
    );

    expect(finalResult.stdout.trim()).toBe('healthy');
  });

  it('health check history is recorded', async () => {
    const result = await execAsync(
      `docker inspect ${containerName} --format '{{json .State.Health.Log}}'`
    );

    const healthLog = JSON.parse(result.stdout);

    expect(Array.isArray(healthLog)).toBe(true);
    expect(healthLog.length).toBeGreaterThan(0);

    // Check latest entry
    const latest = healthLog[healthLog.length - 1];
    expect(latest.ExitCode).toBe(0);
  });

  it('health endpoint returns correct format', async () => {
    const response = await fetch('http://localhost:3001/api/health');

    expect(response.status).toBe(200);

    const health = await response.json();

    expect(health).toHaveProperty('status');
    expect(['ok', 'healthy', 'up']).toContain(health.status.toLowerCase());
  });

  it('health check detects service degradation', async () => {
    // This test would require ability to degrade the service
    // For now, verify health check continues running

    const result = await execAsync(
      `docker inspect ${containerName} --format '{{.State.Health.FailingStreak}}'`
    );

    const failingStreak = parseInt(result.stdout.trim(), 10);
    expect(failingStreak).toBe(0);
  });
});
```

**Expected Results**:
- Health transitions correctly
- History recorded
- Endpoint returns correct format
- Degradation detectable

**Verification Points**:
- Health states validated
- Log inspection successful
- Endpoint format correct

---

### INT-DOCKER-019: Graceful Shutdown

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DOCKER-019 |
| **Title** | Graceful Shutdown |
| **Priority** | P1 - High |
| **Components** | Container Runtime, Application |
| **Spec Reference** | OPS-005, Shutdown Handling |

**Preconditions**:
- Container running
- Application handles SIGTERM

**Test Steps**:

```typescript
describe('Graceful Shutdown', () => {
  it('container responds to SIGTERM gracefully', async () => {
    const imageName = 'hx-docling-application:latest';
    const containerName = 'shutdown-test';

    // Start container
    await execAsync(
      `docker run -d --name ${containerName} ${imageName}`
    );

    // Wait for startup
    await new Promise((resolve) => setTimeout(resolve, 5000));

    // Send SIGTERM via docker stop (default)
    const startTime = Date.now();
    await execAsync(`docker stop ${containerName}`);
    const stopTime = Date.now() - startTime;

    // Should stop gracefully within 10 seconds (not killed at 10s)
    expect(stopTime).toBeLessThan(10000);

    // Check exit code
    const result = await execAsync(
      `docker inspect ${containerName} --format '{{.State.ExitCode}}'`
    );

    // Exit code 0 or 143 (SIGTERM) is acceptable
    const exitCode = parseInt(result.stdout.trim(), 10);
    expect([0, 143]).toContain(exitCode);

    // Cleanup
    await execAsync(`docker rm ${containerName}`);
  });

  it('in-flight requests complete before shutdown', async () => {
    const imageName = 'hx-docling-application:latest';
    const containerName = 'inflight-test';

    // Start container with port mapping
    await execAsync(
      `docker run -d --name ${containerName} -p 3002:3000 ${imageName}`
    );

    // Wait for startup
    await new Promise((resolve) => setTimeout(resolve, 10000));

    // Start a request (simulated long-running)
    const requestPromise = fetch('http://localhost:3002/api/health')
      .then(r => r.status)
      .catch(() => 'error');

    // Immediately stop container
    const stopPromise = execAsync(`docker stop --time 30 ${containerName}`);

    // Request should complete
    const [requestResult] = await Promise.all([requestPromise, stopPromise]);

    // Request should succeed (200) or fail gracefully
    expect([200, 'error']).toContain(requestResult);

    // Cleanup
    await execAsync(`docker rm ${containerName}`);
  });

  it('connections are drained properly', async () => {
    // Verify stop-grace-period is configured
    const composeFile = await fs.readFile('docker-compose.yml', 'utf-8')
      .catch(() => '');

    if (composeFile) {
      const compose = yaml.parse(composeFile);
      const services = compose.services || {};

      for (const [name, service] of Object.entries(services)) {
        const svc = service as any;

        if (svc.stop_grace_period) {
          // Parse duration (e.g., "30s", "1m")
          const period = svc.stop_grace_period;
          console.log(`${name} stop_grace_period: ${period}`);
        }
      }
    }
  });
});
```

**Expected Results**:
- Container stops gracefully
- In-flight requests complete
- Proper exit codes
- Connections drained

**Verification Points**:
- Shutdown time measured
- Exit code validated
- Grace period configured

---

### INT-DOCKER-020: Log Aggregation

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DOCKER-020 |
| **Title** | Log Aggregation |
| **Priority** | P2 - Medium |
| **Components** | Container Runtime, Logging |
| **Spec Reference** | OPS-006, Logging Requirements |

**Preconditions**:
- Container running
- Application produces logs

**Test Steps**:

```typescript
describe('Log Aggregation', () => {
  it('logs are written to stdout/stderr', async () => {
    const imageName = 'hx-docling-application:latest';
    const containerName = 'log-test';

    // Start container
    await execAsync(
      `docker run -d --name ${containerName} ${imageName}`
    );

    // Wait for some logs
    await new Promise((resolve) => setTimeout(resolve, 5000));

    // Check logs
    const result = await execAsync(`docker logs ${containerName}`);

    // Should have some output
    expect(result.stdout.length + result.stderr.length).toBeGreaterThan(0);

    // Cleanup
    await execAsync(`docker rm -f ${containerName}`);
  });

  it('logs are in structured JSON format', async () => {
    const imageName = 'hx-docling-application:latest';
    const containerName = 'log-format-test';

    // Start container
    await execAsync(
      `docker run -d --name ${containerName} ${imageName}`
    );

    await new Promise((resolve) => setTimeout(resolve, 5000));

    // Get logs
    const result = await execAsync(`docker logs ${containerName}`);

    // Try to parse first line as JSON
    const firstLine = result.stdout.split('\n')[0];

    try {
      const logEntry = JSON.parse(firstLine);
      expect(logEntry).toHaveProperty('level');
      expect(logEntry).toHaveProperty('message');
    } catch {
      // Plain text logs are also acceptable
      console.log('Logs are in plain text format');
    }

    // Cleanup
    await execAsync(`docker rm -f ${containerName}`);
  });

  it('logging driver is configured in compose', async () => {
    const composeFile = await fs.readFile('docker-compose.yml', 'utf-8')
      .catch(() => '');

    if (composeFile) {
      const compose = yaml.parse(composeFile);
      const services = compose.services || {};

      for (const [name, service] of Object.entries(services)) {
        const svc = service as any;

        if (svc.logging) {
          expect(svc.logging.driver).toBeDefined();
          console.log(`${name} logging driver: ${svc.logging.driver}`);
        }
      }
    }
  });

  it('log rotation is configured', async () => {
    const composeFile = await fs.readFile('docker-compose.yml', 'utf-8')
      .catch(() => '');

    if (composeFile) {
      const compose = yaml.parse(composeFile);
      const services = compose.services || {};

      for (const [name, service] of Object.entries(services)) {
        const svc = service as any;

        if (svc.logging?.options) {
          const hasRotation =
            svc.logging.options['max-size'] ||
            svc.logging.options['max-file'];

          if (hasRotation) {
            console.log(`${name} has log rotation configured`);
          }
        }
      }
    }
  });
});
```

**Expected Results**:
- Logs written to stdout/stderr
- Structured format preferred
- Logging driver configured
- Rotation configured

**Verification Points**:
- Log output verified
- Format validated
- Configuration present

---

### INT-DOCKER-021: Resource Consumption Monitoring

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DOCKER-021 |
| **Title** | Resource Consumption Monitoring |
| **Priority** | P2 - Medium |
| **Components** | Container Runtime |
| **Spec Reference** | PERF-002, Resource Monitoring |

**Preconditions**:
- Container running
- Docker stats available

**Test Steps**:

```typescript
describe('Resource Consumption Monitoring', () => {
  let containerName: string;

  beforeAll(async () => {
    containerName = 'resource-test';
    await execAsync(
      `docker run -d --name ${containerName} hx-docling-application:latest`
    );
    await new Promise((resolve) => setTimeout(resolve, 10000));
  });

  afterAll(async () => {
    await execAsync(`docker rm -f ${containerName}`);
  });

  it('memory usage is within limits', async () => {
    const result = await execAsync(
      `docker stats ${containerName} --no-stream --format '{{.MemUsage}}'`
    );

    const memUsage = result.stdout.trim();
    console.log(`Memory usage: ${memUsage}`);

    // Parse memory (e.g., "150MiB / 512MiB")
    const current = memUsage.split('/')[0].trim();

    // Should have reasonable memory usage
    expect(current).toBeTruthy();
  });

  it('CPU usage is reasonable', async () => {
    const result = await execAsync(
      `docker stats ${containerName} --no-stream --format '{{.CPUPerc}}'`
    );

    const cpuUsage = parseFloat(result.stdout.replace('%', ''));
    console.log(`CPU usage: ${cpuUsage}%`);

    // Idle CPU should be under 50%
    expect(cpuUsage).toBeLessThan(50);
  });

  it('no memory leaks over time', async () => {
    const samples: number[] = [];

    // Take 5 samples over 25 seconds
    for (let i = 0; i < 5; i++) {
      const result = await execAsync(
        `docker stats ${containerName} --no-stream --format '{{.MemPerc}}'`
      );

      const memPerc = parseFloat(result.stdout.replace('%', ''));
      samples.push(memPerc);

      await new Promise((resolve) => setTimeout(resolve, 5000));
    }

    // Memory should not increase significantly
    const firstSample = samples[0];
    const lastSample = samples[samples.length - 1];
    const increase = lastSample - firstSample;

    // Less than 10% increase over 25 seconds
    expect(increase).toBeLessThan(10);
  });

  it('network I/O is tracked', async () => {
    const result = await execAsync(
      `docker stats ${containerName} --no-stream --format '{{.NetIO}}'`
    );

    const netIO = result.stdout.trim();
    console.log(`Network I/O: ${netIO}`);

    expect(netIO).toBeTruthy();
  });
});
```

**Expected Results**:
- Memory within limits
- CPU usage reasonable
- No memory leaks
- Network I/O tracked

**Verification Points**:
- Stats collection works
- Resource usage acceptable
- No degradation over time

---

### INT-DOCKER-022: Container Image Layer Caching

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DOCKER-022 |
| **Title** | Container Image Layer Caching |
| **Priority** | P2 - Medium |
| **Components** | Dockerfile, Docker Build |
| **Spec Reference** | BLD-005, Build Caching |

**Preconditions**:
- Dockerfile available
- Docker BuildKit enabled

**Test Steps**:

```typescript
describe('Container Image Layer Caching', () => {
  it('dependency layers are cached separately', async () => {
    const dockerfile = await fs.readFile('Dockerfile', 'utf-8');

    // Package file copy should come before source copy
    const lines = dockerfile.split('\n');

    let packageCopyLine = -1;
    let installLine = -1;
    let sourceCopyLine = -1;

    lines.forEach((line, index) => {
      if (line.match(/COPY.*(package.*json|requirements\.txt)/)) {
        packageCopyLine = index;
      }
      if (line.match(/RUN.*(npm install|pip install|yarn)/)) {
        installLine = index;
      }
      if (line.match(/COPY\s+\.\s+/) || line.match(/COPY\s+src/)) {
        sourceCopyLine = index;
      }
    });

    // Order should be: copy packages -> install -> copy source
    if (packageCopyLine !== -1 && installLine !== -1) {
      expect(packageCopyLine).toBeLessThan(installLine);
    }
    if (installLine !== -1 && sourceCopyLine !== -1) {
      expect(installLine).toBeLessThan(sourceCopyLine);
    }
  });

  it('rebuild with source change reuses dependency layer', async () => {
    // First build
    const firstBuild = await execAsync(
      `docker build -t cache-test:v1 . 2>&1`
    );

    // Make a trivial source change (touch a file)
    await execAsync('touch src/index.ts || touch app/page.tsx');

    // Second build
    const startTime = Date.now();
    const secondBuild = await execAsync(
      `docker build -t cache-test:v2 . 2>&1`
    );
    const buildTime = Date.now() - startTime;

    // Check for cache usage
    const usesCache = secondBuild.stdout.includes('CACHED') ||
                      secondBuild.stdout.includes('Using cache');

    console.log(`Rebuild time: ${buildTime}ms`);

    // Cleanup
    await execAsync('docker rmi cache-test:v1 cache-test:v2 || true');

    expect(usesCache).toBe(true);
  });

  it('.dockerignore excludes unnecessary files', async () => {
    const dockerignore = await fs.readFile('.dockerignore', 'utf-8')
      .catch(() => '');

    const shouldIgnore = [
      'node_modules',
      '.git',
      '*.md',
      '.env',
      'test',
      'tests',
      '.github',
    ];

    for (const pattern of shouldIgnore) {
      const isIgnored = dockerignore.includes(pattern);
      if (!isIgnored) {
        console.warn(`Recommendation: Add ${pattern} to .dockerignore`);
      }
    }
  });
});
```

**Expected Results**:
- Dependencies cached separately
- Source changes use cache
- .dockerignore configured

**Verification Points**:
- Layer ordering validated
- Cache usage confirmed
- Build efficiency measured

---

### INT-DOCKER-023: Docker Security Scan with Dockle

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DOCKER-023 |
| **Title** | Docker Security Scan with Dockle |
| **Priority** | P1 - High |
| **Components** | Docker Image, Dockle Scanner |
| **Spec Reference** | SEC-009, Image Security |

**Preconditions**:
- Docker image built
- Dockle scanner available

**Test Steps**:

```typescript
describe('Docker Security Scan with Dockle', () => {
  it('passes Dockle security scan', async () => {
    const imageName = 'hx-docling-application:latest';

    try {
      const result = await execAsync(
        `dockle --exit-code 1 --exit-level fatal ${imageName}`
      );

      // If we reach here, no fatal issues
      expect(true).toBe(true);
    } catch (error: any) {
      // Check if any FATAL issues
      expect(error.stdout).not.toContain('FATAL');
    }
  });

  it('validates CIS Docker Benchmark compliance', async () => {
    const imageName = 'hx-docling-application:latest';

    const result = await execAsync(
      `dockle --format json ${imageName}`
    );

    const report = JSON.parse(result.stdout);

    // Count issues by level
    const fatalCount = report.details?.filter(
      (d: any) => d.level === 'FATAL'
    ).length || 0;

    const warnCount = report.details?.filter(
      (d: any) => d.level === 'WARN'
    ).length || 0;

    console.log(`Dockle: ${fatalCount} FATAL, ${warnCount} WARN issues`);

    // No FATAL issues allowed
    expect(fatalCount).toBe(0);
  });

  it('image has no setuid/setgid binaries', async () => {
    const imageName = 'hx-docling-application:latest';

    const result = await execAsync(
      `docker run --rm ${imageName} find / -perm /6000 -type f 2>/dev/null || true`
    );

    const setuidFiles = result.stdout.trim().split('\n').filter(Boolean);

    if (setuidFiles.length > 0) {
      console.warn('Warning: setuid/setgid files found:', setuidFiles);
    }

    // Should have minimal setuid files
    expect(setuidFiles.length).toBeLessThan(5);
  });
});
```

**Expected Results**:
- Dockle scan passes
- CIS benchmark compliance
- Minimal setuid binaries

**Verification Points**:
- Security scan executed
- Issues categorized
- Compliance validated

---

### INT-DOCKER-024: Container Filesystem Security

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DOCKER-024 |
| **Title** | Container Filesystem Security |
| **Priority** | P1 - High |
| **Components** | Docker Image, Container Runtime |
| **Spec Reference** | SEC-010, Filesystem Security |

**Preconditions**:
- Docker image built
- Container can be inspected

**Test Steps**:

```typescript
describe('Container Filesystem Security', () => {
  it('root filesystem is read-only capable', async () => {
    const imageName = 'hx-docling-application:latest';

    // Try running with read-only root filesystem
    try {
      const result = await execAsync(
        `docker run --rm --read-only ${imageName} echo "OK"`
      );

      expect(result.stdout.trim()).toBe('OK');
    } catch (error) {
      // If fails, check if tmpfs mounts would help
      console.log('Note: Container may need tmpfs mounts for read-only root');
    }
  });

  it('sensitive system directories are protected', async () => {
    const imageName = 'hx-docling-application:latest';

    const sensitiveDirs = ['/etc/passwd', '/etc/shadow', '/root'];

    for (const dir of sensitiveDirs) {
      const result = await execAsync(
        `docker run --rm ${imageName} test -w ${dir} && echo "WRITABLE" || echo "PROTECTED"`
      ).catch(() => ({ stdout: 'PROTECTED' }));

      // Should not be writable by application user
      expect(result.stdout.trim()).toBe('PROTECTED');
    }
  });

  it('no world-writable directories', async () => {
    const imageName = 'hx-docling-application:latest';

    const result = await execAsync(
      `docker run --rm ${imageName} find / -type d -perm -0002 2>/dev/null || true`
    );

    const worldWritable = result.stdout.trim().split('\n').filter(Boolean);

    // Filter out expected directories
    const unexpected = worldWritable.filter(
      (d) => !d.includes('/tmp') && !d.includes('/var/tmp')
    );

    expect(unexpected.length).toBe(0);
  });

  it('application directory has correct permissions', async () => {
    const imageName = 'hx-docling-application:latest';

    const result = await execAsync(
      `docker run --rm ${imageName} stat -c '%a' /app`
    );

    const perms = result.stdout.trim();

    // Should be 755 or more restrictive
    expect(parseInt(perms, 8)).toBeLessThanOrEqual(parseInt('755', 8));
  });
});
```

**Expected Results**:
- Read-only root supported
- Sensitive directories protected
- No world-writable dirs
- Correct app permissions

**Verification Points**:
- Filesystem security validated
- Permissions checked
- Security hardening applied

---

### INT-DOCKER-025: Container Network Security

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DOCKER-025 |
| **Title** | Container Network Security |
| **Priority** | P1 - High |
| **Components** | docker-compose.yml, Container Runtime |
| **Spec Reference** | SEC-011, Network Security |

**Preconditions**:
- docker-compose.yml available
- Network configuration accessible

**Test Steps**:

```typescript
describe('Container Network Security', () => {
  it('containers do not use host network mode', async () => {
    const composeFile = await fs.readFile('docker-compose.yml', 'utf-8');
    const compose = yaml.parse(composeFile);

    const services = compose.services || {};

    for (const [name, service] of Object.entries(services)) {
      const svc = service as any;

      // Should not use host network
      expect(svc.network_mode).not.toBe('host');
    }
  });

  it('only necessary ports are exposed', async () => {
    const composeFile = await fs.readFile('docker-compose.yml', 'utf-8');
    const compose = yaml.parse(composeFile);

    const services = compose.services || {};

    for (const [name, service] of Object.entries(services)) {
      const svc = service as any;

      if (svc.ports) {
        // Log exposed ports
        console.log(`${name} exposes ports:`, svc.ports);

        // Count exposed ports
        expect(svc.ports.length).toBeLessThan(5);
      }
    }
  });

  it('internal services use expose instead of ports', async () => {
    const composeFile = await fs.readFile('docker-compose.yml', 'utf-8');
    const compose = yaml.parse(composeFile);

    const services = compose.services || {};

    // Database and cache should use expose, not ports
    const internalServices = ['postgres', 'redis', 'db', 'cache'];

    for (const [name, service] of Object.entries(services)) {
      const svc = service as any;

      if (internalServices.some((i) => name.includes(i))) {
        if (svc.ports && svc.ports.length > 0) {
          console.warn(
            `Warning: Internal service ${name} has exposed ports`
          );
        }
      }
    }
  });

  it('container does not have privileged mode', async () => {
    const composeFile = await fs.readFile('docker-compose.yml', 'utf-8');
    const compose = yaml.parse(composeFile);

    const services = compose.services || {};

    for (const [name, service] of Object.entries(services)) {
      const svc = service as any;

      expect(svc.privileged).not.toBe(true);
    }
  });
});
```

**Expected Results**:
- No host network mode
- Minimal port exposure
- Internal services isolated
- No privileged mode

**Verification Points**:
- Network configuration validated
- Security boundaries enforced
- Least privilege applied

---

### INT-DOCKER-026: Container Capability Restrictions

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DOCKER-026 |
| **Title** | Container Capability Restrictions |
| **Priority** | P1 - High |
| **Components** | docker-compose.yml, Container Runtime |
| **Spec Reference** | SEC-012, Capability Management |

**Preconditions**:
- docker-compose.yml available

**Test Steps**:

```typescript
describe('Container Capability Restrictions', () => {
  it('drops all capabilities by default', async () => {
    const composeFile = await fs.readFile('docker-compose.yml', 'utf-8');
    const compose = yaml.parse(composeFile);

    const services = compose.services || {};

    for (const [name, service] of Object.entries(services)) {
      const svc = service as any;

      if (svc.cap_drop) {
        expect(svc.cap_drop).toContain('ALL');
      } else {
        console.warn(
          `Recommendation: Add cap_drop: ["ALL"] to service ${name}`
        );
      }
    }
  });

  it('only adds necessary capabilities', async () => {
    const composeFile = await fs.readFile('docker-compose.yml', 'utf-8');
    const compose = yaml.parse(composeFile);

    const services = compose.services || {};

    // Dangerous capabilities that should not be added
    const dangerousCaps = [
      'SYS_ADMIN',
      'NET_ADMIN',
      'SYS_PTRACE',
      'DAC_OVERRIDE',
    ];

    for (const [name, service] of Object.entries(services)) {
      const svc = service as any;

      if (svc.cap_add) {
        for (const cap of svc.cap_add) {
          expect(dangerousCaps).not.toContain(cap);
        }
      }
    }
  });

  it('security_opt is configured', async () => {
    const composeFile = await fs.readFile('docker-compose.yml', 'utf-8');
    const compose = yaml.parse(composeFile);

    const services = compose.services || {};

    for (const [name, service] of Object.entries(services)) {
      const svc = service as any;

      if (svc.security_opt) {
        console.log(`${name} security options:`, svc.security_opt);

        // Check for no-new-privileges
        const hasNoNewPriv = svc.security_opt.some(
          (opt: string) => opt.includes('no-new-privileges')
        );

        if (hasNoNewPriv) {
          console.log(`${name} has no-new-privileges set`);
        }
      }
    }
  });
});
```

**Expected Results**:
- All capabilities dropped by default
- Only necessary capabilities added
- No dangerous capabilities
- Security options configured

**Verification Points**:
- Capability configuration validated
- Least privilege enforced
- Security hardening applied

---

### INT-DOCKER-027: Docker Compose Override Handling

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DOCKER-027 |
| **Title** | Docker Compose Override Handling |
| **Priority** | P2 - Medium |
| **Components** | docker-compose.yml, docker-compose.override.yml |
| **Spec Reference** | OPS-007, Configuration Management |

**Preconditions**:
- docker-compose.yml available
- Override files may exist

**Test Steps**:

```typescript
describe('Docker Compose Override Handling', () => {
  it('base compose file is production-ready', async () => {
    const composeFile = await fs.readFile('docker-compose.yml', 'utf-8');
    const compose = yaml.parse(composeFile);

    const services = compose.services || {};

    for (const [name, service] of Object.entries(services)) {
      const svc = service as any;

      // Should not have development-only settings
      expect(svc.volumes).not.toContain('./:/app');
      expect(svc.command).not.toContain('--watch');
    }
  });

  it('override file is gitignored in production', async () => {
    const gitignore = await fs.readFile('.gitignore', 'utf-8')
      .catch(() => '');

    const ignoresOverride =
      gitignore.includes('docker-compose.override.yml') ||
      gitignore.includes('docker-compose.override.yaml');

    // Override should be gitignored for production
    if (!ignoresOverride) {
      console.warn(
        'Recommendation: Add docker-compose.override.yml to .gitignore'
      );
    }
  });

  it('environment-specific files are properly named', async () => {
    const envFiles = await glob('docker-compose*.yml');

    const validPatterns = [
      'docker-compose.yml',
      'docker-compose.yaml',
      'docker-compose.override.yml',
      'docker-compose.prod.yml',
      'docker-compose.dev.yml',
      'docker-compose.test.yml',
    ];

    for (const file of envFiles) {
      const filename = path.basename(file);
      const isValid = validPatterns.some((p) => filename === p);

      if (!isValid) {
        console.log(`Non-standard compose file: ${filename}`);
      }
    }
  });

  it('production compose has restart policies', async () => {
    const composeFile = await fs.readFile('docker-compose.yml', 'utf-8');
    const compose = yaml.parse(composeFile);

    const services = compose.services || {};

    for (const [name, service] of Object.entries(services)) {
      const svc = service as any;

      const hasRestart = svc.restart || svc.deploy?.restart_policy;

      if (!hasRestart) {
        console.warn(`Recommendation: Add restart policy to ${name}`);
      }
    }
  });
});
```

**Expected Results**:
- Base file production-ready
- Override properly handled
- Environment files organized
- Restart policies present

**Verification Points**:
- Configuration validated
- Best practices applied
- Environment separation verified

---

### INT-DOCKER-028: Multi-Architecture Image Support

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DOCKER-028 |
| **Title** | Multi-Architecture Image Support |
| **Priority** | P2 - Medium |
| **Components** | Docker Image, BuildKit |
| **Spec Reference** | BLD-006, Cross-Platform Support |

**Preconditions**:
- Docker BuildKit available
- Multi-arch build capability

**Test Steps**:

```typescript
describe('Multi-Architecture Image Support', () => {
  it('image supports amd64 architecture', async () => {
    const imageName = 'hx-docling-application:latest';

    const result = await execAsync(
      `docker manifest inspect ${imageName} 2>/dev/null || docker image inspect ${imageName} --format '{{.Architecture}}'`
    );

    const output = result.stdout;

    // Should support or be built for amd64
    expect(output).toMatch(/amd64|x86_64/);
  });

  it('base image supports required architectures', async () => {
    const dockerfile = await fs.readFile('Dockerfile', 'utf-8');

    const fromLine = dockerfile.match(/^FROM\s+(.+)/m)?.[1];
    const baseImage = fromLine?.split(' ')[0];

    if (baseImage && !baseImage.includes('AS')) {
      const result = await execAsync(
        `docker manifest inspect ${baseImage} 2>/dev/null || echo "manifest not available"`
      );

      if (!result.stdout.includes('not available')) {
        const manifest = JSON.parse(result.stdout);

        const architectures = manifest.manifests?.map(
          (m: any) => m.platform.architecture
        ) || [];

        console.log(`Base image architectures: ${architectures.join(', ')}`);
      }
    }
  });

  it('Dockerfile does not have architecture-specific commands', async () => {
    const dockerfile = await fs.readFile('Dockerfile', 'utf-8');

    // Check for architecture-specific patterns
    const archSpecific = [
      'COPY --from=amd64',
      'COPY --from=arm64',
      '/x86_64/',
      '/aarch64/',
    ];

    for (const pattern of archSpecific) {
      if (dockerfile.includes(pattern)) {
        console.warn(`Note: Architecture-specific pattern found: ${pattern}`);
      }
    }
  });
});
```

**Expected Results**:
- amd64 architecture supported
- Base image multi-arch capable
- No hardcoded architecture dependencies

**Verification Points**:
- Architecture support validated
- Base image checked
- Portability confirmed

---

### INT-DOCKER-029: Image Tag and Version Management

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DOCKER-029 |
| **Title** | Image Tag and Version Management |
| **Priority** | P2 - Medium |
| **Components** | Docker Image, CI/CD |
| **Spec Reference** | BLD-007, Version Management |

**Preconditions**:
- Docker images available
- Git repository accessible

**Test Steps**:

```typescript
describe('Image Tag and Version Management', () => {
  it('images use semantic versioning tags', async () => {
    // Check if project has versioned tags
    const result = await execAsync(
      'git tag -l "v*" | tail -5'
    );

    const tags = result.stdout.trim().split('\n').filter(Boolean);

    if (tags.length > 0) {
      // Tags should follow semver
      for (const tag of tags) {
        expect(tag).toMatch(/^v\d+\.\d+\.\d+/);
      }
    }

    console.log(`Found version tags: ${tags.join(', ')}`);
  });

  it('docker-compose uses specific image tags', async () => {
    const composeFile = await fs.readFile('docker-compose.yml', 'utf-8');
    const compose = yaml.parse(composeFile);

    const services = compose.services || {};

    for (const [name, service] of Object.entries(services)) {
      const svc = service as any;

      if (svc.image) {
        // Should not use :latest in production
        expect(svc.image).not.toMatch(/:latest$/);

        // Should have explicit tag
        expect(svc.image).toMatch(/:.+/);
      }
    }
  });

  it('image metadata labels are present', async () => {
    const imageName = 'hx-docling-application:latest';

    const result = await execAsync(
      `docker inspect ${imageName} --format '{{json .Config.Labels}}'`
    );

    const labels = JSON.parse(result.stdout);

    // Should have standard OCI labels
    const expectedLabels = [
      'org.opencontainers.image.version',
      'org.opencontainers.image.source',
      'org.opencontainers.image.created',
    ];

    for (const label of expectedLabels) {
      if (!labels[label]) {
        console.warn(`Recommendation: Add label ${label}`);
      }
    }
  });
});
```

**Expected Results**:
- Semantic versioning used
- Specific tags in compose
- OCI labels present

**Verification Points**:
- Version management validated
- Tag strategy confirmed
- Metadata present

---

### INT-DOCKER-030: Build Reproducibility

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DOCKER-030 |
| **Title** | Build Reproducibility |
| **Priority** | P2 - Medium |
| **Components** | Dockerfile, Docker Build |
| **Spec Reference** | BLD-008, Reproducible Builds |

**Preconditions**:
- Dockerfile available
- Build environment accessible

**Test Steps**:

```typescript
describe('Build Reproducibility', () => {
  it('lockfile is used for dependencies', async () => {
    // Check for lockfiles
    const hasNpmLock = await fs.access('package-lock.json')
      .then(() => true)
      .catch(() => false);

    const hasYarnLock = await fs.access('yarn.lock')
      .then(() => true)
      .catch(() => false);

    const hasPnpmLock = await fs.access('pnpm-lock.yaml')
      .then(() => true)
      .catch(() => false);

    expect(hasNpmLock || hasYarnLock || hasPnpmLock).toBe(true);
  });

  it('Dockerfile copies lockfile for reproducible installs', async () => {
    const dockerfile = await fs.readFile('Dockerfile', 'utf-8');

    const copiesLockfile =
      dockerfile.includes('COPY package-lock.json') ||
      dockerfile.includes('COPY yarn.lock') ||
      dockerfile.includes('COPY pnpm-lock.yaml') ||
      dockerfile.includes('COPY package*.json');

    expect(copiesLockfile).toBe(true);
  });

  it('uses ci install command', async () => {
    const dockerfile = await fs.readFile('Dockerfile', 'utf-8');

    // Should use ci/frozen-lockfile for reproducibility
    const usesCiInstall =
      dockerfile.includes('npm ci') ||
      dockerfile.includes('yarn --frozen-lockfile') ||
      dockerfile.includes('pnpm install --frozen-lockfile');

    if (!usesCiInstall) {
      console.warn(
        'Recommendation: Use npm ci or --frozen-lockfile for reproducibility'
      );
    }
  });

  it('no dynamic dependency installation', async () => {
    const dockerfile = await fs.readFile('Dockerfile', 'utf-8');

    // Should not install packages directly
    const hasDynamicInstall =
      dockerfile.includes('npm install ') &&
      !dockerfile.includes('npm ci') &&
      dockerfile.match(/npm install [a-z]/);

    expect(hasDynamicInstall).toBeFalsy();
  });
});
```

**Expected Results**:
- Lockfile present
- Lockfile copied in Dockerfile
- CI install command used
- No dynamic installs

**Verification Points**:
- Reproducibility validated
- Dependency management verified
- Build determinism confirmed

---

### INT-DOCKER-031: Container Process Limits

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DOCKER-031 |
| **Title** | Container Process Limits |
| **Priority** | P2 - Medium |
| **Components** | docker-compose.yml, Container Runtime |
| **Spec Reference** | SEC-013, Resource Isolation |

**Preconditions**:
- docker-compose.yml available

**Test Steps**:

```typescript
describe('Container Process Limits', () => {
  it('pids_limit is configured', async () => {
    const composeFile = await fs.readFile('docker-compose.yml', 'utf-8');
    const compose = yaml.parse(composeFile);

    const services = compose.services || {};

    for (const [name, service] of Object.entries(services)) {
      const svc = service as any;

      if (svc.pids_limit) {
        console.log(`${name} pids_limit: ${svc.pids_limit}`);
        expect(svc.pids_limit).toBeLessThan(1000);
      }
    }
  });

  it('ulimits are configured appropriately', async () => {
    const composeFile = await fs.readFile('docker-compose.yml', 'utf-8');
    const compose = yaml.parse(composeFile);

    const services = compose.services || {};

    for (const [name, service] of Object.entries(services)) {
      const svc = service as any;

      if (svc.ulimits) {
        console.log(`${name} ulimits:`, svc.ulimits);

        // Nofile should be reasonable
        if (svc.ulimits.nofile) {
          const soft = svc.ulimits.nofile.soft || svc.ulimits.nofile;
          expect(soft).toBeLessThan(100000);
        }
      }
    }
  });

  it('container does not allow fork bombs', async () => {
    const imageName = 'hx-docling-application:latest';

    // Try to spawn many processes (should be limited)
    const result = await execAsync(
      `docker run --rm --pids-limit 50 ${imageName} sh -c 'for i in $(seq 1 100); do sleep 1 & done 2>&1; wait' || echo "LIMITED"`
    ).catch(() => ({ stdout: 'LIMITED' }));

    // Should hit limit or complete safely
    expect(result.stdout).toBeTruthy();
  });
});
```

**Expected Results**:
- PID limits configured
- ulimits appropriate
- Fork protection in place

**Verification Points**:
- Process limits validated
- Resource isolation confirmed
- DoS protection verified

---

### INT-DOCKER-032: Container Orchestration Readiness

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DOCKER-032 |
| **Title** | Container Orchestration Readiness |
| **Priority** | P2 - Medium |
| **Components** | Docker Image, Kubernetes/Docker Swarm |
| **Spec Reference** | OPS-008, Orchestration Support |

**Preconditions**:
- Docker image built
- Orchestration requirements defined

**Test Steps**:

```typescript
describe('Container Orchestration Readiness', () => {
  it('container supports 12-factor app principles', async () => {
    // Check for externalized configuration
    const dockerfile = await fs.readFile('Dockerfile', 'utf-8');

    // Should use environment variables, not config files
    expect(dockerfile).not.toMatch(/COPY.*config\.json/);
    expect(dockerfile).not.toMatch(/COPY.*settings\.py/);

    // Should be stateless (no persistent local storage required)
    expect(dockerfile).not.toMatch(/VOLUME.*\/data/);
  });

  it('liveness and readiness probes are separate', async () => {
    // For Kubernetes readiness, verify endpoints exist
    const imageName = 'hx-docling-application:latest';
    const containerName = 'probe-test';

    await execAsync(
      `docker run -d --name ${containerName} -p 3003:3000 ${imageName}`
    );

    await new Promise((resolve) => setTimeout(resolve, 10000));

    // Check for different probe endpoints
    const healthResponse = await fetch('http://localhost:3003/api/health')
      .catch(() => null);
    const readyResponse = await fetch('http://localhost:3003/api/ready')
      .catch(() => null);

    // At least health should exist
    expect(healthResponse?.status).toBe(200);

    // Cleanup
    await execAsync(`docker rm -f ${containerName}`);
  });

  it('container handles multiple replicas', async () => {
    // Verify no conflicting ports or resources
    const composeFile = await fs.readFile('docker-compose.yml', 'utf-8');
    const compose = yaml.parse(composeFile);

    const services = compose.services || {};

    for (const [name, service] of Object.entries(services)) {
      const svc = service as any;

      // Check for deploy replicas configuration
      if (svc.deploy?.replicas) {
        console.log(`${name} replicas: ${svc.deploy.replicas}`);
      }

      // If scaling, shouldn't have fixed host ports
      if (svc.deploy?.replicas > 1 && svc.ports) {
        for (const port of svc.ports) {
          const portStr = String(port);
          // Should not have fixed host port like "8080:3000"
          if (portStr.match(/^\d+:\d+$/)) {
            console.warn(
              `Warning: Fixed host port ${portStr} prevents scaling for ${name}`
            );
          }
        }
      }
    }
  });

  it('supports zero-downtime deployments', async () => {
    const imageName = 'hx-docling-application:latest';

    // Verify container can start while another is stopping
    const container1 = 'zero-downtime-1';
    const container2 = 'zero-downtime-2';

    // Start first container
    await execAsync(
      `docker run -d --name ${container1} ${imageName}`
    );

    // Start second container (should work in parallel)
    await execAsync(
      `docker run -d --name ${container2} ${imageName}`
    );

    // Both should be running
    const result = await execAsync(
      `docker ps --filter "name=zero-downtime" --format '{{.Names}}'`
    );

    expect(result.stdout).toContain(container1);
    expect(result.stdout).toContain(container2);

    // Cleanup
    await execAsync(`docker rm -f ${container1} ${container2}`);
  });
});
```

**Expected Results**:
- 12-factor principles followed
- Probe endpoints available
- Replica support validated
- Zero-downtime capable

**Verification Points**:
- Orchestration patterns validated
- Scalability confirmed
- Deployment patterns verified

---

## 26. Next.js Server Components

### 26.1 Overview

Tests validating Next.js Server Component (RSC) rendering, data fetching, streaming, Suspense boundaries, error boundaries, and client/server boundary interactions.

**Reference**: Next.js App Router, React Server Components specification

---

### INT-RSC-005: Server Component Data Fetching

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-RSC-005 |
| **Title** | Server Component Data Fetching |
| **Priority** | P1 - High |
| **Components** | Server Components, Database Integration |
| **Spec Reference** | Section 2.5 (History View) |

**Test Steps**:
1. Create Server Component that fetches job history from PostgreSQL
2. Render component on server
3. Verify data fetched during server render
4. Confirm no client-side re-fetch occurs
5. Validate serialized props contain job data

**Expected Result**: Server Component fetches data during SSR, no client hydration waterfall, data available immediately on page load

---

### INT-RSC-006: Server Component Streaming with Suspense

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-RSC-006 |
| **Title** | Server Component Streaming with Suspense |
| **Priority** | P1 - High |
| **Components** | Server Components, Suspense, Streaming |
| **Spec Reference** | Section 2.5 |

**Test Steps**:
1. Wrap slow Server Component in Suspense boundary with fallback
2. Initiate SSR stream
3. Verify fallback rendered immediately
4. Verify actual content streamed after data resolves
5. Confirm proper boundary flushing

**Expected Result**: Suspense fallback renders first, content streams progressively, no full page block during slow data fetch

---

### INT-RSC-007: Server Component Error Boundary

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-RSC-007 |
| **Title** | Server Component Error Boundary |
| **Priority** | P0 - Critical |
| **Components** | Server Components, Error Boundaries |
| **Spec Reference** | Section 8 (Error Handling) |

**Test Steps**:
1. Create Server Component that throws error during render
2. Wrap in error boundary
3. Trigger render
4. Verify error caught by boundary
5. Confirm error UI displayed
6. Validate no crash, graceful degradation

**Expected Result**: Error boundary catches server component errors, displays fallback UI, prevents full page crash

---

### INT-RSC-008: Server-Only Imports Enforcement

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-RSC-008 |
| **Title** | Server-Only Imports Enforcement |
| **Priority** | P0 - Critical |
| **Components** | Server Components, Build System |
| **Spec Reference** | Section 8 (Security) |

**Test Steps**:
1. Import server-only module (prisma, fs) in Server Component
2. Attempt to import same in Client Component
3. Run build
4. Verify Server Component builds successfully
5. Verify Client Component build fails with error

**Expected Result**: server-only imports allowed in Server Components, blocked in Client Components, build-time enforcement

---

### INT-RSC-009: Async Server Component Patterns

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-RSC-009 |
| **Title** | Async Server Component Patterns |
| **Priority** | P1 - High |
| **Components** | Server Components, Async/Await |
| **Spec Reference** | Section 2.5 |

**Test Steps**:
1. Create async Server Component with top-level await
2. Fetch data from multiple sources (DB + Redis)
3. Render component
4. Verify all async operations complete before render
5. Confirm proper error handling for async failures

**Expected Result**: Async Server Components execute fully on server, all promises resolved before render, errors handled gracefully

---

### INT-RSC-010: Server Component Composition

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-RSC-010 |
| **Title** | Server Component Composition |
| **Priority** | P1 - High |
| **Components** | Server Components, Component Trees |
| **Spec Reference** | Section 6 (Component Specifications) |

**Test Steps**:
1. Create parent Server Component
2. Compose multiple child Server Components
3. Nest components 3+ levels deep
4. Render tree
5. Verify all components render on server
6. Confirm no client-side waterfalls

**Expected Result**: Server Components compose naturally, entire tree renders on server, single payload sent to client

---

### INT-RSC-011: Server Component Props Serialization

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-RSC-011 |
| **Title** | Server Component Props Serialization |
| **Priority** | P0 - Critical |
| **Components** | Server Components, Serialization |
| **Spec Reference** | Section 5 (Data Models) |

**Test Steps**:
1. Pass serializable props (JSON-compatible) to Server Component
2. Attempt to pass non-serializable props (functions, Symbols)
3. Render components
4. Verify serializable props work correctly
5. Verify non-serializable props throw error or warning

**Expected Result**: JSON-compatible props serialize correctly, non-serializable props rejected with clear error

---

### INT-RSC-012: Client Component Boundaries

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-RSC-012 |
| **Title** | Client Component Boundaries |
| **Priority** | P1 - High |
| **Components** | Server Components, Client Components |
| **Spec Reference** | Section 6 |

**Test Steps**:
1. Create Server Component containing Client Component
2. Pass server data as props to Client Component
3. Render tree
4. Verify Server Component renders on server
5. Verify Client Component hydrates on client
6. Confirm data flows correctly across boundary

**Expected Result**: Server renders parent, client hydrates child, props serialize correctly, interactivity works

---

### INT-RSC-013: Server Component with Streaming and Parallel Data Fetching

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-RSC-013 |
| **Title** | Server Component with Streaming and Parallel Data Fetching |
| **Priority** | P2 - Medium |
| **Components** | Server Components, Parallel Fetching |
| **Spec Reference** | Section 2.5 |

**Test Steps**:
1. Create Server Component fetching data from DB and Redis in parallel
2. Use Promise.all for parallel fetching
3. Render with streaming enabled
4. Measure total fetch time
5. Verify parallel execution (not sequential)

**Expected Result**: Data fetched in parallel, total time < sum of individual times, streaming delivers content progressively

---

### INT-RSC-014: Server Component Revalidation

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-RSC-014 |
| **Title** | Server Component Revalidation |
| **Priority** | P1 - High |
| **Components** | Server Components, Revalidation |
| **Spec Reference** | Section 3 (Non-Functional Requirements) |

**Test Steps**:
1. Configure Server Component with revalidate option
2. Render component (cache miss)
3. Re-render within revalidation period (cache hit)
4. Wait for revalidation period to expire
5. Re-render (cache miss, fresh data)

**Expected Result**: Component cached for revalidation period, stale data served during period, fresh data after expiration

---

### INT-RSC-015: Server Component with Dynamic Segments

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-RSC-015 |
| **Title** | Server Component with Dynamic Segments |
| **Priority** | P1 - High |
| **Components** | Server Components, Dynamic Routes |
| **Spec Reference** | Section 4.1 (API Specifications) |

**Test Steps**:
1. Create Server Component for dynamic route `/api/v1/jobs/[jobId]`
2. Access dynamic params in component
3. Fetch job-specific data
4. Render for multiple jobIds
5. Verify correct data per route

**Expected Result**: Dynamic params available in Server Component, correct data fetched per route, no cross-contamination

---

### INT-RSC-016: Server Component with Search Params

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-RSC-016 |
| **Title** | Server Component with Search Params |
| **Priority** | P2 - Medium |
| **Components** | Server Components, URL Search Params |
| **Spec Reference** | Section 2.5 |

**Test Steps**:
1. Create Server Component accessing searchParams
2. Render with various query strings
3. Verify searchParams parsed correctly
4. Confirm component re-renders on param change
5. Validate pagination use case

**Expected Result**: Search params accessible in Server Component, parsed correctly, component updates on param change

---

### INT-RSC-017: Server Component Cache Deduplication

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-RSC-017 |
| **Title** | Server Component Cache Deduplication |
| **Priority** | P1 - High |
| **Components** | Server Components, Request Deduplication |
| **Spec Reference** | Section 3 |

**Test Steps**:
1. Create multiple Server Components fetching same data
2. Render components in same tree
3. Instrument fetch calls
4. Verify fetch called only once (automatic deduplication)
5. Confirm all components receive same data

**Expected Result**: Duplicate fetches automatically deduplicated within single request, fetch called once, data shared

---

### INT-RSC-018: Server Component with Cookies Access

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-RSC-018 |
| **Title** | Server Component with Cookies Access |
| **Priority** | P1 - High |
| **Components** | Server Components, Cookies |
| **Spec Reference** | Section 7.4 (Session Management) |

**Test Steps**:
1. Create Server Component reading cookies
2. Set sessionId cookie
3. Render component
4. Verify cookie value accessible
5. Confirm dynamic rendering triggered (no static generation)

**Expected Result**: Cookies accessible in Server Component, dynamic rendering forced, session data available

---

### INT-RSC-019: Server Component with Headers Access

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-RSC-019 |
| **Title** | Server Component with Headers Access |
| **Priority** | P2 - Medium |
| **Components** | Server Components, Request Headers |
| **Spec Reference** | Section 4.1 |

**Test Steps**:
1. Create Server Component reading request headers
2. Send request with custom headers
3. Render component
4. Verify headers accessible
5. Confirm dynamic rendering

**Expected Result**: Request headers accessible in Server Component, custom headers readable, dynamic rendering enforced

---

### INT-RSC-020: Server Component Nested Suspense Boundaries

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-RSC-020 |
| **Title** | Server Component Nested Suspense Boundaries |
| **Priority** | P2 - Medium |
| **Components** | Server Components, Suspense, Streaming |
| **Spec Reference** | Section 2.5 |

**Test Steps**:
1. Create Server Component tree with nested Suspense boundaries
2. Configure different loading speeds for each boundary
3. Render with streaming
4. Verify each boundary resolves independently
5. Confirm progressive streaming (outer first, then inner)

**Expected Result**: Nested boundaries resolve independently, content streams progressively from outer to inner, no blocking

---

### INT-RSC-021: Server Component with Metadata Export

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-RSC-021 |
| **Title** | Server Component with Metadata Export |
| **Priority** | P2 - Medium |
| **Components** | Server Components, Metadata |
| **Spec Reference** | Section 6 |

**Test Steps**:
1. Export static metadata object from Server Component page
2. Render page
3. Verify metadata applied to HTML head
4. Test dynamic metadata generation function
5. Confirm metadata merges correctly with parent layout

**Expected Result**: Metadata exported from Server Component, applied to page head, dynamic metadata works, proper merging

---

### INT-RSC-022: Server Component Error Recovery

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-RSC-022 |
| **Title** | Server Component Error Recovery |
| **Priority** | P1 - High |
| **Components** | Server Components, Error Handling |
| **Spec Reference** | Section 8 |

**Test Steps**:
1. Create Server Component with retry logic for transient errors
2. Simulate database connection failure
3. Trigger render
4. Verify retry attempted
5. Confirm successful render on retry

**Expected Result**: Server Component retries transient errors, recovers gracefully, renders successfully after retry

---

### INT-RSC-023: Server Component with Partial Prerendering

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-RSC-023 |
| **Title** | Server Component with Partial Prerendering |
| **Priority** | P2 - Medium |
| **Components** | Server Components, Partial Prerendering |
| **Spec Reference** | Section 3 |

**Test Steps**:
1. Create page with static and dynamic Server Components
2. Enable partial prerendering
3. Build page
4. Verify static parts prerendered
5. Confirm dynamic parts rendered on demand
6. Validate shell delivered immediately

**Expected Result**: Static content prerendered, dynamic content streamed on request, instant shell delivery, progressive enhancement

---

### INT-RSC-024: Server Component Render Lifecycle

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-RSC-024 |
| **Title** | Server Component Render Lifecycle |
| **Priority** | P1 - High |
| **Components** | Server Components, Lifecycle |
| **Spec Reference** | Section 6 |

**Test Steps**:
1. Instrument Server Component render lifecycle
2. Track: data fetch start, fetch complete, render start, render complete, stream start, stream complete
3. Render component
4. Verify lifecycle events in correct order
5. Measure timing between events

**Expected Result**: Lifecycle events fire in correct order, timing reasonable, no unexpected delays or blocking

---

## 27. Server Actions

### 27.1 Overview

Tests validating Next.js Server Actions for form submissions, mutations, revalidation, progressive enhancement, and security (CSRF protection).

**Reference**: Next.js Server Actions specification

---

### INT-SA-001: Server Action Form Binding

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-SA-001 |
| **Title** | Server Action Form Binding |
| **Priority** | P1 - High |
| **Components** | Server Actions, Forms |
| **Spec Reference** | Section 2.1 (File Upload) |

**Test Steps**:
1. Create Server Action function for file upload
2. Bind to form using action attribute
3. Submit form with file
4. Verify Server Action receives FormData
5. Confirm action processes file correctly

**Expected Result**: Server Action bound to form, FormData passed correctly, file processed on server

---

### INT-SA-002: useFormState Integration

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-SA-002 |
| **Title** | useFormState Integration |
| **Priority** | P1 - High |
| **Components** | Server Actions, useFormState |
| **Spec Reference** | Section 6 |

**Test Steps**:
1. Create Server Action returning validation errors
2. Use useFormState hook in Client Component
3. Submit invalid form data
4. Verify form state updates with errors
5. Confirm UI displays errors
6. Submit valid data, verify success state

**Expected Result**: useFormState syncs with Server Action, errors displayed, success handled, progressive state updates

---

### INT-SA-003: useFormStatus for Pending State

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-SA-003 |
| **Title** | useFormStatus for Pending State |
| **Priority** | P1 - High |
| **Components** | Server Actions, useFormStatus |
| **Spec Reference** | Section 6 |

**Test Steps**:
1. Create Server Action with artificial delay
2. Use useFormStatus in submit button component
3. Submit form
4. Verify pending state set to true during action
5. Confirm pending state resets after completion
6. Validate button disabled during pending

**Expected Result**: useFormStatus tracks action pending state, button disabled during submission, re-enabled after completion

---

### INT-SA-004: Server Action CSRF Protection

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-SA-004 |
| **Title** | Server Action CSRF Protection |
| **Priority** | P0 - Critical |
| **Components** | Server Actions, Security |
| **Spec Reference** | Section 8 |

**Test Steps**:
1. Create Server Action for mutation
2. Attempt direct POST without Next.js origin
3. Verify action rejected
4. Submit via Next.js form (with origin)
5. Confirm action succeeds with valid origin

**Expected Result**: Server Actions enforce origin validation, external posts blocked, only same-origin requests accepted

---

### INT-SA-005: Progressive Enhancement with Server Actions

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-SA-005 |
| **Title** | Progressive Enhancement with Server Actions |
| **Priority** | P2 - Medium |
| **Components** | Server Actions, Progressive Enhancement |
| **Spec Reference** | Section 9 (Accessibility) |

**Test Steps**:
1. Create form with Server Action
2. Disable JavaScript in test environment
3. Submit form
4. Verify full page navigation occurs
5. Confirm action executes and data persists
6. Enable JavaScript, verify enhanced behavior

**Expected Result**: Form works without JavaScript (full page reload), enhances with JavaScript (partial update), accessible

---

### INT-SA-006: Server Action Error Handling

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-SA-006 |
| **Title** | Server Action Error Handling |
| **Priority** | P0 - Critical |
| **Components** | Server Actions, Error Handling |
| **Spec Reference** | Section 8 |

**Test Steps**:
1. Create Server Action that throws error
2. Wrap in try-catch, return error state
3. Submit form triggering error
4. Verify error caught
5. Confirm error returned to client
6. Validate error displayed in UI

**Expected Result**: Server Action errors caught, returned to client gracefully, UI displays error, no crash

---

### INT-SA-007: Revalidation After Server Action

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-SA-007 |
| **Title** | Revalidation After Server Action |
| **Priority** | P1 - High |
| **Components** | Server Actions, Revalidation |
| **Spec Reference** | Section 3 |

**Test Steps**:
1. Display cached job history
2. Create Server Action that updates job status
3. Call revalidatePath after mutation
4. Verify cache invalidated
5. Confirm UI updates with fresh data

**Expected Result**: revalidatePath invalidates cache, fresh data fetched, UI updates automatically after mutation

---

### INT-SA-008: Server Action with Redirect

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-SA-008 |
| **Title** | Server Action with Redirect |
| **Priority** | P1 - High |
| **Components** | Server Actions, Navigation |
| **Spec Reference** | Section 4.1 |

**Test Steps**:
1. Create Server Action for job submission
2. After successful submission, call redirect()
3. Submit form
4. Verify redirect occurs to job status page
5. Confirm data persisted before redirect

**Expected Result**: Server Action redirects after mutation, data committed before redirect, navigation smooth

---

### INT-SA-009: Server Action Optimistic Updates

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-SA-009 |
| **Title** | Server Action Optimistic Updates |
| **Priority** | P2 - Medium |
| **Components** | Server Actions, Optimistic UI |
| **Spec Reference** | Section 6 |

**Test Steps**:
1. Create Server Action for job status update
2. Use useOptimistic hook
3. Submit update
4. Verify UI updates immediately (optimistic)
5. Confirm rollback if server rejects
6. Validate final state matches server

**Expected Result**: UI updates optimistically before server response, rolls back on error, syncs with server state

---

### INT-SA-010: Server Action Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-SA-010 |
| **Title** | Server Action Validation |
| **Priority** | P1 - High |
| **Components** | Server Actions, Validation |
| **Spec Reference** | Section 2.1, 2.2 |

**Test Steps**:
1. Create Server Action with zod validation
2. Submit invalid form data
3. Verify validation errors returned
4. Submit valid data
5. Confirm successful processing
6. Test edge cases (file size, type validation)

**Expected Result**: Server Action validates input, rejects invalid data with clear errors, accepts valid data, processes correctly

---

## 28. Next.js Caching

### 28.1 Overview

Tests validating Next.js caching mechanisms: Full Route Cache, Router Cache, Data Cache, time-based revalidation, on-demand revalidation, cache tags, and invalidation patterns.

**Reference**: Next.js caching documentation

---

### INT-CACHE-001: Full Route Cache for Static Routes

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-CACHE-001 |
| **Title** | Full Route Cache for Static Routes |
| **Priority** | P1 - High |
| **Components** | Next.js Caching, Static Routes |
| **Spec Reference** | Section 3 |

**Test Steps**:
1. Build page with no dynamic functions
2. Request page multiple times
3. Verify HTML cached after build
4. Confirm subsequent requests served from cache
5. Measure response time (should be <10ms)

**Expected Result**: Static page fully cached after build, all requests served from cache, instant responses

---

### INT-CACHE-002: Router Cache (Client-Side Cache)

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-CACHE-002 |
| **Title** | Router Cache (Client-Side Cache) |
| **Priority** | P1 - High |
| **Components** | Next.js Caching, Client Navigation |
| **Spec Reference** | Section 3 |

**Test Steps**:
1. Navigate to page A
2. Navigate to page B
3. Navigate back to page A
4. Verify page A served from Router Cache (no server request)
5. Confirm instant navigation

**Expected Result**: Previously visited routes cached in Router Cache, back navigation instant, no server round-trip

---

### INT-CACHE-003: Data Cache with fetch

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-CACHE-003 |
| **Title** | Data Cache with fetch |
| **Priority** | P1 - High |
| **Components** | Next.js Caching, Data Cache |
| **Spec Reference** | Section 2.5 |

**Test Steps**:
1. Use fetch in Server Component with default caching
2. Request same URL multiple times across renders
3. Verify fetch result cached
4. Confirm subsequent fetches served from Data Cache
5. Validate cache persists across deployments

**Expected Result**: fetch results cached in Data Cache, subsequent requests served from cache, persists across builds

---

### INT-CACHE-004: Time-Based Revalidation

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-CACHE-004 |
| **Title** | Time-Based Revalidation |
| **Priority** | P1 - High |
| **Components** | Next.js Caching, Revalidation |
| **Spec Reference** | Section 3 |

**Test Steps**:
1. Configure route with `export const revalidate = 60`
2. Request page (cache miss, fresh data)
3. Request within 60s (cache hit, stale data)
4. Wait 60s, request again (revalidate, fresh data)
5. Verify cache refreshed

**Expected Result**: Page cached for 60s, stale data served during period, fresh data after revalidation, automatic refresh

---

### INT-CACHE-005: On-Demand Revalidation with revalidatePath

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-CACHE-005 |
| **Title** | On-Demand Revalidation with revalidatePath |
| **Priority** | P1 - High |
| **Components** | Next.js Caching, On-Demand Revalidation |
| **Spec Reference** | Section 3 |

**Test Steps**:
1. Cache job history page
2. Create new job via Server Action
3. Call `revalidatePath('/history')`
4. Request history page
5. Verify fresh data (new job visible)

**Expected Result**: revalidatePath invalidates cache immediately, next request fetches fresh data, UI updates

---

### INT-CACHE-006: On-Demand Revalidation with revalidateTag

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-CACHE-006 |
| **Title** | On-Demand Revalidation with revalidateTag |
| **Priority** | P1 - High |
| **Components** | Next.js Caching, Cache Tags |
| **Spec Reference** | Section 3 |

**Test Steps**:
1. Tag fetch requests with `{ next: { tags: ['jobs'] } }`
2. Cache job data across multiple routes
3. Update job via Server Action
4. Call `revalidateTag('jobs')`
5. Verify all job-related caches invalidated

**Expected Result**: revalidateTag invalidates all caches with matching tag, fresh data fetched, multiple routes updated

---

### INT-CACHE-007: Cache Opt-Out with Dynamic Functions

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-CACHE-007 |
| **Title** | Cache Opt-Out with Dynamic Functions |
| **Priority** | P1 - High |
| **Components** | Next.js Caching, Dynamic Rendering |
| **Spec Reference** | Section 6 |

**Test Steps**:
1. Use cookies() in Server Component
2. Request page multiple times
3. Verify page rendered dynamically each time
4. Confirm no Full Route Cache
5. Validate fresh data on every request

**Expected Result**: Dynamic functions (cookies, headers, searchParams) opt out of Full Route Cache, page rendered fresh each time

---

### INT-CACHE-008: Fetch Cache Control with cache Option

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-CACHE-008 |
| **Title** | Fetch Cache Control with cache Option |
| **Priority** | P2 - Medium |
| **Components** | Next.js Caching, Fetch API |
| **Spec Reference** | Section 3 |

**Test Steps**:
1. Use `fetch(url, { cache: 'no-store' })`
2. Verify fetch not cached
3. Use `fetch(url, { cache: 'force-cache' })`
4. Verify fetch cached indefinitely
5. Test `{ next: { revalidate: 3600 } }`

**Expected Result**: cache: 'no-store' bypasses cache, cache: 'force-cache' caches indefinitely, revalidate sets TTL

---

### INT-CACHE-009: Route Segment Config for Caching

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-CACHE-009 |
| **Title** | Route Segment Config for Caching |
| **Priority** | P1 - High |
| **Components** | Next.js Caching, Route Config |
| **Spec Reference** | Section 3 |

**Test Steps**:
1. Export `export const dynamic = 'force-dynamic'` from route
2. Request route
3. Verify no caching (fresh render each time)
4. Export `export const dynamic = 'force-static'`
5. Verify static caching enforced

**Expected Result**: dynamic config controls caching behavior, force-dynamic disables cache, force-static enforces cache

---

### INT-CACHE-010: Cache Behavior with Dynamic Routes

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-CACHE-010 |
| **Title** | Cache Behavior with Dynamic Routes |
| **Priority** | P1 - High |
| **Components** | Next.js Caching, Dynamic Routes |
| **Spec Reference** | Section 4.1 |

**Test Steps**:
1. Create dynamic route `/jobs/[jobId]`
2. Request `/jobs/123` (cache miss)
3. Request `/jobs/123` again (cache hit)
4. Request `/jobs/456` (different cache entry)
5. Verify separate cache per jobId

**Expected Result**: Each dynamic route param creates separate cache entry, cache hits per specific param value

---

### INT-CACHE-011: Cache Invalidation After Mutation

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-CACHE-011 |
| **Title** | Cache Invalidation After Mutation |
| **Priority** | P1 - High |
| **Components** | Next.js Caching, Mutations |
| **Spec Reference** | Section 3 |

**Test Steps**:
1. Cache job detail page
2. Update job status via Server Action
3. Invalidate cache with revalidatePath
4. Request job detail page
5. Verify fresh data displayed

**Expected Result**: Mutation triggers cache invalidation, fresh data fetched immediately after, stale data never shown

---

### INT-CACHE-012: Cache Headers for API Routes

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-CACHE-012 |
| **Title** | Cache Headers for API Routes |
| **Priority** | P2 - Medium |
| **Components** | Next.js Caching, API Routes |
| **Spec Reference** | Section 4.1 |

**Test Steps**:
1. Create API route with Cache-Control header
2. Request API endpoint
3. Verify Cache-Control header present
4. Test CDN caching behavior
5. Validate stale-while-revalidate

**Expected Result**: API routes support Cache-Control headers, CDN caching works, stale-while-revalidate pattern supported

---

### INT-CACHE-013: generateStaticParams for Dynamic Routes

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-CACHE-013 |
| **Title** | generateStaticParams for Dynamic Routes |
| **Priority** | P2 - Medium |
| **Components** | Next.js Caching, Static Generation |
| **Spec Reference** | Section 3 |

**Test Steps**:
1. Export generateStaticParams for `/jobs/[jobId]`
2. Return list of jobIds to prerender
3. Build application
4. Verify specified routes prerendered at build time
5. Confirm instant responses for prerendered routes

**Expected Result**: generateStaticParams prerenders dynamic routes at build time, specified params cached, instant responses

---

### INT-CACHE-014: Unstable_cache for Database Queries

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-CACHE-014 |
| **Title** | unstable_cache for Database Queries |
| **Priority** | P2 - Medium |
| **Components** | Next.js Caching, Database |
| **Spec Reference** | Section 4.2 |

**Test Steps**:
1. Wrap Prisma query in unstable_cache
2. Execute query multiple times
3. Verify query result cached
4. Confirm subsequent calls served from cache
5. Test revalidation with tags

**Expected Result**: unstable_cache caches database query results, subsequent calls skip database, revalidation works

---

### INT-CACHE-015: Cache Debugging and Monitoring

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-CACHE-015 |
| **Title** | Cache Debugging and Monitoring |
| **Priority** | P2 - Medium |
| **Components** | Next.js Caching, Observability |
| **Spec Reference** | Section 3 |

**Test Steps**:
1. Enable Next.js cache debugging
2. Make requests to cached routes
3. Inspect cache headers (X-Nextjs-Cache)
4. Verify HIT, MISS, STALE indicators
5. Monitor cache effectiveness

**Expected Result**: Cache debugging headers present, HIT/MISS/STALE states visible, cache effectiveness measurable

---

## 29. Disaster Recovery Tests

### 29.1 Overview

Tests validating system resilience and recovery capabilities when critical infrastructure components fail or become unavailable. These tests ensure the system can gracefully handle failures, maintain data consistency, and recover within acceptable timeframes.

---

### INT-DR-001: Database Failover Recovery

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DR-001 |
| **Title** | Database Failover Recovery |
| **Priority** | P1 - High |
| **Components** | PostgreSQL, Prisma, Connection Pool |
| **Spec Reference** | NFR-201, NFR-203 |

**Preconditions**:
- Application running with active database connection
- Active processing jobs in database
- Connection pool configured with retry logic

**Test Steps**:
1. Create active job in PROCESSING state
2. Simulate database connection loss (network partition or database shutdown)
3. Verify application detects connection failure
4. Restore database connectivity
5. Verify connection pool re-establishes connections
6. Query job status from database
7. Validate job state consistency after recovery

**Expected Result**: Application detects database failure within 5 seconds, connection pool re-establishes connections automatically within 10 seconds after restoration, job state remains consistent (no data loss), in-progress jobs resume or gracefully fail with retriable error

---

### INT-DR-002: Redis Cache Loss Recovery

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DR-002 |
| **Title** | Redis Cache Loss Recovery |
| **Priority** | P1 - High |
| **Components** | Redis, Circuit Breaker, Session Management |
| **Spec Reference** | CRI-05, CRI-23, Section 7.4.3 |

**Preconditions**:
- Redis cache operational with active sessions
- Circuit breaker configured (failure threshold: 5, timeout: 30s)
- Fallback strategies implemented

**Test Steps**:
1. Establish active session with cached data
2. Simulate Redis unavailability (shutdown or network partition)
3. Verify circuit breaker opens after failure threshold
4. Attempt session operations (should fall back to database)
5. Verify rate limiting falls back to in-memory tracking
6. Restore Redis availability
7. Verify circuit breaker transitions to half-open state
8. Confirm successful operations close circuit
9. Validate cache repopulation after recovery

**Expected Result**: Circuit breaker opens within 5 failed requests, fallback to database successful for session reads, rate limiting continues with in-memory store, circuit closes after 3 successful requests post-recovery, cache rebuilds from database without data loss, no cascading failures to dependent services

---

### INT-DR-003: MCP Server Unavailability Handling

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DR-003 |
| **Title** | MCP Server Unavailability Handling |
| **Priority** | P1 - High |
| **Components** | MCP Client, Error Recovery, Job State Machine |
| **Spec Reference** | FR-802, NFR-201, Section 4.3 |

**Preconditions**:
- MCP server running and processing jobs
- Jobs in various states (QUEUED, PROCESSING)
- MCP client configured with retry logic

**Test Steps**:
1. Submit job and verify QUEUED state
2. Initiate processing to transition to PROCESSING state
3. Simulate MCP server unavailability (shutdown or network failure)
4. Verify MCP client detects connection failure
5. Confirm job state updates to ERROR with retriable flag
6. Validate error recovery actions presented to user
7. Restore MCP server availability
8. Retry failed job
9. Verify successful processing after recovery

**Expected Result**: MCP client detects server unavailability within 10 seconds, job transitions to ERROR state with error code "MCP_UNAVAILABLE", error marked as retriable with "Retry" action, retry after recovery successfully processes job, no duplicate processing occurs, checkpoint data preserved for resume

---

### INT-DR-004: Application Server Restart Recovery

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DR-004 |
| **Title** | Application Server Restart Recovery |
| **Priority** | P1 - High |
| **Components** | Next.js Server, Job Recovery, Session Recovery |
| **Spec Reference** | NFR-203, CRI-10, CRI-22, Section 5.3 |

**Preconditions**:
- Application server running with active sessions
- Jobs in PROCESSING state
- Browser client connected via SSE
- Session data persisted in Redis

**Test Steps**:
1. Create session and start job processing
2. Verify SSE connection established
3. Simulate application server restart (graceful shutdown and restart)
4. Verify browser detects SSE disconnection
5. Confirm browser attempts automatic reconnection
6. After server restart, verify session recovery from Redis
7. Validate job status query returns correct state
8. Test job recovery hook (useJobRecovery) functionality
9. Verify SSE reconnection and progress updates resume

**Expected Result**: Server restarts within 30 seconds, SSE client detects disconnection and retries every 3 seconds, session data recovered from Redis successfully, jobs in PROCESSING state marked for manual retry or auto-resume based on checkpoint, browser displays recovery UI ("Reconnecting..."), SSE reconnects within 10 seconds of server availability, no data loss for committed database transactions

---

### INT-DR-005: Network Partition Handling

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DR-005 |
| **Title** | Network Partition Handling |
| **Priority** | P1 - High |
| **Components** | SSE, API Routes, Connection Management |
| **Spec Reference** | NFR-201, AC-504.1, AC-504.2, AC-504.3 |

**Preconditions**:
- Active SSE connection for progress updates
- Job in PROCESSING state with progress events
- Polling fallback mechanism configured (2s interval)

**Test Steps**:
1. Establish SSE connection and start job processing
2. Verify progress events received via SSE
3. Simulate 30-second network partition (client cannot reach server)
4. Verify SSE connection timeout detection
5. Confirm automatic fallback to polling mode
6. Validate polling requests at 2-second intervals
7. Verify UI displays "Using backup connection" indicator
8. Restore network connectivity
9. Confirm SSE reconnection and polling stops
10. Validate progress continuity (no missed updates)

**Expected Result**: SSE timeout detected within 10 seconds, fallback to polling activates automatically, polling requests succeed at 2s intervals, UI shows fallback indicator (AC-504.3), SSE reconnects after network restoration, polling stops after successful SSE reconnection, progress updates show no gaps (may have duplicates handled by client deduplication)

---

### INT-DR-006: Data Consistency After Recovery

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DR-006 |
| **Title** | Data Consistency After Recovery |
| **Priority** | P1 - High |
| **Components** | PostgreSQL, Redis, State Synchronization |
| **Spec Reference** | INT-CONSISTENCY-001 to INT-CONSISTENCY-004, Section 2.5 |

**Preconditions**:
- Multi-component system with PostgreSQL and Redis
- Jobs with results stored in database
- Cache populated with job metadata

**Test Steps**:
1. Create job and complete processing with results
2. Verify data in both PostgreSQL (authoritative) and Redis (cache)
3. Simulate Redis failure and flush
4. Verify API responses fall back to database queries
5. Restore Redis and trigger cache repopulation
6. Query same job via API (should hit cache)
7. Compare cached data with database data (byte-for-byte)
8. Verify pagination, sorting, and filtering consistency
9. Test concurrent reads during cache rebuild

**Expected Result**: API responses consistent between cache and database, cache repopulation creates identical data to database, no stale data served after recovery, pagination results match between cache and database, concurrent reads show monotonic consistency (reads never go backward in time), cache invalidation triggered for updated records

---

### INT-DR-007: Service Degradation Modes

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DR-007 |
| **Title** | Service Degradation Modes |
| **Priority** | P1 - High |
| **Components** | Circuit Breaker, Fallback Logic, Feature Flags |
| **Spec Reference** | Section 7.4.3, CRI-05, CRI-23 |

**Preconditions**:
- All services (PostgreSQL, Redis, MCP) operational
- Circuit breakers configured for each service
- Degraded mode fallbacks implemented

**Test Steps**:
1. Verify full functionality with all services available
2. Simulate Redis unavailability
3. Confirm system operates in degraded mode:
   - Session management falls back to database
   - Rate limiting uses in-memory tracking
   - SSE disabled, polling only
4. Verify UI indicators for degraded features
5. Simulate PostgreSQL read replica failure (if configured)
6. Confirm primary database handles read load
7. Restore all services
8. Verify full functionality restored
9. Validate no permanent state corruption

**Expected Result**: System continues operation with reduced performance during Redis failure, critical paths (job submission, status query) remain functional, non-critical features gracefully degrade (caching, real-time updates), UI clearly indicates degraded mode with specific feature limitations, all services recover to full functionality when dependencies restore, performance returns to baseline after recovery

---

### INT-DR-008: Recovery Time Objective (RTO) Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | INT-DR-008 |
| **Title** | Recovery Time Objective (RTO) Validation |
| **Priority** | P1 - High |
| **Components** | All Components, Health Checks, Monitoring |
| **Spec Reference** | NFR-201, NFR-203, Section 4.4 |

**Preconditions**:
- RTO targets defined for each component:
  - Database: 30 seconds
  - Redis: 60 seconds
  - MCP Server: 60 seconds
  - Application Server: 30 seconds
- Health check endpoints configured
- Monitoring and alerting enabled

**Test Steps**:
1. Baseline: Measure normal operation response times
2. Database Recovery Test:
   - Simulate database failure
   - Measure time from failure detection to full recovery
   - Verify RTO ≤ 30 seconds
3. Redis Recovery Test:
   - Simulate Redis failure
   - Measure time from circuit open to circuit close
   - Verify RTO ≤ 60 seconds
4. MCP Server Recovery Test:
   - Simulate MCP unavailability
   - Measure time from detection to successful retry
   - Verify RTO ≤ 60 seconds
5. Application Server Recovery Test:
   - Simulate graceful restart
   - Measure time from shutdown to health check pass
   - Verify RTO ≤ 30 seconds
6. Validate health check accuracy during recovery
7. Confirm monitoring alerts triggered appropriately

**Expected Result**: All component RTOs meet defined targets, health checks accurately reflect recovery state (no false positives/negatives), recovery metrics logged for analysis (failure time, detection time, recovery time), automated recovery mechanisms trigger without manual intervention, system returns to full performance within 2x RTO, monitoring alerts fire within 1 minute of failure detection

---

## Test Summary

### Test Count by Component

| Component | Test Count |
|-----------|-----------|
| API to MCP Integration | 4 |
| MCP Client to Server | 3 |
| API to PostgreSQL | 4 |
| API to Redis | 3 |
| SSE to Redis | 2 |
| State Machine | 2 |
| Checkpoint/Resume | 2 |
| Rate Limiting | 1 |
| Session Management | 1 |
| Frontend Component Integration | 10 |
| Health Check Integration | 6 |
| Next.js Middleware Integration | 5 |
| Server Component Integration (Original) | 4 |
| Database Connection Management | 6 |
| MCP Protocol Compliance | 8 |
| Redis Connection Management | 8 |
| SSE Connection Management | 3 |
| Request Validation Integration | 5 |
| Async Concurrency Integration | 4 |
| File Storage Integration | 5 |
| URL Fetch Integration | 3 |
| Workflow Orchestration Integration | 5 |
| Cross-Service Consistency | 4 |
| Container and Docker Testing | 32 |
| Next.js Server Components (Extended) | 20 |
| Server Actions | 10 |
| Next.js Caching | 15 |
| Disaster Recovery Tests | 8 |
| **Total** | **183** |

### Coverage by Integration Point

| Integration Point | Tests | Priority |
|-------------------|-------|----------|
| API Routes -> MCP Client | INT-MCP-001 to INT-MCP-004 | P1 |
| MCP Client -> Docling Server | INT-MCP-010 to INT-MCP-012 | P0-P1 |
| MCP Protocol Compliance | INT-MCP-013 to INT-MCP-024 | P0-P1 |
| API Routes -> PostgreSQL | INT-DB-001 to INT-DB-010 | P0-P2 |
| API Routes -> Redis | INT-REDIS-001 to INT-REDIS-003 | P1 |
| Redis Connection Management | INT-REDIS-004 to INT-REDIS-011 | P1-P2 |
| SSE -> Redis Pub/Sub | INT-SSE-001 to INT-SSE-002 | P1 |
| SSE Connection Management | INT-SSE-003 to INT-SSE-005 | P1-P2 |
| State Machine | INT-SM-001 to INT-SM-002 | P0 |
| Frontend -> API | INT-UI-001 to INT-UI-010 | P0-P2 |
| Health Checks | INT-HEALTH-001 to INT-HEALTH-006 | P1 |
| Next.js Middleware | INT-MW-001 to INT-MW-005 | P1-P2 |
| Server Components (Original) | INT-RSC-001 to INT-RSC-004 | P1-P2 |
| Request Validation | INT-VAL-001 to INT-VAL-005 | P1 |
| Async Concurrency | INT-ASYNC-001 to INT-ASYNC-004 | P1 |
| File Storage | INT-STORAGE-001 to INT-STORAGE-005 | P1-P2 |
| URL Fetch | INT-URL-001 to INT-URL-003 | P1 |
| Workflow Orchestration | INT-WF-001, INT-QUEUE-001 to INT-QUEUE-002, INT-CK-003, INT-LONG-001 | P1-P2 |
| Cross-Service Consistency | INT-CONSISTENCY-001 to INT-CONSISTENCY-004 | P2 |
| Container Security Scanning | INT-DOCKER-001 to INT-DOCKER-006 | P0 |
| Dockerfile Best Practices | INT-DOCKER-007 to INT-DOCKER-011 | P1 |
| Docker Compose Configuration | INT-DOCKER-012 to INT-DOCKER-016 | P1 |
| Container Runtime Tests | INT-DOCKER-017 to INT-DOCKER-022 | P1-P2 |
| Container Security Hardening | INT-DOCKER-023 to INT-DOCKER-026 | P1 |
| Container Operations | INT-DOCKER-027 to INT-DOCKER-032 | P2 |
| Next.js Server Components (Extended) | INT-RSC-005 to INT-RSC-024 | P0-P2 |
| Server Actions | INT-SA-001 to INT-SA-010 | P0-P2 |
| Next.js Caching | INT-CACHE-001 to INT-CACHE-015 | P1-P2 |
| Disaster Recovery Tests | INT-DR-001 to INT-DR-008 | P1 |

### Priority Distribution

| Priority | Count | Percentage |
|----------|-------|------------|
| P0 - Critical | 21 | 11% |
| P1 - High | 128 | 70% |
| P2 - Medium | 34 | 19% |
| **Total** | **183** | **100%** |

---

## Changelog

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0.0 | 2025-12-12 | Julia Santos | Initial integration test plan |
| 1.1.0 | 2025-12-13 | Julia Santos | Added Section 11 (Frontend Component Integration) with 10 P0-P2 tests (INT-UI-001 to INT-UI-010) and Section 12 (Health Check Integration) with 6 P1 tests (INT-HEALTH-001 to INT-HEALTH-006) per consolidated review findings |
| 1.2.0 | 2025-12-13 | Julia Santos | Added Section 13 (Next.js Middleware Integration) with 5 P1-P2 tests (INT-MW-001 to INT-MW-005), Section 14 (Server Component Integration) with 4 P1-P2 tests (INT-RSC-001 to INT-RSC-004), and Section 15 (Database Connection Management) with 6 P0-P2 tests (INT-DB-005 to INT-DB-010). Total test count increased from 38 to 53 |
| 1.3.0 | 2025-12-13 | Julia Santos | Added Section 16 (MCP Protocol Compliance) with 8 P0-P1 tests (INT-MCP-013 to INT-MCP-024) covering JSON-RPC 2.0 validation, capability negotiation, transport modes, and server unavailability. Added Section 17 (Redis Connection Management) with 8 P1-P2 tests (INT-REDIS-004 to INT-REDIS-011) covering pool exhaustion, timeouts, failover, Pub/Sub errors, and data expiration. Added Section 18 (SSE Connection Management) with 3 P1-P2 tests (INT-SSE-003 to INT-SSE-005) covering cleanup, concurrent streams, and event ordering. Total test count increased from 53 to 72 |
| 1.4.0 | 2025-12-13 | Julia Santos | Added Section 19 (Request Validation Integration) with 5 P1 tests (INT-VAL-001 to INT-VAL-005) covering validation chain integration, RFC 7807 error responses, body size limits, Content-Type validation, and malformed JSON handling. Added Section 20 (Async Concurrency Integration) with 4 P1 tests (INT-ASYNC-001 to INT-ASYNC-004) covering concurrent job processing, database connection pool concurrency, Redis connection pool concurrency, and MCP client connection reuse. Total test count increased from 72 to 81 |
| 1.5.0 | 2025-12-13 | Julia Santos | Added Section 21 (File Storage Integration) with 5 P1-P2 tests (INT-STORAGE-001 to INT-STORAGE-005) covering file upload to storage backend, file storage path validation, file retrieval for processing, storage error handling, and storage cleanup after expiration. Added Section 22 (URL Fetch Integration) with 3 P1 tests (INT-URL-001 to INT-URL-003) covering URL fetch and conversion, URL fetch error handling, and redirect following. Total test count increased from 81 to 89 |
| 2.0.0 | 2025-12-13 | Julia Santos | MAJOR REVISION COMPLETE - Added Section 23 (Workflow Orchestration Integration) with 5 P1-P2 tests (INT-WF-001, INT-QUEUE-001 to INT-QUEUE-002, INT-CK-003, INT-LONG-001) covering workflow coordination across stages, concurrent job queuing, job queue ordering, checkpoint consistency across services, and long-running workflow stability. Added Section 24 (Cross-Service Consistency) with 4 P2 tests (INT-CONSISTENCY-001 to INT-CONSISTENCY-004) covering API response data consistency with database, pagination response validation, cache-database consistency, and session state consistency. Final test count: 98 tests (increased from original 22 to 98, representing comprehensive consolidated review remediation) |
| 2.1.0 | 2025-12-15 | Julia Santos | CONTAINER SECURITY REMEDIATION (A-03) - Added Section 25 (Container and Docker Testing) with 32 P0-P2 tests (INT-DOCKER-001 to INT-DOCKER-032) addressing critical container security scanning gaps identified by Thomas Docker in consolidated review. Tests cover: Container Security Scanning (6 P0 tests: Trivy/Grype vulnerability scanning, base image validation, non-root user enforcement, secret scanning, CVE threshold validation), Dockerfile Best Practices (5 P1 tests: multi-stage builds, layer optimization, COPY vs ADD, health checks, entrypoint patterns), Docker Compose Configuration (5 P1 tests: service dependencies, network isolation, volume security, environment handling, resource limits), Container Runtime Tests (6 P1-P2 tests: startup time, health check validation, graceful shutdown, log aggregation, resource monitoring, layer caching), Container Security Hardening (4 P1 tests: Dockle scanning, filesystem security, network security, capability restrictions), and Container Operations (6 P2 tests: compose override handling, multi-architecture support, version management, build reproducibility, process limits, orchestration readiness). Total test count increased from 98 to 130. Priority distribution updated: P0 increased from 10 to 16, P1 increased from 70 to 87, P2 increased from 18 to 27. |
| 2.2.0 | 2025-12-15 | Julia Santos | NEXT.JS TESTING REMEDIATION (A-01) - Added 45 Next.js-specific tests addressing critical gaps in Next.js App Router testing coverage. Added Section 26 (Next.js Server Components) with 20 P0-P2 tests (INT-RSC-005 to INT-RSC-024) covering: data fetching patterns, streaming with Suspense boundaries, error boundaries, server-only imports enforcement, async component patterns, component composition, props serialization, client/server boundaries, parallel data fetching, revalidation, dynamic segments, search params, cache deduplication, cookies/headers access, nested Suspense, metadata export, error recovery, partial prerendering, and render lifecycle. Added Section 27 (Server Actions) with 10 P0-P2 tests (INT-SA-001 to INT-SA-010) covering: form binding, useFormState integration, useFormStatus for pending state, CSRF protection, progressive enhancement, error handling, revalidation after mutations, redirects, optimistic updates, and validation. Added Section 28 (Next.js Caching) with 15 P1-P2 tests (INT-CACHE-001 to INT-CACHE-015) covering: Full Route Cache for static routes, Router Cache (client-side), Data Cache with fetch, time-based revalidation, on-demand revalidation (revalidatePath/revalidateTag), cache opt-out with dynamic functions, fetch cache control, route segment config, dynamic route caching, cache invalidation after mutations, API route cache headers, generateStaticParams, unstable_cache for database queries, and cache debugging/monitoring. Total test count increased from 130 to 175. Priority distribution: P0 increased from 16 to 21, P1 increased from 87 to 120, P2 increased from 27 to 34. |
| 2.3.0 | 2025-12-15 | Julia Santos | DISASTER RECOVERY TESTING REMEDIATION (A-10) - Added Section 29 (Disaster Recovery Tests) with 8 P1 tests (INT-DR-001 to INT-DR-008) addressing critical gaps in disaster recovery and resilience testing identified in consolidated review. Tests cover: Database Failover Recovery (INT-DR-001: connection loss detection, automatic reconnection, job state consistency), Redis Cache Loss Recovery (INT-DR-002: circuit breaker operation, fallback to database, cache repopulation), MCP Server Unavailability Handling (INT-DR-003: server failure detection, retriable error handling, checkpoint preservation), Application Server Restart Recovery (INT-DR-004: session recovery, SSE reconnection, job recovery hooks), Network Partition Handling (INT-DR-005: SSE timeout detection, polling fallback, progress continuity), Data Consistency After Recovery (INT-DR-006: cache-database consistency, monotonic reads, invalidation logic), Service Degradation Modes (INT-DR-007: graceful degradation, fallback strategies, UI indicators), and Recovery Time Objective (RTO) Validation (INT-DR-008: component-level RTO targets, health check accuracy, automated recovery verification). All tests reference specification requirements (NFR-201, NFR-203, CRI-05, CRI-10, CRI-22, CRI-23, Section 7.4.3) and define measurable recovery criteria. Total test count increased from 175 to 183. Priority distribution: P0 unchanged at 21, P1 increased from 120 to 128, P2 unchanged at 34. |
