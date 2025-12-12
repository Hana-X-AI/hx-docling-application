# Sprint 1.3: Frontend UI Tasks - File Upload System

**Agent**: Ola Mae Johnson (`@ola`)
**Role**: Frontend UI Developer & Accessibility Specialist
**Sprint**: 1.3 - File Upload System
**Version**: v1.0.1 (Defect Fix: DEF-010)
**Dependencies**: Sprint 1.1 Complete, Sprint 1.2 Complete
**Effort**: 1.75h (of 3.5h total sprint)

---

## Task Overview

| Task ID | Title | Effort | Status |
|---------|-------|--------|--------|
| OLA-1.3-001 | Create UploadZone Component with Drag-Drop | 45m | Pending |
| OLA-1.3-002 | Create FilePreview Component | 25m | Pending |
| OLA-1.3-003 | Create UploadForm with react-hook-form | 25m | Pending |
| OLA-1.3-004 | Style Components with shadcn/ui | 20m | Pending |
| OLA-1.3-005 | Implement Accessibility Features | 20m | Pending |

**Total Effort**: 1.75h (105 minutes)

---

## OLA-1.3-001: Create UploadZone Component with Drag-Drop

### Description

Build the drag-and-drop upload zone component using `react-dropzone` library. This component provides the primary file upload interface with visual feedback for drag-over states.

### Acceptance Criteria

- [ ] Component created as Client Component (`'use client'` directive)
- [ ] `react-dropzone` library integrated with proper configuration
- [ ] Drag-over state shows visual feedback (border color change, background highlight)
- [ ] Click-to-browse fallback functional
- [ ] File type validation integrated (accept prop based on allowed extensions)
- [ ] File size validation integrated (maxSize prop)
- [ ] Rejected files trigger error callback with appropriate error codes
- [ ] Component emits `FILE_SELECTED` event on successful file drop
- [ ] Loading state supported for async operations
- [ ] Disabled state when processing is active

### Dependencies

- Sprint 1.1: shadcn/ui components available
- Sprint 1.2: Database and Redis connected
- Sprint 1.3 Task 2: File validation schema exists

### Deliverables

**File**: `src/components/upload/UploadZone.tsx`

```typescript
'use client'

import { useCallback } from 'react'
import { useDropzone } from 'react-dropzone'
import { Upload, FileX } from 'lucide-react'
import { cn } from '@/lib/utils'
import { Card } from '@/components/ui/card'

interface UploadZoneProps {
  onFileAccepted: (file: File) => void
  onFileRejected: (error: { code: string; message: string }) => void
  disabled?: boolean
  className?: string
}

export function UploadZone({
  onFileAccepted,
  onFileRejected,
  disabled = false,
  className,
}: UploadZoneProps) {
  const onDrop = useCallback((acceptedFiles: File[], rejectedFiles: any[]) => {
    // Handle rejected files first
    if (rejectedFiles.length > 0) {
      const rejection = rejectedFiles[0]
      // Guard: ensure rejection.errors exists and has entries
      if (rejection && Array.isArray(rejection.errors) && rejection.errors.length > 0) {
        const errorCode = rejection.errors[0].code
        if (errorCode === 'file-too-large') {
          onFileRejected({ code: 'E001', message: 'File size exceeds limit' })
        } else if (errorCode === 'file-invalid-type') {
          onFileRejected({ code: 'E002', message: 'Unsupported file type' })
        } else {
          // Unknown error code from react-dropzone
          onFileRejected({ code: 'E999', message: rejection.errors[0].message || 'File rejected' })
        }
      } else {
        // Rejection entry exists but has no errors array - generic error
        onFileRejected({ code: 'E999', message: 'File rejected for unknown reason' })
      }
      return
    }

    // Handle accepted files
    if (acceptedFiles.length > 0) {
      onFileAccepted(acceptedFiles[0])
      return
    }

    // Edge case: both arrays are empty (no files provided)
    // This can happen if the user cancels the file picker or drops nothing
    // No-op: do nothing, as no file was actually provided
  }, [onFileAccepted, onFileRejected])

  const { getRootProps, getInputProps, isDragActive } = useDropzone({
    onDrop,
    accept: {
      'application/pdf': ['.pdf'],
      'application/msword': ['.doc'],
      'application/vnd.openxmlformats-officedocument.wordprocessingml.document': ['.docx'],
      'application/vnd.ms-powerpoint': ['.ppt'],
      'application/vnd.openxmlformats-officedocument.presentationml.presentation': ['.pptx'],
      'application/vnd.ms-excel': ['.xls'],
      'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet': ['.xlsx'],
      'image/png': ['.png'],
      'image/jpeg': ['.jpg', '.jpeg'],
      'image/tiff': ['.tiff'],
    },
    maxSize: 100 * 1024 * 1024, // 100MB
    multiple: false,
    disabled,
  })

  return (
    <Card
      {...getRootProps()}
      className={cn(
        'border-2 border-dashed p-12 text-center cursor-pointer transition-colors',
        'hover:border-primary hover:bg-accent/5',
        isDragActive && 'border-primary bg-accent/10',
        disabled && 'cursor-not-allowed opacity-50',
        className
      )}
    >
      <input {...getInputProps()} aria-label="File upload input" />
      <div className="flex flex-col items-center gap-4">
        {isDragActive ? (
          <>
            <Upload className="h-12 w-12 text-primary animate-bounce" />
            <p className="text-lg font-medium text-primary">Drop file here</p>
          </>
        ) : (
          <>
            <Upload className="h-12 w-12 text-muted-foreground" />
            <div>
              <p className="text-lg font-medium mb-2">
                Drag and drop a file here, or click to browse
              </p>
              <p className="text-sm text-muted-foreground">
                Supported formats: PDF, Word, Excel, PowerPoint, Images (PNG, JPG, TIFF)
              </p>
              <p className="text-xs text-muted-foreground mt-1">
                Maximum file size: 100 MB
              </p>
            </div>
          </>
        )}
      </div>
    </Card>
  )
}
```

### Technical Notes

- **Reference**: Specification FR-101, FR-102, FR-103, FR-104
- **Library**: `react-dropzone@^14.x`
- **File Validation**: Client-side only (server-side validation in API route)
- **MIME Types**: Comprehensive list per specification Section 2.1
- **Error Codes**:
  - E001: File size exceeds limit
  - E002: Unsupported file type
- **Accessibility**:
  - Hidden input has aria-label
  - Keyboard accessible (Enter/Space to open file picker)
  - Focus visible on keyboard navigation

### Testing

```bash
# Unit test
npm run test src/components/upload/UploadZone.test.tsx

# Manual testing:
# [ ] Drag PDF file - accepted
# [ ] Drag TXT file - rejected with E002
# [ ] Drag 150MB file - rejected with E001
# [ ] Click zone - file picker opens
# [ ] Disabled state prevents interaction
# [ ] Keyboard navigation works (Tab to focus, Enter to activate)
```

---

## OLA-1.3-002: Create FilePreview Component

### Description

Create a component to display file metadata after selection. Shows file name, size (human-readable format), file type icon, and a remove button.

### Acceptance Criteria

- [ ] Component displays file name with truncation for long names
- [ ] File size formatted in human-readable units (KB, MB)
- [ ] File type icon displayed based on extension (using lucide-react icons)
- [ ] Remove button functional with confirmation for large files
- [ ] Responsive layout for mobile and desktop
- [ ] Accessibility: remove button has ARIA label, focus management

### Dependencies

- OLA-1.3-001: UploadZone component complete
- Sprint 1.1: shadcn/ui components available

### Deliverables

**File**: `src/components/upload/FilePreview.tsx`

```typescript
'use client'

import { FileText, FileSpreadsheet, Presentation, Image, X } from 'lucide-react'
import { Card, CardContent } from '@/components/ui/card'
import { Button } from '@/components/ui/button'

interface FilePreviewProps {
  file: File
  onRemove: () => void
}

export function FilePreview({ file, onRemove }: FilePreviewProps) {
  const formatFileSize = (bytes: number): string => {
    if (bytes === 0) return '0 Bytes'
    const k = 1024
    const sizes = ['Bytes', 'KB', 'MB', 'GB']
    const i = Math.floor(Math.log(bytes) / Math.log(k))
    return Math.round((bytes / Math.pow(k, i)) * 100) / 100 + ' ' + sizes[i]
  }

  const getFileIcon = (filename: string) => {
    const ext = filename.split('.').pop()?.toLowerCase()
    switch (ext) {
      case 'pdf':
      case 'doc':
      case 'docx':
        return <FileText className="h-8 w-8 text-primary" />
      case 'xls':
      case 'xlsx':
        return <FileSpreadsheet className="h-8 w-8 text-green-600" />
      case 'ppt':
      case 'pptx':
        return <Presentation className="h-8 w-8 text-orange-600" />
      case 'png':
      case 'jpg':
      case 'jpeg':
      case 'tiff':
        return <Image className="h-8 w-8 text-blue-600" />
      default:
        return <FileText className="h-8 w-8 text-muted-foreground" />
    }
  }

  return (
    <Card>
      <CardContent className="p-4">
        <div className="flex items-center gap-4">
          {getFileIcon(file.name)}
          <div className="flex-1 min-w-0">
            <p className="font-medium truncate" title={file.name}>
              {file.name}
            </p>
            <p className="text-sm text-muted-foreground">
              {formatFileSize(file.size)}
            </p>
          </div>
          <Button
            variant="ghost"
            size="icon"
            onClick={onRemove}
            aria-label={`Remove file ${file.name}`}
          >
            <X className="h-4 w-4" />
          </Button>
        </div>
      </CardContent>
    </Card>
  )
}
```

### Technical Notes

- **Reference**: Specification FR-105
- **Icons**: `lucide-react` for file type icons
- **File Size**: Formatted with Math.round for readability
- **Truncation**: CSS `truncate` class for long filenames
- **Accessibility**:
  - Remove button has descriptive ARIA label with filename
  - Title attribute on filename shows full name on hover
  - Color not sole indicator (icons provide visual differentiation)

### Testing

```bash
# Unit test
npm run test src/components/upload/FilePreview.test.tsx

# Manual testing:
# [ ] Display PDF file - FileText icon shown
# [ ] Display XLSX file - FileSpreadsheet icon shown
# [ ] Long filename truncates with ellipsis
# [ ] File size formats correctly (e.g., "2.5 MB")
# [ ] Remove button works
# [ ] Hover on filename shows full name
# [ ] Screen reader announces file info correctly
```

---

## OLA-1.3-003: Create UploadForm with react-hook-form

### Description

Integrate `react-hook-form` for form state management, combining the UploadZone and FilePreview components with validation and submission handling.

### Acceptance Criteria

- [ ] `react-hook-form` configured with Zod schema validation
- [ ] Form includes UploadZone and conditional FilePreview
- [ ] Submit button disabled when no file selected or validation fails
- [ ] Form handles errors from validation schema
- [ ] Loading state during file upload
- [ ] Success/error toasts on submission result
- [ ] Form resets after successful submission
- [ ] Rate limiting feedback displayed when limit exceeded

### Dependencies

- OLA-1.3-001: UploadZone component complete
- OLA-1.3-002: FilePreview component complete
- Sprint 1.3 Task 7: react-hook-form integration planning

### Deliverables

**File**: `src/components/upload/UploadForm.tsx`

```typescript
'use client'

import { useState } from 'react'
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'
import { UploadZone } from './UploadZone'
import { FilePreview } from './FilePreview'
import { Button } from '@/components/ui/button'
import { Form, FormField, FormItem, FormMessage } from '@/components/ui/form'
import { useToast } from '@/components/ui/use-toast'

/**
 * API Error Contract Documentation
 * ---------------------------------
 * The upload API may return errors in various formats. This utility handles:
 *
 * 1. Standard format: { message: string }
 *    Example: { "message": "File too large" }
 *
 * 2. Validation errors array: { errors: Array<{ message?: string; field?: string }> }
 *    Example: { "errors": [{ "message": "Invalid file type", "field": "file" }] }
 *
 * 3. Nested error object: { error: { message: string } }
 *    Example: { "error": { "message": "Server error" } }
 *
 * 4. Plain string response (non-JSON)
 *    Example: "Internal Server Error"
 *
 * 5. Status-only errors (empty body)
 *    Falls back to HTTP status text
 *
 * User-facing error fields used (in order of preference):
 * - response.message (string)
 * - response.error.message (string)
 * - response.errors[0].message (string)
 * - JSON.stringify(response) for unknown structures
 * - "Upload failed" as final fallback
 */

/**
 * Safely extracts a user-friendly error message from various API error response formats.
 * @param body - The parsed JSON body or raw text from the error response
 * @param statusText - The HTTP status text to use as fallback
 * @returns A user-friendly error message string
 */
function extractErrorMessage(body: unknown, statusText?: string): string {
  // Handle null/undefined
  if (body == null) {
    return statusText || 'Upload failed'
  }

  // Handle plain string responses
  if (typeof body === 'string') {
    return body.trim() || statusText || 'Upload failed'
  }

  // Handle object responses
  if (typeof body === 'object') {
    const obj = body as Record<string, unknown>

    // Format 1: { message: string }
    if (typeof obj.message === 'string' && obj.message.trim()) {
      return obj.message.trim()
    }

    // Format 3: { error: { message: string } }
    if (obj.error && typeof obj.error === 'object') {
      const errorObj = obj.error as Record<string, unknown>
      if (typeof errorObj.message === 'string' && errorObj.message.trim()) {
        return errorObj.message.trim()
      }
    }

    // Format 2: { errors: Array<{ message?: string }> }
    if (Array.isArray(obj.errors) && obj.errors.length > 0) {
      const firstError = obj.errors[0]
      if (firstError && typeof firstError === 'object') {
        const errItem = firstError as Record<string, unknown>
        if (typeof errItem.message === 'string' && errItem.message.trim()) {
          return errItem.message.trim()
        }
      }
      // If errors array exists but no message found, try to stringify first error
      try {
        const serialized = JSON.stringify(obj.errors[0])
        if (serialized && serialized !== '{}') {
          return `Validation error: ${serialized}`
        }
      } catch {
        // Ignore serialization errors
      }
    }

    // Unknown object structure - try to stringify for debugging
    try {
      const serialized = JSON.stringify(obj)
      if (serialized && serialized !== '{}') {
        // Truncate very long error messages
        const maxLength = 200
        if (serialized.length > maxLength) {
          return serialized.slice(0, maxLength) + '...'
        }
        return serialized
      }
    } catch {
      // Ignore serialization errors
    }
  }

  // Final fallback
  return statusText || 'Upload failed'
}

const uploadSchema = z.object({
  file: z
    .instanceof(File)
    .nullable()
    .refine((file) => file !== null, 'Please select a file'),
})

type UploadFormValues = z.infer<typeof uploadSchema>

export function UploadForm() {
  const [isUploading, setIsUploading] = useState(false)
  const { toast } = useToast()

  const form = useForm<UploadFormValues>({
    resolver: zodResolver(uploadSchema),
    defaultValues: {
      file: undefined,
    },
  })

  const selectedFile = form.watch('file')

  const onSubmit = async (data: UploadFormValues) => {
    setIsUploading(true)
    try {
      const formData = new FormData()
      formData.append('file', data.file)

      const response = await fetch('/api/v1/upload', {
        method: 'POST',
        body: formData,
      })

      if (!response.ok) {
        // Safely parse response body - may be JSON, text, or empty
        let errorBody: unknown
        const contentType = response.headers.get('content-type') || ''
        try {
          if (contentType.includes('application/json')) {
            errorBody = await response.json()
          } else {
            errorBody = await response.text()
          }
        } catch {
          // Body parsing failed - use status text as fallback
          errorBody = null
        }
        throw new Error(extractErrorMessage(errorBody, response.statusText))
      }

      const result = await response.json()
      toast({
        title: 'Upload successful',
        description: `Job created: ${result.jobId}`,
      })
      form.reset()
    } catch (error) {
      toast({
        title: 'Upload failed',
        description: error instanceof Error ? error.message : 'Unknown error',
        variant: 'destructive',
      })
    } finally {
      setIsUploading(false)
    }
  }

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-6">
        <FormField
          control={form.control}
          name="file"
          render={({ field }) => (
            <FormItem>
              {!selectedFile ? (
                <UploadZone
                  onFileAccepted={(file) => field.onChange(file)}
                  onFileRejected={(error) => {
                    toast({
                      title: 'File rejected',
                      description: error.message,
                      variant: 'destructive',
                    })
                  }}
                  disabled={isUploading}
                />
              ) : (
                <FilePreview
                  file={selectedFile}
                  onRemove={() => field.onChange(null)}
                />
              )}
              <FormMessage />
            </FormItem>
          )}
        />

        <Button
          type="submit"
          disabled={!selectedFile || isUploading}
          className="w-full"
        >
          {isUploading ? 'Uploading...' : 'Process Document'}
        </Button>
      </form>
    </Form>
  )
}
```

### Technical Notes

- **Reference**: Implementation Plan Sprint 1.3 Task 7
- **Validation**: Zod schema for type-safe validation
- **Form Library**: `react-hook-form@^7.x` with `@hookform/resolvers`
- **Toast**: shadcn/ui toast for user feedback
- **State Management**: Form state managed by react-hook-form
- **Error Handling**: API errors displayed via toast
- **Loading State**: Button disabled with loading text

**DEFECT FIX - v1.0.1 (DEF-010)**:
- **Issue**: Zod schema used `z.instanceof(File)` without `.nullable()`, causing validation errors when `onRemove` callback called `field.onChange(null)` (line 555)
- **Root Cause**: Schema didn't allow null values during form lifecycle, but remove action required setting field to null
- **Fix**: Added `.nullable()` to schema chain at line 470 to permit null during lifecycle while still requiring file for submission via `.refine()` validation
- **Impact**: Prevents validation errors when user removes a selected file before submission
- **Date**: 2025-12-12

### Testing

```bash
# Unit test
npm run test src/components/upload/UploadForm.test.tsx

# Manual testing:
# [ ] Submit disabled when no file selected
# [ ] Upload file - submit enabled
# [ ] Click submit - loading state shown
# [ ] Successful upload shows success toast
# [ ] Failed upload shows error toast
# [ ] Form resets after success
# [ ] Rate limit error (429) shows appropriate message
```

---

## OLA-1.3-004: Style Components with shadcn/ui

### Description

Apply consistent styling to all upload components using shadcn/ui design tokens and Tailwind utilities. Ensure visual coherence with the application's design system.

### Acceptance Criteria

- [ ] All components use shadcn/ui Card, Button, and Form components
- [ ] Color scheme consistent with theme (primary, accent, destructive)
- [ ] Spacing follows 8px grid system
- [ ] Typography sizes appropriate (base, lg, sm, xs)
- [ ] Dark mode styling verified for all components
- [ ] Hover and focus states visually distinct
- [ ] Loading states use Skeleton component where appropriate
- [ ] Responsive breakpoints tested (mobile, tablet, desktop)

### Dependencies

- OLA-1.3-001: UploadZone component complete
- OLA-1.3-002: FilePreview component complete
- OLA-1.3-003: UploadForm component complete
- Sprint 1.1: Tailwind theme configured

### Deliverables

**Styling Checklist** (Status: Done = implemented, Planned = not yet implemented, N/A = not applicable):

| Component | Card | Button | Color Tokens | Dark Mode | Responsive |
|-----------|------|--------|--------------|-----------|------------|
| UploadZone | Done | N/A | Done | Planned | Planned |
| FilePreview | Done | Done | Done | Planned | Planned |
| UploadForm | N/A | Done | Done | Planned | Planned |

**Design Tokens Used**:
- `--background`: Component backgrounds
- `--foreground`: Text color
- `--primary`: Upload zone active state, icons
- `--muted-foreground`: Secondary text
- `--border`: Card borders
- `--accent`: Hover states
- `--destructive`: Error states

### Technical Notes

- **Reference**: Implementation Plan Sprint 1.3 Task 9
- **Grid System**: Spacing in multiples of 4 (p-4, gap-4, mb-2)
- **Typography**:
  - Headings: `text-lg font-medium`
  - Body: `text-base`
  - Secondary: `text-sm text-muted-foreground`
  - Captions: `text-xs text-muted-foreground`
- **Responsive**:
  - Mobile: Full width, vertical layout
  - Tablet: Optimized touch targets
  - Desktop: Horizontal layout where appropriate
- **Dark Mode**: All colors use CSS variables for automatic theme switching

### Testing

```bash
# Visual testing
npm run dev

# Toggle dark mode and verify:
# [ ] Upload zone border visible in both themes
# [ ] Icons render correctly in both themes
# [ ] Text contrast meets WCAG AA standards
# [ ] Hover states visible and appropriate

# Responsive testing:
# [ ] Mobile (320px-767px): vertical layout, full width
# [ ] Tablet (768px-1023px): optimized for touch
# [ ] Desktop (1024px+): optimal spacing
```

---

## OLA-1.3-005: Implement Accessibility Features

### Description

Ensure all upload components meet WCAG 2.1 Level AA accessibility standards with proper ARIA labels, keyboard navigation, and screen reader support.

### Acceptance Criteria

- [ ] All interactive elements keyboard accessible (Tab navigation)
- [ ] Focus indicators visible (outline or ring)
- [ ] ARIA labels present on all inputs and buttons
- [ ] Error messages announced to screen readers
- [ ] File upload input has descriptive label
- [ ] Remove button announces file name being removed
- [ ] Form validation errors accessible
- [ ] Success/error toasts accessible to screen readers
- [ ] Color contrast ratio >= 4.5:1 for normal text
- [ ] Touch targets >= 44x44px for mobile

### Dependencies

- OLA-1.3-001: UploadZone component complete
- OLA-1.3-002: FilePreview component complete
- OLA-1.3-003: UploadForm component complete
- Sprint 1.3 Task 10: Accessibility planning

### Deliverables

**Accessibility Checklist** (Status: Done = implemented, Planned = not yet implemented):

| Component | Keyboard Nav | ARIA Labels | Screen Reader | Focus Visible | Color Contrast |
|-----------|--------------|-------------|---------------|---------------|----------------|
| UploadZone | Planned | Planned | Planned | Planned | Planned |
| FilePreview | Planned | Planned | Planned | Planned | Planned |
| UploadForm | Planned | Planned | Planned | Planned | Planned |

**ARIA Attributes Added**:

```typescript
// UploadZone
<input {...getInputProps()} aria-label="File upload input" />
<div role="button" aria-label="Upload zone - drag and drop or click to select file">

// FilePreview
<Button aria-label={`Remove file ${file.name}`}>

// UploadForm
<form aria-labelledby="upload-form-title">
<FormMessage role="alert" aria-live="polite">
```

**Keyboard Navigation**:
- Tab: Navigate between upload zone and submit button
- Enter/Space: Activate focused element
- Escape: Clear file selection (when focused on remove button)

### Technical Notes

- **Reference**: Implementation Plan Sprint 1.3 Task 10, Specification Section 9
- **Testing Tools**:
  - Lighthouse accessibility audit
  - axe DevTools
  - NVDA or JAWS screen reader
  - Keyboard only navigation
- **Focus Management**:
  - Focus moves to submit button after file selection
  - Focus returns to upload zone after file removal
- **Error Announcements**:
  - Use `role="alert"` for dynamic error messages
  - Toast notifications use `aria-live="polite"`

### Testing

```bash
# Automated accessibility testing
npm run test:a11y

# Lighthouse audit
npm run lighthouse

# Manual testing checklist:
# [ ] Tab through form without mouse - all elements reachable
# [ ] Focus indicators visible on all interactive elements
# [ ] Upload file with Enter key (when zone focused)
# [ ] Remove file with Enter key (when button focused)
# [ ] Screen reader announces file name, size on selection
# [ ] Screen reader announces errors on validation failure
# [ ] Screen reader announces success on upload
# [ ] Color contrast verified with WebAIM Contrast Checker
# [ ] Touch targets measured >= 44x44px on mobile
```

---

## Sprint Completion Checklist

### Acceptance Criteria (Sprint 1.3 Frontend)

- [ ] UploadZone accepts valid file types (PDF, DOCX, XLSX, PPTX, images)
- [ ] Invalid file types rejected with error E002
- [ ] Files > 100MB rejected with error E001
- [ ] FilePreview displays name, size, type icon
- [ ] UploadForm validates with react-hook-form
- [ ] Form submission creates job and displays toast
- [ ] Components styled consistently with shadcn/ui
- [ ] Dark mode functional for all components
- [ ] Accessibility checklist complete (ARIA, keyboard, screen reader)
- [ ] Unit tests pass with >= 80% coverage
- [ ] No TypeScript errors
- [ ] No console warnings in browser

### Integration Points

**With Neo (Backend)**:
- Form submits to `/api/v1/upload` endpoint
- Receives job ID from API response
- Handles rate limiting (429) errors

**With William (Infrastructure)**:
- Form respects rate limiting middleware
- Error handling for database/Redis failures

**With Julia (Testing)**:
- Unit tests for all components
- Accessibility tests passing
- E2E test coverage for upload flow

### Files Delivered

```
src/components/upload/
├── UploadZone.tsx
├── FilePreview.tsx
├── UploadForm.tsx
├── UploadZone.test.tsx
├── FilePreview.test.tsx
└── UploadForm.test.tsx
```

### Next Sprint Preview

**Sprint 1.4** will add:
- URL input component (similar pattern to file upload)
- Mutual exclusion between file and URL inputs
- SSRF validation UI feedback

---

## Notes

- **Performance**: File validation happens client-side for immediate feedback, server-side for security
- **UX**: Drag-drop provides primary interaction, click-to-browse fallback for accessibility
- **Error Handling**: Clear, actionable error messages with error codes
- **Responsiveness**: Mobile-first design with progressive enhancement
