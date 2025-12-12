# Tasks: Agent Orchestration - Job Cancellation & Resume Workflow (Sprint 1.7)

**Agent**: Sophia Martinez (@sophia) - LangGraph Agent Orchestration SME
**Sprint**: 1.7 - History View & Persistence
**Duration**: Partial session (~1.5h of 3.5h total)
**Lead**: Neo (@neo)
**Support**: Ola (@ola), Trinity (@trinity), Sophia (@sophia)
**Review**: Julia (@julia)

---

## Responsibility Split: Bob (API Contract) vs Sophia (Orchestration)

This document focuses on **orchestration and business logic** for cancel/resume workflows.
For authoritative API contract details, see:
- `/home/agent0/hx-docling-application/project/0.4-tasks/sprint-1.7-history/bob-api-tasks.md`

| Responsibility | Owner | Examples |
|----------------|-------|----------|
| **API Contract** | Bob (@bob) | Error codes (E501-E705), HTTP status codes, request/response schemas, route signatures, validation schemas, content negotiation |
| **Orchestration Logic** | Sophia (@sophia) | State machine coordination, AbortController registry, checkpoint loading/clearing, SSE event publishing, cleanup behavior, transaction safety |
| **Route Handlers** | Joint (Bob + Sophia) | Bob provides the contract (validation, error responses); Sophia implements the business flow within that contract |

**Note on Route Handler Tasks**: SOP-1.7-001 and SOP-1.7-004 define route handlers that implement orchestration logic. The API contract (error codes, HTTP semantics, Zod schemas) is defined authoritatively in `bob-api-tasks.md` (BOB-1.7-003, BOB-1.7-004). Sophia's tasks focus on the orchestration function calls within those handlers.

---

**Input**:
- `/home/agent0/hx-docling-application/project/0.3-specification/0.3.1-detailed-specification.md` (Section 5.5, FR-406, 4.3.1)
- `/home/agent0/hx-docling-application/project/0.1-plan/0.1.1-implementation-plan.md` (Section 4.7, Tasks 10-11, 14)

**Prerequisites**:
- Sprint 1.5b complete (checkpoint manager, SSE streaming)
- State machine from Sprint 1.5a operational
- MCP AbortController integration from Sprint 1.5b
- Progress tracking implemented

---

## Context

Sprint 1.7 is primarily led by Neo for the history view UI. Sophia contributes orchestration patterns for:
- Job cancellation workflow implementation
- Job resume workflow implementation
- State transition coordination during cancel/resume
- Cleanup behavior configuration

**Neo's Tasks** (not in this file):
- History page layout
- HistoryView with table (shadcn Table)
- JobRow component with status badges
- Pagination component
- History API with pagination and session filter
- Job detail API and modal
- Re-download functionality
- useHistory hook

**Bob's Tasks** (not in this file):
- API route design and validation
- Error response formatting

---

## Design Note: Resume Endpoint Error Codes (E703, E704, E705)

> **API Contract Reference**: For authoritative error code definitions, HTTP status codes, and response schemas, see `bob-api-tasks.md` BOB-1.7-004 and "Resume Error Code Details" section.

The resume endpoint uses three distinct error codes for checkpoint-related failures. These codes are **intentionally separate** because they represent different failure modes requiring different user actions:

| Code | Name | Condition | User Action |
|------|------|-----------|-------------|
| **E703** | Checkpoint Missing | `checkpointData === null` OR job status not in `RESUMABLE_STATES` | Restart job from beginning |
| **E704** | Checkpoint Expired | `savedAt + CHECKPOINT_TTL_HOURS` exceeded (default 24h) | Restart job from beginning |
| **E705** | Checkpoint Corrupted | MD5 checksum validation failed OR JSON parse error | Restart job from beginning (indicates data integrity issue) |

**Validation Order**: The backend validates in this sequence: status check (E703) → null check (E703) → expiry check (E704) → integrity check (E705)

**Rationale**: While all three require restarting the job, the distinction enables:
1. **Debugging**: Operators can identify whether checkpoints are expiring (TTL tuning needed) vs corrupting (storage issues)
2. **Metrics**: Track failure mode distribution for system health monitoring
3. **Future UX**: Could offer different recovery options (e.g., extend TTL for E704, but not for E705)

See also:
- Specification Section 5.5 (Error Responses table)
- `sprint-1.5b/sophia-orchestration-tasks.md` SOP-1.5b-002 (Checkpoint Error Classes)
- `sprint-1.7-history/bob-api-tasks.md` Resume Error Code Details section (authoritative API contract)

---

## Phase 3.0: AbortController Registry (Prerequisite)

### SOP-1.7-000: Create AbortController Registry Module

**Description**: Create a centralized registry module for managing AbortController instances by jobId. This module enables coordinated cancellation between the cancel endpoint, MCP client, and route handlers.

**File**: `src/lib/abortRegistry.ts`

**Acceptance Criteria**:
- [ ] Singleton `Map<string, AbortController>` for single-instance deployments
- [ ] `initRegistry(): void` - initialize or reset the registry (useful for testing)
- [ ] `registerController(jobId: string): AbortController` - create and store a new controller
- [ ] `getController(jobId: string): AbortController | undefined` - retrieve by jobId
- [ ] `abortAndRemove(jobId: string): boolean` - abort controller and remove from registry
- [ ] `removeController(jobId: string): boolean` - remove without aborting (for cleanup after completion)
- [ ] `hasController(jobId: string): boolean` - check if job has active controller
- [ ] Export all functions and the registry type for testing
- [ ] Document coordination points with James's MCP client

**Dependencies**: None (prerequisite for SOP-1.7-001 and SOP-1.7-002)

**Effort**: 20 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/abortRegistry.ts`

**Technical Notes**:
- **Single-Instance Deployment**: Uses in-memory `Map<string, AbortController>` singleton
- **Multi-Instance Deployment**: For horizontally scaled deployments, the in-memory registry only works for the instance that initiated the request. Cancellation from a different instance requires one of:
  1. **Redis Pub/Sub**: Publish cancel event to `job:{jobId}:cancel` channel; all instances subscribe and abort locally if they hold the controller
  2. **Database Flag**: Set `Job.cancelRequested = true`; processing worker checks flag periodically
  3. **Sticky Sessions**: Route all requests for a job to the same instance (load balancer affinity)
- For Phase 1 (single hx-cc-server deployment), in-memory registry is sufficient
- Cleanup MUST occur on both cancel AND terminal state transitions to prevent memory leaks
- The MCP client (James's responsibility) MUST call `registerController()` before invoking MCP tools and `removeController()` on completion
- The cancel endpoint (SOP-1.7-001) calls `abortAndRemove()` to trigger cancellation

**Coordination Points with James (MCP Client)**:
- MCP client creates controller via `registerController(jobId)` before `invoke()`
- MCP client passes `controller.signal` to the fetch/request
- MCP client calls `removeController(jobId)` in finally block after request completes
- Cancel endpoint calls `abortAndRemove(jobId)` to trigger abort

```typescript
// AbortController Registry - src/lib/abortRegistry.ts
// Single-instance deployment using in-memory Map

/**
 * AbortController Registry for job cancellation coordination.
 *
 * IMPORTANT: This registry is process-local. For multi-instance deployments,
 * additional coordination is required via Redis pub/sub or database flags.
 * See Technical Notes in SOP-1.7-000 for details.
 */

// Singleton registry - one Map per process
let registry: Map<string, AbortController> = new Map();

/**
 * Initialize or reset the registry.
 * Useful for testing to ensure clean state.
 */
export function initRegistry(): void {
  registry = new Map();
}

/**
 * Register a new AbortController for a job.
 * Called by MCP client before invoking MCP tools.
 *
 * @param jobId - The job identifier
 * @returns The created AbortController (caller should use .signal)
 * @throws Error if jobId already has an active controller (indicates bug)
 */
export function registerController(jobId: string): AbortController {
  if (registry.has(jobId)) {
    console.warn(`[abortRegistry] Controller already exists for job ${jobId}, replacing`);
    // Abort existing controller before replacing to prevent orphans
    registry.get(jobId)?.abort();
  }

  const controller = new AbortController();
  registry.set(jobId, controller);
  return controller;
}

/**
 * Get the AbortController for a job, if one exists.
 *
 * @param jobId - The job identifier
 * @returns The controller or undefined if not found
 */
export function getController(jobId: string): AbortController | undefined {
  return registry.get(jobId);
}

/**
 * Check if a job has an active AbortController.
 *
 * @param jobId - The job identifier
 * @returns true if controller exists
 */
export function hasController(jobId: string): boolean {
  return registry.has(jobId);
}

/**
 * Abort the controller and remove from registry.
 * Called by cancel endpoint to trigger cancellation.
 *
 * @param jobId - The job identifier
 * @returns true if controller existed and was aborted, false if not found
 */
export function abortAndRemove(jobId: string): boolean {
  const controller = registry.get(jobId);
  if (controller) {
    controller.abort();
    registry.delete(jobId);
    return true;
  }
  return false;
}

/**
 * Remove controller from registry without aborting.
 * Called by MCP client after request completes (success or error).
 * MUST be called to prevent memory leaks.
 *
 * @param jobId - The job identifier
 * @returns true if controller existed and was removed
 */
export function removeController(jobId: string): boolean {
  return registry.delete(jobId);
}

/**
 * Get current registry size (for monitoring/debugging).
 */
export function getRegistrySize(): number {
  return registry.size;
}

// Export type for testing
export type AbortRegistry = Map<string, AbortController>;
```

---

## Phase 3.1: Job Cancellation Workflow

### SOP-1.7-001: Implement Cancel Endpoint Orchestration Logic

**Description**: Implement the orchestration logic called by the cancel route handler. The route handler structure, error codes, and response schemas are defined in `bob-api-tasks.md` BOB-1.7-003.

> **Joint Task**: Bob owns API contract (route signature, error codes E501/E502/E702, response schema). Sophia owns the `cancelJob()` orchestration function called from within the route handler.

**File**: `src/app/api/v1/jobs/[id]/cancel/route.ts` (route handler calls orchestration from `src/lib/job/cancel.ts`)

**Acceptance Criteria** (Sophia's orchestration scope):
- [ ] Call `cancelJob()` orchestration function from route handler
- [ ] Transition job to CANCELLED via state machine
- [ ] Trigger AbortController to stop MCP processing
- [ ] Send SSE `cancelled` event to connected clients
- [ ] Handle partial result cleanup per configuration
- [ ] Log cancellation for audit trail

**API Contract** (see BOB-1.7-003 for authoritative details):
- Route: `POST /api/v1/jobs/{id}/cancel`
- Error codes: E501 (not found), E702 (not cancellable)
- Request/response schemas defined in bob-api-tasks.md

**Dependencies**: State machine (SOP-1.5a-003), MCP AbortController (James's task), BOB-1.7-003 (API contract)

**Effort**: 45 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/job/cancel.ts` (orchestration logic - Sophia's primary deliverable)
- Route handler integration with Bob's API contract

**Technical Notes**:
- Reference Specification FR-406 for complete acceptance criteria
- Reference Specification Section 4.3.1 for cancellation state transitions
- Coordinate with James for AbortController abort() call
- SSE event sent via Redis pub/sub (job:{jobId}:progress channel)
- CLEANUP_ON_CANCEL env variable controls partial result behavior
- Route handler validation/error responses follow Bob's contract in BOB-1.7-003

---

### SOP-1.7-002: Implement Cancel Job Orchestration Function

**Description**: Create the core cancellation orchestration logic that coordinates multiple subsystems.

**File**: `src/lib/job/cancel.ts`

**Acceptance Criteria**:
- [ ] `cancelJob(jobId: string): Promise<CancelResult>` function
- [ ] Abort in-flight MCP request via AbortController
- [ ] Transition job status to CANCELLED
- [ ] Publish SSE `cancelled` event
- [ ] Handle partial result cleanup:
  - If CLEANUP_ON_CANCEL=true: delete partial results from database
  - If CLEANUP_ON_CANCEL=false: preserve partial results, mark as partial
- [ ] Clear checkpoint data (optional based on config)
- [ ] Return `CancelResult` with: `{ jobId, cancelledAt, partialResultsPreserved }`
- [ ] Transaction safety for multi-step cleanup
- [ ] Error handling: rollback on failure

**Dependencies**: SOP-1.5a-003 (state machine), SOP-1.5b-005 (checkpoint clear)

**Effort**: 40 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/job/cancel.ts`

**Technical Notes**:
- Reference Specification Section 4.3.1 for cleanup behavior
- AbortController stored in global registry by jobId
- SSE event via Redis: `job:{jobId}:progress` channel
- Environment variable: `CLEANUP_ON_CANCEL` (default: true)
- Use Prisma transaction for atomic multi-step operations

```typescript
// Cancellation orchestration per Specification FR-406
import { abortAndRemove } from '@/lib/abortRegistry';
import { transitionJobStatus } from '@/lib/job/transition-service';
import { publishCancelled } from '@/lib/sse/publisher';
import { clearCheckpoint } from '@/lib/checkpoint/manager';
import { prisma } from '@/lib/prisma';

interface CancelResult {
  jobId: string;
  cancelledAt: string;
  partialResultsPreserved: boolean;
}

export class CancellationError extends Error {
  constructor(
    message: string,
    public readonly jobId: string,
    public readonly phase: 'transaction' | 'abort' | 'publish',
    public readonly cause?: Error
  ) {
    super(message);
    this.name = 'CancellationError';
  }
}

export async function cancelJob(jobId: string): Promise<CancelResult> {
  // 1. Determine cleanup behavior from environment
  const cleanupOnCancel = process.env.CLEANUP_ON_CANCEL !== 'false';
  const cancelledAt = new Date().toISOString();

  // 2. Execute DB operations in transaction FIRST for atomicity
  // This ensures the job state is persisted before we abort the controller
  try {
    await prisma.$transaction(async (tx) => {
      // Transition to CANCELLED via state machine
      await transitionJobStatus(jobId, 'CANCELLED');

      // Handle partial results based on configuration
      if (cleanupOnCancel) {
        await tx.result.deleteMany({ where: { jobId } });
      }

      // Clear checkpoint data
      await clearCheckpoint(jobId);
    });
  } catch (error) {
    // Transaction failed - abort controller was NOT called, so no orphaned requests
    throw new CancellationError(
      `Failed to transition job ${jobId} to CANCELLED state`,
      jobId,
      'transaction',
      error instanceof Error ? error : undefined
    );
  }

  // 3. Abort MCP request AFTER transaction succeeds
  // This ensures DB state is consistent before removing the controller
  // If this fails, the job is already marked CANCELLED in DB, which is the safe state
  try {
    const wasAborted = abortAndRemove(jobId);
    if (wasAborted) {
      console.log(`[cancelJob] Aborted in-flight MCP request for job ${jobId}`);
    }
  } catch (error) {
    // Log but don't throw - job is already CANCELLED in DB
    // The MCP request will eventually timeout or complete and find job is cancelled
    console.error(
      `[cancelJob] Failed to abort controller for job ${jobId}, but job is marked CANCELLED:`,
      error
    );
  }

  // 4. Publish SSE event to notify connected clients
  try {
    await publishCancelled(jobId, {
      cancelledAt,
      partialResultsPreserved: !cleanupOnCancel,
    });
  } catch (error) {
    // Log but don't throw - cancellation succeeded, just notification failed
    // Clients will see CANCELLED status on next poll
    console.error(
      `[cancelJob] Failed to publish cancelled event for job ${jobId}:`,
      error
    );
  }

  return {
    jobId,
    cancelledAt,
    partialResultsPreserved: !cleanupOnCancel,
  };
}
```

---

### SOP-1.7-003: Implement Cancellation State Transitions

**Description**: Define and validate the specific state transitions allowed for job cancellation.

**File**: `src/lib/job/cancel-transitions.ts`

**Acceptance Criteria**:
- [ ] `CANCELLABLE_STATES` constant: ['PENDING', 'UPLOADING', 'PROCESSING', 'RETRY_1', 'RETRY_2', 'RETRY_3']
- [ ] `isCancellable(status: JobStatus): boolean` type guard
- [ ] `assertCancellable(status: JobStatus): void` throws if not cancellable
- [ ] Transition behavior per state:
  - PENDING -> CANCELLED: immediate
  - UPLOADING -> CANCELLED: abort upload, cleanup temp file
  - PROCESSING -> CANCELLED: abort MCP, cleanup per config
  - RETRY_* -> CANCELLED: stop retry loop, cleanup per config
- [ ] Document expected cleanup actions per source state
- [ ] Export all functions and constants

**Dependencies**: SOP-1.5a-001 (JobStatus types)

**Effort**: 20 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/job/cancel-transitions.ts`

**Technical Notes**:
- Reference Specification Section 4.3.1 cancellation state transitions
- All transitions to CANCELLED are valid from non-terminal states
- Different cleanup actions based on source state
- Type guard enables compile-time state validation

```typescript
// Per Specification Section 4.3.1
export const CANCELLABLE_STATES: JobStatus[] = [
  'PENDING',
  'UPLOADING',
  'PROCESSING',
  'RETRY_1',
  'RETRY_2',
  'RETRY_3',
];

export function isCancellable(status: JobStatus): status is CancellableStatus {
  return CANCELLABLE_STATES.includes(status);
}

export function assertCancellable(status: JobStatus): void {
  if (!isCancellable(status)) {
    throw new TerminalStateError(status);
  }
}
```

---

## Phase 3.2: Job Resume Workflow

### SOP-1.7-004: Implement Resume Endpoint Orchestration Logic

**Description**: Implement the orchestration logic called by the resume route handler. The route handler structure, error codes, and response schemas are defined in `bob-api-tasks.md` BOB-1.7-004.

> **Joint Task**: Bob owns API contract (route signature, error codes E501/E703/E704/E705, response schema). Sophia owns the `resumeJobFromCheckpoint()` orchestration function called from within the route handler.

**File**: `src/app/api/v1/jobs/[id]/resume/route.ts` (route handler calls orchestration from `src/lib/job/resume.ts`)

**Acceptance Criteria** (Sophia's orchestration scope):
- [ ] Call `resumeJobFromCheckpoint()` orchestration function from route handler
- [ ] Load checkpoint and determine resume stage
- [ ] Initiate processing from checkpoint stage
- [ ] Transition job status to PROCESSING
- [ ] Coordinate with processing pipeline

**API Contract** (see BOB-1.7-004 for authoritative details):
- Route: `POST /api/v1/jobs/{id}/resume`
- Error codes: E501 (not found), E703 (not resumable), E704 (checkpoint expired), E705 (checkpoint corrupted)
- Request/response schemas defined in bob-api-tasks.md
- Validation order: status check (E703) → expiry check (E704) → integrity check (E705)

**Dependencies**: SOP-1.5b-004 (loadCheckpoint), SOP-1.5a-003 (state machine), BOB-1.7-004 (API contract)

**Effort**: 40 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/job/resume.ts` (orchestration logic - Sophia's primary deliverable)
- Route handler integration with Bob's API contract

**Technical Notes**:
- Reference Specification Section 5.5 for resume endpoint specification
- Resume logic: load checkpoint -> determine stage -> call processor
- SSE connection re-established by client after resume
- Route handler validation/error responses follow Bob's contract in BOB-1.7-004

---

### SOP-1.7-005: Implement Resume Job Orchestration Function

**Description**: Create the core resume orchestration logic that continues processing from a checkpoint.

**File**: `src/lib/job/resume.ts`

**Acceptance Criteria**:
- [ ] `resumeJobFromCheckpoint(jobId: string, checkpoint: Checkpoint): Promise<void>`
- [ ] Determine which processing steps to skip based on checkpoint stage
- [ ] Transition job status to PROCESSING
- [ ] Resume logic per checkpoint stage:
  - 'uploaded': re-run conversion and all exports
  - 'converted': re-run all exports
  - 'export_markdown': re-run HTML and JSON exports
  - 'export_html': re-run JSON export only
  - 'export_json': finalize job (all exports complete)
  - 'completed': throw E706 (already done)
- [ ] Publish SSE `progress` event to notify UI
- [ ] Coordinate with processing pipeline (James's processor.ts)
- [ ] Update checkpoint as new stages complete

**Dependencies**: SOP-1.5b-003 (checkpoint save), processing pipeline

**Effort**: 35 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/job/resume.ts`

**Technical Notes**:
- Reference Specification Section 5.5 resume logic
- Use checkpoint.doclingDocument for conversion stage
- Use checkpoint.exportResults for partial export results
- Integration with James's processor.ts required
- SSE state_sync event sent on resume

```typescript
// Resume orchestration per Specification Section 5.5
export async function resumeJobFromCheckpoint(
  jobId: string,
  checkpoint: Checkpoint
): Promise<void> {
  // Transition to PROCESSING
  await transitionJobStatus(jobId, 'PROCESSING');

  // Publish resume event
  await publishProgress(jobId, {
    stage: checkpoint.stage,
    percent: getProgressForStage(checkpoint.stage),
    message: `Resuming from ${checkpoint.stage}...`,
  });

  // Resume based on checkpoint stage
  switch (checkpoint.stage) {
    case 'uploaded':
      await runConversion(jobId, checkpoint.uploadedFilePath!);
      break;
    case 'converted':
      await runAllExports(jobId, checkpoint.doclingDocument!);
      break;
    case 'export_markdown':
    case 'export_html':
    case 'export_json':
      await runRemainingExports(jobId, checkpoint);
      break;
    case 'completed':
      throw new AppError('E706', 'Job already completed');
  }
}

async function runRemainingExports(
  jobId: string,
  checkpoint: Checkpoint
): Promise<void> {
  const completed = checkpoint.exportResults;

  if (!completed.markdown) {
    await runExport(jobId, 'markdown', checkpoint.doclingDocument!);
  }
  if (!completed.html) {
    await runExport(jobId, 'html', checkpoint.doclingDocument!);
  }
  if (!completed.json) {
    await runExport(jobId, 'json', checkpoint.doclingDocument!);
  }

  // Finalize
  await finalizeJob(jobId);
}
```

---

### SOP-1.7-006: Implement Resumable State Detection

**Description**: Create utility functions to determine if a job can be resumed and from which stage.

**File**: `src/lib/job/resumable.ts`

**Acceptance Criteria**:
- [ ] `isResumable(job: Job): boolean` - checks if job has valid checkpoint
- [ ] `getResumeStage(checkpoint: Checkpoint): CheckpointStage` - determines resume point
- [ ] `getSkippedStages(checkpoint: Checkpoint): CheckpointStage[]` - lists completed stages
- [ ] `getRemainingStages(checkpoint: Checkpoint): CheckpointStage[]` - lists pending stages
- [ ] Handle edge cases:
  - No checkpoint: not resumable
  - Expired checkpoint: not resumable
  - Corrupted checkpoint: not resumable
  - Terminal state job: not resumable
- [ ] Export functions for use in API routes and UI

**Dependencies**: SOP-1.5b-001 (Checkpoint types), SOP-1.5b-004 (loadCheckpoint)

**Effort**: 20 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/job/resumable.ts`

**Technical Notes**:
- Used by history UI to show "Resume" button conditionally
- Used by resume endpoint to validate request
- Checkpoint validation delegated to loadCheckpoint()
- Stage ordering defined in SOP-1.5b-008

```typescript
// Resumability detection
const STAGE_ORDER: CheckpointStage[] = [
  'uploaded',
  'converted',
  'export_markdown',
  'export_html',
  'export_json',
  'completed',
];

export function getSkippedStages(checkpoint: Checkpoint): CheckpointStage[] {
  const currentIndex = STAGE_ORDER.indexOf(checkpoint.stage);
  return STAGE_ORDER.slice(0, currentIndex + 1);
}

export function getRemainingStages(checkpoint: Checkpoint): CheckpointStage[] {
  const currentIndex = STAGE_ORDER.indexOf(checkpoint.stage);
  return STAGE_ORDER.slice(currentIndex + 1);
}

export async function isResumable(job: Job): Promise<boolean> {
  if (isTerminalStatus(job.status)) return false;

  try {
    const checkpoint = await loadCheckpoint(job.id);
    return checkpoint !== null;
  } catch {
    return false;
  }
}
```

---

## Phase 3.3: Unit Tests for Cancel/Resume

### SOP-1.7-007: Write Unit Tests for Cancel/Resume Workflows

**Description**: Create comprehensive unit tests for job cancellation and resume workflows.

**File**: `src/lib/job/cancel.test.ts` and `src/lib/job/resume.test.ts`

**Acceptance Criteria**:
- [ ] Test `cancelJob()`:
  - Cancels job from PENDING state
  - Cancels job from PROCESSING state
  - Aborts in-flight MCP request
  - Publishes SSE cancelled event
  - Cleanup behavior when CLEANUP_ON_CANCEL=true
  - Preserve behavior when CLEANUP_ON_CANCEL=false
  - Rejects cancellation of terminal state job
- [ ] Test `resumeJobFromCheckpoint()`:
  - Resumes from 'uploaded' stage
  - Resumes from 'converted' stage
  - Resumes from partial export stages
  - Rejects resume of completed job
  - Handles missing checkpoint
  - Handles expired checkpoint
  - Handles corrupted checkpoint
- [ ] Test state transitions:
  - Verify CANCELLED is terminal
  - Verify all cancellable states transition correctly
- [ ] Mock dependencies: Prisma, Redis, MCP client
- [ ] >= 90% coverage for cancel and resume modules

**Dependencies**: SOP-1.7-001 through SOP-1.7-006

**Effort**: 45 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/job/cancel.test.ts`
- `/home/agent0/hx-docling-application/src/lib/job/resume.test.ts`

**Technical Notes**:
- Reference Plan Section 4.7 Task 14: "Write unit tests for cancel/resume endpoints"
- Mock AbortController and verify abort() called
- Mock Redis publisher for SSE events
- Test both success and error paths
- Test environment variable handling (CLEANUP_ON_CANCEL)

**Required Error Path Tests** (ensure each error code is validated):
```typescript
// resume.test.ts - Error code validation tests
describe('Resume Error Paths', () => {
  it('returns E703 when job status is not resumable', async () => {
    // Setup: job with status='COMPLETED'
    const response = await POST(request, { params: { id: jobId } });
    expect(response.status).toBe(409);
    const data = await response.json();
    expect(data.error.code).toBe('E703');
    expect(data.error.message).toContain('not in resumable state');
  });

  it('returns E703 when checkpoint data is null', async () => {
    // Setup: job with status='ERROR', checkpointData=null
    const response = await POST(request, { params: { id: jobId } });
    expect(response.status).toBe(409);
    const data = await response.json();
    expect(data.error.code).toBe('E703');
    expect(data.error.message).toContain('No valid checkpoint');
  });

  it('returns E704 when checkpoint TTL has expired', async () => {
    // Setup: job with checkpointExpiry in the past
    const response = await POST(request, { params: { id: jobId } });
    expect(response.status).toBe(409);
    const data = await response.json();
    expect(data.error.code).toBe('E704');
    expect(data.error.message).toContain('expired');
  });

  it('returns E705 when checkpoint checksum is invalid', async () => {
    // Setup: job with modified checkpointData (checksum mismatch)
    const response = await POST(request, { params: { id: jobId } });
    expect(response.status).toBe(409);
    const data = await response.json();
    expect(data.error.code).toBe('E705');
    expect(data.error.message).toContain('corrupted');
  });

  it('returns E705 when checkpoint deserialization fails', async () => {
    // Setup: job with malformed JSON in checkpointData
    const response = await POST(request, { params: { id: jobId } });
    expect(response.status).toBe(409);
    const data = await response.json();
    expect(data.error.code).toBe('E705');
    expect(data.error.message).toContain('corrupted');
  });
});
```

---

## Dependencies

| Task ID | Depends On | Reason |
|---------|------------|--------|
| SOP-1.7-000 | None | Prerequisite - AbortController registry module |
| SOP-1.7-001 | SOP-1.7-000, SOP-1.5a-003 | AbortRegistry, state machine |
| SOP-1.7-002 | SOP-1.7-000, SOP-1.7-003, SOP-1.5b-005 | AbortRegistry, cancel transitions, checkpoint clear |
| SOP-1.7-003 | SOP-1.5a-001 | JobStatus types |
| SOP-1.7-004 | SOP-1.5b-004, SOP-1.5a-003 | Checkpoint load, state machine |
| SOP-1.7-005 | SOP-1.7-006, processing pipeline | Resumability, processor integration |
| SOP-1.7-006 | SOP-1.5b-001, SOP-1.5b-004 | Checkpoint types and load |
| SOP-1.7-007 | All above tasks | Tests all implementations |

---

## Parallel Execution

```
# AbortRegistry prerequisite (first):
[SOP-1.7-000]

# Tasks that can run in parallel after SOP-1.7-000 (different files):
[SOP-1.7-003] [SOP-1.7-006]

# Cancel flow (sequential, after SOP-1.7-000):
SOP-1.7-000 -> SOP-1.7-003 -> SOP-1.7-002 -> SOP-1.7-001

# Resume flow (sequential):
SOP-1.7-006 -> SOP-1.7-005 -> SOP-1.7-004

# Tests last:
[SOP-1.7-007]
```

---

## Summary

| Task ID | Description | Effort | File | Notes |
|---------|-------------|--------|------|-------|
| SOP-1.7-000 | Create AbortController Registry Module | 20m | `src/lib/abortRegistry.ts` | Sophia owns |
| SOP-1.7-001 | Implement Cancel Endpoint Orchestration Logic | 45m | `src/lib/job/cancel.ts` | Joint w/ BOB-1.7-003 |
| SOP-1.7-002 | Implement Cancel Job Orchestration Function | 40m | `src/lib/job/cancel.ts` | Sophia owns |
| SOP-1.7-003 | Implement Cancellation State Transitions | 20m | `src/lib/job/cancel-transitions.ts` | Sophia owns |
| SOP-1.7-004 | Implement Resume Endpoint Orchestration Logic | 40m | `src/lib/job/resume.ts` | Joint w/ BOB-1.7-004 |
| SOP-1.7-005 | Implement Resume Job Orchestration Function | 35m | `src/lib/job/resume.ts` | Sophia owns |
| SOP-1.7-006 | Implement Resumable State Detection | 20m | `src/lib/job/resumable.ts` | Sophia owns |
| SOP-1.7-007 | Write Unit Tests for Cancel/Resume Workflows | 45m | `src/lib/job/*.test.ts` | Sophia owns |

**Total Effort**: 4h 25m (265 minutes)

> **Cross-Reference**: For API contract details (error codes, HTTP status, request/response schemas), see `bob-api-tasks.md` BOB-1.7-003 (Cancel) and BOB-1.7-004 (Resume).

---

## Quality Checklist

Before marking tasks complete:

**Orchestration (Sophia's scope)**:
- [ ] AbortController registry initialized and accessible from MCP client and cancel endpoint
- [ ] Registry cleanup occurs on both cancel AND terminal state transitions (no memory leaks)
- [ ] AbortController integration verified with MCP client (James coordination complete)
- [ ] SSE cancelled event published correctly
- [ ] Cleanup behavior matches CLEANUP_ON_CANCEL setting
- [ ] Checkpoint cleared after successful cancellation
- [ ] Resume continues from correct checkpoint stage
- [ ] State transitions valid per state machine
- [ ] Unit tests pass with >= 90% coverage
- [ ] Multi-instance deployment considerations documented (Redis pub/sub or DB flag approach)

**API Contract (Bob's scope - verify alignment)**:
- [ ] Cancel endpoint returns correct status codes per BOB-1.7-003 (404, 409, 200)
- [ ] Resume endpoint returns correct status codes per BOB-1.7-004 (404, 409, 200)
- [ ] Error codes match authoritative definitions in bob-api-tasks.md

**Integration**:
- [ ] Integration with Neo's history UI verified
- [ ] Code follows HX-Infrastructure standards
