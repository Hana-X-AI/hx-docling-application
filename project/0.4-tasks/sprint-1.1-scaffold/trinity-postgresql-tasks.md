# Sprint 1.1: Prisma Schema Design

**Sprint**: Sprint 1.1 (Scaffold)
**Agent**: Trinity Smith (PostgreSQL DBA SME)
**Duration**: 1.5 hours
**Dependencies**: Sprint 0 prerequisites validation complete
**References**:
- Implementation Plan Section 4.2 (Sprint 1.1)
- Specification Section 5.1 (Database Schema)
- Specification Section 5.1.2 (Job Status State Machine)

---

## Task Overview

Design and implement the Prisma schema for the hx-docling-application, including Job and Result models, enums, indexes, and relationships. Ensure the schema supports all application requirements: job state management, result storage with size validation, checkpoint data for resumption, and efficient querying.

---

## Tasks

### TRI-1.1-001: Create Prisma Schema File with Base Configuration

**Priority**: P0 (Blocking)
**Effort**: 15 minutes
**Dependencies**: Sprint 0 complete

**Description**:
Initialize the `prisma/schema.prisma` file with generator, datasource, and base configuration. Configure for PostgreSQL with proper connection URLs for application (PgBouncer) and migrations (direct).

**Acceptance Criteria**:
- [ ] File `prisma/schema.prisma` created
- [ ] `generator client` configured with `prisma-client-js`
- [ ] `datasource db` configured with provider `postgresql`
- [ ] `url` references `DATABASE_URL` (PgBouncer, port 6432)
- [ ] `directUrl` references `DIRECT_DATABASE_URL` (direct PostgreSQL, port 5432)
- [ ] Schema is valid: `npx prisma validate` succeeds

**Implementation**:
```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")       // PgBouncer (port 6432) for application
  directUrl = env("DIRECT_DATABASE_URL") // Direct (port 5432) for migrations
}

// Models will be added in subsequent tasks
```

**Deliverables**:
- `prisma/schema.prisma` file created
- Validation output showing schema is valid

**Technical Notes**:
- `directUrl` is required for migrations to bypass PgBouncer (DDL operations need direct connection)
- Schema version should be documented in comments for future migrations
- Coordinate with Neo Anderson to ensure file is created in correct location

---

### TRI-1.1-002: Define JobStatus Enum

**Priority**: P0 (Blocking)
**Effort**: 10 minutes
**Dependencies**: TRI-1.1-001

**Description**:
Create the `JobStatus` enum with all required states per the state machine specification. Ensure enum values match application requirements and support retry, cancellation, and partial completion states.

**Acceptance Criteria**:
- [ ] `JobStatus` enum defined with all 13 states
- [ ] Enum values match Specification Section 5.1.2 exactly
- [ ] Values support state machine transitions (PENDING, UPLOADING, PROCESSING, COMPLETE, FAILED, CANCELLED, PARTIAL_COMPLETE, RETRY_*)
- [ ] Schema validation passes

**Implementation**:
```prisma
// JobStatus enum - supports full state machine per Spec 5.1.2
enum JobStatus {
  PENDING           // Job created, awaiting processing
  UPLOADING         // File upload in progress
  PROCESSING        // MCP processing in progress
  COMPLETE          // Successfully completed with all exports
  FAILED            // Permanent failure, no retry
  CANCELLED         // User-cancelled job
  PARTIAL_COMPLETE  // Some exports succeeded, some failed

  // Retry states (transient)
  RETRY_PENDING     // Scheduled for retry
  RETRY_UPLOAD      // Retrying upload
  RETRY_PROCESSING  // Retrying MCP processing
  RETRY_EXPORT      // Retrying export generation
  RETRY_SAVE        // Retrying result save
  RETRY_FAILED      // Max retries exhausted
}
```

**Deliverables**:
- `JobStatus` enum in `prisma/schema.prisma`
- Comment documentation explaining each state

**Technical Notes**:
- Enum order should match state progression for clarity
- PostgreSQL will create an ENUM type in the database
- State transitions are validated in application code (see Spec 5.1.2)
- RETRY_* states are transient and should transition quickly

---

### TRI-1.1-003: Define Job Model with All Fields

**Priority**: P0 (Blocking)
**Effort**: 30 minutes
**Dependencies**: TRI-1.1-002

**Description**:
Create the `Job` model with all required fields, including identifiers, session tracking, input metadata, processing state, timestamps, and checkpoint data for resumption.

**Acceptance Criteria**:
- [ ] All fields from Specification Section 5.1 included
- [ ] `id` is UUID with `@default(uuid())`
- [ ] `sessionId` is String for Redis session correlation
- [ ] `status` uses `JobStatus` enum with default PENDING
- [ ] `inputType`, `inputPath`, `inputUrl` for file/URL tracking
- [ ] `currentStage` and `progressPercent` for progress tracking
- [ ] `retryCount` and `maxRetries` for retry logic
- [ ] `checkpointData` as `Json?` for resumption
- [ ] `errorCode`, `errorMessage`, `errorDetails` for error tracking
- [ ] Timestamps: `createdAt`, `updatedAt`, `startedAt`, `completedAt`
- [ ] Indexes on commonly queried fields
- [ ] Schema validation passes

**Implementation**:
```prisma
model Job {
  // Primary identifier
  id        String   @id @default(uuid())

  // Session correlation
  sessionId String   @db.VarChar(255) // Redis session ID

  // Job status and processing
  status         JobStatus @default(PENDING)
  currentStage   String?   @db.VarChar(50)    // upload, parsing, conversion, export, saving
  progressPercent Int?     @default(0)        // 0-100

  // Input metadata
  inputType  String  @db.VarChar(10)  // FILE or URL
  inputPath  String? @db.VarChar(500) // File path on disk (if FILE)
  inputUrl   String? @db.VarChar(2048) // URL (if URL)
  fileName   String? @db.VarChar(255) // Original filename
  fileSize   BigInt? // File size in bytes
  fileMime   String? @db.VarChar(100) // MIME type

  // Processing metadata
  mcpToolName String? @db.VarChar(50)  // convert_pdf, convert_docx, etc.
  timeoutMs   Int?                      // Configured timeout for this job

  // Retry logic
  retryCount  Int @default(0)
  maxRetries  Int @default(3)

  // Checkpoint for resumption (JSON)
  checkpointData Json? // See Spec 5.5 for Checkpoint schema

  // Error tracking
  errorCode    String? @db.VarChar(10)  // E001, E201, E301, etc.
  errorMessage String? @db.VarChar(500)
  errorDetails Json?   // Detailed error context

  // Timestamps
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
  startedAt   DateTime? // When processing started
  completedAt DateTime? // When processing completed (success or failure)

  // Relationships
  results Result[]

  // Indexes for efficient queries
  @@index([sessionId])             // Query jobs by session
  @@index([status])                // Filter by status
  @@index([createdAt(sort: Desc)]) // Sort by creation time (history view)
  @@index([sessionId, createdAt(sort: Desc)]) // Session history queries

  @@map("jobs") // Table name in database
}
```

**Deliverables**:
- Complete `Job` model in `prisma/schema.prisma`
- Field comments explaining purpose of each field
- Indexes documented with query rationale

**Technical Notes**:
- `@db.VarChar()` specifies PostgreSQL column types for optimal storage
- `BigInt` for `fileSize` supports files up to 100MB+ without overflow
- `Json` type for `checkpointData` and `errorDetails` for flexibility
- Indexes are critical for performance: sessionId queries (history), status filtering, time-based sorting
- `@@map("jobs")` creates table name in lowercase plural (PostgreSQL convention)

---

### TRI-1.1-004: Define Result Model with Size Validation

**Priority**: P0 (Blocking)
**Effort**: 20 minutes
**Dependencies**: TRI-1.1-003

**Description**:
Create the `Result` model to store export outputs (markdown, HTML, JSON) with proper relationships, timestamps, and size constraints. Ensure the schema supports efficient retrieval by job ID.

**Acceptance Criteria**:
- [ ] All fields from Specification Section 5.1 included
- [ ] `id` is UUID with `@default(uuid())`
- [ ] Foreign key relationship to `Job` model
- [ ] `formatType` field for markdown/html/json
- [ ] `content` as `Text` for large export content
- [ ] `sizeBytes` to track result size
- [ ] `generatedAt` timestamp
- [ ] Index on `jobId` for efficient queries
- [ ] Cascade delete: deleting Job deletes Results
- [ ] Schema validation passes

**Implementation**:
```prisma
model Result {
  // Primary identifier
  id String @id @default(uuid())

  // Foreign key to Job
  jobId String
  job   Job    @relation(fields: [jobId], references: [id], onDelete: Cascade)

  // Export format
  formatType String @db.VarChar(20) // markdown, html, json

  // Result content
  content   String  @db.Text     // Large text content (markdown, HTML, JSON)
  sizeBytes Int                  // Content size in bytes

  // Metadata
  generatedAt DateTime @default(now())

  // Indexes
  @@index([jobId]) // Query all results for a job

  @@map("results")
}
```

**Deliverables**:
- Complete `Result` model in `prisma/schema.prisma`
- Foreign key relationship documented
- Index on `jobId` for efficient queries

**Technical Notes**:
- `@db.Text` supports large content (up to 1GB in PostgreSQL, but application enforces 5MB per Spec 5.1.4)
- `onDelete: Cascade` ensures results are deleted when parent job is deleted (data cleanup)
- Size validation (5MB per result) is enforced in application code before insert (see Spec 5.1.4)
- Index on `jobId` is critical for retrieving all results for a job in ResultsViewer

---

### TRI-1.1-005: Add Schema Comments and Documentation

**Priority**: P1 (High)
**Effort**: 10 minutes
**Dependencies**: TRI-1.1-004

**Description**:
Add comprehensive comments to the Prisma schema explaining the purpose of each model, field, enum, and index. Include references to specification sections and design decisions.

**Acceptance Criteria**:
- [ ] Header comment with schema version and last updated date
- [ ] Model-level comments explaining purpose
- [ ] Enum value comments explaining each state
- [ ] Field comments for non-obvious fields
- [ ] Index comments explaining query patterns
- [ ] References to specification sections where applicable
- [ ] Schema validation passes

**Implementation**:
```prisma
// ============================================================
// Prisma Schema: HX Docling UI Application
// ============================================================
// Version: 1.0.0
// Last Updated: 2025-12-12
// Specification: project/0.3-specification/0.3.1-detailed-specification.md
// Database: PostgreSQL 16+ via PgBouncer
//
// Connection URLs:
// - DATABASE_URL: PgBouncer (port 6432) for application queries
// - DIRECT_DATABASE_URL: Direct PostgreSQL (port 5432) for migrations
//
// Design Notes:
// - Job status follows state machine per Spec 5.1.2
// - Checkpoints enable resumption per Spec 5.5
// - Indexes optimized for history queries and status filtering
// - Result size limits enforced in application code (5MB per result)
// ============================================================

// [Continue with existing generator, datasource, models...]
```

**Deliverables**:
- Comprehensive header comment
- Model and field comments throughout schema
- Documentation of design decisions

**Technical Notes**:
- Comments are preserved in generated Prisma Client and appear in IDE tooltips
- Specification references help future developers understand design rationale
- Schema version should be updated with each migration

---

### TRI-1.1-006: Validate Schema and Generate Prisma Client

**Priority**: P0 (Blocking)
**Effort**: 10 minutes
**Dependencies**: TRI-1.1-005

**Description**:
Validate the complete Prisma schema, generate the Prisma Client, and verify that TypeScript types are correctly generated for Job and Result models.

**Acceptance Criteria**:
- [ ] `npx prisma validate` succeeds with no errors
- [ ] `npx prisma generate` succeeds
- [ ] `node_modules/@prisma/client` directory created
- [ ] TypeScript types generated for Job, Result, JobStatus
- [ ] No type errors when importing PrismaClient in TypeScript
- [ ] Schema formatted with `npx prisma format`

**Commands**:
```bash
# Validate schema syntax and semantics
npx prisma validate

# Format schema (consistent formatting)
npx prisma format

# Generate Prisma Client
npx prisma generate

# Verify client generation
ls -la node_modules/@prisma/client/

# Test import in TypeScript (quick validation)
cat > /tmp/test-prisma.ts << 'EOF'
import { PrismaClient, JobStatus } from '@prisma/client';

const prisma = new PrismaClient();

// Type check: this should compile without errors
const job: { status: JobStatus } = { status: 'PENDING' };

console.log('Prisma Client types validated');
EOF

npx tsx /tmp/test-prisma.ts
```

**Deliverables**:
- Schema validation output (no errors)
- Generated Prisma Client in `node_modules/@prisma/client`
- Confirmation that TypeScript types work correctly

**Technical Notes**:
- `prisma generate` creates TypeScript types from schema
- Generated types include: `Job`, `Result`, `JobStatus`, `Prisma` namespace
- Client generation is required before running any database operations
- If validation fails, review error messages for syntax or type issues

---

### TRI-1.1-007: Review Schema with Neo and Alex

**Priority**: P1 (High)
**Effort**: 15 minutes
**Dependencies**: TRI-1.1-006

**Description**:
Conduct a peer review of the Prisma schema with Neo Anderson (Next.js Developer) and Alex Rivera (Architect) to ensure it meets application requirements and follows best practices.

**Acceptance Criteria**:
- [ ] Schema reviewed by Neo (application perspective)
- [ ] Schema reviewed by Alex (architecture perspective)
- [ ] Field types appropriate for data (UUID, VarChar sizes, BigInt for file size)
- [ ] Indexes aligned with expected queries
- [ ] Relationships correctly defined (Job 1:N Results, cascade delete)
- [ ] Enum values match application state machine
- [ ] No missing fields identified
- [ ] Review feedback documented and addressed

**Review Checklist**:
- [ ] Do field types match application usage? (e.g., `inputUrl` String vs Text)
- [ ] Are indexes sufficient for history queries (`sessionId`, `createdAt`)?
- [ ] Is `checkpointData Json` flexible enough for resumption?
- [ ] Are error tracking fields adequate (`errorCode`, `errorMessage`, `errorDetails`)?
- [ ] Does `onDelete: Cascade` make sense for Results?
- [ ] Are `VarChar` sizes appropriate? (e.g., 255 for sessionId, 2048 for URL)
- [ ] Is `BigInt` needed for `fileSize`, or would `Int` suffice?

**Deliverables**:
- Review meeting notes or async review comments
- List of any schema changes required
- Sign-off from Neo and Alex confirming schema is ready

**Technical Notes**:
- Neo should validate that schema supports all application queries (see Spec Section 4.5 History endpoint)
- Alex should validate that schema aligns with architecture decisions (ADR-004)
- Any schema changes should be made before Sprint 1.2 migrations
- Document any deferred features (e.g., full-text search, analytics tables)

---

## Dependencies

**Upstream**:
- Sprint 0 PostgreSQL prerequisites validation (TRI-0-007)
- Next.js project initialization by Neo Anderson

**Downstream**:
- Sprint 1.2 Prisma migration creation (TRI-1.2-002)
- Sprint 1.2 Prisma client singleton (TRI-1.2-001)

---

## Validation

The Prisma schema is considered complete when:
1. All tasks TRI-1.1-001 through TRI-1.1-007 are complete
2. `npx prisma validate` succeeds with no errors
3. `npx prisma generate` creates TypeScript types
4. Schema review approved by Neo and Alex
5. No missing fields or indexes identified

---

## Risk Mitigation

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Missing fields discovered in Sprint 1.2 | Medium | Medium | Comprehensive review with Neo and Alex in TRI-1.1-007 |
| Index performance issues in production | Low | Medium | Indexes based on query patterns from Spec Section 4.5 |
| Schema migration conflicts | Low | High | Use `directUrl` for migrations to bypass PgBouncer |
| Enum value mismatch with application | Low | High | Sync enum values with Spec 5.1.2 state machine |

---

## Total Effort Summary

**Total Estimated Effort**: 1.5 hours (110 minutes)
**Task Breakdown**:
- TRI-1.1-001: 15 minutes (Base configuration)
- TRI-1.1-002: 10 minutes (JobStatus enum)
- TRI-1.1-003: 30 minutes (Job model)
- TRI-1.1-004: 20 minutes (Result model)
- TRI-1.1-005: 10 minutes (Documentation)
- TRI-1.1-006: 10 minutes (Validation)
- TRI-1.1-007: 15 minutes (Review)

**Critical Path**: Sequential execution (TRI-1.1-001 → 002 → 003 → 004 → 005 → 006 → 007)

---

## Deliverables Checklist

- [ ] `prisma/schema.prisma` created with all models, enums, indexes
- [ ] Schema validated with `npx prisma validate`
- [ ] Prisma Client generated with `npx prisma generate`
- [ ] Schema review completed and approved
- [ ] Schema committed to version control
- [ ] Ready for Sprint 1.2 migration creation
