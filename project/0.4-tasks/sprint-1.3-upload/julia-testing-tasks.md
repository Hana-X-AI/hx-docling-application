# Julia Santos - Testing & QA Tasks: Sprint 1.3 (File Upload System)

**Sprint**: 1.3 - File Upload System
**Role**: Testing & Quality Assurance Specialist
**Document Version**: 1.0.0
**Created**: 2025-12-12
**References**:
- Implementation Plan v1.2.0 Section 4.3
- Detailed Specification v1.2.0 Sections 10.2, 10.3

---

## Sprint 1.3 Testing Objectives

Implement incremental unit tests for the file upload system components. Focus on TDD practices by writing tests for file validation, UploadZone component, FilePreview component, and the upload API route. Ensure 80%+ coverage for all new code.

---

## Tasks

### JUL-1.3-001: Write Unit Tests for File Validation

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.3-001 |
| **Title** | Write Unit Tests for File Validation Library |
| **Priority** | P0 (Critical) |
| **Effort** | 1.0 hour |
| **Dependencies** | NEO-1.3-002 (File validation schema created) |

#### Description

Write comprehensive unit tests for the file validation library covering all file types, size limits, and edge cases. Tests should be written following TDD principles - ideally before or alongside implementation.

#### Acceptance Criteria

- [ ] Tests for valid PDF files (all sizes up to 100MB)
- [ ] Tests for valid Word documents (.doc, .docx up to 50MB)
- [ ] Tests for valid Excel files (.xls, .xlsx up to 50MB)
- [ ] Tests for valid PowerPoint files (.ppt, .pptx up to 50MB)
- [ ] Tests for valid images (.png, .jpg, .jpeg, .tiff up to 25MB)
- [ ] Tests for file size limit rejection (E001)
- [ ] Tests for unsupported file type rejection (E002)
- [ ] Tests for malformed filename handling
- [ ] Tests for MIME type validation
- [ ] Tests for edge cases (empty file, null, undefined)
- [ ] Coverage >= 95% for `lib/validation/file.ts`

#### Technical Notes

**Important**: Zod's `safeParse` returns Zod-specific issue codes (e.g., `'custom'`, `'invalid_type'`), not application-level codes. To return application error codes like `E001` (file size exceeded) and `E002` (unsupported file type), the implementation must either:

1. **Option A**: Use `.superRefine()` with custom error objects that include application codes in a custom field (e.g., `params.appCode`), OR
2. **Option B** (Recommended): Create a `validateFile` wrapper function that catches `ZodError` and maps issues to a `ValidationResult` type with application codes.

**Implementation Pattern (file.ts)**:

```typescript
// src/lib/validation/file.ts
import { z, ZodError } from 'zod';

// =============================================================================
// Type Definitions
// =============================================================================

/** Input type for file validation */
export interface FileInput {
  name: string;
  size: number;
  type: string;
}

// =============================================================================
// Constants
// =============================================================================

/** Allowed file extensions for upload */
export const ALLOWED_EXTENSIONS: string[] = [
  'pdf',
  'doc', 'docx',
  'xls', 'xlsx',
  'ppt', 'pptx',
  'png', 'jpg', 'jpeg', 'tiff',
];

/** Application error codes */
export const FILE_ERROR_CODES = {
  SIZE_EXCEEDED: 'E001',
  UNSUPPORTED_TYPE: 'E002',
} as const;

/** Maximum file sizes by extension (in bytes) */
const MAX_SIZES: Record<string, number> = {
  pdf: 100 * 1024 * 1024,  // 100MB
  doc: 50 * 1024 * 1024,   // 50MB
  docx: 50 * 1024 * 1024,  // 50MB
  xls: 50 * 1024 * 1024,   // 50MB
  xlsx: 50 * 1024 * 1024,  // 50MB
  ppt: 50 * 1024 * 1024,   // 50MB
  pptx: 50 * 1024 * 1024,  // 50MB
  png: 25 * 1024 * 1024,   // 25MB
  jpg: 25 * 1024 * 1024,   // 25MB
  jpeg: 25 * 1024 * 1024,  // 25MB
  tiff: 25 * 1024 * 1024,  // 25MB
};

/** Default max size for unknown extensions (25MB) */
const DEFAULT_MAX_SIZE = 25 * 1024 * 1024;

// =============================================================================
// Helper Functions
// =============================================================================

/**
 * Returns the maximum allowed file size for a given extension.
 * @param ext - File extension (without leading dot)
 * @returns Maximum size in bytes
 */
export function getMaxSizeForExtension(ext: string): number {
  return MAX_SIZES[ext.toLowerCase()] ?? DEFAULT_MAX_SIZE;
}

/**
 * Formats a byte count into a human-readable string.
 * @param bytes - Size in bytes
 * @returns Formatted string (e.g., "1.5 MB", "256 KB")
 */
export function formatBytes(bytes: number): string {
  if (bytes === 0) return '0 Bytes';

  const k = 1024;
  const sizes = ['Bytes', 'KB', 'MB', 'GB'];
  const i = Math.min(
    Math.max(0, Math.floor(Math.log(bytes) / Math.log(k))),
    sizes.length - 1
  );

  return `${parseFloat((bytes / Math.pow(k, i)).toFixed(2))} ${sizes[i]}`;
}

// Result type for validation
export type ValidationResult<T> =
  | { success: true; data: T }
  | { success: false; code: string; message: string };

// Base Zod schema with superRefine for custom error metadata
export const fileSchema = z.object({
  name: z.string().min(1),
  size: z.number(),
  type: z.string(),
}).superRefine((data, ctx) => {
  const ext = data.name.split('.').pop()?.toLowerCase();

  // Check file type first
  if (!ALLOWED_EXTENSIONS.includes(ext ?? '')) {
    ctx.addIssue({
      code: z.ZodIssueCode.custom,
      message: `Unsupported file type: .${ext}`,
      params: { appCode: FILE_ERROR_CODES.UNSUPPORTED_TYPE },
    });
    return;
  }

  // Check size limits based on file type
  const maxSize = getMaxSizeForExtension(ext!);
  if (data.size > maxSize) {
    ctx.addIssue({
      code: z.ZodIssueCode.custom,
      message: `File exceeds maximum size of ${formatBytes(maxSize)}`,
      params: { appCode: FILE_ERROR_CODES.SIZE_EXCEEDED },
    });
  }
});

// Wrapper function that maps Zod errors to application Result type
export function validateFile(input: unknown): ValidationResult<FileInput> {
  const result = fileSchema.safeParse(input);

  if (result.success) {
    return { success: true, data: result.data };
  }

  // Map Zod issue to application error code
  const issue = result.error.issues[0];
  const appCode = (issue.params?.appCode as string) ?? 'E000';

  return {
    success: false,
    code: appCode,
    message: issue.message,
  };
}
```

**Test Pattern (file.test.ts)**:

```typescript
// src/lib/validation/__tests__/file.test.ts
import { describe, it, expect } from 'vitest';
import {
  validateFile,
  fileSchema,
  ALLOWED_EXTENSIONS,
  FILE_ERROR_CODES,
  getMaxSizeForExtension,
  formatBytes,
  type FileInput,
} from '../file';

describe('File Validation', () => {
  describe('validateFile (returns application Result type)', () => {
    it('accepts valid PDF file under 100MB', () => {
      const result = validateFile({
        name: 'document.pdf',
        size: 50 * 1024 * 1024, // 50MB
        type: 'application/pdf',
      });
      expect(result.success).toBe(true);
      if (result.success) {
        expect(result.data.name).toBe('document.pdf');
      }
    });

    it('rejects PDF file over 100MB with E001 code', () => {
      const result = validateFile({
        name: 'large.pdf',
        size: 101 * 1024 * 1024,
        type: 'application/pdf',
      });
      expect(result.success).toBe(false);
      if (!result.success) {
        expect(result.code).toBe('E001');
        expect(result.message).toMatch(/exceeds maximum size/i);
      }
    });

    it('rejects image file over 25MB with E001 code', () => {
      const result = validateFile({
        name: 'image.png',
        size: 26 * 1024 * 1024,
        type: 'image/png',
      });
      expect(result.success).toBe(false);
      if (!result.success) {
        expect(result.code).toBe('E001');
      }
    });

    it('rejects unsupported extension with E002 code', () => {
      const result = validateFile({
        name: 'script.exe',
        size: 1024,
        type: 'application/octet-stream',
      });
      expect(result.success).toBe(false);
      if (!result.success) {
        expect(result.code).toBe('E002');
        expect(result.message).toMatch(/unsupported file type/i);
      }
    });
  });

  describe('fileSchema (raw Zod parsing)', () => {
    it('accepts valid PDF file under 100MB', () => {
      const result = fileSchema.safeParse({
        name: 'document.pdf',
        size: 50 * 1024 * 1024,
        type: 'application/pdf',
      });
      expect(result.success).toBe(true);
    });

    it('includes appCode in error params for size exceeded', () => {
      const result = fileSchema.safeParse({
        name: 'large.pdf',
        size: 101 * 1024 * 1024,
        type: 'application/pdf',
      });
      expect(result.success).toBe(false);
      if (!result.success) {
        // Zod issue.code will be 'custom', but params.appCode has our code
        expect(result.error.issues[0].code).toBe('custom');
        expect(result.error.issues[0].params?.appCode).toBe('E001');
      }
    });

    it('includes appCode in error params for unsupported type', () => {
      const result = fileSchema.safeParse({
        name: 'script.exe',
        size: 1024,
        type: 'application/octet-stream',
      });
      expect(result.success).toBe(false);
      if (!result.success) {
        expect(result.error.issues[0].code).toBe('custom');
        expect(result.error.issues[0].params?.appCode).toBe('E002');
      }
    });
  });

  // Additional test cases per acceptance criteria...
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| File validation tests | `src/lib/validation/__tests__/file.test.ts` |

---

### JUL-1.3-002: Write Component Tests for UploadZone

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.3-002 |
| **Title** | Write Component Tests for UploadZone |
| **Priority** | P0 (Critical) |
| **Effort** | 1.25 hours |
| **Dependencies** | NEO-1.3-001 (UploadZone created), JUL-1.1-001 (Vitest configured) |

#### Description

Write comprehensive component tests for the UploadZone component covering drag-and-drop functionality, click-to-browse, file validation, error states, and accessibility.

#### Acceptance Criteria

- [ ] Test renders dropzone with correct label
- [ ] Test drag-over visual feedback
- [ ] Test accepts valid PDF file on drop
- [ ] Test accepts valid file via click-to-browse
- [ ] Test shows error for oversized file
- [ ] Test shows error for unsupported file type
- [ ] Test keyboard accessibility (Enter/Space to activate)
- [ ] Test ARIA labels present
- [ ] Test focus management
- [ ] Test disabled state during processing
- [ ] Test multiple file rejection (single file only)
- [ ] Coverage >= 90% for `UploadZone.tsx`

#### Technical Notes

```typescript
// src/components/upload/__tests__/UploadZone.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, it, expect, vi } from 'vitest';
import { UploadZone } from '../UploadZone';

describe('UploadZone', () => {
  it('renders dropzone with correct label', () => {
    render(<UploadZone onFileSelect={vi.fn()} />);
    expect(screen.getByText(/drop file here/i)).toBeInTheDocument();
    expect(screen.getByText(/click to browse/i)).toBeInTheDocument();
  });

  it('shows drag-over feedback', async () => {
    render(<UploadZone onFileSelect={vi.fn()} />);
    const dropzone = screen.getByTestId('upload-dropzone');

    fireEvent.dragEnter(dropzone);
    expect(dropzone).toHaveClass('border-primary');
    expect(dropzone).toHaveClass('bg-accent/10');

    fireEvent.dragLeave(dropzone);
    expect(dropzone).not.toHaveClass('border-primary', { exact: false });
    expect(dropzone).not.toHaveClass('bg-accent/10', { exact: false });
  });

  it('accepts valid PDF file on drop', async () => {
    const onFileSelect = vi.fn();
    render(<UploadZone onFileSelect={onFileSelect} />);

    const file = new File(['content'], 'test.pdf', { type: 'application/pdf' });
    const dropzone = screen.getByTestId('upload-dropzone');

    fireEvent.drop(dropzone, {
      dataTransfer: { files: [file] },
    });

    expect(onFileSelect).toHaveBeenCalledWith(file);
  });

  it('shows error for oversized file', async () => {
    render(<UploadZone onFileSelect={vi.fn()} />);

    // Create mock oversized file
    const largeFile = new File([new ArrayBuffer(101 * 1024 * 1024)], 'large.pdf', {
      type: 'application/pdf',
    });
    Object.defineProperty(largeFile, 'size', { value: 101 * 1024 * 1024 });

    const dropzone = screen.getByTestId('upload-dropzone');
    fireEvent.drop(dropzone, { dataTransfer: { files: [largeFile] } });

    expect(screen.getByRole('alert')).toHaveTextContent(/100 MB/i);
  });

  it('is keyboard accessible', async () => {
    const user = userEvent.setup();
    render(<UploadZone onFileSelect={vi.fn()} />);

    const dropzone = screen.getByTestId('upload-dropzone');
    await user.tab();

    expect(dropzone).toHaveFocus();
    expect(dropzone).toHaveAttribute('tabIndex', '0');
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| UploadZone component tests | `src/components/upload/__tests__/UploadZone.test.tsx` |

---

### JUL-1.3-003: Write Component Tests for FilePreview

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.3-003 |
| **Title** | Write Component Tests for FilePreview |
| **Priority** | P1 (High) |
| **Effort** | 0.75 hours |
| **Dependencies** | NEO-1.3-003 (FilePreview created) |

#### Description

Write component tests for the FilePreview component covering display of file metadata, type icons, and remove functionality.

#### Acceptance Criteria

- [ ] Test displays file name correctly
- [ ] Test displays formatted file size (KB, MB)
- [ ] Test displays correct icon for each file type
- [ ] Test remove button functionality
- [ ] Test remove button keyboard accessibility
- [ ] Test handles long file names gracefully
- [ ] Coverage >= 90% for `FilePreview.tsx`

#### Technical Notes

```typescript
// src/components/upload/__tests__/FilePreview.test.tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, it, expect, vi } from 'vitest';
import { FilePreview } from '../FilePreview';

describe('FilePreview', () => {
  const mockFile = new File(['content'], 'document.pdf', { type: 'application/pdf' });
  Object.defineProperty(mockFile, 'size', { value: 1024 * 1024 }); // 1MB

  it('displays file name', () => {
    render(<FilePreview file={mockFile} onRemove={vi.fn()} />);
    expect(screen.getByText('document.pdf')).toBeInTheDocument();
  });

  it('displays formatted file size', () => {
    render(<FilePreview file={mockFile} onRemove={vi.fn()} />);
    expect(screen.getByText('1.00 MB')).toBeInTheDocument();
  });

  it('displays PDF icon for PDF files', () => {
    render(<FilePreview file={mockFile} onRemove={vi.fn()} />);
    expect(screen.getByTestId('icon-pdf')).toBeInTheDocument();
  });

  it('calls onRemove when remove button clicked', async () => {
    const user = userEvent.setup();
    const onRemove = vi.fn();
    render(<FilePreview file={mockFile} onRemove={onRemove} />);

    await user.click(screen.getByRole('button', { name: /remove/i }));
    expect(onRemove).toHaveBeenCalledOnce();
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| FilePreview component tests | `src/components/upload/__tests__/FilePreview.test.tsx` |

---

### JUL-1.3-004: Write Integration Tests for Upload API Route

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.3-004 |
| **Title** | Write Integration Tests for Upload API Route |
| **Priority** | P0 (Critical) |
| **Effort** | 1.5 hours |
| **Dependencies** | NEO-1.3-005 (Upload API route created), JUL-1.1-003 (MSW configured) |

#### Description

Write integration tests for the upload API route covering successful uploads, validation errors, rate limiting, and job creation.

#### Acceptance Criteria

- [ ] Test successful file upload creates job with PENDING status
- [ ] Test returns 201 with jobId for valid upload
- [ ] Test returns 400 with E001 for oversized file
- [ ] Test returns 400 with E002 for unsupported file type
- [ ] Test returns 429 when rate limit exceeded
- [ ] Test file stored in correct directory structure (YYYY/MM/DD)
- [ ] Test job record created in database
- [ ] Test session ID extracted from cookie
- [ ] Test validation middleware applied
- [ ] Coverage >= 90% for upload route

#### Technical Notes

```typescript
// src/app/api/v1/upload/__tests__/route.test.ts
import { describe, it, expect, beforeEach, afterEach, vi } from 'vitest';
import { POST } from '../route';
import { prisma } from '@/lib/db/prisma';
import * as sessionModule from '@/lib/redis/session';

// Mock the getSession function to return controlled session IDs
vi.mock('@/lib/redis/session', () => ({
  getSession: vi.fn(),
}));

describe('POST /api/v1/upload', () => {
  beforeEach(async () => {
    // Clean up test data
    vi.clearAllMocks();
  });

  afterEach(async () => {
    // Clean up created jobs
  });

  it('creates job for valid file upload', async () => {
    // Mock getSession to return the expected session ID
    vi.mocked(sessionModule.getSession).mockResolvedValue({ id: 'test-session-id' });

    const formData = new FormData();
    // Use File instead of Blob so file.name is available
    const file = new File(['content'], 'test.pdf', { type: 'application/pdf' });
    formData.append('file', file);

    const request = new Request('http://localhost/api/v1/upload', {
      method: 'POST',
      body: formData,
    });

    const response = await POST(request);
    const data = await response.json();

    expect(response.status).toBe(201);
    expect(data.jobId).toBeDefined();
    expect(data.status).toBe('PENDING');

    // Verify job in database
    const job = await prisma.job.findUnique({ where: { id: data.jobId } });
    expect(job).not.toBeNull();
    expect(job?.sessionId).toBe('test-session-id');
  });

  it('returns 400 with E001 for oversized file', async () => {
    // Mock getSession for this test
    vi.mocked(sessionModule.getSession).mockResolvedValue({ id: 'test-session-id' });

    const formData = new FormData();
    // Use File instead of Blob so file.name is available
    const largeFile = new File([new ArrayBuffer(101 * 1024 * 1024)], 'large.pdf', { type: 'application/pdf' });
    formData.append('file', largeFile);

    const request = new Request('http://localhost/api/v1/upload', {
      method: 'POST',
      body: formData,
    });

    const response = await POST(request);
    const data = await response.json();

    expect(response.status).toBe(400);
    expect(data.code).toBe('E001');
  });

  it('returns 429 when rate limit exceeded', async () => {
    // Mock getSession to return consistent session ID for rate limiting
    vi.mocked(sessionModule.getSession).mockResolvedValue({ id: 'rate-limit-test' });

    // Make 11 requests (limit is 10/min)
    for (let i = 0; i < 11; i++) {
      const formData = new FormData();
      // Use File instead of Blob so file.name is available
      const file = new File(['x'], 'test.pdf', { type: 'application/pdf' });
      formData.append('file', file);

      const request = new Request('http://localhost/api/v1/upload', {
        method: 'POST',
        body: formData,
      });

      const response = await POST(request);

      if (i === 10) {
        expect(response.status).toBe(429);
        expect(response.headers.get('Retry-After')).toBeDefined();
      }
    }
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| Upload API route tests | `src/app/api/v1/upload/__tests__/route.test.ts` |

---

### JUL-1.3-005: Verify Accessibility Compliance

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.3-005 |
| **Title** | Verify Upload Components Accessibility Compliance |
| **Priority** | P1 (High) |
| **Effort** | 0.5 hours |
| **Dependencies** | JUL-1.3-002, JUL-1.3-003 |

#### Description

Run accessibility audit on upload components and verify ARIA labels, keyboard navigation, and screen reader compatibility.

#### Acceptance Criteria

- [ ] axe-core audit passes with no violations
- [ ] ARIA labels present on all interactive elements
- [ ] Keyboard navigation verified (Tab, Enter, Space)
- [ ] Focus indicators visible
- [ ] Error messages properly announced to screen readers
- [ ] Color contrast meets WCAG AA standards
- [ ] No accessibility violations in upload components

#### Technical Notes

```typescript
// src/components/upload/__tests__/accessibility.test.tsx
import { render } from '@testing-library/react';
import { axe, toHaveNoViolations } from 'jest-axe';
import { describe, it, expect } from 'vitest';
import { UploadZone } from '../UploadZone';
import { FilePreview } from '../FilePreview';

expect.extend(toHaveNoViolations);

describe('Upload Components Accessibility', () => {
  it('UploadZone has no accessibility violations', async () => {
    const { container } = render(<UploadZone onFileSelect={() => {}} />);
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });

  it('FilePreview has no accessibility violations', async () => {
    const mockFile = new File([''], 'test.pdf');
    const { container } = render(<FilePreview file={mockFile} onRemove={() => {}} />);
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });

  it('error states are accessible', async () => {
    const { container, rerender } = render(<UploadZone onFileSelect={() => {}} />);
    // Trigger error state
    // Check that error is announced
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| Accessibility tests | `src/components/upload/__tests__/accessibility.test.tsx` |

---

## Sprint 1.3 Testing Summary

| Metric | Target | Notes |
|--------|--------|-------|
| Total Tasks | 5 | |
| Total Effort | 5.0 hours | |
| Critical Tasks | 3 | Validation, UploadZone, API route |
| High Priority Tasks | 2 | FilePreview, Accessibility |

### Coverage Targets

| Component | Target Coverage |
|-----------|-----------------|
| `lib/validation/file.ts` | >= 95% |
| `UploadZone.tsx` | >= 90% |
| `FilePreview.tsx` | >= 90% |
| `api/v1/upload/route.ts` | >= 90% |

### Quality Gates for Sprint 1.3

- [ ] All unit tests pass
- [ ] Coverage >= 80% for sprint components
- [ ] No accessibility violations
- [ ] Rate limiting tested
- [ ] All validation scenarios covered

---

## Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0.0 | 2025-12-12 | Julia Santos | Initial creation |
