# Tasks: Redis Integration - Sprint 1.2

**Sprint**: 1.2 - Database & Redis Integration
**Agent**: Sri Venkateswaran (`@sri`)
**Role**: Redis SME (Support)
**Lead**: William (`@william`)
**Input**: Implementation Plan v1.2.0, Detailed Specification v1.2.0
**Prerequisites**: Sprint 0 prerequisites validated (Redis accessible, TLS certificates available)

---

## Overview

This task file covers all Redis-related implementation work for Sprint 1.2. As the Redis SME, Sri provides support for Redis client configuration, TLS setup, session management, rate limiting with sliding window algorithm, circuit breaker implementation, and Redis health checks.

---

## Task List

### SRI-1.2-001: Configure Redis Connection with TLS

**Description**: Create the Redis client singleton with TLS configuration for secure connections to hx-redis-server. This includes CA certificate validation and optional client certificate authentication per CRI-06.

**File**: `src/lib/redis/client.ts`

**Acceptance Criteria**:
- [ ] Redis client connects to `hx-redis-server.hx.dev.local:6379`
- [ ] TLS enabled when `REDIS_TLS_ENABLED=true`
- [ ] CA certificate loaded from `REDIS_CA_CERT_PATH`
- [ ] Optional client certificate authentication supported
- [ ] Connection retry strategy with exponential backoff (max 3s, 10 attempts)
- [ ] Keep-alive configured at 30 seconds
- [ ] Connect timeout set to 10 seconds
- [ ] Command timeout set to 5 seconds
- [ ] Singleton pattern prevents multiple client instances in development

**Dependencies**:
- Sprint 0: Redis accessible (`redis-cli -h hx-redis-server PING` returns PONG)
- Sprint 0: TLS certificates available at `/etc/ssl/certs/hx-redis*`

**Effort**: 45 minutes

**Deliverables**:
- `src/lib/redis/client.ts` - Redis client singleton with TLS configuration
- Environment variables documented in `.env.example`

**Technical Notes**:
```typescript
// Key configuration from Specification Section 7.4.1
const redisConfig = {
  host: process.env.REDIS_HOST || 'hx-redis-server.hx.dev.local',
  port: parseInt(process.env.REDIS_PORT || '6379'),
  password: process.env.REDIS_PASSWORD,
  db: 0,
  maxRetriesPerRequest: 3,
  keepAlive: 30000,
  connectTimeout: 10000,
  commandTimeout: 5000,
  retryStrategy: (times: number) => {
    if (times > 10) return null;
    return Math.min(times * 100, 3000);
  },
};

// TLS configuration
const tlsConfig = process.env.REDIS_TLS_ENABLED === 'true' ? {
  tls: {
    rejectUnauthorized: true,
    ca: fs.readFileSync(process.env.REDIS_CA_CERT_PATH || '/etc/docling/ssl/redis-ca.crt'),
  },
} : {};
```

**Environment Variables**:
```env
REDIS_HOST=hx-redis-server.hx.dev.local
REDIS_PORT=6379
REDIS_PASSWORD=<from-vault>
REDIS_TLS_ENABLED=true
REDIS_CA_CERT_PATH=/etc/docling/ssl/redis-ca.crt
REDIS_URL=rediss://:${REDIS_PASSWORD}@hx-redis-server.hx.dev.local:6379/0
REDIS_CONNECT_TIMEOUT_MS=10000
REDIS_COMMAND_TIMEOUT_MS=5000
```

---

### SRI-1.2-002: Implement Redis Circuit Breaker

**Description**: Create a circuit breaker wrapper for Redis operations to provide graceful degradation when Redis is unavailable. The circuit breaker prevents cascading failures and implements CLOSED/OPEN/HALF_OPEN states per CRI-05.

**File**: `src/lib/redis/circuit-breaker.ts`

**Acceptance Criteria**:
- [ ] Circuit breaker implements three states: CLOSED, OPEN, HALF_OPEN
- [ ] Opens after 5 consecutive failures (configurable)
- [ ] Waits 30 seconds before transitioning to HALF_OPEN (configurable)
- [ ] Requires 3 successful operations to transition from HALF_OPEN to CLOSED
- [ ] Throws `CircuitOpenError` when circuit is open
- [ ] Exposes `getState()` method for monitoring
- [ ] Provides fallback strategies: FAIL_FAST, ALLOW_ALL, IN_MEMORY

**Dependencies**:
- SRI-1.2-001: Redis client configured

**Effort**: 30 minutes

**Deliverables**:
- `src/lib/redis/circuit-breaker.ts` - Circuit breaker implementation
- `src/lib/redis/errors.ts` - CircuitOpenError, ServiceUnavailableError

**Technical Notes**:
```typescript
// Circuit breaker states per Specification Section 7.4.3
interface CircuitBreakerConfig {
  failureThreshold: number;      // Default: 5
  resetTimeout: number;          // Default: 30000ms
  successThreshold: number;      // Default: 3
}

// State transitions:
// CLOSED --(5 failures)--> OPEN --(30s timeout)--> HALF_OPEN
// HALF_OPEN --(3 successes)--> CLOSED
// HALF_OPEN --(1 failure)--> OPEN

export enum FallbackStrategy {
  FAIL_FAST = 'fail-fast',     // Return 503 immediately
  ALLOW_ALL = 'allow-all',     // Skip rate limiting (risky)
  IN_MEMORY = 'in-memory',     // Use in-memory rate limiting
}
```

---

### SRI-1.2-003: Implement Rate Limiting with Sliding Window

**Description**: Create the rate limiting implementation using a sliding window algorithm per Charter 8.1.1. Uses Redis sorted sets with a Lua script for atomic operations to prevent race conditions (CRI-04, CRI-07).

**File**: `src/lib/redis/rate-limit.ts`

**Acceptance Criteria**:
- [ ] Sliding window algorithm (not fixed window) per ADR-007
- [ ] Limits to 10 requests per 60-second window
- [ ] Uses Lua script for atomic ZREMRANGEBYSCORE + ZADD operations
- [ ] Returns `allowed`, `remaining`, and `retryAfter` values
- [ ] Key pattern: `hx-docling:rate:{sessionId}`
- [ ] TTL set on rate limit keys for automatic cleanup
- [ ] Integrates with circuit breaker for fallback behavior

**Dependencies**:
- SRI-1.2-001: Redis client configured
- SRI-1.2-002: Circuit breaker implemented

**Effort**: 45 minutes

**Deliverables**:
- `src/lib/redis/rate-limit.ts` - Sliding window rate limiting
- `src/lib/middleware/rate-limit.ts` - Rate limiting middleware

**Technical Notes**:
```typescript
// Lua script for atomic sliding window per Specification Section 7.4.2
const SLIDING_WINDOW_SCRIPT = `
local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])

-- Remove entries outside the window
redis.call('ZREMRANGEBYSCORE', key, 0, now - window)

-- Count current requests in window
local count = redis.call('ZCARD', key)

if count < limit then
  -- Add new request with current timestamp as score
  redis.call('ZADD', key, now, now .. '-' .. math.random())
  -- Set expiry on the key (cleanup)
  redis.call('EXPIRE', key, window)
  return {1, limit - count - 1, 0}  -- allowed, remaining, retryAfter
else
  -- Get oldest entry to calculate retry time
  local oldest = redis.call('ZRANGE', key, 0, 0, 'WITHSCORES')
  local retryAfter = oldest[2] and (window - (now - oldest[2])) or window
  return {0, 0, math.ceil(retryAfter)}  -- denied, remaining, retryAfter
end
`;

// Key pattern
const RATE_LIMIT_KEY = (sessionId: string) => `hx-docling:rate:${sessionId}`;
```

---

### SRI-1.2-004: Implement Session Management

**Description**: Create session CRUD operations using Redis hashes for efficient storage. Sessions support anonymous users with 24-hour TTL and sliding expiration. Uses circuit breaker for all Redis operations.

**File**: `src/lib/redis/session.ts`

**Acceptance Criteria**:
- [ ] Sessions stored as Redis hashes (not JSON strings) for efficiency
- [ ] Key pattern: `hx-docling:session:{sessionId}`
- [ ] 24-hour TTL with sliding expiration (TTL resets on activity)
- [ ] Session includes: id, createdAt, lastActivity, jobCount, activeJobIds[]
- [ ] `createSession()` creates new session with initial values
- [ ] `getSession()` retrieves session data
- [ ] `updateSessionActivity()` updates lastActivity and resets TTL
- [ ] `deleteSession()` removes session
- [ ] All operations wrapped with circuit breaker

**Dependencies**:
- SRI-1.2-001: Redis client configured
- SRI-1.2-002: Circuit breaker implemented

**Effort**: 45 minutes

**Deliverables**:
- `src/lib/redis/session.ts` - Session CRUD operations

**Technical Notes**:
```typescript
// Session structure per Specification Section 5.4
interface Session {
  id: string;
  createdAt: number;       // Unix timestamp (seconds)
  lastActivity: number;    // Unix timestamp (seconds)
  jobCount: number;
  activeJobIds: string[];  // Support multiple tabs (MAJ-12)
}

const SESSION_TTL = 86400; // 24 hours
const SESSION_KEY = (sessionId: string) => `hx-docling:session:${sessionId}`;

// Use HSET for hash storage (ziplist encoding saves memory)
await redis.multi()
  .hset(key, {
    id: session.id,
    createdAt: session.createdAt.toString(),
    lastActivity: session.lastActivity.toString(),
    jobCount: session.jobCount.toString(),
    activeJobIds: JSON.stringify(session.activeJobIds),
  })
  .expire(key, SESSION_TTL)
  .exec();
```

---

### SRI-1.2-005: Implement Redis Health Check

**Description**: Create comprehensive Redis health check for the `/api/v1/health` endpoint. Returns detailed metrics including PING latency, memory usage, connected clients, and evicted keys.

**File**: `src/lib/redis/health.ts`

**Acceptance Criteria**:
- [ ] PING check with latency measurement
- [ ] Memory usage percentage from `INFO memory`
- [ ] Connected clients count from `INFO stats`
- [ ] Evicted keys count from `INFO stats`
- [ ] Status: `healthy` (all ok), `degraded` (high latency/memory), `unhealthy` (connection failed)
- [ ] Latency threshold: 100ms
- [ ] Memory usage threshold: 75%

**Dependencies**:
- SRI-1.2-001: Redis client configured

**Effort**: 15 minutes

**Deliverables**:
- `src/lib/redis/health.ts` - Redis health check

**Technical Notes**:
```typescript
// Health status per Specification Section 7.4.4
interface RedisHealthStatus {
  status: 'healthy' | 'degraded' | 'unhealthy';
  latency: number;
  details: {
    pingOk: boolean;
    pingLatency: number;
    memoryUsagePercent?: number;
    connectedClients?: number;
    evictedKeys?: number;
  };
}

const HEALTH_THRESHOLDS = {
  pingLatencyMs: 100,
  memoryUsagePercent: 75,
};
```

---

### SRI-1.2-006: Write Unit Tests for Circuit Breaker

**Description**: Create unit tests verifying circuit breaker state transitions through CLOSED -> OPEN -> HALF_OPEN -> CLOSED cycle.

**File**: `src/lib/redis/__tests__/circuit-breaker.test.ts`

**Acceptance Criteria**:
- [ ] Test CLOSED state allows operations
- [ ] Test transitions to OPEN after failureThreshold failures
- [ ] Test OPEN state throws CircuitOpenError
- [ ] Test transitions to HALF_OPEN after resetTimeout
- [ ] Test transitions back to CLOSED after successThreshold successes
- [ ] Test transitions back to OPEN on failure during HALF_OPEN
- [ ] Test `getState()` returns correct state

**Dependencies**:
- SRI-1.2-002: Circuit breaker implemented

**Effort**: 20 minutes

**Deliverables**:
- `src/lib/redis/__tests__/circuit-breaker.test.ts` - Unit tests

**Technical Notes**:
```typescript
// Test scenarios:
describe('RedisCircuitBreaker', () => {
  it('starts in CLOSED state');
  it('transitions to OPEN after 5 consecutive failures');
  it('throws CircuitOpenError when OPEN');
  it('transitions to HALF_OPEN after 30s timeout');
  it('transitions to CLOSED after 3 successes in HALF_OPEN');
  it('transitions back to OPEN on failure in HALF_OPEN');
});
```

---

## Summary

| Task ID | Title | Effort | Dependencies |
|---------|-------|--------|--------------|
| SRI-1.2-001 | Configure Redis Connection with TLS | 45m | Sprint 0 |
| SRI-1.2-002 | Implement Redis Circuit Breaker | 30m | SRI-1.2-001 |
| SRI-1.2-003 | Implement Rate Limiting with Sliding Window | 45m | SRI-1.2-001, SRI-1.2-002 |
| SRI-1.2-004 | Implement Session Management | 45m | SRI-1.2-001, SRI-1.2-002 |
| SRI-1.2-005 | Implement Redis Health Check | 15m | SRI-1.2-001 |
| SRI-1.2-006 | Write Unit Tests for Circuit Breaker | 20m | SRI-1.2-002 |

**Total Effort**: 3 hours 20 minutes

---

## Redis Key Patterns Summary

| Key Pattern | Purpose | TTL |
|-------------|---------|-----|
| `hx-docling:session:{sessionId}` | Session storage | 24h (sliding) |
| `hx-docling:rate:{sessionId}` | Rate limiting | 60s (auto-cleanup) |
| `hx-docling:events:{jobId}` | SSE event buffer | 5 min |
| `hx-docling:idempotency:{key}` | Idempotency records | 24h |

---

## Risk Mitigation

| Risk | Mitigation | Owner |
|------|------------|-------|
| Redis unavailable | Circuit breaker with IN_MEMORY fallback | Sri |
| TLS certificate issues | Validate certificates in Sprint 0 | William |
| Rate limiting race conditions | Atomic Lua script operations | Sri |
| Session loss on restart | Sessions are ephemeral (anonymous users) - acceptable | Sri |

---

## Validation Commands

```bash
# Verify Redis connection
redis-cli -h hx-redis-server.hx.dev.local PING

# Verify TLS connection
redis-cli -h hx-redis-server.hx.dev.local --tls --cacert /etc/ssl/certs/hx-redis-ca.crt PING

# Check session stored
redis-cli -h hx-redis-server HGETALL hx-docling:session:test-session-id

# Check rate limit key
redis-cli -h hx-redis-server ZRANGE hx-docling:rate:test-session-id 0 -1 WITHSCORES

# Check Redis INFO
redis-cli -h hx-redis-server INFO memory
redis-cli -h hx-redis-server INFO stats
```
