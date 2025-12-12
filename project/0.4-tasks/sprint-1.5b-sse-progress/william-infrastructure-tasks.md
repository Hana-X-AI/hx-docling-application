# Infrastructure Tasks: Sprint 1.5b - SSE & Progress Integration

**Sprint**: 1.5b - SSE & Progress Integration
**Lead**: William Chen (`@william`)
**Support**: Neo (`@neo`), Ola (`@ola`), Sophia (`@sophia`)
**Review**: Alex (`@alex`)
**Document Version**: 1.1.0
**Created**: 2025-12-12
**Updated**: 2025-12-12
**Reference**: `project/0.1-plan/0.1.1-implementation-plan.md` Section 4.5b

**Change Log v1.1.0**: Fixed 9 CodeRabbit defects (DEF-001 through DEF-009). Added shared types section, corrected useSSE/useProcess hooks, added subscribeToJobUpdates implementation, fixed accessibility issues.

---

## Overview

Sprint 1.5b establishes the Server-Sent Events (SSE) infrastructure for real-time progress tracking. This includes SSE connection management, event buffering, checkpoint management for recovery, and progress interpolation for smooth UI updates.

**Total Sprint Effort**: 4.0 hours

---

## Shared Types (Required for TypeScript Compilation)

The following shared types must be defined for all Sprint 1.5b code to compile correctly. These address CodeRabbit defects DEF-001, DEF-002, and DEF-003.

### Progress Interface (DEF-001 Fix)

```typescript
// src/types/progress.ts
/**
 * Progress update received from SSE events or polling.
 * Used throughout the progress tracking system.
 */
export interface Progress {
  status: 'PENDING' | 'PROCESSING' | 'COMPLETED' | 'FAILED';
  progress: number;       // 0-100
  stage: string;
  currentStage?: string;
  message?: string;
}
```

### AppError Class (DEF-002 Fix)

```typescript
// src/types/errors.ts
/**
 * Application-specific error class with code and HTTP status.
 * Used throughout the application for consistent error handling.
 */
export class AppError extends Error {
  constructor(
    public code: string,
    public statusCode: number,
    message: string
  ) {
    super(message);
    this.name = 'AppError';
    Object.setPrototypeOf(this, AppError.prototype);
  }

  toJSON() {
    return {
      name: this.name,
      code: this.code,
      statusCode: this.statusCode,
      message: this.message,
    };
  }
}
```

### ProcessResult Interface (DEF-003 Fix)

```typescript
// src/types/process.ts
/**
 * Result returned when a document processing job completes.
 */
export interface ProcessResult {
  jobId: string;
  status: 'COMPLETED' | 'FAILED';
  markdown?: string;
  html?: string;
  json?: string;
  error?: string;
  completedAt: number;
}
```

### Types Index Export

```typescript
// src/types/index.ts
export { Progress } from './progress';
export { AppError } from './errors';
export { ProcessResult } from './process';
```

---

## Task WIL-1.5b-001: SSE Connection Manager

### Description

Create the SSE connection manager that handles client connections, event streaming, and connection lifecycle management on the server side.

### Acceptance Criteria

- [ ] SSE endpoint streams events correctly
- [ ] Content-Type set to `text/event-stream`
- [ ] Connection kept alive with keep-alive messages
- [ ] Event ID tracking for reconnection support
- [ ] Proper cleanup on client disconnect
- [ ] Multiple concurrent connections supported

### Dependencies

- Sprint 1.5a MCP client complete
- Redis available for event buffering

### Effort Estimate

**Duration**: 30 minutes
**Complexity**: Medium

### Deliverables

1. `src/lib/sse/manager.ts` - SSE manager implementation
2. Connection lifecycle documentation

### Technical Notes

```typescript
// src/lib/sse/manager.ts
import { NextResponse } from 'next/server';

interface SSEManagerConfig {
  keepAliveInterval: number;  // ms
  maxConnectionTime: number;  // ms
}

const DEFAULT_CONFIG: SSEManagerConfig = {
  keepAliveInterval: 30000,  // 30 seconds
  maxConnectionTime: 3600000, // 1 hour max
};

export class SSEManager {
  private controller: ReadableStreamDefaultController | null = null;
  private encoder = new TextEncoder();
  private eventId = 0;
  private config: SSEManagerConfig;
  private keepAliveTimer: NodeJS.Timeout | null = null;

  constructor(config: Partial<SSEManagerConfig> = {}) {
    this.config = { ...DEFAULT_CONFIG, ...config };
  }

  createStream(): ReadableStream {
    return new ReadableStream({
      start: (controller) => {
        this.controller = controller;
        this.startKeepAlive();
      },
      cancel: () => {
        this.cleanup();
      },
    });
  }

  sendEvent(event: string, data: unknown): void {
    if (!this.controller) return;

    this.eventId++;
    const eventData = [
      `id: ${this.eventId}`,
      `event: ${event}`,
      `data: ${JSON.stringify(data)}`,
      '',
      '',
    ].join('\n');

    this.controller.enqueue(this.encoder.encode(eventData));
  }

  private startKeepAlive(): void {
    this.keepAliveTimer = setInterval(() => {
      if (this.controller) {
        this.controller.enqueue(this.encoder.encode(': keep-alive\n\n'));
      }
    }, this.config.keepAliveInterval);
  }

  private cleanup(): void {
    if (this.keepAliveTimer) {
      clearInterval(this.keepAliveTimer);
    }
    this.controller = null;
  }

  close(): void {
    this.cleanup();
  }
}

// Response helper
export function createSSEResponse(stream: ReadableStream): NextResponse {
  return new NextResponse(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache, no-transform',
      'Connection': 'keep-alive',
      'X-Accel-Buffering': 'no', // Disable nginx buffering
    },
  });
}
```

---

## Task WIL-1.5b-002: Exponential Backoff Reconnection Logic

### Description

Implement exponential backoff reconnection logic for SSE clients. This ensures graceful recovery from network interruptions without overwhelming the server.

### Acceptance Criteria

- [ ] Base retry interval: 1 second
- [ ] Maximum retry interval: 30 seconds
- [ ] Backoff multiplier: 2x
- [ ] Maximum retry attempts: 10 (then fallback to polling)
- [ ] Jitter added to prevent thundering herd
- [ ] Reconnection status exposed to UI

### Dependencies

- Task WIL-1.5b-001 completed

### Effort Estimate

**Duration**: 25 minutes
**Complexity**: Medium

### Deliverables

1. `src/lib/sse/reconnect.ts` - Reconnection logic
2. Reconnection configuration documentation

### Technical Notes

```typescript
// src/lib/sse/reconnect.ts
interface ReconnectConfig {
  baseDelay: number;      // 1000ms
  maxDelay: number;       // 30000ms
  multiplier: number;     // 2
  maxRetries: number;     // 10
  jitterFactor: number;   // 0.1 (10% jitter)
}

const DEFAULT_RECONNECT_CONFIG: ReconnectConfig = {
  baseDelay: 1000,
  maxDelay: 30000,
  multiplier: 2,
  maxRetries: 10,
  jitterFactor: 0.1,
};

export class ReconnectionManager {
  private retryCount = 0;
  private config: ReconnectConfig;
  private aborted = false;

  constructor(config: Partial<ReconnectConfig> = {}) {
    this.config = { ...DEFAULT_RECONNECT_CONFIG, ...config };
  }

  getNextDelay(): number | null {
    if (this.retryCount >= this.config.maxRetries) {
      return null; // Signal to switch to polling
    }

    const exponentialDelay = Math.min(
      this.config.baseDelay * Math.pow(this.config.multiplier, this.retryCount),
      this.config.maxDelay
    );

    // Add jitter to prevent thundering herd
    const jitter = exponentialDelay * this.config.jitterFactor * (Math.random() * 2 - 1);
    const delay = Math.round(exponentialDelay + jitter);

    this.retryCount++;
    return delay;
  }

  reset(): void {
    this.retryCount = 0;
    this.aborted = false;
  }

  abort(): void {
    this.aborted = true;
  }

  isAborted(): boolean {
    return this.aborted;
  }

  getRetryCount(): number {
    return this.retryCount;
  }

  shouldFallbackToPolling(): boolean {
    return this.retryCount >= this.config.maxRetries;
  }
}
```

---

## Task WIL-1.5b-003: Polling Fallback Mechanism

### Description

Implement a polling fallback mechanism that activates when SSE reconnection exhausts all retry attempts. This ensures progress updates continue even without SSE support.

### Acceptance Criteria

- [ ] Polling interval: 2 seconds
- [ ] Fallback triggers after SSE retry exhaustion
- [ ] Polling stops when job completes or is cancelled
- [ ] UI indicates "backup connection" mode
- [ ] Seamless switch from SSE to polling

### Dependencies

- Task WIL-1.5b-002 completed

### Effort Estimate

**Duration**: 30 minutes
**Complexity**: Medium

### Deliverables

1. Polling client implementation
2. Status endpoint for polling

### Technical Notes

```typescript
// src/lib/sse/polling-fallback.ts
import { Progress } from '@/types/progress';

// Note: Progress interface is defined in src/types/progress.ts
// export interface Progress {
//   status: 'PENDING' | 'PROCESSING' | 'COMPLETED' | 'FAILED';
//   progress: number;       // 0-100
//   stage: string;
//   currentStage?: string;
//   message?: string;
// }

interface PollingConfig {
  interval: number;         // 2000ms
  maxPollDuration: number;  // 600000ms (10 minutes)
}

export class PollingFallback {
  private pollingTimer: NodeJS.Timeout | null = null;
  private config: PollingConfig;
  private onProgress: (progress: Progress) => void;
  private jobId: string;

  constructor(
    jobId: string,
    onProgress: (progress: Progress) => void,
    config: Partial<PollingConfig> = {}
  ) {
    this.jobId = jobId;
    this.onProgress = onProgress;
    this.config = {
      interval: config.interval ?? 2000,
      maxPollDuration: config.maxPollDuration ?? 600000,
    };
  }

  start(): void {
    const startTime = Date.now();

    const poll = async () => {
      if (Date.now() - startTime > this.config.maxPollDuration) {
        this.stop();
        return;
      }

      // DEF-005 Fix: Add AbortController with 5s timeout
      const controller = new AbortController();
      const timeout = setTimeout(() => controller.abort(), 5000);

      try {
        const response = await fetch(`/api/v1/jobs/${this.jobId}/status`, {
          signal: controller.signal,
        });

        if (!response.ok) {
          throw new Error(`HTTP ${response.status}`);
        }

        // DEF-005 Fix: Wrap JSON parsing in try-catch
        let progress: Progress;
        try {
          progress = await response.json();
        } catch (parseError) {
          console.error('Failed to parse polling response:', parseError);
          this.pollingTimer = setTimeout(poll, this.config.interval);
          return;
        }

        this.onProgress(progress);

        if (progress.status === 'COMPLETED' || progress.status === 'FAILED') {
          this.stop();
          return;
        }
      } catch (error) {
        // DEF-005 Fix: Handle AbortError separately from other errors
        if (error instanceof DOMException && error.name === 'AbortError') {
          console.warn('Polling request timeout');
        } else {
          console.error('Polling error:', error);
        }
      } finally {
        // Always clear the timeout to prevent timer leaks
        clearTimeout(timeout);
      }

      this.pollingTimer = setTimeout(poll, this.config.interval);
    };

    poll();
  }

  stop(): void {
    if (this.pollingTimer) {
      clearTimeout(this.pollingTimer);
      this.pollingTimer = null;
    }
  }
}
```

---

## Task WIL-1.5b-004: Redis Event Buffering for Last-Event-ID

### Description

Implement Redis-based event buffering to support Last-Event-ID reconnection. Events are stored temporarily to enable clients to resume from their last received event.

### Acceptance Criteria

- [ ] Events stored in Redis with job-scoped keys
- [ ] Event TTL: 5 minutes (configurable)
- [ ] Last-Event-ID lookup on reconnection
- [ ] Missed events replayed on reconnect
- [ ] Buffer size limit per job (100 events max)
- [ ] Automatic cleanup after job completion

### Dependencies

- Task WIL-1.5b-001 completed
- Redis circuit breaker from Sprint 1.2

### Effort Estimate

**Duration**: 25 minutes
**Complexity**: Medium

### Deliverables

1. `src/lib/sse/event-buffer.ts` - Event buffer implementation
2. Redis key patterns documentation

### Technical Notes

```typescript
// src/lib/sse/event-buffer.ts
import { redis } from '@/lib/redis/client';
import { redisCircuitBreaker } from '@/lib/redis/circuit-breaker';

const EVENT_TTL = 300; // 5 minutes
const MAX_EVENTS_PER_JOB = 100;
const EVENT_BUFFER_PREFIX = 'sse:events:';

interface BufferedEvent {
  id: number;
  event: string;
  data: unknown;
  timestamp: number;
}

export class EventBuffer {
  private jobId: string;

  constructor(jobId: string) {
    this.jobId = jobId;
  }

  private get key(): string {
    return `${EVENT_BUFFER_PREFIX}${this.jobId}`;
  }

  async addEvent(eventId: number, event: string, data: unknown): Promise<void> {
    const bufferedEvent: BufferedEvent = {
      id: eventId,
      event,
      data,
      timestamp: Date.now(),
    };

    await redisCircuitBreaker.execute(
      async () => {
        const pipeline = redis.pipeline();
        pipeline.zadd(this.key, eventId, JSON.stringify(bufferedEvent));
        pipeline.expire(this.key, EVENT_TTL);

        // Trim to max events (keep highest IDs)
        pipeline.zremrangebyrank(this.key, 0, -MAX_EVENTS_PER_JOB - 1);

        await pipeline.exec();
      },
      () => { /* Fallback: event not buffered */ }
    );
  }

  async getEventsSince(lastEventId: number): Promise<BufferedEvent[]> {
    return redisCircuitBreaker.execute(
      async () => {
        const events = await redis.zrangebyscore(
          this.key,
          lastEventId + 1,
          '+inf'
        );
        return events.map(e => JSON.parse(e) as BufferedEvent);
      },
      () => [] // Fallback: no events available
    );
  }

  async cleanup(): Promise<void> {
    await redisCircuitBreaker.execute(
      () => redis.del(this.key),
      () => { /* Fallback: cleanup skipped */ }
    );
  }
}

// Redis key patterns for documentation:
// sse:events:{jobId} - Sorted set of buffered events (score = eventId)
// TTL: 5 minutes per key
// Cleanup: Automatic via TTL or explicit on job completion
```

---

## Task WIL-1.5b-005: State Synchronization on Reconnect

### Description

Implement state synchronization logic that runs when a client reconnects to SSE. This ensures the client has the latest progress state after recovering from a disconnection.

### Acceptance Criteria

- [ ] Current job state fetched on reconnect
- [ ] Missed events replayed from buffer
- [ ] Progress never appears to go backwards
- [ ] UI state updated atomically
- [ ] Reconnection completes within 2 seconds

### Dependencies

- Task WIL-1.5b-004 completed

### Effort Estimate

**Duration**: 20 minutes
**Complexity**: Medium

### Deliverables

1. State sync logic in SSE handler
2. Reconnection sequence documentation

### Technical Notes

```typescript
// State synchronization on reconnect
async function handleReconnection(
  jobId: string,
  lastEventId: number,
  sseManager: SSEManager,
  eventBuffer: EventBuffer
): Promise<void> {
  // 1. Get current job state
  const currentState = await prisma.job.findUnique({
    where: { id: jobId },
    select: { status: true, progress: true, currentStage: true },
  });

  if (!currentState) {
    sseManager.sendEvent('error', { code: 'E404', message: 'Job not found' });
    return;
  }

  // 2. Get missed events from buffer
  const missedEvents = await eventBuffer.getEventsSince(lastEventId);

  // 3. Replay missed events in order
  for (const event of missedEvents) {
    sseManager.sendEvent(event.event, event.data);
  }

  // 4. Send current state sync event
  sseManager.sendEvent('sync', {
    status: currentState.status,
    progress: currentState.progress,
    stage: currentState.currentStage,
    reconnected: true,
  });
}
```

---

## Task WIL-1.5b-006: Process Route POST Handler

### Description

Create the `/api/v1/process` POST endpoint that initiates document processing and returns a job ID for progress tracking.

### Acceptance Criteria

- [ ] Accepts job initiation request with file/URL reference
- [ ] Creates job record in database (PENDING status)
- [ ] Returns job ID for SSE connection
- [ ] Validates session ownership
- [ ] Rate limiting applied
- [ ] Triggers background processing

### Dependencies

- Sprint 1.5a MCP client complete
- Sprint 1.2 database/session infrastructure

### Effort Estimate

**Duration**: 30 minutes
**Complexity**: Medium

### Deliverables

1. `src/app/api/v1/process/route.ts` - Process endpoint
2. Job creation flow documentation

### Technical Notes

```typescript
// src/app/api/v1/process/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { prisma } from '@/lib/db/prisma';
import { getSession } from '@/lib/redis/session';
import { checkRateLimit, getRateLimitHeaders } from '@/lib/middleware/rate-limit';
import { validateBody } from '@/lib/middleware/validate';
import { z } from 'zod';

const processRequestSchema = z.object({
  fileId: z.string().uuid().optional(),
  url: z.string().url().optional(),
}).refine(data => data.fileId || data.url, {
  message: 'Either fileId or url must be provided',
});

export async function POST(request: NextRequest) {
  // 1. Get session
  const sessionId = request.headers.get('X-Session-Id');
  if (!sessionId) {
    return NextResponse.json(
      { code: 'E101', message: 'Session required' },
      { status: 401 }
    );
  }

  // 2. Check rate limit
  const rateLimit = await checkRateLimit(sessionId, 'process', { maxRequests: 5 });
  if (!rateLimit.allowed) {
    return NextResponse.json(
      { code: 'E429', message: 'Rate limit exceeded' },
      { status: 429, headers: getRateLimitHeaders(rateLimit) }
    );
  }

  // 3. Validate request body
  const validation = await validateBody(processRequestSchema)(request);
  if (validation instanceof NextResponse) return validation;

  // 4. Create job record
  const job = await prisma.job.create({
    data: {
      sessionId,
      status: 'PENDING',
      fileId: validation.fileId,
      sourceUrl: validation.url,
    },
  });

  // 5. Trigger background processing (non-blocking)
  // This will be handled by a separate background worker or queue
  processJobAsync(job.id).catch(console.error);

  // 6. Return job ID
  return NextResponse.json(
    { jobId: job.id, status: 'PENDING' },
    { status: 201, headers: getRateLimitHeaders(rateLimit) }
  );
}
```

---

## Task WIL-1.5b-007: SSE Streaming Response Implementation

### Description

Implement the SSE streaming response endpoint that clients connect to for receiving real-time progress updates.

### Acceptance Criteria

- [ ] Endpoint at `/api/v1/process/{jobId}/events`
- [ ] Validates job ownership (session match)
- [ ] Handles Last-Event-ID header for reconnection
- [ ] Streams progress events in real-time
- [ ] Sends completion/error events
- [ ] Proper connection cleanup on close

### Dependencies

- Task WIL-1.5b-001 completed
- Task WIL-1.5b-004 completed
- Task WIL-1.5b-006 completed

### Effort Estimate

**Duration**: 30 minutes
**Complexity**: High

### Deliverables

1. `src/app/api/v1/process/[jobId]/events/route.ts` - SSE endpoint
2. Event stream documentation

### Technical Notes

```typescript
// src/app/api/v1/process/[jobId]/events/route.ts
import { NextRequest } from 'next/server';
import { SSEManager, createSSEResponse } from '@/lib/sse/manager';
import { EventBuffer } from '@/lib/sse/event-buffer';
import { prisma } from '@/lib/db/prisma';
import { subscribeToJobUpdates } from '@/lib/sse/job-updates';

// DEF-004 Fix: subscribeToJobUpdates is defined in src/lib/sse/job-updates.ts
// See implementation below after route handler

export async function GET(
  request: NextRequest,
  { params }: { params: { jobId: string } }
) {
  const { jobId } = params;
  const sessionId = request.headers.get('X-Session-Id');
  const lastEventId = parseInt(request.headers.get('Last-Event-ID') || '0');

  // 1. Validate job exists and belongs to session
  const job = await prisma.job.findFirst({
    where: { id: jobId, sessionId: sessionId! },
  });

  if (!job) {
    return new Response('Job not found', { status: 404 });
  }

  // 2. Create SSE manager
  const sseManager = new SSEManager();
  const eventBuffer = new EventBuffer(jobId);

  // 3. Handle reconnection with Last-Event-ID
  if (lastEventId > 0) {
    const missedEvents = await eventBuffer.getEventsSince(lastEventId);
    for (const event of missedEvents) {
      sseManager.sendEvent(event.event, event.data);
    }
  }

  // 4. Subscribe to job updates (via Redis pub/sub or polling)
  const unsubscribe = subscribeToJobUpdates(jobId, (update) => {
    sseManager.sendEvent(update.event, update.data);
    eventBuffer.addEvent(update.eventId, update.event, update.data);
  });

  // 5. Handle cleanup
  request.signal.addEventListener('abort', () => {
    unsubscribe();
    sseManager.close();
  });

  return createSSEResponse(sseManager.createStream());
}
```

**DEF-004 Fix: subscribeToJobUpdates Implementation**

```typescript
// src/lib/sse/job-updates.ts
import { redis } from '@/lib/redis/client';

interface JobUpdate {
  event: string;
  data: unknown;
  eventId: number;
}

/**
 * Subscribe to real-time job updates via Redis Pub/Sub.
 * Returns an unsubscribe function to clean up the subscription.
 */
export function subscribeToJobUpdates(
  jobId: string,
  callback: (update: JobUpdate) => void
): () => void {
  const channel = `job:${jobId}:updates`;
  let isSubscribed = true;

  // Create a dedicated Redis connection for subscription
  const subscriber = redis.duplicate();

  subscriber.subscribe(channel, (err) => {
    if (err) {
      console.error('Failed to subscribe to job updates:', err);
      return;
    }
  });

  subscriber.on('message', (receivedChannel, message) => {
    if (!isSubscribed || receivedChannel !== channel) return;

    try {
      const update = JSON.parse(message) as JobUpdate;
      callback(update);
    } catch (error) {
      console.error('Failed to parse job update:', error);
    }
  });

  // Return unsubscribe function
  return () => {
    isSubscribed = false;
    subscriber.unsubscribe(channel);
    subscriber.quit();
  };
}

/**
 * Publish a job update event.
 * Called by the job processing worker when progress changes.
 */
export async function publishJobUpdate(
  jobId: string,
  update: JobUpdate
): Promise<void> {
  const channel = `job:${jobId}:updates`;
  await redis.publish(channel, JSON.stringify(update));
}
```

---

## Task WIL-1.5b-008: Last-Event-ID Reconnection Support

### Description

Implement full support for Last-Event-ID header in SSE reconnection, including event replay and state synchronization.

### Acceptance Criteria

- [ ] Last-Event-ID header parsed correctly
- [ ] Events replayed from buffer in order
- [ ] No duplicate events sent
- [ ] State sync event sent after replay
- [ ] Handles case when buffer is expired

### Dependencies

- Task WIL-1.5b-004 completed
- Task WIL-1.5b-007 completed

### Effort Estimate

**Duration**: 20 minutes
**Complexity**: Medium

### Deliverables

1. Reconnection handler integration
2. Reconnection test scenarios

### Technical Notes

```typescript
// Reconnection sequence:
// 1. Client disconnects (network issue)
// 2. Client reconnects with Last-Event-ID header
// 3. Server receives reconnection request
// 4. Server fetches events since Last-Event-ID from buffer
// 5. Server replays missed events
// 6. Server sends sync event with current state
// 7. Normal streaming resumes

// Edge cases:
// - Buffer expired (events older than TTL): Start from current state
// - Job completed during disconnect: Send completion event
// - Job failed during disconnect: Send error event
```

---

## Task WIL-1.5b-009: Checkpoint Manager Implementation

### Description

Implement the checkpoint manager for stage-based recovery. Checkpoints allow jobs to resume from the last successful stage after a failure.

### Acceptance Criteria

- [ ] Checkpoint saved on each stage completion
- [ ] Checkpoint includes: stage, progress, partial results
- [ ] Checkpoint stored in database with job record
- [ ] Checkpoint restore initializes job state correctly
- [ ] Checkpoint cleaned up on job completion
- [ ] Maximum checkpoint size: 1MB

### Dependencies

- Database schema supports checkpoint field
- Job status includes RETRY states

### Effort Estimate

**Duration**: 45 minutes
**Complexity**: High

### Deliverables

1. `src/lib/checkpoint/manager.ts` - Checkpoint manager
2. Checkpoint serialization format documentation

### Technical Notes

```typescript
// src/lib/checkpoint/manager.ts
import { prisma } from '@/lib/db/prisma';
import { z } from 'zod';

// Checkpoint schema
const checkpointSchema = z.object({
  version: z.literal(1),
  stage: z.enum(['upload', 'parsing', 'conversion', 'export', 'saving']),
  progress: z.number().min(0).max(100),
  startedAt: z.number(),
  lastUpdatedAt: z.number(),
  partialResults: z.object({
    markdown: z.string().optional(),
    html: z.string().optional(),
    json: z.string().optional(),
  }).optional(),
  metadata: z.record(z.unknown()).optional(),
});

type Checkpoint = z.infer<typeof checkpointSchema>;

const MAX_CHECKPOINT_SIZE = 1024 * 1024; // 1MB

export class CheckpointManager {
  private jobId: string;

  constructor(jobId: string) {
    this.jobId = jobId;
  }

  async save(checkpoint: Omit<Checkpoint, 'version'>): Promise<void> {
    const fullCheckpoint: Checkpoint = {
      version: 1,
      ...checkpoint,
      lastUpdatedAt: Date.now(),
    };

    const serialized = JSON.stringify(fullCheckpoint);
    if (serialized.length > MAX_CHECKPOINT_SIZE) {
      throw new Error('Checkpoint exceeds maximum size');
    }

    await prisma.job.update({
      where: { id: this.jobId },
      data: {
        checkpoint: serialized,
        lastCheckpointAt: new Date(),
      },
    });
  }

  async restore(): Promise<Checkpoint | null> {
    const job = await prisma.job.findUnique({
      where: { id: this.jobId },
      select: { checkpoint: true },
    });

    if (!job?.checkpoint) return null;

    try {
      const checkpoint = JSON.parse(job.checkpoint as string);
      return checkpointSchema.parse(checkpoint);
    } catch (error) {
      console.error('Failed to restore checkpoint:', error);
      return null;
    }
  }

  async clear(): Promise<void> {
    await prisma.job.update({
      where: { id: this.jobId },
      data: {
        checkpoint: null,
        lastCheckpointAt: null,
      },
    });
  }

  static async getResumeableJobs(sessionId: string): Promise<string[]> {
    const jobs = await prisma.job.findMany({
      where: {
        sessionId,
        checkpoint: { not: null },
        status: { in: ['RETRY_PENDING', 'FAILED'] },
      },
      select: { id: true },
    });

    return jobs.map(j => j.id);
  }
}

// Checkpoint serialization format:
// {
//   "version": 1,
//   "stage": "conversion",
//   "progress": 65,
//   "startedAt": 1702345678000,
//   "lastUpdatedAt": 1702345690000,
//   "partialResults": {
//     "markdown": "# Document Title\n..."
//   },
//   "metadata": {
//     "mcpRequestId": "uuid",
//     "fileHash": "sha256"
//   }
// }
```

---

## Task WIL-1.5b-010: Progress Interpolation with Monotonic Guarantee

### Description

Implement progress interpolation that smooths progress updates for better UX while guaranteeing progress never decreases (monotonic property).

### Acceptance Criteria

- [ ] Progress interpolated between updates for smooth animation
- [ ] Progress NEVER decreases (monotonic)
- [ ] Maximum interpolation rate: 1% per 100ms
- [ ] Interpolation pauses if no updates for 5 seconds
- [ ] Actual progress can jump ahead of interpolated
- [ ] Final jump to 100% on completion

### Dependencies

- Task WIL-1.5b-007 completed

### Effort Estimate

**Duration**: 30 minutes
**Complexity**: Medium

### Deliverables

1. `src/lib/progress/interpolation.ts` - Progress interpolation
2. Monotonic guarantee tests

### Technical Notes

```typescript
// src/lib/progress/interpolation.ts
interface ProgressState {
  actual: number;      // Actual progress from server
  displayed: number;   // Interpolated progress shown to user
  lastUpdate: number;  // Timestamp of last server update
}

const INTERPOLATION_INTERVAL = 100; // ms
const MAX_RATE = 1; // 1% per interval
const STALE_THRESHOLD = 5000; // 5 seconds

export class ProgressInterpolator {
  private state: ProgressState;
  private timer: NodeJS.Timeout | null = null;
  private onUpdate: (progress: number) => void;

  constructor(onUpdate: (progress: number) => void) {
    this.state = {
      actual: 0,
      displayed: 0,
      lastUpdate: Date.now(),
    };
    this.onUpdate = onUpdate;
  }

  start(): void {
    this.timer = setInterval(() => this.interpolate(), INTERPOLATION_INTERVAL);
  }

  stop(): void {
    if (this.timer) {
      clearInterval(this.timer);
      this.timer = null;
    }
  }

  setActualProgress(progress: number): void {
    // Monotonic guarantee: never decrease
    if (progress < this.state.actual && progress !== 0) {
      console.warn('Ignoring non-monotonic progress update');
      return;
    }

    this.state.actual = progress;
    this.state.lastUpdate = Date.now();

    // If actual jumps ahead significantly, catch up immediately
    if (progress > this.state.displayed + 10) {
      this.state.displayed = progress - 5; // Smooth catch-up
    }
  }

  complete(): void {
    this.state.actual = 100;
    this.state.displayed = 100;
    this.onUpdate(100);
    this.stop();
  }

  private interpolate(): void {
    const timeSinceUpdate = Date.now() - this.state.lastUpdate;

    // Pause interpolation if updates are stale
    if (timeSinceUpdate > STALE_THRESHOLD) {
      return;
    }

    // Interpolate towards actual, respecting max rate
    if (this.state.displayed < this.state.actual) {
      const diff = this.state.actual - this.state.displayed;
      const increment = Math.min(diff, MAX_RATE);
      this.state.displayed = Math.min(
        this.state.displayed + increment,
        this.state.actual
      );
      this.onUpdate(this.state.displayed);
    }
  }

  getDisplayedProgress(): number {
    return this.state.displayed;
  }
}
```

---

## Task WIL-1.5b-011: ProgressCard Component

### Description

Create the ProgressCard component that displays processing progress with stage information, progress bar, and status messages.

### Acceptance Criteria

- [ ] Progress bar shows interpolated progress
- [ ] Current stage displayed with description
- [ ] Stage icons for visual feedback
- [ ] Estimated time remaining (when available)
- [ ] Animation smooth and performant
- [ ] Accessible (ARIA progress attributes)

### Dependencies

- Task WIL-1.5b-010 completed
- shadcn/ui components from Sprint 1.1

### Effort Estimate

**Duration**: 25 minutes
**Complexity**: Medium

### Deliverables

1. `src/components/processing/ProgressCard.tsx`
2. Progress card styling

### Technical Notes

```typescript
// src/components/processing/ProgressCard.tsx
'use client';

import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Progress } from '@/components/ui/progress';
import { Loader2 } from 'lucide-react';

interface ProgressCardProps {
  progress: number;
  stage: string;
  message: string;
  isProcessing: boolean;
}

const STAGE_INFO: Record<string, { icon: string; label: string }> = {
  upload: { icon: 'Upload', label: 'Uploading document' },
  parsing: { icon: 'FileSearch', label: 'Analyzing structure' },
  conversion: { icon: 'Cpu', label: 'Processing with AI' },
  export: { icon: 'FileOutput', label: 'Generating outputs' },
  saving: { icon: 'Save', label: 'Saving results' },
  complete: { icon: 'Check', label: 'Complete' },
};

export function ProgressCard({
  progress,
  stage,
  message,
  isProcessing,
}: ProgressCardProps) {
  const stageInfo = STAGE_INFO[stage] || STAGE_INFO.parsing;

  return (
    <Card className="w-full max-w-md">
      <CardHeader className="pb-2">
        <CardTitle className="flex items-center gap-2 text-lg">
          {isProcessing && <Loader2 className="h-4 w-4 animate-spin" />}
          {stageInfo.label}
        </CardTitle>
      </CardHeader>
      <CardContent>
        <Progress
          value={progress}
          className="h-2 mb-2"
          aria-label="Processing progress"
          aria-valuenow={progress}
          aria-valuemin={0}
          aria-valuemax={100}
        />
        <p className="text-sm text-muted-foreground">
          {message || `${Math.round(progress)}% complete`}
        </p>
      </CardContent>
    </Card>
  );
}
```

---

## Task WIL-1.5b-012: Loading State Variants

### Description

Implement loading state variants for different processing stages, including skeleton loaders, spinners, and stage-specific indicators.

### Acceptance Criteria

- [ ] Skeleton loader for initial state
- [ ] Spinner for active processing
- [ ] Stage-specific icons and colors
- [ ] Smooth transitions between states
- [ ] Reduced motion support
- [ ] Consistent with design system

### Dependencies

- Task WIL-1.5b-011 completed

### Effort Estimate

**Duration**: 15 minutes
**Complexity**: Low

### Deliverables

1. `src/components/processing/LoadingStates.tsx`
2. Loading state variants

### Technical Notes

```typescript
// src/components/processing/LoadingStates.tsx
'use client';

import { Skeleton } from '@/components/ui/skeleton';
import { Loader2, FileSearch, Cpu, FileOutput, Save, Check } from 'lucide-react';
import { cn } from '@/lib/utils';

type LoadingVariant = 'skeleton' | 'spinner' | 'stage';

interface LoadingStateProps {
  variant: LoadingVariant;
  stage?: string;
  className?: string;
}

export function LoadingState({ variant, stage, className }: LoadingStateProps) {
  if (variant === 'skeleton') {
    return (
      <div className={cn('space-y-2', className)}>
        <Skeleton className="h-4 w-3/4" />
        <Skeleton className="h-2 w-full" />
        <Skeleton className="h-4 w-1/2" />
      </div>
    );
  }

  if (variant === 'spinner') {
    return (
      <div className={cn('flex items-center justify-center', className)}>
        <Loader2 className="h-8 w-8 animate-spin text-primary" />
      </div>
    );
  }

  // Stage variant with appropriate icon
  // DEF-007 Fix: Use motion-safe: prefix to respect prefers-reduced-motion
  const StageIcon = getStageIcon(stage);
  return (
    <div className={cn('flex items-center gap-2', className)}>
      <StageIcon
        className={cn(
          'h-5 w-5 text-primary',
          'motion-safe:animate-pulse'
        )}
      />
      <span className="text-sm text-muted-foreground">{getStageLabel(stage)}</span>
    </div>
  );
}

function getStageIcon(stage?: string) {
  switch (stage) {
    case 'upload': return Loader2;
    case 'parsing': return FileSearch;
    case 'conversion': return Cpu;
    case 'export': return FileOutput;
    case 'saving': return Save;
    case 'complete': return Check;
    default: return Loader2;
  }
}

function getStageLabel(stage?: string): string {
  // Return appropriate label for stage
  const labels: Record<string, string> = {
    upload: 'Uploading...',
    parsing: 'Analyzing...',
    conversion: 'Converting...',
    export: 'Exporting...',
    saving: 'Saving...',
    complete: 'Complete',
  };
  return labels[stage || ''] || 'Processing...';
}
```

---

## Task WIL-1.5b-013: Reconnection Overlay

### Description

Create the reconnection overlay component that displays when SSE connection is being re-established.

### Acceptance Criteria

- [ ] Overlay appears on SSE disconnection
- [ ] Shows retry count and next retry time
- [ ] "Reconnecting..." message displayed
- [ ] Spinner animation
- [ ] Disappears automatically on reconnection
- [ ] Option to cancel and use polling fallback

### Dependencies

- Task WIL-1.5b-002 completed

### Effort Estimate

**Duration**: 10 minutes
**Complexity**: Low

### Deliverables

1. Reconnection overlay component
2. Integration with SSE hook

### Technical Notes

```typescript
// Component integrated into processing page
interface ReconnectionOverlayProps {
  isReconnecting: boolean;
  retryCount: number;
  nextRetryIn: number; // seconds
  onSwitchToPolling: () => void;
}

export function ReconnectionOverlay({
  isReconnecting,
  retryCount,
  nextRetryIn,
  onSwitchToPolling,
}: ReconnectionOverlayProps) {
  if (!isReconnecting) return null;

  return (
    <div className="fixed inset-0 bg-background/80 backdrop-blur-sm z-50 flex items-center justify-center">
      <Card className="p-6">
        <div className="flex flex-col items-center gap-4">
          <Loader2 className="h-8 w-8 animate-spin" />
          <p className="text-lg font-medium">Reconnecting...</p>
          <p className="text-sm text-muted-foreground">
            Attempt {retryCount}/10 - Retrying in {nextRetryIn}s
          </p>
          <Button variant="outline" size="sm" onClick={onSwitchToPolling}>
            Use backup connection
          </Button>
        </div>
      </Card>
    </div>
  );
}
```

---

## Task WIL-1.5b-014: useSSE Hook

### Description

Create the useSSE hook that encapsulates all SSE connection logic, including connection management, reconnection, and progress state.

### Acceptance Criteria

- [ ] Manages EventSource lifecycle
- [ ] Handles automatic reconnection
- [ ] Exposes connection status
- [ ] Provides progress updates via callback
- [ ] Supports Last-Event-ID for reconnection
- [ ] Cleanup on unmount

### Dependencies

- Tasks WIL-1.5b-001 through WIL-1.5b-004 completed

### Effort Estimate

**Duration**: 20 minutes
**Complexity**: Medium

### Deliverables

1. `src/hooks/useSSE.ts`
2. Hook usage documentation

### Technical Notes

```typescript
// src/hooks/useSSE.ts
'use client';

import { useEffect, useRef, useState, useCallback } from 'react';
import { ReconnectionManager } from '@/lib/sse/reconnect';
import { PollingFallback } from '@/lib/sse/polling-fallback';
import { Progress } from '@/types/progress';

// DEF-008 Fix: Options can be null to support conditional calling
interface UseSSEOptions {
  jobId: string;
  onProgress: (progress: Progress) => void;
  onComplete: () => void;
  onError: (error: Error) => void;
}

interface UseSSEReturn {
  isConnected: boolean;
  isReconnecting: boolean;
  retryCount: number;
  usePolling: boolean;
  disconnect: () => void;
}

// DEF-008 Fix: Accept null options to support conditional calling from useProcess
export function useSSE(options: UseSSEOptions | null): UseSSEReturn {
  const [isConnected, setIsConnected] = useState(false);
  const [isReconnecting, setIsReconnecting] = useState(false);
  const [usePolling, setUsePolling] = useState(false);
  const eventSourceRef = useRef<EventSource | null>(null);
  const reconnectManager = useRef(new ReconnectionManager());
  // DEF-008 Fix: Add connectingRef guard to prevent duplicate connections
  const connectingRef = useRef(false);

  // DEF-006 Fix: Removed manual lastEventId tracking
  // Browser automatically handles Last-Event-ID for reconnection when server sends id: fields

  const connect = useCallback(() => {
    // DEF-008 Fix: Guard against null options
    if (!options) return;

    const { jobId, onProgress, onComplete } = options;

    // DEF-008 Fix: Prevent duplicate connection attempts
    if (connectingRef.current || eventSourceRef.current?.readyState === EventSource.OPEN) {
      return;
    }
    connectingRef.current = true;

    const url = new URL(`/api/v1/process/${jobId}/events`, window.location.origin);
    // DEF-006 Fix: Removed manual lastEventId query param
    // Browser's EventSource automatically sends Last-Event-ID header on reconnection

    const es = new EventSource(url.toString());

    es.onopen = () => {
      connectingRef.current = false;
      setIsConnected(true);
      setIsReconnecting(false);
      reconnectManager.current.reset();
    };

    // DEF-006 Fix: Removed e.lastEventId usage - EventSource API doesn't expose this on events
    // The browser handles Last-Event-ID automatically for reconnection
    es.addEventListener('progress', (e) => {
      onProgress(JSON.parse(e.data));
    });

    // DEF-006 Fix: sync events can include state after reconnection
    es.addEventListener('sync', (e) => {
      // Sync events provide current state after reconnection
      onProgress(JSON.parse(e.data));
    });

    es.addEventListener('complete', () => {
      options.onComplete();
      es.close();
    });

    es.onerror = () => {
      connectingRef.current = false;
      setIsConnected(false);
      es.close();

      const delay = reconnectManager.current.getNextDelay();
      if (delay !== null) {
        setIsReconnecting(true);
        setTimeout(connect, delay);
      } else {
        // Fall back to polling
        setUsePolling(true);
      }
    };

    eventSourceRef.current = es;
  // DEF-020 Fix: Use individual properties instead of entire options object
  // to prevent unnecessary reconnects when options object reference changes
  }, [options?.jobId, options?.onProgress, options?.onComplete, options?.onError]);

  useEffect(() => {
    // DEF-008 Fix: Only connect if options (and therefore jobId) is provided
    if (!options?.jobId) return;

    connect();

    // DEF-008 Fix: Proper cleanup to close connections
    return () => {
      connectingRef.current = false;
      if (eventSourceRef.current) {
        eventSourceRef.current.close();
        eventSourceRef.current = null;
      }
    };
  }, [options?.jobId, connect]);

  return {
    isConnected,
    isReconnecting,
    retryCount: reconnectManager.current.getRetryCount(),
    usePolling,
    disconnect: () => {
      connectingRef.current = false;
      eventSourceRef.current?.close();
      eventSourceRef.current = null;
    },
  };
}
```

---

## Task WIL-1.5b-015: useProcess Hook

### Description

Create the useProcess hook that orchestrates the entire processing flow, including job creation, progress tracking, and result handling.

### Acceptance Criteria

- [ ] Initiates processing with file/URL
- [ ] Manages SSE connection via useSSE
- [ ] Exposes processing state
- [ ] Handles cancellation
- [ ] Provides error recovery
- [ ] Integrates with progress interpolation

### Dependencies

- Task WIL-1.5b-014 completed
- Task WIL-1.5b-010 completed

### Effort Estimate

**Duration**: 20 minutes
**Complexity**: Medium

### Deliverables

1. `src/hooks/useProcess.ts`
2. Hook usage documentation

### Technical Notes

**DEF-002 Fix: AppError Class Definition**

```typescript
// src/types/errors.ts
/**
 * Application-specific error class with code and HTTP status.
 * Used throughout the application for consistent error handling.
 */
export class AppError extends Error {
  constructor(
    public code: string,
    public statusCode: number,
    message: string
  ) {
    super(message);
    this.name = 'AppError';
    // Ensure prototype chain is properly set for instanceof checks
    Object.setPrototypeOf(this, AppError.prototype);
  }

  toJSON() {
    return {
      name: this.name,
      code: this.code,
      statusCode: this.statusCode,
      message: this.message,
    };
  }
}
```

**DEF-003 Fix: ProcessResult Interface Definition**

```typescript
// src/types/process.ts
/**
 * Result returned when a document processing job completes.
 */
export interface ProcessResult {
  jobId: string;
  status: 'COMPLETED' | 'FAILED';
  markdown?: string;
  html?: string;
  json?: string;
  error?: string;
  completedAt: number;
}
```

**useProcess Hook Implementation (DEF-009 Fix: useSSE Integration)**

```typescript
// src/hooks/useProcess.ts
'use client';

import { useState, useCallback, useRef } from 'react';
import { useSSE } from './useSSE';
import { ProgressInterpolator } from '@/lib/progress/interpolation';
import { AppError } from '@/types/errors';
import { ProcessResult } from '@/types/process';
import { Progress } from '@/types/progress';

interface UseProcessOptions {
  onComplete: (result: ProcessResult) => void;
  onError: (error: AppError) => void;
}

interface UseProcessReturn {
  process: (input: { fileId?: string; url?: string }) => Promise<void>;
  cancel: () => void;
  isProcessing: boolean;
  isCancelling: boolean;
  progress: number;
  stage: string;
  error: AppError | null;
}

export function useProcess({ onComplete, onError }: UseProcessOptions): UseProcessReturn {
  const [isProcessing, setIsProcessing] = useState(false);
  const [isCancelling, setIsCancelling] = useState(false);
  const [progress, setProgress] = useState(0);
  const [stage, setStage] = useState('');
  const [error, setError] = useState<AppError | null>(null);
  const [jobId, setJobId] = useState<string | null>(null);

  const interpolator = useRef<ProgressInterpolator | null>(null);

  // DEF-009 Fix: Conditionally call useSSE only when jobId is set
  // Pass null when no jobId is available to satisfy hook rules
  const sse = useSSE(
    jobId
      ? {
          jobId,
          onProgress: (data: Progress) => {
            setStage(data.stage || '');
            if (interpolator.current) {
              interpolator.current.setActualProgress(data.progress);
            }
          },
          onComplete: () => {
            if (interpolator.current) {
              interpolator.current.complete();
            }
            setIsProcessing(false);
            onComplete({
              jobId: jobId!,
              status: 'COMPLETED',
              completedAt: Date.now(),
            });
          },
          onError: (err: Error) => {
            const appError = new AppError('SSE_ERROR', 500, err.message);
            setError(appError);
            onError(appError);
            setIsProcessing(false);
          },
        }
      : null
  );

  const process = useCallback(async (input: { fileId?: string; url?: string }) => {
    setIsProcessing(true);
    setError(null);
    setProgress(0);
    setStage('upload');

    try {
      // 1. Create job
      const response = await fetch('/api/v1/process', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(input),
      });

      if (!response.ok) {
        const errorData = await response.json().catch(() => ({}));
        throw new AppError(
          errorData.code || 'JOB_CREATE_ERROR',
          response.status,
          errorData.message || 'Failed to create job'
        );
      }

      const { jobId: newJobId } = await response.json();
      setJobId(newJobId);

      // 2. Start progress interpolation
      interpolator.current = new ProgressInterpolator(setProgress);
      interpolator.current.start();

      // Note: SSE connection is automatically initiated when jobId state updates
      // via the useSSE hook above (DEF-009 Fix)

    } catch (err) {
      const appError = err instanceof AppError
        ? err
        : new AppError('UNKNOWN_ERROR', 500, (err as Error).message);
      setError(appError);
      onError(appError);
      setIsProcessing(false);
    }
  }, [onError]);

  const cancel = useCallback(async () => {
    if (!jobId) return;

    setIsCancelling(true);
    try {
      await fetch(`/api/v1/jobs/${jobId}/cancel`, { method: 'POST' });
      // Disconnect SSE on cancel
      sse.disconnect();
    } finally {
      setIsCancelling(false);
      setIsProcessing(false);
      setJobId(null);
    }
  }, [jobId, sse]);

  return {
    process,
    cancel,
    isProcessing,
    isCancelling,
    progress,
    stage,
    error,
  };
}
```

---

## Task WIL-1.5b-016: Checkpoint Manager Unit Tests

### Description

Write comprehensive unit tests for the checkpoint manager, covering save, restore, and edge cases.

### Acceptance Criteria

- [ ] Test checkpoint save success
- [ ] Test checkpoint restore success
- [ ] Test checkpoint validation (schema)
- [ ] Test checkpoint size limit
- [ ] Test checkpoint clear
- [ ] Test resumeable jobs query

### Dependencies

- Task WIL-1.5b-009 completed
- Vitest configured

### Effort Estimate

**Duration**: 20 minutes
**Complexity**: Medium

### Deliverables

1. `src/lib/checkpoint/manager.test.ts`
2. Test coverage report

### Technical Notes

```typescript
// src/lib/checkpoint/manager.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { CheckpointManager } from './manager';
import { prisma } from '@/lib/db/prisma';

vi.mock('@/lib/db/prisma');

describe('CheckpointManager', () => {
  let manager: CheckpointManager;

  beforeEach(() => {
    manager = new CheckpointManager('test-job-id');
    vi.clearAllMocks();
  });

  it('should save checkpoint successfully', async () => {
    vi.mocked(prisma.job.update).mockResolvedValue({} as any);

    await expect(manager.save({
      stage: 'conversion',
      progress: 50,
      startedAt: Date.now(),
      lastUpdatedAt: Date.now(),
    })).resolves.not.toThrow();

    expect(prisma.job.update).toHaveBeenCalledWith({
      where: { id: 'test-job-id' },
      data: expect.objectContaining({
        checkpoint: expect.any(String),
      }),
    });
  });

  it('should restore checkpoint successfully', async () => {
    const checkpoint = {
      version: 1,
      stage: 'conversion',
      progress: 50,
      startedAt: Date.now(),
      lastUpdatedAt: Date.now(),
    };

    vi.mocked(prisma.job.findUnique).mockResolvedValue({
      checkpoint: JSON.stringify(checkpoint),
    } as any);

    const restored = await manager.restore();
    expect(restored).toEqual(checkpoint);
  });

  it('should reject checkpoint exceeding size limit', async () => {
    const largePartialResults = {
      markdown: 'x'.repeat(2 * 1024 * 1024), // 2MB
    };

    await expect(manager.save({
      stage: 'export',
      progress: 80,
      startedAt: Date.now(),
      lastUpdatedAt: Date.now(),
      partialResults: largePartialResults,
    })).rejects.toThrow('Checkpoint exceeds maximum size');
  });
});
```

---

## Task WIL-1.5b-017: Progress Interpolation Unit Tests

### Description

Write unit tests for progress interpolation, specifically testing the monotonic guarantee.

### Acceptance Criteria

- [ ] Test progress never decreases
- [ ] Test interpolation stops on stale updates
- [ ] Test immediate catch-up for large jumps
- [ ] Test completion sets to 100%
- [ ] Test max rate limiting

### Dependencies

- Task WIL-1.5b-010 completed

### Effort Estimate

**Duration**: 15 minutes
**Complexity**: Medium

### Deliverables

1. `src/lib/progress/interpolation.test.ts`
2. Test coverage report

### Technical Notes

```typescript
// src/lib/progress/interpolation.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { ProgressInterpolator } from './interpolation';

describe('ProgressInterpolator', () => {
  let interpolator: ProgressInterpolator;
  let updates: number[];

  beforeEach(() => {
    vi.useFakeTimers();
    updates = [];
    interpolator = new ProgressInterpolator((p) => updates.push(p));
  });

  afterEach(() => {
    interpolator.stop();
    vi.useRealTimers();
  });

  it('should never allow progress to decrease (monotonic)', () => {
    interpolator.setActualProgress(50);
    interpolator.setActualProgress(40); // Should be ignored
    interpolator.setActualProgress(30); // Should be ignored

    interpolator.start();
    vi.advanceTimersByTime(1000);

    // All updates should be >= previous
    for (let i = 1; i < updates.length; i++) {
      expect(updates[i]).toBeGreaterThanOrEqual(updates[i - 1]);
    }
  });

  it('should stop interpolating on stale updates', () => {
    interpolator.setActualProgress(50);
    interpolator.start();

    vi.advanceTimersByTime(2000);
    const countBefore = updates.length;

    // No updates for 5+ seconds
    vi.advanceTimersByTime(6000);
    const countAfter = updates.length;

    // Should have stopped interpolating
    expect(countAfter - countBefore).toBeLessThan(50);
  });

  it('should jump to 100% on complete', () => {
    interpolator.setActualProgress(50);
    interpolator.start();
    interpolator.complete();

    expect(interpolator.getDisplayedProgress()).toBe(100);
  });
});
```

---

## Sprint Summary

| Task ID | Title | Effort | Dependencies |
|---------|-------|--------|--------------|
| WIL-1.5b-001 | SSE Connection Manager | 30m | Sprint 1.5a |
| WIL-1.5b-002 | Exponential Backoff Reconnection Logic | 25m | WIL-1.5b-001 |
| WIL-1.5b-003 | Polling Fallback Mechanism | 30m | WIL-1.5b-002 |
| WIL-1.5b-004 | Redis Event Buffering for Last-Event-ID | 25m | WIL-1.5b-001 |
| WIL-1.5b-005 | State Synchronization on Reconnect | 20m | WIL-1.5b-004 |
| WIL-1.5b-006 | Process Route POST Handler | 30m | Sprint 1.5a, Sprint 1.2 |
| WIL-1.5b-007 | SSE Streaming Response Implementation | 30m | WIL-1.5b-001, WIL-1.5b-004, WIL-1.5b-006 |
| WIL-1.5b-008 | Last-Event-ID Reconnection Support | 20m | WIL-1.5b-004, WIL-1.5b-007 |
| WIL-1.5b-009 | Checkpoint Manager Implementation | 45m | Database schema |
| WIL-1.5b-010 | Progress Interpolation with Monotonic Guarantee | 30m | WIL-1.5b-007 |
| WIL-1.5b-011 | ProgressCard Component | 25m | WIL-1.5b-010 |
| WIL-1.5b-012 | Loading State Variants | 15m | WIL-1.5b-011 |
| WIL-1.5b-013 | Reconnection Overlay | 10m | WIL-1.5b-002 |
| WIL-1.5b-014 | useSSE Hook | 20m | WIL-1.5b-001 through WIL-1.5b-004 |
| WIL-1.5b-015 | useProcess Hook | 20m | WIL-1.5b-014, WIL-1.5b-010 |
| WIL-1.5b-016 | Checkpoint Manager Unit Tests | 20m | WIL-1.5b-009 |
| WIL-1.5b-017 | Progress Interpolation Unit Tests | 15m | WIL-1.5b-010 |

**Total Tasks**: 17
**Total Effort**: 4 hours 10 minutes

---

## Risk Considerations

| Risk | Mitigation |
|------|------------|
| SSE not supported by proxy | Ensure X-Accel-Buffering: no header set |
| Redis event buffer growth | TTL and size limits prevent unbounded growth |
| Checkpoint serialization complexity | Use Zod schema validation |
| Progress interpolation edge cases | Comprehensive unit tests for monotonic guarantee |
| Connection cleanup on abort | AbortController signal handling |

---

## Changelog

| Version | Date | Change | Author |
|---------|------|--------|--------|
| v1.0.0 | 2025-12-12 | Initial task file creation | William Chen |
| v1.1.0 | 2025-12-12 | Fixed DEF-001 through DEF-009: Type definitions, error handling, EventSource fixes | William Chen |
| v1.2.0 | 2025-12-12 | Fixed DEF-020: useSSE options dependency array to prevent unnecessary reconnects | William Chen |

---

**Version:** v1.2.0
**Last Updated:** 2025-12-12
