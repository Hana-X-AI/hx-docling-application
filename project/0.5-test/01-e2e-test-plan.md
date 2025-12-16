# HX Docling Application - E2E Test Plan

**Document Type**: End-to-End Test Plan
**Version**: 1.3.0
**Status**: DRAFT
**Created**: 2025-12-12
**Last Updated**: 2025-12-15
**Author**: Julia Santos (Testing & QA Specialist)
**Master Plan Reference**: `project/0.5-test/00-test-plan-overview.md`
**Specification Reference**: `project/0.3-specification/0.3.1-detailed-specification.md` v1.2.1

---

## Table of Contents

1. [Overview](#1-overview)
2. [Critical Path Tests](#2-critical-path-tests)
3. [File Upload Workflow Tests](#3-file-upload-workflow-tests)
4. [URL Conversion Workflow Tests](#4-url-conversion-workflow-tests)
5. [Progress Tracking and SSE Tests](#5-progress-tracking-and-sse-tests)
6. [Results Display Tests](#6-results-display-tests)
7. [History View Tests](#7-history-view-tests)
8. [Error Handling and Recovery Tests](#8-error-handling-and-recovery-tests)
9. [Session Management Tests](#9-session-management-tests)
10. [Accessibility Tests](#10-accessibility-tests)
11. [Performance Tests](#11-performance-tests)
12. [Cross-Browser Tests](#12-cross-browser-tests)
13. [Rate Limiting Tests](#13-rate-limiting-tests)
14. [MCP Tool Coverage Tests](#14-mcp-tool-coverage-tests)
15. [Database State Verification Tests](#15-database-state-verification-tests)
16. [Checkpoint and Resume Tests](#16-checkpoint-and-resume-tests)
17. [Security Tests](#17-security-tests)
18. [State Machine Edge Cases](#18-state-machine-edge-cases)
19. [Theme & Responsive Tests](#19-theme--responsive-tests)
20. [Infrastructure Health Tests](#20-infrastructure-health-tests)
21. [Next.js Specific Tests](#21-nextjs-specific-tests)

---

## 1. Overview

### 1.1 Purpose

This document defines comprehensive end-to-end test scenarios that validate complete user workflows from file upload through processing to results display and download. E2E tests ensure the integrated system functions correctly from the user's perspective.

### 1.2 Test Framework

- **Tool**: Playwright
- **Test Directory**: `test/e2e/`
- **Configuration**: `playwright.config.ts`
- **Run Command**: `npm run test:e2e`

### 1.3 Test Execution Environment

| Environment | URL | Purpose |
|-------------|-----|---------|
| Local Development | http://localhost:3000 | Developer testing |
| CI/CD | http://localhost:3000 (built) | Automated testing |
| Staging | https://staging.docling.hx.dev.local | Pre-release testing |

### 1.4 Priority Definitions

| Priority | Definition | Execution |
|----------|------------|-----------|
| P0 | Critical - must pass for release | Every build |
| P1 | High - core functionality | Every PR |
| P2 | Medium - important features | Daily |
| P3 | Low - edge cases | Weekly |

---

## 2. Critical Path Tests

### 2.1 Overview

Critical path tests cover the primary user journeys that MUST work for the application to be considered functional.

### E2E-CP-001: Complete File Upload to Download Flow

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-CP-001 |
| **Title** | Complete File Upload to Download Flow |
| **Priority** | P0 - Critical |
| **FR References** | FR-101, FR-103, FR-104, FR-401, FR-403, FR-404, FR-501, FR-601, FR-606 |
| **Preconditions** | Application running, test PDF available |

**Test Steps**:

1. Navigate to application home page
2. Upload a valid PDF file (sample.pdf, 100KB) via drag-and-drop
3. Verify file preview displays filename and size
4. Click "Process" button
5. Verify progress indicator appears with SSE connection
6. Wait for processing completion (100%)
7. Verify results tabs appear (Markdown, HTML, JSON, Raw)
8. Click Markdown tab
9. Verify markdown content is displayed
10. Click Download button
11. Verify file downloads with correct naming convention

**Expected Results**:

- File upload accepted and preview shown
- Processing starts with SSE streaming progress
- Progress updates from 0% to 100%
- All four result tabs populated
- Markdown content renders correctly
- Download produces file: `{filename}_{timestamp}_markdown.md`

**Acceptance Criteria Reference**: AC-101.3, AC-103.1, AC-104.1, AC-401.1, AC-403.1, AC-404.1, AC-501.1, AC-601.1, AC-606.1

---

### E2E-CP-002: Complete URL Conversion Flow

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-CP-002 |
| **Title** | Complete URL Conversion Flow |
| **Priority** | P0 - Critical |
| **FR References** | FR-201, FR-202, FR-401, FR-403, FR-404, FR-501, FR-601 |
| **Preconditions** | Application running, valid external URL available |

**Test Steps**:

1. Navigate to application home page
2. Enter valid URL (https://example.com/document.html)
3. Verify URL validation passes (no error displayed)
4. Click "Process" button
5. Verify progress indicator appears
6. Wait for processing completion
7. Verify results tabs appear with content
8. Switch between all four tabs
9. Download results in each format

**Expected Results**:

- URL accepted after validation
- Processing completes successfully
- All result formats available
- Downloads work for all formats

**Acceptance Criteria Reference**: AC-201.1, AC-202.1, AC-401.1, AC-601.1

---

### E2E-CP-003: History View and Re-download

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-CP-003 |
| **Title** | History View and Re-download |
| **Priority** | P0 - Critical |
| **FR References** | FR-701, FR-702, FR-703, FR-704 |
| **Preconditions** | At least one completed job exists in session |

**Test Steps**:

1. Complete a file processing flow (E2E-CP-001)
2. Navigate to History page
3. Verify completed job appears in history table
4. Verify job shows correct status, filename, date
5. Click on job row to view details
6. Verify job detail modal/view shows all result formats
7. Download results from history view
8. Navigate back to history, verify pagination works (if multiple jobs)

**Expected Results**:

- History shows all jobs for current session
- Jobs sorted by createdAt DESC
- Job details accessible and complete
- Re-download works from history

**Acceptance Criteria Reference**: AC-701.1, AC-701.2, AC-702.1, AC-703.1, AC-704.1

---

### E2E-CP-004: SSE Reconnection on Network Interruption

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-CP-004 |
| **Title** | SSE Reconnection on Network Interruption |
| **Priority** | P0 - Critical |
| **FR References** | FR-501, FR-502, FR-503, NFR-201 |
| **Preconditions** | Application running, multi-page PDF for longer processing |

**Test Steps**:

1. Upload a larger PDF file (multi-page.pdf, 5MB)
2. Start processing
3. Wait for progress to reach ~30%
4. Simulate network interruption (context.setOffline(true))
5. Verify "Reconnecting..." UI appears
6. Wait 5 seconds
7. Restore network (context.setOffline(false))
8. Verify reconnection succeeds
9. Verify processing continues and completes
10. Verify final results are correct

**Expected Results**:

- Reconnection UI displayed during offline period
- Reconnection succeeds within 30 seconds
- Processing continues from checkpoint
- Final results complete and correct

**Acceptance Criteria Reference**: AC-503.1, AC-503.2, AC-503.4, NFR-201

---

## 3. File Upload Workflow Tests

### E2E-FU-001: Drag-and-Drop File Upload

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-FU-001 |
| **Title** | Drag-and-Drop File Upload |
| **Priority** | P1 - High |
| **FR References** | FR-101 |
| **Preconditions** | Application home page loaded |

**Test Steps**:

1. Navigate to home page
2. Locate upload zone
3. Drag file onto upload zone
4. Verify visual feedback on drag-over (border highlight)
5. Drop file
6. Verify file accepted and preview shown

**Expected Results**:

- Upload zone shows drag-over visual feedback
- File accepted on drop
- File preview displays name, size, type icon

**Acceptance Criteria Reference**: AC-101.1, AC-101.2, AC-101.3

---

### E2E-FU-002: Click-to-Browse File Upload

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-FU-002 |
| **Title** | Click-to-Browse File Upload |
| **Priority** | P1 - High |
| **FR References** | FR-102 |
| **Preconditions** | Application home page loaded |

**Test Steps**:

1. Navigate to home page
2. Click on upload zone
3. Verify file browser opens
4. Select valid file
5. Verify file appears in upload zone

**Expected Results**:

- Click opens file browser dialog
- Selected file populates upload zone
- File preview shown

**Acceptance Criteria Reference**: AC-102.1, AC-102.2

---

### E2E-FU-003: File Type Validation - Supported Types

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-FU-003 |
| **Title** | File Type Validation - Supported Types |
| **Priority** | P1 - High |
| **FR References** | FR-103 |
| **Preconditions** | Test files of each supported type available |

**Test Steps**:

For each supported file type (.pdf, .doc, .docx, .ppt, .pptx, .xls, .xlsx, .png, .jpg, .jpeg, .tiff):

1. Upload file of that type
2. Verify file accepted
3. Verify processing can be initiated

**Expected Results**:

All supported file types accepted without validation error.

**Acceptance Criteria Reference**: AC-103.1

---

### E2E-FU-004: File Type Validation - Rejected Types

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-FU-004 |
| **Title** | File Type Validation - Rejected Types |
| **Priority** | P1 - High |
| **FR References** | FR-103 |
| **Preconditions** | Unsupported file types available (.exe, .zip, .txt) |

**Test Steps**:

1. Attempt to upload unsupported file type (.exe)
2. Verify rejection with error E002

**Expected Results**:

- File rejected before upload
- Error message: "Unsupported file type" (E002)
- No file preview shown

**Acceptance Criteria Reference**: AC-103.2

---

### E2E-FU-005: File Size Validation - Within Limits

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-FU-005 |
| **Title** | File Size Validation - Within Limits |
| **Priority** | P1 - High |
| **FR References** | FR-104 |
| **Preconditions** | Files within size limits for each type |

**Test Steps**:

1. Upload PDF file under 100MB
2. Verify acceptance
3. Upload DOCX file under 50MB
4. Verify acceptance
5. Upload image under 25MB
6. Verify acceptance

**Expected Results**:

All files within limits accepted.

**Acceptance Criteria Reference**: AC-104.1

---

### E2E-FU-006: File Size Validation - Exceeds Limits

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-FU-006 |
| **Title** | File Size Validation - Exceeds Limits |
| **Priority** | P1 - High |
| **FR References** | FR-104 |
| **Preconditions** | File exceeding 100MB available |

**Test Steps**:

1. Attempt to upload file exceeding size limit (too-large.pdf, 150MB)
2. Verify rejection with error E001

**Expected Results**:

- File rejected before upload
- Error message: "File too large" (E001)
- Size limit displayed in error

**Acceptance Criteria Reference**: AC-104.2

---

### E2E-FU-007: File Preview Display

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-FU-007 |
| **Title** | File Preview Display |
| **Priority** | P2 - Medium |
| **FR References** | FR-105 |
| **Preconditions** | Valid file available |

**Test Steps**:

1. Upload valid PDF file
2. Verify file name displayed
3. Verify file size displayed (formatted, e.g., "1.5 MB")
4. Verify file type icon displayed

**Expected Results**:

- File name visible
- Size formatted appropriately
- Correct icon for file type

**Acceptance Criteria Reference**: AC-105.1, AC-105.2, AC-105.3

---

## 4. URL Conversion Workflow Tests

### E2E-URL-001: Valid URL Entry

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-URL-001 |
| **Title** | Valid URL Entry |
| **Priority** | P1 - High |
| **FR References** | FR-201, FR-202 |
| **Preconditions** | Application home page loaded |

**Test Steps**:

1. Navigate to home page
2. Click URL input field
3. Type valid HTTPS URL
4. Verify no validation error
5. Process button becomes enabled

**Expected Results**:

- URL input accepts text
- No validation error for valid URL
- Process button enabled

**Acceptance Criteria Reference**: AC-201.1, AC-202.1

---

### E2E-URL-002: Invalid URL Format Rejection

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-URL-002 |
| **Title** | Invalid URL Format Rejection |
| **Priority** | P1 - High |
| **FR References** | FR-202 |
| **Preconditions** | Application home page loaded |

**Test Steps**:

1. Enter invalid URL format (e.g., "not-a-url", "ftp://server")
2. Blur field or submit
3. Verify error E101 displayed

**Expected Results**:

- Validation error appears
- Error code E101: "Invalid URL format"
- Process button disabled

**Acceptance Criteria Reference**: AC-202.2

---

### E2E-URL-003: SSRF Prevention - Private IPs Blocked

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-URL-003 |
| **Title** | SSRF Prevention - Private IPs Blocked |
| **Priority** | P0 - Critical (Security) |
| **FR References** | FR-203 |
| **Preconditions** | Application home page loaded |

**Test Steps**:

For each blocked URL pattern:
- http://localhost/
- http://127.0.0.1/
- http://10.0.0.1/
- http://172.16.0.1/
- http://192.168.1.1/
- http://169.254.169.254/ (AWS metadata)
- http://internal.hx.dev.local/

1. Enter blocked URL
2. Attempt to process
3. Verify rejection with error E104

**Expected Results**:

- All private/internal URLs rejected
- Error code E104: "URL blocked"
- No request sent to blocked address

**Acceptance Criteria Reference**: AC-203.1 through AC-203.7

---

### E2E-URL-004: URL Clear Button

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-URL-004 |
| **Title** | URL Clear Button |
| **Priority** | P2 - Medium |
| **FR References** | FR-201 |
| **Preconditions** | URL entered in input field |

**Test Steps**:

1. Enter URL in input field
2. Click clear button
3. Verify URL input emptied

**Expected Results**:

- URL field cleared
- Process button disabled

**Acceptance Criteria Reference**: AC-201.3

---

### E2E-URL-005: Paste from Clipboard

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-URL-005 |
| **Title** | Paste from Clipboard |
| **Priority** | P2 - Medium |
| **FR References** | FR-201 |
| **Preconditions** | Valid URL in clipboard |

**Test Steps**:

1. Copy URL to clipboard
2. Focus URL input field
3. Paste (Ctrl+V / Cmd+V)
4. Verify URL pasted correctly

**Expected Results**:

- URL pasted into field
- Validation runs on pasted content

**Acceptance Criteria Reference**: AC-201.2

---

## 5. Progress Tracking and SSE Tests

### E2E-SSE-001: Progress Indicator Display

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-SSE-001 |
| **Title** | Progress Indicator Display |
| **Priority** | P1 - High |
| **FR References** | FR-501, FR-502 |
| **Preconditions** | Valid file uploaded, ready to process |

**Test Steps**:

1. Start processing
2. Verify progress card appears
3. Verify SSE connection indicator shows "Connected"
4. Verify progress bar updates
5. Verify stage messages display correctly
6. Verify progress percentage never goes backwards

**Expected Results**:

- Progress card visible immediately
- Connection status shown
- Progress updates smoothly 0% to 100%
- Stage messages match spec (upload, parsing, conversion, export, saving, complete)

**Acceptance Criteria Reference**: AC-501.1, AC-502.1, AC-502.2, AC-502.3

---

### E2E-SSE-002: Progress Stage Transitions

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-SSE-002 |
| **Title** | Progress Stage Transitions |
| **Priority** | P1 - High |
| **FR References** | FR-502 |
| **Preconditions** | Processing in progress |

**Test Steps**:

1. Start processing
2. Record all stage transitions
3. Verify stages occur in correct order:
   - upload (0-10%)
   - parsing (10-40%)
   - conversion (40-80%)
   - export (80-95%)
   - saving (95-99%)
   - complete (100%)

**Expected Results**:

- Stages occur in specified order
- Percentage ranges match specification

**Acceptance Criteria Reference**: FR-502 Progress Stages table

---

### E2E-SSE-003: SSE Automatic Reconnection

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-SSE-003 |
| **Title** | SSE Automatic Reconnection |
| **Priority** | P1 - High |
| **FR References** | FR-503 |
| **Preconditions** | Processing in progress |

**Test Steps**:

1. Start processing with larger file
2. Wait for progress to reach 20%
3. Simulate brief network interruption (3 seconds)
4. Verify "Reconnecting..." displayed
5. Restore network
6. Verify reconnection with exponential backoff
7. Verify Last-Event-ID sent on reconnect
8. Verify processing continues

**Expected Results**:

- Reconnection UI appears during disruption
- Exponential backoff followed (1s, 2s, 4s...)
- Reconnection succeeds
- No duplicate events after reconnection

**Acceptance Criteria Reference**: AC-503.1, AC-503.2, AC-503.3, AC-503.4

---

### E2E-SSE-004: Polling Fallback

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-SSE-004 |
| **Title** | Polling Fallback |
| **Priority** | P1 - High |
| **FR References** | FR-504 |
| **Preconditions** | SSE connection failing repeatedly |

**Test Steps**:

1. Start processing
2. Block SSE endpoint to simulate unavailability
3. Allow max retries (10) to be exhausted
4. Verify fallback to polling mode
5. Verify "Using backup connection" displayed
6. Verify polling occurs at 2s intervals
7. Verify processing completes via polling

**Expected Results**:

- Fallback message displayed
- Polling at 2s intervals
- Processing completes successfully

**Acceptance Criteria Reference**: AC-504.1, AC-504.2, AC-504.3

---

## 6. Results Display Tests

### E2E-RD-001: Tabbed Results Viewer

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-RD-001 |
| **Title** | Tabbed Results Viewer |
| **Priority** | P1 - High |
| **FR References** | FR-601 |
| **Preconditions** | Processing complete with results |

**Test Steps**:

1. Complete file processing
2. Verify four tabs visible: Markdown, HTML, JSON, Raw
3. Verify tab order is fixed (Markdown first)
4. Verify default tab is Markdown
5. Click each tab and verify content loads

**Expected Results**:

- Four tabs in correct order
- Default tab is Markdown
- All tabs have content

**Acceptance Criteria Reference**: AC-601.1, AC-601.2, AC-601.3

---

### E2E-RD-002: Markdown Rendering

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-RD-002 |
| **Title** | Markdown Rendering |
| **Priority** | P1 - High |
| **FR References** | FR-602 |
| **Preconditions** | Processing complete with markdown result |

**Test Steps**:

1. View Markdown tab
2. Verify GFM syntax rendered (headings, lists, links)
3. Verify code blocks have syntax highlighting
4. Verify tables render correctly
5. Verify images render (if present as base64)

**Expected Results**:

- Markdown rendered as formatted HTML
- Syntax highlighting on code blocks
- Tables display correctly

**Acceptance Criteria Reference**: AC-602.1, AC-602.2, AC-602.3, AC-602.4

---

### E2E-RD-003: HTML Preview in Sandbox

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-RD-003 |
| **Title** | HTML Preview in Sandbox |
| **Priority** | P1 - High |
| **FR References** | FR-603 |
| **Preconditions** | Processing complete with HTML result |

**Test Steps**:

1. Click HTML tab
2. Verify HTML renders in iframe
3. Verify iframe has sandbox attribute
4. Verify content is scrollable

**Expected Results**:

- HTML displayed in sandboxed iframe
- Sandbox attribute present for security
- Content scrolls if overflow

**Acceptance Criteria Reference**: AC-603.1, AC-603.2, AC-603.3

---

### E2E-RD-004: JSON Display

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-RD-004 |
| **Title** | JSON Display |
| **Priority** | P1 - High |
| **FR References** | FR-604 |
| **Preconditions** | Processing complete with JSON result |

**Test Steps**:

1. Click JSON tab
2. Verify JSON is pretty-printed
3. Verify syntax highlighting applied
4. Verify large JSON is scrollable

**Expected Results**:

- JSON formatted with 2-space indentation
- Syntax highlighting (keys, values, strings)
- Scrollable container

**Acceptance Criteria Reference**: AC-604.1, AC-604.2

---

### E2E-RD-005: Download All Formats

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-RD-005 |
| **Title** | Download All Formats |
| **Priority** | P1 - High |
| **FR References** | FR-606 |
| **Preconditions** | Processing complete with results |

**Test Steps**:

For each format (Markdown, HTML, JSON, Raw):

1. Navigate to format tab
2. Click Download button
3. Verify download dialog triggered
4. Verify filename format: `{original}_{timestamp}_{format}.{ext}`

**Expected Results**:

- All formats downloadable
- Correct file extensions (.md, .html, .json, .json)
- Naming convention followed

**Acceptance Criteria Reference**: AC-606.1, AC-606.2, AC-606.3

---

### E2E-RD-006: Tab Memory in Session

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-RD-006 |
| **Title** | Tab Memory in Session |
| **Priority** | P2 - Medium |
| **FR References** | FR-601 |
| **Preconditions** | Results displayed |

**Test Steps**:

1. View results
2. Click JSON tab
3. Navigate away (e.g., to History)
4. Return to results
5. Verify JSON tab still selected

**Expected Results**:

- Last selected tab remembered within session

**Acceptance Criteria Reference**: AC-601.4

---

## 7. History View Tests

### E2E-HV-001: History Page Display

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-HV-001 |
| **Title** | History Page Display |
| **Priority** | P1 - High |
| **FR References** | FR-701 |
| **Preconditions** | Multiple jobs completed in session |

**Test Steps**:

1. Complete 3+ processing jobs
2. Navigate to History page
3. Verify table displays with columns: File/URL, Status, Date, Actions
4. Verify only current session jobs shown
5. Verify sorted by createdAt DESC (newest first)

**Expected Results**:

- History table visible
- Correct columns displayed
- Session filtering applied
- Sorted newest first

**Acceptance Criteria Reference**: AC-701.1, AC-701.2, AC-701.3

---

### E2E-HV-002: Pagination

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-HV-002 |
| **Title** | Pagination |
| **Priority** | P1 - High |
| **FR References** | FR-702 |
| **Preconditions** | 25+ jobs in session |

**Test Steps**:

1. Create 25+ completed jobs
2. Navigate to History
3. Verify page 1 shows 20 jobs (default)
4. Verify page navigation controls visible
5. Verify total count displayed
6. Click next page
7. Verify remaining jobs shown

**Expected Results**:

- Default page size 20
- Pagination controls work
- Total count accurate
- Page navigation functions

**Acceptance Criteria Reference**: AC-702.1, AC-702.3, AC-702.4

---

### E2E-HV-003: Job Detail View

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-HV-003 |
| **Title** | Job Detail View |
| **Priority** | P1 - High |
| **FR References** | FR-703 |
| **Preconditions** | Completed job in history |

**Test Steps**:

1. Navigate to History
2. Click on a completed job row
3. Verify detail view/modal opens
4. Verify all result formats displayed
5. Verify download buttons available
6. Close detail view

**Expected Results**:

- Detail view opens
- All result formats accessible
- Download buttons work

**Acceptance Criteria Reference**: AC-703.1, AC-703.2, AC-703.3

---

### E2E-HV-004: Re-download Past Results

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-HV-004 |
| **Title** | Re-download Past Results |
| **Priority** | P1 - High |
| **FR References** | FR-704 |
| **Preconditions** | Completed job in history |

**Test Steps**:

1. Navigate to History
2. Open job detail
3. Download markdown result
4. Verify download succeeds
5. Verify content matches original

**Expected Results**:

- Download from history works
- Content identical to original processing

**Acceptance Criteria Reference**: AC-704.1, AC-704.2

---

## 8. Error Handling and Recovery Tests

### E2E-ERR-001: Error Display

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-ERR-001 |
| **Title** | Error Display |
| **Priority** | P1 - High |
| **FR References** | FR-801 |
| **Preconditions** | Trigger processing error |

**Test Steps**:

1. Upload corrupted file (corrupted.pdf)
2. Start processing
3. Verify error display appears
4. Verify error code displayed
5. Verify user-friendly message displayed
6. Verify suggested action displayed

**Expected Results**:

- Error card/modal appears
- Error code visible (e.g., E004)
- Clear message for user
- Actionable suggestion

**Acceptance Criteria Reference**: AC-801.1, AC-801.2, AC-801.3

---

### E2E-ERR-002: Retry Button for Retryable Errors

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-ERR-002 |
| **Title** | Retry Button for Retryable Errors |
| **Priority** | P1 - High |
| **FR References** | FR-802 |
| **Preconditions** | Transient error occurred (E201, E202, E301) |

**Test Steps**:

1. Trigger retryable error (e.g., MCP timeout)
2. Verify error displayed
3. Verify "Retry" button visible
4. Click Retry button
5. Verify processing restarts

**Expected Results**:

- Retry button shown for retryable errors
- Retry initiates new processing attempt

**Acceptance Criteria Reference**: AC-802.1

---

### E2E-ERR-003: Modify Input for File Errors

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-ERR-003 |
| **Title** | Modify Input for File Errors |
| **Priority** | P1 - High |
| **FR References** | FR-802 |
| **Preconditions** | File error occurred (E001, E002) |

**Test Steps**:

1. Upload invalid file (triggers E002)
2. Verify error displayed
3. Verify "Try Different File" button visible
4. Click button
5. Verify upload zone cleared and ready

**Expected Results**:

- "Try Different File" option shown
- Clicking clears state for new upload

**Acceptance Criteria Reference**: AC-802.2

---

### E2E-ERR-004: Automatic Retry with Exponential Backoff

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-ERR-004 |
| **Title** | Automatic Retry with Exponential Backoff |
| **Priority** | P1 - High |
| **FR References** | FR-803 |
| **Preconditions** | MCP server configured to fail transiently |

**Test Steps**:

1. Configure MCP mock to fail first 2 requests
2. Start processing
3. Verify RETRY_1 state shown after first failure
4. Verify 1s backoff before retry
5. Verify RETRY_2 state after second failure
6. Verify 2s backoff
7. Verify success on third attempt
8. Verify final result correct

**Expected Results**:

- Retry states visible (RETRY_1, RETRY_2)
- Backoff delays respected (1s, 2s, 4s)
- Processing completes after retries

**Acceptance Criteria Reference**: AC-803.1, AC-803.2, AC-803.3

---

### E2E-ERR-005: Max Retries Exhausted

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-ERR-005 |
| **Title** | Max Retries Exhausted |
| **Priority** | P1 - High |
| **FR References** | FR-803 |
| **Preconditions** | MCP server configured to always fail |

**Test Steps**:

1. Configure MCP mock to fail all requests
2. Start processing
3. Verify RETRY_1, RETRY_2, RETRY_3 states
4. Verify ERROR state after max retries
5. Verify error message indicates retries exhausted

**Expected Results**:

- All 3 retry states traversed
- Final ERROR state reached
- Clear error message

**Acceptance Criteria Reference**: FR-803 State Transitions table

---

### E2E-ERR-006: Job Cancellation

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-ERR-006 |
| **Title** | Job Cancellation |
| **Priority** | P1 - High |
| **FR References** | FR-406 |
| **Preconditions** | Processing in progress |

**Test Steps**:

1. Start processing with larger file
2. Verify Cancel button visible during PROCESSING state
3. Click Cancel button
4. Verify job transitions to CANCELLED
5. Verify SSE sends `cancelled` event
6. Verify partial results cleaned up (or preserved per config)

**Expected Results**:

- Cancel button visible during processing
- Job status becomes CANCELLED
- Processing stops
- UI returns to ready state

**Acceptance Criteria Reference**: AC-406.1, AC-406.2, AC-406.3, AC-406.4

---

### E2E-ERR-007: Partial Result Handling

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-ERR-007 |
| **Title** | Partial Result Handling |
| **Priority** | P1 - High |
| **FR References** | FR-405 |
| **Preconditions** | Configure export_html to fail while others succeed |

**Test Steps**:

1. Configure MCP mock: convert succeeds, export_markdown succeeds, export_html fails, export_json succeeds
2. Start processing
3. Verify job status becomes PARTIAL_COMPLETE
4. Verify successful exports (markdown, json) are displayed
5. Verify failed export (html) shows error indicator
6. Verify UI indicates which exports failed

**Expected Results**:

- PARTIAL_COMPLETE status shown
- Available results displayed
- Clear indication of failed export

**Acceptance Criteria Reference**: AC-405.1, AC-405.2, AC-405.3

---

## 9. Session Management Tests

### E2E-SM-001: Session Creation

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-SM-001 |
| **Title** | Session Creation |
| **Priority** | P1 - High |
| **FR References** | Session Schema (5.4) |
| **Preconditions** | No existing session |

**Test Steps**:

1. Clear cookies
2. Navigate to application
3. Verify session cookie created
4. Verify session ID is UUID format

**Expected Results**:

- Session cookie set automatically
- Valid UUID as session ID

---

### E2E-SM-002: Session Persistence Across Pages

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-SM-002 |
| **Title** | Session Persistence Across Pages |
| **Priority** | P1 - High |
| **FR References** | Session Schema (5.4) |
| **Preconditions** | Active session |

**Test Steps**:

1. Navigate to home page
2. Complete a processing job
3. Navigate to History
4. Verify session maintained (same job visible)
5. Navigate back home
6. Verify session still valid

**Expected Results**:

- Same session across all navigation
- Data persists within session

---

### E2E-SM-003: Browser Refresh Recovery

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-SM-003 |
| **Title** | Browser Refresh Recovery |
| **Priority** | P1 - High |
| **FR References** | NFR-203, Zustand Store (5.3) |
| **Preconditions** | Processing in progress |

**Test Steps**:

1. Start processing with larger file
2. Wait for progress to reach 40%
3. Refresh browser (F5)
4. Verify application recovers
5. Verify reconnection to job
6. Verify processing continues to completion

**Expected Results**:

- State recovered from localStorage
- Job reconnection succeeds
- Processing completes successfully

**Acceptance Criteria Reference**: NFR-203

---

### E2E-SM-004: Session Job Isolation

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-SM-004 |
| **Title** | Session Job Isolation |
| **Priority** | P1 - High |
| **FR References** | FR-701 |
| **Preconditions** | Two browser contexts/sessions |

**Test Steps**:

1. Open application in two browser contexts (incognito)
2. Create job in session A
3. Create job in session B
4. View history in session A
5. Verify only session A jobs visible
6. View history in session B
7. Verify only session B jobs visible

**Expected Results**:

- Complete session isolation
- No cross-session data leakage

**Acceptance Criteria Reference**: AC-701.2

---

## 10. Accessibility Tests

### E2E-A11Y-001: Full Page Accessibility Audit

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-A11Y-001 |
| **Title** | Full Page Accessibility Audit |
| **Priority** | P1 - High |
| **FR References** | NFR-501, NFR-502 |
| **Preconditions** | All pages accessible |

**Test Steps**:

For each page (Home, History, Results):

1. Navigate to page
2. Run axe-core audit with WCAG 2.1 AA rules
3. Capture violations
4. Verify zero violations

**Expected Results**:

- WCAG 2.1 AA compliance on all pages
- Lighthouse accessibility score >= 90

**Acceptance Criteria Reference**: NFR-501, NFR-502

---

### E2E-A11Y-002: Keyboard Navigation

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-A11Y-002 |
| **Title** | Keyboard Navigation |
| **Priority** | P1 - High |
| **FR References** | Keyboard Navigation (9.3) |
| **Preconditions** | Application loaded |

**Test Steps**:

1. Use Tab to navigate through all interactive elements
2. Verify logical focus order (upload -> URL -> process)
3. Use Enter/Space to activate buttons
4. Use Escape to close modals
5. Use Arrow keys to navigate tabs
6. Verify focus visible indicator on all elements

**Expected Results**:

- All elements keyboard accessible
- Logical focus order
- Clear focus indicators

---

### E2E-A11Y-003: Screen Reader Compatibility

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-A11Y-003 |
| **Title** | Screen Reader Compatibility |
| **Priority** | P1 - High |
| **FR References** | Accessibility (Section 9) |
| **Preconditions** | NVDA/VoiceOver available |

**Test Steps**:

1. Enable screen reader
2. Navigate through application
3. Verify all interactive elements announced
4. Verify progress updates announced (debounced)
5. Verify error announcements (assertive)
6. Verify form labels read correctly

**Expected Results**:

- All content accessible to screen reader
- Meaningful announcements
- No duplicate/overlapping speech

---

### E2E-A11Y-004: Color Contrast Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-A11Y-004 |
| **Title** | Color Contrast Validation |
| **Priority** | P1 - High |
| **FR References** | NFR-503 |
| **Preconditions** | Light and dark mode available |

**Test Steps**:

1. Run color-contrast axe rule in light mode
2. Toggle to dark mode
3. Run color-contrast axe rule in dark mode
4. Verify 4.5:1 ratio for text
5. Verify 3:1 ratio for UI components

**Expected Results**:

- Zero contrast violations in both modes

**Acceptance Criteria Reference**: NFR-503

---

### E2E-A11Y-005: Reduced Motion Support

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-A11Y-005 |
| **Title** | Reduced Motion Support |
| **Priority** | P2 - Medium |
| **FR References** | Reduced Motion (9.7) |
| **Preconditions** | prefers-reduced-motion enabled |

**Test Steps**:

1. Enable prefers-reduced-motion in browser
2. Navigate application
3. Verify animations disabled/reduced
4. Verify progress bar still functions (instant transitions)

**Expected Results**:

- Animations respect user preference
- Functionality preserved

---

## 11. Performance Tests

### E2E-PERF-001: Page Load Performance

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-PERF-001 |
| **Title** | Page Load Performance |
| **Priority** | P1 - High |
| **FR References** | NFR-101, NFR-102 |
| **Preconditions** | Production build, cache cleared |

**Test Steps**:

1. Clear browser cache
2. Navigate to home page
3. Measure LCP (Largest Contentful Paint)
4. Measure FCP (First Contentful Paint)
5. Verify LCP < 2.5s
6. Verify FCP < 1.8s

**Expected Results**:

- LCP < 2.5 seconds
- FCP < 1.8 seconds

**Acceptance Criteria Reference**: NFR-101, NFR-102

---

### E2E-PERF-002: Time to First Progress

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-PERF-002 |
| **Title** | Time to First Progress |
| **Priority** | P1 - High |
| **FR References** | NFR-103 |
| **Preconditions** | Small file ready (< 1MB) |

**Test Steps**:

1. Upload small file
2. Start timer at Process button click
3. Measure time to first SSE progress event
4. Verify < 500ms

**Expected Results**:

- First progress event within 500ms

**Acceptance Criteria Reference**: NFR-103

---

### E2E-PERF-003: Bundle Size Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-PERF-003 |
| **Title** | Bundle Size Validation |
| **Priority** | P1 - High |
| **FR References** | NFR-105 |
| **Preconditions** | Production build available |

**Test Steps**:

1. Build production bundle
2. Analyze initial JavaScript bundle size
3. Verify < 100KB (excluding React runtime)

**Expected Results**:

- Initial JS bundle < 100KB

**Acceptance Criteria Reference**: NFR-105

---

### 11.1 Load Testing Scenarios

Load testing validates system behavior under varying levels of concurrent user load, stress conditions, and sustained throughput. These tests use Locust for HTTP load generation and measure response times, throughput, error rates, and resource utilization against defined SLAs.

### E2E-LOAD-001: Concurrent User Baseline (10 Users)

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-LOAD-001 |
| **Title** | Concurrent User Baseline (10 Users) |
| **Priority** | P1 - High |
| **Load Profile** | 10 concurrent users, 5 minute duration, 1 user/sec spawn rate |
| **FR References** | NFR-101, NFR-103, Section 5.1.3 (Connection Pool) |
| **Preconditions** | Production build deployed, database connection pool configured (5 connections) |

**Test Steps (Locust Configuration)**:

```python
# locustfile.py - Baseline load test
from locust import HttpUser, task, between

class DoclingUser(HttpUser):
    wait_time = between(5, 15)  # 5-15s think time

    @task(3)
    def upload_and_process_pdf(self):
        """Upload small PDF and process - 60% of load"""
        with open('test-files/small.pdf', 'rb') as f:
            self.client.post('/api/v1/upload',
                files={'file': ('small.pdf', f, 'application/pdf')})

    @task(2)
    def view_history(self):
        """View history page - 40% of load"""
        self.client.get('/history')

    @task(1)
    def download_result(self):
        """Download markdown result - 20% of load"""
        self.client.get('/api/v1/jobs/test-job-id/download?format=markdown')

# Run: locust -f locustfile.py --users 10 --spawn-rate 1 --run-time 5m
```

**Expected Results (SLA Thresholds)**:

- **Response Time**: p50 < 200ms, p95 < 500ms, p99 < 1000ms
- **Throughput**: >= 5 requests/sec sustained
- **Error Rate**: < 1% (4xx/5xx responses)
- **Resource Utilization**: CPU < 50%, Memory < 70%, DB connections < 5/5
- **SSE Connection Stability**: 100% successful connections

**Acceptance Criteria**: All SLA thresholds met, zero service crashes, database connection pool stable.

---

### E2E-LOAD-002: Concurrent File Uploads (10 Simultaneous)

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-LOAD-002 |
| **Title** | Concurrent File Uploads (10 Simultaneous) |
| **Priority** | P1 - High |
| **Load Profile** | 10 concurrent uploads, 50MB files each, 10 user spawn rate |
| **FR References** | FR-104, Section 5.1.4 (Result Size Validation) |
| **Preconditions** | 10x 50MB test PDF files available, temp storage cleared |

**Test Steps (Locust Configuration)**:

```python
from locust import HttpUser, task, constant_pacing
import os

class UploadStressUser(HttpUser):
    wait_time = constant_pacing(1)  # 1 upload per second per user

    @task
    def upload_large_file(self):
        """Upload 50MB PDF file"""
        file_path = f'test-files/large-{self.environment.runner.user_count}.pdf'
        with open(file_path, 'rb') as f:
            with self.client.post('/api/v1/upload',
                files={'file': (os.path.basename(file_path), f, 'application/pdf')},
                catch_response=True) as response:
                if response.status_code == 201:
                    response.success()
                elif response.status_code == 413:
                    response.failure("Payload too large")
                else:
                    response.failure(f"Unexpected status: {response.status_code}")

# Run: locust -f upload_stress.py --users 10 --spawn-rate 10 --run-time 3m
```

**Expected Results (SLA Thresholds)**:

- **Upload Response Time**: p95 < 30s for 50MB files
- **Throughput**: >= 10 concurrent uploads without queueing
- **Error Rate**: < 5% (allow for transient failures)
- **Disk I/O**: Sustained write speed >= 100 MB/s
- **Temp Storage**: No disk space exhaustion, cleanup after processing
- **Memory**: No memory leaks, stable heap usage

**Acceptance Criteria**: All 10 uploads complete successfully, no file corruption, temp files cleaned up post-processing.

---

### E2E-LOAD-003: SSE Connection Scaling (50 Connections)

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-LOAD-003 |
| **Title** | SSE Connection Scaling (50 Concurrent Connections) |
| **Priority** | P1 - High |
| **Load Profile** | 50 concurrent SSE connections, 10 minute duration, 2 user/sec spawn rate |
| **FR References** | FR-501, FR-502, FR-503, NFR-104 |
| **Preconditions** | Redis pub/sub configured, SSE endpoint available |

**Test Steps (Locust Configuration)**:

```python
from locust import HttpUser, task, constant
import sseclient

class SSEConnectionUser(HttpUser):
    wait_time = constant(0)  # Keep connection open

    @task
    def maintain_sse_connection(self):
        """Maintain long-lived SSE connection"""
        response = self.client.get('/api/v1/process/test-job-id/events',
            stream=True,
            timeout=600,  # 10 minute timeout
            catch_response=True)

        try:
            client = sseclient.SSEClient(response)
            event_count = 0
            for event in client.events():
                event_count += 1
                if event_count >= 100:  # Expect 100 progress events
                    response.success()
                    break
        except Exception as e:
            response.failure(f"SSE error: {str(e)}")

# Run: locust -f sse_load.py --users 50 --spawn-rate 2 --run-time 10m
```

**Expected Results (SLA Thresholds)**:

- **Connection Establishment**: < 200ms per connection
- **Reconnection Time**: < 5s on network interruption (NFR-104)
- **Message Delivery Latency**: < 100ms from publish to delivery
- **Connection Stability**: >= 95% uptime per connection over 10 minutes
- **Redis Pub/Sub**: Sustained 5000 messages/sec (50 connections * 100 events/min)
- **Memory per Connection**: < 10KB overhead per SSE connection

**Acceptance Criteria**: 50 concurrent SSE connections maintained, all progress events delivered in order, zero message loss.

---

### E2E-LOAD-004: API Rate Limit Stress Test

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-LOAD-004 |
| **Title** | API Rate Limit Enforcement Under Load |
| **Priority** | P1 - High |
| **Load Profile** | 100 requests/sec burst, 2 minute duration |
| **FR References** | NFR-302, FR-801 (Rate Limit Error E601) |
| **Preconditions** | Rate limiter configured: 60 req/min per IP |

**Test Steps (Locust Configuration)**:

```python
from locust import HttpUser, task, constant_pacing

class RateLimitStressUser(HttpUser):
    wait_time = constant_pacing(0.6)  # 100 req/sec across users

    @task
    def burst_requests(self):
        """Generate burst traffic to trigger rate limiting"""
        with self.client.get('/api/v1/jobs',
            catch_response=True) as response:
            if response.status_code == 200:
                response.success()
            elif response.status_code == 429:
                # Expected - rate limit hit
                retry_after = response.headers.get('Retry-After', 'unknown')
                response.success()  # Mark as success - this is expected behavior
            else:
                response.failure(f"Unexpected: {response.status_code}")

# Run: locust -f rate_limit_test.py --users 100 --spawn-rate 10 --run-time 2m
```

**Expected Results (SLA Thresholds)**:

- **Rate Limit Enforcement**: 429 status returned when limit exceeded (60 req/min)
- **Retry-After Header**: Present in all 429 responses
- **Error Message**: E601 returned with clear user-facing message
- **Service Stability**: No service degradation or crashes under burst load
- **Legitimate Traffic**: Requests within rate limit receive 200 responses
- **Recovery**: Service handles traffic normally after burst ends

**Acceptance Criteria**: Rate limiting enforced correctly, 429 responses include Retry-After header, no service instability.

---

### E2E-LOAD-005: Database Connection Pool Exhaustion

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-LOAD-005 |
| **Title** | Database Connection Pool Exhaustion Test |
| **Priority** | P1 - High |
| **Load Profile** | 30 concurrent users, 5 minute duration (exceeds pool size of 25) |
| **FR References** | Section 5.1.3 (Connection Pool Configuration) |
| **Preconditions** | Connection pool configured: max 25 connections, queue timeout 10s |

**Test Steps (Locust Configuration)**:

```python
from locust import HttpUser, task, between
import time

class DatabaseStressUser(HttpUser):
    wait_time = between(1, 3)

    @task(5)
    def query_history(self):
        """Database-heavy history queries"""
        with self.client.get('/api/v1/jobs?limit=50&offset=0',
            catch_response=True) as response:
            start_time = time.time()
            if response.status_code == 200:
                query_time = time.time() - start_time
                if query_time > 2.0:
                    response.failure(f"Query too slow: {query_time}s")
                else:
                    response.success()
            elif response.status_code == 503:
                response.failure("Database connection pool exhausted")

    @task(1)
    def create_job(self):
        """Create new job (INSERT operation)"""
        self.client.post('/api/v1/upload',
            json={'url': 'https://example.com/test.pdf'})

# Run: locust -f db_pool_test.py --users 30 --spawn-rate 3 --run-time 5m
```

**Expected Results (SLA Thresholds)**:

- **Connection Pool Utilization**: Peak usage 25/25 connections
- **Queue Wait Time**: p95 < 2s, p99 < 5s (connections queued when pool full)
- **Query Response Time**: p95 < 500ms under pool contention
- **Error Rate**: < 1% connection timeout errors
- **Pool Recovery**: Connections returned to pool after query completion
- **No Connection Leaks**: Pool size stable throughout test

**Acceptance Criteria**: Connection pool handles contention gracefully, no connection leaks, query timeouts < 1%.

---

### E2E-LOAD-006: Redis Pub/Sub High Throughput

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-LOAD-006 |
| **Title** | Redis Pub/Sub High Message Throughput |
| **Priority** | P1 - High |
| **Load Profile** | 100 concurrent jobs, 100 progress events each (10,000 messages), 5 minute duration |
| **FR References** | FR-502, Section 5.4 (Redis Pub/Sub) |
| **Preconditions** | Redis cluster configured, pub/sub channels ready |

**Test Steps (Locust Configuration)**:

```python
from locust import HttpUser, task, constant
import json

class PubSubLoadUser(HttpUser):
    wait_time = constant(1)

    @task
    def simulate_processing_with_events(self):
        """Simulate job processing with 100 progress events"""
        # Upload and start processing
        upload_response = self.client.post('/api/v1/upload',
            files={'file': ('test.pdf', open('test-files/small.pdf', 'rb'))})
        job_id = upload_response.json().get('jobId')

        # Connect to SSE stream
        with self.client.get(f'/api/v1/process/{job_id}/events',
            stream=True,
            timeout=300,
            catch_response=True) as response:

            events_received = 0
            for line in response.iter_lines():
                if line.startswith(b'data:'):
                    events_received += 1
                    if events_received >= 100:
                        response.success()
                        break

# Run: locust -f pubsub_load.py --users 100 --spawn-rate 5 --run-time 5m
```

**Expected Results (SLA Thresholds)**:

- **Message Throughput**: >= 10,000 messages/minute sustained
- **Publish Latency**: p95 < 10ms (Redis PUBLISH command)
- **Subscribe Latency**: p95 < 50ms (message delivery to subscriber)
- **Message Loss**: 0% (all published messages delivered)
- **Redis Memory**: Stable (no unbounded queue growth)
- **Connection Stability**: 100% SSE connection uptime

**Acceptance Criteria**: All 10,000 progress events delivered successfully, zero message loss, Redis memory stable.

---

### E2E-LOAD-007: MCP Tool Invocation Under Load

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-LOAD-007 |
| **Title** | MCP Tool Invocation Concurrency Test |
| **Priority** | P1 - High |
| **Load Profile** | 20 concurrent PDF conversions, 3 minute duration |
| **FR References** | FR-402 (MCP Tools), Section 4.3 (Process Endpoint) |
| **Preconditions** | hx-docling-mcp-server running, 20x test PDF files available |

**Test Steps (Locust Configuration)**:

```python
from locust import HttpUser, task, between
import random

class MCPLoadUser(HttpUser):
    wait_time = between(10, 20)

    @task
    def process_pdf_via_mcp(self):
        """Upload and process PDF using MCP convert_pdf tool"""
        file_name = f'test-{random.randint(1, 20)}.pdf'

        # Upload
        with open(f'test-files/{file_name}', 'rb') as f:
            upload_resp = self.client.post('/api/v1/upload',
                files={'file': (file_name, f, 'application/pdf')})

        job_id = upload_resp.json().get('jobId')

        # Process (invokes MCP convert_pdf tool)
        with self.client.post('/api/v1/process',
            json={'jobId': job_id},
            catch_response=True,
            timeout=180) as response:
            if response.status_code == 202:
                response.success()
            elif response.status_code == 504:
                response.failure("MCP tool timeout")

# Run: locust -f mcp_load.py --users 20 --spawn-rate 2 --run-time 3m
```

**Expected Results (SLA Thresholds)**:

- **MCP Tool Response Time**: p95 < 60s for small PDFs (< 5MB)
- **Concurrent MCP Calls**: 20 simultaneous convert_pdf invocations handled
- **Timeout Rate**: < 5% (within 180s timeout per FR-104)
- **Error Rate**: < 2% (transient MCP errors allowed)
- **Resource Usage**: MCP server CPU < 80%, Memory < 90%
- **Queue Depth**: MCP request queue < 10 waiting requests

**Acceptance Criteria**: 20 concurrent MCP tool invocations succeed, timeout rate < 5%, MCP server stable.

---

### E2E-LOAD-008: Memory Leak Detection (Extended Run)

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-LOAD-008 |
| **Title** | Memory Leak Detection Under Sustained Load |
| **Priority** | P1 - High |
| **Load Profile** | 25 concurrent users, 60 minute duration, constant load |
| **FR References** | NFR-201 (Resource Cleanup), Section 5.1.4 |
| **Preconditions** | Monitoring enabled (Prometheus/Grafana), baseline memory usage recorded |

**Test Steps (Locust Configuration)**:

```python
from locust import HttpUser, task, between

class MemoryLeakDetectionUser(HttpUser):
    wait_time = between(5, 15)

    @task(3)
    def upload_process_download_cycle(self):
        """Full lifecycle to detect leaks in cleanup"""
        # Upload
        with open('test-files/medium.pdf', 'rb') as f:
            upload_resp = self.client.post('/api/v1/upload',
                files={'file': ('medium.pdf', f)})
        job_id = upload_resp.json().get('jobId')

        # Process
        self.client.post('/api/v1/process', json={'jobId': job_id})

        # Wait for completion (poll status)
        import time
        for _ in range(30):
            status_resp = self.client.get(f'/api/v1/jobs/{job_id}')
            if status_resp.json().get('status') == 'COMPLETE':
                break
            time.sleep(2)

        # Download
        self.client.get(f'/api/v1/jobs/{job_id}/download?format=markdown')

    @task(1)
    def query_history(self):
        """History queries to detect pagination leaks"""
        self.client.get('/api/v1/jobs?limit=20&offset=0')

# Run: locust -f memory_test.py --users 25 --spawn-rate 5 --run-time 60m
# Monitor: Node.js heap size, RSS memory, file descriptors
```

**Expected Results (SLA Thresholds)**:

- **Memory Growth Rate**: < 5% increase per hour (indicates stability)
- **Heap Size**: Stable within 20% variance after initial ramp-up
- **RSS Memory**: No unbounded growth, GC cycles effective
- **File Descriptors**: Stable count (no descriptor leaks)
- **Temp File Cleanup**: All temp files deleted after processing
- **Database Connections**: Pool size stable (no connection leaks)

**Measurement**:
```bash
# Monitor every 5 minutes
while true; do
  echo "$(date): Heap=$(node -e 'console.log(process.memoryUsage().heapUsed)')"
  sleep 300
done
```

**Acceptance Criteria**: Memory usage stable over 60 minutes, no unbounded growth, temp files cleaned up, GC effective.

---

### E2E-LOAD-009: CPU Utilization Under Peak Load

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-LOAD-009 |
| **Title** | CPU Utilization and Throttling Under Peak Load |
| **Priority** | P1 - High |
| **Load Profile** | 50 concurrent users, 10 minute duration, CPU-intensive workload |
| **FR References** | NFR-201, Section 5.1.3 (Connection Pool) |
| **Preconditions** | CPU monitoring enabled, baseline CPU usage < 20% |

**Test Steps (Locust Configuration)**:

```python
from locust import HttpUser, task, constant_pacing

class CPUStressUser(HttpUser):
    wait_time = constant_pacing(2)  # Constant 0.5 req/sec per user

    @task(5)
    def process_large_pdf(self):
        """CPU-intensive PDF processing"""
        with open('test-files/large-complex.pdf', 'rb') as f:
            upload_resp = self.client.post('/api/v1/upload',
                files={'file': ('complex.pdf', f)})
        job_id = upload_resp.json().get('jobId')

        # Process (CPU-intensive conversion)
        with self.client.post('/api/v1/process',
            json={'jobId': job_id},
            timeout=300,
            catch_response=True) as response:
            if response.elapsed.total_seconds() > 120:
                response.failure(f"Slow processing: {response.elapsed.total_seconds()}s")

    @task(1)
    def query_complex_history(self):
        """CPU-bound database query"""
        self.client.get('/api/v1/jobs?limit=100&offset=0&sort=createdAt')

# Run: locust -f cpu_load.py --users 50 --spawn-rate 5 --run-time 10m
```

**Expected Results (SLA Thresholds)**:

- **Average CPU Utilization**: 60-80% (efficient resource usage)
- **Peak CPU Utilization**: < 95% (avoid CPU saturation)
- **CPU Throttling**: Zero throttling events (cgroups/containers)
- **Response Time Degradation**: < 20% slowdown at peak vs baseline
- **Queue Depth**: Request queue < 5 waiting requests
- **Throughput**: >= 25 req/sec sustained (50 users * 0.5 req/sec)

**Measurement**:
```bash
# Monitor CPU with top/htop during test
top -b -n 60 -d 10 | grep node > cpu_log.txt
```

**Acceptance Criteria**: CPU utilization stable 60-80%, no throttling, response time degradation < 20%, throughput maintained.

---

### E2E-LOAD-010: Response Time Degradation Curve

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-LOAD-010 |
| **Title** | Response Time Degradation Under Increasing Load |
| **Priority** | P1 - High |
| **Load Profile** | Ramp from 10 to 100 users over 20 minutes, measure degradation |
| **FR References** | NFR-101, NFR-103, Section 5.1.3 |
| **Preconditions** | Production environment, monitoring enabled |

**Test Steps (Locust Configuration)**:

```python
from locust import HttpUser, task, between

class DegradationTestUser(HttpUser):
    wait_time = between(3, 8)

    @task(3)
    def upload_small_file(self):
        """Baseline operation - measure response time"""
        with open('test-files/small.pdf', 'rb') as f:
            with self.client.post('/api/v1/upload',
                files={'file': ('small.pdf', f)},
                catch_response=True) as response:
                if response.elapsed.total_seconds() > 5.0:
                    response.failure(f"Upload slow: {response.elapsed.total_seconds()}s")

    @task(2)
    def query_history(self):
        """Database query - measure degradation"""
        with self.client.get('/api/v1/jobs?limit=20',
            catch_response=True) as response:
            if response.elapsed.total_seconds() > 1.0:
                response.failure(f"Query slow: {response.elapsed.total_seconds()}s")

# Run with step-load:
# locust -f degradation_test.py \
#   --step-load \
#   --step-users 10 \
#   --step-time 2m \
#   --max-users 100
```

**Expected Results (SLA Thresholds)**:

| User Count | p50 Response Time | p95 Response Time | Throughput | Error Rate |
|------------|------------------|------------------|------------|------------|
| 10 users   | < 200ms          | < 500ms          | 5 req/s    | < 1%       |
| 25 users   | < 300ms          | < 800ms          | 12 req/s   | < 1%       |
| 50 users   | < 500ms          | < 1500ms         | 20 req/s   | < 2%       |
| 75 users   | < 800ms          | < 2500ms         | 25 req/s   | < 5%       |
| 100 users  | < 1200ms         | < 4000ms         | 28 req/s   | < 10%      |

**Degradation Curve Analysis**:
- **Linear Degradation**: Response time increases linearly with load (acceptable)
- **Exponential Degradation**: Response time increases exponentially (indicates bottleneck)
- **Breaking Point**: User count where error rate exceeds 10% (system capacity)

**Acceptance Criteria**: Response time degradation is linear (not exponential), breaking point > 75 users, no crashes during ramp-up.

---

## 12. Cross-Browser Tests

### E2E-XB-001: Chrome Compatibility

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-XB-001 |
| **Title** | Chrome Compatibility |
| **Priority** | P1 - High |
| **FR References** | Browser compatibility |
| **Preconditions** | Chrome latest stable |

**Test Steps**:

1. Run E2E-CP-001 through E2E-CP-004 in Chrome
2. Verify all pass

**Expected Results**:

- Full functionality in Chrome

---

### E2E-XB-002: Firefox Compatibility

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-XB-002 |
| **Title** | Firefox Compatibility |
| **Priority** | P1 - High |
| **FR References** | Browser compatibility |
| **Preconditions** | Firefox latest stable |

**Test Steps**:

1. Run E2E-CP-001 through E2E-CP-004 in Firefox
2. Verify all pass

**Expected Results**:

- Full functionality in Firefox

---

### E2E-XB-003: Safari Compatibility

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-XB-003 |
| **Title** | Safari Compatibility |
| **Priority** | P1 - High |
| **FR References** | Browser compatibility |
| **Preconditions** | Safari latest stable |

**Test Steps**:

1. Run E2E-CP-001 through E2E-CP-004 in Safari
2. Verify all pass

**Expected Results**:

- Full functionality in Safari

---

## 13. Rate Limiting Tests

### 13.1 Overview

Rate limiting tests validate that the application correctly enforces the 10 requests per minute per session limit as defined in NFR-302. These tests ensure proper rate limit header handling, 429 response behavior, and rate limit recovery after the sliding window resets.

### E2E-RL-001: Rate Limit Enforcement (10 req/min)

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-RL-001 |
| **Title** | Rate Limit Enforcement (10 req/min) |
| **Priority** | P0 - Critical |
| **FR References** | NFR-302, Error E601 |
| **Preconditions** | Fresh session with no prior requests |

**Test Steps**:

1. Create a new browser session (clear cookies)
2. Navigate to application home page
3. Send 10 consecutive file upload requests rapidly (within 60 seconds)
4. Verify all 10 requests succeed with 200/201 status
5. Send 11th upload request immediately
6. Verify 429 Too Many Requests response
7. Verify error code E601 in response body
8. Verify error message indicates rate limit exceeded
9. Verify UI displays appropriate rate limit error notification

**Expected Results**:

- First 10 requests return 200/201 (success)
- 11th request returns 429 Too Many Requests
- Response body contains E601 error code
- User-friendly message: "Rate limit exceeded. Please wait before trying again."
- Rate limit applies per session, not globally

**Acceptance Criteria Reference**: NFR-302 (Rate limit enforcement accuracy: 100%)

---

### E2E-RL-002: Rate Limit Headers Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-RL-002 |
| **Title** | Rate Limit Headers Validation |
| **Priority** | P1 - High |
| **FR References** | NFR-302, Section 4.8 (API Response Headers) |
| **Preconditions** | Fresh session |

**Test Steps**:

1. Create new session
2. Send first upload request
3. Capture response headers
4. Verify `X-RateLimit-Limit` header equals 10
5. Verify `X-RateLimit-Remaining` header equals 9
6. Verify `X-RateLimit-Reset` header contains valid Unix timestamp
7. Send 5 more requests
8. Verify `X-RateLimit-Remaining` decrements correctly (should be 4)
9. Send requests until rate limit exceeded (429 response)
10. Verify `Retry-After` header is present on 429 response
11. Verify `Retry-After` value is reasonable (1-60 seconds)

**Expected Results**:

- `X-RateLimit-Limit: 10` present on all responses
- `X-RateLimit-Remaining` decrements correctly with each request
- `X-RateLimit-Reset` contains valid future timestamp
- 429 responses include `Retry-After` header with seconds until reset
- Headers conform to RFC 9238 rate limit header format

**Acceptance Criteria Reference**: NFR-302, Section 4.8 API Headers

---

### E2E-RL-003: Rate Limit Recovery After Window

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-RL-003 |
| **Title** | Rate Limit Recovery After Window |
| **Priority** | P1 - High |
| **FR References** | NFR-302, Section 7.4.2 (Sliding Window) |
| **Preconditions** | Session at rate limit (10 requests exhausted) |

**Test Steps**:

1. Create new session
2. Send 10 upload requests rapidly to exhaust rate limit
3. Verify 11th request returns 429
4. Record `Retry-After` header value (or `X-RateLimit-Reset` timestamp)
5. Wait for rate limit window to reset (approximately 60 seconds or as indicated by Retry-After)
6. Send new upload request
7. Verify request succeeds with 200/201 status
8. Verify `X-RateLimit-Remaining` shows refreshed quota
9. Verify processing can complete successfully

**Expected Results**:

- Rate limit window resets after specified time (sliding window)
- New requests succeed after window reset
- `X-RateLimit-Remaining` shows full or refreshed quota
- Normal processing resumes without errors
- No persistent rate limit state affecting future sessions

**Acceptance Criteria Reference**: NFR-302, Section 7.4.2 (Sliding Window Rate Limiting)

---

## 14. MCP Tool Coverage Tests

### 14.1 Overview

MCP (Model Context Protocol) tool coverage tests validate that all 8 MCP tools defined in FR-402 function correctly through the end-to-end workflow. Each test validates the correct tool invocation, response handling, and output format generation for the specific document type.

### E2E-MCP-001: convert_pdf Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-MCP-001 |
| **Title** | convert_pdf Tool Validation |
| **Priority** | P0 - Critical |
| **FR References** | FR-402 (Tool Routing Table), convert_pdf tool |
| **Preconditions** | Valid PDF file with known content structure (headings, tables, text) |

**Test Steps**:

1. Upload test PDF file (structured-doc.pdf with 3 headings, 1 table, 2 paragraphs)
2. Verify file accepted and job created
3. Click Process button
4. Wait for processing completion
5. Verify convert_pdf tool was invoked (via processing logs or API response)
6. View Markdown tab
7. Verify markdown contains expected heading structure (H1, H2 tags)
8. Verify table content is extracted and formatted as markdown table
9. Verify paragraph text is preserved
10. Download all formats and verify content integrity

**Expected Results**:

- convert_pdf tool correctly invoked for .pdf files
- DoclingDocument generated with correct schema
- Markdown export preserves document structure (headings, tables, paragraphs)
- HTML export contains semantic HTML elements
- JSON export contains full DoclingDocument schema fields
- All exports downloadable with correct content

**Acceptance Criteria Reference**: FR-402 Tool Routing Table (File .pdf -> convert_pdf)

---

### E2E-MCP-002: convert_docx Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-MCP-002 |
| **Title** | convert_docx Tool Validation |
| **Priority** | P0 - Critical |
| **FR References** | FR-402 (Tool Routing Table), convert_docx tool |
| **Preconditions** | Valid DOCX file with known content |

**Test Steps**:

1. Upload test Word document (sample.docx with headings and formatted text)
2. Verify file accepted (.docx extension validated)
3. Click Process button
4. Wait for processing completion
5. Verify convert_docx tool was invoked
6. View Markdown tab
7. Verify Word formatting preserved (bold, italic, headings)
8. Verify lists are converted correctly
9. Download markdown and verify content matches source structure
10. Verify all export formats generated

**Expected Results**:

- convert_docx tool correctly invoked for .docx files
- Word document structure preserved in output
- Formatting (bold, italic, lists) converted to markdown equivalents
- All four export formats available and correct
- Processing completes within timeout (180s for Word files)

**Acceptance Criteria Reference**: FR-402 Tool Routing Table (File .docx -> convert_docx)

---

### E2E-MCP-003: convert_xlsx Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-MCP-003 |
| **Title** | convert_xlsx Tool Validation |
| **Priority** | P0 - Critical |
| **FR References** | FR-402 (Tool Routing Table), convert_xlsx tool |
| **Preconditions** | Valid Excel file with multiple sheets and tables |

**Test Steps**:

1. Upload test Excel file (sample.xlsx with 2 sheets, data tables)
2. Verify file accepted (.xlsx extension validated)
3. Click Process button
4. Wait for processing completion
5. Verify convert_xlsx tool was invoked
6. View Markdown tab
7. Verify spreadsheet data converted to markdown tables
8. Verify multiple sheets are represented in output
9. Verify numeric data and formulas rendered as values
10. Download all formats and verify tabular structure

**Expected Results**:

- convert_xlsx tool correctly invoked for .xlsx files
- Spreadsheet data converted to markdown table format
- Multiple sheets handled (concatenated or separated)
- Cell data preserved correctly
- All export formats contain tabular data
- Processing completes within timeout (180s for Excel files)

**Acceptance Criteria Reference**: FR-402 Tool Routing Table (File .xlsx -> convert_xlsx)

---

### E2E-MCP-004: convert_pptx Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-MCP-004 |
| **Title** | convert_pptx Tool Validation |
| **Priority** | P0 - Critical |
| **FR References** | FR-402 (Tool Routing Table), convert_pptx tool |
| **Preconditions** | Valid PowerPoint file with multiple slides |

**Test Steps**:

1. Upload test PowerPoint file (sample.pptx with 3 slides)
2. Verify file accepted (.pptx extension validated)
3. Click Process button
4. Wait for processing completion
5. Verify convert_pptx tool was invoked
6. View Markdown tab
7. Verify slide content converted to markdown sections
8. Verify slide titles become headings
9. Verify bullet points preserved
10. Download all formats and verify slide structure

**Expected Results**:

- convert_pptx tool correctly invoked for .pptx files
- Each slide content extracted and structured
- Slide titles converted to markdown headings
- Bullet points and text content preserved
- All export formats contain slide content
- Processing completes within timeout (180s for PowerPoint files)

**Acceptance Criteria Reference**: FR-402 Tool Routing Table (File .pptx -> convert_pptx)

---

### E2E-MCP-005: convert_url Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-MCP-005 |
| **Title** | convert_url Tool Validation |
| **Priority** | P0 - Critical |
| **FR References** | FR-201, FR-202, FR-402 (Tool Routing Table), convert_url tool |
| **Preconditions** | Valid external HTTPS URL with known HTML content |

**Test Steps**:

1. Enter valid URL (https://example.com or test endpoint with known content)
2. Verify URL validation passes (no E101 error)
3. Click Process button
4. Wait for processing completion
5. Verify convert_url tool was invoked
6. View Markdown tab
7. Verify HTML content converted to markdown
8. Verify heading tags (H1-H6) converted to markdown headings
9. Verify links preserved in markdown format
10. Download all formats and verify HTML-to-markdown fidelity

**Expected Results**:

- convert_url tool correctly invoked for URL inputs
- URL fetched and HTML parsed successfully
- HTML semantic elements converted to markdown equivalents
- Links and formatting preserved
- All export formats generated from URL content
- SSRF protection enforced (private IPs blocked)

**Acceptance Criteria Reference**: FR-402 Tool Routing Table (URL -> convert_url)

---

### E2E-MCP-006: export_markdown Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-MCP-006 |
| **Title** | export_markdown Tool Validation |
| **Priority** | P0 - Critical |
| **FR References** | FR-402, FR-602, export_markdown tool |
| **Preconditions** | Completed document conversion with DoclingDocument |

**Test Steps**:

1. Upload and process a PDF file with rich formatting
2. Wait for processing completion
3. Click Markdown tab to view results
4. Verify markdown content displays correctly
5. Verify GFM (GitHub Flavored Markdown) syntax supported
6. Verify code blocks have syntax highlighting
7. Verify tables rendered as markdown tables
8. Click Download on Markdown tab
9. Verify downloaded file has .md extension
10. Open downloaded file and verify content matches preview

**Expected Results**:

- export_markdown tool correctly generates markdown from DoclingDocument
- GFM syntax properly formatted (headings, lists, code blocks, tables)
- Markdown preview renders correctly in UI
- Downloaded .md file contains valid markdown
- Content fidelity maintained between preview and download
- File naming follows convention: {filename}_{timestamp}_markdown.md

**Acceptance Criteria Reference**: FR-602 (Markdown Rendering), AC-602.1-4

---

### E2E-MCP-007: export_html Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-MCP-007 |
| **Title** | export_html Tool Validation |
| **Priority** | P0 - Critical |
| **FR References** | FR-402, FR-603, export_html tool |
| **Preconditions** | Completed document conversion with DoclingDocument |

**Test Steps**:

1. Upload and process a PDF file
2. Wait for processing completion
3. Click HTML tab to view results
4. Verify HTML renders in sandboxed iframe
5. Verify iframe has sandbox attribute for security
6. Verify HTML contains proper semantic tags (h1-h6, p, table, ul/ol)
7. Verify content is scrollable if overflow
8. Click Download on HTML tab
9. Verify downloaded file has .html extension
10. Open downloaded file in browser and verify rendering

**Expected Results**:

- export_html tool correctly generates HTML from DoclingDocument
- HTML displayed in secure sandboxed iframe (AC-603.2)
- Semantic HTML structure preserved
- Downloaded .html file is valid HTML5
- Content renders correctly when opened in browser
- File naming follows convention: {filename}_{timestamp}_html.html

**Acceptance Criteria Reference**: FR-603 (HTML Preview), AC-603.1-3

---

### E2E-MCP-008: export_json Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-MCP-008 |
| **Title** | export_json Tool Validation |
| **Priority** | P0 - Critical |
| **FR References** | FR-402, FR-604, export_json tool |
| **Preconditions** | Completed document conversion with DoclingDocument |

**Test Steps**:

1. Upload and process a PDF file
2. Wait for processing completion
3. Click JSON tab to view results
4. Verify JSON is pretty-printed with 2-space indentation
5. Verify syntax highlighting applied (keys, values, strings colored)
6. Verify JSON contains DoclingDocument schema fields (schema_name, version, body)
7. Verify JSON is valid (can be parsed without errors)
8. Click Download on JSON tab
9. Verify downloaded file has .json extension
10. Parse downloaded JSON and verify schema compliance

**Expected Results**:

- export_json tool correctly serializes DoclingDocument to JSON
- JSON formatted with 2-space indentation (AC-604.1)
- Syntax highlighting applied for readability (AC-604.2)
- JSON conforms to DoclingDocumentSchema (schema_name, version, name, body required)
- Downloaded .json file parses without errors
- File naming follows convention: {filename}_{timestamp}_json.json

**Acceptance Criteria Reference**: FR-604 (JSON Display), AC-604.1-2, DoclingDocumentSchema

---

## 15. Database State Verification Tests

### 15.1 Overview

Database state verification tests ensure that the PostgreSQL persistence layer correctly stores and maintains data integrity throughout the application lifecycle. These tests validate that database records accurately reflect application state, preventing silent data corruption and orphaned records.

### E2E-DB-001: Job Record Persistence Verification

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-DB-001 |
| **Title** | Job Record Persistence Verification |
| **Priority** | P0 - Critical |
| **FR References** | Section 5.1 (Job Schema), FR-401, FR-403 |
| **Preconditions** | Application running with database access |

**Test Steps**:

1. Upload a valid PDF file
2. Capture jobId from upload response
3. Verify Job record created in database with status PENDING
4. Verify Job record has correct fields: sessionId, fileName, inputType='FILE', fileSize
5. Click Process button
6. Wait for status transition to PROCESSING
7. Query database and verify Job status is PROCESSING
8. Wait for processing completion
9. Query database and verify Job status is COMPLETE
10. Verify completedAt timestamp is set and is recent (within 5 seconds)
11. Verify all state transitions occurred in valid order (PENDING -> PROCESSING -> COMPLETE)

**Expected Results**:

- Job record created immediately upon file upload
- Job.status accurately reflects current processing state
- Job.sessionId matches browser session
- Job.fileName, Job.fileSize, Job.inputType correctly populated
- Job.createdAt set on creation
- Job.completedAt set only when status is COMPLETE or PARTIAL_COMPLETE
- No orphaned Job records if test fails mid-execution (transaction rollback)

**Acceptance Criteria Reference**: Section 5.1 Job Schema, AC-401.1, AC-403.1

---

### E2E-DB-002: Result Record Persistence Verification

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-DB-002 |
| **Title** | Result Record Persistence Verification |
| **Priority** | P0 - Critical |
| **FR References** | Section 5.2 (Result Schema), FR-601, FR-606 |
| **Preconditions** | Completed job with all export formats |

**Test Steps**:

1. Upload and process a PDF file to completion
2. Capture jobId from processing
3. Query database for Result records with jobId
4. Verify 4 Result records created (MARKDOWN, HTML, JSON, RAW)
5. Verify each Result has: format, content (non-empty), size (> 0)
6. Verify Result.size matches actual content length
7. Verify Result.size does not exceed format limits (MARKDOWN: 10MB, HTML: 15MB, JSON: 20MB, RAW: 25MB)
8. Download each format via UI
9. Verify downloaded content matches Result.content in database
10. Verify Result records are linked to correct Job via jobId foreign key

**Expected Results**:

- 4 Result records created for each completed Job
- Result.format enum: MARKDOWN, HTML, JSON, RAW
- Result.content contains non-empty export data
- Result.size accurately reflects content size in bytes
- Result records correctly linked via jobId foreign key
- CASCADE delete: deleting Job deletes associated Results
- Content downloaded matches database record exactly

**Acceptance Criteria Reference**: Section 5.2 Result Schema, AC-601.1, AC-606.1-3

---

### E2E-DB-003: Test Data Isolation Between Runs

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-DB-003 |
| **Title** | Test Data Isolation Between Runs |
| **Priority** | P0 - Critical |
| **FR References** | Test Database Strategy, Session Isolation (FR-701) |
| **Preconditions** | Isolated test database environment |

**Test Steps**:

1. Execute E2E test run A, create 3 jobs
2. Record jobIds from test run A
3. Execute global test cleanup/teardown
4. Verify all test run A jobs and results deleted from database
5. Execute E2E test run B
6. Verify test run B starts with clean database state (no test run A data)
7. Create new jobs in test run B
8. Verify no data leakage from test run A
9. Verify session isolation: jobs from different sessions not visible to each other
10. Execute final cleanup

**Expected Results**:

- Test database isolated from development/production databases
- Global teardown removes all test data (Jobs, Results)
- Each test run starts with clean state
- No data leakage between test runs
- Session isolation enforced at database query level
- Test failures do not leave orphaned data (cleanup runs regardless)

**Acceptance Criteria Reference**: Test Database Configuration, Session Isolation (AC-701.2)

---

### E2E-DB-004: Transaction Rollback on Failure

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-DB-004 |
| **Title** | Transaction Rollback on Failure |
| **Priority** | P0 - Critical |
| **FR References** | Section 10.9 (Transaction Handling), FR-405 |
| **Preconditions** | MCP mock configured to fail during export |

**Test Steps**:

1. Configure MCP mock to fail export_html tool (simulate partial failure)
2. Upload a valid PDF file
3. Capture jobId from response
4. Click Process button
5. Wait for processing to reach export stage
6. Verify export_html fails, other exports succeed
7. Query database for Job status
8. Verify Job status is PARTIAL_COMPLETE (not ERROR)
9. Query Result records for jobId
10. Verify successful exports (MARKDOWN, JSON, RAW) have Result records
11. Verify failed export (HTML) has no Result record OR has error indicator
12. Verify no orphaned partial data from failed export
13. Verify rollback did not affect successful exports

**Expected Results**:

- Partial failure results in PARTIAL_COMPLETE status (not full rollback)
- Successful exports committed to database
- Failed exports do not leave partial/corrupt Result records
- Transaction boundaries correctly implemented per operation
- Database remains in consistent state after partial failure
- UI correctly indicates which exports succeeded/failed

**Acceptance Criteria Reference**: Section 10.9 Transaction Handling, AC-405.1-3

---

## 16. Checkpoint and Resume Tests

### 16.1 Overview

Checkpoint and resume tests validate the durable execution capability that allows interrupted jobs to resume from their last successful checkpoint stage. This is critical for reliability when processing large files or handling system restarts.

### E2E-CKPT-001: Save Checkpoint on Stage Completion

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-CKPT-001 |
| **Title** | Save Checkpoint on Stage Completion |
| **Priority** | P0 - Critical |
| **FR References** | Section 5.5 (Checkpoint Schema), CRI-08 |
| **Preconditions** | Large file (>10MB) for extended processing time |

**Test Steps**:

1. Upload large PDF file (15MB+ for 180s timeout tier)
2. Capture jobId from response
3. Click Process button
4. Monitor SSE progress events
5. Wait for progress to reach 'conversion' stage (~40%)
6. Query checkpoint storage for jobId (Redis or database)
7. Verify checkpoint exists with stage='converted' or 'uploading'
8. Wait for progress to reach 'export' stage (~80%)
9. Verify checkpoint updated with stage='export_markdown' or similar
10. Wait for processing completion
11. Verify final checkpoint stage='completed'
12. Verify checkpoint contains: jobId, stage, timestamp, checksum

**Expected Results**:

- Checkpoint saved at each stage transition: uploaded, converted, export_markdown, export_html, export_json, completed
- Checkpoint contains jobId, current stage, timestamp, data checksum
- Checkpoint storage (Redis) accessible for validation
- Checkpoint TTL set (24 hours per specification)
- Checkpoint overwritten on each stage completion (not accumulated)
- Final 'completed' checkpoint allows verification of successful completion

**Acceptance Criteria Reference**: Section 5.5 Checkpoint Schema, CRI-08

---

### E2E-CKPT-002: Resume from Checkpoint After Failure

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-CKPT-002 |
| **Title** | Resume from Checkpoint After Failure |
| **Priority** | P0 - Critical |
| **FR References** | Section 5.5, POST /api/v1/jobs/{id}/resume, E703-E706 |
| **Preconditions** | Job interrupted after conversion checkpoint |

**Test Steps**:

1. Upload large PDF file (multi-page.pdf, 15MB)
2. Capture jobId from response
3. Click Process button
4. Wait for progress to reach conversion stage (~40-50%)
5. Simulate application failure (stop backend server or kill process)
6. Verify checkpoint exists for jobId with stage='converted'
7. Restart application
8. Navigate to job detail or history
9. Verify job shows as resumable (status indicates interrupted)
10. Click Resume button or call POST /api/v1/jobs/{id}/resume
11. Verify resume request returns 200 OK
12. Verify processing continues from checkpoint (not from beginning)
13. Verify progress starts from checkpoint stage (not 0%)
14. Wait for processing completion
15. Verify all export formats generated correctly

**Expected Results**:

- Resume endpoint accepts job with valid checkpoint
- Processing resumes from checkpointed stage, not from start
- Progress indicator shows resumption from checkpoint percentage
- No duplicate work performed for stages before checkpoint
- Final results identical to uninterrupted processing
- Resume respects original job configuration (file, session, etc.)

**Acceptance Criteria Reference**: Section 5.5, POST /api/v1/jobs/{id}/resume endpoint

---

### E2E-CKPT-003: Checkpoint Expiration Handling

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-CKPT-003 |
| **Title** | Checkpoint Expiration Handling |
| **Priority** | P1 - High |
| **FR References** | Section 5.5 (Checkpoint TTL), Error E704 |
| **Preconditions** | Job with expired checkpoint (simulate or use time manipulation) |

**Test Steps**:

1. Create a job checkpoint with timestamp older than 24 hours (mock/fixture)
2. Set checkpoint stage='converted' with expired timestamp
3. Attempt to resume job via POST /api/v1/jobs/{id}/resume
4. Verify response is error with code E704
5. Verify error message indicates checkpoint expired
6. Verify UI displays appropriate error message
7. Verify job status updated to reflect unresumable state
8. Verify option to restart processing from beginning is offered
9. Click restart option
10. Verify processing starts from stage 0% (not from checkpoint)

**Expected Results**:

- Resume request for expired checkpoint returns error E704
- Error message: "Checkpoint expired. Please restart processing."
- Checkpoint TTL of 24 hours enforced
- Expired checkpoints do not allow resume
- User presented with option to restart from beginning
- Restarting creates new job or resets existing job to PENDING

**Acceptance Criteria Reference**: Section 5.5 Checkpoint TTL, Error E704

---

### E2E-CKPT-004: Resume After Browser Crash

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-CKPT-004 |
| **Title** | Resume After Browser Crash |
| **Priority** | P1 - High |
| **FR References** | NFR-203 (Session Recovery), Section 5.3 (Zustand Store), E2E-SM-003 |
| **Preconditions** | Job processing with checkpoint saved |

**Test Steps**:

1. Upload large PDF file and start processing
2. Wait for progress to reach 40%
3. Verify checkpoint saved (query checkpoint storage)
4. Close browser abruptly (simulate crash by closing tab/window)
5. Open new browser window
6. Navigate to application
7. Verify session restored from localStorage/cookie
8. Navigate to History page
9. Verify interrupted job visible with status indicating resumable
10. Click on job to view details
11. Click Resume button
12. Verify processing continues from checkpoint
13. Verify SSE reconnection establishes
14. Wait for completion
15. Verify results are complete and correct

**Expected Results**:

- Session ID preserved after browser crash (cookie or localStorage)
- Interrupted job recoverable from History view
- Job state accurately reflects checkpoint status
- Resume functionality works after browser restart
- SSE connection re-established for resumed job
- Final results identical to uninterrupted processing
- No duplicate job created on browser restart

**Acceptance Criteria Reference**: NFR-203, E2E-SM-003 (Browser Refresh Recovery), Checkpoint System

---

## 17. Security Tests

### 17.1 Overview

Security tests validate that the application properly protects against common web vulnerabilities including XSS, content-type mismatch attacks, and CSP violations. These tests complement the SSRF prevention tests in Section 4 (E2E-URL-003) to provide comprehensive security coverage.

### E2E-SEC-001: XSS Prevention in URL Input

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-SEC-001 |
| **Title** | XSS Prevention in URL Input |
| **Priority** | P1 - High (Security) |
| **FR References** | FR-202, NFR-401 |
| **Preconditions** | Application home page loaded |

**Test Steps**:

1. Navigate to URL input field
2. Enter URL with script injection: `https://example.com/<script>alert('xss')</script>`
3. Blur field to trigger validation
4. Verify validation error E101 displayed
5. Enter URL with event handler injection: `https://example.com/" onload="alert('xss')`
6. Verify validation error displayed
7. Enter URL with javascript protocol: `javascript:alert('xss')`
8. Verify validation rejects non-HTTP/HTTPS protocols
9. Attempt form submission
10. Verify no JavaScript execution occurs in any scenario

**Expected Results**:

- All XSS payload URLs rejected with E101 validation error
- Non-HTTP/HTTPS protocols blocked
- No script execution in browser
- Input properly sanitized before display
- Process button remains disabled for invalid URLs

**Acceptance Criteria Reference**: FR-202.2, NFR-401

---

### E2E-SEC-002: File Magic Byte Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-SEC-002 |
| **Title** | File Magic Byte Validation |
| **Priority** | P1 - High (Security) |
| **FR References** | FR-103, NFR-401 |
| **Preconditions** | Prepared test files with mismatched extensions/content |

**Test Steps**:

1. Create file with .pdf extension but containing GIF magic bytes (47 49 46 38)
2. Attempt to upload the file
3. Verify rejection with error E004 (file corrupted/invalid)
4. Create file with .docx extension but containing executable magic bytes (4D 5A)
5. Attempt to upload the file
6. Verify rejection with appropriate error
7. Create file with .png extension but containing HTML content
8. Attempt to upload the file
9. Verify rejection with E004 error
10. Verify processing never initiates for any invalid file

**Expected Results**:

- Files with mismatched magic bytes and extensions rejected
- Error code E004 displayed: "File corrupted or invalid format"
- No processing initiated for rejected files
- Security logging captures validation failures
- Upload zone returns to ready state after rejection

**Acceptance Criteria Reference**: FR-103.2, Section 10.1.1 (Security Tests)

---

### E2E-SEC-003: CSP Header Enforcement

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-SEC-003 |
| **Title** | CSP Header Enforcement |
| **Priority** | P1 - High (Security) |
| **FR References** | Section 8.2 (Content Security Policy), NFR-401 |
| **Preconditions** | Application loaded |

**Test Steps**:

1. Navigate to application home page
2. Capture HTTP response headers
3. Verify `Content-Security-Policy` header present
4. Verify CSP includes `default-src 'self'`
5. Verify CSP includes `script-src 'self'` (no unsafe-inline/unsafe-eval in production)
6. Verify CSP includes `connect-src 'self'` for API and SSE
7. Inject inline script via browser console targeting document
8. Verify CSP blocks execution
9. Verify browser console shows CSP violation
10. Complete a full processing flow
11. Verify all resources load correctly under CSP

**Expected Results**:

- Content-Security-Policy header present on all responses
- CSP policy restricts script sources appropriately
- Inline script injection blocked by CSP
- CSP violation reported in browser console
- Application functions correctly within CSP constraints
- SSE connections allowed via connect-src directive

**Acceptance Criteria Reference**: Section 8.2, NFR-401

---

### E2E-SEC-004: Content-Type Validation on Upload

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-SEC-004 |
| **Title** | Content-Type Validation on Upload |
| **Priority** | P1 - High (Security) |
| **FR References** | FR-103, Section 4.2 (Upload Endpoint) |
| **Preconditions** | Application home page loaded |

**Test Steps**:

1. Upload valid PDF with Content-Type: application/pdf
2. Verify upload succeeds
3. Intercept upload request and modify Content-Type header to text/html
4. Attempt upload with mismatched Content-Type
5. Verify server validates Content-Type against file extension
6. Attempt upload with no Content-Type header
7. Verify server infers type from file extension and validates
8. Attempt upload with Content-Type: application/x-executable
9. Verify rejection with appropriate error
10. Verify all Content-Type mismatches are logged

**Expected Results**:

- Valid Content-Type headers accepted
- Mismatched Content-Type headers rejected or corrected server-side
- Missing Content-Type inferred from file extension
- Dangerous Content-Types (executable, script) rejected
- Server validates against allowlist: application/pdf, application/msword, application/vnd.openxmlformats-*, image/png, image/jpeg, image/tiff

**Acceptance Criteria Reference**: FR-103.1, Section 4.2

---

## 18. State Machine Edge Cases

### 18.1 Overview

State machine edge case tests validate the job state machine transitions beyond the happy path scenarios. These tests ensure the system correctly handles cancellation from all valid states, error transitions, and concurrent state change attempts as defined in Section 5.1.2 (MAJ-DB-002).

### E2E-STM-001: PENDING to CANCELLED Transition

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-STM-001 |
| **Title** | PENDING to CANCELLED Transition |
| **Priority** | P1 - High |
| **FR References** | FR-406, Section 5.1.2 (State Machine) |
| **Preconditions** | Application home page loaded, file selected |

**Test Steps**:

1. Select a file for upload (file appears in upload zone)
2. Verify job is created in PENDING state (via UI or API check)
3. Verify Cancel button is visible
4. Click Cancel button before clicking Process
5. Verify immediate state transition to CANCELLED
6. Verify no SSE connection was established
7. Verify no partial data remains in system
8. Verify upload zone returns to ready state (empty)
9. Verify file can be selected again for new job

**Expected Results**:

- State transition: PENDING -> CANCELLED
- No upload initiated
- No SSE events beyond potential 'cancelled' confirmation
- Clean state reset in UI
- No orphaned job records in database
- System ready for new job immediately

**Acceptance Criteria Reference**: FR-406, AC-406.1, State Machine (5.1.2)

---

### E2E-STM-002: UPLOADING to ERROR Transition

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-STM-002 |
| **Title** | UPLOADING to ERROR Transition |
| **Priority** | P1 - High |
| **FR References** | FR-401, Section 5.1.2 (State Machine) |
| **Preconditions** | Large file (>50MB) prepared for extended upload time |

**Test Steps**:

1. Upload large file to extend upload duration
2. Verify job transitions to UPLOADING state
3. Verify progress shows upload stage (0-10%)
4. Simulate upload failure (network disconnect during upload)
5. Verify job transitions to ERROR state (not RETRY)
6. Verify error code displayed (E003 or E501)
7. Verify error message indicates upload failure
8. Verify "Try Again" option available
9. Verify partial upload data cleaned up
10. Click "Try Again" and verify new job can start

**Expected Results**:

- State transition: UPLOADING -> ERROR
- Upload failure detected promptly
- Clear error message displayed
- No retry attempted for upload failures (retries only for MCP failures)
- Partial upload data not persisted
- Recovery option available to user

**Acceptance Criteria Reference**: FR-401, Section 5.1.2

---

### E2E-STM-003: Invalid State Transition Rejection

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-STM-003 |
| **Title** | Invalid State Transition Rejection |
| **Priority** | P1 - High |
| **FR References** | Section 5.1.2, E701 |
| **Preconditions** | Job in COMPLETE state |

**Test Steps**:

1. Complete a processing job to COMPLETE state
2. Capture the jobId
3. Attempt to POST /api/v1/process with completed jobId
4. Verify 409 Conflict response returned
5. Verify error code E701 in response body
6. Attempt to POST /api/v1/jobs/{id}/cancel with completed jobId
7. Verify rejection (cannot cancel completed job)
8. Attempt to POST /api/v1/jobs/{id}/resume with completed jobId
9. Verify rejection (cannot resume completed job)
10. Verify job status remains COMPLETE throughout

**Expected Results**:

- Process request on COMPLETE job returns 409 E701
- Cancel request on COMPLETE job rejected
- Resume request on COMPLETE job rejected
- Job state remains unchanged (COMPLETE)
- All invalid transitions logged for security audit
- Error messages clearly indicate invalid operation

**Acceptance Criteria Reference**: Section 5.1.2 State Transitions, E701

---

### E2E-STM-004: Concurrent State Change Handling

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-STM-004 |
| **Title** | Concurrent State Change Handling |
| **Priority** | P1 - High |
| **FR References** | Section 5.1.2, Section 10.9 (Transaction Handling) |
| **Preconditions** | Job in PROCESSING state |

**Test Steps**:

1. Start processing a file
2. Wait for PROCESSING state (~20% progress)
3. Open a second browser tab/window with same session
4. In Tab 1: Click Cancel button
5. In Tab 2: Simultaneously click Cancel button (within 100ms)
6. Verify only one cancel operation succeeds
7. Verify job transitions to CANCELLED exactly once
8. Verify no race condition errors displayed
9. Verify both tabs show consistent CANCELLED state
10. Query database to verify single state transition recorded

**Expected Results**:

- Only one state transition occurs despite concurrent requests
- Database transaction isolation prevents duplicate operations
- Both clients receive consistent state after resolution
- No data corruption or orphaned state
- Optimistic locking or transaction isolation prevents conflicts
- Audit log shows single cancellation event

**Acceptance Criteria Reference**: Section 5.1.2, Section 10.9, Transaction Handling

---

## 19. Theme & Responsive Tests

### 19.1 Overview

Theme and responsive tests validate dark mode functionality, theme persistence, and responsive layout behavior across different viewport sizes. These tests ensure the application meets accessibility requirements for theme preferences and provides proper user experience across desktop, tablet, and mobile devices.

### E2E-THEME-001: Dark Mode Toggle Functionality

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-THEME-001 |
| **Title** | Dark Mode Toggle Functionality |
| **Priority** | P1 - High |
| **FR References** | Section 6.0.1 (ThemeProvider), MAJ-01 |
| **Preconditions** | Application loaded |

**Test Steps**:

1. Navigate to application home page
2. Locate ThemeToggle component (typically in header/navigation)
3. Verify current theme state (light by default or system preference)
4. Click theme toggle button
5. Verify dropdown shows options: Light, Dark, System
6. Select "Dark" mode
7. Verify `dark` class applied to HTML document root
8. Verify background color changes to dark theme value
9. Verify all text remains readable (contrast validation)
10. Select "Light" mode
11. Verify `dark` class removed from HTML document root
12. Verify background returns to light theme

**Expected Results**:

- ThemeToggle component visible and accessible
- Theme selection dropdown functions correctly
- Dark mode applies `dark` class to `<html>` element
- CSS variables update for dark theme colors
- All components render correctly in dark mode
- Theme switch is immediate (no flash)

**Acceptance Criteria Reference**: Section 6.0.1, MAJ-01

---

### E2E-THEME-002: Theme Persistence Across Sessions

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-THEME-002 |
| **Title** | Theme Persistence Across Sessions |
| **Priority** | P1 - High |
| **FR References** | Section 6.0.1, next-themes persistence |
| **Preconditions** | Application loaded |

**Test Steps**:

1. Navigate to application
2. Switch theme to "Dark" mode
3. Verify dark theme applied
4. Refresh the browser (F5)
5. Verify dark theme persists after refresh
6. Navigate to History page
7. Verify dark theme maintained across navigation
8. Close browser tab
9. Open new browser tab and navigate to application
10. Verify dark theme restored from storage
11. Clear localStorage
12. Refresh page
13. Verify theme defaults to "System" preference

**Expected Results**:

- Theme preference stored in localStorage
- Theme persists across page refresh
- Theme maintained across navigation
- Theme restored on new browser session
- System preference applied when no stored preference
- No theme flash on page load (FOUC prevention)

**Acceptance Criteria Reference**: Section 6.0.1, next-themes

---

### E2E-RESP-001: Mobile Viewport Layout (375px)

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-RESP-001 |
| **Title** | Mobile Viewport Layout (375px) |
| **Priority** | P1 - High |
| **FR References** | Section 6.2.6 (Responsive Design), NFR-401 |
| **Preconditions** | Application loaded |

**Test Steps**:

1. Set viewport to 375x667 (iPhone SE/8 size)
2. Navigate to home page
3. Verify UploadZone displays full width (no horizontal overflow)
4. Verify URL input displays below UploadZone (stacked layout)
5. Verify Process button spans full width
6. Verify all interactive elements meet 44x44px touch target minimum
7. Upload a file and process
8. Verify progress card displays correctly
9. Verify results tabs are accessible (horizontal scroll if needed)
10. Navigate to History page
11. Verify history items display in card format (not table)
12. Verify pagination controls are touch-friendly

**Expected Results**:

- Single-column stacked layout on mobile
- No horizontal scrolling on main content
- All buttons/inputs meet 44x44px touch target
- Results tabs scrollable horizontally
- History displays as cards, not table
- All functionality accessible on mobile

**Acceptance Criteria Reference**: Section 6.2.6, NFR-401

---

### E2E-RESP-002: Tablet Viewport Layout (768px)

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-RESP-002 |
| **Title** | Tablet Viewport Layout (768px) |
| **Priority** | P1 - High |
| **FR References** | Section 6.2.6 (Responsive Design), SC-NF4 |
| **Preconditions** | Application loaded |

**Test Steps**:

1. Set viewport to 1024x768 (tablet landscape)
2. Navigate to home page
3. Verify UploadZone and URL input display side-by-side
4. Verify appropriate column split (60%/40% per spec)
5. Verify Process button positions correctly
6. Upload a file and complete processing
7. Verify results viewer displays all tabs without scrolling
8. Verify code blocks in Markdown view are readable
9. Navigate to History page
10. Verify table layout renders (not card layout)
11. Verify all table columns visible
12. Set viewport to 768x1024 (tablet portrait)
13. Verify layout adapts appropriately

**Expected Results**:

- Two-column layout on tablet landscape
- Single-column or adjusted layout on tablet portrait
- History displays as table on tablet
- All columns visible (File/URL, Status, Date, Actions)
- No content overflow or truncation
- Touch targets remain accessible

**Acceptance Criteria Reference**: Section 6.2.6, SC-NF4

---

## 20. Infrastructure Health Tests

### 20.1 Overview

Infrastructure health tests validate that the application correctly handles infrastructure dependency states including database connectivity, Redis availability, and graceful degradation scenarios. These tests ensure deployment readiness and operational reliability.

### E2E-INFRA-001: Health Endpoint Validation

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-INFRA-001 |
| **Title** | Health Endpoint Validation |
| **Priority** | P1 - High |
| **FR References** | Section 4.7 (Health Endpoint) |
| **Preconditions** | Application running with all dependencies |

**Test Steps**:

1. Send GET request to /api/v1/health
2. Verify 200 OK response
3. Verify response body contains status: "healthy"
4. Verify response includes database connectivity status
5. Verify response includes Redis connectivity status
6. Verify response includes MCP server status
7. Verify response includes version information
8. Verify response time < 500ms
9. Make multiple health requests
10. Verify consistent responses

**Expected Results**:

- Health endpoint returns 200 when all systems operational
- Response includes component health breakdown
- Database, Redis, MCP statuses all show "healthy"
- Version matches deployed application version
- Response time indicates no connection issues
- Endpoint suitable for load balancer health checks

**Acceptance Criteria Reference**: Section 4.7 (Health Endpoint)

---

### E2E-INFRA-002: Graceful Degradation on Redis Failure

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-INFRA-002 |
| **Title** | Graceful Degradation on Redis Failure |
| **Priority** | P1 - High |
| **FR References** | Section 7.4.3 (Circuit Breaker), FR-504 |
| **Preconditions** | Redis unavailability can be simulated in test environment |

**Test Steps**:

1. Start a processing job normally
2. Verify SSE connection established
3. Verify progress events received
4. Simulate Redis unavailability (via test environment control)
5. Verify application detects Redis failure
6. Verify SSE falls back to polling mode (FR-504)
7. Verify "Using backup connection" message displayed
8. Verify polling continues at 2s intervals
9. Verify processing can still complete
10. Restore Redis availability
11. Verify subsequent jobs use SSE normally

**Expected Results**:

- Redis failure detected gracefully
- No unhandled errors displayed to user
- Fallback to polling mode automatic
- User informed of degraded connection
- Processing functionality preserved
- Recovery automatic when Redis returns
- Circuit breaker prevents repeated failed connections

**Acceptance Criteria Reference**: FR-504, Section 7.4.3

---

### E2E-INFRA-003: Database Connection Recovery

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-INFRA-003 |
| **Title** | Database Connection Recovery |
| **Priority** | P1 - High |
| **FR References** | Section 10.9, validateDatabaseConnection() |
| **Preconditions** | Database connectivity can be toggled in test environment |

**Test Steps**:

1. Complete a processing job successfully
2. Navigate to History page
3. Verify job appears in history
4. Simulate brief database unavailability (5 seconds)
5. Refresh History page during unavailability
6. Verify appropriate error message (not stack trace)
7. Verify retry mechanism available
8. Restore database connectivity
9. Refresh or click retry
10. Verify history loads correctly
11. Verify all job data intact
12. Start new processing job
13. Verify job completes and persists correctly

**Expected Results**:

- Database failure shows user-friendly error
- No sensitive error details exposed
- Retry/refresh option available
- Connection recovery automatic
- No data loss during brief outage
- New operations succeed after recovery
- Connection pool recovers correctly

**Acceptance Criteria Reference**: Section 10.9, Connection Pool Configuration

---

## 21. Next.js Specific Tests

### 21.1 Overview

Next.js specific tests validate framework-specific behaviors including Server Component hydration, client-side navigation with prefetching, and Zustand state persistence. These tests ensure the application leverages Next.js features correctly without hydration mismatches or state management issues.

### E2E-NEXT-001: Server Component Hydration Verification

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-NEXT-001 |
| **Title** | Server Component Hydration Verification |
| **Priority** | P1 - High |
| **FR References** | Section 6.1 (Component Architecture) |
| **Preconditions** | Application loaded, console monitoring enabled |

**Test Steps**:

1. Configure Playwright to capture console messages
2. Set up listener for hydration errors:
   - "Hydration failed"
   - "Text content did not match"
   - "There was an error while hydrating"
3. Navigate to home page
4. Verify no hydration errors in console
5. Complete a file processing flow
6. Navigate to results view
7. Switch between all result tabs (Markdown, HTML, JSON, Raw)
8. Verify no hydration errors during tab switches
9. Navigate to History page
10. Verify no hydration errors
11. Return to home page via navigation
12. Verify no hydration errors throughout session

**Expected Results**:

- Zero hydration mismatch errors in console
- Server Components (MarkdownView, HtmlView, JsonView, RawView) render correctly
- Client Components hydrate without errors
- Tab switching does not cause hydration issues
- Navigation between pages is error-free
- All content renders identically on server and client

**Acceptance Criteria Reference**: Section 6.1, Server/Client Component Architecture

---

### E2E-NEXT-002: Client-side Navigation with Prefetch

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-NEXT-002 |
| **Title** | Client-side Navigation with Prefetch |
| **Priority** | P1 - High |
| **FR References** | Next.js Link component, Section 6.1 |
| **Preconditions** | Application loaded |

**Test Steps**:

1. Navigate to home page
2. Identify History link in navigation
3. Monitor network requests
4. Hover over History link
5. Verify prefetch request initiated (RSC payload or page chunk)
6. Click History link
7. Verify navigation is instant (< 100ms)
8. Verify no full page reload (SPA navigation)
9. Verify URL updates to /history
10. Click browser back button
11. Verify instant navigation back to home
12. Verify no content flash during transitions

**Expected Results**:

- Link prefetching occurs on hover
- Navigation is instant (client-side)
- No full page reload on navigation
- Browser history maintained
- Back/forward navigation works correctly
- Prefetched content used for instant load

**Acceptance Criteria Reference**: Next.js Link, Section 6.1

---

### E2E-NEXT-003: Zustand State Persistence on Refresh

| Attribute | Value |
|-----------|-------|
| **Test ID** | E2E-NEXT-003 |
| **Title** | Zustand State Persistence on Refresh |
| **Priority** | P1 - High |
| **FR References** | Section 5.3 (Zustand Store), CRI-10 |
| **Preconditions** | Application loaded |

**Test Steps**:

1. Navigate to home page
2. Upload a file (do not process yet)
3. Verify file appears in upload zone
4. Check localStorage for 'document-store' key
5. Verify file metadata persisted (name, size, type - not File object)
6. Start processing
7. Wait for progress to reach ~40%
8. Refresh browser (F5)
9. Verify application recovers from localStorage state
10. Verify jobId restored from persisted state
11. Verify SSE reconnection initiated
12. Verify "Reconnecting to job..." message displayed
13. Wait for processing to complete
14. Verify results are correct
15. Clear localStorage and refresh
16. Verify application starts fresh (no stale state)

**Expected Results**:

- Zustand persist middleware saves to localStorage
- File metadata (not File object) persisted via partialize
- Job ID persisted for recovery
- Browser refresh triggers recovery flow
- SSE reconnection occurs automatically
- Recovery UI indicates reconnection status
- Processing completes successfully after recovery
- Clearing storage resets application state

**Acceptance Criteria Reference**: Section 5.3, CRI-10, NFR-203

---

## Test Summary

### Test Count by Priority

| Priority | Count | Run Frequency |
|----------|-------|---------------|
| P0 - Critical | 22 | Every build |
| P1 - High | 71 | Every PR |
| P2 - Medium | 8 | Daily |
| P3 - Low | 0 | Weekly |
| **Total** | **101** | |

### Coverage by Functional Requirement

| FR Series | Tests | Coverage |
|-----------|-------|----------|
| FR-100 (File Upload) | 7 | 100% |
| FR-200 (URL Input) | 5 | 100% |
| FR-300 (Input State) | 2 | 100% |
| FR-400 (Processing) | 6 | 100% |
| FR-500 (Progress) | 4 | 100% |
| FR-600 (Results) | 6 | 100% |
| FR-700 (History) | 4 | 100% |
| FR-800 (Error) | 7 | 100% |
| NFR-100 (Performance) | 3 | 100% |
| NFR-100 (Load Testing) | 10 | 100% |
| NFR-302 (Rate Limiting) | 3 | 100% |
| FR-402 (MCP Tools) | 8 | 100% |
| Section 5.1-5.2 (Database) | 4 | 100% |
| Section 5.5 (Checkpoints) | 4 | 100% |
| NFR-401 (Security) | 4 | 100% |
| Section 5.1.2 (State Machine) | 4 | 100% |
| Section 6.0.1 (Theme) | 2 | 100% |
| Section 6.2.6 (Responsive) | 2 | 100% |
| Section 4.7, 7.4.3 (Infrastructure) | 3 | 100% |
| Section 6.1 (Next.js) | 3 | 100% |

---

## Changelog

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0.0 | 2025-12-12 | Julia Santos | Initial E2E test plan |
| 1.1.0 | 2025-12-13 | Julia Santos | Added Phase 1 critical priority tests based on team reviews: Rate Limiting (Section 13), MCP Tool Coverage (Section 14), Database State Verification (Section 15), Checkpoint & Resume (Section 16). Added 19 new test cases. |
| 1.2.0 | 2025-12-13 | Julia Santos | Added Phase 2 high-priority tests based on team reviews: Security Tests (Section 17 - Alex, Bob), State Machine Edge Cases (Section 18 - Sophia), Theme & Responsive Tests (Section 19 - Gordon), Infrastructure Health Tests (Section 20 - William), Next.js Specific Tests (Section 21 - Neo). Added 18 new test cases covering XSS prevention, file magic byte validation, CSP enforcement, state machine transitions, dark mode, responsive layouts, infrastructure health, and Next.js hydration/state persistence. |
| 1.3.0 | 2025-12-15 | Julia Santos | Remediated A-05 defect by adding comprehensive load testing scenarios (Section 11.1). Added 10 new P1 load tests: E2E-LOAD-001 (Concurrent User Baseline), E2E-LOAD-002 (Concurrent File Uploads), E2E-LOAD-003 (SSE Connection Scaling), E2E-LOAD-004 (API Rate Limit Stress), E2E-LOAD-005 (Database Connection Pool Exhaustion), E2E-LOAD-006 (Redis Pub/Sub High Throughput), E2E-LOAD-007 (MCP Tool Invocation Under Load), E2E-LOAD-008 (Memory Leak Detection), E2E-LOAD-009 (CPU Utilization Under Peak Load), E2E-LOAD-010 (Response Time Degradation Curve). Each test includes complete Locust configuration, SLA thresholds, and acceptance criteria. Total test count increased from 91 to 101 tests. |
