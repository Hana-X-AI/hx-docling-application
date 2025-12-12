# API Tasks: Sprint 1.5a - MCP Client (Core)

**Sprint**: 1.5a - MCP Client (Core)
**Author**: Bob Martin (@bob) - API Design SME
**Created**: 2025-12-12
**Status**: PENDING
**Dependencies**: Sprint 1.4 Complete

---

## Overview

This task file covers API-related deliverables for Sprint 1.5a, focusing on the SSE streaming endpoint implementation. While the primary MCP client work is led by James (@james), the API layer needs to expose SSE streaming for real-time progress updates.

**Note**: The majority of Sprint 1.5a is MCP-focused (led by James). API tasks here complement the MCP integration.

---

## Tasks

### BOB-1.5a-001: Implement SSE Events Endpoint

| Field | Value |
|-------|-------|
| **Task ID** | BOB-1.5a-001 |
| **Title** | Create GET /api/v1/process/{jobId}/events SSE endpoint |
| **Priority** | P0 (Critical) |
| **Effort** | 45 minutes |
| **Dependencies** | James's MCP client (Task 2-3), William's SSE manager (Sprint 1.5b) |

#### Description

Implement the Server-Sent Events (SSE) endpoint that streams real-time progress updates to clients during document processing. This endpoint must handle connection establishment, event streaming, and reconnection via Last-Event-ID header.

#### Acceptance Criteria

- [ ] Endpoint accessible at `GET /api/v1/process/{jobId}/events`
- [ ] Returns `Content-Type: text/event-stream`
- [ ] Returns `Cache-Control: no-cache`
- [ ] Returns `Connection: keep-alive`
- [ ] Streams `connected` event on connection
- [ ] Streams `progress` events during processing
- [ ] Streams `complete` event on success
- [ ] Streams `error` event on failure
- [ ] Streams `cancelled` event on cancellation
- [ ] Supports `Last-Event-ID` header for reconnection
- [ ] Streams `state_sync` event on reconnection
- [ ] Event IDs follow format: `{jobId8}-{timestamp}-{sequence}`

#### Technical Notes

```typescript
// File: src/app/api/v1/process/[jobId]/events/route.ts

import { NextRequest } from 'next/server';

export async function GET(
  request: NextRequest,
  { params }: { params: { jobId: string } }
) {
  const { jobId } = params;
  const lastEventId = request.headers.get('Last-Event-ID');

  // Validate job exists and user has access
  const job = await validateJobAccess(jobId, request);
  if (!job) {
    return new Response('Job not found', { status: 404 });
  }

  // Create SSE stream
  const stream = new ReadableStream({
    async start(controller) {
      const encoder = new TextEncoder();

      // Send connected event
      const connectionId = crypto.randomUUID();
      controller.enqueue(
        encoder.encode(`event: connected\ndata: ${JSON.stringify({ connectionId })}\n\n`)
      );

      // Handle reconnection - sync missed events
      if (lastEventId) {
        try {
          const syncState = await getStateSince(jobId, lastEventId);
          controller.enqueue(
            encoder.encode(`event: state_sync\ndata: ${JSON.stringify(syncState)}\n\n`)
          );
        } catch (err) {
          controller.error(err);
          return;
        }
      }

      // Subscribe to progress events
      const unsubscribe = subscribeToProgress(jobId, (event) => {
        const eventId = generateEventId(jobId, event.sequence);
        controller.enqueue(
          encoder.encode(
            `event: ${event.type}\nid: ${eventId}\ndata: ${JSON.stringify(event.data)}\n\n`
          )
        );

        // Close on terminal events
        if (['complete', 'error', 'cancelled'].includes(event.type)) {
          controller.close();
        }
      });

      // Cleanup on abort
      request.signal.addEventListener('abort', () => {
        unsubscribe();
        controller.close();
      });
    },
  });

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
      'X-Accel-Buffering': 'no', // Disable Nginx buffering
    },
  });
}
```

**Event ID Format**:
```typescript
function generateEventId(jobId: string, sequence: number): string {
  const timestamp = Math.floor(Date.now() / 1000);
  const jobPrefix = jobId.substring(0, 8);
  const seqStr = sequence.toString().padStart(4, '0');
  return `${jobPrefix}-${timestamp}-${seqStr}`;
}
```

#### Deliverables

- `src/app/api/v1/process/[jobId]/events/route.ts` - SSE endpoint
- `src/lib/sse/event-format.ts` - Event formatting utilities

---

### BOB-1.5a-002: Define SSE Event Schemas

| Field | Value |
|-------|-------|
| **Task ID** | BOB-1.5a-002 |
| **Title** | Define TypeScript schemas for all SSE event types |
| **Priority** | P1 (Critical) |
| **Effort** | 20 minutes |
| **Dependencies** | None |

#### Description

Define strongly-typed schemas for all SSE event types to ensure consistent event structure between server and client.

#### Acceptance Criteria

- [ ] Schema for `connected` event
- [ ] Schema for `progress` event with stage, percent, message
- [ ] Schema for `state_sync` event for reconnection
- [ ] Schema for `retry` event with attempt details
- [ ] Schema for `complete` event with formats array
- [ ] Schema for `cancelled` event with preservation flag
- [ ] Schema for `error` event with error details
- [ ] All schemas validated with Zod

#### Technical Notes

```typescript
// File: src/lib/sse/schemas.ts

import { z } from 'zod';

export const connectedEventSchema = z.object({
  connectionId: z.string().uuid(),
});

export const progressEventSchema = z.object({
  stage: z.enum(['upload', 'parsing', 'conversion', 'export', 'saving', 'complete']),
  percent: z.number().min(0).max(100),
  message: z.string(),
});

export const stateSyncEventSchema = z.object({
  stage: z.string(),
  percent: z.number(),
  message: z.string(),
  jobStatus: z.enum([
    'PENDING', 'UPLOADING', 'PROCESSING', 'RETRY_1', 'RETRY_2', 'RETRY_3',
    'COMPLETE', 'PARTIAL_COMPLETE', 'CANCELLED', 'ERROR'
  ]),
});

export const retryEventSchema = z.object({
  attempt: z.number().int().min(1).max(3),
  maxAttempts: z.literal(3),
  nextRetryMs: z.number(),
  reason: z.string(),
  stage: z.string(),
});

export const completeEventSchema = z.object({
  jobId: z.string().uuid(),
  status: z.literal('COMPLETE'),
  formats: z.array(z.enum(['markdown', 'html', 'json'])),
});

export const cancelledEventSchema = z.object({
  jobId: z.string().uuid(),
  cancelledAt: z.string().datetime(),
  partialResultsPreserved: z.boolean(),
});

export const errorEventSchema = z.object({
  code: z.string(),
  message: z.string(),
  retryable: z.boolean(),
});

// Union type for all events
export type SSEEvent =
  | { type: 'connected'; data: z.infer<typeof connectedEventSchema> }
  | { type: 'progress'; data: z.infer<typeof progressEventSchema> }
  | { type: 'state_sync'; data: z.infer<typeof stateSyncEventSchema> }
  | { type: 'retry'; data: z.infer<typeof retryEventSchema> }
  | { type: 'complete'; data: z.infer<typeof completeEventSchema> }
  | { type: 'cancelled'; data: z.infer<typeof cancelledEventSchema> }
  | { type: 'error'; data: z.infer<typeof errorEventSchema> };
```

#### Deliverables

- `src/lib/sse/schemas.ts` - SSE event schemas
- `src/lib/sse/types.ts` - TypeScript type exports

---

### BOB-1.5a-003: Implement Job Access Validation

| Field | Value |
|-------|-------|
| **Task ID** | BOB-1.5a-003 |
| **Title** | Create session-based job access validation |
| **Priority** | P1 (Critical) |
| **Effort** | 15 minutes |
| **Dependencies** | Sri's session management (Sprint 1.2) |

#### Description

Implement validation logic to ensure users can only access SSE streams for their own jobs, based on session ID.

#### Acceptance Criteria

- [ ] Validates job exists in database
- [ ] Validates session ID from cookie matches job's sessionId
- [ ] Returns null for unauthorized/missing jobs
- [ ] Returns job for authorized access

#### Technical Notes

```typescript
// File: src/lib/jobs/access.ts

export async function validateJobAccess(
  jobId: string,
  request: NextRequest
): Promise<Job | null> {
  const sessionId = getSessionId(request);
  if (!sessionId) {
    return null;
  }

  const job = await prisma.job.findUnique({
    where: { id: jobId },
  });

  if (!job || job.sessionId !== sessionId) {
    return null;
  }

  return job;
}
```

#### Deliverables

- `src/lib/jobs/access.ts` - Job access validation

---

## Sprint Summary

| Task ID | Title | Effort | Priority |
|---------|-------|--------|----------|
| BOB-1.5a-001 | SSE Events Endpoint | 45m | P0 |
| BOB-1.5a-002 | SSE Event Schemas | 20m | P1 |
| BOB-1.5a-003 | Job Access Validation | 15m | P1 |

**Total Effort**: 1 hour 20 minutes

---

## Dependencies Graph

```
Sprint 1.4 Complete
        |
        +-- James's MCP Client (Tasks 2-3)
        |
        +-- Sri's Session Mgmt (Sprint 1.2)
        |
        v
BOB-1.5a-002 (Event Schemas)
        |
        v
BOB-1.5a-003 (Access Validation)
        |
        v
BOB-1.5a-001 (SSE Endpoint)
        |
        v
William's SSE Manager (Sprint 1.5b)
```

---

## Coordination Notes

- **James (@james)**: SSE endpoint will subscribe to MCP progress events
- **William (@william)**: SSE manager in Sprint 1.5b will handle reconnection logic
- **Neo (@neo)**: Client-side useSSE hook will consume these events
- **Sophia (@sophia)**: Event schemas align with LangGraph state machine patterns

---

## SSE Protocol Notes

### Content-Type

The endpoint MUST return `text/event-stream` content type per the SSE specification.

### Event Format

Each event follows the SSE format:
```
event: <event-type>
id: <event-id>
data: <json-payload>

```

Note: Two newlines terminate each event.

### Connection Handling

- Browser EventSource handles reconnection automatically
- Server should not buffer responses (Nginx: X-Accel-Buffering: no)
- Keep-alive is managed by the HTTP layer

### Error Handling

If the job is not found or unauthorized, return a standard HTTP 404 response (not SSE).
Terminal errors during streaming should emit an `error` event before closing.
