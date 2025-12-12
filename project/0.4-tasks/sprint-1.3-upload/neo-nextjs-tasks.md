# Neo Next.js Tasks: Sprint 1.3 - File Upload System

**Sprint**: 1.3 - File Upload System
**Duration**: ~3.5 hours
**Role**: Lead Developer
**Agent**: Neo (Next.js Senior Developer)
**Support**: Ola (@ola), William (@william)
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
- `get_component_demo` - Get usage examples and demo code for components like Button, Card, Form
- `get_component_metadata` - Get props, installation instructions, and documentation

**Example - Get Card Component Demo:**
```json
{
  "method": "tools/call",
  "params": {
    "name": "get_component_demo",
    "arguments": {"componentName": "card"}
  }
}
```

**Sprint 1.3 Component Usage:** This sprint uses Card, Button, Form, Input, Skeleton, and Sonner (toast) components for the upload interface. Use `get_component_demo` to retrieve usage patterns.

---

## Overview

Implement the complete file upload system including drag-drop upload zone with react-dropzone, file validation (type, size), persistent file storage, upload API route, and integration with react-hook-form. This sprint builds the foundation for document processing.

---

## Tasks

### NEO-1.3-001: Create UploadZone with react-dropzone

**Priority**: P0 (Critical)
**Effort**: 45 minutes
**Dependencies**: Sprint 1.2 Complete
**Reference**: FR-101, FR-102

**Description**:
Implement the drag-and-drop file upload zone using react-dropzone. The component must support both drag-drop and click-to-browse file selection with proper visual feedback during drag operations.

**Acceptance Criteria**:
- [ ] User can drag file onto upload zone
- [ ] Visual feedback on drag-over (border highlight, background change)
- [ ] File accepted on drop
- [ ] Click on upload zone opens file browser
- [ ] Selected file populates upload zone
- [ ] 'use client' directive present (Client Component)
- [ ] Accessibility: keyboard accessible, ARIA labels for screen readers
- [ ] FILE_SELECTED event emitted on successful selection

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/components/upload/UploadZone.tsx`

**Technical Notes**:
```typescript
'use client';

import { useCallback, useState } from 'react';
import { useDropzone, Accept, FileRejection } from 'react-dropzone';
import { Upload, FileText, X } from 'lucide-react';
import { cn } from '@/lib/utils';

interface UploadZoneProps {
  onFileSelect: (file: File) => void;
  onFileReject?: (errors: string[]) => void;
  disabled?: boolean;
  maxSize?: number;
  accept?: Accept;
}

export function UploadZone({
  onFileSelect,
  onFileReject,
  disabled = false,
  maxSize = 100 * 1024 * 1024, // 100MB default
  accept,
}: UploadZoneProps) {
  const [isDragActive, setIsDragActive] = useState(false);

  const onDrop = useCallback(
    (acceptedFiles: File[], rejectedFiles: FileRejection[]) => {
      if (acceptedFiles.length > 0) {
        onFileSelect(acceptedFiles[0]);
      }
      if (rejectedFiles.length > 0 && onFileReject) {
        const errors = rejectedFiles.flatMap((f) =>
          f.errors.map((e) => e.message)
        );
        onFileReject(errors);
      }
    },
    [onFileSelect, onFileReject]
  );

  const { getRootProps, getInputProps, isDragAccept, isDragReject } =
    useDropzone({
      onDrop,
      disabled,
      maxSize,
      accept,
      multiple: false,
      onDragEnter: () => setIsDragActive(true),
      onDragLeave: () => setIsDragActive(false),
    });

  return (
    <div
      {...getRootProps()}
      className={cn(
        'relative flex flex-col items-center justify-center w-full h-64',
        'border-2 border-dashed rounded-lg cursor-pointer',
        'transition-colors duration-200',
        {
          'border-gray-300 dark:border-gray-700': !isDragActive,
          'border-primary bg-primary/5': isDragAccept,
          'border-destructive bg-destructive/5': isDragReject,
          'opacity-50 cursor-not-allowed': disabled,
        }
      )}
      role="button"
      aria-label="Upload file area. Drag and drop a file here or click to select"
      tabIndex={disabled ? -1 : 0}
    >
      <input {...getInputProps()} aria-label="File input" />
      <Upload className="w-12 h-12 text-gray-400 mb-4" aria-hidden="true" />
      <p className="text-lg font-medium">
        {isDragActive ? 'Drop the file here' : 'Drag & drop a file here'}
      </p>
      <p className="text-sm text-gray-500 mt-2">or click to select</p>
    </div>
  );
}
```

---

### NEO-1.3-002: Implement File Validation Schema

**Priority**: P0 (Critical)
**Effort**: 30 minutes
**Dependencies**: None
**Reference**: FR-103, FR-104

**Description**:
Create comprehensive file validation using Zod schemas for file type and size validation. Implement both allowed extensions and MIME type validation per specification.

**Acceptance Criteria**:
- [ ] Accept: .pdf, .doc, .docx, .ppt, .pptx, .xls, .xlsx, .png, .jpg, .jpeg, .tiff
- [ ] Reject unsupported types with error E002
- [ ] Enforce size limits per document type (PDF: 100MB, Word/Excel/PPT: 50MB, Images: 25MB)
- [ ] Reject oversized files with error E001
- [ ] MIME type validation alongside extension validation
- [ ] Validation functions are pure and testable

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/lib/validation/file.ts`

**Technical Notes**:
```typescript
import { z } from 'zod';

export const ALLOWED_EXTENSIONS = [
  '.pdf',
  '.doc', '.docx',
  '.ppt', '.pptx',
  '.xls', '.xlsx',
  '.png', '.jpg', '.jpeg', '.tiff',
] as const;

export const ALLOWED_MIME_TYPES = [
  'application/pdf',
  'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
  'application/msword',
  'application/vnd.openxmlformats-officedocument.presentationml.presentation',
  'application/vnd.ms-powerpoint',
  'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
  'application/vnd.ms-excel',
  'image/png',
  'image/jpeg',
  'image/tiff',
] as const;

export const FILE_SIZE_LIMITS: Record<string, number> = {
  'application/pdf': 100 * 1024 * 1024, // 100MB
  'application/vnd.openxmlformats-officedocument.wordprocessingml.document': 50 * 1024 * 1024,
  'application/msword': 50 * 1024 * 1024,
  // ... etc
  default: 25 * 1024 * 1024, // Images and unknown: 25MB
};

export const fileValidationSchema = z.object({
  name: z.string().min(1, 'File name is required'),
  size: z.number().positive(),
  type: z.string(),
}).refine((file) => {
  const ext = '.' + file.name.split('.').pop()?.toLowerCase();
  return ALLOWED_EXTENSIONS.includes(ext as typeof ALLOWED_EXTENSIONS[number]);
}, {
  message: 'Unsupported file type',
  path: ['type'],
}).refine((file) => {
  const maxSize = FILE_SIZE_LIMITS[file.type] || FILE_SIZE_LIMITS.default;
  return file.size <= maxSize;
}, {
  message: 'File too large',
  path: ['size'],
});

export type FileValidationResult = {
  valid: boolean;
  errors: Array<{ code: string; message: string }>;
};

export function validateFile(file: File): FileValidationResult {
  const result = fileValidationSchema.safeParse({
    name: file.name,
    size: file.size,
    type: file.type,
  });

  if (result.success) {
    return { valid: true, errors: [] };
  }

  const errors = result.error.errors.map((err) => ({
    code: err.path.includes('size') ? 'E001' : 'E002',
    message: err.message,
  }));

  return { valid: false, errors };
}
```

---

### NEO-1.3-003: Create FilePreview Component

**Priority**: P1 (High)
**Effort**: 25 minutes
**Dependencies**: NEO-1.3-001
**Reference**: FR-105

**Description**:
Display file metadata after selection including file name, formatted size, file type icon, and a remove button.

**Acceptance Criteria**:
- [ ] Display file name (truncated if too long)
- [ ] Display file size (formatted: KB, MB, GB)
- [ ] Display file type icon (using lucide-react)
- [ ] Remove button to clear selection
- [ ] Responsive layout
- [ ] Accessibility: remove button has accessible name

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/components/upload/FilePreview.tsx`

**Technical Notes**:
```typescript
'use client';

import { FileText, FileSpreadsheet, FileImage, File, X } from 'lucide-react';
import { Button } from '@/components/ui/button';
import { formatFileSize } from '@/lib/utils/format';

interface FilePreviewProps {
  file: File;
  onRemove: () => void;
}

function getFileIcon(mimeType: string) {
  if (mimeType === 'application/pdf') return FileText;
  if (mimeType.includes('spreadsheet') || mimeType.includes('excel')) return FileSpreadsheet;
  if (mimeType.startsWith('image/')) return FileImage;
  return File;
}

export function FilePreview({ file, onRemove }: FilePreviewProps) {
  const Icon = getFileIcon(file.type);

  return (
    <div className="flex items-center gap-4 p-4 bg-muted rounded-lg">
      <Icon className="w-10 h-10 text-primary" aria-hidden="true" />
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
        <X className="w-4 h-4" />
      </Button>
    </div>
  );
}
```

---

### NEO-1.3-004: Implement File Storage Utility

**Priority**: P0 (Critical)
**Effort**: 30 minutes
**Dependencies**: None

**Description**:
Create file storage utility that saves uploaded files to the `/data/docling-uploads/` directory with YYYY/MM/DD organization structure.

**Acceptance Criteria**:
- [ ] Files stored in `/data/docling-uploads/YYYY/MM/DD/` structure
- [ ] Unique filename generation (UUID prefix)
- [ ] Directory creation if not exists
- [ ] File write with proper error handling
- [ ] Return stored file path
- [ ] Cleanup utility for orphaned files

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/lib/utils/storage.ts`

**Technical Notes**:
```typescript
import { mkdir, writeFile, unlink } from 'fs/promises';
import { join } from 'path';
import { randomUUID } from 'crypto';

const UPLOAD_BASE_PATH = process.env.UPLOAD_PATH || '/data/docling-uploads';

export interface StoredFile {
  path: string;
  filename: string;
  originalName: string;
  size: number;
}

export async function storeFile(file: File): Promise<StoredFile> {
  const now = new Date();
  const year = now.getFullYear().toString();
  const month = String(now.getMonth() + 1).padStart(2, '0');
  const day = String(now.getDate()).padStart(2, '0');

  const dirPath = join(UPLOAD_BASE_PATH, year, month, day);
  await mkdir(dirPath, { recursive: true });

  const ext = file.name.split('.').pop() || '';
  const filename = `${randomUUID()}.${ext}`;
  const filePath = join(dirPath, filename);

  const buffer = Buffer.from(await file.arrayBuffer());
  await writeFile(filePath, buffer);

  return {
    path: filePath,
    filename,
    originalName: file.name,
    size: file.size,
  };
}

export async function deleteFile(filePath: string): Promise<void> {
  try {
    await unlink(filePath);
  } catch (error) {
    console.error(`Failed to delete file: ${filePath}`, error);
  }
}
```

---

### NEO-1.3-005: Create Upload API Route

**Priority**: P0 (Critical)
**Effort**: 60 minutes
**Dependencies**: NEO-1.3-002, NEO-1.3-004
**Reference**: API Spec 4.2

**Description**:
Implement the POST /api/v1/upload route handler for file uploads with comprehensive validation, storage, and job creation.

**Acceptance Criteria**:
- [ ] Accept multipart/form-data for file uploads
- [ ] Validate file type and size server-side
- [ ] Store file using storage utility
- [ ] Create Job record in database with PENDING status
- [ ] Return 201 Created with job details
- [ ] Include Location header per RFC 7231
- [ ] Include X-Request-ID header
- [ ] Return appropriate error codes (E001, E002, E401)
- [ ] Handle upload errors gracefully

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/app/api/v1/upload/route.ts`

**Technical Notes**:
```typescript
import { NextRequest, NextResponse } from 'next/server';
import { randomUUID } from 'crypto';
import { prisma } from '@/lib/db/prisma';
import { validateFile } from '@/lib/validation/file';
import { storeFile, deleteFile } from '@/lib/utils/storage';
import { getSession } from '@/lib/redis/session';

export async function POST(request: NextRequest) {
  const requestId = request.headers.get('X-Request-ID') || randomUUID();

  try {
    // Get session
    const sessionId = await getSession(request);
    if (!sessionId) {
      return NextResponse.json(
        { error: { code: 'E502', message: 'Invalid session', userMessage: 'Session invalid', retryable: false } },
        { status: 401, headers: { 'X-Request-ID': requestId } }
      );
    }

    // Parse form data
    const formData = await request.formData();
    const file = formData.get('file') as File | null;

    if (!file) {
      return NextResponse.json(
        { error: { code: 'E100', message: 'No file provided', userMessage: 'Please select a file', retryable: false } },
        { status: 400, headers: { 'X-Request-ID': requestId } }
      );
    }

    // Validate file
    const validation = validateFile(file);
    if (!validation.valid) {
      return NextResponse.json(
        { error: { code: validation.errors[0].code, message: validation.errors[0].message, userMessage: validation.errors[0].message, retryable: false } },
        { status: 400, headers: { 'X-Request-ID': requestId } }
      );
    }

    // Store file
    const storedFile = await storeFile(file);

    // Create job record
    const job = await prisma.job.create({
      data: {
        sessionId,
        status: 'PENDING',
        inputType: 'FILE',
        fileName: file.name,
        fileSize: file.size,
        filePath: storedFile.path,
        mimeType: file.type,
      },
    });

    return NextResponse.json(
      {
        jobId: job.id,
        fileId: storedFile.filename,
        fileName: file.name,
        fileSize: file.size,
        mimeType: file.type,
        status: 'PENDING',
      },
      {
        status: 201,
        headers: {
          'Location': `/api/v1/jobs/${job.id}`,
          'X-Request-ID': requestId,
        },
      }
    );
  } catch (error) {
    console.error('Upload error:', error);
    return NextResponse.json(
      { error: { code: 'E401', message: 'Database error', userMessage: 'Failed to process upload', retryable: true } },
      { status: 500, headers: { 'X-Request-ID': requestId } }
    );
  }
}
```

---

### NEO-1.3-006: Add Job Creation on Upload

**Priority**: P0 (Critical)
**Effort**: 25 minutes
**Dependencies**: NEO-1.3-005

**Description**:
Ensure job creation logic is properly integrated with upload, including proper status tracking and error handling.

**Acceptance Criteria**:
- [ ] Job created with correct sessionId
- [ ] Job status set to PENDING
- [ ] Job inputType set to FILE
- [ ] All file metadata stored (fileName, fileSize, filePath, mimeType)
- [ ] Job ID returned in response
- [ ] Job timestamps automatically set

**Deliverables**:
- Integration in `/home/agent0/hx-docling-ui/src/app/api/v1/upload/route.ts`

**Technical Notes**:
This task is integrated into NEO-1.3-005 but validates the job creation specifically:
```typescript
// Verify job creation fields
const job = await prisma.job.create({
  data: {
    sessionId,
    status: 'PENDING',
    inputType: 'FILE',
    fileName: file.name,
    fileSize: file.size,
    filePath: storedFile.path,
    mimeType: file.type,
    // createdAt and updatedAt auto-set by Prisma
  },
});

// Validate job was created
if (!job.id) {
  throw new Error('Job creation failed');
}
```

---

### NEO-1.3-007: Integrate react-hook-form into UploadForm

**Priority**: P1 (High)
**Effort**: 25 minutes
**Dependencies**: NEO-1.3-001, NEO-1.3-003

**Description**:
Create UploadForm component that wraps UploadZone with react-hook-form for comprehensive form state management and validation.

**Acceptance Criteria**:
- [ ] Form state managed by react-hook-form
- [ ] File selection updates form value
- [ ] Form submission triggers upload API
- [ ] Loading state during submission
- [ ] Error state displayed from API response
- [ ] Form reset on successful upload
- [ ] Disabled state during processing

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/components/upload/UploadForm.tsx`

**Technical Notes**:
```typescript
'use client';

import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { useState } from 'react';
import { UploadZone } from './UploadZone';
import { FilePreview } from './FilePreview';
import { Button } from '@/components/ui/button';
import { Form, FormField, FormItem, FormMessage } from '@/components/ui/form';
import { useDocumentStore } from '@/stores/documentStore';

const uploadFormSchema = z.object({
  file: z.instanceof(File).nullable(),
});

type UploadFormValues = z.infer<typeof uploadFormSchema>;

export function UploadForm() {
  const [isUploading, setIsUploading] = useState(false);
  const { setFile, isProcessing } = useDocumentStore();

  const form = useForm<UploadFormValues>({
    resolver: zodResolver(uploadFormSchema),
    defaultValues: { file: null },
  });

  const selectedFile = form.watch('file');

  const onSubmit = async (data: UploadFormValues) => {
    if (!data.file) return;

    setIsUploading(true);
    try {
      const formData = new FormData();
      formData.append('file', data.file);

      const response = await fetch('/api/v1/upload', {
        method: 'POST',
        body: formData,
      });

      if (!response.ok) {
        const error = await response.json();
        throw new Error(error.error?.message || 'Upload failed');
      }

      const result = await response.json();
      setFile(data.file, result.jobId);
      form.reset();
    } catch (error) {
      form.setError('file', {
        message: error instanceof Error ? error.message : 'Upload failed',
      });
    } finally {
      setIsUploading(false);
    }
  };

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
        <FormField
          control={form.control}
          name="file"
          render={({ field }) => (
            <FormItem>
              {selectedFile ? (
                <FilePreview
                  file={selectedFile}
                  onRemove={() => field.onChange(null)}
                />
              ) : (
                <UploadZone
                  onFileSelect={(file) => field.onChange(file)}
                  disabled={isProcessing || isUploading}
                />
              )}
              <FormMessage />
            </FormItem>
          )}
        />
        {selectedFile && (
          <Button
            type="submit"
            disabled={isProcessing || isUploading}
            className="w-full"
          >
            {isUploading ? 'Uploading...' : 'Upload & Process'}
          </Button>
        )}
      </form>
    </Form>
  );
}
```

---

### NEO-1.3-008: Apply Rate Limiting Middleware to Upload Route

**Priority**: P1 (High)
**Effort**: 15 minutes
**Dependencies**: NEO-1.3-005, Sprint 1.2 (Rate Limiting Middleware)

**Description**:
Apply the rate limiting middleware from Sprint 1.2 to the upload API route to enforce 10 requests per minute per session.

**Acceptance Criteria**:
- [ ] Rate limiting middleware applied to upload route
- [ ] 10 requests per minute per session enforced
- [ ] Rate limit exceeded returns 429 with E601 error
- [ ] X-RateLimit-* headers included in response
- [ ] Sliding window algorithm used

**Deliverables**:
- Update `/home/agent0/hx-docling-ui/src/app/api/v1/upload/route.ts`

**Technical Notes**:
```typescript
import { withRateLimit } from '@/lib/middleware/rate-limit';

async function uploadHandler(request: NextRequest) {
  // ... existing upload logic
}

export const POST = withRateLimit({
  limit: 10,
  windowMs: 60 * 1000, // 1 minute
  keyPrefix: 'upload',
})(uploadHandler);
```

---

### NEO-1.3-009: Style Components with shadcn/ui

**Priority**: P1 (High)
**Effort**: 20 minutes
**Dependencies**: NEO-1.3-001, NEO-1.3-003

**Description**:
Apply consistent styling to upload components using shadcn/ui Card, Button, and Progress components.

**Acceptance Criteria**:
- [ ] UploadZone wrapped in Card component
- [ ] Consistent button styling
- [ ] Progress indicator during upload
- [ ] Dark mode compatible
- [ ] Responsive layout (mobile-first)

**Deliverables**:
- Updates to upload components with shadcn/ui styling

**Technical Notes**:
- Use Card for container styling
- Use Button variants for different states
- Use Skeleton for loading states
- Ensure all styles support dark mode via CSS variables

---

### NEO-1.3-010: Add Accessibility (ARIA, Keyboard Navigation)

**Priority**: P1 (High)
**Effort**: 20 minutes
**Dependencies**: NEO-1.3-001, NEO-1.3-003

**Description**:
Ensure all upload components meet WCAG 2.1 AA accessibility standards with proper ARIA labels, keyboard navigation, and focus management.

**Acceptance Criteria**:
- [ ] UploadZone has role="button" and aria-label
- [ ] File input has accessible label
- [ ] Keyboard navigation works (Tab, Enter, Space)
- [ ] Focus visible indicator on interactive elements
- [ ] Screen reader announces file selection
- [ ] Error messages announced via aria-live
- [ ] Color contrast meets 4.5:1 ratio

**Deliverables**:
- Accessibility improvements to all upload components

**Technical Notes**:
```typescript
// Accessibility checklist
// 1. ARIA labels on all interactive elements
// 2. role="button" on dropzone
// 3. aria-disabled when disabled
// 4. aria-live="polite" for status updates
// 5. Focus trap during modal interactions
// 6. Escape key to cancel/close
```

---

### NEO-1.3-011: Write Unit Tests for UploadZone and FilePreview

**Priority**: P1 (High)
**Effort**: 30 minutes
**Dependencies**: NEO-1.3-001, NEO-1.3-003

**Description**:
Write comprehensive unit tests for UploadZone and FilePreview components covering user interactions, validation, and edge cases.

**Acceptance Criteria**:
- [ ] Test drag-and-drop file selection
- [ ] Test click-to-browse file selection
- [ ] Test visual feedback during drag
- [ ] Test file rejection for invalid types
- [ ] Test file rejection for oversized files
- [ ] Test FilePreview displays correct info
- [ ] Test remove button functionality
- [ ] Tests achieve 80%+ coverage

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/components/upload/UploadZone.test.tsx`
- `/home/agent0/hx-docling-ui/src/components/upload/FilePreview.test.tsx`

**Technical Notes**:
```typescript
// UploadZone.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { describe, it, expect, vi } from 'vitest';
import { UploadZone } from './UploadZone';

describe('UploadZone', () => {
  it('renders upload instructions', () => {
    render(<UploadZone onFileSelect={vi.fn()} />);
    expect(screen.getByText(/drag & drop/i)).toBeInTheDocument();
  });

  it('calls onFileSelect when file is dropped', async () => {
    const onFileSelect = vi.fn();
    render(<UploadZone onFileSelect={onFileSelect} />);

    const dropzone = screen.getByRole('button');
    const file = new File(['test'], 'test.pdf', { type: 'application/pdf' });

    fireEvent.drop(dropzone, {
      dataTransfer: { files: [file] },
    });

    expect(onFileSelect).toHaveBeenCalledWith(file);
  });

  it('shows drag active state', () => {
    render(<UploadZone onFileSelect={vi.fn()} />);
    const dropzone = screen.getByRole('button');

    fireEvent.dragEnter(dropzone);
    expect(screen.getByText(/drop the file/i)).toBeInTheDocument();
  });

  it('disables when disabled prop is true', () => {
    render(<UploadZone onFileSelect={vi.fn()} disabled />);
    const dropzone = screen.getByRole('button');
    expect(dropzone).toHaveAttribute('tabIndex', '-1');
  });
});
```

---

### NEO-1.3-012: Write Unit Tests for File Validation

**Priority**: P1 (High)
**Effort**: 15 minutes
**Dependencies**: NEO-1.3-002

**Description**:
Write unit tests for file validation logic covering all file types and size scenarios.

**Acceptance Criteria**:
- [ ] Test all allowed file extensions
- [ ] Test all allowed MIME types
- [ ] Test rejection of unsupported extensions
- [ ] Test size limits per document type
- [ ] Test edge cases (empty files, very large files)
- [ ] Tests achieve 100% coverage of validation module

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/lib/validation/file.test.ts`

**Technical Notes**:
```typescript
// file.test.ts
import { describe, it, expect } from 'vitest';
import { validateFile, ALLOWED_EXTENSIONS, FILE_SIZE_LIMITS } from './file';

describe('validateFile', () => {
  describe('file type validation', () => {
    it.each(ALLOWED_EXTENSIONS)('accepts %s files', (ext) => {
      const file = new File(['test'], `test${ext}`, {
        type: getMimeType(ext),
      });
      const result = validateFile(file);
      expect(result.valid).toBe(true);
    });

    it('rejects unsupported file types', () => {
      const file = new File(['test'], 'test.exe', {
        type: 'application/x-msdownload',
      });
      const result = validateFile(file);
      expect(result.valid).toBe(false);
      expect(result.errors[0].code).toBe('E002');
    });
  });

  describe('file size validation', () => {
    it('accepts files within size limit', () => {
      const file = new File(['x'.repeat(1024)], 'test.pdf', {
        type: 'application/pdf',
      });
      Object.defineProperty(file, 'size', { value: 1024 });
      const result = validateFile(file);
      expect(result.valid).toBe(true);
    });

    it('rejects PDF files over 100MB', () => {
      const file = new File(['test'], 'test.pdf', {
        type: 'application/pdf',
      });
      Object.defineProperty(file, 'size', { value: 101 * 1024 * 1024 });
      const result = validateFile(file);
      expect(result.valid).toBe(false);
      expect(result.errors[0].code).toBe('E001');
    });
  });
});
```

---

## Summary

| Task ID | Task Title | Effort | Priority |
|---------|-----------|--------|----------|
| NEO-1.3-001 | Create UploadZone with react-dropzone | 45m | P0 |
| NEO-1.3-002 | Implement File Validation Schema | 30m | P0 |
| NEO-1.3-003 | Create FilePreview Component | 25m | P1 |
| NEO-1.3-004 | Implement File Storage Utility | 30m | P0 |
| NEO-1.3-005 | Create Upload API Route | 60m | P0 |
| NEO-1.3-006 | Add Job Creation on Upload | 25m | P0 |
| NEO-1.3-007 | Integrate react-hook-form | 25m | P1 |
| NEO-1.3-008 | Apply Rate Limiting Middleware | 15m | P1 |
| NEO-1.3-009 | Style Components with shadcn/ui | 20m | P1 |
| NEO-1.3-010 | Add Accessibility | 20m | P1 |
| NEO-1.3-011 | Write Unit Tests for Components | 30m | P1 |
| NEO-1.3-012 | Write Unit Tests for Validation | 15m | P1 |

**Total Effort**: ~3.5 hours (210 minutes)
**Total Tasks**: 12

---

## Dependencies Graph

```
Sprint 1.2 Complete
    |
    +-> NEO-1.3-001 (UploadZone)
    |       |
    |       +-> NEO-1.3-003 (FilePreview)
    |       |       |
    |       |       +-> NEO-1.3-007 (react-hook-form)
    |       |       +-> NEO-1.3-009 (Styling)
    |       |       +-> NEO-1.3-010 (Accessibility)
    |       |       +-> NEO-1.3-011 (Component Tests)
    |       |
    |       +-> NEO-1.3-007 (react-hook-form)
    |
    +-> NEO-1.3-002 (File Validation)
    |       |
    |       +-> NEO-1.3-005 (Upload API)
    |       |       |
    |       |       +-> NEO-1.3-006 (Job Creation)
    |       |       +-> NEO-1.3-008 (Rate Limiting)
    |       |
    |       +-> NEO-1.3-012 (Validation Tests)
    |
    +-> NEO-1.3-004 (Storage Utility)
            |
            +-> NEO-1.3-005 (Upload API)
```

---

## Parallel Execution Opportunities

**Parallel Group A** (Independent tasks):
- [P] NEO-1.3-001 (UploadZone)
- [P] NEO-1.3-002 (File Validation)
- [P] NEO-1.3-004 (Storage Utility)

**Parallel Group B** (After Group A):
- [P] NEO-1.3-003 (FilePreview) - depends on 001
- [P] NEO-1.3-012 (Validation Tests) - depends on 002

**Parallel Group C** (After NEO-1.3-005):
- [P] NEO-1.3-006 (Job Creation)
- [P] NEO-1.3-008 (Rate Limiting)

**Parallel Group D** (After NEO-1.3-003):
- [P] NEO-1.3-009 (Styling)
- [P] NEO-1.3-010 (Accessibility)
- [P] NEO-1.3-011 (Component Tests)
