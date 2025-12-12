# Sprint 1.4: Frontend UI Tasks - URL Input

**Agent**: Ola Mae Johnson (`@ola`)
**Role**: Frontend UI Developer & Accessibility Specialist
**Sprint**: 1.4 - URL Input with Security
**Dependencies**: Sprint 1.3 Complete
**Effort**: 0.75h (of 2.0h total sprint)

---

## Task Overview

| Task ID | Title | Effort | Status |
|---------|-------|--------|--------|
| OLA-1.4-001 | Create URL Input Component with Validation UI | 25m | Pending |
| OLA-1.4-002 | Create UI Store for Toast and Modal State | 20m | Pending |
| OLA-1.4-003 | Style and Test URL Input Component | 15m | Pending |
| OLA-1.4-004 | Implement Accessibility for URL Input | 15m | Pending |

**Total Effort**: 0.75h (45 minutes)

---

## OLA-1.4-001: Create URL Input Component with Validation UI

### Description

Build the URL input component using `react-hook-form` with real-time validation feedback. Displays error messages for invalid URLs, SSRF-blocked addresses, and format violations.

### Acceptance Criteria

- [ ] Component created as Client Component with `'use client'` directive
- [ ] Input field integrated with react-hook-form
- [ ] Real-time URL format validation with visual feedback
- [ ] SSRF validation errors display with error code E104
- [ ] Submit button disabled for invalid or blocked URLs
- [ ] Clear button to reset input
- [ ] Paste from clipboard supported
- [ ] Character counter shows remaining characters (max 2048)
- [ ] Error messages positioned below input with appropriate styling
- [ ] Loading state during URL preview fetch (optional preview feature)

### Dependencies

- Sprint 1.3: UploadForm pattern established
- Sprint 1.4 Task 1: URL input component planning
- Sprint 1.4 Task 2: SSRF prevention implementation
- Sprint 1.4 Task 3: URL format validation schema

### Deliverables

**File**: `src/components/upload/UrlInput.tsx`

```typescript
'use client'

import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'
import { X } from 'lucide-react'
import { Input } from '@/components/ui/input'
import { Button } from '@/components/ui/button'
import { Form, FormControl, FormField, FormItem, FormLabel, FormMessage } from '@/components/ui/form'
import { Card, CardContent } from '@/components/ui/card'
import { useToast } from '@/components/ui/use-toast'
import { useState } from 'react'

const urlSchema = z.object({
  url: z
    .string()
    .min(1, 'URL is required')
    .max(2048, 'URL must be less than 2048 characters')
    .url('Invalid URL format')
    .refine((url) => url.startsWith('http://') || url.startsWith('https://'), {
      message: 'URL must start with http:// or https://',
    }),
})

type UrlFormValues = z.infer<typeof urlSchema>

interface UrlInputProps {
  onUrlSubmit: (url: string) => void
  disabled?: boolean
}

export function UrlInput({ onUrlSubmit, disabled = false }: UrlInputProps) {
  const [isValidating, setIsValidating] = useState(false)
  const { toast } = useToast()

  const form = useForm<UrlFormValues>({
    resolver: zodResolver(urlSchema),
    defaultValues: {
      url: '',
    },
    mode: 'onChange', // Real-time validation
  })

  const urlValue = form.watch('url')

  const onSubmit = async (data: UrlFormValues) => {
    setIsValidating(true)
    try {
      // Validate SSRF on server-side
      const response = await fetch('/api/v1/validate-url', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ url: data.url }),
      })

      if (!response.ok) {
        const error = await response.json()
        if (error.code === 'E104') {
          toast({
            title: 'URL Blocked',
            description: 'This URL points to a private or internal address',
            variant: 'destructive',
          })
        } else {
          throw new Error(error.message || 'Validation failed')
        }
        return
      }

      onUrlSubmit(data.url)
    } catch (error) {
      toast({
        title: 'Validation failed',
        description: error instanceof Error ? error.message : 'Unknown error',
        variant: 'destructive',
      })
    } finally {
      setIsValidating(false)
    }
  }

  return (
    <Card>
      <CardContent className="p-6">
        <Form {...form}>
          <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
            <FormField
              control={form.control}
              name="url"
              render={({ field }) => (
                <FormItem>
                  <FormLabel>Enter URL to Process</FormLabel>
                  <div className="relative">
                    <FormControl>
                      <Input
                        {...field}
                        type="url"
                        placeholder="https://example.com/document.pdf"
                        disabled={disabled || isValidating}
                        className="pr-10"
                        aria-describedby="url-help url-error"
                      />
                    </FormControl>
                    {urlValue && (
                      <Button
                        type="button"
                        variant="ghost"
                        size="icon"
                        className="absolute right-2 top-1/2 -translate-y-1/2 h-6 w-6"
                        onClick={() => form.reset()}
                        aria-label="Clear URL"
                      >
                        <X className="h-4 w-4" />
                      </Button>
                    )}
                  </div>
                  <div className="flex justify-between items-center">
                    <FormMessage id="url-error" />
                    <p
                      id="url-help"
                      className="text-xs text-muted-foreground"
                    >
                      {urlValue.length} / 2048
                    </p>
                  </div>
                </FormItem>
              )}
            />

            <Button
              type="submit"
              disabled={!form.formState.isValid || disabled || isValidating}
              className="w-full"
            >
              {isValidating ? 'Validating...' : 'Process URL'}
            </Button>
          </form>
        </Form>
      </CardContent>
    </Card>
  )
}
```

### Technical Notes

- **Reference**: Specification FR-201, FR-202, FR-203
- **Validation**: Client-side Zod + server-side SSRF check
- **Max Length**: 2048 characters per specification
- **Error Codes**:
  - E101: Malformed URL
  - E104: SSRF-blocked URL
- **Real-time Validation**: `mode: 'onChange'` for immediate feedback
- **Accessibility**:
  - Input has descriptive label
  - Error messages linked via `aria-describedby`
  - Character counter visible
  - Clear button accessible

### Testing

```bash
# Unit test
npm run test src/components/upload/UrlInput.test.tsx

# Manual testing:
# [ ] Enter valid URL - submit enabled
# [ ] Enter invalid URL - error shown, submit disabled
# [ ] Enter localhost URL - E104 error on submit
# [ ] Enter 192.168.x.x URL - E104 error on submit
# [ ] Paste URL from clipboard - works
# [ ] Clear button removes URL
# [ ] Character counter updates in real-time
# [ ] Validation happens as user types
```

---

## OLA-1.4-002: Create UI Store for Toast and Modal State

### Description

Create a Zustand store for managing global UI state including toast notifications, modal dialogs, and other transient UI states.

### Acceptance Criteria

- [ ] Zustand store created with TypeScript types
- [ ] Toast state management (show, hide, queue)
- [ ] Modal state management (open, close, data)
- [ ] Loading state indicators
- [ ] Store persists to sessionStorage for toast queue
- [ ] Actions for common UI patterns (showError, showSuccess, showLoading)

### Dependencies

- Sprint 1.4 Task 7: UI store planning
- Sprint 1.1: Zustand package installed

### Deliverables

**File**: `src/stores/uiStore.ts`

```typescript
import { create } from 'zustand'
import { persist, createJSONStorage, type StateStorage } from 'zustand/middleware'

// SSR-safe storage that returns a no-op storage on the server
// and sessionStorage on the client
const getSessionStorage = (): StateStorage => {
  if (typeof window === 'undefined') {
    // Return no-op storage for SSR
    return {
      getItem: () => null,
      setItem: () => {},
      removeItem: () => {},
    }
  }
  return sessionStorage
}

export interface Toast {
  id: string
  title: string
  description?: string
  variant?: 'default' | 'destructive'
  duration?: number
  createdAt: number // Unix timestamp for computing remaining duration on rehydration
}

export interface Modal {
  id: string
  title: string
  content: React.ReactNode
  onClose?: () => void
}

interface UIState {
  // Toast state
  toasts: Toast[]
  addToast: (toast: Omit<Toast, 'id'>) => void
  removeToast: (id: string) => void
  clearToasts: () => void

  // Modal state
  modal: Modal | null
  openModal: (modal: Omit<Modal, 'id'>) => void
  closeModal: () => void

  // Loading state
  isLoading: boolean
  loadingMessage: string | null
  setLoading: (isLoading: boolean, message?: string) => void

  // Convenience actions
  showSuccess: (title: string, description?: string) => void
  showError: (title: string, description?: string) => void
  showInfo: (title: string, description?: string) => void
}

// Timer tracking Map - stored outside Zustand state to avoid persistence
// and ensure timers are properly managed without serialization issues
const toastTimers = new Map<string, ReturnType<typeof setTimeout>>()

// Helper to clear a single toast timer
const clearToastTimer = (id: string) => {
  const timerId = toastTimers.get(id)
  if (timerId !== undefined) {
    clearTimeout(timerId)
    toastTimers.delete(id)
  }
}

// Helper to clear all toast timers
const clearAllToastTimers = () => {
  toastTimers.forEach((timerId) => clearTimeout(timerId))
  toastTimers.clear()
}

export const useUIStore = create<UIState>()(
  persist(
    (set, get) => ({
      // Toast state
      toasts: [],
      addToast: (toast) => {
        const id = Math.random().toString(36).substr(2, 9)
        const createdAt = Date.now()
        const duration = toast.duration || 5000
        set((state) => ({
          toasts: [...state.toasts, { ...toast, id, createdAt, duration }],
        }))
        // Auto-remove after duration - store timer ID for cleanup
        const timerId = setTimeout(() => {
          toastTimers.delete(id) // Clean up timer reference before removal
          get().removeToast(id)
        }, duration)
        toastTimers.set(id, timerId)
      },
      removeToast: (id) => {
        // Clear any pending timer for this toast before removal
        clearToastTimer(id)
        set((state) => ({
          toasts: state.toasts.filter((t) => t.id !== id),
        }))
      },
      clearToasts: () => {
        // Clear all pending timers before clearing toasts
        clearAllToastTimers()
        set({ toasts: [] })
      },

      // Modal state
      modal: null,
      openModal: (modal) => {
        const id = Math.random().toString(36).substr(2, 9)
        set({ modal: { ...modal, id } })
      },
      closeModal: () => set({ modal: null }),

      // Loading state
      isLoading: false,
      loadingMessage: null,
      setLoading: (isLoading, message = null) => {
        set({ isLoading, loadingMessage: message })
      },

      // Convenience actions
      showSuccess: (title, description) => {
        get().addToast({
          title,
          description,
          variant: 'default',
          duration: 5000,
        })
      },
      showError: (title, description) => {
        get().addToast({
          title,
          description,
          variant: 'destructive',
          duration: 7000,
        })
      },
      showInfo: (title, description) => {
        get().addToast({
          title,
          description,
          variant: 'default',
          duration: 5000,
        })
      },
    }),
    {
      name: 'ui-storage',
      storage: createJSONStorage(getSessionStorage),
      // Only persist toast payload fields - timers are managed externally
      // and should not be serialized to sessionStorage
      partialize: (state) => ({
        toasts: state.toasts.map(({ id, title, description, variant, duration, createdAt }) => ({
          id,
          title,
          description,
          variant,
          duration,
          createdAt,
        })),
      }),
      // Rehydrate storage callback - schedule timers for persisted toasts
      onRehydrateStorage: () => (state) => {
        if (!state?.toasts?.length) return

        const now = Date.now()
        const removeToast = state.removeToast

        state.toasts.forEach((toast) => {
          const duration = toast.duration || 5000
          const elapsed = now - toast.createdAt
          const remaining = Math.max(0, duration - elapsed)

          if (remaining <= 0) {
            // Toast has already expired - remove immediately
            removeToast(toast.id)
          } else {
            // Schedule timer for remaining duration
            const timerId = setTimeout(() => {
              toastTimers.delete(toast.id)
              removeToast(toast.id)
            }, remaining)
            toastTimers.set(toast.id, timerId)
          }
        })
      },
    }
  )
)
```

### Technical Notes

- **Reference**: Implementation Plan Sprint 1.4 Task 7
- **Library**: `zustand@^5.x`
- **Persistence**: Toast queue persists to sessionStorage
- **Auto-cleanup**: Toasts auto-remove after duration
- **Type Safety**: Full TypeScript types for all state
- **Usage Example**:
  ```typescript
  const { showSuccess, showError } = useUIStore()
  showSuccess('Upload complete', 'Job ID: 12345')
  showError('Upload failed', 'File too large')
  ```

### Testing

```bash
# Unit test
npm run test src/stores/uiStore.test.ts

# Manual testing:
# [ ] Add toast - appears in state
# [ ] Toast auto-removes after duration
# [ ] Multiple toasts queue correctly
# [ ] Modal opens and closes
# [ ] Loading state toggles
# [ ] sessionStorage contains toast data
# [ ] Clear toasts removes all
```

---

## OLA-1.4-003: Style and Test URL Input Component

### Description

Apply consistent shadcn/ui styling to the URL input component and ensure visual coherence with the upload components. Test across light/dark modes and responsive breakpoints.

### Acceptance Criteria

- [ ] Component uses shadcn/ui Card, Input, Button, Form components
- [ ] Styling consistent with UploadZone and UploadForm
- [ ] Dark mode styling verified
- [ ] Responsive layout tested (mobile, tablet, desktop)
- [ ] Focus states visually distinct
- [ ] Error states clearly indicated with color and icon
- [ ] Character counter positioned appropriately
- [ ] Clear button positioned within input (right side)

### Dependencies

- OLA-1.4-001: URL input component complete
- Sprint 1.1: Tailwind theme configured

### Deliverables

**Visual Design Specifications**:

| Element | Light Mode | Dark Mode | Notes |
|---------|-----------|-----------|-------|
| Card Background | `bg-background` | `bg-background` | CSS variable |
| Input Border | `border-border` | `border-border` | Default state |
| Input Border (Focus) | `ring-primary` | `ring-primary` | Focus ring |
| Input Border (Error) | `border-destructive` | `border-destructive` | Error state |
| Label Text | `text-foreground` | `text-foreground` | Primary text |
| Helper Text | `text-muted-foreground` | `text-muted-foreground` | Character counter |
| Error Message | `text-destructive` | `text-destructive` | Error text |

**Responsive Breakpoints**:
- Mobile (< 768px): Full width card, stacked layout
- Tablet (768px - 1023px): Optimized input width
- Desktop (>= 1024px): Max width container

### Technical Notes

- **Reference**: Implementation Plan Sprint 1.4 Task 9
- **Consistency**: Matches UploadZone styling patterns
- **Spacing**: Uses 8px grid (p-4, gap-4, mt-2)
- **Typography**:
  - Label: `text-sm font-medium`
  - Input: `text-base`
  - Helper: `text-xs text-muted-foreground`
  - Error: `text-sm text-destructive`
- **Transitions**: Smooth border and background transitions on state changes

### Testing

```bash
# Visual regression testing
npm run test:visual

# Manual testing:
# [ ] Light mode: all colors render correctly
# [ ] Dark mode: all colors render correctly
# [ ] Toggle theme: smooth transitions
# [ ] Mobile: full width layout
# [ ] Tablet: optimized width
# [ ] Desktop: centered with max width
# [ ] Focus state: visible ring
# [ ] Error state: red border and message
# [ ] Valid state: normal border
```

---

## OLA-1.4-004: Implement Accessibility for URL Input

### Description

Ensure URL input component meets WCAG 2.1 Level AA accessibility standards with proper ARIA attributes, keyboard navigation, and screen reader support.

### Acceptance Criteria

- [ ] Input has descriptive label
- [ ] Error messages linked via `aria-describedby`
- [ ] Helper text (character counter) linked via `aria-describedby`
- [ ] Clear button has ARIA label
- [ ] Submit button disabled state communicated to screen readers
- [ ] Validation errors announced to screen readers (role="alert")
- [ ] Keyboard navigation functional (Tab, Enter, Escape)
- [ ] Focus management: clear button returns focus to input
- [ ] Color contrast >= 4.5:1 for all text
- [ ] Touch targets >= 44x44px on mobile

### Dependencies

- OLA-1.4-001: URL input component complete
- OLA-1.4-003: Styling complete

### Deliverables

**Accessibility Enhancements**:

```typescript
// UrlInput.tsx - Accessibility attributes
<FormField
  control={form.control}
  name="url"
  render={({ field }) => (
    <FormItem>
      <FormLabel htmlFor="url-input">Enter URL to Process</FormLabel>
      <div className="relative">
        <FormControl>
          <Input
            {...field}
            id="url-input"
            type="url"
            placeholder="https://example.com/document.pdf"
            aria-describedby="url-help url-error"
            aria-invalid={!!form.formState.errors.url}
            aria-required="true"
          />
        </FormControl>
        {urlValue && (
          <Button
            type="button"
            variant="ghost"
            size="icon"
            onClick={() => {
              form.reset()
              document.getElementById('url-input')?.focus()
            }}
            aria-label="Clear URL input"
          >
            <X className="h-4 w-4" />
          </Button>
        )}
      </div>
      <div className="flex justify-between items-center">
        <FormMessage id="url-error" role="alert" aria-live="polite" />
        <p id="url-help" className="text-xs text-muted-foreground">
          {urlValue.length} / 2048 characters
        </p>
      </div>
    </FormItem>
  )}
/>
```

**Keyboard Navigation**:
- Tab: Navigate to input field
- Tab: Navigate to clear button (if URL present)
- Tab: Navigate to submit button
- Enter: Submit form (when input or submit button focused)
- Escape: Clear input (when clear button focused)

### Technical Notes

- **Reference**: Implementation Plan Sprint 1.4 Task 10, Specification Section 9
- **ARIA Attributes**:
  - `aria-describedby`: Links input to helper text and error
  - `aria-invalid`: Indicates validation state
  - `aria-required`: Indicates required field
  - `role="alert"`: Announces errors immediately
  - `aria-live="polite"`: Announces changes when user is idle
- **Focus Management**:
  - Clear button returns focus to input
  - Form submission maintains focus context
- **Screen Reader Testing**: Test with NVDA (Windows) or VoiceOver (Mac)

### Testing

```bash
# Automated accessibility testing
npm run test:a11y

# Lighthouse audit
npm run lighthouse

# Manual testing checklist:
# [ ] Tab through form - all elements reachable
# [ ] Focus indicators visible
# [ ] Screen reader announces label when input focused
# [ ] Screen reader announces error when validation fails
# [ ] Screen reader announces character count
# [ ] Enter key submits form from input
# [ ] Clear button accessible with keyboard
# [ ] Clear button returns focus to input
# [ ] Color contrast verified (WebAIM Contrast Checker)
# [ ] Touch targets measured (>= 44x44px)
# [ ] Form works without mouse (keyboard only)
```

---

## Sprint Completion Checklist

### Acceptance Criteria (Sprint 1.4 Frontend)

- [ ] URL input accepts valid HTTP/HTTPS URLs
- [ ] Invalid URLs show validation errors
- [ ] SSRF-blocked URLs show E104 error
- [ ] Character counter shows remaining characters (max 2048)
- [ ] Clear button removes URL and returns focus
- [ ] Submit button disabled for invalid URLs
- [ ] UI store manages toast notifications
- [ ] Styling consistent with upload components
- [ ] Dark mode functional
- [ ] Accessibility checklist complete (ARIA, keyboard, screen reader)
- [ ] Unit tests pass for URL validation
- [ ] Unit tests pass for UI store
- [ ] No TypeScript errors
- [ ] No console warnings in browser

### Integration Points

**With Neo (Backend)**:
- URL validates against `/api/v1/validate-url` endpoint
- Receives SSRF validation result
- Handles rate limiting (429) errors

**With Sophia (Document Store)**:
- Integrates with `documentStore` for mutual exclusion
- Clears file input when URL entered
- URL state managed via Zustand

**With Julia (Testing)**:
- Unit tests for URL validation
- Unit tests for UI store
- E2E test coverage for URL input flow
- Accessibility tests passing

### Files Delivered

```
src/components/upload/
├── UrlInput.tsx
└── UrlInput.test.tsx

src/stores/
├── uiStore.ts
└── uiStore.test.ts
```

### Next Sprint Preview

**Sprint 1.5b** will add:
- Progress UI components (ProgressCard, LoadingStates)
- Real-time progress display with SSE
- Reconnection overlay

---

## Notes

- **Mutual Exclusion**: URL input will integrate with documentStore in Sprint 1.5 to enforce file XOR URL pattern
- **Security**: SSRF validation is server-side; client-side validation is for UX only
- **UX**: Real-time validation provides immediate feedback
- **Accessibility**: Form fully functional without mouse, screen reader compatible
