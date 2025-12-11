# Charter Review: Stakeholder Checklist & Decision Matrix

**Charter**: `0.1-hx-docling-ui-charter.md` (v0.6.0 - ✅ APPROVED)  
**Review Date**: December 11, 2025  
**Purpose**: Practical checklist for clearing blockers before development

---

## Critical Decisions Required (Before Development)

### Decision 1: SSE Reconnection Strategy

**Question**: What should happen if user's network drops during processing?

| Option | Pros | Cons | Recommendation |
|--------|------|------|----------------|
| **A) Resume** | User doesn't lose progress; good UX | Complex state tracking; risk of duplicate processing | ⭐ **RECOMMENDED** for Phase 1 |
| **B) Restart** | Simple to implement; no state conflicts | Poor UX; users must re-upload | ❌ Not ideal |
| **C) Retry Auto** | Transparent to user | May mask real issues; unbounded retry loop | ⚠️ Dangerous |

**Decision Needed**: Choose Option A, B, or C

**If Chosen (A)**:
- [ ] Design state reconciliation logic
- [ ] Define max retry attempts (suggest: 5)
- [ ] Define backoff strategy (suggest: 1s, 2s, 4s, 8s, 16s)
- [ ] Specify UI feedback during reconnect
- [ ] Document in `/docs/sse-resilience.md`
- [ ] Add useProcess hook reconnect logic

**Estimated Impact**: 4-6 hours development time

---

### Decision 2: Session Management on Browser Close

**Question**: What happens to user's processing history if they close the browser?

| Scenario | Current Spec | Expected Behavior | Decision Needed |
|----------|--------------|-------------------|-----------------|
| User has sessionId cookie | 24-hour TTL in Redis | Can they view history? | YES/NO |
| User closes browser, returns next day | Cookie persists? | Can they resume session? | YES/NO |
| User clears cookies | Session lost | Lose all history? | YES/NO |
| Multiple browser tabs | Same sessionId | Can both process simultaneously? | YES/NO |

**Decision Needed**: Fill in "Decision Needed" column

**Implications**:
- If YES to cookie persistence: Longer than 24h? (30 days?)
- If YES to simultaneous processing: Need queueing logic
- If NO to cookie persistence: Need device fingerprinting or authentication

**Estimated Impact**: 3-5 hours development time

---

### Decision 3: MCP Failure Recovery UX

**Question**: When MCP server has an error, what's the user experience?

| Failure Type | Current Spec | User Sees | Recoverable? |
|--------------|--------------|-----------|--------------|
| PDF → Markdown export fails | Retry 3x with backoff | Error message | Need to specify |
| All 3 retries fail | Job → ERROR state | ? | User action needed |
| HTML export fails but Markdown succeeds | ? | Show partial results? | Defer or implement? |
| Timeout on 95 MB PDF | 300s timeout exceeded | "Processing timed out" | User can re-try? |

**Decision Needed**: Fill in "User Sees" and "Recoverable?" columns

**Choose**:
- Fail-safe: Show any successful results, warn about missing formats
- Fail-hard: All-or-nothing (all formats must succeed)

**Recommendation**: Fail-safe is more user-friendly

**Estimated Impact**: 2-3 hours development time

---

### Decision 4: Result Persistence for Large Files

**Question**: How to handle very large documents (95 MB PDF)?

| Concern | Current Spec | Issue | Decision |
|---------|--------------|-------|----------|
| Result size in DB | Up to 100 MB file | Markdown/HTML could be 200 MB+ | Compress? Truncate? |
| Query performance | No pagination mentioned | History page loads slow | Add pagination? |
| Disk space | 90-day retention | On high usage: could use TBs | Alert threshold? |
| Download speed | No specification | User downloads 200 MB HTML | Streaming? Compression? |

**Decision Needed**: For each row, specify approach

**Options**:
- Compression: Store Markdown/HTML as gzip in DB
- Pagination: Show first 50 results, "Load More" button
- Size limit: Truncate exports at 50 MB
- Streaming: Don't load full result into memory, stream from DB

**Recommendation**: All of the above

**Estimated Impact**: 4-6 hours development time

---

### Decision 5: Health Check Frequency

**Question**: How often should health checks run?

| Endpoint | Current Spec | Recommendation | Decision |
|----------|--------------|-----------------|----------|
| MCP Server | "On page load" | Also every 30s? | YES/NO |
| PostgreSQL | "On startup" | Also periodically? | YES/NO |
| Redis | "On startup" | Also periodically? | YES/NO |
| Response time | Not specified | Cache 30s to avoid hammering? | YES/NO |

**Decision Needed**: Choose caching strategy

**Implications**:
- If cached 30s: Stale health info possible
- If not cached: More load on dependencies
- If background polling: Can show degraded mode UI

**Recommendation**:
- MCP: 30s cache, background polling
- PostgreSQL: 30s cache, background check
- Redis: 30s cache, background ping

**Estimated Impact**: 2-3 hours development time

---

### Decision 6: Rate Limiting Enforcement

**Question**: How strictly to enforce rate limit (10 req/min)?

| Question | Options | Decision |
|----------|---------|----------|
| Per-client scope | Per IP / Per session / Per auth user | Per session (no auth yet) |
| Enforcement location | Browser / API middleware / MCP server | API middleware |
| On limit exceeded | 429 error / Queue request / Deny silently | Return 429 with Retry-After |
| Burst handling | Allow 10 rapid then wait / Smooth throughout minute | Smooth throughout minute |

**Decision Needed**: Confirm choices above

**Estimated Impact**: 1-2 hours development time

---

### Decision 7: Plugin Architecture Scope

**Question**: Should extensibility plugins be in Phase 1?

| Aspect | Phase 1 | Phase 2 | Decision |
|--------|---------|---------|----------|
| Plugin loading | Built-in only | Support external plugins | Recommend: Phase 2 |
| Result renderers | Custom (code-level) | Plugin-based | Recommend: Phase 1 (basic) |
| Toolbar actions | None | Plugin-based | Recommend: Phase 2 |
| Event system | Defined (Section 9.2) | Fully implemented | Recommend: Phase 2 |

**Decision Needed**: Confirm plugin scope

**Recommendation**: Keep Phase 1 minimal:
- [ ] Define plugin interface (TypeScript types)
- [ ] Document in README
- [ ] Don't implement registration mechanism yet
- [ ] Defer to Phase 2

**Estimated Impact**: 0 hours for Phase 1 (defer all implementation)

---

### Decision 8: Keyboard Shortcuts vs Browser Conflicts

**Question**: Are the defined shortcuts causing browser conflicts?

| Shortcut | Browser Default | Conflict? | Alternative |
|----------|-----------------|-----------|-------------|
| Ctrl+Enter | None | ❌ Safe | OK to use |
| Ctrl+D | Browser favorites | ⚠️ Conflict | Use Cmd+D only? |
| Ctrl+H | Browser history | ⚠️ Conflict | Use Ctrl+Shift+H? |
| Ctrl+L | Browser address bar | ⚠️ Conflict | Use Ctrl+Shift+L? |
| Ctrl+U | View page source | ⚠️ Conflict | Use Ctrl+Shift+U? |

**Decision Needed**: Accept conflicts or reassign shortcuts?

**Recommendation**: Use modifier combinations:
- Ctrl+Enter: ✅ Keep (safe)
- Ctrl+Shift+D: Download (avoid Ctrl+D)
- Ctrl+Shift+H: History (avoid Ctrl+H)
- Ctrl+Shift+L: URL focus (avoid Ctrl+L)
- Ctrl+Shift+U: Upload (avoid Ctrl+U)
- Escape: ✅ Keep (close dialogs)

**Estimated Impact**: 0.5 hours development time

---

## Implementation Blockers Matrix

| Blocker | Severity | Dependency | Owner | Target Date | Status |
|---------|----------|------------|-------|-------------|--------|
| SSE reconnection strategy | CRITICAL | Core loop | Dev Lead | Before Sprint 1.5 | 🔴 Open |
| Session edge cases | CRITICAL | Auth/persistence | Dev Lead | Before Sprint 1.2 | 🔴 Open |
| MCP error recovery | CRITICAL | Core loop | Dev Lead | Before Sprint 1.5 | 🔴 Open |
| Result persistence strategy | CRITICAL | Storage | Dev Lead | Before Sprint 1.3 | 🔴 Open |
| Health check endpoint spec | CRITICAL | Bootstrap | Dev Lead | Before Sprint 1.1 | 🔴 Open |
| Rate limit enforcement | MAJOR | Compliance | Dev Lead | Before Sprint 1.5 | 🟡 Design |
| Keyboard shortcuts | MAJOR | UX | Designer | Before Sprint 1.8 | 🟡 Design |
| Plugin scope definition | MAJOR | Architecture | Dev Lead | Before Sprint 1.1 | 🟡 Design |

---

## Pre-Development Checklist

**Complete before starting Sprint 1.1 (Project Scaffold):**

### Configuration & Setup (30 minutes)
- [ ] Confirm Node.js 20.9+ installed on hx-cc-server
- [ ] Confirm npm 10.x installed
- [ ] Verify network access to hx-postgres-server:5432
- [ ] Verify network access to hx-redis-server:6379
- [ ] Verify network access to hx-docling-mcp-server:8000/health
- [ ] Create project directory: `/home/agent0/hx-docling-ui/`
- [ ] Create persistent storage: `/data/docling-uploads/`

### Documentation (2 hours)
- [ ] Create `docs/database-strategy.md` (migration procedure, rollback, seed)
- [ ] Create `docs/sse-resilience.md` (reconnection strategy)
- [ ] Create `docs/session-management.md` (edge case handling)
- [ ] Create `docs/error-recovery.md` (MCP failure flows)
- [ ] Create `docs/api-health-check.md` (endpoint specification)
- [ ] Update charter Section 8.6 (MCP error recovery)
- [ ] Update charter Section 7.4 (session edge cases)
- [ ] Update charter Section 7.6 (result persistence for large files)

### Design Decisions (2 hours)
- [ ] **Decision 1**: SSE reconnection strategy (Option A/B/C)
- [ ] **Decision 2**: Session persistence across browser closes (YES/NO)
- [ ] **Decision 3**: MCP failure UX (fail-safe vs fail-hard)
- [ ] **Decision 4**: Result persistence approach (compression/pagination/truncation)
- [ ] **Decision 5**: Health check caching strategy (30s cache? Polling?)
- [ ] **Decision 6**: Rate limit enforcement (per session, API middleware, 429 response)
- [ ] **Decision 7**: Plugin architecture scope (defer to Phase 2, confirm)
- [ ] **Decision 8**: Keyboard shortcuts (reassign conflicts, confirm)

### Infrastructure Verification (1 hour)
- [ ] [ ] Test PostgreSQL connection: `psql -h hx-postgres-server.hx.dev.local -U docling_user`
- [ ] Test Redis connection: `redis-cli -h hx-redis-server.hx.dev.local ping`
- [ ] Test MCP health: `curl http://hx-docling-mcp-server.hx.dev.local:8000/health`
- [ ] Verify `/data/docling-uploads/` permissions (app can write)
- [ ] Verify `/tmp/docling-processing/` exists and writable
- [ ] Record IPs and ports in setup document

### Stakeholder Sign-Off (1 hour)
- [x] Confirm charter v0.6.0 is APPROVED
- [ ] Confirm all 8 decisions made
- [ ] Confirm all blockers resolved or accepted
- [ ] Confirm development can start

**Total Pre-Dev Time**: ~6 hours (1-2 days)

---

## Risk Acceptance Template

For each remaining ambiguity, complete this:

**Risk**: [Brief description]  
**Impact if not addressed**: [What goes wrong in development]  
**Mitigation**: [How to handle uncertainty]  
**Owner**: [Who decides]  
**Target Resolution**: [By when]  

### Example Filled:

**Risk**: SSE reconnection strategy undefined  
**Impact if not addressed**: Developer guesses at retry logic; could implement wrong backoff, causing usability issues  
**Mitigation**: Dev Lead creates `docs/sse-resilience.md` with explicit choices before Sprint 1.5  
**Owner**: Dev Lead  
**Target Resolution**: Before starting Sprint 1.5 (Integration Checkpoint)

---

## Sign-Off

| Role | Name | Date | Decision |
|------|------|------|----------|
| CAIO/Dev Lead | _______________ | _________ | Approved |
| Technical Lead | _______________ | _________ | Approved |
| Stakeholder | _______________ | _________ | Approved |

---

## Next Steps

1. **This week**: Complete pre-development checklist (6 hours)
2. **Before Sprint 1.1**: All 8 critical decisions made and documented
3. **Sprint 1.1**: Project scaffold (can proceed with clear requirements)
4. **Sprint 1.5**: Integration checkpoint (verify all dependencies work)
5. **Sprint 1.8**: Monitoring & observability added (complete observability)

---

**Review Generated**: December 11, 2025  
**Reviewed By**: Code Analysis Agent  
**Use in Kickoff Meeting**: YES - Share this checklist in development kickoff
