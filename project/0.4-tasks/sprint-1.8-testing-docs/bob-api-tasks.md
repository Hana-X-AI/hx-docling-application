# API Tasks: Sprint 1.8 - Testing & Documentation

**Sprint**: 1.8 - Testing & Documentation
**Author**: Bob Martin (@bob) - API Design SME
**Created**: 2025-12-12
**Status**: PENDING
**Dependencies**: Sprint 1.7 Complete

---

## Overview

This task file covers API-related deliverables for Sprint 1.8, focusing on comprehensive API testing, security testing, and API documentation for the CLAUDE.md file.

---

## Tasks

### BOB-1.8-001: Write Comprehensive API Route Tests

| Field | Value |
|-------|-------|
| **Task ID** | BOB-1.8-001 |
| **Title** | Create comprehensive test suite for all API routes |
| **Priority** | P0 (Critical) |
| **Effort** | 90 minutes |
| **Dependencies** | All previous API tasks, Julia's test infrastructure |

#### Description

Write comprehensive unit and integration tests for all API routes, ensuring complete coverage of success paths, error handling, validation, and edge cases.

#### Acceptance Criteria

- [ ] Tests for `/api/v1/upload` (file and URL variants)
- [ ] Tests for `/api/v1/process` endpoint
- [ ] Tests for `/api/v1/process/{jobId}/events` SSE endpoint
- [ ] Tests for `/api/v1/history` with pagination
- [ ] Tests for `/api/v1/jobs/{id}` detail endpoint
- [ ] Tests for `/api/v1/jobs/{id}/cancel` endpoint
- [ ] Tests for `/api/v1/jobs/{id}/resume` endpoint
- [ ] Tests for `/api/v1/jobs/{id}/download` endpoint
- [ ] Tests for `/api/v1/health` endpoint
- [ ] Validation error handling tested for all endpoints
- [ ] Rate limiting behavior tested
- [ ] Authorization (session) tested for all protected endpoints

#### Technical Notes

```typescript
// Test Structure
// src/app/api/v1/upload/route.test.ts
// src/app/api/v1/process/route.test.ts
// src/app/api/v1/process/[jobId]/events/route.test.ts
// src/app/api/v1/history/route.test.ts
// src/app/api/v1/jobs/[id]/route.test.ts
// src/app/api/v1/jobs/[id]/cancel/route.test.ts
// src/app/api/v1/jobs/[id]/resume/route.test.ts
// src/app/api/v1/jobs/[id]/download/route.test.ts
// src/app/api/v1/health/route.test.ts

// Test Categories per Route
describe('POST /api/v1/upload', () => {
  describe('File Upload', () => {
    it('should create job for valid PDF upload', async () => {});
    it('should reject file exceeding size limit (E001)', async () => {});
    it('should reject unsupported file type (E002)', async () => {});
    it('should return 201 with Location header', async () => {});
  });

  describe('URL Upload', () => {
    it('should create job for valid URL', async () => {});
    it('should reject blocked URL (E104)', async () => {});
    it('should reject invalid URL format (E101)', async () => {});
  });

  describe('Validation', () => {
    it('should return E100 for malformed request', async () => {});
  });

  describe('Rate Limiting', () => {
    it('should return 429 after 10 requests', async () => {});
    it('should include Retry-After header', async () => {});
  });
});
```

**MSW Handlers for API Mocking**:
```typescript
// src/mocks/handlers.ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  // Mock database responses
  http.post('/api/v1/upload', async ({ request }) => {
    // Test handler logic
  }),

  // Mock MCP server
  http.post('http://hx-docling-server:8000/jsonrpc', () => {
    return HttpResponse.json({
      jsonrpc: '2.0',
      result: { /* mock response */ },
      id: 1,
    });
  }),
];
```

#### Deliverables

- `src/app/api/v1/upload/route.test.ts`
- `src/app/api/v1/process/route.test.ts`
- `src/app/api/v1/process/[jobId]/events/route.test.ts`
- `src/app/api/v1/history/route.test.ts`
- `src/app/api/v1/jobs/[id]/route.test.ts`
- `src/app/api/v1/jobs/[id]/cancel/route.test.ts`
- `src/app/api/v1/jobs/[id]/resume/route.test.ts`
- `src/app/api/v1/jobs/[id]/download/route.test.ts`
- `src/app/api/v1/health/route.test.ts`

---

### BOB-1.8-002: Write Security Tests for API Layer

| Field | Value |
|-------|-------|
| **Task ID** | BOB-1.8-002 |
| **Title** | Create security test suite for API endpoints |
| **Priority** | P0 (Critical) |
| **Effort** | 45 minutes |
| **Dependencies** | BOB-1.4-004 (SSRF tests), Frank's security review |

#### Description

Write security-focused tests covering SSRF prevention, input validation/sanitization, and authorization bypass attempts.

#### Acceptance Criteria

- [ ] SSRF tests for all blocked IP patterns
- [ ] SSRF tests for DNS rebinding attempts (encoded hostnames)
- [ ] Input validation tests for SQL injection patterns
- [ ] Input validation tests for XSS patterns in filenames
- [ ] Authorization bypass tests (accessing other session's jobs)
- [ ] Rate limiting bypass attempts
- [ ] Path traversal prevention in file storage

#### Technical Notes

```typescript
// File: test/security/api-security.test.ts

describe('API Security Tests', () => {
  describe('SSRF Prevention', () => {
    it.each([
      'http://localhost/admin',
      'http://127.0.0.1/admin',
      'http://[::1]/admin',
      'http://10.0.0.1/internal',
      'http://172.16.0.1/internal',
      'http://192.168.1.1/internal',
      'http://169.254.169.254/metadata', // AWS metadata
      'http://internal.hx.dev.local/api',
    ])('should block SSRF attempt: %s', async (blockedUrl) => {
      const response = await fetch('/api/v1/upload', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ url: blockedUrl }),
      });
      expect(response.status).toBe(400);
      const body = await response.json();
      expect(body.error.code).toBe('E104');
    });

    it('should block URL-encoded SSRF attempts', async () => {
      const encoded = 'http://127%2E0%2E0%2E1/admin';
      // Test encoding bypass attempt
    });
  });

  describe('Input Validation', () => {
    it('should sanitize filename for path traversal', async () => {
      const maliciousName = '../../../etc/passwd';
      // Test file upload with malicious filename
    });

    it('should reject SQL injection in query params', async () => {
      const response = await fetch('/api/v1/history?sortBy=createdAt; DROP TABLE jobs;--');
      expect(response.status).toBe(400);
    });
  });

  describe('Authorization', () => {
    it('should not expose jobs from other sessions', async () => {
      // Create job with session A
      // Try to access with session B
      // Expect 404 (not 403 for security)
    });
  });
});
```

#### Deliverables

- `test/security/api-security.test.ts` - Security test suite
- `test/security/ssrf.test.ts` - Extended SSRF tests
- `test/security/input-validation.test.ts` - Input validation tests

---

### BOB-1.8-003: Write State Machine Transition Tests

| Field | Value |
|-------|-------|
| **Task ID** | BOB-1.8-003 |
| **Title** | Test all job state machine transitions via API |
| **Priority** | P1 (Critical) |
| **Effort** | 45 minutes |
| **Dependencies** | Sophia's state machine definition |

#### Description

Write tests that verify all 10 job states and their valid/invalid transitions are enforced by the API layer.

#### Acceptance Criteria

- [ ] Tests for PENDING -> PROCESSING transition (process endpoint)
- [ ] Tests for PROCESSING -> COMPLETE transition
- [ ] Tests for PROCESSING -> ERROR transition
- [ ] Tests for PROCESSING -> CANCELLED transition (cancel endpoint)
- [ ] Tests for PROCESSING -> RETRY_* transitions
- [ ] Tests for ERROR -> PROCESSING transition (resume endpoint)
- [ ] Tests for CANCELLED -> PROCESSING transition (resume with checkpoint)
- [ ] Tests for invalid transitions (409 Conflict responses)
- [ ] Tests for all 10 states: PENDING, UPLOADING, PROCESSING, RETRY_1-3, COMPLETE, PARTIAL_COMPLETE, CANCELLED, ERROR

#### Technical Notes

```typescript
// File: test/state-machine/job-transitions.test.ts

describe('Job State Machine Transitions', () => {
  const ALL_STATES: JobStatus[] = [
    'PENDING', 'UPLOADING', 'PROCESSING',
    'RETRY_1', 'RETRY_2', 'RETRY_3',
    'COMPLETE', 'PARTIAL_COMPLETE', 'CANCELLED', 'ERROR',
  ];

  describe('Valid Transitions', () => {
    it('PENDING -> PROCESSING via process endpoint', async () => {
      const job = await createJobInState('PENDING');
      const response = await fetch('/api/v1/process', {
        method: 'POST',
        body: JSON.stringify({ jobId: job.id }),
      });
      expect(response.status).toBe(202);
      const updated = await getJob(job.id);
      expect(updated.status).toBe('PROCESSING');
    });

    it('PROCESSING -> CANCELLED via cancel endpoint', async () => {
      const job = await createJobInState('PROCESSING');
      const response = await fetch(`/api/v1/jobs/${job.id}/cancel`, {
        method: 'POST',
      });
      expect(response.status).toBe(200);
      const updated = await getJob(job.id);
      expect(updated.status).toBe('CANCELLED');
    });

    it('ERROR -> PROCESSING via resume endpoint', async () => {
      const job = await createJobInState('ERROR', { hasCheckpoint: true });
      const response = await fetch(`/api/v1/jobs/${job.id}/resume`, {
        method: 'POST',
      });
      expect(response.status).toBe(200);
    });
  });

  describe('Invalid Transitions', () => {
    it.each([
      ['COMPLETE', 'cancel'],
      ['CANCELLED', 'cancel'],
      ['ERROR', 'cancel'],
    ])('should reject %s -> cancel', async (fromState, action) => {
      const job = await createJobInState(fromState);
      const response = await fetch(`/api/v1/jobs/${job.id}/cancel`, {
        method: 'POST',
      });
      expect(response.status).toBe(409);
    });

    it.each([
      ['PENDING', 'resume'],
      ['PROCESSING', 'resume'],
      ['COMPLETE', 'resume'],
    ])('should reject %s -> resume', async (fromState, action) => {
      const job = await createJobInState(fromState);
      const response = await fetch(`/api/v1/jobs/${job.id}/resume`, {
        method: 'POST',
      });
      expect(response.status).toBe(409);
    });
  });
});
```

#### Deliverables

- `test/state-machine/job-transitions.test.ts` - State transition tests

---

### BOB-1.8-004: Document API in CLAUDE.md

| Field | Value |
|-------|-------|
| **Task ID** | BOB-1.8-004 |
| **Title** | Add API documentation section to CLAUDE.md |
| **Priority** | P1 (Critical) |
| **Effort** | 30 minutes |
| **Dependencies** | All API implementation complete |

#### Description

Add comprehensive API documentation to CLAUDE.md including endpoint reference, error codes, and usage examples.

#### Acceptance Criteria

- [ ] API endpoint reference table
- [ ] Error code catalog with meanings
- [ ] Request/Response examples for each endpoint
- [ ] Rate limiting documentation
- [ ] SSE event documentation
- [ ] Authentication (session) documentation

#### Technical Notes

```markdown
## API Reference

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | /api/v1/upload | Upload file or register URL |
| POST | /api/v1/process | Start processing |
| GET | /api/v1/process/{jobId}/events | SSE progress stream |
| GET | /api/v1/history | Get job history |
| GET | /api/v1/jobs/{id} | Get job details |
| POST | /api/v1/jobs/{id}/cancel | Cancel job |
| POST | /api/v1/jobs/{id}/resume | Resume from checkpoint |
| GET | /api/v1/jobs/{id}/download | Download results |
| GET | /api/v1/health | Health check |

### Error Codes

| Code | HTTP | Description |
|------|------|-------------|
| E001 | 400/413 | File too large |
| E002 | 400 | Unsupported file type |
| E100 | 400 | Validation error |
| E101 | 400 | Invalid URL format |
| E104 | 400 | URL blocked (SSRF) |
| E501 | 404 | Job not found |
| E601 | 429 | Rate limit exceeded |
| E701 | 409 | Job already processing |
| E702 | 409 | Job not cancellable |
| E703 | 409 | Job not resumable (status check) |
| E704 | 409 | Checkpoint expired (TTL exceeded) |
| E705 | 409 | Checkpoint corrupted (checksum mismatch) |

### SSE Events

| Event | Description |
|-------|-------------|
| connected | Connection established |
| progress | Processing update |
| state_sync | Reconnection state |
| retry | Retry initiated |
| complete | Processing complete |
| cancelled | Job cancelled |
| error | Processing failed |
```

#### Deliverables

- Updated `CLAUDE.md` with API section

---

### BOB-1.8-005: Document MCP Error Code Mapping

| Field | Value |
|-------|-------|
| **Task ID** | BOB-1.8-005 |
| **Title** | Document JSON-RPC to application error code mapping |
| **Priority** | P2 (High) |
| **Effort** | 15 minutes |
| **Dependencies** | James's error mapping (Sprint 1.5a) |

#### Description

Document the mapping from MCP JSON-RPC error codes to application error codes for troubleshooting and debugging.

#### Acceptance Criteria

- [ ] Table mapping JSON-RPC codes (-32700 to -32099) to E2xx
- [ ] Description of each error condition
- [ ] Recovery suggestions for each error type

#### Technical Notes

```markdown
### MCP Error Code Mapping

| JSON-RPC Code | App Code | Description | Recovery |
|---------------|----------|-------------|----------|
| -32700 | E200 | Parse error | Check request format |
| -32600 | E201 | Invalid request | Verify parameters |
| -32601 | E202 | Method not found | Check tool name |
| -32602 | E203 | Invalid params | Verify tool arguments |
| -32603 | E204 | Internal error | Retry with backoff |
| -32000 to -32099 | E210-E299 | Server errors | Contact support |
```

#### Deliverables

- Updated `CLAUDE.md` with MCP error mapping section

---

## Sprint Summary

| Task ID | Title | Effort | Priority |
|---------|-------|--------|----------|
| BOB-1.8-001 | Comprehensive API Route Tests | 90m | P0 |
| BOB-1.8-002 | Security Tests | 45m | P0 |
| BOB-1.8-003 | State Machine Tests | 45m | P1 |
| BOB-1.8-004 | API Documentation in CLAUDE.md | 30m | P1 |
| BOB-1.8-005 | MCP Error Mapping Docs | 15m | P2 |

**Total Effort**: 3 hours 45 minutes

---

## Dependencies Graph

```
Sprint 1.7 Complete
        |
        +-- Julia's test infrastructure
        |
        +-- Frank's security review
        |
        +-- Sophia's state machine
        |
        v
BOB-1.8-001 (API Route Tests)
        |
        +----> BOB-1.8-002 (Security Tests)
        |
        +----> BOB-1.8-003 (State Machine Tests)
        |
        v
BOB-1.8-004 (CLAUDE.md API Docs)
        |
        v
BOB-1.8-005 (Error Mapping Docs)
```

---

## Quality Gates

### Test Coverage Requirements

| Category | Target |
|----------|--------|
| API route line coverage | >= 85% |
| API route branch coverage | >= 80% |
| Security tests | 100% of SSRF patterns |
| State transitions | 100% of valid/invalid |

### Documentation Requirements

- All endpoints documented with examples
- All error codes documented with recovery actions
- SSE event format documented with schemas
- Rate limiting documented with headers

---

## Coordination Notes

- **Julia (@julia)**: Lead for Sprint 1.8, coordinate test strategy
- **Frank (@frank)**: Security review of API security tests
- **Sophia (@sophia)**: State machine definition for transition tests
- **Neo (@neo)**: Integration testing coordination
- **William (@william)**: SSE-related test scenarios
