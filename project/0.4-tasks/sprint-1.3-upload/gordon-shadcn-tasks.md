# Tasks: shadcn/ui Components for Upload System (Sprint 1.3)

**Agent**: Gordon Zain (`@gordon`) - shadcn/ui Specialist
**Sprint**: 1.3 - File Upload System
**Duration**: ~1.0h
**Prerequisites**: Sprint 1.1 complete, shadcn/ui components installed

## MCP Integration

**hx-shadcn MCP Server**: Use the MCP server for programmatic component access during development.

**Service Reference**: `project/0.8-references/hx-shadcn-service-descriptor.md`
**Endpoint**: `http://hx-shadcn.hx.dev.local:7423/sse`

**MCP Tools for This Sprint**:
| Tool | Usage in Sprint 1.3 |
|------|---------------------|
| `get_component` | Retrieve Card, Button, Form component source for styling reference |
| `get_component_demo` | View usage examples for Form + react-hook-form integration |
| `get_component_metadata` | Review Form component props for validation patterns |

**Example: Retrieve Form Component Metadata**
```json
{
  "method": "tools/call",
  "params": {
    "name": "get_component_metadata",
    "arguments": {"componentName": "form"}
  }
}
```

**Development Notes**:
- Components already installed in Sprint 1.1 (Card, Button, Form)
- MCP server provides reference implementations for styling patterns
- Use `get_component_demo` to view react-hook-form integration examples

## Task List

### GOR-1.3-001: Style UploadZone with shadcn/ui Card Component
**Status**: Pending
**Effort**: 20 minutes
**Dependencies**: Sprint 1.3 Task 1 (UploadZone component created)
**Priority**: P1 - High

#### Description
Apply shadcn/ui Card component to the UploadZone with proper styling for drag-drop states (idle, hover, active, error). Ensure consistent design with the application theme and dark mode support.

#### Acceptance Criteria
- [ ] Card component wraps UploadZone content
- [ ] Border styling changes on drag hover (dashed → solid, color change)
- [ ] Background color transitions smoothly on state changes
- [ ] Error state shows destructive color variant
- [ ] Success state shows success color (green tint)
- [ ] Card padding and spacing match design spec
- [ ] Dark mode styling works correctly
- [ ] Accessibility: ARIA labels describe drag-drop region

#### Technical Implementation
```tsx
// src/components/upload/UploadZone.tsx
import { Card, CardContent } from '@/components/ui/card';
import { cn } from '@/lib/utils';

interface UploadZoneProps {
  onFileSelect: (file: File) => void;
  disabled?: boolean;
}

export function UploadZone({ onFileSelect, disabled }: UploadZoneProps) {
  const [isDragActive, setIsDragActive] = useState(false);
  const [error, setError] = useState<string | null>(null);

  // react-dropzone setup (implemented in Sprint 1.3 Task 1)
  const { getRootProps, getInputProps, isDragActive: dropzoneActive } = useDropzone({
    onDrop: handleDrop,
    accept: {
      'application/pdf': ['.pdf'],
      'application/vnd.openxmlformats-officedocument.wordprocessingml.document': ['.docx'],
      // ... other types
    },
    maxSize: 100 * 1024 * 1024, // 100MB
    disabled,
  });

  return (
    <Card
      {...getRootProps()}
      role="button"
      tabIndex={disabled ? -1 : 0}
      className={cn(
        'relative cursor-pointer transition-all duration-200',
        'border-2 border-dashed',
        // Idle state
        'border-border bg-background hover:bg-accent/5',
        // Drag active state
        isDragActive && 'border-primary bg-primary/5 ring-2 ring-primary/20',
        // Error state
        error && 'border-destructive bg-destructive/5',
        // Disabled state
        disabled && 'cursor-not-allowed opacity-50',
        // Focus visible for keyboard navigation
        'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2',
      )}
      aria-label="File upload zone. Drag and drop files or click to browse."
    >
      <CardContent className="flex flex-col items-center justify-center p-12 text-center">
        <input {...getInputProps()} />

        {/* Upload icon */}
        <UploadIcon className="mb-4 h-12 w-12 text-muted-foreground" />

        {/* Instructions */}
        <p className="text-lg font-medium text-foreground">
          {isDragActive ? 'Drop file here' : 'Drag and drop file'}
        </p>
        <p className="mt-1 text-sm text-muted-foreground">
          or click to browse
        </p>

        {/* Supported formats */}
        <p className="mt-4 text-xs text-muted-foreground">
          Supports: PDF, DOCX, XLSX, PPTX, Images (max 100MB)
        </p>

        {/* Error message */}
        {error && (
          <p className="mt-2 text-sm text-destructive" role="alert">
            {error}
          </p>
        )}
      </CardContent>
    </Card>
  );
}
```

**Styling States** (using CSS variables):
| State | Border | Background | Ring |
|-------|--------|------------|------|
| Idle | `border-border` | `bg-background` | None |
| Hover | `border-border` | `bg-accent/5` | None |
| Drag Active | `border-primary` | `bg-primary/5` | `ring-primary/20` |
| Error | `border-destructive` | `bg-destructive/5` | None |
| Disabled | `border-border` (50% opacity) | `bg-background` | None |

#### Validation
```bash
# Test in browser
npm run dev

# Visual checks:
# 1. Idle state: dashed border, neutral colors
# 2. Hover: slight background tint
# 3. Drag file over: solid blue border, blue tint
# 4. Drop invalid file: red border, error message
# 5. Toggle dark mode: all states adapt
```

#### Deliverables
- Styled UploadZone component using Card
- State-based styling with `cn()` utility
- Dark mode compatible

---

### GOR-1.3-002: Style FilePreview with shadcn/ui Card and Button
**Status**: Pending
**Effort**: 15 minutes
**Dependencies**: Sprint 1.3 Task 3 (FilePreview component created)
**Priority**: P1 - High

#### Description
Apply shadcn/ui Card and Button components to the FilePreview display, showing file metadata with a remove button. Ensure consistent styling with UploadZone.

#### Acceptance Criteria
- [ ] Card component displays file metadata
- [ ] File icon displayed based on file type
- [ ] File name, size, and type shown clearly
- [ ] Remove button uses Button component with destructive variant
- [ ] Hover state on remove button works
- [ ] Card matches UploadZone styling
- [ ] Dark mode styling works
- [ ] Accessibility: Remove button has ARIA label

#### Technical Implementation
```tsx
// src/components/upload/FilePreview.tsx
import { Card, CardContent } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { X, FileText, FileSpreadsheet, Presentation, Image as ImageIcon } from 'lucide-react';
import { cn } from '@/lib/utils';

interface FilePreviewProps {
  file: File;
  onRemove: () => void;
}

export function FilePreview({ file, onRemove }: FilePreviewProps) {
  const formatFileSize = (bytes: number): string => {
    if (bytes < 1024) return `${bytes} B`;
    if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`;
    return `${(bytes / (1024 * 1024)).toFixed(1)} MB`;
  };

  const getFileIcon = (type: string) => {
    if (type.includes('pdf')) return <FileText className="h-10 w-10 text-red-500" />;
    if (type.includes('word')) return <FileText className="h-10 w-10 text-blue-500" />;
    if (type.includes('spreadsheet') || type.includes('excel'))
      return <FileSpreadsheet className="h-10 w-10 text-green-500" />;
    if (type.includes('presentation') || type.includes('powerpoint'))
      return <Presentation className="h-10 w-10 text-orange-500" />;
    if (type.includes('image')) return <ImageIcon className="h-10 w-10 text-purple-500" />;
    return <FileText className="h-10 w-10 text-muted-foreground" />;
  };

  return (
    <Card className="relative">
      <CardContent className="flex items-center gap-4 p-4">
        {/* File icon */}
        {getFileIcon(file.type)}

        {/* File metadata */}
        <div className="flex-1 min-w-0">
          <p className="font-medium text-foreground truncate">
            {file.name}
          </p>
          <p className="text-sm text-muted-foreground">
            {formatFileSize(file.size)} • {file.type || 'Unknown type'}
          </p>
        </div>

        {/* Remove button */}
        <Button
          variant="ghost"
          size="icon"
          onClick={onRemove}
          aria-label={`Remove ${file.name}`}
          className="shrink-0 hover:bg-destructive/10 hover:text-destructive"
        >
          <X className="h-4 w-4" />
        </Button>
      </CardContent>
    </Card>
  );
}
```

**Icon Strategy** (using lucide-react):
- **PDF**: FileText with red color
- **DOCX**: FileText with blue color
- **XLSX**: FileSpreadsheet with green color
- **PPTX**: Presentation with orange color
- **Images**: ImageIcon with purple color

**Anti-Pattern Prevention**:
- ❌ **DO NOT** import entire lucide-react library: `import * as Icons from 'lucide-react'`
- ✅ **DO** use named imports for tree-shaking: `import { FileText, X } from 'lucide-react'`

#### Validation
```bash
# Test with different file types
# 1. Upload PDF → see red PDF icon
# 2. Upload DOCX → see blue document icon
# 3. Upload XLSX → see green spreadsheet icon
# 4. Upload image → see purple image icon
# 5. Click remove → file preview disappears
# 6. Toggle dark mode → card adapts
```

#### Deliverables
- Styled FilePreview component
- File type icon mapping
- Remove button with hover state

---

### GOR-1.3-003: Integrate shadcn/ui Form Component with react-hook-form
**Status**: Pending
**Effort**: 25 minutes
**Dependencies**: Sprint 1.3 Task 7 (react-hook-form integration)
**Priority**: P0 - Critical Path

#### Description
Wrap UploadZone and FilePreview in shadcn/ui Form component with react-hook-form integration. This provides type-safe form state management and validation error display, addressing **FE-M2** from the implementation plan.

#### Acceptance Criteria
- [ ] Form component wraps upload UI
- [ ] react-hook-form manages file state
- [ ] FormField components used for inputs
- [ ] Validation errors display with FormMessage
- [ ] Form submission handled via react-hook-form
- [ ] Loading state disabled form during submission
- [ ] TypeScript types enforce form schema
- [ ] No `@ts-ignore` or `any` types used

#### Technical Implementation
```tsx
// src/components/upload/UploadForm.tsx
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
} from '@/components/ui/form';
import { Button } from '@/components/ui/button';
import { UploadZone } from './UploadZone';
import { FilePreview } from './FilePreview';

// Zod schema for file upload validation
const uploadFormSchema = z.object({
  file: z.custom<File>(
    (val) => val instanceof File,
    { message: 'Please select a file' }
  ).refine(
    (file) => file.size <= 100 * 1024 * 1024,
    { message: 'File size must be less than 100MB (E001)' }
  ).refine(
    (file) => {
      const validTypes = [
        'application/pdf',
        'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
        'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
        'application/vnd.openxmlformats-officedocument.presentationml.presentation',
        'image/jpeg',
        'image/png',
      ];
      return validTypes.includes(file.type);
    },
    { message: 'Invalid file type (E002)' }
  ),
});

type UploadFormValues = z.infer<typeof uploadFormSchema>;

export function UploadForm() {
  const form = useForm<UploadFormValues>({
    resolver: zodResolver(uploadFormSchema),
    defaultValues: {
      file: undefined,
    },
  });

  const onSubmit = async (data: UploadFormValues) => {
    // Sprint 1.3 Task 5: Upload API integration
    const formData = new FormData();
    formData.append('file', data.file);

    try {
      const response = await fetch('/api/v1/upload', {
        method: 'POST',
        body: formData,
      });

      if (!response.ok) {
        throw new Error('Upload failed');
      }

      const result = await response.json();
      console.log('Job created:', result.jobId);
    } catch (error) {
      form.setError('file', {
        message: 'Upload failed. Please try again.',
      });
    }
  };

  const selectedFile = form.watch('file');

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-6">
        <FormField
          control={form.control}
          name="file"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Upload Document</FormLabel>
              <FormControl>
                {selectedFile ? (
                  <FilePreview
                    file={selectedFile}
                    onRemove={() => field.onChange(undefined)}
                  />
                ) : (
                  <UploadZone
                    onFileSelect={field.onChange}
                    disabled={form.formState.isSubmitting}
                  />
                )}
              </FormControl>
              <FormMessage /> {/* Displays validation errors */}
            </FormItem>
          )}
        />

        <Button
          type="submit"
          disabled={!selectedFile || form.formState.isSubmitting}
          className="w-full"
        >
          {form.formState.isSubmitting ? 'Processing...' : 'Process Document'}
        </Button>
      </form>
    </Form>
  );
}
```

**Type Safety Benefits**:
- `UploadFormValues` inferred from Zod schema
- No manual type definitions needed
- TypeScript errors if form fields don't match schema
- Automatic validation before submission

**Validation Flow**:
1. User selects file → Zod validates size and type
2. Validation fails → FormMessage displays error
3. Validation passes → Submit button enabled
4. User clicks submit → `onSubmit` handler called
5. Upload fails → `setError` displays server error

#### Validation
```bash
# Test validation scenarios
# 1. No file selected → "Please select a file"
# 2. File > 100MB → "File size must be less than 100MB (E001)"
# 3. Invalid file type → "Invalid file type (E002)"
# 4. Valid file → Submit button enabled
# 5. Submit → Loading state, button disabled
```

#### Deliverables
- UploadForm component with react-hook-form
- Zod validation schema
- Type-safe form state management

---

### GOR-1.3-004: Add Accessibility Features to Upload Components
**Status**: Pending
**Effort**: 20 minutes
**Dependencies**: GOR-1.3-001, GOR-1.3-002, GOR-1.3-003
**Priority**: P1 - High

#### Description
Ensure all upload components meet WCAG 2.1 AA accessibility standards with proper ARIA attributes, keyboard navigation, and screen reader support.

#### Acceptance Criteria
- [ ] UploadZone has `role="button"` and `aria-label`
- [ ] File input has `aria-describedby` for instructions
- [ ] Keyboard Enter/Space triggers file picker
- [ ] Focus visible on UploadZone when keyboard navigating
- [ ] Error messages have `role="alert"` and are announced by screen readers
- [ ] Remove button has descriptive `aria-label` with file name
- [ ] Tab order is logical (UploadZone → Remove → Submit)
- [ ] Color contrast meets AA standards (4.5:1 for text)
- [ ] Screen reader announces file selection
- [ ] Keyboard users can remove selected file (Tab to Remove, Enter to confirm)

#### Technical Implementation
```tsx
// Enhanced UploadZone with accessibility
<Card
  {...getRootProps()}
  role="button"
  tabIndex={disabled ? -1 : 0}
  aria-label="File upload zone. Drag and drop files or click to browse."
  aria-describedby="upload-instructions"
  onKeyDown={(e) => {
    if (e.key === 'Enter' || e.key === ' ') {
      e.preventDefault();
      open(); // Trigger file picker
    }
  }}
  className={cn(
    'relative cursor-pointer transition-all duration-200',
    'border-2 border-dashed',
    'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2',
    // ... other classes
  )}
>
  <CardContent className="flex flex-col items-center justify-center p-12 text-center">
    <input {...getInputProps()} aria-label="File input" />

    <UploadIcon className="mb-4 h-12 w-12 text-muted-foreground" aria-hidden="true" />

    <p id="upload-instructions" className="text-lg font-medium text-foreground">
      {isDragActive ? 'Drop file here' : 'Drag and drop file'}
    </p>
    <p className="mt-1 text-sm text-muted-foreground">
      or click to browse
    </p>

    {error && (
      <p className="mt-2 text-sm text-destructive" role="alert" aria-live="assertive">
        {error}
      </p>
    )}
  </CardContent>
</Card>

// Enhanced FilePreview with accessibility
<Button
  variant="ghost"
  size="icon"
  onClick={onRemove}
  aria-label={`Remove ${file.name}`}
  className="shrink-0 hover:bg-destructive/10 hover:text-destructive"
>
  <X className="h-4 w-4" aria-hidden="true" />
</Button>

// Announce file selection to screen readers
useEffect(() => {
  if (selectedFile) {
    const announcement = document.createElement('div');
    announcement.setAttribute('role', 'status');
    announcement.setAttribute('aria-live', 'polite');
    announcement.className = 'sr-only';
    announcement.textContent = `File selected: ${selectedFile.name}, ${formatFileSize(selectedFile.size)}`;
    document.body.appendChild(announcement);

    setTimeout(() => document.body.removeChild(announcement), 1000);
  }
}, [selectedFile]);
```

**Accessibility Checklist**:
- [x] Keyboard navigation (Tab, Enter, Space)
- [x] Screen reader announcements (file selection, errors)
- [x] Focus indicators (ring on UploadZone)
- [x] ARIA labels (descriptive button labels)
- [x] ARIA live regions (errors, status updates)
- [x] Color contrast (4.5:1 minimum)
- [x] Logical tab order
- [x] Semantic HTML (buttons, forms)

#### Validation
```bash
# Manual accessibility testing
# 1. Navigate with Tab key → UploadZone receives focus
# 2. Press Enter on UploadZone → File picker opens
# 3. Select file → Screen reader announces file name
# 4. Tab to Remove button → Focus visible
# 5. Press Enter on Remove → File removed, announced
# 6. Upload invalid file → Error announced immediately

# Automated testing
npm run test:a11y  # Run axe-core accessibility tests
```

#### Deliverables
- ARIA-enhanced UploadZone
- Keyboard navigation support
- Screen reader announcements
- Accessibility documentation

---

## Summary

**Total Tasks**: 4
**Total Effort**: 1.33 hours
**Critical Path**: GOR-1.3-001 → GOR-1.3-003

**Success Criteria**:
- [ ] UploadZone styled with Card component
- [ ] FilePreview styled with Card and Button
- [ ] react-hook-form integrated with Form component
- [ ] WCAG 2.1 AA accessibility compliance
- [ ] Zero TypeScript errors
- [ ] Dark mode styling works
- [ ] Validation errors display correctly

**Coordination Points**:
- **Neo (Sprint 1.3 Lead)**: Coordinate UploadZone component implementation (Task 1)
- **Ola (Accessibility)**: Review accessibility features in GOR-1.3-004
- **William (TailwindCSS)**: Verify custom color utilities if needed

**Anti-Patterns Prevented**:
- ✅ Named imports for lucide-react icons (tree-shaking)
- ✅ Zod schemas for type-safe validation (no `any` types)
- ✅ CSS variables for colors (not hardcoded)
- ✅ Proper ARIA attributes (accessibility)
- ✅ FormMessage for error display (consistent UX)

**Notes**:
- Form component uses shadcn/ui + react-hook-form pattern
- Zod schema aligns with backend validation (coordinate with Paul Warfield)
- File type icons use lucide-react (already in dependencies)
- Accessibility features ensure WCAG 2.1 AA compliance
