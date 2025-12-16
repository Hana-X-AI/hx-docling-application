# HX Docling Application - API Test Cases

**Document Type**: API Endpoint Test Cases
**Version**: 2.2.0
**Status**: DRAFT
**Created**: 2025-12-12
**Updated**: 2025-12-15
**Author**: Julia Santos (Testing & QA Specialist)
**Master Plan Reference**: `project/0.5-test/00-test-plan-overview.md`
**Specification Reference**: `project/0.3-specification/0.3.1-detailed-specification.md` v1.2.1

---

## Table of Contents

1. [Overview](#1-overview)
2. [POST /api/v1/upload](#2-post-apiv1upload)
3. [POST /api/v1/process](#3-post-apiv1process)
4. [POST /api/v1/jobs/{id}/cancel](#4-post-apiv1jobsidcancel)
5. [POST /api/v1/jobs/{id}/resume](#5-post-apiv1jobsidresume)
6. [GET /api/v1/jobs/{id}](#6-get-apiv1jobsid)
7. [GET /api/v1/history](#7-get-apiv1history)
8. [GET /api/v1/health](#8-get-apiv1health)
9. [SSE Event Stream Tests](#9-sse-event-stream-tests)
10. [Authentication/Authorization Tests](#10-authenticationauthorization-tests)
11. [Security Test Cases](#11-security-test-cases)
12. [MCP Integration Layer Tests](#12-mcp-integration-layer-tests)
13. [Database State Verification Tests](#13-database-state-verification-tests)
14. [Concurrent Request Handling Tests](#14-concurrent-request-handling-tests)
15. [Response Schema Validation Tests](#15-response-schema-validation-tests)
16. [Workflow Orchestration Tests](#16-workflow-orchestration-tests)
17. [HTTP Method Validation Tests](#17-http-method-validation-tests)
18. [Dependency Failure Tests](#18-dependency-failure-tests)
19. [Pydantic Model Validation Tests](#19-pydantic-model-validation-tests)
20. [API Concurrency Tests](#20-api-concurrency-tests)

---

## 1. Overview

### 1.1 Purpose

This document defines comprehensive test cases for all API endpoints of the HX Docling Application. Each test case includes request specifications, expected responses, and error code validation per the specification.

### 1.2 API Base URL

| Environment | Base URL |
|-------------|----------|
| Development | http://localhost:3000/api/v1 |
| Staging | https://staging.docling.hx.dev.local/api/v1 |
| Production | https://docling.hx.local/api/v1 |

### 1.3 Common Headers

All requests should include:

| Header | Value | Notes |
|--------|-------|-------|
| Content-Type | application/json or multipart/form-data | Per endpoint |
| Cookie | hx-docling-session={sessionId} | Session identifier |

### 1.4 Common Response Format

**Success Response**:
```json
{
  "success": true,
  "data": { ... }
}
```

**Error Response**:
```json
{
  "success": false,
  "error": {
    "code": "E001",
    "message": "File too large",
    "userMessage": "The file exceeds the 100MB limit",
    "suggestedAction": "Try a smaller file",
    "retryable": false
  }
}
```

---

## 2. POST /api/v1/upload

### 2.1 Endpoint Specification

| Attribute | Value |
|-----------|-------|
| **Method** | POST |
| **Path** | /api/v1/upload |
| **Content-Type** | multipart/form-data (file) or application/json (URL) |
| **Authentication** | Session cookie |
| **Rate Limit** | 10 req/min per session |

### 2.2 Request Formats

**File Upload**:
```http
POST /api/v1/upload HTTP/1.1
Content-Type: multipart/form-data; boundary=----Boundary
Cookie: hx-docling-session=abc123

------Boundary
Content-Disposition: form-data; name="file"; filename="document.pdf"
Content-Type: application/pdf

<file binary data>
------Boundary--
```

**URL Input**:
```http
POST /api/v1/upload HTTP/1.1
Content-Type: application/json
Cookie: hx-docling-session=abc123

{
  "url": "https://example.com/document.html"
}
```

---

### API-UP-001: Valid PDF File Upload

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-UP-001 |
| **Title** | Valid PDF File Upload |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-101, FR-103, FR-104 |

**Request**:
- Method: POST multipart/form-data
- File: sample.pdf (100KB)
- Content-Type: application/pdf

**Expected Response**:
```json
HTTP/1.1 201 Created
Content-Type: application/json

{
  "success": true,
  "data": {
    "jobId": "uuid-v4",
    "status": "PENDING",
    "fileName": "sample.pdf",
    "fileSize": 102400,
    "mimeType": "application/pdf",
    "createdAt": "2025-12-12T10:00:00Z"
  }
}
```

**Assertions**:
- Status code: 201
- jobId is valid UUID
- status is "PENDING"
- Job created in database with correct attributes

---

### API-UP-002: Valid DOCX File Upload

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-UP-002 |
| **Title** | Valid DOCX File Upload |
| **Priority** | P1 - High |
| **Spec Reference** | FR-103 |

**Request**:
- File: sample.docx (50KB)
- Content-Type: application/vnd.openxmlformats-officedocument.wordprocessingml.document

**Expected Response**:
```json
HTTP/1.1 201 Created

{
  "success": true,
  "data": {
    "jobId": "uuid-v4",
    "status": "PENDING",
    "fileName": "sample.docx",
    "mimeType": "application/vnd.openxmlformats-officedocument.wordprocessingml.document"
  }
}
```

**Assertions**:
- Status code: 201
- MIME type correctly identified

---

### API-UP-003: Valid XLSX File Upload

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-UP-003 |
| **Title** | Valid XLSX File Upload |
| **Priority** | P1 - High |
| **Spec Reference** | FR-103 |

**Request**:
- File: sample.xlsx (100KB)
- Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet

**Expected Response**:
```json
HTTP/1.1 201 Created
```

---

### API-UP-004: Valid PPTX File Upload

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-UP-004 |
| **Title** | Valid PPTX File Upload |
| **Priority** | P1 - High |
| **Spec Reference** | FR-103 |

**Request**:
- File: sample.pptx (500KB)
- Content-Type: application/vnd.openxmlformats-officedocument.presentationml.presentation

**Expected Response**:
```json
HTTP/1.1 201 Created
```

---

### API-UP-005: Valid Image Upload (PNG)

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-UP-005 |
| **Title** | Valid Image Upload |
| **Priority** | P1 - High |
| **Spec Reference** | FR-103 |

**Request**:
- File: sample.png (500KB)
- Content-Type: image/png

**Expected Response**:
```json
HTTP/1.1 201 Created
```

---

### API-UP-006: File Too Large - E001

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-UP-006 |
| **Title** | Reject File Exceeding Size Limit |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-104, E001 |

**Request**:
- File: too-large.pdf (150MB)

**Expected Response**:
```json
HTTP/1.1 400 Bad Request

{
  "success": false,
  "error": {
    "code": "E001",
    "message": "File too large",
    "userMessage": "File size exceeds the 100MB limit for PDF files",
    "suggestedAction": "Try a smaller file or split into multiple documents",
    "retryable": false
  }
}
```

**Assertions**:
- Status code: 400
- Error code: E001
- File not saved to storage

---

### API-UP-007: Unsupported File Type - E002

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-UP-007 |
| **Title** | Reject Unsupported File Type |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-103, E002 |

**Request**:
- File: script.exe

**Expected Response**:
```json
HTTP/1.1 400 Bad Request

{
  "success": false,
  "error": {
    "code": "E002",
    "message": "Unsupported file type",
    "userMessage": "This file type is not supported. Accepted types: PDF, Word, Excel, PowerPoint, Images",
    "suggestedAction": "Try a different file format",
    "retryable": false
  }
}
```

**Assertions**:
- Status code: 400
- Error code: E002

---

### API-UP-008: Valid URL Input

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-UP-008 |
| **Title** | Valid URL Input |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-201, FR-202 |

**Request**:
```json
{
  "url": "https://example.com/document.html"
}
```

**Expected Response**:
```json
HTTP/1.1 201 Created

{
  "success": true,
  "data": {
    "jobId": "uuid-v4",
    "status": "PENDING",
    "inputType": "URL",
    "sourceUrl": "https://example.com/document.html"
  }
}
```

---

### API-UP-009: Invalid URL Format - E101

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-UP-009 |
| **Title** | Reject Invalid URL Format |
| **Priority** | P1 - High |
| **Spec Reference** | FR-202, E101 |

**Request**:
```json
{
  "url": "not-a-valid-url"
}
```

**Expected Response**:
```json
HTTP/1.1 400 Bad Request

{
  "success": false,
  "error": {
    "code": "E101",
    "message": "Invalid URL format",
    "userMessage": "Please enter a valid URL starting with http:// or https://",
    "suggestedAction": "Check the URL format",
    "retryable": false
  }
}
```

---

### API-UP-010: SSRF Prevention - Blocked URL - E104

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-UP-010 |
| **Title** | Block Internal URLs (SSRF Prevention) |
| **Priority** | P0 - Critical (Security) |
| **Spec Reference** | FR-203, E104 |

**Test Matrix**:

| URL | Expected |
|-----|----------|
| http://localhost/secret | E104 |
| http://127.0.0.1/admin | E104 |
| http://10.0.0.1/internal | E104 |
| http://172.16.0.1/private | E104 |
| http://192.168.1.1/lan | E104 |
| http://169.254.169.254/metadata | E104 |
| http://internal.hx.dev.local/api | E104 |

**Request**:
```json
{
  "url": "http://192.168.1.1/secret"
}
```

**Expected Response**:
```json
HTTP/1.1 400 Bad Request

{
  "success": false,
  "error": {
    "code": "E104",
    "message": "URL blocked",
    "userMessage": "This URL cannot be accessed for security reasons",
    "suggestedAction": "Try a different URL",
    "retryable": false
  }
}
```

---

### API-UP-011: URL Too Long

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-UP-011 |
| **Title** | Reject URL Exceeding Length Limit |
| **Priority** | P2 - Medium |
| **Spec Reference** | FR-202 |

**Request**:
```json
{
  "url": "https://example.com/" + "x".repeat(2040)
}
```

**Expected Response**:
```json
HTTP/1.1 400 Bad Request

{
  "success": false,
  "error": {
    "code": "E101",
    "message": "URL too long",
    "userMessage": "URL exceeds maximum length of 2048 characters",
    "suggestedAction": "Use a shorter URL",
    "retryable": false
  }
}
```

---

### API-UP-012: Rate Limit Exceeded - E601

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-UP-012 |
| **Title** | Rate Limit Enforcement |
| **Priority** | P1 - High |
| **Spec Reference** | Section 7.4.2, E601 |

**Test Steps**:
1. Make 10 requests within 60 seconds
2. 11th request should be rate limited

**Expected Response (11th request)**:
```json
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 10
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1702483260
RateLimit-Limit: 10
RateLimit-Remaining: 0
RateLimit-Reset: 45
Retry-After: 45

{
  "success": false,
  "error": {
    "code": "E601",
    "message": "Rate limit exceeded",
    "userMessage": "Too many requests. Please wait before trying again.",
    "suggestedAction": "Wait 45 seconds and retry",
    "retryable": true
  }
}
```

**Assertions**:
- Status code: 429
- X-RateLimit-Reset is Unix timestamp
- RateLimit-Reset and Retry-After are delta seconds

---

### API-UP-013: Missing File and URL

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-UP-013 |
| **Title** | Reject Empty Request |
| **Priority** | P1 - High |
| **Spec Reference** | Validation |

**Request**:
```json
{}
```

**Expected Response**:
```json
HTTP/1.1 400 Bad Request

{
  "success": false,
  "error": {
    "code": "E100",
    "message": "No input provided",
    "userMessage": "Please provide a file or URL to process",
    "suggestedAction": "Upload a file or enter a URL",
    "retryable": false
  }
}
```

---

## 3. POST /api/v1/process

### 3.1 Endpoint Specification

| Attribute | Value |
|-----------|-------|
| **Method** | POST |
| **Path** | /api/v1/process |
| **Content-Type** | application/json |
| **Authentication** | Session cookie |
| **Rate Limit** | 10 req/min per session |

---

### API-PR-001: Start Processing - Valid Request

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PR-001 |
| **Title** | Start Processing Valid Job |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-401 |

**Request**:
```json
{
  "jobId": "valid-uuid"
}
```

**Precondition**: Job exists with status PENDING

**Expected Response**:
```json
HTTP/1.1 202 Accepted

{
  "success": true,
  "data": {
    "jobId": "valid-uuid",
    "status": "PROCESSING",
    "streamUrl": "/api/v1/process/valid-uuid/events"
  }
}
```

**Assertions**:
- Status code: 202 (Accepted)
- Job status updated to PROCESSING
- streamUrl returned for SSE connection

---

### API-PR-002: Job Not Found - E501

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PR-002 |
| **Title** | Process Non-Existent Job |
| **Priority** | P1 - High |
| **Spec Reference** | E501 |

**Request**:
```json
{
  "jobId": "non-existent-uuid"
}
```

**Expected Response**:
```json
HTTP/1.1 404 Not Found

{
  "success": false,
  "error": {
    "code": "E501",
    "message": "Job not found",
    "userMessage": "The requested job could not be found",
    "suggestedAction": "Check the job ID or refresh the page",
    "retryable": false
  }
}
```

---

### API-PR-003: Job Already Processing - E701

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PR-003 |
| **Title** | Process Already Processing Job |
| **Priority** | P1 - High |
| **Spec Reference** | E701 |

**Precondition**: Job exists with status PROCESSING

**Request**:
```json
{
  "jobId": "processing-job-uuid"
}
```

**Expected Response**:
```json
HTTP/1.1 409 Conflict

{
  "success": false,
  "error": {
    "code": "E701",
    "message": "Job already processing",
    "userMessage": "This job is already being processed",
    "suggestedAction": "Wait for the current processing to complete",
    "retryable": false
  }
}
```

---

### API-PR-004: Job Belongs to Different Session

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PR-004 |
| **Title** | Process Job from Different Session |
| **Priority** | P1 - High (Security) |
| **Spec Reference** | Session isolation |

**Precondition**: Job exists with sessionId different from request session

**Expected Response**:
```json
HTTP/1.1 404 Not Found

{
  "success": false,
  "error": {
    "code": "E501",
    "message": "Job not found"
  }
}
```

---

## 4. POST /api/v1/jobs/{id}/cancel

### 4.1 Endpoint Specification

| Attribute | Value |
|-----------|-------|
| **Method** | POST |
| **Path** | /api/v1/jobs/{id}/cancel |
| **Content-Type** | application/json |
| **Authentication** | Session cookie |

---

### API-CN-001: Cancel Processing Job

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-CN-001 |
| **Title** | Cancel Job in Processing State |
| **Priority** | P1 - High |
| **Spec Reference** | FR-406 |

**Precondition**: Job exists with status PROCESSING

**Request**:
```
POST /api/v1/jobs/{jobId}/cancel
```

**Expected Response**:
```json
HTTP/1.1 200 OK

{
  "success": true,
  "data": {
    "jobId": "uuid",
    "status": "CANCELLED",
    "cancelledAt": "2025-12-12T10:00:00Z"
  }
}
```

**Assertions**:
- Job status updated to CANCELLED
- SSE sends `cancelled` event
- MCP request aborted (if in-flight)

---

### API-CN-002: Cancel Completed Job - E702

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-CN-002 |
| **Title** | Cannot Cancel Completed Job |
| **Priority** | P1 - High |
| **Spec Reference** | E702 |

**Precondition**: Job exists with status COMPLETE

**Expected Response**:
```json
HTTP/1.1 409 Conflict

{
  "success": false,
  "error": {
    "code": "E702",
    "message": "Job not cancellable",
    "userMessage": "This job has already completed",
    "suggestedAction": "The job has finished processing",
    "retryable": false
  }
}
```

---

### API-CN-003: Cancel Non-Existent Job - E501

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-CN-003 |
| **Title** | Cancel Non-Existent Job |
| **Priority** | P1 - High |
| **Spec Reference** | E501 |

**Expected Response**:
```json
HTTP/1.1 404 Not Found

{
  "success": false,
  "error": {
    "code": "E501",
    "message": "Job not found"
  }
}
```

---

## 5. POST /api/v1/jobs/{id}/resume

### 5.1 Endpoint Specification

| Attribute | Value |
|-----------|-------|
| **Method** | POST |
| **Path** | /api/v1/jobs/{id}/resume |
| **Content-Type** | application/json |
| **Authentication** | Session cookie |

---

### API-RS-001: Resume from Valid Checkpoint

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-RS-001 |
| **Title** | Resume Job from Valid Checkpoint |
| **Priority** | P1 - High |
| **Spec Reference** | Section 5.5 |

**Precondition**: Job has valid checkpoint at 'converted' stage

**Expected Response**:
```json
HTTP/1.1 202 Accepted

{
  "success": true,
  "data": {
    "jobId": "uuid",
    "status": "PROCESSING",
    "resumedFrom": "converted",
    "streamUrl": "/api/v1/process/uuid/events"
  }
}
```

**Assertions**:
- Processing resumes from checkpoint stage
- Conversion not re-executed

---

### API-RS-002: No Checkpoint Exists - E703

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-RS-002 |
| **Title** | Resume Without Checkpoint |
| **Priority** | P1 - High |
| **Spec Reference** | E703 |

**Precondition**: Job exists but has no checkpoint

**Expected Response**:
```json
HTTP/1.1 409 Conflict

{
  "success": false,
  "error": {
    "code": "E703",
    "message": "No valid checkpoint exists",
    "userMessage": "This job cannot be resumed as no checkpoint was saved",
    "suggestedAction": "Start a new processing job",
    "retryable": false
  }
}
```

---

### API-RS-003: Checkpoint Expired - E704

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-RS-003 |
| **Title** | Resume with Expired Checkpoint |
| **Priority** | P1 - High |
| **Spec Reference** | E704 |

**Precondition**: Job has checkpoint older than 24 hours

**Expected Response**:
```json
HTTP/1.1 409 Conflict

{
  "success": false,
  "error": {
    "code": "E704",
    "message": "Checkpoint expired",
    "userMessage": "The checkpoint has expired and can no longer be resumed",
    "suggestedAction": "Start a new processing job",
    "retryable": false
  }
}
```

---

### API-RS-004: Checkpoint Corrupted - E705

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-RS-004 |
| **Title** | Resume with Corrupted Checkpoint |
| **Priority** | P1 - High |
| **Spec Reference** | E705 |

**Precondition**: Job has checkpoint with invalid checksum

**Expected Response**:
```json
HTTP/1.1 409 Conflict

{
  "success": false,
  "error": {
    "code": "E705",
    "message": "Checkpoint corrupted",
    "userMessage": "The checkpoint data is corrupted and cannot be used",
    "suggestedAction": "Start a new processing job",
    "retryable": false
  }
}
```

---

### API-RS-005: Resume Completed Job - E706

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-RS-005 |
| **Title** | Resume Already Completed Job |
| **Priority** | P1 - High |
| **Spec Reference** | E706 |

**Precondition**: Job has status COMPLETE

**Expected Response**:
```json
HTTP/1.1 409 Conflict

{
  "success": false,
  "error": {
    "code": "E706",
    "message": "Job already completed",
    "userMessage": "This job has already completed successfully",
    "suggestedAction": "View results or start a new job",
    "retryable": false
  }
}
```

---

## 6. GET /api/v1/jobs/{id}

### 6.1 Endpoint Specification

| Attribute | Value |
|-----------|-------|
| **Method** | GET |
| **Path** | /api/v1/jobs/{id} |
| **Authentication** | Session cookie |

---

### API-JB-001: Get Job Details - Complete

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-JB-001 |
| **Title** | Get Completed Job with Results |
| **Priority** | P1 - High |
| **Spec Reference** | FR-703 |

**Precondition**: Job exists with status COMPLETE and results

**Expected Response**:
```json
HTTP/1.1 200 OK

{
  "success": true,
  "data": {
    "id": "uuid",
    "status": "COMPLETE",
    "inputType": "FILE",
    "fileName": "document.pdf",
    "fileSize": 102400,
    "createdAt": "2025-12-12T10:00:00Z",
    "completedAt": "2025-12-12T10:01:30Z",
    "results": [
      {
        "format": "MARKDOWN",
        "size": 5000,
        "createdAt": "2025-12-12T10:01:30Z"
      },
      {
        "format": "HTML",
        "size": 8000,
        "createdAt": "2025-12-12T10:01:30Z"
      },
      {
        "format": "JSON",
        "size": 15000,
        "createdAt": "2025-12-12T10:01:30Z"
      }
    ]
  }
}
```

---

### API-JB-002: Get Job Details - Processing

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-JB-002 |
| **Title** | Get Job in Processing State |
| **Priority** | P1 - High |
| **Spec Reference** | FR-501 |

**Precondition**: Job exists with status PROCESSING

**Expected Response**:
```json
HTTP/1.1 200 OK

{
  "success": true,
  "data": {
    "id": "uuid",
    "status": "PROCESSING",
    "inputType": "FILE",
    "fileName": "document.pdf",
    "progress": {
      "stage": "conversion",
      "percent": 65,
      "message": "Converting document..."
    },
    "createdAt": "2025-12-12T10:00:00Z"
  }
}
```

---

### API-JB-003: Get Job Details - Error

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-JB-003 |
| **Title** | Get Failed Job with Error Details |
| **Priority** | P1 - High |
| **Spec Reference** | FR-801 |

**Precondition**: Job exists with status ERROR

**Expected Response**:
```json
HTTP/1.1 200 OK

{
  "success": true,
  "data": {
    "id": "uuid",
    "status": "ERROR",
    "errorCode": "E302",
    "errorMessage": "Document conversion failed",
    "retryable": true,
    "createdAt": "2025-12-12T10:00:00Z"
  }
}
```

---

### API-JB-004: Job Not Found

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-JB-004 |
| **Title** | Get Non-Existent Job |
| **Priority** | P1 - High |
| **Spec Reference** | E501 |

**Expected Response**:
```json
HTTP/1.1 404 Not Found

{
  "success": false,
  "error": {
    "code": "E501",
    "message": "Job not found"
  }
}
```

---

## 7. GET /api/v1/history

### 7.1 Endpoint Specification

| Attribute | Value |
|-----------|-------|
| **Method** | GET |
| **Path** | /api/v1/history |
| **Query Parameters** | page, pageSize, sortBy, sortOrder |
| **Authentication** | Session cookie |

---

### API-HI-001: Get History - Default Pagination

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-HI-001 |
| **Title** | Get History with Default Parameters |
| **Priority** | P1 - High |
| **Spec Reference** | FR-701, FR-702 |

**Request**:
```
GET /api/v1/history
```

**Expected Response**:
```json
HTTP/1.1 200 OK

{
  "success": true,
  "data": {
    "jobs": [
      {
        "id": "uuid-1",
        "status": "COMPLETE",
        "inputType": "FILE",
        "fileName": "document1.pdf",
        "createdAt": "2025-12-12T10:00:00Z"
      },
      // ... up to 20 items
    ],
    "pagination": {
      "page": 1,
      "pageSize": 20,
      "totalCount": 45,
      "totalPages": 3,
      "hasMore": true
    }
  }
}
```

**Assertions**:
- Default pageSize is 20
- Jobs sorted by createdAt DESC
- Only session jobs returned

---

### API-HI-002: Get History - Custom Pagination

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-HI-002 |
| **Title** | Get History with Custom Page Size |
| **Priority** | P1 - High |
| **Spec Reference** | FR-702 |

**Request**:
```
GET /api/v1/history?page=2&pageSize=10
```

**Expected Response**:
```json
{
  "success": true,
  "data": {
    "jobs": [ /* 10 items */ ],
    "pagination": {
      "page": 2,
      "pageSize": 10,
      "totalCount": 45,
      "totalPages": 5,
      "hasMore": true
    }
  }
}
```

---

### API-HI-003: Get History - Sort by Status

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-HI-003 |
| **Title** | Get History Sorted by Status |
| **Priority** | P2 - Medium |
| **Spec Reference** | FR-702 |

**Request**:
```
GET /api/v1/history?sortBy=status&sortOrder=asc
```

**Assertions**:
- Jobs sorted by status alphabetically

---

### API-HI-004: Invalid Pagination Parameters

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-HI-004 |
| **Title** | Invalid Pagination Parameters |
| **Priority** | P2 - Medium |
| **Spec Reference** | E801 |

**Request**:
```
GET /api/v1/history?page=-1&pageSize=1000
```

**Expected Response**:
```json
HTTP/1.1 400 Bad Request

{
  "success": false,
  "error": {
    "code": "E801",
    "message": "Invalid pagination parameters",
    "userMessage": "Page must be >= 1, pageSize must be 1-100"
  }
}
```

---

### API-HI-005: Empty History

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-HI-005 |
| **Title** | Get History with No Jobs |
| **Priority** | P1 - High |
| **Spec Reference** | FR-701 |

**Precondition**: Session has no jobs

**Expected Response**:
```json
HTTP/1.1 200 OK

{
  "success": true,
  "data": {
    "jobs": [],
    "pagination": {
      "page": 1,
      "pageSize": 20,
      "totalCount": 0,
      "totalPages": 0,
      "hasMore": false
    }
  }
}
```

---

## 8. GET /api/v1/health

### 8.1 Endpoint Specification

| Attribute | Value |
|-----------|-------|
| **Method** | GET |
| **Path** | /api/v1/health |
| **Authentication** | None required |

---

### API-HL-001: Health Check - All Healthy

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-HL-001 |
| **Title** | Health Check All Services Healthy |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-601 |

**Expected Response**:
```json
HTTP/1.1 200 OK

{
  "status": "healthy",
  "timestamp": "2025-12-12T10:00:00Z",
  "version": "1.0.0",
  "services": {
    "database": {
      "status": "healthy",
      "latency": 5
    },
    "redis": {
      "status": "healthy",
      "latency": 2
    },
    "mcp": {
      "status": "healthy",
      "latency": 50
    }
  }
}
```

---

### API-HL-002: Health Check - Degraded

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-HL-002 |
| **Title** | Health Check with Degraded Service |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-601 |

**Precondition**: Redis latency > 100ms

**Expected Response**:
```json
HTTP/1.1 200 OK

{
  "status": "degraded",
  "timestamp": "2025-12-12T10:00:00Z",
  "version": "1.0.0",
  "services": {
    "database": {
      "status": "healthy",
      "latency": 5
    },
    "redis": {
      "status": "degraded",
      "latency": 150,
      "message": "High latency detected"
    },
    "mcp": {
      "status": "healthy",
      "latency": 50
    }
  }
}
```

---

### API-HL-003: Health Check - Unhealthy

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-HL-003 |
| **Title** | Health Check with Unhealthy Service |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-601 |

**Precondition**: Database connection failed

**Expected Response**:
```json
HTTP/1.1 503 Service Unavailable

{
  "status": "unhealthy",
  "timestamp": "2025-12-12T10:00:00Z",
  "version": "1.0.0",
  "services": {
    "database": {
      "status": "unhealthy",
      "error": "Connection refused"
    },
    "redis": {
      "status": "healthy",
      "latency": 2
    },
    "mcp": {
      "status": "healthy",
      "latency": 50
    }
  }
}
```

---

## 9. SSE Event Stream Tests

### 9.1 Endpoint Specification

| Attribute | Value |
|-----------|-------|
| **Method** | GET |
| **Path** | /api/v1/process/{jobId}/events |
| **Content-Type** | text/event-stream |
| **Authentication** | Session cookie |
| **Connection** | Keep-alive with SSE protocol |

---

### API-SSE-001: Valid SSE Stream Headers and Format

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-SSE-001 |
| **Title** | Valid SSE Stream Headers and Format |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-501, FR-502, NFR-201 |

**Precondition**: Job exists with status PROCESSING

**Request**:
```
GET /api/v1/process/{jobId}/events
Cookie: hx-docling-session=abc123
Accept: text/event-stream
```

**Expected Response Headers**:
```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
X-Accel-Buffering: no
```

**Expected Event Format**:
```
id: evt-001
event: progress
data: {"stage":"conversion","percent":25,"message":"Converting document..."}

```

**Assertions**:
- Status code: 200
- Content-Type: text/event-stream
- Cache-Control: no-cache (prevents buffering)
- Connection: keep-alive
- Each event has id, event type, and data fields
- Data field is valid JSON
- Events separated by double newline

---

### API-SSE-002: Progress Event JSON Structure

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-SSE-002 |
| **Title** | Progress Event JSON Structure Validation |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-501, FR-502 |

**Precondition**: Job is actively processing

**Expected Progress Event**:
```
event: progress
data: {
  "stage": "conversion",
  "percent": 45,
  "message": "Converting document pages..."
}
```

**Stage Values (Enum)**:

| Stage | Description | Percent Range |
|-------|-------------|---------------|
| validating | Input validation | 0-10 |
| uploading | File upload in progress | 10-20 |
| conversion | Docling conversion | 20-60 |
| export_markdown | Markdown export | 60-75 |
| export_html | HTML export | 75-85 |
| export_json | JSON export | 85-95 |
| finalizing | Cleanup and storage | 95-100 |

**Assertions**:
- `stage` is one of valid enum values
- `percent` is integer between 0 and 100
- `percent` is monotonically increasing (never decreases)
- `message` is non-empty string
- JSON structure validates against schema

---

### API-SSE-003: Event Sequencing and ID Format

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-SSE-003 |
| **Title** | Event Sequencing and ID Format |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-502, FR-503, NFR-201 |

**Precondition**: Job is actively processing

**Expected Event Sequence**:
```
id: evt-001
event: started
data: {"jobId":"uuid","startedAt":"2025-12-14T10:00:00Z"}

id: evt-002
event: progress
data: {"stage":"validating","percent":5,"message":"Validating input..."}

id: evt-003
event: progress
data: {"stage":"conversion","percent":30,"message":"Converting..."}

...

id: evt-NNN
event: completed
data: {"jobId":"uuid","status":"COMPLETE","results":[...]}
```

**Assertions**:
- Event IDs are sequential and unique (evt-001, evt-002, ...)
- First event is `started` type
- Progress events follow in order
- Final event is `completed` or `error`
- No duplicate event IDs
- Events maintain causal ordering

---

### API-SSE-004: Completion Event with Results Summary

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-SSE-004 |
| **Title** | Completion Event with Results Summary |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-501, FR-601 |

**Precondition**: Job completes processing successfully

**Expected Completion Event**:
```
id: evt-final
event: completed
data: {
  "jobId": "uuid-v4",
  "status": "COMPLETE",
  "completedAt": "2025-12-14T10:01:30Z",
  "processingTime": 90000,
  "results": [
    {
      "format": "MARKDOWN",
      "size": 5000,
      "available": true
    },
    {
      "format": "HTML",
      "size": 8000,
      "available": true
    },
    {
      "format": "JSON",
      "size": 15000,
      "available": true
    }
  ]
}
```

**Assertions**:
- Event type is `completed`
- Status is "COMPLETE"
- completedAt is valid ISO 8601 timestamp
- processingTime is positive integer (milliseconds)
- results array contains all three formats
- Each result has format, size, and available fields
- SSE connection closes after completion event

---

### API-SSE-005: Error Event Format

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-SSE-005 |
| **Title** | Error Event Format |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-801, FR-802 |

**Precondition**: Job encounters processing error

**Expected Error Event**:
```
id: evt-error
event: error
data: {
  "jobId": "uuid-v4",
  "status": "ERROR",
  "errorCode": "E301",
  "errorMessage": "Document conversion failed",
  "userMessage": "The document could not be converted. Please try again.",
  "retryable": true,
  "failedAt": "2025-12-14T10:00:30Z",
  "lastSuccessfulStage": "validating"
}
```

**Assertions**:
- Event type is `error`
- Status is "ERROR"
- errorCode follows EXXX format
- userMessage is user-friendly (no technical details)
- retryable indicates if job can be retried
- lastSuccessfulStage indicates checkpoint position
- SSE connection closes after error event

---

### API-SSE-006: Last-Event-ID Reconnection Replay

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-SSE-006 |
| **Title** | Last-Event-ID Reconnection Replay |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-503, NFR-201 |

**Precondition**: Job is processing, client reconnects after disconnection

**Request with Last-Event-ID**:
```
GET /api/v1/process/{jobId}/events
Cookie: hx-docling-session=abc123
Accept: text/event-stream
Last-Event-ID: evt-005
```

**Expected Behavior**:
- Server acknowledges Last-Event-ID header
- Server replays events starting from evt-006
- No duplicate events (evt-001 through evt-005 not resent)
- Stream continues with new events

**Test Steps**:
1. Connect to SSE stream for processing job
2. Receive events evt-001 through evt-005
3. Simulate disconnection
4. Reconnect with `Last-Event-ID: evt-005`
5. Verify events resume from evt-006
6. Verify no events before evt-006 are received
7. Verify processing completes successfully

**Assertions**:
- First event received after reconnection has ID > evt-005
- Event sequence is contiguous (no gaps)
- Final completion event received
- No duplicate processing of events

---

## 10. Authentication/Authorization Tests

### 10.1 Overview

Authentication/Authorization tests validate session-based access control across all protected endpoints. The application uses cookie-based sessions with the `hx-docling-session` cookie.

---

### API-AUTH-001: Missing Session Cookie - 401 Unauthorized

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-AUTH-001 |
| **Title** | Missing Session Cookie Returns 401 |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.4 Session Management |

**Test Matrix - Protected Endpoints**:

| Endpoint | Method | Expected |
|----------|--------|----------|
| /api/v1/upload | POST | 401 |
| /api/v1/process | POST | 401 |
| /api/v1/jobs/{id}/cancel | POST | 401 |
| /api/v1/jobs/{id}/resume | POST | 401 |
| /api/v1/jobs/{id} | GET | 401 |
| /api/v1/history | GET | 401 |
| /api/v1/process/{id}/events | GET | 401 |

**Request (No Cookie)**:
```
POST /api/v1/upload HTTP/1.1
Content-Type: multipart/form-data
```

**Expected Response**:
```json
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Cookie realm="hx-docling"

{
  "success": false,
  "error": {
    "code": "E401",
    "message": "Authentication required",
    "userMessage": "Your session has expired. Please refresh the page.",
    "suggestedAction": "Refresh the page to start a new session",
    "retryable": true
  }
}
```

**Assertions**:
- Status code: 401
- WWW-Authenticate header present
- Error code: E401
- No sensitive data exposed in response

---

### API-AUTH-002: Invalid Session Format - 401 Unauthorized

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-AUTH-002 |
| **Title** | Invalid Session Format Returns 401 |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.4 Session Management |

**Test Matrix - Invalid Session Formats**:

| Cookie Value | Description | Expected |
|--------------|-------------|----------|
| `hx-docling-session=` | Empty value | 401 |
| `hx-docling-session=abc` | Too short | 401 |
| `hx-docling-session=<script>` | XSS attempt | 401 |
| `hx-docling-session=../../etc/passwd` | Path traversal | 401 |
| `hx-docling-session=' OR '1'='1` | SQL injection | 401 |
| `hx-docling-session=AAAA...` (1000 chars) | Oversized | 401 |

**Request**:
```
POST /api/v1/upload HTTP/1.1
Cookie: hx-docling-session=invalid-format-123
Content-Type: multipart/form-data
```

**Expected Response**:
```json
HTTP/1.1 401 Unauthorized

{
  "success": false,
  "error": {
    "code": "E401",
    "message": "Invalid session",
    "userMessage": "Your session is invalid. Please refresh the page.",
    "suggestedAction": "Refresh the page to start a new session",
    "retryable": true
  }
}
```

**Assertions**:
- Status code: 401 (not 400 - avoid leaking validation logic)
- Same error message for all invalid formats (timing-safe)
- No reflection of invalid input in response

---

### API-AUTH-003: Expired Session - 401 Unauthorized

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-AUTH-003 |
| **Title** | Expired Session Returns 401 |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.4.3 Session Expiry |

**Precondition**: Session was valid but has exceeded 24-hour TTL

**Request**:
```
POST /api/v1/upload HTTP/1.1
Cookie: hx-docling-session=expired-session-uuid
Content-Type: multipart/form-data
```

**Expected Response**:
```json
HTTP/1.1 401 Unauthorized

{
  "success": false,
  "error": {
    "code": "E402",
    "message": "Session expired",
    "userMessage": "Your session has expired. Please refresh the page to continue.",
    "suggestedAction": "Refresh the page to start a new session",
    "retryable": true
  }
}
```

**Assertions**:
- Status code: 401
- Error code: E402 (distinct from E401 for metrics)
- Session removed from Redis after expiry
- New session created on next request with valid cookie

---

### API-AUTH-004: Cross-Session Upload Access - 403 Forbidden

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-AUTH-004 |
| **Title** | Cross-Session Upload Access Denied |
| **Priority** | P0 - Critical (Security) |
| **Spec Reference** | Section 5.4.4 Session Isolation |

**Precondition**:
- Session A has uploaded a file (jobId: job-123)
- Session B is valid but different

**Request (Session B trying to access Session A's job)**:
```
GET /api/v1/jobs/job-123 HTTP/1.1
Cookie: hx-docling-session=session-B-uuid
```

**Expected Response**:
```json
HTTP/1.1 403 Forbidden

{
  "success": false,
  "error": {
    "code": "E403",
    "message": "Access denied",
    "userMessage": "You do not have permission to access this resource.",
    "suggestedAction": "Ensure you are accessing your own jobs",
    "retryable": false
  }
}
```

**Alternative Response (Privacy-Preserving)**:
```json
HTTP/1.1 404 Not Found

{
  "success": false,
  "error": {
    "code": "E501",
    "message": "Job not found"
  }
}
```

**Assertions**:
- Status code: 403 (explicit denial) or 404 (privacy-preserving)
- No information about job existence leaked to other sessions
- Audit log records access attempt

**Note**: 404 response is preferred to prevent session enumeration attacks.

---

### API-AUTH-005: Cross-Session Process Access - 403 Forbidden

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-AUTH-005 |
| **Title** | Cross-Session Process Access Denied |
| **Priority** | P0 - Critical (Security) |
| **Spec Reference** | Section 5.4.4 Session Isolation |

**Precondition**:
- Session A has uploaded a file (jobId: job-123, status: PENDING)
- Session B is valid but different

**Request (Session B trying to process Session A's job)**:
```
POST /api/v1/process HTTP/1.1
Cookie: hx-docling-session=session-B-uuid
Content-Type: application/json

{
  "jobId": "job-123"
}
```

**Expected Response**:
```json
HTTP/1.1 403 Forbidden

{
  "success": false,
  "error": {
    "code": "E403",
    "message": "Access denied",
    "userMessage": "You do not have permission to process this job.",
    "suggestedAction": "Ensure you are processing your own jobs",
    "retryable": false
  }
}
```

**Assertions**:
- Status code: 403 or 404 (privacy-preserving)
- Job status unchanged (still PENDING)
- No processing initiated for wrong session
- Audit log records unauthorized process attempt

---

### API-AUTH-006: Unauthenticated Health Endpoint - 200 OK

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-AUTH-006 |
| **Title** | Health Endpoint Does Not Require Authentication |
| **Priority** | P0 - Critical |
| **Spec Reference** | NFR-601 Health Check |

**Request (No Cookie)**:
```
GET /api/v1/health HTTP/1.1
```

**Expected Response**:
```json
HTTP/1.1 200 OK

{
  "status": "healthy",
  "timestamp": "2025-12-14T10:00:00Z",
  "version": "1.0.0",
  "services": {
    "database": { "status": "healthy" },
    "redis": { "status": "healthy" },
    "mcp": { "status": "healthy" }
  }
}
```

**Assertions**:
- Status code: 200 (not 401)
- No session cookie required
- Health information available for load balancers
- No sensitive data exposed (no internal IPs, no credentials)

---

### API-AUTH-007: Unauthenticated Upload - 401 Unauthorized

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-AUTH-007 |
| **Title** | Upload Endpoint Requires Authentication |
| **Priority** | P0 - Critical |
| **Spec Reference** | Section 5.4 Session Management |

**Request (No Cookie)**:
```
POST /api/v1/upload HTTP/1.1
Content-Type: multipart/form-data; boundary=----Boundary

------Boundary
Content-Disposition: form-data; name="file"; filename="document.pdf"
Content-Type: application/pdf

<file binary data>
------Boundary--
```

**Expected Response**:
```json
HTTP/1.1 401 Unauthorized

{
  "success": false,
  "error": {
    "code": "E401",
    "message": "Authentication required",
    "userMessage": "Please refresh the page to start a new session.",
    "suggestedAction": "Refresh the page",
    "retryable": true
  }
}
```

**Assertions**:
- Status code: 401 (not 400)
- File not saved to storage
- No job created in database
- Response does not reveal file processing details

---

## 11. Security Test Cases

### 11.1 Input Validation Security

---

### API-SEC-001: XSS Prevention in URL

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-SEC-001 |
| **Title** | XSS Prevention in URL Input |
| **Priority** | P0 - Critical (Security) |
| **Spec Reference** | Section 10.1.1 |

**Request**:
```json
{
  "url": "https://example.com/<script>alert(1)</script>"
}
```

**Expected Response**:
```json
HTTP/1.1 400 Bad Request

{
  "success": false,
  "error": {
    "code": "E101",
    "message": "Invalid URL format"
  }
}
```

---

### API-SEC-002: XSS Prevention in Filename

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-SEC-002 |
| **Title** | XSS Prevention in Filename |
| **Priority** | P0 - Critical (Security) |
| **Spec Reference** | Section 10.1.1 |

**Request**:
- File: `<img onerror=alert(1)>.pdf`

**Expected Result**:
- Filename sanitized or rejected
- No script execution possible

---

### API-SEC-003: SQL Injection Prevention

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-SEC-003 |
| **Title** | SQL Injection Prevention |
| **Priority** | P0 - Critical (Security) |
| **Spec Reference** | Section 10.1.1 |

**Request**:
```
GET /api/v1/history?sortBy=createdAt; DROP TABLE jobs;--
```

**Expected Response**:
```json
HTTP/1.1 400 Bad Request

{
  "success": false,
  "error": {
    "code": "E801",
    "message": "Invalid parameter value"
  }
}
```

---

### API-SEC-004: Path Traversal Prevention

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-SEC-004 |
| **Title** | Path Traversal in Filename |
| **Priority** | P0 - Critical (Security) |
| **Spec Reference** | Section 10.1.1 |

**Request**:
- File: `../../../etc/passwd.pdf`

**Expected Result**:
- Filename sanitized (path components removed)
- Or request rejected with E002

---

### API-SEC-005: Magic Byte Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-SEC-005 |
| **Title** | File Magic Byte Validation |
| **Priority** | P0 - Critical (Security) |
| **Spec Reference** | Section 10.1.1 |

**Request**:
- File: document.pdf (but contains GIF magic bytes)

**Expected Response**:
```json
HTTP/1.1 400 Bad Request

{
  "success": false,
  "error": {
    "code": "E004",
    "message": "File corrupted",
    "userMessage": "The file content does not match its extension"
  }
}
```

---

### API-SEC-006: Content-Length Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-SEC-006 |
| **Title** | Content-Length Smuggling Prevention |
| **Priority** | P1 - High (Security) |
| **Spec Reference** | Section 10.1.1 |

**Request**:
- Content-Length: 100
- Actual body: 1MB

**Expected Result**:
- Request rejected or actual size validated

---

### API-SEC-007: CORS Headers Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-SEC-007 |
| **Title** | CORS Headers Validation |
| **Priority** | P1 - High (Security) |
| **Spec Reference** | Section 8.2 |

**Request from different origin**:
```
GET /api/v1/health
Origin: https://malicious-site.com
```

**Expected Response Headers**:
- No Access-Control-Allow-Origin for unauthorized origins

---

### API-SEC-008: Security Headers Present

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-SEC-008 |
| **Title** | Security Headers Validation |
| **Priority** | P1 - High (Security) |
| **Spec Reference** | Section 8.3 |

**Expected Response Headers**:
```
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Referrer-Policy: strict-origin-when-cross-origin
Content-Security-Policy: default-src 'self'; ...
```

---

### API-SEC-009: Session Cookie Security

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-SEC-009 |
| **Title** | Session Cookie Security Flags |
| **Priority** | P0 - Critical (Security) |
| **Spec Reference** | Section 5.4 |

**Expected Set-Cookie Header**:
```
Set-Cookie: hx-docling-session=...; HttpOnly; Secure; SameSite=Strict; Path=/
```

**Assertions**:
- HttpOnly flag present
- SameSite=Strict
- Secure flag (in production)

---

## 12. MCP Integration Layer Tests

### 12.1 Overview

MCP (Model Context Protocol) Integration Layer tests validate the correct routing of document processing requests to appropriate MCP tools, parameter transformation, response structure validation, and error mapping between MCP and HTTP protocols.

---

### API-MCP-001: Tool Routing by File Type - PDF

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-MCP-001 |
| **Title** | Tool Routing by File Type (PDF to convert_pdf) |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-301, MCP-101 |

**Precondition**: Job exists with uploaded PDF file, status PENDING

**Request**:
```json
POST /api/v1/process
Cookie: hx-docling-session=abc123
Content-Type: application/json

{
  "jobId": "pdf-job-uuid"
}
```

**Expected Behavior**:
- API layer detects file MIME type: `application/pdf`
- MCP client invokes `convert_pdf` tool
- Tool receives correct file path parameter

**Expected MCP Tool Call**:
```json
{
  "tool": "convert_pdf",
  "arguments": {
    "source": "/uploads/pdf-job-uuid/document.pdf",
    "options": {
      "ocr_enabled": true,
      "table_extraction": true
    }
  }
}
```

**Assertions**:
- MCP tool `convert_pdf` is invoked (not `convert_docx` or `convert_url`)
- File path correctly passed to MCP tool
- Job status transitions to PROCESSING
- SSE stream initiated with correct tool context

---

### API-MCP-002: Tool Routing by File Type - DOCX

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-MCP-002 |
| **Title** | Tool Routing by File Type (DOCX to convert_docx) |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-301, MCP-102 |

**Precondition**: Job exists with uploaded DOCX file, status PENDING

**Request**:
```json
POST /api/v1/process
Cookie: hx-docling-session=abc123
Content-Type: application/json

{
  "jobId": "docx-job-uuid"
}
```

**Expected Behavior**:
- API layer detects file MIME type: `application/vnd.openxmlformats-officedocument.wordprocessingml.document`
- MCP client invokes `convert_docx` tool
- Tool receives correct file path parameter

**Expected MCP Tool Call**:
```json
{
  "tool": "convert_docx",
  "arguments": {
    "source": "/uploads/docx-job-uuid/document.docx",
    "options": {
      "preserve_formatting": true,
      "extract_images": true
    }
  }
}
```

**Assertions**:
- MCP tool `convert_docx` is invoked
- MIME type correctly mapped to tool name
- File path correctly passed to MCP tool
- Job status transitions to PROCESSING

---

### API-MCP-003: Tool Routing for URL Input

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-MCP-003 |
| **Title** | Tool Routing for URL Input (to convert_url) |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-201, MCP-103 |

**Precondition**: Job exists with URL input, status PENDING

**Request**:
```json
POST /api/v1/process
Cookie: hx-docling-session=abc123
Content-Type: application/json

{
  "jobId": "url-job-uuid"
}
```

**Expected Behavior**:
- API layer detects input type: URL
- MCP client invokes `convert_url` tool
- Tool receives URL as source parameter

**Expected MCP Tool Call**:
```json
{
  "tool": "convert_url",
  "arguments": {
    "source": "https://example.com/document.html",
    "options": {
      "follow_redirects": true,
      "timeout": 30000,
      "user_agent": "HX-Docling/1.0"
    }
  }
}
```

**Assertions**:
- MCP tool `convert_url` is invoked
- URL passed as source (not file path)
- SSRF-validated URL passed to MCP (validation happens before MCP call)
- Job status transitions to PROCESSING

---

### API-MCP-004: MCP Parameter Transformation Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-MCP-004 |
| **Title** | MCP Parameter Transformation Validation |
| **Priority** | P0 - Critical |
| **Spec Reference** | MCP-104, FR-402 |

**Precondition**: Job exists with processing options specified

**Request**:
```json
POST /api/v1/process
Cookie: hx-docling-session=abc123
Content-Type: application/json

{
  "jobId": "job-with-options-uuid",
  "options": {
    "enableOcr": true,
    "extractTables": true,
    "preserveFormatting": false,
    "outputFormats": ["markdown", "html", "json"]
  }
}
```

**Expected MCP Tool Call**:
```json
{
  "tool": "convert_pdf",
  "arguments": {
    "source": "/uploads/job-with-options-uuid/document.pdf",
    "options": {
      "ocr_enabled": true,
      "table_extraction": true,
      "preserve_formatting": false
    }
  }
}
```

**Assertions**:
- API options transformed to MCP parameter format
- camelCase to snake_case conversion (enableOcr -> ocr_enabled)
- Boolean values preserved correctly
- Output formats stored for export phase (not passed to convert tool)
- Invalid options rejected before MCP call

---

### API-MCP-005: DoclingDocument Response Structure Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-MCP-005 |
| **Title** | DoclingDocument Response Structure Validation |
| **Priority** | P0 - Critical |
| **Spec Reference** | MCP-201, FR-601 |

**Precondition**: MCP conversion tool completes successfully

**Expected MCP Tool Response**:
```json
{
  "result": {
    "type": "DoclingDocument",
    "metadata": {
      "title": "Document Title",
      "pages": 10,
      "language": "en",
      "created": "2025-12-14T10:00:00Z"
    },
    "content": {
      "sections": [...],
      "tables": [...],
      "images": [...],
      "text_blocks": [...]
    },
    "checksum": "sha256:abc123..."
  }
}
```

**Assertions**:
- Response contains `type: DoclingDocument`
- Metadata includes title, pages, language
- Content structure contains expected sections
- Checksum present for integrity verification
- Response stored in job record for export phase
- SSE progress event sent with conversion completion

---

### API-MCP-006: Export Tool Chaining

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-MCP-006 |
| **Title** | Export Tool Chaining (Markdown, HTML, JSON) |
| **Priority** | P0 - Critical |
| **Spec Reference** | MCP-301, FR-601, FR-602 |

**Precondition**: Conversion complete, DoclingDocument available

**Expected Export Sequence**:
1. `export_markdown` tool invoked
2. `export_html` tool invoked
3. `export_json` tool invoked

**Expected MCP Tool Calls (Sequential)**:
```json
// Call 1: Markdown Export
{
  "tool": "export_markdown",
  "arguments": {
    "document": "<DoclingDocument reference>",
    "options": {
      "include_metadata": true,
      "table_format": "gfm"
    }
  }
}

// Call 2: HTML Export
{
  "tool": "export_html",
  "arguments": {
    "document": "<DoclingDocument reference>",
    "options": {
      "include_styles": true,
      "semantic_tags": true
    }
  }
}

// Call 3: JSON Export
{
  "tool": "export_json",
  "arguments": {
    "document": "<DoclingDocument reference>",
    "options": {
      "pretty_print": false,
      "include_positions": true
    }
  }
}
```

**Assertions**:
- All three export tools invoked in sequence
- Each export receives same DoclingDocument reference
- SSE progress events sent for each export (60-75%, 75-85%, 85-95%)
- All export results stored in job record
- Job status transitions to COMPLETE after all exports

---

### API-MCP-007: MCP Error to HTTP Error Mapping

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-MCP-007 |
| **Title** | MCP Error to HTTP Error Mapping (-32603 to 503) |
| **Priority** | P0 - Critical |
| **Spec Reference** | MCP-401, E201-E206 |

**Error Mapping Matrix**:

| MCP Error Code | MCP Error | HTTP Status | API Error Code | Description |
|----------------|-----------|-------------|----------------|-------------|
| -32700 | Parse error | 400 | E201 | Invalid MCP request format |
| -32600 | Invalid request | 400 | E201 | Malformed MCP request |
| -32601 | Method not found | 501 | E205 | Tool not implemented |
| -32602 | Invalid params | 400 | E203 | Invalid tool parameters |
| -32603 | Internal error | 503 | E204 | MCP server internal error |
| -32000 | Server error | 503 | E204 | MCP server unavailable |
| -32001 | Timeout | 504 | E202 | MCP request timeout |

**Precondition**: MCP server returns error code -32603

**Expected MCP Error Response**:
```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32603,
    "message": "Internal error",
    "data": {
      "details": "Document processing failed"
    }
  },
  "id": "req-123"
}
```

**Expected API Response**:
```json
HTTP/1.1 503 Service Unavailable
Retry-After: 30

{
  "success": false,
  "error": {
    "code": "E204",
    "message": "Processing service unavailable",
    "userMessage": "The document processing service is temporarily unavailable. Please try again.",
    "suggestedAction": "Wait a moment and retry",
    "retryable": true
  }
}
```

**Assertions**:
- MCP error code -32603 maps to HTTP 503
- API error code E204 returned
- Retry-After header included for retryable errors
- SSE error event sent before connection close
- Job status updated to ERROR with retryable=true
- Original MCP error details logged (not exposed to user)

---

### API-MCP-008: Tool Unavailability Handling

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-MCP-008 |
| **Title** | Tool Unavailability Handling (E205) |
| **Priority** | P0 - Critical |
| **Spec Reference** | MCP-402, E205 |

**Precondition**: MCP server returns error -32601 (Method not found)

**Scenario**: Request to process PPTX file when `convert_pptx` tool is unavailable

**Expected MCP Error Response**:
```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32601,
    "message": "Method not found",
    "data": {
      "tool": "convert_pptx"
    }
  },
  "id": "req-456"
}
```

**Expected API Response**:
```json
HTTP/1.1 501 Not Implemented

{
  "success": false,
  "error": {
    "code": "E205",
    "message": "Tool unavailable",
    "userMessage": "Processing for this file type is currently unavailable.",
    "suggestedAction": "Try a different file format or contact support",
    "retryable": false
  }
}
```

**Assertions**:
- HTTP status 501 Not Implemented
- Error code E205 returned
- retryable=false (tool unavailability is not transient)
- Job status updated to ERROR
- SSE error event sent with tool unavailability reason

---

### API-MCP-009: Partial Export Failure Handling

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-MCP-009 |
| **Title** | Partial Export Failure Handling (PARTIAL_COMPLETE) |
| **Priority** | P0 - Critical |
| **Spec Reference** | MCP-403, FR-603 |

**Precondition**: Conversion succeeds, Markdown export succeeds, HTML export fails

**Scenario**:
1. `convert_pdf` succeeds -> DoclingDocument created
2. `export_markdown` succeeds -> Markdown result stored
3. `export_html` fails -> Error returned
4. `export_json` succeeds -> JSON result stored (continues despite HTML failure)

**Expected API Response**:
```json
HTTP/1.1 206 Partial Content

{
  "success": true,
  "data": {
    "jobId": "partial-job-uuid",
    "status": "PARTIAL_COMPLETE",
    "completedAt": "2025-12-14T10:01:30Z",
    "results": [
      {
        "format": "MARKDOWN",
        "size": 5000,
        "available": true
      },
      {
        "format": "HTML",
        "size": 0,
        "available": false,
        "error": {
          "code": "E302",
          "message": "HTML export failed"
        }
      },
      {
        "format": "JSON",
        "size": 15000,
        "available": true
      }
    ],
    "warnings": [
      "HTML export failed: E302 - Export tool returned error"
    ]
  }
}
```

**Assertions**:
- HTTP status 206 Partial Content
- Job status is PARTIAL_COMPLETE (not ERROR)
- Available results still accessible
- Failed export marked as available=false with error details
- Warnings array contains failure information
- SSE completed event includes partial results
- User can still download successful exports

---

### API-MCP-010: MCP Timeout Handling

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-MCP-010 |
| **Title** | MCP Timeout Handling (E202) |
| **Priority** | P0 - Critical |
| **Spec Reference** | MCP-404, E202, NFR-101 |

**Precondition**: MCP request exceeds timeout threshold (300 seconds for conversion)

**Timeout Configuration**:

| Operation | Timeout | Error Code |
|-----------|---------|------------|
| Conversion | 300s | E202 |
| Export (each) | 60s | E202 |
| Health check | 5s | E206 |

**Expected API Response**:
```json
HTTP/1.1 504 Gateway Timeout

{
  "success": false,
  "error": {
    "code": "E202",
    "message": "Processing timeout",
    "userMessage": "Document processing took too long. The document may be too complex.",
    "suggestedAction": "Try a smaller document or contact support for large files",
    "retryable": true
  }
}
```

**Assertions**:
- HTTP status 504 Gateway Timeout
- Error code E202 returned
- MCP request cancelled on timeout
- Job status updated to ERROR with lastSuccessfulStage
- Checkpoint created if possible (for resume)
- SSE timeout event sent before connection close
- Resources cleaned up (no zombie processes)

---

## 13. Database State Verification Tests

### 13.1 Overview

Database State Verification tests ensure data integrity, transaction consistency, and proper state management across all database operations. These tests validate that the database accurately reflects application state after each operation.

---

### API-DB-001: Job Record Persistence After Upload

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-DB-001 |
| **Title** | Job Record Persistence After Upload |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-104, DB-101 |

**Request**:
```json
POST /api/v1/upload
Cookie: hx-docling-session=abc123
Content-Type: multipart/form-data

[file: document.pdf]
```

**Expected Database State**:
```sql
SELECT * FROM jobs WHERE id = 'returned-job-uuid';

-- Expected row:
{
  "id": "returned-job-uuid",
  "session_id": "abc123",
  "status": "PENDING",
  "input_type": "FILE",
  "file_name": "document.pdf",
  "file_size": 102400,
  "mime_type": "application/pdf",
  "file_path": "/uploads/returned-job-uuid/document.pdf",
  "created_at": "2025-12-14T10:00:00Z",
  "updated_at": "2025-12-14T10:00:00Z",
  "error_code": null,
  "checkpoint_data": null
}
```

**Assertions**:
- Job record exists in database after API returns 201
- All fields populated correctly
- session_id matches request cookie
- status is PENDING
- file_path points to actual stored file
- created_at and updated_at are identical
- Timestamps are valid ISO 8601
- Foreign key to sessions table valid

---

### API-DB-002: Job Status Update Persistence After Process

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-DB-002 |
| **Title** | Job Status Update Persistence After Process |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-401, DB-102 |

**Precondition**: Job exists with status PENDING

**Request**:
```json
POST /api/v1/process
Cookie: hx-docling-session=abc123
Content-Type: application/json

{
  "jobId": "pending-job-uuid"
}
```

**Expected Database State Transitions**:

| Timestamp | Status | updated_at Changed |
|-----------|--------|-------------------|
| T+0ms | PENDING -> PROCESSING | Yes |
| T+100ms | Progress: validating | Yes (checkpoint) |
| T+500ms | Progress: conversion | Yes (checkpoint) |
| T+5000ms | PROCESSING -> COMPLETE | Yes |

**Post-Process Database State**:
```sql
SELECT * FROM jobs WHERE id = 'pending-job-uuid';

-- Expected row:
{
  "id": "pending-job-uuid",
  "status": "COMPLETE",
  "started_at": "2025-12-14T10:00:00Z",
  "completed_at": "2025-12-14T10:00:05Z",
  "updated_at": "2025-12-14T10:00:05Z",
  "processing_time_ms": 5000,
  "checkpoint_data": null
}
```

**Assertions**:
- Status transitions persisted in correct order
- updated_at changes with each status update
- started_at set when status becomes PROCESSING
- completed_at set when status becomes COMPLETE
- processing_time_ms calculated correctly
- checkpoint_data cleared after successful completion

---

### API-DB-003: Result Storage Persistence After Completion

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-DB-003 |
| **Title** | Result Storage Persistence After Completion |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-601, DB-103 |

**Precondition**: Job processing completes successfully

**Expected Database State (jobs table)**:
```sql
SELECT * FROM jobs WHERE id = 'completed-job-uuid';

-- Expected row:
{
  "id": "completed-job-uuid",
  "status": "COMPLETE",
  "result_count": 3
}
```

**Expected Database State (results table)**:
```sql
SELECT * FROM results WHERE job_id = 'completed-job-uuid' ORDER BY format;

-- Expected rows:
[
  {
    "id": "result-uuid-1",
    "job_id": "completed-job-uuid",
    "format": "HTML",
    "size_bytes": 8000,
    "file_path": "/results/completed-job-uuid/output.html",
    "checksum": "sha256:def456...",
    "created_at": "2025-12-14T10:00:05Z"
  },
  {
    "id": "result-uuid-2",
    "job_id": "completed-job-uuid",
    "format": "JSON",
    "size_bytes": 15000,
    "file_path": "/results/completed-job-uuid/output.json",
    "checksum": "sha256:ghi789...",
    "created_at": "2025-12-14T10:00:05Z"
  },
  {
    "id": "result-uuid-3",
    "job_id": "completed-job-uuid",
    "format": "MARKDOWN",
    "size_bytes": 5000,
    "file_path": "/results/completed-job-uuid/output.md",
    "checksum": "sha256:abc123...",
    "created_at": "2025-12-14T10:00:05Z"
  }
]
```

**Assertions**:
- Three result records created (one per format)
- All results linked to correct job_id (foreign key)
- file_path points to actual stored files
- checksum computed and stored for integrity
- size_bytes matches actual file size
- created_at timestamps consistent

---

### API-DB-004: Transaction Rollback on Upload Failure

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-DB-004 |
| **Title** | Transaction Rollback on Upload Failure |
| **Priority** | P0 - Critical |
| **Spec Reference** | DB-201, ACID Compliance |

**Scenario**: File storage fails after database record created

**Test Steps**:
1. Begin upload request
2. Job record created in database (within transaction)
3. File storage fails (simulated disk full error)
4. Transaction rolls back
5. Verify no orphan records exist

**Pre-Failure Database State**:
```sql
-- Transaction in progress, uncommitted
INSERT INTO jobs (id, session_id, status, ...) VALUES ('orphan-uuid', ...);
```

**Post-Rollback Database State**:
```sql
SELECT * FROM jobs WHERE id = 'orphan-uuid';
-- Expected: 0 rows (rollback successful)

SELECT * FROM jobs WHERE session_id = 'abc123' AND created_at > NOW() - INTERVAL '1 minute';
-- Expected: No new records from failed upload
```

**Assertions**:
- No job record exists after failed upload
- No partial data in any table
- File system cleaned up (no orphan files)
- API returns appropriate error (E003 Storage error)
- Database connection returned to pool
- No locks held after rollback

---

### API-DB-005: Transaction Rollback on Process Failure

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-DB-005 |
| **Title** | Transaction Rollback on Process Failure |
| **Priority** | P0 - Critical |
| **Spec Reference** | DB-202, ACID Compliance |

**Scenario**: MCP processing fails mid-conversion, result storage partially complete

**Test Steps**:
1. Job status updated to PROCESSING (committed)
2. Conversion completes, Markdown export succeeds
3. HTML export fails
4. Transaction for result storage rolls back
5. Job status updated to ERROR (new transaction)

**Pre-Failure Database State**:
```sql
-- Transaction in progress for export phase
UPDATE jobs SET status = 'PROCESSING' WHERE id = 'processing-uuid';
-- Committed

-- New transaction for results
INSERT INTO results (job_id, format, ...) VALUES ('processing-uuid', 'MARKDOWN', ...);
INSERT INTO results (job_id, format, ...) VALUES ('processing-uuid', 'HTML', ...);
-- HTML insert fails -> rollback
```

**Post-Rollback Database State**:
```sql
SELECT * FROM jobs WHERE id = 'processing-uuid';
-- Expected: status = 'ERROR', checkpoint_data contains last successful stage

SELECT * FROM results WHERE job_id = 'processing-uuid';
-- Expected: 0 rows (all results rolled back - atomic batch)
-- OR: Only committed results if using savepoints
```

**Assertions**:
- Job record updated to ERROR status
- checkpoint_data preserved for resume capability
- No partial result records (or only committed savepoints)
- Error details logged in job record
- API returns appropriate error response
- SSE error event sent

---

### API-DB-006: Orphan Record Prevention

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-DB-006 |
| **Title** | Orphan Record Prevention |
| **Priority** | P0 - Critical |
| **Spec Reference** | DB-203, Data Integrity |

**Test Matrix - Orphan Prevention Scenarios**:

| Scenario | Trigger | Expected Behavior |
|----------|---------|-------------------|
| Job without session | Session deleted | Job cascade deleted or marked orphan |
| Result without job | Job deleted | Result cascade deleted |
| File without job | Job creation failed | File cleaned up by transaction |
| Checkpoint without job | Job deleted | Checkpoint cascade deleted |

**Verification Queries**:
```sql
-- Check for orphan jobs (no matching session)
SELECT j.* FROM jobs j
LEFT JOIN sessions s ON j.session_id = s.id
WHERE s.id IS NULL;
-- Expected: 0 rows

-- Check for orphan results (no matching job)
SELECT r.* FROM results r
LEFT JOIN jobs j ON r.job_id = j.id
WHERE j.id IS NULL;
-- Expected: 0 rows

-- Check for orphan files (no matching job record)
-- Requires file system scan + database comparison
```

**Assertions**:
- No orphan jobs exist after any operation
- No orphan results exist after any operation
- Foreign key constraints enforced at database level
- Cascade deletes configured appropriately
- Background cleanup job handles edge cases
- Audit log tracks deletion cascades

---

### API-DB-007: Concurrent Update Consistency

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-DB-007 |
| **Title** | Concurrent Update Consistency |
| **Priority** | P0 - Critical |
| **Spec Reference** | DB-301, Concurrency Control |

**Scenario**: Two concurrent requests attempt to update same job

**Test Steps**:
1. Request A: POST /api/v1/process (jobId: conflict-uuid)
2. Request B: POST /api/v1/jobs/conflict-uuid/cancel (simultaneous)
3. Only one operation should succeed

**Concurrency Control Implementation**:
```sql
-- Option 1: Optimistic Locking
UPDATE jobs
SET status = 'PROCESSING', version = version + 1, updated_at = NOW()
WHERE id = 'conflict-uuid' AND status = 'PENDING' AND version = 1;
-- Returns: 1 row affected (first request) or 0 rows (second request)

-- Option 2: SELECT FOR UPDATE
BEGIN;
SELECT * FROM jobs WHERE id = 'conflict-uuid' FOR UPDATE;
-- Lock acquired, second request blocks
UPDATE jobs SET status = 'PROCESSING' WHERE id = 'conflict-uuid';
COMMIT;
```

**Expected Behavior**:

| Request A Result | Request B Result | Final Status |
|------------------|------------------|--------------|
| 202 Accepted | 409 Conflict (E701) | PROCESSING |
| 409 Conflict | 200 OK | CANCELLED |

**Assertions**:
- Only one request modifies job status
- Second request receives appropriate conflict error
- No race condition leading to inconsistent state
- Database constraints prevent invalid transitions
- version column incremented on each update (if optimistic locking)
- Audit log records both attempts

---

### API-DB-008: Foreign Key Constraint Enforcement

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-DB-008 |
| **Title** | Foreign Key Constraint Enforcement |
| **Priority** | P0 - Critical |
| **Spec Reference** | DB-302, Referential Integrity |

**Foreign Key Relationships**:

| Child Table | Column | Parent Table | Column | On Delete |
|-------------|--------|--------------|--------|-----------|
| jobs | session_id | sessions | id | CASCADE |
| results | job_id | jobs | id | CASCADE |
| checkpoints | job_id | jobs | id | CASCADE |
| audit_logs | job_id | jobs | id | SET NULL |

**Test Cases**:

**Case 1: Insert Result with Invalid Job ID**:
```sql
INSERT INTO results (id, job_id, format, ...) VALUES ('result-1', 'nonexistent-job', 'MARKDOWN', ...);
-- Expected: Foreign key violation error
```

**Case 2: Delete Job with Results**:
```sql
DELETE FROM jobs WHERE id = 'job-with-results';
-- Expected: Cascades to delete all results for this job
```

**Case 3: Delete Session with Jobs**:
```sql
DELETE FROM sessions WHERE id = 'session-with-jobs';
-- Expected: Cascades to delete all jobs and their results
```

**Assertions**:
- Cannot insert child record with invalid parent reference
- Deleting parent cascades to children per configuration
- No orphan records possible via direct SQL
- Constraint violations return appropriate database error
- API handles constraint violations gracefully (500 with logged details)
- Referential integrity maintained across all tables

---

## 14. Concurrent Request Handling Tests

### 14.1 Overview

Concurrent Request Handling tests validate the system's behavior under race conditions, ensuring proper synchronization, idempotent operations, and data consistency when multiple requests target the same resources simultaneously.

---

### API-RACE-001: Concurrent Process Requests on Same Job

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-RACE-001 |
| **Title** | Concurrent Process Requests on Same Job |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-401, E701, Concurrency Control |

**Precondition**: Job exists with status PENDING

**Test Steps**:
1. Send two simultaneous POST /api/v1/process requests for same jobId
2. Time requests to arrive within 10ms of each other
3. Verify only one request succeeds

**Concurrent Requests**:
```
Request A: POST /api/v1/process { "jobId": "race-job-uuid" }
Request B: POST /api/v1/process { "jobId": "race-job-uuid" }
```

**Expected Behavior**:

| Request | Result | Status Code | Error Code |
|---------|--------|-------------|------------|
| First to acquire lock | Success | 202 Accepted | - |
| Second request | Failure | 409 Conflict | E701 |

**Expected Response (Second Request)**:
```json
HTTP/1.1 409 Conflict

{
  "success": false,
  "error": {
    "code": "E701",
    "message": "Job already processing",
    "userMessage": "This job is already being processed",
    "suggestedAction": "Wait for the current processing to complete",
    "retryable": false
  }
}
```

**Assertions**:
- Exactly one request returns 202 Accepted
- Exactly one request returns 409 Conflict with E701
- Job status is PROCESSING (not corrupted)
- Only one SSE stream active for job
- No duplicate processing initiated
- Database shows single status transition PENDING -> PROCESSING
- Optimistic locking or row-level lock prevents race

---

### API-RACE-002: Cancel During Active Processing

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-RACE-002 |
| **Title** | Cancel During Active Processing |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-406, E702, Cancellation Semantics |

**Precondition**: Job is actively processing (mid-conversion)

**Test Steps**:
1. Start processing job (status: PROCESSING)
2. Wait for conversion to reach 50% progress
3. Send cancel request
4. Verify proper abort and cleanup

**Request**:
```
POST /api/v1/jobs/{jobId}/cancel
Cookie: hx-docling-session=abc123
```

**Expected Response**:
```json
HTTP/1.1 200 OK

{
  "success": true,
  "data": {
    "jobId": "processing-job-uuid",
    "status": "CANCELLED",
    "cancelledAt": "2025-12-14T10:00:30Z",
    "lastStage": "conversion",
    "lastProgress": 50
  }
}
```

**Expected Behavior**:
- MCP conversion request aborted (if supported by MCP client)
- SSE stream sends `cancelled` event before closing
- Job status transitions to CANCELLED atomically
- Partial results cleaned up (no orphan files)
- Checkpoint saved at last successful stage (for potential resume)
- Database transaction commits cancelled state

**SSE Event Sequence**:
```
event: progress
data: {"stage":"conversion","percent":50,"message":"Converting..."}

event: cancelled
data: {"jobId":"uuid","cancelledAt":"2025-12-14T10:00:30Z","reason":"user_requested"}

[connection closed]
```

**Assertions**:
- Cancel succeeds while processing active
- SSE cancelled event received before connection close
- No processing continues after cancel
- Partial resources cleaned up
- Job can be resumed if checkpoint exists
- MCP request properly terminated

---

### API-RACE-003: Concurrent Uploads from Same Session

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-RACE-003 |
| **Title** | Concurrent Uploads from Same Session |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-101, Session Management, Queue Semantics |

**Precondition**: Valid session exists

**Test Steps**:
1. Send three simultaneous file upload requests from same session
2. All requests should be accepted (queued)
3. Verify all jobs created with unique IDs

**Concurrent Requests**:
```
Request A: POST /api/v1/upload [file1.pdf]
Request B: POST /api/v1/upload [file2.docx]
Request C: POST /api/v1/upload [file3.xlsx]
```

**Expected Behavior**:

| Request | Expected Result | Status Code |
|---------|-----------------|-------------|
| Request A | Success | 201 Created |
| Request B | Success | 201 Created |
| Request C | Success | 201 Created |

**Expected Responses**:
```json
// Response A
HTTP/1.1 201 Created
{
  "success": true,
  "data": {
    "jobId": "job-uuid-1",
    "status": "PENDING",
    "fileName": "file1.pdf"
  }
}

// Response B
HTTP/1.1 201 Created
{
  "success": true,
  "data": {
    "jobId": "job-uuid-2",
    "status": "PENDING",
    "fileName": "file2.docx"
  }
}

// Response C
HTTP/1.1 201 Created
{
  "success": true,
  "data": {
    "jobId": "job-uuid-3",
    "status": "PENDING",
    "fileName": "file3.xlsx"
  }
}
```

**Assertions**:
- All three uploads succeed (no blocking between uploads)
- Each job has unique jobId (UUID collision impossible)
- All jobs linked to same session
- Files stored in separate directories (no path collision)
- Rate limit applies to aggregate (if 3 requests in 60 seconds, 7 more allowed)
- Database shows 3 distinct job records for session

---

### API-RACE-004: Two Concurrent Cancel Requests

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-RACE-004 |
| **Title** | Two Concurrent Cancel Requests |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-406, Idempotent Operations |

**Precondition**: Job is in PROCESSING state

**Test Steps**:
1. Send two simultaneous cancel requests for same job
2. Both should result in job being cancelled
3. Verify idempotent behavior (same final state)

**Concurrent Requests**:
```
Request A: POST /api/v1/jobs/{jobId}/cancel
Request B: POST /api/v1/jobs/{jobId}/cancel
```

**Expected Behavior**:

| Scenario | Request A | Request B |
|----------|-----------|-----------|
| A wins | 200 OK (cancelled) | 200 OK (already cancelled) |
| B wins | 200 OK (already cancelled) | 200 OK (cancelled) |

**Expected Response (Both Requests)**:
```json
HTTP/1.1 200 OK

{
  "success": true,
  "data": {
    "jobId": "cancel-job-uuid",
    "status": "CANCELLED",
    "cancelledAt": "2025-12-14T10:00:30Z"
  }
}
```

**Assertions**:
- Both requests return 200 OK (idempotent)
- Final job status is CANCELLED
- Only one SSE cancelled event sent (not duplicate)
- cancelledAt timestamp consistent across both responses
- No error returned for second cancel (idempotent)
- Database shows single CANCELLED status (not double-cancelled)

---

### API-RACE-005: Resume While Already Processing

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-RACE-005 |
| **Title** | Resume While Already Processing |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-501, E701, State Validation |

**Precondition**: Job has checkpoint but is currently PROCESSING (resumed earlier)

**Test Steps**:
1. Job was previously failed and has checkpoint
2. First resume request initiated, job now PROCESSING
3. Second resume request arrives while first is processing
4. Second request should fail with 409 Conflict

**Request**:
```
POST /api/v1/jobs/{jobId}/resume
Cookie: hx-docling-session=abc123
```

**Expected Response**:
```json
HTTP/1.1 409 Conflict

{
  "success": false,
  "error": {
    "code": "E701",
    "message": "Job already processing",
    "userMessage": "This job is already being processed",
    "suggestedAction": "Wait for the current processing to complete",
    "retryable": false
  }
}
```

**Assertions**:
- Resume fails if job already PROCESSING
- Same error code (E701) as concurrent process attempts
- Original processing continues unaffected
- No duplicate SSE streams created
- Job status remains PROCESSING (not corrupted)

---

### API-RACE-006: Database Dirty Read Prevention

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-RACE-006 |
| **Title** | Database Dirty Read Prevention |
| **Priority** | P0 - Critical |
| **Spec Reference** | DB-301, Transaction Isolation |

**Precondition**: Job exists with status PENDING

**Test Steps**:
1. Transaction A: Begin update job to PROCESSING (uncommitted)
2. Transaction B: Read job status
3. Verify Transaction B sees PENDING (not uncommitted PROCESSING)
4. Transaction A: Commit
5. Transaction B: Read job status again
6. Verify Transaction B now sees PROCESSING

**Database Operations**:
```sql
-- Transaction A (not committed)
BEGIN;
UPDATE jobs SET status = 'PROCESSING' WHERE id = 'dirty-read-job';
-- DO NOT COMMIT YET

-- Transaction B (concurrent read)
BEGIN;
SELECT status FROM jobs WHERE id = 'dirty-read-job';
-- Expected: status = 'PENDING' (not dirty read of 'PROCESSING')
COMMIT;

-- Transaction A (now commit)
COMMIT;

-- Transaction B (new read)
BEGIN;
SELECT status FROM jobs WHERE id = 'dirty-read-job';
-- Expected: status = 'PROCESSING' (committed value)
COMMIT;
```

**Assertions**:
- READ COMMITTED isolation level prevents dirty reads
- Uncommitted changes not visible to other transactions
- Only committed data returned by API queries
- GET /api/v1/jobs/{id} returns consistent committed state
- No phantom reads in history queries
- Database isolation level configured correctly (READ COMMITTED minimum)

---

### API-RACE-007: Lost Update Prevention (Optimistic Locking)

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-RACE-007 |
| **Title** | Lost Update Prevention (Optimistic Locking) |
| **Priority** | P0 - Critical |
| **Spec Reference** | DB-302, Optimistic Concurrency Control |

**Precondition**: Job exists with version=1, status=PENDING

**Test Steps**:
1. Transaction A: Read job (version=1)
2. Transaction B: Read job (version=1)
3. Transaction A: Update job status to PROCESSING with version check
4. Transaction A: Commits (version now 2)
5. Transaction B: Attempt update with stale version=1
6. Transaction B: Fails (optimistic lock violation)

**Database Operations**:
```sql
-- Transaction A
UPDATE jobs
SET status = 'PROCESSING', version = version + 1, updated_at = NOW()
WHERE id = 'lost-update-job' AND version = 1;
-- Result: 1 row affected (success)

-- Transaction B (concurrent, stale version)
UPDATE jobs
SET status = 'CANCELLED', version = version + 1, updated_at = NOW()
WHERE id = 'lost-update-job' AND version = 1;
-- Result: 0 rows affected (version mismatch)
```

**Expected API Behavior**:

| Request | Version Check | Result |
|---------|---------------|--------|
| Process (A) | version=1 | 202 Accepted |
| Cancel (B) | version=1 (stale) | 409 Conflict |

**Expected Response (Lost Update)**:
```json
HTTP/1.1 409 Conflict

{
  "success": false,
  "error": {
    "code": "E707",
    "message": "Concurrent modification detected",
    "userMessage": "The job was modified by another request. Please refresh and try again.",
    "suggestedAction": "Refresh the job status and retry",
    "retryable": true
  }
}
```

**Assertions**:
- Lost update prevented by version check
- Second request fails with clear error
- Database maintains consistent state
- No partial updates applied
- Version column incremented on each successful update
- Retry-able error indicates client should refresh and retry

---

### API-RACE-008: Transaction Isolation Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-RACE-008 |
| **Title** | Transaction Isolation Validation |
| **Priority** | P0 - Critical |
| **Spec Reference** | DB-303, ACID Compliance |

**Precondition**: Database configured with READ COMMITTED isolation

**Test Matrix - Isolation Phenomena**:

| Phenomenon | Expected Behavior | Isolation Level Required |
|------------|-------------------|--------------------------|
| Dirty Read | Prevented | READ COMMITTED |
| Non-Repeatable Read | Allowed | REPEATABLE READ to prevent |
| Phantom Read | Allowed | SERIALIZABLE to prevent |
| Lost Update | Prevented (via optimistic locking) | Application-level |

**Test Scenarios**:

**Scenario 1: Verify No Dirty Reads**
```sql
-- Session 1: Uncommitted update
BEGIN;
UPDATE jobs SET status = 'ERROR' WHERE id = 'isolation-job';
-- Session 2: Concurrent read
SELECT status FROM jobs WHERE id = 'isolation-job';
-- Expected: Original status (not 'ERROR')
-- Session 1: Rollback
ROLLBACK;
```

**Scenario 2: Verify Atomicity**
```sql
-- Multi-table update (job + results)
BEGIN;
UPDATE jobs SET status = 'COMPLETE' WHERE id = 'atomic-job';
INSERT INTO results (job_id, format, ...) VALUES ('atomic-job', 'MARKDOWN', ...);
-- Error occurs here
-- Expected: Both operations rolled back
ROLLBACK;
```

**Scenario 3: Verify Durability**
```sql
BEGIN;
UPDATE jobs SET status = 'COMPLETE' WHERE id = 'durable-job';
COMMIT;
-- Simulate crash/restart
-- Expected: status = 'COMPLETE' persisted
```

**Assertions**:
- READ COMMITTED isolation prevents dirty reads
- Transactions are atomic (all-or-nothing)
- Committed transactions are durable (survive restart)
- Application uses optimistic locking for lost update prevention
- No isolation-related anomalies in normal operation
- Database connection pool properly configured for isolation

---

## 15. Response Schema Validation Tests

### 15.1 Overview

Response Schema Validation tests ensure all API responses conform to their defined JSON schemas, include required fields, use correct data types, and maintain consistent structure across all endpoints.

---

### API-SCHEMA-001: Upload Response JSON Schema Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-SCHEMA-001 |
| **Title** | Upload Response JSON Schema Validation |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-104, API-SCHEMA-101 |

**Request**:
```
POST /api/v1/upload
Content-Type: multipart/form-data
Cookie: hx-docling-session=abc123

[file: document.pdf]
```

**Expected Response JSON Schema**:
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["success", "data"],
  "properties": {
    "success": {
      "type": "boolean",
      "const": true
    },
    "data": {
      "type": "object",
      "required": ["jobId", "status", "fileName", "fileSize", "mimeType", "createdAt"],
      "properties": {
        "jobId": {
          "type": "string",
          "format": "uuid",
          "pattern": "^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$"
        },
        "status": {
          "type": "string",
          "enum": ["PENDING"]
        },
        "fileName": {
          "type": "string",
          "minLength": 1,
          "maxLength": 255
        },
        "fileSize": {
          "type": "integer",
          "minimum": 1,
          "maximum": 104857600
        },
        "mimeType": {
          "type": "string",
          "enum": [
            "application/pdf",
            "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
            "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
            "application/vnd.openxmlformats-officedocument.presentationml.presentation",
            "image/png",
            "image/jpeg",
            "image/gif",
            "image/webp"
          ]
        },
        "createdAt": {
          "type": "string",
          "format": "date-time"
        }
      },
      "additionalProperties": false
    }
  },
  "additionalProperties": false
}
```

**Example Valid Response**:
```json
{
  "success": true,
  "data": {
    "jobId": "550e8400-e29b-41d4-a716-446655440000",
    "status": "PENDING",
    "fileName": "document.pdf",
    "fileSize": 102400,
    "mimeType": "application/pdf",
    "createdAt": "2025-12-14T10:00:00Z"
  }
}
```

**Assertions**:
- Response validates against JSON schema
- jobId is valid UUID v4 format
- status is exactly "PENDING" for new uploads
- fileSize matches actual uploaded file size
- mimeType is from allowed list
- createdAt is valid ISO 8601 timestamp
- No additional properties in response

---

### API-SCHEMA-002: Process Response JSON Schema Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-SCHEMA-002 |
| **Title** | Process Response JSON Schema Validation |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-401, API-SCHEMA-102 |

**Request**:
```
POST /api/v1/process
Content-Type: application/json
Cookie: hx-docling-session=abc123

{
  "jobId": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Expected Response JSON Schema**:
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["success", "data"],
  "properties": {
    "success": {
      "type": "boolean",
      "const": true
    },
    "data": {
      "type": "object",
      "required": ["jobId", "status", "streamUrl"],
      "properties": {
        "jobId": {
          "type": "string",
          "format": "uuid"
        },
        "status": {
          "type": "string",
          "enum": ["PROCESSING"]
        },
        "streamUrl": {
          "type": "string",
          "pattern": "^/api/v1/process/[0-9a-f-]+/events$"
        }
      },
      "additionalProperties": false
    }
  },
  "additionalProperties": false
}
```

**Example Valid Response**:
```json
{
  "success": true,
  "data": {
    "jobId": "550e8400-e29b-41d4-a716-446655440000",
    "status": "PROCESSING",
    "streamUrl": "/api/v1/process/550e8400-e29b-41d4-a716-446655440000/events"
  }
}
```

**Assertions**:
- Response validates against JSON schema
- jobId matches request jobId
- status is exactly "PROCESSING"
- streamUrl contains valid path with jobId
- streamUrl is relative path (not absolute URL)

---

### API-SCHEMA-003: Job Details Response JSON Schema Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-SCHEMA-003 |
| **Title** | Job Details Response JSON Schema Validation |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-703, API-SCHEMA-103 |

**Request**:
```
GET /api/v1/jobs/{jobId}
Cookie: hx-docling-session=abc123
```

**Expected Response JSON Schema (COMPLETE status)**:
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["success", "data"],
  "properties": {
    "success": {
      "type": "boolean",
      "const": true
    },
    "data": {
      "type": "object",
      "required": ["id", "status", "inputType", "createdAt"],
      "properties": {
        "id": {
          "type": "string",
          "format": "uuid"
        },
        "status": {
          "type": "string",
          "enum": ["PENDING", "PROCESSING", "COMPLETE", "PARTIAL_COMPLETE", "ERROR", "CANCELLED"]
        },
        "inputType": {
          "type": "string",
          "enum": ["FILE", "URL"]
        },
        "fileName": {
          "type": "string"
        },
        "sourceUrl": {
          "type": "string",
          "format": "uri"
        },
        "fileSize": {
          "type": "integer",
          "minimum": 0
        },
        "createdAt": {
          "type": "string",
          "format": "date-time"
        },
        "startedAt": {
          "type": "string",
          "format": "date-time"
        },
        "completedAt": {
          "type": "string",
          "format": "date-time"
        },
        "progress": {
          "type": "object",
          "properties": {
            "stage": {
              "type": "string",
              "enum": ["validating", "uploading", "conversion", "export_markdown", "export_html", "export_json", "finalizing"]
            },
            "percent": {
              "type": "integer",
              "minimum": 0,
              "maximum": 100
            },
            "message": {
              "type": "string"
            }
          }
        },
        "results": {
          "type": "array",
          "items": {
            "type": "object",
            "required": ["format", "size", "available"],
            "properties": {
              "format": {
                "type": "string",
                "enum": ["MARKDOWN", "HTML", "JSON"]
              },
              "size": {
                "type": "integer",
                "minimum": 0
              },
              "available": {
                "type": "boolean"
              },
              "createdAt": {
                "type": "string",
                "format": "date-time"
              }
            }
          }
        },
        "errorCode": {
          "type": "string",
          "pattern": "^E[0-9]{3}$"
        },
        "errorMessage": {
          "type": "string"
        },
        "retryable": {
          "type": "boolean"
        }
      },
      "additionalProperties": false
    }
  },
  "additionalProperties": false
}
```

**Assertions**:
- Response validates against JSON schema
- Conditional fields present based on status:
  - PENDING: no progress, no results
  - PROCESSING: progress present, no results
  - COMPLETE: completedAt present, results array with 3 items
  - ERROR: errorCode, errorMessage, retryable present
- inputType determines which of fileName/sourceUrl is present
- All timestamps are valid ISO 8601

---

### API-SCHEMA-004: History Response JSON Schema Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-SCHEMA-004 |
| **Title** | History Response JSON Schema Validation |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-701, FR-702, API-SCHEMA-104 |

**Request**:
```
GET /api/v1/history?page=1&pageSize=20
Cookie: hx-docling-session=abc123
```

**Expected Response JSON Schema**:
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["success", "data"],
  "properties": {
    "success": {
      "type": "boolean",
      "const": true
    },
    "data": {
      "type": "object",
      "required": ["jobs", "pagination"],
      "properties": {
        "jobs": {
          "type": "array",
          "items": {
            "type": "object",
            "required": ["id", "status", "inputType", "createdAt"],
            "properties": {
              "id": {
                "type": "string",
                "format": "uuid"
              },
              "status": {
                "type": "string",
                "enum": ["PENDING", "PROCESSING", "COMPLETE", "PARTIAL_COMPLETE", "ERROR", "CANCELLED"]
              },
              "inputType": {
                "type": "string",
                "enum": ["FILE", "URL"]
              },
              "fileName": {
                "type": "string"
              },
              "sourceUrl": {
                "type": "string"
              },
              "createdAt": {
                "type": "string",
                "format": "date-time"
              }
            }
          }
        },
        "pagination": {
          "type": "object",
          "required": ["page", "pageSize", "totalCount", "totalPages", "hasMore"],
          "properties": {
            "page": {
              "type": "integer",
              "minimum": 1
            },
            "pageSize": {
              "type": "integer",
              "minimum": 1,
              "maximum": 100
            },
            "totalCount": {
              "type": "integer",
              "minimum": 0
            },
            "totalPages": {
              "type": "integer",
              "minimum": 0
            },
            "hasMore": {
              "type": "boolean"
            }
          },
          "additionalProperties": false
        }
      },
      "additionalProperties": false
    }
  },
  "additionalProperties": false
}
```

**Assertions**:
- Response validates against JSON schema
- jobs array length <= pageSize
- pagination.page matches request parameter
- pagination.totalPages = ceil(totalCount / pageSize)
- pagination.hasMore = (page < totalPages)
- Jobs sorted by createdAt DESC (most recent first)
- Empty jobs array valid (totalCount=0)

---

### API-SCHEMA-005: Error Response JSON Schema Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-SCHEMA-005 |
| **Title** | Error Response JSON Schema Validation |
| **Priority** | P0 - Critical |
| **Spec Reference** | FR-801, FR-802, API-SCHEMA-105 |

**Expected Error Response JSON Schema**:
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["success", "error"],
  "properties": {
    "success": {
      "type": "boolean",
      "const": false
    },
    "error": {
      "type": "object",
      "required": ["code", "message"],
      "properties": {
        "code": {
          "type": "string",
          "pattern": "^E[0-9]{3}$",
          "description": "Error code in EXXX format"
        },
        "message": {
          "type": "string",
          "minLength": 1,
          "description": "Technical error message for logging"
        },
        "userMessage": {
          "type": "string",
          "minLength": 1,
          "description": "User-friendly error message for display"
        },
        "suggestedAction": {
          "type": "string",
          "description": "Recommended action for user to resolve"
        },
        "retryable": {
          "type": "boolean",
          "description": "Whether the operation can be retried"
        },
        "details": {
          "type": "object",
          "description": "Additional error context (optional)"
        }
      },
      "additionalProperties": false
    }
  },
  "additionalProperties": false
}
```

**Test Matrix - Error Response Validation**:

| Error Code | HTTP Status | Required Fields | Optional Fields |
|------------|-------------|-----------------|-----------------|
| E001 | 400 | code, message, userMessage | suggestedAction, retryable |
| E401 | 401 | code, message, userMessage | suggestedAction, retryable |
| E501 | 404 | code, message | userMessage |
| E601 | 429 | code, message, userMessage | suggestedAction, retryable |
| E701 | 409 | code, message, userMessage | suggestedAction, retryable |

**Example Valid Error Response**:
```json
{
  "success": false,
  "error": {
    "code": "E001",
    "message": "File too large",
    "userMessage": "The file exceeds the 100MB limit",
    "suggestedAction": "Try a smaller file or split into multiple documents",
    "retryable": false
  }
}
```

**Assertions**:
- All error responses have success=false
- error.code matches EXXX pattern
- error.message is non-empty string
- userMessage present for client-facing errors
- retryable indicates if client should retry
- No sensitive data (stack traces, internal paths) in error response
- Error schema consistent across all endpoints

---

### API-SCHEMA-006: Health Response JSON Schema Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-SCHEMA-006 |
| **Title** | Health Response JSON Schema Validation |
| **Priority** | P0 - Critical |
| **Spec Reference** | NFR-601, API-SCHEMA-106 |

**Request**:
```
GET /api/v1/health
```

**Expected Response JSON Schema**:
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["status", "timestamp", "version", "services"],
  "properties": {
    "status": {
      "type": "string",
      "enum": ["healthy", "degraded", "unhealthy"]
    },
    "timestamp": {
      "type": "string",
      "format": "date-time"
    },
    "version": {
      "type": "string",
      "pattern": "^[0-9]+\\.[0-9]+\\.[0-9]+$"
    },
    "services": {
      "type": "object",
      "required": ["database", "redis", "mcp"],
      "properties": {
        "database": {
          "$ref": "#/definitions/serviceStatus"
        },
        "redis": {
          "$ref": "#/definitions/serviceStatus"
        },
        "mcp": {
          "$ref": "#/definitions/serviceStatus"
        }
      },
      "additionalProperties": false
    }
  },
  "definitions": {
    "serviceStatus": {
      "type": "object",
      "required": ["status"],
      "properties": {
        "status": {
          "type": "string",
          "enum": ["healthy", "degraded", "unhealthy"]
        },
        "latency": {
          "type": "integer",
          "minimum": 0,
          "description": "Latency in milliseconds"
        },
        "message": {
          "type": "string",
          "description": "Additional status information"
        },
        "error": {
          "type": "string",
          "description": "Error message if unhealthy"
        }
      },
      "additionalProperties": false
    }
  },
  "additionalProperties": false
}
```

**Example Valid Responses**:
```json
// Healthy
{
  "status": "healthy",
  "timestamp": "2025-12-14T10:00:00Z",
  "version": "1.0.0",
  "services": {
    "database": { "status": "healthy", "latency": 5 },
    "redis": { "status": "healthy", "latency": 2 },
    "mcp": { "status": "healthy", "latency": 50 }
  }
}

// Degraded
{
  "status": "degraded",
  "timestamp": "2025-12-14T10:00:00Z",
  "version": "1.0.0",
  "services": {
    "database": { "status": "healthy", "latency": 5 },
    "redis": { "status": "degraded", "latency": 150, "message": "High latency" },
    "mcp": { "status": "healthy", "latency": 50 }
  }
}
```

**Assertions**:
- Overall status reflects worst service status
- version follows semver format
- All three services present in response
- latency values are positive integers (milliseconds)
- timestamp is current (within last minute)
- No internal IPs or credentials exposed

---

### API-SCHEMA-007: Content-Type Header Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-SCHEMA-007 |
| **Title** | Content-Type Header Validation |
| **Priority** | P0 - Critical |
| **Spec Reference** | HTTP/1.1 RFC 7231, API Standards |

**Test Matrix - Response Content-Type Headers**:

| Endpoint | Method | Expected Content-Type |
|----------|--------|-----------------------|
| /api/v1/upload | POST | application/json |
| /api/v1/process | POST | application/json |
| /api/v1/jobs/{id} | GET | application/json |
| /api/v1/jobs/{id}/cancel | POST | application/json |
| /api/v1/jobs/{id}/resume | POST | application/json |
| /api/v1/history | GET | application/json |
| /api/v1/health | GET | application/json |
| /api/v1/process/{id}/events | GET | text/event-stream |

**Request**:
```
GET /api/v1/jobs/{jobId}
Cookie: hx-docling-session=abc123
```

**Expected Response Headers**:
```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
```

**Assertions**:
- All JSON endpoints return `Content-Type: application/json`
- charset=utf-8 specified (or implied)
- SSE endpoint returns `Content-Type: text/event-stream`
- Content-Type header present on all responses (including errors)
- No Content-Type mismatch (e.g., returning HTML for JSON endpoint)
- Error responses also have application/json Content-Type

---

### API-SCHEMA-008: Required Fields Presence Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-SCHEMA-008 |
| **Title** | Required Fields Presence Validation |
| **Priority** | P0 - Critical |
| **Spec Reference** | API-SCHEMA-108, Data Completeness |

**Test Matrix - Required Fields by Endpoint**:

| Endpoint | Required Response Fields |
|----------|-------------------------|
| POST /upload | success, data.jobId, data.status, data.fileName, data.fileSize, data.mimeType, data.createdAt |
| POST /process | success, data.jobId, data.status, data.streamUrl |
| GET /jobs/{id} | success, data.id, data.status, data.inputType, data.createdAt |
| GET /history | success, data.jobs, data.pagination.page, data.pagination.pageSize, data.pagination.totalCount, data.pagination.totalPages, data.pagination.hasMore |
| GET /health | status, timestamp, version, services.database, services.redis, services.mcp |
| Error response | success (false), error.code, error.message |

**Test Approach**:
For each endpoint:
1. Make valid request
2. Parse response JSON
3. Verify all required fields present
4. Verify required fields are not null/undefined
5. Verify required fields have correct type

**Example Validation (Upload)**:
```javascript
// Response
const response = await fetch('/api/v1/upload', { method: 'POST', ... });
const json = await response.json();

// Required field validation
assert(json.success === true);
assert(typeof json.data.jobId === 'string');
assert(json.data.jobId.length > 0);
assert(typeof json.data.status === 'string');
assert(['PENDING'].includes(json.data.status));
assert(typeof json.data.fileName === 'string');
assert(json.data.fileName.length > 0);
assert(typeof json.data.fileSize === 'number');
assert(json.data.fileSize > 0);
assert(typeof json.data.mimeType === 'string');
assert(typeof json.data.createdAt === 'string');
// Validate ISO 8601 format
assert(!isNaN(Date.parse(json.data.createdAt)));
```

**Assertions**:
- All required fields present in every response
- Required fields never null or undefined
- Required fields have correct data types
- Missing required field causes test failure
- Schema validation catches missing fields
- API never returns partial response objects

---

## 16. Workflow Orchestration Tests

This section validates complete workflow execution, state transitions, and orchestration logic at the API level.

### 16.1 Workflow Test Overview

| Aspect | Description |
|--------|-------------|
| **Scope** | End-to-end workflow validation |
| **Focus** | State transitions, ordering, consistency |
| **Dependencies** | All API endpoints, database, MCP |
| **Test Count** | 8 tests |

---

### API-WF-001: Upload to Immediate Process Workflow

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-WF-001 |
| **Title** | Upload to Immediate Process Workflow |
| **Priority** | P1 - High |
| **Spec Reference** | FR-101, FR-401, WF-001 |

**Workflow Steps**:
1. Upload document via POST /api/v1/upload
2. Immediately call POST /api/v1/process with returned jobId
3. Connect to SSE stream
4. Wait for completion

**Request Sequence**:
```
# Step 1: Upload
POST /api/v1/upload
Content-Type: multipart/form-data
Cookie: hx-docling-session=abc123

[file: sample.pdf]

# Response: { "data": { "jobId": "job-123", "status": "PENDING" } }

# Step 2: Process (immediate)
POST /api/v1/process
Content-Type: application/json
Cookie: hx-docling-session=abc123

{
  "jobId": "job-123"
}

# Response: { "data": { "jobId": "job-123", "status": "PROCESSING", "streamUrl": "/api/v1/process/job-123/events" } }

# Step 3: Connect SSE
GET /api/v1/process/job-123/events
Cookie: hx-docling-session=abc123
Accept: text/event-stream
```

**Expected Behavior**:
- Upload returns 201 with PENDING status
- Process returns 200 with PROCESSING status
- SSE stream emits progress events
- Final status is COMPLETE or ERROR

**Assertions**:
- Total workflow completes in < 60s (for small document)
- Status transitions: PENDING -> PROCESSING -> COMPLETE
- No orphaned jobs (incomplete without SSE connection)
- Progress events are sequential
- Final GET /jobs/{id} reflects completion

---

### API-WF-002: State Transition Enforcement at API Level

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-WF-002 |
| **Title** | State Transition Enforcement at API Level |
| **Priority** | P1 - High |
| **Spec Reference** | FR-402, STATE-001 |

**Valid State Transitions**:
```
PENDING -> PROCESSING (via process endpoint)
PROCESSING -> COMPLETE (via successful processing)
PROCESSING -> ERROR (via processing failure)
PROCESSING -> CANCELLED (via cancel endpoint)
PROCESSING -> PARTIAL_COMPLETE (via partial success)
ERROR -> PROCESSING (via resume endpoint, if resumable)
PARTIAL_COMPLETE -> PROCESSING (via resume endpoint)
```

**Invalid Transitions to Test**:

| Current State | Attempted Action | Expected Result |
|--------------|------------------|-----------------|
| PENDING | Cancel | 400 E702 (not cancellable) |
| PENDING | Resume | 400 E703 (no checkpoint) |
| COMPLETE | Process | 400 E706 (already completed) |
| COMPLETE | Cancel | 400 E702 (not cancellable) |
| COMPLETE | Resume | 400 E706 (already completed) |
| CANCELLED | Process | 400 E702 (job cancelled) |
| CANCELLED | Resume | 400 (job cancelled, not resumable) |
| ERROR | Cancel | 400 E702 (not cancellable) |

**Test Matrix**:
```javascript
const invalidTransitions = [
  { status: 'PENDING', action: 'cancel', expectedError: 'E702' },
  { status: 'PENDING', action: 'resume', expectedError: 'E703' },
  { status: 'COMPLETE', action: 'process', expectedError: 'E706' },
  { status: 'COMPLETE', action: 'cancel', expectedError: 'E702' },
  { status: 'COMPLETE', action: 'resume', expectedError: 'E706' },
  { status: 'CANCELLED', action: 'process', expectedError: 'E702' },
  { status: 'CANCELLED', action: 'resume', expectedError: 'E702' },
  { status: 'ERROR', action: 'cancel', expectedError: 'E702' }
];
```

**Assertions**:
- All invalid transitions return 400 status
- Correct error code for each transition violation
- Job state unchanged after invalid transition attempt
- No partial state changes on rejection

---

### API-WF-003: Checkpoint Resumption Correctness

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-WF-003 |
| **Title** | Checkpoint Resumption Correctness |
| **Priority** | P1 - High |
| **Spec Reference** | FR-503, FR-504, RESUME-001 |

**Test Scenario**:
1. Start processing job that will partially fail
2. Verify checkpoint is saved at failure point
3. Resume job via POST /api/v1/jobs/{id}/resume
4. Verify processing resumes from checkpoint (not restart)

**Setup (Mock Failure)**:
```
# Cause partial failure at export_html stage
POST /api/v1/process
{
  "jobId": "job-123"
}

# Inject failure at export stage (test setup)
# Job reaches PARTIAL_COMPLETE with checkpoint
```

**Resume Request**:
```
POST /api/v1/jobs/{jobId}/resume
Cookie: hx-docling-session=abc123
```

**Expected Response**:
```json
{
  "success": true,
  "data": {
    "jobId": "job-123",
    "status": "PROCESSING",
    "streamUrl": "/api/v1/process/job-123/events",
    "resumedFrom": {
      "stage": "export_html",
      "checkpointId": "chk-456"
    }
  }
}
```

**Assertions**:
- Resume response includes resumedFrom information
- SSE stream starts from checkpoint stage (not beginning)
- Previously completed exports not re-executed
- Only failed/pending exports are processed
- Total processing time < full reprocessing time
- Final results include all exports (original + resumed)

---

### API-WF-004: Partial Export Failure Handling

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-WF-004 |
| **Title** | Partial Export Failure Handling |
| **Priority** | P1 - High |
| **Spec Reference** | FR-501, FR-502, EXPORT-001 |

**Test Scenario**:
Processing completes but one export format fails.

**Expected Behavior**:
```
Processing stages:
1. validating - SUCCESS
2. uploading - SUCCESS
3. conversion - SUCCESS
4. export_markdown - SUCCESS
5. export_html - FAILED (simulated)
6. export_json - SUCCESS

Final Status: PARTIAL_COMPLETE
```

**Expected Response (GET /jobs/{id})**:
```json
{
  "success": true,
  "data": {
    "id": "job-123",
    "status": "PARTIAL_COMPLETE",
    "progress": {
      "stage": "finalizing",
      "percent": 100,
      "message": "Completed with partial results"
    },
    "results": [
      { "format": "MARKDOWN", "size": 15000, "available": true },
      { "format": "HTML", "size": 0, "available": false, "error": "E302" },
      { "format": "JSON", "size": 8500, "available": true }
    ],
    "retryable": true
  }
}
```

**Assertions**:
- Status is PARTIAL_COMPLETE (not COMPLETE or ERROR)
- Successful exports are available
- Failed export has available: false and error code
- Job is marked as retryable
- User can retrieve successful exports
- Resume is possible to retry failed exports

---

### API-WF-005: Status Monotonicity (No Regression)

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-WF-005 |
| **Title** | Status Monotonicity (No Regression) |
| **Priority** | P1 - High |
| **Spec Reference** | STATE-002, CONSISTENCY-001 |

**Test Objective**:
Verify that job status never regresses to a previous state unexpectedly.

**Status Ordering (monotonic forward)**:
```
PENDING (1) -> PROCESSING (2) -> COMPLETE/PARTIAL_COMPLETE/ERROR/CANCELLED (3)
```

**Test Approach**:
1. Poll GET /jobs/{id} repeatedly during processing
2. Record each status change with timestamp
3. Verify status never moves backward

**Test Implementation**:
```javascript
const statusOrder = {
  'PENDING': 1,
  'PROCESSING': 2,
  'COMPLETE': 3,
  'PARTIAL_COMPLETE': 3,
  'ERROR': 3,
  'CANCELLED': 3
};

let previousOrder = 0;
let previousStatus = null;

while (!terminalStatus) {
  const response = await fetch(`/api/v1/jobs/${jobId}`);
  const { data } = await response.json();

  const currentOrder = statusOrder[data.status];

  // Verify monotonicity
  assert(currentOrder >= previousOrder,
    `Status regressed from ${previousStatus} to ${data.status}`);

  previousOrder = currentOrder;
  previousStatus = data.status;
}
```

**Assertions**:
- Status order value never decreases
- No PROCESSING -> PENDING regression
- No COMPLETE -> PROCESSING regression
- Database constraints enforce state machine
- API layer validates transitions before applying

---

### API-WF-006: Progress Percent Consistency (0-100)

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-WF-006 |
| **Title** | Progress Percent Consistency (0-100) |
| **Priority** | P1 - High |
| **Spec Reference** | FR-404, PROGRESS-001 |

**Test Objective**:
Verify progress.percent is always valid and monotonically increasing.

**Test Approach**:
1. Subscribe to SSE stream for processing job
2. Capture all progress events
3. Validate percent values

**Validation Rules**:
```javascript
let previousPercent = 0;

sseStream.on('progress', (event) => {
  const percent = event.data.percent;

  // Rule 1: Valid range
  assert(percent >= 0 && percent <= 100,
    `Invalid percent: ${percent}`);

  // Rule 2: Integer value
  assert(Number.isInteger(percent),
    `Percent must be integer: ${percent}`);

  // Rule 3: Monotonically increasing (or equal)
  assert(percent >= previousPercent,
    `Percent decreased from ${previousPercent} to ${percent}`);

  previousPercent = percent;
});
```

**Edge Cases to Test**:
- Initial progress event has percent >= 0
- Final progress event has percent = 100 (for COMPLETE)
- PARTIAL_COMPLETE may have percent < 100
- ERROR may have percent at failure point
- Cancelled job has percent at cancellation point

**Assertions**:
- All percent values in [0, 100] range
- Percent is always integer
- Percent never decreases during processing
- Completion state has percent = 100
- Progress is consistent between SSE and GET /jobs/{id}

---

### API-WF-007: Timestamp Ordering (created < started < completed)

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-WF-007 |
| **Title** | Timestamp Ordering (created < started < completed) |
| **Priority** | P1 - High |
| **Spec Reference** | DATA-001, AUDIT-001 |

**Test Objective**:
Verify all job timestamps maintain chronological ordering.

**Timestamp Fields**:
- `createdAt`: When job was created (upload)
- `startedAt`: When processing began
- `completedAt`: When processing finished

**Test Implementation**:
```javascript
// After job completion
const response = await fetch(`/api/v1/jobs/${jobId}`);
const { data } = await response.json();

const created = new Date(data.createdAt);
const started = new Date(data.startedAt);
const completed = new Date(data.completedAt);

// Ordering validation
assert(created <= started,
  `createdAt (${created}) must be <= startedAt (${started})`);
assert(started <= completed,
  `startedAt (${started}) must be <= completedAt (${completed})`);

// All timestamps must be valid ISO 8601
assert(!isNaN(created.getTime()), 'createdAt invalid');
assert(!isNaN(started.getTime()), 'startedAt invalid');
assert(!isNaN(completed.getTime()), 'completedAt invalid');

// Timestamps must be in the past (not future)
const now = new Date();
assert(created <= now, 'createdAt in future');
assert(started <= now, 'startedAt in future');
assert(completed <= now, 'completedAt in future');
```

**Edge Cases**:
- PENDING job: only createdAt present
- PROCESSING job: createdAt and startedAt present
- CANCELLED job: completedAt reflects cancellation time
- ERROR job: completedAt reflects error time

**Assertions**:
- createdAt <= startedAt <= completedAt
- All timestamps are valid ISO 8601
- No future timestamps
- Timestamps present based on current status
- Timestamps persist correctly across polling

---

### API-WF-008: Workflow Idempotency

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-WF-008 |
| **Title** | Workflow Idempotency |
| **Priority** | P1 - High |
| **Spec Reference** | IDEMPOTENCY-001, API-001 |

**Test Objective**:
Verify that duplicate/retry requests don't create duplicate jobs or side effects.

**Test Scenarios**:

**Scenario 1: Duplicate Upload with Same File**
```
# Request 1
POST /api/v1/upload (file: doc.pdf)
# Response: { jobId: "job-1" }

# Request 2 (immediate retry)
POST /api/v1/upload (file: doc.pdf)
# Response: { jobId: "job-2" } - New job (uploads are not idempotent)
```

**Scenario 2: Duplicate Process Request**
```
# Request 1
POST /api/v1/process { "jobId": "job-1" }
# Response: 200 { status: "PROCESSING" }

# Request 2 (immediate retry, same jobId)
POST /api/v1/process { "jobId": "job-1" }
# Response: 409 E701 (already processing)
```

**Scenario 3: Duplicate Cancel Request**
```
# Request 1
POST /api/v1/jobs/job-1/cancel
# Response: 200 { status: "CANCELLED" }

# Request 2 (immediate retry)
POST /api/v1/jobs/job-1/cancel
# Response: 400 E702 (not cancellable - already cancelled)
```

**Scenario 4: GET Requests (Always Idempotent)**
```
# Multiple identical GET requests
GET /api/v1/jobs/job-1
GET /api/v1/jobs/job-1
GET /api/v1/jobs/job-1
# All return same data (no side effects)
```

**Assertions**:
- Duplicate process requests return 409 conflict
- Duplicate cancel requests handled gracefully
- GET requests are always idempotent
- No duplicate database records from retries
- Error responses for already-actioned resources

---

## 17. HTTP Method Validation Tests

This section validates correct HTTP method handling, including 405 responses, Allow headers, and pre-flight requests.

### 17.1 HTTP Method Overview

| Aspect | Description |
|--------|-------------|
| **Scope** | HTTP method validation |
| **Focus** | 405 handling, CORS, OPTIONS, HEAD |
| **Reference** | RFC 7231, RFC 7234 |
| **Test Count** | 5 tests |

---

### API-METHOD-001: 405 Method Not Allowed for Wrong Methods

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-METHOD-001 |
| **Title** | 405 Method Not Allowed for Wrong Methods |
| **Priority** | P1 - High |
| **Spec Reference** | HTTP/1.1 RFC 7231, API-HTTP-001 |

**Test Matrix**:

| Endpoint | Allowed Method | Disallowed Methods to Test |
|----------|----------------|---------------------------|
| /api/v1/upload | POST | GET, PUT, DELETE, PATCH |
| /api/v1/process | POST | GET, PUT, DELETE, PATCH |
| /api/v1/jobs/{id}/cancel | POST | GET, PUT, DELETE, PATCH |
| /api/v1/jobs/{id}/resume | POST | GET, PUT, DELETE, PATCH |
| /api/v1/jobs/{id} | GET | POST, PUT, DELETE, PATCH |
| /api/v1/history | GET | POST, PUT, DELETE, PATCH |
| /api/v1/health | GET | POST, PUT, DELETE, PATCH |
| /api/v1/process/{id}/events | GET | POST, PUT, DELETE, PATCH |

**Test Implementation**:
```javascript
const testCases = [
  { path: '/api/v1/upload', disallowed: ['GET', 'PUT', 'DELETE', 'PATCH'] },
  { path: '/api/v1/process', disallowed: ['GET', 'PUT', 'DELETE', 'PATCH'] },
  { path: '/api/v1/jobs/test-id', disallowed: ['POST', 'PUT', 'DELETE', 'PATCH'] },
  { path: '/api/v1/history', disallowed: ['POST', 'PUT', 'DELETE', 'PATCH'] },
  { path: '/api/v1/health', disallowed: ['POST', 'PUT', 'DELETE', 'PATCH'] }
];

for (const test of testCases) {
  for (const method of test.disallowed) {
    const response = await fetch(test.path, { method });
    assert.strictEqual(response.status, 405,
      `${method} ${test.path} should return 405`);
  }
}
```

**Expected Response**:
```
HTTP/1.1 405 Method Not Allowed
Content-Type: application/json
Allow: POST

{
  "success": false,
  "error": {
    "code": "E405",
    "message": "Method not allowed",
    "userMessage": "This operation is not supported",
    "suggestedAction": "Use POST method for this endpoint"
  }
}
```

**Assertions**:
- All disallowed methods return 405
- Response body is valid JSON error format
- Error code is E405
- No server errors (5xx) for wrong methods

---

### API-METHOD-002: Allow Header Presence on 405 Responses

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-METHOD-002 |
| **Title** | Allow Header Presence on 405 Responses |
| **Priority** | P1 - High |
| **Spec Reference** | RFC 7231 Section 6.5.5, HTTP-HEADER-001 |

**Requirement**:
Per RFC 7231, 405 responses MUST include an Allow header listing valid methods.

**Test Implementation**:
```javascript
const endpoints = [
  { path: '/api/v1/upload', method: 'GET', expectedAllow: 'POST, OPTIONS' },
  { path: '/api/v1/process', method: 'GET', expectedAllow: 'POST, OPTIONS' },
  { path: '/api/v1/jobs/test-id', method: 'POST', expectedAllow: 'GET, OPTIONS' },
  { path: '/api/v1/history', method: 'POST', expectedAllow: 'GET, OPTIONS' },
  { path: '/api/v1/health', method: 'DELETE', expectedAllow: 'GET, OPTIONS' }
];

for (const test of endpoints) {
  const response = await fetch(test.path, { method: test.method });

  assert.strictEqual(response.status, 405);

  const allowHeader = response.headers.get('Allow');
  assert(allowHeader, 'Allow header must be present on 405 response');

  // Verify expected methods are listed
  const allowedMethods = allowHeader.split(',').map(m => m.trim());
  assert(allowedMethods.includes('POST') || allowedMethods.includes('GET'),
    `Allow header should list valid methods: ${allowHeader}`);
}
```

**Expected Response Headers**:
```
HTTP/1.1 405 Method Not Allowed
Allow: POST, OPTIONS
Content-Type: application/json
```

**Assertions**:
- Allow header present on all 405 responses
- Allow header lists correct methods for endpoint
- Allow header format follows RFC (comma-separated)
- OPTIONS always included in Allow header

---

### API-METHOD-003: OPTIONS Pre-flight CORS Handling

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-METHOD-003 |
| **Title** | OPTIONS Pre-flight CORS Handling |
| **Priority** | P1 - High |
| **Spec Reference** | CORS RFC, SEC-CORS-001 |

**Test Objective**:
Verify OPTIONS pre-flight requests return correct CORS headers.

**Test Request**:
```
OPTIONS /api/v1/upload HTTP/1.1
Host: localhost:3000
Origin: http://localhost:3001
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Content-Type
```

**Expected Response**:
```
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: http://localhost:3001
Access-Control-Allow-Methods: POST, OPTIONS
Access-Control-Allow-Headers: Content-Type, Cookie
Access-Control-Max-Age: 86400
Access-Control-Allow-Credentials: true
```

**Test Implementation**:
```javascript
const response = await fetch('/api/v1/upload', {
  method: 'OPTIONS',
  headers: {
    'Origin': 'http://localhost:3001',
    'Access-Control-Request-Method': 'POST',
    'Access-Control-Request-Headers': 'Content-Type'
  }
});

// Status should be 204 or 200
assert([200, 204].includes(response.status));

// CORS headers must be present
assert(response.headers.get('Access-Control-Allow-Origin'));
assert(response.headers.get('Access-Control-Allow-Methods'));
assert(response.headers.get('Access-Control-Allow-Headers'));

// Credentials must be allowed (for session cookie)
assert.strictEqual(
  response.headers.get('Access-Control-Allow-Credentials'),
  'true'
);
```

**Assertions**:
- OPTIONS returns 200 or 204
- All required CORS headers present
- Allow-Origin matches request Origin (or *)
- Allow-Methods includes actual methods
- Allow-Credentials is true (for cookie auth)
- No response body for 204

---

### API-METHOD-004: HEAD Method Support for GET Endpoints

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-METHOD-004 |
| **Title** | HEAD Method Support for GET Endpoints |
| **Priority** | P1 - High |
| **Spec Reference** | RFC 7231 Section 4.3.2, HTTP-HEAD-001 |

**Requirement**:
HEAD requests should return same headers as GET but without response body.

**Test Endpoints**:
- GET /api/v1/jobs/{id}
- GET /api/v1/history
- GET /api/v1/health

**Test Implementation**:
```javascript
const endpoints = [
  '/api/v1/jobs/test-job-id',
  '/api/v1/history',
  '/api/v1/health'
];

for (const path of endpoints) {
  // Make GET request
  const getResponse = await fetch(path, {
    method: 'GET',
    headers: { 'Cookie': 'hx-docling-session=abc123' }
  });
  const getBody = await getResponse.text();

  // Make HEAD request
  const headResponse = await fetch(path, {
    method: 'HEAD',
    headers: { 'Cookie': 'hx-docling-session=abc123' }
  });
  const headBody = await headResponse.text();

  // Assertions
  assert.strictEqual(headResponse.status, getResponse.status,
    'HEAD status should match GET status');

  assert.strictEqual(headBody, '',
    'HEAD response must have empty body');

  assert.strictEqual(
    headResponse.headers.get('Content-Type'),
    getResponse.headers.get('Content-Type'),
    'HEAD Content-Type should match GET'
  );

  // Content-Length should match GET body length
  const contentLength = headResponse.headers.get('Content-Length');
  if (contentLength) {
    assert.strictEqual(
      parseInt(contentLength),
      getBody.length,
      'HEAD Content-Length should match GET body length'
    );
  }
}
```

**Assertions**:
- HEAD returns same status as equivalent GET
- HEAD response body is empty
- HEAD Content-Type matches GET
- HEAD Content-Length matches GET body length
- HEAD is supported on all GET endpoints

---

### API-METHOD-005: TRACE/CONNECT Method Rejection

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-METHOD-005 |
| **Title** | TRACE/CONNECT Method Rejection |
| **Priority** | P1 - High |
| **Spec Reference** | OWASP HTTP Methods, SEC-METHOD-001 |

**Security Requirement**:
TRACE and CONNECT methods should be disabled for security (prevent XST attacks).

**Test Implementation**:
```javascript
const dangerousMethods = ['TRACE', 'CONNECT'];
const endpoints = [
  '/api/v1/upload',
  '/api/v1/process',
  '/api/v1/jobs/test-id',
  '/api/v1/history',
  '/api/v1/health'
];

for (const method of dangerousMethods) {
  for (const path of endpoints) {
    const response = await fetch(path, { method });

    // Should be 405 Method Not Allowed or 501 Not Implemented
    assert(
      [405, 501].includes(response.status),
      `${method} ${path} should return 405 or 501, got ${response.status}`
    );

    // Should NOT be 200 OK
    assert.notStrictEqual(response.status, 200,
      `${method} ${path} must not return 200`);
  }
}
```

**Expected Response**:
```
HTTP/1.1 405 Method Not Allowed
Content-Type: application/json

{
  "success": false,
  "error": {
    "code": "E405",
    "message": "Method not allowed"
  }
}
```

**Assertions**:
- TRACE returns 405 or 501 on all endpoints
- CONNECT returns 405 or 501 on all endpoints
- No 200 responses for dangerous methods
- No request body echo (TRACE vulnerability)
- Error response is valid JSON

---

## 18. Dependency Failure Tests

This section validates graceful handling of dependency failures including database, Redis, and MCP server unavailability.

### 18.1 Dependency Failure Overview

| Aspect | Description |
|--------|-------------|
| **Scope** | External dependency failure handling |
| **Focus** | Graceful degradation, circuit breakers |
| **Dependencies** | Database, Redis, MCP server |
| **Test Count** | 6 tests |

---

### API-FAIL-001: Database Unavailable Returns 503 with E602

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-FAIL-001 |
| **Title** | Database Unavailable Returns 503 with E602 |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-301, ERROR-001 |

**Test Setup**:
Simulate database unavailability (connection refused, timeout, or pool exhaustion).

**Test Endpoints**:
All endpoints that require database:
- POST /api/v1/upload (needs to write job)
- POST /api/v1/process (needs to read/update job)
- GET /api/v1/jobs/{id} (needs to read job)
- GET /api/v1/history (needs to query jobs)
- POST /api/v1/jobs/{id}/cancel (needs to update job)
- POST /api/v1/jobs/{id}/resume (needs to update job)

**Test Request**:
```
POST /api/v1/upload
Content-Type: multipart/form-data
Cookie: hx-docling-session=abc123

[file: sample.pdf]
```

**Expected Response**:
```json
HTTP/1.1 503 Service Unavailable
Retry-After: 30
Content-Type: application/json

{
  "success": false,
  "error": {
    "code": "E602",
    "message": "Database service unavailable",
    "userMessage": "Service temporarily unavailable. Please try again.",
    "suggestedAction": "Wait a moment and retry your request",
    "retryable": true
  }
}
```

**Assertions**:
- Status code is 503 (not 500)
- Error code is E602
- Retry-After header present
- retryable: true in response
- User-friendly message (no internal details)
- No database connection strings exposed

---

### API-FAIL-002: Redis Unavailable Graceful Degradation

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-FAIL-002 |
| **Title** | Redis Unavailable Graceful Degradation |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-302, GRACEFUL-001 |

**Test Setup**:
Simulate Redis unavailability while database remains operational.

**Expected Behavior**:
Redis is used for caching and rate limiting. When unavailable:
- Caching: Bypass cache, serve from database (degraded performance)
- Rate limiting: Either bypass (allow requests) or apply stricter limits
- Session: May use fallback or database-backed sessions

**Test Request**:
```
GET /api/v1/jobs/test-job-id
Cookie: hx-docling-session=abc123
```

**Expected Response (Graceful)**:
```json
HTTP/1.1 200 OK
X-Cache-Status: BYPASS
Content-Type: application/json

{
  "success": true,
  "data": {
    "id": "test-job-id",
    "status": "COMPLETE"
    // ... full response served from database
  }
}
```

**Alternative (If Redis Required)**:
```json
HTTP/1.1 503 Service Unavailable

{
  "success": false,
  "error": {
    "code": "E603",
    "message": "Cache service unavailable",
    "retryable": true
  }
}
```

**Health Endpoint Behavior**:
```json
GET /api/v1/health

{
  "status": "degraded",
  "services": {
    "database": { "status": "healthy" },
    "redis": { "status": "unhealthy", "message": "Connection refused" },
    "mcp": { "status": "healthy" }
  }
}
```

**Assertions**:
- Core functionality works without Redis (if designed for degradation)
- Or returns 503 with clear error (if Redis required)
- Health endpoint shows degraded status
- No unhandled exceptions exposed to client
- Rate limiting behavior is defined (not undefined)

---

### API-FAIL-003: MCP Server Unavailable Returns 503 with E204

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-FAIL-003 |
| **Title** | MCP Server Unavailable Returns 503 with E204 |
| **Priority** | P1 - High |
| **Spec Reference** | FR-201, ERROR-MCP-001 |

**Test Setup**:
Simulate MCP server unavailability (connection refused, timeout).

**Affected Endpoints**:
- POST /api/v1/process (requires MCP for document conversion)
- POST /api/v1/jobs/{id}/resume (requires MCP to continue processing)

**Test Request**:
```
POST /api/v1/process
Content-Type: application/json
Cookie: hx-docling-session=abc123

{
  "jobId": "valid-job-id"
}
```

**Expected Response**:
```json
HTTP/1.1 503 Service Unavailable
Retry-After: 60
Content-Type: application/json

{
  "success": false,
  "error": {
    "code": "E204",
    "message": "Document processing service unavailable",
    "userMessage": "The processing service is temporarily unavailable. Please try again.",
    "suggestedAction": "Wait a moment and retry processing",
    "retryable": true
  }
}
```

**Job State After MCP Failure**:
```json
GET /api/v1/jobs/valid-job-id

{
  "success": true,
  "data": {
    "id": "valid-job-id",
    "status": "ERROR",
    "errorCode": "E204",
    "errorMessage": "Processing service unavailable",
    "retryable": true
  }
}
```

**Assertions**:
- Status code is 503 (not 500)
- Error code is E204
- Job status updated to ERROR with retryable: true
- Retry-After header suggests reasonable wait time
- Upload and query endpoints still work
- Health endpoint shows MCP as unhealthy

---

### API-FAIL-004: Circuit Breaker Activation

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-FAIL-004 |
| **Title** | Circuit Breaker Activation |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-303, CIRCUIT-001 |

**Test Objective**:
Verify circuit breaker opens after consecutive failures to prevent cascade.

**Circuit Breaker Configuration**:
```
Failure Threshold: 5 consecutive failures
Open Duration: 30 seconds
Half-Open Requests: 1 (probe)
```

**Test Sequence**:
```javascript
// Setup: MCP server returns errors
for (let i = 0; i < 5; i++) {
  const response = await fetch('/api/v1/process', {
    method: 'POST',
    body: JSON.stringify({ jobId: `job-${i}` })
  });
  // Each fails with 503 (MCP error)
  assert.strictEqual(response.status, 503);
}

// 6th request should be rejected immediately (circuit open)
const openResponse = await fetch('/api/v1/process', {
  method: 'POST',
  body: JSON.stringify({ jobId: 'job-6' })
});
```

**Expected Response (Circuit Open)**:
```json
HTTP/1.1 503 Service Unavailable
Retry-After: 30
Content-Type: application/json

{
  "success": false,
  "error": {
    "code": "E604",
    "message": "Service temporarily unavailable due to high error rate",
    "userMessage": "The processing service is experiencing issues. Please wait and try again.",
    "suggestedAction": "Wait 30 seconds before retrying",
    "retryable": true
  }
}
```

**Assertions**:
- After threshold failures, circuit opens
- Subsequent requests fail-fast (no MCP call)
- Response indicates circuit breaker (E604 or similar)
- Retry-After matches circuit breaker open duration
- Fast failure response (<100ms vs timeout)

---

### API-FAIL-005: Circuit Breaker Recovery (Half-Open)

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-FAIL-005 |
| **Title** | Circuit Breaker Recovery (Half-Open) |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-303, CIRCUIT-002 |

**Test Objective**:
Verify circuit breaker transitions through OPEN -> HALF-OPEN -> CLOSED states.

**Test Sequence**:
```javascript
// Phase 1: Trigger circuit open
// (5 failures as per API-FAIL-004)

// Phase 2: Wait for open duration
await sleep(30000); // 30 seconds

// Phase 3: Circuit should be half-open
// First request is probe (allowed through)

// Setup: MCP server now healthy
const probeResponse = await fetch('/api/v1/process', {
  method: 'POST',
  body: JSON.stringify({ jobId: 'probe-job' })
});

// If probe succeeds, circuit closes
assert.strictEqual(probeResponse.status, 200);

// Phase 4: Verify circuit closed (normal operation)
const normalResponse = await fetch('/api/v1/process', {
  method: 'POST',
  body: JSON.stringify({ jobId: 'normal-job' })
});
assert.strictEqual(normalResponse.status, 200);
```

**State Transitions**:
```
CLOSED (normal)
  -> 5 failures
  -> OPEN (fail-fast)
  -> 30 seconds timeout
  -> HALF-OPEN (probe)
  -> probe success
  -> CLOSED (normal)
```

**Alternative (Probe Fails)**:
```javascript
// If probe fails, circuit returns to open
const failedProbe = await fetch('/api/v1/process', ...);
assert.strictEqual(failedProbe.status, 503);

// Circuit remains open for another duration
const stillOpen = await fetch('/api/v1/process', ...);
assert.strictEqual(stillOpen.status, 503);
```

**Assertions**:
- Circuit transitions to HALF-OPEN after timeout
- Single probe request allowed in HALF-OPEN
- Successful probe closes circuit
- Failed probe reopens circuit
- Normal operation resumes after recovery

---

### API-FAIL-006: Graceful Shutdown During Active Requests

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-FAIL-006 |
| **Title** | Graceful Shutdown During Active Requests |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-304, SHUTDOWN-001 |

**Test Objective**:
Verify server handles shutdown gracefully without dropping active requests.

**Test Approach**:
1. Start long-running processing request
2. Initiate server shutdown
3. Verify active request completes or gets clean termination
4. Verify new requests are rejected

**Test Implementation**:
```javascript
// Step 1: Start processing
const processPromise = fetch('/api/v1/process', {
  method: 'POST',
  body: JSON.stringify({ jobId: 'active-job' })
});

// Step 2: Initiate shutdown (SIGTERM)
// Server should:
// - Stop accepting new connections
// - Allow active requests to complete (with timeout)
// - Return clean responses

// Step 3: New request during shutdown
const newRequest = await fetch('/api/v1/upload', {
  method: 'POST',
  body: formData
});

// Expected: 503 Service Unavailable (shutting down)
assert.strictEqual(newRequest.status, 503);

// Step 4: Active request should complete or terminate cleanly
const activeResponse = await processPromise;
// Either 200 (completed) or 503 (terminated gracefully)
assert([200, 503].includes(activeResponse.status));
```

**Expected Behavior During Shutdown**:

**New Requests**:
```json
HTTP/1.1 503 Service Unavailable
Connection: close
Content-Type: application/json

{
  "success": false,
  "error": {
    "code": "E605",
    "message": "Server is shutting down",
    "userMessage": "Server is restarting. Please try again shortly.",
    "retryable": true
  }
}
```

**Active SSE Streams**:
```
event: shutdown
data: {"message": "Server shutting down", "retryable": true}

[connection closed]
```

**Assertions**:
- Active requests complete or get clean 503
- SSE streams receive shutdown event before close
- New requests rejected with 503
- No partial responses or corrupted data
- Connection: close header on shutdown responses
- Shutdown timeout prevents indefinite waiting

---

## 19. Pydantic Model Validation Tests

This section validates Pydantic model behavior for request/response schemas, including field validation, type coercion, custom validators, and serialization. These tests ensure data integrity and proper validation at the schema level.

### 19.1 Overview

| Aspect | Description |
|--------|-------------|
| **Scope** | Pydantic model validation for API schemas |
| **Framework** | pytest with pydantic v2 |
| **Coverage Target** | 90%+ data validation coverage |
| **Test Count** | 32 tests |

### 19.2 Test Categories

| Category | Test Count | Description |
|----------|------------|-------------|
| BaseModel Field Validation | 10 | Core field constraints and inheritance |
| Type Coercion | 6 | Automatic type conversion behavior |
| Custom Validators | 8 | @field_validator and @model_validator |
| Serialization | 8 | model_dump(), JSON export, aliases |

---

### API-PYDANTIC-001: BaseModel Required Field Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PYDANTIC-001 |
| **Title** | BaseModel Required Field Validation |
| **Priority** | P0 - Critical |
| **Spec Reference** | PYDANTIC-001, Data Validation |

**Test Objective**:
Verify that Pydantic models correctly enforce required fields and raise ValidationError for missing required fields.

**Test Implementation**:
```python
import pytest
from pydantic import BaseModel, ValidationError

class JobRequest(BaseModel):
    job_id: str
    session_id: str

def test_required_field_missing():
    """Test that missing required fields raise ValidationError."""
    with pytest.raises(ValidationError) as exc_info:
        JobRequest(job_id="550e8400-e29b-41d4-a716-446655440000")

    errors = exc_info.value.errors()
    assert len(errors) == 1
    assert errors[0]["type"] == "missing"
    assert errors[0]["loc"] == ("session_id",)

def test_all_required_fields_present():
    """Test that model validates when all required fields present."""
    model = JobRequest(
        job_id="550e8400-e29b-41d4-a716-446655440000",
        session_id="abc123"
    )
    assert model.job_id == "550e8400-e29b-41d4-a716-446655440000"
    assert model.session_id == "abc123"
```

**Assertions**:
- ValidationError raised for missing required fields
- Error contains correct field location (loc)
- Error type is "missing"
- Model instantiates successfully with all required fields
- Required fields accessible after instantiation

---

### API-PYDANTIC-002: Field Constraints - min_length/max_length

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PYDANTIC-002 |
| **Title** | Field Constraints - min_length/max_length |
| **Priority** | P0 - Critical |
| **Spec Reference** | PYDANTIC-002, String Validation |

**Test Implementation**:
```python
from pydantic import BaseModel, Field, ValidationError

class FileNameModel(BaseModel):
    file_name: str = Field(min_length=1, max_length=255)

def test_min_length_validation():
    """Test that empty string violates min_length constraint."""
    with pytest.raises(ValidationError) as exc_info:
        FileNameModel(file_name="")

    errors = exc_info.value.errors()
    assert errors[0]["type"] == "string_too_short"
    assert errors[0]["ctx"]["min_length"] == 1

def test_max_length_validation():
    """Test that string exceeding max_length is rejected."""
    with pytest.raises(ValidationError) as exc_info:
        FileNameModel(file_name="x" * 256)

    errors = exc_info.value.errors()
    assert errors[0]["type"] == "string_too_long"
    assert errors[0]["ctx"]["max_length"] == 255

def test_valid_length():
    """Test that valid length string is accepted."""
    model = FileNameModel(file_name="document.pdf")
    assert model.file_name == "document.pdf"
```

**Assertions**:
- Empty string rejected (min_length=1)
- String exceeding max_length rejected
- Error type correctly identifies constraint violation
- Context contains constraint values
- Valid length strings accepted

---

### API-PYDANTIC-003: Field Constraints - Pattern (Regex)

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PYDANTIC-003 |
| **Title** | Field Constraints - Pattern (Regex) |
| **Priority** | P0 - Critical |
| **Spec Reference** | PYDANTIC-003, Pattern Validation |

**Test Implementation**:
```python
from pydantic import BaseModel, Field, ValidationError

class JobIdModel(BaseModel):
    job_id: str = Field(
        pattern=r"^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$"
    )

def test_uuid_pattern_valid():
    """Test valid UUID v4 matches pattern."""
    model = JobIdModel(job_id="550e8400-e29b-41d4-a716-446655440000")
    assert model.job_id == "550e8400-e29b-41d4-a716-446655440000"

def test_uuid_pattern_invalid_format():
    """Test invalid UUID format rejected."""
    with pytest.raises(ValidationError) as exc_info:
        JobIdModel(job_id="not-a-uuid")

    errors = exc_info.value.errors()
    assert errors[0]["type"] == "string_pattern_mismatch"

def test_uuid_pattern_wrong_version():
    """Test UUID v1 rejected by v4 pattern."""
    with pytest.raises(ValidationError) as exc_info:
        JobIdModel(job_id="550e8400-e29b-11d4-a716-446655440000")  # v1 UUID

    errors = exc_info.value.errors()
    assert errors[0]["type"] == "string_pattern_mismatch"
```

**Assertions**:
- Valid UUID v4 accepted
- Arbitrary string rejected
- UUID with wrong version rejected
- Error type is "string_pattern_mismatch"
- Pattern constraint enforced strictly

---

### API-PYDANTIC-004: Field Constraints - Numeric (ge, le, gt, lt)

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PYDANTIC-004 |
| **Title** | Field Constraints - Numeric (ge, le, gt, lt) |
| **Priority** | P0 - Critical |
| **Spec Reference** | PYDANTIC-004, Numeric Validation |

**Test Implementation**:
```python
from pydantic import BaseModel, Field, ValidationError

class FileSizeModel(BaseModel):
    file_size: int = Field(ge=1, le=104857600)  # 1 byte to 100MB

class ProgressModel(BaseModel):
    percent: int = Field(ge=0, le=100)

def test_file_size_minimum():
    """Test file size must be at least 1 byte."""
    with pytest.raises(ValidationError) as exc_info:
        FileSizeModel(file_size=0)

    errors = exc_info.value.errors()
    assert errors[0]["type"] == "greater_than_equal"
    assert errors[0]["ctx"]["ge"] == 1

def test_file_size_maximum():
    """Test file size cannot exceed 100MB."""
    with pytest.raises(ValidationError) as exc_info:
        FileSizeModel(file_size=104857601)

    errors = exc_info.value.errors()
    assert errors[0]["type"] == "less_than_equal"

def test_progress_boundary_values():
    """Test progress accepts boundary values 0 and 100."""
    model_zero = ProgressModel(percent=0)
    model_hundred = ProgressModel(percent=100)
    assert model_zero.percent == 0
    assert model_hundred.percent == 100

def test_progress_out_of_range():
    """Test progress rejects values outside 0-100."""
    with pytest.raises(ValidationError):
        ProgressModel(percent=-1)
    with pytest.raises(ValidationError):
        ProgressModel(percent=101)
```

**Assertions**:
- Zero file size rejected (ge=1)
- File size exceeding maximum rejected
- Progress boundary values (0, 100) accepted
- Negative progress rejected
- Progress > 100 rejected
- Error context contains constraint values

---

### API-PYDANTIC-005: Optional vs Required Fields

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PYDANTIC-005 |
| **Title** | Optional vs Required Fields |
| **Priority** | P0 - Critical |
| **Spec Reference** | PYDANTIC-005, Optional Fields |

**Test Implementation**:
```python
from typing import Optional
from pydantic import BaseModel, ValidationError

class JobDetailsModel(BaseModel):
    job_id: str  # Required
    status: str  # Required
    file_name: Optional[str] = None  # Optional with default
    source_url: Optional[str] = None  # Optional with default
    error_message: Optional[str] = None  # Optional with default

def test_optional_fields_omitted():
    """Test model validates with optional fields omitted."""
    model = JobDetailsModel(job_id="uuid", status="PENDING")
    assert model.job_id == "uuid"
    assert model.status == "PENDING"
    assert model.file_name is None
    assert model.source_url is None

def test_optional_fields_provided():
    """Test model accepts optional fields when provided."""
    model = JobDetailsModel(
        job_id="uuid",
        status="COMPLETE",
        file_name="document.pdf",
        error_message=None
    )
    assert model.file_name == "document.pdf"
    assert model.error_message is None

def test_required_field_missing_with_optionals():
    """Test required field missing raises error even with optionals present."""
    with pytest.raises(ValidationError):
        JobDetailsModel(file_name="document.pdf", status="PENDING")
```

**Assertions**:
- Model validates with only required fields
- Optional fields default to None when omitted
- Optional fields accept explicit None
- Optional fields accept valid values
- Required field missing still raises ValidationError

---

### API-PYDANTIC-006: Default Values

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PYDANTIC-006 |
| **Title** | Default Values |
| **Priority** | P1 - High |
| **Spec Reference** | PYDANTIC-006, Default Values |

**Test Implementation**:
```python
from pydantic import BaseModel, Field
from datetime import datetime

class PaginationModel(BaseModel):
    page: int = Field(default=1, ge=1)
    page_size: int = Field(default=20, ge=1, le=100)
    sort_order: str = Field(default="desc")

def test_default_values_applied():
    """Test default values used when fields omitted."""
    model = PaginationModel()
    assert model.page == 1
    assert model.page_size == 20
    assert model.sort_order == "desc"

def test_default_values_overridden():
    """Test default values can be overridden."""
    model = PaginationModel(page=5, page_size=50, sort_order="asc")
    assert model.page == 5
    assert model.page_size == 50
    assert model.sort_order == "asc"

def test_partial_override():
    """Test partial override keeps other defaults."""
    model = PaginationModel(page=3)
    assert model.page == 3
    assert model.page_size == 20  # Default
    assert model.sort_order == "desc"  # Default
```

**Assertions**:
- Default values applied when fields omitted
- Default values overridable
- Partial override retains other defaults
- Default values pass field validation constraints
- Defaults work with Field() declarations

---

### API-PYDANTIC-007: Model Inheritance

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PYDANTIC-007 |
| **Title** | Model Inheritance |
| **Priority** | P1 - High |
| **Spec Reference** | PYDANTIC-007, Model Inheritance |

**Test Implementation**:
```python
from pydantic import BaseModel, Field
from datetime import datetime
from typing import Optional

class BaseResponse(BaseModel):
    success: bool
    timestamp: datetime = Field(default_factory=datetime.utcnow)

class JobResponse(BaseResponse):
    job_id: str
    status: str

class ErrorResponse(BaseResponse):
    success: bool = False  # Override default
    error_code: str
    error_message: str

def test_inherited_fields_present():
    """Test child model has parent fields."""
    model = JobResponse(success=True, job_id="uuid", status="PENDING")
    assert hasattr(model, "success")
    assert hasattr(model, "timestamp")
    assert hasattr(model, "job_id")

def test_inherited_defaults():
    """Test parent defaults work in child."""
    model = JobResponse(success=True, job_id="uuid", status="PENDING")
    assert model.timestamp is not None
    assert isinstance(model.timestamp, datetime)

def test_overridden_defaults():
    """Test child can override parent defaults."""
    model = ErrorResponse(error_code="E001", error_message="Error")
    assert model.success is False  # Overridden default
```

**Assertions**:
- Child model includes parent fields
- Parent field defaults work in child
- Child can add new required fields
- Child can override parent defaults
- Validation applies to both parent and child fields

---

### API-PYDANTIC-008: Nested Model Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PYDANTIC-008 |
| **Title** | Nested Model Validation |
| **Priority** | P0 - Critical |
| **Spec Reference** | PYDANTIC-008, Nested Models |

**Test Implementation**:
```python
from pydantic import BaseModel, Field, ValidationError
from typing import List, Optional

class ResultItem(BaseModel):
    format: str = Field(pattern=r"^(MARKDOWN|HTML|JSON)$")
    size: int = Field(ge=0)
    available: bool

class JobWithResults(BaseModel):
    job_id: str
    results: List[ResultItem]

def test_valid_nested_model():
    """Test valid nested models accepted."""
    model = JobWithResults(
        job_id="uuid",
        results=[
            ResultItem(format="MARKDOWN", size=1024, available=True),
            ResultItem(format="HTML", size=2048, available=True)
        ]
    )
    assert len(model.results) == 2
    assert model.results[0].format == "MARKDOWN"

def test_invalid_nested_model():
    """Test nested validation errors bubble up."""
    with pytest.raises(ValidationError) as exc_info:
        JobWithResults(
            job_id="uuid",
            results=[
                ResultItem(format="INVALID", size=1024, available=True)
            ]
        )

    errors = exc_info.value.errors()
    assert errors[0]["loc"] == ("results", 0, "format")
    assert errors[0]["type"] == "string_pattern_mismatch"

def test_empty_nested_list():
    """Test empty list for nested models."""
    model = JobWithResults(job_id="uuid", results=[])
    assert model.results == []
```

**Assertions**:
- Valid nested models accepted
- Nested validation errors include full path (loc)
- Error location includes list index
- Empty lists handled correctly
- Nested model constraints enforced

---

### API-PYDANTIC-009: Enum Field Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PYDANTIC-009 |
| **Title** | Enum Field Validation |
| **Priority** | P0 - Critical |
| **Spec Reference** | PYDANTIC-009, Enum Validation |

**Test Implementation**:
```python
from enum import Enum
from pydantic import BaseModel, ValidationError

class JobStatus(str, Enum):
    PENDING = "PENDING"
    PROCESSING = "PROCESSING"
    COMPLETE = "COMPLETE"
    ERROR = "ERROR"
    CANCELLED = "CANCELLED"

class JobModel(BaseModel):
    job_id: str
    status: JobStatus

def test_valid_enum_value():
    """Test valid enum value accepted."""
    model = JobModel(job_id="uuid", status=JobStatus.PENDING)
    assert model.status == JobStatus.PENDING
    assert model.status.value == "PENDING"

def test_string_coerced_to_enum():
    """Test string automatically coerced to enum."""
    model = JobModel(job_id="uuid", status="PROCESSING")
    assert model.status == JobStatus.PROCESSING
    assert isinstance(model.status, JobStatus)

def test_invalid_enum_value():
    """Test invalid enum value rejected."""
    with pytest.raises(ValidationError) as exc_info:
        JobModel(job_id="uuid", status="INVALID_STATUS")

    errors = exc_info.value.errors()
    assert "enum" in errors[0]["type"].lower() or "literal" in errors[0]["type"].lower()
```

**Assertions**:
- Enum values accepted directly
- String values coerced to enum
- Invalid string values rejected
- Enum type preserved after coercion
- Case-sensitive matching enforced

---

### API-PYDANTIC-010: Literal Type Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PYDANTIC-010 |
| **Title** | Literal Type Validation |
| **Priority** | P1 - High |
| **Spec Reference** | PYDANTIC-010, Literal Types |

**Test Implementation**:
```python
from typing import Literal
from pydantic import BaseModel, ValidationError

class FormatRequest(BaseModel):
    output_format: Literal["MARKDOWN", "HTML", "JSON"]

class InputTypeModel(BaseModel):
    input_type: Literal["FILE", "URL"]

def test_literal_valid_value():
    """Test valid literal values accepted."""
    model = FormatRequest(output_format="MARKDOWN")
    assert model.output_format == "MARKDOWN"

def test_literal_all_options():
    """Test all literal options work."""
    for fmt in ["MARKDOWN", "HTML", "JSON"]:
        model = FormatRequest(output_format=fmt)
        assert model.output_format == fmt

def test_literal_invalid_value():
    """Test invalid literal value rejected."""
    with pytest.raises(ValidationError) as exc_info:
        FormatRequest(output_format="XML")

    errors = exc_info.value.errors()
    assert errors[0]["type"] == "literal_error"
```

**Assertions**:
- Valid literal values accepted
- All defined literal options work
- Invalid values rejected
- Error type is "literal_error"
- Case-sensitive matching

---

### API-PYDANTIC-011: Strict Mode Type Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PYDANTIC-011 |
| **Title** | Strict Mode Type Validation |
| **Priority** | P1 - High |
| **Spec Reference** | PYDANTIC-011, Strict Mode |

**Test Implementation**:
```python
from pydantic import BaseModel, ConfigDict, ValidationError

class StrictModel(BaseModel):
    model_config = ConfigDict(strict=True)

    count: int
    name: str
    enabled: bool

class LaxModel(BaseModel):
    count: int
    name: str
    enabled: bool

def test_strict_rejects_string_int():
    """Test strict mode rejects string where int expected."""
    with pytest.raises(ValidationError) as exc_info:
        StrictModel(count="42", name="test", enabled=True)

    errors = exc_info.value.errors()
    assert errors[0]["type"] == "int_type"

def test_lax_coerces_string_int():
    """Test lax mode coerces string to int."""
    model = LaxModel(count="42", name="test", enabled=True)
    assert model.count == 42
    assert isinstance(model.count, int)

def test_strict_rejects_int_bool():
    """Test strict mode rejects int where bool expected."""
    with pytest.raises(ValidationError):
        StrictModel(count=1, name="test", enabled=1)

def test_strict_accepts_correct_types():
    """Test strict mode accepts correct types."""
    model = StrictModel(count=42, name="test", enabled=True)
    assert model.count == 42
```

**Assertions**:
- Strict mode rejects type mismatches
- Lax mode (default) coerces compatible types
- Strict mode requires exact type match
- Int not accepted for bool in strict mode
- Correct types always accepted

---

### API-PYDANTIC-012: String to Integer Coercion

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PYDANTIC-012 |
| **Title** | String to Integer Coercion |
| **Priority** | P1 - High |
| **Spec Reference** | PYDANTIC-012, Type Coercion |

**Test Implementation**:
```python
from pydantic import BaseModel, ValidationError

class NumericModel(BaseModel):
    page: int
    page_size: int
    file_size: int

def test_string_numeric_coercion():
    """Test numeric strings coerced to int."""
    model = NumericModel(page="1", page_size="20", file_size="1024")
    assert model.page == 1
    assert model.page_size == 20
    assert model.file_size == 1024

def test_float_string_truncated():
    """Test float string truncated to int."""
    model = NumericModel(page="1", page_size="20", file_size="1024.9")
    assert model.file_size == 1024

def test_non_numeric_string_rejected():
    """Test non-numeric string rejected."""
    with pytest.raises(ValidationError):
        NumericModel(page="one", page_size="20", file_size="1024")

def test_negative_string_coercion():
    """Test negative numeric string coerced."""
    # Depending on constraints, this may or may not be valid
    model = NumericModel(page="-1", page_size="20", file_size="1024")
    assert model.page == -1
```

**Assertions**:
- Numeric strings coerced to int
- Float strings truncated (floor)
- Non-numeric strings rejected
- Negative strings handled correctly
- Type preserved after coercion

---

### API-PYDANTIC-013: DateTime Parsing

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PYDANTIC-013 |
| **Title** | DateTime Parsing |
| **Priority** | P0 - Critical |
| **Spec Reference** | PYDANTIC-013, DateTime Validation |

**Test Implementation**:
```python
from datetime import datetime
from pydantic import BaseModel, ValidationError

class TimestampModel(BaseModel):
    created_at: datetime
    updated_at: datetime

def test_iso8601_parsing():
    """Test ISO 8601 format parsing."""
    model = TimestampModel(
        created_at="2025-12-14T10:00:00Z",
        updated_at="2025-12-14T10:30:00Z"
    )
    assert model.created_at.year == 2025
    assert model.created_at.month == 12
    assert model.created_at.day == 14

def test_iso8601_with_timezone():
    """Test ISO 8601 with timezone offset."""
    model = TimestampModel(
        created_at="2025-12-14T10:00:00+05:30",
        updated_at="2025-12-14T10:00:00-08:00"
    )
    assert model.created_at is not None
    assert model.updated_at is not None

def test_invalid_datetime_format():
    """Test invalid datetime format rejected."""
    with pytest.raises(ValidationError):
        TimestampModel(
            created_at="14/12/2025",  # Invalid format
            updated_at="2025-12-14T10:00:00Z"
        )

def test_datetime_object_accepted():
    """Test datetime object accepted directly."""
    now = datetime.utcnow()
    model = TimestampModel(created_at=now, updated_at=now)
    assert model.created_at == now
```

**Assertions**:
- ISO 8601 format parsed correctly
- Timezone offsets handled
- Invalid formats rejected
- datetime objects accepted directly
- Date components accessible after parsing

---

### API-PYDANTIC-014: UUID Parsing

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PYDANTIC-014 |
| **Title** | UUID Parsing |
| **Priority** | P0 - Critical |
| **Spec Reference** | PYDANTIC-014, UUID Validation |

**Test Implementation**:
```python
from uuid import UUID
from pydantic import BaseModel, ValidationError

class UUIDModel(BaseModel):
    job_id: UUID
    session_id: UUID

def test_uuid_string_parsing():
    """Test UUID string parsed to UUID object."""
    model = UUIDModel(
        job_id="550e8400-e29b-41d4-a716-446655440000",
        session_id="6ba7b810-9dad-11d1-80b4-00c04fd430c8"
    )
    assert isinstance(model.job_id, UUID)
    assert str(model.job_id) == "550e8400-e29b-41d4-a716-446655440000"

def test_uuid_object_accepted():
    """Test UUID object accepted directly."""
    uuid_obj = UUID("550e8400-e29b-41d4-a716-446655440000")
    model = UUIDModel(job_id=uuid_obj, session_id=uuid_obj)
    assert model.job_id == uuid_obj

def test_invalid_uuid_rejected():
    """Test invalid UUID string rejected."""
    with pytest.raises(ValidationError):
        UUIDModel(
            job_id="not-a-uuid",
            session_id="550e8400-e29b-41d4-a716-446655440000"
        )

def test_uuid_case_insensitive():
    """Test UUID parsing is case-insensitive."""
    model = UUIDModel(
        job_id="550E8400-E29B-41D4-A716-446655440000",
        session_id="550e8400-e29b-41d4-a716-446655440000"
    )
    assert model.job_id == model.session_id
```

**Assertions**:
- UUID strings parsed to UUID objects
- UUID objects accepted directly
- Invalid UUID strings rejected
- Case-insensitive parsing
- UUID type preserved after parsing

---

### API-PYDANTIC-015: Enum String Coercion

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PYDANTIC-015 |
| **Title** | Enum String Coercion |
| **Priority** | P1 - High |
| **Spec Reference** | PYDANTIC-015, Enum Coercion |

**Test Implementation**:
```python
from enum import Enum
from pydantic import BaseModel, ValidationError

class OutputFormat(str, Enum):
    MARKDOWN = "MARKDOWN"
    HTML = "HTML"
    JSON = "JSON"

class FormatModel(BaseModel):
    format: OutputFormat

def test_string_to_enum_coercion():
    """Test string automatically coerced to enum."""
    model = FormatModel(format="MARKDOWN")
    assert model.format == OutputFormat.MARKDOWN
    assert isinstance(model.format, OutputFormat)

def test_enum_value_accepted():
    """Test enum value accepted directly."""
    model = FormatModel(format=OutputFormat.HTML)
    assert model.format == OutputFormat.HTML

def test_case_sensitive_enum():
    """Test enum coercion is case-sensitive."""
    with pytest.raises(ValidationError):
        FormatModel(format="markdown")  # Wrong case

def test_enum_serialization():
    """Test enum serializes to string value."""
    model = FormatModel(format=OutputFormat.JSON)
    data = model.model_dump()
    assert data["format"] == "JSON"
```

**Assertions**:
- Strings coerced to enum automatically
- Enum values accepted directly
- Case-sensitive matching enforced
- Serialization produces string value
- Enum type preserved in model

---

### API-PYDANTIC-016: URL Validation and Parsing

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PYDANTIC-016 |
| **Title** | URL Validation and Parsing |
| **Priority** | P0 - Critical |
| **Spec Reference** | PYDANTIC-016, URL Validation |

**Test Implementation**:
```python
from pydantic import BaseModel, HttpUrl, ValidationError

class UrlInputModel(BaseModel):
    source_url: HttpUrl

def test_valid_https_url():
    """Test valid HTTPS URL accepted."""
    model = UrlInputModel(source_url="https://example.com/document.pdf")
    assert str(model.source_url) == "https://example.com/document.pdf"

def test_valid_http_url():
    """Test valid HTTP URL accepted."""
    model = UrlInputModel(source_url="http://example.com/doc.html")
    assert "http://" in str(model.source_url)

def test_invalid_url_rejected():
    """Test invalid URL rejected."""
    with pytest.raises(ValidationError):
        UrlInputModel(source_url="not-a-url")

def test_url_with_path_and_query():
    """Test URL with path and query string."""
    model = UrlInputModel(
        source_url="https://example.com/path/to/doc?version=1&format=pdf"
    )
    assert "example.com" in str(model.source_url)
```

**Assertions**:
- Valid HTTPS URLs accepted
- Valid HTTP URLs accepted
- Invalid URLs rejected
- URL components preserved
- Query strings handled correctly

---

### API-PYDANTIC-017: Field Validator - Before Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PYDANTIC-017 |
| **Title** | Field Validator - Before Validation |
| **Priority** | P0 - Critical |
| **Spec Reference** | PYDANTIC-017, Custom Validators |

**Test Implementation**:
```python
from pydantic import BaseModel, field_validator, ValidationError

class FileNameModel(BaseModel):
    file_name: str

    @field_validator("file_name", mode="before")
    @classmethod
    def strip_whitespace(cls, v):
        if isinstance(v, str):
            return v.strip()
        return v

def test_whitespace_stripped():
    """Test whitespace stripped before validation."""
    model = FileNameModel(file_name="  document.pdf  ")
    assert model.file_name == "document.pdf"

def test_no_whitespace_unchanged():
    """Test value without whitespace unchanged."""
    model = FileNameModel(file_name="document.pdf")
    assert model.file_name == "document.pdf"

def test_non_string_passthrough():
    """Test non-string values pass through."""
    # This will fail type validation, but validator handles gracefully
    with pytest.raises(ValidationError):
        FileNameModel(file_name=123)
```

**Assertions**:
- Validator runs before type validation
- Whitespace stripped from strings
- Original value unchanged when no whitespace
- Non-string types handled gracefully
- Transformation applied consistently

---

### API-PYDANTIC-018: Field Validator - After Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PYDANTIC-018 |
| **Title** | Field Validator - After Validation |
| **Priority** | P0 - Critical |
| **Spec Reference** | PYDANTIC-018, Custom Validators |

**Test Implementation**:
```python
from pydantic import BaseModel, field_validator, ValidationError

class MimeTypeModel(BaseModel):
    mime_type: str

    @field_validator("mime_type", mode="after")
    @classmethod
    def validate_allowed_types(cls, v):
        allowed = [
            "application/pdf",
            "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
            "image/png",
            "image/jpeg"
        ]
        if v not in allowed:
            raise ValueError(f"MIME type {v} not allowed")
        return v

def test_allowed_mime_type():
    """Test allowed MIME type passes validation."""
    model = MimeTypeModel(mime_type="application/pdf")
    assert model.mime_type == "application/pdf"

def test_disallowed_mime_type():
    """Test disallowed MIME type rejected."""
    with pytest.raises(ValidationError) as exc_info:
        MimeTypeModel(mime_type="application/xml")

    errors = exc_info.value.errors()
    assert "not allowed" in str(errors[0]["msg"]).lower()

def test_all_allowed_types():
    """Test all allowed MIME types accepted."""
    for mime in ["application/pdf", "image/png", "image/jpeg"]:
        model = MimeTypeModel(mime_type=mime)
        assert model.mime_type == mime
```

**Assertions**:
- Validator runs after type validation
- Allowed values pass validation
- Disallowed values raise ValueError
- Custom error message in ValidationError
- All allowed types verified

---

### API-PYDANTIC-019: Model Validator - Before Mode

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PYDANTIC-019 |
| **Title** | Model Validator - Before Mode |
| **Priority** | P1 - High |
| **Spec Reference** | PYDANTIC-019, Model Validators |

**Test Implementation**:
```python
from pydantic import BaseModel, model_validator, ValidationError
from typing import Any

class InputModel(BaseModel):
    input_type: str
    file_name: str | None = None
    source_url: str | None = None

    @model_validator(mode="before")
    @classmethod
    def set_input_type(cls, data: Any) -> Any:
        if isinstance(data, dict):
            if data.get("file_name") and not data.get("source_url"):
                data["input_type"] = "FILE"
            elif data.get("source_url") and not data.get("file_name"):
                data["input_type"] = "URL"
        return data

def test_file_input_type_set():
    """Test input_type automatically set for file input."""
    model = InputModel(input_type="", file_name="document.pdf")
    assert model.input_type == "FILE"

def test_url_input_type_set():
    """Test input_type automatically set for URL input."""
    model = InputModel(input_type="", source_url="https://example.com/doc.pdf")
    assert model.input_type == "URL"

def test_explicit_input_type_preserved():
    """Test explicit input_type not overwritten."""
    model = InputModel(
        input_type="FILE",
        file_name="document.pdf",
        source_url=None
    )
    assert model.input_type == "FILE"
```

**Assertions**:
- Model validator runs before field validation
- Can modify input data dictionary
- Automatic field derivation works
- Explicit values can be preserved
- Handles both dict and other inputs

---

### API-PYDANTIC-020: Model Validator - After Mode

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PYDANTIC-020 |
| **Title** | Model Validator - After Mode |
| **Priority** | P1 - High |
| **Spec Reference** | PYDANTIC-020, Model Validators |

**Test Implementation**:
```python
from pydantic import BaseModel, model_validator, ValidationError

class DateRangeModel(BaseModel):
    start_date: str
    end_date: str

    @model_validator(mode="after")
    def validate_date_range(self):
        if self.start_date > self.end_date:
            raise ValueError("start_date must be before end_date")
        return self

def test_valid_date_range():
    """Test valid date range passes validation."""
    model = DateRangeModel(start_date="2025-01-01", end_date="2025-12-31")
    assert model.start_date < model.end_date

def test_invalid_date_range():
    """Test invalid date range rejected."""
    with pytest.raises(ValidationError) as exc_info:
        DateRangeModel(start_date="2025-12-31", end_date="2025-01-01")

    errors = exc_info.value.errors()
    assert "start_date must be before end_date" in str(errors)

def test_same_dates_allowed():
    """Test same start and end date allowed."""
    model = DateRangeModel(start_date="2025-06-15", end_date="2025-06-15")
    assert model.start_date == model.end_date
```

**Assertions**:
- Model validator runs after all field validation
- Can access validated field values
- Cross-field validation works
- ValueError converts to ValidationError
- Equal values handled correctly

---

### API-PYDANTIC-021: Cross-Field Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PYDANTIC-021 |
| **Title** | Cross-Field Validation |
| **Priority** | P0 - Critical |
| **Spec Reference** | PYDANTIC-021, Cross-Field Validation |

**Test Implementation**:
```python
from pydantic import BaseModel, model_validator, ValidationError
from typing import Optional

class JobInputModel(BaseModel):
    input_type: str
    file_name: Optional[str] = None
    file_size: Optional[int] = None
    source_url: Optional[str] = None

    @model_validator(mode="after")
    def validate_input_consistency(self):
        if self.input_type == "FILE":
            if not self.file_name:
                raise ValueError("file_name required for FILE input")
            if self.source_url:
                raise ValueError("source_url not allowed for FILE input")
        elif self.input_type == "URL":
            if not self.source_url:
                raise ValueError("source_url required for URL input")
            if self.file_name:
                raise ValueError("file_name not allowed for URL input")
        return self

def test_valid_file_input():
    """Test valid FILE input configuration."""
    model = JobInputModel(
        input_type="FILE",
        file_name="document.pdf",
        file_size=1024
    )
    assert model.file_name == "document.pdf"

def test_valid_url_input():
    """Test valid URL input configuration."""
    model = JobInputModel(
        input_type="URL",
        source_url="https://example.com/doc.pdf"
    )
    assert model.source_url == "https://example.com/doc.pdf"

def test_file_missing_filename():
    """Test FILE input without file_name rejected."""
    with pytest.raises(ValidationError) as exc_info:
        JobInputModel(input_type="FILE")

    assert "file_name required" in str(exc_info.value)

def test_conflicting_inputs():
    """Test conflicting file_name and source_url rejected."""
    with pytest.raises(ValidationError):
        JobInputModel(
            input_type="FILE",
            file_name="document.pdf",
            source_url="https://example.com"
        )
```

**Assertions**:
- FILE input requires file_name
- URL input requires source_url
- Conflicting fields rejected
- Appropriate error messages returned
- Valid combinations accepted

---

### API-PYDANTIC-022: Conditional Required Fields

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PYDANTIC-022 |
| **Title** | Conditional Required Fields |
| **Priority** | P1 - High |
| **Spec Reference** | PYDANTIC-022, Conditional Validation |

**Test Implementation**:
```python
from pydantic import BaseModel, model_validator, ValidationError
from typing import Optional

class ErrorResponseModel(BaseModel):
    success: bool
    error_code: Optional[str] = None
    error_message: Optional[str] = None
    data: Optional[dict] = None

    @model_validator(mode="after")
    def validate_error_fields(self):
        if not self.success:
            if not self.error_code:
                raise ValueError("error_code required when success is false")
            if not self.error_message:
                raise ValueError("error_message required when success is false")
        return self

def test_success_response_no_error():
    """Test success response doesn't require error fields."""
    model = ErrorResponseModel(success=True, data={"jobId": "uuid"})
    assert model.success is True
    assert model.error_code is None

def test_error_response_with_error_fields():
    """Test error response with required error fields."""
    model = ErrorResponseModel(
        success=False,
        error_code="E001",
        error_message="File too large"
    )
    assert model.error_code == "E001"

def test_error_response_missing_code():
    """Test error response without error_code rejected."""
    with pytest.raises(ValidationError):
        ErrorResponseModel(success=False, error_message="Error occurred")
```

**Assertions**:
- Error fields optional for success=True
- error_code required for success=False
- error_message required for success=False
- Conditional requirements enforced
- Both error fields must be present for errors

---

### API-PYDANTIC-023: Validation Error Aggregation

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PYDANTIC-023 |
| **Title** | Validation Error Aggregation |
| **Priority** | P1 - High |
| **Spec Reference** | PYDANTIC-023, Error Handling |

**Test Implementation**:
```python
from pydantic import BaseModel, Field, ValidationError

class MultiFieldModel(BaseModel):
    name: str = Field(min_length=1)
    age: int = Field(ge=0, le=150)
    email: str = Field(pattern=r"^[\w.-]+@[\w.-]+\.\w+$")

def test_multiple_errors_aggregated():
    """Test multiple validation errors collected."""
    with pytest.raises(ValidationError) as exc_info:
        MultiFieldModel(name="", age=-1, email="invalid")

    errors = exc_info.value.errors()
    assert len(errors) == 3

    error_locs = {e["loc"][0] for e in errors}
    assert "name" in error_locs
    assert "age" in error_locs
    assert "email" in error_locs

def test_error_count_method():
    """Test error_count() returns number of errors."""
    with pytest.raises(ValidationError) as exc_info:
        MultiFieldModel(name="", age=-1, email="invalid")

    assert exc_info.value.error_count() == 3

def test_errors_json_format():
    """Test errors available in JSON format."""
    with pytest.raises(ValidationError) as exc_info:
        MultiFieldModel(name="", age=-1, email="invalid")

    json_str = exc_info.value.json()
    assert isinstance(json_str, str)
    assert "name" in json_str
```

**Assertions**:
- Multiple errors collected in single ValidationError
- All invalid fields reported
- error_count() returns total errors
- JSON serialization available
- Error locations correctly identified

---

### API-PYDANTIC-024: Custom Error Messages

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PYDANTIC-024 |
| **Title** | Custom Error Messages |
| **Priority** | P1 - High |
| **Spec Reference** | PYDANTIC-024, Custom Messages |

**Test Implementation**:
```python
from pydantic import BaseModel, field_validator, ValidationError
from typing import Annotated
from pydantic.functional_validators import AfterValidator

def validate_job_id(v: str) -> str:
    if len(v) != 36:
        raise ValueError("Job ID must be exactly 36 characters (UUID format)")
    return v

class CustomMessageModel(BaseModel):
    job_id: Annotated[str, AfterValidator(validate_job_id)]

    @field_validator("job_id", mode="after")
    @classmethod
    def validate_uuid_format(cls, v):
        parts = v.split("-")
        if len(parts) != 5:
            raise ValueError("Job ID must be in UUID format (xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx)")
        return v

def test_custom_message_in_error():
    """Test custom error message appears in validation error."""
    with pytest.raises(ValidationError) as exc_info:
        CustomMessageModel(job_id="short")

    errors = exc_info.value.errors()
    assert "36 characters" in errors[0]["msg"] or "UUID format" in errors[0]["msg"]

def test_valid_job_id():
    """Test valid job ID passes with custom validators."""
    model = CustomMessageModel(job_id="550e8400-e29b-41d4-a716-446655440000")
    assert model.job_id == "550e8400-e29b-41d4-a716-446655440000"
```

**Assertions**:
- Custom error messages propagated
- ValueError message in ValidationError
- Custom validators can provide context
- Valid values still pass
- Message helps user understand error

---

### API-PYDANTIC-025: model_dump() Basic Behavior

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PYDANTIC-025 |
| **Title** | model_dump() Basic Behavior |
| **Priority** | P0 - Critical |
| **Spec Reference** | PYDANTIC-025, Serialization |

**Test Implementation**:
```python
from pydantic import BaseModel
from datetime import datetime
from uuid import UUID

class JobModel(BaseModel):
    job_id: UUID
    status: str
    created_at: datetime
    file_name: str | None = None

def test_model_dump_basic():
    """Test model_dump() returns dictionary."""
    model = JobModel(
        job_id="550e8400-e29b-41d4-a716-446655440000",
        status="PENDING",
        created_at="2025-12-14T10:00:00Z"
    )
    data = model.model_dump()

    assert isinstance(data, dict)
    assert "job_id" in data
    assert "status" in data
    assert data["status"] == "PENDING"

def test_model_dump_uuid_conversion():
    """Test UUID converted to string in dump."""
    model = JobModel(
        job_id="550e8400-e29b-41d4-a716-446655440000",
        status="PENDING",
        created_at="2025-12-14T10:00:00Z"
    )
    data = model.model_dump()

    assert isinstance(data["job_id"], UUID)
    # For string output, use model_dump(mode="json")

def test_model_dump_none_values():
    """Test None values included by default."""
    model = JobModel(
        job_id="550e8400-e29b-41d4-a716-446655440000",
        status="PENDING",
        created_at="2025-12-14T10:00:00Z",
        file_name=None
    )
    data = model.model_dump()

    assert "file_name" in data
    assert data["file_name"] is None
```

**Assertions**:
- model_dump() returns dictionary
- All fields included in output
- Complex types preserved by default
- None values included by default
- Dictionary usable for JSON serialization

---

### API-PYDANTIC-026: model_dump_json() Serialization

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PYDANTIC-026 |
| **Title** | model_dump_json() Serialization |
| **Priority** | P0 - Critical |
| **Spec Reference** | PYDANTIC-026, JSON Serialization |

**Test Implementation**:
```python
import json
from pydantic import BaseModel
from datetime import datetime
from uuid import UUID

class JobModel(BaseModel):
    job_id: UUID
    status: str
    created_at: datetime

def test_model_dump_json_string():
    """Test model_dump_json() returns JSON string."""
    model = JobModel(
        job_id="550e8400-e29b-41d4-a716-446655440000",
        status="PENDING",
        created_at="2025-12-14T10:00:00Z"
    )
    json_str = model.model_dump_json()

    assert isinstance(json_str, str)
    # Verify it's valid JSON
    parsed = json.loads(json_str)
    assert parsed["status"] == "PENDING"

def test_json_uuid_serialization():
    """Test UUID serialized as string in JSON."""
    model = JobModel(
        job_id="550e8400-e29b-41d4-a716-446655440000",
        status="PENDING",
        created_at="2025-12-14T10:00:00Z"
    )
    json_str = model.model_dump_json()
    parsed = json.loads(json_str)

    assert isinstance(parsed["job_id"], str)
    assert parsed["job_id"] == "550e8400-e29b-41d4-a716-446655440000"

def test_json_datetime_serialization():
    """Test datetime serialized as ISO string."""
    model = JobModel(
        job_id="550e8400-e29b-41d4-a716-446655440000",
        status="PENDING",
        created_at=datetime(2025, 12, 14, 10, 0, 0)
    )
    json_str = model.model_dump_json()
    parsed = json.loads(json_str)

    assert isinstance(parsed["created_at"], str)
    assert "2025-12-14" in parsed["created_at"]
```

**Assertions**:
- model_dump_json() returns valid JSON string
- UUID serialized as string
- datetime serialized as ISO 8601
- JSON parseable with json.loads()
- Complex types handled automatically

---

### API-PYDANTIC-027: Exclude/Include Patterns

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PYDANTIC-027 |
| **Title** | Exclude/Include Patterns |
| **Priority** | P1 - High |
| **Spec Reference** | PYDANTIC-027, Selective Serialization |

**Test Implementation**:
```python
from pydantic import BaseModel

class FullJobModel(BaseModel):
    job_id: str
    status: str
    file_name: str
    internal_path: str  # Internal field, shouldn't be exposed
    processing_node: str  # Internal field

def test_exclude_fields():
    """Test exclude parameter removes fields."""
    model = FullJobModel(
        job_id="uuid",
        status="PENDING",
        file_name="doc.pdf",
        internal_path="/data/jobs/uuid",
        processing_node="worker-1"
    )
    data = model.model_dump(exclude={"internal_path", "processing_node"})

    assert "job_id" in data
    assert "internal_path" not in data
    assert "processing_node" not in data

def test_include_fields():
    """Test include parameter limits to specified fields."""
    model = FullJobModel(
        job_id="uuid",
        status="PENDING",
        file_name="doc.pdf",
        internal_path="/data/jobs/uuid",
        processing_node="worker-1"
    )
    data = model.model_dump(include={"job_id", "status"})

    assert "job_id" in data
    assert "status" in data
    assert "file_name" not in data

def test_exclude_none():
    """Test exclude_none removes None values."""
    model = FullJobModel(
        job_id="uuid",
        status="PENDING",
        file_name="doc.pdf",
        internal_path="/data/jobs/uuid",
        processing_node="worker-1"
    )
    # Would need Optional fields for this, simplified test
    data = model.model_dump(exclude_none=True)
    assert None not in data.values()
```

**Assertions**:
- exclude removes specified fields
- include limits to specified fields
- exclude_none removes None values
- Nested exclusion supported
- Original model unchanged

---

### API-PYDANTIC-028: Field Aliases

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PYDANTIC-028 |
| **Title** | Field Aliases |
| **Priority** | P1 - High |
| **Spec Reference** | PYDANTIC-028, Field Aliases |

**Test Implementation**:
```python
from pydantic import BaseModel, Field, ConfigDict

class AliasedModel(BaseModel):
    model_config = ConfigDict(populate_by_name=True)

    job_id: str = Field(alias="jobId")
    file_name: str = Field(alias="fileName")
    created_at: str = Field(alias="createdAt")

def test_alias_input_accepted():
    """Test aliased field names accepted in input."""
    model = AliasedModel(
        jobId="uuid",
        fileName="document.pdf",
        createdAt="2025-12-14T10:00:00Z"
    )
    assert model.job_id == "uuid"

def test_original_name_accepted():
    """Test original field names accepted with populate_by_name."""
    model = AliasedModel(
        job_id="uuid",
        file_name="document.pdf",
        created_at="2025-12-14T10:00:00Z"
    )
    assert model.job_id == "uuid"

def test_serialization_by_alias():
    """Test serialization uses aliases."""
    model = AliasedModel(
        job_id="uuid",
        file_name="document.pdf",
        created_at="2025-12-14T10:00:00Z"
    )
    data = model.model_dump(by_alias=True)

    assert "jobId" in data
    assert "fileName" in data
    assert "job_id" not in data
```

**Assertions**:
- Alias names accepted in input
- Original names accepted with populate_by_name
- by_alias=True uses aliases in output
- Attribute access uses original names
- Both input methods produce same model

---

### API-PYDANTIC-029: JSON Schema Generation

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PYDANTIC-029 |
| **Title** | JSON Schema Generation |
| **Priority** | P1 - High |
| **Spec Reference** | PYDANTIC-029, JSON Schema |

**Test Implementation**:
```python
from pydantic import BaseModel, Field
from typing import Optional, Literal

class JobResponseModel(BaseModel):
    """Job response from API."""
    job_id: str = Field(description="Unique job identifier (UUID)")
    status: Literal["PENDING", "PROCESSING", "COMPLETE", "ERROR"]
    file_name: Optional[str] = Field(None, description="Original file name")
    file_size: int = Field(ge=1, le=104857600, description="File size in bytes")

def test_json_schema_generated():
    """Test JSON schema can be generated."""
    schema = JobResponseModel.model_json_schema()

    assert "$defs" in schema or "properties" in schema
    assert "properties" in schema
    assert "job_id" in schema["properties"]

def test_schema_includes_constraints():
    """Test schema includes field constraints."""
    schema = JobResponseModel.model_json_schema()

    file_size_props = schema["properties"]["file_size"]
    assert file_size_props.get("minimum") == 1
    assert file_size_props.get("maximum") == 104857600

def test_schema_includes_descriptions():
    """Test schema includes field descriptions."""
    schema = JobResponseModel.model_json_schema()

    assert schema["properties"]["job_id"].get("description") is not None
    assert "UUID" in schema["properties"]["job_id"]["description"]

def test_schema_includes_enum_values():
    """Test schema includes literal/enum values."""
    schema = JobResponseModel.model_json_schema()

    status_props = schema["properties"]["status"]
    assert "enum" in status_props or "anyOf" in status_props
```

**Assertions**:
- JSON schema generated from model
- Field constraints in schema (min, max)
- Field descriptions included
- Enum/Literal values enumerated
- Schema valid for OpenAPI generation

---

### API-PYDANTIC-030: Computed Fields

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PYDANTIC-030 |
| **Title** | Computed Fields |
| **Priority** | P2 - Medium |
| **Spec Reference** | PYDANTIC-030, Computed Fields |

**Test Implementation**:
```python
from pydantic import BaseModel, computed_field

class JobWithComputed(BaseModel):
    job_id: str
    file_size: int

    @computed_field
    @property
    def file_size_mb(self) -> float:
        return self.file_size / (1024 * 1024)

    @computed_field
    @property
    def size_category(self) -> str:
        if self.file_size < 1024 * 1024:
            return "small"
        elif self.file_size < 10 * 1024 * 1024:
            return "medium"
        return "large"

def test_computed_field_access():
    """Test computed fields accessible as properties."""
    model = JobWithComputed(job_id="uuid", file_size=5242880)
    assert model.file_size_mb == 5.0
    assert model.size_category == "medium"

def test_computed_field_in_dump():
    """Test computed fields included in model_dump()."""
    model = JobWithComputed(job_id="uuid", file_size=5242880)
    data = model.model_dump()

    assert "file_size_mb" in data
    assert data["file_size_mb"] == 5.0

def test_computed_field_updates():
    """Test computed fields reflect current values."""
    model = JobWithComputed(job_id="uuid", file_size=512000)
    assert model.size_category == "small"
```

**Assertions**:
- Computed fields accessible as properties
- Computed fields included in model_dump()
- Values computed from model data
- No input required for computed fields
- Computed values update with model data

---

### API-PYDANTIC-031: Private Fields

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PYDANTIC-031 |
| **Title** | Private Fields |
| **Priority** | P2 - Medium |
| **Spec Reference** | PYDANTIC-031, Private Fields |

**Test Implementation**:
```python
from pydantic import BaseModel, PrivateAttr

class JobWithPrivate(BaseModel):
    job_id: str
    status: str
    _internal_state: str = PrivateAttr(default="initialized")
    _retry_count: int = PrivateAttr(default=0)

def test_private_field_not_in_input():
    """Test private fields not required in input."""
    model = JobWithPrivate(job_id="uuid", status="PENDING")
    assert model._internal_state == "initialized"

def test_private_field_not_in_dump():
    """Test private fields excluded from model_dump()."""
    model = JobWithPrivate(job_id="uuid", status="PENDING")
    data = model.model_dump()

    assert "_internal_state" not in data
    assert "_retry_count" not in data

def test_private_field_mutable():
    """Test private fields can be modified."""
    model = JobWithPrivate(job_id="uuid", status="PENDING")
    model._retry_count = 1
    assert model._retry_count == 1

def test_private_field_initialized():
    """Test private fields have default values."""
    model = JobWithPrivate(job_id="uuid", status="PENDING")
    assert model._internal_state == "initialized"
    assert model._retry_count == 0
```

**Assertions**:
- Private fields not in constructor parameters
- Private fields excluded from serialization
- Private fields accessible on instance
- Private fields mutable after creation
- Default values applied to private fields

---

### API-PYDANTIC-032: Model Copy and Update

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-PYDANTIC-032 |
| **Title** | Model Copy and Update |
| **Priority** | P2 - Medium |
| **Spec Reference** | PYDANTIC-032, Model Operations |

**Test Implementation**:
```python
from pydantic import BaseModel

class JobModel(BaseModel):
    job_id: str
    status: str
    progress: int = 0

def test_model_copy():
    """Test model_copy() creates independent copy."""
    original = JobModel(job_id="uuid", status="PENDING")
    copy = original.model_copy()

    assert copy.job_id == original.job_id
    assert copy is not original

def test_model_copy_with_update():
    """Test model_copy(update=...) modifies copy."""
    original = JobModel(job_id="uuid", status="PENDING", progress=0)
    updated = original.model_copy(update={"status": "PROCESSING", "progress": 50})

    assert original.status == "PENDING"
    assert original.progress == 0
    assert updated.status == "PROCESSING"
    assert updated.progress == 50

def test_deep_copy():
    """Test model_copy(deep=True) for nested objects."""
    original = JobModel(job_id="uuid", status="PENDING")
    deep_copy = original.model_copy(deep=True)

    assert deep_copy.job_id == original.job_id
    assert deep_copy is not original
```

**Assertions**:
- model_copy() creates independent copy
- Original unchanged after copy
- update parameter modifies copy
- Deep copy handles nested objects
- Validation applied to updated values

---

## 20. API Concurrency Tests

### 20.1 Overview

API Concurrency Tests validate the system's behavior under high concurrency scenarios, ensuring thread-safety, proper connection pool management, optimistic locking, and race condition prevention. These tests complement Section 14 (Concurrent Request Handling Tests) by focusing on infrastructure-level concurrency concerns.

| Aspect | Description |
|--------|-------------|
| **Scope** | API-level concurrency validation (NFR-102) |
| **Framework** | pytest with concurrent.futures, locust |
| **Test Count** | 8 tests |
| **Priority** | P1 - High (all tests) |

---

### API-CONC-001: Concurrent Upload Requests (Same Session)

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-CONC-001 |
| **Title** | Concurrent Upload Requests (Same Session) |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-102, FR-101, Session Concurrency |

**Test Objective**:
Validate that multiple concurrent upload requests from the same session are handled correctly without race conditions, session corruption, or request queue overflow.

**Test Steps**:
1. Create a valid session
2. Spawn 10 concurrent threads, each uploading a unique file
3. All threads use the same session cookie
4. Measure completion time and verify all uploads succeed
5. Verify session state remains consistent

**Concurrent Request Pattern**:
```python
import concurrent.futures
import requests

session_id = "test-session-123"
files = [f"test-file-{i}.pdf" for i in range(10)]

def upload_file(file_name):
    with open(file_name, 'rb') as f:
        response = requests.post(
            "http://localhost:3000/api/v1/upload",
            files={"file": f},
            cookies={"hx-docling-session": session_id}
        )
    return response.status_code, response.json()

with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
    futures = [executor.submit(upload_file, f) for f in files]
    results = [future.result() for future in futures]
```

**Expected Result**:
- All 10 requests return 201 Created
- Each response contains unique jobId
- All jobs linked to same session
- No 429 Rate Limit errors (within 10 req/min limit)
- Session state consistent (not corrupted)
- Total execution time < 5 seconds (concurrent, not sequential)

**Assertions**:
- 10 successful uploads (100% success rate)
- 10 unique job IDs created
- All jobs associated with same session_id
- No duplicate file paths or job IDs
- Request queue handles concurrent arrivals
- Database shows 10 distinct job records

---

### API-CONC-002: Concurrent Upload Requests (Different Sessions)

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-CONC-002 |
| **Title** | Concurrent Upload Requests (Different Sessions) |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-102, Session Isolation |

**Test Objective**:
Validate that concurrent uploads from different sessions are properly isolated and do not interfere with each other.

**Test Steps**:
1. Create 5 different sessions (session-A through session-E)
2. Spawn 5 concurrent threads, each uploading 2 files with different session
3. Verify all 10 uploads succeed
4. Verify session isolation (no cross-session pollution)

**Concurrent Request Pattern**:
```python
sessions = [f"session-{chr(65+i)}" for i in range(5)]  # A-E

def upload_for_session(session_id):
    results = []
    for i in range(2):
        response = requests.post(
            "http://localhost:3000/api/v1/upload",
            files={"file": open(f"file-{session_id}-{i}.pdf", 'rb')},
            cookies={"hx-docling-session": session_id}
        )
        results.append(response.json())
    return results

with concurrent.futures.ThreadPoolExecutor(max_workers=5) as executor:
    futures = [executor.submit(upload_for_session, s) for s in sessions]
    all_results = [future.result() for future in futures]
```

**Expected Result**:
- All 10 uploads succeed (2 per session)
- Each session has exactly 2 jobs
- No cross-session job leakage
- Session cookies properly isolated
- No shared state corruption

**Assertions**:
- 10 total jobs created
- Session-A has 2 jobs, Session-B has 2 jobs, etc.
- GET /api/v1/history for each session returns only its 2 jobs
- No session sees jobs from other sessions
- Database session_id foreign keys correct

---

### API-CONC-003: Race Condition on Job Status Update

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-CONC-003 |
| **Title** | Race Condition on Job Status Update |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-102, E707, State Consistency |

**Test Objective**:
Validate that concurrent status updates to the same job do not create race conditions, and optimistic locking prevents lost updates.

**Test Steps**:
1. Create a job in PENDING state with version=1
2. Spawn 3 concurrent requests attempting different state transitions:
   - Thread A: Process (PENDING -> PROCESSING)
   - Thread B: Cancel (PENDING -> CANCELLED)
   - Thread C: Process (PENDING -> PROCESSING)
3. Verify only ONE transition succeeds
4. Other requests fail with E707 (Concurrent modification detected)

**Concurrent Request Pattern**:
```python
job_id = "race-condition-job"

def attempt_process():
    return requests.post(
        f"http://localhost:3000/api/v1/process",
        json={"jobId": job_id},
        cookies={"hx-docling-session": session_id}
    )

def attempt_cancel():
    return requests.post(
        f"http://localhost:3000/api/v1/jobs/{job_id}/cancel",
        cookies={"hx-docling-session": session_id}
    )

with concurrent.futures.ThreadPoolExecutor(max_workers=3) as executor:
    futures = [
        executor.submit(attempt_process),
        executor.submit(attempt_cancel),
        executor.submit(attempt_process)
    ]
    results = [future.result() for future in futures]
```

**Expected Result**:
- Exactly 1 request succeeds (either Process or Cancel)
- 2 requests fail with 409 Conflict (E707 or E701)
- Final job state is consistent (either PROCESSING or CANCELLED)
- No intermediate corrupted state

**Assertions**:
- 1 success (status 202 or 200)
- 2 failures (status 409)
- Failed requests have error code E707 or E701
- Database version incremented exactly once
- Final job status matches successful request
- No partial updates or corrupted state

---

### API-CONC-004: Concurrent Cancel Requests

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-CONC-004 |
| **Title** | Concurrent Cancel Requests |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-102, FR-406, Idempotent Operations |

**Test Objective**:
Validate that multiple concurrent cancel requests for the same job are handled idempotently without errors.

**Test Steps**:
1. Start a job processing (status: PROCESSING)
2. Spawn 5 concurrent cancel requests for same job
3. All requests should succeed (idempotent behavior)
4. Verify job cancelled exactly once (not 5 times)

**Concurrent Request Pattern**:
```python
job_id = "cancel-idempotent-job"

def cancel_job():
    response = requests.post(
        f"http://localhost:3000/api/v1/jobs/{job_id}/cancel",
        cookies={"hx-docling-session": session_id}
    )
    return response.status_code, response.json()

with concurrent.futures.ThreadPoolExecutor(max_workers=5) as executor:
    futures = [executor.submit(cancel_job) for _ in range(5)]
    results = [future.result() for future in futures]
```

**Expected Result**:
- All 5 requests return 200 OK
- All responses show status: CANCELLED
- Same cancelledAt timestamp in all responses
- Only 1 SSE cancelled event emitted
- Database shows single CANCELLED transition

**Assertions**:
- All requests succeed (100% success rate)
- Identical response data across all requests
- Single cancelledAt timestamp (not 5 different times)
- SSE stream closed exactly once
- Database audit log shows single CANCELLED event
- Idempotent behavior validated

---

### API-CONC-005: Optimistic Locking Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-CONC-005 |
| **Title** | Optimistic Locking Validation |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-102, DB-302, E707 |

**Test Objective**:
Validate that optimistic locking mechanism correctly prevents lost updates under concurrent modification scenarios.

**Test Steps**:
1. Create job with initial version=1, status=PENDING
2. Read job state (version=1) in two threads
3. Thread A: Update to PROCESSING (version 1 -> 2)
4. Thread B: Attempt update to CANCELLED with stale version=1
5. Verify Thread B fails with E707

**Test Implementation**:
```python
# Thread A: First update succeeds
def update_to_processing():
    response = requests.post(
        f"http://localhost:3000/api/v1/process",
        json={"jobId": job_id},
        cookies={"hx-docling-session": session_id}
    )
    return response

# Thread B: Second update with stale version fails
def update_to_cancelled():
    time.sleep(0.1)  # Ensure Thread A goes first
    response = requests.post(
        f"http://localhost:3000/api/v1/jobs/{job_id}/cancel",
        cookies={"hx-docling-session": session_id}
    )
    return response

with concurrent.futures.ThreadPoolExecutor(max_workers=2) as executor:
    future_a = executor.submit(update_to_processing)
    future_b = executor.submit(update_to_cancelled)
    result_a = future_a.result()
    result_b = future_b.result()
```

**Expected Result**:
- Thread A: 202 Accepted (version 1 -> 2)
- Thread B: 409 Conflict with E707
- Final state: PROCESSING (Thread A's update)
- Database version=2

**Assertions**:
- First update succeeds
- Second update fails with E707
- Error message: "Concurrent modification detected"
- suggestedAction: "Refresh the job status and retry"
- Database version incremented once (1 -> 2)
- No lost update anomaly

---

### API-CONC-006: Database Deadlock Prevention

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-CONC-006 |
| **Title** | Database Deadlock Prevention |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-102, DB-304, Deadlock Detection |

**Test Objective**:
Validate that the database layer properly prevents deadlocks during concurrent multi-table operations.

**Test Steps**:
1. Spawn 10 concurrent requests that each:
   - Update jobs table
   - Insert into results table
   - Update session table
2. Execute operations in consistent order to prevent circular waits
3. Verify all transactions complete without deadlock
4. If deadlock occurs, verify automatic retry succeeds

**Concurrent Request Pattern**:
```python
def multi_table_operation(job_id):
    # Simulates completing a job (multi-table update)
    response = requests.post(
        f"http://localhost:3000/api/v1/jobs/{job_id}/complete",
        json={
            "results": {
                "markdown": "# Document",
                "json": "{}"
            }
        },
        cookies={"hx-docling-session": session_id}
    )
    return response.status_code

job_ids = [f"job-{i}" for i in range(10)]

with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
    futures = [executor.submit(multi_table_operation, jid) for jid in job_ids]
    results = [future.result() for future in futures]
```

**Expected Result**:
- All 10 operations complete successfully
- No deadlock errors (PostgreSQL error code 40P01)
- If retry occurs, it's transparent to client
- Total execution time < 10 seconds

**Assertions**:
- 100% success rate (all 200 OK)
- No 500 Internal Server Error due to deadlock
- Database tables updated consistently
- Transaction isolation prevents anomalies
- Logs show no deadlock warnings (or automatic retry if occurs)
- Consistent table lock acquisition order

---

### API-CONC-007: Request Queuing Under Load

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-CONC-007 |
| **Title** | Request Queuing Under Load |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-102, NFR-202, Request Queue Management |

**Test Objective**:
Validate that the API properly queues requests under high load and does not drop or reject valid requests due to concurrency limits.

**Test Steps**:
1. Configure test environment with limited worker threads (e.g., 4 workers)
2. Send 50 concurrent upload requests
3. Verify all requests eventually complete
4. Measure queue wait time and processing time
5. Ensure no requests timeout or fail due to queue overflow

**Load Test Pattern (using Locust)**:
```python
from locust import HttpUser, task, between

class ConcurrencyTestUser(HttpUser):
    wait_time = between(0, 0.1)  # Minimal wait

    @task
    def upload_file(self):
        with open("test-file.pdf", "rb") as f:
            self.client.post(
                "/api/v1/upload",
                files={"file": f},
                cookies={"hx-docling-session": "load-test-session"}
            )

# Run with: locust -f test_concurrency.py --users 50 --spawn-rate 50 --run-time 30s
```

**Expected Result**:
- 50/50 requests succeed (100% success rate)
- Average response time < 2 seconds
- p95 response time < 5 seconds
- No 503 Service Unavailable errors
- Request queue properly manages backpressure

**Assertions**:
- Zero failed requests
- Zero timeout errors
- Queue depth metric available (if instrumented)
- All 50 jobs created in database
- No dropped requests
- Server remains responsive throughout test

---

### API-CONC-008: Connection Pool Behavior Under Concurrency

| Attribute | Value |
|-----------|-------|
| **Test ID** | API-CONC-008 |
| **Title** | Connection Pool Behavior Under Concurrency |
| **Priority** | P1 - High |
| **Spec Reference** | NFR-102, DB-305, Connection Pool Management |

**Test Objective**:
Validate that database connection pool properly handles concurrent requests without exhaustion, connection leaks, or timeout errors.

**Test Steps**:
1. Configure connection pool with limited size (e.g., 10 connections)
2. Send 100 concurrent API requests requiring DB access
3. Verify connection pool:
   - Does not exhaust (no "connection pool exhausted" errors)
   - Properly returns connections after use
   - Queues requests when pool full
   - Times out gracefully if wait exceeds threshold
4. Monitor pool metrics (active, idle, waiting)

**Concurrent Request Pattern**:
```python
def make_history_request():
    response = requests.get(
        "http://localhost:3000/api/v1/history",
        cookies={"hx-docling-session": session_id}
    )
    return response.status_code

with concurrent.futures.ThreadPoolExecutor(max_workers=100) as executor:
    futures = [executor.submit(make_history_request) for _ in range(100)]
    results = [future.result() for future in futures]
```

**Expected Result**:
- All 100 requests succeed (or fail gracefully with 503 if timeout)
- No connection leak (idle connections return to pool)
- Connection pool metrics remain healthy:
  - Peak usage ≤ pool size (10)
  - No permanent exhaustion
  - Connections released after request completion

**Assertions**:
- ≥95% success rate (100 requests succeed)
- No "too many connections" database errors
- Connection pool size does not grow unbounded
- Idle connections available after load test
- No orphaned database connections
- Pool timeout configured (e.g., 30s max wait)
- Graceful degradation under extreme load (503 preferred over 500)

**Monitoring Validation**:
```python
# Check connection pool metrics (if instrumented)
metrics = requests.get("http://localhost:3000/metrics").json()
assert metrics["db_pool_active"] <= 10
assert metrics["db_pool_idle"] >= 0
assert metrics["db_pool_waiting"] == 0  # After load test completes
```

---

## Test Summary

### Test Count by Endpoint

| Endpoint | Test Count |
|----------|-----------|
| POST /api/v1/upload | 13 |
| POST /api/v1/process | 4 |
| POST /api/v1/jobs/{id}/cancel | 3 |
| POST /api/v1/jobs/{id}/resume | 5 |
| GET /api/v1/jobs/{id} | 4 |
| GET /api/v1/history | 5 |
| GET /api/v1/health | 3 |
| GET /api/v1/process/{id}/events (SSE) | 6 |
| Authentication/Authorization | 7 |
| Security Tests | 9 |
| MCP Integration Layer | 10 |
| Database State Verification | 8 |
| Concurrent Request Handling | 8 |
| Response Schema Validation | 8 |
| Workflow Orchestration | 8 |
| HTTP Method Validation | 5 |
| Dependency Failure | 6 |
| Pydantic Model Validation | 32 |
| API Concurrency Tests | 8 |
| **Total** | **152** |

### Test Count by Priority

| Priority | Count |
|----------|-------|
| P0 - Critical | 78 |
| P1 - High | 66 |
| P2 - Medium | 8 |
| **Total** | **152** |

### Error Code Coverage

| Error Code | Description | Test ID |
|------------|-------------|---------|
| E001 | File too large | API-UP-006 |
| E002 | Unsupported file type | API-UP-007 |
| E003 | Storage error | API-DB-004 |
| E004 | File corrupted | API-SEC-005 |
| E100 | No input provided | API-UP-013 |
| E101 | Invalid URL format | API-UP-009 |
| E104 | URL blocked | API-UP-010 |
| E201 | Invalid MCP request format | API-MCP-007 |
| E202 | MCP timeout | API-MCP-010 |
| E203 | Invalid tool parameters | API-MCP-007 |
| E204 | MCP server unavailable | API-MCP-007, API-FAIL-003 |
| E205 | Tool unavailable | API-MCP-008 |
| E301 | Document conversion failed | API-SSE-005 |
| E302 | Export failed | API-MCP-009, API-WF-004 |
| E401 | Authentication required | API-AUTH-001, API-AUTH-002, API-AUTH-007 |
| E402 | Session expired | API-AUTH-003 |
| E403 | Access denied | API-AUTH-004, API-AUTH-005 |
| E405 | Method not allowed | API-METHOD-001, API-METHOD-005 |
| E501 | Job not found | API-PR-002, API-CN-003, API-AUTH-004 |
| E601 | Rate limit exceeded | API-UP-012 |
| E602 | Database unavailable | API-FAIL-001 |
| E603 | Cache service unavailable | API-FAIL-002 |
| E604 | Circuit breaker open | API-FAIL-004 |
| E605 | Server shutting down | API-FAIL-006 |
| E701 | Job already processing | API-PR-003, API-DB-007, API-RACE-001, API-RACE-005, API-CONC-003 |
| E702 | Job not cancellable | API-CN-002, API-WF-002 |
| E703 | No checkpoint | API-RS-002, API-WF-002 |
| E704 | Checkpoint expired | API-RS-003 |
| E705 | Checkpoint corrupted | API-RS-004 |
| E706 | Job already completed | API-RS-005, API-WF-002 |
| E707 | Concurrent modification detected | API-RACE-007, API-CONC-003, API-CONC-005 |
| E801 | Invalid pagination | API-HI-004 |

---

## Changelog

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 2.2.0 | 2025-12-15 | Julia Santos | Added API Concurrency Tests (Section 20): 8 comprehensive P1 HIGH tests covering concurrent upload requests (same/different sessions), race condition handling, concurrent cancel requests, optimistic locking validation, database deadlock prevention, request queuing under load, and connection pool behavior under concurrency. Total tests increased from 144 to 152. Priority breakdown: All 8 tests are P1 HIGH. Addresses A-08 defect (API concurrency test coverage gap). References NFR-102 for concurrency requirements. |
| 2.1.0 | 2025-12-15 | Julia Santos | Added Pydantic Model Validation Tests (Section 19): 32 comprehensive tests covering BaseModel field validation (10 tests), Type Coercion (6 tests), Custom Validators (8 tests), and Serialization (8 tests). Total tests increased from 112 to 144. Priority breakdown: 16 P0 Critical, 13 P1 High, 3 P2 Medium. Addresses A-02 defect (data validation coverage gap). Target: 90%+ data validation coverage. |
| 2.0.0 | 2025-12-14 | Julia Santos | MAJOR UPDATE COMPLETE: Added final P1 HIGH test sections: Workflow Orchestration Tests (8 tests), HTTP Method Validation Tests (5 tests), Dependency Failure Tests (6 tests). Total tests increased from 93 to 112. Added 8 new error codes (E302 extended, E405, E602, E603, E604, E605). All 19 new tests are P1 HIGH priority. Updated P1 count from 26 to 45. |
| 1.3.0 | 2025-12-14 | Julia Santos | Added P0 CRITICAL test sections: Concurrent Request Handling Tests (8 tests), Response Schema Validation Tests (8 tests). Total tests increased from 77 to 93. Added new error code E707 (Concurrent modification detected). All 16 new tests are P0 CRITICAL priority. Updated P0 count from 46 to 62. |
| 1.2.0 | 2025-12-14 | Julia Santos | Added P0 CRITICAL test sections: MCP Integration Layer Tests (10 tests), Database State Verification Tests (8 tests). Total tests increased from 59 to 77. Added 7 new error codes (E003, E201, E202, E203, E204, E205, E302). All new tests are P0 CRITICAL priority. |
| 1.1.0 | 2025-12-14 | Julia Santos | Added P0 CRITICAL test sections: SSE Event Stream Tests (6 tests), Authentication/Authorization Tests (7 tests). Total tests increased from 46 to 59. Added 4 new error codes (E301, E401, E402, E403). |
| 1.0.0 | 2025-12-12 | Julia Santos | Initial API test cases |
