# API Tasks: Sprint 1.3 - File Upload System

**Sprint**: 1.3 - File Upload System
**Author**: Bob Martin (@bob) - API Design SME
**Created**: 2025-12-12
**Status**: PENDING
**Dependencies**: Sprint 1.2 Complete

---

## Overview

This task file covers API-related deliverables for Sprint 1.3, focusing on the file upload endpoint implementation with comprehensive validation, rate limiting, and proper response handling.

---

## Tasks

### BOB-1.3-001: Implement Upload API Route

| Field | Value |
|-------|-------|
| **Task ID** | BOB-1.3-001 |
| **Title** | Create POST /api/v1/upload endpoint for file uploads |
| **Priority** | P0 (Critical) |
| **Effort** | 60 minutes |
| **Dependencies** | BOB-1.2-002 (Validation Middleware), BOB-1.2-003 (Rate Limiting) |

#### Description

Implement the file upload API endpoint that accepts multipart/form-data file uploads. The endpoint must validate file type, size, and name, create a Job record in the database, store the file to persistent storage, and return appropriate response with Location header per RFC 7231.

#### Acceptance Criteria

- [ ] Endpoint accessible at `POST /api/v1/upload`
- [ ] Accepts `multipart/form-data` with file field
- [ ] Validates file type against allowed extensions (PDF, DOCX, XLSX, PPTX, images)
- [ ] Validates file size against type-specific limits (100MB PDF, 50MB docs, 25MB images)
- [ ] Validates filename for dangerous characters
- [ ] Creates Job record in PENDING status
- [ ] Stores file to `/data/docling-uploads/YYYY/MM/DD/{uuid}.{ext}`
- [ ] Returns 201 Created with Location header
- [ ] Returns X-Request-ID header (UUID v4)
- [ ] Integrates rate limiting middleware
- [ ] Returns structured errors (E001, E002) on validation failure

#### Technical Notes

```typescript
// File: src/app/api/v1/upload/route.ts

// Request: multipart/form-data with 'file' field
// Response: 201 Created
{
  "data": {
    "jobId": "uuid-v4",
    "fileId": "uuid-v4",
    "fileName": "document.pdf",
    "fileSize": 1048576,
    "mimeType": "application/pdf",
    "status": "PENDING"
  }
}

// Response Headers:
// Location: /api/v1/jobs/{jobId}
// X-Request-ID: uuid-v4
// Content-Type: application/json
```

**File Validation Schema**:
```typescript
const ALLOWED_EXTENSIONS = [
  '.pdf',
  '.doc', '.docx',
  '.ppt', '.pptx',
  '.xls', '.xlsx',
  '.png', '.jpg', '.jpeg', '.tiff'
];

const SIZE_LIMITS: Record<string, number> = {
  '.pdf': 100 * 1024 * 1024,      // 100 MB
  '.doc': 50 * 1024 * 1024,       // 50 MB
  '.docx': 50 * 1024 * 1024,
  '.ppt': 50 * 1024 * 1024,
  '.pptx': 50 * 1024 * 1024,
  '.xls': 50 * 1024 * 1024,
  '.xlsx': 50 * 1024 * 1024,
  '.png': 25 * 1024 * 1024,       // 25 MB
  '.jpg': 25 * 1024 * 1024,
  '.jpeg': 25 * 1024 * 1024,
  '.tiff': 25 * 1024 * 1024,
};
```

#### Deliverables

- `src/app/api/v1/upload/route.ts` - Upload endpoint
- `src/lib/validation/file.ts` - File validation logic (coordinate with Neo)
- `src/lib/utils/storage.ts` - File storage utilities (coordinate with Neo)

---

### BOB-1.3-002: Implement URL Upload Variant

| Field | Value |
|-------|-------|
| **Task ID** | BOB-1.3-002 |
| **Title** | Extend upload endpoint to accept URL input |
| **Priority** | P0 (Critical) |
| **Effort** | 30 minutes |
| **Dependencies** | BOB-1.3-001, SSRF validation from Sprint 1.4 |

#### Description

Extend the upload endpoint to accept JSON body with URL for URL-based document processing. This shares the same endpoint but with different content type.

#### Acceptance Criteria

- [ ] Same endpoint accepts `application/json` with URL field
- [ ] URL validated for format (HTTP/HTTPS only)
- [ ] SSRF prevention blocks private IPs (delegated to validation lib)
- [ ] Creates Job record with inputType: URL
- [ ] Returns same response structure as file upload
- [ ] URL max length: 2048 characters

#### Technical Notes

```typescript
// Request: application/json
{
  "url": "https://example.com/document.html"
}

// URL Validation Schema
const urlUploadSchema = z.object({
  url: z.string()
    .url()
    .max(2048)
    .refine((url) => {
      const protocol = new URL(url).protocol;
      return protocol === 'http:' || protocol === 'https:';
    }, { message: 'Only HTTP and HTTPS URLs are allowed' })
    .refine((url) => !isURLBlocked(url), {
      message: 'URL is blocked for security reasons'
    }),
});
```

#### Deliverables

- Updated `src/app/api/v1/upload/route.ts` - URL handling branch
- `src/lib/validation/url.ts` - URL validation (coordinate with Neo)

---

### BOB-1.3-003: Apply Rate Limiting to Upload Route

| Field | Value |
|-------|-------|
| **Task ID** | BOB-1.3-003 |
| **Title** | Integrate rate limiting middleware with upload endpoint |
| **Priority** | P1 (Critical) |
| **Effort** | 15 minutes |
| **Dependencies** | BOB-1.2-003 (Rate Limit Middleware), BOB-1.3-001 |

#### Description

Apply the sliding window rate limiting middleware from Sprint 1.2 to the upload endpoint to protect against abuse and DoS attacks.

#### Acceptance Criteria

- [ ] Upload endpoint rate limited to 10 requests/minute/session
- [ ] Rate limit headers returned on all responses
- [ ] 429 returned with E601 error when limit exceeded
- [ ] Retry-After header included in 429 responses

#### Technical Notes

```typescript
// Wrap the upload handler
export const POST = withRateLimit(
  { windowMs: 60000, maxRequests: 10 },
  withValidation(
    { body: uploadSchema },
    uploadHandler
  )
);
```

#### Deliverables

- Updated `src/app/api/v1/upload/route.ts` with rate limiting

---

### BOB-1.3-004: Implement Upload Response Headers

| Field | Value |
|-------|-------|
| **Task ID** | BOB-1.3-004 |
| **Title** | Ensure RFC-compliant response headers for upload |
| **Priority** | P1 (Critical) |
| **Effort** | 15 minutes |
| **Dependencies** | BOB-1.3-001 |

#### Description

Implement all required response headers for the upload endpoint as specified in the API conventions (Section 4.8).

#### Acceptance Criteria

- [ ] `Location` header points to job resource URL
- [ ] `X-Request-ID` header contains UUID v4 (echo client or generate)
- [ ] `X-RateLimit-*` headers included
- [ ] `Content-Type: application/json` always set
- [ ] `Cache-Control: no-store` for non-cacheable response

#### Technical Notes

```typescript
// Response header utility
function createUploadResponse(
  data: UploadResponseData,
  requestId: string
): NextResponse {
  return NextResponse.json(
    { data },
    {
      status: 201,
      headers: {
        'Location': `/api/v1/jobs/${data.jobId}`,
        'X-Request-ID': requestId,
        'Cache-Control': 'no-store',
        'Content-Type': 'application/json',
      },
    }
  );
}
```

#### Deliverables

- `src/lib/utils/response.ts` - Response helper utilities
- Updated upload route with proper headers

---

## Sprint Summary

| Task ID | Title | Effort | Priority |
|---------|-------|--------|----------|
| BOB-1.3-001 | Upload API Route (File) | 60m | P0 |
| BOB-1.3-002 | Upload API Route (URL) | 30m | P0 |
| BOB-1.3-003 | Rate Limiting Integration | 15m | P1 |
| BOB-1.3-004 | Response Headers | 15m | P1 |

**Total Effort**: 2 hours

---

## Dependencies Graph

```
Sprint 1.2 Complete
        |
        +-- BOB-1.2-002 (Validation)
        |
        +-- BOB-1.2-003 (Rate Limiting)
        |
        v
BOB-1.3-001 (Upload File)
        |
        +----> BOB-1.3-003 (Rate Limit Integration)
        |
        +----> BOB-1.3-004 (Response Headers)
        |
        v
BOB-1.3-002 (Upload URL)
```

---

## Coordination Notes

- **Neo (@neo)**: Coordinate on file validation schema and storage utilities
- **William (@william)**: Ensure database Job creation follows Prisma patterns
- **Ola (@ola)**: Align error response format for UI error handling

---

## Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| E001 | 400/413 | File too large |
| E002 | 400 | Unsupported file type |
| E003 | 400 | Invalid filename characters |
| E101 | 400 | Invalid URL format |
| E104 | 400 | URL blocked (SSRF) |
| E601 | 429 | Rate limit exceeded |
| E401 | 500 | Database error |
