# Tasks: MCP Progress Integration (Sprint 1.5b)

**Agent**: James Dean (@james) - Docling MCP Gateway SME
**Sprint**: 1.5b - SSE & Progress Integration (MCP Aspects)
**Duration**: Partial session (~1.5h of 4.0h total)
**Lead**: William (@william)
**Support**: Neo (@neo), Ola (@ola), Sophia (@sophia)
**Review**: Alex (@alex)

**Input**:
- `/home/agent0/hx-docling-application/project/0.3-specification/0.3.1-detailed-specification.md` (Section 4.2, 5.5, 7.1)
- `/home/agent0/hx-docling-application/project/0.1-plan/0.1.1-implementation-plan.md` (Section 4.5b)

**Prerequisites**:
- Sprint 1.5a complete (MCP client operational)
- MCPClient initialized and tools discovered
- Redis pub/sub configured for progress events

---

## Context

Sprint 1.5b is primarily led by William for SSE streaming infrastructure. James contributes MCP-specific integration tasks:
- MCP request abortion via AbortController
- MCP progress event publishing to Redis
- MCP error handling in processing pipeline
- Checkpoint coordination with MCP processing stages

**William's Tasks** (not in this file):
- SSE connection manager
- Exponential backoff reconnection
- Polling fallback mechanism
- Event buffering for Last-Event-ID
- State synchronization
- /api/v1/process POST handler (orchestration)
- SSE streaming response
- Checkpoint manager (coordination)

---

## Phase 3.1: MCP Request Abortion & Cancellation

### JAM-1.5b-001: Implement MCP AbortController Integration

**Description**: Add AbortController support to MCP client for request cancellation when user cancels job.

**File**: `src/lib/mcp/client.ts` (enhance request method)

**Acceptance Criteria**:
- [ ] `MCPClient` stores active `AbortController` instance
- [ ] `invoke()` method accepts optional `signal: AbortSignal` parameter
- [ ] Fetch requests use `signal` for cancellation
- [ ] Abort error mapped to E206 (request cancelled)
- [ ] Cleanup abort controller on request completion
- [ ] Multiple concurrent requests each have own controller
- [ ] Expose `abort()` method to cancel in-flight request

**Dependencies**: Sprint 1.5a complete (JAM-1.5a-008)

**Effort**: 30 minutes

**Deliverables**:
- Updated `/home/agent0/hx-docling-application/src/lib/mcp/client.ts` with AbortController support

**Technical Notes**:
- Reference FR-406 specification (cancel endpoint)
- AbortController pattern allows graceful request cancellation
- Map `AbortError` to E206 error code
- Used by /api/v1/jobs/{id}/cancel endpoint
- Coordinate with William on job cancellation flow

---

### JAM-1.5b-002: Add MCP Cancellation Error Handling

**Description**: Extend error catalog and mapping to handle cancellation scenarios gracefully.

**File**: `src/lib/mcp/errors.ts` (add E206 error)

**Acceptance Criteria**:
- [ ] E206 error code: "Request cancelled by user"
- [ ] User message: "Processing cancelled"
- [ ] Suggested action: "You can start a new job"
- [ ] `retryable: false` (user-initiated cancellation)
- [ ] Error serialization includes cancellation timestamp
- [ ] Update error mapping to detect AbortError

**Dependencies**: JAM-1.5b-001

**Effort**: 15 minutes

**Deliverables**:
- Updated `/home/agent0/hx-docling-application/src/lib/mcp/errors.ts`

**Technical Notes**:
- Distinguish cancellation (user action) from timeout (system error)
- E202 = timeout (retryable), E206 = cancelled (not retryable)
- Include cleanup logic for partial results

---

## Phase 3.2: MCP Progress Publishing

### JAM-1.5b-003: Implement MCP Progress Event Publisher

**Description**: Create utility to publish MCP processing progress to Redis pub/sub for SSE streaming.

**File**: `src/lib/mcp/progress-publisher.ts`

**Acceptance Criteria**:
- [ ] `publishProgress()` function: `(jobId, stage, percent, message) => Promise<void>`
- [ ] Redis publish to channel `job:{jobId}:progress`
- [ ] Event format matches SSE event structure
- [ ] Monotonic progress guarantee (never publish backwards progress)
- [ ] Stage tracking: 'uploaded', 'converted', 'export_markdown', 'export_html', 'export_json', 'complete'
- [ ] Error publishing: `publishError(jobId, error)` function
- [ ] Completion publishing: `publishComplete(jobId, resultId)` function

**Dependencies**: Sprint 1.2 complete (Redis client), Sprint 1.5a complete

**Effort**: 35 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/mcp/progress-publisher.ts`

**Technical Notes**:
- Coordinate with William on Redis pub/sub channel naming
- Event format: `{ stage, percent, message, timestamp }`
- Publish to Redis, William's SSE manager subscribes
- Ensure progress never goes backwards (track last percent)

---

### JAM-1.5b-004: Integrate Progress Publishing into Processing Pipeline

**Description**: Add progress event publishing at key stages of MCP processing workflow.

**File**: `src/lib/processing/processor.ts` (new file)

**Acceptance Criteria**:
- [ ] `processDocument()` orchestration function
- [ ] Publish 10% after file uploaded
- [ ] Publish 30% after MCP convert_* tool completes
- [ ] Publish 60% after export_markdown completes
- [ ] Publish 80% after export_html completes
- [ ] Publish 95% after export_json completes
- [ ] Publish 100% on job completion
- [ ] Publish error event on any failure
- [ ] Coordinate with checkpoint manager for resume capability

**Dependencies**: JAM-1.5b-003, William's checkpoint manager

**Effort**: 45 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/processing/processor.ts`

**Technical Notes**:
- Reference Specification Section 5.5 (checkpoint stages)
- Checkpoints: uploaded, converted, export_markdown, export_html, export_json
- Each checkpoint stored in database for resume capability
- Coordinate with William on checkpoint manager API
- This is the main orchestration function called by /api/v1/process

---

## Phase 3.3: MCP Checkpoint Integration

### JAM-1.5b-005: Implement MCP Processing Checkpoints

**Description**: Store intermediate MCP results at checkpoints to enable resume after interruption.

**File**: `src/lib/processing/checkpoints.ts`

**Acceptance Criteria**:
- [ ] `saveCheckpoint()` function: `(jobId, stage, data) => Promise<void>`
- [ ] `loadCheckpoint()` function: `(jobId) => Promise<Checkpoint | null>`
- [ ] Checkpoint stages align with MCP processing flow
- [ ] Store DoclingDocument at 'converted' checkpoint
- [ ] Store export results at export_* checkpoints
- [ ] Checkpoint data stored in PostgreSQL `Job.checkpointData` JSONB field
- [ ] Resume from latest checkpoint on retry

**Dependencies**: JAM-1.5b-004, William's checkpoint manager coordination

**Effort**: 30 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/processing/checkpoints.ts`

**Technical Notes**:
- Uses `Job.checkpointData` JSONB field per spec (0.3.1-detailed-specification.md §8.5)
- Coordinate with William on checkpoint manager interface
- Checkpoints enable RETRY_* state transitions without re-running MCP
- Serialize DoclingDocument as JSON for storage
- Consider size limits (DoclingDocument can be 1-50KB)
- Use `prisma.job.update()` with `checkpointData` field, not a separate table

---

## Phase 3.4: MCP Error Recovery

### JAM-1.5b-006: Implement MCP Transient Error Detection

**Description**: Enhance error handling to distinguish transient MCP errors (should retry) from permanent errors (should fail).

**File**: `src/lib/mcp/error-classification.ts`

**Acceptance Criteria**:
- [ ] `isTransientError()` function returning boolean
- [ ] Transient: E201 (unavailable), E202 (timeout), -32000 (server error), -32603 (internal error)
- [ ] Permanent: E203 (invalid), E204 (unknown tool), E205 (missing capability)
- [ ] Integration with retry logic from Sprint 1.5a
- [ ] Error classification logged for monitoring
- [ ] Return classification in error response

**Dependencies**: JAM-1.5a-010 (error mapping)

**Effort**: 20 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/mcp/error-classification.ts`

**Technical Notes**:
- Used by state machine for PROCESSING -> RETRY_1 transitions
- Transient errors trigger retry with checkpoint resume
- Permanent errors immediately transition to FAILED state
- Coordinate with William on retry state machine

---

## Phase 3.5: Testing & Integration

### JAM-1.5b-007: Write Unit Tests for MCP Progress Publishing

**Description**: Test progress event publishing to Redis with correct event format and monotonic guarantee.

**File**: `src/lib/mcp/__tests__/progress-publisher.test.ts`

**Acceptance Criteria**:
- [ ] Test publishProgress() publishes to correct Redis channel
- [ ] Test event format matches SSE structure
- [ ] Test monotonic progress (reject backwards progress)
- [ ] Test error event publishing
- [ ] Test completion event publishing
- [ ] Mock Redis client
- [ ] 100% coverage of progress-publisher.ts

**Dependencies**: JAM-1.5b-003

**Effort**: 20 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/mcp/__tests__/progress-publisher.test.ts`

**Technical Notes**:
- Use Vitest with ioredis-mock
- Verify Redis publish called with correct channel
- Test monotonic guarantee: 50% -> 40% should stay at 50%

---

### JAM-1.5b-008: Write Integration Tests for MCP Processing Flow

**Description**: End-to-end tests for document processing pipeline with MCP client, progress publishing, and checkpoints.

**File**: `src/lib/processing/__tests__/processor.integration.test.ts`

**Acceptance Criteria**:
- [ ] Test successful PDF processing flow (upload -> convert -> export)
- [ ] Test progress events published at each stage
- [ ] Test checkpoints saved at each stage
- [ ] Test resume from checkpoint after interruption
- [ ] Test error handling and retry logic
- [ ] Test cancellation mid-processing
- [ ] Mock MCP server responses
- [ ] Integration with real Redis (testcontainers)

**Dependencies**: JAM-1.5b-004, JAM-1.5b-005

**Effort**: 45 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/processing/__tests__/processor.integration.test.ts`

**Technical Notes**:
- Use MSW to mock MCP server
- Use testcontainers for Redis integration
- Test full processing pipeline end-to-end
- Verify checkpoint resume logic
- Coordinate with William on SSE subscription testing

---

## Dependencies

**Sequential Dependencies**:
- JAM-1.5b-001 blocks JAM-1.5b-002
- JAM-1.5b-003 blocks JAM-1.5b-004, JAM-1.5b-007
- JAM-1.5b-004 blocks JAM-1.5b-005, JAM-1.5b-008
- JAM-1.5b-005 blocks JAM-1.5b-008

**Parallel Opportunities**:
- JAM-1.5b-001 [P] JAM-1.5b-003 (independent modules)
- JAM-1.5b-002 [P] JAM-1.5b-006 (both error handling, different aspects)
- JAM-1.5b-007 [P] JAM-1.5b-006 (independent)

**Coordinate with William**:
- William creates checkpoint manager interface
- James implements MCP-specific checkpoint storage
- William handles SSE subscription to Redis
- James handles Redis publishing from MCP processing

---

## Total Effort Summary

| Phase | Tasks | Total Hours |
|-------|-------|-------------|
| 3.1: MCP Abortion | JAM-1.5b-001 to JAM-1.5b-002 | 0.75h |
| 3.2: Progress Publishing | JAM-1.5b-003 to JAM-1.5b-004 | 1.33h |
| 3.3: Checkpoints | JAM-1.5b-005 | 0.50h |
| 3.4: Error Recovery | JAM-1.5b-006 | 0.33h |
| 3.5: Testing | JAM-1.5b-007 to JAM-1.5b-008 | 1.08h |
| **Total** | **8 tasks** | **4.00h** |

**Note**: Sprint 1.5b is 4.0h total. William leads SSE infrastructure (~2.5h), James handles MCP integration (~1.5h). Tasks are coordinated but can be executed in parallel after interfaces defined.

---

## Acceptance Criteria (Sprint Level - MCP Aspects)

- [ ] MCP requests can be aborted via AbortController
- [ ] Cancellation mapped to E206 error with user-friendly message
- [ ] Progress events published to Redis at each MCP processing stage
- [ ] Monotonic progress guarantee enforced (no backwards progress)
- [ ] Checkpoints saved after each MCP tool invocation
- [ ] Resume from checkpoint works after interruption
- [ ] Transient MCP errors (E201, E202) trigger retry with checkpoint resume
- [ ] Permanent MCP errors (E203, E204, E205) immediately fail
- [ ] All unit tests pass with >= 80% coverage
- [ ] Integration tests verify end-to-end MCP processing flow

---

## Integration Points

**Coordinates with**:
- **William** (SSE Lead): Redis pub/sub channel format, checkpoint manager API
- **Neo** (Frontend): Cancellation button triggers AbortController via API
- **Trinity** (Database): Checkpoint storage schema in PostgreSQL
- **Sophia** (LangGraph): Error classification patterns for orchestration

**Escalation Points**:
- Checkpoint resume failures: Coordinate with William and Trinity
- Progress event format issues: Coordinate with William on SSE event structure
- MCP cancellation not working: Check AbortController signal propagation
