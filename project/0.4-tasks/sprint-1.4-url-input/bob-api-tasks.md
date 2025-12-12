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

Implement the process endpoint that initiates document processing for a previously uploaded job. This endpoint atomically transitions the job from PENDING to PROCESSING status using a conditional update pattern to prevent TOCTOU (Time-of-Check-Time-of-Use) race conditions, then returns the SSE stream URL for progress tracking.

#### Acceptance Criteria

- [ ] Endpoint accessible at `POST /api/v1/process`
- [ ] Accepts JSON body with `jobId` field
- [ ] **Uses atomic conditional update** (`UPDATE ... WHERE status = 'PENDING'`) to prevent race conditions
- [ ] Checks affected row count to determine success vs failure
- [ ] Returns 202 Accepted with streamUrl on successful transition
- [ ] Returns 404 with E501 if job not found (0 rows affected + job doesn't exist)
- [ ] Returns 409 with E701 if job already processing/complete (0 rows affected + job exists with different status)
- [ ] Integrates rate limiting middleware
- [ ] Concurrent requests cannot both transition the same job

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

**Atomic State Transition** (TOCTOU-safe):
```typescript
/**
 * Atomically transition a job from PENDING to PROCESSING.
 * Uses a conditional update to prevent TOCTOU race conditions where
 * concurrent requests could both transition the same job.
 *
 * @param jobId - The job ID to transition
 * @returns The updated job if successful
 * @throws AppError E501 (404) if job not found
 * @throws AppError E701 (409) if job not in PENDING status
 */
async function transitionJobToProcessing(jobId: string): Promise<Job> {
  // Atomic conditional update: only succeeds if job exists AND status is PENDING
  // This eliminates the TOCTOU race between checking status and updating it
  const result = await prisma.job.updateMany({
    where: {
      id: jobId,
      status: 'PENDING',
    },
    data: {
      status: 'PROCESSING',
      processingStartedAt: new Date(),
    },
  });

  // Check affected rows to determine outcome
  if (result.count === 0) {
    // No rows updated - either job doesn't exist or status isn't PENDING
    // Query to distinguish between 404 and 409
    const existingJob = await prisma.job.findUnique({
      where: { id: jobId },
      select: { id: true, status: true },
    });

    if (!existingJob) {
      throw new AppError('E501', 'Job not found', 404);
    }

    // Job exists but wasn't PENDING - already processing or complete
    throw new AppError('E701', `Job already ${existingJob.status.toLowerCase()}`, 409);
  }

  // Fetch the updated job to return
  const updatedJob = await prisma.job.findUnique({ where: { id: jobId } });
  return updatedJob!;
}
```

**Alternative using Transaction with Row Lock** (for databases supporting SELECT FOR UPDATE):
```typescript
async function transitionJobToProcessingWithLock(jobId: string): Promise<Job> {
  return prisma.$transaction(async (tx) => {
    // Acquire row lock with SELECT FOR UPDATE
    const jobs = await tx.$queryRaw<Job[]>`
      SELECT * FROM "Job" WHERE id = ${jobId} FOR UPDATE
    `;

    if (jobs.length === 0) {
      throw new AppError('E501', 'Job not found', 404);
    }

    const job = jobs[0];
    if (job.status !== 'PENDING') {
      throw new AppError('E701', `Job already ${job.status.toLowerCase()}`, 409);
    }

    // Update with lock held - guaranteed no race
    return tx.job.update({
      where: { id: jobId },
      data: {
        status: 'PROCESSING',
        processingStartedAt: new Date(),
      },
    });
  });
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

  /** IPv4 private and reserved ranges */
  privateRanges: [
    /^127\./,                           // 127.0.0.0/8 - Loopback (full range)
    /^0\./,                             // 0.0.0.0/8 - "This" network (RFC 1122)
    /^10\./,                            // 10.0.0.0/8 - RFC 1918 Class A private
    /^172\.(1[6-9]|2[0-9]|3[01])\./,    // 172.16.0.0/12 - RFC 1918 Class B private
    /^192\.168\./,                       // 192.168.0.0/16 - RFC 1918 Class C private
    /^100\.(6[4-9]|[7-9][0-9]|1[01][0-9]|12[0-7])\./, // 100.64.0.0/10 - CGNAT (RFC 6598)
  ],

  /** Link-local/APIPA addresses (RFC 3927) and cloud metadata ranges */
  linkLocalRanges: [
    /^169\.254\./,                       // 169.254.0.0/16 - Link-local (includes AWS/GCP/Azure metadata at 169.254.169.254)
  ],

  /** Cloud provider metadata endpoints (explicit for clarity) */
  cloudMetadataHosts: [
    '169.254.169.254',                   // AWS/GCP/Azure Instance Metadata Service (IMDS)
    '169.254.170.2',                     // AWS ECS task metadata
    'metadata.google.internal',          // GCP alternative metadata hostname
  ],

  /** IPv6 private and reserved ranges */
  ipv6PrivateRanges: [
    /^::1$/,                             // IPv6 loopback
    /^::$/,                              // IPv6 unspecified
    /^::ffff:127\./,                     // IPv4-mapped loopback
    /^::ffff:0\./,                       // IPv4-mapped "this" network
    /^::ffff:10\./,                      // IPv4-mapped Class A private
    /^::ffff:172\.(1[6-9]|2[0-9]|3[01])\./, // IPv4-mapped Class B private
    /^::ffff:192\.168\./,                // IPv4-mapped Class C private
    /^fc[0-9a-f]{2}:/i,                  // fc00::/7 - Unique Local Addresses (ULA, RFC 4193)
    /^fd[0-9a-f]{2}:/i,                  // fd00::/8 - ULA (commonly used subset)
    /^fe[89ab][0-9a-f]:/i,               // fe80::/10 - Link-local (RFC 4291)
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
 * Check if an IP address string matches any SSRF blocking rule.
 * This is a synchronous check used for both direct IP hostnames and resolved IPs.
 * @param ip - The IP address to check (already normalized)
 * @returns Object with blocked status and optional reason
 */
function checkIPAgainstRules(ip: string): { blocked: boolean; reason?: string } {
  // Check IPv4 private and reserved ranges
  for (const range of SSRF_RULES.privateRanges) {
    if (range.test(ip)) {
      return { blocked: true, reason: 'Private/reserved IP addresses are not allowed' };
    }
  }

  // Check link-local addresses (RFC 3927)
  for (const range of SSRF_RULES.linkLocalRanges) {
    if (range.test(ip)) {
      return { blocked: true, reason: 'Link-local addresses are not allowed' };
    }
  }

  // Check cloud metadata hosts explicitly
  if (SSRF_RULES.cloudMetadataHosts.includes(ip)) {
    return { blocked: true, reason: 'Cloud metadata endpoints are not allowed' };
  }

  // Check IPv6 private and reserved ranges
  for (const range of SSRF_RULES.ipv6PrivateRanges) {
    if (range.test(ip)) {
      return { blocked: true, reason: 'Private/reserved IPv6 addresses are not allowed' };
    }
  }

  return { blocked: false };
}

/**
 * Check if a hostname matches any SSRF blocking rule (synchronous, hostname-only check).
 * The hostname is normalized internally (trimmed, lowercased, trailing dot removed,
 * IPv6 brackets stripped) so callers don't need to pre-normalize.
 *
 * NOTE: This function only checks the hostname string itself. For complete SSRF protection,
 * use checkSSRFRulesWithDNS() which also resolves the hostname and validates resolved IPs.
 *
 * @param hostname - The hostname to check
 * @returns Object with blocked status and optional reason
 */
export function checkSSRFRules(hostname: string): { blocked: boolean; reason?: string } {
  // Normalize hostname for consistent checking
  const normalizedHostname = normalizeHostname(hostname);

  // Check blocked hostnames (localhost variants)
  if (SSRF_RULES.blockedHostnames.includes(normalizedHostname)) {
    return { blocked: true, reason: 'Localhost addresses are not allowed' };
  }

  // Check cloud metadata hostnames (e.g., metadata.google.internal)
  if (SSRF_RULES.cloudMetadataHosts.includes(normalizedHostname)) {
    return { blocked: true, reason: 'Cloud metadata endpoints are not allowed' };
  }

  // Check internal domains
  if (SSRF_RULES.internalDomains.test(normalizedHostname)) {
    return { blocked: true, reason: 'Internal domains are not allowed' };
  }

  // Check if hostname is an IP address and validate against IP rules
  const ipCheck = checkIPAgainstRules(normalizedHostname);
  if (ipCheck.blocked) {
    return ipCheck;
  }

  return { blocked: false };
}

/**
 * Check SSRF rules with DNS resolution (async).
 * Performs A and AAAA lookups on the hostname and validates each resolved IP
 * against the configured blocking rules. This prevents DNS rebinding and
 * hostname-to-internal-IP mapping attacks.
 *
 * @param hostname - The hostname to check and resolve
 * @param options - Optional configuration
 * @param options.timeout - DNS resolution timeout in milliseconds (default: 5000)
 * @param options.failOnDNSError - If true, block when DNS resolution fails (default: true for security)
 * @returns Promise resolving to blocked status with reason, or resolvedIPs on success
 *
 * @example
 * // Basic usage
 * const result = await checkSSRFRulesWithDNS('example.com');
 * if (result.blocked) {
 *   console.error(`Blocked: ${result.reason}`);
 * }
 *
 * @example
 * // With custom timeout
 * const result = await checkSSRFRulesWithDNS('example.com', { timeout: 3000 });
 */
export async function checkSSRFRulesWithDNS(
  hostname: string,
  options: { timeout?: number; failOnDNSError?: boolean } = {}
): Promise<{ blocked: boolean; reason?: string; resolvedIPs?: string[] }> {
  const { timeout = 5000, failOnDNSError = true } = options;

  // First, perform synchronous hostname checks
  const hostnameCheck = checkSSRFRules(hostname);
  if (hostnameCheck.blocked) {
    return hostnameCheck;
  }

  const normalizedHostname = normalizeHostname(hostname);

  // Skip DNS resolution for IP addresses (already validated above)
  if (isIPAddress(normalizedHostname)) {
    return { blocked: false, resolvedIPs: [normalizedHostname] };
  }

  // Perform DNS resolution with timeout
  let resolvedIPs: string[] = [];
  try {
    resolvedIPs = await resolveHostnameWithTimeout(normalizedHostname, timeout);
  } catch (error) {
    // DNS resolution failed
    if (failOnDNSError) {
      return {
        blocked: true,
        reason: `DNS resolution failed: ${error instanceof Error ? error.message : 'Unknown error'}. Blocking for security.`,
      };
    }
    // If failOnDNSError is false, allow the request (caller handles fetch errors)
    return { blocked: false, resolvedIPs: [] };
  }

  // Validate each resolved IP against SSRF rules
  for (const ip of resolvedIPs) {
    const ipCheck = checkIPAgainstRules(ip);
    if (ipCheck.blocked) {
      return {
        blocked: true,
        reason: `Hostname '${hostname}' resolves to blocked IP ${ip}: ${ipCheck.reason}`,
      };
    }
  }

  return { blocked: false, resolvedIPs };
}

/**
 * Check if a string is an IP address (v4 or v6).
 */
function isIPAddress(value: string): boolean {
  // IPv4 pattern
  const ipv4Pattern = /^(\d{1,3}\.){3}\d{1,3}$/;
  // IPv6 pattern (simplified - matches most common formats)
  const ipv6Pattern = /^([0-9a-fA-F]{0,4}:){2,7}[0-9a-fA-F]{0,4}$/;

  return ipv4Pattern.test(value) || ipv6Pattern.test(value);
}

/**
 * Resolve hostname to IP addresses with timeout.
 * Performs both A (IPv4) and AAAA (IPv6) lookups.
 *
 * @param hostname - The hostname to resolve
 * @param timeoutMs - Timeout in milliseconds
 * @returns Array of resolved IP addresses
 * @throws Error if resolution fails or times out
 */
async function resolveHostnameWithTimeout(hostname: string, timeoutMs: number): Promise<string[]> {
  const dns = await import('node:dns');
  const { promisify } = await import('node:util');
  const resolve4 = promisify(dns.resolve4);
  const resolve6 = promisify(dns.resolve6);

  const timeoutPromise = new Promise<never>((_, reject) => {
    setTimeout(() => reject(new Error(`DNS resolution timed out after ${timeoutMs}ms`)), timeoutMs);
  });

  // Perform A and AAAA lookups in parallel
  const [ipv4Results, ipv6Results] = await Promise.race([
    Promise.all([
      resolve4(hostname).catch(() => [] as string[]),
      resolve6(hostname).catch(() => [] as string[]),
    ]),
    timeoutPromise,
  ]);

  const allIPs = [...ipv4Results, ...ipv6Results];

  if (allIPs.length === 0) {
    throw new Error(`No DNS records found for hostname: ${hostname}`);
  }

  return allIPs;
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

**IPv4 Blocking:**
- [ ] Blocks localhost/127.0.0.1 and full 127.0.0.0/8 loopback range
- [ ] Blocks 0.0.0.0/8 ("this" network, RFC 1122)
- [ ] Blocks 10.0.0.0/8 (RFC 1918 Class A private)
- [ ] Blocks 172.16.0.0/12 (RFC 1918 Class B private)
- [ ] Blocks 192.168.0.0/16 (RFC 1918 Class C private)
- [ ] Blocks 100.64.0.0/10 (CGNAT, RFC 6598)
- [ ] Blocks 169.254.0.0/16 (link-local, RFC 3927)

**IPv6 Blocking:**
- [ ] Blocks ::1 (IPv6 loopback)
- [ ] Blocks :: (IPv6 unspecified)
- [ ] Blocks fc00::/7 (Unique Local Addresses, RFC 4193)
- [ ] Blocks fe80::/10 (IPv6 link-local, RFC 4291)
- [ ] Blocks IPv4-mapped IPv6 addresses (::ffff:127.0.0.0/8, ::ffff:10.0.0.0/8, etc.)

**Cloud Metadata Protection:**
- [ ] Blocks 169.254.169.254 (AWS/GCP/Azure IMDS)
- [ ] Blocks 169.254.170.2 (AWS ECS task metadata)
- [ ] Blocks metadata.google.internal (GCP metadata hostname)

**Domain Blocking:**
- [ ] Blocks *.hx.dev.local internal domains

**DNS Resolution Check (async):**
- [ ] `checkSSRFRulesWithDNS()` performs A and AAAA lookups on hostnames
- [ ] Each resolved IP is validated against all blocking rules
- [ ] Blocks requests where any resolved IP matches a blocked range
- [ ] Configurable timeout for DNS resolution (default: 5000ms)
- [ ] Configurable behavior on DNS failure (default: block for security)
- [ ] Returns descriptive reason including the blocked IP when DNS-resolved IP is blocked

**Error Handling:**
- [ ] Returns E104 error code for blocked URLs
- [ ] DNS timeout returns appropriate error with timeout duration
- [ ] DNS NXDOMAIN/SERVFAIL returns appropriate error

#### Technical Notes

**IMPORTANT**: This task now imports from the shared SSRF config (BOB-1.4-005) instead of defining rules inline. This eliminates duplication with Neo's client-side validation.

```typescript
// File: src/lib/validation/url.ts

import { checkSSRFRules, checkSSRFRulesWithDNS } from './ssrf-config';

export interface ValidationResult {
  valid: boolean;
  error?: string;
  message?: string;
}

/**
 * Check if a URL is blocked by SSRF prevention rules (synchronous, hostname-only).
 * For client-side or quick checks. Does NOT perform DNS resolution.
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

/**
 * Check if a URL is blocked by SSRF rules with DNS resolution (async).
 * This performs A/AAAA lookups and validates each resolved IP.
 * Use this for server-side validation before making outbound requests.
 */
export async function isURLBlockedWithDNS(
  urlString: string,
  options?: { timeout?: number; failOnDNSError?: boolean }
): Promise<{ blocked: boolean; reason?: string }> {
  try {
    const url = new URL(urlString);
    const hostname = url.hostname.toLowerCase();
    return await checkSSRFRulesWithDNS(hostname, options);
  } catch {
    return { blocked: true, reason: 'Invalid URL format' };
  }
}

/**
 * Synchronous URL validation (hostname checks only).
 * Suitable for client-side validation or quick server-side pre-checks.
 */
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

  // Check SSRF using shared config (synchronous hostname check)
  if (isURLBlocked(urlString)) {
    return { valid: false, error: 'E104', message: 'URL blocked for security' };
  }

  return { valid: true };
}

/**
 * Full async URL validation with DNS resolution check.
 * Use this for server-side validation before making outbound requests.
 *
 * @param urlString - The URL to validate
 * @param options - DNS resolution options
 * @returns Validation result with optional reason for blocking
 */
export async function validateURLWithDNS(
  urlString: string,
  options?: { timeout?: number; failOnDNSError?: boolean }
): Promise<ValidationResult & { reason?: string }> {
  // Perform synchronous checks first (fast-fail)
  const syncResult = validateURL(urlString);
  if (!syncResult.valid) {
    return syncResult;
  }

  // Perform async DNS resolution check
  const dnsResult = await isURLBlockedWithDNS(urlString, options);
  if (dnsResult.blocked) {
    return {
      valid: false,
      error: 'E104',
      message: 'URL blocked for security',
      reason: dnsResult.reason,
    };
  }

  return { valid: true };
}
```

**Usage in API Route:**
```typescript
// File: src/app/api/v1/upload/route.ts (or process/route.ts)

import { validateURLWithDNS } from '@/lib/validation/url';

async function handleURLUpload(urlString: string) {
  // Full SSRF validation with DNS resolution
  const validation = await validateURLWithDNS(urlString, {
    timeout: 5000,       // 5 second DNS timeout
    failOnDNSError: true // Block if DNS fails (safer)
  });

  if (!validation.valid) {
    throw new AppError(validation.error!, validation.message!, 400);
  }

  // Safe to fetch the URL - all resolved IPs have been validated
  const response = await fetch(urlString);
  // ...
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

**Synchronous Tests (checkSSRFRules / isURLBlocked):**
- [ ] Tests for localhost variants (localhost, 127.0.0.1, 127.x.x.x, [::1], 0.0.0.0, 0.x.x.x)
- [ ] Tests for RFC 1918 private ranges (10.x, 172.16-31.x, 192.168.x)
- [ ] Tests for CGNAT range (100.64.0.0/10)
- [ ] Tests for link-local addresses (169.254.x.x)
- [ ] Tests for cloud metadata endpoints (169.254.169.254, metadata.google.internal)
- [ ] Tests for IPv6 ULA (fc00::/7, fd00::/8)
- [ ] Tests for IPv6 link-local (fe80::/10)
- [ ] Tests for IPv4-mapped IPv6 addresses (::ffff:127.0.0.1, ::ffff:10.0.0.1)
- [ ] Tests for internal domain blocking (*.hx.dev.local)
- [ ] Tests for valid URLs (should pass)
- [ ] Tests for edge cases (encoded characters, punycode, trailing dots)

**Async Tests (checkSSRFRulesWithDNS / isURLBlockedWithDNS):**
- [ ] Tests for DNS resolution that returns blocked IPs (mocked)
- [ ] Tests for DNS timeout behavior
- [ ] Tests for DNS failure handling (NXDOMAIN, SERVFAIL)
- [ ] Tests for failOnDNSError option (true/false)
- [ ] Tests for hostnames resolving to multiple IPs (any blocked = block all)
- [ ] Tests for valid hostname resolution (should pass)

#### Technical Notes

```typescript
// File: src/lib/validation/url.test.ts

import { isURLBlocked, isURLBlockedWithDNS } from './url';
import { checkSSRFRules, checkSSRFRulesWithDNS } from './ssrf-config';

describe('SSRF Prevention', () => {
  describe('isURLBlocked (synchronous)', () => {
    // Localhost variants (full 127.0.0.0/8 range)
    it.each([
      'http://localhost/path',
      'http://127.0.0.1/path',
      'http://127.255.255.255/path',
      'http://[::1]/path',
      'http://0.0.0.0/path',
      'http://0.255.255.255/path',
    ])('should block localhost: %s', (url) => {
      expect(isURLBlocked(url)).toBe(true);
    });

    // RFC 1918 Private ranges
    it.each([
      'http://10.0.0.1/path',
      'http://10.255.255.255/path',
      'http://172.16.0.1/path',
      'http://172.31.255.255/path',
      'http://192.168.0.1/path',
      'http://192.168.255.255/path',
    ])('should block RFC 1918 private IP: %s', (url) => {
      expect(isURLBlocked(url)).toBe(true);
    });

    // CGNAT range (100.64.0.0/10)
    it.each([
      'http://100.64.0.1/path',
      'http://100.127.255.255/path',
    ])('should block CGNAT IP: %s', (url) => {
      expect(isURLBlocked(url)).toBe(true);
    });

    // Cloud metadata endpoints
    it.each([
      'http://169.254.169.254/latest/meta-data/',
      'http://169.254.170.2/v2/credentials',
      'http://metadata.google.internal/computeMetadata/v1/',
    ])('should block cloud metadata: %s', (url) => {
      expect(isURLBlocked(url)).toBe(true);
    });

    // IPv6 private ranges
    it.each([
      'http://[fc00::1]/path',          // ULA
      'http://[fd12:3456::1]/path',     // ULA (commonly used)
      'http://[fe80::1]/path',          // Link-local
      'http://[::ffff:127.0.0.1]/path', // IPv4-mapped loopback
      'http://[::ffff:10.0.0.1]/path',  // IPv4-mapped private
    ])('should block IPv6 private: %s', (url) => {
      expect(isURLBlocked(url)).toBe(true);
    });

    // Valid URLs
    it.each([
      'https://example.com/document.pdf',
      'https://docs.google.com/document/d/123',
      'http://8.8.8.8/path',            // Public IP
      'http://203.0.113.1/path',        // TEST-NET-3 (documentation, but public)
    ])('should allow valid URL: %s', (url) => {
      expect(isURLBlocked(url)).toBe(false);
    });
  });

  describe('isURLBlockedWithDNS (async)', () => {
    // Mock DNS resolution
    beforeEach(() => {
      jest.resetModules();
    });

    it('should block hostname resolving to private IP', async () => {
      // Mock dns.resolve4 to return a private IP
      jest.doMock('node:dns', () => ({
        resolve4: jest.fn((hostname, cb) => cb(null, ['10.0.0.1'])),
        resolve6: jest.fn((hostname, cb) => cb(null, [])),
      }));

      const result = await isURLBlockedWithDNS('http://evil.example.com/path');
      expect(result.blocked).toBe(true);
      expect(result.reason).toContain('10.0.0.1');
    });

    it('should block when DNS resolution times out', async () => {
      const result = await isURLBlockedWithDNS('http://slow.example.com/path', {
        timeout: 1, // 1ms timeout
      });
      expect(result.blocked).toBe(true);
      expect(result.reason).toContain('timed out');
    });

    it('should allow when failOnDNSError is false', async () => {
      const result = await isURLBlockedWithDNS('http://nonexistent.example.com/path', {
        timeout: 1,
        failOnDNSError: false,
      });
      expect(result.blocked).toBe(false);
    });

    it('should allow hostname resolving to public IP', async () => {
      jest.doMock('node:dns', () => ({
        resolve4: jest.fn((hostname, cb) => cb(null, ['93.184.216.34'])), // example.com
        resolve6: jest.fn((hostname, cb) => cb(null, [])),
      }));

      const result = await isURLBlockedWithDNS('http://example.com/path');
      expect(result.blocked).toBe(false);
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
   - Full 127.0.0.0/8 loopback range
   - 0.0.0.0/8 "this" network
   - RFC 1918 private ranges (10/8, 172.16/12, 192.168/16)
   - CGNAT range (100.64.0.0/10)
   - Link-local (169.254.0.0/16)

2. **DNS Rebinding / Hostname-to-IP Mapping**: `checkSSRFRulesWithDNS()` performs A/AAAA lookups and validates each resolved IP against blocking rules
   - Blocks hostnames resolving to any private/reserved IP
   - Configurable timeout (default 5s) prevents DNS-based DoS
   - Configurable fail-closed behavior on DNS errors (default: block)

3. **Cloud Metadata Attacks**: Explicit blocking of cloud provider metadata endpoints
   - 169.254.169.254 (AWS/GCP/Azure IMDS)
   - 169.254.170.2 (AWS ECS task metadata)
   - metadata.google.internal (GCP hostname)

4. **URL Encoding Bypass**: Tests include encoded character variants

5. **IPv6 Bypass**: Comprehensive IPv6 blocking
   - Loopback (::1, ::)
   - Unique Local Addresses (fc00::/7)
   - Link-local (fe80::/10)
   - IPv4-mapped addresses (::ffff:127.0.0.1, ::ffff:10.x.x.x, etc.)

6. **Domain Bypass**: Internal domain patterns blocked (*.hx.dev.local)

### Error Response Security

- Error messages do not reveal internal network topology
- Generic "blocked for security" message for all SSRF blocks
- Logging captures detailed SSRF attempts for monitoring

### Race Condition Prevention (TOCTOU)

The process endpoint uses an **atomic conditional update** pattern to prevent Time-of-Check-Time-of-Use (TOCTOU) race conditions:

1. **Problem**: A validate-then-update flow (`findUnique` → check status → `update`) allows concurrent requests to both read PENDING status before either updates it
2. **Solution**: Use `updateMany` with `WHERE id = :jobId AND status = 'PENDING'` to atomically check-and-update in a single database operation
3. **Detection**: Check `result.count` - if 0, query the job to distinguish between 404 (not found) and 409 (wrong status)
4. **Guarantee**: Only one concurrent request can successfully transition a job; others receive 409 Conflict

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
| 1.2.0 | 2025-12-12 | Bob Martin | Fixed TOCTOU race condition in BOB-1.4-001: Replaced validate-then-update flow with atomic conditional update using `updateMany` with status predicate. Added alternative transaction-based approach with row locking. Updated acceptance criteria to require atomic operations. |
| 1.3.0 | 2025-12-12 | Bob Martin | Enhanced SSRF protection with comprehensive coverage: (1) Expanded SSRF_RULES to include 127.0.0.0/8, 0.0.0.0/8, 100.64.0.0/10 (CGNAT), IPv6 ULA fc00::/7, IPv6 link-local fe80::/10, and cloud metadata endpoints (AWS/GCP/Azure IMDS). (2) Added async `checkSSRFRulesWithDNS()` function that performs A/AAAA lookups and validates resolved IPs against blocking rules, with configurable timeout and failure handling. (3) Updated BOB-1.4-003 acceptance criteria to fully specify DNS resolution behavior. (4) Expanded BOB-1.4-004 test cases to cover all new ranges and async DNS resolution scenarios. |
