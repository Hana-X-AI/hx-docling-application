# API Tasks: Sprint 1.4 - URL Input with Security

**Sprint**: 1.4 - URL Input with Security
**Author**: Bob Martin (@bob) - API Design SME
**Created**: 2025-12-12
**Status**: PENDING
**Dependencies**: Sprint 1.3 Complete

---

## Overview

This task file covers API-related deliverables for Sprint 1.4, focusing on URL validation, SSRF prevention, and the Process API endpoint that initiates document processing.

---

## Tasks

### BOB-1.4-001: Implement Process API Endpoint

| Field | Value |
|-------|-------|
| **Task ID** | BOB-1.4-001 |
| **Title** | Create POST /api/v1/process endpoint to initiate processing |
| **Priority** | P0 (Critical) |
| **Effort** | 30 minutes |
| **Dependencies** | BOB-1.3-001 (Upload complete), Sprint 1.5a (MCP client) |

#### Description

Implement the process endpoint that initiates document processing for a previously uploaded job. This endpoint transitions the job from PENDING to PROCESSING status and returns the SSE stream URL for progress tracking.

#### Acceptance Criteria

- [ ] Endpoint accessible at `POST /api/v1/process`
- [ ] Accepts JSON body with `jobId` field
- [ ] Validates job exists and is in PENDING status
- [ ] Transitions job status to PROCESSING
- [ ] Returns 202 Accepted with streamUrl
- [ ] Returns 404 with E501 if job not found
- [ ] Returns 409 with E701 if job already processing/complete
- [ ] Integrates rate limiting middleware

#### Technical Notes

```typescript
// File: src/app/api/v1/process/route.ts

// Request Schema
const processSchema = z.object({
  jobId: z.string().uuid(),
});

// Request
POST /api/v1/process
{
  "jobId": "uuid-v4"
}

// Response: 202 Accepted
{
  "data": {
    "jobId": "uuid-v4",
    "status": "PROCESSING",
    "streamUrl": "/api/v1/process/{jobId}/events"
  }
}

// Error: 404 Not Found
{
  "error": {
    "code": "E501",
    "message": "Job not found",
    "userMessage": "The requested job could not be found.",
    "retryable": false
  }
}

// Error: 409 Conflict
{
  "error": {
    "code": "E701",
    "message": "Job already processing",
    "userMessage": "This job is already being processed.",
    "retryable": false
  }
}
```

**State Validation**:
```typescript
const PROCESSABLE_STATES: JobStatus[] = ['PENDING'];

async function validateJobForProcessing(jobId: string): Promise<Job> {
  const job = await prisma.job.findUnique({ where: { id: jobId } });

  if (!job) {
    throw new AppError('E501', 'Job not found', 404);
  }

  if (!PROCESSABLE_STATES.includes(job.status)) {
    throw new AppError('E701', 'Job not in processable state', 409);
  }

  return job;
}
```

#### Deliverables

- `src/app/api/v1/process/route.ts` - Process endpoint
- `src/lib/jobs/validation.ts` - Job state validation utilities

---

### BOB-1.4-002: Apply Rate Limiting to Process Endpoint

| Field | Value |
|-------|-------|
| **Task ID** | BOB-1.4-002 |
| **Title** | Integrate rate limiting with process endpoint |
| **Priority** | P1 (Critical) |
| **Effort** | 10 minutes |
| **Dependencies** | BOB-1.2-003 (Rate Limit Middleware), BOB-1.4-001 |

#### Description

Apply rate limiting middleware to the process endpoint to prevent abuse of the MCP server resources.

#### Acceptance Criteria

- [ ] Process endpoint rate limited to 10 requests/minute/session
- [ ] Rate limit headers returned on all responses
- [ ] 429 returned with E601 error when limit exceeded

#### Technical Notes

```typescript
export const POST = withRateLimit(
  { windowMs: 60000, maxRequests: 10 },
  withValidation(
    { body: processSchema },
    processHandler
  )
);
```

#### Deliverables

- Updated `src/app/api/v1/process/route.ts` with rate limiting

---

### BOB-1.4-005: Create Shared SSRF Configuration

| Field | Value |
|-------|-------|
| **Task ID** | BOB-1.4-005 |
| **Title** | Create centralized SSRF prevention rules for API and client |
| **Priority** | P0 (Security Critical) |
| **Effort** | 15 minutes |
| **Dependencies** | None |

#### Description

Create a shared SSRF (Server-Side Request Forgery) configuration module that centralizes all blocking rules. This ensures consistent validation between API-side and client-side Zod schemas, eliminating duplication and reducing security bypass risk from rule divergence.

#### Acceptance Criteria

- [ ] Shared config defines blockedHostnames array
- [ ] Shared config defines privateRanges regex array
- [ ] Shared config defines internalDomains regex
- [ ] checkSSRFRules function returns blocked status with reason
- [ ] Config is importable by both API validation and client Zod schemas
- [ ] TypeScript types exported for type safety

#### Technical Notes

```typescript
// File: src/lib/validation/ssrf-config.ts

/**
 * Centralized SSRF (Server-Side Request Forgery) prevention rules.
 * Used by both API validation (url.ts) and client-side Zod schemas.
 *
 * SECURITY: Any changes here affect ALL URL validation in the application.
 * Review with @frank before modifying.
 */

export const SSRF_RULES = {
  /** Hostnames that resolve to localhost
   *  NOTE: URL.hostname strips brackets from IPv6 addresses (e.g., 'http://[::1]/'
   *  yields hostname '::1'), so we use un-bracketed forms for matching.
   *  Also blocks IPv4-mapped IPv6 localhost variants.
   */
  blockedHostnames: ['localhost', '127.0.0.1', '0.0.0.0', '::1', '::ffff:127.0.0.1', '::ffff:7f00:1'],

  /** RFC 1918 private IP ranges */
  privateRanges: [
    /^10\./,                           // 10.0.0.0/8
    /^172\.(1[6-9]|2[0-9]|3[01])\./,   // 172.16.0.0/12
    /^192\.168\./,                      // 192.168.0.0/16
  ],

  /** Link-local/APIPA addresses (RFC 3927) */
  linkLocalRanges: [
    /^169\.254\./,                      // 169.254.0.0/16
  ],

  /** Internal domains to block */
  internalDomains: /\.hx\.dev\.local$/i,
} as const;

export type SSRFRules = typeof SSRF_RULES;

/**
 * Normalize a hostname for consistent SSRF checking.
 * - Trims whitespace
 * - Converts to lowercase
 * - Removes trailing dot (FQDN normalization)
 * - Strips surrounding IPv6 brackets if present
 * @param hostname - The raw hostname to normalize
 * @returns Normalized hostname
 */
function normalizeHostname(hostname: string): string {
  let normalized = hostname.trim().toLowerCase();

  // Remove trailing dot (FQDN format)
  if (normalized.endsWith('.')) {
    normalized = normalized.slice(0, -1);
  }

  // Strip surrounding IPv6 brackets (e.g., "[::1]" -> "::1")
  if (normalized.startsWith('[') && normalized.endsWith(']')) {
    normalized = normalized.slice(1, -1);
  }

  return normalized;
}

/**
 * Check if a hostname matches any SSRF blocking rule.
 * The hostname is normalized internally (trimmed, lowercased, trailing dot removed,
 * IPv6 brackets stripped) so callers don't need to pre-normalize.
 * @param hostname - The hostname to check
 * @returns Object with blocked status and optional reason
 */
export function checkSSRFRules(hostname: string): { blocked: boolean; reason?: string } {
  // Normalize hostname for consistent checking
  const normalizedHostname = normalizeHostname(hostname);

  // Check blocked hostnames
  if (SSRF_RULES.blockedHostnames.includes(normalizedHostname)) {
    return { blocked: true, reason: 'Localhost addresses are not allowed' };
  }

  // Check private IP ranges (RFC 1918)
  for (const range of SSRF_RULES.privateRanges) {
    if (range.test(normalizedHostname)) {
      return { blocked: true, reason: 'Private IP addresses are not allowed' };
    }
  }

  // Check link-local addresses (RFC 3927)
  for (const range of SSRF_RULES.linkLocalRanges) {
    if (range.test(normalizedHostname)) {
      return { blocked: true, reason: 'Link-local addresses are not allowed' };
    }
  }

  // Check internal domains
  if (SSRF_RULES.internalDomains.test(normalizedHostname)) {
    return { blocked: true, reason: 'Internal domains are not allowed' };
  }

  return { blocked: false };
}
```

#### Deliverables

- `src/lib/validation/ssrf-config.ts` - Shared SSRF configuration module

---

### BOB-1.4-003: Implement URL SSRF Prevention Validation

| Field | Value |
|-------|-------|
| **Task ID** | BOB-1.4-003 |
| **Title** | Create comprehensive SSRF prevention validation |
| **Priority** | P0 (Security Critical) |
| **Effort** | 20 minutes |
| **Dependencies** | BOB-1.4-005 (Shared SSRF Config) |

#### Description

Implement server-side URL validation that prevents Server-Side Request Forgery (SSRF) attacks by blocking requests to private IP ranges, localhost, and internal domains.

#### Acceptance Criteria

- [ ] Blocks localhost/127.0.0.1
- [ ] Blocks 10.0.0.0/8 (Class A private)
- [ ] Blocks 172.16.0.0/12 (Class B private)
- [ ] Blocks 192.168.0.0/16 (Class C private)
- [ ] Blocks 169.254.0.0/16 (link-local)
- [ ] Blocks *.hx.dev.local internal domains
- [ ] Blocks IPv6 localhost (::1)
- [ ] Returns E104 error code for blocked URLs
- [ ] DNS resolution check for hostname-to-IP mapping attacks

#### Technical Notes

**IMPORTANT**: This task now imports from the shared SSRF config (BOB-1.4-005) instead of defining rules inline. This eliminates duplication with Neo's client-side validation.

```typescript
// File: src/lib/validation/url.ts

import { checkSSRFRules } from './ssrf-config';

/**
 * Check if a URL is blocked by SSRF prevention rules.
 * Delegates to shared checkSSRFRules for consistent validation.
 */
export function isURLBlocked(urlString: string): boolean {
  try {
    const url = new URL(urlString);
    const hostname = url.hostname.toLowerCase();
    return checkSSRFRules(hostname).blocked;
  } catch {
    return true; // Invalid URL format
  }
}

export function validateURL(urlString: string): ValidationResult {
  // Check format
  if (!isValidURLFormat(urlString)) {
    return { valid: false, error: 'E101', message: 'Invalid URL format' };
  }

  // Check protocol
  const url = new URL(urlString);
  if (!['http:', 'https:'].includes(url.protocol)) {
    return { valid: false, error: 'E102', message: 'Only HTTP/HTTPS allowed' };
  }

  // Check length
  if (urlString.length > 2048) {
    return { valid: false, error: 'E103', message: 'URL exceeds maximum length' };
  }

  // Check SSRF using shared config
  if (isURLBlocked(urlString)) {
    return { valid: false, error: 'E104', message: 'URL blocked for security' };
  }

  return { valid: true };
}
```

#### Deliverables

- `src/lib/validation/url.ts` - URL validation with SSRF prevention
- `src/lib/validation/url.test.ts` - Comprehensive SSRF test cases

---

### BOB-1.4-004: Write Unit Tests for SSRF Validation

| Field | Value |
|-------|-------|
| **Task ID** | BOB-1.4-004 |
| **Title** | Create comprehensive unit tests for SSRF prevention |
| **Priority** | P0 (Security Critical) |
| **Effort** | 20 minutes |
| **Dependencies** | BOB-1.4-003 |

#### Description

Write unit tests covering all SSRF prevention patterns to ensure complete security coverage.

#### Acceptance Criteria

- [ ] Tests for localhost variants (localhost, 127.0.0.1, [::1])
- [ ] Tests for all private IP ranges
- [ ] Tests for link-local addresses
- [ ] Tests for internal domain blocking
- [ ] Tests for valid URLs (should pass)
- [ ] Tests for edge cases (encoded characters, punycode)

#### Technical Notes

```typescript
// File: src/lib/validation/url.test.ts

describe('SSRF Prevention', () => {
  describe('isURLBlocked', () => {
    // Localhost variants
    it.each([
      'http://localhost/path',
      'http://127.0.0.1/path',
      'http://[::1]/path',
      'http://0.0.0.0/path',
    ])('should block localhost: %s', (url) => {
      expect(isURLBlocked(url)).toBe(true);
    });

    // Private ranges
    it.each([
      'http://10.0.0.1/path',
      'http://10.255.255.255/path',
      'http://172.16.0.1/path',
      'http://172.31.255.255/path',
      'http://192.168.0.1/path',
      'http://192.168.255.255/path',
    ])('should block private IP: %s', (url) => {
      expect(isURLBlocked(url)).toBe(true);
    });

    // Valid URLs
    it.each([
      'https://example.com/document.pdf',
      'https://docs.google.com/document/d/123',
      'http://8.8.8.8/path', // Public IP
    ])('should allow valid URL: %s', (url) => {
      expect(isURLBlocked(url)).toBe(false);
    });
  });
});
```

#### Deliverables

- `src/lib/validation/url.test.ts` - SSRF test suite

---

## Sprint Summary

| Task ID | Title | Effort | Priority |
|---------|-------|--------|----------|
| BOB-1.4-001 | Process API Endpoint | 30m | P0 |
| BOB-1.4-002 | Rate Limiting Integration | 10m | P1 |
| BOB-1.4-005 | Shared SSRF Configuration | 15m | P0 |
| BOB-1.4-003 | SSRF Prevention Validation | 20m | P0 |
| BOB-1.4-004 | SSRF Unit Tests | 20m | P0 |

**Total Effort**: 1 hour 35 minutes

---

## Dependencies Graph

```
Sprint 1.3 Complete
        |
        v
BOB-1.4-005 (Shared SSRF Config) ---> NEO-1.4-002 (Neo imports shared config)
        |
        v
BOB-1.4-003 (SSRF Validation)
        |
        v
BOB-1.4-004 (SSRF Tests)
        |
BOB-1.4-001 (Process Endpoint)
        |
        v
BOB-1.4-002 (Rate Limiting)
```

---

## Security Considerations

### SSRF Attack Vectors Covered

1. **Direct IP Access**: Blocking private IP ranges prevents direct access to internal services
2. **DNS Rebinding**: DNS resolution check recommended for future enhancement
3. **URL Encoding Bypass**: Tests include encoded character variants
4. **IPv6 Bypass**: IPv6 localhost and private ranges checked
5. **Domain Bypass**: Internal domain patterns blocked

### Error Response Security

- Error messages do not reveal internal network topology
- Generic "blocked for security" message for all SSRF blocks
- Logging captures detailed SSRF attempts for monitoring

---

## Coordination Notes

- **Neo (@neo)**: Import shared SSRF config from `@/lib/validation/ssrf-config` in NEO-1.4-002. Remove duplicated `BLOCKED_HOSTNAMES`, `isPrivateIP()`, and `isBlockedDomain()` definitions. Add dependency on BOB-1.4-005.
- **Frank (@frank)**: Review SSRF prevention patterns for security approval
- **Julia (@julia)**: Include SSRF tests in security test suite (Sprint 1.8)

---

## Changelog

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0.0 | 2025-12-12 | Bob Martin | Initial task definitions |
| 1.1.0 | 2025-12-12 | Bob Martin | DEF-015: Added BOB-1.4-005 (Shared SSRF Config) to eliminate duplicated SSRF validation logic between API and client layers. Updated BOB-1.4-003 to import from shared config. |
