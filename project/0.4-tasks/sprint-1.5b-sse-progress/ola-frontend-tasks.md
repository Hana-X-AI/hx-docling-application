# Sprint 1.5b: Frontend UI Tasks - SSE & Progress

**Agent**: Ola Mae Johnson (`@ola`)
**Role**: Frontend UI Developer & Accessibility Specialist
**Sprint**: 1.5b - SSE & Progress Integration
**Dependencies**: Sprint 1.5a Complete
**Effort**: 1.0h (of 4.0h total sprint)

---

## Task Overview

| Task ID | Title | Effort | Status |
|---------|-------|--------|--------|
| OLA-1.5b-001 | Create ProgressCard Component | 25m | Pending |
| OLA-1.5b-002 | Implement Loading State Variants | 15m | Pending |
| OLA-1.5b-003 | Add Reconnection Overlay | 10m | Pending |
| OLA-1.5b-004 | Integrate Progress UI with SSE Hook | 20m | Pending |

**Total Effort**: 1.0h (60 minutes)

---

## OLA-1.5b-001: Create ProgressCard Component

### Description

Build a real-time progress card component that displays processing stage, percentage completion, and status messages. Updates dynamically from SSE events with smooth animations.

### Acceptance Criteria

- [ ] Component displays current processing stage (upload, parsing, conversion, export, saving, complete)
- [ ] Progress bar shows percentage with smooth animation
- [ ] Status message updates with each stage
- [ ] Stage indicator shows visual timeline (past, current, future stages)
- [ ] Cancel button visible during PROCESSING state
- [ ] Progress percentage never decreases (monotonic guarantee)
- [ ] Completion state shows success checkmark
- [ ] Error state shows error icon and message
- [ ] Responsive layout for mobile and desktop
- [ ] Accessibility: progress updates announced to screen readers

### Dependencies

- Sprint 1.5b Task 7: SSE streaming implementation
- Sprint 1.5b Task 10: Progress interpolation with monotonic guarantee
- Sprint 1.1: shadcn/ui components available

### Deliverables

**File**: `src/components/processing/ProgressCard.tsx`

```typescript
'use client'

import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card'
import { Progress } from '@/components/ui/progress'
import { Button } from '@/components/ui/button'
import { CheckCircle, XCircle, Loader2, X } from 'lucide-react'
import { cn } from '@/lib/utils'

export type ProcessingStage =
  | 'upload'
  | 'parsing'
  | 'conversion'
  | 'export'
  | 'saving'
  | 'complete'
  | 'error'

export interface ProgressState {
  stage: ProcessingStage
  percent: number
  message: string
  jobId?: string
}

interface ProgressCardProps {
  progress: ProgressState
  onCancel?: () => void
  className?: string
}

const STAGE_LABELS: Record<ProcessingStage, string> = {
  upload: 'Uploading',
  parsing: 'Analyzing',
  conversion: 'Processing',
  export: 'Generating Outputs',
  saving: 'Saving Results',
  complete: 'Complete',
  error: 'Error',
}

const STAGE_ORDER: ProcessingStage[] = [
  'upload',
  'parsing',
  'conversion',
  'export',
  'saving',
  'complete',
]

export function ProgressCard({ progress, onCancel, className }: ProgressCardProps) {
  const isProcessing =
    progress.stage !== 'complete' && progress.stage !== 'error'
  const currentStageIndex = STAGE_ORDER.indexOf(progress.stage)

  return (
    <Card className={cn('w-full', className)}>
      <CardHeader>
        <div className="flex items-center justify-between">
          <CardTitle className="text-lg">
            {progress.stage === 'complete' && 'Processing Complete'}
            {progress.stage === 'error' && 'Processing Failed'}
            {isProcessing && `Processing Document`}
          </CardTitle>
          {isProcessing && onCancel && (
            <Button
              variant="ghost"
              size="sm"
              onClick={onCancel}
              aria-label="Cancel processing"
            >
              <X className="h-4 w-4 mr-1" />
              Cancel
            </Button>
          )}
        </div>
      </CardHeader>
      <CardContent className="space-y-4">
        {/* Progress Bar */}
        <div className="space-y-2">
          <div className="flex items-center justify-between text-sm">
            <span className="text-muted-foreground">
              {STAGE_LABELS[progress.stage]}
            </span>
            <span className="font-medium">{progress.percent}%</span>
          </div>
          <Progress
            value={progress.percent}
            className="h-2"
            aria-label={`Progress: ${progress.percent}%`}
            aria-valuemin={0}
            aria-valuemax={100}
            aria-valuenow={progress.percent}
            aria-valuetext={`${progress.percent}% - ${progress.message}`}
          />
        </div>

        {/* Status Message */}
        <div
          className="flex items-center gap-2 text-sm"
          role="status"
          aria-live="polite"
          aria-atomic="true"
        >
          {progress.stage === 'complete' && (
            <CheckCircle className="h-5 w-5 text-green-600" />
          )}
          {progress.stage === 'error' && (
            <XCircle className="h-5 w-5 text-destructive" />
          )}
          {isProcessing && (
            <Loader2 className="h-5 w-5 text-primary animate-spin" />
          )}
          <span className="text-muted-foreground">{progress.message}</span>
        </div>

        {/* Stage Timeline */}
        <div className="flex items-center justify-between pt-2">
          {STAGE_ORDER.slice(0, -1).map((stage, index) => (
            <div
              key={stage}
              className="flex flex-col items-center gap-1 flex-1"
            >
              <div
                className={cn(
                  'h-2 w-2 rounded-full transition-colors',
                  index < currentStageIndex
                    ? 'bg-green-600'
                    : index === currentStageIndex
                    ? 'bg-primary animate-pulse'
                    : 'bg-muted'
                )}
                aria-label={`${STAGE_LABELS[stage]} - ${
                  index < currentStageIndex
                    ? 'complete'
                    : index === currentStageIndex
                    ? 'in progress'
                    : 'pending'
                }`}
              />
              <span className="text-xs text-muted-foreground hidden sm:inline">
                {STAGE_LABELS[stage]}
              </span>
            </div>
          ))}
        </div>

        {/* Job ID (for debugging) */}
        {progress.jobId && (
          <p className="text-xs text-muted-foreground mt-2">
            Job ID: {progress.jobId}
          </p>
        )}
      </CardContent>
    </Card>
  )
}
```

### Technical Notes

- **Reference**: Specification FR-502, Implementation Plan Sprint 1.5b Task 11
- **Progress Stages**: Per Specification Section 2.5 (FR-502)
- **Monotonic Guarantee**: Progress percentage never decreases (enforced by progress interpolation)
- **Animations**:
  - Progress bar: Smooth transition with Tailwind `transition-all`
  - Current stage indicator: Pulsing animation
  - Loading spinner: Continuous rotation
- **Accessibility**:
  - Progress bar has ARIA attributes (`role="progressbar"`, `aria-valuenow`, etc.)
  - Status updates announced via `aria-live="polite"`
  - Stage timeline items have descriptive ARIA labels
  - Cancel button accessible with keyboard
- **Responsive**:
  - Stage labels hidden on mobile (`hidden sm:inline`)
  - Card adjusts width on mobile
  - Touch-friendly cancel button (44x44px minimum)

### Testing

```bash
# Unit test
npm run test src/components/processing/ProgressCard.test.tsx

# Manual testing:
# [ ] Progress bar animates smoothly from 0% to 100%
# [ ] Stage indicator moves through all stages
# [ ] Status message updates correctly
# [ ] Cancel button visible during processing
# [ ] Completion shows checkmark icon
# [ ] Error shows error icon
# [ ] Progress never decreases (test with simulated events)
# [ ] Screen reader announces progress updates
# [ ] Responsive on mobile (stage labels hidden)
# [ ] Dark mode renders correctly
```

---

## OLA-1.5b-002: Implement Loading State Variants

### Description

Create various loading state components for different UI contexts: skeleton loaders, spinners, and stage-based loading indicators.

### Acceptance Criteria

- [ ] Skeleton loader created using shadcn/ui Skeleton component
- [ ] Inline spinner variant for buttons
- [ ] Full-page loading overlay variant
- [ ] Stage-based loading (shows which stage is loading)
- [ ] All variants support dark mode
- [ ] Accessibility: loading states announced to screen readers
- [ ] Animations performant (CSS-based, not JavaScript)

### Dependencies

- Sprint 1.1: shadcn/ui Skeleton component installed
- Sprint 1.5b Task 11: ProgressCard component

### Deliverables

**File**: `src/components/processing/LoadingStates.tsx`

```typescript
import { Skeleton } from '@/components/ui/skeleton'
import { Loader2 } from 'lucide-react'
import { cn } from '@/lib/utils'

// Skeleton Loader for Progress Card
export function ProgressCardSkeleton({ className }: { className?: string }) {
  return (
    <div className={cn('space-y-4 p-6 border rounded-lg', className)}>
      <div className="flex items-center justify-between">
        <Skeleton className="h-6 w-40" />
        <Skeleton className="h-8 w-20" />
      </div>
      <div className="space-y-2">
        <div className="flex justify-between">
          <Skeleton className="h-4 w-24" />
          <Skeleton className="h-4 w-12" />
        </div>
        <Skeleton className="h-2 w-full" />
      </div>
      <Skeleton className="h-5 w-64" />
      <div className="flex justify-between">
        {[1, 2, 3, 4, 5].map((i) => (
          <Skeleton key={i} className="h-2 w-2 rounded-full" />
        ))}
      </div>
    </div>
  )
}

// Inline Spinner (for buttons)
export function InlineSpinner({ className }: { className?: string }) {
  return (
    <Loader2
      className={cn('h-4 w-4 animate-spin', className)}
      role="status"
      aria-label="Loading"
    />
  )
}

// Full-page Loading Overlay
export function LoadingOverlay({
  message = 'Loading...',
  className,
}: {
  message?: string
  className?: string
}) {
  return (
    <div
      className={cn(
        'fixed inset-0 z-50 flex items-center justify-center',
        'bg-background/80 backdrop-blur-sm',
        className
      )}
      role="dialog"
      aria-modal="true"
      aria-labelledby="loading-message"
    >
      <div className="flex flex-col items-center gap-4 p-8 rounded-lg bg-card border shadow-lg">
        <Loader2 className="h-12 w-12 animate-spin text-primary" />
        <p id="loading-message" className="text-lg font-medium">
          {message}
        </p>
      </div>
    </div>
  )
}

// Stage Loading Indicator
export function StageLoader({
  stage,
  className,
}: {
  stage: string
  className?: string
}) {
  return (
    <div
      className={cn('flex items-center gap-3 p-4', className)}
      role="status"
      aria-live="polite"
    >
      <Loader2 className="h-5 w-5 animate-spin text-primary" />
      <div>
        <p className="font-medium">{stage}</p>
        <p className="text-sm text-muted-foreground">Please wait...</p>
      </div>
    </div>
  )
}
```

### Technical Notes

- **Reference**: Implementation Plan Sprint 1.5b Task 12
- **Skeleton Component**: Uses shadcn/ui Skeleton with shimmer effect
- **Spinner**: Lucide-react Loader2 icon with CSS animation
- **Performance**: All animations use CSS `transform` and `opacity` (GPU-accelerated)
- **Accessibility**:
  - All loaders have `role="status"`
  - Loading messages announced via `aria-live="polite"`
  - Overlay has `role="dialog"` and `aria-modal="true"`
- **Usage Examples**:
  ```typescript
  // In a component
  if (isLoading) return <ProgressCardSkeleton />

  // In a button
  <Button disabled={isLoading}>
    {isLoading && <InlineSpinner className="mr-2" />}
    Submit
  </Button>

  // Full-page loading
  {isInitializing && <LoadingOverlay message="Initializing..." />}
  ```

### Testing

```bash
# Unit test
npm run test src/components/processing/LoadingStates.test.tsx

# Manual testing:
# [ ] ProgressCardSkeleton matches ProgressCard layout
# [ ] InlineSpinner rotates smoothly
# [ ] LoadingOverlay covers full viewport
# [ ] StageLoader shows correct stage message
# [ ] All variants render in dark mode
# [ ] Animations smooth (60fps)
# [ ] Screen reader announces loading states
# [ ] Overlay traps focus (modal behavior)
```

---

## OLA-1.5b-003: Add Reconnection Overlay

### Description

Create a reconnection overlay that displays when SSE connection is lost and attempts are being made to reconnect. Shows retry count and provides manual retry option.

### Acceptance Criteria

- [ ] Overlay displays when SSE connection lost
- [ ] Shows reconnection attempt count (e.g., "Reconnecting... (Attempt 2 of 10)")
- [ ] Shows exponential backoff countdown timer
- [ ] Provides manual "Retry Now" button
- [ ] Automatically dismisses when connection restored
- [ ] Shows "Reconnected" message briefly on success
- [ ] Falls back to "Connection Failed" after max retries
- [ ] Accessibility: status updates announced to screen readers
- [ ] Does not block user from canceling job

### Dependencies

- Sprint 1.5b Task 2: SSE reconnection logic
- Sprint 1.5b Task 3: Polling fallback mechanism
- OLA-1.5b-002: Loading state variants

### Deliverables

**File**: `src/components/processing/ReconnectionOverlay.tsx`

```typescript
'use client'

import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card'
import { Button } from '@/components/ui/button'
import { WifiOff, Loader2, CheckCircle, XCircle } from 'lucide-react'
import { cn } from '@/lib/utils'

export type ReconnectionStatus =
  | 'reconnecting'
  | 'connected'
  | 'failed'
  | 'polling'

interface ReconnectionOverlayProps {
  status: ReconnectionStatus
  attemptCount: number
  maxAttempts: number
  nextRetryIn?: number // seconds
  onRetryNow?: () => void
  onDismiss?: () => void
  className?: string
}

export function ReconnectionOverlay({
  status,
  attemptCount,
  maxAttempts,
  nextRetryIn,
  onRetryNow,
  onDismiss,
  className,
}: ReconnectionOverlayProps) {
  if (status === 'connected') {
    return null // Don't show overlay when connected
  }

  return (
    <div
      className={cn(
        'fixed bottom-4 right-4 z-50 max-w-sm',
        'animate-in slide-in-from-bottom-4',
        className
      )}
      role="alert"
      aria-live="assertive"
      aria-atomic="true"
    >
      <Card className="border-2 shadow-lg">
        <CardHeader className="pb-3">
          <div className="flex items-center gap-2">
            {status === 'reconnecting' && (
              <Loader2 className="h-5 w-5 animate-spin text-orange-600" />
            )}
            {status === 'failed' && (
              <XCircle className="h-5 w-5 text-destructive" />
            )}
            {status === 'polling' && (
              <WifiOff className="h-5 w-5 text-orange-600" />
            )}
            <CardTitle className="text-base">
              {status === 'reconnecting' && 'Reconnecting...'}
              {status === 'failed' && 'Connection Failed'}
              {status === 'polling' && 'Using Backup Connection'}
            </CardTitle>
          </div>
        </CardHeader>
        <CardContent className="space-y-3">
          {status === 'reconnecting' && (
            <>
              <p className="text-sm text-muted-foreground">
                Attempt {attemptCount} of {maxAttempts}
                {nextRetryIn && ` • Next retry in ${nextRetryIn}s`}
              </p>
              {onRetryNow && (
                <Button
                  variant="outline"
                  size="sm"
                  onClick={onRetryNow}
                  className="w-full"
                >
                  Retry Now
                </Button>
              )}
            </>
          )}

          {status === 'failed' && (
            <>
              <p className="text-sm text-muted-foreground">
                Unable to reconnect after {maxAttempts} attempts. Progress
                updates may be delayed.
              </p>
              <div className="flex gap-2">
                {onRetryNow && (
                  <Button
                    variant="default"
                    size="sm"
                    onClick={onRetryNow}
                    className="flex-1"
                  >
                    Retry
                  </Button>
                )}
                {onDismiss && (
                  <Button
                    variant="outline"
                    size="sm"
                    onClick={onDismiss}
                    className="flex-1"
                  >
                    Dismiss
                  </Button>
                )}
              </div>
            </>
          )}

          {status === 'polling' && (
            <p className="text-sm text-muted-foreground">
              Real-time updates unavailable. Checking progress every 2 seconds.
            </p>
          )}
        </CardContent>
      </Card>
    </div>
  )
}

// Success Toast (shown briefly when reconnected)
export function ReconnectedToast({ onDismiss }: { onDismiss?: () => void }) {
  return (
    <div
      className="fixed bottom-4 right-4 z-50 max-w-sm animate-in slide-in-from-bottom-4"
      role="status"
      aria-live="polite"
    >
      <Card className="border-2 border-green-600 shadow-lg">
        <CardContent className="p-4">
          <div className="flex items-center gap-2">
            <CheckCircle className="h-5 w-5 text-green-600" />
            <p className="font-medium">Reconnected</p>
          </div>
        </CardContent>
      </Card>
    </div>
  )
}
```

### Technical Notes

- **Reference**: Specification FR-503, Implementation Plan Sprint 1.5b Task 13
- **Positioning**: Fixed bottom-right corner (non-blocking)
- **Auto-dismiss**: Reconnection success toast auto-dismisses after 3 seconds
- **Retry Logic**: Manual retry button triggers immediate reconnection attempt
- **Status Types**:
  - `reconnecting`: Active reconnection attempts
  - `failed`: Max retries exceeded
  - `polling`: Fallback to polling mode
  - `connected`: Normal operation (overlay hidden)
- **Accessibility**:
  - `role="alert"` for immediate attention
  - `aria-live="assertive"` for connection failures
  - All buttons keyboard accessible
  - Status changes announced to screen readers
- **Animation**: Smooth slide-in from bottom using Tailwind animate utilities

### Testing

```bash
# Unit test
npm run test src/components/processing/ReconnectionOverlay.test.tsx

# Manual testing (simulate disconnection):
# [ ] Disconnect network - overlay appears
# [ ] Attempt counter increments
# [ ] Next retry countdown updates
# [ ] "Retry Now" button triggers immediate retry
# [ ] After max attempts - shows "Connection Failed"
# [ ] Reconnect network - overlay dismisses
# [ ] "Reconnected" toast appears briefly
# [ ] Polling fallback message shows correctly
# [ ] Screen reader announces status changes
# [ ] Overlay does not block other UI interactions
# [ ] Dark mode renders correctly
```

---

## OLA-1.5b-004: Integrate Progress UI with SSE Hook

### Description

Connect the ProgressCard and ReconnectionOverlay components to the SSE hook, creating the complete progress tracking UI experience.

### Acceptance Criteria

- [ ] ProgressCard receives real-time updates from SSE hook
- [ ] Progress percentage updates smoothly (monotonic guarantee enforced)
- [ ] Stage transitions trigger visual updates
- [ ] Reconnection overlay shows on SSE disconnection
- [ ] Cancel button triggers job cancellation
- [ ] Completion state shows results viewer
- [ ] Error state shows error message with retry option
- [ ] Component handles all job statuses (PENDING, PROCESSING, COMPLETED, FAILED, CANCELLED, PARTIAL_COMPLETE)
- [ ] Loading skeleton shown during initial connection

### Dependencies

- OLA-1.5b-001: ProgressCard component complete
- OLA-1.5b-002: Loading states complete
- OLA-1.5b-003: Reconnection overlay complete
- Sprint 1.5b Task 14: useSSE hook implementation
- Sprint 1.5b Task 15: useProcess hook implementation

### Deliverables

**File**: `src/components/processing/ProcessingView.tsx`

```typescript
'use client'

import { useEffect } from 'react'
import { ProgressCard, ProcessingStage, ProgressState } from './ProgressCard'
import { ProgressCardSkeleton } from './LoadingStates'
import { ReconnectionOverlay, ReconnectedToast } from './ReconnectionOverlay'
import { useSSE } from '@/hooks/useSSE'
import { useProcess } from '@/hooks/useProcess'
import { Alert, AlertDescription, AlertTitle } from '@/components/ui/alert'
import { Button } from '@/components/ui/button'
import { AlertCircle } from 'lucide-react'

interface ProcessingViewProps {
  jobId: string
  onComplete?: (jobId: string) => void
  onCancel?: (jobId: string) => void
}

export function ProcessingView({
  jobId,
  onComplete,
  onCancel,
}: ProcessingViewProps) {
  const {
    progress,
    reconnectionStatus,
    reconnectionAttempts,
    isConnecting,
    retryConnection,
    dismissReconnectionOverlay,
  } = useSSE(jobId)

  const { cancel, isCancelling } = useProcess(jobId)

  useEffect(() => {
    if (progress?.stage === 'complete') {
      onComplete?.(jobId)
    }
  }, [progress?.stage, jobId, onComplete])

  const handleCancel = async () => {
    await cancel()
    onCancel?.(jobId)
  }

  // Initial loading state
  if (isConnecting && !progress) {
    return <ProgressCardSkeleton />
  }

  // No progress data
  if (!progress) {
    return (
      <Alert variant="destructive">
        <AlertCircle className="h-4 w-4" />
        <AlertTitle>Connection Error</AlertTitle>
        <AlertDescription>
          Unable to establish connection. Please try again.
        </AlertDescription>
      </Alert>
    )
  }

  return (
    <div className="space-y-4">
      {/* Progress Card */}
      <ProgressCard
        progress={progress}
        onCancel={progress.stage !== 'complete' ? handleCancel : undefined}
      />

      {/* Reconnection Overlay */}
      {reconnectionStatus !== 'connected' && (
        <ReconnectionOverlay
          status={reconnectionStatus}
          attemptCount={reconnectionAttempts}
          maxAttempts={10}
          onRetryNow={retryConnection}
          onDismiss={dismissReconnectionOverlay}
        />
      )}

      {/* Reconnected Toast */}
      {reconnectionStatus === 'connected' && reconnectionAttempts > 0 && (
        <ReconnectedToast onDismiss={dismissReconnectionOverlay} />
      )}

      {/* Error State */}
      {progress.stage === 'error' && (
        <Alert variant="destructive">
          <AlertCircle className="h-4 w-4" />
          <AlertTitle>Processing Failed</AlertTitle>
          <AlertDescription>
            {progress.message}
            <Button
              variant="outline"
              size="sm"
              className="mt-2"
              onClick={() => window.location.reload()}
            >
              Try Again
            </Button>
          </AlertDescription>
        </Alert>
      )}
    </div>
  )
}
```

**Usage Example**:

```typescript
// In a page or parent component
import { ProcessingView } from '@/components/processing/ProcessingView'

export function DocumentProcessingPage({ jobId }: { jobId: string }) {
  const handleComplete = (jobId: string) => {
    console.log('Processing complete:', jobId)
    // Navigate to results page or show results
  }

  const handleCancel = (jobId: string) => {
    console.log('Processing cancelled:', jobId)
    // Navigate back or show cancellation message
  }

  return (
    <div className="container mx-auto py-8">
      <ProcessingView
        jobId={jobId}
        onComplete={handleComplete}
        onCancel={handleCancel}
      />
    </div>
  )
}
```

### Technical Notes

- **Reference**: Implementation Plan Sprint 1.5b Tasks 14-15
- **SSE Hook Integration**: Receives progress events and connection status
- **Progress Hook Integration**: Provides cancel functionality
- **State Management**:
  - Loading: Show skeleton loader
  - Processing: Show progress card
  - Reconnecting: Show reconnection overlay
  - Complete: Trigger onComplete callback
  - Error: Show error alert
- **Monotonic Guarantee**: Enforced by progress interpolation in SSE hook
- **Auto-navigation**: Component triggers callbacks for parent to handle routing
- **Error Recovery**: Reload button on error state

### Testing

```bash
# Unit test
npm run test src/components/processing/ProcessingView.test.tsx

# E2E test
npm run test:e2e -- --grep "processing flow"

# Manual testing:
# [ ] Upload file - processing view appears
# [ ] Progress updates in real-time
# [ ] Progress bar never decreases
# [ ] Stage indicator moves through stages
# [ ] Disconnect network - reconnection overlay appears
# [ ] Reconnect network - overlay dismisses
# [ ] Click cancel - job cancels and callback fires
# [ ] Processing completes - onComplete callback fires
# [ ] Error occurs - error alert shown
# [ ] Screen reader announces all updates
# [ ] Works in dark mode
```

---

## Sprint Completion Checklist

### Acceptance Criteria (Sprint 1.5b Frontend)

- [ ] ProgressCard displays real-time progress updates
- [ ] Progress bar animates smoothly with monotonic guarantee
- [ ] Stage timeline shows current and completed stages
- [ ] Loading states render correctly (skeleton, spinner, overlay)
- [ ] Reconnection overlay shows on SSE disconnection
- [ ] Reconnection attempts count down with retry option
- [ ] Polling fallback message displays correctly
- [ ] Cancel button functional during processing
- [ ] Completion state shows success checkmark
- [ ] Error state shows error message with retry
- [ ] All components accessible (ARIA, keyboard, screen reader)
- [ ] Dark mode functional for all components
- [ ] Responsive layout on mobile, tablet, desktop
- [ ] Unit tests pass with >= 80% coverage
- [ ] E2E test: Upload → Progress → Complete
- [ ] No TypeScript errors
- [ ] No console warnings

### Integration Points

**With William (SSE Backend)**:
- Receives progress events via SSE hook
- Handles reconnection logic from backend
- Displays Last-Event-ID recovery status

**With Neo (Process Hook)**:
- Cancel button triggers process hook cancel method
- Receives job status updates
- Handles completion and error states

**With Julia (Testing)**:
- Unit tests for all progress components
- E2E tests for complete processing flow
- Accessibility tests for progress updates

### Files Delivered

```
src/components/processing/
├── ProgressCard.tsx
├── LoadingStates.tsx
├── ReconnectionOverlay.tsx
├── ProcessingView.tsx
├── ProgressCard.test.tsx
├── LoadingStates.test.tsx
├── ReconnectionOverlay.test.tsx
└── ProcessingView.test.tsx
```

### Next Sprint Preview

**Sprint 1.6** will add:
- ResultsViewer component with tabbed interface
- Format-specific renderers (Markdown, HTML, JSON, Raw)
- Download functionality
- Partial result indicators

---

## Notes

- **Performance**: All animations use CSS for 60fps smooth experience
- **UX**: Progress never decreases for better user confidence
- **Accessibility**: All state changes announced to screen readers
- **Resilience**: Graceful degradation to polling fallback
- **Responsiveness**: Mobile-optimized with touch-friendly targets
