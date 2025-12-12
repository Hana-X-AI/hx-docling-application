# Neo Next.js Tasks: Sprint 1.5a - MCP Client (Support Role)

**Sprint**: 1.5a - MCP Client (Core)
**Duration**: ~4.0 hours (Sprint Total)
**Role**: Support Developer
**Agent**: Neo (Next.js Senior Developer)
**Lead**: James (@james)
**Support**: Neo (@neo), George (@george)
**Review**: Alex (@alex)

---

## Development Tools

### shadcn/ui MCP Server Reference

For component source code and metadata reference during API integration work, the **hx-shadcn MCP Server** is available:

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

**Example MCP Component Retrieval:**
```json
{
  "method": "tools/call",
  "params": {
    "name": "get_component",
    "arguments": {"componentName": "button"}
  }
}
```

**Note:** Sprint 1.5a is primarily backend/API focused. The hx-shadcn MCP server is referenced here for completeness and may be useful for understanding MCP protocol patterns.

---

## Overview

Sprint 1.5a focuses on MCP client implementation. Neo provides support for Next.js API route integration and TypeScript type definitions. The primary implementation is led by James (Docling MCP Integration specialist).

---

## Neo's Support Tasks

### NEO-1.5a-001: Create MCP Types for Next.js Integration

**Priority**: P1 (High)
**Effort**: 20 minutes
**Dependencies**: Sprint 1.4 Complete
**Lead**: James, Support: Neo

**Description**:
Create TypeScript type definitions for MCP client integration with Next.js API routes. These types bridge the MCP protocol with Next.js request/response handling.

**Acceptance Criteria**:
- [ ] MCP request/response types defined
- [ ] Tool parameter types for all 8 tools
- [ ] Error types aligned with API error catalog
- [ ] Types exported from central location
- [ ] Types compatible with JSON serialization

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/types/mcp.ts`

**Technical Notes**:
```typescript
// src/types/mcp.ts

/**
 * JSON-RPC 2.0 types for MCP protocol
 */
export interface JsonRpcRequest {
  jsonrpc: '2.0';
  id: string;
  method: string;
  params?: Record<string, unknown>;
}

export interface JsonRpcResponse<T = unknown> {
  jsonrpc: '2.0';
  id: string;
  result?: T;
  error?: JsonRpcError;
}

export interface JsonRpcError {
  code: number;
  message: string;
  data?: unknown;
}

/**
 * MCP initialization types
 */
export interface MCPClientInfo {
  name: string;
  version: string;
}

export interface MCPServerCapabilities {
  tools?: { listChanged?: boolean };
  resources?: { subscribe?: boolean };
  prompts?: { listChanged?: boolean };
}

export interface MCPInitializeResponse {
  protocolVersion: string;
  capabilities: MCPServerCapabilities;
  serverInfo: {
    name: string;
    version: string;
  };
}

/**
 * Tool definitions for 8 supported MCP tools
 */
export type MCPToolName =
  | 'convert_pdf'
  | 'convert_docx'
  | 'convert_xlsx'
  | 'convert_pptx'
  | 'convert_url'
  | 'export_markdown'
  | 'export_html'
  | 'export_json';

export interface MCPToolCallParams {
  convert_pdf: { file_path: string };
  convert_docx: { file_path: string };
  convert_xlsx: { file_path: string };
  convert_pptx: { file_path: string };
  convert_url: { url: string };
  export_markdown: { document: DoclingDocument };
  export_html: { document: DoclingDocument };
  export_json: { document: DoclingDocument };
}

/**
 * DoclingDocument type (simplified)
 */
export interface DoclingDocument {
  name: string;
  origin: {
    filename?: string;
    mimetype?: string;
    uri?: string;
  };
  furniture: {
    pages: Array<{
      size: { width: number; height: number };
      page_no: number;
    }>;
  };
  body: {
    children: DoclingElement[];
  };
}

export interface DoclingElement {
  type: string;
  text?: string;
  children?: DoclingElement[];
  // ... additional element properties
}

/**
 * Processing result types
 */
export interface ProcessingResult {
  markdown?: string;
  html?: string;
  json?: string;
  raw?: DoclingDocument;
}

export interface ProcessingError {
  code: string;
  message: string;
  retryable: boolean;
  stage?: string;
}
```

---

### NEO-1.5a-002: Create API Route Wrapper for MCP Calls

**Priority**: P1 (High)
**Effort**: 30 minutes
**Dependencies**: NEO-1.5a-001
**Lead**: James, Support: Neo

**Description**:
Create a Next.js API route utility that wraps MCP client calls with proper error handling, request ID tracking, and response formatting.

**Acceptance Criteria**:
- [ ] Wrapper handles MCP client initialization
- [ ] Request ID (X-Request-ID) propagated to MCP
- [ ] JSON-RPC errors mapped to HTTP status codes
- [ ] Timeout handling integrated
- [ ] TypeScript types enforced

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/lib/mcp/api-wrapper.ts`

**Technical Notes**:
```typescript
// src/lib/mcp/api-wrapper.ts

import { NextRequest, NextResponse } from 'next/server';
import { randomUUID } from 'crypto';
import { MCPClient } from './client';
import { mapMCPErrorToHTTP } from './error-mapping';
import type { MCPToolName, MCPToolCallParams, JsonRpcError } from '@/types/mcp';

interface MCPCallOptions {
  timeout?: number;
  requestId?: string;
}

/**
 * Execute MCP tool call with proper error handling for API routes
 */
export async function executeMCPTool<T extends MCPToolName>(
  toolName: T,
  params: MCPToolCallParams[T],
  options: MCPCallOptions = {}
): Promise<{ success: true; result: unknown } | { success: false; error: JsonRpcError }> {
  const requestId = options.requestId || randomUUID();
  const timeout = options.timeout || getDefaultTimeout(toolName);

  try {
    const client = await MCPClient.getInstance();
    const result = await client.callTool(toolName, params, { timeout, requestId });

    return { success: true, result };
  } catch (error) {
    if (isJsonRpcError(error)) {
      return { success: false, error };
    }

    // Wrap unknown errors
    return {
      success: false,
      error: {
        code: -32603,
        message: error instanceof Error ? error.message : 'Internal error',
      },
    };
  }
}

/**
 * Create API response from MCP result
 */
export function createMCPResponse(
  result: { success: true; result: unknown } | { success: false; error: JsonRpcError },
  requestId: string
): NextResponse {
  if (result.success) {
    return NextResponse.json(
      { data: result.result },
      {
        status: 200,
        headers: { 'X-Request-ID': requestId },
      }
    );
  }

  const { status, body } = mapMCPErrorToHTTP(result.error);
  return NextResponse.json(body, {
    status,
    headers: { 'X-Request-ID': requestId },
  });
}

function getDefaultTimeout(toolName: MCPToolName): number {
  // URL conversion has fixed timeout
  if (toolName === 'convert_url') return 30000;

  // Other tools use size-based timeout (set by caller)
  return 60000; // Default 60s
}

function isJsonRpcError(error: unknown): error is JsonRpcError {
  return (
    typeof error === 'object' &&
    error !== null &&
    'code' in error &&
    'message' in error
  );
}
```

---

### NEO-1.5a-003: Integrate MCP Error Codes with API Error Catalog

**Priority**: P1 (High)
**Effort**: 20 minutes
**Dependencies**: NEO-1.5a-001
**Lead**: James, Support: Neo

**Description**:
Map JSON-RPC error codes from MCP to the application's error catalog (E2xx series) for consistent error handling in API routes.

**Acceptance Criteria**:
- [ ] JSON-RPC errors (-32700 to -32099) mapped to E2xx
- [ ] HTTP status codes determined by error type
- [ ] User-friendly messages generated
- [ ] Retryable flag set correctly
- [ ] Mapping documented in code comments

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/lib/mcp/error-mapping.ts`

**Technical Notes**:
```typescript
// src/lib/mcp/error-mapping.ts

import type { JsonRpcError } from '@/types/mcp';

interface MappedError {
  status: number;
  body: {
    error: {
      code: string;
      message: string;
      userMessage: string;
      retryable: boolean;
    };
  };
}

/**
 * JSON-RPC error code mapping to application errors
 *
 * JSON-RPC Standard Errors:
 * -32700: Parse error
 * -32600: Invalid request
 * -32601: Method not found
 * -32602: Invalid params
 * -32603: Internal error
 * -32000 to -32099: Server errors (reserved)
 */
export function mapMCPErrorToHTTP(error: JsonRpcError): MappedError {
  const { code, message } = error;

  // Parse error
  if (code === -32700) {
    return {
      status: 400,
      body: {
        error: {
          code: 'E201',
          message: 'MCP parse error',
          userMessage: 'Failed to parse server response',
          retryable: true,
        },
      },
    };
  }

  // Invalid request
  if (code === -32600) {
    return {
      status: 400,
      body: {
        error: {
          code: 'E202',
          message: 'Invalid MCP request',
          userMessage: 'Invalid request format',
          retryable: false,
        },
      },
    };
  }

  // Method not found
  if (code === -32601) {
    return {
      status: 404,
      body: {
        error: {
          code: 'E203',
          message: 'MCP tool not found',
          userMessage: 'The requested operation is not available',
          retryable: false,
        },
      },
    };
  }

  // Invalid params
  if (code === -32602) {
    return {
      status: 400,
      body: {
        error: {
          code: 'E204',
          message: 'Invalid MCP parameters',
          userMessage: 'Invalid parameters provided',
          retryable: false,
        },
      },
    };
  }

  // Internal error
  if (code === -32603) {
    return {
      status: 500,
      body: {
        error: {
          code: 'E205',
          message: 'MCP internal error',
          userMessage: 'An internal error occurred during processing',
          retryable: true,
        },
      },
    };
  }

  // Server errors (-32000 to -32099)
  if (code >= -32099 && code <= -32000) {
    return {
      status: 503,
      body: {
        error: {
          code: 'E206',
          message: `MCP server error: ${message}`,
          userMessage: 'The document processing service is temporarily unavailable',
          retryable: true,
        },
      },
    };
  }

  // Unknown error
  return {
    status: 500,
    body: {
      error: {
        code: 'E299',
        message: `Unknown MCP error: ${message}`,
        userMessage: 'An unexpected error occurred',
        retryable: true,
      },
    },
  };
}

/**
 * Determine if an error is retryable based on code
 */
export function isRetryableError(code: number): boolean {
  // Parse error, internal error, server errors are retryable
  return code === -32700 || code === -32603 || (code >= -32099 && code <= -32000);
}
```

---

### NEO-1.5a-004: Support MCP Client Unit Testing Setup

**Priority**: P2 (Medium)
**Effort**: 15 minutes
**Dependencies**: Sprint 1.1 (MSW Setup)

**Description**:
Support James in setting up MSW mock handlers for MCP protocol testing, ensuring the test infrastructure properly mocks JSON-RPC communication.

**Acceptance Criteria**:
- [ ] MSW handlers for MCP JSON-RPC endpoint
- [ ] Mock responses for initialize sequence
- [ ] Mock responses for tool calls
- [ ] Mock error scenarios

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/mocks/mcp-handlers.ts`

**Technical Notes**:
```typescript
// src/mocks/mcp-handlers.ts

import { http, HttpResponse } from 'msw';

const MCP_SERVER_URL = process.env.MCP_SERVER_URL || 'http://hx-docling-server:8000';

export const mcpHandlers = [
  // MCP JSON-RPC endpoint
  http.post(`${MCP_SERVER_URL}/jsonrpc`, async ({ request }) => {
    const body = await request.json();
    const { method, id } = body as { method: string; id: string };

    // Initialize request
    if (method === 'initialize') {
      return HttpResponse.json({
        jsonrpc: '2.0',
        id,
        result: {
          protocolVersion: '2024-11-05',
          capabilities: { tools: { listChanged: false } },
          serverInfo: { name: 'hx-docling-mcp-server', version: '1.0.0' },
        },
      });
    }

    // Notifications (no response needed)
    if (method === 'notifications/initialized') {
      return new HttpResponse(null, { status: 204 });
    }

    // Tools list
    if (method === 'tools/list') {
      return HttpResponse.json({
        jsonrpc: '2.0',
        id,
        result: {
          tools: [
            { name: 'convert_pdf', description: 'Convert PDF to DoclingDocument' },
            { name: 'convert_docx', description: 'Convert DOCX to DoclingDocument' },
            // ... other tools
          ],
        },
      });
    }

    // Tool call
    if (method === 'tools/call') {
      const { params } = body as { params: { name: string; arguments: unknown } };

      // Mock convert_pdf response
      if (params.name === 'convert_pdf') {
        return HttpResponse.json({
          jsonrpc: '2.0',
          id,
          result: {
            content: [{ type: 'text', text: JSON.stringify({ name: 'test.pdf', body: {} }) }],
          },
        });
      }

      // Mock export_markdown response
      if (params.name === 'export_markdown') {
        return HttpResponse.json({
          jsonrpc: '2.0',
          id,
          result: {
            content: [{ type: 'text', text: '# Document Title\n\nContent here.' }],
          },
        });
      }
    }

    // Unknown method
    return HttpResponse.json({
      jsonrpc: '2.0',
      id,
      error: { code: -32601, message: 'Method not found' },
    });
  }),
];
```

---

## Summary

| Task ID | Task Title | Effort | Priority |
|---------|-----------|--------|----------|
| NEO-1.5a-001 | Create MCP Types for Next.js | 20m | P1 |
| NEO-1.5a-002 | Create API Route Wrapper | 30m | P1 |
| NEO-1.5a-003 | Integrate MCP Error Mapping | 20m | P1 |
| NEO-1.5a-004 | Support MCP Testing Setup | 15m | P2 |

**Total Neo Effort**: ~1.4 hours (85 minutes)
**Total Tasks**: 4

---

## Dependencies

Neo's tasks in Sprint 1.5a support James's primary implementation:

```
Sprint 1.4 Complete
    |
    +-> JAMES: MCP Client Implementation (Lead)
    |       |
    |       +-> NEO-1.5a-001 (MCP Types) [Support]
    |       |
    |       +-> NEO-1.5a-002 (API Wrapper) [Support]
    |               |
    |               +-> NEO-1.5a-003 (Error Mapping) [Support]
    |
    +-> JAMES: MCP Protocol Compliance (Lead)
            |
            +-> NEO-1.5a-004 (MSW Handlers) [Support]
```

---

## Coordination Notes

- **Primary Lead**: James (@james) owns MCP client implementation
- **Neo's Role**: Provide Next.js API route integration support
- **George's Role**: Provide FastMCP expertise
- **Handoff**: Types and wrappers should be ready before James implements API routes
- **Review**: All MCP-related code reviewed by Alex for protocol compliance
