# Tasks: Redis Integration - Sprint 1.3

**Sprint**: 1.3 - File Upload System
**Agent**: Sri Venkateswaran (`@sri`)
**Role**: Redis SME (Consultant)
**Lead**: Neo (`@neo`)
**Input**: Implementation Plan v1.2.0, Detailed Specification v1.2.0
**Prerequisites**: Sprint 1.2 completed (Redis client, rate limiting, session management operational)

---

## Overview

This task file covers Redis integration support for Sprint 1.3 File Upload System. Sri provides consultation and review for rate limiting middleware integration on the upload route. The primary implementation is handled by Neo with Sri providing Redis expertise.

---

## Task List

### SRI-1.3-001: Review Rate Limiting Integration for Upload Route

**Description**: Provide technical review and consultation for integrating the sliding window rate limiting middleware (from Sprint 1.2) with the file upload API route. Ensure proper circuit breaker fallback behavior and correct session ID extraction.

**File**: `src/app/api/v1/upload/route.ts` (review)

**Acceptance Criteria**:
- [ ] Rate limiting middleware correctly applied to POST `/api/v1/upload`
- [ ] Session ID extracted from cookie or header for rate limit key
- [ ] Circuit breaker fallback strategy configured (recommend IN_MEMORY)
- [ ] Rate limit headers returned in response: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`
- [ ] 429 Too Many Requests returned when limit exceeded with `Retry-After` header

**Dependencies**:
- SRI-1.2-003: Rate limiting with sliding window implemented
- SRI-1.2-002: Circuit breaker implemented
- NEO-1.3-xxx: Upload route created

**Effort**: 15 minutes (review and consultation)

**Deliverables**:
- Code review approval for upload route rate limiting integration
- Technical guidance document if needed

**Technical Notes**:
```typescript
// Rate limiting middleware usage per Implementation Plan Sprint 1.3 Task 8
import { checkRateLimitWithFallback, FallbackStrategy } from '@/lib/redis/rate-limit';

export async function POST(request: Request) {
  // Extract session ID from cookie
  const sessionId = request.cookies.get('session_id')?.value;

  if (!sessionId) {
    return NextResponse.json({ error: 'Session required' }, { status: 401 });
  }

  // Check rate limit with circuit breaker fallback
  const rateLimitResult = await checkRateLimitWithFallback(
    sessionId,
    FallbackStrategy.IN_MEMORY
  );

  if (!rateLimitResult.allowed) {
    return NextResponse.json(
      { error: 'Rate limit exceeded', retryAfter: rateLimitResult.retryAfter },
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

  // Set rate limit headers on successful response
  const response = NextResponse.json({ ... });
  response.headers.set('X-RateLimit-Limit', '10');
  response.headers.set('X-RateLimit-Remaining', String(rateLimitResult.remaining));

  return response;
}
```

**Review Checklist**:
- [ ] Session ID extraction is secure (no user-controlled input in key)
- [ ] Fallback strategy appropriate for upload endpoint
- [ ] Rate limit headers follow RFC 6585 conventions
- [ ] Error response includes actionable retry information
- [ ] Logging captures rate limit events for monitoring

---

### SRI-1.3-002: Support Session Activity Update on Upload

**Description**: Provide consultation for updating session activity when files are uploaded. This ensures sliding expiration works correctly and job counts are tracked.

**File**: `src/lib/redis/session.ts` (enhancement)

**Acceptance Criteria**:
- [ ] Session `lastActivity` updated on successful upload
- [ ] Session `jobCount` incremented on job creation
- [ ] Session `activeJobIds` updated when processing begins
- [ ] TTL reset (sliding expiration) after activity update

**Dependencies**:
- SRI-1.2-004: Session management implemented

**Effort**: 10 minutes (consultation)

**Deliverables**:
- Technical guidance for session update integration

**Technical Notes**:
```typescript
// Session update on upload
import { updateSessionActivity } from '@/lib/redis/session';
import { redis } from '@/lib/redis/client';

// After successful upload and job creation:
await updateSessionActivity(sessionId);

// Increment job count (atomic operation)
const key = `hx-docling:session:${sessionId}`;
await redis.hincrby(key, 'jobCount', 1);

// Add to activeJobIds when processing starts
const session = await getSession(sessionId);
session.activeJobIds.push(newJobId);
await redis.hset(key, 'activeJobIds', JSON.stringify(session.activeJobIds));
```

---

## Summary

| Task ID | Title | Effort | Dependencies |
|---------|-------|--------|--------------|
| SRI-1.3-001 | Review Rate Limiting Integration for Upload Route | 15m | SRI-1.2-003 |
| SRI-1.3-002 | Support Session Activity Update on Upload | 10m | SRI-1.2-004 |

**Total Effort**: 25 minutes

---

## Integration Points

The following Redis operations are used by the upload system:

| Operation | Purpose | Redis Command |
|-----------|---------|---------------|
| Rate limit check | Enforce 10 req/min | EVAL (Lua script) |
| Session activity update | Reset TTL, update lastActivity | MULTI + HSET + EXPIRE |
| Job count increment | Track uploads per session | HINCRBY |
| Active job tracking | Support multiple tabs | HSET (JSON array) |

---

## Monitoring Recommendations

For the upload endpoint, monitor these Redis metrics:

1. **Rate Limit Events**:
   - Track allowed vs. denied requests
   - Alert on high denial rates (possible attack or misconfiguration)

2. **Session Activity**:
   - Monitor session creation/expiration rates
   - Track activeJobIds array size (prevent unbounded growth)

3. **Circuit Breaker State**:
   - Alert when circuit opens
   - Track fallback usage frequency

```typescript
// Suggested logging for monitoring
logger.info('rate_limit_check', {
  sessionId,
  allowed: rateLimitResult.allowed,
  remaining: rateLimitResult.remaining,
  circuitState: redisCircuitBreaker.getState().state,
});
```
