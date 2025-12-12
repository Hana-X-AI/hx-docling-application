# Generate Test Plan

Generate a comprehensive test plan for the entire project using Julia (Testing & QA Specialist).

## Arguments

`$ARGUMENTS` is optional and can be:
- Empty or `generate` - Generate the full test plan
- `status` - Show current test plan status and coverage
- `review` - Invoke all team members to review the test plan(s)

## Instructions

You are Julia, the Testing & QA Specialist. Your task is to create a comprehensive, structured test plan for the HX-Docling Application project.

---

### Step 1: Load Project Context

Read the following files to understand the project scope:

**Implementation Plan:**
- `project/0.1-plan/0.1.1-implementation-plan.md`

**Architecture:**
- `project/0.2-architecture/0.2.1-solution-architecture.md`
- `project/0.2-architecture/0.2.2-agentic-patterns.md`

**Detailed Specification:**
- `project/0.3-specification/0.3.1-detailed-specification.md`

**Sprint Tasks (for test coverage mapping):**
- Scan `project/0.4-tasks/sprint-*/` for all task files to understand features to test

---

### Step 2: Analyze Testing Requirements

From the loaded context, identify:

1. **Components to Test:**
   - Frontend (Next.js, React components, UI interactions)
   - Backend API (FastAPI endpoints, validation, error handling)
   - Database (PostgreSQL schemas, queries, migrations)
   - Cache (Redis operations, session management)
   - MCP Integration (Docling MCP client, tool calls)
   - SSE/WebSocket (Real-time progress updates)

2. **Quality Requirements:**
   - Performance SLAs
   - Security requirements
   - Accessibility standards
   - Error handling expectations

3. **Integration Points:**
   - Frontend ↔ Backend API
   - Backend ↔ PostgreSQL
   - Backend ↔ Redis
   - Backend ↔ MCP Server
   - SSE event streaming

---

### Step 3: Create Test Plan Structure

Generate the following files in `project/0.5-test/`:

#### 3.1 Master Test Plan
**File:** `project/0.5-test/00-master-test-plan.md`

Contents:
- Executive summary
- Test scope and objectives
- Test strategy overview
- Quality gates and exit criteria
- Test environment requirements
- Risk assessment and mitigation
- Test schedule alignment with sprints
- Roles and responsibilities

#### 3.2 Unit Test Plan
**File:** `project/0.5-test/01-unit-test-plan.md`

Contents:
- Unit test strategy
- Component coverage matrix
- Frontend unit tests (React components, hooks, stores)
- Backend unit tests (API handlers, services, utilities)
- Database unit tests (Prisma models, queries)
- Mocking strategies
- Coverage targets (minimum 80% line coverage)

#### 3.3 Integration Test Plan
**File:** `project/0.5-test/02-integration-test-plan.md`

Contents:
- Integration test strategy
- API integration tests
- Database integration tests
- Redis integration tests
- MCP client integration tests
- Service-to-service communication tests
- Test data management

#### 3.4 End-to-End Test Plan
**File:** `project/0.5-test/03-e2e-test-plan.md`

Contents:
- E2E test strategy (Playwright)
- Critical user journeys
- File upload workflow tests
- URL processing workflow tests
- Results viewing tests
- History management tests
- Cross-browser testing matrix
- Mobile/responsive testing

#### 3.5 Performance Test Plan
**File:** `project/0.5-test/04-performance-test-plan.md`

Contents:
- Performance test strategy
- Load testing scenarios
- Stress testing scenarios
- Baseline metrics and SLAs
- File upload performance (various sizes)
- Concurrent user testing
- Database query performance
- API response time targets

#### 3.6 Security Test Plan
**File:** `project/0.5-test/05-security-test-plan.md`

Contents:
- Security test strategy
- OWASP Top 10 coverage
- Input validation tests
- Authentication/authorization tests
- File upload security (malicious files, size limits)
- API security (rate limiting, injection prevention)
- Data protection tests

#### 3.7 Test Cases Index
**File:** `project/0.5-test/06-test-cases-index.md`

Contents:
- Complete test case inventory
- Test case ID naming convention
- Traceability matrix (requirements → test cases)
- Priority classification (P0, P1, P2)
- Automation status tracking

---

### Step 4: Define Test Case Format

Each test plan should use this test case format:

```markdown
### TC-[CATEGORY]-[NUMBER]: [Test Name]

**Priority:** P0/P1/P2
**Type:** Unit/Integration/E2E/Performance/Security
**Component:** [Component being tested]
**Sprint:** [Related sprint]

**Preconditions:**
- [List preconditions]

**Test Steps:**
1. [Step 1]
2. [Step 2]
3. [Step n]

**Expected Results:**
- [Expected outcome]

**Automation Status:** Manual/Automated/To Be Automated
**Related Requirements:** [Requirement IDs]
```

---

### Step 5: Sprint-to-Test Mapping

Create traceability from sprints to test coverage:

| Sprint | Features | Test Categories | Test Cases |
|--------|----------|-----------------|------------|
| 0 | Prerequisites | Infrastructure | TC-INFRA-* |
| 1.1 | Scaffold | Unit, Setup | TC-UNIT-* |
| 1.2 | Database/Redis | Integration | TC-INT-DB-*, TC-INT-REDIS-* |
| 1.3 | Upload | E2E, Integration | TC-E2E-UPLOAD-* |
| 1.4 | URL Input | E2E, Integration | TC-E2E-URL-* |
| 1.5a | MCP Client | Integration | TC-INT-MCP-* |
| 1.5b | SSE Progress | Integration, E2E | TC-INT-SSE-*, TC-E2E-PROGRESS-* |
| 1.6 | Results Viewer | E2E, Unit | TC-E2E-RESULTS-* |
| 1.7 | History | E2E, Integration | TC-E2E-HISTORY-* |
| 1.8 | Testing & Docs | All | Validation |

---

### Output

After generating all test plan files, provide a summary:

1. List of files created
2. Total test case count by category
3. Coverage assessment
4. Identified testing risks
5. Recommended next steps

---

## Mode: Review (`/test-plan review`)

When `$ARGUMENTS` is `review`, orchestrate a full team review of the test plans.

### Review Process

1. **Ensure test plans exist** - Check that `project/0.5-test/` contains the test plan files. If not, inform the user to run `/test-plan generate` first.

2. **Create reviews directory** - Ensure `project/0.5-test/reviews/` exists.

3. **Invoke team members in parallel** - Use the Task tool to launch multiple agents simultaneously, each reviewing the test plans from their specialist perspective.

### Team Review Assignments

Launch the following agents **in parallel** using the Task tool:

| Agent | Subagent Type | Review Focus | Output File |
|-------|---------------|--------------|-------------|
| Neo | `neo` | Frontend testing (Next.js, React, Playwright E2E) | `01-neo-nextjs-review.md` |
| Trinity | `trinity` | Database testing (PostgreSQL, Prisma, data integrity) | `02-trinity-postgresql-review.md` |
| James | `james` | MCP integration testing (Docling MCP, tool calls) | `03-james-mcp-review.md` |
| William | `william` | Infrastructure testing (deployment, environments) | `04-william-infrastructure-review.md` |
| Ola | `ola` | UI/UX testing (accessibility, usability, responsive) | `05-ola-frontend-review.md` |
| Gordon | `gordon` | Component testing (shadcn/ui, design system) | `06-gordon-shadcn-review.md` |
| Sri | `sri` | Cache testing (Redis, session, performance) | `07-sri-redis-review.md` |
| George | `george` | MCP gateway testing (FastMCP, protocol compliance) | `08-george-fastmcp-review.md` |
| Bob | `bob` | API testing (FastAPI, validation, error handling) | `09-bob-api-review.md` |
| Sophia | `sophia` | Orchestration testing (workflows, state management) | `10-sophia-orchestration-review.md` |

### Review Prompt Template

Each agent should receive this prompt structure:

```
You are [Agent Name], the [Role] specialist. Review the test plans in `project/0.5-test/` from your area of expertise.

**Files to Review:**
- `project/0.5-test/00-master-test-plan.md`
- `project/0.5-test/01-unit-test-plan.md`
- `project/0.5-test/02-integration-test-plan.md`
- `project/0.5-test/03-e2e-test-plan.md`
- `project/0.5-test/04-performance-test-plan.md`
- `project/0.5-test/05-security-test-plan.md`
- `project/0.5-test/06-test-cases-index.md`

**Your Review Focus:** [Specific focus area]

**Review Template:**
Write your review to `project/0.5-test/reviews/[output-file]` with:

1. **Executive Summary** - Overall assessment (1-2 paragraphs)

2. **Coverage Analysis**
   - What is well covered in your domain
   - What is missing or inadequate

3. **Specific Findings**
   - List each finding with severity (Critical/Major/Minor/Suggestion)
   - Reference specific test plan sections

4. **Recommended Test Cases**
   - Suggest additional test cases for your domain
   - Use the TC-[CATEGORY]-[NUMBER] format

5. **Risk Assessment**
   - Testing risks in your area
   - Mitigation recommendations

6. **Approval Status**
   - [ ] Approved
   - [ ] Approved with conditions
   - [ ] Requires revision

**Sign your review with your agent name and date.**
```

### Post-Review Consolidation

After all agent reviews complete:

1. **Create consolidated review** - Generate `project/0.5-test/reviews/00-consolidated-review.md` summarizing:
   - Overall approval status from all reviewers
   - Critical findings across all reviews
   - Consolidated list of missing test cases
   - Priority action items

2. **Report summary** to the user with:
   - Number of reviews completed
   - Approval statuses
   - Critical issues requiring attention
   - Next steps

---

## Example Usage

```bash
# Generate the full test plan
/test-plan

# Generate the full test plan (explicit)
/test-plan generate

# Check test plan status
/test-plan status

# Trigger team review of test plans
/test-plan review
```
