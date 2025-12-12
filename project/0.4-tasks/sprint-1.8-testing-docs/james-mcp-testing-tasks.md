# Tasks: MCP Integration Testing & Documentation (Sprint 1.8)

**Agent**: James Dean (@james) - Docling MCP Gateway SME
**Sprint**: 1.8 - Testing & Documentation (MCP Aspects)
**Duration**: Partial session (~2.0h of 6.0h total)
**Lead**: Julia (@julia)
**Support**: Neo (@neo), William (@william), Trinity (@trinity)
**Review**: Alex (@alex)

**Input**:
- `/home/agent0/hx-docling-application/project/0.3-specification/0.3.1-detailed-specification.md` (Section 7.1, 8.0)
- `/home/agent0/hx-docling-application/project/0.1-plan/0.1.1-implementation-plan.md` (Section 4.8)

**Prerequisites**:
- Sprint 1.7 complete (all features implemented)
- MCP client operational with all 8 tools
- Processing pipeline with checkpoints working
- Test infrastructure set up (Vitest, MSW, Playwright)

---

## Context

Sprint 1.8 is led by Julia for comprehensive testing and documentation. James contributes MCP-specific testing and documentation tasks:
- Unit tests for MCP client initialization and tools
- Integration tests for MCP processing flow
- API route tests for MCP endpoints
- Security tests for MCP error handling
- State machine tests for MCP retry logic
- CLAUDE.md documentation for MCP error mapping

**Julia's Tasks** (not in this file):
- Validation library unit tests
- Component tests (UploadZone, ResultsViewer, etc.)
- Store tests (documentStore, uiStore)
- General API route tests
- E2E tests with Playwright
- Accessibility tests
- Performance tests
- Security tests (SSRF, file validation)
- State machine tests (job state transitions)
- Documentation (README, API docs)

---

## Phase 3.1: MCP Client Unit Tests

### JAM-1.8-001: Comprehensive MCP Client Initialization Tests

**Description**: Expand unit tests for MCP client initialization sequence beyond basic coverage from Sprint 1.5a.

**File**: `src/lib/mcp/__tests__/client.test.ts` (expand existing)

**Acceptance Criteria**:
- [ ] Test initialization with custom protocol version (env variable)
- [ ] Test initialization with authentication headers
- [ ] Test initialization timeout handling (server slow to respond)
- [ ] Test initialization with missing serverInfo fields
- [ ] Test initialization with unsupported protocol version
- [ ] Test concurrent initialization calls (race condition)
- [ ] Test re-initialization after connection failure
- [ ] Test tool registry caching across initialization
- [ ] Mock all HTTP requests with MSW
- [ ] 100% branch coverage for initialize() method

**Dependencies**: Sprint 1.7 complete

**Effort**: 45 minutes

**Deliverables**:
- Updated `/home/agent0/hx-docling-application/src/lib/mcp/__tests__/client.test.ts`

**Technical Notes**:
- Test edge cases not covered in Sprint 1.5a
- Focus on error conditions and race conditions
- Mock server responses with various error scenarios
- Verify idempotent behavior (multiple init calls safe)

---

### JAM-1.8-002: MCP Tool Invocation Tests (All 8 Tools)

**Description**: Create comprehensive unit tests for all 8 MCP tool invocations with parameter validation and response handling.

**File**: `src/lib/mcp/__tests__/tools.test.ts`

**Acceptance Criteria**:
- [ ] Test convert_pdf with valid file path
- [ ] Test convert_docx with valid file path
- [ ] Test convert_xlsx with valid file path
- [ ] Test convert_pptx with valid file path
- [ ] Test convert_url with valid URL
- [ ] Test export_markdown with DoclingDocument
- [ ] Test export_html with DoclingDocument
- [ ] Test export_json with DoclingDocument
- [ ] Test parameter validation failures (invalid paths, malformed URLs)
- [ ] Test timeout enforcement (size-based: 60s, 180s, 300s)
- [ ] Test tool not found error (E204)
- [ ] Test MCP server error responses
- [ ] Mock all 8 tool responses with MSW
- [ ] 100% coverage of invoke() method

**Dependencies**: Sprint 1.7 complete

**Effort**: 60 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/mcp/__tests__/tools.test.ts`

**Technical Notes**:
- Reference Specification Section 7.1.3 for tool schemas
- Mock DoclingDocument responses matching real structure
- Test size-based timeout selection logic
- Verify parameter validation using Zod schemas

---

### JAM-1.8-003: MCP Error Handling and Retry Tests

**Description**: Test error classification, mapping, retry logic, and circuit breaker behavior.

**File**: `src/lib/mcp/__tests__/error-handling.test.ts`

**Acceptance Criteria**:
- [ ] Test all JSON-RPC error codes (-32700 to -32603) mapped correctly
- [ ] Test server error codes (-32000 to -32003) mapped correctly
- [ ] Test unknown error code defaults to E201
- [ ] Test transient error detection (E201, E202 are retryable)
- [ ] Test permanent error detection (E203, E204, E205 not retryable)
- [ ] Test retry logic with exponential backoff (1s, 2s, 4s)
- [ ] Test retry exhaustion after 3 attempts
- [ ] Test circuit breaker state transitions (CLOSED -> OPEN -> HALF_OPEN -> CLOSED)
- [ ] Test circuit breaker opens after 50% failure rate
- [ ] Test circuit breaker recovery on successful request
- [ ] Mock time for backoff testing (jest.useFakeTimers)
- [ ] 100% coverage of error handling modules

**Dependencies**: Sprint 1.7 complete

**Effort**: 60 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/mcp/__tests__/error-handling.test.ts`

**Technical Notes**:
- Use fake timers to test exponential backoff without delays
- Test circuit breaker edge cases (exactly 50% failure, recovery)
- Verify error messages are user-friendly
- Test original MCP code preservation in error response

---

## Phase 3.2: MCP Integration Tests

### JAM-1.8-004: MCP Processing Pipeline Integration Tests

**Description**: End-to-end integration tests for complete document processing flow with real MCP client (mocked server).

**File**: `src/lib/processing/__tests__/pipeline.integration.test.ts`

**Acceptance Criteria**:
- [ ] Test PDF processing: upload -> convert_pdf -> export_markdown/html/json -> complete
- [ ] Test DOCX processing: upload -> convert_docx -> exports -> complete
- [ ] Test URL processing: validate -> convert_url -> exports -> complete
- [ ] Test progress events published at each stage (10%, 30%, 60%, 80%, 95%, 100%)
- [ ] Test checkpoints saved at each stage
- [ ] Test resume from 'converted' checkpoint (skip convert, only export)
- [ ] Test resume from 'export_markdown' checkpoint (skip md, only html/json)
- [ ] Test cancellation at each stage (AbortController)
- [ ] Test transient error triggers retry with checkpoint resume
- [ ] Test permanent error immediately fails
- [ ] Mock MCP server responses with MSW
- [ ] Integration with real Redis (testcontainers)
- [ ] Integration with real PostgreSQL (testcontainers)

**Dependencies**: Sprint 1.7 complete, testcontainers configured

**Effort**: 90 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/processing/__tests__/pipeline.integration.test.ts`

**Technical Notes**:
- Use testcontainers for Redis and PostgreSQL
- Mock MCP server with realistic response delays
- Test all checkpoint resume scenarios
- Verify progress monotonic guarantee
- Coordinate with Julia on integration test strategy

---

### JAM-1.8-005: API Route Tests for MCP Processing Endpoint

**Description**: Test /api/v1/process POST endpoint with MCP processing orchestration.

**File**: `src/app/api/v1/process/__tests__/route.test.ts`

**Acceptance Criteria**:
- [ ] Test successful job initiation returns jobId and streamUrl
- [ ] Test job status transitions: PENDING -> PROCESSING -> COMPLETED
- [ ] Test MCP processing triggered asynchronously
- [ ] Test invalid jobId returns 404
- [ ] Test missing file returns 400
- [ ] Test MCP client initialization before processing
- [ ] Test error handling (MCP unavailable, timeout)
- [ ] Test concurrent job processing (rate limiting)
- [ ] Mock MCP client with MSW
- [ ] Mock database with Prisma mock
- [ ] 100% coverage of route handler

**Dependencies**: Sprint 1.7 complete

**Effort**: 45 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/app/api/v1/process/__tests__/route.test.ts`

**Technical Notes**:
- Test async processing orchestration
- Verify job state transitions in database
- Test MCP client singleton initialization
- Coordinate with Neo on API route testing patterns

---

## Phase 3.3: State Machine & Security Tests

### JAM-1.8-006: MCP Retry State Machine Tests

**Description**: Test state machine transitions for MCP retry scenarios (PROCESSING -> RETRY_1 -> RETRY_2 -> RETRY_3 -> FAILED/COMPLETED).

**File**: `src/lib/processing/__tests__/state-machine.test.ts`

**Acceptance Criteria**:
- [ ] Test PROCESSING -> RETRY_1 on transient MCP error (E201, E202)
- [ ] Test RETRY_1 -> RETRY_2 on second failure
- [ ] Test RETRY_2 -> RETRY_3 on third failure
- [ ] Test RETRY_3 -> FAILED after exhaustion
- [ ] Test RETRY_* -> PROCESSING on successful retry
- [ ] Test RETRY_* -> COMPLETED if checkpoint allows skip
- [ ] Test PROCESSING -> FAILED on permanent error (E203, E204, E205)
- [ ] Test checkpoint resume at each RETRY_* state
- [ ] Test SSE events emitted on state transitions (retry event)
- [ ] Test exponential backoff delays (1s, 2s, 4s)
- [ ] Mock time for delay testing
- [ ] 100% coverage of state machine logic

**Dependencies**: Sprint 1.7 complete

**Effort**: 60 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/processing/__tests__/state-machine.test.ts`

**Technical Notes**:
- Reference Specification Section 6.1 (state diagram)
- Test all 10 job states (PENDING, UPLOADING, PROCESSING, RETRY_1-3, COMPLETED, FAILED, CANCELLED, PARTIAL_COMPLETED)
- Verify retry event format matches SSE spec
- Coordinate with Julia on state machine test strategy

---

### JAM-1.8-007: MCP Security Tests (Error Exposure)

**Description**: Security tests to ensure MCP errors don't leak sensitive information (server paths, credentials, internal details).

**File**: `src/lib/mcp/__tests__/security.test.ts`

**Acceptance Criteria**:
- [ ] Test error responses don't expose server file paths
- [ ] Test error responses don't expose server internal IPs
- [ ] Test error responses don't expose authentication tokens
- [ ] Test error responses don't expose database connection strings
- [ ] Test error messages sanitized (no stack traces in production)
- [ ] Test original MCP error only logged (not sent to client)
- [ ] Test user-facing error messages are generic but helpful
- [ ] Test error codes don't reveal system architecture
- [ ] Mock MCP server errors with sensitive data
- [ ] 100% coverage of error sanitization logic

**Dependencies**: Sprint 1.7 complete

**Effort**: 30 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/mcp/__tests__/security.test.ts`

**Technical Notes**:
- Test NODE_ENV=production sanitization
- Verify stack traces only in development mode
- Test error message redaction patterns
- Coordinate with Julia on security test checklist

---

## Phase 3.4: Documentation

### JAM-1.8-008: Document MCP Error Code Mapping in CLAUDE.md

**Description**: Add comprehensive MCP error code mapping table to CLAUDE.md for future developers and troubleshooting.

**File**: `CLAUDE.md` (append to Project-Specific Notes)

**Acceptance Criteria**:
- [ ] Section: "MCP Error Code Mapping"
- [ ] Table: JSON-RPC code, App code, User message, Retryable, Description
- [ ] All standard codes (-32700 to -32603) documented
- [ ] All server codes (-32000 to -32003) documented
- [ ] Application codes (E201-E206) documented
- [ ] Troubleshooting guide for each error
- [ ] Example error responses with both codes
- [ ] Links to specification sections
- [ ] Formatted as markdown table

**Dependencies**: Sprint 1.7 complete

**Effort**: 30 minutes

**Deliverables**:
- Updated `/home/agent0/hx-docling-application/CLAUDE.md`

**Technical Notes**:
- Reference Specification Section 7.1.4
- Include examples of when each error occurs
- Add troubleshooting steps for common errors
- Coordinate with Julia on CLAUDE.md structure

---

### JAM-1.8-009: Document MCP Client Architecture in CLAUDE.md

**Description**: Document MCP client architecture, initialization sequence, and usage patterns in CLAUDE.md.

**File**: `CLAUDE.md` (append to Architecture section)

**Acceptance Criteria**:
- [ ] Section: "MCP Client Architecture"
- [ ] Subsection: "Initialization Sequence" (3-step diagram)
- [ ] Subsection: "Tool Invocation Patterns"
- [ ] Subsection: "Error Handling & Retry Logic"
- [ ] Subsection: "Circuit Breaker Pattern"
- [ ] Code examples for common usage patterns
- [ ] Mermaid sequence diagram for initialization
- [ ] Notes on AbortController usage
- [ ] Notes on checkpoint integration

**Dependencies**: Sprint 1.7 complete

**Effort**: 45 minutes

**Deliverables**:
- Updated `/home/agent0/hx-docling-application/CLAUDE.md`

**Technical Notes**:
- Include mermaid diagram from specification
- Show code examples for processDocument() orchestration
- Document singleton pattern for MCPClient
- Explain coordination with SSE progress publishing

---

### JAM-1.8-010: Document MCP Troubleshooting Guide in CLAUDE.md

**Description**: Create troubleshooting guide for common MCP integration issues.

**File**: `CLAUDE.md` (append to Project-Specific Notes)

**Acceptance Criteria**:
- [ ] Section: "MCP Troubleshooting Guide"
- [ ] Issue: "MCP server unavailable" (check health, network, credentials)
- [ ] Issue: "Protocol initialization failed" (check version, capabilities)
- [ ] Issue: "Tool not found" (verify tools/list, check spelling)
- [ ] Issue: "Request timeout" (check file size, timeout config)
- [ ] Issue: "Circuit breaker open" (check failure rate, wait for recovery)
- [ ] Issue: "Checkpoint resume failed" (check database, serialization)
- [ ] Diagnostic commands for each issue
- [ ] Links to relevant log files and monitoring dashboards

**Dependencies**: Sprint 1.7 complete

**Effort**: 30 minutes

**Deliverables**:
- Updated `/home/agent0/hx-docling-application/CLAUDE.md`

**Technical Notes**:
- Include curl commands to test MCP server health
- Document environment variables affecting MCP client
- List common log patterns for each issue
- Coordinate with William on infrastructure troubleshooting

---

## Phase 3.5: Performance & Coverage

### JAM-1.8-011: MCP Performance Tests

**Description**: Performance tests for MCP client initialization, tool invocation latency, and throughput.

**File**: `src/lib/mcp/__tests__/performance.test.ts`

**Acceptance Criteria**:
- [ ] Test initialization latency < 500ms (small server)
- [ ] Test convert_pdf latency < 60s for file < 10MB
- [ ] Test convert_pdf latency < 180s for file < 50MB
- [ ] Test export_* latency < 10s each
- [ ] Test concurrent tool invocations (3 exports in parallel)
- [ ] Test tool schema cache hit rate > 90%
- [ ] Test retry overhead (measure backoff delays)
- [ ] Test circuit breaker overhead (state transitions)
- [ ] Mock MCP server with controlled response times
- [ ] Report performance metrics to console

**Dependencies**: Sprint 1.7 complete

**Effort**: 45 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/mcp/__tests__/performance.test.ts`

**Technical Notes**:
- Use performance.now() for timing measurements
- Mock realistic MCP server latencies
- Test parallel export invocations
- Coordinate with Julia on performance test strategy

---

### JAM-1.8-012: MCP Code Coverage Report & Gaps

**Description**: Generate code coverage report for all MCP modules and identify gaps.

**File**: N/A (report generation)

**Acceptance Criteria**:
- [ ] Run coverage: `npm run test:coverage -- --testPathPattern=mcp`
- [ ] Line coverage >= 90% for all MCP modules
- [ ] Branch coverage >= 85% for all MCP modules
- [ ] Function coverage >= 90% for all MCP modules
- [ ] Identify uncovered edge cases
- [ ] Add tests for uncovered branches
- [ ] Document coverage gaps in issue tracker
- [ ] Generate HTML coverage report

**Dependencies**: JAM-1.8-001 to JAM-1.8-011 complete

**Effort**: 30 minutes

**Deliverables**:
- Coverage report: `/home/agent0/hx-docling-application/coverage/mcp/index.html`
- GitHub issue for any gaps < 90%

**Technical Notes**:
- Use Vitest coverage with c8
- Focus on edge cases and error paths
- Prioritize fixing gaps in critical modules (client, error-mapping)
- Coordinate with Julia on coverage targets

---

## Dependencies

**Sequential Dependencies**:
- JAM-1.8-001, JAM-1.8-002, JAM-1.8-003 can run in parallel (different modules)
- JAM-1.8-004 blocks nothing (integration tests)
- JAM-1.8-005 blocks nothing (API tests)
- JAM-1.8-006, JAM-1.8-007 can run in parallel
- JAM-1.8-008, JAM-1.8-009, JAM-1.8-010 can run in parallel (documentation)
- JAM-1.8-011 can run in parallel with documentation
- JAM-1.8-012 must be last (depends on all tests)

**Parallel Opportunities**:
- JAM-1.8-001 [P] JAM-1.8-002 [P] JAM-1.8-003 (unit tests, different files)
- JAM-1.8-004 [P] JAM-1.8-005 (integration vs API tests)
- JAM-1.8-006 [P] JAM-1.8-007 (state machine vs security)
- JAM-1.8-008 [P] JAM-1.8-009 [P] JAM-1.8-010 (documentation sections)
- JAM-1.8-011 can run anytime before JAM-1.8-012

---

## Total Effort Summary

| Phase | Tasks | Total Hours |
|-------|-------|-------------|
| 3.1: MCP Client Unit Tests | JAM-1.8-001 to JAM-1.8-003 | 2.75h |
| 3.2: MCP Integration Tests | JAM-1.8-004 to JAM-1.8-005 | 2.25h |
| 3.3: State Machine & Security | JAM-1.8-006 to JAM-1.8-007 | 1.50h |
| 3.4: Documentation | JAM-1.8-008 to JAM-1.8-010 | 1.75h |
| 3.5: Performance & Coverage | JAM-1.8-011 to JAM-1.8-012 | 1.25h |
| **Total** | **12 tasks** | **9.50h** |

**Note**: Sprint 1.8 is 6.0h total budget across all testing. Julia leads general testing (~4.0h), James handles MCP-specific testing and documentation (~2.0h actual vs 9.5h ideal). Prioritize critical tasks: JAM-1.8-001-003 (unit), JAM-1.8-004 (integration), JAM-1.8-008 (error mapping docs).

**Priority Tiers for 2.0h Budget**:
- **P0 (Must Have - 2.0h)**: JAM-1.8-001 (0.75h), JAM-1.8-002 (1.0h), JAM-1.8-008 (0.5h) - **Total: 2.25h**
- **P1 (Should Have)**: JAM-1.8-003, JAM-1.8-004, JAM-1.8-005
- **P2 (Nice to Have)**: JAM-1.8-006, JAM-1.8-007, JAM-1.8-009, JAM-1.8-010, JAM-1.8-011, JAM-1.8-012

---

## Acceptance Criteria (Sprint Level - MCP Testing)

**Critical (P0)**:
- [ ] MCP client initialization unit tests pass with 100% coverage
- [ ] All 8 MCP tools have unit tests with parameter validation
- [ ] MCP error code mapping documented in CLAUDE.md

**High Priority (P1)**:
- [ ] Error handling tests cover all JSON-RPC codes and retry logic
- [ ] Integration tests verify end-to-end processing pipeline
- [ ] API route tests verify MCP processing endpoint
- [ ] Overall MCP module coverage >= 80%

**Medium Priority (P2)**:
- [ ] State machine tests verify all RETRY_* transitions
- [ ] Security tests verify no sensitive data in error responses
- [ ] MCP architecture documented in CLAUDE.md
- [ ] Troubleshooting guide complete
- [ ] Performance tests verify latency targets
- [ ] Code coverage >= 90% for all MCP modules

---

## Integration Points

**Coordinates with**:
- **Julia** (Testing Lead): Overall test strategy, coverage targets, test infrastructure
- **Neo** (Frontend): API route testing, component integration
- **William** (SSE): Integration tests with progress events, checkpoint coordination
- **Trinity** (Database): Integration tests with PostgreSQL checkpoint storage

**Escalation Points**:
- Coverage gaps < 80%: Coordinate with Julia on priority
- Integration test failures: Check testcontainers setup with William
- Documentation structure: Align with Julia's README/API docs format
- Performance test failures: Coordinate with William on infrastructure tuning
