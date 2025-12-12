# Julia Santos - Testing & QA Tasks: Sprint 1.5a (MCP Client Core)

**Sprint**: 1.5a - MCP Client (Core)
**Role**: Testing & Quality Assurance Specialist
**Document Version**: 1.0.0
**Created**: 2025-12-12
**References**:
- Implementation Plan v1.2.0 Section 4.5a
- Detailed Specification v1.2.0 Section 7.1

---

## Sprint 1.5a Testing Objectives

Implement unit tests for the MCP client core functionality with focus on protocol compliance testing. Ensure the critical 3-step MCP initialization sequence is thoroughly tested, along with error code mapping and tool invocation.

---

## Tasks

### JUL-1.5a-001: Write Unit Tests for MCP 3-Step Initialization

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.5a-001 |
| **Title** | Write Unit Tests for MCP Protocol Initialization Sequence |
| **Priority** | P0 (Critical - Protocol Compliance) |
| **Effort** | 1.5 hours |
| **Dependencies** | JAMES-1.5a-003 (MCP initialization implemented) |

#### Description

Write comprehensive unit tests for the MCP 3-step initialization sequence ensuring protocol compliance. This is a critical test as the MCP server rejects tool calls without proper initialization.

#### Acceptance Criteria

- [ ] Test Step 1: `initialize` request sent with correct clientInfo
- [ ] Test Step 2: `initialize` response parsed correctly
- [ ] Test Step 3: `notifications/initialized` notification sent AFTER response
- [ ] Test server capabilities extracted correctly
- [ ] Test initialization fails if server rejects
- [ ] Test tools/list called ONLY after notifications/initialized
- [ ] Test capability validation (required tools present)
- [ ] Test initialization timeout handling
- [ ] Test initialization retry on failure
- [ ] Coverage 100% for MCP initialization sequence

#### Technical Notes

```typescript
// src/lib/mcp/__tests__/initialize.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { MCPClient } from '../client';
import { setupServer } from 'msw/node';
import { http, HttpResponse } from 'msw';

describe('MCP Initialization', () => {
  const requestLog: string[] = [];

  const server = setupServer(
    http.post('http://hx-docling-server:8000/rpc', async ({ request }) => {
      const body = await request.json();
      requestLog.push(body.method);

      if (body.method === 'initialize') {
        return HttpResponse.json({
          jsonrpc: '2.0',
          id: body.id,
          result: {
            protocolVersion: '2024-11-05',
            serverInfo: { name: 'hx-docling-mcp-server', version: '1.0.0' },
            capabilities: { tools: {} },
          },
        });
      }

      if (body.method === 'notifications/initialized') {
        return HttpResponse.json({ jsonrpc: '2.0', id: body.id, result: {} });
      }

      if (body.method === 'tools/list') {
        return HttpResponse.json({
          jsonrpc: '2.0',
          id: body.id,
          result: {
            tools: [
              { name: 'convert_pdf', description: 'Convert PDF', inputSchema: {} },
              { name: 'export_markdown', description: 'Export MD', inputSchema: {} },
            ],
          },
        });
      }

      return HttpResponse.json({ jsonrpc: '2.0', id: body.id, result: {} });
    })
  );

  beforeEach(() => {
    server.listen();
    requestLog.length = 0;
  });

  afterEach(() => server.close());

  describe('3-Step Initialization Sequence', () => {
    it('executes initialize -> response -> notifications/initialized in order', async () => {
      const client = new MCPClient('http://hx-docling-server:8000');
      await client.initialize();

      expect(requestLog[0]).toBe('initialize');
      expect(requestLog[1]).toBe('notifications/initialized');
    });

    it('sends correct clientInfo in initialize request', async () => {
      let capturedBody: any;
      server.use(
        http.post('http://hx-docling-server:8000/rpc', async ({ request }) => {
          capturedBody = await request.json();
          return HttpResponse.json({
            jsonrpc: '2.0',
            id: capturedBody.id,
            result: { protocolVersion: '2024-11-05', serverInfo: {}, capabilities: {} },
          });
        })
      );

      const client = new MCPClient('http://hx-docling-server:8000');
      await client.initialize();

      expect(capturedBody.method).toBe('initialize');
      expect(capturedBody.params.clientInfo).toHaveProperty('name');
      expect(capturedBody.params.clientInfo).toHaveProperty('version');
    });

    it('extracts server capabilities from response', async () => {
      const client = new MCPClient('http://hx-docling-server:8000');
      await client.initialize();

      expect(client.serverCapabilities).toBeDefined();
      expect(client.serverCapabilities.tools).toBeDefined();
    });

    it('sends notifications/initialized ONLY after successful initialize response', async () => {
      const initOrder: string[] = [];

      server.use(
        http.post('http://hx-docling-server:8000/rpc', async ({ request }) => {
          const body = await request.json();
          initOrder.push(body.method);

          if (body.method === 'initialize') {
            // Delay response to verify notification waits
            await new Promise((r) => setTimeout(r, 100));
            return HttpResponse.json({
              jsonrpc: '2.0',
              id: body.id,
              result: { protocolVersion: '2024-11-05', serverInfo: {}, capabilities: {} },
            });
          }
          return HttpResponse.json({ jsonrpc: '2.0', id: body.id, result: {} });
        })
      );

      const client = new MCPClient('http://hx-docling-server:8000');
      await client.initialize();

      expect(initOrder.indexOf('initialize')).toBeLessThan(
        initOrder.indexOf('notifications/initialized')
      );
    });
  });

  describe('Tools List After Initialization', () => {
    it('throws error if tools/list called before initialization', async () => {
      const client = new MCPClient('http://hx-docling-server:8000');

      await expect(client.getTools()).rejects.toThrow(/not initialized/i);
    });

    it('calls tools/list only after notifications/initialized', async () => {
      const client = new MCPClient('http://hx-docling-server:8000');
      await client.initialize();

      const tools = await client.getTools();

      expect(requestLog).toContain('tools/list');
      expect(requestLog.indexOf('notifications/initialized')).toBeLessThan(
        requestLog.indexOf('tools/list')
      );
    });
  });

  describe('Initialization Failure Handling', () => {
    it('throws error on server rejection', async () => {
      server.use(
        http.post('http://hx-docling-server:8000/rpc', () => {
          return HttpResponse.json({
            jsonrpc: '2.0',
            error: { code: -32600, message: 'Invalid Request' },
          });
        })
      );

      const client = new MCPClient('http://hx-docling-server:8000');
      await expect(client.initialize()).rejects.toThrow();
    });

    it('handles initialization timeout', async () => {
      server.use(
        http.post('http://hx-docling-server:8000/rpc', async () => {
          await new Promise((r) => setTimeout(r, 10000)); // Long delay
          return HttpResponse.json({});
        })
      );

      const client = new MCPClient('http://hx-docling-server:8000', { timeout: 1000 });
      await expect(client.initialize()).rejects.toThrow(/timeout/i);
    });
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| MCP initialization tests | `src/lib/mcp/__tests__/initialize.test.ts` |

---

### JUL-1.5a-002: Write Unit Tests for JSON-RPC Error Mapping

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.5a-002 |
| **Title** | Write Unit Tests for JSON-RPC Error Code Mapping |
| **Priority** | P0 (Critical) |
| **Effort** | 1.0 hour |
| **Dependencies** | JAMES-1.5a-008 (Error mapping implemented) |

#### Description

Write unit tests for mapping JSON-RPC error codes (-32700 to -32099) to user-friendly E2xx error codes as defined in the specification.

#### Acceptance Criteria

- [ ] Test -32700 (Parse error) maps to E200
- [ ] Test -32600 (Invalid Request) maps to E201
- [ ] Test -32601 (Method not found) maps to E202
- [ ] Test -32602 (Invalid params) maps to E203
- [ ] Test -32603 (Internal error) maps to E204
- [ ] Test -32000 to -32099 (Server error) maps appropriately
- [ ] Test unknown error codes handled gracefully
- [ ] Test error message extraction
- [ ] Test error details preserved
- [ ] Coverage 100% for error mapping

#### Technical Notes

```typescript
// src/lib/mcp/__tests__/error-mapping.test.ts
import { describe, it, expect } from 'vitest';
import { mapJsonRpcError, MCPErrorCode } from '../error-mapping';

/**
 * Error Code Mapping Reference (from specification):
 *
 * JSON-RPC Standard Errors:
 *   -32700 → E200 (Parse error)
 *   -32600 → E201 (Invalid Request)
 *   -32601 → E202 (Method not found)
 *   -32602 → E203 (Invalid params)
 *   -32603 → E204 (Internal error)
 *
 * Server Errors (-32000 to -32099):
 *   -32000 → E201 (Processing service unavailable)
 *   -32001 → E202 (Request timeout)
 *   -32002 → E302 (Document conversion failed)
 *   -32003 → E303 (Export generation failed)
 *   -32004 to -32099 → E299 (fallback for unmapped server errors)
 *
 * Unknown/Out-of-range errors → E299 (Generic MCP error)
 *
 * NOTE: Canonical mapping per Specification Section 7.1.4 and JSON-RPC 2.0 spec.
 * Server errors split between MCP errors (E2xx) and Processing errors (E3xx) per DEF-011.
 */

describe('JSON-RPC Error Mapping', () => {
  // Standard JSON-RPC error codes with explicit mappings
  const STANDARD_ERROR_MAPPINGS = [
    { jsonRpc: -32700, expected: 'E200', name: 'Parse error' },
    { jsonRpc: -32600, expected: 'E201', name: 'Invalid Request' },
    { jsonRpc: -32601, expected: 'E202', name: 'Method not found' },
    { jsonRpc: -32602, expected: 'E203', name: 'Invalid params' },
    { jsonRpc: -32603, expected: 'E204', name: 'Internal error' },
  ];

  // Server error codes (-32000 to -32099) with explicit mappings
  // Canonical mapping per Specification Section 7.1.4 (DEF-011)
  const SERVER_ERROR_MAPPINGS = [
    { jsonRpc: -32000, expected: 'E201', name: 'Processing service unavailable' },
    { jsonRpc: -32001, expected: 'E202', name: 'Request timeout' },
    { jsonRpc: -32002, expected: 'E302', name: 'Document conversion failed' },
    { jsonRpc: -32003, expected: 'E303', name: 'Export generation failed' },
  ];

  describe('Standard JSON-RPC error mapping', () => {
    STANDARD_ERROR_MAPPINGS.forEach(({ jsonRpc, expected, name }) => {
      it(`maps JSON-RPC ${jsonRpc} (${name}) to ${expected}`, () => {
        const result = mapJsonRpcError({
          code: jsonRpc,
          message: name,
        });

        expect(result.code).toBe(expected);
      });
    });
  });

  describe('Server error mapping (-32000 to -32099)', () => {
    // Test explicitly mapped server errors
    SERVER_ERROR_MAPPINGS.forEach(({ jsonRpc, expected, name }) => {
      it(`maps JSON-RPC ${jsonRpc} (${name}) to ${expected}`, () => {
        const result = mapJsonRpcError({
          code: jsonRpc,
          message: name,
        });

        expect(result.code).toBe(expected);
      });
    });

    // Boundary and representative sample tests for server error range
    describe('Server error range coverage (-32000 to -32099)', () => {
      const SERVER_ERROR_SAMPLES = [
        { code: -32000, desc: 'lower boundary' },
        { code: -32050, desc: 'midpoint representative' },
        { code: -32099, desc: 'upper boundary' },
      ];

      SERVER_ERROR_SAMPLES.forEach(({ code, desc }) => {
        it(`handles server error ${code} (${desc}) within valid range`, () => {
          const result = mapJsonRpcError({
            code,
            message: `Server error at ${code}`,
          });

          // Server errors map to MCP errors (E2xx) or Processing errors (E3xx) or fallback (E299)
          // Per Specification Section 7.1.4: -32000/-32001 → E20x, -32002/-32003 → E30x
          expect(result.code).toMatch(/^E(20[12]|30[23]|299)$/);
        });
      });

      it('maps all codes in -32000 to -32099 range to appropriate error codes per spec', () => {
        // Test representative samples across the range
        const testCodes = [-32000, -32010, -32025, -32050, -32075, -32099];

        testCodes.forEach((code) => {
          const result = mapJsonRpcError({
            code,
            message: `Server error ${code}`,
          });

          // Per Specification Section 7.1.4:
          // - -32000/-32001 map to MCP errors (E201/E202)
          // - -32002/-32003 map to Processing errors (E302/E303)
          // - Other codes in range fall back to E299
          const isMcpError = /^E20[12]$/.test(result.code);
          const isProcessingError = /^E30[23]$/.test(result.code);
          const isGenericFallback = result.code === 'E299';

          expect(
            isMcpError || isProcessingError || isGenericFallback,
            `Expected ${code} to map to E201/E202, E302/E303, or E299, got ${result.code}`
          ).toBe(true);
        });
      });
    });
  });

  describe('Error message preservation', () => {
    it('preserves original error message', () => {
      const result = mapJsonRpcError({
        code: -32600,
        message: 'Custom error message',
      });

      expect(result.originalMessage).toBe('Custom error message');
    });

    it('preserves error data if present', () => {
      const result = mapJsonRpcError({
        code: -32602,
        message: 'Invalid params',
        data: { param: 'file_path', reason: 'required' },
      });

      expect(result.details).toEqual({ param: 'file_path', reason: 'required' });
    });
  });

  describe('Unknown error handling', () => {
    it('maps unknown error codes to E299 (Generic MCP error)', () => {
      const result = mapJsonRpcError({
        code: -99999,
        message: 'Unknown error',
      });

      expect(result.code).toBe('E299');
    });

    it('maps error codes outside server range to E299', () => {
      // Codes outside -32000 to -32099 and not standard codes
      const outOfRangeCodes = [-31999, -32100, -40000, 0, 500];

      outOfRangeCodes.forEach((code) => {
        const result = mapJsonRpcError({
          code,
          message: `Out of range error ${code}`,
        });

        expect(result.code).toBe('E299');
      });
    });

    it('preserves error message when mapping to E299', () => {
      const result = mapJsonRpcError({
        code: -99999,
        message: 'Unexpected error occurred',
      });

      expect(result.code).toBe('E299');
      expect(result.message).toContain('Unexpected error occurred');
    });

    it('E299 represents the default unknown error mapping', () => {
      // Verify E299 is used as the catch-all for unmapped codes
      // This is the documented fallback per the error mapping specification
      const result = mapJsonRpcError({
        code: -12345, // Arbitrary unmapped code
        message: 'Some unmapped error',
      });

      expect(result.code).toBe('E299');
      expect(MCPErrorCode.GENERIC_MCP_ERROR).toBe('E299');
    });
  });

  describe('User-friendly messages', () => {
    it('provides user-friendly message for E200', () => {
      const result = mapJsonRpcError({
        code: -32700,
        message: 'Parse error',
      });

      expect(result.userMessage).toBe(
        'The server could not parse the request. Please try again.'
      );
    });

    it('provides actionable message for E202', () => {
      const result = mapJsonRpcError({
        code: -32601,
        message: 'Method not found',
      });

      expect(result.userMessage).toContain('feature is not available');
    });
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| Error mapping tests | `src/lib/mcp/__tests__/error-mapping.test.ts` |

---

### JUL-1.5a-003: Write Unit Tests for MCP Tool Invocation

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.5a-003 |
| **Title** | Write Unit Tests for MCP Tool Invocation |
| **Priority** | P1 (High) |
| **Effort** | 1.25 hours |
| **Dependencies** | JAMES-1.5a-005 (Tool definitions implemented) |

#### Description

Write unit tests for invoking MCP tools (convert_pdf, export_markdown, etc.) including parameter validation, timeout handling, and response parsing.

#### Acceptance Criteria

- [ ] Test convert_pdf tool invocation
- [ ] Test convert_docx tool invocation
- [ ] Test convert_xlsx tool invocation
- [ ] Test convert_pptx tool invocation
- [ ] Test convert_url tool invocation
- [ ] Test export_markdown tool invocation
- [ ] Test export_html tool invocation
- [ ] Test export_json tool invocation
- [ ] Test parameter validation against tool schema
- [ ] Test size-based timeout (60s/180s/300s)
- [ ] Test tool response parsing
- [ ] Test error handling for failed tool calls
- [ ] Coverage >= 90% for tool invocation

#### Technical Notes

```typescript
// src/lib/mcp/__tests__/tools.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { MCPClient } from '../client';
import { setupServer } from 'msw/node';
import { http, HttpResponse } from 'msw';

describe('MCP Tool Invocation', () => {
  let client: MCPClient;

  const server = setupServer(
    // MCP RPC endpoint mock
    http.post('http://hx-docling-server:8000/rpc', async ({ request }) => {
      const body = await request.json();

      if (body.method === 'initialize') {
        return HttpResponse.json({
          jsonrpc: '2.0',
          id: body.id,
          result: { protocolVersion: '2024-11-05', serverInfo: {}, capabilities: { tools: {} } },
        });
      }

      if (body.method === 'notifications/initialized') {
        return HttpResponse.json({ jsonrpc: '2.0', id: body.id, result: {} });
      }

      if (body.method === 'tools/call') {
        const { name, arguments: args } = body.params;

        if (name === 'convert_pdf') {
          return HttpResponse.json({
            jsonrpc: '2.0',
            id: body.id,
            result: {
              content: [{ type: 'document', document: { text: 'Converted content' } }],
            },
          });
        }

        if (name === 'export_markdown') {
          return HttpResponse.json({
            jsonrpc: '2.0',
            id: body.id,
            result: {
              content: [{ type: 'text', text: '# Exported Markdown' }],
            },
          });
        }
      }

      return HttpResponse.json({ jsonrpc: '2.0', id: body.id, result: {} });
    })
  );

  beforeEach(async () => {
    server.listen();
    client = new MCPClient('http://hx-docling-server:8000');
    await client.initialize();
  });

  afterEach(() => server.close());

  describe('convert_pdf', () => {
    it('invokes convert_pdf with file path', async () => {
      const result = await client.invokeTool('convert_pdf', {
        file_path: '/data/uploads/test.pdf',
      });

      expect(result).toBeDefined();
      expect(result.content[0].type).toBe('document');
    });

    it('validates required parameters', async () => {
      await expect(client.invokeTool('convert_pdf', {})).rejects.toThrow(/file_path/i);
    });
  });

  describe('export_markdown', () => {
    it('invokes export_markdown with document', async () => {
      const result = await client.invokeTool('export_markdown', {
        document: { text: 'Sample content' },
      });

      expect(result.content[0].text).toContain('# Exported Markdown');
    });
  });

  describe('Timeout handling', () => {
    it('uses 60s timeout for files < 10MB', async () => {
      const client = new MCPClient('http://hx-docling-server:8000');
      await client.initialize();

      const timeout = client.getTimeoutForSize(5 * 1024 * 1024); // 5MB
      expect(timeout).toBe(60000);
    });

    it('uses 180s timeout for files 10-50MB', async () => {
      const timeout = client.getTimeoutForSize(30 * 1024 * 1024); // 30MB
      expect(timeout).toBe(180000);
    });

    it('uses 300s timeout for files >= 50MB', async () => {
      const timeout = client.getTimeoutForSize(75 * 1024 * 1024); // 75MB
      expect(timeout).toBe(300000);
    });
  });

  describe('UUID v4 Request IDs', () => {
    it('generates UUID v4 format request IDs', async () => {
      let capturedId: string;
      server.use(
        http.post('http://hx-docling-server:8000/rpc', async ({ request }) => {
          const body = await request.json();
          capturedId = body.id;
          return HttpResponse.json({
            jsonrpc: '2.0',
            id: body.id,
            result: { content: [] },
          });
        })
      );

      await client.invokeTool('convert_pdf', { file_path: '/test.pdf' });

      // UUID v4 format: xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx
      expect(capturedId).toMatch(
        /^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i
      );
    });

    it('generates unique IDs for concurrent requests', async () => {
      const ids = new Set<string>();
      server.use(
        http.post('http://hx-docling-server:8000/rpc', async ({ request }) => {
          const body = await request.json();
          ids.add(body.id);
          return HttpResponse.json({
            jsonrpc: '2.0',
            id: body.id,
            result: { content: [] },
          });
        })
      );

      await Promise.all([
        client.invokeTool('convert_pdf', { file_path: '/test1.pdf' }),
        client.invokeTool('convert_pdf', { file_path: '/test2.pdf' }),
        client.invokeTool('convert_pdf', { file_path: '/test3.pdf' }),
      ]);

      expect(ids.size).toBe(3); // All unique
    });
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| Tool invocation tests | `src/lib/mcp/__tests__/tools.test.ts` |

---

### JUL-1.5a-004: Write Unit Tests for Tool Schema Caching

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.5a-004 |
| **Title** | Write Unit Tests for Tool Schema Caching |
| **Priority** | P2 (Medium) |
| **Effort** | 0.75 hours |
| **Dependencies** | JAMES-1.5a-006 (Tool schema caching implemented) |

#### Description

Write unit tests for the tool schema caching mechanism ensuring TTL behavior and cache invalidation work correctly.

#### Acceptance Criteria

- [ ] Test tools cached on first fetch
- [ ] Test cached tools returned on subsequent calls
- [ ] Test cache expires after TTL
- [ ] Test cache refresh after expiration
- [ ] Test manual cache invalidation
- [ ] Test cache populated after initialization
- [ ] Coverage >= 90% for tool cache

#### Technical Notes

```typescript
// src/lib/mcp/__tests__/tool-cache.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { ToolCache } from '../tool-cache';

describe('Tool Schema Cache', () => {
  let cache: ToolCache;
  const mockTools = [
    { name: 'convert_pdf', description: 'Convert PDF', inputSchema: {} },
    { name: 'export_markdown', description: 'Export MD', inputSchema: {} },
  ];

  beforeEach(() => {
    vi.useFakeTimers();
    cache = new ToolCache({ ttlMs: 60000 }); // 60s TTL
  });

  afterEach(() => {
    vi.useRealTimers();
  });

  it('caches tools on first fetch', () => {
    cache.set(mockTools);
    expect(cache.get()).toEqual(mockTools);
  });

  it('returns cached tools without refetch', () => {
    const fetchFn = vi.fn().mockResolvedValue(mockTools);
    cache.set(mockTools);

    const result = cache.getOrFetch(fetchFn);

    expect(fetchFn).not.toHaveBeenCalled();
    expect(result).toEqual(mockTools);
  });

  it('cache expires after TTL', async () => {
    cache.set(mockTools);

    vi.advanceTimersByTime(61000); // Past TTL

    expect(cache.get()).toBeNull();
  });

  it('refetches after cache expiration', async () => {
    const newTools = [...mockTools, { name: 'new_tool', description: 'New', inputSchema: {} }];
    const fetchFn = vi.fn().mockResolvedValue(newTools);

    cache.set(mockTools);
    vi.advanceTimersByTime(61000); // Past TTL

    const result = await cache.getOrFetch(fetchFn);

    expect(fetchFn).toHaveBeenCalled();
    expect(result).toEqual(newTools);
  });

  it('allows manual cache invalidation', () => {
    cache.set(mockTools);
    cache.invalidate();

    expect(cache.get()).toBeNull();
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| Tool cache tests | `src/lib/mcp/__tests__/tool-cache.test.ts` |

---

## Sprint 1.5a Testing Summary

| Metric | Target | Notes |
|--------|--------|-------|
| Total Tasks | 4 | |
| Total Effort | 4.5 hours | |
| Critical Tasks | 2 | Initialization, Error mapping |
| High Priority Tasks | 1 | Tool invocation |
| Medium Priority Tasks | 1 | Tool caching |

### Coverage Targets

| Component | Target Coverage |
|-----------|-----------------|
| `lib/mcp/initialize.ts` | 100% (protocol compliance) |
| `lib/mcp/error-mapping.ts` | 100% |
| `lib/mcp/client.ts` | >= 90% |
| `lib/mcp/tool-cache.ts` | >= 90% |

### Quality Gates for Sprint 1.5a

- [ ] MCP 3-step initialization verified
- [ ] notifications/initialized sent in correct order
- [ ] All JSON-RPC error codes mapped
- [ ] Tool invocations tested
- [ ] UUID v4 request IDs verified
- [ ] Size-based timeouts configured correctly

---

## Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0.0 | 2025-12-12 | Julia Santos | Initial creation |
