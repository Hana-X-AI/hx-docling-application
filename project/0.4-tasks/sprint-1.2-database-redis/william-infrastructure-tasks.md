# Infrastructure Tasks: Sprint 1.2 - Database & Redis Integration

**Sprint**: 1.2 - Database & Redis Integration
**Lead**: William Chen (`@william`)
**Support**: Trinity (`@trinity`), Neo (`@neo`), Sri (`@sri`)
**Review**: Alex (`@alex`)
**Document Version**: 1.0.0
**Created**: 2025-12-12
**Reference**: `project/0.1-plan/0.1.1-implementation-plan.md` Section 4.2

---

## Overview

Sprint 1.2 establishes database and Redis connectivity with production-grade reliability. This includes connection pooling, SSL/TLS, circuit breakers, health monitoring, and rate limiting infrastructure.

**Total Sprint Effort**: 4.5 hours

---

## Task WIL-1.2-001: DATABASE_URL SSL Configuration

### Description

Configure the DATABASE_URL environment variable with SSL and connection pooling parameters. Verify SSL certificate paths and connection parameters.

### Acceptance Criteria

- [ ] DATABASE_URL includes `sslmode=require` parameter
- [ ] SSL root certificate path configured correctly
- [ ] Connection pooling URL points to PgBouncer (port 6432)
- [ ] Direct connection URL points to PostgreSQL (port 5432) for migrations
- [ ] Both URLs tested and verified

### Dependencies

- Sprint 0 prerequisites validated
- Sprint 1.1 project scaffold complete
- SSL certificates available at documented paths

### Effort Estimate

**Duration**: 15 minutes
**Complexity**: Low

### Deliverables

1. Updated `.env` configuration
2. Connection verification script

### Technical Notes

```typescript
// Two connection URLs required:
// 1. Direct connection for Prisma migrations
DATABASE_URL="postgresql://docling_app:${PASSWORD}@hx-postgres-server:5432/docling_db?sslmode=require&sslrootcert=/etc/ssl/certs/hx-postgres-ca.crt"

// 2. Pooled connection for runtime
DATABASE_POOL_URL="postgresql://docling_app:${PASSWORD}@hx-postgres-server:6432/docling_db?sslmode=require"

// Verification
psql "$DATABASE_URL" -c "SELECT ssl_is_used();"
// Expected: t (true)
```

---

## Task WIL-1.2-002: Prisma Client Singleton with Connection Pooling

### Description

Create a Prisma client singleton with proper connection pooling configuration. Ensure the client handles connection limits and cleanup correctly for serverless/long-running environments.

### Acceptance Criteria

- [ ] Single Prisma client instance across application
- [ ] Connection pool configured (max 10 connections)
- [ ] Connection timeout set (20 seconds)
- [ ] Query timeout configured (60 seconds)
- [ ] Global declaration for development hot reload
- [ ] Proper cleanup on process exit

### Dependencies

- Task WIL-1.2-001 completed
- Prisma schema from Sprint 1.1

### Effort Estimate

**Duration**: 30 minutes
**Complexity**: Medium

### Deliverables

1. `src/lib/db/prisma.ts` - Prisma client singleton
2. Connection pool monitoring integration

### Technical Notes

```typescript
// src/lib/db/prisma.ts
import { PrismaClient } from '@prisma/client';

declare global {
  var prisma: PrismaClient | undefined;
}

const prismaClientSingleton = () => {
  return new PrismaClient({
    log: process.env.NODE_ENV === 'development'
      ? ['query', 'error', 'warn']
      : ['error'],
    datasources: {
      db: {
        url: process.env.DATABASE_POOL_URL || process.env.DATABASE_URL,
      },
    },
  });
};

export const prisma = globalThis.prisma ?? prismaClientSingleton();

if (process.env.NODE_ENV !== 'production') {
  globalThis.prisma = prisma;
}

// Graceful shutdown
process.on('beforeExit', async () => {
  await prisma.$disconnect();
});
```

---

## Task WIL-1.2-003: Initial Prisma Migration with Rollback Test

### Description

Run the initial Prisma migration to create database tables. Test rollback procedure to ensure migration safety.

### Acceptance Criteria

- [ ] Migration created with descriptive name
- [ ] Migration applies successfully
- [ ] Tables created: Job, Result, enums
- [ ] Indexes created on job_id, created_at columns
- [ ] Rollback tested and documented
- [ ] Migration history preserved

### Dependencies

- Task WIL-1.2-002 completed
- Prisma schema finalized

### Effort Estimate

**Duration**: 15 minutes
**Complexity**: Low

### Deliverables

1. Migration files in `prisma/migrations/`
2. Rollback test documentation

### Technical Notes

```bash
# Create and apply migration
npx prisma migrate dev --name init_schema

# Verify tables created
npx prisma db push --accept-data-loss  # NOT for production

# Test rollback (development only)
npx prisma migrate reset

# Production migration
npx prisma migrate deploy
```

---

## Task WIL-1.2-004: Migration Rollback Procedure Documentation

### Description

Document and test the migration rollback procedure for disaster recovery scenarios.

### Acceptance Criteria

- [ ] Rollback procedure documented step-by-step
- [ ] Rollback tested on development database
- [ ] Data preservation considerations documented
- [ ] Recovery time objective (RTO) measured
- [ ] Procedure added to operational runbook

### Dependencies

- Task WIL-1.2-003 completed

### Effort Estimate

**Duration**: 15 minutes
**Complexity**: Low

### Deliverables

1. Migration rollback runbook section
2. Rollback verification log

### Technical Notes

```bash
# Rollback steps (development)
1. Stop application
2. npx prisma migrate reset
3. Restore from backup if needed
4. Re-apply migrations to desired version
5. Restart application

# Production rollback requires:
- Database backup before migration
- Down migration scripts (manual creation)
- Application downtime coordination
```

---

## Task WIL-1.2-005: Redis TLS Connection Configuration

### Description

Configure Redis client connection with TLS encryption. This is a critical security requirement for production connectivity.

### Acceptance Criteria

- [ ] Redis URL uses `rediss://` protocol (TLS)
- [ ] CA certificate path configured
- [ ] Client certificate path configured (if mutual TLS)
- [ ] Connection timeout set (5 seconds)
- [ ] Command timeout set (10 seconds)
- [ ] Connection verified with PING

### Dependencies

- Sprint 0 Redis TLS prerequisites validated
- Node.js ioredis package installed

### Effort Estimate

**Duration**: 45 minutes
**Complexity**: Medium

### Deliverables

1. `src/lib/redis/client.ts` - Redis client configuration
2. TLS verification log

### Technical Notes

```typescript
// src/lib/redis/client.ts
import Redis from 'ioredis';
import fs from 'fs';

const createRedisClient = () => {
  const tlsOptions = process.env.REDIS_TLS_ENABLED === 'true'
    ? {
        tls: {
          ca: fs.readFileSync(process.env.REDIS_TLS_CA_PATH!),
          // Add if mutual TLS required
          // cert: fs.readFileSync(process.env.REDIS_TLS_CERT_PATH!),
          // key: fs.readFileSync(process.env.REDIS_TLS_KEY_PATH!),
          rejectUnauthorized: true,
        },
      }
    : {};

  return new Redis({
    host: process.env.REDIS_HOST || 'hx-redis-server',
    port: parseInt(process.env.REDIS_PORT || '6379'),
    password: process.env.REDIS_PASSWORD,
    connectTimeout: 5000,
    commandTimeout: 10000,
    retryStrategy: (times) => {
      if (times > 3) return null; // Stop retrying
      return Math.min(times * 200, 2000);
    },
    ...tlsOptions,
  });
};

export const redis = createRedisClient();
```

---

## Task WIL-1.2-006: Redis Circuit Breaker Implementation

### Description

Implement a circuit breaker wrapper for Redis operations to provide graceful degradation when Redis is unavailable.

### Acceptance Criteria

- [ ] Circuit breaker states: CLOSED, OPEN, HALF_OPEN
- [ ] Failure threshold: 5 failures in 60 seconds
- [ ] Recovery timeout: 30 seconds
- [ ] Half-open test: single request allowed
- [ ] State transitions logged
- [ ] Fallback behavior defined for each operation

### Dependencies

- Task WIL-1.2-005 completed

### Effort Estimate

**Duration**: 30 minutes
**Complexity**: Medium

### Deliverables

1. `src/lib/redis/circuit-breaker.ts` - Circuit breaker implementation
2. Unit tests for state transitions

### Technical Notes

```typescript
// src/lib/redis/circuit-breaker.ts
type CircuitState = 'CLOSED' | 'OPEN' | 'HALF_OPEN';

interface CircuitBreakerConfig {
  failureThreshold: number;
  resetTimeout: number;
  monitoringPeriod: number;
}

class RedisCircuitBreaker {
  private state: CircuitState = 'CLOSED';
  private failures: number = 0;
  private lastFailureTime: number = 0;
  private config: CircuitBreakerConfig;

  constructor(config: Partial<CircuitBreakerConfig> = {}) {
    this.config = {
      failureThreshold: config.failureThreshold ?? 5,
      resetTimeout: config.resetTimeout ?? 30000,
      monitoringPeriod: config.monitoringPeriod ?? 60000,
    };
  }

  async execute<T>(operation: () => Promise<T>, fallback?: () => T): Promise<T> {
    if (this.state === 'OPEN') {
      if (Date.now() - this.lastFailureTime > this.config.resetTimeout) {
        this.state = 'HALF_OPEN';
      } else {
        if (fallback) return fallback();
        throw new Error('Circuit breaker is OPEN');
      }
    }

    try {
      const result = await operation();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      if (fallback) return fallback();
      throw error;
    }
  }

  private onSuccess(): void {
    if (this.state === 'HALF_OPEN') {
      this.state = 'CLOSED';
    }
    this.failures = 0;
  }

  private onFailure(): void {
    this.failures++;
    this.lastFailureTime = Date.now();
    if (this.failures >= this.config.failureThreshold) {
      this.state = 'OPEN';
      console.warn('[CircuitBreaker] State changed to OPEN');
    }
  }

  getState(): CircuitState {
    return this.state;
  }
}

export const redisCircuitBreaker = new RedisCircuitBreaker();
```

---

## Task WIL-1.2-007: Session Management with Circuit Breaker

### Description

Implement Redis-based session management using the circuit breaker for reliability. Sessions enable anonymous user tracking with 24-hour TTL.

### Acceptance Criteria

- [ ] Session creation generates UUID
- [ ] Session stored in Redis with 24h TTL
- [ ] Session retrieval uses circuit breaker
- [ ] Fallback to in-memory session on Redis failure
- [ ] Session validation middleware created
- [ ] Session ID returned in response header

### Dependencies

- Task WIL-1.2-005 completed
- Task WIL-1.2-006 completed

### Effort Estimate

**Duration**: 45 minutes
**Complexity**: Medium

### Deliverables

1. `src/lib/redis/session.ts` - Session management
2. Session middleware integration

### Technical Notes

```typescript
// src/lib/redis/session.ts
import { v4 as uuidv4 } from 'uuid';
import { redis } from './client';
import { redisCircuitBreaker } from './circuit-breaker';

const SESSION_TTL = 60 * 60 * 24; // 24 hours
const SESSION_PREFIX = 'session:';

interface Session {
  id: string;
  createdAt: number;
  lastAccessedAt: number;
}

export async function createSession(): Promise<Session> {
  const session: Session = {
    id: uuidv4(),
    createdAt: Date.now(),
    lastAccessedAt: Date.now(),
  };

  await redisCircuitBreaker.execute(
    () => redis.setex(
      `${SESSION_PREFIX}${session.id}`,
      SESSION_TTL,
      JSON.stringify(session)
    ),
    () => { /* Fallback: session created but not persisted */ }
  );

  return session;
}

export async function getSession(sessionId: string): Promise<Session | null> {
  return redisCircuitBreaker.execute(
    async () => {
      const data = await redis.get(`${SESSION_PREFIX}${sessionId}`);
      if (!data) return null;

      const session: Session = JSON.parse(data);
      // Refresh TTL on access
      await redis.expire(`${SESSION_PREFIX}${sessionId}`, SESSION_TTL);
      return session;
    },
    () => null // Fallback: session not found
  );
}
```

---

## Task WIL-1.2-008: Health Check Endpoint at /api/v1/health

### Description

Create a comprehensive health check endpoint that reports status of all infrastructure dependencies (PostgreSQL, Redis, MCP server).

### Acceptance Criteria

- [ ] Endpoint at `/api/v1/health` (versioned path)
- [ ] Returns JSON with status of each dependency
- [ ] Overall status: "healthy", "degraded", or "unhealthy"
- [ ] Response includes version information
- [ ] Response time < 500ms under normal conditions
- [ ] Caching header set (no-cache)

### Dependencies

- Task WIL-1.2-002 completed (Prisma client)
- Task WIL-1.2-005 completed (Redis client)

### Effort Estimate

**Duration**: 45 minutes
**Complexity**: Medium

### Deliverables

1. `src/app/api/v1/health/route.ts` - Health endpoint
2. Health check utility functions

### Technical Notes

```typescript
// src/app/api/v1/health/route.ts
import { NextResponse } from 'next/server';
import { prisma } from '@/lib/db/prisma';
import { redis } from '@/lib/redis/client';

interface HealthStatus {
  status: 'healthy' | 'degraded' | 'unhealthy';
  version: string;
  timestamp: string;
  services: {
    database: ServiceStatus;
    redis: ServiceStatus;
    mcp: ServiceStatus;
  };
}

interface ServiceStatus {
  status: 'up' | 'down';
  latencyMs: number;
  message?: string;
}

async function checkDatabase(): Promise<ServiceStatus> {
  const start = Date.now();
  try {
    await prisma.$queryRaw`SELECT 1`;
    return { status: 'up', latencyMs: Date.now() - start };
  } catch (error) {
    return {
      status: 'down',
      latencyMs: Date.now() - start,
      message: error instanceof Error ? error.message : 'Unknown error',
    };
  }
}

async function checkRedis(): Promise<ServiceStatus> {
  const start = Date.now();
  try {
    await redis.ping();
    return { status: 'up', latencyMs: Date.now() - start };
  } catch (error) {
    return {
      status: 'down',
      latencyMs: Date.now() - start,
      message: error instanceof Error ? error.message : 'Unknown error',
    };
  }
}

async function checkMcp(): Promise<ServiceStatus> {
  const start = Date.now();
  try {
    const response = await fetch(`${process.env.MCP_SERVER_URL}/health`, {
      signal: AbortSignal.timeout(5000),
    });
    return {
      status: response.ok ? 'up' : 'down',
      latencyMs: Date.now() - start,
    };
  } catch (error) {
    return {
      status: 'down',
      latencyMs: Date.now() - start,
      message: error instanceof Error ? error.message : 'Unknown error',
    };
  }
}

export async function GET() {
  const [database, redis, mcp] = await Promise.all([
    checkDatabase(),
    checkRedis(),
    checkMcp(),
  ]);

  const services = { database, redis, mcp };
  const allUp = Object.values(services).every(s => s.status === 'up');
  const allDown = Object.values(services).every(s => s.status === 'down');

  const health: HealthStatus = {
    status: allDown ? 'unhealthy' : allUp ? 'healthy' : 'degraded',
    version: process.env.APP_VERSION || '1.0.0',
    timestamp: new Date().toISOString(),
    services,
  };

  return NextResponse.json(health, {
    status: health.status === 'unhealthy' ? 503 : 200,
    headers: {
      'Cache-Control': 'no-cache, no-store, must-revalidate',
    },
  });
}
```

---

## Task WIL-1.2-009: Health Check Redis Metrics Expansion

### Description

Expand the health check to include detailed Redis metrics: memory usage percentage, evicted keys count, and connection latency status.

### Acceptance Criteria

- [ ] Redis memory usage percentage included
- [ ] Evicted keys count included
- [ ] Connection latency status (good/slow/timeout)
- [ ] Max memory threshold warning (>80%)
- [ ] Metrics refreshed on each health check

### Dependencies

- Task WIL-1.2-008 completed

### Effort Estimate

**Duration**: 15 minutes
**Complexity**: Low

### Deliverables

1. Enhanced Redis health check with metrics
2. Memory threshold alerting configuration

### Technical Notes

```typescript
// Enhanced Redis health check
async function checkRedisWithMetrics(): Promise<ServiceStatus & { metrics?: RedisMetrics }> {
  const start = Date.now();
  try {
    const [pong, info] = await Promise.all([
      redis.ping(),
      redis.info('memory'),
    ]);

    const latencyMs = Date.now() - start;
    const usedMemory = parseRedisMemory(info, 'used_memory');
    const maxMemory = parseRedisMemory(info, 'maxmemory') || Infinity;
    const evictedKeys = parseRedisValue(info, 'evicted_keys');

    const memoryPercent = maxMemory > 0 ? (usedMemory / maxMemory) * 100 : 0;

    return {
      status: 'up',
      latencyMs,
      metrics: {
        memoryPercent: Math.round(memoryPercent * 100) / 100,
        evictedKeys,
        latencyStatus: latencyMs < 10 ? 'good' : latencyMs < 100 ? 'slow' : 'timeout',
        memoryWarning: memoryPercent > 80,
      },
    };
  } catch (error) {
    return {
      status: 'down',
      latencyMs: Date.now() - start,
      message: error instanceof Error ? error.message : 'Unknown error',
    };
  }
}
```

---

## Task WIL-1.2-010: Rate Limiting Middleware (Sliding Window)

### Description

Implement rate limiting middleware using Redis with sliding window algorithm. Apply to upload and processing endpoints.

### Acceptance Criteria

- [ ] Sliding window algorithm implemented (NOT fixed window)
- [ ] Default limit: 10 requests per minute
- [ ] Configurable per-route limits
- [ ] Rate limit headers in response (X-RateLimit-*)
- [ ] 429 Too Many Requests response when exceeded
- [ ] Session-based rate limiting (not IP-based)

### Dependencies

- Task WIL-1.2-005 completed (Redis client)
- Task WIL-1.2-007 completed (Session)

### Effort Estimate

**Duration**: 45 minutes
**Complexity**: Medium

### Deliverables

1. `src/lib/middleware/rate-limit.ts` - Rate limiting implementation
2. Configuration for different endpoints

### Technical Notes

```typescript
// src/lib/middleware/rate-limit.ts
import { redis } from '@/lib/redis/client';
import { redisCircuitBreaker } from '@/lib/redis/circuit-breaker';

interface RateLimitConfig {
  windowMs: number;  // Window size in milliseconds
  maxRequests: number;  // Max requests per window
}

const DEFAULT_CONFIG: RateLimitConfig = {
  windowMs: 60000,  // 1 minute
  maxRequests: 10,
};

export async function checkRateLimit(
  sessionId: string,
  endpoint: string,
  config: Partial<RateLimitConfig> = {}
): Promise<{ allowed: boolean; remaining: number; resetAt: number }> {
  const { windowMs, maxRequests } = { ...DEFAULT_CONFIG, ...config };
  const now = Date.now();
  const windowStart = now - windowMs;
  const key = `ratelimit:${sessionId}:${endpoint}`;

  return redisCircuitBreaker.execute(
    async () => {
      // Sliding window: count requests in last windowMs
      await redis.zremrangebyscore(key, '-inf', windowStart.toString());
      const requestCount = await redis.zcard(key);

      if (requestCount >= maxRequests) {
        const oldestRequest = await redis.zrange(key, 0, 0, 'WITHSCORES');
        const resetAt = oldestRequest.length > 1
          ? parseInt(oldestRequest[1]) + windowMs
          : now + windowMs;
        return { allowed: false, remaining: 0, resetAt };
      }

      // Add current request to sliding window
      await redis.zadd(key, now.toString(), `${now}-${Math.random()}`);
      await redis.expire(key, Math.ceil(windowMs / 1000));

      return {
        allowed: true,
        remaining: maxRequests - requestCount - 1,
        resetAt: now + windowMs,
      };
    },
    () => ({ allowed: true, remaining: maxRequests, resetAt: now + windowMs })
  );
}

// Rate limit response headers
export function getRateLimitHeaders(result: { remaining: number; resetAt: number }) {
  return {
    'X-RateLimit-Remaining': result.remaining.toString(),
    'X-RateLimit-Reset': Math.ceil(result.resetAt / 1000).toString(),
  };
}
```

---

## Task WIL-1.2-011: Centralized Validation Middleware

### Description

Implement centralized validation middleware using Zod schemas for request validation across all API routes.

### Acceptance Criteria

- [ ] Zod schema validation for request body
- [ ] Zod schema validation for query parameters
- [ ] Structured error response on validation failure
- [ ] Error code E400 series for validation errors
- [ ] Reusable across all API routes
- [ ] Type-safe validated request data

### Dependencies

- Zod package installed from Sprint 1.1

### Effort Estimate

**Duration**: 30 minutes
**Complexity**: Medium

### Deliverables

1. `src/lib/middleware/validate.ts` - Validation middleware
2. Common validation schemas

### Technical Notes

```typescript
// src/lib/middleware/validate.ts
import { z, ZodError, ZodSchema } from 'zod';
import { NextRequest, NextResponse } from 'next/server';

interface ValidationError {
  code: string;
  message: string;
  details: Array<{
    field: string;
    message: string;
  }>;
}

export function validateBody<T extends ZodSchema>(schema: T) {
  return async (req: NextRequest): Promise<z.infer<T> | NextResponse> => {
    try {
      const body = await req.json();
      return schema.parse(body);
    } catch (error) {
      if (error instanceof ZodError) {
        const validationError: ValidationError = {
          code: 'E400',
          message: 'Validation failed',
          details: error.errors.map(e => ({
            field: e.path.join('.'),
            message: e.message,
          })),
        };
        return NextResponse.json(validationError, { status: 400 });
      }
      return NextResponse.json(
        { code: 'E400', message: 'Invalid request body' },
        { status: 400 }
      );
    }
  };
}

export function validateQuery<T extends ZodSchema>(schema: T) {
  return (req: NextRequest): z.infer<T> | NextResponse => {
    try {
      const params = Object.fromEntries(req.nextUrl.searchParams);
      return schema.parse(params);
    } catch (error) {
      if (error instanceof ZodError) {
        const validationError: ValidationError = {
          code: 'E401',
          message: 'Invalid query parameters',
          details: error.errors.map(e => ({
            field: e.path.join('.'),
            message: e.message,
          })),
        };
        return NextResponse.json(validationError, { status: 400 });
      }
      return NextResponse.json(
        { code: 'E401', message: 'Invalid query parameters' },
        { status: 400 }
      );
    }
  };
}
```

---

## Task WIL-1.2-012: PgBouncer Connection Pool Validation

### Description

Validate and test PgBouncer connection pooling behavior under simulated load. Document pool configuration and monitoring.

### Acceptance Criteria

- [ ] PgBouncer connection verified via port 6432
- [ ] Pool behavior validated (transaction mode)
- [ ] Connection limit tested (simulate concurrent requests)
- [ ] Pool statistics accessible for monitoring
- [ ] Configuration documented

### Dependencies

- Task WIL-1.2-002 completed
- PgBouncer accessible

### Effort Estimate

**Duration**: 15 minutes
**Complexity**: Low

### Deliverables

1. PgBouncer validation test script
2. Pool configuration documentation

### Technical Notes

```bash
# Pool validation script
#!/bin/bash

# Test connection via PgBouncer
psql -h hx-postgres-server -p 6432 -U docling_app -d docling_db -c 'SELECT 1;'

# Check pool status
psql -h hx-postgres-server -p 6432 -U docling_app -d pgbouncer -c 'SHOW POOLS;'

# Check active connections
psql -h hx-postgres-server -p 6432 -U docling_app -d pgbouncer -c 'SHOW CLIENTS;'

# Simulate concurrent load (using pgbench or similar)
# Verify connections are pooled, not created for each request
```

---

## Task WIL-1.2-013: Database Operations Verification

### Description

Test all CRUD operations against the database using Prisma client. Verify data integrity and transaction behavior.

### Acceptance Criteria

- [ ] CREATE operations work correctly
- [ ] READ operations return expected data
- [ ] UPDATE operations modify data correctly
- [ ] DELETE operations remove data correctly
- [ ] Transactions commit/rollback correctly
- [ ] Concurrent operations handled properly

### Dependencies

- Task WIL-1.2-002 completed
- Task WIL-1.2-003 completed

### Effort Estimate

**Duration**: 30 minutes
**Complexity**: Medium

### Deliverables

1. CRUD test suite
2. Transaction test results

### Technical Notes

```typescript
// CRUD verification tests
import { prisma } from '@/lib/db/prisma';

async function verifyCRUD() {
  // CREATE
  const job = await prisma.job.create({
    data: {
      sessionId: 'test-session',
      fileName: 'test.pdf',
      fileType: 'application/pdf',
      fileSize: 1024,
      status: 'PENDING',
    },
  });
  console.log('CREATE:', job.id);

  // READ
  const foundJob = await prisma.job.findUnique({ where: { id: job.id } });
  console.log('READ:', foundJob?.status);

  // UPDATE
  const updatedJob = await prisma.job.update({
    where: { id: job.id },
    data: { status: 'PROCESSING' },
  });
  console.log('UPDATE:', updatedJob.status);

  // DELETE
  await prisma.job.delete({ where: { id: job.id } });
  console.log('DELETE: complete');

  // TRANSACTION
  const [job1, job2] = await prisma.$transaction([
    prisma.job.create({ data: { /* ... */ } }),
    prisma.job.create({ data: { /* ... */ } }),
  ]);
  console.log('TRANSACTION:', job1.id, job2.id);
}
```

---

## Task WIL-1.2-014: Circuit Breaker Unit Tests

### Description

Write comprehensive unit tests for the Redis circuit breaker, covering all state transitions and edge cases.

### Acceptance Criteria

- [ ] Test CLOSED -> OPEN transition on failures
- [ ] Test OPEN -> HALF_OPEN transition after timeout
- [ ] Test HALF_OPEN -> CLOSED on success
- [ ] Test HALF_OPEN -> OPEN on failure
- [ ] Test fallback execution when OPEN
- [ ] Test failure count reset in monitoring period

### Dependencies

- Task WIL-1.2-006 completed
- Vitest configured from Sprint 1.1

### Effort Estimate

**Duration**: 20 minutes
**Complexity**: Medium

### Deliverables

1. `src/lib/redis/circuit-breaker.test.ts`
2. Test coverage report

### Technical Notes

```typescript
// src/lib/redis/circuit-breaker.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { RedisCircuitBreaker } from './circuit-breaker';

describe('RedisCircuitBreaker', () => {
  let cb: RedisCircuitBreaker;

  beforeEach(() => {
    cb = new RedisCircuitBreaker({
      failureThreshold: 3,
      resetTimeout: 1000,
    });
  });

  it('should start in CLOSED state', () => {
    expect(cb.getState()).toBe('CLOSED');
  });

  it('should transition to OPEN after failure threshold', async () => {
    const failingOp = () => Promise.reject(new Error('fail'));

    for (let i = 0; i < 3; i++) {
      try { await cb.execute(failingOp); } catch {}
    }

    expect(cb.getState()).toBe('OPEN');
  });

  it('should transition to HALF_OPEN after reset timeout', async () => {
    // Force OPEN state
    // Wait for reset timeout
    // Verify HALF_OPEN on next execute attempt
  });

  it('should use fallback when OPEN', async () => {
    // Force OPEN state
    const result = await cb.execute(
      () => Promise.reject(new Error('fail')),
      () => 'fallback'
    );
    expect(result).toBe('fallback');
  });
});
```

---

## Task WIL-1.2-015: Database Index Verification

### Description

Verify that database indexes are properly created on job_id and created_at columns for query performance.

### Acceptance Criteria

- [ ] Index on job_id column verified
- [ ] Index on created_at column verified
- [ ] Index on sessionId column verified
- [ ] Index usage confirmed via EXPLAIN ANALYZE
- [ ] No missing indexes for common queries

### Dependencies

- Task WIL-1.2-003 completed

### Effort Estimate

**Duration**: 15 minutes
**Complexity**: Low

### Deliverables

1. Index verification script
2. Query plan analysis

### Technical Notes

```sql
-- Index verification
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'Job';

-- Verify index usage
EXPLAIN ANALYZE SELECT * FROM "Job" WHERE "sessionId" = 'test';
EXPLAIN ANALYZE SELECT * FROM "Job" ORDER BY "createdAt" DESC LIMIT 20;
```

---

## Task WIL-1.2-016: Database Backup Before Test Data

### Description

Create database backup before loading test data to enable clean slate recovery.

### Acceptance Criteria

- [ ] Backup script created
- [ ] Backup executed successfully
- [ ] Backup file verified (can be restored)
- [ ] Backup location documented
- [ ] Retention policy defined

### Dependencies

- Task WIL-1.2-003 completed
- Backup storage available

### Effort Estimate

**Duration**: 15 minutes
**Complexity**: Low

### Deliverables

1. Backup script
2. Backup verification log
3. Retention policy documentation

### Technical Notes

```bash
#!/bin/bash
# Database backup script

BACKUP_DIR="/data/backups/docling"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/docling_db_${TIMESTAMP}.sql.gz"

# Create backup directory
mkdir -p ${BACKUP_DIR}

# Create backup
pg_dump -h hx-postgres-server -U docling_app docling_db | gzip > ${BACKUP_FILE}

# Verify backup
gunzip -t ${BACKUP_FILE}
echo "Backup created: ${BACKUP_FILE}"

# Retention: keep 7 days
find ${BACKUP_DIR} -name "*.sql.gz" -mtime +7 -delete
```

---

## Sprint Summary

| Task ID | Title | Effort | Dependencies |
|---------|-------|--------|--------------|
| WIL-1.2-001 | DATABASE_URL SSL Configuration | 15m | Sprint 0, Sprint 1.1 |
| WIL-1.2-002 | Prisma Client Singleton with Connection Pooling | 30m | WIL-1.2-001 |
| WIL-1.2-003 | Initial Prisma Migration with Rollback Test | 15m | WIL-1.2-002 |
| WIL-1.2-004 | Migration Rollback Procedure Documentation | 15m | WIL-1.2-003 |
| WIL-1.2-005 | Redis TLS Connection Configuration | 45m | Sprint 0 |
| WIL-1.2-006 | Redis Circuit Breaker Implementation | 30m | WIL-1.2-005 |
| WIL-1.2-007 | Session Management with Circuit Breaker | 45m | WIL-1.2-005, WIL-1.2-006 |
| WIL-1.2-008 | Health Check Endpoint at /api/v1/health | 45m | WIL-1.2-002, WIL-1.2-005 |
| WIL-1.2-009 | Health Check Redis Metrics Expansion | 15m | WIL-1.2-008 |
| WIL-1.2-010 | Rate Limiting Middleware (Sliding Window) | 45m | WIL-1.2-005 |
| WIL-1.2-011 | Centralized Validation Middleware | 30m | Sprint 1.1 |
| WIL-1.2-012 | PgBouncer Connection Pool Validation | 15m | WIL-1.2-002 |
| WIL-1.2-013 | Database Operations Verification | 30m | WIL-1.2-002, WIL-1.2-003 |
| WIL-1.2-014 | Circuit Breaker Unit Tests | 20m | WIL-1.2-006 |
| WIL-1.2-015 | Database Index Verification | 15m | WIL-1.2-003 |
| WIL-1.2-016 | Database Backup Before Test Data | 15m | WIL-1.2-003 |

**Total Tasks**: 16
**Total Effort**: 4 hours 30 minutes

---

## Risk Considerations

| Risk | Mitigation |
|------|------------|
| Redis TLS certificate issues | Allow extra 30m buffer, coordinate with security team |
| PgBouncer not available | Fallback to direct PostgreSQL connection (document limitation) |
| Circuit breaker complexity | Follow simple state machine pattern, comprehensive tests |
| Rate limiting accuracy | Use sliding window, verify with load testing |
