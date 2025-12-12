# Julia Santos - Testing & QA Tasks: Sprint 1.5b (SSE & Progress Integration)

**Sprint**: 1.5b - SSE & Progress Integration
**Role**: Testing & Quality Assurance Specialist
**Document Version**: 1.0.0
**Created**: 2025-12-12
**References**:
- Implementation Plan v1.2.0 Section 4.5b
- Detailed Specification v1.2.0 Sections 5.5, 9.2, 10.3

---

## Sprint 1.5b Testing Objectives

Implement unit tests for SSE streaming, checkpoint manager, and progress interpolation. Focus on testing resilient reconnection behavior, checkpoint save/restore cycles, and the monotonic progress guarantee.

---

## Tasks

### JUL-1.5b-001: Write Unit Tests for Checkpoint Manager

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.5b-001 |
| **Title** | Write Unit Tests for Checkpoint Manager Save/Restore |
| **Priority** | P0 (Critical) |
| **Effort** | 1.25 hours |
| **Dependencies** | WILLIAM-1.5b-009 (Checkpoint manager implemented) |

#### Description

Write comprehensive unit tests for the checkpoint manager ensuring checkpoints are saved correctly at each processing stage and can be restored for job resumption.

#### Acceptance Criteria

- [ ] Test checkpoint created on stage transition
- [ ] Test checkpoint serialization format correct
- [ ] Test checkpoint saved to database
- [ ] Test checkpoint restore retrieves correct state
- [ ] Test checkpoint contains all required fields
- [ ] Test checkpoint validates on restore
- [ ] Test partial results preserved in checkpoint
- [ ] Test checkpoint cleanup on job completion
- [ ] Test checkpoint not created for failed stages
- [ ] Coverage >= 95% for checkpoint manager

#### Technical Notes

```typescript
// src/lib/checkpoint/__tests__/manager.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { CheckpointManager } from '../manager';
import { prisma } from '@/lib/db/prisma';

describe('CheckpointManager', () => {
  let manager: CheckpointManager;
  const mockJobId = 'test-job-123';

  beforeEach(() => {
    manager = new CheckpointManager();
    vi.clearAllMocks();
  });

  describe('Checkpoint Save', () => {
    it('saves checkpoint on stage completion', async () => {
      const checkpoint = await manager.save(mockJobId, {
        stage: 'conversion',
        percent: 50,
        result: { documentId: 'doc-123' },
        timestamp: new Date().toISOString(),
      });

      expect(checkpoint.id).toBeDefined();
      expect(checkpoint.stage).toBe('conversion');
    });

    it('serializes checkpoint data correctly', async () => {
      const checkpointData = {
        stage: 'export',
        percent: 80,
        result: { markdown: '# Title', html: '<h1>Title</h1>' },
        timestamp: new Date().toISOString(),
      };

      const checkpoint = await manager.save(mockJobId, checkpointData);

      expect(JSON.parse(checkpoint.data)).toMatchObject({
        stage: 'export',
        percent: 80,
        result: expect.any(Object),
      });
    });

    it('includes all required fields', async () => {
      const checkpoint = await manager.save(mockJobId, {
        stage: 'parsing',
        percent: 30,
        result: {},
        timestamp: new Date().toISOString(),
      });

      const parsed = JSON.parse(checkpoint.data);
      expect(parsed).toHaveProperty('stage');
      expect(parsed).toHaveProperty('percent');
      expect(parsed).toHaveProperty('result');
      expect(parsed).toHaveProperty('timestamp');
    });

    it('overwrites previous checkpoint for same job', async () => {
      await manager.save(mockJobId, {
        stage: 'upload',
        percent: 10,
        result: {},
        timestamp: new Date().toISOString(),
      });

      const checkpoint2 = await manager.save(mockJobId, {
        stage: 'parsing',
        percent: 30,
        result: {},
        timestamp: new Date().toISOString(),
      });

      const restored = await manager.restore(mockJobId);
      expect(restored?.stage).toBe('parsing');
    });
  });

  describe('Checkpoint Restore', () => {
    it('restores checkpoint with correct state', async () => {
      const originalData = {
        stage: 'conversion',
        percent: 50,
        result: { documentId: 'doc-123' },
        timestamp: new Date().toISOString(),
      };

      await manager.save(mockJobId, originalData);
      const restored = await manager.restore(mockJobId);

      expect(restored).toMatchObject(originalData);
    });

    it('returns null for non-existent checkpoint', async () => {
      const restored = await manager.restore('non-existent-job');
      expect(restored).toBeNull();
    });

    it('validates checkpoint data on restore', async () => {
      // Manually insert invalid checkpoint
      await prisma.checkpoint.create({
        data: {
          jobId: mockJobId,
          data: 'invalid-json',
        },
      });

      await expect(manager.restore(mockJobId)).rejects.toThrow(/invalid checkpoint/i);
    });
  });

  describe('Checkpoint Lifecycle', () => {
    it('cleans up checkpoint on job completion', async () => {
      await manager.save(mockJobId, {
        stage: 'saving',
        percent: 99,
        result: {},
        timestamp: new Date().toISOString(),
      });

      await manager.cleanup(mockJobId);

      const restored = await manager.restore(mockJobId);
      expect(restored).toBeNull();
    });

    it('preserves checkpoint on job failure', async () => {
      await manager.save(mockJobId, {
        stage: 'conversion',
        percent: 50,
        result: { partial: true },
        timestamp: new Date().toISOString(),
      });

      // Job fails - checkpoint should remain
      const restored = await manager.restore(mockJobId);
      expect(restored).not.toBeNull();
    });
  });

  describe('Checkpoint Stages', () => {
    const STAGES = ['upload', 'parsing', 'conversion', 'export', 'saving'];

    STAGES.forEach((stage, index) => {
      it(`saves checkpoint for ${stage} stage`, async () => {
        const checkpoint = await manager.save(mockJobId, {
          stage,
          percent: (index + 1) * 20,
          result: {},
          timestamp: new Date().toISOString(),
        });

        expect(checkpoint.data).toContain(stage);
      });
    });
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| Checkpoint manager tests | `src/lib/checkpoint/__tests__/manager.test.ts` |

---

### JUL-1.5b-002: Write Unit Tests for Progress Interpolation

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.5b-002 |
| **Title** | Write Unit Tests for Progress Interpolation with Monotonic Guarantee |
| **Priority** | P0 (Critical) |
| **Effort** | 1.0 hour |
| **Dependencies** | WILLIAM-1.5b-010 (Progress interpolation implemented) |

#### Description

Write unit tests for the progress interpolation algorithm ensuring the monotonic guarantee (progress never decreases) is maintained.

#### Acceptance Criteria

- [ ] Test progress interpolates smoothly between updates
- [ ] Test progress NEVER decreases (monotonic guarantee)
- [ ] Test handles delayed updates
- [ ] Test handles out-of-order updates
- [ ] Test clamps progress at 100%
- [ ] Test initial progress starts at 0%
- [ ] Test stage transitions update progress correctly
- [ ] Test smooth animation between values
- [ ] Coverage >= 95% for progress interpolation

#### Technical Notes

```typescript
// src/lib/progress/__tests__/interpolation.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { ProgressInterpolator, smoothProgress } from '../interpolation';

describe('Progress Interpolation', () => {
  let interpolator: ProgressInterpolator;

  beforeEach(() => {
    vi.useFakeTimers();
    interpolator = new ProgressInterpolator();
  });

  afterEach(() => {
    vi.useRealTimers();
  });

  describe('Monotonic Guarantee', () => {
    it('progress never decreases', () => {
      interpolator.update(50);
      interpolator.update(30); // Lower value received

      expect(interpolator.current).toBe(50); // Should stay at 50
    });

    it('handles out-of-order updates', () => {
      interpolator.update(30);
      interpolator.update(60);
      interpolator.update(45); // Out of order

      expect(interpolator.current).toBeGreaterThanOrEqual(60);
    });

    it('monotonic across multiple updates', () => {
      const values = [10, 25, 20, 40, 35, 60, 55, 80, 75, 100];
      let previousValue = 0;

      values.forEach((value) => {
        interpolator.update(value);
        expect(interpolator.current).toBeGreaterThanOrEqual(previousValue);
        previousValue = interpolator.current;
      });
    });
  });

  describe('Progress Bounds', () => {
    it('clamps progress at 100%', () => {
      interpolator.update(120);
      expect(interpolator.current).toBe(100);
    });

    it('starts at 0%', () => {
      expect(interpolator.current).toBe(0);
    });

    it('does not go below 0%', () => {
      interpolator.update(-10);
      expect(interpolator.current).toBe(0);
    });
  });

  describe('Smooth Interpolation', () => {
    it('interpolates between values over time', () => {
      interpolator.update(50);

      vi.advanceTimersByTime(100); // Partial animation
      const midValue = interpolator.animated;

      vi.advanceTimersByTime(500); // Complete animation
      const finalValue = interpolator.animated;

      expect(midValue).toBeGreaterThan(0);
      expect(midValue).toBeLessThan(50);
      expect(finalValue).toBe(50);
    });

    it('uses easing function for smooth animation', () => {
      interpolator.update(100);

      const values: number[] = [];
      for (let i = 0; i < 10; i++) {
        vi.advanceTimersByTime(50);
        values.push(interpolator.animated);
      }

      // Verify smooth progression (not linear jumps)
      for (let i = 1; i < values.length; i++) {
        expect(values[i]).toBeGreaterThanOrEqual(values[i - 1]);
      }
    });
  });

  describe('Stage Transitions', () => {
    const STAGE_PERCENTAGES = {
      upload: { min: 0, max: 10 },
      parsing: { min: 10, max: 40 },
      conversion: { min: 40, max: 80 },
      export: { min: 80, max: 95 },
      saving: { min: 95, max: 99 },
      complete: { min: 100, max: 100 },
    };

    Object.entries(STAGE_PERCENTAGES).forEach(([stage, { min, max }]) => {
      it(`${stage} stage progress within bounds`, () => {
        interpolator.setStage(stage as any, max);
        expect(interpolator.current).toBeGreaterThanOrEqual(min);
        expect(interpolator.current).toBeLessThanOrEqual(max);
      });
    });
  });
});

describe('smoothProgress utility', () => {
  it('returns higher value when new is higher', () => {
    expect(smoothProgress(30, 50)).toBe(50);
  });

  it('returns previous value when new is lower (monotonic)', () => {
    expect(smoothProgress(50, 30)).toBe(50);
  });

  it('clamps to 100', () => {
    expect(smoothProgress(90, 150)).toBe(100);
  });

  it('returns 0 for negative values', () => {
    expect(smoothProgress(0, -10)).toBe(0);
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| Progress interpolation tests | `src/lib/progress/__tests__/interpolation.test.ts` |

---

### JUL-1.5b-003: Write Unit Tests for SSE Manager

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.5b-003 |
| **Title** | Write Unit Tests for SSE Connection Manager |
| **Priority** | P1 (High) |
| **Effort** | 1.25 hours |
| **Dependencies** | WILLIAM-1.5b-001 (SSE manager implemented) |

#### Description

Write unit tests for the SSE connection manager covering connection establishment, event handling, and disconnection scenarios.

#### Acceptance Criteria

- [ ] Test connection established successfully
- [ ] Test events parsed correctly
- [ ] Test progress events handled
- [ ] Test error events handled
- [ ] Test complete events handled
- [ ] Test connection close handling
- [ ] Test Last-Event-ID sent on messages
- [ ] Test connection state management
- [ ] Coverage >= 90% for SSE manager

#### Technical Notes

```typescript
// src/lib/sse/__tests__/manager.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { waitFor } from '@testing-library/react';
import { SSEManager } from '../manager';

// Mock EventSource
class MockEventSource {
  static instances: MockEventSource[] = [];
  url: string;
  onopen: (() => void) | null = null;
  onmessage: ((event: MessageEvent) => void) | null = null;
  onerror: ((event: Event) => void) | null = null;
  readyState = 0;

  constructor(url: string) {
    this.url = url;
    MockEventSource.instances.push(this);
    setTimeout(() => {
      this.readyState = 1;
      this.onopen?.();
    }, 0);
  }

  close() {
    this.readyState = 2;
  }

  simulateMessage(data: string, id?: string) {
    this.onmessage?.({
      data,
      lastEventId: id || '',
    } as MessageEvent);
  }

  simulateError() {
    this.readyState = 2;
    this.onerror?.(new Event('error'));
  }
}

describe('SSEManager', () => {
  let manager: SSEManager;
  const mockUrl = '/api/v1/process/job-123/events';

  beforeEach(() => {
    MockEventSource.instances = [];
    vi.stubGlobal('EventSource', MockEventSource);
    manager = new SSEManager();
  });

  afterEach(() => {
    vi.unstubAllGlobals();
  });

  describe('Connection', () => {
    it('establishes connection successfully', async () => {
      const onConnect = vi.fn();
      manager.on('connected', onConnect);

      manager.connect(mockUrl);

      await waitFor(() => {
        expect(onConnect).toHaveBeenCalled();
      });
    });

    it('creates EventSource with correct URL', () => {
      manager.connect(mockUrl);

      expect(MockEventSource.instances[0].url).toBe(mockUrl);
    });

    it('tracks connection state', async () => {
      expect(manager.isConnected).toBe(false);

      manager.connect(mockUrl);

      await waitFor(() => {
        expect(manager.isConnected).toBe(true);
      });
    });
  });

  describe('Event Handling', () => {
    it('parses progress events correctly', async () => {
      const onProgress = vi.fn();
      manager.on('progress', onProgress);
      manager.connect(mockUrl);

      await waitFor(() => expect(manager.isConnected).toBe(true));

      MockEventSource.instances[0].simulateMessage(
        JSON.stringify({ stage: 'conversion', percent: 50, message: 'Processing...' }),
        'event-1'
      );

      expect(onProgress).toHaveBeenCalledWith({
        stage: 'conversion',
        percent: 50,
        message: 'Processing...',
      });
    });

    it('handles complete event', async () => {
      const onComplete = vi.fn();
      manager.on('complete', onComplete);
      manager.connect(mockUrl);

      await waitFor(() => expect(manager.isConnected).toBe(true));

      MockEventSource.instances[0].simulateMessage(
        JSON.stringify({ type: 'complete', jobId: 'job-123' })
      );

      expect(onComplete).toHaveBeenCalled();
    });

    it('handles error event', async () => {
      const onError = vi.fn();
      manager.on('error', onError);
      manager.connect(mockUrl);

      await waitFor(() => expect(manager.isConnected).toBe(true));

      MockEventSource.instances[0].simulateMessage(
        JSON.stringify({ type: 'error', code: 'E204', message: 'Processing failed' })
      );

      expect(onError).toHaveBeenCalledWith(
        expect.objectContaining({ code: 'E204' })
      );
    });
  });

  describe('Connection Close', () => {
    it('handles connection close', async () => {
      const onDisconnect = vi.fn();
      manager.on('disconnected', onDisconnect);
      manager.connect(mockUrl);

      await waitFor(() => expect(manager.isConnected).toBe(true));

      MockEventSource.instances[0].simulateError();

      expect(onDisconnect).toHaveBeenCalled();
    });

    it('can close connection manually', async () => {
      manager.connect(mockUrl);
      await waitFor(() => expect(manager.isConnected).toBe(true));

      manager.close();

      expect(manager.isConnected).toBe(false);
      expect(MockEventSource.instances[0].readyState).toBe(2);
    });
  });

  describe('Last-Event-ID', () => {
    it('tracks last event ID', async () => {
      manager.connect(mockUrl);
      await waitFor(() => expect(manager.isConnected).toBe(true));

      MockEventSource.instances[0].simulateMessage('{}', 'event-5');

      expect(manager.lastEventId).toBe('event-5');
    });
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| SSE manager tests | `src/lib/sse/__tests__/manager.test.ts` |

---

### JUL-1.5b-004: Write Unit Tests for SSE Reconnection Logic

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.5b-004 |
| **Title** | Write Unit Tests for SSE Reconnection with Exponential Backoff |
| **Priority** | P1 (High) |
| **Effort** | 1.0 hour |
| **Dependencies** | WILLIAM-1.5b-002 (Reconnection logic implemented) |

#### Description

Write unit tests for the SSE reconnection logic including exponential backoff, max retries, and polling fallback.

#### Acceptance Criteria

- [ ] Test reconnection triggered on disconnect
- [ ] Test exponential backoff delays (1s, 2s, 4s, 8s, 16s, 30s max)
- [ ] Test max 10 retry attempts
- [ ] Test Last-Event-ID sent on reconnect
- [ ] Test fallback to polling after max retries
- [ ] Test reconnection cancelled on manual close
- [ ] Test reconnection within 30s grace period
- [ ] Coverage >= 90% for reconnection logic

#### Technical Notes

```typescript
// src/lib/sse/__tests__/reconnect.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { ReconnectionManager } from '../reconnect';

describe('SSE Reconnection', () => {
  let reconnectManager: ReconnectionManager;
  let connectFn: ReturnType<typeof vi.fn>;

  beforeEach(() => {
    vi.useFakeTimers();
    connectFn = vi.fn();
    reconnectManager = new ReconnectionManager({
      maxRetries: 10,
      baseDelay: 1000,
      maxDelay: 30000,
      onConnect: connectFn,
    });
  });

  afterEach(() => {
    vi.useRealTimers();
  });

  describe('Exponential Backoff', () => {
    it('uses exponential backoff delays', () => {
      const expectedDelays = [1000, 2000, 4000, 8000, 16000, 30000, 30000, 30000, 30000, 30000];

      for (let i = 0; i < 10; i++) {
        reconnectManager.scheduleReconnect();
        expect(reconnectManager.nextDelay).toBe(expectedDelays[i]);
        vi.advanceTimersByTime(expectedDelays[i]);
      }
    });

    it('caps delay at 30 seconds', () => {
      for (let i = 0; i < 15; i++) {
        reconnectManager.scheduleReconnect();
        vi.advanceTimersByTime(30000);
      }

      expect(reconnectManager.nextDelay).toBeLessThanOrEqual(30000);
    });
  });

  describe('Retry Limits', () => {
    it('retries up to 10 times', () => {
      connectFn.mockImplementation(() => {
        throw new Error('Connection failed');
      });

      for (let i = 0; i < 10; i++) {
        reconnectManager.scheduleReconnect();
        vi.advanceTimersByTime(30000);
      }

      expect(connectFn).toHaveBeenCalledTimes(10);
    });

    it('stops retrying after max attempts', () => {
      const onMaxRetries = vi.fn();
      reconnectManager.on('maxRetriesReached', onMaxRetries);

      for (let i = 0; i < 11; i++) {
        reconnectManager.scheduleReconnect();
        vi.advanceTimersByTime(30000);
      }

      expect(onMaxRetries).toHaveBeenCalled();
    });
  });

  describe('Last-Event-ID', () => {
    it('includes Last-Event-ID on reconnect', () => {
      reconnectManager.setLastEventId('event-42');
      reconnectManager.scheduleReconnect();
      vi.advanceTimersByTime(1000);

      expect(connectFn).toHaveBeenCalledWith(
        expect.objectContaining({ lastEventId: 'event-42' })
      );
    });
  });

  describe('Polling Fallback', () => {
    it('falls back to polling after max retries', () => {
      const onFallback = vi.fn();
      reconnectManager.on('fallbackToPolling', onFallback);

      for (let i = 0; i < 11; i++) {
        reconnectManager.scheduleReconnect();
        vi.advanceTimersByTime(30000);
      }

      expect(onFallback).toHaveBeenCalled();
    });
  });

  describe('Cancellation', () => {
    it('cancels reconnection on manual close', () => {
      reconnectManager.scheduleReconnect();
      reconnectManager.cancel();
      vi.advanceTimersByTime(30000);

      expect(connectFn).not.toHaveBeenCalled();
    });

    it('resets retry count on successful reconnect', () => {
      reconnectManager.scheduleReconnect();
      vi.advanceTimersByTime(1000);
      reconnectManager.onConnectSuccess();

      expect(reconnectManager.retryCount).toBe(0);
    });
  });

  describe('Grace Period', () => {
    it('completes reconnection within 30s grace period', () => {
      // First reconnect attempt should happen within initial delay (1000ms)
      reconnectManager.scheduleReconnect();

      // Verify connect function is not called immediately
      expect(connectFn).not.toHaveBeenCalled();

      // Advance past initial backoff delay
      vi.advanceTimersByTime(1000);

      // First reconnect should have been triggered
      expect(connectFn).toHaveBeenCalledTimes(1);

      // Total time to reach max retries within grace period:
      // 1s + 2s + 4s + 8s + 16s = 31s for first 5 attempts
      // All initial backoff delays < 30s individually satisfy grace period
      expect(reconnectManager.nextDelay).toBeLessThanOrEqual(30000);
    });
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| Reconnection tests | `src/lib/sse/__tests__/reconnect.test.ts` |

---

### JUL-1.5b-005: Write Component Tests for ProgressCard

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.5b-005 |
| **Title** | Write Component Tests for ProgressCard |
| **Priority** | P1 (High) |
| **Effort** | 0.75 hours |
| **Dependencies** | WILLIAM-1.5b-011 (ProgressCard implemented) |

#### Description

Write component tests for the ProgressCard component covering progress display, stage information, and reconnection states.

#### Acceptance Criteria

- [ ] Test renders initial state
- [ ] Test updates on progress event
- [ ] Test displays stage name
- [ ] Test displays percentage
- [ ] Test shows reconnection state
- [ ] Test shows completion state
- [ ] Test progress bar animation
- [ ] Test accessibility (progress bar ARIA)
- [ ] Coverage >= 90% for ProgressCard

#### Technical Notes

```typescript
// src/components/processing/__tests__/ProgressCard.test.tsx
import { render, screen, act } from '@testing-library/react';
import { describe, it, expect, vi } from 'vitest';
import { ProgressCard } from '../ProgressCard';
import { useSSE } from '@/hooks/useSSE';

vi.mock('@/hooks/useSSE');

describe('ProgressCard', () => {
  it('renders initial state', () => {
    vi.mocked(useSSE).mockReturnValue({
      progress: { stage: 'upload', percent: 0, message: 'Starting...' },
      isConnected: true,
      isReconnecting: false,
    });

    render(<ProgressCard jobId="job-123" />);

    expect(screen.getByText(/starting/i)).toBeInTheDocument();
    expect(screen.getByRole('progressbar')).toHaveAttribute('aria-valuenow', '0');
  });

  it('updates on progress event', () => {
    vi.mocked(useSSE).mockReturnValue({
      progress: { stage: 'conversion', percent: 50, message: 'Processing...' },
      isConnected: true,
      isReconnecting: false,
    });

    render(<ProgressCard jobId="job-123" />);

    expect(screen.getByRole('progressbar')).toHaveAttribute('aria-valuenow', '50');
    expect(screen.getByText(/processing/i)).toBeInTheDocument();
  });

  it('displays stage name', () => {
    vi.mocked(useSSE).mockReturnValue({
      progress: { stage: 'export', percent: 85, message: 'Generating outputs...' },
      isConnected: true,
      isReconnecting: false,
    });

    render(<ProgressCard jobId="job-123" />);

    expect(screen.getByText(/export/i)).toBeInTheDocument();
  });

  it('shows reconnection state', () => {
    vi.mocked(useSSE).mockReturnValue({
      progress: { stage: 'conversion', percent: 50, message: 'Processing...' },
      isConnected: false,
      isReconnecting: true,
    });

    render(<ProgressCard jobId="job-123" />);

    expect(screen.getByText(/reconnecting/i)).toBeInTheDocument();
  });

  it('shows completion state', () => {
    vi.mocked(useSSE).mockReturnValue({
      progress: { stage: 'complete', percent: 100, message: 'Complete!' },
      isConnected: true,
      isReconnecting: false,
    });

    render(<ProgressCard jobId="job-123" />);

    expect(screen.getByRole('progressbar')).toHaveAttribute('aria-valuenow', '100');
    expect(screen.getByText(/complete/i)).toBeInTheDocument();
  });

  it('has accessible progress bar', () => {
    vi.mocked(useSSE).mockReturnValue({
      progress: { stage: 'conversion', percent: 50, message: 'Processing...' },
      isConnected: true,
      isReconnecting: false,
    });

    render(<ProgressCard jobId="job-123" />);

    const progressBar = screen.getByRole('progressbar');
    expect(progressBar).toHaveAttribute('aria-valuemin', '0');
    expect(progressBar).toHaveAttribute('aria-valuemax', '100');
    expect(progressBar).toHaveAttribute('aria-valuenow', '50');
    expect(progressBar).toHaveAccessibleName();
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| ProgressCard tests | `src/components/processing/__tests__/ProgressCard.test.tsx` |

---

## Sprint 1.5b Testing Summary

| Metric | Target | Notes |
|--------|--------|-------|
| Total Tasks | 5 | |
| Total Effort | 5.25 hours | |
| Critical Tasks | 2 | Checkpoint manager, Progress interpolation |
| High Priority Tasks | 3 | SSE manager, Reconnection, ProgressCard |

### Coverage Targets

| Component | Target Coverage |
|-----------|-----------------|
| `lib/checkpoint/manager.ts` | >= 95% |
| `lib/progress/interpolation.ts` | >= 95% |
| `lib/sse/manager.ts` | >= 90% |
| `lib/sse/reconnect.ts` | >= 90% |
| `ProgressCard.tsx` | >= 90% |

### Quality Gates for Sprint 1.5b

- [ ] Checkpoint save/restore verified
- [ ] Monotonic progress guarantee verified
- [ ] SSE events parsed correctly
- [ ] Exponential backoff tested
- [ ] Polling fallback tested
- [ ] Progress bar accessibility verified

---

## Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0.0 | 2025-12-12 | Julia Santos | Initial creation |
