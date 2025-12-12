# Tasks: MCP Client Integration (Sprint 1.5a)

**Version**: 1.1.0 (2025-12-12)
**Agent**: James Dean (@james) - Docling MCP Gateway SME
**Sprint**: 1.5a - MCP Client (Core)
**Duration**: 1 session (~4.0h)
**Support**: Neo (@neo), George (@george)
**Review**: Alex (@alex)

**Changelog**:
- v1.1.0 (2025-12-12): Fixed DEF-011 (error code mapping inconsistency) and DEF-012 (missing initialization guard)
  - DEF-011: Separated error codes into non-overlapping ranges (E200-E204 for JSON-RPC protocol, E301-E305 for MCP runtime)
  - DEF-012: Added explicit `initialized` flag guard in `invoke()` method to prevent tool calls before initialization completes

**Input**:
- `/home/agent0/hx-docling-application/project/0.3-specification/0.3.1-detailed-specification.md` (Section 7.1)
- `/home/agent0/hx-docling-application/project/0.1-plan/0.1.1-implementation-plan.md` (Section 4.5a)

**Prerequisites**:
- Sprint 1.4 complete (URL input and validation)
- Environment variable `DOCLING_MCP_ORIGIN` configured
- hx-docling-mcp-server accessible at http://hx-docling-server.hx.dev.local:8000

---

## Phase 3.1: MCP Type Definitions & Configuration

### JAM-1.5a-001: Define MCP Protocol Types (JSON-RPC 2.0)

**Description**: Create comprehensive TypeScript types for MCP JSON-RPC 2.0 protocol messages, requests, responses, notifications, and errors.

**File**: `src/lib/mcp/types.ts`

**Acceptance Criteria**:
- [ ] `JsonRpcRequest` interface with `jsonrpc`, `method`, `params`, `id` fields
- [ ] `JsonRpcResponse` interface with `jsonrpc`, `result`, `id` fields
- [ ] `JsonRpcError` interface with `code`, `message`, `data` fields
- [ ] `JsonRpcNotification` interface (no `id` field)
- [ ] `MCPToolDefinition` interface matching server schema
- [ ] `MCPServerCapabilities` interface for capability negotiation
- [ ] `MCPInitializeResponse` interface with `serverInfo`, `capabilities`, `protocolVersion`
- [ ] All types exported and documented with JSDoc

**Dependencies**: None (foundational task)

**Effort**: 30 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/mcp/types.ts` with complete type definitions

**Technical Notes**:
- Use TypeScript 5.x features (template literal types, satisfies)
- Include JSON-RPC 2.0 error codes as const enum
- Reference specification Section 7.1.4 for error code mappings
- Ensure strict null checks compatibility

---

### JAM-1.5a-002: Create MCP Client Configuration

**Description**: Implement configuration module with environment variable support, default values, and authentication options.

**File**: `src/lib/mcp/config.ts`

**Acceptance Criteria**:
- [ ] `MCPClientConfig` interface with endpoint, timeout, retries, auth
- [ ] `MCPAuthConfig` interface supporting 'none', 'bearer', 'api-key' types
- [ ] `DEFAULT_CONFIG` with fallback to http://hx-docling-mcp-server.hx.dev.local:8000
- [ ] Environment variable `DOCLING_MCP_ORIGIN` support
- [ ] Environment variable `MCP_PROTOCOL_VERSION` support (default: '2024-11-05')
- [ ] Config validation using Zod schema
- [ ] Export `getMCPConfig()` factory function

**Dependencies**: JAM-1.5a-001

**Effort**: 20 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/mcp/config.ts`
- Updated `/home/agent0/hx-docling-application/.env.example` with MCP variables

**Technical Notes**:
- Phase 1 uses `auth: { type: 'none' }` for internal network
- Production will require api-key authentication
- Timeout defaults: 300s for large files
- Use process.env with fallbacks for Next.js compatibility

---

### JAM-1.5a-003: Implement Tool Parameter Schemas

**Description**: Define Zod validation schemas for all 8 MCP tool parameters (convert_pdf, convert_docx, convert_xlsx, convert_pptx, convert_url, export_markdown, export_html, export_json).

**File**: `src/lib/mcp/schemas.ts`

**Acceptance Criteria**:
- [ ] `FilePathSchema` for file conversion tools (file_path, options)
- [ ] `UrlSchema` for URL conversion (url, options)
- [ ] `ExportSchema` for export tools (document, options)
- [ ] `DoclingDocumentSchema` with explicit field validation
- [ ] `TOOL_SCHEMAS` record mapping tool names to schemas
- [ ] All schemas include `.passthrough()` for extensibility
- [ ] Export validation helper functions

**Dependencies**: JAM-1.5a-001

**Effort**: 25 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/mcp/schemas.ts`

**Technical Notes**:
- Reference Specification Section 7.1.3 for schema structure
- DoclingDocument includes: schema_name, version, name, origin, furniture, body, groups, pictures, tables
- Use `.passthrough()` to allow additional fields from MCP server
- Validate parameters before MCP tool invocation

---

## Phase 3.2: MCP Initialization Sequence

### JAM-1.5a-004: Implement MCP Client Class Foundation

**Description**: Create MCPClient class with connection management, request/response handling, and state tracking.

**File**: `src/lib/mcp/client.ts`

**Acceptance Criteria**:
- [ ] `MCPClient` class with private `endpoint`, `initialized`, `toolRegistry` fields
- [ ] `request()` method for JSON-RPC requests with ID generation
- [ ] `notify()` method for JSON-RPC notifications (no response)
- [ ] `fetch()` wrapper with timeout handling
- [ ] Request ID generation using UUID v4
- [ ] Error response parsing and handling
- [ ] Connection state tracking

**Dependencies**: JAM-1.5a-001, JAM-1.5a-002

**Effort**: 30 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/mcp/client.ts` (foundation)

**Technical Notes**:
- Use `fetch` API with AbortController for timeouts
- Separate request (expects response) from notify (no response)
- Store tool registry as Map<string, MCPToolDefinition>
- Implement singleton pattern for global client instance

---

### JAM-1.5a-005: Implement 3-Step MCP Initialization (CRITICAL)

**Description**: Implement protocol-compliant initialization sequence: initialize request, initialized notification, tools/list call.

**File**: `src/lib/mcp/client.ts` (add `initialize()` method)

**Acceptance Criteria**:
- [ ] **Step 1**: Send `initialize` request with clientInfo, capabilities
- [ ] **Step 2**: Process initialize response and validate server capabilities
- [ ] **Step 3**: Send `notifications/initialized` notification (no id field)
- [ ] **Step 4**: Call `tools/list` to discover available tools
- [ ] Tool schemas cached in `toolRegistry` Map
- [ ] `initialized` flag set to true ONLY after Step 3 completes successfully
- [ ] Guard in `invoke()` method throws E301 error if `initialized` is false
- [ ] Idempotent: calling `initialize()` multiple times is safe

**Dependencies**: JAM-1.5a-004

**Effort**: 45 minutes

**Deliverables**:
- Updated `/home/agent0/hx-docling-application/src/lib/mcp/client.ts` with `initialize()` method

**Technical Notes**:
- **CRITICAL**: Must send `notifications/initialized` before ANY tool calls (Spec 7.1.2)
- Notification format: `{ jsonrpc: '2.0', method: 'notifications/initialized', params: {} }`
- No `id` field in notifications
- Cache tool schemas for parameter validation
- Reference Implementation Plan Section 4.5a Task 3
- **Guard Example**: Prevent tool calls before initialization:
  ```typescript
  // In invoke() method
  if (!this.initialized) {
    throw new MCPError('E301', 'MCP not initialized. Call initialize() first.');
  }
  ```

---

### JAM-1.5a-006: Implement Server Capabilities Validation

**Description**: Validate server capabilities response to ensure required features (tools) are supported.

**File**: `src/lib/mcp/capabilities.ts`

**Acceptance Criteria**:
- [ ] `validateServerCapabilities()` function
- [ ] Verify `capabilities.tools` is present (required)
- [ ] Store server info (name, version) for logging
- [ ] Store `protocolVersion` for compatibility checks
- [ ] Throw `MCPError` E305 if tools capability missing (runtime error)
- [ ] Log server connection details to console
- [ ] Return validated capabilities object

**Dependencies**: JAM-1.5a-001, JAM-1.5a-005

**Effort**: 20 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/mcp/capabilities.ts`

**Technical Notes**:
- Reference Specification Section 7.1.1.2 (MAJ-MCP-002)
- Server capabilities structure: `{ tools?: { listChanged?: boolean } }`
- Optional capabilities: prompts, resources, logging
- Phase 1 only requires tools capability

---

## Phase 3.3: Tool Discovery & Invocation

### JAM-1.5a-007: Implement Tool Schema Caching

**Description**: Create tool schema cache with TTL management and lookup helpers.

**File**: `src/lib/mcp/tool-cache.ts`

**Acceptance Criteria**:
- [ ] `ToolCache` class with Map-based storage
- [ ] `set()` method to cache tool schemas with timestamp
- [ ] `get()` method with TTL validation
- [ ] `has()` method to check cache presence
- [ ] `clear()` method to invalidate cache
- [ ] Configurable TTL (default: 5 minutes)
- [ ] Cache statistics (hits, misses) for monitoring

**Dependencies**: JAM-1.5a-001

**Effort**: 25 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/mcp/tool-cache.ts`

**Technical Notes**:
- TTL prevents stale schemas if MCP server updates
- Cache invalidation on initialization failure
- Export singleton `toolCache` instance
- Track cache hit rate for performance monitoring

---

### JAM-1.5a-008: Implement Tool Invocation Method

**Description**: Add `invoke()` method to MCPClient for calling MCP tools with parameter validation and response handling.

**File**: `src/lib/mcp/client.ts` (add `invoke()` method)

**Acceptance Criteria**:
- [ ] `invoke(toolName: string, params: unknown)` method
- [ ] Guard: Check `initialized` flag before proceeding (throw E301 if false)
- [ ] Auto-initialize if not already initialized
- [ ] Tool existence validation (check `toolRegistry`)
- [ ] Parameter validation using cached schemas
- [ ] Call `tools/call` JSON-RPC method
- [ ] Return result payload or throw error
- [ ] Size-based timeout enforcement (60s < 10MB, 180s < 50MB, 300s >= 50MB)

**Dependencies**: JAM-1.5a-005, JAM-1.5a-007

**Effort**: 35 minutes

**Deliverables**:
- Updated `/home/agent0/hx-docling-application/src/lib/mcp/client.ts` with `invoke()` method

**Technical Notes**:
- Validate params against `TOOL_SCHEMAS[toolName]` before sending
- Size-based timeouts configurable via environment
- Throw MCPError E304 if tool not found (runtime error)
- Include request timing for monitoring

---

## Phase 3.4: Error Handling & Mapping

### JAM-1.5a-009: Create MCP Error Catalog

**Description**: Define comprehensive error catalog with user-friendly messages and retry indicators.

**File**: `src/lib/mcp/errors.ts`

**Acceptance Criteria**:
- [ ] `MCPError` class extending Error
- [ ] JSON-RPC protocol error codes (E200-E204)
- [ ] MCP runtime error codes (E301-E305)
- [ ] Error messages mapping:
  - E200 (parse error), E201 (invalid request), E202 (method not found), E203 (invalid params), E204 (internal error)
  - E301 (not initialized), E302 (server unavailable), E303 (timeout), E304 (unknown tool), E305 (missing capability)
- [ ] User-facing messages for each error
- [ ] Suggested actions for each error
- [ ] `retryable` boolean flag for each error
- [ ] `toJSON()` method for error serialization

**Dependencies**: JAM-1.5a-001

**Effort**: 25 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/mcp/errors.ts`

**Technical Notes**:
- **JSON-RPC Protocol Errors (E200-E204)**:
  - E200: Parse error (not retryable)
  - E201: Invalid request (not retryable)
  - E202: Method not found (not retryable)
  - E203: Invalid params (not retryable)
  - E204: Internal error (retryable)
- **MCP Runtime Errors (E301-E305)**:
  - E301: MCP not initialized (not retryable)
  - E302: MCP server unavailable (retryable)
  - E303: Request timeout (retryable)
  - E304: Unknown tool requested (not retryable)
  - E305: Missing server capability (not retryable)

---

### JAM-1.5a-010: Implement JSON-RPC Error Mapping

**Description**: Map JSON-RPC 2.0 error codes (-32700 to -32099) to application error codes (E200 protocol, E3xx runtime).

**File**: `src/lib/mcp/error-mapping.ts`

**Acceptance Criteria**:
- [ ] `MCP_ERROR_MAP` record mapping JSON-RPC codes to app errors
- [ ] Standard protocol errors: -32700 → E200 (parse), -32600 → E201 (invalid request), -32601 → E202 (method not found), -32602 → E203 (invalid params), -32603 → E204 (internal)
- [ ] Server runtime errors: -32000 → E201 (unavailable), -32001 → E202 (timeout), -32002 → E302 (conversion failed), -32003 → E303 (export failed)
  - **Note**: Canonical mapping per Specification Section 7.1.4 and JSON-RPC 2.0 spec (server error range -32000 to -32099). See DEF-011 for error code separation rationale.
- [ ] `mapMCPError()` function converting JsonRpcError to AppError
- [ ] Preserve original MCP code in mapped error
- [ ] Default mapping for unknown error codes

**Dependencies**: JAM-1.5a-001, JAM-1.5a-009

**Effort**: 30 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/mcp/error-mapping.ts`

**Technical Notes**:
- Reference Specification Section 7.1.4 for canonical mapping table
- Include both application code and original MCP code in error response
- **Error Code Ranges** (per JSON-RPC 2.0 spec and DEF-011):
  - E200-E204: JSON-RPC protocol errors (standard spec codes -32700 to -32603)
  - E201-E202: MCP availability errors (-32000 unavailable, -32001 timeout)
  - E302-E303: Processing errors (-32002 conversion, -32003 export)
- Unknown codes default to E299 (generic MCP error)

---

### JAM-1.5a-011: Implement Retry Logic with Exponential Backoff

**Description**: Add retry mechanism for transient MCP errors with exponential backoff.

**File**: `src/lib/mcp/retry.ts`

**Acceptance Criteria**:
- [ ] `retryWithBackoff()` wrapper function
- [ ] Max 3 retry attempts
- [ ] Exponential backoff: 1s, 2s, 4s
- [ ] Only retry errors marked as `retryable: true`
- [ ] Preserve error context across retries
- [ ] Log retry attempts with timestamps
- [ ] Configurable via environment variables

**Dependencies**: JAM-1.5a-009, JAM-1.5a-010

**Effort**: 25 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/mcp/retry.ts`

**Technical Notes**:
- Retry on: E204 (internal error), E302 (unavailable), E303 (timeout)
- Do NOT retry: E200-E203 (protocol errors), E301 (not initialized), E304 (unknown tool), E305 (missing capability)
- Use `setTimeout` for backoff delays
- Include jitter to prevent thundering herd

---

## Phase 3.5: Circuit Breaker & Testing

### JAM-1.5a-012: Implement MCP Circuit Breaker (Optional Enhancement)

**Description**: Add circuit breaker pattern for graceful degradation when MCP server is consistently failing.

**File**: `src/lib/mcp/circuit-breaker.ts`

**Acceptance Criteria**:
- [ ] `CircuitBreaker` class with CLOSED, OPEN, HALF_OPEN states
- [ ] Track failure rate over sliding window (default: 10 requests)
- [ ] Open circuit after 50% failure rate
- [ ] Auto-transition to HALF_OPEN after timeout (default: 30s)
- [ ] Allow single test request in HALF_OPEN state
- [ ] Close circuit on successful test request
- [ ] Emit events for state transitions
- [ ] Integration with MCPClient

**Dependencies**: JAM-1.5a-004, JAM-1.5a-011

**Effort**: 25 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/mcp/circuit-breaker.ts`

**Technical Notes**:
- Optional enhancement for production resilience
- Prevents cascading failures when MCP server is down
- Configurable thresholds via environment
- Log state transitions for monitoring

---

### JAM-1.5a-013: Write Unit Tests for MCP Client Initialization

**Description**: Create comprehensive unit tests for 3-step initialization sequence and protocol compliance.

**File**: `src/lib/mcp/__tests__/client.test.ts`

**Acceptance Criteria**:
- [ ] Test successful initialization sequence (initialize -> initialized -> tools/list)
- [ ] Test initialization sends `notifications/initialized` after initialize response
- [ ] Test initialization fails if server missing tools capability
- [ ] Test initialization caches tool schemas
- [ ] Test idempotent initialization (multiple calls safe)
- [ ] Test initialization timeout handling
- [ ] Mock MCP server responses using MSW
- [ ] All tests pass with 100% coverage

**Dependencies**: JAM-1.5a-005, JAM-1.5a-006

**Effort**: 25 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/mcp/__tests__/client.test.ts`

**Technical Notes**:
- Use Vitest and MSW for mocking
- Mock HTTP responses for initialize, initialized, tools/list
- Verify `notifications/initialized` has no `id` field
- Test error scenarios (timeout, missing capability)

---

### JAM-1.5a-014: Write Unit Tests for Error Mapping

**Description**: Test JSON-RPC error code mapping to application error codes with all edge cases.

**File**: `src/lib/mcp/__tests__/error-mapping.test.ts`

**Acceptance Criteria**:
- [ ] Test all standard JSON-RPC codes (-32700 to -32603) map to E200-E204
- [ ] Test all server error codes (-32000 to -32003) map per Specification Section 7.1.4:
  - -32000 → E201 (unavailable), -32001 → E202 (timeout), -32002 → E302 (conversion), -32003 → E303 (export)
- [ ] Test unknown error code defaults to E299 (generic MCP error)
- [ ] Test retryable flag correctly set (E201, E202, E302, E303 are retryable)
- [ ] Test original MCP code preserved in mapped error
- [ ] Test user message generation
- [ ] 100% coverage of MCP_ERROR_MAP

**Dependencies**: JAM-1.5a-010

**Effort**: 15 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/mcp/__tests__/error-mapping.test.ts`

**Technical Notes**:
- Use Vitest for test runner
- Test each error code in mapping table
- Verify retryable flag matches specification
- Test error serialization with toJSON()

---

## Dependencies

**Sequential Dependencies**:
- JAM-1.5a-001 blocks JAM-1.5a-002, JAM-1.5a-003
- JAM-1.5a-004 blocks JAM-1.5a-005
- JAM-1.5a-005 blocks JAM-1.5a-006, JAM-1.5a-008, JAM-1.5a-013
- JAM-1.5a-009 blocks JAM-1.5a-010, JAM-1.5a-011

**Parallel Opportunities**:
- JAM-1.5a-002 [P] JAM-1.5a-003 (both depend only on types)
- JAM-1.5a-007 [P] JAM-1.5a-009 (independent modules)
- JAM-1.5a-013 [P] JAM-1.5a-014 (different test files)

---

## Total Effort Summary

| Phase | Tasks | Total Hours |
|-------|-------|-------------|
| 3.1: Type Definitions & Config | JAM-1.5a-001 to JAM-1.5a-003 | 1.25h |
| 3.2: Initialization Sequence | JAM-1.5a-004 to JAM-1.5a-006 | 1.58h |
| 3.3: Tool Discovery | JAM-1.5a-007 to JAM-1.5a-008 | 1.00h |
| 3.4: Error Handling | JAM-1.5a-009 to JAM-1.5a-011 | 1.33h |
| 3.5: Circuit Breaker & Testing | JAM-1.5a-012 to JAM-1.5a-014 | 1.08h |
| **Total** | **14 tasks** | **6.25h** |

**Note**: Original plan estimated 4.0h, but detailed breakdown reveals 6.25h needed for production-quality implementation with comprehensive testing.

---

## Acceptance Criteria (Sprint Level)

- [ ] MCP client connects to hx-docling-mcp-server successfully
- [ ] **CRITICAL**: `notifications/initialized` sent after initialize response
- [ ] **CRITICAL**: Tools/list called only AFTER notifications/initialized
- [ ] Server capability validation passes before tool calls
- [ ] All 8 MCP tools callable and return expected responses
- [ ] Tool schemas cached with configurable TTL
- [ ] Size-based timeouts enforced (60s < 10MB, 180s < 50MB, 300s >= 50MB)
- [ ] JSON-RPC protocol errors (-32700 to -32603) mapped to E200-E204
- [ ] MCP runtime errors mapped to E301-E305
- [ ] Retry logic with exponential backoff (3 attempts: 1s, 2s, 4s)
- [ ] All unit tests pass with >= 80% coverage
- [ ] No TypeScript errors or ESLint violations

---

## Integration Points

**Coordinates with**:
- **Neo** (Frontend): Provides MCP client for /api/v1/process route
- **George** (FastMCP): Ensures protocol compliance with MCP specification
- **William** (SSE): Publishes MCP progress events to Redis for SSE streaming
- **Trinity** (Database): Stores MCP processing results in PostgreSQL

**Escalation Points**:
- MCP server unavailable: Check hx-docling-server health
- Protocol errors: Coordinate with George on MCP spec compliance
- Timeout issues: Coordinate with William on async processing patterns
