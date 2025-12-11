# Deep Charter Review: hx-docling-ui
**Document**: `0.1-hx-docling-ui-charter.md`  
**Version Reviewed**: 0.6.0 (✅ APPROVED)  
**Review Date**: December 11, 2025  
**Reviewer**: Code Analysis Agent  
**Status**: COMPREHENSIVE ANALYSIS - 23 Findings (All Addressed in v0.6.0)

---

## Executive Summary

The charter is **well-structured, technically sound, and ready for implementation**, with comprehensive coverage of Phase 1 development. The document demonstrates excellent architectural planning with clear constraints, assumptions, and extensibility framework. However, **9 critical gaps** have been identified that require attention before development commences, primarily around:

1. **Error recovery and resilience patterns** not explicitly documented
2. **Session management edge cases** not fully specified
3. **SSE reconnection strategy** undefined
4. **Performance monitoring and metrics** missing
5. **Database migration strategy** not detailed

**Overall Assessment**: ⭐⭐⭐⭐ (4/5) - Excellent strategic document with identified tactical gaps

---

## 1. CRITICAL FINDINGS (9)

### 1.1 [CRITICAL] Missing SSE Reconnection Strategy

**Section**: 6.7 (Data Flow), 15 (Risks & Mitigations)  
**Severity**: Critical  
**Impact**: User experience degradation if network connection drops

**Issue**:
The document mentions SSE for progress updates and identifies "SSE connection drops" as Risk R-5 with "Auto-reconnect" as mitigation, but **no implementation details are provided**:
- No reconnection backoff strategy specified
- No max retry attempts defined
- No fallback if reconnection fails
- No client-side state reconciliation logic documented

**Example Gap**:
```typescript
// What happens here?
UI->>API: POST /api/process (SSE)
// ... user's network drops after 40% progress
// ... document says "auto-reconnect" but HOW?
```

**Recommendation**:
Add a new subsection "6.8 SSE Resilience Strategy" documenting:
- Exponential backoff (1s, 2s, 4s, 8s max)
- Max retry attempts (5-10)
- Fallback to polling if SSE unavailable
- State reconciliation after reconnect
- Grace period (e.g., 30s) before giving up

**Implementation Impact**: Medium (affects /api/process route handler and useProcess hook)

---

### 1.2 [CRITICAL] Session Management Edge Cases Undefined

**Section**: 7.4 (Session Management), 11.1 (Security)  
**Severity**: Critical  
**Impact**: Potential data loss or session confusion

**Issue**:
Session management is documented but edge cases are not addressed:

| Edge Case | Current Spec | Problem |
|-----------|--------------|---------|
| Session cookie expires during processing | TTL: 24 hours | What if user's session times out mid-job? Result orphaned? |
| User closes browser mid-processing | N/A | Can they resume? Is job still running on server? |
| User clears browser cookies | N/A | Session lost. How to recover history? |
| Session ID collision (unlikely but...) | N/A | No mention of collision handling |
| Multiple tabs in same session | Allowed implicitly | Can two tabs process simultaneously? Race conditions? |
| Database connection lost mid-process | N/A | Job persists to DB - what if write fails? |

**Current Documentation**:
```typescript
interface Session {
  id: string;           // UUID
  createdAt: number;    // Unix timestamp
  lastActivity: number; // Unix timestamp
  jobCount: number;     // Number of jobs in session
}
// TTL: 24 hours (86400 seconds)
```

**Missing**:
- Session reaping/cleanup strategy
- Job cleanup if session expires
- User notification if session lost
- Multiple-tab conflict resolution

**Recommendation**:
Expand Section 7.4 with subsection "7.4.1 Session Edge Cases":
- Document each edge case
- Specify expected behavior
- Define timeout cascades
- Add handling code examples

**Implementation Impact**: High (affects middleware, session recovery UI, database cleanup)

---

### 1.3 [CRITICAL] MCP Error Recovery Strategy Missing

**Section**: 8.1, 15 (Risks & Mitigations)  
**Severity**: Critical  
**Impact**: Loss of user data/progress on MCP failures

**Issue**:
Document specifies MCP integration requirements (timeout: 300s, retry: 3 attempts, exponential backoff) but does NOT document:

1. **What happens after 3 retries fail?**
   - Does job state revert to PENDING?
   - Can user retry manually?
   - Is partial progress lost?

2. **MCP tool-level error handling**
   - `convert_pdf` fails → what's the error UX?
   - `export_markdown` fails but `export_html` succeeds → partial results?

3. **Timeout handling**
   - After 300s timeout, can user continue?
   - Is there a "processing took too long" UX path?

4. **Partial completion**
   - If `convert_pdf` succeeds but `export_markdown` fails
   - Should we show partial results or fail entirely?

**Current Spec**:
```
Retry: 3 attempts with exponential backoff (1s, 2s, 4s)
Timeout: 300 seconds (large documents)
```

**Missing**:
- Post-failure UX flow
- Partial result handling
- Job state machine for failures
- User recovery actions
- Error catalog integration with UI

**Recommendation**:
Add Section "8.6 MCP Error Recovery Strategy" with:
- State machine diagram (PROCESSING → RETRY_1 → RETRY_2 → RETRY_3 → ERROR)
- UI flow for each failure scenario
- Partial result handling policy
- Error code mapping to user messages
- Manual retry button placement

**Implementation Impact**: High (affects /api/process route, error handling, results persistence)

---

### 1.4 [CRITICAL] Database Migration & Schema Evolution Not Specified

**Section**: 7.3 (Database Schema), 12.3 (Setup Commands)  
**Severity**: Critical  
**Impact**: Deployment friction, potential data loss

**Issue**:
Prisma schema is documented but migration strategy is not:

1. **Initial migration**
   - `npx prisma migrate dev` mentioned in setup but...
   - What if database already has partial schema?
   - Idempotency not guaranteed
   - No rollback strategy documented

2. **Schema evolution**
   - How to add new columns/tables mid-project?
   - Backwards compatibility requirements?
   - Zero-downtime migration strategy?

3. **Seed data**
   - Should database have test data?
   - How to populate initial state?
   - No seed script documented

4. **Testing database**
   - Separate test database or shared?
   - How to reset between test runs?
   - Snapshot strategy?

**Current Setup Command**:
```bash
npx prisma migrate dev
```

**Missing**:
- Migration naming convention
- Rollback procedure
- Production deployment migration plan
- Zero-downtime strategy
- Seed script location
- Test data initialization

**Recommendation**:
Add Section "12.4 Database Management Strategy":
- Document migration naming: `YYYYMMDD_description`
- Create `prisma/migrations/` structure
- Document rollback procedure
- Create separate `prisma/seed.ts` script
- Add `npx prisma db seed` to setup
- Document test database reset strategy

**Implementation Impact**: High (affects setup, CI/CD, data integrity)

---

### 1.5 [CRITICAL] Health Check Implementation Undefined

**Section**: 8.5 (Health Checks)  
**Severity**: Critical  
**Impact**: Silent failures, no monitoring capability

**Issue**:
Health checks are mentioned as requirement but NOT implemented:

| Check | Spec | Implementation |
|-------|------|---|
| Docling MCP | `GET /health` | ✅ On page load |
| PostgreSQL | Prisma connection test | ❌ How? When? |
| Redis | `PING` command | ❌ How? When? |

**Current Requirement**:
```
| PostgreSQL | Prisma connection test | Yes | On startup |
| Redis      | PING command           | Yes | On startup |
```

**Missing**:
- `/api/health` endpoint specification
- What "on startup" means (app startup? page load? both?)
- Health check timeout values
- Response format specification
- Dependency cascade logic (if DB fails, fail health check)
- Caching strategy (don't hammer services)
- Logging/alerting on failures

**Recommendation**:
Add Section "8.6 Health Check Implementation":
```typescript
// Spec for /api/health endpoint
interface HealthCheck {
  status: 'healthy' | 'degraded' | 'unhealthy';
  timestamp: ISO8601;
  checks: {
    mcp: { status: 'ok' | 'error'; latency: number; };
    postgres: { status: 'ok' | 'error'; latency: number; };
    redis: { status: 'ok' | 'error'; latency: number; };
  };
}

// Cache strategy: 30s
// Timeout per service: 5s
// Log failures to console
```

**Implementation Impact**: Medium (affects /api/health route, startup flow, UI loading state)

---

### 1.6 [CRITICAL] Result Persistence Strategy for Large Documents Not Specified

**Section**: 7.2, 7.6  
**Severity**: Critical  
**Impact**: Disk/DB space exhaustion, slow page loads

**Issue**:
Result persistence shows retention policy (90 days) but doesn't address:

1. **Large result handling**
   - 100 MB PDF → potentially large Markdown/HTML/JSON exports
   - Max result size per format?
   - Compression strategy?
   - Text-only or binary-safe storage?

2. **Storage capacity planning**
   - How much space will 90 days retain?
   - No calculation provided
   - Monitoring/alerting strategy?

3. **Query performance**
   - Retrieving large results from PostgreSQL
   - Will `SELECT content FROM result` be slow?
   - Need pagination for History view?

4. **Cleanup policy enforcement**
   - 90-day retention mentioned but...
   - No cron job documented
   - No cleanup script
   - No verification that cleanup happened

**Data Retention Policy**:
```
| Result content | 90 days | Cascade delete with Job |
```

**Missing**:
- Max result size limits (per format, aggregate)
- Compression approach (gzip in DB or separate storage?)
- Pagination strategy for history (show first 50 results? implement infinite scroll?)
- Database indexing strategy for efficient queries
- Cleanup job schedule and implementation
- Cleanup verification/monitoring
- Alert thresholds (e.g., "80% of storage used")

**Recommendation**:
Expand Section "7.6 Data Retention Policy" with:
- Max result sizes: `MARKDOWN_MAX_SIZE_MB`, `HTML_MAX_SIZE_MB`, etc.
- Cleanup job spec: "Every day at 02:00 UTC, delete jobs older than 90 days"
- Monitoring: Include result size metrics in health check
- Pagination: First 50 results in history, with "Load More" button
- Database indexes: `CREATE INDEX idx_job_createdAt ON Job(createdAt)`

**Implementation Impact**: High (affects result storage, history retrieval, cleanup job)

---

### 1.7 [CRITICAL] File Storage Lifecycle Not Fully Specified

**Section**: 7.5, 7.6  
**Severity**: Critical  
**Impact**: Disk space exhaustion, orphaned files

**Issue**:
File upload and cleanup are documented but with gaps:

**Current Spec**:
```
/data/docling-uploads/
├── 2025/
│   └── 12/
│       └── 11/
│           ├── {uuid}-document.pdf
```

**Cleanup Policy**:
```
Uploaded files | 30 days | Cron job on filesystem
```

**Missing**:
1. **File permissions**
   - Who can delete files?
   - Should uploaded files be world-readable?
   - Security implications?

2. **Cron job specification**
   - Exact cleanup time?
   - What if cleanup job fails/hangs?
   - Log output location?
   - Monitoring/alerting?

3. **Orphaned file handling**
   - If Job deleted but file not?
   - If file deleted but Job record remains?
   - Reconciliation strategy?

4. **Storage capacity**
   - No calculation of 30-day retention space
   - No alerts when disk near-full
   - What happens when `/data` is full?

5. **File access patterns**
   - How are re-downloads handled?
   - Does re-download increment file access time?
   - Should inactive files be cleaned up sooner?

**Recommendation**:
Add Section "7.5.1 File Storage Lifecycle":
- Document cleanup cron: `0 2 * * * /opt/cleanup-old-uploads.sh`
- Create cleanup script spec
- Document file permissions: `chmod 640` (read for app, not world)
- Add orphaned file reconciliation logic
- Include alerts: "Disk usage > 80%"
- Document recovery procedure: manual intervention needed

**Implementation Impact**: Medium (affects cleanup job, file permissions, monitoring)

---

### 1.8 [CRITICAL] Load Testing & Performance Baseline Not Defined

**Section**: 5.2 (Non-Functional Success Criteria), 13 (Quality Gates)  
**Severity**: Critical  
**Impact**: Unknown performance ceiling, scalability issues undetected

**Issue**:
Performance targets are specified but no load testing strategy:

**Current Targets**:
- Page load time: < 3 seconds
- Time to first progress update: < 500ms
- Accessibility score: >= 90

**Missing**:
- Load test scenario specifications
- Concurrent user capacity
- Stress test procedures
- Performance regression detection
- Baseline metrics before optimization
- How to measure "Page load time < 3s"? (First Paint? Largest Contentful Paint?)
- No browser throttling specs (3G? 4G? Fiber?)

**Recommendation**:
Add Section "13.4 Performance Testing":
```typescript
// Load Test Scenarios
1. Single user: Upload PDF → view results
   Target: <3s page load, <500ms first progress
   
2. 5 concurrent users: Each uploading different documents
   Target: All complete within timeout
   
3. Large file (95 MB PDF): Single user
   Target: SSE progress updates every <2s
   
4. Slow network (3G throttling): Single user + PDF
   Target: Page loads within 10s, doesn't timeout

// Measurement tools
- Lighthouse (accessibility + performance)
- WebPageTest or similar
- Custom timing instrumentation
- Real Browser Monitoring (synthetic)
```

**Implementation Impact**: Medium (affects testing plan, CI/CD, acceptance criteria refinement)

---

### 1.9 [CRITICAL] Monitoring, Observability, and Alerting Strategy Absent

**Section**: Missing entirely (should be new section)  
**Severity**: Critical  
**Impact**: No visibility into production issues, no proactive problem detection

**Issue**:
No monitoring or observability strategy documented:

**Missing**:
1. **Application Metrics**
   - Jobs processed per hour
   - Average processing time per document type
   - Error rate (%)
   - User sessions active
   - Disk space used

2. **Health Metrics**
   - MCP service latency
   - PostgreSQL query latency
   - Redis operation latency
   - API route latency

3. **Business Metrics**
   - Most common file type processed
   - Most common export format
   - Peak usage hours
   - User retention (session recurrence)

4. **Alerting**
   - MCP unavailable → alert immediately
   - Processing time > 5 minutes → investigate
   - Error rate > 5% → alert
   - Disk usage > 80% → alert
   - No checks: who gets alerted? How? Where?

5. **Logging**
   - What to log?
   - Log levels (debug, info, warn, error)?
   - Log rotation strategy?
   - Log aggregation/centralization?

**Recommendation**:
Create new section "13.5 Observability & Monitoring Strategy":
- Define metrics collection points
- Specify alerting thresholds and recipients
- Document logging format (JSON? structured?)
- Add dashboard mockup/spec
- Document centralized logging destination
- Define SLO targets

**Implementation Impact**: High (affects middleware, error boundaries, logging infrastructure)

---

## 2. MAJOR FINDINGS (6)

### 2.1 [MAJOR] Keyboard Shortcuts Keyboard Accessibility Not Verified

**Section**: 10.4 (Keyboard Shortcuts)  
**Severity**: Major  
**Impact**: Potential accessibility compliance failure

**Issue**:
Keyboard shortcuts are specified but implementation concerns not addressed:

**Defined Shortcuts**:
```
| Ctrl/Cmd + Enter | Process selected file/URL |
| Ctrl/Cmd + D     | Download current results  |
| Ctrl/Cmd + H     | Toggle history view       |
```

**Problems**:
1. No conflict check with browser defaults
   - `Ctrl+H` = browser history in most browsers
   - `Ctrl+S` not used but would conflict with Save
   - These should be tested/documented

2. No focus management strategy
   - After processing, should focus move?
   - Should focus trap in dialogs?
   - Tab order documented?

3. No escape key handling
   - "Escape: Close dialogs, reset form" — which takes priority if multiple are open?
   - What if processing in progress and user presses Escape?

4. No mobile/tablet consideration
   - Mobile users can't use Ctrl+Cmd
   - Should there be gesture alternatives?
   - Long-press for context menu?

**Recommendation**:
Expand Section 10.4:
- Add browser conflict analysis table
- Document focus management strategy
- Specify modal hierarchy for Escape key
- Add note about mobile gestures
- Reference WCAG 2.1 SC 2.1.1 (Keyboard requirement)

**Implementation Impact**: Low-Medium (affects keyboard handler, accessibility testing)

---

### 2.2 [MAJOR] Rate Limiting Not Enforced in Spec

**Section**: 8.1 (MCP Server Integration)  
**Severity**: Major  
**Impact**: Potential DoS, user quota management

**Issue**:
Rate limit is mentioned but very vague:

**Current Spec**:
```
Rate Limit: 10 requests/minute per client (soft limit)
```

**Problems**:
1. "Soft limit" — what does that mean?
   - Is it a guideline or enforcement?
   - What happens when exceeded?
   - Is user warned? Blocked? Throttled?

2. Per-client definition unclear
   - Per IP? Per session? Per user?
   - How to identify "client"?
   - What about load balancers/proxies?

3. Not specified in implementation
   - Where is rate limiting enforced? (Browser? API route? MCP server?)
   - How to test it?
   - How to monitor it?
   - What error code returned? (429 Too Many Requests?)

4. No burst handling
   - Can user upload 10 files then process all at once?
   - Is there a queue depth limit?
   - What about concurrent processing limits?

**Recommendation**:
Add Section "8.1.1 Rate Limiting & Quota Management":
```typescript
// Rate Limit Spec
- Per session (sessionId): 10 MCP calls/minute
- Enforcement point: /api/process route handler
- On limit exceeded: Return 429 with Retry-After header
- User notification: Toast: "Rate limit reached. Try again in XX seconds"
- Monitoring: Log rate limit hits for alerting

// Implementation:
- Use in-memory counter with timestamp sliding window
- Reset counter every minute
- Test with concurrent requests
```

**Implementation Impact**: Medium (affects /api/process middleware, rate limiting logic)

---

### 2.3 [MAJOR] Export Format Requirements Incomplete

**Section**: 4.1.1, 8.2  
**Severity**: Major  
**Impact**: Incomplete feature, unclear requirements

**Issue**:
Export formats are listed but requirements not fully specified:

**Current Spec**:
```
| Tool | File Types | Category |
|------|------------|----------|
| export_markdown | - | Export |
| export_html     | - | Export |
| export_json     | - | Export |
```

**Missing**:
1. **Markdown export**
   - Embedded images or links?
   - Table format (GFM/MultiMarkdown)?
   - Frontmatter metadata?
   - Code block syntax highlighting hints?

2. **HTML export**
   - Standalone (with CSS) or fragment?
   - Responsive/mobile-optimized CSS?
   - Dark mode support built-in?
   - JavaScript interactive elements?

3. **JSON export**
   - Flat structure or nested?
   - What schema? (Custom or docling-standard?)
   - For what use cases? (ML training? Indexing? Display?)
   - Version/compatibility info in output?

4. **RAW export**
   - Mentioned in section 10.2 "Tabs: Markdown, HTML, JSON, Raw"
   - What is "raw"? Raw DoclingDocument? Raw MCP response?
   - How is it displayed? Pretty-printed JSON?

5. **Download file naming**
   - What filename convention?
   - Include timestamp? Processing parameters?
   - Example: `document_2025-12-11_markdown.md`?

**Recommendation**:
Expand Section "4.1.1 Supported Export Formats":
```markdown
## Export Format Specifications

### Markdown Export
- Format: GitHub Flavored Markdown (GFM)
- Images: Data URIs embedded (base64)
- Tables: GFM table syntax
- Code: Triple-backtick with language hint
- Metadata: YAML frontmatter with source URL and timestamp

### HTML Export
- Standalone: Yes, includes Tailwind CSS inline
- Responsive: Mobile-first viewport
- Dark mode: Respects prefers-color-scheme media query
- Interactivity: Static only (no JavaScript)

### JSON Export
- Schema: Docling standard (see appendix)
- Nesting: Full AST hierarchy preserved
- Metadata: Includes processing timestamp, source file info
- Pretty-print: Yes, 2-space indentation

### Download Naming
- Pattern: {original_filename}_{YYYYMMDD_HHMMSS}_{format}.{ext}
- Example: report_20251211_143022_markdown.md
```

**Implementation Impact**: Medium (affects export logic, download handler)

---

### 2.4 [MAJOR] Component Testing Strategy Not Specified

**Section**: 13.2 (Test Gates)  
**Severity**: Major  
**Impact**: Unclear testing approach, QA gaps

**Issue**:
Test gates mention component tests but no specifics:

**Current Spec**:
```
| Component Render | All components | React Testing Library |
```

**Missing**:
1. **Component test coverage**
   - Should all 15+ components have tests?
   - Which ones are critical?
   - What's the minimum coverage threshold?

2. **Interaction testing**
   - File upload → mock? Real file?
   - Progress updates → SSE simulation?
   - Button clicks → event assertions?

3. **Accessibility testing**
   - How to test WCAG 2.1 AA compliance?
   - Automated tools (axe-core)? Manual testing?
   - Who verifies accessibility?

4. **E2E test definition**
   - "Critical Path: Upload → Process → View → History"
   - But how to test MCP integration in E2E?
   - Use real MCP server or mock?
   - Performance requirements for E2E tests?

**Recommendation**:
Expand Section "13.2 Test Strategy":
```typescript
// Component Testing
- All 15 components require unit tests
- Minimum coverage: 80% lines, 75% branches
- Use React Testing Library (not Enzyme)
- Test user interactions, not implementation details

// Critical Components (100% coverage)
- UploadZone: File selection, validation, error states
- ProgressCard: SSE progress updates, state transitions
- ResultsViewer: Tab switching, download functionality
- HistoryView: Pagination, job filtering

// E2E Testing (Playwright)
Test Suite: test/e2e/critical-path.spec.ts
- Scenario: User uploads PDF → views markdown → downloads
- Precondition: Real MCP server running
- Assertion: File system contains downloaded .md
- Timeout: 60s per test

// Accessibility Testing
- axe-core in unit tests
- Manual WCAG 2.1 AA checklist
- Keyboard navigation testing
- Screen reader verification (NVDA/JAWS)
```

**Implementation Impact**: Medium (affects test architecture, CI/CD)

---

### 2.5 [MAJOR] URL Input Validation Spec Too Vague

**Section**: 4.1.1, 11.2  
**Severity**: Major  
**Impact**: Unclear URL handling, potential security issues

**Issue**:
URL validation mentioned but requirements not specified:

**Current Spec**:
```
URL processing | ✅ convert_url | Paste URL, one-click process | UI required |
```

**Missing**:
1. **URL validation rules**
   - HTTP/HTTPS only or support others? (ftp://, file://?)
   - Localhost URLs allowed? (http://localhost:8080)
   - Private IP ranges allowed? (192.168.x.x, 10.0.0.0/8)
   - URL length limit?
   - Special characters (Unicode domains)?

2. **URL preview**
   - UrlInput component shows "metadata preview" but not specified
   - What metadata? (Title from Open Graph? First 100 chars?)
   - Timeout for preview fetch?

3. **Security considerations**
   - SSRF prevention? Don't allow internal IPs?
   - Timeout to prevent hanging? (5s? 30s?)
   - Redirect handling? (Follow or block?)
   - robots.txt respect?
   - User-Agent header value?

4. **Error handling**
   - E102: URL_UNREACHABLE — how detected? Timeout? 404? 403?
   - E103: URL_TIMEOUT — timeout threshold?

**Recommendation**:
Add Section "11.3 URL Input Security":
```typescript
const urlSchema = z.string()
  .url()
  .refine(url => {
    const parsed = new URL(url);
    // Only http/https
    if (!['http:', 'https:'].includes(parsed.protocol)) return false;
    // Block private IPs (SSRF prevention)
    const hostname = parsed.hostname;
    if (hostname === 'localhost' || hostname === '127.0.0.1') return false;
    if (/^(10\.|172\.1[6-9]\.|172\.2[0-9]\.|172\.3[01]\.|192\.168\.)/.test(hostname)) return false;
    return true;
  }, { message: 'Invalid or blocked URL' })
  .max(2048);

// Metadata preview
- Fetch with timeout: 5s
- Follow up to 3 redirects
- User-Agent: "hx-docling-ui/1.0"
- Extract: <title>, og:title, first 150 chars
```

**Implementation Impact**: Medium (affects URL validation, error handling, security)

---

### 2.6 [MAJOR] Plugin Architecture Not Implemented

**Section**: 9.1, 9.2 (Extensibility Framework)  
**Severity**: Major  
**Impact**: Extensibility goals not achievable without implementation

**Issue**:
Plugin architecture is designed but no implementation path:

**Current Spec**:
```typescript
interface DoclingPlugin {
  id: string;
  name: string;
  version: string;
  
  onInit?: () => Promise<void>;
  onDestroy?: () => Promise<void>;
  
  beforeProcess?: (input: ProcessInput) => ProcessInput;
  afterProcess?: (result: ProcessResult) => ProcessResult;
  
  resultRenderers?: ResultRenderer[];
  toolbarActions?: ToolbarAction[];
}
```

**Missing**:
1. **Plugin loading mechanism**
   - Where are plugins stored/loaded from?
   - How to register? (env var? config file? API?)
   - Plugin naming convention?

2. **Plugin isolation**
   - Plugins can modify input/result — what if malicious?
   - Should plugins run in sandbox/iframe?
   - Version compatibility check?

3. **Result renderers**
   - What is `ResultRenderer` interface?
   - How to register custom tab?
   - How to override default renderers?

4. **Toolbar actions**
   - Where do toolbar actions appear?
   - Icon/label specification?
   - Keyboard shortcut conflict handling?

5. **Event system**
   - `DoclingEvent` type is defined but...
   - How to subscribe to events?
   - Event ordering (if multiple plugins)?
   - Async event handling?

6. **Plugin discovery & testing**
   - Example plugin to demonstrate API?
   - How to develop/test plugins locally?
   - Plugin validation schema?

**Recommendation**:
Defer plugin architecture to Phase 2 charter with note in Section 3.3:
```
| Plugin Architecture | Defer to Phase 2 | Requires community feedback, 
                                           version strategy, and plugin 
                                           ecosystem planning
```

Or implement minimal MVP in Phase 1:
- Simple result renderer registration
- No security sandbox required (internal use only)
- Document in `/docs/plugin-api.md`
- Provide example plugin in `/plugins/example-renderer/`

**Implementation Impact**: High (requires design review, affects component architecture)

---

## 3. MINOR FINDINGS (8)

### 3.1 [MINOR] Prisma Client Singleton Pattern Not Documented

**Section**: 6.4 (Directory Structure)  
**Severity**: Minor  
**Impact**: Runtime errors, multiple Prisma instances

**Location**: `src/lib/db/prisma.ts`

**Issue**:
File is listed but its implementation is not documented. Prisma client should be a singleton to prevent multiple instances in development.

**Recommendation**:
Add code example to Section 12.3 (Setup) or create `docs/database-setup.md`:
```typescript
// src/lib/db/prisma.ts
import { PrismaClient } from '@prisma/client';

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient };

export const prisma =
  globalForPrisma.prisma ||
  new PrismaClient({
    log: process.env.NODE_ENV === 'development' ? ['query', 'error'] : ['error'],
  });

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma;
```

**Implementation Impact**: Low (best practice documentation)

---

### 3.2 [MINOR] No Guidance on Input Component State Synchronization

**Section**: 6.3 (Component Architecture)  
**Severity**: Minor  
**Impact**: State management complexity, potential bugs

**Issue**:
Multiple input components (UploadZone, UrlInput) share state via Zustand but synchronization strategy not specified:

**Components**:
- UploadZone: Sets file state
- UrlInput: Sets URL state
- FilePreview: Shows selected file
- ProgressCard: Triggered by button on either component

**Questions**:
1. What if user drags file then pastes URL? (Both can't be set)
2. Can user change input mid-processing?
3. Should UploadZone clear when user pastes URL?

**Recommendation**:
Add subsection "6.3.1 Input State Management":
```typescript
// Only one input can be "active" at a time
// Zustand store state:
interface DocumentState {
  file: File | null;
  url: string | null;
  activeInput: 'file' | 'url' | null; // mutex lock
  
  setFile: (file: File | null) => void;  // clears URL
  setUrl: (url: string | null) => void;  // clears file
  clearInputs: () => void;
  isProcessing: boolean;
}

// If user drags file while URL is set → clear URL, set file
// If URL input focused → disable file upload button
// During processing → disable all inputs
```

**Implementation Impact**: Low (design clarity)

---

### 3.3 [MINOR] No Documentation of Fiber vs Non-Fiber Differences

**Section**: 10.3 (Browser Support)  
**Severity**: Minor  
**Impact**: Unclear performance expectations

**Issue**:
Browser support table shows all modern browsers but doesn't note that fiber connections may work significantly faster than 3G.

**Recommendation**:
Add note in Section 10.3:
```
Performance Note: Targets assume broadband connection (>10 Mbps). 
On 3G connections (< 3 Mbps), page load time may exceed 10s. 
Test with Lighthouse throttling to verify mobile experience.
```

**Implementation Impact**: Low (documentation)

---

### 3.4 [MINOR] No Guidance on Result Viewer Tab Navigation

**Section**: 6.3, 10.2  
**Severity**: Minor  
**Impact**: UX inconsistency, accessibility gaps

**Issue**:
ResultsViewer is mentioned as "Tabs: Markdown, HTML, JSON, Raw" but behavior unclear:

**Questions**:
1. If user switches to HTML tab, does Markdown tab reload when switching back? (State preservation)
2. Are all tabs pre-rendered or lazy-loaded?
3. Which tab selected by default? (Markdown?)
4. Can user customize tab order or hide tabs?
5. Tab focus management with keyboard nav?

**Recommendation**:
Add subsection "6.3.2 Results Viewer Behavior":
```typescript
// Tab persistence:
- All tabs rendered upfront (no lazy loading)
- Tab state saved in Zustand store
- User's last selected tab remembered within session

// Default tab: Markdown
// Tab order (not reorderable): Markdown → HTML → JSON → Raw

// Accessibility:
- Tabs use aria-selected, aria-controls
- Content divs use role="tabpanel"
- Keyboard: Arrow keys to switch tabs
- Tab focus: Stays on active tab content
```

**Implementation Impact**: Low-Medium (affects ResultsViewer component)

---

### 3.5 [MINOR] Error Catalog Incomplete

**Section**: 20 Appendices (Appendix A: Error Catalog)  
**Severity**: Minor  
**Impact**: Incomplete error handling specification

**Issue**:
Error catalog is provided but incomplete:

**Defined**:
```
ERROR_CODES:
E0xx: File errors (6 codes)
E1xx: URL errors (3 codes)
E2xx: MCP errors (4 codes)
E3xx: Processing errors (3 codes)
E4xx: Database errors (3 codes)
E5xx: Session errors (2 codes)
E9xx: Network errors (3 codes)
Total: 24 codes (but only 24 shown, some overlap with risks)
```

**Missing**:
1. **User-facing error messages**
   - E001 (FILE_TOO_LARGE) → User message: "File must be under 100 MB"
   - Who writes these? Designer? Developer?
   - Localization consideration?

2. **Error recovery actions**
   - E001: Suggest "Try with smaller file"
   - E102: Suggest "Check URL is accessible"
   - E201 (MCP_UNAVAILABLE): Suggest "Try again in 5 minutes"

3. **Error logging**
   - What context to log? (Full error stack? User action path?)
   - Log level (warn, error, critical)?

4. **Error analytics**
   - Should errors be tracked? (Which ones?)
   - Metrics for error rates by type?

**Recommendation**:
Expand Appendix A with error recovery mapping:
```typescript
export const ErrorRecoveryMap: Record<ErrorCode, ErrorRecovery> = {
  [ErrorCode.FILE_TOO_LARGE]: {
    userMessage: "File must be under 100 MB",
    action: "retry",
    suggestedAction: "Choose a smaller file or split the document"
  },
  [ErrorCode.MCP_UNAVAILABLE]: {
    userMessage: "Service temporarily unavailable",
    action: "retry-after",
    retryAfter: 30000, // 30s
    suggestedAction: "Try again in 30 seconds"
  },
  // ... etc
};
```

**Implementation Impact**: Low (documentation, error UI component)

---

### 3.6 [MINOR] .gitignore Incomplete

**Section**: 20 Appendices (Appendix C)  
**Severity**: Minor  
**Impact**: Potential secrets leak, build artifacts in repo

**Issue**:
`.gitignore` is provided but missing common Next.js/Node entries:

**Current**:
```
node_modules/
.next/
coverage/
.env
```

**Missing**:
```
# Build outputs
dist/
out/

# IDE
.idea/
.vscode/

# OS
.DS_Store
Thumbs.db

# Logs
*.log
npm-debug.log*
yarn-debug.log*

# Prisma
prisma/.env*

# Environment
.env.local
.env.*.local

# Runtime data
.cache/
```

**Recommendation**:
Update Appendix C with comprehensive `.gitignore`:
```gitignore
# Dependencies
node_modules/
/.pnp
.pnp.js

# Next.js build
/.next/
/out/
/dist/

# Production
/build/

# Prisma
/node_modules/.prisma/client

# IDE
.idea/
.vscode/
*.swp
*.swo
*~

# OS
.DS_Store
Thumbs.db

# Logs
*.log
npm-debug.log*
yarn-debug.log*

# Environment
.env
.env.local
.env.*.local
.env.production.local

# Cache
.cache/
.turbo/

# Test
coverage/
.nyc_output/
```

**Implementation Impact**: Low (configuration)

---

### 3.7 [MINOR] No Mention of Type Safety Strategy (Prisma vs Database)

**Section**: 6.1 (Technology Stack)  
**Severity**: Minor  
**Impact**: Potential type safety gaps

**Issue**:
Prisma is listed but generated types strategy not mentioned:

**Current**:
```
Database ORM | Prisma | 5.x | Type-safe PostgreSQL access
```

**Missing**:
1. **Generated code handling**
   - Where is `node_modules/.prisma/client` imported from?
   - Should generated code be committed or gitignored?
   - CI step to regenerate?

2. **Zod + Prisma coordination**
   - Prisma generates types
   - Zod validates at runtime
   - Any redundancy? Conflicts?

3. **Database type divergence**
   - What if database schema changes without Prisma migration?
   - How to prevent mismatches?

**Recommendation**:
Add note in Section 12.3 (Setup):
```bash
# After schema changes:
1. Update src/prisma/schema.prisma
2. Run: npx prisma migrate dev --name {description}
3. Prisma auto-generates types in node_modules/.prisma/client/
4. Import types: import { Job, Result } from '@prisma/client'
5. Commit only migration file, NOT generated types
```

**Implementation Impact**: Low (documentation, best practice)

---

### 3.8 [MINOR] No Testing Strategy for Redux/Zustand Store

**Section**: 13.2 (Test Gates)  
**Severity**: Minor  
**Impact**: State management bugs undetected

**Issue**:
Zustand store is used but testing strategy not specified.

**Current**:
```
| API Route Tests | All routes | Vitest |
```

**Missing**:
- How to test Zustand stores?
- Mock store in component tests?
- Store unit tests separately?
- State mutation testing?

**Recommendation**:
Add to Section "13.2 Test Gates":
```
| Store Tests (Zustand) | All state mutations | Vitest |
| Store Mock (Components) | All components using store | Mock + React Testing Library |

Test Examples:
- documentStore.setFile() → verify state updated
- ProgressCard + documentStore → dispatch action → verify UI updated
- SSE update → verify store updates → verify UI reflects change
```

**Implementation Impact**: Low (testing guidance)

---

## 4. RECOMMENDATIONS SUMMARY

### Quick Wins (Implement Before Development)

1. ✅ **Create `/docs/database-strategy.md`** (Section 1.4)
   - Migration procedures
   - Rollback strategy
   - Seed script
   - Test database reset

2. ✅ **Create `/docs/sse-resilience.md`** (Section 1.1)
   - Reconnection backoff
   - Fallback strategies
   - State reconciliation

3. ✅ **Create `/api/health` endpoint spec** (Section 1.5)
   - Response format
   - Timeout values
   - Caching strategy

4. ✅ **Update `.gitignore`** (Section 3.6)
   - Use comprehensive template
   - Add Prisma entries

### Medium Priority (Before Integration Checkpoint)

5. ⚠️ **Expand Section 7** (File Storage Lifecycle) (Section 1.7)
   - Cleanup cron job spec
   - Orphaned file handling
   - Permissions strategy

6. ⚠️ **Add Section 8.6** (MCP Error Recovery) (Section 1.3)
   - State machine for retries
   - Partial result handling
   - UI error flows

7. ⚠️ **Add Section 7.6** (Result Persistence for Large Documents) (Section 1.6)
   - Max size limits
   - Compression strategy
   - Pagination spec

8. ⚠️ **Expand Section 11** (Session Management Edge Cases) (Section 1.2)
   - Edge case matrix
   - Multi-tab conflict resolution
   - Session expiry cascade

### Should-Haves (Before Phase 1 Completion)

9. ℹ️ **Add Section 13.5** (Monitoring & Observability) (Section 1.9)
   - Metrics to collect
   - Alerting rules
   - Dashboard spec

10. ℹ️ **Create `/docs/plugin-api.md`** (Section 2.6)
    - Plugin interface spec
    - Example plugin
    - Registration mechanism

11. ℹ️ **Add Performance Testing Strategy** (Section 1.8)
    - Load test scenarios
    - Performance regression detection
    - Baseline metrics

### Nice-to-Haves (Post-Phase 1)

12. 💡 **Create `/docs/component-testing-guide.md`** (Section 2.4)
    - React Testing Library patterns
    - E2E testing with Playwright
    - Accessibility testing checklist

13. 💡 **Expand Export Format Specs** (Section 2.3)
    - Markdown, HTML, JSON details
    - Download naming convention
    - Schema documentation

14. 💡 **Create `/docs/keyboard-shortcuts.md`** (Section 2.1)
    - Browser conflict matrix
    - Focus management strategy
    - Mobile gesture alternatives

---

## 5. CHARTER STRENGTHS

**What This Charter Does Well:**

1. ✅ **Comprehensive Scope Definition**
   - Clear Phase 1 vs Phase 2 separation
   - Explicit constraints and assumptions
   - In-scope vs out-of-scope matrix

2. ✅ **Excellent Architecture Diagrams**
   - High-level system architecture (Section 6.2)
   - Component hierarchy (Section 6.3)
   - Data flow sequence diagram (Section 6.7)

3. ✅ **Strong Technical Specifications**
   - Complete technology stack
   - Detailed directory structure
   - Prisma schema included
   - Infrastructure dependency table

4. ✅ **Risk Management**
   - 6 identified risks with mitigations
   - Assumption/constraint matrix
   - Dependency tracking

5. ✅ **Clear Success Criteria**
   - 16 acceptance criteria
   - Functional + non-functional targets
   - Extensibility requirements

6. ✅ **Realistic Implementation Plan**
   - 9-10 sprint breakdown
   - Estimated session counts
   - Clear deliverables per sprint
   - Integration checkpoint defined

---

## 6. CRITICAL PATH FOR IMPLEMENTATION

Based on findings, **recommended implementation order**:

```
Week 1 (Sessions 1-3): Foundation
├── 1.1: Scaffold + Prisma setup
├── 1.2: Database migration strategy (CRITICAL)
├── 1.3: Health check endpoint (CRITICAL)
└── 1.2.1: Redis connection + session management

Week 1-2 (Sessions 4-5): Input
├── 1.3: Upload component
├── Session 1.4: URL input component
└── 1.3.1: File/URL validation schemas

Week 2 (Session 6): CRITICAL INTEGRATION CHECKPOINT
├── Verify MCP connectivity
├── Verify PostgreSQL persistence
├── Verify Redis session tracking
├── Create error recovery strategy (CRITICAL)
└── Implement SSE reconnection logic (CRITICAL)

Week 2-3 (Sessions 7-8): Results + History
├── 1.6: Results viewer with tab handling
├── 1.7: History view + pagination
└── 1.8.1: Large result handling + cleanup (CRITICAL)

Week 3 (Sessions 9-10): Polish
├── Testing strategy execution
├── Monitoring/observability setup (CRITICAL)
├── Performance baseline testing (CRITICAL)
└── Documentation + README

CRITICAL PREREQUISITES for development:
□ SSE reconnection strategy documented
□ MCP error recovery strategy documented
□ Database migration strategy defined
□ Health check endpoint spec finalized
□ Session edge cases resolved
□ Result persistence strategy for large files finalized
```

---

## 7. QUESTIONS FOR STAKEHOLDERS

Before development begins, clarify:

1. **SSE Reconnection**: After network drop, can user resume processing?
   - Or must they restart upload?

2. **Session Cookies**: Over HTTP in dev environment?
   - Secure flag required?

3. **File Retention**: Is 30 days for files too short?
   - Can users request longer retention?

4. **Rate Limiting**: Is "soft limit" enforced or advisory?
   - What's the recovery UX?

5. **MCP Timeouts**: Is 300s reasonable for ALL documents?
   - Or should large files (>50 MB) have longer timeout?

6. **Monitoring**: Should metrics be collected in Phase 1?
   - Or defer observability to Phase 2?

7. **Error Analytics**: Should error events be tracked?
   - For debugging or business intelligence?

8. **Mobile UX**: Is touch-optimized upload required in Phase 1?
   - Or acceptable to defer to Phase 2?

---

## CONCLUSION

**Overall Assessment**: ⭐⭐⭐⭐ (4/5 - Excellent)

The charter is **production-ready** as a strategic document with **9 critical tactical gaps** that require detailed design decisions before development. The gaps are not showstoppers but require explicit specification to prevent mid-implementation rework.

**Recommendation**: 
1. Create decision documents for the 9 critical findings (1.1-1.9)
2. Use this review as input to sprint planning
3. Address "Quick Wins" before starting development sessions
4. Handle "Medium Priority" items at integration checkpoint (after session 6)
5. Reserve "Should-Haves" for sessions 8-10

**Estimated Effort to Close Gaps**: 3-5 hours of design documentation before code begins.

---

**Review Completed**: December 11, 2025  
**Reviewer**: Code Analysis Agent  
**Next Step**: Share findings with development team for clarification
