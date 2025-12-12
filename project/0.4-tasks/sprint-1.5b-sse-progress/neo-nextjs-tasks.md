# Neo Next.js Tasks: Sprint 1.5b - SSE & Progress Integration (Support Role)

**Sprint**: 1.5b - SSE & Progress Integration
**Duration**: ~4.0 hours (Sprint Total)
**Role**: Support Developer
**Agent**: Neo (Next.js Senior Developer)
**Lead**: William (@william)
**Support**: Neo (@neo), Ola (@ola), Sophia (@sophia)
**Review**: Alex (@alex)

---

## Development Tools

### shadcn/ui MCP Server Reference

For component discovery and retrieval during progress UI development, use the **hx-shadcn MCP Server**:

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
- `get_component_demo` - Get usage examples for Progress, Card, Skeleton components
- `get_component_metadata` - Get props, installation instructions, and documentation

**Example - Get Progress Component Demo:**
```json
{
  "method": "tools/call",
  "params": {
    "name": "get_component_demo",
    "arguments": {"componentName": "progress"}
  }
}
```

**Sprint 1.5b Component Usage:** This sprint uses Progress, Card, Button, and Skeleton components for the processing status interface. Use `get_component_demo` to retrieve usage patterns for progress indicators.

---

## Overview

Sprint 1.5b focuses on SSE streaming and progress integration. Neo provides support for React hooks and UI components while William leads the SSE infrastructure implementation.

---

## Neo's Support Tasks

### NEO-1.5b-001: Create useSSE React Hook

**Priority**: P0 (Critical)
**Effort**: 35 minutes
**Dependencies**: Sprint 1.5a Complete
**Lead**: William, Support: Neo

**Description**:
Create a React hook that encapsulates SSE (Server-Sent Events) connection management, including connection establishment, event handling, reconnection logic, and cleanup.

**Acceptance Criteria**:
- [ ] Hook connects to SSE endpoint on mount
- [ ] Handles all event types (progress, complete, error, retry, cancelled)
- [ ] Implements reconnection with exponential backoff
- [ ] Tracks Last-Event-ID for reconnection
- [ ] Provides connection status (connected, connecting, disconnected, error)
- [ ] Cleanup on unmount (closes EventSource)
- [ ] TypeScript types for all events

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/hooks/useSSE.ts`

**Technical Notes**:
```typescript
'use client';

import { useEffect, useRef, useState, useCallback } from 'react';

export type SSEConnectionStatus =
  | 'idle'
  | 'connecting'
  | 'connected'
  | 'reconnecting'
  | 'disconnected'
  | 'error';

export interface SSEProgress {
  stage: string;
  percent: number;
  message: string;
}

export interface SSERetryEvent {
  attempt: number;
  maxAttempts: number;
  nextRetryMs: number;
  reason: string;
}

export interface SSECompleteEvent {
  jobId: string;
  status: string;
  formats: string[];
}

export interface SSEErrorEvent {
  code: string;
  message: string;
  retryable: boolean;
}

export interface UseSSEOptions {
  maxRetries?: number;
  baseDelay?: number;
  maxDelay?: number;
  onProgress?: (progress: SSEProgress) => void;
  onComplete?: (data: SSECompleteEvent) => void;
  onError?: (error: SSEErrorEvent) => void;
  onRetry?: (retry: SSERetryEvent) => void;
  onCancelled?: () => void;
}

const DEFAULT_OPTIONS = {
  maxRetries: 10,
  baseDelay: 1000,
  maxDelay: 30000,
};

// Maximum number of event IDs to retain for deduplication (prevents memory leak)
const MAX_DEDUP_SIZE = 1000;

/**
 * Trims a Set to keep only the most recent entries (FIFO).
 * Removes oldest entries when size exceeds maxSize.
 */
function trimSet<T>(set: Set<T>, maxSize: number): void {
  if (set.size <= maxSize) return;
  const iterator = set.values();
  const toRemove = set.size - maxSize;
  for (let i = 0; i < toRemove; i++) {
    const { value } = iterator.next();
    set.delete(value);
  }
}

export function useSSE(jobId: string | null, options: UseSSEOptions = {}) {
  const {
    maxRetries = DEFAULT_OPTIONS.maxRetries,
    baseDelay = DEFAULT_OPTIONS.baseDelay,
    maxDelay = DEFAULT_OPTIONS.maxDelay,
    onProgress,
    onComplete,
    onError,
    onRetry,
    onCancelled,
  } = options;

  const [status, setStatus] = useState<SSEConnectionStatus>('idle');
  const [progress, setProgress] = useState<SSEProgress | null>(null);
  const [error, setError] = useState<SSEErrorEvent | null>(null);

  const eventSourceRef = useRef<EventSource | null>(null);
  const retryCountRef = useRef(0);
  const lastEventIdRef = useRef<string | null>(null);
  const processedEventsRef = useRef(new Set<string>());

  const connect = useCallback(() => {
    if (!jobId) return;

    setStatus('connecting');
    setError(null);

    const url = new URL(`/api/v1/process/${jobId}/events`, window.location.origin);

    // Include Last-Event-ID if reconnecting
    if (lastEventIdRef.current) {
      url.searchParams.set('lastEventId', lastEventIdRef.current);
      // IMPORTANT: Deduplication set is intentionally NOT cleared on reconnect.
      // This assumes the server preserves event IDs across reconnects when replaying
      // events from lastEventId. If the server reassigns new IDs to replayed events,
      // deduplication will fail and events may be processed twice. In that case,
      // consider clearing processedEventsRef.current here or implementing
      // content-based deduplication instead of ID-based.
    } else {
      // Fresh connection (not a reconnect) - clear deduplication set
      processedEventsRef.current.clear();
    }

    const eventSource = new EventSource(url.toString());
    eventSourceRef.current = eventSource;

    eventSource.onopen = () => {
      setStatus('connected');
      retryCountRef.current = 0;
    };

    eventSource.addEventListener('connected', (event) => {
      const data = JSON.parse(event.data);
      console.log('SSE connected:', data.connectionId);
    });

    eventSource.addEventListener('progress', (event) => {
      // Deduplicate events using event ID.
      // ASSUMPTION: Server preserves event IDs across reconnects. If the server
      // reassigns IDs to replayed events (e.g., evt-1 becomes evt-100), this
      // deduplication will silently fail and the event will be dropped as a
      // false-positive duplicate. See the reconnect handling above for details.
      if (processedEventsRef.current.has(event.lastEventId)) {
        return;
      }
      processedEventsRef.current.add(event.lastEventId);
      // Trim set to prevent unbounded growth (FIFO - removes oldest entries)
      trimSet(processedEventsRef.current, MAX_DEDUP_SIZE);
      lastEventIdRef.current = event.lastEventId;

      const data: SSEProgress = JSON.parse(event.data);
      setProgress(data);
      onProgress?.(data);
    });

    eventSource.addEventListener('state_sync', (event) => {
      lastEventIdRef.current = event.lastEventId;
      const data = JSON.parse(event.data);
      setProgress({
        stage: data.stage,
        percent: data.percent,
        message: data.message,
      });
    });

    eventSource.addEventListener('retry', (event) => {
      lastEventIdRef.current = event.lastEventId;
      const data: SSERetryEvent = JSON.parse(event.data);
      onRetry?.(data);
    });

    eventSource.addEventListener('complete', (event) => {
      lastEventIdRef.current = event.lastEventId;
      const data: SSECompleteEvent = JSON.parse(event.data);
      setStatus('disconnected');
      eventSource.close();
      onComplete?.(data);
    });

    eventSource.addEventListener('cancelled', (event) => {
      lastEventIdRef.current = event.lastEventId;
      setStatus('disconnected');
      eventSource.close();
      onCancelled?.();
    });

    eventSource.addEventListener('error', (event) => {
      if (event instanceof MessageEvent) {
        lastEventIdRef.current = event.lastEventId;
        const data: SSEErrorEvent = JSON.parse(event.data);
        setError(data);
        onError?.(data);
      }
    });

    eventSource.onerror = () => {
      eventSource.close();

      if (retryCountRef.current < maxRetries) {
        setStatus('reconnecting');
        const delay = Math.min(
          baseDelay * Math.pow(2, retryCountRef.current),
          maxDelay
        );
        retryCountRef.current++;

        setTimeout(() => {
          connect();
        }, delay);
      } else {
        setStatus('error');
        setError({
          code: 'E503',
          message: 'Connection failed after maximum retries',
          retryable: false,
        });
      }
    };
  }, [jobId, maxRetries, baseDelay, maxDelay, onProgress, onComplete, onError, onRetry, onCancelled]);

  const disconnect = useCallback(() => {
    if (eventSourceRef.current) {
      eventSourceRef.current.close();
      eventSourceRef.current = null;
    }
    // Clear deduplication set on disconnect to prevent memory leak
    processedEventsRef.current.clear();
    setStatus('disconnected');
  }, []);

  useEffect(() => {
    if (jobId) {
      connect();
    }

    return () => {
      disconnect();
    };
  }, [jobId, connect, disconnect]);

  return {
    status,
    progress,
    error,
    reconnect: connect,
    disconnect,
  };
}
```

---

### NEO-1.5b-002: Create useProcess React Hook

**Priority**: P0 (Critical)
**Effort**: 30 minutes
**Dependencies**: NEO-1.5b-001
**Lead**: William, Support: Neo

**Description**:
Create a React hook that orchestrates the complete document processing flow, combining file/URL upload initiation with SSE progress tracking and cancellation support.

**Acceptance Criteria**:
- [ ] Initiates processing via API call
- [ ] Integrates with useSSE for progress tracking
- [ ] Provides cancel functionality with AbortController
- [ ] Tracks processing state (idle, uploading, processing, complete, error, cancelled)
- [ ] Returns final results on completion
- [ ] Handles partial results (PARTIAL_COMPLETE status)

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/hooks/useProcess.ts`

**Technical Notes**:
```typescript
'use client';

import { useState, useCallback, useRef } from 'react';
import { useSSE, SSEProgress, SSECompleteEvent, SSEErrorEvent } from './useSSE';
import { useDocumentStore } from '@/stores/documentStore';

export type ProcessingStatus =
  | 'idle'
  | 'uploading'
  | 'processing'
  | 'complete'
  | 'partial_complete'
  | 'error'
  | 'cancelled'
  | 'cancelling';

export interface ProcessResult {
  jobId: string;
  status: string;
  formats: string[];
}

export interface UseProcessOptions {
  onProgress?: (progress: SSEProgress) => void;
  onComplete?: (result: ProcessResult) => void;
  onError?: (error: SSEErrorEvent) => void;
}

export function useProcess(options: UseProcessOptions = {}) {
  const { onProgress, onComplete, onError } = options;

  const [status, setStatus] = useState<ProcessingStatus>('idle');
  const [result, setResult] = useState<ProcessResult | null>(null);
  const [error, setError] = useState<SSEErrorEvent | null>(null);
  const [progress, setProgress] = useState<SSEProgress | null>(null);

  const abortControllerRef = useRef<AbortController | null>(null);
  const { currentJobId, setProcessing, clearInputs } = useDocumentStore();

  // SSE connection for progress
  const { status: sseStatus, reconnect } = useSSE(
    status === 'processing' ? currentJobId : null,
    {
      onProgress: (p) => {
        setProgress(p);
        onProgress?.(p);
      },
      onComplete: (data) => {
        const isPartial = data.status === 'PARTIAL_COMPLETE';
        setStatus(isPartial ? 'partial_complete' : 'complete');
        setResult(data);
        setProcessing(false);
        onComplete?.(data);
      },
      onError: (err) => {
        setStatus('error');
        setError(err);
        setProcessing(false);
        onError?.(err);
      },
      onCancelled: () => {
        setStatus('cancelled');
        setProcessing(false);
      },
    }
  );

  const process = useCallback(async (jobId: string) => {
    setStatus('processing');
    setError(null);
    setProcessing(true);

    abortControllerRef.current = new AbortController();

    try {
      const response = await fetch('/api/v1/process', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ jobId }),
        signal: abortControllerRef.current.signal,
      });

      if (!response.ok) {
        const errorData = await response.json();
        throw new Error(errorData.error?.message || 'Processing failed');
      }

      // SSE will handle progress from here
    } catch (err) {
      if (err instanceof Error && err.name === 'AbortError') {
        setStatus('cancelled');
      } else {
        setStatus('error');
        setError({
          code: 'E301',
          message: err instanceof Error ? err.message : 'Processing failed',
          retryable: true,
        });
        onError?.({
          code: 'E301',
          message: err instanceof Error ? err.message : 'Processing failed',
          retryable: true,
        });
      }
      setProcessing(false);
    }
  }, [setProcessing, onError]);

  const cancel = useCallback(async () => {
    if (!currentJobId || status !== 'processing') return;

    setStatus('cancelling');

    // Abort the current request
    abortControllerRef.current?.abort();

    try {
      const response = await fetch(`/api/v1/jobs/${currentJobId}/cancel`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ preservePartialResults: false }),
      });

      if (response.ok) {
        setStatus('cancelled');
      }
    } catch (err) {
      console.error('Failed to cancel job:', err);
    }

    setProcessing(false);
  }, [currentJobId, status, setProcessing]);

  const reset = useCallback(() => {
    setStatus('idle');
    setResult(null);
    setError(null);
    setProgress(null);
    clearInputs();
  }, [clearInputs]);

  return {
    status,
    result,
    error,
    progress,
    sseStatus,
    process,
    cancel,
    reset,
    reconnect,
    isProcessing: status === 'processing' || status === 'uploading',
    isCancelling: status === 'cancelling',
  };
}
```

---

### NEO-1.5b-003: Create ProgressCard Component

**Priority**: P1 (High)
**Effort**: 25 minutes
**Dependencies**: NEO-1.5b-001

**Description**:
Create a progress card component that displays real-time processing progress with stage information, percentage, message, and visual progress bar.

**Acceptance Criteria**:
- [ ] Displays current stage name
- [ ] Shows progress percentage with smooth animation
- [ ] Displays progress message
- [ ] Progress bar never decreases (monotonic)
- [ ] Cancel button available during processing
- [ ] Reconnecting indicator when SSE reconnecting
- [ ] Accessible with proper ARIA attributes

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/components/processing/ProgressCard.tsx`

**Technical Notes**:
```typescript
'use client';

import { useEffect, useState } from 'react';
import { Loader2, X, RefreshCw } from 'lucide-react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Progress } from '@/components/ui/progress';
import { cn } from '@/lib/utils';
import type { SSEProgress, SSEConnectionStatus } from '@/hooks/useSSE';

interface ProgressCardProps {
  progress: SSEProgress | null;
  connectionStatus: SSEConnectionStatus;
  onCancel?: () => void;
  isCancelling?: boolean;
}

const STAGE_LABELS: Record<string, string> = {
  upload: 'Uploading',
  parsing: 'Analyzing',
  conversion: 'Converting',
  export: 'Exporting',
  saving: 'Saving',
  complete: 'Complete',
};

export function ProgressCard({
  progress,
  connectionStatus,
  onCancel,
  isCancelling,
}: ProgressCardProps) {
  // Ensure progress never decreases (monotonic guarantee)
  const [displayPercent, setDisplayPercent] = useState(0);

  useEffect(() => {
    if (progress && progress.percent > displayPercent) {
      setDisplayPercent(progress.percent);
    }
  }, [progress, displayPercent]);

  const stageLabel = progress?.stage
    ? STAGE_LABELS[progress.stage] || progress.stage
    : 'Initializing';

  const isReconnecting = connectionStatus === 'reconnecting';

  return (
    <Card className="w-full">
      <CardHeader className="flex flex-row items-center justify-between pb-2">
        <CardTitle className="text-lg font-medium flex items-center gap-2">
          {connectionStatus === 'connected' ? (
            <Loader2 className="w-4 h-4 animate-spin" />
          ) : isReconnecting ? (
            <RefreshCw className="w-4 h-4 animate-spin" />
          ) : null}
          <span>{isReconnecting ? 'Reconnecting...' : stageLabel}</span>
        </CardTitle>
        {onCancel && (
          <Button
            variant="ghost"
            size="sm"
            onClick={onCancel}
            disabled={isCancelling}
            aria-label="Cancel processing"
          >
            {isCancelling ? (
              <Loader2 className="w-4 h-4 animate-spin" />
            ) : (
              <X className="w-4 h-4" />
            )}
            <span className="ml-2">Cancel</span>
          </Button>
        )}
      </CardHeader>
      <CardContent className="space-y-4">
        <div className="space-y-2">
          <div className="flex justify-between text-sm">
            <span className="text-muted-foreground">
              {progress?.message || 'Starting...'}
            </span>
            <span className="font-medium">{displayPercent}%</span>
          </div>
          <Progress
            value={displayPercent}
            className={cn('h-2', {
              'animate-pulse': isReconnecting,
            })}
            aria-label={`Processing progress: ${displayPercent}%`}
          />
        </div>

        {isReconnecting && (
          <p className="text-sm text-yellow-600 dark:text-yellow-400">
            Connection interrupted. Attempting to reconnect...
          </p>
        )}
      </CardContent>
    </Card>
  );
}
```

---

### NEO-1.5b-004: Create Loading State Components

**Priority**: P1 (High)
**Effort**: 15 minutes
**Dependencies**: None

**Description**:
Create reusable loading state components including skeleton loaders, spinners, and stage indicators for use throughout the application.

**Acceptance Criteria**:
- [ ] Skeleton loader for content areas
- [ ] Spinner component with size variants
- [ ] Stage indicator showing processing stages
- [ ] Accessible loading states with aria-live regions

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/components/processing/LoadingStates.tsx`

**Technical Notes**:
```typescript
'use client';

import { Loader2, Check } from 'lucide-react';
import { Skeleton } from '@/components/ui/skeleton';
import { cn } from '@/lib/utils';

interface SpinnerProps {
  size?: 'sm' | 'md' | 'lg';
  className?: string;
}

export function Spinner({ size = 'md', className }: SpinnerProps) {
  const sizeClasses = {
    sm: 'w-4 h-4',
    md: 'w-6 h-6',
    lg: 'w-8 h-8',
  };

  return (
    <Loader2
      className={cn('animate-spin', sizeClasses[size], className)}
      aria-hidden="true"
    />
  );
}

interface ContentSkeletonProps {
  lines?: number;
  className?: string;
}

export function ContentSkeleton({ lines = 3, className }: ContentSkeletonProps) {
  return (
    <div className={cn('space-y-3', className)} aria-busy="true" aria-label="Loading content">
      {Array.from({ length: lines }).map((_, i) => (
        <Skeleton
          key={i}
          className={cn('h-4', {
            'w-full': i < lines - 1,
            'w-2/3': i === lines - 1,
          })}
        />
      ))}
    </div>
  );
}

interface ProcessingStage {
  id: string;
  label: string;
  complete: boolean;
  active: boolean;
}

interface StageIndicatorProps {
  stages: ProcessingStage[];
}

export function StageIndicator({ stages }: StageIndicatorProps) {
  return (
    <div className="space-y-2" role="list" aria-label="Processing stages">
      {stages.map((stage) => (
        <div
          key={stage.id}
          className={cn('flex items-center gap-2 text-sm', {
            'text-primary': stage.active,
            'text-muted-foreground': !stage.active && !stage.complete,
            'text-green-600 dark:text-green-400': stage.complete,
          })}
          role="listitem"
        >
          {stage.complete ? (
            <Check className="w-4 h-4" />
          ) : stage.active ? (
            <Spinner size="sm" />
          ) : (
            <div className="w-4 h-4 rounded-full border-2 border-current" />
          )}
          <span>{stage.label}</span>
        </div>
      ))}
    </div>
  );
}

interface LoadingOverlayProps {
  message?: string;
}

export function LoadingOverlay({ message = 'Loading...' }: LoadingOverlayProps) {
  return (
    <div
      className="absolute inset-0 flex items-center justify-center bg-background/80 backdrop-blur-sm z-50"
      role="alert"
      aria-live="polite"
    >
      <div className="flex flex-col items-center gap-4">
        <Spinner size="lg" />
        <p className="text-sm text-muted-foreground">{message}</p>
      </div>
    </div>
  );
}
```

---

### NEO-1.5b-005: Create Reconnection Overlay Component

**Priority**: P1 (High)
**Effort**: 10 minutes
**Dependencies**: NEO-1.5b-004

**Description**:
Create a reconnection overlay that displays when SSE connection is interrupted, showing reconnection attempts and status.

**Acceptance Criteria**:
- [ ] Shows "Reconnecting..." message
- [ ] Displays retry attempt count
- [ ] Allows manual reconnection trigger
- [ ] Non-blocking (user can still see progress)
- [ ] Accessible with aria-live

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/components/processing/ReconnectionOverlay.tsx`

**Technical Notes**:
```typescript
'use client';

import { RefreshCw } from 'lucide-react';
import { Button } from '@/components/ui/button';
import type { SSEConnectionStatus } from '@/hooks/useSSE';

interface ReconnectionOverlayProps {
  status: SSEConnectionStatus;
  onReconnect?: () => void;
}

export function ReconnectionOverlay({
  status,
  onReconnect,
}: ReconnectionOverlayProps) {
  if (status !== 'reconnecting' && status !== 'error') {
    return null;
  }

  return (
    <div
      className="fixed bottom-4 right-4 bg-yellow-100 dark:bg-yellow-900 border border-yellow-300 dark:border-yellow-700 rounded-lg p-4 shadow-lg max-w-sm"
      role="alert"
      aria-live="assertive"
    >
      <div className="flex items-center gap-3">
        {status === 'reconnecting' ? (
          <>
            <RefreshCw className="w-5 h-5 animate-spin text-yellow-600 dark:text-yellow-400" />
            <div>
              <p className="font-medium text-yellow-800 dark:text-yellow-200">
                Reconnecting...
              </p>
              <p className="text-sm text-yellow-600 dark:text-yellow-400">
                Connection interrupted. Attempting to restore.
              </p>
            </div>
          </>
        ) : (
          <>
            <div className="w-5 h-5 rounded-full bg-red-500" />
            <div className="flex-1">
              <p className="font-medium text-red-800 dark:text-red-200">
                Connection Failed
              </p>
              <p className="text-sm text-red-600 dark:text-red-400">
                Unable to reconnect after multiple attempts.
              </p>
            </div>
            {onReconnect && (
              <Button size="sm" variant="outline" onClick={onReconnect}>
                Retry
              </Button>
            )}
          </>
        )}
      </div>
    </div>
  );
}
```

---

### NEO-1.5b-006: Write Unit Tests for Hooks

**Priority**: P1 (High)
**Effort**: 25 minutes
**Dependencies**: NEO-1.5b-001, NEO-1.5b-002

**Description**:
Write unit tests for useSSE and useProcess hooks, testing connection management, event handling, and state transitions.

**Acceptance Criteria**:
- [ ] Test SSE connection establishment
- [ ] Test event handling (progress, complete, error)
- [ ] Test reconnection logic
- [ ] Test process initiation
- [ ] Test cancellation flow
- [ ] Tests achieve 80%+ coverage

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/hooks/useSSE.test.ts`
- `/home/agent0/hx-docling-ui/src/hooks/useProcess.test.ts`

**Technical Notes**:
```typescript
// useSSE.test.ts
import { renderHook, act, waitFor } from '@testing-library/react';
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { useSSE } from './useSSE';

// Mock EventSource
class MockEventSource {
  url: string;
  onopen: (() => void) | null = null;
  onerror: ((event: Event) => void) | null = null;
  listeners: Map<string, ((event: MessageEvent) => void)[]> = new Map();

  constructor(url: string) {
    this.url = url;
  }

  addEventListener(type: string, listener: (event: MessageEvent) => void) {
    if (!this.listeners.has(type)) {
      this.listeners.set(type, []);
    }
    this.listeners.get(type)!.push(listener);
  }

  close() {}

  // Test helper to simulate events
  emit(type: string, data: unknown, lastEventId = 'evt-1') {
    const listeners = this.listeners.get(type) || [];
    const event = new MessageEvent(type, {
      data: JSON.stringify(data),
      lastEventId,
    });
    listeners.forEach((listener) => listener(event));
  }
}

describe('useSSE', () => {
  beforeEach(() => {
    // @ts-ignore
    global.EventSource = MockEventSource;
  });

  afterEach(() => {
    vi.clearAllMocks();
  });

  it('connects when jobId is provided', () => {
    const { result } = renderHook(() => useSSE('job-123'));

    expect(result.current.status).toBe('connecting');
  });

  it('handles progress events', async () => {
    const onProgress = vi.fn();
    const { result } = renderHook(() =>
      useSSE('job-123', { onProgress })
    );

    // Simulate connection open
    // ... (would need to access the mock EventSource instance)

    expect(result.current.status).toBeDefined();
  });

  it('does not connect when jobId is null', () => {
    const { result } = renderHook(() => useSSE(null));

    expect(result.current.status).toBe('idle');
  });
});
```

---

## Summary

| Task ID | Task Title | Effort | Priority |
|---------|-----------|--------|----------|
| NEO-1.5b-001 | Create useSSE React Hook | 35m | P0 |
| NEO-1.5b-002 | Create useProcess React Hook | 30m | P0 |
| NEO-1.5b-003 | Create ProgressCard Component | 25m | P1 |
| NEO-1.5b-004 | Create Loading State Components | 15m | P1 |
| NEO-1.5b-005 | Create Reconnection Overlay | 10m | P1 |
| NEO-1.5b-006 | Write Unit Tests for Hooks | 25m | P1 |

**Total Neo Effort**: ~2.3 hours (140 minutes)
**Total Tasks**: 6

---

## Dependencies Graph

```
Sprint 1.5a Complete
    |
    +-> WILLIAM: SSE Manager Implementation (Lead)
    |       |
    |       +-> NEO-1.5b-001 (useSSE Hook) [Support]
    |               |
    |               +-> NEO-1.5b-002 (useProcess Hook) [Support]
    |               |       |
    |               |       +-> NEO-1.5b-006 (Hook Tests)
    |               |
    |               +-> NEO-1.5b-003 (ProgressCard) [Support]
    |                       |
    |                       +-> NEO-1.5b-005 (Reconnection Overlay)
    |
    +-> NEO-1.5b-004 (Loading States) [Independent]
```

---

## Coordination Notes

- **Primary Lead**: William (@william) owns SSE infrastructure
- **Neo's Role**: Provide React hooks and UI components for SSE integration
- **Ola's Role**: Support with UI/UX accessibility
- **Sophia's Role**: Support with progress interpolation patterns
- **Handoff**: Hooks depend on SSE manager API from William
- **Review**: Alex reviews for consistency with specification
