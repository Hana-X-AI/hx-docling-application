# Neo Next.js Tasks: Sprint 1.4 - URL Input with Security

**Sprint**: 1.4 - URL Input with Security
**Duration**: ~2.0 hours
**Role**: Lead Developer
**Agent**: Neo (Next.js Senior Developer)
**Support**: Ola (@ola)
**Review**: Julia (@julia)

---

## Development Tools

### shadcn/ui MCP Server Reference

For component discovery and retrieval during UI development, use the **hx-shadcn MCP Server**:

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
- `get_component_demo` - Get usage examples and demo code for components like Input, Button, Form
- `get_component_metadata` - Get props, installation instructions, and documentation

**Example - Get Input Component Demo:**
```json
{
  "method": "tools/call",
  "params": {
    "name": "get_component_demo",
    "arguments": {"componentName": "input"}
  }
}
```

**Sprint 1.4 Component Usage:** This sprint uses Input, Button, Form, and toast (Sonner) components for the URL input interface. Use `get_component_demo` to retrieve usage patterns and styling conventions.

---

## Overview

Implement URL input component with SSRF prevention, URL validation, Zustand state management for document store, and mutual exclusion logic between file and URL inputs. This sprint completes the input layer for document processing.

---

## Tasks

### NEO-1.4-001: Create URL Input Component with react-hook-form

**Priority**: P0 (Critical)
**Effort**: 25 minutes
**Dependencies**: Sprint 1.3 Complete
**Reference**: FR-201

**Description**:
Implement URL input component using react-hook-form for form state management. Include input field, paste from clipboard support, clear button, and submit functionality.

**Acceptance Criteria**:
- [ ] Input accepts URL strings
- [ ] Paste from clipboard supported (Ctrl+V / Cmd+V)
- [ ] Clear button provided to reset input
- [ ] Submit button triggers processing
- [ ] Loading state during submission
- [ ] Error display for validation failures
- [ ] 'use client' directive present
- [ ] Accessible with proper ARIA labels

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/components/upload/UrlInput.tsx`

**Technical Notes**:
```typescript
'use client';

import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { useState } from 'react';
import { Link, X, Clipboard, Loader2 } from 'lucide-react';
import { Input } from '@/components/ui/input';
import { Button } from '@/components/ui/button';
import { Form, FormField, FormItem, FormLabel, FormMessage } from '@/components/ui/form';
import { urlValidationSchema } from '@/lib/validation/url';
import { useDocumentStore } from '@/stores/documentStore';

const formSchema = z.object({
  url: urlValidationSchema,
});

type FormValues = z.infer<typeof formSchema>;

export function UrlInput() {
  const [isSubmitting, setIsSubmitting] = useState(false);
  const { setUrl, isProcessing } = useDocumentStore();

  const form = useForm<FormValues>({
    resolver: zodResolver(formSchema),
    defaultValues: { url: '' },
  });

  const handlePaste = async () => {
    try {
      const text = await navigator.clipboard.readText();
      form.setValue('url', text.trim());
    } catch (error) {
      console.error('Failed to read clipboard:', error);
    }
  };

  const handleClear = () => {
    form.reset();
  };

  const onSubmit = async (data: FormValues) => {
    setIsSubmitting(true);
    try {
      const response = await fetch('/api/v1/upload', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ url: data.url }),
      });

      if (!response.ok) {
        const error = await response.json();
        throw new Error(error.error?.message || 'Failed to process URL');
      }

      const result = await response.json();
      setUrl(data.url, result.jobId);
      form.reset();
    } catch (error) {
      form.setError('url', {
        message: error instanceof Error ? error.message : 'Failed to process URL',
      });
    } finally {
      setIsSubmitting(false);
    }
  };

  const currentUrl = form.watch('url');
  const isDisabled = isProcessing || isSubmitting;

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
        <FormField
          control={form.control}
          name="url"
          render={({ field }) => (
            <FormItem>
              <FormLabel>URL</FormLabel>
              <div className="flex gap-2">
                <div className="relative flex-1">
                  <Link className="absolute left-3 top-1/2 -translate-y-1/2 w-4 h-4 text-muted-foreground" />
                  <Input
                    {...field}
                    type="url"
                    placeholder="https://example.com/document"
                    className="pl-10 pr-10"
                    disabled={isDisabled}
                    aria-label="Enter URL to process"
                  />
                  {currentUrl && (
                    <Button
                      type="button"
                      variant="ghost"
                      size="icon"
                      className="absolute right-1 top-1/2 -translate-y-1/2 h-7 w-7"
                      onClick={handleClear}
                      disabled={isDisabled}
                      aria-label="Clear URL"
                    >
                      <X className="w-4 h-4" />
                    </Button>
                  )}
                </div>
                <Button
                  type="button"
                  variant="outline"
                  size="icon"
                  onClick={handlePaste}
                  disabled={isDisabled}
                  aria-label="Paste URL from clipboard"
                >
                  <Clipboard className="w-4 h-4" />
                </Button>
              </div>
              <FormMessage />
            </FormItem>
          )}
        />
        <Button
          type="submit"
          disabled={!currentUrl || isDisabled}
          className="w-full"
        >
          {isSubmitting ? (
            <>
              <Loader2 className="w-4 h-4 mr-2 animate-spin" />
              Processing...
            </>
          ) : (
            'Process URL'
          )}
        </Button>
      </form>
    </Form>
  );
}
```

---

### NEO-1.4-002: Implement SSRF Prevention

**Priority**: P0 (Security Critical)
**Effort**: 20 minutes
**Dependencies**: BOB-1.4-005 (Shared SSRF Config)
**Reference**: FR-203

**Description**:
Implement comprehensive SSRF (Server-Side Request Forgery) prevention by blocking requests to private/internal addresses including localhost, RFC 1918 private ranges, link-local addresses, and internal HX domains.

**Acceptance Criteria**:
- [ ] Block localhost/127.0.0.1
- [ ] Block 10.0.0.0/8 (Class A private)
- [ ] Block 172.16.0.0/12 (Class B private)
- [ ] Block 192.168.0.0/16 (Class C private)
- [ ] Block 169.254.0.0/16 (Link-local)
- [ ] Block *.hx.dev.local domains
- [ ] Block IPv6 localhost (::1)
- [ ] Return error E104 for blocked URLs
- [ ] Validation is server-side enforced

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/lib/validation/url.ts`

**Technical Notes**:

**IMPORTANT**: This task now imports from the shared SSRF config (BOB-1.4-005) instead of defining rules inline. This ensures consistent validation between API and client layers, eliminating the security risk of rule divergence.

```typescript
// File: src/lib/validation/url.ts

import { z } from 'zod';
import { checkSSRFRules } from './ssrf-config';

// REMOVED: Duplicated BLOCKED_HOSTNAMES, isPrivateIP(), isBlockedDomain()
// These are now centralized in ssrf-config.ts (BOB-1.4-005)

/**
 * Check if a URL is blocked by SSRF prevention rules.
 * Delegates to shared checkSSRFRules for consistent validation.
 */
export function isURLBlocked(url: string): { blocked: boolean; reason?: string } {
  try {
    const parsed = new URL(url);
    const hostname = parsed.hostname.toLowerCase();
    return checkSSRFRules(hostname);
  } catch {
    return { blocked: true, reason: 'Invalid URL format' };
  }
}

export const urlValidationSchema = z
  .string()
  .min(1, 'URL is required')
  .max(2048, 'URL must be less than 2048 characters')
  .url('Please enter a valid URL')
  .refine(
    (url) => {
      try {
        const parsed = new URL(url);
        return ['http:', 'https:'].includes(parsed.protocol);
      } catch {
        return false;
      }
    },
    { message: 'Only HTTP and HTTPS URLs are allowed' }
  )
  .refine(
    (url) => !isURLBlocked(url).blocked,
    (url) => ({
      message: isURLBlocked(url).reason || 'URL is blocked',
    })
  );

export type URLValidationResult = {
  valid: boolean;
  errors: Array<{ code: string; message: string }>;
};

export function validateURL(url: string): URLValidationResult {
  const result = urlValidationSchema.safeParse(url);

  if (result.success) {
    return { valid: true, errors: [] };
  }

  const errors = result.error.errors.map((err) => {
    // Map to error codes
    const message = err.message;
    let code = 'E101'; // Default: invalid URL

    if (message.includes('blocked') || message.includes('not allowed')) {
      code = 'E104'; // SSRF blocked
    } else if (message.includes('HTTP')) {
      code = 'E102'; // Protocol not allowed
    } else if (message.includes('2048')) {
      code = 'E103'; // URL too long
    }

    return { code, message };
  });

  return { valid: false, errors };
}
```

---

### NEO-1.4-003: Add URL Format Validation

**Priority**: P0 (Critical)
**Effort**: 15 minutes
**Dependencies**: NEO-1.4-002
**Reference**: FR-202

**Description**:
Create comprehensive URL format validation using Zod schema. Enforce HTTP/HTTPS only and maximum URL length.

**Acceptance Criteria**:
- [ ] Accept http:// URLs
- [ ] Accept https:// URLs
- [ ] Reject ftp://, file://, javascript:, and other protocols
- [ ] Reject malformed URLs with E101
- [ ] Max length 2048 characters
- [ ] Zod schema exportable for reuse

**Deliverables**:
- Integration in `/home/agent0/hx-docling-ui/src/lib/validation/url.ts`

**Technical Notes**:
This is integrated into NEO-1.4-002 but focuses on protocol validation:
```typescript
// Protocol validation
.refine(
  (url) => {
    try {
      const parsed = new URL(url);
      return ['http:', 'https:'].includes(parsed.protocol);
    } catch {
      return false;
    }
  },
  { message: 'Only HTTP and HTTPS URLs are allowed' }
)
```

---

### NEO-1.4-004: Create Zustand Document Store

**Priority**: P0 (Critical)
**Effort**: 30 minutes
**Dependencies**: None
**Reference**: FR-301

**Description**:
Create Zustand store for managing document input state, including file/URL selection, mutual exclusion, and processing state.

**Acceptance Criteria**:
- [ ] Store tracks file, url, activeInput, isProcessing states
- [ ] setFile action clears URL and sets activeInput to 'file'
- [ ] setUrl action clears file and sets activeInput to 'url'
- [ ] clearInputs action resets all input state
- [ ] setProcessing action controls processing lock
- [ ] Store persists across component re-renders
- [ ] TypeScript types exported

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/stores/documentStore.ts`

**Technical Notes**:
```typescript
import { create } from 'zustand';
import { devtools } from 'zustand/middleware';

export interface DocumentState {
  // Input state
  file: File | null;
  url: string | null;
  activeInput: 'file' | 'url' | null;

  // Job tracking
  currentJobId: string | null;

  // Processing state
  isProcessing: boolean;
}

export interface DocumentActions {
  setFile: (file: File | null, jobId?: string) => void;
  setUrl: (url: string | null, jobId?: string) => void;
  clearInputs: () => void;
  setProcessing: (isProcessing: boolean) => void;
  setCurrentJobId: (jobId: string | null) => void;
  reset: () => void;
}

const initialState: DocumentState = {
  file: null,
  url: null,
  activeInput: null,
  currentJobId: null,
  isProcessing: false,
};

export const useDocumentStore = create<DocumentState & DocumentActions>()(
  devtools(
    (set) => ({
      ...initialState,

      setFile: (file, jobId) =>
        set(
          {
            file,
            url: null,
            activeInput: file ? 'file' : null,
            currentJobId: jobId || null,
          },
          false,
          'setFile'
        ),

      setUrl: (url, jobId) =>
        set(
          {
            url,
            file: null,
            activeInput: url ? 'url' : null,
            currentJobId: jobId || null,
          },
          false,
          'setUrl'
        ),

      clearInputs: () =>
        set(
          {
            file: null,
            url: null,
            activeInput: null,
            currentJobId: null,
          },
          false,
          'clearInputs'
        ),

      setProcessing: (isProcessing) =>
        set({ isProcessing }, false, 'setProcessing'),

      setCurrentJobId: (jobId) =>
        set({ currentJobId: jobId }, false, 'setCurrentJobId'),

      reset: () => set(initialState, false, 'reset'),
    }),
    { name: 'document-store' }
  )
);
```

---

### NEO-1.4-005: Implement Mutual Exclusion (File XOR URL)

**Priority**: P0 (Critical)
**Effort**: 20 minutes
**Dependencies**: NEO-1.4-001, NEO-1.4-004
**Reference**: FR-301

**Description**:
Implement mutual exclusion logic ensuring only one input type (file or URL) is active at a time. Selecting a file clears any entered URL and vice versa.

**Acceptance Criteria**:
- [ ] Setting file clears URL
- [ ] Setting URL clears file
- [ ] Only one activeInput at a time
- [ ] UI reflects mutual exclusion (disabled states)
- [ ] State transitions are atomic

**Deliverables**:
- Integration in document store and input components

**Technical Notes**:
```typescript
// In UploadForm.tsx
const { file, setFile, isProcessing } = useDocumentStore();

// When file selected, it automatically clears URL via store
const handleFileSelect = (selectedFile: File) => {
  setFile(selectedFile);
};

// In UrlInput.tsx
const { url, setUrl, isProcessing } = useDocumentStore();

// When URL submitted, it automatically clears file via store
const handleUrlSubmit = (submittedUrl: string, jobId: string) => {
  setUrl(submittedUrl, jobId);
};

// Visual feedback in both components
// Show which input is active, disable the other
```

---

### NEO-1.4-006: Add Processing Lock

**Priority**: P1 (High)
**Effort**: 10 minutes
**Dependencies**: NEO-1.4-004
**Reference**: FR-302

**Description**:
Implement processing lock that prevents input changes during document processing.

**Acceptance Criteria**:
- [ ] Input fields disabled when isProcessing=true
- [ ] Process/Submit buttons disabled when isProcessing=true
- [ ] Visual indication of locked state
- [ ] Lock released on processing complete or error

**Deliverables**:
- Integration in input components

**Technical Notes**:
```typescript
// In UploadForm.tsx and UrlInput.tsx
const { isProcessing } = useDocumentStore();

// Disable all inputs during processing
<UploadZone disabled={isProcessing} />
<Input disabled={isProcessing} />
<Button disabled={isProcessing}>Process</Button>

// Show loading state
{isProcessing && <Loader2 className="animate-spin" />}
```

---

### NEO-1.4-007: Create UI Store for Toasts/Modals

**Priority**: P1 (High)
**Effort**: 20 minutes
**Dependencies**: None

**Description**:
Create centralized Zustand store for UI state management including toast notifications and modal dialogs.

**Acceptance Criteria**:
- [ ] Toast state management (add, remove, clear)
- [ ] Modal state management (open, close)
- [ ] Toast types: success, error, warning, info
- [ ] Auto-dismiss for toasts (configurable duration)
- [ ] TypeScript types exported

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/stores/uiStore.ts`

**Technical Notes**:
```typescript
import { create } from 'zustand';
import { devtools } from 'zustand/middleware';

export type ToastType = 'success' | 'error' | 'warning' | 'info';

export interface Toast {
  id: string;
  type: ToastType;
  title: string;
  message?: string;
  duration?: number;
}

export interface ModalState {
  isOpen: boolean;
  content: React.ReactNode | null;
  title?: string;
}

interface UIState {
  toasts: Toast[];
  modal: ModalState;
}

interface UIActions {
  addToast: (toast: Omit<Toast, 'id'>) => void;
  removeToast: (id: string) => void;
  clearToasts: () => void;
  openModal: (content: React.ReactNode, title?: string) => void;
  closeModal: () => void;
}

const initialState: UIState = {
  toasts: [],
  modal: { isOpen: false, content: null },
};

export const useUIStore = create<UIState & UIActions>()(
  devtools(
    (set) => ({
      ...initialState,

      addToast: (toast) =>
        set(
          (state) => ({
            toasts: [
              ...state.toasts,
              { ...toast, id: crypto.randomUUID() },
            ],
          }),
          false,
          'addToast'
        ),

      removeToast: (id) =>
        set(
          (state) => ({
            toasts: state.toasts.filter((t) => t.id !== id),
          }),
          false,
          'removeToast'
        ),

      clearToasts: () =>
        set({ toasts: [] }, false, 'clearToasts'),

      openModal: (content, title) =>
        set(
          { modal: { isOpen: true, content, title } },
          false,
          'openModal'
        ),

      closeModal: () =>
        set(
          { modal: { isOpen: false, content: null, title: undefined } },
          false,
          'closeModal'
        ),
    }),
    { name: 'ui-store' }
  )
);
```

---

### NEO-1.4-008: Apply Rate Limiting Middleware to URL Route

**Priority**: P1 (High)
**Effort**: 10 minutes
**Dependencies**: NEO-1.4-001, Sprint 1.2 (Rate Limiting)

**Description**:
Apply rate limiting middleware to the URL processing endpoint to enforce 10 requests per minute per session.

**Acceptance Criteria**:
- [ ] Rate limiting middleware applied to URL upload endpoint
- [ ] Same rate limit as file upload (10 req/min)
- [ ] Returns 429 with E601 on limit exceeded
- [ ] X-RateLimit-* headers present

**Deliverables**:
- Update `/home/agent0/hx-docling-ui/src/app/api/v1/upload/route.ts` (URL handling)

**Technical Notes**:
```typescript
// The upload route already handles both file and URL
// Ensure rate limiting applies to both paths

import { withRateLimit } from '@/lib/middleware/rate-limit';

async function uploadHandler(request: NextRequest) {
  const contentType = request.headers.get('Content-Type');

  if (contentType?.includes('application/json')) {
    // URL upload path
    const { url } = await request.json();
    // ... URL processing
  } else if (contentType?.includes('multipart/form-data')) {
    // File upload path
    // ... File processing
  }
}

// Rate limit applies to both paths
export const POST = withRateLimit({
  limit: 10,
  windowMs: 60 * 1000,
  keyPrefix: 'upload',
})(uploadHandler);
```

---

### NEO-1.4-009: Style and Test URL Input

**Priority**: P1 (High)
**Effort**: 15 minutes
**Dependencies**: NEO-1.4-001, NEO-1.4-007

**Description**:
Apply shadcn/ui styling to URL input component and ensure visual consistency with the upload zone.

**Acceptance Criteria**:
- [ ] shadcn/ui Input component used
- [ ] Consistent styling with UploadForm
- [ ] Dark mode compatible
- [ ] Error states styled appropriately
- [ ] Loading states styled appropriately
- [ ] Toast notifications on success/error

**Deliverables**:
- Style updates to UrlInput component

**Technical Notes**:
- Use Input with icon prefix (Link icon)
- Use Button variants for submit/clear
- Integrate Sonner toast for notifications
- Ensure consistent spacing and typography

---

### NEO-1.4-010: Write Unit Tests for SSRF Validation

**Priority**: P1 (High)
**Effort**: 20 minutes
**Dependencies**: NEO-1.4-002

**Description**:
Write comprehensive unit tests for SSRF validation covering all blocked patterns.

**Acceptance Criteria**:
- [ ] Test localhost blocking (127.0.0.1, localhost)
- [ ] Test 10.x.x.x blocking
- [ ] Test 172.16-31.x.x blocking
- [ ] Test 192.168.x.x blocking
- [ ] Test 169.254.x.x blocking
- [ ] Test *.hx.dev.local blocking
- [ ] Test IPv6 localhost (::1) blocking
- [ ] Test valid public URLs pass
- [ ] Tests achieve 100% coverage of SSRF logic

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/lib/validation/url.test.ts`

**Technical Notes**:
```typescript
import { describe, it, expect } from 'vitest';
import { isURLBlocked, validateURL } from './url';

describe('SSRF Prevention', () => {
  describe('isURLBlocked', () => {
    describe('localhost variations', () => {
      it.each([
        'http://localhost',
        'http://127.0.0.1',
        'http://127.0.0.1:8080',
        'http://localhost:3000/path',
        'http://LOCALHOST',
        'http://[::1]',
      ])('blocks %s', (url) => {
        expect(isURLBlocked(url).blocked).toBe(true);
      });
    });

    describe('private IP ranges', () => {
      it.each([
        // 10.0.0.0/8
        'http://10.0.0.1',
        'http://10.255.255.255',
        // 172.16.0.0/12
        'http://172.16.0.1',
        'http://172.31.255.255',
        // 192.168.0.0/16
        'http://192.168.0.1',
        'http://192.168.255.255',
        // 169.254.0.0/16
        'http://169.254.1.1',
      ])('blocks private IP %s', (url) => {
        expect(isURLBlocked(url).blocked).toBe(true);
      });

      it.each([
        'http://172.15.0.1', // Just outside 172.16.0.0/12
        'http://172.32.0.1', // Just outside 172.16.0.0/12
      ])('allows non-private %s', (url) => {
        expect(isURLBlocked(url).blocked).toBe(false);
      });
    });

    describe('internal domains', () => {
      it.each([
        'http://app.hx.dev.local',
        'http://api.hx.dev.local:8080',
        'https://subdomain.hx.dev.local/path',
      ])('blocks internal domain %s', (url) => {
        expect(isURLBlocked(url).blocked).toBe(true);
      });
    });

    describe('valid public URLs', () => {
      it.each([
        'https://example.com',
        'https://www.google.com',
        'http://api.github.com/users',
        'https://192.0.2.1', // TEST-NET-1 (documentation range, but public)
      ])('allows public URL %s', (url) => {
        expect(isURLBlocked(url).blocked).toBe(false);
      });
    });
  });

  describe('validateURL', () => {
    it('rejects non-HTTP protocols', () => {
      const result = validateURL('ftp://files.example.com');
      expect(result.valid).toBe(false);
      expect(result.errors[0].code).toBe('E102');
    });

    it('rejects URLs over 2048 characters', () => {
      const longUrl = 'https://example.com/' + 'a'.repeat(2048);
      const result = validateURL(longUrl);
      expect(result.valid).toBe(false);
      expect(result.errors[0].code).toBe('E103');
    });

    it('rejects blocked URLs with E104', () => {
      const result = validateURL('http://192.168.1.1');
      expect(result.valid).toBe(false);
      expect(result.errors[0].code).toBe('E104');
    });
  });
});
```

---

### NEO-1.4-011: Write Unit Tests for Document Store

**Priority**: P1 (High)
**Effort**: 15 minutes
**Dependencies**: NEO-1.4-004

**Description**:
Write unit tests for Zustand document store state management, focusing on state transitions and mutual exclusion.

**Acceptance Criteria**:
- [ ] Test initial state
- [ ] Test setFile clears URL
- [ ] Test setUrl clears file
- [ ] Test clearInputs resets state
- [ ] Test processing lock state
- [ ] Test reset functionality
- [ ] Tests achieve 100% coverage of store

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/stores/documentStore.test.ts`

**Technical Notes**:
```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import { useDocumentStore } from './documentStore';

describe('documentStore', () => {
  beforeEach(() => {
    useDocumentStore.getState().reset();
  });

  describe('initial state', () => {
    it('has null file and url', () => {
      const state = useDocumentStore.getState();
      expect(state.file).toBeNull();
      expect(state.url).toBeNull();
      expect(state.activeInput).toBeNull();
    });
  });

  describe('setFile', () => {
    it('sets file and clears url', () => {
      const store = useDocumentStore.getState();

      // First set a URL
      store.setUrl('https://example.com', 'job-1');
      expect(store.url).toBe('https://example.com');

      // Then set a file - should clear URL
      const file = new File(['test'], 'test.pdf');
      store.setFile(file, 'job-2');

      const newState = useDocumentStore.getState();
      expect(newState.file).toBe(file);
      expect(newState.url).toBeNull();
      expect(newState.activeInput).toBe('file');
      expect(newState.currentJobId).toBe('job-2');
    });
  });

  describe('setUrl', () => {
    it('sets url and clears file', () => {
      const store = useDocumentStore.getState();

      // First set a file
      const file = new File(['test'], 'test.pdf');
      store.setFile(file, 'job-1');
      expect(store.file).toBe(file);

      // Then set a URL - should clear file
      store.setUrl('https://example.com', 'job-2');

      const newState = useDocumentStore.getState();
      expect(newState.url).toBe('https://example.com');
      expect(newState.file).toBeNull();
      expect(newState.activeInput).toBe('url');
    });
  });

  describe('clearInputs', () => {
    it('resets all input state', () => {
      const store = useDocumentStore.getState();
      store.setFile(new File(['test'], 'test.pdf'), 'job-1');
      store.clearInputs();

      const newState = useDocumentStore.getState();
      expect(newState.file).toBeNull();
      expect(newState.url).toBeNull();
      expect(newState.activeInput).toBeNull();
      expect(newState.currentJobId).toBeNull();
    });
  });

  describe('processing lock', () => {
    it('sets processing state', () => {
      const store = useDocumentStore.getState();
      store.setProcessing(true);
      expect(useDocumentStore.getState().isProcessing).toBe(true);

      store.setProcessing(false);
      expect(useDocumentStore.getState().isProcessing).toBe(false);
    });
  });
});
```

---

## Summary

| Task ID | Task Title | Effort | Priority | Dependency |
|---------|-----------|--------|----------|------------|
| NEO-1.4-001 | Create URL Input Component | 25m | P0 | - |
| NEO-1.4-002 | Implement SSRF Prevention | 20m | P0 | BOB-1.4-005 |
| NEO-1.4-003 | Add URL Format Validation | 15m | P0 | NEO-1.4-002 |
| NEO-1.4-004 | Create Zustand Document Store | 30m | P0 | - |
| NEO-1.4-005 | Implement Mutual Exclusion | 20m | P0 | NEO-1.4-001, NEO-1.4-004 |
| NEO-1.4-006 | Add Processing Lock | 10m | P1 | NEO-1.4-004 |
| NEO-1.4-007 | Create UI Store | 20m | P1 | - |
| NEO-1.4-008 | Apply Rate Limiting Middleware | 10m | P1 | NEO-1.4-001 |
| NEO-1.4-009 | Style and Test URL Input | 15m | P1 | NEO-1.4-001, NEO-1.4-007 |
| NEO-1.4-010 | Write Unit Tests for SSRF | 20m | P1 | NEO-1.4-002 |
| NEO-1.4-011 | Write Unit Tests for Store | 15m | P1 | NEO-1.4-004 |

**Total Effort**: ~1.8 hours (105 minutes)
**Total Tasks**: 11

---

## Dependencies Graph

```
Sprint 1.3 Complete
    |
    +---------------------------+
    |                           |
    v                           v
BOB-1.4-005 (Shared SSRF)   NEO-1.4-004 (Document Store)
    |                           |
    v                           +-> NEO-1.4-005 (Mutual Exclusion)
NEO-1.4-002 (SSRF Prevention)   +-> NEO-1.4-006 (Processing Lock)
    |                           +-> NEO-1.4-011 (Store Tests)
    +-> NEO-1.4-003 (URL Format Validation)
    |       |
    |       +-> NEO-1.4-001 (URL Input Component)
    |               |
    |               +-> NEO-1.4-005 (Mutual Exclusion)
    |               +-> NEO-1.4-008 (Rate Limiting)
    |               +-> NEO-1.4-009 (Styling)
    |
    +-> NEO-1.4-010 (SSRF Tests)

NEO-1.4-007 (UI Store)
    |
    +-> NEO-1.4-009 (Styling)

Note: NEO-1.4-002 now depends on BOB-1.4-005 (Bob's shared SSRF config)
```

---

## Parallel Execution Opportunities

**Parallel Group A** (Independent tasks - can start immediately):
- [P] NEO-1.4-004 (Document Store)
- [P] NEO-1.4-007 (UI Store)

**Blocked on BOB-1.4-005** (Wait for Bob's shared SSRF config):
- NEO-1.4-002 (SSRF Prevention) - imports from shared config

**Parallel Group B** (After NEO-1.4-002 and Group A):
- [P] NEO-1.4-010 (SSRF Tests) - depends on 002
- [P] NEO-1.4-011 (Store Tests) - depends on 004

**Sequential** (Must wait for dependencies):
- NEO-1.4-003 -> NEO-1.4-001 -> NEO-1.4-005, NEO-1.4-008, NEO-1.4-009

---

## Changelog

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0.0 | 2025-12-12 | Neo | Initial task definitions |
| 1.1.0 | 2025-12-12 | Neo | DEF-015: Updated NEO-1.4-002 to import from shared SSRF config (BOB-1.4-005). Removed duplicated BLOCKED_HOSTNAMES, isPrivateIP(), isBlockedDomain() definitions. Added cross-team dependency. |
