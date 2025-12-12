# Tasks: Agent Orchestration - Progress Tracking & Checkpoint Manager (Sprint 1.5b)

**Agent**: Sophia Martinez (@sophia) - LangGraph Agent Orchestration SME
**Sprint**: 1.5b - SSE & Progress Integration
**Duration**: Partial session (~1.5h of 4.0h total)
**Lead**: William (@william)
**Support**: Neo (@neo), Ola (@ola), Sophia (@sophia)
**Review**: Alex (@alex)

**Input**:
- `/home/agent0/hx-docling-application/project/0.3-specification/0.3.1-detailed-specification.md` (Section 5.5, 4.2, 9.2)
- `/home/agent0/hx-docling-application/project/0.1-plan/0.1.1-implementation-plan.md` (Section 4.5b, Tasks 9-10, 16-17)

**Prerequisites**:
- Sprint 1.5a complete (MCP client, state machine)
- Redis client configured with TLS
- Prisma schema with checkpointData field (JsonB)
- State machine validation from SOP-1.5a-002

---

## Context

Sprint 1.5b is primarily led by William for SSE streaming infrastructure. Sophia contributes orchestration patterns for:
- Checkpoint manager implementation (save/restore job state)
- Progress tracking with monotonic guarantee
- Processing workflow orchestration
- Error recovery patterns

**William's Tasks** (not in this file):
- SSE connection manager
- Exponential backoff reconnection
- Polling fallback mechanism
- Event buffering for Last-Event-ID
- State synchronization
- /api/v1/process POST handler
- SSE streaming response
- Progress UI components (ProgressCard, LoadingStates)

**James's Tasks** (not in this file):
- MCP AbortController integration
- MCP progress event publishing
- MCP error handling in processing pipeline

---

## Phase 3.1: Checkpoint Types and Schema

### SOP-1.5b-001: Define Checkpoint Types and Interfaces

**Description**: Create comprehensive TypeScript types for checkpoint management, including the checkpoint schema and stage definitions.

**File**: `src/types/checkpoint.ts`

**Acceptance Criteria**:
- [ ] `CheckpointStage` type: 'uploaded' | 'converted' | 'export_markdown' | 'export_html' | 'export_json' | 'completed'
- [ ] `Checkpoint` interface per Specification Section 5.5:
  - `jobId: string`
  - `version: string` (schema version for migration)
  - `stage: CheckpointStage`
  - `stageCompletedAt: string` (ISO 8601)
  - `uploadedFilePath?: string`
  - `doclingDocument?: DoclingDocument`
  - `exportResults: ExportResultMap`
  - `savedAt: string` (ISO 8601)
  - `expiresAt: string` (ISO 8601)
  - `checksum?: string` (MD5 for integrity)
- [ ] `ExportResultMap` type with format-specific results
- [ ] `ExportResult` interface: `{ format, content?, size?, error? }`
- [ ] `CHECKPOINT_VERSION` constant: '1.0.0'
- [ ] Type guard: `isValidCheckpoint()` for runtime validation
- [ ] Export all types

**Dependencies**: None (pure types)

**Effort**: 25 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/types/checkpoint.ts`

**Technical Notes**:
- Reference Specification Section 5.5 for complete schema
- Checkpoint version enables future migrations
- 24-hour TTL per JOB_CHECKPOINT_TTL_HOURS environment variable
- Checksum calculated without the checksum field itself
- ExportResultMap allows partial export tracking

```typescript
// Example structure per Specification Section 5.5
export type CheckpointStage =
  | 'uploaded'
  | 'converted'
  | 'export_markdown'
  | 'export_html'
  | 'export_json'
  | 'completed';

export interface Checkpoint {
  jobId: string;
  version: string;
  stage: CheckpointStage;
  stageCompletedAt: string;
  uploadedFilePath?: string;
  doclingDocument?: DoclingDocument;
  exportResults: ExportResultMap;
  savedAt: string;
  expiresAt: string;
  checksum?: string;
}

export interface ExportResultMap {
  markdown?: ExportResult;
  html?: ExportResult;
  json?: ExportResult;
}
```

---

### SOP-1.5b-002: Create Checkpoint Error Classes

**Description**: Define custom error classes for checkpoint operations to enable structured error handling.

**File**: `src/lib/checkpoint/errors.ts`

**Acceptance Criteria**:
- [ ] `CheckpointNotFoundError` class (E703):
  - Message: "No valid checkpoint exists for job: {jobId}"
  - Property: `jobId`
- [ ] `CheckpointExpiredError` class (E704):
  - Message: "Checkpoint expired for job: {jobId}"
  - Properties: `jobId`, `expiredAt`
- [ ] `CheckpointCorruptedError` class (E705):
  - Message: "Checkpoint data corrupted for job: {jobId}"
  - Properties: `jobId`, `expectedChecksum`, `actualChecksum`
- [ ] `CheckpointSaveError` class (E708):
  - Message: "Failed to save checkpoint for job: {jobId}"
  - Properties: `jobId`, `stage`, `cause`
- [ ] All errors extend base `AppError` class
- [ ] All errors include `retryable: false` (require user action)
- [ ] Export all error classes

**Dependencies**: Base AppError class

**Effort**: 20 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/checkpoint/errors.ts`

**Technical Notes**:
- Reference Specification Section 5.5 error responses
- E703-E705 are documented in resume endpoint specification
- These errors help the resume endpoint return appropriate HTTP status codes
- Include original cause for debugging when wrapping errors

---

## Phase 3.2: Checkpoint Manager Implementation

### SOP-1.5b-003: Implement Checkpoint Save Function

**Description**: Implement the checkpoint save function that persists job state at key processing stages.

**File**: `src/lib/checkpoint/manager.ts`

**Acceptance Criteria**:
- [ ] `saveCheckpoint(jobId: string, stage: CheckpointStage, data: Partial<Checkpoint>): Promise<void>`
- [ ] Calculate expiry: `savedAt + CHECKPOINT_TTL_HOURS` (default 24h)
- [ ] Generate checksum using MD5 of serialized checkpoint (excluding checksum field)
- [ ] Update job record with checkpoint data using Prisma
- [ ] Update `currentStage` field on job record
- [ ] Handle Prisma errors with `CheckpointSaveError`
- [ ] Log checkpoint save operations for debugging
- [ ] Validate stage progression (warn if stage goes backwards)
- [ ] Support partial updates (merge with existing checkpoint)

**Dependencies**: SOP-1.5b-001, SOP-1.5b-002, Prisma client

**Effort**: 40 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/checkpoint/manager.ts` (saveCheckpoint function)

**Technical Notes**:
- Reference Specification Section 5.5 `saveCheckpoint()` implementation
- Use crypto.createHash('md5') for checksum calculation
- CHECKPOINT_TTL_HOURS from environment (default: 24)
- Checkpoint stored in `checkpointData` field (JsonB in PostgreSQL)
- Merge partial data: useful for incremental export results

```typescript
// Per Specification Section 5.5
export async function saveCheckpoint(
  jobId: string,
  stage: CheckpointStage,
  data: Partial<Checkpoint>
): Promise<void> {
  const now = new Date();
  const ttlHours = parseInt(process.env.JOB_CHECKPOINT_TTL_HOURS || '24', 10);
  const expiresAt = new Date(now.getTime() + ttlHours * 60 * 60 * 1000);

  const checkpoint: Checkpoint = {
    jobId,
    version: CHECKPOINT_VERSION,
    stage,
    stageCompletedAt: now.toISOString(),
    exportResults: {},
    ...data,
    savedAt: now.toISOString(),
    expiresAt: expiresAt.toISOString(),
  };

  checkpoint.checksum = calculateChecksum(checkpoint);

  await prisma.job.update({
    where: { id: jobId },
    data: {
      checkpointData: checkpoint as Prisma.InputJsonValue,
      currentStage: stage,
    },
  });
}
```

---

### SOP-1.5b-004: Implement Checkpoint Load Function

**Description**: Implement the checkpoint load function that retrieves and validates stored checkpoints.

**File**: `src/lib/checkpoint/manager.ts` (add to existing file)

**Acceptance Criteria**:
- [ ] `loadCheckpoint(jobId: string): Promise<Checkpoint | null>`
- [ ] Retrieve job record with checkpointData field
- [ ] Return null if no checkpoint exists
- [ ] Validate checkpoint expiry (compare expiresAt with current time)
- [ ] Clear expired checkpoint and return null
- [ ] Validate checksum integrity (recalculate and compare)
- [ ] Throw `CheckpointCorruptedError` on checksum mismatch
- [ ] Return validated checkpoint on success
- [ ] Log checkpoint load operations for debugging

**Dependencies**: SOP-1.5b-001, SOP-1.5b-002, SOP-1.5b-003

**Effort**: 30 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/checkpoint/manager.ts` (loadCheckpoint function)

**Technical Notes**:
- Reference Specification Section 5.5 `loadCheckpoint()` implementation
- Checksum validation: remove checksum, calculate, compare
- Auto-clear expired checkpoints for cleanup
- Return null (not throw) for missing/expired - caller decides action

```typescript
// Per Specification Section 5.5
export async function loadCheckpoint(jobId: string): Promise<Checkpoint | null> {
  const job = await prisma.job.findUnique({
    where: { id: jobId },
    select: { checkpointData: true },
  });

  if (!job?.checkpointData) return null;

  const checkpoint = job.checkpointData as unknown as Checkpoint;

  // Validate expiry
  if (new Date(checkpoint.expiresAt) < new Date()) {
    await clearCheckpoint(jobId);
    return null;
  }

  // Validate checksum
  const savedChecksum = checkpoint.checksum;
  checkpoint.checksum = undefined;
  if (calculateChecksum(checkpoint) !== savedChecksum) {
    await clearCheckpoint(jobId);
    throw new CheckpointCorruptedError(jobId);
  }
  checkpoint.checksum = savedChecksum;

  return checkpoint;
}
```

---

### SOP-1.5b-005: Implement Checkpoint Clear Function

**Description**: Implement the checkpoint clear function for cleanup operations.

**File**: `src/lib/checkpoint/manager.ts` (add to existing file)

**Acceptance Criteria**:
- [ ] `clearCheckpoint(jobId: string): Promise<void>`
- [ ] Update job record to set checkpointData to null
- [ ] Optionally clear currentStage field
- [ ] Handle Prisma errors gracefully
- [ ] Log checkpoint clear operations
- [ ] Idempotent: clearing non-existent checkpoint is not an error

**Dependencies**: Prisma client

**Effort**: 10 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/checkpoint/manager.ts` (clearCheckpoint function)

**Technical Notes**:
- Called after successful job completion
- Called when checkpoint expires or corrupts
- Called on job deletion/cleanup
- Simple update operation, minimal complexity

```typescript
export async function clearCheckpoint(jobId: string): Promise<void> {
  await prisma.job.update({
    where: { id: jobId },
    data: { checkpointData: null },
  });
}
```

---

### SOP-1.5b-006: Implement Checksum Utility Function

**Description**: Create utility function for calculating checkpoint integrity checksums.

**File**: `src/lib/checkpoint/checksum.ts`

**Acceptance Criteria**:
- [ ] `calculateChecksum(checkpoint: Omit<Checkpoint, 'checksum'>): string`
- [ ] Use MD5 hash algorithm (crypto.createHash)
- [ ] Serialize checkpoint deterministically (sorted keys)
- [ ] Return hex-encoded hash string
- [ ] Exclude undefined/null fields from hash calculation
- [ ] Pure function: no side effects

**Dependencies**: Node.js crypto module

**Effort**: 15 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/checkpoint/checksum.ts`

**Technical Notes**:
- Deterministic serialization is critical for consistent checksums
- Use JSON.stringify with sorted keys replacer
- MD5 is sufficient for integrity validation (not security)
- Fast execution important for save/load performance

```typescript
import crypto from 'crypto';

export function calculateChecksum(
  checkpoint: Omit<Checkpoint, 'checksum'>
): string {
  const serialized = JSON.stringify(checkpoint, Object.keys(checkpoint).sort());
  return crypto.createHash('md5').update(serialized).digest('hex');
}
```

---

## Phase 3.3: Progress Tracking with Monotonic Guarantee

### SOP-1.5b-007: Implement Progress Interpolation Function

**Description**: Create progress interpolation utility that ensures progress values never decrease (monotonic guarantee).

**File**: `src/lib/progress/interpolation.ts`

**Acceptance Criteria**:
- [ ] `smoothProgress(current: number, target: number, lastReported: number): number`
- [ ] Monotonic guarantee: returned value >= lastReported
- [ ] Smooth interpolation: never jump more than configured max (e.g., 5%)
- [ ] Clamp to 0-100 range
- [ ] Handle edge cases: target < lastReported, current > 100
- [ ] `createProgressTracker(initialProgress?: number)` factory function
- [ ] Tracker maintains lastReported state internally
- [ ] Tracker exposes `update(newProgress)` method
- [ ] Tracker exposes `reset()` method for new jobs

**Dependencies**: None (pure functions)

**Effort**: 25 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/progress/interpolation.ts`

**Technical Notes**:
- Reference Specification Section 9.2 for progress interpolation requirements
- Monotonic guarantee critical for UX: users confused if progress goes backwards
- Smooth transitions improve perceived performance
- Progress tracker object maintains state between calls

```typescript
// Per Specification Section 9.2
export function smoothProgress(
  current: number,
  target: number,
  lastReported: number
): number {
  // Monotonic guarantee
  const minProgress = lastReported;

  // Smooth interpolation (max 5% jump)
  const maxJump = 5;
  const smoothed = Math.min(current, lastReported + maxJump);

  // Apply target and clamp
  const bounded = Math.max(minProgress, Math.min(target, smoothed));

  return Math.min(100, Math.max(0, bounded));
}

export function createProgressTracker(initialProgress = 0) {
  let lastReported = initialProgress;

  return {
    update(newProgress: number): number {
      const smoothed = smoothProgress(newProgress, newProgress, lastReported);
      lastReported = smoothed;
      return smoothed;
    },
    reset(): void {
      lastReported = 0;
    },
    get current(): number {
      return lastReported;
    },
  };
}
```

---

### SOP-1.5b-008: Define Progress Stage Mapping

**Description**: Create constants mapping checkpoint stages to progress percentages for consistent UI updates.

**File**: `src/lib/progress/stages.ts`

**Acceptance Criteria**:
- [ ] `STAGE_PROGRESS_MAP` constant mapping stages to percent ranges:
  - 'uploaded': 10
  - 'converted': 30
  - 'export_markdown': 60
  - 'export_html': 80
  - 'export_json': 95
  - 'completed': 100
- [ ] `STAGE_MESSAGES` constant with user-friendly messages:
  - 'uploaded': "Uploading document..."
  - 'converted': "Analyzing document structure..."
  - 'export_markdown': "Generating Markdown..."
  - 'export_html': "Generating HTML..."
  - 'export_json': "Generating JSON..."
  - 'completed': "Complete!"
- [ ] `getProgressForStage(stage: CheckpointStage): number` helper
- [ ] `getMessageForStage(stage: CheckpointStage): string` helper
- [ ] `getNextStage(current: CheckpointStage): CheckpointStage | null` helper

**Dependencies**: SOP-1.5b-001 (CheckpointStage type)

**Effort**: 15 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/progress/stages.ts`

**Technical Notes**:
- Reference Specification Section 4.2 for stage progress ranges
- Messages should be user-friendly (shown in ProgressCard)
- Progress values align with checkpoint save points
- Helper functions simplify processing pipeline code

---

## Phase 3.4: Unit Tests for Checkpoint Manager

### SOP-1.5b-009: Write Unit Tests for Checkpoint Manager

**Description**: Create comprehensive unit tests for checkpoint manager functions.

**File**: `src/lib/checkpoint/manager.test.ts`

**Acceptance Criteria**:
- [ ] Test `saveCheckpoint()`:
  - Saves checkpoint with correct data
  - Calculates checksum correctly
  - Sets expiry time correctly (24h TTL)
  - Updates job record in database
  - Handles Prisma errors gracefully
- [ ] Test `loadCheckpoint()`:
  - Returns null for missing checkpoint
  - Returns null and clears expired checkpoint
  - Throws `CheckpointCorruptedError` on bad checksum
  - Returns valid checkpoint with all fields
  - Validates checksum correctly
- [ ] Test `clearCheckpoint()`:
  - Clears existing checkpoint
  - Idempotent for non-existent checkpoint
- [ ] Mock Prisma client for unit tests
- [ ] >= 90% coverage for checkpoint manager

**Dependencies**: SOP-1.5b-003, SOP-1.5b-004, SOP-1.5b-005, Vitest

**Effort**: 35 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/checkpoint/manager.test.ts`

**Technical Notes**:
- Reference Plan Section 4.5b Task 16: "Write unit tests for checkpoint manager"
- Use Vitest mocking for Prisma
- Test checkpoint lifecycle: save -> load -> clear
- Test error scenarios: expired, corrupted, missing
- Test checksum validation thoroughly

---

### SOP-1.5b-010: Write Unit Tests for Progress Interpolation

**Description**: Create unit tests for progress interpolation with monotonic guarantee.

**File**: `src/lib/progress/interpolation.test.ts`

**Acceptance Criteria**:
- [ ] Test `smoothProgress()`:
  - Never returns value less than lastReported (monotonic)
  - Limits jumps to configured maximum
  - Clamps to 0-100 range
  - Handles edge cases (backwards progress, overflow)
- [ ] Test `createProgressTracker()`:
  - Initial progress starts at 0 or configured value
  - `update()` maintains monotonic guarantee
  - `reset()` returns tracker to initial state
  - Multiple sequential updates work correctly
- [ ] Test edge cases:
  - Progress already at 100
  - Target less than current
  - Negative input values
  - Very small increments
- [ ] >= 95% coverage for interpolation module

**Dependencies**: SOP-1.5b-007, Vitest

**Effort**: 25 minutes

**Deliverables**:
- `/home/agent0/hx-docling-application/src/lib/progress/interpolation.test.ts`

**Technical Notes**:
- Reference Plan Section 4.5b Task 17: "Write unit tests for progress interpolation"
- Monotonic guarantee is the critical requirement
- Test with many sequential updates to verify state management
- Test with real-world progress sequences from MCP

---

## Dependencies

| Task ID | Depends On | Reason |
|---------|------------|--------|
| SOP-1.5b-001 | None | Pure type definitions |
| SOP-1.5b-002 | AppError base class | Error class inheritance |
| SOP-1.5b-003 | SOP-1.5b-001, SOP-1.5b-002, SOP-1.5b-006 | Types, errors, checksum |
| SOP-1.5b-004 | SOP-1.5b-003 | Must have save before load |
| SOP-1.5b-005 | None (Prisma only) | Can be implemented independently |
| SOP-1.5b-006 | SOP-1.5b-001 | Uses Checkpoint type |
| SOP-1.5b-007 | None | Pure functions |
| SOP-1.5b-008 | SOP-1.5b-001 | Uses CheckpointStage type |
| SOP-1.5b-009 | SOP-1.5b-003, SOP-1.5b-004, SOP-1.5b-005 | Tests all manager functions |
| SOP-1.5b-010 | SOP-1.5b-007 | Tests interpolation functions |

---

## Parallel Execution

```
# Tasks that can run in parallel (different files, no dependencies):
[SOP-1.5b-001] [SOP-1.5b-007] [SOP-1.5b-005]

# After SOP-1.5b-001:
[SOP-1.5b-002] [SOP-1.5b-006] [SOP-1.5b-008]

# After errors and checksum:
[SOP-1.5b-003] -> [SOP-1.5b-004]

# Tests last:
[SOP-1.5b-009] [SOP-1.5b-010]
```

---

## Summary

| Task ID | Description | Effort | File |
|---------|-------------|--------|------|
| SOP-1.5b-001 | Define Checkpoint Types and Interfaces | 25m | `src/types/checkpoint.ts` |
| SOP-1.5b-002 | Create Checkpoint Error Classes | 20m | `src/lib/checkpoint/errors.ts` |
| SOP-1.5b-003 | Implement Checkpoint Save Function | 40m | `src/lib/checkpoint/manager.ts` |
| SOP-1.5b-004 | Implement Checkpoint Load Function | 30m | `src/lib/checkpoint/manager.ts` |
| SOP-1.5b-005 | Implement Checkpoint Clear Function | 10m | `src/lib/checkpoint/manager.ts` |
| SOP-1.5b-006 | Implement Checksum Utility Function | 15m | `src/lib/checkpoint/checksum.ts` |
| SOP-1.5b-007 | Implement Progress Interpolation | 25m | `src/lib/progress/interpolation.ts` |
| SOP-1.5b-008 | Define Progress Stage Mapping | 15m | `src/lib/progress/stages.ts` |
| SOP-1.5b-009 | Write Unit Tests for Checkpoint Manager | 35m | `src/lib/checkpoint/manager.test.ts` |
| SOP-1.5b-010 | Write Unit Tests for Progress Interpolation | 25m | `src/lib/progress/interpolation.test.ts` |

**Total Effort**: 4h 0m (240 minutes)

---

## Quality Checklist

Before marking tasks complete:
- [ ] All TypeScript types compile without errors
- [ ] Checkpoint save/load cycle verified
- [ ] Checksum validation works correctly
- [ ] Progress interpolation maintains monotonic guarantee
- [ ] Error classes map to correct error codes (E703-E708)
- [ ] Unit tests pass with >= 90% coverage
- [ ] Integration with job state machine verified
- [ ] Code follows HX-Infrastructure standards
- [ ] JSDoc documentation complete
