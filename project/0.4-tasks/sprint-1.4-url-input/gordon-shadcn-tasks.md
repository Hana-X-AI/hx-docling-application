# Tasks: shadcn/ui Components for URL Input (Sprint 1.4)

**Agent**: Gordon Zain (`@gordon`) - shadcn/ui Specialist
**Sprint**: 1.4 - URL Input with Security
**Duration**: ~0.5h
**Prerequisites**: Sprint 1.3 complete, Form components functional

## MCP Integration

**hx-shadcn MCP Server**: Use the MCP server for component reference and validation patterns.

**Service Reference**: `project/0.8-references/hx-shadcn-service-descriptor.md`
**Endpoint**: `http://hx-shadcn.hx.dev.local:7423/sse`

**MCP Tools for This Sprint**:
| Tool | Usage in Sprint 1.4 |
|------|---------------------|
| `get_component` | Retrieve Input, Button, Alert component source |
| `get_component_demo` | View Form validation examples with Input |
| `get_component_metadata` | Review Alert component variants for mutual exclusion feedback |

**Example: Retrieve Alert Component**
```json
{
  "method": "tools/call",
  "params": {
    "name": "get_component",
    "arguments": {"componentName": "alert"}
  }
}
```

**Component Installation Note**:
- Alert component not pre-installed in Sprint 1.1
- Install if needed: `npx shadcn@latest add alert`
- MCP server can provide source preview before installation

## Task List

### GOR-1.4-001: Style UrlInput Component with shadcn/ui Form
**Status**: Pending
**Effort**: 20 minutes
**Dependencies**: Sprint 1.4 Task 1 (UrlInput component created)
**Priority**: P0 - Critical Path

#### Description
Apply shadcn/ui Input, Button, and Form components to the UrlInput with react-hook-form integration and SSRF validation. Ensure consistent styling with UploadZone and proper error display.

#### Acceptance Criteria
- [ ] Input component used for URL entry
- [ ] Button component used for submit and clear actions
- [ ] Form component manages validation state
- [ ] Validation errors display with FormMessage
- [ ] Basic URL validation in Zod schema (server handles SSRF)
- [ ] Clear button appears when URL is entered
- [ ] Loading state during URL validation
- [ ] Dark mode styling works
- [ ] Accessibility: ARIA labels and keyboard navigation

#### Technical Implementation
```tsx
// src/components/upload/UrlInput.tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import {
  Form,
  FormControl,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
  FormDescription,
} from '@/components/ui/form';
import { Input } from '@/components/ui/input';
import { Button } from '@/components/ui/button';
import { X, Link as LinkIcon } from 'lucide-react';
import { cn } from '@/lib/utils';

/**
 * Client-side URL validation schema (basic checks only)
 *
 * IMPORTANT: This schema performs ONLY basic format validation.
 * All SSRF prevention and security blocklist checks are performed
 * server-side by the authoritative validation in:
 *   src/lib/validation/url.ts (see BOB-1.4-003)
 *
 * The server will return E104 errors for blocked URLs.
 * Do NOT duplicate SSRF regex patterns here to avoid divergence.
 */
const urlFormSchema = z.object({
  url: z.string()
    .min(1, { message: 'URL is required' })
    .max(2048, { message: 'URL must be 2048 characters or less' })
    .url({ message: 'Please enter a valid URL' })
    .refine(
      (url) => {
        try {
          const parsed = new URL(url);
          // Basic protocol check - server enforces full SSRF rules
          return ['http:', 'https:'].includes(parsed.protocol);
        } catch {
          return false;
        }
      },
      { message: 'Only HTTP and HTTPS URLs are allowed' }
    ),
});

type UrlFormValues = z.infer<typeof urlFormSchema>;

interface UrlInputProps {
  disabled?: boolean;
  onUrlSubmit: (url: string) => Promise<void>;
}

export function UrlInput({ disabled, onUrlSubmit }: UrlInputProps) {
  const form = useForm<UrlFormValues>({
    resolver: zodResolver(urlFormSchema),
    defaultValues: {
      url: '',
    },
  });

  const onSubmit = async (data: UrlFormValues) => {
    try {
      await onUrlSubmit(data.url);
      form.reset(); // Clear form after successful submission
    } catch (error) {
      // Map API error codes to safe user-facing messages
      const errorMessages: Record<string, string> = {
        E104: 'This URL is not allowed for security reasons',
        E105: 'The URL request timed out. Please try again',
        E106: 'Unable to fetch the document from this URL',
        E107: 'The document format is not supported',
      };

      let userMessage = 'URL processing failed';

      // Handle structured API error envelope: { error: { code, userMessage } }
      if (error && typeof error === 'object') {
        const err = error as Record<string, unknown>;

        // Check for structured error payload
        if (err.error && typeof err.error === 'object') {
          const errorPayload = err.error as { code?: string; userMessage?: string };
          if (errorPayload.userMessage) {
            userMessage = errorPayload.userMessage;
          } else if (errorPayload.code && errorMessages[errorPayload.code]) {
            userMessage = errorMessages[errorPayload.code];
          }
        }
        // Handle Response objects with JSON error body
        else if (err instanceof Response) {
          try {
            const body = await err.json() as { error?: { code?: string; userMessage?: string } };
            if (body.error?.userMessage) {
              userMessage = body.error.userMessage;
            } else if (body.error?.code && errorMessages[body.error.code]) {
              userMessage = errorMessages[body.error.code];
            }
          } catch {
            // JSON parsing failed, use default message
          }
        }
        // Handle Error objects with code property (custom ApiError)
        else if (err instanceof Error && 'code' in err) {
          const code = (err as Error & { code?: string }).code;
          if (code && errorMessages[code]) {
            userMessage = errorMessages[code];
          }
        }
      }

      form.setError('url', { message: userMessage });
    }
  };

  const currentUrl = form.watch('url');

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
        <FormField
          control={form.control}
          name="url"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Document URL</FormLabel>
              <FormDescription>
                Enter a publicly accessible URL to a document
              </FormDescription>
              <FormControl>
                <div className="relative">
                  <LinkIcon className="absolute left-3 top-1/2 h-4 w-4 -translate-y-1/2 text-muted-foreground" />
                  <Input
                    {...field}
                    type="url"
                    placeholder="https://example.com/document.pdf"
                    disabled={disabled || form.formState.isSubmitting}
                    className={cn(
                      'pl-10 pr-10',
                      field.value && 'pr-10' // Make room for clear button
                    )}
                    aria-label="Document URL input"
                  />
                  {field.value && (
                    <Button
                      type="button"
                      variant="ghost"
                      size="icon"
                      onClick={() => field.onChange('')}
                      className="absolute right-1 top-1/2 h-7 w-7 -translate-y-1/2"
                      aria-label="Clear URL"
                    >
                      <X className="h-4 w-4" />
                    </Button>
                  )}
                </div>
              </FormControl>
              <FormMessage /> {/* Displays validation errors */}
            </FormItem>
          )}
        />

        <Button
          type="submit"
          disabled={!currentUrl || form.formState.isSubmitting || disabled}
          className="w-full"
        >
          {form.formState.isSubmitting ? 'Validating...' : 'Process URL'}
        </Button>
      </form>
    </Form>
  );
}
```

**Client-Side Validation** (Zod schema - basic checks only):
| Check | Client Validates | Error Message |
|-------|------------------|---------------|
| Required | Yes | "URL is required" |
| Max length (2048) | Yes | "URL must be 2048 characters or less" |
| Valid URL format | Yes | "Please enter a valid URL" |
| HTTP/HTTPS protocol | Yes | "Only HTTP and HTTPS URLs are allowed" |

**Server-Side SSRF Prevention** (authoritative - see `src/lib/validation/url.ts`):
| Pattern | Blocked | Error Code |
|---------|---------|------------|
| `localhost`, `127.0.0.1`, `[::1]` | Yes | E104 |
| `10.x.x.x` | Yes | E104 |
| `172.16-31.x.x` | Yes | E104 |
| `192.168.x.x` | Yes | E104 |
| `169.254.x.x` (link-local) | Yes | E104 |
| `*.hx.dev.local` | Yes | E104 |

> **Note**: The client schema does NOT enforce SSRF rules. All SSRF/blocklist
> decisions are delegated to the server to avoid rule divergence. The server
> returns E104 errors which are displayed via `form.setError()`.

**Validation Flow**:
1. User enters URL
2. Client Zod validates: required, length, format, protocol
3. Validation passes → Submit button enabled
4. User clicks submit → API call initiated
5. **Server validates SSRF rules** (authoritative)
6. Server returns E104 if URL blocked → Error displayed
7. Success → Form resets
8. Other failure → Error displayed

#### Validation
```bash
# Test client-side validation (basic checks)
# 1. Leave empty → "URL is required"
# 2. Enter very long URL (>2048 chars) → "URL must be 2048 characters or less"
# 3. Enter "not-a-url" → "Please enter a valid URL"
# 4. Enter "ftp://example.com/file" → "Only HTTP and HTTPS URLs are allowed"
# 5. Enter "https://example.com/doc.pdf" → Submit button enabled
# 6. Clear URL → Submit button disabled
# 7. Toggle dark mode → Input styling adapts

# Test server-side SSRF validation (submit required)
# 1. Enter "http://localhost/test" → Client passes, submit → Server returns E104
# 2. Enter "http://192.168.1.1/doc" → Client passes, submit → Server returns E104
# 3. Enter "http://10.0.0.1/internal" → Client passes, submit → Server returns E104
# Note: SSRF errors only appear after form submission (server-side validation)
```

#### Deliverables
- UrlInput component with Form integration
- Zod schema with basic validation (format, length, protocol only)
- Clear button functionality
- Server-side SSRF error handling via `form.setError()`

---

### GOR-1.4-002: Implement Mutual Exclusion UI Feedback
**Status**: Pending
**Effort**: 15 minutes
**Dependencies**: GOR-1.4-001, Sprint 1.4 Task 5 (Mutual exclusion logic)
**Priority**: P1 - High

#### Description
Add visual feedback when mutual exclusion between file and URL inputs is triggered. Show a subtle message and disable the inactive input type when one is selected.

#### Acceptance Criteria
- [ ] File upload zone disabled when URL is entered
- [ ] URL input disabled when file is selected
- [ ] Informational message shows which input is active
- [ ] Clear/remove action re-enables both inputs
- [ ] Visual indicator (opacity, disabled cursor) on inactive input
- [ ] Dark mode styling works
- [ ] Accessibility: Screen reader announces input state changes

#### Technical Implementation
```tsx
// src/components/upload/DocumentInputManager.tsx
import { Card, CardContent } from '@/components/ui/card';
import { Alert, AlertDescription } from '@/components/ui/alert';
import { Info } from 'lucide-react';
import { useDocumentStore } from '@/stores/documentStore';
import { UploadForm } from './UploadForm';
import { UrlInput } from './UrlInput';

export function DocumentInputManager() {
  const { file, url, activeInput, clearInputs } = useDocumentStore();

  return (
    <div className="space-y-6">
      {/* Mutual exclusion alert */}
      {activeInput && (
        <Alert>
          <Info className="h-4 w-4" />
          <AlertDescription>
            {activeInput === 'file' ? (
              <>
                File selected. URL input is disabled.{' '}
                <button
                  type="button"
                  onClick={clearInputs}
                  className="underline hover:no-underline"
                  aria-label="Clear file and enable URL input"
                >
                  Clear file
                </button>{' '}
                to use URL instead.
              </>
            ) : (
              <>
                URL entered. File upload is disabled.{' '}
                <button
                  type="button"
                  onClick={clearInputs}
                  className="underline hover:no-underline"
                  aria-label="Clear URL and enable file upload"
                >
                  Clear URL
                </button>{' '}
                to upload a file instead.
              </>
            )}
          </AlertDescription>
        </Alert>
      )}

      {/* File upload section */}
      <Card className={activeInput === 'url' ? 'opacity-50' : ''}>
        <CardContent className="p-6">
          <h3 className="text-lg font-semibold mb-4">Upload File</h3>
          <UploadForm disabled={activeInput === 'url'} />
        </CardContent>
      </Card>

      {/* Divider */}
      <div className="relative">
        <div className="absolute inset-0 flex items-center">
          <div className="w-full border-t border-border" />
        </div>
        <div className="relative flex justify-center text-xs uppercase">
          <span className="bg-background px-2 text-muted-foreground">Or</span>
        </div>
      </div>

      {/* URL input section */}
      <Card className={activeInput === 'file' ? 'opacity-50' : ''}>
        <CardContent className="p-6">
          <h3 className="text-lg font-semibold mb-4">Process from URL</h3>
          <UrlInput
            disabled={activeInput === 'file'}
            onUrlSubmit={async (url) => {
              // TODO: Sprint 1.4 Task 5 - Integrate with document processing API
              // Example implementation:
              // try {
              //   const response = await processDocumentUrl(url);
              //   if (response.jobId) {
              //     // Navigate to results or update state
              //     router.push(`/results/${response.jobId}`);
              //   }
              // } catch (error) {
              //   // Handle API errors (E104 SSRF, E105 timeout, etc.)
              //   if (error instanceof ApiError) {
              //     form.setError('url', { message: error.message });
              //   }
              // }
            }}
          />
        </CardContent>
      </Card>
    </div>
  );
}
```

**Visual States**:
| State | File Section | URL Section | Alert Message |
|-------|-------------|-------------|---------------|
| Idle | Enabled (opacity 100%) | Enabled (opacity 100%) | None |
| File Selected | Enabled (opacity 100%) | Disabled (opacity 50%) | "File selected. URL input is disabled." |
| URL Entered | Disabled (opacity 50%) | Enabled (opacity 100%) | "URL entered. File upload is disabled." |

**Accessibility Enhancements**:
```tsx
// Screen reader announcement on state change
useEffect(() => {
  if (activeInput) {
    const message = activeInput === 'file'
      ? 'File input active. URL input disabled.'
      : 'URL input active. File upload disabled.';

    const announcement = document.createElement('div');
    announcement.setAttribute('role', 'status');
    announcement.setAttribute('aria-live', 'polite');
    announcement.className = 'sr-only';
    announcement.textContent = message;
    document.body.appendChild(announcement);

    setTimeout(() => document.body.removeChild(announcement), 1000);
  }
}, [activeInput]);
```

#### Validation
```bash
# Test mutual exclusion
# 1. Select file → URL section dims, alert appears
# 2. Click "Clear file" → Both sections enabled, alert disappears
# 3. Enter URL → File section dims, alert appears
# 4. Click "Clear URL" → Both sections enabled
# 5. Screen reader: Announces state changes
```

#### Deliverables
- DocumentInputManager component
- Mutual exclusion visual feedback
- Screen reader announcements

---

### GOR-1.4-003: Add Loading States to URL Processing
**Status**: Pending
**Effort**: 10 minutes
**Dependencies**: GOR-1.4-001
**Priority**: P1 - High

#### Description
Implement loading states for URL validation and processing using shadcn/ui Button and Skeleton components. Provide clear visual feedback during async operations.

#### Acceptance Criteria
- [ ] Submit button shows loading spinner during processing
- [ ] Button text changes to "Validating..." or "Processing..."
- [ ] Input disabled during processing
- [ ] Loading state prevents double submission
- [ ] Error state shows after failed validation
- [ ] Success state clears form
- [ ] Accessibility: Screen reader announces loading state

#### Technical Implementation
```tsx
// Enhanced UrlInput with loading states
import { Loader2 } from 'lucide-react';

// Inside UrlInput component
<Button
  type="submit"
  disabled={!currentUrl || form.formState.isSubmitting || disabled}
  className="w-full"
>
  {form.formState.isSubmitting && (
    <Loader2 className="mr-2 h-4 w-4 animate-spin" aria-hidden="true" />
  )}
  {form.formState.isSubmitting ? 'Processing URL...' : 'Process URL'}
</Button>

// Add screen reader announcement
<span className="sr-only" aria-live="polite" aria-atomic="true">
  {form.formState.isSubmitting ? 'Processing URL, please wait' : ''}
</span>
```

**Loading State Sequence**:
1. User clicks "Process URL"
2. Button disabled, text changes to "Processing URL..."
3. Spinner icon appears (rotating animation)
4. Input field disabled
5. On success: Form resets, button returns to idle
6. On error: Error message shown, button returns to idle

#### Validation
```bash
# Test loading states
# 1. Enter valid URL, click submit
# 2. Observe: Button shows spinner + "Processing URL..."
# 3. Observe: Input disabled during processing
# 4. Simulate slow network (DevTools → Network → Slow 3G)
# 5. Verify: Loading state persists until response
```

#### Deliverables
- Loading spinner in Button
- Disabled input during processing
- Screen reader announcements

---

## Summary

**Total Tasks**: 3
**Total Effort**: 0.75 hours
**Critical Path**: GOR-1.4-001 → GOR-1.4-002

**Success Criteria**:
- [ ] UrlInput styled with shadcn/ui components
- [ ] Basic URL validation in Zod (format, length, protocol)
- [ ] SSRF validation delegated to server (BOB-1.4-003)
- [ ] Mutual exclusion visual feedback
- [ ] Loading states during URL processing
- [ ] Zero TypeScript errors
- [ ] WCAG 2.1 AA accessibility compliance

**Coordination Points**:
- **Neo (Sprint 1.4 Lead)**: Coordinate URL validation logic (Task 2)
- **Bob Martin (API)**: Server owns SSRF validation (BOB-1.4-003); client delegates
- **Ola (Accessibility)**: Review screen reader announcements

**Anti-Patterns Prevented**:
- ✅ Client Zod schema for basic validation only (SSRF delegated to server)
- ✅ No duplicated SSRF regex patterns (single source of truth in `src/lib/validation/url.ts`)
- ✅ Named imports for lucide-react icons
- ✅ CSS variables for colors
- ✅ Proper loading state management (no manual spinners)
- ✅ FormMessage for error display (consistent UX)

**Notes**:
- Client performs basic URL validation only; SSRF enforcement is server-side (see BOB-1.4-003)
- Server-side SSRF validation is authoritative source in `src/lib/validation/url.ts`
- Mutual exclusion managed by Zustand store (Sprint 1.4 Task 4)
- Loading states use shadcn/ui Button variants
- Accessibility features ensure screen reader compatibility
