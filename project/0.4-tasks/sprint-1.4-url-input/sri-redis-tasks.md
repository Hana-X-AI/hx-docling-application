# Tasks: Redis Integration - Sprint 1.4

**Sprint**: 1.4 - URL Input with Security
**Agent**: Sri Venkateswaran (`@sri`)
**Role**: Redis SME (Consultant)
**Lead**: Neo (`@neo`)
**Input**: Implementation Plan v1.2.0, Detailed Specification v1.2.0
**Prerequisites**: Sprint 1.3 completed (Rate limiting integrated with upload route)

---

## Overview

This task file covers Redis integration support for Sprint 1.4 URL Input with Security. Sri provides consultation and review for rate limiting middleware integration on the URL processing endpoint. The URL endpoint follows the same rate limiting pattern established in Sprint 1.3.

---

## Task List

### SRI-1.4-001: Review Rate Limiting Integration for URL Route

**Description**: Provide technical review and consultation for integrating the sliding window rate limiting middleware with the URL processing endpoint. Ensure consistent behavior with the file upload route rate limiting.

**File**: `src/app/api/v1/process/url/route.ts` (review) or applicable URL processing route

**Acceptance Criteria**:
- [ ] Rate limiting middleware correctly applied to URL processing endpoint
- [ ] Same rate limit (10 req/min) applied consistently with upload route
- [ ] Session ID extracted consistently with upload route
- [ ] Circuit breaker fallback strategy matches upload route (IN_MEMORY)
- [ ] Rate limit headers returned in response
- [ ] 429 Too Many Requests returned with `Retry-After` header when limit exceeded

**Dependencies**:
- SRI-1.2-003: Rate limiting with sliding window implemented
- SRI-1.3-001: Upload route rate limiting reviewed
- NEO-1.4-xxx: URL processing route created

**Effort**: 10 minutes (review and consultation)

**Deliverables**:
- Code review approval for URL route rate limiting integration
- Consistency verification with upload route implementation

**Technical Notes**:
```typescript
// Rate limiting for URL endpoint - same pattern as upload
import { checkRateLimitWithFallback, FallbackStrategy } from '@/lib/redis/rate-limit';

export async function POST(request: Request) {
  const sessionId = request.cookies.get('session_id')?.value;

  if (!sessionId) {
    return NextResponse.json(
      { code: 'E401', message: 'Session required' },
      { status: 401 }
    );
  }

  // Rate limiting - shared limit with file upload
  const rateLimitResult = await checkRateLimitWithFallback(
    sessionId,
    FallbackStrategy.IN_MEMORY
  );

  if (!rateLimitResult.allowed) {
    return NextResponse.json(
      {
        code: 'E107',
        message: 'Rate limit exceeded',
        details: { retryAfter: rateLimitResult.retryAfter }
      },
      {
        status: 429,
        headers: {
          'X-RateLimit-Limit': '10',
          'X-RateLimit-Remaining': '0',
          'Retry-After': String(rateLimitResult.retryAfter || 60),
        },
      }
    );
  }

  // Continue with URL processing...
}
```

**Review Checklist**:
- [ ] Rate limiting applied before SSRF validation (fail fast)
- [ ] Same session ID extraction as upload route
- [ ] Error code E107 used for rate limit exceeded (per error catalog)
- [ ] Response headers consistent with upload route
- [ ] Logging includes rate limit event details

---

### SRI-1.4-002: Verify Shared Rate Limit Behavior

**Description**: Verify that rate limiting is correctly shared between file upload and URL processing endpoints. Both endpoints should consume from the same rate limit window per session.

**File**: N/A (verification task)

**Acceptance Criteria**:
- [ ] File upload and URL processing share the same rate limit key: `hx-docling:rate:{sessionId}`
- [ ] 5 file uploads + 5 URL requests = 10 total (limit reached)
- [ ] Rate limit window slides correctly across both endpoints
- [ ] Retry-After header accurate regardless of which endpoint exhausted the limit

**Dependencies**:
- SRI-1.3-001: Upload route rate limiting integrated
- SRI-1.4-001: URL route rate limiting reviewed

**Effort**: 10 minutes (verification)

**Deliverables**:
- Verification test results confirming shared rate limit behavior

**Technical Notes**:
```bash
# Manual verification commands
# Session rate limit key is shared between endpoints
redis-cli -h hx-redis-server ZRANGE hx-docling:rate:test-session 0 -1 WITHSCORES

# After 5 uploads and 5 URL requests, expect 10 entries in the sorted set
# Both endpoints should show 0 remaining after 10 total requests
```

**Test Scenario**:
```typescript
// Integration test for shared rate limit
describe('Shared Rate Limit', () => {
  it('shares rate limit between upload and URL endpoints', async () => {
    const sessionId = 'test-session';

    // Make 5 upload requests
    for (let i = 0; i < 5; i++) {
      const response = await fetch('/api/v1/upload', {
        method: 'POST',
        headers: { Cookie: `session_id=${sessionId}` },
        body: formData,
      });
      expect(response.status).toBe(200);
    }

    // Make 4 URL requests (should succeed)
    for (let i = 0; i < 4; i++) {
      const response = await fetch('/api/v1/process/url', {
        method: 'POST',
        headers: { Cookie: `session_id=${sessionId}` },
        body: JSON.stringify({ url: 'https://example.com' }),
      });
      expect(response.status).toBe(200);
    }

    // 10th request should succeed
    const response9 = await fetch('/api/v1/process/url', {
      method: 'POST',
      headers: { Cookie: `session_id=${sessionId}` },
      body: JSON.stringify({ url: 'https://example.com' }),
    });
    expect(response9.status).toBe(200);

    // 11th request should fail with 429
    const response10 = await fetch('/api/v1/upload', {
      method: 'POST',
      headers: { Cookie: `session_id=${sessionId}` },
      body: formData,
    });
    expect(response10.status).toBe(429);
    expect(response10.headers.get('Retry-After')).toBeDefined();
  });
});
```

---

## Summary

| Task ID | Title | Effort | Dependencies |
|---------|-------|--------|--------------|
| SRI-1.4-001 | Review Rate Limiting Integration for URL Route | 10m | SRI-1.2-003, SRI-1.3-001 |
| SRI-1.4-002 | Verify Shared Rate Limit Behavior | 10m | SRI-1.4-001 |

**Total Effort**: 20 minutes

---

## Integration Notes

### Rate Limiting Architecture

```
                  +------------------+
                  |   Client (User)  |
                  +--------+---------+
                           |
              +------------+------------+
              |                         |
    +---------v---------+     +---------v---------+
    | POST /api/v1/     |     | POST /api/v1/     |
    |     upload        |     |   process/url     |
    +---------+---------+     +---------+---------+
              |                         |
              +------------+------------+
                           |
              +------------v------------+
              |  Rate Limit Middleware  |
              |  checkRateLimitWith     |
              |      Fallback()         |
              +------------+------------+
                           |
              +------------v------------+
              |    Redis Sorted Set     |
              | hx-docling:rate:{sid}   |
              +-------------------------+
```

### Error Code Reference

| Code | HTTP Status | Message | Scenario |
|------|-------------|---------|----------|
| E107 | 429 | Rate limit exceeded | More than 10 requests in 60 seconds |

### Response Headers

All rate-limited endpoints must return these headers:

| Header | Description | Example |
|--------|-------------|---------|
| `X-RateLimit-Limit` | Maximum requests per window | `10` |
| `X-RateLimit-Remaining` | Requests remaining in window | `7` |
| `X-RateLimit-Reset` | Unix timestamp when window resets | `1702400000` |
| `Retry-After` | Seconds until retry allowed (429 only) | `45` |
