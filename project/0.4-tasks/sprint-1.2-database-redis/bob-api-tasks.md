# API Tasks: Sprint 1.2 - Database & Redis Integration

**Sprint**: 1.2 - Database & Redis Integration
**Author**: Bob Martin (@bob) - API Design SME
**Created**: 2025-12-12
**Status**: PENDING
**Dependencies**: Sprint 1.1 Complete

---

## Overview

This task file covers API-related deliverables for Sprint 1.2, focusing on the health check endpoint, centralized validation middleware, and rate limiting middleware implementation.

---

## Tasks

### BOB-1.2-001: Implement Health Check Endpoint

| Field | Value |
|-------|-------|
| **Task ID** | BOB-1.2-001 |
| **Title** | Implement comprehensive health check endpoint at /api/v1/health |
| **Priority** | P1 (Critical) |
| **Effort** | 45 minutes |
| **Dependencies** | William's database connection (Task 2), Redis connection (Task 5) |

#### Description

Create a production-ready health check endpoint that reports the status of all system dependencies (PostgreSQL, Redis, MCP server, file storage). The endpoint must follow the API specification in Section 4.7 and return appropriate HTTP status codes based on service health.

#### Acceptance Criteria

- [ ] Endpoint accessible at `GET /api/v1/health`
- [ ] Returns 200 OK when all services healthy
- [ ] Returns 503 Service Unavailable when critical service fails (postgres, redis, mcp)
- [ ] Returns 200 OK with `status: "degraded"` when non-critical service fails (fileStorage)
- [ ] Response includes latency measurements for each service
- [ ] Response includes application version from package.json
- [ ] Response includes uptime in seconds
- [ ] All checks have timestamps in ISO 8601 format
- [ ] Response follows API envelope pattern (Section 4.8)

#### Technical Notes

```typescript
// File: src/app/api/v1/health/route.ts

interface HealthCheckResult {
  status: 'healthy' | 'degraded' | 'unhealthy';
  timestamp: string;
  version: string;
  uptime: number;
  checks: {
    mcp: ServiceCheck;
    postgres: ServiceCheck;
    redis: ServiceCheck;
    fileStorage: ServiceCheck;
  };
}

interface ServiceCheck {
  status: 'ok' | 'error';
  latency: number; // milliseconds
  lastCheck: string;
  error?: string;
}
```

**Health Check Logic**:
- Execute all checks in parallel using `Promise.allSettled`
- Set timeout of 5 seconds per check
- Critical services: mcp, postgres, redis
- Non-critical services: fileStorage

#### Deliverables

- `src/app/api/v1/health/route.ts` - Health endpoint implementation
- `src/lib/health/checks.ts` - Individual health check functions
- `src/lib/health/types.ts` - TypeScript interfaces

---

### BOB-1.2-002: Implement Centralized Validation Middleware

| Field | Value |
|-------|-------|
| **Task ID** | BOB-1.2-002 |
| **Title** | Create reusable validation middleware with Zod schemas |
| **Priority** | P1 (Critical) |
| **Effort** | 30 minutes |
| **Dependencies** | Sprint 1.1 (Zod installed) |

#### Description

Implement centralized request validation middleware using Zod schemas. This middleware will be used by all API routes to validate request body, query parameters, and path parameters, ensuring consistent error responses across the application.

#### Acceptance Criteria

- [ ] `withValidation()` HOF wraps API route handlers
- [ ] Validates request body for POST/PUT/PATCH requests
- [ ] Validates query parameters for all requests
- [ ] Validates path parameters from route context
- [ ] Returns structured error response with code E100
- [ ] Includes Zod validation errors in response details
- [ ] Handles malformed JSON gracefully
- [ ] Type-safe validated data passed to handler

#### Technical Notes

```typescript
// File: src/lib/middleware/validate.ts

interface ValidationConfig {
  body?: ZodSchema;
  query?: ZodSchema;
  params?: ZodSchema;
}

// Usage example:
export const POST = withValidation(
  {
    body: uploadSchema,
    query: paginationSchema,
  },
  async (request, context, validated) => {
    // validated.body is typed based on uploadSchema
    // validated.query is typed based on paginationSchema
  }
);
```

**Error Response Format (E100)**:
```json
{
  "error": {
    "code": "E100",
    "message": "Invalid request body",
    "userMessage": "Please check your input",
    "retryable": false,
    "details": {
      "errors": {
        "fieldErrors": { "url": ["Invalid URL format"] },
        "formErrors": []
      }
    }
  }
}
```

#### Deliverables

- `src/lib/middleware/validate.ts` - Validation middleware
- `src/lib/middleware/types.ts` - Middleware TypeScript types

---

### BOB-1.2-003: Implement Rate Limiting Middleware

| Field | Value |
|-------|-------|
| **Task ID** | BOB-1.2-003 |
| **Title** | Create sliding window rate limiting middleware |
| **Priority** | P1 (Critical) |
| **Effort** | 45 minutes |
| **Dependencies** | Sri's Redis client (Task 5, 6) |

#### Description

Implement rate limiting middleware using the sliding window algorithm as specified in ADR-007. The middleware must integrate with Redis via the circuit breaker wrapper and return standard rate limit headers per RFC 9238.

#### Acceptance Criteria

- [ ] Implements sliding window algorithm (not fixed window)
- [ ] Default limit: 10 requests per minute per session
- [ ] Uses Redis for distributed rate limiting
- [ ] Graceful degradation via circuit breaker when Redis unavailable
- [ ] Returns `X-RateLimit-Limit` header (max requests)
- [ ] Returns `X-RateLimit-Remaining` header (requests left)
- [ ] Returns `X-RateLimit-Reset` header (window reset timestamp)
- [ ] Returns 429 Too Many Requests with E601 error code
- [ ] `Retry-After` header included in 429 responses

#### Technical Notes

```typescript
// File: src/lib/middleware/rate-limit.ts

interface RateLimitConfig {
  windowMs: number;      // Window size in milliseconds (60000 = 1 min)
  maxRequests: number;   // Max requests per window (10)
  keyGenerator?: (req: NextRequest) => string;  // Session-based by default
}

// Sliding window implementation using Redis sorted sets
// Key pattern: hx-docling:ratelimit:{sessionId}
// Score: timestamp, Value: request ID

const DEFAULT_CONFIG: RateLimitConfig = {
  windowMs: 60000,
  maxRequests: 10,
  keyGenerator: (req) => getSessionId(req),
};
```

**Rate Limit Response Headers**:
```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 10
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1702311060
Retry-After: 45
Content-Type: application/json

{
  "error": {
    "code": "E601",
    "message": "Rate limit exceeded",
    "userMessage": "Too many requests. Please wait before trying again.",
    "retryable": true,
    "retryAfter": 45
  }
}
```

#### Deliverables

- `src/lib/middleware/rate-limit.ts` - Rate limiting middleware
- `src/lib/middleware/rate-limit.test.ts` - Unit tests (basic)

---

### BOB-1.2-004: Expand Health Endpoint with Redis Metrics

| Field | Value |
|-------|-------|
| **Task ID** | BOB-1.2-004 |
| **Title** | Add comprehensive Redis metrics to health endpoint |
| **Priority** | P2 (High) |
| **Effort** | 15 minutes |
| **Dependencies** | BOB-1.2-001, Sri's Redis circuit breaker (Task 6) |

#### Description

Extend the health check endpoint to include detailed Redis metrics including memory usage percentage, evicted keys count, and connection latency status.

#### Acceptance Criteria

- [ ] Health response includes Redis memory usage percentage
- [ ] Health response includes Redis evicted keys count
- [ ] Health response includes Redis connected clients count
- [ ] Latency thresholds: <10ms (good), 10-50ms (acceptable), >50ms (slow)
- [ ] Circuit breaker state reflected in health check

#### Technical Notes

```typescript
// Extended Redis health check response
interface RedisHealthCheck extends ServiceCheck {
  metrics?: {
    memoryUsagePercent: number;
    evictedKeys: number;
    connectedClients: number;
    latencyStatus: 'good' | 'acceptable' | 'slow';
    circuitBreakerState: 'CLOSED' | 'OPEN' | 'HALF_OPEN';
  };
}
```

#### Deliverables

- Updated `src/lib/health/checks.ts` - Extended Redis check

---

## Sprint Summary

| Task ID | Title | Effort | Priority |
|---------|-------|--------|----------|
| BOB-1.2-001 | Health Check Endpoint | 45m | P1 |
| BOB-1.2-002 | Validation Middleware | 30m | P1 |
| BOB-1.2-003 | Rate Limiting Middleware | 45m | P1 |
| BOB-1.2-004 | Redis Health Metrics | 15m | P2 |

**Total Effort**: 2 hours 15 minutes

---

## Dependencies Graph

```
Sprint 1.1 Complete
        |
        v
+-------+--------+
|                |
v                v
William's DB   Sri's Redis
(Task 2)       (Tasks 5,6)
|                |
+-------+--------+
        |
        v
BOB-1.2-001 (Health Endpoint)
        |
        v
BOB-1.2-002 (Validation Middleware)
        |
        v
BOB-1.2-003 (Rate Limit Middleware)
        |
        v
BOB-1.2-004 (Redis Metrics)
```

---

## Notes

- Health endpoint is critical for production monitoring and load balancer integration
- Validation middleware will be used by all API routes in subsequent sprints
- Rate limiting protects the MCP server from overload
- All middleware should be designed for testability with dependency injection
