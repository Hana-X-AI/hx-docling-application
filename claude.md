# Claude AI Assistant Guide: hx-docling-ui Project

**Project**: hx-docling-ui  
**Phase**: 1 - Development  
**Charter Version**: 0.6.0 (✅ APPROVED)  
**Last Updated**: 2025-12-11  
**Target AI**: Claude Sonnet 4 (Claude Code)

---

## 🎯 Quick Start: What You Need to Know

You are assisting with development of **hx-docling-ui**, a Next.js 16 web application that provides a user interface for IBM Granite Docling document processing capabilities. This is **Phase 1: Development only** - running on bare metal Node.js, NO Docker.

### Critical Context at a Glance

| Aspect | Value |
|--------|-------|
| **What** | Web UI for document processing (PDF, Word, Excel, PowerPoint, URLs) |
| **Where** | hx-cc-server (192.168.10.224:3000) - development environment |
| **Stack** | Next.js 16 + TypeScript + shadcn/ui + Prisma + Redis |
| **Phase** | Phase 1 Development ONLY (Docker/deployment is Phase 2) |
| **Status** | Charter approved, all review findings addressed, ready to start |

---

## 📋 Table of Contents

1. [Project Overview](#1-project-overview)
2. [What to DO ✅](#2-what-to-do-)
3. [What NOT to Do ❌](#3-what-not-to-do-)
4. [Technical Architecture](#4-technical-architecture)
5. [Development Workflow](#5-development-workflow)
6. [Team Context](#6-team-context)
7. [Lessons Learned](#7-lessons-learned)
8. [Quality Standards](#8-quality-standards)
9. [Troubleshooting](#9-troubleshooting)
10. [Reference Documents](#10-reference-documents)

---

## 1. Project Overview

### 1.1 What Are We Building?

A professional web application that allows users to:
- Upload documents (PDF, Word, Excel, PowerPoint, images)
- Process web pages via URL
- See real-time processing progress via Server-Sent Events (SSE)
- View results in multiple formats (Markdown, HTML, JSON)
- Access processing history and re-download past results

### 1.2 Key Business Value

- **Accessibility**: Expose document processing to non-technical users
- **Persistence**: Store results for audit trail and re-use
- **Resilience**: Robust error recovery and SSE reconnection
- **Foundation**: Extensible platform for future document AI features

### 1.3 Project Constraints

| Constraint | Requirement |
|------------|-------------|
| **No External Dependencies** | All processing via on-premises HX-Infrastructure |
| **Model Serving** | IBM Granite Docling 258M on hx-ollama3-server |
| **Technology Stack** | Next.js 16 + shadcn/ui (per hx-shadcn-server standard) |
| **Data Persistence** | PostgreSQL (jobs/results) + Redis (sessions) |
| **Phase Scope** | Development ONLY - no Docker, no deployment |

### 1.4 Success Criteria (Phase 1 Complete When...)

- [ ] Application runs at `http://hx-cc-server.hx.dev.local:3000`
- [ ] User can drag-drop PDF/Word/Excel/PowerPoint and see output
- [ ] User can paste URL and see processed webpage
- [ ] Real-time progress displays during processing
- [ ] Results persist to PostgreSQL
- [ ] History view shows past jobs with re-download capability
- [ ] Session tracking works via Redis
- [ ] Zero TypeScript errors, zero ESLint errors
- [ ] All unit tests pass
- [ ] `npm run build` succeeds

---

## 2. What to DO ✅

### 2.1 Environment Setup

```bash
# Project documentation location
# /home/agent0/hx-docling-application/ (charter, governance docs)

# Application development location (to be created in Sprint 1.1)
cd /home/agent0/hx-docling-ui/

# Development server
npm run dev  # Runs on port 3000

# File storage
mkdir -p /data/docling-uploads/
mkdir -p /tmp/docling-processing/
```

### 2.2 Technology Stack (Required)

| Layer | Technology | Version | Purpose |
|-------|------------|---------|---------|
| Framework | Next.js | 16.x (App Router) | Application framework |
| Runtime | Node.js | >= 20.9.0 | JavaScript runtime |
| Language | TypeScript | >= 5.1.0 | Type safety |
| UI Components | shadcn/ui | Latest | Component library |
| Styling | Tailwind CSS | 3.4.x | CSS framework |
| State Management | Zustand | 5.x | Client state |
| Validation | Zod | 3.x | Runtime validation |
| Database ORM | Prisma | 5.x | PostgreSQL access |
| Redis Client | ioredis | 5.x | Session management |
| Icons | Lucide React | Latest | Icon library |

### 2.3 MCP Integration (8 Tools in Phase 1)

Connect to: `http://hx-docling-mcp-server.hx.dev.local:8000/mcp`

| Tool | File Types | Purpose |
|------|------------|---------|
| `convert_pdf` | .pdf | PDF conversion |
| `convert_docx` | .doc, .docx | Word conversion |
| `convert_xlsx` | .xls, .xlsx | Excel conversion |
| `convert_pptx` | .ppt, .pptx | PowerPoint conversion |
| `convert_url` | URLs | Web page conversion |
| `export_markdown` | - | Export to Markdown |
| `export_html` | - | Export to HTML |
| `export_json` | - | Export to JSON |

### 2.4 Database Setup (Prisma)

**Connection Strings:**
```env
# PostgreSQL (hx-postgres-server)
DATABASE_URL=postgresql://docling_user:${DB_PASSWORD}@hx-postgres-server.hx.dev.local:5432/docling_db

# Redis (hx-redis-server)
REDIS_URL=redis://hx-redis-server.hx.dev.local:6379/0
```

**Schema Overview:**
- `Job` table: Processing jobs with status, metadata, timestamps
- `Result` table: Processed outputs (Markdown, HTML, JSON)
- Session tracking via Redis (24h TTL)
- File storage at `/data/docling-uploads/` (30-day retention)

### 2.5 Quality Gates (Run Before Each Commit)

```bash
# Type checking
npx tsc --noEmit

# Linting
npm run lint

# Unit tests
npm run test

# Build verification
npm run build

# Prisma schema check
npx prisma validate

# MCP health check
curl http://hx-docling-mcp-server.hx.dev.local:8000/health
```

### 2.6 File Structure (Follow This)

```
hx-docling-ui/
├── src/
│   ├── app/                    # Next.js App Router
│   │   ├── layout.tsx
│   │   ├── page.tsx
│   │   ├── history/
│   │   │   └── page.tsx
│   │   └── api/
│   │       ├── upload/route.ts
│   │       ├── process/route.ts    # SSE endpoint
│   │       ├── history/route.ts
│   │       └── health/route.ts
│   ├── components/
│   │   ├── ui/                 # shadcn/ui components
│   │   ├── upload/             # UploadZone, FilePreview
│   │   ├── processing/         # ProgressCard, StatusBadge
│   │   ├── results/            # ResultsViewer, MarkdownView
│   │   └── history/            # HistoryView, JobRow
│   ├── lib/
│   │   ├── db/prisma.ts        # Prisma client
│   │   ├── redis/client.ts     # Redis client
│   │   ├── mcp/                # MCP integration
│   │   ├── sse/                # SSE manager with reconnection
│   │   └── validation/         # Zod schemas
│   ├── stores/                 # Zustand stores
│   ├── hooks/                  # Custom React hooks
│   └── types/                  # TypeScript types
├── prisma/
│   ├── schema.prisma
│   └── migrations/
└── scripts/
    └── cleanup-uploads.sh
```

---

## 3. What NOT to Do ❌

### 3.1 Docker - ABSOLUTELY PROHIBITED in Phase 1

❌ **DO NOT** create a Dockerfile  
❌ **DO NOT** create docker-compose.yml  
❌ **DO NOT** reference Docker in any configuration  
❌ **DO NOT** containerize the application  
❌ **DO NOT** mention Docker in documentation  
❌ **DO NOT** deploy to hx-dev-server (that's Phase 2)

### 3.2 Scope Boundaries (Out of Scope)

❌ **DO NOT** implement user authentication (sessions are anonymous)  
❌ **DO NOT** implement the 13 backlog MCP tools (see Section 4.5 below)  
❌ **DO NOT** add features not in the charter  
❌ **DO NOT** use Redux (use Zustand)  
❌ **DO NOT** use Pages Router (use App Router)  
❌ **DO NOT** use external CDNs or third-party services  
❌ **DO NOT** develop on any server other than hx-cc-server

### 3.3 Development Anti-Patterns

❌ **DO NOT** skip the integration checkpoint after Sprint 1.5  
❌ **DO NOT** proceed without MCP health check passing  
❌ **DO NOT** commit .env files with secrets  
❌ **DO NOT** skip TypeScript/ESLint validation before pushing  

---

## 4. Technical Architecture

### 4.1 System Architecture

```
┌─────────────┐
│   Users     │
└──────┬──────┘
       │
       ▼
┌─────────────────────────────────────────┐
│  hx-docling-ui (Next.js 16)             │
│  ┌─────────────────────────────────┐   │
│  │ Presentation Layer              │   │
│  │ - UploadZone, UrlInput          │   │
│  │ - ResultsViewer, HistoryView    │   │
│  └─────────────────────────────────┘   │
│  ┌─────────────────────────────────┐   │
│  │ Application Layer               │   │
│  │ - Zustand Store                 │   │
│  │ - MCP Client                    │   │
│  │ - SSE Manager (reconnection)    │   │
│  └─────────────────────────────────┘   │
│  ┌─────────────────────────────────┐   │
│  │ Data Layer                      │   │
│  │ - Prisma Client                 │   │
│  │ - Redis Client                  │   │
│  └─────────────────────────────────┘   │
└──────┬──────────┬───────────┬──────────┘
       │          │           │
       ▼          ▼           ▼
┌─────────┐  ┌────────┐  ┌────────┐
│   MCP   │  │  Postgres│ │ Redis  │
│ Server  │  │  Server  │ │ Server │
└─────────┘  └────────┘  └────────┘
```

### 4.2 Data Flow (User Uploads PDF)

1. User drags PDF into UploadZone
2. Zustand store: `setFile(file)`
3. API route `/api/upload` receives file
4. File stored at `/data/docling-uploads/{uuid}.pdf`
5. Prisma creates Job record (status: PENDING)
6. User clicks "Process"
7. SSE connection opened to `/api/process`
8. API calls MCP `convert_pdf` tool
9. MCP returns progress events via SSE
10. SSE Manager handles reconnection if network drops
11. MCP returns DoclingDocument
12. API calls `export_markdown`, `export_html`, `export_json`
13. If some exports fail → Job status = PARTIAL_COMPLETE, available results still stored
14. Prisma stores Results (all successful exports)
15. Job status updated to COMPLETE (or PARTIAL_COMPLETE if some failed)
16. UI displays tabbed results (Markdown/HTML/JSON), shows which exports failed if partial

### 4.3 SSE Resilience Strategy

**Critical Implementation Detail:**

- **Max Retries**: 10 attempts
- **Backoff**: Exponential (1s → 2s → 4s → 8s → 16s → 30s max)
- **Grace Period**: 30 seconds total before giving up
- **Fallback**: Polling at 2-second intervals if SSE exhausted
- **State Recovery**: Use `Last-Event-ID` header on reconnect
- **User Feedback**: Show "Reconnecting..." toast, success on reconnect

```typescript
// SSE Reconnection Config (lib/sse/manager.ts)
interface SSEConfig {
  maxRetries: 10;
  backoffStrategy: 'exponential';
  backoffBase: 1000;        // 1 second
  backoffMax: 30000;        // 30 seconds max
  gracePeriod: 30000;
  fallbackToPolling: true;
  pollingInterval: 2000;
}
```

### 4.4 Session Management (Redis)

- **Session ID**: UUID stored in HTTP-only cookie
- **TTL**: 24 hours (86400 seconds)
- **Storage**: Redis key `session:{sessionId}`
- **Anonymous**: No user authentication
- **Tracking**: Job count, last activity, active job

**Edge Cases Handled:**
- Session expires during processing → Job continues, new session on next visit
- User closes browser → Job completes server-side
- Multiple tabs → Allowed, share rate limit quota (10 req/min)

### 4.4.1 Export Formats

| Format | Specification | Download Naming |
|--------|---------------|----------------|
| **Markdown** | GitHub Flavored Markdown with YAML frontmatter, images as base64 data URIs | `{filename}_{YYYYMMDD_HHMMSS}_markdown.md` |
| **HTML** | Standalone with inline Tailwind CSS, dark mode support (prefers-color-scheme) | `{filename}_{YYYYMMDD_HHMMSS}_html.html` |
| **JSON** | Docling schema with full AST hierarchy, 2-space indentation | `{filename}_{YYYYMMDD_HHMMSS}_json.json` |
| **Raw** | Pretty-printed DoclingDocument (debugging) | `{filename}_{YYYYMMDD_HHMMSS}_raw.json` |

**Example**: `quarterly_report_20251211_143022_markdown.md`

### 4.5 Backlog Items (DO NOT IMPLEMENT)

These 13 MCP tools are out of scope for Phase 1:

| Tool | Description | Status |
|------|-------------|--------|
| `generate_title` | Auto-generate document title | Backlog |
| `generate_toc` | Generate table of contents | Backlog |
| `generate_sections` | Extract document sections | Backlog |
| `generate_headings` | Extract heading hierarchy | Backlog |
| `generate_paragraphs` | Extract paragraphs | Backlog |
| `generate_lists` | Extract lists | Backlog |
| `generate_tables` | Extract tables | Backlog |
| `generate_images` | Extract images | Backlog |
| `generate_code_blocks` | Extract code blocks | Backlog |
| `generate_references` | Extract references | Backlog |
| `generate_knowledge_graph` | Entity extraction graph | Backlog |
| `split_document` | Split into sections | Backlog |
| `merge_documents` | Merge multiple documents | Backlog |

---

## 5. Development Workflow

### 5.1 Sprint Sequence (9-10 Sessions)

| Sprint | Deliverable | Key Files |
|--------|-------------|-----------|
| **1.1** | Project scaffold, Next.js 16, shadcn, Prisma setup | `package.json`, `next.config.ts`, `prisma/schema.prisma` |
| **1.2** | Database setup, Redis connection, session tracking | `lib/db/prisma.ts`, `lib/redis/client.ts` |
| **1.3** | Upload component, file validation, persistent storage | `UploadZone.tsx`, `/api/upload/route.ts` |
| **1.4** | URL input component | `UrlInput.tsx`, `lib/validation/url.ts` |
| **1.5** | MCP client integration (8 tools), error catalog | `lib/mcp/client.ts`, `/api/process/route.ts` |
| **CHECKPOINT** | **Validate with real MCP server before proceeding** | |
| **1.6** | Results viewer (tabs, renderers) | `ResultsViewer.tsx`, `MarkdownView.tsx` |
| **1.7** | History view (past jobs, re-download) | `HistoryView.tsx`, `/api/history/route.ts` |
| **1.8** | Polish, testing, documentation | Tests, README |

### 5.2 Integration Checkpoint (After Sprint 1.5)

**Required Validations:**

```bash
# 1. MCP health check
curl http://hx-docling-mcp-server.hx.dev.local:8000/health
# Expected: {"status": "healthy"}

# 2. Database connectivity
npx prisma migrate status
# Expected: All migrations applied

# 3. Redis connectivity
redis-cli -h hx-redis-server.hx.dev.local ping
# Expected: PONG

# 4. End-to-end test
# Upload test PDF, verify:
# - File stored in /data/docling-uploads/
# - Job created in PostgreSQL
# - MCP processing succeeds
# - Results stored in database
# - No console errors
```

**Do not proceed to Sprint 1.6 until all checks pass.**

### 5.3 Code Review Checklist

Before requesting review:

- [ ] TypeScript compiles: `npx tsc --noEmit`
- [ ] ESLint passes: `npm run lint`
- [ ] Tests pass: `npm run test`
- [ ] Build succeeds: `npm run build`
- [ ] No console errors in browser DevTools
- [ ] Component renders correctly on mobile (>= 768px)
- [ ] Accessibility score >= 90 (Lighthouse)
- [ ] File follows naming convention (see Section 7.2)
- [ ] Prisma schema validated: `npx prisma validate`

### 5.4 New Component Development Checklist

When creating a new component:

- [ ] TypeScript interface for props with JSDoc comments
- [ ] Zod schema for any form inputs or validation
- [ ] Loading state (Skeleton component)
- [ ] Error state with recovery action
- [ ] Empty state (if component displays lists/data)
- [ ] Accessibility attributes (`aria-*`, `role`)
- [ ] Keyboard navigation support (Tab, Enter, Escape, arrows)
- [ ] Focus management (trapped focus for modals)
- [ ] Unit test file created (`ComponentName.test.tsx`)
- [ ] Component documented in README (if public API)
- [ ] Props validated at runtime (Zod schema)
- [ ] Error boundary wraps async operations

---

## 6. Team Context

### 6.1 Team Structure

| Role | Agent | Responsibility |
|------|-------|----------------|
| **CAIO** | Jarvis Richardson (Agent Zero) | Strategic decisions, final approval |
| **Orchestration** | Agent Zero | Multi-agent coordination, quality gates |
| **Platform Architect** | Alex Rivera | ADR, architecture validation |
| **Testing & QA** | Julia Santos | Test strategy, quality validation |
| **Database & Infra** | William Chen | PostgreSQL, Redis, file storage setup |
| **Primary Developer** | Trinity | Next.js implementation (YOU assist this role) |
| **Frontend UI** | Ola Mae Johnson | UI/UX, accessibility |
| **MCP Integration** | James Dean | MCP client library |

### 6.2 Escalation Path

```
Technical Issue
    ↓
Core Team SME (Alex/Julia/William)
    ↓
Agent Zero (Orchestration)
    ↓
CAIO (Final Authority)
```

### 6.3 Invocation Commands

When suggesting collaboration:

- Platform architecture questions: `@alex`
- Testing strategy questions: `@julia`
- Database/infrastructure questions: `@william`
- Frontend UI questions: `@ola`
- MCP integration questions: `@james`
- Orchestration/escalation: `@agent-zero`

---

## 7. Lessons Learned

### 7.1 Critical Lessons from Past Projects

| ID | Lesson | Impact | Prevention |
|----|--------|--------|------------|
| **LL-003** | Review all reference documentation before starting work | HIGH | Read charter + coding instructions first |
| **LL-006** | Verify task parameters match intended targets | HIGH | Double-check file paths, server names |
| **LL-012** | Reviewer findings must be independently verified | MEDIUM | Don't accept review comments blindly |
| **LL-016** | Enforce lowercase file naming conventions | MEDIUM | Use `lowercase-with-hyphens.tsx` |
| **LL-021** | Requirements-to-task traceability gaps cause scope creep | HIGH | Check charter before adding features |
| **LL-022** | Specification updates indicate earlier traceability gaps | MEDIUM | Reference charter frequently |

### 7.2 Core Commitments

1. ✅ **Always verify** that requested work aligns with charter scope
2. ✅ **Always follow naming conventions**:
   - React Components: `PascalCase.tsx` (e.g., `UploadZone.tsx`)
   - Hooks: `camelCase.ts` (e.g., `useUpload.ts`)
   - Utils/Libs: `kebab-case.ts` (e.g., `mcp-client.ts`)
   - API Routes: `route.ts` in kebab-case folders (e.g., `api/upload/route.ts`)
   - Documentation: `kebab-case.md` (e.g., `team-roster.md`)
3. ✅ **Always validate** generated code against TypeScript/ESLint
4. ✅ **Always check** MCP server health before integration testing
5. ✅ **Never implement** Docker-related features in Phase 1
6. ✅ **Never skip** the integration checkpoint after Sprint 1.5
7. ✅ **Always apply** proper lifecycle management (error recovery, graceful shutdown)

---

## 8. Quality Standards

### 8.1 Code Quality Requirements

| Standard | Requirement | Validation |
|----------|-------------|------------|
| **TypeScript** | Zero errors | `npx tsc --noEmit` |
| **ESLint** | Zero warnings/errors | `npm run lint` |
| **Test Coverage** | 100% for utils/libs | `npm run test:coverage` |
| **Build** | Must succeed | `npm run build` |
| **Accessibility** | Lighthouse >= 90 | Manual audit |
| **Performance** | Lighthouse >= 80 | Manual audit |

### 8.2 Component Standards

```typescript
// Example: Every component must have types
interface UploadZoneProps {
  onFileSelect: (file: File) => void;
  disabled?: boolean;
  className?: string;
}

export function UploadZone({ onFileSelect, disabled, className }: UploadZoneProps) {
  // Implementation
}

// Example: Every API route must validate input
export async function POST(req: Request) {
  const schema = z.object({
    sessionId: z.string().uuid(),
    jobId: z.string().uuid(),
  });
  
  const body = await req.json();
  const validated = schema.parse(body); // Throws if invalid
  
  // Process...
}
```

### 8.3 Error Handling Standards

**Every error must have:**
1. Error code (E001-E999)
2. User-friendly message
3. Technical details (for logs)
4. Recovery action

```typescript
// Example error structure
interface AppError {
  code: string;           // "E201"
  message: string;        // "MCP server unavailable"
  details?: string;       // "Connection timeout after 5s"
  recovery: string;       // "Check MCP server health and retry"
  recoveryAction?: () => void;
}
```

### 8.4 Performance Standards

| Metric | Target | Measurement |
|--------|--------|-------------|
| LCP (Largest Contentful Paint) | < 2.5s | Lighthouse |
| FCP (First Contentful Paint) | < 1.8s | Lighthouse |
| Time to first progress update | < 500ms | Manual timing |
| SSE reconnection time | < 30s | Network simulation |

---

## 9. Troubleshooting

### 9.1 Common Issues & Solutions

#### Issue: TypeScript errors in Prisma client

```bash
# Solution: Regenerate Prisma client
npx prisma generate
```

#### Issue: Cannot connect to PostgreSQL

```bash
# Check connection
psql -h hx-postgres-server.hx.dev.local -U docling_user -d docling_db

# Verify network
ping hx-postgres-server.hx.dev.local
```

#### Issue: Redis connection fails

```bash
# Check Redis
redis-cli -h hx-redis-server.hx.dev.local ping

# Verify network
ping hx-redis-server.hx.dev.local
```

#### Issue: MCP server returns 500 error

```bash
# Check MCP health
curl http://hx-docling-mcp-server.hx.dev.local:8000/health

# Check MCP logs on server (if accessible)
ssh hx-docling-mcp-server.hx.dev.local
journalctl -u docling-mcp -f
```

#### Issue: File upload fails

```bash
# Check directory permissions
ls -la /data/docling-uploads/

# Verify disk space
df -h /data/

# Check file size limits (100 MB max)
```

#### Issue: HTTP 429 "Too Many Requests"

```bash
# Cause: Rate limit exceeded (10 requests/minute per session)

# Solution: Wait for Retry-After header duration
# Check current session usage in Redis:
redis-cli -h hx-redis-server.hx.dev.local GET "ratelimit:{sessionId}"

# Rate limit resets after 1 minute window
# Each processing request counts as 1 request
```

#### Issue: SSE connection drops immediately

```typescript
// Common causes:
// 1. CORS not configured (add headers to API route)
// 2. Nginx/proxy timeout (increase timeout)
// 3. Client closed connection (check browser console)

// Check SSE headers in API route:
return new Response(stream, {
  headers: {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
  },
});
```

### 9.2 Environment Validation Script

```bash
#!/bin/bash
# Save as: scripts/validate-environment.sh

echo "=== Environment Validation ==="

# Check Node.js version
node_version=$(node -v)
echo "✓ Node.js: $node_version"

# Check npm
npm_version=$(npm -v)
echo "✓ npm: $npm_version"

# Check PostgreSQL connectivity
if pg_isready -h hx-postgres-server.hx.dev.local -p 5432 > /dev/null 2>&1; then
  echo "✓ PostgreSQL: Connected"
else
  echo "✗ PostgreSQL: Connection failed"
fi

# Check Redis connectivity
if redis-cli -h hx-redis-server.hx.dev.local ping > /dev/null 2>&1; then
  echo "✓ Redis: Connected"
else
  echo "✗ Redis: Connection failed"
fi

# Check MCP server
mcp_health=$(curl -s http://hx-docling-mcp-server.hx.dev.local:8000/health)
if [ $? -eq 0 ]; then
  echo "✓ MCP Server: $mcp_health"
else
  echo "✗ MCP Server: Connection failed"
fi

# Check file storage
if [ -d "/data/docling-uploads" ]; then
  echo "✓ File storage: /data/docling-uploads exists"
else
  echo "✗ File storage: /data/docling-uploads not found"
fi

echo "=== Validation Complete ==="
```

---

## 10. Reference Documents

### 10.1 Primary Documents

| Document | Location | Purpose |
|----------|----------|---------|
| **Charter** | `/home/agent0/hx-docling-application/project/0.0-charter/0.1-hx-docling-ui-charter.md` | Complete project specification |
| **Charter Review** | `/home/agent0/hx-docling-application/project/0.0-charter/reviews/` | Deep review findings (all addressed) |
| **Coding Instructions** | `/home/agent0/hx-docling-application/project/0.6-governance/0.6.0-laude-code-instructions.md` | Detailed technical guidance |
| **Team Roster** | `/home/agent0/hx-docling-application/project/0.6-governance/0.6.5-team-roster.md` | Agent roles and responsibilities |
| **Lessons Learned** | `/home/agent0/hx-docling-application/project/0.6-governance/0.6.4-lessons-learned.md` | Past project learnings |

### 10.2 Infrastructure Servers

| Server | IP | Port | Purpose |
|--------|-----|------|---------|
| hx-cc-server | 192.168.10.224 | 3000 | Development environment |
| hx-docling-mcp-server | 192.168.10.217 | 8000 | Document processing (MCP) |
| hx-postgres-server | 192.168.10.208 | 5432 | Job/results persistence |
| hx-redis-server | 192.168.10.209 | 6379 | Session tracking |
| hx-ollama3-server | 192.168.10.206 | 11434 | Granite Docling 258M model |
| hx-litellm-server | 192.168.10.212 | 4000 | LLM gateway |

### 10.3 Quick Command Reference

```bash
# Start development server
npm run dev

# Type check
npx tsc --noEmit

# Lint
npm run lint

# Test
npm run test

# Build
npm run build

# Prisma commands
npx prisma generate          # Generate client
npx prisma migrate dev       # Run migrations (dev)
npx prisma migrate status    # Check migration status
npx prisma studio            # Open database GUI

# Health checks
curl http://localhost:3000/api/health                          # Application health
curl http://hx-docling-mcp-server.hx.dev.local:8000/health    # MCP server
redis-cli -h hx-redis-server.hx.dev.local ping                # Redis
pg_isready -h hx-postgres-server.hx.dev.local                 # PostgreSQL

# File cleanup (cron job)
bash scripts/cleanup-uploads.sh
```

### 10.4 Error Code Quick Reference

| Range | Category | Examples |
|-------|----------|----------|
| **E0xx** | File errors | E001 = File too large, E002 = Invalid type, E003 = Corrupted |
| **E1xx** | URL errors | E101 = Invalid URL, E102 = Unreachable, E103 = Timeout |
| **E2xx** | MCP errors | E201 = MCP unavailable, E202 = Invalid request, E203 = Processing failed |
| **E3xx** | Processing | E301 = Timeout, E302 = Partial failure, E303 = Format error |
| **E4xx** | Database | E401 = Connection failed, E402 = Query failed, E403 = Persist failed |
| **E5xx** | Session | E501 = Session expired, E502 = Invalid session, E503 = Session conflict |
| **E6xx** | Rate limit | E601 = Limit exceeded, E602 = Quota depleted |
| **E9xx** | Network | E901 = Network error, E902 = SSE failed, E903 = Reconnect failed |

**Error Structure (All Errors):**
- Code: E001-E999
- User message: Friendly explanation
- Technical details: For logs
- Recovery action: What user should do

### 10.5 Required Environment Variables

```env
# Database
DATABASE_URL=postgresql://docling_user:${DB_PASSWORD}@hx-postgres-server.hx.dev.local:5432/docling_db

# Redis
REDIS_URL=redis://hx-redis-server.hx.dev.local:6379/0

# MCP Server
DOCLING_MCP_URL=http://hx-docling-mcp-server.hx.dev.local:8000/mcp
DOCLING_MCP_TIMEOUT=300000

# File Storage
MAX_FILE_SIZE_MB=100
ALLOWED_FILE_TYPES=.pdf,.docx,.doc,.pptx,.ppt,.xlsx,.xls,.png,.jpg,.jpeg,.tiff
TEMP_STORAGE_PATH=/tmp/docling-processing
PERSISTENT_STORAGE_PATH=/data/docling-uploads

# Application
NEXT_PUBLIC_APP_VERSION=1.0.0-dev
NODE_ENV=development
```

---

## 11. Final Reminders

### ✅ Always Remember

1. **This is Phase 1 Development** - NO Docker, NO deployment
2. **Charter is approved** - All 23 review findings addressed
3. **Follow the team structure** - Suggest appropriate agent when help needed
4. **Quality first** - Zero TypeScript errors, 100% test coverage
5. **Integration checkpoint is mandatory** - After Sprint 1.5
6. **SSE resilience is critical** - Implement reconnection properly
7. **File naming is lowercase** - `upload-zone.tsx`, not `UploadZone.tsx`
8. **Reference the charter** - When in doubt, check the charter

### 🎯 Success Indicators

- User can process PDF → Markdown in under 10 seconds
- SSE reconnects automatically after network drop
- History view loads past jobs with pagination
- All 20 formal acceptance criteria met (see charter Section 17)
- Zero console errors
- Lighthouse scores: Accessibility >= 90, Performance >= 80

### 📞 When to Ask for Help

- **Architectural decisions** → Suggest: "Should we consult @alex on this?"
- **Testing strategy** → Suggest: "Let's get @julia's input on test coverage"
- **Database schema changes** → Suggest: "We should verify with @william"
- **UI/UX concerns** → Suggest: "Let's check with @ola on accessibility"
- **MCP integration issues** → Suggest: "This might need @james's expertise"
- **Scope questions** → Suggest: "Let's clarify with @agent-zero"

---

## 12. Project Status

| Aspect | Status |
|--------|--------|
| **Charter** | ✅ v0.6.0 Approved |
| **Review Findings** | ✅ All 23 addressed |
| **Team Roster** | ✅ Approved |
| **Coding Instructions** | ✅ Complete |
| **Development Environment** | 🟡 To be set up in Sprint 1.1 |
| **Database Setup** | 🟡 To be set up in Sprint 1.2 |
| **Implementation** | ⏸️ Ready to start |

**Next Step:** Sprint 1.1 - Project scaffold, Next.js 16, shadcn/ui, Prisma setup

---

## 13. Document Change Log

### Version 1.1.0 (2025-12-11)

**Improvements:**
1. ✅ Fixed path inconsistency - clarified project structure (Section 2.1)
2. ✅ Updated acceptance criteria count from 16 to 20 (Section 11)
3. ✅ Clarified file naming conventions by file type (Section 7.2)
4. ✅ Added export format details with download naming (Section 4.4.1)
5. ✅ Added partial results handling to data flow (Section 4.2)
6. ✅ Added rate limiting troubleshooting (Section 9.1)
7. ✅ Added error code quick reference table (Section 10.4)
8. ✅ Added required environment variables (Section 10.5)
9. ✅ Added component development checklist (Section 5.4)

**Status**: ⭐⭐⭐⭐⭐ (5/5) - All review findings addressed

---

**Document Version**: 1.1.0  
**Created**: 2025-12-11  
**Last Updated**: 2025-12-11  
**For**: Claude Sonnet 4 (Claude Code)  
**Project Phase**: Phase 1 Development

*This guide consolidates all critical context for AI-assisted development. Always refer to primary documents (charter, coding instructions) for authoritative details.*
