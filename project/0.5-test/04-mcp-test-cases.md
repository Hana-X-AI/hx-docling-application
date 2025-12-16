# HX Docling Application - MCP Test Cases

**Document Type**: MCP Tool Invocation Test Cases
**Version**: 2.2.0
**Status**: APPROVED
**Created**: 2025-12-12
**Last Updated**: 2025-12-15
**Author**: Julia Santos (Testing & QA Specialist)
**Master Plan Reference**: `project/0.5-test/00-test-plan-overview.md`
**Specification Reference**: `project/0.3-specification/0.3.1-detailed-specification.md` v1.2.1

---

## Table of Contents

1. [Overview](#1-overview)
2. [MCP Protocol Tests](#2-mcp-protocol-tests)
3. [Conversion Tool Tests](#3-conversion-tool-tests)
4. [Export Tool Tests](#4-export-tool-tests)
5. [Error Handling Tests](#5-error-handling-tests)
6. [Timeout and Retry Tests](#6-timeout-and-retry-tests)
7. [Tool Schema Validation Tests](#7-tool-schema-validation-tests)
8. [Database Transaction Tests](#8-database-transaction-tests)
9. [Redis Integration Tests](#9-redis-integration-tests)
10. [Frontend Integration Tests](#10-frontend-integration-tests)
11. [Infrastructure & Operations Tests](#11-infrastructure--operations-tests)
12. [Next.js Integration Tests](#12-nextjs-integration-tests)
13. [Workflow Orchestration Tests](#13-workflow-orchestration-tests)
14. [Protocol Compliance Tests](#14-protocol-compliance-tests)
15. [End-to-End Integration Tests](#15-end-to-end-integration-tests)
16. [Authentication & Security Tests](#16-authentication--security-tests)

---

## 1. Overview

### 1.1 Purpose

This document defines comprehensive test cases for MCP (Model Context Protocol) tool invocations between the hx-docling-ui application and the hx-docling-mcp-server. Tests cover the 8 MCP tools (4 conversion tools + 3 export tools + URL conversion), protocol compliance, error handling, and timeout behavior.

### 1.2 MCP Server Configuration

| Configuration | Value |
|---------------|-------|
| Endpoint | http://hx-docling-mcp-server.hx.dev.local:8000 |
| Protocol | HTTP/JSON-RPC 2.0 |
| Transport | HTTP (NOT SSE) |
| Protocol Version | 2024-11-05 |

### 1.3 MCP Tools Under Test

| Tool Name | Purpose | Input Schema |
|-----------|---------|--------------|
| convert_pdf | Convert PDF to DoclingDocument | `{ file_path: string }` |
| convert_docx | Convert DOCX to DoclingDocument | `{ file_path: string }` |
| convert_xlsx | Convert XLSX to DoclingDocument | `{ file_path: string }` |
| convert_pptx | Convert PPTX to DoclingDocument | `{ file_path: string }` |
| convert_url | Convert URL content to DoclingDocument | `{ url: string }` |
| export_markdown | Export DoclingDocument to Markdown | `{ document: object }` |
| export_html | Export DoclingDocument to HTML | `{ document: object }` |
| export_json | Export DoclingDocument to JSON | `{ document: object }` |

### 1.4 Test Strategy

- **Mock Strategy**: MSW (Mock Service Worker) for HTTP interception
- **Test Directory**: `test/integration/mcp/`
- **Test Framework**: Vitest
- **Coverage Target**: 100% of MCP tools, all error codes

---

## 2. MCP Protocol Tests

### 2.1 Initialization Sequence

---

### MCP-INIT-001: Initialize Request Format

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-INIT-001 |
| **Title** | Verify Initialize Request Format |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 7.1.2 |

**Test Steps**:

```typescript
it('sends correctly formatted initialize request', async () => {
  let initRequest: any = null;

  mockMCPServer((request) => {
    if (request.method === 'initialize') {
      initRequest = request;
      return {
        result: {
          protocolVersion: '2024-11-05',
          serverInfo: { name: 'hx-docling-mcp-server', version: '1.0.0' },
          capabilities: { tools: {} },
        },
      };
    }
  });

  const client = new MCPClient();
  await client.initialize();

  expect(initRequest.jsonrpc).toBe('2.0');
  expect(initRequest.method).toBe('initialize');
  expect(initRequest.params).toEqual({
    protocolVersion: expect.any(String),
    clientInfo: { name: 'hx-docling-ui', version: expect.any(String) },
    capabilities: { tools: { listChanged: false } },
  });
  expect(initRequest.id).toBeDefined();
});
```

**Expected Result**:
- JSON-RPC 2.0 format
- Correct method name
- Client info and capabilities present
- Request ID present

---

### MCP-INIT-002: Initialized Notification

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-INIT-002 |
| **Title** | Verify Initialized Notification Sent |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 7.1.2 |

**Test Steps**:

```typescript
it('sends initialized notification after initialize response', async () => {
  const requestSequence: string[] = [];

  mockMCPServer((request) => {
    requestSequence.push(request.method);
    if (request.method === 'initialize') {
      return { result: { serverInfo: {}, capabilities: { tools: {} } } };
    }
    if (request.method === 'notifications/initialized') {
      // Notification - no response expected
      return null;
    }
    if (request.method === 'tools/list') {
      return { result: { tools: [] } };
    }
  });

  const client = new MCPClient();
  await client.initialize();

  expect(requestSequence).toEqual([
    'initialize',
    'notifications/initialized',
    'tools/list',
  ]);
});
```

**Expected Result**:
- Initialized notification sent AFTER initialize response
- Notification has no request ID (per JSON-RPC notification spec)

---

### MCP-INIT-003: Protocol Version Configuration

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-INIT-003 |
| **Title** | Protocol Version from Environment |
| **Priority** | P1 - High |
| **Spec Reference** | Section 7.1.1.1 (MAJ-MCP-001) |

**Test Steps**:

```typescript
it('uses protocol version from environment variable', async () => {
  process.env.MCP_PROTOCOL_VERSION = '2025-01-01';

  let sentVersion: string | null = null;
  mockMCPServer((request) => {
    if (request.method === 'initialize') {
      sentVersion = request.params.protocolVersion;
      return { result: { serverInfo: {}, capabilities: { tools: {} } } };
    }
  });

  const client = new MCPClient();
  await client.initialize();

  expect(sentVersion).toBe('2025-01-01');
});
```

**Expected Result**:
- Protocol version configurable via MCP_PROTOCOL_VERSION environment variable

---

### MCP-INIT-004: Server Capabilities Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-INIT-004 |
| **Title** | Validate Server Supports Tools |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 7.1.1.2 (MAJ-MCP-002) |

**Test Steps**:

```typescript
it('rejects server without tools capability', async () => {
  mockMCPServer((request) => {
    if (request.method === 'initialize') {
      return {
        result: {
          serverInfo: { name: 'test', version: '1.0' },
          capabilities: {}, // No tools capability
        },
      };
    }
  });

  const client = new MCPClient();

  await expect(client.initialize()).rejects.toMatchObject({
    code: 'E205',
    message: expect.stringContaining('tools capability'),
  });
});
```

**Expected Result**:
- Error E205 if server lacks tools capability

---

### MCP-INIT-005: Tools List Caching

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-INIT-005 |
| **Title** | Cache Tool Schemas After Init |
| **Priority** | P1 - High |
| **Spec Reference** | Section 7.1.2 |

**Test Steps**:

```typescript
it('caches tool schemas from tools/list', async () => {
  let toolsListCallCount = 0;

  mockMCPServer((request) => {
    if (request.method === 'tools/list') {
      toolsListCallCount++;
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

  // Second initialization should use cache
  await client.initialize();

  expect(toolsListCallCount).toBe(1);
  expect(client.hasTool('convert_pdf')).toBe(true);
  expect(client.hasTool('export_markdown')).toBe(true);
});
```

**Expected Result**:
- tools/list called only once
- Tools accessible via hasTool()

---

## 3. Conversion Tool Tests

### 3.1 convert_pdf Tests

---

### MCP-CONV-001: Convert PDF - Success

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-CONV-001 |
| **Title** | Convert PDF Successfully |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-402 |

**Test Steps**:

```typescript
it('successfully converts PDF file', async () => {
  const mockDocument = {
    schema_name: 'docling_core.transforms.v1.DoclingDocument',
    version: '1.0.0',
    name: 'test.pdf',
    origin: { mimetype: 'application/pdf', filename: 'test.pdf' },
    body: { self_ref: '#/body', children: [], content_text: 'Test content' },
  };

  mockMCPServer((request) => {
    if (request.method === 'tools/call' && request.params.name === 'convert_pdf') {
      return {
        result: {
          content: [{ type: 'document', document: mockDocument }],
        },
      };
    }
  });

  const client = await getMCPClient();
  const result = await client.invoke('convert_pdf', { file_path: '/tmp/test.pdf' });

  expect(result.content[0].document.schema_name).toBe('docling_core.transforms.v1.DoclingDocument');
  expect(result.content[0].document.name).toBe('test.pdf');
});
```

**Expected Result**:
- DoclingDocument returned
- Correct schema and structure

---

### MCP-CONV-002: Convert DOCX - Success

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-CONV-002 |
| **Title** | Convert DOCX Successfully |
| **Priority** | P1 - High |
| **Spec Reference** | FR-402 |

**Test Steps**:

```typescript
it('successfully converts DOCX file', async () => {
  mockMCPServer((request) => {
    if (request.params.name === 'convert_docx') {
      return { result: { content: [{ type: 'document', document: mockDoclingDoc }] } };
    }
  });

  const result = await client.invoke('convert_docx', { file_path: '/tmp/test.docx' });

  expect(result.content[0].type).toBe('document');
});
```

---

### MCP-CONV-003: Convert XLSX - Success

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-CONV-003 |
| **Title** | Convert XLSX Successfully |
| **Priority** | P1 - High |
| **Spec Reference** | FR-402 |

**Test Steps**: Similar to MCP-CONV-002 with convert_xlsx tool

---

### MCP-CONV-004: Convert PPTX - Success

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-CONV-004 |
| **Title** | Convert PPTX Successfully |
| **Priority** | P1 - High |
| **Spec Reference** | FR-402 |

**Test Steps**: Similar to MCP-CONV-002 with convert_pptx tool

---

### MCP-CONV-005: Convert URL - Success

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-CONV-005 |
| **Title** | Convert URL Successfully |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-201, FR-402 |

**Test Steps**:

```typescript
it('successfully converts URL content', async () => {
  mockMCPServer((request) => {
    if (request.params.name === 'convert_url') {
      expect(request.params.arguments.url).toBe('https://example.com/doc.html');
      return { result: { content: [{ type: 'document', document: mockDoclingDoc }] } };
    }
  });

  const result = await client.invoke('convert_url', { url: 'https://example.com/doc.html' });

  expect(result.content[0].type).toBe('document');
});
```

---

### MCP-CONV-006: Convert with Options

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-CONV-006 |
| **Title** | Conversion with Options Passthrough |
| **Priority** | P2 - Medium |
| **Spec Reference** | Section 7.1.3 |

**Test Steps**:

```typescript
it('passes conversion options to MCP server', async () => {
  let receivedOptions: any = null;

  mockMCPServer((request) => {
    if (request.params.name === 'convert_pdf') {
      receivedOptions = request.params.arguments.options;
      return { result: { content: [{ type: 'document', document: {} }] } };
    }
  });

  await client.invoke('convert_pdf', {
    file_path: '/tmp/test.pdf',
    options: { ocr_enabled: true, language: 'en' },
  });

  expect(receivedOptions).toEqual({ ocr_enabled: true, language: 'en' });
});
```

---

## 4. Export Tool Tests

### 4.1 export_markdown Tests

---

### MCP-EXP-001: Export Markdown - Success

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-EXP-001 |
| **Title** | Export to Markdown Successfully |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-404 |

**Test Steps**:

```typescript
it('successfully exports document to markdown', async () => {
  const mockMarkdown = '# Document Title\n\nThis is the content.';

  mockMCPServer((request) => {
    if (request.params.name === 'export_markdown') {
      expect(request.params.arguments.document).toHaveProperty('schema_name');
      return {
        result: {
          content: [{ type: 'text', text: mockMarkdown }],
        },
      };
    }
  });

  const result = await client.invoke('export_markdown', { document: mockDoclingDoc });

  expect(result.content[0].type).toBe('text');
  expect(result.content[0].text).toContain('# Document Title');
});
```

---

### MCP-EXP-002: Export HTML - Success

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-EXP-002 |
| **Title** | Export to HTML Successfully |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-404 |

**Test Steps**:

```typescript
it('successfully exports document to HTML', async () => {
  const mockHtml = '<html><body><h1>Document Title</h1><p>Content</p></body></html>';

  mockMCPServer((request) => {
    if (request.params.name === 'export_html') {
      return { result: { content: [{ type: 'text', text: mockHtml }] } };
    }
  });

  const result = await client.invoke('export_html', { document: mockDoclingDoc });

  expect(result.content[0].text).toContain('<html>');
  expect(result.content[0].text).toContain('<h1>Document Title</h1>');
});
```

---

### MCP-EXP-003: Export JSON - Success

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-EXP-003 |
| **Title** | Export to JSON Successfully |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-404 |

**Test Steps**:

```typescript
it('successfully exports document to JSON', async () => {
  const mockJson = JSON.stringify(mockDoclingDoc, null, 2);

  mockMCPServer((request) => {
    if (request.params.name === 'export_json') {
      return { result: { content: [{ type: 'text', text: mockJson }] } };
    }
  });

  const result = await client.invoke('export_json', { document: mockDoclingDoc });

  const parsed = JSON.parse(result.content[0].text);
  expect(parsed).toHaveProperty('schema_name');
});
```

---

### MCP-EXP-004: Parallel Export Execution

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-EXP-004 |
| **Title** | Export Tools Execute in Parallel |
| **Priority** | P1 - High |
| **Spec Reference** | FR-404 (Prompt Chaining Pattern) |

**Test Steps**:

```typescript
it('executes all exports in parallel', async () => {
  const exportStartTimes: Record<string, number> = {};

  mockMCPServer((request) => {
    if (request.params.name.startsWith('export_')) {
      exportStartTimes[request.params.name] = Date.now();
      // Simulate 100ms delay
      return new Promise((resolve) => {
        setTimeout(() => {
          resolve({ result: { content: [{ type: 'text', text: 'content' }] } });
        }, 100);
      });
    }
  });

  // Execute all exports in parallel
  const results = await Promise.allSettled([
    client.invoke('export_markdown', { document: mockDoclingDoc }),
    client.invoke('export_html', { document: mockDoclingDoc }),
    client.invoke('export_json', { document: mockDoclingDoc }),
  ]);

  // All should be fulfilled
  expect(results.every(r => r.status === 'fulfilled')).toBe(true);

  // Start times should be within 50ms of each other (parallel)
  const times = Object.values(exportStartTimes);
  const maxDiff = Math.max(...times) - Math.min(...times);
  expect(maxDiff).toBeLessThan(50);
});
```

---

### MCP-EXP-005: Partial Export Failure

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-EXP-005 |
| **Title** | Handle Partial Export Failure |
| **Priority** | P1 - High |
| **Spec Reference** | FR-405 |

**Test Steps**:

```typescript
it('handles partial export failure with Promise.allSettled', async () => {
  mockMCPServer((request) => {
    if (request.params.name === 'export_markdown') {
      return { result: { content: [{ type: 'text', text: '# Content' }] } };
    }
    if (request.params.name === 'export_html') {
      // Simulate failure
      return { error: { code: -32003, message: 'Export failed' } };
    }
    if (request.params.name === 'export_json') {
      return { result: { content: [{ type: 'text', text: '{}' }] } };
    }
  });

  const results = await Promise.allSettled([
    client.invoke('export_markdown', { document: mockDoclingDoc }),
    client.invoke('export_html', { document: mockDoclingDoc }),
    client.invoke('export_json', { document: mockDoclingDoc }),
  ]);

  // Check results
  expect(results[0].status).toBe('fulfilled');
  expect(results[1].status).toBe('rejected');
  expect(results[2].status).toBe('fulfilled');
});
```

---

## 5. Error Handling Tests

### 5.1 JSON-RPC Standard Errors

---

### MCP-ERR-001: Parse Error (-32700)

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-ERR-001 |
| **Title** | Handle Parse Error |
| **Priority** | P1 - High |
| **Spec Reference** | Section 7.1.4 |

**Test Steps**:

```typescript
it('maps -32700 to E203', async () => {
  mockMCPServer(() => ({
    error: { code: -32700, message: 'Parse error' },
  }));

  await expect(client.invoke('convert_pdf', { file_path: '/test.pdf' }))
    .rejects.toMatchObject({
      code: 'E203',
      retryable: false,
    });
});
```

---

### MCP-ERR-002: Method Not Found (-32601)

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-ERR-002 |
| **Title** | Handle Unknown Tool Error |
| **Priority** | P1 - High |
| **Spec Reference** | Section 7.1.4, E204 |

**Test Steps**:

```typescript
it('maps -32601 to E204', async () => {
  mockMCPServer((request) => {
    if (request.params.name === 'unknown_tool') {
      return { error: { code: -32601, message: 'Method not found' } };
    }
  });

  await expect(client.invoke('unknown_tool', {}))
    .rejects.toMatchObject({
      code: 'E204',
      message: expect.stringContaining('Tool not found'),
      retryable: false,
    });
});
```

---

### MCP-ERR-003: Invalid Params (-32602)

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-ERR-003 |
| **Title** | Handle Invalid Parameters Error |
| **Priority** | P1 - High |
| **Spec Reference** | Section 7.1.4, E203 |

**Test Steps**:

```typescript
it('maps -32602 to E203', async () => {
  mockMCPServer(() => ({
    error: { code: -32602, message: 'Invalid params: missing file_path' },
  }));

  await expect(client.invoke('convert_pdf', {}))
    .rejects.toMatchObject({
      code: 'E203',
      retryable: false,
    });
});
```

---

### MCP-ERR-004: Internal Error (-32603)

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-ERR-004 |
| **Title** | Handle Internal Error |
| **Priority** | P1 - High |
| **Spec Reference** | Section 7.1.4, E201 |

**Test Steps**:

```typescript
it('maps -32603 to E201 (retryable)', async () => {
  mockMCPServer(() => ({
    error: { code: -32603, message: 'Internal error' },
  }));

  await expect(client.invoke('convert_pdf', { file_path: '/test.pdf' }))
    .rejects.toMatchObject({
      code: 'E201',
      retryable: true,
    });
});
```

---

### 5.2 MCP Server-Specific Errors

---

### MCP-ERR-005: Server Unavailable (-32000)

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-ERR-005 |
| **Title** | Handle Server Unavailable Error |
| **Priority** | P1 - High |
| **Spec Reference** | Section 7.1.4, E201 |

**Test Steps**:

```typescript
it('maps -32000 to E201 (retryable)', async () => {
  mockMCPServer(() => ({
    error: { code: -32000, message: 'Service unavailable' },
  }));

  await expect(client.invoke('convert_pdf', { file_path: '/test.pdf' }))
    .rejects.toMatchObject({
      code: 'E201',
      userMessage: expect.stringContaining('unavailable'),
      retryable: true,
    });
});
```

---

### MCP-ERR-006: Request Timeout (-32001)

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-ERR-006 |
| **Title** | Handle MCP Timeout Error |
| **Priority** | P1 - High |
| **Spec Reference** | Section 7.1.4, E202 |

**Test Steps**:

```typescript
it('maps -32001 to E202 (retryable)', async () => {
  mockMCPServer(() => ({
    error: { code: -32001, message: 'Request timeout' },
  }));

  await expect(client.invoke('convert_pdf', { file_path: '/test.pdf' }))
    .rejects.toMatchObject({
      code: 'E202',
      userMessage: expect.stringContaining('timeout'),
      retryable: true,
    });
});
```

---

### MCP-ERR-007: Conversion Failed (-32002)

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-ERR-007 |
| **Title** | Handle Conversion Failure |
| **Priority** | P1 - High |
| **Spec Reference** | Section 7.1.4, E302 |

**Test Steps**:

```typescript
it('maps -32002 to E302 (retryable)', async () => {
  mockMCPServer(() => ({
    error: { code: -32002, message: 'Document conversion failed' },
  }));

  await expect(client.invoke('convert_pdf', { file_path: '/test.pdf' }))
    .rejects.toMatchObject({
      code: 'E302',
      userMessage: expect.stringContaining('conversion failed'),
      retryable: true,
    });
});
```

---

### MCP-ERR-008: Export Failed (-32003)

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-ERR-008 |
| **Title** | Handle Export Failure |
| **Priority** | P1 - High |
| **Spec Reference** | Section 7.1.4, E303 |

**Test Steps**:

```typescript
it('maps -32003 to E303 (retryable)', async () => {
  mockMCPServer(() => ({
    error: { code: -32003, message: 'Export generation failed' },
  }));

  await expect(client.invoke('export_markdown', { document: {} }))
    .rejects.toMatchObject({
      code: 'E303',
      retryable: true,
    });
});
```

---

### MCP-ERR-009: Unknown Error Code

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-ERR-009 |
| **Title** | Handle Unknown MCP Error Code |
| **Priority** | P2 - Medium |
| **Spec Reference** | Section 7.1.4 |

**Test Steps**:

```typescript
it('maps unknown error to E201 (retryable)', async () => {
  mockMCPServer(() => ({
    error: { code: -99999, message: 'Unknown error' },
  }));

  await expect(client.invoke('convert_pdf', { file_path: '/test.pdf' }))
    .rejects.toMatchObject({
      code: 'E201',
      originalMCPCode: -99999,
      retryable: true,
    });
});
```

---

### MCP-ERR-010: Network Connection Error

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-ERR-010 |
| **Title** | Handle Network Connection Error |
| **Priority** | P1 - High |
| **Spec Reference** | E201 |

**Test Steps**:

```typescript
it('handles network connection failure', async () => {
  // Make MCP endpoint unreachable
  mockMCPServer.close();

  await expect(client.invoke('convert_pdf', { file_path: '/test.pdf' }))
    .rejects.toMatchObject({
      code: 'E201',
      retryable: true,
    });
});
```

---

### 5.3 MCP Error Recovery Tests

---

### MCP-ERR-011: Transient Error Retry with Exponential Backoff

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-ERR-011 |
| **Title** | Verify Retry with Exponential Backoff for Transient Errors |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 7.1.4, FR-803 |

**Test Steps**:

```typescript
it('retries transient errors with exponential backoff', async () => {
  let attemptCount = 0;
  const attemptTimestamps: number[] = [];

  mockMCPServer(() => {
    attemptTimestamps.push(Date.now());
    attemptCount++;

    if (attemptCount <= 2) {
      // Fail first 2 attempts with retryable error
      return { error: { code: -32603, message: 'Internal error' } };
    }

    // Succeed on 3rd attempt
    return {
      result: {
        content: [{ type: 'text', text: 'Success after retries' }],
      },
    };
  });

  const result = await client.invoke('convert_pdf', { file_path: '/test.pdf' });

  // Verify success after retries
  expect(result).toMatchObject({
    content: [{ type: 'text', text: 'Success after retries' }],
  });

  // Verify exponential backoff timing (1s, 2s)
  expect(attemptCount).toBe(3);
  expect(attemptTimestamps.length).toBe(3);

  const backoff1 = attemptTimestamps[1] - attemptTimestamps[0];
  const backoff2 = attemptTimestamps[2] - attemptTimestamps[1];

  expect(backoff1).toBeGreaterThanOrEqual(950); // ~1s ±50ms
  expect(backoff1).toBeLessThan(1100);

  expect(backoff2).toBeGreaterThanOrEqual(1950); // ~2s ±50ms
  expect(backoff2).toBeLessThan(2100);
});
```

**Expected Result**: Transient errors trigger automatic retry with 1s, 2s exponential backoff, operation succeeds on 3rd attempt

---

### MCP-ERR-012: Permanent Error Detection and Fail-Fast

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-ERR-012 |
| **Title** | Verify Immediate Failure for Non-Retryable Errors |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 7.1.4, FR-803 |

**Test Steps**:

```typescript
it('fails immediately for permanent errors without retry', async () => {
  let attemptCount = 0;

  mockMCPServer(() => {
    attemptCount++;
    // Return non-retryable error (parse error)
    return { error: { code: -32700, message: 'Parse error' } };
  });

  const startTime = Date.now();

  await expect(client.invoke('convert_pdf', { file_path: '/test.pdf' }))
    .rejects.toMatchObject({
      code: 'E203',
      retryable: false,
    });

  const duration = Date.now() - startTime;

  // Verify no retries attempted (single attempt only)
  expect(attemptCount).toBe(1);

  // Verify fail-fast (< 100ms, no backoff delay)
  expect(duration).toBeLessThan(100);
});
```

**Expected Result**: Non-retryable errors (parse error, invalid params, method not found) fail immediately without retry attempts

---

### MCP-ERR-013: Partial Failure Handling in Batch Operations

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-ERR-013 |
| **Title** | Verify Partial Success Handling for Batch Tool Invocations |
| **Priority** | P1 - High |
| **Spec Reference** | Section 7.1.4 |

**Test Steps**:

```typescript
it('handles partial failures in batch operations', async () => {
  const batchResults: Array<{ tool: string; success: boolean }> = [];

  // Simulate batch conversion: 3 files, 1 fails
  const files = ['/doc1.pdf', '/doc2.pdf', '/doc3.pdf'];

  let callIndex = 0;
  mockMCPServer((request) => {
    const filePath = request.params.arguments.file_path;
    callIndex++;

    if (filePath === '/doc2.pdf') {
      // Second file fails with permanent error
      return { error: { code: -32602, message: 'Invalid file format' } };
    }

    // Other files succeed
    return {
      result: {
        content: [{ type: 'text', text: `Converted ${filePath}` }],
      },
    };
  });

  // Execute batch operation
  const promises = files.map(async (filePath) => {
    try {
      await client.invoke('convert_pdf', { file_path: filePath });
      batchResults.push({ tool: filePath, success: true });
    } catch (error) {
      batchResults.push({ tool: filePath, success: false });
    }
  });

  await Promise.allSettled(promises);

  // Verify partial success
  expect(batchResults).toEqual([
    { tool: '/doc1.pdf', success: true },
    { tool: '/doc2.pdf', success: false },
    { tool: '/doc3.pdf', success: true },
  ]);

  // Verify all operations were attempted (no cascade failure)
  expect(callIndex).toBe(3);
});
```

**Expected Result**: Batch operations continue despite individual failures, successful operations complete independently, failed operations report errors without affecting other operations

---

### MCP-ERR-014: Error Categorization (Retryable vs Non-Retryable)

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-ERR-014 |
| **Title** | Verify Correct Error Categorization for Retry Logic |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 7.1.4 |

**Test Steps**:

```typescript
it('categorizes errors correctly for retry logic', async () => {
  const errorCategories = {
    retryable: [
      { code: -32603, name: 'Internal Error' },
      { code: -32000, name: 'Server Unavailable' },
      { code: -32001, name: 'Timeout' },
      { code: -32002, name: 'Conversion Failed' },
      { code: -32003, name: 'Export Failed' },
      { code: -99999, name: 'Unknown Error' },
    ],
    nonRetryable: [
      { code: -32700, name: 'Parse Error' },
      { code: -32600, name: 'Invalid Request' },
      { code: -32601, name: 'Method Not Found' },
      { code: -32602, name: 'Invalid Params' },
    ],
  };

  // Test retryable errors
  for (const { code, name } of errorCategories.retryable) {
    mockMCPServer(() => ({ error: { code, message: name } }));

    await expect(client.invoke('convert_pdf', { file_path: '/test.pdf' }))
      .rejects.toMatchObject({
        retryable: true,
      });
  }

  // Test non-retryable errors
  for (const { code, name } of errorCategories.nonRetryable) {
    mockMCPServer(() => ({ error: { code, message: name } }));

    await expect(client.invoke('convert_pdf', { file_path: '/test.pdf' }))
      .rejects.toMatchObject({
        retryable: false,
      });
  }
});
```

**Expected Result**: All transient/server errors marked retryable=true, all client/validation errors marked retryable=false, retry logic respects categorization

---

### MCP-ERR-015: Error Context Preservation Across Retries

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-ERR-015 |
| **Title** | Verify Error Context and Original Request Preserved During Retries |
| **Priority** | P1 - High |
| **Spec Reference** | Section 7.1.4, FR-803 |

**Test Steps**:

```typescript
it('preserves error context and request data across retries', async () => {
  const retryContexts: Array<{
    attempt: number;
    originalRequest: any;
    previousError?: any;
  }> = [];

  let attemptCount = 0;
  const originalFilePath = '/complex/path/to/document.pdf';

  mockMCPServer((request) => {
    attemptCount++;

    // Capture retry context
    retryContexts.push({
      attempt: attemptCount,
      originalRequest: request.params.arguments,
      previousError: request.params.metadata?.previousError,
    });

    if (attemptCount <= 2) {
      return { error: { code: -32603, message: 'Internal error' } };
    }

    return {
      result: {
        content: [{ type: 'text', text: 'Success' }],
      },
    };
  });

  await client.invoke('convert_pdf', { file_path: originalFilePath });

  // Verify all retries used original request parameters
  expect(retryContexts).toHaveLength(3);
  retryContexts.forEach((context) => {
    expect(context.originalRequest.file_path).toBe(originalFilePath);
  });

  // Verify previous error context available on retries
  expect(retryContexts[1].previousError).toMatchObject({
    code: -32603,
    attempt: 1,
  });
  expect(retryContexts[2].previousError).toMatchObject({
    code: -32603,
    attempt: 2,
  });
});
```

**Expected Result**: Original request parameters preserved across all retry attempts, error context from previous attempts available for debugging, retry metadata tracks attempt number and error history

---

### MCP-ERR-016: Circuit Breaker Integration with MCP Client

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-ERR-016 |
| **Title** | Verify Circuit Breaker Opens After Threshold Failures |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 7.4.3 |

**Test Steps**:

```typescript
it('opens circuit breaker after repeated failures', async () => {
  const circuitBreakerConfig = {
    failureThreshold: 5,
    resetTimeout: 10000, // 10s
    successThreshold: 2,
  };

  // Configure circuit breaker
  client.configureCircuitBreaker(circuitBreakerConfig);

  // Simulate repeated failures
  mockMCPServer(() => ({
    error: { code: -32000, message: 'Service unavailable' },
  }));

  // First 5 requests should hit MCP server and fail
  for (let i = 0; i < 5; i++) {
    await expect(client.invoke('convert_pdf', { file_path: '/test.pdf' }))
      .rejects.toMatchObject({
        code: 'E201',
        retryable: true,
      });
  }

  // 6th request should fail immediately with circuit open error
  await expect(client.invoke('convert_pdf', { file_path: '/test.pdf' }))
    .rejects.toMatchObject({
      code: 'E201',
      message: expect.stringContaining('circuit breaker is open'),
      retryable: false, // Don't retry when circuit is open
    });

  // Verify circuit breaker state
  expect(client.getCircuitBreakerState()).toMatchObject({
    state: 'open',
    failureCount: 5,
  });
});
```

**Expected Result**: Circuit breaker opens after threshold failures (5), subsequent requests fail immediately without hitting MCP server, circuit breaker state reflects 'open' status

---

### MCP-ERR-017: Timeout Cascade Prevention

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-ERR-017 |
| **Title** | Verify Timeout Handling Prevents Cascade Failures |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 7.1.4, E202 |

**Test Steps**:

```typescript
it('prevents timeout cascades with proper error handling', async () => {
  const concurrentRequests = 10;
  const requestTimestamps: Array<{ start: number; end: number }> = [];

  // Simulate slow MCP server (2s response time)
  mockMCPServer(async () => {
    await new Promise(resolve => setTimeout(resolve, 2000));
    return {
      result: {
        content: [{ type: 'text', text: 'Slow response' }],
      },
    };
  });

  // Configure aggressive timeout (1s)
  const timeoutConfig = { mcpTimeout: 1000 };

  // Execute concurrent requests
  const promises = Array.from({ length: concurrentRequests }, async () => {
    const start = Date.now();
    try {
      await client.invoke('convert_pdf', { file_path: '/test.pdf' }, timeoutConfig);
    } catch (error) {
      const end = Date.now();
      requestTimestamps.push({ start, end });
      throw error;
    }
  });

  await Promise.allSettled(promises);

  // Verify all requests timed out independently
  expect(requestTimestamps).toHaveLength(concurrentRequests);

  // Verify timeouts occurred around 1s mark (±100ms)
  requestTimestamps.forEach(({ start, end }) => {
    const duration = end - start;
    expect(duration).toBeGreaterThanOrEqual(900);
    expect(duration).toBeLessThan(1200);
  });

  // Verify no cascade delay (all requests started within 100ms of each other)
  const firstStart = Math.min(...requestTimestamps.map(t => t.start));
  const lastStart = Math.max(...requestTimestamps.map(t => t.start));
  expect(lastStart - firstStart).toBeLessThan(100);
});
```

**Expected Result**: Concurrent timeout errors handled independently without cascading delays, each request times out at configured threshold, no request blocking or queueing due to timeout failures

---

### MCP-ERR-018: Error Recovery State Machine

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-ERR-018 |
| **Title** | Verify Complete Error Recovery State Transitions |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 7.1.4, FR-803 |

**Test Steps**:

```typescript
it('transitions through complete error recovery state machine', async () => {
  const stateTransitions: Array<{
    state: string;
    attempt: number;
    error?: any;
  }> = [];

  let attemptCount = 0;

  mockMCPServer(() => {
    attemptCount++;

    // Simulate state transitions: PROCESSING -> RETRY_1 -> RETRY_2 -> RETRY_3 -> ERROR
    if (attemptCount <= 3) {
      stateTransitions.push({
        state: attemptCount === 1 ? 'PROCESSING' : `RETRY_${attemptCount - 1}`,
        attempt: attemptCount,
        error: { code: -32603, message: 'Internal error' },
      });

      return { error: { code: -32603, message: 'Internal error' } };
    }
  });

  // Execute request that exhausts retries
  await expect(client.invoke('convert_pdf', { file_path: '/test.pdf' }))
    .rejects.toMatchObject({
      code: 'E201',
      finalState: 'ERROR',
      totalAttempts: 3,
      retryable: false, // No more retries after exhaustion
    });

  // Verify state transitions
  expect(stateTransitions).toEqual([
    { state: 'PROCESSING', attempt: 1, error: expect.any(Object) },
    { state: 'RETRY_1', attempt: 2, error: expect.any(Object) },
    { state: 'RETRY_2', attempt: 3, error: expect.any(Object) },
  ]);

  // Verify final state after retry exhaustion
  expect(attemptCount).toBe(3);
});

it('transitions from RETRY to SUCCESS on recovery', async () => {
  const stateTransitions: string[] = [];
  let attemptCount = 0;

  mockMCPServer(() => {
    attemptCount++;

    if (attemptCount === 1) {
      stateTransitions.push('PROCESSING');
      return { error: { code: -32603, message: 'Internal error' } };
    }

    if (attemptCount === 2) {
      stateTransitions.push('RETRY_1');
      // Succeed on retry
      return {
        result: {
          content: [{ type: 'text', text: 'Recovered' }],
        },
      };
    }
  });

  const result = await client.invoke('convert_pdf', { file_path: '/test.pdf' });

  // Verify successful recovery
  expect(result).toMatchObject({
    content: [{ type: 'text', text: 'Recovered' }],
  });

  // Verify state transitions: PROCESSING -> RETRY_1 -> SUCCESS
  expect(stateTransitions).toEqual(['PROCESSING', 'RETRY_1']);
  expect(attemptCount).toBe(2);
});
```

**Expected Result**: State machine transitions correctly through PROCESSING -> RETRY_1 -> RETRY_2 -> RETRY_3 -> ERROR for exhausted retries, transitions to SUCCESS on recovery, state transitions logged with attempt counts and error context

---

## 6. Timeout and Retry Tests

### 6.1 Timeout Configuration

---

### MCP-TO-001: Size-Based Timeout Selection

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-TO-001 |
| **Title** | Apply Size-Based Timeout |
| **Priority** | P1 - High |
| **Spec Reference** | Environment Variables (Appendix B) |

**Test Steps**:

```typescript
describe('Size-Based Timeouts', () => {
  const testCases = [
    { size: 500 * 1024, expectedTimeout: 60000, description: 'small (<1MB)' },
    { size: 5 * 1024 * 1024, expectedTimeout: 180000, description: 'medium (1-10MB)' },
    { size: 20 * 1024 * 1024, expectedTimeout: 300000, description: 'large (>10MB)' },
  ];

  testCases.forEach(({ size, expectedTimeout, description }) => {
    it(`applies ${description} timeout for ${size} bytes`, async () => {
      const timeout = getTimeoutForFileSize(size);
      expect(timeout).toBe(expectedTimeout);
    });
  });
});
```

---

### MCP-TO-002: Client-Side Timeout Enforcement

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-TO-002 |
| **Title** | Client Enforces Timeout |
| **Priority** | P1 - High |
| **Spec Reference** | E202 |

**Test Steps**:

```typescript
it('throws E202 when request exceeds timeout', async () => {
  mockMCPServer(() => {
    // Never respond
    return new Promise(() => {});
  });

  const client = new MCPClient({ defaultTimeout: 100 });

  const start = Date.now();
  await expect(client.invoke('convert_pdf', { file_path: '/test.pdf' }))
    .rejects.toMatchObject({ code: 'E202' });

  const elapsed = Date.now() - start;
  expect(elapsed).toBeLessThan(200);
});
```

---

### 6.2 Retry Behavior

---

### MCP-RET-001: Retry on Retryable Error

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-RET-001 |
| **Title** | Automatic Retry on Retryable Error |
| **Priority** | P1 - High |
| **Spec Reference** | FR-803 |

**Test Steps**:

```typescript
it('retries on retryable error (E201)', async () => {
  let callCount = 0;

  mockMCPServer(() => {
    callCount++;
    if (callCount < 3) {
      return { error: { code: -32603, message: 'Internal error' } };
    }
    return { result: { content: [{ type: 'document', document: {} }] } };
  });

  const result = await client.invokeWithRetry('convert_pdf', { file_path: '/test.pdf' });

  expect(callCount).toBe(3);
  expect(result).toBeDefined();
});
```

---

### MCP-RET-002: No Retry on Non-Retryable Error

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-RET-002 |
| **Title** | No Retry on Non-Retryable Error |
| **Priority** | P1 - High |
| **Spec Reference** | FR-803 |

**Test Steps**:

```typescript
it('does not retry on non-retryable error (E204)', async () => {
  let callCount = 0;

  mockMCPServer(() => {
    callCount++;
    return { error: { code: -32601, message: 'Method not found' } };
  });

  await expect(client.invokeWithRetry('unknown_tool', {})).rejects.toThrow();

  expect(callCount).toBe(1); // No retry
});
```

---

### MCP-RET-003: Exponential Backoff

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-RET-003 |
| **Title** | Exponential Backoff Between Retries |
| **Priority** | P1 - High |
| **Spec Reference** | FR-803 |

**Test Steps**:

```typescript
it('applies exponential backoff (1s, 2s, 4s)', async () => {
  const callTimes: number[] = [];

  mockMCPServer(() => {
    callTimes.push(Date.now());
    return { error: { code: -32603, message: 'Internal error' } };
  });

  await expect(client.invokeWithRetry('convert_pdf', { file_path: '/test.pdf' }))
    .rejects.toThrow();

  // Should have 4 calls (initial + 3 retries)
  expect(callTimes.length).toBe(4);

  // Verify backoff delays (with tolerance)
  const delay1 = callTimes[1] - callTimes[0];
  const delay2 = callTimes[2] - callTimes[1];
  const delay3 = callTimes[3] - callTimes[2];

  expect(delay1).toBeGreaterThanOrEqual(900);
  expect(delay1).toBeLessThan(1200);
  expect(delay2).toBeGreaterThanOrEqual(1900);
  expect(delay2).toBeLessThan(2200);
  expect(delay3).toBeGreaterThanOrEqual(3900);
  expect(delay3).toBeLessThan(4200);
});
```

---

### MCP-RET-004: Max Retries Exhausted

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-RET-004 |
| **Title** | Error After Max Retries |
| **Priority** | P1 - High |
| **Spec Reference** | FR-803 |

**Test Steps**:

```typescript
it('throws error after max retries (3)', async () => {
  let callCount = 0;

  mockMCPServer(() => {
    callCount++;
    return { error: { code: -32603, message: 'Internal error' } };
  });

  await expect(client.invokeWithRetry('convert_pdf', { file_path: '/test.pdf' }))
    .rejects.toMatchObject({
      code: 'E201',
      message: expect.stringContaining('retries exhausted'),
    });

  expect(callCount).toBe(4); // Initial + 3 retries
});
```

---

### MCP-RET-005: Cancellation During Retry

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-RET-005 |
| **Title** | Cancel Aborts Pending Retry |
| **Priority** | P1 - High |
| **Spec Reference** | FR-406 |

**Test Steps**:

```typescript
it('cancellation aborts during backoff', async () => {
  const controller = new AbortController();
  let callCount = 0;

  mockMCPServer(() => {
    callCount++;
    return { error: { code: -32603, message: 'Internal error' } };
  });

  const promise = client.invokeWithRetry('convert_pdf', {
    file_path: '/test.pdf',
    signal: controller.signal,
  });

  // Cancel during first backoff delay
  setTimeout(() => controller.abort(), 500);

  await expect(promise).rejects.toMatchObject({ name: 'AbortError' });
  expect(callCount).toBe(1); // Only initial call, no retry after cancel
});
```

---

## 7. Tool Schema Validation Tests

### MCP-SCH-001: Validate convert_pdf Schema

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-SCH-001 |
| **Title** | Validate convert_pdf Input Schema |
| **Priority** | P1 - High |
| **Spec Reference** | Section 7.1.3 |

**Test Steps**:

```typescript
it('validates convert_pdf requires file_path', async () => {
  const schema = TOOL_SCHEMAS['convert_pdf'];

  const validInput = { file_path: '/tmp/test.pdf' };
  const invalidInput = { path: '/tmp/test.pdf' }; // Wrong property name

  expect(schema.safeParse(validInput).success).toBe(true);
  expect(schema.safeParse(invalidInput).success).toBe(false);
});
```

---

### MCP-SCH-002: Validate convert_url Schema

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-SCH-002 |
| **Title** | Validate convert_url Input Schema |
| **Priority** | P1 - High |
| **Spec Reference** | Section 7.1.3 |

**Test Steps**:

```typescript
it('validates convert_url requires valid URL', async () => {
  const schema = TOOL_SCHEMAS['convert_url'];

  const validInput = { url: 'https://example.com/doc.html' };
  const invalidInput = { url: 'not-a-url' };

  expect(schema.safeParse(validInput).success).toBe(true);
  expect(schema.safeParse(invalidInput).success).toBe(false);
});
```

---

### MCP-SCH-003: Validate export_* Schemas

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-SCH-003 |
| **Title** | Validate Export Tool Schemas |
| **Priority** | P1 - High |
| **Spec Reference** | Section 7.1.3 |

**Test Steps**:

```typescript
describe('Export Tool Schemas', () => {
  ['export_markdown', 'export_html', 'export_json'].forEach(toolName => {
    it(`validates ${toolName} requires document`, async () => {
      const schema = TOOL_SCHEMAS[toolName];

      const validInput = { document: { schema_name: 'test', version: '1.0' } };
      const invalidInput = { content: 'raw string' };

      expect(schema.safeParse(validInput).success).toBe(true);
      expect(schema.safeParse(invalidInput).success).toBe(false);
    });
  });
});
```

---

### MCP-SCH-004: DoclingDocument Schema Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-SCH-004 |
| **Title** | Validate DoclingDocument Response Schema |
| **Priority** | P1 - High |
| **Spec Reference** | Section 7.1.3 (MAJ-16) |

**Test Steps**:

```typescript
it('validates DoclingDocument structure', async () => {
  const validDoc = {
    schema_name: 'docling_core.transforms.v1.DoclingDocument',
    version: '1.0.0',
    name: 'test.pdf',
    origin: { mimetype: 'application/pdf', filename: 'test.pdf' },
    body: { self_ref: '#/body', children: [] },
  };

  const invalidDoc = {
    // Missing required fields
    content: 'raw content',
  };

  expect(DoclingDocumentSchema.safeParse(validDoc).success).toBe(true);
  expect(DoclingDocumentSchema.safeParse(invalidDoc).success).toBe(false);
});
```

---

## 8. Database Transaction Tests

### 8.1 Job State Atomicity

---

### MCP-DB-001: Atomic Job State Update

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-DB-001 |
| **Title** | Verify Atomic Job State Transitions |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.3 (Job State Management) |

**Test Steps**:

```typescript
it('updates job state atomically during MCP call', async () => {
  const jobId = await createTestJob({ status: 'pending' });

  // Start MCP conversion
  const conversionPromise = mcpClient.invoke('convert_pdf', {
    file_path: '/tmp/test.pdf',
    jobId
  });

  // Verify job state transitions atomically
  await waitForJobState(jobId, 'processing');
  const job = await getJob(jobId);

  expect(job.status).toBe('processing');
  expect(job.started_at).toBeDefined();
  expect(job.updated_at).toBeDefined();

  await conversionPromise;

  const completedJob = await getJob(jobId);
  expect(completedJob.status).toBe('completed');
  expect(completedJob.completed_at).toBeDefined();
});
```

**Expected Result**:
- Job state transitions from pending -> processing -> completed atomically
- All timestamp fields populated correctly
- No intermediate inconsistent states observable

---

### MCP-DB-002: Result Persistence Transaction

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-DB-002 |
| **Title** | Verify Result and Job State Persisted Together |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.3 (Result Persistence) |

**Test Steps**:

```typescript
it('persists result and job state in single transaction', async () => {
  const jobId = await createTestJob({ status: 'pending' });

  mockMCPServer((request) => {
    if (request.params.name === 'convert_pdf') {
      return {
        result: {
          content: [{ type: 'document', document: mockDoclingDoc }],
        },
      };
    }
  });

  await mcpClient.invoke('convert_pdf', { file_path: '/tmp/test.pdf', jobId });

  // Verify both job and result exist atomically
  const job = await getJob(jobId);
  const result = await getResultByJobId(jobId);

  expect(job.status).toBe('completed');
  expect(result).toBeDefined();
  expect(result.document).toMatchObject({ schema_name: expect.any(String) });

  // Verify foreign key constraint maintained
  expect(result.job_id).toBe(jobId);
});
```

**Expected Result**:
- Job status and result persisted in same transaction
- Foreign key relationship maintained
- No orphaned results possible

---

### MCP-DB-003: Rollback on Partial Failure

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-DB-003 |
| **Title** | Rollback Transaction on Result Persistence Failure |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.3 (ACID Properties) |

**Test Steps**:

```typescript
it('rolls back job state on result persistence failure', async () => {
  const jobId = await createTestJob({ status: 'pending' });

  // Mock successful MCP call but fail result persistence
  mockMCPServer((request) => {
    if (request.params.name === 'convert_pdf') {
      return {
        result: {
          content: [{ type: 'document', document: mockDoclingDoc }],
        },
      };
    }
  });

  // Inject database constraint violation on result insert
  mockResultTableConstraintViolation();

  await expect(
    mcpClient.invoke('convert_pdf', { file_path: '/tmp/test.pdf', jobId })
  ).rejects.toThrow();

  // Verify job state rolled back to failed, not completed
  const job = await getJob(jobId);
  expect(job.status).toBe('failed');
  expect(job.error_code).toBe('E501');
  expect(job.error_message).toContain('persistence');

  // Verify no orphaned result
  const result = await getResultByJobId(jobId);
  expect(result).toBeNull();
});
```

**Expected Result**:
- Transaction rolled back on failure
- Job status set to failed with appropriate error
- No orphaned result records

---

### MCP-DB-004: Error Metadata Persistence

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-DB-004 |
| **Title** | Persist MCP Error Metadata to Database |
| **Priority** | P1 - High |
| **Spec Reference** | Section 7.1.4 (Error Handling) |

**Test Steps**:

```typescript
it('persists MCP error details to job record', async () => {
  const jobId = await createTestJob({ status: 'pending' });

  mockMCPServer(() => ({
    error: {
      code: -32002,
      message: 'Document conversion failed: corrupt PDF structure'
    },
  }));

  await expect(
    mcpClient.invoke('convert_pdf', { file_path: '/tmp/corrupt.pdf', jobId })
  ).rejects.toThrow();

  const job = await getJob(jobId);

  expect(job.status).toBe('failed');
  expect(job.error_code).toBe('E302');
  expect(job.error_message).toContain('conversion failed');
  expect(job.error_metadata).toMatchObject({
    mcp_code: -32002,
    mcp_message: expect.stringContaining('corrupt PDF'),
    retryable: true,
    retry_count: 0,
  });
});
```

**Expected Result**:
- Error code mapped and persisted
- Original MCP error details preserved in metadata
- Retry information captured

---

### MCP-DB-005: Connection Pool Stability Under Load

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-DB-005 |
| **Title** | Database Connection Pool Handles Concurrent MCP Calls |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 4.7 (Resource Management) |

**Test Steps**:

```typescript
it('maintains connection pool stability under concurrent load', async () => {
  const concurrentJobs = 50;
  const jobs = await Promise.all(
    Array(concurrentJobs).fill(null).map(() =>
      createTestJob({ status: 'pending' })
    )
  );

  mockMCPServer((request) => {
    // Simulate variable processing time
    return new Promise((resolve) => {
      setTimeout(() => {
        resolve({
          result: { content: [{ type: 'document', document: mockDoclingDoc }] },
        });
      }, Math.random() * 100);
    });
  });

  // Execute all MCP calls concurrently
  const results = await Promise.allSettled(
    jobs.map(jobId =>
      mcpClient.invoke('convert_pdf', { file_path: '/tmp/test.pdf', jobId })
    )
  );

  // Verify all completed successfully
  const fulfilled = results.filter(r => r.status === 'fulfilled');
  expect(fulfilled.length).toBe(concurrentJobs);

  // Verify connection pool metrics
  const poolStats = await getConnectionPoolStats();
  expect(poolStats.active).toBe(0); // All connections returned
  expect(poolStats.waiting).toBe(0); // No waiting requests
  expect(poolStats.errors).toBe(0); // No connection errors
});
```

**Expected Result**:
- All concurrent operations complete successfully
- Connection pool properly manages connections
- No connection leaks or exhaustion

---

### MCP-DB-006: Connection Release on Timeout

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-DB-006 |
| **Title** | Database Connection Released on MCP Timeout |
| **Priority** | P1 - High |
| **Spec Reference** | Section 4.7 (Resource Cleanup) |

**Test Steps**:

```typescript
it('releases database connection when MCP call times out', async () => {
  const jobId = await createTestJob({ status: 'pending' });
  const initialPoolSize = (await getConnectionPoolStats()).available;

  mockMCPServer(() => {
    // Never respond - simulate timeout
    return new Promise(() => {});
  });

  const client = new MCPClient({ defaultTimeout: 100 });

  await expect(
    client.invoke('convert_pdf', { file_path: '/tmp/test.pdf', jobId })
  ).rejects.toMatchObject({ code: 'E202' });

  // Wait for cleanup
  await new Promise(resolve => setTimeout(resolve, 50));

  // Verify connection returned to pool
  const poolStats = await getConnectionPoolStats();
  expect(poolStats.available).toBe(initialPoolSize);

  // Verify job marked as timed out
  const job = await getJob(jobId);
  expect(job.status).toBe('failed');
  expect(job.error_code).toBe('E202');
});
```

**Expected Result**:
- Database connection released despite timeout
- Connection pool size unchanged
- Job state properly updated to failed

---

### MCP-DB-007: Browser Refresh Recovery - Job State

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-DB-007 |
| **Title** | Recover Job State After Browser Refresh |
| **Priority** | P0 - Critical |
| **Spec Reference** | CRI-10 (Browser Refresh Recovery) |

**Test Steps**:

```typescript
it('recovers job state from database after browser refresh', async () => {
  const jobId = await createTestJob({ status: 'processing' });

  // Simulate job in progress with partial results
  await updateJob(jobId, {
    progress: 45,
    started_at: new Date(Date.now() - 30000),
    last_progress_at: new Date(),
  });

  // Simulate browser refresh - create new session
  const newSession = await createSession();

  // Recover job state
  const recoveredJob = await newSession.getJob(jobId);

  expect(recoveredJob.status).toBe('processing');
  expect(recoveredJob.progress).toBe(45);
  expect(recoveredJob.started_at).toBeDefined();
  expect(recoveredJob.session_id).not.toBe(newSession.id); // Original session

  // Verify job can be resumed or status checked
  expect(recoveredJob.isResumable()).toBe(true);
});
```

**Expected Result**:
- Job state retrieved from database
- Progress information preserved
- Job resumable in new session

---

### MCP-DB-008: Browser Refresh Recovery - Result Retrieval

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-DB-008 |
| **Title** | Retrieve Completed Results After Browser Refresh |
| **Priority** | P0 - Critical |
| **Spec Reference** | CRI-10 (Browser Refresh Recovery) |

**Test Steps**:

```typescript
it('retrieves completed results after browser refresh', async () => {
  const jobId = await createTestJob({ status: 'completed' });

  // Create completed result
  await createResult(jobId, {
    document: mockDoclingDoc,
    exports: {
      markdown: '# Document Content',
      json: JSON.stringify(mockDoclingDoc),
    },
    created_at: new Date(),
  });

  // Simulate browser refresh - create new session
  const newSession = await createSession();

  // Retrieve results
  const result = await newSession.getResultByJobId(jobId);

  expect(result).toBeDefined();
  expect(result.document.schema_name).toBe('docling_core.transforms.v1.DoclingDocument');
  expect(result.exports.markdown).toContain('# Document');
  expect(result.exports.json).toBeDefined();
});
```

**Expected Result**:
- Completed results retrievable after refresh
- All export formats accessible
- No data loss on browser refresh

---

## 9. Redis Integration Tests

### 9.1 SSE Event Buffering

---

### MCP-REDIS-001: Progress Events Buffered

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-REDIS-001 |
| **Title** | Buffer MCP Progress Events in Redis |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 6.2 (SSE Event Buffering) |

**Test Steps**:

```typescript
it('buffers progress events in Redis during MCP call', async () => {
  const jobId = await createTestJob({ status: 'pending' });
  const progressEvents: ProgressEvent[] = [];

  // Mock MCP server with progress notifications
  mockMCPServer((request) => {
    if (request.params.name === 'convert_pdf') {
      // Emit progress events
      emitProgress(jobId, { percent: 25, stage: 'parsing' });
      emitProgress(jobId, { percent: 50, stage: 'analyzing' });
      emitProgress(jobId, { percent: 75, stage: 'converting' });

      return {
        result: { content: [{ type: 'document', document: mockDoclingDoc }] },
      };
    }
  });

  await mcpClient.invoke('convert_pdf', { file_path: '/tmp/test.pdf', jobId });

  // Verify events buffered in Redis
  const bufferedEvents = await redis.lrange(`job:${jobId}:events`, 0, -1);

  expect(bufferedEvents.length).toBeGreaterThanOrEqual(3);

  const parsedEvents = bufferedEvents.map(e => JSON.parse(e));
  expect(parsedEvents[0]).toMatchObject({ percent: 25, stage: 'parsing' });
  expect(parsedEvents[1]).toMatchObject({ percent: 50, stage: 'analyzing' });
  expect(parsedEvents[2]).toMatchObject({ percent: 75, stage: 'converting' });
});
```

**Expected Result**:
- All progress events buffered in Redis list
- Events maintain order
- Events parseable as JSON

---

### MCP-REDIS-002: Event Buffer Respects maxEvents

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-REDIS-002 |
| **Title** | Event Buffer Enforces Maximum Size |
| **Priority** | P1 - High |
| **Spec Reference** | Section 6.2 (Buffer Limits) |

**Test Steps**:

```typescript
it('enforces maxEvents limit on event buffer', async () => {
  const jobId = await createTestJob({ status: 'pending' });
  const maxEvents = 100;

  // Emit more events than buffer allows
  for (let i = 0; i < 150; i++) {
    await emitProgress(jobId, { percent: i, stage: 'processing' });
  }

  // Verify buffer trimmed to maxEvents
  const bufferedEvents = await redis.lrange(`job:${jobId}:events`, 0, -1);

  expect(bufferedEvents.length).toBe(maxEvents);

  // Verify oldest events removed (FIFO)
  const firstEvent = JSON.parse(bufferedEvents[0]);
  expect(firstEvent.percent).toBe(50); // Events 0-49 should be trimmed
});
```

**Expected Result**:
- Buffer never exceeds maxEvents
- Oldest events removed first (FIFO)
- Most recent events preserved

---

### MCP-REDIS-003: Event Buffer TTL Enforcement

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-REDIS-003 |
| **Title** | Event Buffer Expires After TTL |
| **Priority** | P1 - High |
| **Spec Reference** | Section 6.2 (Buffer TTL) |

**Test Steps**:

```typescript
it('expires event buffer after configured TTL', async () => {
  const jobId = await createTestJob({ status: 'pending' });
  const bufferTTL = 2; // 2 seconds for test

  // Set short TTL for test
  await redis.setex(`job:${jobId}:events:ttl`, bufferTTL, 'active');
  await emitProgress(jobId, { percent: 50, stage: 'processing' });

  // Verify events exist
  let bufferedEvents = await redis.lrange(`job:${jobId}:events`, 0, -1);
  expect(bufferedEvents.length).toBe(1);

  // Wait for TTL expiration
  await new Promise(resolve => setTimeout(resolve, (bufferTTL + 1) * 1000));

  // Verify buffer expired
  bufferedEvents = await redis.lrange(`job:${jobId}:events`, 0, -1);
  expect(bufferedEvents.length).toBe(0);
});
```

**Expected Result**:
- Event buffer automatically expires
- No stale data accumulation
- TTL configurable per job

---

### MCP-REDIS-004: Session State Coordination

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-REDIS-004 |
| **Title** | Coordinate Session State Across MCP Calls |
| **Priority** | P1 - High |
| **Spec Reference** | Section 6.3 (Session Management) |

**Test Steps**:

```typescript
it('maintains session state across multiple MCP calls', async () => {
  const sessionId = await createSession();
  const jobs: string[] = [];

  // Create multiple jobs in same session
  for (let i = 0; i < 3; i++) {
    const jobId = await createTestJob({
      status: 'pending',
      session_id: sessionId
    });
    jobs.push(jobId);
  }

  // Verify session tracks all jobs
  const sessionData = await redis.hgetall(`session:${sessionId}`);
  expect(JSON.parse(sessionData.jobs)).toEqual(jobs);
  expect(sessionData.created_at).toBeDefined();
  expect(sessionData.last_activity).toBeDefined();

  // Execute MCP calls
  await Promise.all(
    jobs.map(jobId =>
      mcpClient.invoke('convert_pdf', { file_path: '/tmp/test.pdf', jobId })
    )
  );

  // Verify session updated
  const updatedSession = await redis.hgetall(`session:${sessionId}`);
  expect(new Date(updatedSession.last_activity).getTime())
    .toBeGreaterThan(new Date(sessionData.last_activity).getTime());
});
```

**Expected Result**:
- Session tracks all associated jobs
- Session activity timestamp updated
- Session data accessible across requests

---

### MCP-RATE-001: Rate Limit Checked Before MCP Call

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-RATE-001 |
| **Title** | Enforce Rate Limit Before MCP Invocation |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 4.5 (Rate Limiting) |

**Test Steps**:

```typescript
it('checks rate limit before invoking MCP tool', async () => {
  const userId = 'test-user-001';
  let mcpCallCount = 0;

  mockMCPServer(() => {
    mcpCallCount++;
    return {
      result: { content: [{ type: 'document', document: mockDoclingDoc }] },
    };
  });

  // Exhaust rate limit
  await redis.set(`ratelimit:${userId}:mcp`, 10); // Max 10 calls
  await redis.expire(`ratelimit:${userId}:mcp`, 60);

  // Attempt call when at limit
  await expect(
    mcpClient.invoke('convert_pdf', {
      file_path: '/tmp/test.pdf',
      userId
    })
  ).rejects.toMatchObject({
    code: 'E601',
    message: expect.stringContaining('rate limit'),
  });

  // Verify MCP server NOT called
  expect(mcpCallCount).toBe(0);
});
```

**Expected Result**:
- Rate limit checked BEFORE MCP invocation
- E601 error returned when limit exceeded
- No MCP server resources consumed

---

### MCP-RATE-002: Rate Limit Exceeded Returns E601

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-RATE-002 |
| **Title** | Rate Limit Error Response Format |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 7.1.4 (Error Codes) |

**Test Steps**:

```typescript
it('returns properly formatted E601 error on rate limit', async () => {
  const userId = 'test-user-002';

  // Set rate limit exceeded
  await redis.set(`ratelimit:${userId}:mcp`, 100);
  await redis.set(`ratelimit:${userId}:mcp:limit`, 10);
  await redis.expire(`ratelimit:${userId}:mcp`, 60);

  try {
    await mcpClient.invoke('convert_pdf', {
      file_path: '/tmp/test.pdf',
      userId
    });
    fail('Expected rate limit error');
  } catch (error) {
    expect(error.code).toBe('E601');
    expect(error.retryable).toBe(true);
    expect(error.retryAfter).toBeDefined();
    expect(error.retryAfter).toBeGreaterThan(0);
    expect(error.userMessage).toContain('Too many requests');
    expect(error.metadata).toMatchObject({
      limit: 10,
      current: 100,
      resetAt: expect.any(String),
    });
  }
});
```

**Expected Result**:
- Error includes retry-after header
- Error includes rate limit details
- Error is marked as retryable

---

### MCP-RATE-003: Rate Limit Counter Increment

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-RATE-003 |
| **Title** | Rate Limit Counter Incremented on MCP Call |
| **Priority** | P1 - High |
| **Spec Reference** | Section 4.5 (Rate Limiting) |

**Test Steps**:

```typescript
it('increments rate limit counter on successful MCP call', async () => {
  const userId = 'test-user-003';

  mockMCPServer(() => ({
    result: { content: [{ type: 'document', document: mockDoclingDoc }] },
  }));

  // Initial counter state
  const initialCount = await redis.get(`ratelimit:${userId}:mcp`) || '0';

  await mcpClient.invoke('convert_pdf', {
    file_path: '/tmp/test.pdf',
    userId
  });

  // Verify counter incremented
  const newCount = await redis.get(`ratelimit:${userId}:mcp`);
  expect(parseInt(newCount)).toBe(parseInt(initialCount) + 1);
});
```

**Expected Result**:
- Counter incremented atomically
- Counter expires with rate limit window
- Counter tracks per-user usage

---

### MCP-CB-001: Circuit Breaker Opens on Failures

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-CB-001 |
| **Title** | Circuit Breaker Opens After Consecutive Failures |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 4.6 (Circuit Breaker Pattern) |

**Test Steps**:

```typescript
it('opens circuit breaker after consecutive MCP failures', async () => {
  const failureThreshold = 5;
  let callCount = 0;

  mockMCPServer(() => {
    callCount++;
    return { error: { code: -32603, message: 'Internal error' } };
  });

  // Trigger failures up to threshold
  for (let i = 0; i < failureThreshold; i++) {
    await expect(
      mcpClient.invoke('convert_pdf', { file_path: '/tmp/test.pdf' })
    ).rejects.toThrow();
  }

  // Verify circuit is open
  const circuitState = await redis.get('circuit:mcp:state');
  expect(circuitState).toBe('open');

  // Next call should fail fast without calling MCP
  const callCountBefore = callCount;

  await expect(
    mcpClient.invoke('convert_pdf', { file_path: '/tmp/test.pdf' })
  ).rejects.toMatchObject({
    code: 'E201',
    message: expect.stringContaining('circuit open'),
  });

  expect(callCount).toBe(callCountBefore); // No additional MCP call
});
```

**Expected Result**:
- Circuit opens after failure threshold
- Subsequent calls fail fast
- MCP server not called when circuit open

---

### MCP-CB-002: Circuit Breaker Half-Open State

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-CB-002 |
| **Title** | Circuit Breaker Transitions to Half-Open |
| **Priority** | P1 - High |
| **Spec Reference** | Section 4.6 (Circuit Breaker Pattern) |

**Test Steps**:

```typescript
it('transitions to half-open after cooldown period', async () => {
  const cooldownMs = 1000;

  // Set circuit to open state
  await redis.set('circuit:mcp:state', 'open');
  await redis.set('circuit:mcp:opened_at', Date.now().toString());
  await redis.set('circuit:mcp:cooldown', cooldownMs.toString());

  // Wait for cooldown
  await new Promise(resolve => setTimeout(resolve, cooldownMs + 100));

  // Next call should be allowed (half-open test)
  mockMCPServer(() => ({
    result: { content: [{ type: 'document', document: mockDoclingDoc }] },
  }));

  const result = await mcpClient.invoke('convert_pdf', {
    file_path: '/tmp/test.pdf'
  });

  expect(result).toBeDefined();

  // Verify circuit closed after success
  const circuitState = await redis.get('circuit:mcp:state');
  expect(circuitState).toBe('closed');
});
```

**Expected Result**:
- Circuit transitions to half-open after cooldown
- Single request allowed for testing
- Success closes circuit

---

### MCP-CB-003: Circuit Breaker Failure Count Reset

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-CB-003 |
| **Title** | Reset Failure Count on Successful MCP Call |
| **Priority** | P1 - High |
| **Spec Reference** | Section 4.6 (Circuit Breaker Pattern) |

**Test Steps**:

```typescript
it('resets failure count on successful MCP call', async () => {
  // Accumulate some failures (but below threshold)
  await redis.set('circuit:mcp:failures', '3');

  mockMCPServer(() => ({
    result: { content: [{ type: 'document', document: mockDoclingDoc }] },
  }));

  await mcpClient.invoke('convert_pdf', { file_path: '/tmp/test.pdf' });

  // Verify failure count reset
  const failureCount = await redis.get('circuit:mcp:failures');
  expect(parseInt(failureCount || '0')).toBe(0);
});
```

**Expected Result**:
- Failure count reset to zero
- Circuit remains closed
- Next failures start fresh count

---

### MCP-CB-004: Circuit Breaker Metrics Tracking

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-CB-004 |
| **Title** | Track Circuit Breaker Metrics |
| **Priority** | P2 - Medium |
| **Spec Reference** | Section 4.6 (Observability) |

**Test Steps**:

```typescript
it('tracks circuit breaker metrics in Redis', async () => {
  mockMCPServer(() => ({
    result: { content: [{ type: 'document', document: mockDoclingDoc }] },
  }));

  // Make several calls
  for (let i = 0; i < 5; i++) {
    await mcpClient.invoke('convert_pdf', { file_path: '/tmp/test.pdf' });
  }

  // Verify metrics tracked
  const metrics = await redis.hgetall('circuit:mcp:metrics');

  expect(parseInt(metrics.total_requests)).toBeGreaterThanOrEqual(5);
  expect(parseInt(metrics.successful_requests)).toBeGreaterThanOrEqual(5);
  expect(parseInt(metrics.failed_requests)).toBe(0);
  expect(metrics.last_success).toBeDefined();
});
```

**Expected Result**:
- Request counts tracked
- Success/failure breakdown available
- Timestamps recorded

---

### MCP-CB-005: Graceful Degradation When Redis Unavailable

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-CB-005 |
| **Title** | MCP Calls Succeed When Redis Unavailable |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 4.6 (Graceful Degradation) |

**Test Steps**:

```typescript
it('allows MCP calls when Redis is unavailable', async () => {
  // Simulate Redis connection failure
  await redis.disconnect();

  mockMCPServer(() => ({
    result: { content: [{ type: 'document', document: mockDoclingDoc }] },
  }));

  // MCP call should still succeed (without rate limiting/circuit breaker)
  const result = await mcpClient.invoke('convert_pdf', {
    file_path: '/tmp/test.pdf'
  });

  expect(result).toBeDefined();
  expect(result.content[0].document).toBeDefined();

  // Verify warning logged
  expect(mockLogger.warn).toHaveBeenCalledWith(
    expect.stringContaining('Redis unavailable'),
    expect.objectContaining({ fallback: 'allow-all' })
  );
});
```

**Expected Result**:
- MCP calls succeed without Redis
- Warning logged for observability
- Rate limiting bypassed (documented risk)

---

## 10. Frontend Integration Tests

### 10.1 UI Loading States

---

### MCP-UI-001: Loading State Transitions

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-UI-001 |
| **Title** | UI Loading State Transitions During MCP Call |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-301 (Loading States) |

**Test Steps**:

```typescript
it('transitions through loading states during MCP call', async () => {
  const { getByTestId, queryByTestId } = render(<ConversionPanel />);

  mockMCPServer(() => {
    return new Promise((resolve) => {
      setTimeout(() => {
        resolve({
          result: { content: [{ type: 'document', document: mockDoclingDoc }] },
        });
      }, 100);
    });
  });

  // Initial state - idle
  expect(queryByTestId('loading-spinner')).toBeNull();
  expect(getByTestId('convert-button')).not.toBeDisabled();

  // Trigger conversion
  fireEvent.click(getByTestId('convert-button'));

  // Loading state
  await waitFor(() => {
    expect(getByTestId('loading-spinner')).toBeVisible();
    expect(getByTestId('convert-button')).toBeDisabled();
    expect(getByTestId('status-text')).toHaveTextContent('Converting...');
  });

  // Completed state
  await waitFor(() => {
    expect(queryByTestId('loading-spinner')).toBeNull();
    expect(getByTestId('convert-button')).not.toBeDisabled();
    expect(getByTestId('status-text')).toHaveTextContent('Conversion complete');
  });
});
```

**Expected Result**:
- Spinner visible during processing
- Button disabled during processing
- Status text updates appropriately

---

### MCP-UI-002: Button Disabled During Processing

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-UI-002 |
| **Title** | Convert Button Disabled During MCP Processing |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-301 (UI Responsiveness) |

**Test Steps**:

```typescript
it('prevents duplicate submissions during MCP processing', async () => {
  let mcpCallCount = 0;
  const { getByTestId } = render(<ConversionPanel />);

  mockMCPServer(() => {
    mcpCallCount++;
    return new Promise((resolve) => {
      setTimeout(() => {
        resolve({
          result: { content: [{ type: 'document', document: mockDoclingDoc }] },
        });
      }, 200);
    });
  });

  const button = getByTestId('convert-button');

  // Click multiple times rapidly
  fireEvent.click(button);
  fireEvent.click(button);
  fireEvent.click(button);

  await waitFor(() => {
    expect(button).toBeDisabled();
  });

  // Wait for completion
  await waitFor(() => {
    expect(button).not.toBeDisabled();
  }, { timeout: 500 });

  // Verify only one MCP call made
  expect(mcpCallCount).toBe(1);
});
```

**Expected Result**:
- Only one MCP call despite multiple clicks
- Button visually disabled
- No duplicate job submissions

---

### MCP-UI-003: Progress Bar Updates

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-UI-003 |
| **Title** | Progress Bar Reflects MCP Progress Events |
| **Priority** | P1 - High |
| **Spec Reference** | FR-302 (Progress Indication) |

**Test Steps**:

```typescript
it('updates progress bar from MCP progress events', async () => {
  const { getByTestId, getByRole } = render(<ConversionPanel />);

  mockMCPServer((request) => {
    if (request.params.name === 'convert_pdf') {
      // Emit progress events via SSE mock
      emitSSEProgress({ percent: 25, stage: 'Parsing document' });
      emitSSEProgress({ percent: 50, stage: 'Analyzing content' });
      emitSSEProgress({ percent: 75, stage: 'Generating output' });

      return {
        result: { content: [{ type: 'document', document: mockDoclingDoc }] },
      };
    }
  });

  fireEvent.click(getByTestId('convert-button'));

  // Verify progress bar updates
  await waitFor(() => {
    const progressBar = getByRole('progressbar');
    expect(progressBar).toHaveAttribute('aria-valuenow', '25');
  });

  await waitFor(() => {
    const progressBar = getByRole('progressbar');
    expect(progressBar).toHaveAttribute('aria-valuenow', '50');
  });

  await waitFor(() => {
    const progressBar = getByRole('progressbar');
    expect(progressBar).toHaveAttribute('aria-valuenow', '75');
  });

  // Verify completion
  await waitFor(() => {
    const progressBar = getByRole('progressbar');
    expect(progressBar).toHaveAttribute('aria-valuenow', '100');
  });
});
```

**Expected Result**:
- Progress bar reflects SSE events
- Progress bar accessible (ARIA attributes)
- Progress reaches 100% on completion

---

### MCP-UI-004: Stage Label Display

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-UI-004 |
| **Title** | Display Current Processing Stage |
| **Priority** | P1 - High |
| **Spec Reference** | FR-302 (Progress Indication) |

**Test Steps**:

```typescript
it('displays current processing stage label', async () => {
  const { getByTestId } = render(<ConversionPanel />);

  mockMCPServer((request) => {
    if (request.params.name === 'convert_pdf') {
      emitSSEProgress({ percent: 25, stage: 'Parsing document structure' });
      emitSSEProgress({ percent: 50, stage: 'Extracting text content' });
      emitSSEProgress({ percent: 75, stage: 'Generating DoclingDocument' });

      return {
        result: { content: [{ type: 'document', document: mockDoclingDoc }] },
      };
    }
  });

  fireEvent.click(getByTestId('convert-button'));

  await waitFor(() => {
    expect(getByTestId('stage-label')).toHaveTextContent('Parsing document structure');
  });

  await waitFor(() => {
    expect(getByTestId('stage-label')).toHaveTextContent('Extracting text content');
  });

  await waitFor(() => {
    expect(getByTestId('stage-label')).toHaveTextContent('Generating DoclingDocument');
  });
});
```

**Expected Result**:
- Stage label updates with each progress event
- Label readable and informative
- Final stage shown before completion

---

### MCP-PROGRESS-001: Progress Events During MCP Call

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-PROGRESS-001 |
| **Title** | SSE Progress Events Received During MCP Call |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 6.2 (SSE Integration) |

**Test Steps**:

```typescript
it('receives SSE progress events during MCP processing', async () => {
  const progressEvents: ProgressEvent[] = [];
  const { getByTestId } = render(
    <ConversionPanel onProgress={(e) => progressEvents.push(e)} />
  );

  mockMCPServer((request) => {
    if (request.params.name === 'convert_pdf') {
      // Simulate server sending progress via SSE
      emitSSEProgress({ percent: 10, stage: 'Starting' });
      emitSSEProgress({ percent: 30, stage: 'Processing' });
      emitSSEProgress({ percent: 60, stage: 'Analyzing' });
      emitSSEProgress({ percent: 90, stage: 'Finalizing' });

      return {
        result: { content: [{ type: 'document', document: mockDoclingDoc }] },
      };
    }
  });

  fireEvent.click(getByTestId('convert-button'));

  await waitFor(() => {
    expect(progressEvents.length).toBeGreaterThanOrEqual(4);
  });

  expect(progressEvents[0]).toMatchObject({ percent: 10, stage: 'Starting' });
  expect(progressEvents[progressEvents.length - 1]).toMatchObject({
    percent: 90,
    stage: 'Finalizing'
  });
});
```

**Expected Result**:
- All progress events received
- Events in correct order
- Events contain percent and stage

---

### MCP-ERROR-UI-001: Error Boundary Renders for MCP Errors

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-ERROR-UI-001 |
| **Title** | Error Boundary Displays MCP Errors |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-303 (Error Display) |

**Test Steps**:

```typescript
it('displays error boundary for MCP server errors', async () => {
  const { getByTestId, getByRole } = render(<ConversionPanel />);

  mockMCPServer(() => ({
    error: { code: -32603, message: 'Internal server error' },
  }));

  fireEvent.click(getByTestId('convert-button'));

  await waitFor(() => {
    const errorBoundary = getByRole('alert');
    expect(errorBoundary).toBeVisible();
    expect(errorBoundary).toHaveTextContent('Service temporarily unavailable');
  });

  // Verify retry button present
  expect(getByTestId('retry-button')).toBeVisible();
});
```

**Expected Result**:
- Error boundary displays user-friendly message
- Technical details hidden from user
- Retry option available

---

### MCP-ERROR-UI-002: Error Message Formatting

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-ERROR-UI-002 |
| **Title** | Format MCP Errors for User Display |
| **Priority** | P1 - High |
| **Spec Reference** | Section 7.1.4 (User Messages) |

**Test Steps**:

```typescript
describe('Error Message Formatting', () => {
  const errorCases = [
    { mcpCode: -32700, expectedMessage: 'Invalid request format' },
    { mcpCode: -32601, expectedMessage: 'Requested operation not available' },
    { mcpCode: -32602, expectedMessage: 'Invalid file or parameters' },
    { mcpCode: -32603, expectedMessage: 'Service temporarily unavailable' },
    { mcpCode: -32002, expectedMessage: 'Document conversion failed' },
    { mcpCode: -32003, expectedMessage: 'Export generation failed' },
  ];

  errorCases.forEach(({ mcpCode, expectedMessage }) => {
    it(`displays "${expectedMessage}" for MCP code ${mcpCode}`, async () => {
      const { getByTestId, getByRole } = render(<ConversionPanel />);

      mockMCPServer(() => ({
        error: { code: mcpCode, message: 'Technical error message' },
      }));

      fireEvent.click(getByTestId('convert-button'));

      await waitFor(() => {
        const errorBoundary = getByRole('alert');
        expect(errorBoundary).toHaveTextContent(expectedMessage);
      });
    });
  });
});
```

**Expected Result**:
- Each error code maps to user-friendly message
- Technical messages not exposed
- Messages are actionable

---

### MCP-ERROR-UI-003: Retryable Error Indication

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-ERROR-UI-003 |
| **Title** | Indicate Retryable vs Non-Retryable Errors |
| **Priority** | P1 - High |
| **Spec Reference** | FR-803 (Retry Logic) |

**Test Steps**:

```typescript
it('shows retry button only for retryable errors', async () => {
  const { getByTestId, queryByTestId, rerender } = render(<ConversionPanel />);

  // Test retryable error (E201)
  mockMCPServer(() => ({
    error: { code: -32603, message: 'Internal error' },
  }));

  fireEvent.click(getByTestId('convert-button'));

  await waitFor(() => {
    expect(getByTestId('retry-button')).toBeVisible();
  });

  // Reset and test non-retryable error (E204)
  rerender(<ConversionPanel />);

  mockMCPServer(() => ({
    error: { code: -32601, message: 'Method not found' },
  }));

  fireEvent.click(getByTestId('convert-button'));

  await waitFor(() => {
    expect(queryByTestId('retry-button')).toBeNull();
    expect(getByTestId('contact-support-link')).toBeVisible();
  });
});
```

**Expected Result**:
- Retry button for retryable errors
- Support link for non-retryable errors
- Clear distinction in UI

---

### MCP-RENDER-001: DoclingDocument Renders in Components

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-RENDER-001 |
| **Title** | Render DoclingDocument Content in UI |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-401 (Document Display) |

**Test Steps**:

```typescript
it('renders DoclingDocument content correctly', async () => {
  const docWithContent = {
    ...mockDoclingDoc,
    body: {
      self_ref: '#/body',
      children: [
        { type: 'heading', level: 1, text: 'Document Title' },
        { type: 'paragraph', text: 'This is the first paragraph.' },
        { type: 'heading', level: 2, text: 'Section Header' },
        { type: 'paragraph', text: 'Section content here.' },
      ],
    },
  };

  const { getByTestId, getByRole } = render(<ConversionPanel />);

  mockMCPServer(() => ({
    result: { content: [{ type: 'document', document: docWithContent }] },
  }));

  fireEvent.click(getByTestId('convert-button'));

  await waitFor(() => {
    const preview = getByTestId('document-preview');

    expect(getByRole('heading', { level: 1 })).toHaveTextContent('Document Title');
    expect(getByRole('heading', { level: 2 })).toHaveTextContent('Section Header');
    expect(preview).toHaveTextContent('This is the first paragraph.');
    expect(preview).toHaveTextContent('Section content here.');
  });
});
```

**Expected Result**:
- Headings rendered with correct levels
- Paragraphs displayed
- Document structure preserved

---

### MCP-RENDER-002: Export Format Selection

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-RENDER-002 |
| **Title** | Select and Display Export Formats |
| **Priority** | P1 - High |
| **Spec Reference** | FR-404 (Export Selection) |

**Test Steps**:

```typescript
it('allows selection of export format after conversion', async () => {
  const { getByTestId, getByRole } = render(<ConversionPanel />);

  mockMCPServer((request) => {
    if (request.params.name === 'convert_pdf') {
      return {
        result: { content: [{ type: 'document', document: mockDoclingDoc }] },
      };
    }
    if (request.params.name === 'export_markdown') {
      return {
        result: { content: [{ type: 'text', text: '# Exported Markdown' }] },
      };
    }
    if (request.params.name === 'export_json') {
      return {
        result: { content: [{ type: 'text', text: '{"exported": true}' }] },
      };
    }
  });

  // Complete conversion
  fireEvent.click(getByTestId('convert-button'));

  await waitFor(() => {
    expect(getByTestId('export-selector')).toBeVisible();
  });

  // Select Markdown export
  fireEvent.click(getByRole('tab', { name: 'Markdown' }));

  await waitFor(() => {
    expect(getByTestId('export-content')).toHaveTextContent('# Exported Markdown');
  });

  // Select JSON export
  fireEvent.click(getByRole('tab', { name: 'JSON' }));

  await waitFor(() => {
    expect(getByTestId('export-content')).toHaveTextContent('"exported": true');
  });
});
```

**Expected Result**:
- Export format selector visible after conversion
- Tab switching triggers export call
- Correct content displayed per format

---

### MCP-CANCEL-UI-001: Cancel Button Aborts Operation

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-CANCEL-UI-001 |
| **Title** | Cancel Button Aborts MCP Operation |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-406 (Cancellation) |

**Test Steps**:

```typescript
it('cancels MCP operation when cancel button clicked', async () => {
  const { getByTestId, queryByTestId } = render(<ConversionPanel />);
  let abortSignalAborted = false;

  mockMCPServer((request, { signal }) => {
    signal.addEventListener('abort', () => {
      abortSignalAborted = true;
    });

    return new Promise((resolve) => {
      setTimeout(() => {
        if (!signal.aborted) {
          resolve({
            result: { content: [{ type: 'document', document: mockDoclingDoc }] },
          });
        }
      }, 5000);
    });
  });

  // Start conversion
  fireEvent.click(getByTestId('convert-button'));

  await waitFor(() => {
    expect(getByTestId('cancel-button')).toBeVisible();
  });

  // Click cancel
  fireEvent.click(getByTestId('cancel-button'));

  await waitFor(() => {
    // Verify operation cancelled
    expect(abortSignalAborted).toBe(true);

    // Verify UI reset to idle
    expect(queryByTestId('loading-spinner')).toBeNull();
    expect(getByTestId('convert-button')).not.toBeDisabled();
    expect(getByTestId('status-text')).toHaveTextContent('Cancelled');
  });
});
```

**Expected Result**:
- Abort signal triggered
- UI returns to idle state
- Cancelled status displayed

---

### MCP-CANCEL-UI-002: Cancel During Retry Backoff

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-CANCEL-UI-002 |
| **Title** | Cancel During Retry Backoff Period |
| **Priority** | P1 - High |
| **Spec Reference** | FR-406 (Cancellation During Retry) |

**Test Steps**:

```typescript
it('cancels during retry backoff period', async () => {
  const { getByTestId, queryByTestId } = render(<ConversionPanel />);
  let callCount = 0;

  mockMCPServer(() => {
    callCount++;
    return { error: { code: -32603, message: 'Internal error' } };
  });

  // Start conversion (will fail and enter retry)
  fireEvent.click(getByTestId('convert-button'));

  // Wait for first failure and retry message
  await waitFor(() => {
    expect(getByTestId('status-text')).toHaveTextContent('Retrying');
  });

  // Cancel during backoff
  fireEvent.click(getByTestId('cancel-button'));

  await waitFor(() => {
    // Verify no more retries happened
    const finalCallCount = callCount;
    expect(queryByTestId('loading-spinner')).toBeNull();
    expect(getByTestId('status-text')).toHaveTextContent('Cancelled');

    // Wait and verify no additional calls
    return new Promise((resolve) => {
      setTimeout(() => {
        expect(callCount).toBe(finalCallCount);
        resolve(true);
      }, 2000);
    });
  });
});
```

**Expected Result**:
- Retry loop terminated
- No additional MCP calls after cancel
- UI shows cancelled state

---

## 11. Infrastructure & Operations Tests

### 11.1 Health Check Integration

---

### MCP-HEALTH-001: MCP Server Health Check Success

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-HEALTH-001 |
| **Title** | Verify MCP Server Health Check Returns Healthy |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 4.7 (Health Checks) |

**Test Steps**:

```typescript
it('returns healthy status when MCP server is available', async () => {
  mockMCPServer((request) => {
    if (request.method === 'initialize') {
      return {
        result: {
          protocolVersion: '2024-11-05',
          serverInfo: { name: 'hx-docling-mcp-server', version: '1.0.0' },
          capabilities: { tools: {} },
        },
      };
    }
  });

  const healthCheck = await checkMCPHealth();

  expect(healthCheck.status).toBe('healthy');
  expect(healthCheck.mcp).toMatchObject({
    connected: true,
    serverVersion: '1.0.0',
    protocolVersion: '2024-11-05',
    latencyMs: expect.any(Number),
  });
  expect(healthCheck.mcp.latencyMs).toBeLessThan(1000);
});
```

**Expected Result**:
- Health check returns healthy status
- MCP server version and protocol version reported
- Latency within acceptable bounds

---

### MCP-HEALTH-002: MCP Server Health Check Failure

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-HEALTH-002 |
| **Title** | Health Check Reports Unhealthy When MCP Unavailable |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 4.7 (Health Checks) |

**Test Steps**:

```typescript
it('returns unhealthy status when MCP server unavailable', async () => {
  // Simulate MCP server down
  mockMCPServer.close();

  const healthCheck = await checkMCPHealth();

  expect(healthCheck.status).toBe('unhealthy');
  expect(healthCheck.mcp).toMatchObject({
    connected: false,
    error: expect.stringContaining('connection'),
    lastSuccessfulCheck: expect.any(String),
  });
});
```

**Expected Result**:
- Health check returns unhealthy status
- Error details provided
- Last successful check timestamp preserved

---

### MCP-HEALTH-003: Health Check Timeout

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-HEALTH-003 |
| **Title** | Health Check Enforces Timeout |
| **Priority** | P1 - High |
| **Spec Reference** | Section 4.7 (Health Checks) |

**Test Steps**:

```typescript
it('reports unhealthy when health check times out', async () => {
  mockMCPServer(() => {
    // Never respond - simulate hung server
    return new Promise(() => {});
  });

  const start = Date.now();
  const healthCheck = await checkMCPHealth({ timeout: 500 });
  const elapsed = Date.now() - start;

  expect(healthCheck.status).toBe('unhealthy');
  expect(healthCheck.mcp.error).toContain('timeout');
  expect(elapsed).toBeLessThan(1000); // Should not hang
});
```

**Expected Result**:
- Health check completes within timeout
- Unhealthy status reported
- Timeout error clearly indicated

---

### MCP-HEALTH-004: Health Check Integration with /api/v1/health

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-HEALTH-004 |
| **Title** | MCP Health Integrated in API Health Endpoint |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 4.7 (API Health Endpoint) |

**Test Steps**:

```typescript
it('includes MCP health in /api/v1/health response', async () => {
  mockMCPServer((request) => {
    if (request.method === 'initialize') {
      return {
        result: {
          serverInfo: { name: 'hx-docling-mcp-server', version: '1.0.0' },
          capabilities: { tools: {} },
        },
      };
    }
  });

  const response = await fetch('/api/v1/health');
  const health = await response.json();

  expect(response.status).toBe(200);
  expect(health).toMatchObject({
    status: 'healthy',
    components: {
      database: { status: 'healthy' },
      redis: { status: 'healthy' },
      mcp: {
        status: 'healthy',
        serverVersion: '1.0.0',
      },
    },
    timestamp: expect.any(String),
  });
});
```

**Expected Result**:
- API health endpoint includes MCP status
- All component statuses reported
- Overall status reflects MCP health

---

### 11.2 Connection Management

---

### MCP-CONN-001: Connection Pool Initialization

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-CONN-001 |
| **Title** | Initialize MCP Connection Pool at Startup |
| **Priority** | P1 - High |
| **Spec Reference** | Section 4.7 (Connection Management) |

**Test Steps**:

```typescript
it('initializes MCP connection pool at application startup', async () => {
  const app = await createTestApplication();

  const poolStats = await app.getMCPConnectionPoolStats();

  expect(poolStats).toMatchObject({
    minConnections: expect.any(Number),
    maxConnections: expect.any(Number),
    activeConnections: expect.any(Number),
    idleConnections: expect.any(Number),
    initialized: true,
  });
  expect(poolStats.minConnections).toBeGreaterThanOrEqual(1);
  expect(poolStats.maxConnections).toBeGreaterThanOrEqual(poolStats.minConnections);
});
```

**Expected Result**:
- Connection pool initialized at startup
- Minimum connections established
- Pool configuration accessible

---

### MCP-CONN-002: Connection Pool Exhaustion Handling

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-CONN-002 |
| **Title** | Handle Connection Pool Exhaustion Gracefully |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 4.7 (Resource Management) |

**Test Steps**:

```typescript
it('queues requests when connection pool exhausted', async () => {
  const maxConnections = 5;
  const app = await createTestApplication({ mcp: { maxConnections } });

  mockMCPServer(() => {
    return new Promise((resolve) => {
      setTimeout(() => {
        resolve({
          result: { content: [{ type: 'document', document: mockDoclingDoc }] },
        });
      }, 500);
    });
  });

  // Create more concurrent requests than pool allows
  const requests = Array(maxConnections + 3).fill(null).map(() =>
    app.mcpClient.invoke('convert_pdf', { file_path: '/tmp/test.pdf' })
  );

  const start = Date.now();
  const results = await Promise.allSettled(requests);
  const elapsed = Date.now() - start;

  // All should succeed (queued, not rejected)
  expect(results.every(r => r.status === 'fulfilled')).toBe(true);

  // Should take longer due to queuing (not all parallel)
  expect(elapsed).toBeGreaterThan(500);
});
```

**Expected Result**:
- Requests queued when pool exhausted
- All requests eventually succeed
- No connection errors thrown

---

### MCP-CONN-003: Connection Reuse

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-CONN-003 |
| **Title** | Reuse Connections for Multiple Requests |
| **Priority** | P1 - High |
| **Spec Reference** | Section 4.7 (Connection Efficiency) |

**Test Steps**:

```typescript
it('reuses connections for sequential requests', async () => {
  let connectionCount = 0;

  mockMCPServer((request) => {
    if (request.method === 'initialize') {
      connectionCount++;
    }
    return {
      result: { content: [{ type: 'document', document: mockDoclingDoc }] },
    };
  });

  // Make multiple sequential requests
  for (let i = 0; i < 10; i++) {
    await mcpClient.invoke('convert_pdf', { file_path: '/tmp/test.pdf' });
  }

  // Should reuse connections, not create new ones each time
  expect(connectionCount).toBeLessThan(10);
});
```

**Expected Result**:
- Connections reused across requests
- No unnecessary connection overhead
- Pool efficiency maintained

---

### MCP-CONN-004: Connection Timeout and Reconnection

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-CONN-004 |
| **Title** | Reconnect After Connection Timeout |
| **Priority** | P1 - High |
| **Spec Reference** | Section 4.7 (Connection Recovery) |

**Test Steps**:

```typescript
it('reconnects after connection timeout', async () => {
  let requestCount = 0;

  mockMCPServer((request) => {
    requestCount++;
    if (requestCount === 1) {
      // First request times out
      return new Promise(() => {});
    }
    return {
      result: { content: [{ type: 'document', document: mockDoclingDoc }] },
    };
  });

  // First request times out
  await expect(
    mcpClient.invoke('convert_pdf', { file_path: '/tmp/test.pdf' }, { timeout: 100 })
  ).rejects.toMatchObject({ code: 'E202' });

  // Second request should succeed with new connection
  const result = await mcpClient.invoke('convert_pdf', { file_path: '/tmp/test.pdf' });

  expect(result).toBeDefined();
  expect(result.content[0].document).toBeDefined();
});
```

**Expected Result**:
- Connection recovered after timeout
- Subsequent requests succeed
- No permanent connection failure

---

### MCP-CONN-005: Keep-Alive Connection Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-CONN-005 |
| **Title** | Validate Keep-Alive Connections Before Use |
| **Priority** | P1 - High |
| **Spec Reference** | Section 4.7 (Connection Validation) |

**Test Steps**:

```typescript
it('validates keep-alive connections before reuse', async () => {
  let initCount = 0;

  mockMCPServer((request) => {
    if (request.method === 'initialize') {
      initCount++;
    }
    return {
      result: { content: [{ type: 'document', document: mockDoclingDoc }] },
    };
  });

  // Make first request
  await mcpClient.invoke('convert_pdf', { file_path: '/tmp/test.pdf' });

  // Simulate server restart (connection becomes stale)
  mockMCPServer.restart();

  // Make second request - should detect stale connection and reconnect
  await mcpClient.invoke('convert_pdf', { file_path: '/tmp/test.pdf' });

  // Should have re-initialized after detecting stale connection
  expect(initCount).toBe(2);
});
```

**Expected Result**:
- Stale connections detected
- Automatic reconnection on stale connection
- Request succeeds after reconnection

---

### 11.3 Startup and Shutdown

---

### MCP-STARTUP-001: MCP Validation at Application Startup

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-STARTUP-001 |
| **Title** | Validate MCP Server Availability at Startup |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 4.7 (Startup Validation) |

**Test Steps**:

```typescript
it('validates MCP server during application startup', async () => {
  mockMCPServer((request) => {
    if (request.method === 'initialize') {
      return {
        result: {
          serverInfo: { name: 'hx-docling-mcp-server', version: '1.0.0' },
          capabilities: { tools: {} },
        },
      };
    }
    if (request.method === 'tools/list') {
      return {
        result: {
          tools: [
            { name: 'convert_pdf', inputSchema: {} },
            { name: 'export_markdown', inputSchema: {} },
          ],
        },
      };
    }
  });

  const app = await createTestApplication();

  expect(app.isReady()).toBe(true);
  expect(app.mcpClient.isInitialized()).toBe(true);
  expect(app.mcpClient.hasTool('convert_pdf')).toBe(true);
});
```

**Expected Result**:
- Application validates MCP at startup
- Tool availability confirmed
- Application marked ready only after MCP validated

---

### MCP-STARTUP-002: Fail Fast When MCP Unavailable

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-STARTUP-002 |
| **Title** | Application Fails Fast When MCP Unavailable |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 4.7 (Fail-Fast Startup) |

**Test Steps**:

```typescript
it('fails application startup when MCP server unavailable', async () => {
  // MCP server not running
  mockMCPServer.close();

  await expect(createTestApplication()).rejects.toMatchObject({
    code: 'STARTUP_FAILED',
    message: expect.stringContaining('MCP server'),
    details: {
      component: 'mcp',
      action: 'Ensure MCP server is running',
    },
  });
});
```

**Expected Result**:
- Application startup fails
- Clear error message with remediation
- No partial startup state

---

### MCP-STARTUP-003: Startup with MCP Retry

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-STARTUP-003 |
| **Title** | Retry MCP Connection During Startup |
| **Priority** | P1 - High |
| **Spec Reference** | Section 4.7 (Startup Resilience) |

**Test Steps**:

```typescript
it('retries MCP connection during startup', async () => {
  let attemptCount = 0;

  mockMCPServer((request) => {
    attemptCount++;
    if (attemptCount < 3) {
      throw new Error('Connection refused');
    }
    return {
      result: {
        serverInfo: { name: 'hx-docling-mcp-server', version: '1.0.0' },
        capabilities: { tools: {} },
      },
    };
  });

  const app = await createTestApplication({
    mcp: { startupRetries: 5, startupRetryDelay: 100 }
  });

  expect(app.isReady()).toBe(true);
  expect(attemptCount).toBe(3);
});
```

**Expected Result**:
- Startup retries MCP connection
- Application starts after successful retry
- Retry count within configured limit

---

### MCP-SHUTDOWN-001: Graceful Shutdown Completes Pending Requests

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-SHUTDOWN-001 |
| **Title** | Complete Pending MCP Requests on Shutdown |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 4.7 (Graceful Shutdown) |

**Test Steps**:

```typescript
it('completes pending MCP requests during graceful shutdown', async () => {
  const app = await createTestApplication();
  let requestCompleted = false;

  mockMCPServer(() => {
    return new Promise((resolve) => {
      setTimeout(() => {
        requestCompleted = true;
        resolve({
          result: { content: [{ type: 'document', document: mockDoclingDoc }] },
        });
      }, 500);
    });
  });

  // Start a request
  const requestPromise = app.mcpClient.invoke('convert_pdf', {
    file_path: '/tmp/test.pdf'
  });

  // Initiate shutdown while request in progress
  await new Promise(resolve => setTimeout(resolve, 100));
  const shutdownPromise = app.shutdown();

  // Both should complete
  const [result] = await Promise.all([requestPromise, shutdownPromise]);

  expect(requestCompleted).toBe(true);
  expect(result.content[0].document).toBeDefined();
});
```

**Expected Result**:
- Pending requests complete before shutdown
- No data loss or corruption
- Clean shutdown achieved

---

### MCP-SHUTDOWN-002: Shutdown Timeout Enforcement

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-SHUTDOWN-002 |
| **Title** | Enforce Shutdown Timeout for MCP Requests |
| **Priority** | P1 - High |
| **Spec Reference** | Section 4.7 (Shutdown Timeout) |

**Test Steps**:

```typescript
it('enforces shutdown timeout for hanging MCP requests', async () => {
  const app = await createTestApplication({ shutdownTimeout: 500 });

  mockMCPServer(() => {
    // Never respond
    return new Promise(() => {});
  });

  // Start a request that will hang
  const requestPromise = app.mcpClient.invoke('convert_pdf', {
    file_path: '/tmp/test.pdf'
  });

  // Initiate shutdown
  await new Promise(resolve => setTimeout(resolve, 100));
  const start = Date.now();
  await app.shutdown();
  const elapsed = Date.now() - start;

  // Should complete within shutdown timeout
  expect(elapsed).toBeLessThan(1000);

  // Hanging request should be cancelled
  await expect(requestPromise).rejects.toMatchObject({
    name: 'AbortError',
  });
});
```

**Expected Result**:
- Shutdown completes within timeout
- Hanging requests cancelled
- Application terminates cleanly

---

### 11.4 Logging and Observability

---

### MCP-LOG-001: Log MCP Request/Response

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-LOG-001 |
| **Title** | Log MCP Requests and Responses |
| **Priority** | P1 - High |
| **Spec Reference** | Section 4.7 (Observability) |

**Test Steps**:

```typescript
it('logs MCP requests and responses', async () => {
  const logs: LogEntry[] = [];
  mockLogger.onLog((entry) => logs.push(entry));

  mockMCPServer(() => ({
    result: { content: [{ type: 'document', document: mockDoclingDoc }] },
  }));

  await mcpClient.invoke('convert_pdf', { file_path: '/tmp/test.pdf' });

  // Verify request logged
  const requestLog = logs.find(l => l.message.includes('MCP request'));
  expect(requestLog).toBeDefined();
  expect(requestLog.metadata).toMatchObject({
    tool: 'convert_pdf',
    requestId: expect.any(String),
  });

  // Verify response logged
  const responseLog = logs.find(l => l.message.includes('MCP response'));
  expect(responseLog).toBeDefined();
  expect(responseLog.metadata).toMatchObject({
    tool: 'convert_pdf',
    requestId: requestLog.metadata.requestId,
    durationMs: expect.any(Number),
    success: true,
  });
});
```

**Expected Result**:
- Request logged with tool and request ID
- Response logged with duration and status
- Request/response correlated by ID

---

### MCP-LOG-002: Log MCP Errors

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-LOG-002 |
| **Title** | Log MCP Errors with Details |
| **Priority** | P1 - High |
| **Spec Reference** | Section 4.7 (Error Logging) |

**Test Steps**:

```typescript
it('logs MCP errors with full details', async () => {
  const logs: LogEntry[] = [];
  mockLogger.onLog((entry) => logs.push(entry));

  mockMCPServer(() => ({
    error: { code: -32002, message: 'Document conversion failed' },
  }));

  await expect(
    mcpClient.invoke('convert_pdf', { file_path: '/tmp/test.pdf' })
  ).rejects.toThrow();

  const errorLog = logs.find(l => l.level === 'error');
  expect(errorLog).toBeDefined();
  expect(errorLog.metadata).toMatchObject({
    tool: 'convert_pdf',
    mcpErrorCode: -32002,
    mappedErrorCode: 'E302',
    retryable: true,
    errorMessage: expect.stringContaining('conversion failed'),
  });
});
```

**Expected Result**:
- Errors logged at error level
- MCP error code and mapped code included
- Retryable flag indicated

---

### MCP-LOG-003: Structured Logging Format

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-LOG-003 |
| **Title** | Use Structured Logging Format |
| **Priority** | P2 - Medium |
| **Spec Reference** | Section 4.7 (Log Format) |

**Test Steps**:

```typescript
it('outputs logs in structured JSON format', async () => {
  const rawLogs: string[] = [];
  mockLogger.onRawLog((log) => rawLogs.push(log));

  mockMCPServer(() => ({
    result: { content: [{ type: 'document', document: mockDoclingDoc }] },
  }));

  await mcpClient.invoke('convert_pdf', { file_path: '/tmp/test.pdf' });

  // Verify logs are valid JSON
  rawLogs.forEach(log => {
    const parsed = JSON.parse(log);
    expect(parsed).toHaveProperty('timestamp');
    expect(parsed).toHaveProperty('level');
    expect(parsed).toHaveProperty('message');
    expect(parsed).toHaveProperty('service', 'hx-docling-ui');
  });
});
```

**Expected Result**:
- Logs output as valid JSON
- Required fields present
- Service identifier included

---

### MCP-LOG-004: Sensitive Data Redaction

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-LOG-004 |
| **Title** | Redact Sensitive Data in Logs |
| **Priority** | P1 - High |
| **Spec Reference** | Section 4.7 (Security) |

**Test Steps**:

```typescript
it('redacts sensitive data from MCP logs', async () => {
  const logs: LogEntry[] = [];
  mockLogger.onLog((entry) => logs.push(entry));

  mockMCPServer(() => ({
    result: { content: [{ type: 'document', document: mockDoclingDoc }] },
  }));

  await mcpClient.invoke('convert_pdf', {
    file_path: '/tmp/test.pdf',
    auth_token: 'secret-token-12345',
  });

  // Verify sensitive data redacted
  const allLogText = JSON.stringify(logs);
  expect(allLogText).not.toContain('secret-token-12345');
  expect(allLogText).toContain('[REDACTED]');
});
```

**Expected Result**:
- Sensitive data not in logs
- Redaction markers visible
- Log utility preserved

---

### MCP-LOG-005: Performance Metrics Logging

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-LOG-005 |
| **Title** | Log MCP Performance Metrics |
| **Priority** | P2 - Medium |
| **Spec Reference** | Section 4.7 (Performance Monitoring) |

**Test Steps**:

```typescript
it('logs MCP performance metrics', async () => {
  const metrics: MetricEntry[] = [];
  mockMetrics.onMetric((entry) => metrics.push(entry));

  mockMCPServer(() => {
    return new Promise((resolve) => {
      setTimeout(() => {
        resolve({
          result: { content: [{ type: 'document', document: mockDoclingDoc }] },
        });
      }, 150);
    });
  });

  await mcpClient.invoke('convert_pdf', { file_path: '/tmp/test.pdf' });

  const durationMetric = metrics.find(m => m.name === 'mcp_request_duration_ms');
  expect(durationMetric).toBeDefined();
  expect(durationMetric.value).toBeGreaterThanOrEqual(150);
  expect(durationMetric.tags).toMatchObject({
    tool: 'convert_pdf',
    status: 'success',
  });
});
```

**Expected Result**:
- Duration metric recorded
- Tool and status tagged
- Metric values accurate

---

## 12. Next.js Integration Tests

### 12.1 API Route Integration

---

### MCP-NEXT-001: API Route MCP Client Initialization

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-NEXT-001 |
| **Title** | Initialize MCP Client in API Route |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.2 (API Routes) |

**Test Steps**:

```typescript
it('initializes MCP client in API route handler', async () => {
  const response = await fetch('/api/v1/convert', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ file_path: '/tmp/test.pdf' }),
  });

  expect(response.status).toBe(200);

  const data = await response.json();
  expect(data.jobId).toBeDefined();
  expect(data.status).toBe('processing');
});
```

**Expected Result**:
- API route successfully uses MCP client
- Job created and returned
- Processing initiated

---

### MCP-NEXT-002: API Route Request Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-NEXT-002 |
| **Title** | Validate Request Body in API Route |
| **Priority** | P1 - High |
| **Spec Reference** | Section 5.2 (Input Validation) |

**Test Steps**:

```typescript
it('validates request body before MCP invocation', async () => {
  const response = await fetch('/api/v1/convert', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ invalid_field: 'test' }),
  });

  expect(response.status).toBe(400);

  const error = await response.json();
  expect(error).toMatchObject({
    code: 'E203',
    message: expect.stringContaining('file_path'),
  });
});
```

**Expected Result**:
- Invalid request rejected with 400
- Validation error details provided
- MCP not invoked for invalid request

---

### MCP-NEXT-003: API Route Error Response Format

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-NEXT-003 |
| **Title** | Format MCP Errors in API Response |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.2 (Error Handling) |

**Test Steps**:

```typescript
it('returns properly formatted error response for MCP failures', async () => {
  mockMCPServer(() => ({
    error: { code: -32002, message: 'Document conversion failed' },
  }));

  const response = await fetch('/api/v1/convert', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ file_path: '/tmp/corrupt.pdf' }),
  });

  expect(response.status).toBe(500);

  const error = await response.json();
  expect(error).toMatchObject({
    code: 'E302',
    message: expect.stringContaining('conversion failed'),
    retryable: true,
    requestId: expect.any(String),
  });
});
```

**Expected Result**:
- Error response includes mapped code
- User-friendly message returned
- Request ID for debugging

---

### MCP-NEXT-004: API Route Timeout Handling

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-NEXT-004 |
| **Title** | Handle MCP Timeout in API Route |
| **Priority** | P1 - High |
| **Spec Reference** | Section 5.2 (Timeout Handling) |

**Test Steps**:

```typescript
it('returns timeout error when MCP call exceeds limit', async () => {
  mockMCPServer(() => {
    return new Promise(() => {}); // Never respond
  });

  const controller = new AbortController();
  setTimeout(() => controller.abort(), 5000);

  const response = await fetch('/api/v1/convert', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ file_path: '/tmp/large.pdf' }),
    signal: controller.signal,
  });

  expect(response.status).toBe(504);

  const error = await response.json();
  expect(error.code).toBe('E202');
});
```

**Expected Result**:
- Timeout returns 504 Gateway Timeout
- Error code E202 returned
- Request properly terminated

---

### 12.2 Server Components

---

### MCP-SC-001: Server Component Result Consumption

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-SC-001 |
| **Title** | Server Component Consumes MCP Results |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.3 (Server Components) |

**Test Steps**:

```typescript
it('renders MCP result in Server Component', async () => {
  mockMCPServer(() => ({
    result: {
      content: [{
        type: 'document',
        document: {
          ...mockDoclingDoc,
          body: {
            self_ref: '#/body',
            children: [
              { type: 'heading', level: 1, text: 'Test Document' },
              { type: 'paragraph', text: 'Document content here.' },
            ],
          },
        },
      }],
    },
  }));

  const html = await renderServerComponent(<DocumentViewer jobId="test-job-123" />);

  expect(html).toContain('Test Document');
  expect(html).toContain('Document content here.');
});
```

**Expected Result**:
- Server component fetches MCP result
- Content rendered in HTML
- Document structure preserved

---

### MCP-SC-002: Server Component Loading State

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-SC-002 |
| **Title** | Server Component Suspense Loading |
| **Priority** | P1 - High |
| **Spec Reference** | Section 5.3 (Loading States) |

**Test Steps**:

```typescript
it('shows loading skeleton during MCP fetch', async () => {
  mockMCPServer(() => {
    return new Promise((resolve) => {
      setTimeout(() => {
        resolve({
          result: { content: [{ type: 'document', document: mockDoclingDoc }] },
        });
      }, 100);
    });
  });

  const { container } = render(
    <Suspense fallback={<DocumentSkeleton />}>
      <DocumentViewer jobId="test-job-123" />
    </Suspense>
  );

  // Initially shows skeleton
  expect(container.querySelector('[data-testid="document-skeleton"]')).toBeInTheDocument();

  // After load, shows content
  await waitFor(() => {
    expect(container.querySelector('[data-testid="document-skeleton"]')).not.toBeInTheDocument();
    expect(container.querySelector('[data-testid="document-content"]')).toBeInTheDocument();
  });
});
```

**Expected Result**:
- Skeleton shown during loading
- Content replaces skeleton on completion
- Smooth transition

---

### MCP-SC-003: Server Component Error Boundary

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-SC-003 |
| **Title** | Server Component Error Boundary |
| **Priority** | P1 - High |
| **Spec Reference** | Section 5.3 (Error Handling) |

**Test Steps**:

```typescript
it('renders error boundary on MCP failure in Server Component', async () => {
  mockMCPServer(() => ({
    error: { code: -32603, message: 'Internal error' },
  }));

  const html = await renderServerComponent(
    <ErrorBoundary fallback={<ErrorDisplay />}>
      <DocumentViewer jobId="test-job-123" />
    </ErrorBoundary>
  );

  expect(html).toContain('data-testid="error-display"');
  expect(html).toContain('Something went wrong');
});
```

**Expected Result**:
- Error boundary catches MCP error
- Fallback UI rendered
- Error details not exposed

---

### MCP-SC-004: Server Component Caching

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-SC-004 |
| **Title** | Cache Server Component MCP Results |
| **Priority** | P2 - Medium |
| **Spec Reference** | Section 5.3 (Caching) |

**Test Steps**:

```typescript
it('caches MCP results for repeated Server Component renders', async () => {
  let mcpCallCount = 0;

  mockMCPServer(() => {
    mcpCallCount++;
    return {
      result: { content: [{ type: 'document', document: mockDoclingDoc }] },
    };
  });

  // First render
  await renderServerComponent(<DocumentViewer jobId="test-job-123" />);

  // Second render with same props (should use cache)
  await renderServerComponent(<DocumentViewer jobId="test-job-123" />);

  // Should only call MCP once
  expect(mcpCallCount).toBe(1);
});
```

**Expected Result**:
- MCP result cached
- Subsequent renders use cache
- Cache key based on job ID

---

### 12.3 Server Actions

---

### MCP-ACTION-001: Cancel Server Action Aborts MCP

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-ACTION-001 |
| **Title** | Cancel Server Action Aborts MCP Operation |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.4 (Server Actions) |

**Test Steps**:

```typescript
it('server action cancel aborts MCP operation', async () => {
  let abortSignalAborted = false;

  mockMCPServer((request, { signal }) => {
    signal.addEventListener('abort', () => {
      abortSignalAborted = true;
    });

    return new Promise((resolve) => {
      setTimeout(() => {
        if (!signal.aborted) {
          resolve({
            result: { content: [{ type: 'document', document: mockDoclingDoc }] },
          });
        }
      }, 5000);
    });
  });

  // Start conversion via server action
  const { jobId } = await convertDocument({ file_path: '/tmp/test.pdf' });

  // Cancel via server action
  await cancelJob(jobId);

  expect(abortSignalAborted).toBe(true);
});
```

**Expected Result**:
- Server action triggers abort
- MCP operation cancelled
- Resources cleaned up

---

### MCP-ACTION-002: Server Action Retry

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-ACTION-002 |
| **Title** | Retry Failed MCP via Server Action |
| **Priority** | P1 - High |
| **Spec Reference** | Section 5.4 (Retry Action) |

**Test Steps**:

```typescript
it('retries failed MCP operation via server action', async () => {
  let callCount = 0;

  mockMCPServer(() => {
    callCount++;
    if (callCount === 1) {
      return { error: { code: -32603, message: 'Internal error' } };
    }
    return {
      result: { content: [{ type: 'document', document: mockDoclingDoc }] },
    };
  });

  // First attempt fails
  const { jobId, error } = await convertDocument({ file_path: '/tmp/test.pdf' });
  expect(error).toBeDefined();

  // Retry via server action
  const retryResult = await retryJob(jobId);

  expect(retryResult.success).toBe(true);
  expect(callCount).toBe(2);
});
```

**Expected Result**:
- Retry creates new MCP call
- Previous job state preserved
- Success on retry

---

### MCP-ACTION-003: Server Action Progress Subscription

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-ACTION-003 |
| **Title** | Subscribe to Progress via Server Action |
| **Priority** | P1 - High |
| **Spec Reference** | Section 5.4 (Progress Subscription) |

**Test Steps**:

```typescript
it('subscribes to MCP progress via server action', async () => {
  const progressUpdates: ProgressEvent[] = [];

  mockMCPServer((request) => {
    if (request.params.name === 'convert_pdf') {
      emitProgress({ percent: 25, stage: 'Starting' });
      emitProgress({ percent: 50, stage: 'Processing' });
      emitProgress({ percent: 75, stage: 'Finalizing' });

      return {
        result: { content: [{ type: 'document', document: mockDoclingDoc }] },
      };
    }
  });

  // Start conversion
  const { jobId } = await convertDocument({ file_path: '/tmp/test.pdf' });

  // Subscribe to progress
  const unsubscribe = subscribeToProgress(jobId, (event) => {
    progressUpdates.push(event);
  });

  await waitFor(() => {
    expect(progressUpdates.length).toBeGreaterThanOrEqual(3);
  });

  unsubscribe();

  expect(progressUpdates).toContainEqual(
    expect.objectContaining({ percent: 25, stage: 'Starting' })
  );
});
```

**Expected Result**:
- Progress subscription works
- All progress events received
- Unsubscribe cleans up

---

### 12.4 SSE Coordination

---

### MCP-SSE-001: MCP Progress to SSE Event Mapping

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-SSE-001 |
| **Title** | Map MCP Progress to SSE Events |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 6.2 (SSE Integration) |

**Test Steps**:

```typescript
it('maps MCP progress events to SSE stream', async () => {
  const sseEvents: SSEEvent[] = [];

  mockMCPServer((request) => {
    if (request.params.name === 'convert_pdf') {
      emitMCPProgress({ percent: 33, stage: 'Parsing' });
      emitMCPProgress({ percent: 66, stage: 'Converting' });

      return {
        result: { content: [{ type: 'document', document: mockDoclingDoc }] },
      };
    }
  });

  // Start conversion and connect to SSE
  const response = await fetch('/api/v1/convert', {
    method: 'POST',
    body: JSON.stringify({ file_path: '/tmp/test.pdf' }),
  });
  const { jobId } = await response.json();

  const eventSource = new EventSource(`/api/v1/jobs/${jobId}/progress`);
  eventSource.onmessage = (event) => {
    sseEvents.push(JSON.parse(event.data));
  };

  await waitFor(() => {
    expect(sseEvents.length).toBeGreaterThanOrEqual(2);
  });

  eventSource.close();

  expect(sseEvents[0]).toMatchObject({ type: 'progress', data: { percent: 33 } });
  expect(sseEvents[1]).toMatchObject({ type: 'progress', data: { percent: 66 } });
});
```

**Expected Result**:
- MCP progress mapped to SSE events
- Events delivered in order
- Correct event format

---

### MCP-SSE-002: SSE Reconnection Recovery

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-SSE-002 |
| **Title** | Recover SSE Events After Reconnection |
| **Priority** | P1 - High |
| **Spec Reference** | Section 6.2 (SSE Recovery) |

**Test Steps**:

```typescript
it('recovers missed events after SSE reconnection', async () => {
  const sseEvents: SSEEvent[] = [];

  mockMCPServer((request) => {
    if (request.params.name === 'convert_pdf') {
      // Emit events that client will miss during disconnect
      emitMCPProgress({ percent: 25, stage: 'Stage 1' });
      emitMCPProgress({ percent: 50, stage: 'Stage 2' });
      emitMCPProgress({ percent: 75, stage: 'Stage 3' });

      return new Promise((resolve) => {
        setTimeout(() => {
          resolve({
            result: { content: [{ type: 'document', document: mockDoclingDoc }] },
          });
        }, 500);
      });
    }
  });

  // Start conversion
  const response = await fetch('/api/v1/convert', {
    method: 'POST',
    body: JSON.stringify({ file_path: '/tmp/test.pdf' }),
  });
  const { jobId } = await response.json();

  // Connect after some events already emitted
  await new Promise(resolve => setTimeout(resolve, 200));

  const eventSource = new EventSource(
    `/api/v1/jobs/${jobId}/progress?lastEventId=0`
  );
  eventSource.onmessage = (event) => {
    sseEvents.push(JSON.parse(event.data));
  };

  await waitFor(() => {
    expect(sseEvents.some(e => e.data?.percent === 75)).toBe(true);
  });

  eventSource.close();

  // Should have received buffered events
  expect(sseEvents.length).toBeGreaterThanOrEqual(2);
});
```

**Expected Result**:
- Missed events recovered from buffer
- No duplicate events
- Complete progress history available

---

### MCP-SSE-003: SSE Completion Event

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-SSE-003 |
| **Title** | Send SSE Completion Event After MCP Success |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 6.2 (Completion Events) |

**Test Steps**:

```typescript
it('sends completion event when MCP operation succeeds', async () => {
  const sseEvents: SSEEvent[] = [];

  mockMCPServer(() => ({
    result: { content: [{ type: 'document', document: mockDoclingDoc }] },
  }));

  const response = await fetch('/api/v1/convert', {
    method: 'POST',
    body: JSON.stringify({ file_path: '/tmp/test.pdf' }),
  });
  const { jobId } = await response.json();

  const eventSource = new EventSource(`/api/v1/jobs/${jobId}/progress`);
  eventSource.onmessage = (event) => {
    sseEvents.push(JSON.parse(event.data));
  };

  await waitFor(() => {
    expect(sseEvents.some(e => e.type === 'complete')).toBe(true);
  });

  eventSource.close();

  const completeEvent = sseEvents.find(e => e.type === 'complete');
  expect(completeEvent).toMatchObject({
    type: 'complete',
    data: {
      jobId,
      status: 'completed',
      hasResult: true,
    },
  });
});
```

**Expected Result**:
- Completion event sent
- Event includes job status
- Result availability indicated

---

### MCP-SSE-004: SSE Error Event

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-SSE-004 |
| **Title** | Send SSE Error Event on MCP Failure |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 6.2 (Error Events) |

**Test Steps**:

```typescript
it('sends error event when MCP operation fails', async () => {
  const sseEvents: SSEEvent[] = [];

  mockMCPServer(() => ({
    error: { code: -32002, message: 'Conversion failed' },
  }));

  const response = await fetch('/api/v1/convert', {
    method: 'POST',
    body: JSON.stringify({ file_path: '/tmp/corrupt.pdf' }),
  });
  const { jobId } = await response.json();

  const eventSource = new EventSource(`/api/v1/jobs/${jobId}/progress`);
  eventSource.onmessage = (event) => {
    sseEvents.push(JSON.parse(event.data));
  };

  await waitFor(() => {
    expect(sseEvents.some(e => e.type === 'error')).toBe(true);
  });

  eventSource.close();

  const errorEvent = sseEvents.find(e => e.type === 'error');
  expect(errorEvent).toMatchObject({
    type: 'error',
    data: {
      code: 'E302',
      message: expect.stringContaining('conversion'),
      retryable: true,
    },
  });
});
```

**Expected Result**:
- Error event sent
- Error code and message included
- Retryable flag set

---

### MCP-SSE-005: SSE Heartbeat During Long Operations

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-SSE-005 |
| **Title** | Send SSE Heartbeat During Long MCP Operations |
| **Priority** | P1 - High |
| **Spec Reference** | Section 6.2 (Connection Keep-Alive) |

**Test Steps**:

```typescript
it('sends heartbeat events during long MCP operations', async () => {
  const sseEvents: SSEEvent[] = [];

  mockMCPServer(() => {
    return new Promise((resolve) => {
      setTimeout(() => {
        resolve({
          result: { content: [{ type: 'document', document: mockDoclingDoc }] },
        });
      }, 10000);
    });
  });

  const response = await fetch('/api/v1/convert', {
    method: 'POST',
    body: JSON.stringify({ file_path: '/tmp/large.pdf' }),
  });
  const { jobId } = await response.json();

  const eventSource = new EventSource(`/api/v1/jobs/${jobId}/progress`);
  eventSource.onmessage = (event) => {
    sseEvents.push(JSON.parse(event.data));
  };

  // Wait for heartbeats (should be sent every 5 seconds)
  await new Promise(resolve => setTimeout(resolve, 12000));

  eventSource.close();

  const heartbeats = sseEvents.filter(e => e.type === 'heartbeat');
  expect(heartbeats.length).toBeGreaterThanOrEqual(2);
}, 15000);
```

**Expected Result**:
- Heartbeat events sent periodically
- Connection kept alive
- No client timeout

---

## 13. Workflow Orchestration Tests

### 13.1 Checkpoint Integration

---

### MCP-CKPT-001: Create Checkpoint Before MCP Call

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-CKPT-001 |
| **Title** | Create Checkpoint Before MCP Invocation |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.5 (Checkpoint Pattern) |

**Test Steps**:

```typescript
it('creates checkpoint before MCP tool invocation', async () => {
  const checkpoints: Checkpoint[] = [];
  mockCheckpointStore.onCreate((cp) => checkpoints.push(cp));

  mockMCPServer(() => ({
    result: { content: [{ type: 'document', document: mockDoclingDoc }] },
  }));

  await workflowEngine.executeConversionWorkflow({
    file_path: '/tmp/test.pdf',
    jobId: 'test-job-123',
  });

  const preMcpCheckpoint = checkpoints.find(
    cp => cp.stage === 'pre_mcp_invoke'
  );

  expect(preMcpCheckpoint).toBeDefined();
  expect(preMcpCheckpoint.data).toMatchObject({
    tool: 'convert_pdf',
    arguments: { file_path: '/tmp/test.pdf' },
    timestamp: expect.any(String),
  });
});
```

**Expected Result**:
- Checkpoint created before MCP call
- Tool and arguments captured
- Timestamp recorded

---

### MCP-CKPT-002: Resume From Checkpoint After Failure

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-CKPT-002 |
| **Title** | Resume Workflow From Checkpoint After Failure |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.5 (Checkpoint Recovery) |

**Test Steps**:

```typescript
it('resumes workflow from checkpoint after MCP failure', async () => {
  let callCount = 0;

  mockMCPServer(() => {
    callCount++;
    if (callCount === 1) {
      throw new Error('Network error');
    }
    return {
      result: { content: [{ type: 'document', document: mockDoclingDoc }] },
    };
  });

  // First execution fails
  await expect(
    workflowEngine.executeConversionWorkflow({
      file_path: '/tmp/test.pdf',
      jobId: 'test-job-123',
    })
  ).rejects.toThrow();

  // Resume from checkpoint
  const result = await workflowEngine.resumeFromCheckpoint('test-job-123');

  expect(result.success).toBe(true);
  expect(callCount).toBe(2);
});
```

**Expected Result**:
- Workflow resumes from checkpoint
- No duplicate processing
- Second attempt succeeds

---

### MCP-CKPT-003: Checkpoint Contains MCP Response

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-CKPT-003 |
| **Title** | Store MCP Response in Checkpoint |
| **Priority** | P1 - High |
| **Spec Reference** | Section 5.5 (Response Checkpointing) |

**Test Steps**:

```typescript
it('stores MCP response in checkpoint after success', async () => {
  const checkpoints: Checkpoint[] = [];
  mockCheckpointStore.onCreate((cp) => checkpoints.push(cp));

  mockMCPServer(() => ({
    result: { content: [{ type: 'document', document: mockDoclingDoc }] },
  }));

  await workflowEngine.executeConversionWorkflow({
    file_path: '/tmp/test.pdf',
    jobId: 'test-job-123',
  });

  const postMcpCheckpoint = checkpoints.find(
    cp => cp.stage === 'post_mcp_invoke'
  );

  expect(postMcpCheckpoint).toBeDefined();
  expect(postMcpCheckpoint.data).toMatchObject({
    tool: 'convert_pdf',
    success: true,
    response: {
      content: expect.arrayContaining([
        expect.objectContaining({ type: 'document' }),
      ]),
    },
  });
});
```

**Expected Result**:
- Response stored in checkpoint
- Full response preserved
- Success flag set

---

### MCP-CKPT-004: Skip Completed Steps on Resume

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-CKPT-004 |
| **Title** | Skip Already Completed MCP Steps on Resume |
| **Priority** | P1 - High |
| **Spec Reference** | Section 5.5 (Idempotent Recovery) |

**Test Steps**:

```typescript
it('skips completed MCP steps when resuming workflow', async () => {
  let convertCallCount = 0;
  let exportCallCount = 0;

  mockMCPServer((request) => {
    if (request.params.name === 'convert_pdf') {
      convertCallCount++;
      return {
        result: { content: [{ type: 'document', document: mockDoclingDoc }] },
      };
    }
    if (request.params.name === 'export_markdown') {
      exportCallCount++;
      if (exportCallCount === 1) {
        throw new Error('Network error');
      }
      return {
        result: { content: [{ type: 'text', text: '# Content' }] },
      };
    }
  });

  // First execution: convert succeeds, export fails
  await expect(
    workflowEngine.executeConversionWorkflow({
      file_path: '/tmp/test.pdf',
      jobId: 'test-job-123',
      exportFormats: ['markdown'],
    })
  ).rejects.toThrow();

  // Resume: should skip convert, retry export
  await workflowEngine.resumeFromCheckpoint('test-job-123');

  expect(convertCallCount).toBe(1); // Not called again
  expect(exportCallCount).toBe(2); // Retried
});
```

**Expected Result**:
- Completed steps not repeated
- Only failed step retried
- Checkpoint used for step tracking

---

### MCP-CKPT-005: Checkpoint Cleanup After Success

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-CKPT-005 |
| **Title** | Clean Up Checkpoints After Successful Workflow |
| **Priority** | P2 - Medium |
| **Spec Reference** | Section 5.5 (Checkpoint Lifecycle) |

**Test Steps**:

```typescript
it('cleans up checkpoints after successful workflow completion', async () => {
  mockMCPServer(() => ({
    result: { content: [{ type: 'document', document: mockDoclingDoc }] },
  }));

  await workflowEngine.executeConversionWorkflow({
    file_path: '/tmp/test.pdf',
    jobId: 'test-job-123',
  });

  // Verify checkpoints cleaned up
  const remainingCheckpoints = await mockCheckpointStore.getByJobId('test-job-123');
  expect(remainingCheckpoints.length).toBe(0);
});
```

**Expected Result**:
- Checkpoints removed after success
- No stale checkpoint data
- Storage cleaned up

---

### 13.2 State Machine Integration

---

### MCP-STATE-001: Store State on MCP Initiation

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-STATE-001 |
| **Title** | Update State Machine on MCP Call Start |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.5 (State Management) |

**Test Steps**:

```typescript
it('transitions state machine when MCP call starts', async () => {
  const stateTransitions: StateTransition[] = [];
  mockStateMachine.onTransition((t) => stateTransitions.push(t));

  mockMCPServer(() => {
    return new Promise((resolve) => {
      setTimeout(() => {
        resolve({
          result: { content: [{ type: 'document', document: mockDoclingDoc }] },
        });
      }, 100);
    });
  });

  const workflow = workflowEngine.executeConversionWorkflow({
    file_path: '/tmp/test.pdf',
    jobId: 'test-job-123',
  });

  await waitFor(() => {
    const mcpTransition = stateTransitions.find(
      t => t.to === 'invoking_mcp'
    );
    expect(mcpTransition).toBeDefined();
  });

  await workflow;

  expect(stateTransitions).toContainEqual(
    expect.objectContaining({
      from: 'initialized',
      to: 'invoking_mcp',
      event: 'MCP_CALL_START',
    })
  );
});
```

**Expected Result**:
- State transitions to invoking_mcp
- Transition event recorded
- Previous state captured

---

### MCP-STATE-002: Update State on MCP Success

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-STATE-002 |
| **Title** | Update State Machine on MCP Success |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.5 (State Management) |

**Test Steps**:

```typescript
it('transitions state machine when MCP call succeeds', async () => {
  const stateTransitions: StateTransition[] = [];
  mockStateMachine.onTransition((t) => stateTransitions.push(t));

  mockMCPServer(() => ({
    result: { content: [{ type: 'document', document: mockDoclingDoc }] },
  }));

  await workflowEngine.executeConversionWorkflow({
    file_path: '/tmp/test.pdf',
    jobId: 'test-job-123',
  });

  expect(stateTransitions).toContainEqual(
    expect.objectContaining({
      from: 'invoking_mcp',
      to: 'mcp_completed',
      event: 'MCP_CALL_SUCCESS',
    })
  );
});
```

**Expected Result**:
- State transitions to mcp_completed
- Success event recorded
- State machine consistent

---

### MCP-STATE-003: Update State on MCP Failure

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-STATE-003 |
| **Title** | Update State Machine on MCP Failure |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.5 (State Management) |

**Test Steps**:

```typescript
it('transitions state machine when MCP call fails', async () => {
  const stateTransitions: StateTransition[] = [];
  mockStateMachine.onTransition((t) => stateTransitions.push(t));

  mockMCPServer(() => ({
    error: { code: -32002, message: 'Conversion failed' },
  }));

  await expect(
    workflowEngine.executeConversionWorkflow({
      file_path: '/tmp/test.pdf',
      jobId: 'test-job-123',
    })
  ).rejects.toThrow();

  expect(stateTransitions).toContainEqual(
    expect.objectContaining({
      from: 'invoking_mcp',
      to: 'mcp_failed',
      event: 'MCP_CALL_FAILURE',
      metadata: expect.objectContaining({
        errorCode: 'E302',
      }),
    })
  );
});
```

**Expected Result**:
- State transitions to mcp_failed
- Error metadata captured
- Failure event recorded

---

### MCP-STATE-004: State Recovery After Crash

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-STATE-004 |
| **Title** | Recover State Machine After Application Crash |
| **Priority** | P1 - High |
| **Spec Reference** | Section 5.5 (Crash Recovery) |

**Test Steps**:

```typescript
it('recovers state machine from persistent storage after crash', async () => {
  // Simulate workflow in progress
  await mockStateMachine.persistState('test-job-123', {
    currentState: 'invoking_mcp',
    data: {
      tool: 'convert_pdf',
      arguments: { file_path: '/tmp/test.pdf' },
      startedAt: new Date().toISOString(),
    },
  });

  // Simulate application restart
  const newWorkflowEngine = await createWorkflowEngine();

  // Recover state
  const recoveredState = await newWorkflowEngine.recoverState('test-job-123');

  expect(recoveredState.currentState).toBe('invoking_mcp');
  expect(recoveredState.data.tool).toBe('convert_pdf');
});
```

**Expected Result**:
- State recovered from storage
- In-progress state preserved
- Workflow can resume

---

### 13.3 Multi-Step Coordination

---

### MCP-CHAIN-001: Chain Convert and Export Operations

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-CHAIN-001 |
| **Title** | Chain Conversion and Export MCP Operations |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-404 (Prompt Chaining) |

**Test Steps**:

```typescript
it('chains conversion and export MCP operations', async () => {
  const callSequence: string[] = [];

  mockMCPServer((request) => {
    callSequence.push(request.params.name);

    if (request.params.name === 'convert_pdf') {
      return {
        result: { content: [{ type: 'document', document: mockDoclingDoc }] },
      };
    }
    if (request.params.name === 'export_markdown') {
      return {
        result: { content: [{ type: 'text', text: '# Markdown' }] },
      };
    }
    if (request.params.name === 'export_json') {
      return {
        result: { content: [{ type: 'text', text: '{}' }] },
      };
    }
  });

  const result = await workflowEngine.executeConversionWorkflow({
    file_path: '/tmp/test.pdf',
    jobId: 'test-job-123',
    exportFormats: ['markdown', 'json'],
  });

  // Verify correct sequence
  expect(callSequence[0]).toBe('convert_pdf');
  expect(callSequence.slice(1)).toContain('export_markdown');
  expect(callSequence.slice(1)).toContain('export_json');

  // Verify results
  expect(result.document).toBeDefined();
  expect(result.exports.markdown).toBe('# Markdown');
  expect(result.exports.json).toBe('{}');
});
```

**Expected Result**:
- Conversion completes before exports
- Exports executed (can be parallel)
- All results collected

---

### MCP-CHAIN-002: Handle Partial Chain Failure

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-CHAIN-002 |
| **Title** | Handle Partial Failure in MCP Chain |
| **Priority** | P1 - High |
| **Spec Reference** | FR-405 (Partial Failure) |

**Test Steps**:

```typescript
it('handles partial failure in MCP operation chain', async () => {
  mockMCPServer((request) => {
    if (request.params.name === 'convert_pdf') {
      return {
        result: { content: [{ type: 'document', document: mockDoclingDoc }] },
      };
    }
    if (request.params.name === 'export_markdown') {
      return {
        result: { content: [{ type: 'text', text: '# Markdown' }] },
      };
    }
    if (request.params.name === 'export_json') {
      return {
        error: { code: -32003, message: 'Export failed' },
      };
    }
  });

  const result = await workflowEngine.executeConversionWorkflow({
    file_path: '/tmp/test.pdf',
    jobId: 'test-job-123',
    exportFormats: ['markdown', 'json'],
    continueOnExportFailure: true,
  });

  // Conversion and successful export available
  expect(result.document).toBeDefined();
  expect(result.exports.markdown).toBe('# Markdown');

  // Failed export has error
  expect(result.exports.json).toBeUndefined();
  expect(result.exportErrors.json).toMatchObject({
    code: 'E303',
  });
});
```

**Expected Result**:
- Successful operations preserved
- Failed operations have errors
- Workflow completes with partial results

---

### MCP-CHAIN-003: Dependent MCP Operations

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-CHAIN-003 |
| **Title** | Execute Dependent MCP Operations in Order |
| **Priority** | P1 - High |
| **Spec Reference** | Section 7.1 (Operation Dependencies) |

**Test Steps**:

```typescript
it('executes dependent MCP operations in correct order', async () => {
  const callTimestamps: Record<string, number> = {};

  mockMCPServer((request) => {
    callTimestamps[request.params.name] = Date.now();

    if (request.params.name === 'convert_pdf') {
      return {
        result: { content: [{ type: 'document', document: mockDoclingDoc }] },
      };
    }
    if (request.params.name === 'export_markdown') {
      // Verify document from convert is available
      expect(request.params.arguments.document).toBeDefined();
      return {
        result: { content: [{ type: 'text', text: '# Markdown' }] },
      };
    }
  });

  await workflowEngine.executeConversionWorkflow({
    file_path: '/tmp/test.pdf',
    jobId: 'test-job-123',
    exportFormats: ['markdown'],
  });

  // Export must happen after convert
  expect(callTimestamps['export_markdown']).toBeGreaterThan(
    callTimestamps['convert_pdf']
  );
});
```

**Expected Result**:
- Dependencies respected
- Data passed between operations
- Order enforced

---

### 13.4 Timeout Coordination

---

### MCP-TO-003: Cumulative Timeout for Chained Operations

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-TO-003 |
| **Title** | Enforce Cumulative Timeout for Workflow |
| **Priority** | P1 - High |
| **Spec Reference** | Section 7.1 (Workflow Timeout) |

**Test Steps**:

```typescript
it('enforces cumulative timeout for entire workflow', async () => {
  mockMCPServer((request) => {
    return new Promise((resolve) => {
      // Each operation takes 150ms
      setTimeout(() => {
        if (request.params.name === 'convert_pdf') {
          resolve({
            result: { content: [{ type: 'document', document: mockDoclingDoc }] },
          });
        } else {
          resolve({
            result: { content: [{ type: 'text', text: 'content' }] },
          });
        }
      }, 150);
    });
  });

  // Total should exceed 400ms (convert + 3 exports)
  await expect(
    workflowEngine.executeConversionWorkflow({
      file_path: '/tmp/test.pdf',
      jobId: 'test-job-123',
      exportFormats: ['markdown', 'html', 'json'],
      workflowTimeout: 400, // Less than total time
    })
  ).rejects.toMatchObject({
    code: 'E202',
    message: expect.stringContaining('workflow timeout'),
  });
});
```

**Expected Result**:
- Workflow timeout enforced
- Timeout error thrown
- In-progress operations cancelled

---

### MCP-TO-004: Individual Operation Timeout

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-TO-004 |
| **Title** | Enforce Individual Operation Timeout |
| **Priority** | P1 - High |
| **Spec Reference** | Section 7.1 (Operation Timeout) |

**Test Steps**:

```typescript
it('enforces individual operation timeout within workflow', async () => {
  mockMCPServer((request) => {
    if (request.params.name === 'convert_pdf') {
      return new Promise((resolve) => {
        setTimeout(() => {
          resolve({
            result: { content: [{ type: 'document', document: mockDoclingDoc }] },
          });
        }, 50);
      });
    }
    if (request.params.name === 'export_markdown') {
      // This operation hangs
      return new Promise(() => {});
    }
  });

  await expect(
    workflowEngine.executeConversionWorkflow({
      file_path: '/tmp/test.pdf',
      jobId: 'test-job-123',
      exportFormats: ['markdown'],
      operationTimeout: 100,
    })
  ).rejects.toMatchObject({
    code: 'E202',
    metadata: expect.objectContaining({
      operation: 'export_markdown',
    }),
  });
});
```

**Expected Result**:
- Individual operation timeout enforced
- Specific operation identified
- Workflow fails appropriately

---

### MCP-TO-005: Timeout Extension for Large Files

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-TO-005 |
| **Title** | Extend Timeout Based on File Size |
| **Priority** | P2 - Medium |
| **Spec Reference** | Section 7.1 (Dynamic Timeout) |

**Test Steps**:

```typescript
describe('Dynamic Timeout Extension', () => {
  const testCases = [
    { size: 500 * 1024, expectedMinTimeout: 60000 },
    { size: 5 * 1024 * 1024, expectedMinTimeout: 180000 },
    { size: 20 * 1024 * 1024, expectedMinTimeout: 300000 },
  ];

  testCases.forEach(({ size, expectedMinTimeout }) => {
    it(`extends timeout to ${expectedMinTimeout}ms for ${size} bytes`, async () => {
      let actualTimeout: number | undefined;

      mockMCPServer((request, { timeout }) => {
        actualTimeout = timeout;
        return {
          result: { content: [{ type: 'document', document: mockDoclingDoc }] },
        };
      });

      await workflowEngine.executeConversionWorkflow({
        file_path: '/tmp/test.pdf',
        jobId: 'test-job-123',
        fileSize: size,
      });

      expect(actualTimeout).toBeGreaterThanOrEqual(expectedMinTimeout);
    });
  });
});
```

**Expected Result**:
- Timeout scaled with file size
- Minimum timeout enforced
- Large files get extended time

---

### 13.5 Progress Coordination

---

### MCP-PROG-001: Aggregate Progress Across Operations

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-PROG-001 |
| **Title** | Aggregate Progress from Multiple MCP Operations |
| **Priority** | P1 - High |
| **Spec Reference** | Section 6.2 (Progress Aggregation) |

**Test Steps**:

```typescript
it('aggregates progress from chained MCP operations', async () => {
  const progressEvents: ProgressEvent[] = [];

  mockMCPServer((request) => {
    if (request.params.name === 'convert_pdf') {
      emitProgress({ percent: 50, stage: 'Converting' });
      return {
        result: { content: [{ type: 'document', document: mockDoclingDoc }] },
      };
    }
    if (request.params.name === 'export_markdown') {
      emitProgress({ percent: 50, stage: 'Exporting' });
      return {
        result: { content: [{ type: 'text', text: '# Markdown' }] },
      };
    }
  });

  await workflowEngine.executeConversionWorkflow({
    file_path: '/tmp/test.pdf',
    jobId: 'test-job-123',
    exportFormats: ['markdown'],
    onProgress: (event) => progressEvents.push(event),
  });

  // Progress should be aggregated (convert = 50%, export = 50%)
  expect(progressEvents).toContainEqual(
    expect.objectContaining({
      overallPercent: 25, // 50% of convert phase (50% of total)
      phase: 'conversion',
    })
  );
  expect(progressEvents).toContainEqual(
    expect.objectContaining({
      overallPercent: 75, // 50% + 50% of export phase (50% of remaining)
      phase: 'export',
    })
  );
});
```

**Expected Result**:
- Progress aggregated across phases
- Overall percent calculated correctly
- Phase information included

---

### MCP-PROG-002: Progress Checkpoints

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-PROG-002 |
| **Title** | Store Progress in Checkpoints |
| **Priority** | P1 - High |
| **Spec Reference** | Section 5.5 (Progress Persistence) |

**Test Steps**:

```typescript
it('stores progress in checkpoints for recovery', async () => {
  const checkpoints: Checkpoint[] = [];
  mockCheckpointStore.onCreate((cp) => checkpoints.push(cp));

  mockMCPServer((request) => {
    if (request.params.name === 'convert_pdf') {
      emitProgress({ percent: 25, stage: 'Stage 1' });
      emitProgress({ percent: 50, stage: 'Stage 2' });
      emitProgress({ percent: 75, stage: 'Stage 3' });
      return {
        result: { content: [{ type: 'document', document: mockDoclingDoc }] },
      };
    }
  });

  await workflowEngine.executeConversionWorkflow({
    file_path: '/tmp/test.pdf',
    jobId: 'test-job-123',
  });

  // Verify progress stored in checkpoints
  const progressCheckpoints = checkpoints.filter(
    cp => cp.type === 'progress'
  );

  expect(progressCheckpoints.length).toBeGreaterThanOrEqual(3);
  expect(progressCheckpoints[progressCheckpoints.length - 1].data.percent)
    .toBeGreaterThanOrEqual(75);
});
```

**Expected Result**:
- Progress checkpoints created
- Latest progress persisted
- Recovery can show progress

---

## 14. Protocol Compliance Tests

### 14.1 Request ID Correlation

---

### MCP-PROTO-001: Request ID Uniqueness

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-PROTO-001 |
| **Title** | Verify Request ID Uniqueness |
| **Priority** | P0 - Critical |
| **Spec Reference** | JSON-RPC 2.0 Specification |

**Test Steps**:

```typescript
it('generates unique request IDs for each request', async () => {
  const requestIds: string[] = [];

  mockMCPServer((request) => {
    if (request.id) {
      requestIds.push(request.id);
    }
    return { result: { content: [{ type: 'document', document: mockDoclingDoc }] } };
  });

  // Send multiple requests
  await Promise.all([
    client.invoke('convert_pdf', { file_path: '/tmp/test1.pdf' }),
    client.invoke('convert_pdf', { file_path: '/tmp/test2.pdf' }),
    client.invoke('convert_pdf', { file_path: '/tmp/test3.pdf' }),
  ]);

  // All IDs should be unique
  const uniqueIds = new Set(requestIds);
  expect(uniqueIds.size).toBe(requestIds.length);
});
```

**Expected Result**:
- Each request has unique ID
- No ID collisions

---

### MCP-PROTO-002: Request ID Correlation

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-PROTO-002 |
| **Title** | Verify Response ID Matches Request |
| **Priority** | P0 - Critical |
| **Spec Reference** | JSON-RPC 2.0 Specification |

**Test Steps**:

```typescript
it('correlates response ID with request ID', async () => {
  const requestResponsePairs: { requestId: string; responseId: string }[] = [];

  mockMCPServer((request) => {
    return {
      jsonrpc: '2.0',
      id: request.id, // Echo back request ID
      result: { content: [{ type: 'document', document: mockDoclingDoc }] },
    };
  });

  const interceptor = vi.fn((request, response) => {
    requestResponsePairs.push({
      requestId: request.id,
      responseId: response.id,
    });
  });

  client.addResponseInterceptor(interceptor);

  await client.invoke('convert_pdf', { file_path: '/tmp/test.pdf' });

  expect(requestResponsePairs[0].requestId).toBe(requestResponsePairs[0].responseId);
});
```

**Expected Result**:
- Response ID matches request ID
- Correlation validated in client

---

### MCP-PROTO-003: Mismatched ID Rejection

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-PROTO-003 |
| **Title** | Reject Response with Mismatched ID |
| **Priority** | P1 - High |
| **Spec Reference** | JSON-RPC 2.0 Specification |

**Test Steps**:

```typescript
it('rejects response with mismatched ID', async () => {
  mockMCPServer((request) => {
    return {
      jsonrpc: '2.0',
      id: 'wrong-id-12345', // Intentionally wrong ID
      result: { content: [{ type: 'document', document: mockDoclingDoc }] },
    };
  });

  await expect(client.invoke('convert_pdf', { file_path: '/tmp/test.pdf' }))
    .rejects.toMatchObject({
      code: 'E203',
      message: expect.stringContaining('ID mismatch'),
    });
});
```

**Expected Result**:
- Client detects mismatched ID
- Throws protocol error E203

---

### 14.2 Concurrent Request Handling

---

### MCP-CONC-001: Concurrent Request Isolation

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-CONC-001 |
| **Title** | Concurrent Requests Remain Isolated |
| **Priority** | P0 - Critical |
| **Spec Reference** | JSON-RPC 2.0 Specification |

**Test Steps**:

```typescript
it('isolates concurrent requests properly', async () => {
  const responses: Map<string, any> = new Map();

  mockMCPServer((request) => {
    const filePath = request.params.arguments.file_path;
    // Different delay for each file
    const delay = filePath.includes('fast') ? 50 : 200;

    return new Promise((resolve) => {
      setTimeout(() => {
        resolve({
          jsonrpc: '2.0',
          id: request.id,
          result: {
            content: [{
              type: 'document',
              document: { name: filePath },
            }],
          },
        });
      }, delay);
    });
  });

  const results = await Promise.all([
    client.invoke('convert_pdf', { file_path: '/tmp/fast.pdf' }),
    client.invoke('convert_pdf', { file_path: '/tmp/slow.pdf' }),
  ]);

  // Each response should match its request
  expect(results[0].content[0].document.name).toBe('/tmp/fast.pdf');
  expect(results[1].content[0].document.name).toBe('/tmp/slow.pdf');
});
```

**Expected Result**:
- Fast request completes first
- Each result matches its request
- No cross-contamination

---

### MCP-CONC-002: High Concurrency Stability

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-CONC-002 |
| **Title** | Handle High Concurrency Without Errors |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-PERF-001 |

**Test Steps**:

```typescript
it('handles 20 concurrent requests without errors', async () => {
  mockMCPServer((request) => {
    return {
      jsonrpc: '2.0',
      id: request.id,
      result: { content: [{ type: 'document', document: mockDoclingDoc }] },
    };
  });

  const requests = Array.from({ length: 20 }, (_, i) =>
    client.invoke('convert_pdf', { file_path: `/tmp/test${i}.pdf` })
  );

  const results = await Promise.allSettled(requests);

  const fulfilled = results.filter(r => r.status === 'fulfilled');
  expect(fulfilled.length).toBe(20);
});
```

**Expected Result**:
- All 20 requests complete successfully
- No race conditions or deadlocks

---

### 14.3 Notification Semantics

---

### MCP-NOTIF-001: Notification No Response Expected

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-NOTIF-001 |
| **Title** | Notifications Have No Request ID |
| **Priority** | P1 - High |
| **Spec Reference** | JSON-RPC 2.0 Specification (Notifications) |

**Test Steps**:

```typescript
it('sends notifications without request ID', async () => {
  let initNotification: any = null;

  mockMCPServer((request) => {
    if (request.method === 'notifications/initialized') {
      initNotification = request;
      return null; // No response for notifications
    }
    if (request.method === 'initialize') {
      return { result: { serverInfo: {}, capabilities: { tools: {} } } };
    }
  });

  await client.initialize();

  expect(initNotification).toBeDefined();
  expect(initNotification.id).toBeUndefined();
});
```

**Expected Result**:
- Notification has no `id` field
- Client does not wait for response

---

### MCP-NOTIF-002: Progress Notification Handling

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-NOTIF-002 |
| **Title** | Handle Server Progress Notifications |
| **Priority** | P1 - High |
| **Spec Reference** | MCP Protocol Progress Notifications |

**Test Steps**:

```typescript
it('handles progress notifications from server', async () => {
  const progressEvents: ProgressEvent[] = [];

  client.onProgress((event) => {
    progressEvents.push(event);
  });

  mockMCPServer((request) => {
    if (request.params.name === 'convert_pdf') {
      // Server sends progress notifications
      emitNotification({
        method: 'notifications/progress',
        params: { progressToken: request.id, progress: 25 },
      });
      emitNotification({
        method: 'notifications/progress',
        params: { progressToken: request.id, progress: 50 },
      });
      emitNotification({
        method: 'notifications/progress',
        params: { progressToken: request.id, progress: 100 },
      });

      return { result: { content: [{ type: 'document', document: mockDoclingDoc }] } };
    }
  });

  await client.invoke('convert_pdf', { file_path: '/tmp/test.pdf' });

  expect(progressEvents.length).toBe(3);
  expect(progressEvents[2].progress).toBe(100);
});
```

**Expected Result**:
- Progress notifications received
- Progress callback invoked
- Final progress is 100%

---

### 14.4 Server Lifecycle

---

### MCP-LIFE-001: Server Graceful Shutdown

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-LIFE-001 |
| **Title** | Handle Server Graceful Shutdown |
| **Priority** | P1 - High |
| **Spec Reference** | Section 4.7 (Infrastructure) |

**Test Steps**:

```typescript
it('handles server graceful shutdown notification', async () => {
  let shutdownHandled = false;

  client.onServerShutdown(() => {
    shutdownHandled = true;
  });

  // Emit server shutdown notification
  emitNotification({
    method: 'notifications/shutdown',
    params: { reason: 'Server maintenance' },
  });

  await vi.waitFor(() => {
    expect(shutdownHandled).toBe(true);
  });

  // Subsequent requests should fail gracefully
  await expect(client.invoke('convert_pdf', { file_path: '/tmp/test.pdf' }))
    .rejects.toMatchObject({
      code: 'E201',
      message: expect.stringContaining('shutting down'),
    });
});
```

**Expected Result**:
- Shutdown notification received
- Client marks server as unavailable
- New requests rejected gracefully

---

### MCP-LIFE-002: Reconnection After Disconnect

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-LIFE-002 |
| **Title** | Reconnect After Server Disconnect |
| **Priority** | P1 - High |
| **Spec Reference** | Section 7.1.2 (Resilience) |

**Test Steps**:

```typescript
it('reconnects automatically after server disconnect', async () => {
  let connectionAttempts = 0;

  mockMCPServer.onConnect(() => {
    connectionAttempts++;
    if (connectionAttempts === 1) {
      // First connection - simulate disconnect
      setTimeout(() => mockMCPServer.disconnect(), 100);
    }
  });

  mockMCPServer((request) => {
    if (request.method === 'initialize') {
      return { result: { serverInfo: {}, capabilities: { tools: {} } } };
    }
    return { result: { content: [{ type: 'document', document: mockDoclingDoc }] } };
  });

  const client = new MCPClient({ autoReconnect: true });
  await client.initialize();

  // Wait for disconnect and reconnect
  await vi.advanceTimersByTimeAsync(2000);

  expect(connectionAttempts).toBeGreaterThanOrEqual(2);

  // Should work after reconnect
  const result = await client.invoke('convert_pdf', { file_path: '/tmp/test.pdf' });
  expect(result.content[0].type).toBe('document');
});
```

**Expected Result**:
- Client detects disconnect
- Automatic reconnection attempted
- Operations resume after reconnect

---

### 14.5 Content-Type Validation

---

### MCP-CONTENT-001: JSON Content-Type Header

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-CONTENT-001 |
| **Title** | Verify JSON Content-Type Header |
| **Priority** | P1 - High |
| **Spec Reference** | JSON-RPC 2.0 over HTTP |

**Test Steps**:

```typescript
it('sends correct Content-Type header', async () => {
  let requestHeaders: Headers | null = null;

  mockMCPServer.onRequest((req) => {
    requestHeaders = req.headers;
  });

  await client.invoke('convert_pdf', { file_path: '/tmp/test.pdf' });

  expect(requestHeaders?.get('Content-Type')).toBe('application/json');
});
```

**Expected Result**:
- Content-Type is application/json
- Request properly formatted

---

### MCP-CONTENT-002: Accept Header

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-CONTENT-002 |
| **Title** | Verify Accept Header |
| **Priority** | P2 - Medium |
| **Spec Reference** | JSON-RPC 2.0 over HTTP |

**Test Steps**:

```typescript
it('sends correct Accept header', async () => {
  let requestHeaders: Headers | null = null;

  mockMCPServer.onRequest((req) => {
    requestHeaders = req.headers;
  });

  await client.invoke('convert_pdf', { file_path: '/tmp/test.pdf' });

  expect(requestHeaders?.get('Accept')).toBe('application/json');
});
```

**Expected Result**:
- Accept header is application/json
- Server compatibility ensured

---

## 15. End-to-End Integration Tests

### 15.1 Full Workflow Integration

---

### MCP-E2E-001: Complete Conversion Workflow

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-E2E-001 |
| **Title** | End-to-End Conversion Workflow |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-402, FR-404 (Full Pipeline) |

**Test Steps**:

```typescript
it('completes full conversion workflow from upload to export', async () => {
  // 1. Initialize MCP client
  const client = await getMCPClient();
  expect(client.isConnected()).toBe(true);

  // 2. Create job record
  const jobId = await jobService.createJob({
    filename: 'test.pdf',
    status: 'pending',
  });

  // 3. Convert document
  const conversionResult = await client.invoke('convert_pdf', {
    file_path: '/tmp/test.pdf',
  });
  expect(conversionResult.content[0].type).toBe('document');

  // 4. Update job with result
  await jobService.updateJob(jobId, {
    status: 'completed',
    result: conversionResult.content[0].document,
  });

  // 5. Export to all formats
  const [markdown, json] = await Promise.all([
    client.invoke('export_markdown', { document: conversionResult.content[0].document }),
    client.invoke('export_json', { document: conversionResult.content[0].document }),
  ]);

  expect(markdown.content[0].type).toBe('text');
  expect(json.content[0].type).toBe('text');

  // 6. Verify job record
  const finalJob = await jobService.getJob(jobId);
  expect(finalJob.status).toBe('completed');
});
```

**Expected Result**:
- Full pipeline executes end-to-end
- All stages complete successfully
- Job record reflects final state

---

### MCP-E2E-002: URL Conversion Workflow

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-E2E-002 |
| **Title** | End-to-End URL Conversion Workflow |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-201, FR-402 |

**Test Steps**:

```typescript
it('completes URL conversion workflow', async () => {
  const client = await getMCPClient();

  // 1. Convert URL
  const result = await client.invoke('convert_url', {
    url: 'https://example.com/document.html',
  });

  expect(result.content[0].document.origin.uri).toBe('https://example.com/document.html');

  // 2. Export result
  const markdown = await client.invoke('export_markdown', {
    document: result.content[0].document,
  });

  expect(markdown.content[0].text).toContain('#');
});
```

**Expected Result**:
- URL converted successfully
- Origin tracked in document
- Export produces valid output

---

### MCP-E2E-003: Error Recovery Workflow

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-E2E-003 |
| **Title** | End-to-End Error Recovery |
| **Priority** | P1 - High |
| **Spec Reference** | CRI-10 (Browser Refresh Recovery) |

**Test Steps**:

```typescript
it('recovers workflow after transient failure', async () => {
  let attemptCount = 0;

  mockMCPServer((request) => {
    if (request.params.name === 'convert_pdf') {
      attemptCount++;
      if (attemptCount < 3) {
        // Fail first 2 attempts
        return { error: { code: -32000, message: 'Server busy' } };
      }
      return { result: { content: [{ type: 'document', document: mockDoclingDoc }] } };
    }
  });

  // Create job
  const jobId = await jobService.createJob({
    filename: 'test.pdf',
    status: 'pending',
  });

  // Execute with retry
  const client = await getMCPClient();
  const result = await client.invoke('convert_pdf', { file_path: '/tmp/test.pdf' });

  expect(attemptCount).toBe(3);
  expect(result.content[0].type).toBe('document');

  // Job should be completed
  await jobService.updateJob(jobId, { status: 'completed' });
  const job = await jobService.getJob(jobId);
  expect(job.status).toBe('completed');
});
```

**Expected Result**:
- Retries occur on transient errors
- Final attempt succeeds
- Job state updated correctly

---

### 15.2 Quality Attribute Scenarios

---

### MCP-QA-001: Performance Under Normal Load

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-QA-001 |
| **Title** | Response Time Under Normal Load |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-PERF-001 (< 10 seconds) |

**Test Steps**:

```typescript
it('completes conversion within SLA under normal load', async () => {
  mockMCPServer((request) => {
    // Simulate realistic processing time
    return new Promise((resolve) => {
      setTimeout(() => {
        resolve({
          result: { content: [{ type: 'document', document: mockDoclingDoc }] },
        });
      }, 500); // 500ms simulated processing
    });
  });

  const client = await getMCPClient();

  const start = performance.now();
  await client.invoke('convert_pdf', { file_path: '/tmp/test.pdf' });
  const elapsed = performance.now() - start;

  expect(elapsed).toBeLessThan(10000); // < 10 seconds SLA
});
```

**Expected Result**:
- Response within 10 second SLA
- No timeout errors

---

### MCP-QA-002: Availability Under Failures

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-QA-002 |
| **Title** | System Availability With Partial Failures |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-REL-001 (99.5% availability) |

**Test Steps**:

```typescript
it('maintains availability despite intermittent failures', async () => {
  let requestCount = 0;

  mockMCPServer((request) => {
    requestCount++;
    // Fail 20% of requests (simulate partial failure)
    if (requestCount % 5 === 0) {
      return { error: { code: -32000, message: 'Server unavailable' } };
    }
    return { result: { content: [{ type: 'document', document: mockDoclingDoc }] } };
  });

  const client = await getMCPClient();

  // Execute 100 requests
  const results = await Promise.allSettled(
    Array.from({ length: 100 }, (_, i) =>
      client.invoke('convert_pdf', { file_path: `/tmp/test${i}.pdf` })
    )
  );

  const successful = results.filter(r => r.status === 'fulfilled').length;
  const successRate = successful / 100;

  // With retries, success rate should be high
  expect(successRate).toBeGreaterThanOrEqual(0.95);
});
```

**Expected Result**:
- >= 95% success rate
- Retries compensate for failures

---

### MCP-QA-003: Resource Cleanup

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-QA-003 |
| **Title** | Verify Resource Cleanup After Operations |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-SEC-002 (Resource Management) |

**Test Steps**:

```typescript
it('cleans up resources after operations complete', async () => {
  const client = await getMCPClient();

  // Execute multiple operations
  for (let i = 0; i < 10; i++) {
    await client.invoke('convert_pdf', { file_path: `/tmp/test${i}.pdf` });
  }

  // Check for resource leaks
  const activeConnections = client.getActiveConnectionCount();
  const pendingRequests = client.getPendingRequestCount();

  expect(activeConnections).toBeLessThanOrEqual(1); // Connection pooling
  expect(pendingRequests).toBe(0); // No pending requests
});
```

**Expected Result**:
- Connections properly pooled
- No pending requests after completion
- No resource leaks

---

### MCP-QA-004: Scalability Test

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-QA-004 |
| **Title** | Scalability Under Increasing Load |
| **Priority** | P2 - Medium |
| **Spec Reference** | NFR-PERF-002 (Scalability) |

**Test Steps**:

```typescript
it('maintains performance as load increases', async () => {
  mockMCPServer((request) => {
    return { result: { content: [{ type: 'document', document: mockDoclingDoc }] } };
  });

  const client = await getMCPClient();
  const responseTimes: number[] = [];

  // Test with increasing concurrency
  for (const concurrency of [1, 5, 10, 20]) {
    const start = performance.now();

    await Promise.all(
      Array.from({ length: concurrency }, (_, i) =>
        client.invoke('convert_pdf', { file_path: `/tmp/test${i}.pdf` })
      )
    );

    const elapsed = performance.now() - start;
    responseTimes.push(elapsed / concurrency); // Average per request
  }

  // Response time should not degrade significantly
  const degradation = responseTimes[3] / responseTimes[0];
  expect(degradation).toBeLessThan(3); // Max 3x degradation at 20x load
});
```

**Expected Result**:
- Linear or sub-linear scaling
- Response time degradation < 3x

---

### 15.3 SSE/MCP Integration

---

### MCP-SSE-006: MCP Progress to SSE E2E Mapping

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-SSE-006 |
| **Title** | Map MCP Progress to SSE Events (E2E) |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-503 (Real-time Progress) |

**Test Steps**:

```typescript
it('maps MCP progress to application SSE events', async () => {
  const sseEvents: SSEEvent[] = [];

  // Subscribe to SSE
  const eventSource = new EventSource('/api/v1/jobs/test-job-123/events');
  eventSource.onmessage = (event) => {
    sseEvents.push(JSON.parse(event.data));
  };

  // Trigger conversion with progress
  mockMCPServer((request) => {
    if (request.params.name === 'convert_pdf') {
      emitNotification({ method: 'notifications/progress', params: { progress: 25 } });
      emitNotification({ method: 'notifications/progress', params: { progress: 50 } });
      emitNotification({ method: 'notifications/progress', params: { progress: 100 } });
      return { result: { content: [{ type: 'document', document: mockDoclingDoc }] } };
    }
  });

  await client.invoke('convert_pdf', { file_path: '/tmp/test.pdf' });

  // Wait for SSE events
  await vi.waitFor(() => {
    expect(sseEvents.length).toBeGreaterThanOrEqual(3);
  });

  expect(sseEvents.some(e => e.type === 'progress')).toBe(true);
  eventSource.close();
});
```

**Expected Result**:
- MCP progress events mapped to SSE
- Client receives real-time updates

---

### MCP-SSE-007: SSE Channel Separation

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-SSE-007 |
| **Title** | SSE and MCP Channels Are Independent |
| **Priority** | P1 - High |
| **Spec Reference** | Section 7.1.1 (Architecture) |

**Test Steps**:

```typescript
it('maintains SSE when MCP has transient errors', async () => {
  const sseEvents: SSEEvent[] = [];

  const eventSource = new EventSource('/api/v1/jobs/test-job-123/events');
  eventSource.onmessage = (event) => {
    sseEvents.push(JSON.parse(event.data));
  };

  // MCP fails but SSE should continue
  mockMCPServer((request) => {
    return { error: { code: -32000, message: 'Server unavailable' } };
  });

  try {
    await client.invoke('convert_pdf', { file_path: '/tmp/test.pdf' });
  } catch (e) {
    // Expected failure
  }

  // SSE should receive error event
  await vi.waitFor(() => {
    expect(sseEvents.some(e => e.type === 'error')).toBe(true);
  });

  // SSE connection should still be alive
  expect(eventSource.readyState).toBe(EventSource.OPEN);
  eventSource.close();
});
```

**Expected Result**:
- SSE receives error notification
- SSE connection remains open
- Channels are independent

---

### MCP-SSE-008: Job Completion via SSE

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-SSE-008 |
| **Title** | SSE Notifies Job Completion (E2E) |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-505 (Job Completion Notification) |

**Test Steps**:

```typescript
it('sends completion event via SSE after MCP success', async () => {
  const sseEvents: SSEEvent[] = [];

  const eventSource = new EventSource('/api/v1/jobs/test-job-123/events');
  eventSource.onmessage = (event) => {
    sseEvents.push(JSON.parse(event.data));
  };

  mockMCPServer((request) => {
    return { result: { content: [{ type: 'document', document: mockDoclingDoc }] } };
  });

  await client.invoke('convert_pdf', { file_path: '/tmp/test.pdf' });

  await vi.waitFor(() => {
    expect(sseEvents.some(e => e.type === 'completed')).toBe(true);
  });

  const completionEvent = sseEvents.find(e => e.type === 'completed');
  expect(completionEvent?.data.jobId).toBe('test-job-123');
  eventSource.close();
});
```

**Expected Result**:
- Completion event sent via SSE
- Contains job ID and result reference

---

## 16. Authentication & Security Tests

### 16.1 Authentication Header Injection

---

### MCP-AUTH-001: Bearer Token Injection

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-AUTH-001 |
| **Title** | Inject Bearer Token in MCP Requests |
| **Priority** | P0 - Critical |
| **Spec Reference** | NFR-SEC-001 (Authentication) |

**Test Steps**:

```typescript
it('injects bearer token in all MCP requests', async () => {
  let authHeader: string | null = null;

  mockMCPServer.onRequest((req) => {
    authHeader = req.headers.get('Authorization');
  });

  const client = new MCPClient({
    authToken: 'test-jwt-token-12345',
  });
  await client.initialize();

  await client.invoke('convert_pdf', { file_path: '/tmp/test.pdf' });

  expect(authHeader).toBe('Bearer test-jwt-token-12345');
});
```

**Expected Result**:
- Authorization header present
- Bearer token format correct

---

### MCP-AUTH-002: Token Refresh

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-AUTH-002 |
| **Title** | Refresh Token on 401 Response |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-SEC-001 (Token Management) |

**Test Steps**:

```typescript
it('refreshes token on 401 and retries', async () => {
  let requestCount = 0;
  const tokens: string[] = [];

  mockMCPServer((request, headers) => {
    requestCount++;
    tokens.push(headers.get('Authorization') || '');

    if (requestCount === 1) {
      // First request - token expired
      return { error: { code: -32000, message: 'Unauthorized', data: { httpStatus: 401 } } };
    }
    return { result: { content: [{ type: 'document', document: mockDoclingDoc }] } };
  });

  const mockRefresh = vi.fn().mockResolvedValue('new-token-67890');

  const client = new MCPClient({
    authToken: 'old-token-12345',
    onTokenRefresh: mockRefresh,
  });

  const result = await client.invoke('convert_pdf', { file_path: '/tmp/test.pdf' });

  expect(mockRefresh).toHaveBeenCalled();
  expect(tokens[1]).toBe('Bearer new-token-67890');
  expect(result.content[0].type).toBe('document');
});
```

**Expected Result**:
- Token refresh triggered on 401
- Request retried with new token
- Operation succeeds

---

### MCP-AUTH-003: Missing Token Handling

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-AUTH-003 |
| **Title** | Handle Missing Authentication Token |
| **Priority** | P1 - High |
| **Spec Reference** | E101 (Authentication Error) |

**Test Steps**:

```typescript
it('throws authentication error when token missing', async () => {
  const client = new MCPClient({
    requireAuth: true,
    // No authToken provided
  });

  await expect(client.invoke('convert_pdf', { file_path: '/tmp/test.pdf' }))
    .rejects.toMatchObject({
      code: 'E101',
      message: expect.stringContaining('authentication'),
    });
});
```

**Expected Result**:
- Error thrown before request
- E101 authentication error code

---

### 16.2 Input Sanitization

---

### MCP-SEC-001: Path Traversal Prevention

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-SEC-001 |
| **Title** | Prevent Path Traversal Attacks |
| **Priority** | P0 - Critical |
| **Spec Reference** | NFR-SEC-003 (Input Validation) |

**Test Steps**:

```typescript
it('rejects path traversal attempts', async () => {
  const maliciousPaths = [
    '../../../etc/passwd',
    '/tmp/../../../etc/shadow',
    '....//....//etc/passwd',
    '/tmp/test.pdf%00.txt',
  ];

  for (const path of maliciousPaths) {
    await expect(client.invoke('convert_pdf', { file_path: path }))
      .rejects.toMatchObject({
        code: 'E301',
        message: expect.stringContaining('Invalid file path'),
      });
  }
});
```

**Expected Result**:
- Path traversal blocked
- E301 validation error returned

---

### MCP-SEC-002: URL Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-SEC-002 |
| **Title** | Validate URL Input |
| **Priority** | P0 - Critical |
| **Spec Reference** | NFR-SEC-003 (Input Validation) |

**Test Steps**:

```typescript
it('rejects malicious URLs', async () => {
  const maliciousUrls = [
    'javascript:alert(1)',
    'file:///etc/passwd',
    'ftp://malicious.com/payload',
    'data:text/html,<script>alert(1)</script>',
  ];

  for (const url of maliciousUrls) {
    await expect(client.invoke('convert_url', { url }))
      .rejects.toMatchObject({
        code: 'E301',
        message: expect.stringContaining('Invalid URL'),
      });
  }
});
```

**Expected Result**:
- Only HTTP/HTTPS URLs accepted
- Malicious schemes rejected

---

### MCP-SEC-003: Document Size Limits

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-SEC-003 |
| **Title** | Enforce Document Size Limits |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-SEC-004 (Resource Protection) |

**Test Steps**:

```typescript
it('rejects documents exceeding size limit', async () => {
  // Create oversized document mock
  const oversizedDocument = {
    schema_name: 'docling_core.transforms.v1.DoclingDocument',
    body: { content: 'x'.repeat(100 * 1024 * 1024) }, // 100MB
  };

  await expect(client.invoke('export_markdown', { document: oversizedDocument }))
    .rejects.toMatchObject({
      code: 'E301',
      message: expect.stringContaining('size limit'),
    });
});
```

**Expected Result**:
- Size limit enforced
- E301 validation error for oversized documents

---

### 16.3 Connection Security

---

### MCP-SEC-004: TLS Enforcement

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-SEC-004 |
| **Title** | Enforce TLS for MCP Connections |
| **Priority** | P0 - Critical |
| **Spec Reference** | NFR-SEC-002 (Transport Security) |

**Test Steps**:

```typescript
it('rejects non-TLS connections in production', async () => {
  process.env.NODE_ENV = 'production';

  const client = new MCPClient({
    endpoint: 'http://insecure-server:8000', // HTTP, not HTTPS
  });

  await expect(client.initialize())
    .rejects.toMatchObject({
      code: 'E101',
      message: expect.stringContaining('TLS required'),
    });
});
```

**Expected Result**:
- Non-TLS connections rejected in production
- Security error thrown

---

### MCP-SEC-005: Certificate Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-SEC-005 |
| **Title** | Validate Server Certificate |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-SEC-002 (Certificate Pinning) |

**Test Steps**:

```typescript
it('rejects invalid server certificates', async () => {
  // Mock server with self-signed certificate
  mockMCPServer.useSelfSignedCert();

  const client = new MCPClient({
    endpoint: 'https://localhost:8443',
    validateCertificate: true,
  });

  await expect(client.initialize())
    .rejects.toMatchObject({
      code: 'E201',
      message: expect.stringContaining('certificate'),
    });
});
```

**Expected Result**:
- Invalid certificates rejected
- Connection not established

---

### 16.3 SSRF Prevention Tests

The following tests validate Server-Side Request Forgery (SSRF) prevention for the `convert_url` MCP tool, ensuring that internal network addresses and potentially malicious URLs are blocked before being processed.

**Reference**: FR-203 (SSRF Prevention), NFR-402 (Blocked Private IP Requests)

---

### MCP-SEC-006: Block Localhost URL

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-SEC-006 |
| **Title** | Block Localhost/127.0.0.1 URLs |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-203, NFR-402 |

**Test Steps**:

```typescript
describe('SSRF Prevention - Localhost', () => {
  const localhostVariants = [
    'http://localhost/document.html',
    'http://localhost:8080/document.html',
    'http://127.0.0.1/document.html',
    'http://127.0.0.1:3000/api/internal',
    'http://127.0.0.1:8000/document.pdf',
    'http://[::1]/document.html',          // IPv6 localhost
    'http://0.0.0.0/document.html',
    'http://0.0.0.0:8080/document.html',
  ];

  localhostVariants.forEach((url) => {
    it(`blocks localhost URL: ${url}`, async () => {
      const response = await fetch('/api/v1/process', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ url }),
      });

      expect(response.status).toBe(400);

      const error = await response.json();
      expect(error).toMatchObject({
        code: 'E104',
        message: expect.stringContaining('blocked'),
        details: expect.objectContaining({
          reason: 'private_ip',
          url: url,
        }),
      });
    });
  });

  it('blocks localhost with URL encoding bypass attempts', async () => {
    const bypassAttempts = [
      'http://localhost%00.evil.com/document.html',
      'http://127.0.0.1%00.evil.com/document.html',
      'http://localhost%2F@evil.com/document.html',
      'http://localhost%252F@evil.com/document.html',
    ];

    for (const url of bypassAttempts) {
      const response = await fetch('/api/v1/process', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ url }),
      });

      expect(response.status).toBe(400);

      const error = await response.json();
      expect(error.code).toBe('E104');
    }
  });
});
```

**Expected Result**:
- All localhost variants blocked with E104 error
- URL encoding bypass attempts detected and blocked
- Clear error message indicating URL is blocked

---

### MCP-SEC-007: Block Private IPv4 Ranges (RFC 1918)

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-SEC-007 |
| **Title** | Block RFC 1918 Private IP Ranges |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-203, NFR-402 |

**Test Steps**:

```typescript
describe('SSRF Prevention - RFC 1918 Private Ranges', () => {
  // 10.0.0.0/8 range (10.0.0.0 - 10.255.255.255)
  const class10Addresses = [
    'http://10.0.0.1/document.html',
    'http://10.0.0.0/document.html',
    'http://10.255.255.255/document.html',
    'http://10.1.2.3:8080/api/internal',
    'http://10.100.50.25/secret.pdf',
  ];

  // 172.16.0.0/12 range (172.16.0.0 - 172.31.255.255)
  const class172Addresses = [
    'http://172.16.0.1/document.html',
    'http://172.16.0.0/document.html',
    'http://172.31.255.255/document.html',
    'http://172.20.10.5:3000/api/data',
    'http://172.24.100.200/internal.pdf',
  ];

  // 192.168.0.0/16 range (192.168.0.0 - 192.168.255.255)
  const class192Addresses = [
    'http://192.168.0.1/document.html',
    'http://192.168.0.0/document.html',
    'http://192.168.255.255/document.html',
    'http://192.168.1.1/admin',
    'http://192.168.10.224:8000/mcp',  // hx-cc-server
  ];

  const allPrivateAddresses = [
    ...class10Addresses,
    ...class172Addresses,
    ...class192Addresses,
  ];

  allPrivateAddresses.forEach((url) => {
    it(`blocks private IP URL: ${url}`, async () => {
      const response = await fetch('/api/v1/process', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ url }),
      });

      expect(response.status).toBe(400);

      const error = await response.json();
      expect(error).toMatchObject({
        code: 'E104',
        message: expect.stringContaining('blocked'),
        details: expect.objectContaining({
          reason: 'private_ip',
        }),
      });
    });
  });

  it('correctly identifies boundary addresses', async () => {
    // These should be BLOCKED (just inside private range)
    const blockedBoundary = [
      'http://172.16.0.0/test',    // Start of 172.16.0.0/12
      'http://172.31.255.255/test', // End of 172.16.0.0/12
    ];

    // These should be ALLOWED (just outside private range)
    const allowedBoundary = [
      'http://172.15.255.255/test', // Just before 172.16.0.0/12
      'http://172.32.0.0/test',     // Just after 172.16.0.0/12
    ];

    for (const url of blockedBoundary) {
      const response = await fetch('/api/v1/process', {
        method: 'POST',
        body: JSON.stringify({ url }),
        headers: { 'Content-Type': 'application/json' },
      });
      expect(response.status).toBe(400);
    }

    // Note: allowedBoundary URLs may still fail for other reasons (unreachable)
    // but should not fail with E104 SSRF error
    for (const url of allowedBoundary) {
      const response = await fetch('/api/v1/process', {
        method: 'POST',
        body: JSON.stringify({ url }),
        headers: { 'Content-Type': 'application/json' },
      });
      if (response.status === 400) {
        const error = await response.json();
        expect(error.code).not.toBe('E104');
      }
    }
  });
});
```

**Expected Result**:
- All RFC 1918 private IP ranges blocked
- 10.x.x.x range (Class A) blocked completely
- 172.16-31.x.x range (Class B) blocked with correct boundaries
- 192.168.x.x range (Class C) blocked completely
- E104 error returned with private_ip reason

---

### MCP-SEC-008: Block Link-Local Addresses (169.254.x.x)

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-SEC-008 |
| **Title** | Block Link-Local Addresses (169.254.0.0/16) |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-203, NFR-402 |

**Test Steps**:

```typescript
describe('SSRF Prevention - Link-Local Addresses', () => {
  const linkLocalAddresses = [
    'http://169.254.0.1/document.html',
    'http://169.254.0.0/document.html',
    'http://169.254.255.255/document.html',
    'http://169.254.169.254/latest/meta-data/',  // AWS metadata endpoint
    'http://169.254.169.254/metadata/instance',   // Azure metadata endpoint
    'http://169.254.169.254/computeMetadata/v1/', // GCP metadata endpoint
  ];

  linkLocalAddresses.forEach((url) => {
    it(`blocks link-local URL: ${url}`, async () => {
      const response = await fetch('/api/v1/process', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ url }),
      });

      expect(response.status).toBe(400);

      const error = await response.json();
      expect(error).toMatchObject({
        code: 'E104',
        message: expect.stringContaining('blocked'),
        details: expect.objectContaining({
          reason: 'private_ip',
        }),
      });
    });
  });

  it('specifically blocks cloud provider metadata endpoints', async () => {
    const metadataEndpoints = [
      { url: 'http://169.254.169.254/latest/meta-data/', provider: 'AWS' },
      { url: 'http://169.254.169.254/latest/user-data/', provider: 'AWS' },
      { url: 'http://169.254.169.254/metadata/instance?api-version=2021-02-01', provider: 'Azure' },
      { url: 'http://169.254.169.254/computeMetadata/v1/instance/', provider: 'GCP' },
    ];

    for (const { url, provider } of metadataEndpoints) {
      const response = await fetch('/api/v1/process', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ url }),
      });

      expect(response.status).toBe(400);

      const error = await response.json();
      expect(error.code).toBe('E104');
      // Log for security audit
      console.log(`[SSRF-TEST] Blocked ${provider} metadata endpoint: ${url}`);
    }
  });
});
```

**Expected Result**:
- All 169.254.x.x addresses blocked
- Cloud provider metadata endpoints (AWS, Azure, GCP) blocked
- E104 error returned with clear explanation

---

### MCP-SEC-009: Block Internal Hostnames

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-SEC-009 |
| **Title** | Block Internal Domain Names (*.hx.dev.local) |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-203, NFR-402 |

**Test Steps**:

```typescript
describe('SSRF Prevention - Internal Hostnames', () => {
  const internalHostnames = [
    'http://hx-docling-mcp-server.hx.dev.local:8000/document',
    'http://hx-postgres-server.hx.dev.local:5432/',
    'http://hx-redis-server.hx.dev.local:6379/',
    'http://hx-cc-server.hx.dev.local/',
    'http://internal.hx.dev.local/secret',
    'http://admin.hx.dev.local:8080/admin',
    'http://api.hx.dev.local/v1/internal',
  ];

  internalHostnames.forEach((url) => {
    it(`blocks internal hostname: ${url}`, async () => {
      const response = await fetch('/api/v1/process', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ url }),
      });

      expect(response.status).toBe(400);

      const error = await response.json();
      expect(error).toMatchObject({
        code: 'E104',
        message: expect.stringContaining('blocked'),
        details: expect.objectContaining({
          reason: 'internal_hostname',
        }),
      });
    });
  });

  it('blocks common internal hostnames', async () => {
    const commonInternalNames = [
      'http://internal/document.html',
      'http://intranet/document.html',
      'http://localhost.localdomain/document.html',
      'http://corp/document.html',
    ];

    for (const url of commonInternalNames) {
      const response = await fetch('/api/v1/process', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ url }),
      });

      // Should fail with either E104 (blocked) or E101 (invalid URL)
      expect(response.status).toBe(400);
    }
  });
});
```

**Expected Result**:
- All *.hx.dev.local hostnames blocked
- Internal infrastructure hostnames blocked
- E104 error returned with internal_hostname reason

---

### MCP-SEC-010: DNS Rebinding Prevention

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-SEC-010 |
| **Title** | Prevent DNS Rebinding Attacks |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-203, NFR-402, OWASP SSRF Prevention |

**Test Steps**:

```typescript
describe('SSRF Prevention - DNS Rebinding', () => {
  it('validates IP at request time, not just at URL parse time', async () => {
    // This test requires a mock DNS that can simulate rebinding
    // The URL initially resolves to public IP, then to private IP

    mockDNS.setResolution('rebind.example.com', {
      first: '93.184.216.34',   // Example public IP
      subsequent: '127.0.0.1',   // Localhost (private)
    });

    const url = 'http://rebind.example.com/document.html';

    const response = await fetch('/api/v1/process', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ url }),
    });

    // If implementation properly validates at request time, should block
    // when the resolved IP is private
    if (response.status === 400) {
      const error = await response.json();
      expect(error.code).toBe('E104');
    }

    // Verify the validation occurred at the correct stage
    expect(mockDNS.getResolutionCount('rebind.example.com')).toBeGreaterThanOrEqual(1);
  });

  it('blocks URLs with numeric IP embedded in hostname', async () => {
    const embeddedIPUrls = [
      'http://127.0.0.1.xip.io/document.html',
      'http://192.168.1.1.nip.io/document.html',
      'http://10.0.0.1.sslip.io/document.html',
      'http://A.10.0.0.1.1time.com/document.html',
    ];

    for (const url of embeddedIPUrls) {
      // Note: These may resolve to private IPs via wildcard DNS
      // Implementation should resolve and re-validate the final IP

      const response = await fetch('/api/v1/process', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ url }),
      });

      // Should be blocked if DNS resolution returns private IP
      expect(response.status).toBe(400);
    }
  });

  it('handles TTL-based DNS rebinding attempts', async () => {
    // Simulate short TTL that allows rebinding between validation and fetch
    mockDNS.setResolutionWithTTL('ttl-rebind.example.com', {
      ip: '93.184.216.34',
      ttl: 0, // Zero TTL - allows immediate rebinding
    });

    const url = 'http://ttl-rebind.example.com/document.html';

    // First request should establish validation
    const response1 = await fetch('/api/v1/process', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ url }),
    });

    // Change DNS resolution to private IP
    mockDNS.setResolutionWithTTL('ttl-rebind.example.com', {
      ip: '192.168.1.1',
      ttl: 0,
    });

    // Second request - implementation should re-validate
    const response2 = await fetch('/api/v1/process', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ url }),
    });

    expect(response2.status).toBe(400);
    const error = await response2.json();
    expect(error.code).toBe('E104');
  });
});
```

**Expected Result**:
- DNS resolution validated at request time
- Rebinding attacks detected and blocked
- Embedded IP hostnames (xip.io, nip.io) validated after resolution
- TTL-based rebinding attempts prevented

---

### MCP-SEC-011: URL Scheme Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | MCP-SEC-011 |
| **Title** | Validate URL Scheme (HTTP/HTTPS Only) |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-202, FR-203, NFR-402 |

**Test Steps**:

```typescript
describe('SSRF Prevention - URL Scheme Validation', () => {
  it('only allows http and https schemes', async () => {
    const validSchemes = [
      { url: 'http://example.com/document.html', expected: 'allowed' },
      { url: 'https://example.com/document.html', expected: 'allowed' },
    ];

    for (const { url, expected } of validSchemes) {
      const response = await fetch('/api/v1/process', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ url }),
      });

      // Should not fail with E101 (scheme error) - may fail for other reasons
      if (response.status === 400) {
        const error = await response.json();
        expect(error.code).not.toBe('E101');
      }
    }
  });

  it('blocks dangerous URL schemes', async () => {
    const dangerousSchemes = [
      'file:///etc/passwd',
      'file:///C:/Windows/System32/config/SAM',
      'ftp://internal-ftp.local/secret.pdf',
      'gopher://evil.com:80/_GET%20/etc/passwd',
      'dict://evil.com:1234/d:test',
      'ldap://evil.com:389/dc=example,dc=com',
      'sftp://internal.local/document.pdf',
      'tftp://internal.local/config.txt',
      'data:text/html,<script>alert(1)</script>',
      'javascript:alert(document.domain)',
    ];

    for (const url of dangerousSchemes) {
      const response = await fetch('/api/v1/process', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ url }),
      });

      expect(response.status).toBe(400);

      const error = await response.json();
      expect(error.code).toBe('E101');
      expect(error.message).toMatch(/invalid.*scheme|unsupported.*protocol/i);
    }
  });

  it('blocks protocol-relative URLs that could be manipulated', async () => {
    const protocolRelativeUrls = [
      '//evil.com/document.html',
      '///evil.com/document.html',
    ];

    for (const url of protocolRelativeUrls) {
      const response = await fetch('/api/v1/process', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ url }),
      });

      expect(response.status).toBe(400);

      const error = await response.json();
      expect(['E101', 'E104']).toContain(error.code);
    }
  });

  it('handles scheme case variations', async () => {
    const caseVariations = [
      'HTTP://example.com/document.html',
      'HTTPS://example.com/document.html',
      'Http://example.com/document.html',
      'HtTpS://example.com/document.html',
    ];

    for (const url of caseVariations) {
      const response = await fetch('/api/v1/process', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ url }),
      });

      // Should be treated as valid http/https
      if (response.status === 400) {
        const error = await response.json();
        expect(error.code).not.toBe('E101');
      }
    }
  });
});
```

**Expected Result**:
- Only http:// and https:// schemes allowed
- file://, ftp://, gopher://, dict://, ldap://, sftp://, tftp://, data:, javascript: schemes blocked
- Protocol-relative URLs rejected
- Scheme case variations handled correctly
- E101 error returned for invalid schemes

---

## Test Summary

### Test Count by Category

| Category | Test Count |
|----------|-----------|
| MCP Protocol Tests (Sec 2) | 5 |
| Conversion Tool Tests (Sec 3) | 6 |
| Export Tool Tests (Sec 4) | 5 |
| Error Handling Tests (Sec 5) | 10 |
| Timeout and Retry Tests (Sec 6) | 10 |
| Schema Validation Tests (Sec 7) | 4 |
| Database Transaction Tests (Sec 8) | 8 |
| Redis Integration Tests (Sec 9) | 12 |
| Frontend Integration Tests (Sec 10) | 8 |
| Infrastructure & Operations Tests (Sec 11) | 14 |
| Next.js Integration Tests (Sec 12) | 17 |
| Workflow Orchestration Tests (Sec 13) | 14 |
| Protocol Compliance Tests (Sec 14) | 11 |
| End-to-End Integration Tests (Sec 15) | 10 |
| Authentication & Security Tests (Sec 16) | 8 |
| SSRF Prevention Tests (Sec 16.3) | 6 |
| **Total** | **148** |

### Test Count by Priority

| Priority | Count |
|----------|-------|
| P0 - Critical | 66 |
| P1 - High | 68 |
| P2 - Medium | 14 |
| **Total** | **148** |

### MCP Tool Coverage

| Tool | Tests |
|------|-------|
| convert_pdf | MCP-CONV-001, MCP-SCH-001 |
| convert_docx | MCP-CONV-002 |
| convert_xlsx | MCP-CONV-003 |
| convert_pptx | MCP-CONV-004 |
| convert_url | MCP-CONV-005, MCP-SCH-002 |
| export_markdown | MCP-EXP-001 |
| export_html | MCP-EXP-002 |
| export_json | MCP-EXP-003 |

### Error Code Coverage

| Error Code | Description | Test IDs |
|------------|-------------|----------|
| E201 | MCP Server unavailable | MCP-ERR-004, MCP-ERR-005, MCP-ERR-009, MCP-ERR-010 |
| E202 | Connection timeout | MCP-ERR-006, MCP-TO-002 |
| E203 | Invalid parameters | MCP-ERR-001, MCP-ERR-003 |
| E204 | Tool not found | MCP-ERR-002 |
| E205 | Missing tools capability | MCP-INIT-004 |
| E302 | Conversion failed | MCP-ERR-007 |
| E303 | Export failed | MCP-ERR-008 |

---

## Changelog

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0.0 | 2025-12-12 | Julia Santos | Initial MCP test cases (35 tests) |
| 1.1.0 | 2025-12-14 | Julia Santos | Added Database Transaction Tests (8 tests), Redis Integration Tests (12 tests), and Frontend Integration Tests (12 tests) based on consolidated MCP review gap analysis |
| 1.2.0 | 2025-12-14 | Julia Santos | Added Infrastructure & Operations Tests (14 tests), Next.js Integration Tests (13 tests), and Workflow Orchestration Tests (13 tests) based on consolidated MCP review gap analysis |
| 2.0.0 | 2025-12-14 | Julia Santos | **FINAL REMEDIATION** - Added Protocol Compliance Tests (11 tests), End-to-End Integration Tests (10 tests), and Authentication & Security Tests (8 tests). All gap domains from consolidated 10-specialist review now addressed. Total: 142 unique tests. Status: APPROVED |
| 2.1.0 | 2025-12-15 | Julia Santos | **BLOCKER REMEDIATION** - Added SSRF Prevention Tests (MCP-SEC-006 through MCP-SEC-011): 6 comprehensive test cases covering localhost blocking (SEC-006), RFC 1918 private IP ranges (SEC-007), link-local/cloud metadata endpoints (SEC-008), internal hostnames (SEC-009), DNS rebinding prevention (SEC-010), and URL scheme validation (SEC-011). Addresses H-08 high priority issue. Total: 148 unique tests. |
| 2.2.0 | 2025-12-15 | Julia Santos | **A-07 REMEDIATION** - Added MCP Error Recovery Tests (Section 5.3, MCP-ERR-011 through MCP-ERR-018): 8 comprehensive test cases covering transient error retry with exponential backoff (ERR-011), permanent error fail-fast detection (ERR-012), partial failure handling in batch operations (ERR-013), error categorization for retry logic (ERR-014), error context preservation across retries (ERR-015), circuit breaker integration (ERR-016), timeout cascade prevention (ERR-017), and complete error recovery state machine validation (ERR-018). All tests reference Section 7.1.4, FR-803, and Section 7.4.3. Total: 156 unique tests. |
